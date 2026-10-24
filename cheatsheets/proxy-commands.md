# 命令行走代理速查卡：git / npm / pip / curl / Homebrew（含 NO_PROXY）

最后核对：2026-10-24。下面的 `127.0.0.1:7890` 只是示例，换成你客户端里显示的本地 HTTP 端口；SOCKS 端口要写成 `socks5h://127.0.0.1:端口`。

每张卡只有一个代码块：上面是设置，下面是取消和查看。

## 卡 1 · 当前终端临时生效（macOS / Linux）

```bash
export http_proxy=http://127.0.0.1:7890 https_proxy=http://127.0.0.1:7890 all_proxy=socks5h://127.0.0.1:7890
export no_proxy=localhost,127.0.0.1,::1
# 取消
unset http_proxy https_proxy all_proxy no_proxy
# 查看
env | grep -i _proxy
```

关掉终端就失效。curl、Homebrew、pip 和大多数命令行工具都会读这几个变量。

## 卡 2 · 当前窗口临时生效（Windows PowerShell）

```powershell
$env:HTTP_PROXY="http://127.0.0.1:7890"; $env:HTTPS_PROXY="http://127.0.0.1:7890"; $env:NO_PROXY="localhost,127.0.0.1"
# 取消
Remove-Item Env:HTTP_PROXY, Env:HTTPS_PROXY, Env:NO_PROXY -ErrorAction SilentlyContinue
# 查看
Get-ChildItem Env: | Where-Object Name -like '*PROXY*'
```

## 卡 3 · git

```bash
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
# 取消
git config --global --unset http.proxy
git config --global --unset https.proxy
# 查看
git config --global --get-regexp 'https?\.proxy'
```

只对 https:// 地址的仓库生效；`git@github.com:` 这种 SSH 地址走的是 SSH，不读这两项。

## 卡 4 · npm

```bash
npm config set proxy http://127.0.0.1:7890
npm config set https-proxy http://127.0.0.1:7890
# 取消
npm config delete proxy
npm config delete https-proxy
# 查看
npm config get proxy; npm config get https-proxy
```

写进的是用户目录下的 `.npmrc`，取消后新开终端再试一次。

## 卡 5 · pip

```bash
# 只对这一次安装生效
pip install requests --proxy http://127.0.0.1:7890
# 长期生效
pip config set global.proxy http://127.0.0.1:7890
# 取消
pip config unset global.proxy
# 查看
pip config list
```

## 卡 6 · curl

```bash
# 只对这一条命令走代理
curl -x http://127.0.0.1:7890 -I https://pypi.org
# 已经设了环境变量，这一条不想走代理
curl --noproxy '*' -I https://pypi.org
```

## 卡 7 · Homebrew

```bash
# Homebrew 用 curl 下载，读卡 1 的环境变量；只想对这一次 brew 生效：
https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 brew update
# 取消：不设这两个变量即可（或执行卡 1 的 unset）
```

## 卡 8 · NO_PROXY：哪些地址不要走代理

```bash
export no_proxy=localhost,127.0.0.1,::1,.local,.internal.example
```

- 用逗号分隔，不要加空格；以点开头的 `.local` 表示整个后缀。
- 网段写法（如 `192.168.0.0/16`）不是所有工具都认，内网地址最好逐个列出或用后缀。
- 有的工具只读小写 `no_proxy`，有的读大写，两个都设上，省得逐个查。

## 设了还是不通，先查这三处

1. 端口对不对：本地代理客户端里显示的 HTTP 端口和这里填的是否一致。
2. 有没有旧配置残留：分别执行每张卡里的“查看”命令，确认没有多余的代理设置。
3. 是不是 SSH 地址：git 用 SSH 克隆时，要在 `~/.ssh/config` 里单独配置，上面的设置不管用。
