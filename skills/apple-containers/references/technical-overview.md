# 技术架构概览

## 什么是 Containers

容器是将应用及其依赖打包为单一单元的运行环境，提供与主机及其他容器的隔离，使应用能在多种环境中安全高效运行。

容器化是服务端关键技术，贯穿软件全生命周期：
- **开发**: 创建可预测的执行环境
- **CI/CD**: 可重复构建和部署
- **生产**: 编排平台运行容器化应用

OCI（Open Container Initiative）制定镜像和运行时标准，确保不同容器实现之间的互操作性。

## container 如何运行容器

### 架构差异

传统方案（Docker Desktop 等）：
- 启动一个共享 Linux VM
- 所有容器在该 VM 内运行
- 通过 Linux 命名空间隔离

Apple container：
- **每个容器 = 独立轻量级 VM**（使用 Apple Virtualization 框架）
- VM 之间完全隔离
- 消耗和产出标准 OCI 镜像

### 三大特性

**安全性**
- 每个容器拥有完整 VM 的隔离属性
- 最小化核心工具和动态库集合
- 减小资源占用和攻击面

**隐私性**
- 仅将必要数据挂载到每个 VM
- 无需预先挂载所有可能用到的数据

**性能**
- 比完整 VM 更少的内存消耗
- 启动时间与共享 VM 中的容器相当

## 集成的 macOS 技术

| 技术 | 用途 |
|------|------|
| Apple Virtualization | 管理 Linux VM 及附加设备 |
| Hypervisor | VM 的底层虚拟化支持 |
| FileProvider | 文件系统集成 |
| VPN / Network Extension | 网络路由和隔离 |
| Rosetta | x86 二进制转译（Apple Silicon） |

## 镜像构建

使用 BuildKit 构建镜像，语法与 Dockerfile 完全兼容：

```bash
container build --tag myapp -f Dockerfile .
```

Builder 本身也运行在轻量级 VM 中。

## 镜像格式

- 支持 OCI Image Spec 和 Docker Image Spec
- 与 Docker Hub、GHCR 等标准 Registry 兼容
- 支持多架构镜像（arm64、amd64 等）

## macOS 版本要求

| 功能 | macOS 15 | macOS 26+ |
|------|---------|-----------|
| 基本容器运行 | 支持 | 支持 |
| 镜像构建 | 支持 | 支持 |
| 容器间网络 | 不支持 | 支持 |
| 完整网络功能 | 受限 | 完整 |

## 与 Docker 的兼容性

| 方面 | Apple container | Docker |
|------|----------------|--------|
| CLI 命令 | `container` | `docker` |
| Dockerfile | 完全兼容 | 原生 |
| OCI 镜像 | 完全兼容 | 完全兼容 |
| Compose | 不原生支持 | `docker compose` |
| 架构 | 每容器独立 VM | 共享 VM + 命名空间 |
| 平台 | macOS only | 跨平台 |

## 启动流程

1. `container system start` 初始化系统服务
2. 首次启动自动下载安装 Linux 内核（来自 kata-containers）
3. 每个 `container run` 创建新的轻量级 VM
4. VM 内运行容器进程
5. 停止时关闭 VM，可选择自动删除
