> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-11-06。

# Mihomo rule-providers 的 format 字段怎么写：yaml 与 text

**一句话结论：`rule-providers` 中的 `format` 必须与规则文件的实际封装格式一致；文件包含 `payload:` 和 YAML 列表时使用 `format: yaml`，文件按行直接保存规则时使用 `format: text`，不能只根据 `.yaml`、`.list` 或 `.txt` 文件后缀判断。**

## 适用场景

本文只处理 Mihomo `rule-providers` 的 `format` 字段，适合以下情况：

- 规则提供者加载失败，但不知道是 `yaml` 还是 `text`；
- 规则文件扩展名和配置中的 `format` 不一致；
- `behavior: domain` 的规则文件明明能打开，却无法被规则提供者正常解析；
- 修改配置后出现规则集为空、`RULE-SET` 不生效等问题。

Mihomo 官方文档将 `format` 的可选值列为 `yaml`、`text` 和 `mrs`，默认值为 `yaml`。同时，`behavior` 可设置为 `domain`、`ipcidr` 或 `classical`，需要根据规则文件的内容类型填写。也就是说，`format` 描述文件如何组织，`behavior` 描述文件中的规则属于哪一类，两者不能混为一谈。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/rule-providers/))

## 第一步：先看文件内容，不要先看文件后缀

判断 `format` 时，先打开规则文件，观察它是 YAML 列表还是逐行文本。文件名可以帮助整理，但不是决定解析方式的依据。

### YAML 格式：文件中包含 `payload:`

如果规则文件包含 `payload:`，并且后面是 YAML 列表，应使用 `format: yaml`。例如，`./rules/example-domains.yaml` 的内容可以写成：

```yaml
payload:
  - "+.example.com"
  - ".example.net"
  - "www.example.org"
```

对应的 `rule-providers` 配置应保持一致：

```yaml
rule-providers:
  example-domains:
    type: file
    path: ./rules/example-domains.yaml
    behavior: domain
    format: yaml
```

这里的关键是：文件内容有 `payload:`，所以配置使用 `format: yaml`。如果把同一个文件改名为 `.list`，但内容仍然保留 `payload:`，也不能因此改成 `format: text`；扩展名变化不会改变文件的实际格式。

Mihomo 的规则提供者内容文档分别展示了 `domain`、`classical` 和 `ipcidr` 规则集的 YAML 写法，这些写法都通过 `payload:` 保存列表内容。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/rule-providers/content/))

### 纯文本格式：每行直接放一条规则

如果文件没有 `payload:`，而是每行直接放置一个域名或一条传统路由规则，应使用 `format: text`。例如，`./rules/example-domains.list` 的内容可以写成：

```text
+.example.com
.example.net
www.example.org
```

对应配置如下：

```yaml
rule-providers:
  example-domains:
    type: file
    path: ./rules/example-domains.list
    behavior: domain
    format: text
```

此时文件内容与配置是一致的：文件是逐行文本，配置是 `format: text`。如果仍然写成 `format: yaml`，Mihomo 会按照 YAML 规则解析，而不是按照逐行文本解析。

对于 `classical` 规则集，纯文本文件也可以每行保存一条传统规则，例如：

```text
DOMAIN-SUFFIX,example.com
DOMAIN-KEYWORD,example
IP-CIDR,192.0.2.0/24,no-resolve
```

配置可以写成：

```yaml
rule-providers:
  example-classical:
    type: file
    path: ./rules/example-classical.list
    behavior: classical
    format: text
```

官方示例同时列出了 `classical` 的 YAML 和 text 内容：前者带有 `payload:`，后者直接逐行保存规则。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/rule-providers/content/))

## 第二步：修正最容易出现的配置错配

最常见的错误是：文件内容使用了 YAML 写法，但配置仍然使用 `format: text`。例如，下面的文件内容属于 YAML：

```yaml
payload:
  - "+.example.com"
  - "www.example.org"
```

如果配置却写成下面这样，就不匹配：

```yaml
rule-providers:
  example-domains:
    type: file
    path: ./rules/example-domains.list
    behavior: domain
    format: text
```

这里同时存在两个问题：

1. 文件内容包含 `payload:`，应当使用 `format: yaml`；
2. 路径虽然不决定解析格式，但 `.list` 容易造成误判，建议将文件名改为 `.yaml`，让文件名与实际内容保持直观一致。

修正后的写法如下：

```yaml
rule-providers:
  example-domains:
    type: file
    path: ./rules/example-domains.yaml
    behavior: domain
    format: yaml
```

如果希望继续使用 `format: text`，则应删除 `payload:`，把文件改成逐行内容：

```text
+.example.com
www.example.org
```

再使用：

```yaml
rule-providers:
  example-domains:
    type: file
    path: ./rules/example-domains.list
    behavior: domain
    format: text
```

排查时建议二选一，不要只修改 `format` 而不检查文件内容。

## 第三步：确认 domain 规则中的通配写法

