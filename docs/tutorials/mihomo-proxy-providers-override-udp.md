> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-11-13。

# Mihomo proxy-providers override udp 怎么写：统一开启订阅节点 UDP

**一句话结论：**如果订阅节点需要允许 UDP 流量，可以在对应 `proxy-providers` 的 `override` 下加入 `udp: true`；该字段会在节点从订阅提供者载入时覆盖节点自身的 UDP 配置，但是否真正可用仍取决于节点协议、客户端模式和实际网络环境。

## 适用场景

这篇文章只处理一个配置字段：`proxy-providers.override.udp`。它适合以下情况：

- 订阅已经能够正常更新，但部分 UDP 应用无法建立连接；
- 不想逐个修改订阅生成的节点，希望对某个订阅提供者中的节点统一设置 UDP；
- 从旧版 Clash 配置迁移到 Mihomo 时，不确定 `udp` 应该写在单个节点下，还是写在 `proxy-providers.override` 下；
- 使用 TUN、游戏、实时通信或其他可能产生 UDP 流量的场景，需要先确认节点配置是否允许 UDP。

本文不讨论 TUN、DNS 劫持、规则集或测速参数，也不把“配置中写了 `udp: true`”等同于“所有 UDP 流量一定可用”。

## 1. 先区分节点字段与订阅覆盖字段

Mihomo 的普通代理节点可以直接使用公共字段 `udp`。官方文档说明，`udp` 用于控制是否允许通过代理发送 UDP，默认值为 `false`；但 TUIC 等基于 UDP 的协议，以及 `direct` 和 `dns` 类型，默认行为有所不同。也就是说，`udp` 是节点级别的能力开关，不是全局代理模式开关。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/proxies/))

当节点来自 `proxy-providers` 时，可以在提供者下增加 `override`。Mihomo 文档将 `override` 定义为“节点载入时覆盖节点配置”，并明确列出 `udp` 属于可覆盖的公共字段。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/proxy-providers/))

因此，下面两种写法作用范围不同：

```yaml
proxies:
  - name: "单个节点"
    type: ss
    server: server.example.com
    port: 443
    cipher: chacha20-ietf-poly1305
    password: "your-password"
    udp: true
```

上面只修改一个手写节点。对于订阅节点，应把字段放在对应的 `proxy-providers.<provider-name>.override` 中：

```yaml
proxy-providers:
  my-subscription:
    type: http
    url: "保留你现有的订阅地址"
    path: ./proxy_providers/my-subscription.yaml
    interval: 3600
    override:
      udp: true
```

