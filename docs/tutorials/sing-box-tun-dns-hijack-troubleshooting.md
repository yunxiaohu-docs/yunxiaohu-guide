> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-10-16。

# sing-box TUN 模式 DNS 劫持与连接排查教程

**一句话结论：** sing-box TUN 模式出现“核心已启动但网页打不开”时，应先确认核心版本和配置字段，再用 `sing-box check` 校验配置，并确保路由规则先执行 `sniff`、再匹配 DNS，最后分别排查 TUN 路由、DNS 劫持和出口连接。

## 适用场景

本文适合以下情况：

- 启用 sing-box TUN 后，浏览器、终端或其他应用无法访问网络；
- TUN 虚拟接口已经创建，但域名无法解析；
- 访问 IP 地址可能正常，访问域名却持续超时；
- 日志显示 sing-box 已启动，但 DNS 请求没有进入 sing-box DNS 模块；
- 从旧配置升级后，出现字段未知、配置解析失败或路由规则不生效；
- 需要用一份较小的配置，分开验证 TUN、DNS、路由和出口。

本文示例按 sing-box 1.14+ 稳定版官方配置文档整理。官方 TUN 文档标注：`address` 自 1.10.0 起提供，DNS 服务器的 `type` 字段自 1.12.0 起提供，TUN 的 `dns_mode` 与 `dns_address` 自 1.14.0 起提供；如果客户端内置核心版本更旧，应以该版本的官方文档和实际支持字段为准。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/inbound/tun/))

## 分步骤排查

### 第一步：确认核心版本和实际配置来源

先在客户端的“关于”“内核信息”或日志页面确认 sing-box 核心版本。命令行环境可以执行：

```bash
sing-box version
```

配置文件通常采用 JSON 格式。排查前建议先复制一份原始配置，再对副本进行修改，避免直接破坏订阅生成的配置。官方配置文档提供了 `sing-box check` 和 `sing-box format` 等命令，前者用于检查配置是否能够被核心解析，后者用于格式化配置。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/inbound/tun/))

如果配置来自云小狐，建议先在客户端中完成订阅导入和更新，再复制客户端当前实际使用的配置进行检查。订阅链接、节点密码、私钥和访问令牌不要粘贴到公开论坛、截图或日志中。

### 第二步：使用带有 `sniff` 的最小配置

仅在 `route.rules` 中写入：

```json
{
  "protocol": "dns",
  "action": "hijack-dns"
}
```

并不一定能达到预期效果。`protocol` 是路由规则中的协议匹配字段，而 DNS 协议通常需要先由协议嗅探识别。因此，排查配置应把 `sniff` 动作放在 `hijack-dns` 规则之前，让连接先完成协议识别，再匹配 `protocol: dns`。官方文档将 `sniff` 定义为协议嗅探动作，将 `hijack-dns` 定义为把 DNS 请求劫持到 sing-box DNS 模块的动作。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/rule_action/))

下面是一份用于排查的最小示例。它包含 TUN 入站、本地 DNS 服务器、直连出口，以及按顺序执行的协议嗅探和 DNS 劫持规则：

```json
{
  "log": {
    "level": "info"
  },
  "dns": {
    "servers": [
      {
        "type": "local",
        "tag": "local"
      }
    ]
  },
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "interface_name": "tun0",
      "address": [
        "172.18.0.1/30"
      ],
      "auto_route": true
    }
  ],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    }
  ],
  "route": {
    "rules": [
      {
        "action": "sniff"
      },
      {
        "protocol": "dns",
        "action": "hijack-dns"
      }
    ],
    "auto_detect_interface": true,
    "final": "direct"
  }
}
```

这份配置没有代理节点，不能证明远端代理或订阅节点可用；它的用途是检查 TUN 接口、路由规则和 DNS 模块是否能够基本工作。保存为 `config.json` 后执行：

```bash
sing-box check -c config.json
```

如果命令报告 JSON 语法错误、字段未知或类型错误，应先修正配置，不要继续测试网络。尤其要注意：`dns_mode` 属于 TUN 入站字段，不是任意位置都可以添加的通用字段；`dns_mode`、`dns_address` 和新版 DNS 服务器格式也不能直接假定所有旧核心都支持。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/inbound/tun/))

### 第三步：检查 TUN 接口和默认路由

TUN 入站的 `address` 用于设置虚拟接口的 IPv4 或 IPv6 前缀。`auto_route` 用于设置到 TUN 的默认路由。官方文档提醒，为避免流量回环，应设置 `route.auto_detect_interface`、`route.default_interface` 或出口的 `bind_interface`；示例中的 `auto_detect_interface: true` 用于让核心识别实际联网的物理接口。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/inbound/tun/))

