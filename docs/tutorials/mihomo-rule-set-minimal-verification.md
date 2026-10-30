> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-10-30。

# Mihomo 规则命中排查：用最小配置验证 `RULE-SET` 顺序

**一句话结论：**当 Mihomo 中某条规则看起来没有生效时，先用 `inline` 类型的最小规则集复现，再通过规则顺序、`MATCH` 兜底和 API 返回的命中信息判断问题是在规则内容、引用关系，还是实际加载的配置文件。

## 适用场景

本文适合以下情况：

- 修改了 `rules` 或 `rule-providers` 后，不确定规则是否被加载；
- 使用 `RULE-SET` 后，目标域名仍然按照兜底规则处理；
- 多条规则同时存在，不清楚究竟是哪一条先匹配；
- 想在加入远程规则集、订阅配置或复杂策略组之前，先验证单条规则；
- 客户端界面没有直接显示预期的规则命中结果，需要通过 Mihomo API 辅助确认。

Mihomo 的路由规则按照配置中的顺序从上到下匹配，排在前面的规则优先级更高；`MATCH` 会匹配所有没有被前面规则处理的请求。因此，最小验证配置应当只保留一个测试规则集、一条对应的 `RULE-SET` 和一条明确的 `MATCH` 兜底规则。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/rules/))

本文只验证规则是否被解析和命中，不测试节点速度、线路质量、在线率或流媒体解锁结果。

## 分步骤验证

### 第 1 步：准备一个独立的最小配置

先不要直接修改正在使用的订阅配置。新建一个临时文件，例如 `rule-test.yaml`，写入以下内容：

```yaml
mixed-port: 7890
mode: rule
log-level: info

external-controller: 127.0.0.1:9090
secret: ""

rule-providers:
  test-domain:
    type: inline
    behavior: classical
    payload:
      - DOMAIN-SUFFIX,example.com

rules:
  - RULE-SET,test-domain,REJECT
  - MATCH,DIRECT
```

这个示例使用 `inline` 规则提供者，把测试规则直接写在配置文件中，因此不依赖远程下载地址。`behavior: classical` 表示规则内容使用带有规则类型和参数的经典格式；`RULE-SET,test-domain,REJECT` 则把名为 `test-domain` 的规则集关联到 `REJECT` 策略。官方文档列出了 `inline`、`classical`、`payload` 以及 `RULE-SET` 的对应关系。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/rule-providers/))

`example.com` 是这里的测试域名。该示例的目标不是访问站点，而是观察匹配结果：如果请求确实命中测试规则，应该由 `REJECT` 处理；如果没有命中，才会继续落到后面的 `MATCH,DIRECT`。

### 第 2 步：确认客户端加载的是测试文件

在客户端中导入或选择 `rule-test.yaml`，然后重新加载配置。不同客户端的菜单名称可能不同，可以重点确认以下信息：

- 当前激活的配置文件名是否为 `rule-test.yaml`；
- 配置详情中的文件路径是否与新建文件一致；
- 是否启用了远程模板、配置合并或自动生成配置；
- 重新加载后，端口是否仍然是示例中的 `7890`；
- API 监听地址是否为 `127.0.0.1:9090`。

如果客户端继续使用旧配置，那么后续请求不会反映这份测试文件的结果。遇到“文件内容看起来正确但规则不生效”时，首先排除加载了错误文件这一情况。

### 第 3 步：观察最小规则的行为

在本机通过 Mihomo 的 HTTP 代理端口发起一个请求：

```bash
curl -I -x http://127.0.0.1:7890/ https://example.com/
```

Windows PowerShell 也可以使用：

```powershell
curl.exe -I -x http://127.0.0.1:7890/ https://example.com/
```

由于测试规则将 `example.com` 的域名后缀关联到 `REJECT`，请求可能直接失败或被代理内核拒绝。这里关注的是请求是否被拒绝，而不是响应速度、网络延迟或目标站点是否可访问。

然后再请求一个不匹配测试规则的域名，例如：

```bash
curl -I -x http://127.0.0.1:7890/ https://www.iana.org/
```

该请求不符合 `DOMAIN-SUFFIX,example.com`，理论上会继续匹配 `MATCH,DIRECT`。如果这两个请求表现不同，说明最小规则集和规则顺序至少已经产生了可观察差异。

### 第 4 步：通过 API 查看规则条目和命中信息

配置中的 `external-controller` 开启了本机 REST API。官方 API 文档说明，`GET /rules` 可以返回规则数组；规则对象可能包含 `index`、`type`、`payload`、`proxy`，以及命中次数和最近命中时间等信息。([wiki.metacubex.one](https://wiki.metacubex.one/en/api/))

Linux、macOS 或其他带有 `curl` 的环境可以执行：

```bash
curl http://127.0.0.1:9090/rules
```

如果配置设置了 API 密钥，应当带上认证请求头：

```bash
curl \
  -H "Authorization: Bearer YOUR_SECRET" \
  http://127.0.0.1:9090/rules
```

示例配置中的 `secret` 为空，仅用于本机临时验证。不要把 API 监听地址随意改为公网可访问的地址，也不要把真实密钥粘贴到公开工单、截图或代码仓库中。官方配置文档将 `external-controller` 定义为控制内核的 REST API 监听地址，并提供了 `secret` 字段用于访问控制。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/general/))

