# 系统配置参考

## 配置文件

路径：`~/.config/container/config.toml`

首次运行时自动创建，手动编辑后需重启服务生效。

## 配置项

### Registry

```toml
[registry]
domain = "your-registry.example.com"   # 默认 registry（默认: Docker Hub）
```

### DNS

```toml
[dns]
domains = ["test", "local"]            # 自动创建的本地 DNS 域
```

### Builder

```toml
[builder]
cpus = 4                               # Builder VM CPU 数
memory = "8g"                          # Builder VM 内存
```

### 完整示例

```toml
# ~/.config/container/config.toml

[registry]
domain = "ghcr.io"

[builder]
cpus = 8
memory = "32g"

[dns]
domains = ["dev", "test"]
```

## 系统服务管理

```bash
# 启动/停止/查看系统服务
container system start
container system stop
container system version --format json

# DNS 域管理
sudo container system dns create <domain>
sudo container system dns create <domain> --localhost <ip>  # localhost 域
sudo container system dns delete <domain>
container system dns list
```

## 内核管理

首次 `container system start` 自动提示安装内核：

```
Install the recommended default kernel from 
[https://github.com/kata-containers/kata-containers/releases/...]? [Y/n]: y
```

内核来源：Kata Containers 项目（专为轻量级 VM 优化）。

## 网络配置

### DNS Resolver

container 使用 macOS DNS resolver 系统：
- DNS 配置文件：`/etc/resolver/<domain>`
- 创建 DNS 域需要管理员权限（sudo）
- 修改 DNS 后 macOS 自动重载 resolver 配置

### localhost 域限制

创建 `--localhost` 类型的 DNS 域时：
- 会禁用 iCloud Private Relay
- 相关 packet filter 规则在系统重启后被清除，需重新配置

## Builder VM 配置

Builder 是独立的 VM，单独管理资源：

```bash
# 查看当前 builder 状态
container builder inspect

# 重新配置 builder 资源
container builder stop
container builder delete
container builder start --cpus <n> --memory <size>
```

## 调试与诊断

```bash
# 启用全局调试
container --debug <command>

# 版本信息
container system version
container system version --format json

# 容器详细信息
container inspect <id>

# 启动日志（排查 VM 启动问题）
container logs --boot <id>
```

## 已知限制

- **macOS 15**: 容器间网络访问不可用，需 macOS 26+
- **Private Relay**: localhost DNS 域与之冲突
- **重启丢失**: localhost DNS 的 packet filter 规则在重启后需重新配置
- **架构**: 默认 arm64，多架构镜像可指定 `-a amd64`

## 网络代理环境变量

container 本身不支持在 `config.toml` 中设置全局容器环境变量（每个容器独立 VM），但可以通过以下方式实现自动代理注入：

### 1. 代理配置文件

创建 `~/.config/container/proxy.env`：

```bash
HTTP_PROXY=http://192.168.64.1:7892
HTTPS_PROXY=http://192.168.64.1:7892
ALL_PROXY=http://192.168.64.1:7892
NO_PROXY=localhost,127.0.0.1,192.168.64.0/24
```

### 2. 自动注入脚本

创建 `~/.config/container/container-proxy` 脚本并 source 到 `~/.zshrc`：

```bash
source ~/.config/container/container-proxy
```

脚本提供 `container_run_proxy` 函数，自动读取 `proxy.env` 并注入 `-e` 参数到 `container run` 中。

### 3. 使用别名

```bash
# 在 .zshrc 中定义
alias crr='container_run_proxy'
alias crpit='container_run_proxy_it'

# 使用方式与 container run 完全一致，代理自动注入
crr -d --name nginx -p 8080:80 nginx:alpine
crpit alpine sh  # 交互式，自动带 --rm
```

详细用法参考 [how-to.md](how-to.md) 的「macOS 系统代理转发」章节。
