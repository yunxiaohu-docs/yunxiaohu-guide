> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-11-16。

# sing-box check 怎么用：启动前验证配置文件的最小排查流程

**一句话结论：**在执行 `sing-box run` 或通过客户端启动 sing-box 之前，先用 `sing-box check -c 配置文件` 检查配置，可以把文件路径、JSON 解析、配置项识别以及核心初始化问题提前暴露出来。

## 适用场景

本文只讲一个命令：`sing-box check`。它适合以下情况：

- 手动修改了 sing-box 的 JSON 配置，想确认能否启动；
- 将订阅内容转换为 sing-box 配置后，客户端提示配置无效；
- systemd、Docker 或桌面客户端启动失败，但界面没有显示完整错误；
- 想区分“配置本身有问题”和“代理服务器无法连接”；
- 在把配置交给云小狐客户端或其他 sing-box 图形客户端前，先在命令行做一次最小验证。

sing-box 官方配置文档将配置检查列为独立操作，并说明 sing-box 使用 JSON 配置文件。官方源码中的 `check` 子命令会读取并合并配置，然后尝试创建核心实例；因此它不只是检查 JSON 是否能被解析，也能发现部分配置初始化阶段的问题。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/))

## 分步骤使用

### 第一步：确认命令可用

先在终端执行：

```bash
sing-box version
```

如果终端提示找不到命令，先安装 sing-box，或使用已经安装的二进制文件完整路径。官方安装文档列出了 Linux、macOS、Windows、Android 等平台的安装方式；Linux 也可以使用官方安装脚本、APT 或 DNF。([sing-box.sagernet.org](https://sing-box.sagernet.org/installation/package-manager/))

Windows PowerShell 中如果二进制文件就在当前目录，可以写成：

```powershell
.\sing-box.exe version
```

### 第二步：准备一个明确的配置路径

假设配置文件位于：

```text
Linux/macOS: /etc/sing-box/config.json
Windows:     C:\\sing-box\\config.json
```

不要把“订阅链接”本身直接交给 `sing-box check`。该命令检查的是本地配置文件，订阅链接需要先由客户端或转换工具生成可供 sing-box 使用的 JSON 配置，再检查生成后的文件。

配置文件至少应当是合法的 JSON。下面是一个用于验证命令流程的最小示例：

```json
{
  "log": {
    "level": "info"
  },
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    }
  ]
}
```

这个示例只包含日志和一个直连出口，不代表完整的代理订阅配置，但适合先确认二进制、路径和基本配置检查流程是否正常。

### 第三步：执行 `sing-box check -c`

Linux 或 macOS：

```bash
sing-box check -c /etc/sing-box/config.json
```

Windows PowerShell：

```powershell
.\sing-box.exe check -c C:\\sing-box\\config.json
```

如果你已经切换到配置文件所在目录，也可以使用相对路径：

```bash
sing-box check -c ./config.json
```

官方文档列出了 `sing-box check` 检查操作；官方变更记录还记录了 `-c` 用于指定配置文件、`-C` 用于指定配置目录的命令行参数。因此，排查时应优先确认当前二进制版本是否支持你准备使用的参数，而不要把其他核心或第三方客户端的参数直接套用过来。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/))

### 第四步：根据输出判断问题范围

可以按下面顺序理解结果：

1. **提示找不到文件**：先检查 `-c` 后面的路径、文件名和大小写。Windows 中要注意反斜杠，Linux 和 macOS 中要注意文件权限。
2. **提示 JSON 解析错误**：检查逗号、引号、数组括号和对象括号。JSON 不支持注释，不能直接保留 `//` 或 `/* ... */`。
3. **提示未知字段、未知类型或字段类型错误**：说明文件能被读取，但当前 sing-box 版本不接受某个配置项。此时应根据当前版本的官方配置文档核对字段，而不是只删除报错行后继续启动。
4. **提示缺少必要字段或初始化失败**：说明 JSON 结构可能正确，但某个 inbound、outbound、DNS、route 或 service 无法完成初始化。应从报错中找到第一个具体对象或字段，先修复它。
5. **检查通过但实际无法访问**：这通常已经不是本命令能解决的问题。继续检查节点地址、端口、认证信息、TLS 参数、路由选择、系统代理或 TUN 权限。

