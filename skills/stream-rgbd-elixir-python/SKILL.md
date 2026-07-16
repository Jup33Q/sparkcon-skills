---
name: stream-rgbd-elixir-python
description: >
  Elixir GenServer + Erlang Port + Python backend integration pattern for real-time
  inference services and web dashboards. Covers Port lifecycle management, binary wire
  protocols, Phoenix PubSub broadcasting, LiveView real-time UI, and Membrane multimedia
  pipelines. Use when: building Elixir/Phoenix apps that orchestrate Python processes for
  ML/AI inference, real-time video processing, or any CPU-heavy task that needs a web frontend.
  Trigger keywords: "elixir python port", "genserver python backend", "phoenix python",
  "erlang port python", "real-time inference elixir", "stream rgbd", "elixir python bridge",
  "python worker genserver", "membrane python pipeline".
version: 1.1.0
category: devops
---

# Skill: StreamRGBD — Elixir + Python 实时推理流水线

## 触发条件
当用户需要构建一个 Elixir/Phoenix LiveView + Python 后端的实时 AI 推理系统，特别是涉及视频/图像流处理、GenServer 编排、Erlang Port 通信、Membrane 多媒体管道时触发。

## 概述

本 Skill 总结了 StreamRGBD 项目的架构模式：将 Python 端的高性能 CoreML/AI 推理与 Elixir/Phoenix 的高并发 Web 前端通过 Erlang Port 和 Membrane 管道桥接，实现低延迟的实时视频流处理。

## 架构分层

### 第 1 层：Python 推理后端
- **职责**：运行 CoreML/AI 模型（img2img、深度估计等），处理原始图像输入，输出处理后的图像
- **通信方式**：通过 Erlang Port 的 stdin/stdout 与 Elixir 交换数据，使用 length-prefixed 二进制协议
- **关键模式**：
  - 设计为无头 CLI 子进程，通过 stdin 接收原始帧，通过 stdout 输出 JPEG
  - 使用 `struct.pack("<III", 0, pid, tid)` 在启动时发送 ready 信标
  - 输入协议：`<<width::32-little, height::32-little, rgb_bytes::binary>>`
  - 输出协议：`<<jpeg_size::32-little, jpeg_bytes::binary>>`

### 第 2 层：Elixir OTP 编排层
- **`InferenceWorker` (GenServer)**：拥有 Python 子进程，管理 Port 生命周期，处理帧的进出
- **`VideoStreamer` (GenServer)**：持有最新的 JPEG 帧，为 MJPEG 流消费者提供服务
- **`StreamRGBD` (GenServer)**：总编排器，协调 CameraPipeline → InferenceWorker → VideoStreamer 的启动/停止
- **`CameraPipeline` (Membrane.Pipeline)**：捕获摄像头，转换为 RGB，通过 FrameSink 推送到 InferenceWorker
- **`PipelineAgent` (Agent)**：轻量级配置状态存储

### 第 3 层：Phoenix Web 层
- **LiveView**：替代静态页面，使用 Phoenix PubSub 实时广播状态更新
- **MJPEG 流 Controller**：从 VideoStreamer 读取帧，通过 chunked HTTP 响应推送
- **JSON API**：引擎控制（start/stop/prompt/status）

## 核心协议设计

### Erlang Port Wire Protocol

```elixir
# Elixir → Python (发送帧)
Port.command(port, <<width::32-little, height::32-little, rgb_binary::binary>>)

# Python → Elixir (返回结果)
# 1. Ready 信标（启动完成）：
struct.pack("<III", 0, os.getpid(), threading.current_thread().ident)
# 2. JPEG 结果：
struct.pack("<I", len(jpeg)) + jpeg
```

### 帧大小校验
```elixir
expected_size = width * height * 3
if byte_size(payload) == expected_size do
  Port.command(port, <<width::32-little, height::32-little, payload::binary>>)
else
  Logger.warning("Frame size mismatch...")
end
```

## 关键代码模式

### 1. 启动 Python Worker (GenServer)

**正确方式** — 使用 `{:spawn_executable, ...}` + `args:` 列表（避免参数解析错误）：

```elixir
python_path = "/Users/.../.venv/bin/python"
script_path = "python/inference_worker.py"
args = ["--prompt", "oil painting", "--model", "sdxs", "--render-size", "512"]

port = Port.open({:spawn_executable, python_path}, [
  :binary,
  :packet, 4,
  :exit_status,
  args: [script_path | args]
])
```

**⚠️ 避免** — 使用 `{:spawn, cmd_string}` 拼接命令和参数，会导致 "invalid option in list" 错误：

```elixir
# ❌ 错误 — 不要这样做
cmd = "#{python_path} #{script_path} --prompt ..."
port = Port.open({:spawn, String.to_charlist(cmd)}, [
  :binary, :packet, 4, :exit_status
])
```

### 2. 处理 Port 数据
```elixir
def handle_info({port, {:data, data}}, %{port: port} = state) do
  <<jpeg_size::32-little, jpeg::binary>> = data
  <<jpeg::binary-size(jpeg_size), _rest::binary>> = jpeg
  send(state.video_streamer, {:frame, jpeg})
  {:noreply, %{state | busy: false}}
end
```

### 3. Membrane FrameSink 转发
```elixir
def handle_buffer(:input, buffer, _ctx, state) do
  send(state.consumer, {:camera_frame, buffer.payload, state.width, state.height})
  {[], state}
end
```

