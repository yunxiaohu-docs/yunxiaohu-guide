# sing-box TUN 配置报错：inet4_address 如何迁移到 address

> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-10-25。

**一句话结论：** 如果 sing-box 报错提示 `inet4_address`、`inet6_address` 或相关旧 TUN 字段不存在，应把 IPv4/IPv6 地址合并到 `address`，把旧路由字段分别合并到 `route_address` 和 `route_exclude_address`，然后再运行配置检查。

## 适用场景

本文适用于以下情况：

- 使用 sing-box TUN 模式时，启动日志提示 `unknown field`、`deprecated` 或旧字段已移除；
- 从较早版本配置文件迁移到 sing-box 1.12 或更高版本；
- 手动编辑 sing-box 配置，发现旧教程仍在使用 `inet4_address`、`inet6_address`；
- 通过订阅或模板生成 sing-box 配置后，需要确认 TUN 字段是否符合当前版本；
- 使用云小狐提供的 sing-box 订阅或本地配置时，客户端提示配置解析失败，但其他出站信息看起来正常。

本文只讨论 TUN 地址字段迁移，不涉及 TUN 与 DNS 劫持、线路测速、节点质量或客户端兼容性测试。

## 版本与字段变化

sing-box 在 1.10.0 中将原本分开的 IPv4、IPv6 TUN 地址字段合并为统一的 `address` 字段；旧的路由地址字段也分别合并为 `route_address` 和 `route_exclude_address`。官方迁移文档明确列出了这三组变化。([sing-box.sagernet.org](https://sing-box.sagernet.org/migration/))

官方 TUN 配置文档还标注，`inet4_address` 和 `inet6_address` 在 1.10.0 中已废弃，并计划在 1.12.0 移除；`inet4_route_address`、`inet6_route_address`、`inet4_route_exclude_address` 和 `inet6_route_exclude_address` 也遵循相同迁移方向。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/inbound/tun/))

| 旧字段 | 新字段 | 迁移方式 |
| --- | --- | --- |
| `inet4_address` | `address` | 将 IPv4 CIDR 放入 `address` 数组 |
| `inet6_address` | `address` | 将 IPv6 CIDR 放入 `address` 数组 |
| `inet4_route_address` | `route_address` | 将 IPv4 路由放入 `route_address` 数组 |
| `inet6_route_address` | `route_address` | 将 IPv6 路由放入 `route_address` 数组 |
| `inet4_route_exclude_address` | `route_exclude_address` | 将 IPv4 排除路由放入 `route_exclude_address` 数组 |
| `inet6_route_exclude_address` | `route_exclude_address` | 将 IPv6 排除路由放入 `route_exclude_address` 数组 |

## 分步骤迁移

### 第一步：备份当前配置

不要直接覆盖原配置。先复制一份备份，并在副本上修改：

```bash
cp config.json config.json.bak
```

如果配置文件不叫 `config.json`，请替换为实际文件名。使用图形客户端时，可以先导出配置或复制配置内容到单独的文本文件中。

### 第二步：确认 sing-box 版本

先查看当前核心版本：

```bash
sing-box version
```

重点不是只看客户端名称，而是看实际运行的 sing-box 核心版本。不同客户端可能内置不同版本的核心，同一份配置在旧核心中可能还能被接受，在新核心中则可能因为字段已经移除而直接检查失败。

如果当前核心版本低于 1.10.0，统一字段 `address`、`route_address` 和 `route_exclude_address` 可能不适用全部旧环境；如果版本为 1.12.0 或更高，不能继续依赖已经移除的旧 TUN 地址字段。本文示例以 sing-box 1.12.0 及更高版本为目标。

### 第三步：迁移 TUN 地址字段

旧配置可能类似下面这样：

```json
{
  "type": "tun",
  "tag": "tun-in",
  "inet4_address": [
    "172.18.0.1/30"
  ],
  "inet6_address": [
    "fdfe:dcba:9876::1/126"
  ],
  "auto_route": true
}
```

迁移后，将两个地址数组合并成一个 `address` 数组：

```json
{
  "type": "tun",
  "tag": "tun-in",
  "address": [
    "172.18.0.1/30",
    "fdfe:dcba:9876::1/126"
  ],
  "auto_route": true
}
```

这里的关键是保留原有 CIDR 值，而不是把 IPv4 和 IPv6 地址改成普通 IP。`address` 表示 TUN 接口使用的 IPv4 和 IPv6 前缀，因此仍应保留 `/30`、`/126` 等前缀长度。

### 第四步：迁移自定义路由字段

如果旧配置中还有以下字段：

```json
{
  "inet4_route_address": [
    "0.0.0.0/1",
    "128.0.0.0/1"
  ],
  "inet6_route_address": [
    "::/1",
    "8000::/1"
  ],
  "inet4_route_exclude_address": [
    "192.168.0.0/16"
  ],
  "inet6_route_exclude_address": [
    "fc00::/7"
  ]
}
```

应改成：

```json
{
  "route_address": [
    "0.0.0.0/1",
    "128.0.0.0/1",
    "::/1",
    "8000::/1"
  ],
  "route_exclude_address": [
    "192.168.0.0/16",
    "fc00::/7"
  ]
}
```