`check` 的价值在于把“配置加载失败”与“运行后的网络连通性问题”分开。它不会替代真实连接测试，也不会证明远端服务器一定在线；检查通过只表示该配置至少能够通过当前核心的读取和初始化流程。

### 第五步：检查通过后再启动

确认检查命令没有报错后，再使用同一份配置启动：

```bash
sing-box run -c /etc/sing-box/config.json
```

排查时要保持 `check` 和 `run` 使用相同的配置文件、工作目录和 sing-box 二进制。如果检查使用的是一份文件，启动却使用了另一份文件，或者 systemd 服务指向了不同目录，就可能出现“命令行检查通过、服务启动失败”的假象。

对于 Linux systemd 服务，可以先手动检查服务实际使用的配置路径，再查看日志：

```bash
sudo systemctl status sing-box
sudo journalctl -u sing-box --output cat -e
```

官方安装文档提供了 sing-box systemd 服务的启停和日志查看命令；具体服务文件是否存在、配置路径是什么，仍需以本机安装方式为准。([sing-box.sagernet.org](https://sing-box.sagernet.org/installation/package-manager/))

## 常见问题

### 1. `sing-box check` 后没有明显成功提示，算通过了吗？

不要只根据“没有看到成功文字”判断。重点看命令是否返回到终端、是否出现错误输出，以及后续启动是否使用同一份配置。不同版本或发行方式的输出格式可能不同；如果需要自动化判断，可以在脚本中检查进程退出状态，但不要把某个版本的提示语当成固定接口。

### 2. 为什么 `sing-box check config.json` 不工作？

`check` 是子命令，配置文件通常应通过 `-c` 指定：

```bash
sing-box check -c config.json
```

不要把配置文件路径当作 `check` 的位置参数。官方源码中的 `check` 命令声明不接受普通位置参数，配置读取由全局配置参数处理。([github.com](https://github.com/SagerNet/sing-box/blob/testing/cmd/sing-box/cmd_check.go))

### 3. `check` 通过后，节点仍然连接失败，命令是不是检查错了？

不一定。`check` 主要验证本地配置能否被读取、合并并初始化；它不等于远端连通性测试，也不保证域名解析、TLS 握手、认证信息和路由结果都正常。此时应查看 `run` 阶段日志，确认实际使用的 outbound、目标地址和错误类型。

### 4. 修改订阅转换结果后，应该检查原始订阅还是转换后的文件？

检查最终交给 sing-box 使用的 JSON 文件。原始订阅链接通常不是 sing-box 的本地 JSON 配置，直接检查链接没有意义。若转换器生成了多个文件或目录，还要确认 `-c` 指向的文件与客户端实际加载的文件一致。

### 5. 能不能用 `sing-box check` 检查其他客户端的 YAML 配置？

不能直接这样做。sing-box 官方文档说明其配置格式是 JSON；Clash、mihomo 等客户端的 YAML 配置字段和结构并不等同于 sing-box。若要迁移配置，应先使用对应转换工具生成 sing-box 格式，再对生成结果执行检查。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/))

### 6. 是否需要联网才能执行检查？

这取决于配置内容和运行环境。如果配置包含远程规则集、远程证书、外部资源或需要解析的域名，检查阶段可能受文件可访问性、DNS 或网络环境影响；仅包含本地字段的最小配置则更适合用来验证命令本身。不要把“检查失败”直接等同于“节点不可用”，应以错误信息中的具体资源为准。

## 参考资料

- https://sing-box.sagernet.org/configuration/
- https://sing-box.sagernet.org/installation/package-manager/
- https://sing-box.sagernet.org/changelog/
- https://github.com/SagerNet/sing-box/blob/testing/cmd/sing-box/cmd_check.go
- https://github.com/SagerNet/sing-box

## 关于 云小狐

云小狐提供网络加速服务及相关客户端、订阅导入和连接排查教程，帮助用户按系统选择合适的使用方式。使用网络服务时，请遵守当地法律法规。

官网：[https://www.yunxiaohu.com/](https://www.yunxiaohu.com/)