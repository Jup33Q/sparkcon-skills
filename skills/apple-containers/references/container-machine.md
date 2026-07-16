# Container Machine

Container machine 提供高度集成的持久化 Linux 环境，基于标准 OCI 镜像，启动快速轻量。

## 核心概念

| 特性 | 普通容器 | Container Machine |
|------|---------|-------------------|
| 设计模型 | 单一应用 | 完整 Linux 环境 |
| 生命周期 | 临时/短期 | 持久/长期 |
| init 系统 | 单一进程 | 完整 init（支持 systemd） |
| 用户映射 | root 或指定 | 自动映射主机用户名 |
| 家目录 | 隔离 | 自动挂载主机 $HOME |
| 适用场景 | 运行应用 | 开发环境/多发行版测试 |

## 快速开始

```bash
# 创建 machine
container machine create alpine:latest --name dev

# 运行命令
container machine run -n dev whoami    # 输出你的主机用户名，非 root
container machine run -n dev pwd       # /home/<you> — Mac 家目录挂载

# 交互式 shell
container machine run -n dev
```

## 命令参考

### create

```bash
container machine create <image> [options]
  -n, --name <name>       machine 名称（必需）
  -c, --cpus <cpus>       CPU 数量
  -m, --memory <memory>   内存
  --size <disk-size>      磁盘大小（默认 20G）
  --ssh                   挂载 SSH agent
  -v, --volume <volume>   额外挂载卷
```

### run

```bash
container machine run [options] [command]
  -n, --name <name>  machine 名称
  -i, --interactive  交互模式
  -t, --tty          分配 TTY
  -u, --user <user>  指定用户
  --workdir <dir>    工作目录
```

无 command 时打开交互式 shell。Machine 停止时 `run` 会自动启动。

### start / stop / delete

```bash
container machine start <name>
container machine stop <name>
container machine delete <name>
```

### list / inspect

```bash
container machine list
container machine inspect <name>
```

### 设置默认 machine

```bash
container machine set-default <name>
```

设置后无需每次指定 `-n` 名称。

## 工作模式

### Edit on Mac, Build Inside

- Mac 上的 repo 位于 `$HOME`，在 machine 内挂载为 `/Users/<username>`
- 使用 macOS 编辑器/IDE，在 machine 内编译运行
- 无 "build→copy→inspect" 步骤

### 多发行版并行

```bash
# 为每个目标发行版创建一个 machine
container machine create alpine:latest --name alpine-dev
container machine create ubuntu:latest --name ubuntu-dev
container machine create debian:latest --name debian-dev

# 快速测试应用兼容性
container machine run -n alpine-dev -- ./test.sh
container machine run -n ubuntu-dev -- ./test.sh
```

### 系统服务测试

安装 systemd 的镜像支持原生 Linux 服务管理：

```bash
container machine create ubuntu:latest --name svc-test
container machine run -n svc-test sudo systemctl start postgresql
```

## 嵌套虚拟化

Container machine 支持嵌套虚拟化（运行 VM 内的 VM），适用于测试虚拟化场景。

要求：
- 主机支持虚拟化扩展
- 分配足够资源给外层 machine

```bash
container machine create ubuntu:latest --name nested-test --cpus 4 --memory 8g
```

## Volume 与 SSH

Volume 挂载和 SSH agent 转发的语法与普通容器相同：

```bash
container machine create alpine:latest --name dev -v ${HOME}/projects:/projects --ssh
```

## 磁盘管理

创建时可指定磁盘大小：

```bash
container machine create ubuntu:latest --name bigdev --size 100g
```

磁盘按需扩展，可通过 `inspect` 查看使用情况。