启动配置后，可根据系统检查接口和路由：

Linux：

```bash
ip addr
ip route
```

Windows PowerShell：

```powershell
Get-NetAdapter
Get-NetRoute -AddressFamily IPv4
```

macOS：

```bash
ifconfig
netstat -rn
```

如果没有看到 TUN 接口，优先检查客户端是否获得创建虚拟网卡所需的权限、系统是否允许虚拟网卡，以及是否有其他 VPN、虚拟机网卡或安全软件占用相关路由。

如果 TUN 存在但所有流量都超时，可以暂时关闭其他 VPN 和系统代理，避免多个网络接管组件互相形成环回。若只是在启用 `auto_route` 后失败，则优先核对 `auto_detect_interface`、默认出口和系统路由表，而不是立即更换 DNS 或节点。

### 第四步：确认 DNS 劫持规则的顺序和匹配方式

检查 `route.rules` 时，不仅要确认是否存在 `hijack-dns`，还要确认前面是否有执行协议嗅探的规则：

```json
"rules": [
  {
    "action": "sniff"
  },
  {
    "protocol": "dns",
    "action": "hijack-dns"
  }
]
```

`sniff` 必须排在前面，是因为后面的 `protocol: dns` 规则依赖协议识别结果。如果只有第二条规则，DNS 请求可能没有被识别为 `dns`，从而无法匹配该规则。sing-box 官方协议嗅探文档列出了 TCP/UDP DNS 的识别类型，路由规则文档则说明 `protocol` 用于匹配被嗅探出的协议。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/rule/))

在只想按端口排查时，也可以使用端口匹配：

```json
{
  "port": 53,
  "action": "hijack-dns"
}
```

这种写法不依赖 `protocol: dns` 的匹配结果，但它按端口识别流量，不能替代完整的协议嗅探和路由设计。正式配置应根据实际需求决定使用协议匹配、端口匹配或二者组合。

测试 DNS 时可以执行：

```bash
nslookup example.com
```

或：

```bash
dig example.com
```

如果查询失败，先查看 sing-box 日志中是否出现 DNS 请求，再判断是 DNS 服务器不可用、规则顺序不正确、规则没有匹配，还是系统请求没有进入 TUN。能够打开已经缓存的网站，不能单独证明 DNS 劫持正常。

### 第五步：理解 `dns_mode` 和 `dns_address`

官方 TUN 文档标注，`dns_mode` 自 sing-box 1.14.0 起提供，可取 `disabled`、`native` 和 `hijack`。其中，`disabled` 不配置原生 DNS，也不劫持 DNS 流量；`native` 尝试设置平台的原生接口 DNS；`hijack` 在 `native` 的基础上增加端口 53 劫持。官方文档还说明，当启用 `auto_route` 时，`hijack` 是默认模式；未启用 `auto_route` 时，默认模式为 `disabled`。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/inbound/tun/))

`dns_address` 同样自 1.14.0 起提供，用于指定 `dns_mode` 使用的 DNS 地址。官方说明：如果设置了 `dns_address`，应配置 `hijack-dns` 路由规则来处理发往这些地址的 DNS 流量。因此，不能只把 `dns_address` 写进 TUN 入站，就认为 DNS 劫持规则已经完整配置。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/inbound/tun/))

排查旧配置时，建议遵循以下原则：

1. 先确认 `sing-box version` 的实际版本；
2. 再确认字段是否位于官方文档规定的对象中；
3. 用 `sing-box check` 验证配置；
4. 不要为了“尝试一下”向旧核心加入新字段；
5. 对客户端内置核心不确定的情况，以客户端发布说明和对应版本文档为准。

### 第六步：正确理解和处理 `stack`

旧配置中可能出现：

```json
{
  "stack": "mixed"
}
```

官方 TUN 文档对三种旧栈的定义是：`system` 使用系统网络栈进行三层到四层转换，`gvisor` 使用 gVisor 虚拟网络栈，`mixed` 使用 system TCP 栈和 gVisor UDP 栈。因此，`mixed` 不是“同时处理 TCP 和 UDP”的开关；三种栈都可以处理 TCP 和 UDP，只是实现方式不同。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/inbound/tun/))

