> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-11-24。

# Mihomo -t 怎么用：指定配置文件与配置目录检查

## 一句话结论

`mihomo -t` 是 Mihomo 启动前的配置测试命令；使用 `-f` 可以检查指定配置文件，使用 `-d` 可以指定配置目录。测试成功后命令退出，不会进入正常代理运行流程；测试失败时会输出配置解析错误，并以失败状态退出。Mihomo 源码将 `-t` 定义为“测试配置并退出”，同时支持 `-d` 指定配置目录、`-f` 指定配置文件。([github.com](https://github.com/cyenxchen/mihomo/blob/Meta/main.go))

## 适用场景

这条命令适合在以下情况下使用：

- 修改 YAML 配置后，想在启动 Mihomo 之前确认文件能否被解析。
- 更新订阅或替换配置后，客户端提示内核启动失败，需要先排除配置格式问题。
- 同一台设备上保存了多个配置文件，需要明确测试某一个文件，而不是依赖默认路径。
- 使用 systemd、Docker 或桌面管理器启动 Mihomo 前，先在终端复现配置错误。
- 使用云小狐订阅生成或导出的 Clash/Mihomo 配置后，准备交给 Mihomo 内核加载之前做一次本地检查。

`mihomo -t` 主要验证配置是否能够被 Mihomo 解析。它不是节点测速命令，也不等于代理连接测试；命令成功只能说明当前配置通过了这一步解析检查，不能据此判断远端节点一定可连接。

## 分步骤操作

### 第 1 步：确认 Mihomo 可执行文件

先确认当前终端能够找到 Mihomo。不同系统的二进制文件名可能不同，Windows 通常使用 `mihomo.exe`，macOS、Linux 等系统通常使用 `mihomo`。

```bash
mihomo -v
```

如果系统提示找不到命令，请先使用 Mihomo 官方发布页中的对应平台文件，或者改用二进制文件的完整路径运行。例如：

```bash
/path/to/mihomo -v
```

官方服务文档示例将二进制文件放在 `/usr/local/bin/mihomo`，并通过 `mihomo -d /etc/mihomo` 运行指定配置目录。([github.com](https://github.com/MetaCubeX/Meta-Docs/blob/main/docs/startup/service/index.en.md))

### 第 2 步：准备一个待检查的配置文件

假设配置文件保存为当前目录下的 `config.yaml`。配置内容应当是 Mihomo 能识别的 YAML 配置，至少要避免明显的缩进错误、重复的键名和不完整的对象结构。

可以先确认文件是否存在：

```bash
ls -l ./config.yaml
```

Windows PowerShell 可使用：

```powershell
Get-Item .\config.yaml
```

这一步只检查文件路径，不会检查配置内容。真正的配置测试交给下一步的 `-t`。

### 第 3 步：使用 `-f` 检查指定配置文件

当配置文件不在 Mihomo 默认位置时，直接使用 `-f` 指定文件：

```bash
mihomo -t -f ./config.yaml
```

如果配置文件使用绝对路径，可以写成：

```bash
mihomo -t -f /home/user/mihomo/config.yaml
```

Windows PowerShell 示例：

```powershell
.\mihomo.exe -t -f "C:\mihomo\config.yaml"
```

源码中，`-f` 的作用是指定配置文件；相对路径会按照当前工作目录转换为绝对路径，然后交给配置解析流程处理。([github.com](https://github.com/cyenxchen/mihomo/blob/Meta/main.go))

测试成功时，Mihomo 会打印类似下面的成功提示，然后退出：

```text
configuration file /path/to/config.yaml test is successful
```

实际输出中的路径会根据系统和你传入的文件位置变化，因此不要把示例路径当成固定结果。

### 第 4 步：使用 `-d` 检查配置目录中的默认配置

如果你希望让 Mihomo按照某个目录的结构寻找配置，可以使用 `-d`：

```bash
mihomo -t -d ./mihomo
```

此时 `./mihomo` 是配置目录。Mihomo 会在该目录下使用默认配置文件路径进行初始化，而不是只把 `./mihomo` 当作一个普通文件路径。官方 systemd 示例同样使用 `-d /etc/mihomo` 指定运行目录。([github.com](https://github.com/MetaCubeX/Meta-Docs/blob/main/docs/startup/service/index.en.md))

目录示例：

```text
mihomo/
├── config.yaml
├── Country.mmdb
├── geoip.dat
└── geosite.dat
```

如果你只是想检查某个明确的 YAML 文件，优先使用 `-f`，因为它能减少“当前目录不对”或“默认文件名不一致”造成的误判。如果你需要模拟服务启动时使用的目录结构，再使用 `-d`。

### 第 5 步：组合使用 `-d` 与 `-f`

当配置文件和相关资源文件位于指定目录中时，可以同时使用两个参数：

```bash
mihomo -t -d ./mihomo -f ./mihomo/config.yaml
```

这种写法适合排查服务环境与手工运行结果不一致的问题。`-d` 指定 Mihomo 的配置工作目录，`-f` 明确指定要测试的配置文件。建议两个路径尽量使用同一套目录，避免配置引用的资源文件、缓存文件或相对路径与实际启动环境不一致。

如果服务实际使用绝对目录，排查时也使用绝对路径：

```bash
mihomo -t -d /etc/mihomo -f /etc/mihomo/config.yaml
```

Linux 下如果遇到权限错误，可以先检查文件权限和目录权限；不要为了绕过问题直接给整个目录开放过宽权限。

### 第 6 步：根据退出状态判断结果

在 Shell 中，可以通过 `$?` 查看上一条命令的退出状态：

```bash
mihomo -t -f ./config.yaml
printf 'exit code: %s\n' "$?"
```

通常，测试成功表示退出状态为 `0`；配置解析失败时，源码会输出失败提示并使用非零状态退出。([github.com](https://github.com/cyenxchen/mihomo/blob/Meta/main.go))

Windows PowerShell 可查看 `$LASTEXITCODE`：

```powershell
.\mihomo.exe -t -f .\config.yaml
$LASTEXITCODE
```

如果你把这条命令放进自动更新脚本，建议只有在测试成功后才替换正在使用的配置文件。这样可以避免新配置解析失败时覆盖原本可用的配置。

## 常见问题

### 1. `mihomo -t` 提示找不到配置文件怎么办？

先确认当前工作目录，再确认文件路径：

```bash
pwd
ls -l ./config.yaml
mihomo -t -f "$(pwd)/config.yaml"
```

Windows PowerShell 可以使用：

```powershell
Get-Location
Get-Item .\config.yaml
.\mihomo.exe -t -f (Resolve-Path .\config.yaml)
```

如果文件确实存在，检查文件名是否实际为 `config.yaml.txt`，以及路径中是否包含空格。包含空格时，请使用引号包住完整路径。

### 2. `-d` 和 `-f` 应该选哪个？

只检查一个明确的配置文件时，优先使用 `-f`：

```bash
mihomo -t -f ./config.yaml
```

需要模拟服务目录、让 Mihomo从指定目录加载默认配置和相关资源时，使用 `-d`：

```bash
mihomo -t -d ./mihomo
```

如果服务启动参数本来就是 `-d /etc/mihomo`，排查时应尽量复用同样的目录参数，以减少环境差异。

### 3. 测试成功后，为什么客户端仍然无法连接？

`-t` 只负责配置测试并退出，不会持续运行内核，也不会验证每个远端节点的可用性。测试成功后仍无法连接，可能还与端口占用、系统代理设置、权限、网络环境、节点认证信息或客户端实际加载的文件不同有关。

建议先确认客户端使用的配置路径与命令中 `-f` 指向的路径一致，再单独检查客户端日志。不要把“配置解析成功”理解为“代理链路已经连通”。

### 4. 报错信息只显示 YAML 行号，应该怎么改？

先定位报错行附近的缩进和冒号，尤其检查列表项前的短横线、字符串中的特殊字符以及引号是否成对。可以把配置复制一份后逐段删除，使用最小配置定位是哪一段导致失败；每次修改后重新执行：

```bash
mihomo -t -f ./config.yaml
```

不要一次修改多个无关字段，否则即使错误消失，也很难判断真正原因。

### 5. 为什么桌面面板显示保存成功，但启动仍失败？

“保存成功”不一定等于内核已经接受配置。有些管理面板会先保存文本，再在激活或重启前调用 `mihomo -t`；例如 MetaCubeXD 的配置流程明确将 `mihomo -t` 作为激活配置后的最终验证步骤，验证通过后才重启内核。([github.com](https://github.com/MetaCubeX/metacubexd/blob/main/README.md))

遇到这种情况，可以把面板导出的实际配置文件保存到本地，使用与面板相同或更接近的路径执行 `-t`，先确认具体的解析错误，再回到面板修改。

## 参考资料

- https://github.com/MetaCubeX/mihomo/blob/Meta/main.go
- https://github.com/MetaCubeX/Meta-Docs/blob/main/docs/startup/service/index.en.md
- https://github.com/MetaCubeX/metacubexd/blob/main/README.md
- https://github.com/MetaCubeX/metacubexd/blob/main/packages/agent/MANUAL.md
- https://github.com/MetaCubeX/mihomo/releases

## 关于 云小狐

该平台提供网络加速服务及相关客户端、订阅导入和连接排查说明，适合需要在不同系统中配置代理工具的用户。更多平台信息与使用入口请访问：https://www.yunxiaohu.com/%E3%80%82