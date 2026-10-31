> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-10-31。

# Mihomo 配置检查报错怎么排查：-t、proxy-groups 与 proxy-providers

**一句话结论：**遇到 Mihomo 启动失败或客户端提示配置无效时，先使用 `mihomo -t` 检查实际配置文件，再根据错误位置排查 YAML 语法、`proxy-groups` 引用名称和 `proxy-providers` 文件路径；配置检查通过后，再处理端口、规则或远程连接问题。

## 适用场景

本文适合以下情况：

- 修改 Mihomo 配置后，程序启动即退出；
- Clash Meta 兼容客户端提示 YAML 格式错误或配置无效；
- `proxy-groups` 中已经写了节点名称，但代理组列表为空；
- 使用 `proxy-providers` 后，出现提供者文件找不到、代理组无法读取节点等问题；
- 导入订阅、合并本地配置或手动调整规则后，想先确认配置文件本身能否被内核解析。

Mihomo 是 MetaCubeX 维护的开源代理内核。配置检查命令的具体参数和行为应以当前使用版本的帮助信息及官方文档为准；不同客户端对配置文件的保存位置、启动参数和内核文件名可能不同，因此第一步应先确认实际运行的二进制和配置文件。

## 一、先确认检查的是哪一个配置文件

### 1. 使用 `-f` 指定配置文件

如果配置文件不在当前目录，建议明确指定路径：

```bash
mihomo -t -f ./config.yaml
```

Linux、macOS 示例：

```bash
mihomo -t -f ~/.config/mihomo/config.yaml
```

Windows PowerShell 示例：

```powershell
.\\mihomo.exe -t -f .\\config.yaml
```

这里的 `-f` 用于指定要检查的 YAML 文件。不要只在编辑器中确认缩进无误后就直接重启，因为客户端实际加载的文件可能位于其他配置目录，或者仍然是上一版本的配置。

### 2. 使用 `-d` 指定配置目录

如果配置文件、代理提供者文件和其他资源都使用相对路径，可以指定配置目录：

```bash
mihomo -t -d ~/.config/mihomo
```

如果部署目录是 `/etc/mihomo`，可以写成：

```bash
mihomo -t -d /etc/mihomo
```

使用 `-d` 时，应同时确认该目录中的默认配置文件、子目录和相对路径是否存在。实际使用云小狐提供的订阅服务时，建议先将订阅导入客户端并保存为客户端实际使用的配置，再检查对应的文件，避免检查示例配置却启动了另一份文件。

### 3. 先查看当前内核支持的参数

如果不确定当前二进制是否支持某个参数，可以先查看帮助信息：

```bash
mihomo -h
mihomo -v
```

Windows PowerShell：

```powershell
.\\mihomo.exe -h
.\\mihomo.exe -v
```

如果客户端自带内核，不要默认认为系统环境变量中的 `mihomo` 就是客户端正在使用的文件。应从客户端设置、安装目录或启动脚本中确认实际路径。

## 二、先处理 YAML 层面的错误

### 1. 检查缩进和列表符号

Mihomo 配置使用 YAML。常见错误包括：

- 同一层级混用了不同数量的空格；
- `- name` 没有和同级列表对齐；
- 冒号后缺少空格；
- 把字符串、数字或布尔值写成了错误类型；
- 复制配置时混入全角冒号、全角引号或制表符；
- 配置块的层级关系与预期不一致。

下面是一个用于结构检查的最小示例，包含一个手动代理、一个代理组和一条兜底规则：

```yaml
mixed-port: 7890
allow-lan: false
mode: rule
log-level: info

proxies:
  - name: "示例节点"
    type: socks5
    server: 127.0.0.1
    port: 1080

proxy-groups:
  - name: "代理选择"
    type: select
    proxies:
      - "示例节点"
      - DIRECT

rules:
  - MATCH,代理选择
```

这只是配置结构示例，不代表可以连接远程服务。`server`、`port`、协议类型以及认证参数必须与实际代理信息一致。

### 2. 一次只修改一个区块

如果原配置可以正常启动，建议先备份，再一次只修改一个区块。比如先调整 `proxy-groups` 并检查，通过后再处理 `proxy-providers` 或规则。这样能够根据错误输出快速判断是哪次修改引入了问题。