`route_address` 和 `route_exclude_address` 都是数组，可以同时放入 IPv4 与 IPv6 CIDR。迁移时不要改变原有网段含义，也不要为了“简化”而删除 IPv6 项；是否使用 IPv6 应根据原配置和实际网络环境单独判断。

### 第五步：合并回完整配置

下面是一个只用于说明字段位置的完整 JSON 示例，包含一个 TUN 入站和一个直连出站。它不包含 DNS、代理出站或复杂路由规则，适合先执行语法与字段检查：

```json
{
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "address": [
        "172.18.0.1/30",
        "fdfe:dcba:9876::1/126"
      ],
      "auto_route": true,
      "route_address": [
        "0.0.0.0/1",
        "128.0.0.0/1",
        "::/1",
        "8000::/1"
      ],
      "route_exclude_address": [
        "192.168.0.0/16",
        "fc00::/7"
      ]
    }
  ],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    }
  ]
}
```

这段示例中的 `route_address` 不是所有配置都必须填写。如果原配置没有使用自定义路由，可以删除整个 `route_address` 和 `route_exclude_address` 字段，只保留 `address`。迁移的原则是“旧字段对应到新字段”，不是无条件增加更多 TUN 选项。

### 第六步：执行配置检查

保存后，在配置文件所在目录执行：

```bash
sing-box check -c config.json
```

sing-box 官方配置文档将 `sing-box check` 列为配置检查命令，同时也提供了 `format` 和 `merge` 等配置处理命令。([sing-box.sagernet.org](https://sing-box.sagernet.org/zh/configuration/)) 如果你的发行版或客户端对命令行参数做了封装，请以该客户端显示的配置检查入口为准。

检查时可以按以下顺序处理结果：

1. 如果提示 JSON 语法错误，先检查逗号、括号和字符串引号；
2. 如果提示 `inet4_address`、`inet6_address` 或旧路由字段未知，继续搜索配置中是否还有残留；
3. 如果提示字段类型错误，确认 `address`、`route_address` 和 `route_exclude_address` 使用的是 JSON 数组；
4. 如果字段检查通过但启动失败，再单独排查权限、TUN 接口创建、系统路由或客户端自身限制。

## 常见问题

### 1. 只把 `inet4_address` 改成 `address`，还需要保留 `inet6_address` 吗？

不需要保留旧字段。应将 IPv4 和 IPv6 地址合并到同一个 `address` 数组中。例如：

```json
"address": [
  "172.18.0.1/30",
  "fdfe:dcba:9876::1/126"
]
```

如果原配置只有 IPv4 地址，就只迁移原有 IPv4 项，不必为了迁移而凭空添加 IPv6 地址。

### 2. `route_address` 和 `address` 是同一个字段吗？

不是。`address` 用于设置 TUN 接口本身的地址前缀；`route_address` 用于在启用 `auto_route` 时指定自定义路由。官方文档将它们列为不同字段，迁移时不能把所有 CIDR 都放进 `address`。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/inbound/tun/))

### 3. 没有自定义路由时，是否必须填写 `route_address`？

不必须。如果旧配置没有 `inet4_route_address`、`inet6_route_address` 或对应排除字段，可以不添加新的路由字段。保留最小配置通常更容易判断问题来自字段迁移，还是来自其他 TUN、DNS 或路由设置。

### 4. 配置检查通过，但启动 TUN 仍然失败，说明迁移错误吗？

不一定。`check` 主要用于发现配置结构、字段和格式问题；TUN 启动还可能受到操作系统权限、虚拟网卡创建权限、系统路由冲突、其他 VPN 软件或客户端实现差异影响。此时应查看启动日志，并确认日志中的错误是否仍然指向旧字段，而不是继续修改已经通过检查的 `address` 配置。

### 5. 为什么旧教程还能看到 `inet4_address`？

很多旧教程是按 sing-box 1.10.0 之前或迁移过渡期的配置编写的。官方文档已经说明这些字段在 1.10.0 废弃，并在 1.12.0 移除，因此复制旧示例时需要先对照当前版本文档。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/inbound/tun/))

### 6. 可以同时保留新旧字段以兼容多个版本吗？

不建议。新版本可能直接拒绝已经移除的旧字段，而旧版本对新字段的支持范围也可能不同。更稳妥的做法是先确认实际运行的核心版本，再为该版本维护一份明确的配置；如果需要兼容多个核心，建议分别保存经过各自版本检查的配置文件。

## 参考资料

- `https://sing-box.sagernet.org/configuration/inbound/tun/%60
- `https://sing-box.sagernet.org/migration/%60
- `https://sing-box.sagernet.org/deprecated/%60
- `https://sing-box.sagernet.org/zh/configuration/%60
- `https://github.com/SagerNet/sing-box/releases%60

使用相关软件和网络服务时，请遵守当地法律法规，并以软件官方文档和实际版本说明为准。

## 关于 云小狐

云小狐提供网络加速服务与多平台使用教程，帮助用户了解客户端安装、订阅导入、代理模式和常见连接问题。更多平台介绍与使用说明请访问[云小狐官网](https://www.yunxiaohu.com/)。