# Container CLI 命令参考

## 全局选项

```
container [--debug] [--version] [-h|--help]
```

- `--debug` — 启用调试输出（环境变量: `CONTAINER_DEBUG`）
- `--version` — 显示 CLI 版本（单行）
- 详细版本：`container system version [--format json|table]`

## 容器命令 (container)

### run
运行容器。未指定命令时执行镜像默认命令。

```bash
container run [options] <image> [arguments...]
```

**进程选项：**
- `-e, --env <env>` — 环境变量（key=value，或仅 key 继承主机）
- `--env-file <file>` — 从文件读取环境变量
- `--gid <gid>` — 设置进程 GID
- `-i, --interactive` — 保持 stdin 打开
- `-t, --tty` — 分配 TTY
- `-u, --user <user>` — 设置用户（name|uid[:gid]）
- `--uid <uid>` — 设置 UID
- `--ulimit <limit>` — 资源限制（type=soft[:hard]）
- `-w, --workdir, --cwd <dir>` — 工作目录

**资源选项：**
- `-c, --cpus <cpus>` — CPU 数量（默认 4）
- `-m, --memory <memory>` — 内存（默认 1G，1MiB 粒度，支持 K/M/G/T/P 后缀）

**管理选项：**
- `-a, --arch <arch>` — 架构（默认 arm64）
- `--cap-add <cap>` — 添加 Linux capability
- `--cap-drop <cap>` — 移除 Linux capability
- `--create-group` — 创建 `--gid` 指定的组
- `-d, --detach` — 后台运行
- `--device <device>` — 添加主机设备到容器
- `--gidmap <gidmap>` — 自定义 GID 映射
- `--group <group>` — 添加用户组
- `--hostname <hostname>` — 设置主机名
- `--init` — 使用 init 进程
- `--network <network>` — 指定网络（默认: default）
- `--new-user-ns` — 在新用户命名空间运行
- `--no-vsock` — 禁用 vsock
- `--read-only` — 只读根文件系统
- `--rm` — 停止后自动删除
- `--root-disk <disk>` — 指定根磁盘镜像
- `--stop-signal <signal>` — 停止信号（默认 SIGTERM）
- `--stop-timeout <seconds>` — 停止超时（默认 30）
- `--uidmap <uidmap>` — 自定义 UID 映射
- `-v, --volume <volume>` — 挂载卷（host:container）

**网络选项：**
- `--mac <address>` — 自定义 MAC 地址
- `-p, --publish <port>` — 端口转发（[host-ip:]host-port:container-port）

**其他选项：**
- `--ssh` — 挂载 SSH 认证 socket
- `--startup-root-disk <disk>` — 启动时根磁盘

### create
创建容器但不启动。

```bash
container create [options] <image> [arguments...]
```

选项与 `run` 相同（除 `--rm` 外）。

### start
启动已创建的容器。

```bash
container start <id|name>
```

### stop
停止容器（发送 SIGTERM，超时后 SIGKILL）。

```bash
container stop [options] <id|name>...
  -t, --time <seconds>  超时时间（默认 30）
```

### kill
强制终止容器。

```bash
container kill [options] <id|name>...
  -s, --signal <signal>  发送的信号（默认 SIGKILL）
```

### delete / rm
删除容器。

```bash
container delete [options] <id|name>...
  -f, --force  强制删除运行中的容器
```

### list / ls
列出容器。

```bash
container list [options]
  -a, --all      显示所有容器（包括停止的）
  -f, --format   输出格式（json|table，默认 table）
  -n, --no-trunc 不截断输出
  -q, --quiet    仅显示 ID
```

输出列：ID, IMAGE, OS, ARCH, STATE, IP

### exec
在运行中的容器内执行命令。

```bash
container exec [options] <id|name> <command> [arguments...]
  -i, --interactive  交互模式
  -t, --tty          分配 TTY
```

### inspect
显示容器详细信息。

```bash
container inspect [options] <id|name>...
  -f, --format <format>  Go 模板格式化输出
```

### logs
获取容器 stdio 或启动日志。

```bash
container logs [options] <id|name>
  --boot    显示启动日志而非 stdio
  -f        跟踪日志输出
  -n <n>    显示最后 n 行
```

### stats
显示容器资源使用统计。

```bash
container stats [options] [id|name...]
  -a, --all       显示所有容器（不仅是运行的）
  --format <fmt>  输出格式（json|table）
  --no-stream     仅输出一次，不持续刷新
```

指标：Cpu %, Memory Usage/Limit, Net Rx/Tx, Block I/O, Pids

## 镜像命令 (image)

### build
从 Dockerfile 构建镜像。

```bash
container build [options] <context>
  -f, --file <file>      Dockerfile 路径（默认: <context>/Dockerfile）
  -t, --tag <tag>        镜像标签
  --build-arg <arg>      构建参数
  --no-cache             不使用缓存
  --platform <platform>  目标平台
  --target <target>      多阶段构建目标
```

### image list / image ls
列出镜像。

```bash
container image list [options]
  -a, --all      显示所有镜像
  -f, --format   输出格式
```

### image delete / image rm
删除镜像。

```bash
container image delete [options] <image>...
  -f, --force  强制删除
```

### image tag
为镜像创建新标签。

```bash
container image tag <source-image> <target-image>
```

### image push
推送镜像到 registry。

```bash
container image push [options] <image>
```

### image pull
从 registry 拉取镜像。

```bash
container image pull [options] <image>
```

## Registry 命令 (registry)

### registry login
登录到 registry。

```bash
container registry login [options] <domain>
  -u, --username <user>  用户名
  -p, --password <pass>  密码
```

### registry logout
登出 registry。

```bash
container registry logout <domain>
```

### registry list
列出已配置的 registry。

```bash
container registry list
```

## Builder 命令 (builder)

管理镜像构建器实例（BuildKit VM）。

```bash
container builder start [options]    # 启动 builder（--cpus, --memory）
container builder stop               # 停止 builder
container builder delete             # 删除 builder
container builder inspect            # 查看 builder 状态
```

默认 builder 资源：2 CPU / 2G 内存。

## 系统命令 (system)

```bash
container system start               # 启动容器系统服务
container system stop                # 停止容器系统服务
container system version             # 显示版本信息
container system dns create <domain> # 创建本地 DNS 域
container system dns delete <domain> # 删除本地 DNS 域
container system dns list            # 列出 DNS 域
```

## Container Machine 命令 (machine)

见 [container-machine.md](container-machine.md) 完整参考。

```bash
container machine create <image> [options]   # 创建 machine
container machine run [options] [command]    # 运行命令/进入 shell
container machine start <name>               # 启动
container machine stop <name>                # 停止
container machine delete <name>              # 删除
container machine list                       # 列出
container machine inspect <name>             # 查看详情
```
