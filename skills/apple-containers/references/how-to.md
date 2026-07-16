# How-to 实用技巧

## 资源管理

### 配置容器资源

默认：1G 内存 / 4 CPU。

```bash
container run --rm --cpus 8 --memory 32g <image>
```

内存单位：K, M, G, T, P（1MiB 粒度）。

### 配置 Builder 资源

默认 builder：2G 内存 / 2 CPU。资源密集型构建需要增加：

```bash
# 首次自定义
container builder start --cpus 8 --memory 32g

# 修改已有 builder
container builder stop
container builder delete
container builder start --cpus 8 --memory 32g
```

## 存储

### Volume 挂载

```bash
# 基本用法：host:container
container run --volume ${HOME}/Desktop/assets:/content/assets <image>

# 挂载后文件权限保持不变
```

### SSH Agent 转发

```bash
container run -it --rm --ssh <image>
```

等效于：
`--volume "${SSH_AUTH_SOCK}:/run/host-services/ssh-auth.sock" --env SSH_AUTH_SOCK=/run/host-services/ssh-auth.sock`

`--ssh` 优势：重新登录后自动更新 socket 路径。

## 网络

### 端口转发

```bash
# IPv4
container run -d --rm -p '127.0.0.1:8080:80' <image>

# IPv6
container run -d --rm -p '[::1]:8080:8000' <image>
curl -6 'http://[::1]:8080'

# 所有接口
container run -d --rm -p '8080:80' <image>
```

### DNS 域名（容器间访问）

```bash
# 创建本地 DNS 域（需要 sudo）
sudo container system dns create test

# 使用 --name 启动的容器可通过 <name>.test 访问
container run --name my-web-server -d <image>
container run -it --rm <image> curl http://my-web-server.test
```

自定义域名：`~/.config/container/config.toml`

### 从容器访问主机服务

```bash
# 创建 localhost 域（需要 sudo）
sudo container system dns create host.container.internal --localhost 203.0.113.113

# 容器内通过该域名访问主机
curl http://host.container.internal:8000
```

**注意**：此功能会禁用 Private Relay，且重启后需重新配置。

### 自定义 MAC 地址

```bash
container run --network default,mac=02:42:ac:11:00:02 ubuntu:latest
```

未指定时自动生成，格式为 `fX:XX:XX:XX:XX:XX`。

## 安全

### Linux Capabilities

默认已授予：
`CAP_AUDIT_WRITE`, `CAP_CHOWN`, `CAP_DAC_OVERRIDE`, `CAP_FOWNER`, `CAP_FSETID`, `CAP_KILL`, `CAP_MKNOD`, `CAP_NET_BIND_SERVICE`, `CAP_NET_RAW`, `CAP_SETFCAP`, `CAP_SETGID`, `CAP_SETPCAP`, `CAP_SETUID`, `CAP_SYS_CHROOT`

```bash
# 添加能力（支持 CAP_ 前缀、大小写不敏感）
container run --cap-add NET_ADMIN alpine ip link set lo down
container run --cap-add CAP_NET_ADMIN alpine ip link set lo down

# 授予全部能力
container run --cap-add ALL alpine sh

# 移除全部再选择性添加（推荐最小权限模式）
container run --cap-drop ALL --cap-add SETUID --cap-add SETGID alpine id

# 移除单个能力
container run --cap-drop NET_RAW alpine sh
```

处理顺序：先 drop 后 add，`--cap-drop ALL --cap-add ALL` = 全部授予。

### 只读根文件系统

```bash
container run --read-only <image>
```

## 监控

### 实时资源监控

```bash
# 交互式实时显示（类似 top）
container stats

# 指定容器
container stats my-web-server db

# 单次快照
container stats --no-stream my-web-server

# JSON 格式输出
container stats --format json --no-stream my-web-server | jq
```

指标说明：
- **Cpu %**: ~100% = 一个核心满负荷，多核可 >100%
- **Memory Usage**: 当前使用 / 限制
- **Net Rx/Tx**: 网络收发字节
- **Block I/O**: 磁盘读写字节
- **Pids**: 容器内进程数

## 多架构

```bash
# 指定架构（镜像支持多架构时）
container run -a amd64 <multi-arch-image>
```

默认使用 arm64。

## 交互式容器工作流

```bash
# 进入容器 shell
container exec -it <name> sh

# 运行单次命令
container exec <name> ls /content

# 查看容器内环境
container exec <name> uname -a
```

## 日志管理

```bash
# 查看应用输出
container logs <name>

# 跟踪日志
container logs -f <name>

# 查看启动日志（排查启动问题）
container logs --boot <name>

# 最后 N 行
container logs -n 50 <name>
```

## 清理

```bash
# 停止并删除容器
container stop <name>
container delete <name>

# 强制删除运行中的容器
container delete -f <name>

# 删除镜像
container image delete <image>

# 批量清理（停止的容器）
container list -a -q | xargs container delete
```

## macOS 系统代理转发

Apple Container 中每个容器运行在独立的轻量级 VM 中，容器网络通过 `bridge100` 虚拟网桥与 macOS 主机通信。主机在该网桥上的 IP 固定为 **`192.168.64.1`**，因此容器可以通过此地址访问 macOS 上运行的代理服务（Clash、Surge、V2Ray、Shadowrocket 等）。

