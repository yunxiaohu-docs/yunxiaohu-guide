> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-11-14。

# Mihomo unified-delay 怎么写：统一延迟配置与异常排查

## 一句话结论

如果你希望 Mihomo 在不同代理类型之间用更一致的方式计算延迟，可以在配置文件顶层加入 `unified-delay: true`；它只接受布尔值 `true` 或 `false`，修改后应先通过 `mihomo -t` 配置检查，再重启内核。官方文档对该字段的说明是：开启后会计算 RTT，以减少连接握手等差异造成的延迟偏差。([wiki.metacubex.one](https://wiki.metacubex.one/config/general/))

## 适用场景

本文适合以下情况：

- 你在使用 Mihomo、Clash Meta 兼容内核或基于 Mihomo 的客户端，需要确认 `unified-delay` 的写法。
- 你看到配置模板里有 `unified-delay`，但不确定它应该放在哪里、能填写什么值。
- 你开启该字段后，策略组健康检查显示节点不可用、延迟为异常值，想先判断是字段问题还是健康检查目标、代理类型导致的问题。
- 你从旧配置迁移到新客户端，只想验证一个字段，不希望同时修改 DNS、TUN、规则集或代理组。

本文只处理 `unified-delay`，不会把其他配置项混在同一个排查流程中。

## 分步骤配置

### 第 1 步：确认配置文件是 Mihomo YAML

找到客户端实际使用的主配置文件。不同客户端的文件位置可能不同，常见名称是 `config.yaml` 或由客户端生成的 YAML 配置。不要直接修改订阅服务器上的内容，也不要把真实订阅链接、令牌或密码粘贴到公开日志中。

如果客户端支持编辑当前配置，建议先复制一份备份。例如在 Linux 或 macOS 终端中，可以执行：

```bash
cp config.yaml config.yaml.backup
```

Windows 用户可以在文件管理器中复制文件并重命名为 `config.yaml.backup`。备份的目的只是方便恢复，不代表必须改变其他配置。

### 第 2 步：在顶层加入字段

`unified-delay` 是全局配置字段，应与 `mode`、`log-level`、`ipv6` 等顶层字段处于同一缩进层级。最小写法如下：

```yaml
unified-delay: true
```

完整片段可以这样写：

```yaml
mixed-port: 7890
mode: rule
log-level: info
unified-delay: true

proxies: []
proxy-groups:
  - name: PROXY
    type: select
    proxies:
      - DIRECT
rules:
  - MATCH,PROXY
```

上面的示例只展示 YAML 层级和字段位置。实际使用时，`proxies`、`proxy-groups` 和 `rules` 应以客户端生成的有效配置为准，不要为了照抄示例而删除原有订阅、规则或策略组。

官方 Mihomo 文档明确列出该字段的可选值为 `true/false`，因此以下写法是有效的：

```yaml
unified-delay: true
```

或：

```yaml
unified-delay: false
```

以下写法不要使用：

```yaml
unified-delay: "开启"
unified-delay: 1
unified-delay: yes
```

原因是这些值并不是官方文档列出的标准布尔写法；即使某些 YAML 解析器能够读取，也不应据此判断 Mihomo 一定会按预期处理。([wiki.metacubex.one](https://wiki.metacubex.one/config/general/))

### 第 3 步：只保留一个同名字段

检查配置文件中是否已经存在 `unified-delay`。如果文件里同时出现多次，例如：

```yaml
unified-delay: false
unified-delay: true
```

不要依赖后出现的值，也不要用重复字段测试客户端行为。直接删除旧行，只保留一个明确值：

```yaml
unified-delay: true
```

这样更容易确认当前配置意图，也能避免不同编辑器、合并器或客户端对重复 YAML 键的处理差异。

### 第 4 步：用 Mihomo 检查配置

保存文件后，在 Mihomo 可执行文件所在环境中运行：

```bash
mihomo -t -f ./config.yaml
```

其中 `-t` 用于测试配置，`-f` 后面跟主配置文件路径。Mihomo 官方面板项目的配置激活流程也会调用 `mihomo -t`，只有检查通过后才继续重启内核。([github.com](https://github.com/MetaCubeX/metacubexd))

如果配置文件位于其他目录，可以写绝对路径：

```bash
mihomo -t -f /path/to/config.yaml
```

Windows PowerShell 示例：

```powershell
.\mihomo.exe -t -f .\config.yaml
```

这里的检查只能说明配置能够被内核解析，不能证明每个节点都可连接，也不能证明健康检查目标一定返回预期结果。配置检查通过后，再回到客户端重载配置或重启 Mihomo。

### 第 5 步：确认修改是否生效

重载后，查看客户端配置编辑器或生成后的配置内容，确认实际生效文件仍包含：

```yaml
unified-delay: true
```

如果客户端每次更新订阅都会重新生成主配置，手动添加的字段可能被覆盖。此时应使用客户端提供的配置覆写、脚本或持久化配置入口；具体入口取决于客户端本身，不能把某一个桌面客户端的操作路径当成所有 Mihomo 客户端的通用规则。使用云小狐订阅时，也应优先在所用客户端的持久化配置位置修改，而不是直接改动订阅原文。

## 常见问题

### 1. `unified-delay` 应该放在 `proxy-groups` 里面吗？

不应该。它是 Mihomo 的全局配置字段，应放在 YAML 顶层，而不是放进 `proxy-groups`、某个代理节点或 `proxy-providers` 条目中。正确示例是：

```yaml
unified-delay: true
proxy-groups:
  - name: PROXY
    type: select
    proxies:
      - DIRECT
```

如果缩进到策略组内部，字段就不再位于官方文档描述的顶层位置，可能导致配置结构错误或字段不生效。([wiki.metacubex.one](https://wiki.metacubex.one/config/general/))

### 2. 开启后是不是一定会让节点更快？

不是。官方说明的是延迟计算方式：Mihomo 会计算 RTT，以减少不同代理类型因连接握手等差异产生的延迟偏差。它不是带宽增强开关，也不是节点质量提升开关，因此不能据此承诺更高速度、更低实际延迟或更稳定的连接。([wiki.metacubex.one](https://wiki.metacubex.one/config/general/))

### 3. 开启后策略组显示 `alive: false` 或 `delay: 65535`，怎么办？

先不要立即判断是配置语法错误。Mihomo 官方仓库曾记录过与 `unified-delay`、特定 ShadowsocksR 兼容配置和健康检查有关的问题：相关案例中，日志出现 `alive: false` 与 `delay: 65535`；后续讨论指出，问题与特定代理组合及健康检查请求有关，并曾通过改用 HTTPS 健康检查地址解决。该案例不能直接推导出所有节点都存在同一问题，但可以作为排查方向。([github.com](https://github.com/MetaCubeX/mihomo/issues/1026))

建议按以下顺序处理：

1. 先运行 `mihomo -t -f ./config.yaml`，确认配置本身能通过检查。
2. 暂时将 `unified-delay` 改为 `false`，重载配置，观察异常是否消失。
3. 如果关闭后恢复，再检查出现问题的代理类型、代理提供器和健康检查 URL。
4. 不要同时修改多个字段，否则无法判断真正原因。
5. 如果只有某个节点或某个代理提供器异常，应保留错误日志、内核版本和脱敏后的配置片段，再到 Mihomo 官方仓库检索或提交问题。

### 4. `unified-delay: false` 能不能作为迁移后的临时方案？

可以作为排查步骤使用。官方文档列出的可选值包含 `false`，因此可以先关闭该字段，验证异常是否与统一延迟计算有关；但这只是定位手段，不代表所有异常都应永久关闭。关闭后仍需重新执行配置检查，并确认客户端确实加载了修改后的文件。([wiki.metacubex.one](https://wiki.metacubex.one/config/general/))

### 5. 配置检查通过，但客户端还是不生效，为什么？

最常见的原因不是字段拼写，而是客户端没有读取你修改的文件。例如你编辑的是备份文件、订阅原文，或者客户端更新订阅后重新生成了主配置。可以重新打开客户端当前生效配置，搜索 `unified-delay`，再确认字段是否位于顶层并且只出现一次。

另外，`mihomo -t` 只负责配置验证；官方面板项目的说明也将它作为激活前的最终配置检查，而不是节点连通性测试。配置通过后仍可能存在代理不可用、健康检查地址不可达或特定协议兼容性问题。([github.com](https://github.com/MetaCubeX/metacubexd))

### 6. 我应该根据哪个 Mihomo 版本照抄配置？

本文没有为 `unified-delay` 指定一个未经核实的最低版本号。不同客户端可能内置不同 Mihomo 内核，版本信息应以客户端设置页、启动日志或官方发布页面为准。若某个客户端提示字段未知，先确认它使用的是否确实是 Mihomo 内核，再查看对应版本的官方文档和发布说明。官方发布页会记录各版本的变更内容，但不能用一个不明来源的配置模板替代实际内核版本。([github.com](https://github.com/MetaCubeX/mihomo/releases))

## 参考资料

1. https://wiki.metacubex.one/config/general/
2. https://wiki.metacubex.one/en/config/general/
3. https://github.com/MetaCubeX/mihomo/issues/1026
4. https://github.com/MetaCubeX/mihomo/releases
5. https://github.com/MetaCubeX/metacubexd

## 关于 云小狐

云小狐提供网络加速服务及相关客户端、订阅导入和连接排查教程，帮助用户理解不同客户端与代理配置的使用方式。使用相关服务和工具时，请遵守当地法律法规。官网：https://www.yunxiaohu.com/