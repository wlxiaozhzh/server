# OrbStack n8n 安装与 Docker 连接问题排查

当你在 macOS 上使用 OrbStack 运行 `docker pull n8nio/n8n:latest` 或尝试安装 n8n 时遇到以下错误：

```
Cannot connect to the Docker daemon at unix:///Users/<username>/.orbstack/run/docker.sock. Is the docker daemon running?
```

可以按照下面的步骤排查并解决问题。

## 1. 确认 OrbStack 的 Docker 服务已启动
1. 在菜单栏打开 OrbStack，确认 **Docker** 状态为 **Running**。如果是 Stopped，点击 **Start**。
2. 在终端执行 `orbstack status` 或 `docker info`，确保没有报错。如果 `docker info` 仍提示无法连接，重启 OrbStack：
   ```bash
   orbstack restart docker
   ```

## 2. 检查 OrbStack 套接字的权限与路径
1. 确认当前用户拥有访问 `~/.orbstack/run/docker.sock` 的权限：
   ```bash
   ls -l ~/.orbstack/run/docker.sock
   ```
2. 如果权限不足，可以重新加载 OrbStack 的 Docker 组件：
   ```bash
   orbstack restart docker
   ```
   或退出 OrbStack 后重新启动。

## 3. 确保未运行冲突的 Docker 服务
在部分机器上同时安装了 Docker Desktop。确保 Docker Desktop 没有在后台运行，以免与 OrbStack 提供的 Docker 套接字冲突。

## 4. 重置 OrbStack 的 Docker 环境
如果上述步骤无效，可以尝试重置：
```bash
orbstack reset docker
```
该命令会重新创建 Docker 环境，但不会删除 OrbStack 中的其他虚拟机或设置。

## 5. 重新执行 n8n 拉取与启动
1. 再次执行拉取命令：
   ```bash
   docker pull n8nio/n8n:latest
   ```
2. 使用以下命令启动 n8n（示例）：
   ```bash
   docker run -it --rm \
     -p 5678:5678 \
     -v ~/.n8n:/home/node/.n8n \
     n8nio/n8n:latest
   ```

## 6. 使用 Surge 代理时的注意事项
1. 确认 Surge 的增强模式或网关已对 `docker` 进程开放。
2. 如果拉取镜像失败但可以访问普通网络，可以在 Surge 中为 `docker` 进程关闭代理或添加直连规则。

## 7. 收集日志以便进一步诊断
若问题依旧，建议导出以下信息以便寻求社区帮助：
- `orbstack diagnose --verbose`
- `docker info`
- `docker version`

将这些信息附在求助帖或 Issues 中，可以加快定位问题。