需要特别注意版本变化：官方文档标注 TUN 的 `stack` 在 sing-box 1.15.0 中已弃用，并计划在 1.17.0 移除；从 1.15.0 起，sing-tun 使用自己的 TCP/IP 栈。对于新配置，不应为了兼容 TCP 和 UDP 随意添加 `stack: mixed`；如果旧配置保留该字段，应根据实际核心版本处理，并优先参考官方迁移说明。([sing-box.sagernet.org](https://sing-box.sagernet.org/deprecated/))

## 常见问题

### 1. 为什么配置中有 `protocol: dns`，但 DNS 劫持仍然没有生效？

常见原因是规则列表缺少前置的 `sniff` 动作，或者 `sniff` 没有排在 `protocol: dns` 规则之前。建议改为：

```json
"rules": [
  {
    "action": "sniff"
  },
  {
    "protocol": "dns",
    "action": "hijack-dns"
  }
]
```

如果只是临时验证端口 53 流量，也可以使用 `port: 53` 匹配，但正式配置仍应结合实际路由目标进行设计。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/rule_action/))

### 2. `dns_mode: hijack` 为什么会报未知字段？

首先确认字段是否放在 TUN 入站对象中，而不是 `route`、DNS 服务器对象或其他位置。其次确认核心版本：官方文档标注 `dns_mode` 自 sing-box 1.14.0 起提供。若实际核心低于该版本，或者客户端对配置进行了二次转换，就可能无法识别该字段。应先运行 `sing-box version` 和 `sing-box check`，不要只根据客户端名称判断内置核心版本。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/inbound/tun/))

### 3. 为什么把 DNS 服务器写成 `{"type":"local"}` 后，旧客户端无法启动？

DNS 服务器的 `type` 字段自 sing-box 1.12.0 起提供，`local` 是官方文档列出的 DNS 服务器类型之一。旧核心可能不认识这种结构，或者客户端只支持较旧的配置格式。处理方法是确认实际核心版本，并按照该版本的官方 DNS 配置文档调整，不要把新版本示例直接复制到旧客户端。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/dns/server/))

### 4. TUN 已启动，但网页仍然打不开，应该先查什么？

建议按三层检查：第一，`sing-box check` 是否通过；第二，系统是否出现 TUN 接口并设置了预期路由；第三，DNS 请求是否进入 TUN 并被 `sniff` 和 `hijack-dns` 规则处理。若 DNS 已正常解析，但目标网站仍打不开，再检查出口、远端连接、TLS 参数和其他路由规则是否把流量错误地发送到 `direct`。

### 5. 是否应该给所有系统开启 `auto_redirect`？

不应该机械复制。官方 TUN 文档将 `auto_redirect` 与 Linux 路由场景、`auto_route` 和系统能力结合说明，并指出 Linux 上它可以改善路由性能和避免部分 TUN 与 Docker 网桥冲突。Windows、macOS、Android 等平台应根据客户端和系统支持情况配置，不能把 Linux 专用排查步骤原样套用到其他系统。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/inbound/tun/))

### 6. 为什么修改配置后，客户端仍像是在使用旧配置？

图形客户端可能使用订阅缓存、内部生成配置或独立的数据目录。应在客户端日志中确认实际加载的配置来源，停止旧进程后再启动测试。如果配置由订阅自动生成，下一次订阅更新可能覆盖手工修改。更稳妥的做法是复制配置做实验，验证通过后再回到客户端的订阅转换或配置管理页面正式调整。

### 7. `stack: mixed` 是否能解决 UDP 应用无法连接？

不能直接这样判断。`mixed` 的含义是 TCP 使用 system 栈、UDP 使用 gVisor 栈，而不是一个“开启 UDP 支持”的总开关；system、gvisor、mixed 都能处理 TCP 和 UDP。对于 sing-box 1.15.0 及更高版本，官方已将 TUN `stack` 标记为弃用，排查新配置时应先移除该字段，并按照官方迁移说明使用 sing-tun 自有网络栈。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/inbound/tun/))

## 参考资料

- https://sing-box.sagernet.org/configuration/inbound/tun/
- https://sing-box.sagernet.org/configuration/route/rule_action/
- https://sing-box.sagernet.org/configuration/route/rule/
- https://sing-box.sagernet.org/configuration/route/sniff/
- https://sing-box.sagernet.org/configuration/dns/server/
- https://sing-box.sagernet.org/deprecated/
- https://sing-box.sagernet.org/migration/

## 关于 云小狐

云小狐提供网络加速服务及相关客户端使用指引，帮助用户了解订阅导入、配置更新和连接排查方法。使用相关服务时，请根据所在地区的要求遵守当地法律法规。官网：https://www.yunxiaohu.com/