```bash
cp config.yaml config.yaml.bak
mihomo -t -f config.yaml
```

Windows PowerShell：

```powershell
Copy-Item .\\config.yaml .\\config.yaml.bak
.\\mihomo.exe -t -f .\\config.yaml
```

不要在没有备份的情况下直接覆盖唯一配置，尤其是订阅转换、批量替换或编辑器自动格式化之后。

## 三、排查 `proxy-groups` 引用不到节点

官方配置文档中，代理组可以通过 `proxies` 引用代理节点或其他代理组，也可以通过 `use` 引用代理提供者。代理组名称、节点名称和提供者名称都属于配置中的引用关系，名称变化后，相关引用也需要同步修改。

### 1. 检查名称是否完全一致

下面的配置中，代理定义叫 `示例节点`，代理组却引用了末尾多一个空格的 `示例节点 `，两者不是同一个名称：

```yaml
proxies:
  - name: "示例节点"
    type: socks5
    server: 127.0.0.1
    port: 1080

proxy-groups:
  - name: "代理选择"
    type: select
    proxies:
      - "示例节点 "
```

应改为：

```yaml
proxy-groups:
  - name: "代理选择"
    type: select
    proxies:
      - "示例节点"
      - DIRECT
```

排查时重点看：

- 中文全角空格和英文半角空格；
- 大小写差异；
- 名称中是否含有不可见字符；
- 手动代理名称是否改过，但代理组仍保留旧名称；
- 节点是否只是注释，并没有真正出现在 `proxies` 列表中。

### 2. 注释内容不会创建节点

以下内容全部是注释，不会创建可被代理组引用的节点：

```yaml
proxies:
  # - name: "示例节点"
  #   type: socks5
```

如果代理组仍然引用 `示例节点`，就需要删除该引用，或补充完整的代理定义。编辑器中显示了这些文本，并不代表 Mihomo 已经加载了它们。

### 3. 区分节点名称和代理组名称

`proxy-groups.proxies` 可以引用代理节点，也可以引用其他代理组。若名称相同或过于接近，容易在修改时误删或误引用。建议为代理组使用明显不同的名称，例如 `代理选择`、`自动选择`，并避免让手动节点与代理组共用完全相同的名称。

## 四、排查 `proxy-providers` 文件和引用关系

`proxy-providers` 用于从本地文件、远程 URL 或内联内容加载代理集合。官方文档列出的提供者类型包括 `http`、`file` 和 `inline`。其中，`file` 类型依赖本地路径，`http` 类型需要 URL；代理组通常通过 `use` 引用提供者键名。

### 1. 本地文件示例

```yaml
proxy-providers:
  my-provider:
    type: file
    path: ./proxy_providers/my-provider.yaml

proxy-groups:
  - name: "代理选择"
    type: select
    use:
      - my-provider
    proxies:
      - DIRECT
```

对应目录可以是：

```text
config.yaml
proxy_providers/
└── my-provider.yaml
```

如果配置目录是 `/etc/mihomo`，而 `path` 写成 `./proxy_providers/my-provider.yaml`，就应检查 `/etc/mihomo/proxy_providers/my-provider.yaml` 是否存在，而不是只检查当前终端所在目录。

官方文档还说明，提供者路径受到配置目录和安全路径规则影响。若文件位于配置目录之外，不要仅凭相对路径猜测是否可读，应以当前版本文档和启动日志为准。

### 2. 检查提供者名称是否一致

`proxy-providers` 中的键名是 `my-provider`，代理组的 `use` 中也应使用相同名称：

```yaml
proxy-providers:
  my-provider:
    type: file
    path: ./proxy_providers/my-provider.yaml

proxy-groups:
  - name: "代理选择"
    type: select
    use:
      - my-provider
```

下面的 `provider1` 并不存在，不能指向 `my-provider`：

```yaml
proxy-groups:
  - name: "代理选择"
    type: select
    use:
      - provider1
```

建议先检查文件，再检查配置：

```bash
ls -l ./proxy_providers/my-provider.yaml
mihomo -t -f ./config.yaml
```

Windows PowerShell：

```powershell
Get-Item .\\proxy_providers\\my-provider.yaml
.\\mihomo.exe -t -f .\\config.yaml
```

### 3. 区分“配置检查通过”和“代理可用”