返回内容中的 `RULE-SET` 条目应当对应 `test-domain` 和 `REJECT`。完成一次测试请求后，如果当前内核版本和客户端提供了命中统计字段，可以再次请求 `/rules`，比较该规则对象中的命中信息是否发生变化。不要把一次请求没有出现命中统计，直接判断为规则失效；应先确认客户端是否隐藏了字段、当前内核是否提供该字段，以及请求是否真的经过这个内核。

### 第 5 步：验证规则顺序

把配置临时改成下面这样：

```yaml
rules:
  - MATCH,DIRECT
  - RULE-SET,test-domain,REJECT
```

再次请求 `https://example.com`.xn--6kqu55g/ `MATCH` 会匹配所有请求，而且它已经排在测试规则前面，测试规则将没有机会处理该请求。这个对照实验可以直观说明：规则不是“全部执行”，而是按照从上到下的顺序尝试匹配，前面的通用规则可能遮挡后面的具体规则。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/rules/))

验证完成后恢复为：

```yaml
rules:
  - RULE-SET,test-domain,REJECT
  - MATCH,DIRECT
```

实际配置中，通常应将更具体的域名、IP 或规则集放在更宽泛的规则前面，把 `MATCH` 放在规则列表末尾作为兜底。具体排列仍应结合你的策略目标和官方字段说明进行检查。

### 第 6 步：再替换为远程规则提供者

确认 `inline` 规则能够命中后，再把测试规则改为远程 `http` 提供者。远程提供者至少要根据实际内容填写 `type`、`url`、`path`、`behavior` 和 `format` 等字段；其中 `behavior` 应与远程规则文件的格式相符，官方文档列出的行为类型包括 `domain`、`ipcidr` 和 `classical`，格式包括 `yaml`、`text` 和 `mrs`。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/rule-providers/))

示例结构如下，网址仅作为格式占位，不应直接当作可用订阅：

```yaml
rule-providers:
  remote-test:
    type: http
    url: https://example.com/rules.yaml
    path: ./rules/remote-test.yaml
    interval: 86400
    behavior: classical
    format: yaml

rules:
  - RULE-SET,remote-test,REJECT
  - MATCH,DIRECT
```

如果正在使用云小狐提供的订阅服务，建议先在客户端复制一份当前配置，再单独验证规则集或订阅转换结果，避免一次更新同时改变节点、代理组和规则，导致问题范围难以判断。

## 常见问题

### 1. `RULE-SET` 写对了，但 API 中看不到对应规则，怎么办？

先确认客户端确实加载了修改后的配置，并检查 `rule-providers` 的名称与 `RULE-SET` 中的名称完全一致。例如提供者名称是 `test-domain`，引用时也必须写成 `RULE-SET,test-domain,REJECT`。同时确认 YAML 缩进正确，且没有被远程模板或配置合并覆盖。

### 2. 为什么把 `MATCH` 放到前面后，后面的规则全部没有效果？

因为 `MATCH` 不带条件，会匹配所有请求。规则按从上到下的顺序处理，前面的 `MATCH` 已经提供了结果，后面的规则自然不会再成为首个匹配项。将具体规则放在 `MATCH` 前面即可进行对照验证。

### 3. `inline` 规则能命中，换成远程规则集后却没有命中，问题通常在哪里？

优先检查远程文件是否能被当前内核读取、`behavior` 是否与文件内容一致、`format` 是否填写正确，以及 `path` 是否可写。还要检查远程文件本身是否包含预期的规则。先用 `inline` 验证 `RULE-SET` 的引用和顺序，再只替换提供者部分，可以减少变量。

### 4. `curl` 请求失败，能否直接证明规则已经命中？

不能。请求失败也可能由端口未监听、客户端没有加载测试配置、代理端口类型不匹配或其他连接问题导致。应同时检查客户端日志、当前配置路径和 `/rules` API 返回结果；规则命中验证与网络连接验证是两个不同层次的问题。

### 5. 为什么看不到 `hitCount` 或最近命中时间？

官方 API 文档将这些字段标为规则对象中可能出现的附加信息。实际返回内容还会受到内核版本、客户端封装方式和当前运行状态影响。因此，不能仅凭缺少某个统计字段就判定规则无效；应结合规则条目、请求行为和日志一起判断。

## 参考资料

- Mihomo 路由规则文档：<https://wiki.metacubex.one/en/config/rules/%3E
- Mihomo 规则提供者文档：<https://wiki.metacubex.one/en/config/rule-providers/%3E
- Mihomo API 文档：<https://wiki.metacubex.one/en/api/%3E
- Mihomo 全局配置文档：<https://wiki.metacubex.one/en/config/general/%3E

## 关于 云小狐

提供网络加速服务相关的平台说明、客户端使用教程与配置排查资料，帮助用户更清晰地完成订阅导入和连接设置。使用相关服务时，请遵守当地法律法规。官网：<https://www.yunxiaohu.com/%3E