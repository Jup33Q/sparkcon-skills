---
name: apple-containers
description: Apple Container (container) 是 Apple 官方出品的 macOS 原生容器平台。基于 Apple Virtualization 框架，为每个容器运行轻量级独立 VM，提供 Docker 兼容的 OCI 容器生命周期管理。支持容器镜像构建 (Dockerfile)、Registry 操作、容器网络 (DNS/端口转发)、Volume 挂载、Linux capabilities 控制、资源限制、容器机器 (container machine) 持久化 Linux 环境等完整功能。要求 macOS 26+。在以下场景触发：执行容器生命周期管理 (run/create/start/stop/delete)、镜像操作 (build/push/pull/tag)、registry 认证、容器网络与存储配置、资源与安全管理、container machine 模式使用、系统配置与故障排查。
---

# Apple Containers Skill

## 快速判断

- **平台要求**: macOS 26+（部分功能在 macOS 15 受限）
- **CLI 工具**: `container`（非 `docker`）
- **核心架构**: 每个容器 = 独立轻量级 VM（Apple Virtualization 框架）
- **镜像标准**: OCI 兼容，可与 Docker 互通

## 安装与启动

```bash
# 安装 container CLI（通过 Xcode Command Line Tools 或其他方式）
# 启动容器系统服务
container system start
# 首次启动会提示安装 Linux 内核，确认即可

# 验证安装
container system version --format json
container list --all
```

## 核心工作流

### 1. 镜像管理

```bash
# 从 Dockerfile 构建镜像
container build --tag <image-name> --file <Dockerfile> <context>

# 列出现有镜像
container image list

# Tag 镜像
container image tag <source> <target>

# 推送/拉取
container image push <image:tag>
container image pull <image:tag>

# 删除镜像
container image delete <image>
```

### 2. 容器生命周期

```bash
# 运行容器（常用选项组合）
container run --name <name> --detach --rm <image>
container run -it --rm <image>                    # 交互式 TTY
container run -d --rm -p <host>:<container> <image>  # 端口映射

# 容器管理
container list [--all]
container start <id|name>
container stop <id|name>
container kill <id|name>
container delete <id|name>

# 执行命令
container exec [-it] <id|name> <command>

# 查看日志
container logs <id|name>

# 资源监控
container stats [--no-stream] [--format json] [id|name...]
```

### 3. Registry 认证

```bash
# 登录（默认使用 Docker Hub，可通过配置修改）
container registry login <registry-domain>

# 登出
container registry logout <registry-domain>

# 配置默认 registry：编辑 ~/.config/container/config.toml
# [registry]
# domain = "your-registry.example.com"
```

### 4. 网络与存储

```bash
# 端口转发（IPv4/IPv6 均支持）
container run -p '[::1]:8080:80' <image>
container run -p '127.0.0.1:8080:80' <image>

# Volume 挂载（host:container）
container run --volume ${HOME}/data:/content/data <image>

# SSH agent 转发
container run --ssh <image>

# DNS 域名（可选，便于容器间访问）
sudo container system dns create <domain-name>
# 设置后可通过 --name myapp 以 myapp.<domain> 访问
```

### 5. 网络代理（自动使用 macOS 系统代理）

Apple Container 容器可以自动使用 macOS 主机的网络代理（如 Clash、Surge、V2Ray 等）。容器通过网关 `192.168.64.1` 访问主机的代理端口。

详细配置见 [references/how-to.md](references/how-to.md) 的「macOS 系统代理转发」章节。

### 5. 资源配置

```bash
# 容器资源限制
container run --cpus <n> --memory <size> <image>
# 内存单位：K, M, G, T, P（默认 1G，1MiB 粒度）

# Builder 资源（镜像构建 VM）
container builder start --cpus <n> --memory <size>
container builder stop
container builder delete
```

## 系统配置

配置文件路径：`~/.config/container/config.toml`

关键配置项：
- `[registry] domain` — 默认 registry
- DNS 域名配置
- Builder 资源预设

完整配置参考见 [references/system-config.md](references/system-config.md)。

## 主题导航

| 主题 | 参考文档 |
|------|---------|
| 完整 CLI 命令参考（所有子命令和选项） | [references/command-reference.md](references/command-reference.md) |
| 实用技巧（网络/存储/资源/Capabilities/监控） | [references/how-to.md](references/how-to.md) |
| Container Machine 持久化 Linux 环境 | [references/container-machine.md](references/container-machine.md) |
| 技术架构与运行原理 | [references/technical-overview.md](references/technical-overview.md) |
| 系统配置参考 | [references/system-config.md](references/system-config.md) |

## 故障排查

```bash
# 查看详细版本信息
container system version --format json

# 启用调试输出
container --debug <command>

# 查看容器详细信息
container inspect <id|name>

# 查看容器启动日志
container logs <id|name>
```

常见问题：
- **macOS 15 限制**: 容器间网络访问需要 macOS 26+
- **内核安装**: 首次 `container system start` 会自动提示
- **Private Relay 冲突**: localhost DNS 域会禁用 Private Relay
- **权限不足**: DNS 配置和部分网络操作需要 `sudo`
