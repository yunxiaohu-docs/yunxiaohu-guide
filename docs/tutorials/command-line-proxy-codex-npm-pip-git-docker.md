# 2026 年 10 月命令行代理配置指南：Codex、npm、pip、Git、Docker 怎么走代理

> 本文整理自 [云小狐 官网原文](https://www.yunxiaohu.com/help/cli-proxy-codex-npm-pip-git-docker)，原文更新于 2026-10-16。

## 结论先看

- **最通用的方式**是设置 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY` 和 `NO_PROXY` 环境变量；npm、pnpm、pip、Git 和 Docker 还可以使用各自的配置项。
- 下面的示例默认本机提供 HTTP 代理地址 `http://127.0.0.1:7890`；建议先用 `curl` 验证代理端口、目标站点和出口地区，再定位具体工具的问题。
- Docker 的 CLI、daemon、构建阶段和容器运行时属于不同作用域；`daemon.json` 的代理配置对 Docker Desktop 不生效，应在桌面端设置。

## 为什么现在值得看

本文原文最后更新于 **2026-10-16**，覆盖 Codex CLI、npm、pnpm、pip、Git、Docker 等常见命令行工具的代理配置，以及晚高峰、连接超时、认证失败、地区支持和取消代理后仍生效等常见排查场景。若你近期遇到工具连接失败、配置互相覆盖或 Docker 代理不生效，可以按本文顺序从本机端口、代理协议、工具配置和出口地区逐层确认。

## 一、先确定代理类型和作用范围

命令行工具怎么走代理？命令行代理通常分为三层：临时环境变量、工具自己的配置、Docker daemon 或桌面应用的独立配置。

环境变量适合临时测试和 CI；工具配置适合长期使用；Docker daemon 配置只影响守护进程拉取镜像等操作，不等于 Docker CLI 或容器内的代理配置。不同作用域的设置方式可参考“代理模式”。

下面的例子假设本机客户端提供 HTTP 代理地址 `http://127.0.0.1:7890`。请将地址替换为实际的本机监听地址；如果只有 SOCKS5 端口，可尝试 `socks5h://127.0.0.1:端口`，但不同工具的支持方式并不完全相同。使用网络与地区设置时，先确认代理地址、节点位置和目标服务的支持范围。

| 对象 | 推荐设置方式 | 取消方式 | 注意事项 |
|---|---|---|---|
| Codex CLI | 先设置 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY` | 删除当前会话或持久化环境变量 | 不同 Codex 网络路径可能对代理支持不同；遇到认证或 WebSocket 问题要单独验证 |
| npm / pnpm | npm/pnpm 配置项或环境变量 | `npm config delete`、`pnpm config delete` | pnpm 版本和配置层级会影响读取方式 |
| pip | `--proxy`、pip config 或环境变量 | `pip config unset` 或取消环境变量 | 命令行参数优先级高于环境变量和配置文件 |
| Git | `git config --global http.proxy` | `git config --global --unset http.proxy` | SSH 地址使用的是 SSH 配置，不会自动使用 HTTP 代理 |
| Docker | CLI 的 `config.json`、`daemon.json` 或 Docker Desktop 设置 | 删除对应配置并重启 daemon | `daemon.json` 配置对 Docker Desktop 不生效，应在桌面端设置 |
| curl | `-x` 参数或 `HTTP_PROXY` 等环境变量 | `-x ""` 或取消环境变量 | `--proxy` 会覆盖环境变量；`NO_PROXY` 可排除目标 |

## 二、Windows PowerShell 设置和取消

### 2.1 仅对当前 PowerShell 窗口生效

```powershell
$proxy = "http://127.0.0.1:7890"
$env:HTTP_PROXY = $proxy
$env:HTTPS_PROXY = $proxy
$env:ALL_PROXY = $proxy
$env:NO_PROXY = "localhost,127.0.0.1,::1"
# 查看是否已写入当前进程
Get-ChildItem Env:HTTP_PROXY,Env:HTTPS_PROXY,Env:ALL_PROXY,Env:NO_PROXY
```

关闭当前窗口后，这些变量通常不会继续存在。设置 `NO_PROXY` 可以让本机地址、回环地址和本地开发服务不经过代理。

### 2.2 持久化到当前用户环境

```powershell
$proxy = "http://127.0.0.1:7890"
[Environment]::SetEnvironmentVariable("HTTP_PROXY", $proxy, "User")
[Environment]::SetEnvironmentVariable("HTTPS_PROXY", $proxy, "User")
[Environment]::SetEnvironmentVariable("ALL_PROXY", $proxy, "User")
[Environment]::SetEnvironmentVariable("NO_PROXY", "localhost,127.0.0.1,::1", "User")
```

执行后请重新打开 PowerShell、IDE、Codex CLI 或 Docker Desktop，使新进程读取新的环境变量。不要把包含用户名和密码的代理 URL 提交到代码仓库或公开日志中。

### 2.3 取消代理

```powershell
# 取消当前 PowerShell 会话中的变量
Remove-Item Env:HTTP_PROXY,Env:HTTPS_PROXY,Env:ALL_PROXY,Env:NO_PROXY -ErrorAction SilentlyContinue
# 删除当前用户的持久化变量
[Environment]::SetEnvironmentVariable("HTTP_PROXY", $null, "User")
[Environment]::SetEnvironmentVariable("HTTPS_PROXY", $null, "User")
[Environment]::SetEnvironmentVariable("ALL_PROXY", $null, "User")
[Environment]::SetEnvironmentVariable("NO_PROXY", $null, "User")
```

## 三、macOS/Linux 终端设置和取消

### 3.1 当前终端会话

```bash
proxy='http://127.0.0.1:7890'
export HTTP_PROXY="$proxy"
export HTTPS_PROXY="$proxy"
export ALL_PROXY="$proxy"
export NO_PROXY='localhost,127.0.0.1,::1'
# 查看当前值，注意不要把含密码的代理 URL 发到公共日志
env | grep -iE '^(http|https|all|no)_proxy='
```

### 3.2 取消代理

```bash
unset HTTP_PROXY HTTPS_PROXY ALL_PROXY NO_PROXY
unset http_proxy https_proxy all_proxy no_proxy
```

如果你把 `export` 写入了 `~/.zshrc`、`~/.bashrc` 或其他 shell 启动文件，还需要删除对应行并重新加载 shell 配置，否则新终端会再次设置代理。

## 四、Codex CLI：先用环境变量，再检查网络路径

本地 Codex CLI 通常应先按通用环境变量方式测试。OpenAI 的开发者文档说明，Codex CLI 和 IDE 扩展共享配置层；官方 OpenAI CLI 文档也说明 HTTP 和 SOCKS 代理仍可使用，但特定 mTLS 场景会拒绝 HTTPS 代理。因此，建议先用 HTTP 代理地址验证，再根据具体版本和认证方式排查。

```powershell
# Windows PowerShell
$env:HTTP_PROXY = "http://127.0.0.1:7890"
$env:HTTPS_PROXY = "http://127.0.0.1:7890"
$env:ALL_PROXY = "http://127.0.0.1:7890"
$env:NO_PROXY = "localhost,127.0.0.1,::1"
codex --version
```

```bash
# macOS/Linux
export HTTP_PROXY='http://127.0.0.1:7890'
export HTTPS_PROXY='http://127.0.0.1:7890'
export ALL_PROXY='http://127.0.0.1:7890'
export NO_PROXY='localhost,127.0.0.1,::1'
codex --version
```

如果 `codex --version` 可以运行，但登录、请求或远程连接失败，先不要修改账号配置，可参考“连接故障排查”中的检测思路：执行下面的 `curl` 检测，确认代理确实能够连接 OpenAI 相关域名；再查看 OpenAI 状态页和支持地区页面。

若服务提示当前地区不受支持，请选择官方支持地区的节点，如美国、日本、新加坡，再结合官方支持地区页面继续排查。

如果你使用云小狐，可选择合适的美国或日本节点后，在推荐客户端中确认本机 HTTP 或 SOCKS5 端口，再将端口填入上述命令。客户端安装和节点选择可参考“快速开始”“Windows 配置”“macOS 配置”和“节点选择与延迟”。

## 五、npm 与 pnpm

### 5.1 npm 设置

```bash
# Windows PowerShell、macOS/Linux 均可执行
npm config set proxy http://127.0.0.1:7890
npm config set https-proxy http://127.0.0.1:7890
# 查看
npm config get proxy
npm config get https-proxy
# 取消
npm config delete proxy
npm config delete https-proxy
```

npm 官方配置文档列出了 `proxy` 和 `https-proxy`，并说明环境变量也可能被底层请求库读取。若只想让单次命令使用代理，也可以只保留环境变量，不写入 npm 配置。

### 5.2 pnpm 设置

```bash
# 全局配置
pnpm config set proxy http://127.0.0.1:7890
pnpm config set https-proxy http://127.0.0.1:7890
# 查看
pnpm config get proxy
pnpm config get https-proxy
# 取消
pnpm config delete proxy
pnpm config delete https-proxy
```

pnpm 使用 npm 兼容的配置文件保存注册表、认证和代理等设置；不同版本的全局文件位置可能不同，可运行 `pnpm config get globalconfig` 查看路径。若项目目录中存在 `.npmrc`，还要检查是否有项目级代理配置覆盖了全局值。

## 六、pip

### 6.1 单次命令设置

```powershell
# Windows PowerShell
py -m pip install --proxy http://127.0.0.1:7890 requests
```

```bash
# macOS/Linux
python3 -m pip install --proxy http://127.0.0.1:7890 requests
```

### 6.2 写入用户配置并取消

```powershell
# Windows PowerShell
py -m pip config --user set global.proxy http://127.0.0.1:7890
py -m pip config --user get global.proxy
py -m pip config --user unset global.proxy
```

```bash
# macOS/Linux
python3 -m pip config --user set global.proxy http://127.0.0.1:7890
python3 -m pip config --user get global.proxy
python3 -m pip config --user unset global.proxy
```

pip 官方文档说明，命令行选项优先于环境变量，环境变量优先于配置文件；可使用 `pip config debug` 查看实际读取了哪些配置文件。若出现“明明取消了代理仍然连接失败”，优先检查环境变量和虚拟环境级配置。

## 七、Git

### 7.1 HTTPS 仓库设置

```bash
# 全局设置
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
# 查看实际值
git config --global --get http.proxy
git config --global --get https.proxy
# 取消全局设置
git config --global --unset http.proxy
git config --global --unset https.proxy
```

命令前的空格不是必需的，复制时可以删除。Git 官方文档说明 `http.proxy` 会覆盖通常由 `http_proxy`、`https_proxy` 和 `all_proxy` 提供的代理。若只想对当前仓库设置，把 `--global` 改成 `--local`。

### 7.2 SSH 仓库的区别

如果远程地址是 `git@主机:组织/仓库.git` 或 `ssh://...`，Git 使用 SSH 通道，不会直接套用 HTTP 代理配置。此时应检查 SSH 的网络路径、端口和密钥配置；不要为了排查 HTTPS 代理而关闭 TLS 校验。

## 八、Docker

### 8.1 Docker CLI 和构建容器使用代理

Docker CLI 可在用户目录的 `~/.docker/config.json` 中配置代理，构建过程和容器运行时也可以通过构建参数或环境变量传入。代理配置不应写入镜像层、Dockerfile 或版本库。

```json
{
  "proxies": {
    "default": {
      "httpProxy": "http://127.0.0.1:7890",
      "httpsProxy": "http://127.0.0.1:7890",
      "noProxy": "localhost,127.0.0.1,::1"
    }
  }
}
```

修改后可用 `docker info`、一次简单的 `docker pull` 或测试构建确认是否生效。Docker 官方文档区分了 CLI、daemon、构建和容器运行时的代理配置。

### 8.2 Docker daemon

Linux Docker Engine 可在 `daemon.json` 中配置 `http-proxy`、`https-proxy` 和 `no-proxy`，修改后需要重启 Docker 服务。Docker 官方特别提醒：写入 `daemon.json` 的代理配置对 Docker Desktop 不生效，Docker Desktop 应在应用设置中配置。

```json
{
  "proxies": {
    "http-proxy": "http://127.0.0.1:7890",
    "https-proxy": "http://127.0.0.1:7890",
    "no-proxy": "localhost,127.0.0.1,::1"
  }
}
```

## 九、统一检测：端口、连通性、出口 IP 与延迟

以下是 PowerShell 设置与检测示例，端口以客户端显示为准。数据来自 云小狐 官网公布的实测（品牌方自测）。本文这里提供的是检测命令，实际返回结果以你的网络环境为准。

### 9.1 Windows PowerShell

```powershell
$proxyHost = "127.0.0.1"
$proxyPort = 7890
Write-Host "[1] 检查本机代理端口"
Test-NetConnection $proxyHost -Port $proxyPort | Select-Object ComputerName,RemotePort,TcpTestSucceeded
Write-Host "[2] 检查目标站点 HTTP 状态"
$targets = @(
  "https://api.openai.com/v1/models",
  "https://registry.npmjs.org/",
  "https://pypi.org/",
  "https://github.com/"
)
foreach ($url in $targets) {
  Write-Host "--- $url"
  curl.exe -I -L --max-time 15 $url
}
Write-Host "[3] 查看出口 IP 与地区"
curl.exe -sS --max-time 15 https://ipinfo.io/json
Write-Host "[4] 测量一次 HTTPS 请求耗时"
curl.exe -sS -o NUL -w "code=%{http_code} connect=%{time_connect}s tls=%{time_appconnect}s total=%{time_total}s`n" --max-time 15 https://registry.npmjs.org/
```

### 9.2 macOS/Linux 终端

```bash
proxy_host='127.0.0.1'
proxy_port='7890'
echo '[1] 检查本机代理端口'
if command -v nc >/dev/null 2>&1; then
  nc -vz "$proxy_host" "$proxy_port"
else
  timeout 3 bash -c "cat </dev/null >/dev/tcp/$proxy_host/$proxy_port" && echo 'port open' || echo 'port closed or unavailable'
fi
echo '[2] 检查目标站点 HTTP 状态'
for url in https://api.openai.com/v1/models https://registry.npmjs.org/ https://pypi.org/ https://github.com/; do
  echo "--- $url"
  curl -I -L --max-time 15 "$url"
done
echo '[3] 查看出口 IP 与地区'
curl -sS --max-time 15 https://ipinfo.io/json
echo
echo '[4] 测量一次 HTTPS 请求耗时'
curl -sS -o /dev/null -w 'code=%{http_code} connect=%{time_connect}s tls=%{time_appconnect}s total=%{time_total}s\n' --max-time 15 https://registry.npmjs.org/
```

输出判断方法：

- `TcpTestSucceeded=True` 或 `port open` 表示本机端口有程序监听。
- `HTTP 200`、`301` 或 `302` 通常说明站点已返回响应。
- OpenAI API 在没有认证信息时返回 `401` 也可能说明网络已到达 API，而不是代理一定失败。
- 若出现 `407`，通常是代理认证问题；若出现 `502`、`503` 或连接超时，应继续区分本机代理、目标站点和服务状态，并参考“连接故障排查”。
- 出口 IP 的 `country`、`region`、`city` 字段只表示第三方 IP 数据库的判断，不能替代服务官方支持地区页面。

## 十、按顺序排查常见报错

1. **先确认本机代理端口。** 使用 `Test-NetConnection`、`nc -vz` 或 `curl` 的 `-x` 参数测试；端口未监听时，npm、pip、Git 和 Codex 都会失败。
2. **确认代理协议和地址。** HTTP 代理写成 `http://主机:端口`；SOCKS5 可尝试 `socks5h://主机:端口`。不要把 HTTPS 目标地址误写成 HTTPS 代理协议。
3. **检查变量大小写和优先级。** curl 文档说明小写变量通常具有更高优先级，Windows 不区分大小写；同时存在多个值时，先清理旧变量，再重新打开工具。
4. **检查工具自己的配置。** 运行 `npm config list`、`pnpm config list`、`python -m pip config debug`、`git config --show-origin --get-regexp 'proxy'`，确认没有旧配置覆盖当前设置。
5. **检查 `NO_PROXY`。** 如果把目标域名、API 域名或注册表域名写进了 `NO_PROXY`，请求会直接连接目标；本地开发地址通常应保留在其中。
6. **区分认证失败和网络失败。** `401` 多为目标服务认证问题，`407` 多为代理认证问题；不要把 API 密钥或代理密码直接粘贴到工单、截图和公开仓库。
7. **检查 Docker 的作用域。** 宿主机 curl 能通，不代表 daemon、构建阶段和容器内程序都能通；分别检查 Docker CLI、daemon、build 和容器环境。
8. **查看官方状态与地区说明。** 如果 Codex 或 API 同时出现异常，查看 OpenAI 状态页；如果提示地区不受支持，请选择官方支持地区的节点，如美国、日本、新加坡，并查看官方支持地区页面；不要只根据出口 IP 下结论。

还没有稳定线路？可以看看云小狐套餐页，客户端在“快速开始”里下载。

以上内容仅供学习与研究参考，使用时请遵守所在地法律与各服务条款。

## 常见问题

### Q1：设置了 HTTPS_PROXY，npm 能用，但 Codex CLI 仍然连接失败怎么办？

先用上面的 `curl` 命令测试 OpenAI 相关域名，再确认 Codex 是从设置变量之后新启动的终端或进程运行。若 `curl` 可用而 Codex 仍失败，可能是具体 Codex 版本、认证方式、WebSocket 路径或代理协议兼容性问题，应查看 Codex 官方文档和 OpenAI 状态页，不要直接修改 TLS 校验或提交密钥。

### Q2：为什么 npm 能使用代理，pnpm 却没有效果？

检查 pnpm 版本、全局配置位置、项目根目录的 `.npmrc` 和环境变量。pnpm 官方文档说明其配置文件分为 pnpm 配置和 npm 兼容配置两类，项目级配置可能覆盖全局设置。

### Q3：pip 报证书错误，能否直接使用 --trusted-host？

不建议把关闭证书校验当作常规解决方案。先检查系统时间、根证书、代理是否进行 TLS 检查以及代理证书是否已被系统信任；确认代理地址、认证和目标站点均正常后再处理证书链问题。

### Q4：Git HTTPS 代理配置了，为什么 SSH 仓库仍然失败？

HTTP 代理配置只影响 Git 的 HTTP、HTTPS 等基于 libcurl 的远程请求。SSH 仓库需要单独检查 SSH 主机、端口、密钥、跳板机或组织网络策略。

### Q5：Docker Desktop 修改 daemon.json 后仍然不走代理，正常吗？

正常。Docker 官方文档说明，Docker Desktop 不使用 `daemon.json` 中的代理配置，应在 Docker Desktop 的应用设置中配置；Linux Docker Engine daemon 则可使用 `daemon.json` 或 systemd 环境配置。

相关教程：代理模式、连接故障排查、流量说明、套餐页。

## 参考资料

- OpenAI：Codex 基础配置
- OpenAI：OpenAI CLI API Reference
- OpenAI：API 支持的国家和地区
- OpenAI：服务状态页
- npm：Config
- pnpm：config
- pip：Configuration
- pip：pip config
- Git：git-config
- Docker：Use a proxy server with the Docker CLI
- Docker：Daemon proxy configuration
- curl：命令行手册

## 下一步阅读

1. **Codex、Claude Code、Cursor 连接超时怎么办**  
   终端或编辑器里报 ETIMEDOUT、ECONNRESET、登录不上时的处理办法。讲清 HTTP_PROXY、H...
   →
2. **规则、全局、直连、TUN：四种模式分别什么时候用**  
   规则模式国内直连、国外走代理，日常用它就行；全局和直连各适合什么场景；系统代理和...
   →
3. **连不上或者速度慢，按这个顺序排查**  
   从本地网络、订阅状态、节点、客户端设置一路查到网络环境。连不上和速度慢，都按这个...
   →

## 相关文章

- 用 ChatGPT、Claude 每月大概要多少流量
  →
- AI 工具老掉线、反复要重新登录怎么办
  →
- 常见 AI 工具支持哪些地区（节点选择对照表）
  →

没找到答案？提交工单联系客服 →

## 关于 云小狐

云小狐提供网络连接与节点配置相关服务，本文整理了命令行工具设置代理时常见的配置方式和排查路径。

官网：https://www.yunxiaohu.com/