`mihomo -t` 通过，并不等于每个远程节点都能连接，也不等于规则访问结果一定符合预期。配置检查主要用于发现解析、初始化和引用关系方面的问题；远程 URL 是否可访问、代理节点是否响应、端口是否监听以及目标网站是否能打开，属于运行阶段的问题。

因此，配置检查通过后仍然无法连接时，不要继续反复修改 YAML，应转向查看客户端日志、代理模式、监听端口和实际请求路径。

## 五、推荐的完整排查顺序

1. **确认内核和配置文件对应。** 使用 `mihomo -v` 查看当前二进制版本，并确认 `-f` 或 `-d` 指向的配置位置。
2. **备份原文件。** 保留可以回退的版本，不要覆盖唯一配置。
3. **执行配置检查。** 使用当前内核支持的检查参数，并保存完整输出。
4. **先修 YAML。** 处理缩进、冒号、列表和数据类型问题。
5. **再查名称引用。** 对照 `proxies`、`proxy-groups.proxies`、`proxy-providers` 和 `proxy-groups.use` 中的名称。
6. **最后查文件路径。** 对所有相对路径，以 Mihomo 配置目录为基准确认文件是否存在。
7. **检查通过后再重启。** 如果仍然无法连接，再单独检查端口、客户端模式、系统代理和远程服务响应。

## 常见问题

### Q1：执行检查命令后提示找不到配置文件，怎么办？

先确认当前目录和文件是否存在：

```bash
pwd
ls -l ./config.yaml
```

然后使用绝对路径重新执行：

```bash
mihomo -t -f /完整路径/config.yaml
```

Windows 下可以执行：

```powershell
Get-Location
Get-ChildItem .\\config.yaml
```

如果使用 `-d`，还要检查配置目录中的默认配置文件，以及配置中引用的相对路径是否位于同一目录结构内。

### Q2：配置提示某个节点不存在，但我明明写过这个节点，为什么？

检查该节点是否被注释、名称是否有空格或大小写差异，以及代理组引用的是不是旧名称。建议使用编辑器的全文搜索，同时搜索节点定义和代理组引用，不要只查看配置文件顶部的 `proxies` 区块。

### Q3：`proxy-providers` 配置通过了，但代理组还是没有节点，怎么办？

继续检查三点：提供者文件是否为空或格式不符合预期；代理组的 `use` 是否引用了正确的提供者键名；提供者节点是否被 `filter`、`exclude-filter` 或 `exclude-type` 排除。还要确认客户端实际加载的是修改后的文件，而不是旧配置。

### Q4：能不能只用 YAML 在线检查工具？

在线 YAML 检查工具只能发现部分语法问题，通常不了解 Mihomo 的代理组、代理提供者及其引用关系，因此不能代替实际内核检查。更稳妥的做法是使用与客户端实际运行相同的 Mihomo 二进制，并保留完整的命令输出。

### Q5：配置检查通过后仍然连不上，是配置检查失效了吗？

不是。静态检查和实际连接是两个阶段。前者主要确认配置能够解析和初始化，后者还涉及监听端口、系统代理、当前工作模式、规则最终匹配的策略组，以及远程代理是否响应。此时应查看客户端日志和运行状态，而不是继续重复修改已经通过检查的 YAML。

### Q6：官方文档中的字段和当前客户端不一致怎么办？

先确认客户端内核版本，再对照该版本对应的官方文档和帮助信息。不同客户端可能内置不同版本的 Mihomo，也可能对配置文件进行额外转换。无法由当前版本文档确认的字段、参数或文件名，不建议直接套用其他教程中的写法。

## 参考资料

- [MetaCubeX/mihomo 官方仓库](https://github.com/MetaCubeX/mihomo)
- [Mihomo 官方 proxy-groups 配置文档](https://wiki.metacubex.one/en/config/proxy-groups/)
- [Mihomo 官方 proxy-providers 配置文档](https://wiki.metacubex.one/en/config/proxy-providers/)
- [Mihomo 官方 Releases 页面](https://github.com/MetaCubeX/mihomo/releases)

## 关于 云小狐

提供网络加速服务相关的平台说明、客户端使用教程和连接排查文档，帮助用户了解订阅导入、代理模式与常见配置问题。更多信息请访问官网：[https://www.yunxiaohu.com/](https://www.yunxiaohu.com/)。