第二种写法的重点是层级：`udp` 不要与 `override` 同级，也不要放到 `proxy-groups` 下。YAML 对缩进和大小写敏感，键名应写成小写 `udp`。Mihomo 的配置语法使用空格缩进，不允许使用 Tab。([github.com](https://github.com/MetaCubeX/Meta-Docs/blob/main/docs/handbook/syntax.en.md))

## 2. 在现有订阅提供者中加入 override.udp

### 步骤一：备份当前配置

修改前先复制一份当前配置文件。例如，可以将配置保存为：

```text
config.yaml
config.yaml.bak
```

如果使用客户端管理配置，不要直接改动订阅服务器返回的原始内容。应修改本地覆写配置，或者在客户端提供的本地 Mihomo 配置中增加字段，否则下一次更新订阅时，手工修改可能会被覆盖。

### 步骤二：找到 proxy-providers

在配置中搜索：

```yaml
proxy-providers:
```

常见结构如下：

```yaml
proxy-providers:
  provider-a:
    type: http
    url: "保留现有地址"
    path: ./proxy_providers/provider-a.yaml
    interval: 3600
```

只处理需要开启 UDP 的那个提供者。如果配置中有多个订阅，不要把 `override` 随意复制到所有提供者；每个提供者的节点协议和用途可能不同。

### 步骤三：加入 override.udp

将配置改为：

```yaml
proxy-providers:
  provider-a:
    type: http
    url: "保留现有地址"
    path: ./proxy_providers/provider-a.yaml
    interval: 3600
    override:
      udp: true
```

如果原来已经有 `override`，只在其中补充一行：

```yaml
proxy-providers:
  provider-a:
    type: http
    url: "保留现有地址"
    path: ./proxy_providers/provider-a.yaml
    interval: 3600
    override:
      additional-prefix: "订阅A-"
      udp: true
```

不要重复写两个 `override`：

```yaml
# 错误示例：同一层出现两个 override
proxy-providers:
  provider-a:
    override:
      udp: true
    override:
      tfo: false
```

YAML 中重复键可能造成后一个对象覆盖前一个对象，最终导致前面的设置没有按预期生效。合并其他字段时，也要确认它们位于同一个 `override` 对象内。

### 步骤四：检查配置并重新载入

保存后，先使用客户端自带的配置检查功能，或使用你当前 Mihomo 安装包支持的配置检查命令。由于不同发行版的启动参数和工作目录可能不同，不要直接复制其他客户端的完整启动命令；重点是确认配置能够被当前核心正常解析。

检查通过后，重新载入配置，再执行一次订阅更新。`override` 的作用时机是节点被提供者载入时，因此只保存文件但不重载配置，界面中可能仍显示旧的节点对象。

如果客户端提供 Mihomo API，还可以通过 `/providers/proxies` 查看代理提供者及其节点列表；官方 API 文档将该接口定义为获取全部代理集合信息，具体提供者可通过 `/providers/proxies/{providers_name}` 查询。([wiki.metacubex.one](https://wiki.metacubex.one/en/api/))

## 3. 用最小范围确认是否生效

确认时不要同时修改多个字段。建议只保留以下验证顺序：

1. 订阅提供者能够更新；
2. 配置检查没有 YAML 解析错误；
3. 重新载入后，目标提供者仍然包含节点；
4. 仅对目标应用或目标流量进行测试；
5. 如果仍失败，再检查节点协议是否支持 UDP、客户端是否启用了对应接管模式，以及本地网络是否允许 UDP。

需要注意，`udp: true` 表示允许该节点使用 UDP，并不负责把 TCP 节点转换成 UDP 节点，也不会改变节点的服务器地址、端口或传输协议。官方文档对该字段的定义是“是否允许 UDP 通过代理”，默认值为 `false`；因此它解决的是配置层面的允许状态，而不是协议兼容性或网络连通性问题。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/proxies/))

如果你通过云小狐获取订阅，建议先在客户端完成订阅更新，再针对本地 Mihomo 配置增加这一处 `override.udp`，不要修改订阅链接本身或把凭据写进公开配置。

## 常见问题

### 1. `udp: true` 应该放在 proxy-groups 里吗？

不应该。本文处理的是订阅提供者载入节点时的覆盖字段，正确路径是：

```text
proxy-providers → 提供者名称 → override → udp
```

`proxy-groups` 负责组织和选择节点，不能替代 `proxy-providers.override` 对节点配置的覆盖。

### 2. 写了 `udp: true`，为什么 UDP 还是无法使用？

首先确认配置是否被重新载入，其次确认修改的是实际使用的提供者。之后检查节点协议是否支持 UDP、客户端是否以适合的模式接管流量，以及本地网络是否放行 UDP。该字段只改变节点的 UDP 允许配置，不保证远端服务、传输协议或网络路径一定支持 UDP。

### 3. 订阅更新后，`udp: true` 消失了怎么办？

不要直接修改订阅返回的节点文件。应把 `override` 放在本地 Mihomo 主配置的 `proxy-providers` 中，并确认客户端没有在更新时替换整个本地配置。如果客户端使用独立的覆写文件，应确认该文件确实被加载。

### 4. 可以同时写 `udp: true` 和 `udp-over-tcp: false` 吗？

这两个字段不是同一个功能。`udp` 是公共节点字段，控制是否允许 UDP；`udp-over-tcp` 是特定协议相关的传输选项。本文只建议处理 `udp`，不要为了“让 UDP 可用”而盲目增加其他字段。只有在确认节点协议和 Mihomo 文档都支持对应选项时，才单独配置它。

### 5. `udp` 写成字符串会有影响吗？

建议使用 YAML 布尔值：

```yaml
udp: true
```

不要写成：

```yaml
udp: "true"
```

前者是布尔值，后者是字符串。配置字段类型应与文档示例保持一致，避免客户端或核心在解析时产生类型错误或忽略字段。

## 参考资料

- [Mihomo proxy-providers 配置文档](https://wiki.metacubex.one/en/config/proxy-providers/)
- [Mihomo 公共代理字段文档](https://wiki.metacubex.one/en/config/proxies/)
- [MetaCubeX Meta-Docs：配置语法](https://github.com/MetaCubeX/Meta-Docs/blob/main/docs/handbook/syntax.en.md)
- [Mihomo 配置示例](https://github.com/MetaCubeX/mihomo/wiki/Configuring-example/b8451d15bc05f4b8f73a9174f09746f363cbd426)
- [Mihomo API 文档](https://wiki.metacubex.one/en/api/)

## 关于 云小狐

云小狐提供网络加速服务及相关客户端、订阅导入和连接排查教程。更多平台介绍与使用说明，请访问[官网](https://www.yunxiaohu.com/)。