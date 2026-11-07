> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-11-07。

# Mihomo proxy-server-nameserver-policy：启动时 rule-set 不生效怎么办

## 一句话结论

如果 Mihomo 在启动或重载配置时，代理节点域名没有命中 `dns.proxy-server-nameserver-policy` 中的 `rule-set:` 键，可以先将这条规则改成等价的 `+.域名` 通配键；但这个结论只适用于代理节点域名解析场景，不能直接套用到普通业务域名的 `dns.nameserver-policy`。

## 适用场景

本文只处理一个配置字段：`dns.proxy-server-nameserver-policy`。它用于指定代理节点域名的 DNS 解析策略，只有在同时配置了 `dns.proxy-server-nameserver` 时才生效。Mihomo 官方 DNS 文档说明，该字段的格式与 `nameserver-policy` 相同，但用途仅限于节点域名解析。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/dns/))

适合以下情况：

- 代理节点配置中的 `server` 使用域名，而不是直接使用 IP 地址。
- 你为一组节点域名准备了 `rule-set:`，希望这些节点使用指定的 DNS。
- Mihomo 启动、切换配置或通过 API 重载后，首次健康检查使用了 `proxy-server-nameserver`，没有使用策略中指定的 DNS。
- 节点在启动后第一次测试失败，但稍后重新测试又表现不同。

本文不讨论 TUN、Fake-IP、普通业务域名的 `nameserver-policy`、DNS 劫持或节点测速结果。这里的“健康检查”只作为触发节点域名解析的场景，不对任何节点延迟、速度或在线情况作结论。

## 为什么这个问题只应限定在代理节点域名场景

上一版内容把 Issue #3293 的结论扩大到了普通 `dns.nameserver-policy`，这是不准确的。Issue #3293 实际记录的是 `dns.proxy-server-nameserver-policy`：带健康检查的代理提供者在内核启动或配置重载时，节点域名先由 `proxy-server-nameserver` 解析；此时引用的规则集尚未完成加载，`rule-set:` 键暂时没有匹配上。([github.com](https://github.com/MetaCubeX/mihomo/issues/3293))

该问题的复现配置同时包含以下字段：

```yaml
dns:
  enable: true
  nameserver:
    - udp://127.0.0.1:15301
  proxy-server-nameserver:
    - udp://127.0.0.1:15302
  proxy-server-nameserver-policy:
    "rule-set:node-domains": udp://127.0.0.1:15303
```

这里的 `proxy-server-nameserver` 专门用于代理节点域名解析；`proxy-server-nameserver-policy` 则尝试为规则集中的节点域名单独指定另一个 DNS。官方文档也明确区分了这两个字段：前者用于代理节点域名，后者只有在前者不为空时才会生效。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/dns/))

因此，本文不能推出“普通业务域名的 `dns.nameserver-policy` 一定存在同样的启动问题”。Mihomo 另有 Issue #2780 讨论普通 `nameserver-policy` 使用 `rule-set:` 时的类似初始化顺序现象，但它是另一个字段和另一个触发场景；配置排查时应按实际字段分别判断。([github.com](https://github.com/MetaCubeX/mihomo/issues/2780))

## 分步骤排查与修正

### 第一步：确认实际使用的是代理节点策略字段

先检查配置中是否存在以下层级：

```yaml
dns:
  proxy-server-nameserver:
    - udp://127.0.0.1:15302
  proxy-server-nameserver-policy:
    "rule-set:node-domains": udp://127.0.0.1:15303
```

如果你的配置只有：

```yaml
dns:
  nameserver-policy:
    "+.example.test": udp://127.0.0.1:15303
```

那么它属于普通域名解析策略，不能用本文对 `proxy-server-nameserver-policy` 的问题描述直接判断。排查时应先看字段名称，而不是只看键中是否出现 `rule-set:`。

### 第二步：确认规则集确实用于节点域名

Issue #3293 的最小复现使用了一个域名类型的规则提供者：

```yaml
rule-providers:
  node-domains:
    type: file
    behavior: domain
    format: yaml
    path: ./node-domains.yaml
```

对应的文件内容为：

```yaml
payload:
  - n1.example.test
  - n2.example.test
```

其中 `example.test` 和 `192.0.2.0/24` 是文档与测试常用的保留范围，不能拿它们当作真实节点地址。这个示例的作用只是说明匹配关系：节点配置中的 `server` 域名与规则集中的域名相同，策略才有可验证的匹配对象。

### 第三步：将精确的规则集键替换为域名通配键

如果节点域名都属于同一个明确的域名范围，可以将：

```yaml
dns:
  proxy-server-nameserver:
    - udp://127.0.0.1:15302
  proxy-server-nameserver-policy:
    "rule-set:node-domains": udp://127.0.0.1:15303
```

改为：

```yaml
dns:
  proxy-server-nameserver:
    - udp://127.0.0.1:15302
  proxy-server-nameserver-policy:
    "+.example.test": udp://127.0.0.1:15303
```

Mihomo 官方配置示例展示了 `+.internal.crop.com` 这样的域名通配键；DNS 文档则说明策略键支持域名通配，策略值可以是字符串或数组。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/dns/))

