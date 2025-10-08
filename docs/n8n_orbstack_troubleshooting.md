# OrbStack n8n 安装与 Docker 连接问题排查

当你在 macOS 上使用 OrbStack 运行 `docker pull n8nio/n8n:latest` 或尝试安装 n8n 时，常见的错误包括：

- `Cannot connect to the Docker daemon at unix:///Users/<username>/.orbstack/run/docker.sock. Is the docker daemon running?`
- `Error response from daemon: Get "https://registry-1.docker.io/v2/": net/http: TLS handshake timeout`

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

## 6. 处理 `TLS handshake timeout` 等网络问题
当错误信息为 `TLS handshake timeout` 时，说明 Docker 客户端与 Docker Hub 建立 HTTPS 连接受阻。可以按如下步骤排查：

1. **验证基础网络与 DNS**：
   ```bash
   ping -c 4 registry-1.docker.io
   dig registry-1.docker.io +short
   ```
   若无法解析或延迟极高，尝试切换到稳定的 DNS（如 1.1.1.1、8.8.8.8）或重启网络。

2. **使用 `curl` 测试 HTTPS 连通性**（确保与 Docker 相同的代理设置）：
   ```bash
   HTTPS_PROXY=$HTTPS_PROXY \
   curl -I https://registry-1.docker.io/v2/
   ```
   - 如果 `curl` 也超时，说明代理或网络仍有问题。
   - 如果 `curl` 正常，但 `docker pull` 超时，请继续下一步。

3. **校验 Surge 代理规则**：
   - 将 Surge 切换到直连模式或为 `docker` 进程单独添加直连规则（包含 `registry-1.docker.io`、`auth.docker.io` 等域名）。
   - 在 Surge 日志中确认 `docker` 相关请求是否被正确转发，确保未被拦截或降级为 HTTP/1.0。

4. **为 Docker 显式配置代理环境变量**（OrbStack shell 与 GUI 代理有时不同步）：
   - 在 `~/.zshrc` 或 `~/.bashrc` 中设置：
     ```bash
     export HTTPS_PROXY="http://127.0.0.1:<port>"
     export HTTP_PROXY="http://127.0.0.1:<port>"
     export NO_PROXY="localhost,127.0.0.1,.local"
     ```
   - 重新打开终端后执行 `docker info | grep -i proxy`，确认 Docker CLI 已读取代理。

5. **同步系统时间与证书信任**：
   - 在「系统设置 > 通用 > 日期与时间」中勾选“自动设置日期与时间”。
   - 确保 Surge 或其他代理导入的自签证书已被 macOS 信任，否则 TLS 握手可能失败。

6. **重启 OrbStack 网络栈并刷新连接**：
   ```bash
   orbstack restart network
   ```
   然后关闭终端重新尝试 `docker pull`。

7. **尝试使用镜像加速或临时 VPN**：
   - 若频繁超时，可改用官方推荐的国内镜像或开启可直连 Docker Hub 的 VPN。
   - 临时测试可执行：
     ```bash
     DOCKERHUB_MIRROR=https://registry.docker-cn.com
     docker pull --registry-mirror "$DOCKERHUB_MIRROR" n8nio/n8n:latest
     ```

## 7. 使用 Surge 代理时的其他注意事项
1. 确认 Surge 的增强模式或网关已对 `docker` 进程开放。
2. 如果拉取镜像失败但可以访问普通网络，可以在 Surge 中为 `docker` 进程关闭代理或添加直连规则。

## 8. 收集日志以便进一步诊断
若问题依旧，建议导出以下信息以便寻求社区帮助：
- `orbstack diagnose --verbose`
- `docker info`
- `docker version`
- Surge 代理日志（如适用）

将这些信息附在求助帖或 Issues 中，可以加快定位问题。