### 4. Phoenix PubSub 状态广播
```elixir
# 在 StreamRGBD 中广播状态变更
Phoenix.PubSub.broadcast(
  StreamdiffusionMac.PubSub,
  "stream_rgbd:status",
  {:status_update, status_map}
)

# LiveView 订阅
@impl true
def mount(_params, _session, socket) do
  if connected?(socket) do
    Phoenix.PubSub.subscribe(StreamdiffusionMac.PubSub, "stream_rgbd:status")
  end
  {:ok, socket}
end
```

### 5. MJPEG 流 Controller
```elixir
def video(conn, _params) do
  conn
  |> put_resp_header("content-type", "multipart/x-mixed-replace; boundary=frame")
  |> put_resp_header("cache-control", "no-cache")
  |> send_chunked(200)
  |> stream_mjpeg()
end

 defp stream_mjpeg(conn) do
  frame = VideoStreamer.latest_frame()
  if frame do
    chunk = ["--frame\r\n", "Content-Type: image/jpeg\r\n\r\n", frame, "\r\n"]
    case Plug.Conn.chunk(conn, chunk) do
      {:ok, conn} -> Process.sleep(33); stream_mjpeg(conn)
      {:error, _} -> conn
    end
  else
    Process.sleep(33); stream_mjpeg(conn)
  end
end
```

### 6. Supervisor 配置
```elixir
children = [
  StreamdiffusionMacWeb.Telemetry,
  StreamdiffusionMac.Repo,
  StreamdiffusionMac.PipelineAgent,
  StreamdiffusionMac.VideoStreamer,
  StreamdiffusionMac.InferenceWorker,
  StreamdiffusionMac.StreamRGBD,
  {Phoenix.PubSub, name: StreamdiffusionMac.PubSub},
  StreamdiffusionMacWeb.Endpoint
]

opts = [strategy: :one_for_one, name: StreamdiffusionMac.Supervisor]
Supervisor.start_link(children, opts)
```

## 错误处理模式

1. **Python 进程崩溃**：Port 会收到 `{:exit_status, status}`，GenServer 应该重置状态并通知客户端
2. **帧大小不匹配**：在发送前校验，避免 Python 端解析错误
3. **启动超时**：设置 `Process.send_after(self(), :ready_timeout, @ready_timeout_ms)`，超时后关闭 Port
4. **Back-pressure**：InferenceWorker 在 busy 时只保留最新的 pending frame，丢弃旧帧以保持低延迟

## 性能优化

1. **低延迟**：只保留最新 pending frame，busy 时丢弃中间帧
2. **统一内存**（Apple Silicon）：CoreML 模型在 Python 端运行，充分利用 GPU
3. **JPEG 压缩**：Python 端输出 JPEG 而非原始 RGB，减少传输带宽
4. **Membrane 管道**：摄像头捕获和格式转换在独立进程中完成

## 扩展模式

### 添加 RGBD 深度流
1. 在 Python inference_worker 中添加深度估计输出
2. 修改 wire 协议支持 RGBD 帧（JPEG + depth 通道）
3. 在 VideoStreamer 中支持深度帧存储
4. 添加 `/api/stream/depth` 端点

### 添加 LiveView 实时控制
1. 将 `PageController` 替换为 `StreamRGBDLive`
2. 使用 `handle_event` 处理 start/stop/prompt 事件
3. 使用 PubSub 广播状态更新，避免客户端轮询
4. 使用 `~H"""` 模板定义 HEEx 界面

## 常见陷阱

1. **Port 编码问题**：必须使用 `:binary` 和 `:packet, 4`，避免字符编码转换
2. **Port 参数传递**：使用 `{:spawn_executable, path}` + `args: [...]` 列表，避免字符串拼接导致 "invalid option in list" 错误
3. **Python 路径**：`.venv` 虚拟环境路径必须在运行时正确解析
4. **Membrane 版本**：Membrane 插件版本需要匹配，注意 `ecto_sqlite3` 和 `decimal` 的冲突
5. **CoreML 版本**：`coremltools` 不支持 Python 3.13+
6. **帧格式转换**：Membrane 输出 `:RGB`，Python OpenCV 默认 BGR，需要转换

## 依赖

**Elixir 端：**
- `phoenix_live_view ~> 1.1`
- `membrane_core ~> 1.3`
- `membrane_camera_capture_plugin ~> 0.7`
- `membrane_ffmpeg_swscale_plugin ~> 0.16`
- `ecto_sqlite3 ~> 0.22`

**Python 端：**
- `torch >= 2.1, < 2.6` (coremltools 限制)
- `coremltools >= 7.0`
- `diffusers >= 0.21`
- `numpy < 2.0`
- `opencv-python`
- `depth_anything_3` (可选，用于 DA3)

## 文件参考

- `stream_rgbd.ex` — GenServer 编排器
- `inference_worker.ex` — Python Port 管理
- `video_streamer.ex` — 帧存储
- `camera_pipeline.ex` — Membrane 摄像头管道
- `frame_sink.ex` — Membrane 帧转发 sink
- `stream_rgbd_controller.ex` — JSON/MJPEG API
- `inference_worker.py` — Python 推理子进程

## 相关技能

- `elixir-phx-liveview-ecosystem` — OTP 设计模式、ETS、LiveView 核心、Membrane 框架
- `apple-containers` — 如需在 Apple Container 中运行 Python 后端
- `obsidian-vault-first` — 修改项目前先从 Obsidian 读取上下文
- `obsidian-sync` — 将项目文档和架构决策归档到 Obsidian