`behavior: domain` 的内容需要遵循 Clash 域名通配规则。Mihomo 官方语法文档说明，`*` 一次只能匹配一层域名；`+` 可以匹配多层，并且 `+.example.com` 也能匹配根域名 `example.com`；`.` 可以匹配多层，但不匹配根域名。使用这些通配符时，官方建议将内容放在引号中。([wiki.metacubex.one](https://wiki.metacubex.one/en/handbook/syntax/))

例如：

```yaml
payload:
  - "+.example.com"
  - ".example.net"
  - "*.example.org"
```

这几种写法的覆盖范围不同：

| 写法 | 官方语法含义 |
| --- | --- |
| `+.example.com` | 匹配 `example.com` 及多层子域名 |
| `.example.net` | 匹配多层子域名，但不匹配 `example.net` 本身 |
| `*.example.org` | 只匹配一层子域名，不匹配根域名或更深层级 |
| `www.example.org` | 匹配明确写出的域名 |

不要把规则提供者中的域名通配符和路由规则 `DOMAIN-WILDCARD` 混用。官方路由规则文档特别说明，这两套通配语法不同；`DOMAIN-WILDCARD` 使用的是 `*` 和 `?` 的另一种匹配方式。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/rules/))

## 第四步：用一个最小规则集验证 format

为了避免代理组、DNS、TUN 或其他配置干扰，建议先使用一个本地文件和一条 `RULE-SET` 进行验证。

先创建 `./rules/test.list`：

```text
DOMAIN-SUFFIX,example.com
```

然后在主配置中加入：

```yaml
rule-providers:
  test-rules:
    type: file
    path: ./rules/test.list
    behavior: classical
    format: text

rules:
  - RULE-SET,test-rules,REJECT
  - MATCH,DIRECT
```

这里需要检查三处内容：

- 文件确实是逐行文本，没有 `payload:`；
- `behavior: classical` 与 `DOMAIN-SUFFIX,...` 这类传统路由规则相匹配；
- `RULE-SET,test-rules,REJECT` 中的 `test-rules` 与 `rule-providers` 下的键名完全相同。

如果你使用云小狐的订阅作为基础配置，建议先复制一份配置文件，再单独加入这个最小规则提供者进行验证。这样可以先确认 `format` 是否正确，避免把订阅更新、代理组选择和规则解析问题混在一起排查。

## 常见问题

### 1. 文件名是 `.yaml`，是否一定要写 `format: yaml`？

不一定。决定 `format` 的是文件内容，而不是后缀。带有 `payload:` 的 YAML 列表通常使用 `format: yaml`；没有 `payload:`、每行一条规则的文件使用 `format: text`。为了降低误判，建议让文件后缀与实际内容保持一致，但不要把后缀当作自动识别依据。Mihomo 官方文档还说明，`format` 默认值是 `yaml`，所以纯文本规则集最好明确写出 `format: text`。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/rule-providers/))

### 2. `payload:` 文件能不能继续使用 `format: text`？

不能按同一份内容这样配置。`payload:` 表示文件使用 YAML 列表封装，应将 `format` 改为 `yaml`；如果必须使用 `text`，就要删除 `payload:`，把每条内容改为独立一行。

### 3. `behavior: domain` 和 `format: yaml` 是不是固定搭配？

不是固定搭配。`behavior: domain` 可以与 YAML 或 text 内容组合：带 `payload:` 的域名列表使用 `yaml`，逐行域名列表使用 `text`。官方配置文档将 `behavior` 和 `format` 分开说明，实际填写时需要同时参考规则类型和文件封装方式。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/rule-providers/))

### 4. 为什么通配域名建议加引号？

因为 YAML 中的 `*` 还可以用于别名引用，某些以通配符开头的内容不加引号时可能被 YAML 解析器当作语法，而不是普通字符串。Mihomo 官方语法示例建议在使用域名通配符时使用单引号或双引号。([wiki.metacubex.one](https://wiki.metacubex.one/en/handbook/syntax/))

### 5. 修改 `format` 后仍然加载失败怎么办？

先把规则文件缩减为一条内容，再逐项检查：文件路径是否正确、文件是否真的存在、内容是否包含 `payload:`、`behavior` 是否匹配规则类型，以及 `RULE-SET` 使用的名称是否与 `rule-providers` 的键名一致。若最小文件能够加载，再逐步恢复原始规则。不要在同一次修改中同时调整代理组、DNS、TUN 和规则文件，否则很难确定实际错误来源。

## 参考资料

1. https://wiki.metacubex.one/en/config/rule-providers/
2. https://wiki.metacubex.one/en/config/rule-providers/content/
3. https://wiki.metacubex.one/en/handbook/syntax/
4. https://wiki.metacubex.one/en/config/rules/

## 关于 云小狐

该平台提供网络连接服务及相关客户端、订阅导入和配置使用指引，具体功能与使用方式以官方页面说明为准。使用相关服务和工具时，请遵守当地法律法规。

官网：https://www.yunxiaohu.com/