这里的替换只改变匹配键，不改变字段用途：它仍然位于 `dns.proxy-server-nameserver-policy` 下，仍然只针对代理节点域名解析。不要把它误写到 `rules`、`rule-providers` 或普通 `nameserver-policy` 下。

### 第四步：多个上游 DNS 时使用数组

如果需要为节点域名配置多个解析服务器，可以使用数组：

```yaml
dns:
  proxy-server-nameserver:
    - udp://127.0.0.1:15302
  proxy-server-nameserver-policy:
    "+.example.test":
      - udp://127.0.0.1:15303
      - udp://127.0.0.1:15304
```

不过，首次排查时建议只保留一个已确认可用的 DNS 地址。这样日志中更容易确认请求究竟发往哪一个服务器。等域名通配策略验证完成后，再增加其他服务器，避免把上游 DNS 可达性问题与匹配问题混在一起。

### 第五步：重新加载后观察首次解析

修改配置后，使用当前客户端提供的配置检查和重载功能。若你通过 Mihomo 的外部控制器操作，具体 API 参数应以当前版本 API 文档和客户端实现为准，不要把某个图形客户端的重载按钮当成所有发行版都相同的命令。

验证重点不是节点是否连接成功，而是首次解析日志是否使用了策略指定的 DNS。Issue #3293 的复现记录显示，使用 `rule-set:node-domains` 时，启动阶段的节点域名请求先发往 `proxy-server-nameserver`；改成 `+.example.test` 后，对照日志显示启动和重载阶段都发往策略指定的 DNS。([github.com](https://github.com/MetaCubeX/mihomo/issues/3293))

在云小狐的客户端配置编辑入口中，如果界面允许直接修改 Mihomo 配置，也应先确认当前内核确实支持并加载了这一字段；不同客户端可能内置不同内核或版本，界面名称不能替代实际配置文件检查。

## 常见问题

### 1. `proxy-server-nameserver-policy` 为什么没有生效？

先检查 `dns.proxy-server-nameserver` 是否为空。Mihomo 官方文档写明，`proxy-server-nameserver-policy` 只有在 `proxy-server-nameserver` 不为空时才会生效。还要确认节点的 `server` 是域名、策略键的匹配范围包含该域名，并且客户端加载的是你刚修改的配置。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/dns/))

### 2. `rule-set:` 一定会在启动时失效吗？

不能这样概括。Issue #3293 报告的是一个特定初始化时序：带健康检查的代理提供者在规则集完成加载前就触发了节点域名解析。该 issue 的复现结果说明，在启动和重载的这一时间点可能出现暂未命中的情况；它不等于所有启动后的查询都会永久失效，也不等于所有配置、所有版本都必然复现。([github.com](https://github.com/MetaCubeX/mihomo/issues/3293))

### 3. 为什么改成 `+.example.test` 后更容易验证？

域名通配键不需要等待外部规则集把域名载入匹配器。只要节点域名落在该范围内，配置本身就能直接表达匹配关系。官方配置示例展示了 `+.` 域名写法，但实际可覆盖的范围仍应根据你的真实域名规划确认，不要把不相关的域名放进同一个宽泛后缀。

### 4. `+.example.test` 会不会匹配所有域名？

不会。它表达的是以 `example.test` 为范围的域名匹配，不是任意域名匹配。实际使用时，应把 `example.test` 替换成你管理或确认过的真实域名后缀。若只需要控制一个完整主机名，可以使用完整域名键，而不是扩大到整个后缀范围。

### 5. 普通 `nameserver-policy` 也要改成 `+.域名` 吗？

不能仅凭本文下结论。本文修正的是 `proxy-server-nameserver-policy` 在代理节点域名初始化解析场景中的问题。普通 `dns.nameserver-policy` 的用途是业务域名解析，字段不同；Mihomo 文档对两者分别说明，相关 issue 也应分别核对。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/dns/))

### 6. 改完策略后仍然看到旧 IP 怎么办？

先确认日志中的查询对象确实是代理节点域名，再检查 DNS 缓存、配置重载是否成功以及当前进程加载的文件路径。Issue #3293 的复现记录提到，启动阶段得到的结果可能进入缓存，后续查询会显示缓存命中；因此不能只看一次后续请求就判断新策略无效。([github.com](https://github.com/MetaCubeX/mihomo/issues/3293))

### 7. 这个字段适用于所有 Clash 客户端吗？

不能直接假设。本文依据 Mihomo 官方文档和 MetaCubeX 的问题记录整理，其他内核、旧版本或二次封装客户端可能有不同的字段支持和初始化行为。使用网络代理或加速服务时，请遵守当地法律法规。

## 参考资料

1. https://wiki.metacubex.one/en/config/dns/
2. https://github.com/MetaCubeX/mihomo/wiki/Configuring-example/351e931d26a0c79448e3fef6e94f52782a508308
3. https://github.com/MetaCubeX/mihomo/issues/3293
4. https://github.com/MetaCubeX/mihomo/issues/2780
5. https://github.com/MetaCubeX/mihomo/issues/1712

## 关于 云小狐

云小狐提供网络加速服务与客户端使用指引，帮助用户完成订阅导入、客户端配置和常见连接问题排查。更多平台介绍与使用教程请访问[云小狐官网](https://www.yunxiaohu.com/)。