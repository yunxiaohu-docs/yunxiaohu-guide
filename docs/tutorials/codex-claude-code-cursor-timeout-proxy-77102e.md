# 2026-10-16 更新：Codex、Claude Code、Cursor 连接超时与代理配置排查

> 本文整理自 [云小狐 官网原文](https://www.yunxiaohu.com/help/codex-claude-code-cursor-timeout-proxy)，原文更新于 2026-10-16。

## 结论先看

- **浏览器能打开 AI 服务，不代表终端或编辑器会自动使用相同的网络设置**。优先确认命令行进程是否读取了 `HTTP_PROXY`/`HTTPS_PROXY`，再检查代理端口、节点地区、TUN 模式和服务状态。
- **Claude Code 官方支持通过 `HTTP_PROXY`、`HTTPS_PROXY` 和 `NO_PROXY` 配置标准 HTTP/HTTPS 代理，但不支持 SOCKS 代理**；需要放行的域名以官方《网络配置》页面为准。
- **如果启用了 TUN 模式，先不要同时设置 `HTTP_PROXY`**；如果使用环境变量代理，则先关闭 TUN 或确认规则不会再次接管同一连接，避免代理套代理。

## 为什么现在值得看

本文原文最后更新于 **2026-10-16**，内容针对 Codex、Claude Code、Cursor 的终端、编辑器连接超时和无法登录问题，并覆盖新版本工具常见的代理继承、地区可用性、TUN 模式和服务状态排查。遇到浏览器能用但 CLI 或编辑器仍然超时、晚高峰连接不稳定，或修改代理后仍保留旧配置时，可以按本文顺序定位问题。

## 一、先判断问题属于哪一层

| 现象 | 更可能的原因 | 优先检查 |
|---|---|---|
| 浏览器能用，终端 ETIMEDOUT | 终端没有继承浏览器代理设置 | `HTTP_PROXY`、`HTTPS_PROXY` 是否存在；代理端口是否正确 |
| 终端提示 ECONNRESET | 代理链路、节点或 TLS 连接中途断开 | 切换节点；检查 HTTPS 代理写法；暂时关闭重复代理 |
| Claude Code 无法连接 | 代理未配置、代理不支持当前协议，或目标域名未放行 | Anthropic 官方代理要求与域名清单 |
| Cursor 编辑器无法登录或 AI 功能失败 | Cursor 服务连接、编辑器网络设置或地区可用性问题 | Cursor Settings > Network > Run Diagnostics；检查状态页 |
| 所有应用都无法连接 | 节点、系统网络、DNS 或服务端异常 | 节点延迟、出口地区、官方状态页和本地 DNS |

Claude Code 官方说明支持通过 `HTTP_PROXY`、`HTTPS_PROXY` 和 `NO_PROXY` 配置标准 HTTP/HTTPS 代理，但不支持 SOCKS 代理；需要放行的域名，如 `api.anthropic.com`、`claude.ai`，以官方《网络配置》页面为准。

Cursor 官方建议在 Settings > Network 中运行 Run Diagnostics，用于测试编辑器到 Cursor 服务的连接；Cursor 也说明部分模型会受模型提供商的地区政策影响。

> 终端、浏览器与编辑器的代理路径示意

## 二、按顺序排查连接超时和无法登录

### 确认服务本身没有故障

先查看 OpenAI、Anthropic 和 Cursor 的官方状态页。如果状态页显示对应服务正在故障，先等待服务恢复，不要反复修改本地配置。

### 确认账号与使用地区符合服务要求

OpenAI 会公布 API 支持的国家和地区，并提示从不在列表中的地区访问可能导致账号被限制；Claude Code 的官方入门要求使用 Anthropic 支持的国家或地区。Cursor 也提醒，部分模型是否显示取决于模型提供商的地区政策。

### 确认代理客户端正在运行

可先参考“快速开始”，检查客户端、配置和节点状态。检查 Clash 当前是否已启动、配置是否已加载、节点是否可用，以及本地 HTTP 或 mixed 端口。不同配置的端口可能不同，不要直接照抄示例端口。

### 先用命令行检测目标站点

使用下方命令测试 DNS、TLS、HTTP 响应和出口 IP。命令返回 401 或 403 并不一定代表网络不通，通常说明请求已经抵达服务端，只是缺少有效身份验证或权限。

### 只选择一种代理路径进行测试

可参考“代理模式说明”，避免多种路径同时接管连接。如果已经启用 TUN 模式，先不要同时设置 `HTTP_PROXY`；如果使用 `HTTP_PROXY`/`HTTPS_PROXY`，先关闭 TUN 或确认规则不会再次接管同一连接，避免代理套代理。

### 为当前终端设置环境变量

终端启动的 Codex、Claude Code、npm、git 或其他程序通常只读取当前进程及其子进程可见的环境变量。浏览器中的代理设置不会自动写入终端。

### 重新启动编辑器和终端

修改系统代理、TUN 或环境变量后，已经运行的 Cursor、VS Code 集成终端和 CLI 进程可能仍保留旧配置。完全退出后重新打开，再测试登录。

### 检查凭据和证书

网络可达但仍无法登录时，检查账号状态、API Key、OAuth 浏览器回调以及企业网络的自定义 CA 证书。不要把 API Key、代理账号密码或完整诊断日志公开。

## 三、HTTP_PROXY、HTTPS_PROXY、系统代理和 TUN 模式的区别

### HTTP_PROXY 与 HTTPS_PROXY

这是进程级设置：程序读取环境变量后，将请求交给指定的代理地址。常见写法是 `HTTP_PROXY` 指向 HTTP 代理，`HTTPS_PROXY` 也可以指向一个 HTTP 代理端口，由代理转发 HTTPS 请求。变量名通常同时设置大写和小写，以兼容不同程序。

Claude Code 官方示例使用 `HTTPS_PROXY=https://proxy.example.com:8080` 和 `HTTP_PROXY=http://proxy.example.com:8080`，并明确说明不支持 SOCKS 代理。实际使用时，应按照本地客户端提供的协议填写，不能把 `socks5://` 地址直接当作 Claude Code 的 HTTP 代理。

### Clash 系统代理

系统代理主要影响遵循操作系统代理设置的应用。浏览器通常会读取系统代理，因此“浏览器能用”只能证明浏览器当前路径可用，不能证明所有终端工具、Node.js 进程或编辑器后端都会读取系统代理。

### TUN 模式

TUN 模式在系统网络层创建虚拟接口，把更多未显式设置代理的软件流量交给 Clash 处理。它更适合排查不读取系统代理或环境变量的程序，但也可能带来规则匹配、DNS、权限和虚拟网卡冲突问题。使用 TUN 时，建议先清空终端中的 `HTTP_PROXY`/`HTTPS_PROXY`，再单独验证。

| 设置方式 | 作用范围 | 适合场景 | 常见风险 |
|---|---|---|---|
| HTTP_PROXY/HTTPS_PROXY | 当前 Shell 及子进程 | 临时测试 CLI、脚本和包管理器 | 变量残留、协议或端口写错 |
| 系统代理 | 遵循系统设置的应用 | 浏览器和部分桌面应用 | 终端或后台服务可能不读取 |
| TUN 模式 | 系统网络层的更多应用 | 程序不支持手动代理时 | DNS、规则和虚拟网卡冲突 |

## 四、Windows PowerShell 检测与配置

可参考 Windows 教程。下面的命令使用示例端口 7890，请将其替换为 Clash 当前显示的 HTTP 或 mixed 端口。

```powershell
$proxy = 'http://127.0.0.1:7890'
$env:HTTP_PROXY = $proxy
$env:HTTPS_PROXY = $proxy
$env:http_proxy = $proxy
$env:https_proxy = $proxy
Write-Host '--- OpenAI connectivity ---'
curl.exe -I -sS --connect-timeout 10 --max-time 20 'https://api.openai.com/v1/models' -w "`nHTTP=%{http_code} TIME=%{time_total}s`n"
Write-Host '--- Anthropic connectivity ---'
curl.exe -I -sS --connect-timeout 10 --max-time 20 'https://api.anthropic.com' -w "`nHTTP=%{http_code} TIME=%{time_total}s`n"
Write-Host '--- Egress IP and approximate region ---'
curl.exe -sS --connect-timeout 10 --max-time 20 'https://ipinfo.io/json'
Write-Host '--- Proxy variables ---'
Get-ChildItem Env: | Where-Object { $_.Name -match 'proxy' } | Format-Table -AutoSize
```

### 怎么看输出

HTTP=200、401、403 或 404 都说明通常已经完成了 DNS、TCP 和 TLS 请求；重点观察是否出现 `Could not resolve host`、`Failed to connect`、`Operation timed out` 或连接被重置。TIME 是本次请求总耗时，只适合横向比较当前不同节点，不应当当作固定速度承诺。

`ipinfo.io/json` 中的 `ip`、`city`、`country`、`org` 是第三方服务根据出口地址给出的估计结果，可能存在误差。

测试结束后，如需恢复当前 PowerShell 会话的环境变量，可运行：

```powershell
Remove-Item Env:HTTP_PROXY, Env:HTTPS_PROXY, Env:http_proxy, Env:https_proxy -ErrorAction SilentlyContinue
```

## 五、macOS/Linux 终端检测与配置

```bash
export HTTP_PROXY='http://127.0.0.1:7890'
export HTTPS_PROXY='http://127.0.0.1:7890'
export http_proxy="$HTTP_PROXY"
export https_proxy="$HTTPS_PROXY"
echo '--- OpenAI connectivity ---'
curl -I -sS --connect-timeout 10 --max-time 20 'https://api.openai.com/v1/models' -w '\nHTTP=%{http_code} TIME=%{time_total}s\n'
echo '--- Anthropic connectivity ---'
curl -I -sS --connect-timeout 10 --max-time 20 'https://api.anthropic.com' -w '\nHTTP=%{http_code} TIME=%{time_total}s\n'
echo '--- Egress IP and approximate region ---'
curl -sS --connect-timeout 10 --max-time 20 'https://ipinfo.io/json'
echo '--- Proxy variables ---'
env | grep -i proxy
```

### 输出判断

如果 OpenAI 或 Anthropic 返回 HTTP 状态码，说明至少已经连到对方 HTTPS 服务；如果两个目标都超时，优先检查本地代理端口、当前节点、DNS 和 TUN 状态。如果只有一个目标失败，再查看对应服务状态、地区支持和域名规则。

`ipinfo` 返回的 `country` 是出口 IP 的估计国家代码，不等同于账号注册地或服务商最终判定地区。

若启用了 TUN 模式，可先取消当前 Shell 的代理变量，再做一次对比测试：

```bash
unset HTTP_PROXY HTTPS_PROXY http_proxy https_proxy
curl -I -sS --connect-timeout 10 --max-time 20 'https://api.openai.com/v1/models' -w '\nHTTP=%{http_code} TIME=%{time_total}s\n'
```

## 六、让 Codex、Claude Code 和 Cursor 使用正确路径

### Codex

先确认 Codex CLI 或 IDE 扩展本身已完成登录，并在同一终端中检查代理变量。OpenAI 官方文档说明 Codex CLI 与 IDE 扩展共享部分配置层；不要把 API Key、访问令牌或代理密码写入项目仓库、截图或公开日志。

如果使用的是 OpenAI API CLI，而不是 Codex CLI，应区分 `OPENAI_API_KEY`、`OPENAI_BASE_URL` 等 API 配置与 `HTTP_PROXY`/`HTTPS_PROXY` 网络配置。OpenAI 官方 CLI 文档也提醒，代理能力会因客户端功能和相互 TLS 配置而有所不同。

### Claude Code

Claude Code 官方支持通过 `HTTP_PROXY`、`HTTPS_PROXY`、`NO_PROXY` 环境变量配置代理，但不支持 SOCKS 代理。若设置了代理后出现证书错误，企业网络可能需要正确配置 `SSL_CERT_FILE` 或 `NODE_EXTRA_CA_CERTS`；不要为了跳过证书校验而长期使用不安全选项。

如果需要选择网络出口，可尝试选择云小狐的美国或日本节点，并参考“节点选择与延迟测试教程”；如仍无法连接，可继续查看“连接故障排查”。

### Cursor

在 Cursor 中先打开 Settings > Network，运行 Run Diagnostics。若诊断失败，记录失败的域名或项目，再检查防火墙、系统代理、TUN 规则和地区可用性。

如果只有某个模型不可用，不一定是本地网络故障。Cursor 说明模型提供商可能按地区限制模型可用性，此时可选择账户中仍启用的模型，或在服务商允许的前提下配置自己的 API Key。

## 七、根据错误信息快速定位

| 错误 | 含义倾向 | 处理建议 |
|---|---|---|
| ETIMEDOUT | 在规定时间内未建立连接或未收到响应 | 检查代理端口、节点、DNS、TUN 规则和目标域名 |
| ECONNRESET | 连接被本地代理、中间网络或服务端重置 | 切换节点；避免 HTTP_PROXY 与 TUN 同时接管；检查 TLS 证书 |
| Could not resolve host | DNS 解析失败 | 比较启用代理和关闭代理时的解析结果；检查 DNS 模式和规则 |
| 401 Unauthorized | 已到达服务端，但缺少或拒绝身份凭据 | 检查登录状态、API Key、环境变量和账号权限 |
| 403 Forbidden 或 unsupported region | 请求可能被地区、账号或服务政策拒绝 | 对照官方支持地区，切换到列表内地区，如美国、日本的节点后重试 |
| 浏览器 OAuth 成功，CLI 仍未登录 | 浏览器回调成功，但 CLI 进程路径不同 | 重启终端，确认 CLI 使用同一网络出口，并检查本地回调是否被安全软件拦截 |

还没有稳定线路？可以查看云小狐套餐页，客户端可在“快速开始”中下载。

以上内容仅供学习与研究参考，使用时请遵守所在地法律与各服务条款。

## 常见问题

### 浏览器能用，为什么 Claude Code 或 Codex 仍然超时？

浏览器可能使用了系统代理或自身代理设置，而终端程序通常只读取当前进程的环境变量，或者完全不读取系统代理。先在终端中设置 `HTTP_PROXY` 和 `HTTPS_PROXY`，再用 curl 测试目标域名。

### HTTP_PROXY 和 HTTPS_PROXY 应该填什么？

应填写本地代理客户端提供的 HTTP 或 mixed 代理地址，例如 `http://127.0.0.1:7890`。端口只是示例，必须以 Clash 当前配置为准。Claude Code 不支持直接使用 SOCKS 代理。

### 已经启用 TUN 模式，还需要设置环境变量吗？

不一定。建议先采用一种方式做基线测试：TUN 模式下清空环境变量；环境变量模式下暂时关闭 TUN。两种方式同时启用可能造成规则重复匹配、代理套代理或难以判断故障来源。

### curl 返回 401，是不是网络设置失败？

通常不是。401 表示服务器收到了请求，但需要有效身份验证。只要没有出现 DNS 失败、连接超时或 TLS 错误，网络链路大概率已经建立；下一步应检查登录状态或 API Key。

### 为什么 Cursor 只有部分模型不能用？如何避免凭据泄露？

Cursor 官方说明，模型提供商可能按地区限制模型可用性。可以运行 Cursor 的网络诊断，并查看当前账户可选模型；不要把单个模型不可用直接判断为所有网络设置失效。

同时，不要把带账号密码的代理 URL 写入仓库、Shell 配置文件的公开片段、截图或工单附件。分享日志前应删除 Authorization、API Key、Cookie、访问令牌和完整代理地址。

## 参考资料

- OpenAI API 支持的国家和地区：官方说明
- OpenAI Codex 基础配置：官方文档
- OpenAI CLI 参考：官方文档
- Anthropic Claude Code 企业代理配置：官方文档
- Anthropic Claude Code 网络配置与放行域名：官方文档
- Anthropic Claude Code 入门与支持地区要求：官方文档
- Anthropic 服务状态页：官方状态页
- Cursor 网络问题排查：官方文档
- Cursor 地区与模型可用性：官方文档
- Cursor 服务状态页：官方状态页
- 快速开始、Windows 教程、macOS 教程、代理模式说明；选购流量可查看套餐页。

## 下一步阅读

- 命令行怎么走代理：Codex、npm、pip、git、Docker

  Windows PowerShell 和 macOS/Linux 终端下，Codex CLI、npm/pnpm、pip、Git、Docker 等工具的代理配置。

- 规则、全局、直连、TUN：四种模式分别什么时候用

  规则模式国内直连、国外走代理，日常用它即可；全局和直连各适合什么场景；系统代理和 TUN 模式如何选择。

- 连不上或者速度慢，按这个顺序排查

  从本地网络、订阅状态、节点、客户端设置一路查到网络环境。连不上和速度慢，都按这个顺序排查。

## 相关文章

- 用 ChatGPT、Claude 每月大概要多少流量
- AI 工具老掉线、反复要重新登录怎么办
- 常见 AI 工具支持哪些地区（节点选择对照表）

没找到答案？提交工单联系客服。

## 关于 云小狐

云小狐提供网络代理客户端、节点和相关使用教程，本文内容用于帮助排查 AI 工具的终端与编辑器连接问题。

官网：[https://www.yunxiaohu.com/](https://www.yunxiaohu.com/)