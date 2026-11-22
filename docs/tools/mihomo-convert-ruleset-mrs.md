> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-11-22。

# Mihomo convert-ruleset 怎么用：YAML/TXT 规则转换为 MRS

**一句话结论：**使用 `mihomo convert-ruleset domain/ipcidr yaml/text 输入文件 输出文件`，可以把域名或 IP-CIDR 规则转换成 `.mrs` 文件，但 MRS 当前只适用于 `domain` 和 `ipcidr` 两类 `rule-provider`，不能把 `classical` 规则直接按 MRS 使用。([github.com](https://github.com/MetaCubeX/Meta-Docs/blob/main/docs/config/rule-providers/index.md))

## 适用场景

当你手里已有一份本地规则文件，希望在 Mihomo 配置中通过 `rule-providers` 加载它时，可以考虑转换为 MRS。常见场景包括：

- 将纯文本域名列表转换为 `domain` 类型的 `.mrs` 文件；
- 将 IPv4 或 IPv6 网段列表转换为 `ipcidr` 类型的 `.mrs` 文件；
- 在本地编辑 YAML/TXT 规则，确认内容无误后生成 MRS 成品；
- 在脚本或 CI 流程中，把规则源文件编译为 Mihomo 可读取的 MRS 文件。

Mihomo 的 `rule-providers` 支持 `yaml`、`text` 和 `mrs` 等文件格式；其中 `mrs` 的 `behavior` 目前只支持 `domain` 或 `ipcidr`。官方文档给出的转换形式是 `mihomo convert-ruleset domain/ipcidr yaml/text XXX.yaml XXX.mrs`。([github.com](https://github.com/MetaCubeX/Meta-Docs/blob/main/docs/config/rule-providers/index.md))

如果你的目标只是临时验证规则，继续使用 YAML 或 TXT 会更直观；如果希望在配置中固定引用一个已经生成的规则文件，再使用 MRS。本文只讲 `convert-ruleset` 这条命令，不展开订阅转换、DNS、TUN 或策略组配置。

## 分步骤操作

### 1. 准备 Mihomo 可执行文件

先准备与当前系统匹配的 Mihomo 可执行文件，并确认终端可以找到它。官方启动文档提供了 Windows、Linux、macOS、FreeBSD 和 Android 等平台的预编译文件入口。([wiki.metacubex.one](https://wiki.metacubex.one/en/startup/))

在终端执行：

```bash
mihomo -v
```

如果系统提示找不到 `mihomo`，可以改用当前目录下的文件路径。例如 Linux 或 macOS 常见写法是：

```bash
./mihomo -v
```

Windows PowerShell 中可以写成：

```powershell
.\mihomo.exe -v
```

本文不指定某个版本号。版本号、系统架构和下载文件应以 Mihomo 官方发布页当前提供的文件为准，不要直接复制其他教程中未经核对的文件名。([github.com](https://github.com/MetaCubeX/mihomo/releases))

### 2. 准备 domain 类型的 TXT 规则

先用最小的域名列表测试。新建 `domain.txt`，每行放一个域名：

```text
example.com
+.example.org
```

这里的文件只是演示输入格式。实际使用时，应替换为你确认过来源和用途的域名列表。不要把订阅地址、账号、密码、访问令牌或其他敏感信息放进规则文件。

`domain` 表示输入内容按域名规则处理，命令中的 `text` 表示输入文件是纯文本格式。执行：

```bash
mihomo convert-ruleset domain text domain.txt domain.mrs
```

Windows PowerShell 的命令形式相同：

```powershell
.\mihomo.exe convert-ruleset domain text .\domain.txt .\domain.mrs
```

命令中的 5 个位置分别是：

```text
convert-ruleset 规则行为 输入格式 输入文件 输出文件
```

不要把 `domain` 和 `ipcidr` 对调。域名列表使用 `domain`，IP 网段列表使用 `ipcidr`。

### 3. 准备 ipcidr 类型的 TXT 规则

新建 `ipcidr.txt`，每行放一个网段。例如：

```text
192.0.2.0/24
2001:db8::/32
```

上面的地址段属于文档示例网段，适合用来验证命令格式。执行转换：

```bash
mihomo convert-ruleset ipcidr text ipcidr.txt ipcidr.mrs
```

Windows PowerShell：

```powershell
.\mihomo.exe convert-ruleset ipcidr text .\ipcidr.txt .\ipcidr.mrs
```

转换完成后，当前目录应出现 `ipcidr.mrs`。如果命令返回错误，先检查输入文件是否存在、每行是否为有效网段，以及 `ipcidr` 拼写是否正确。

### 4. 将 YAML 输入转换为 MRS

如果已有 YAML 格式的规则文件，可以把命令中的输入格式从 `text` 改为 `yaml`。例如：

```bash
mihomo convert-ruleset domain yaml domains.yaml domains.mrs
```

IP-CIDR 类型的 YAML 输入则写成：

```bash
mihomo convert-ruleset ipcidr yaml networks.yaml networks.mrs
```

完整命令仍然遵循以下结构：

```bash
mihomo convert-ruleset domain yaml domains.yaml domains.mrs
mihomo convert-ruleset ipcidr yaml networks.yaml networks.mrs
```

这里的 `yaml` 只表示输入文件格式，不表示输出文件必须使用 `.yaml` 扩展名。输出文件可以命名为 `.mrs`，并在配置中通过 `format: mrs` 声明。

### 5. 在 rule-providers 中引用 MRS

生成 `domain.mrs` 后，可以在 Mihomo 配置中通过 `type: file` 引用本地文件。下面是一个最小示例：

```yaml
rule-providers:
  local-domain:
    type: file
    path: ./domain.mrs
    behavior: domain
    format: mrs

rules:
  - RULE-SET,local-domain,PROXY
  - MATCH,DIRECT
```

如果引用的是 IP-CIDR 类型文件，应将 `behavior` 改为 `ipcidr`：

```yaml
rule-providers:
  local-ipcidr:
    type: file
    path: ./ipcidr.mrs
    behavior: ipcidr
    format: mrs

rules:
  - RULE-SET,local-ipcidr,PROXY
  - MATCH,DIRECT
```

官方配置文档将 `type` 列为 `http`、`file` 和 `inline`，并要求 `behavior` 与实际规则类型对应；`format` 可以写成 `yaml`、`text` 或 `mrs`。([github.com](https://github.com/MetaCubeX/Meta-Docs/blob/main/docs/config/rule-providers/index.md))

在使用云小狐提供的订阅配置时，如果需要加入自己维护的本地规则，应先确认客户端内核确实是 Mihomo，再把生成的 `.mrs` 文件放到配置中 `path` 所指向的位置；不要把本地文件路径误写成订阅 URL。

### 6. 检查文件路径和配置引用

建议把配置文件、规则文件和 MRS 文件放在容易确认的目录中，例如：

```text
profile/
├── config.yaml
├── domain.txt
└── domain.mrs
```

此时 `config.yaml` 中可以写：

```yaml
rule-providers:
  local-domain:
    type: file
    path: ./domain.mrs
    behavior: domain
    format: mrs
```

相对路径通常应以 Mihomo 的工作目录或配置目录为基准。若客户端使用了独立的配置目录、启动参数或沙箱目录，实际查找位置可能与当前终端目录不同。排查时可以先改用绝对路径验证，再决定是否恢复为相对路径。

## 常见问题

### 1. `classical` 规则能不能转换成 MRS？

不能直接按本文命令把 `classical` 规则作为 MRS 使用。官方文档明确说明，`mrs` 当前的 `behavior` 只支持 `domain` 和 `ipcidr`。如果规则文件包含 `DOMAIN-SUFFIX,example.com,PROXY` 这类带完整动作或多字段的 classical 规则，应继续使用适合 classical 的文本或 YAML 格式，或者先整理为单纯的域名/IP-CIDR 数据，再按对应行为转换。([github.com](https://github.com/MetaCubeX/Meta-Docs/blob/main/docs/config/rule-providers/index.md))

### 2. 为什么输入是 TXT，却把格式写成了 `yaml`？

`text` 和 `yaml` 指的是输入文件格式，必须与实际文件内容一致。纯文本逐行列表使用：

```bash
mihomo convert-ruleset domain text domain.txt domain.mrs
```

YAML 文件使用：

```bash
mihomo convert-ruleset domain yaml domains.yaml domains.mrs
```

文件扩展名只是提示，不能代替命令参数。把 TXT 文件标记为 `yaml`，或把 YAML 文件标记为 `text`，都可能导致转换失败或解析结果不符合预期。

### 3. `domain` 和 `ipcidr` 写错了会怎样？

这两个参数决定 Mihomo 如何解释输入内容。域名规则应使用 `domain`，网段规则应使用 `ipcidr`。如果把域名文件按 `ipcidr` 转换，或者把 IP 网段按 `domain` 转换，通常会在解析阶段失败；即使文件成功生成，也不应在配置中用错误的 `behavior` 引用。

建议让命令参数、输出文件和配置字段保持一致：

```text
domain.txt  -> domain.mrs  -> behavior: domain
ipcidr.txt -> ipcidr.mrs -> behavior: ipcidr
```

### 4. MRS 文件生成了，但配置仍然加载失败怎么办？

按以下顺序检查：

1. `path` 是否指向真实存在的文件；
2. `format` 是否写成 `mrs`；
3. `behavior` 是否与转换时使用的 `domain` 或 `ipcidr` 一致；
4. `rule-providers` 的名称是否与 `RULE-SET` 中的名称完全一致；
5. 当前客户端是否使用支持该格式的 Mihomo 内核；
6. 配置文件中的缩进、冒号和相对路径是否正确。

例如，配置中定义的是 `local-domain`，规则中也必须写：

```yaml
rules:
  - RULE-SET,local-domain,PROXY
```

不能写成其他名称。Mihomo 的路由规则按从上到下的顺序匹配，`RULE-SET` 需要先有对应的 `rule-provider` 定义。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/rules/))

### 5. 可以直接下载网上的 MRS 文件吗？

可以使用 `type: http` 引用远程规则，但应核对来源、文件类型和配置中的 `behavior`。官方规则数据仓库提供了面向 Mihomo 的规则数据和 MRS 资源入口；使用第三方文件前，还应确认其内容、许可证和更新方式。([github.com](https://github.com/MetaCubeX/meta-rules-dat))

如果只是测试本地配置，优先使用自己生成的 MRS 文件。不要把来源不明的规则文件直接加入长期使用的配置，也不要在规则文件或 URL 中暴露订阅凭证。

## 参考资料

- [MetaCubeX Meta-Docs：rule-providers 配置](https://github.com/MetaCubeX/Meta-Docs/blob/main/docs/config/rule-providers/index.md)
- [mihomo 官方文档：启动与预编译文件](https://wiki.metacubex.one/en/startup/)
- [mihomo 官方文档：路由规则](https://wiki.metacubex.one/en/config/rules/)
- [MetaCubeX/mihomo 官方发布页](https://github.com/MetaCubeX/mihomo/releases)
- [MetaCubeX/meta-rules-dat 官方规则数据仓库](https://github.com/MetaCubeX/meta-rules-dat)

## 关于 云小狐

云小狐提供网络加速服务相关的平台说明、客户端教程、订阅导入指引和连接排查文档。使用相关服务和工具时，请遵守当地法律法规。

官网：[https://www.yunxiaohu.com/](https://www.yunxiaohu.com/)