### 原理

| 组件 | 地址 | 说明 |
|------|------|------|
| 主机网关（bridge100） | `192.168.64.1` | 容器默认网关，稳定不变 |
| 容器默认路由 | `default via 192.168.64.1` | 容器出网流量必经 |
| DNS | `192.168.64.1` | 由主机 DNS 转发 |
| 代理端口 | 取决于代理软件 | 常见：7890(HTTP)、7891(SOCKS5)、7892(混合) |

### 快速检测主机代理端口

```bash
# 查看 macOS 系统代理配置（确认 HTTP 代理端口）
networksetup -getwebproxy Wi-Fi 2>/dev/null || networksetup -getwebproxy "Ethernet"

# 检测常见代理端口监听状态
netstat -an | grep LISTEN | grep -E '789[0-3]|8080|108[0-9]|9090'
```

### 方案 A：单次启动时手动注入（适合临时使用）

```bash
container run -d \
  --name my-app \
  -e HTTP_PROXY=http://192.168.64.1:7892 \
  -e HTTPS_PROXY=http://192.168.64.1:7892 \
  -e ALL_PROXY=http://192.168.64.1:7892 \
  -e NO_PROXY=localhost,127.0.0.1,192.168.64.0/24 \
  <镜像>
```

### 方案 B：Shell 别名自动注入（推荐日常使用）

在 `~/.zshrc` 或 `~/.bash_profile` 中添加：

```bash
# Apple Container 代理自动注入
container_run_with_proxy() {
  container run \
    -e HTTP_PROXY=http://192.168.64.1:7892 \
    -e HTTPS_PROXY=http://192.168.64.1:7892 \
    -e ALL_PROXY=http://192.168.64.1:7892 \
    -e NO_PROXY=localhost,127.0.0.1,192.168.64.0/24 \
    "$@"
}

alias crr='container_run_with_proxy'
```

使用方式：

```bash
# 用法与 container run 完全一致，代理自动注入
crr -d --name nginx -p 8080:80 nginx:alpine
crr -it --rm alpine sh
```

### 方案 C：代理配置文件 + 自动读取脚本（适合团队共享）

1. 创建代理配置文件 `~/.config/container/proxy.env`：

```bash
mkdir -p ~/.config/container

cat > ~/.config/container/proxy.env << 'EOF'
# Apple Container 代理配置
# 主机网关固定为 192.168.64.1（bridge100 虚拟网桥）
HTTP_PROXY=http://192.168.64.1:7892
HTTPS_PROXY=http://192.168.64.1:7892
ALL_PROXY=http://192.168.64.1:7892
NO_PROXY=localhost,127.0.0.1,192.168.64.0/24
EOF
```

2. 添加辅助函数到 `~/.zshrc`：

```bash
# 读取代理配置并自动注入到 container run
container_run_proxy() {
  local env_file="${HOME}/.config/container/proxy.env"
  if [[ -f "$env_file" ]]; then
    # 读取 .env 文件并构造 -e 参数
    local env_args=()
    while IFS= read -r line || [[ -n "$line" ]]; do
      # 跳过空行和注释
      [[ -z "$line" || "$line" =~ ^# ]] && continue
      env_args+=("-e" "$line")
    done < "$env_file"
    container run "${env_args[@]}" "$@"
  else
    echo "Warning: proxy.env not found, running without proxy" >&2
    container run "$@"
  fi
}

alias crp='container_run_proxy'
```

使用方式：

```bash
# 修改代理端口后，只需编辑 ~/.config/container/proxy.env
crp -d --name my-app alpine
```

### 方案 D：为现有运行中容器临时启用代理

```bash
# 进入容器手动设置（仅当前 shell 会话有效）
container exec -it <容器名> sh

export HTTP_PROXY=http://192.168.64.1:7892
export HTTPS_PROXY=http://192.168.64.1:7892
export ALL_PROXY=http://192.168.64.1:7892
export NO_PROXY=localhost,127.0.0.1,192.168.64.0/24
```

### 验证代理是否生效

```bash
# 在容器内测试外网访问（HTTP 代理连通性）
container exec <容器名> sh -c 'apk add --no-cache curl && curl -s https://ip.sb'

# 对比：不经过代理时容器能否直接访问（用于排查）
container exec <容器名> sh -c 'curl -s --noproxy "*" https://ip.sb'
```

### 常见代理软件端口速查

| 软件 | HTTP 代理 | SOCKS5 | 混合/Mixed | 管理/API |
|------|-----------|--------|-----------|----------|
| Clash Verge | 7890 | 7891 | 7892 | 9090 |
| ClashX / ClashX Pro | 7890 | 7891 | 7892 | 9090 |
| Surge | 6152 | 6153 | — | 6170 |
| V2RayN / V2RayU | 10808 | 10808 | 10809 | — |
| ShadowsocksX-NG | 1087 | 1086 | — | — |
| Qv2ray | 8889 | 8889 | — | — |

> **注意**：请根据你实际使用的代理软件调整端口。通过 `networksetup -getwebproxy Wi-Fi` 可查看系统配置的 HTTP 代理端口。
