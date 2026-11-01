> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-11-01。

# Mihomo geodata-mode 配置迁移：mmdb 与 dat 怎么选

## 一句话结论

迁移 Mihomo 配置时，不要把 `geodata-mode` 和 `geodata-loader` 当成同一个字段：`geodata-mode` 决定使用 `mmdb` 还是 `dat` 格式的 GEO 数据，`geodata-loader` 决定加载方式；如果只是从旧配置迁移，优先保留原有数据格式选择，再单独调整加载器。

## 适用场景

本文适合以下情况：

- 从旧版 Clash Meta 或 Mihomo 配置迁移到当前 Mihomo；
- 配置中出现 `geodata-mode`，但不确定 `true` 和 `false` 分别代表什么；
- 想降低小内存设备加载 GEO 数据时的占用；
- 修改配置后出现规则匹配异常、启动失败或 GEO 数据加载错误；
- 希望在不改动代理节点、代理组和规则顺序的前提下，只迁移 GEO 相关字段。

本文只讨论 GEO 数据字段，不涉及 TUN、DNS 劫持、节点测速或线路质量判断。Mihomo 文档目前将 `geodata-mode` 定义为 GEO 数据格式选择项：`true` 使用 `dat`，`false` 使用 `mmdb`；`geodata-loader` 则提供 `standard` 和 `memconservative` 两种加载模式，默认加载模式为 `memconservative`。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/general/?h=un))

## 一、先备份并确认当前字段

在编辑前，先复制一份当前配置文件，例如：

```bash
cp config.yaml config.yaml.bak
```

Windows 用户可以直接在文件管理器中复制配置文件并改名为 `config.yaml.bak`。备份的目的不是保留一份“旧版本配置”这么简单，而是方便区分字段迁移和其他问题：如果修改后启动失败，可以立即恢复备份，确认故障是否由本次改动引起。

然后在配置中搜索以下字段：

```yaml
geodata-mode:
geodata-loader:
geo-auto-update:
geo-update-interval:
```

常见旧配置可能只有：

```yaml
geodata-mode: true
```

也可能同时包含：

```yaml
geodata-mode: false
geodata-loader: memconservative
```

先记录原值，不要在同一次修改中顺便改动 `dns`、`tun`、`rules`、`proxy-groups` 或代理节点。一次只改一个配置主题，后续排查会更清楚。

## 二、理解 geodata-mode 的 true 和 false

`geodata-mode` 是布尔值，不是字符串，也不是文件路径。它的含义如下：

| 配置值 | 使用的数据格式 | 迁移含义 |
|---|---|---|
| `true` | `dat` | 保留或切换到 dat 格式 GEO 数据 |
| `false` | `mmdb` | 保留或切换到 mmdb 格式 GEO 数据 |
| 未填写 | 使用 Mihomo 默认行为 | 不建议在不清楚默认值时依赖省略字段 |

例如，原配置使用 `dat`：

```yaml
geodata-mode: true
```

如果你希望继续使用相同格式，迁移时通常不需要把它改成 `false`。如果你明确要改用 `mmdb`，才改为：

```yaml
geodata-mode: false
```

这里的关键是：`true` 不表示“开启 GEO”，`false` 也不表示“关闭 GEO”。它们表示两种不同的 GEO 数据格式。把 `geodata-mode: true` 误改成 `false`，并不会自动修复所有规则问题，反而可能让原来可用的 GEO 数据文件与配置选择不匹配。

## 三、单独选择 geodata-loader

`geodata-loader` 是加载模式，和数据格式是两个维度。当前文档列出的可选值为：

```yaml
geodata-loader: standard
```

或：

```yaml
geodata-loader: memconservative
```

其中，`standard` 是标准加载器，`memconservative` 是面向内存受限设备的加载器。若设备内存较小，或者启动时希望优先控制内存占用，可以使用：

```yaml
geodata-mode: false
geodata-loader: memconservative
```

如果设备资源充足，并且你需要按照标准方式加载 GEO 数据，可以使用：

```yaml
geodata-mode: false
geodata-loader: standard
```

这两个示例都选择 `mmdb`，区别只在加载器。下面这个示例则选择 `dat`，同时使用内存更保守的加载方式：

```yaml
geodata-mode: true
geodata-loader: memconservative
```

因此，不要进行下面这种“字段替换”：

```yaml
# 不要把 geodata-mode 当成 geodata-loader 使用
geodata-mode: memconservative
```

`geodata-mode` 只能写布尔值 `true` 或 `false`；`geodata-loader` 才能写 `standard` 或 `memconservative`。YAML 对类型比较敏感，字段名正确但值类型错误，同样可能导致配置解析失败。

## 四、按最小改动方式迁移

推荐按照以下顺序操作。

### 1. 只保留一组明确字段

如果原配置中字段重复，保留最后确认有效的一组，避免同一个字段在文件中出现多次。例如整理为：

```yaml
geodata-mode: false
geodata-loader: memconservative
```

不要写成：

```yaml
geodata-mode: true
geodata-mode: false
```

YAML 文件中重复键的处理可能取决于解析器，不能把重复字段当成覆盖机制使用。

### 2. 先不要启用自动更新

如果你正在排查启动或规则问题，可以先暂时不加入自动更新字段，先验证基础配置：

```yaml
geodata-mode: false
geodata-loader: memconservative
```

确认客户端能够正常启动后，再根据需要加入：

```yaml
geo-auto-update: true
geo-update-interval: 24
```

`geo-update-interval` 的单位是小时。官方配置说明将 `geo-auto-update` 和更新间隔作为独立配置项列出。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/general/?h=un))

### 3. 使用客户端的配置检查功能

保存配置后，优先使用客户端内置的“检查配置”“载入配置”或“重新加载配置”功能。如果你使用的是 Mihomo 命令行程序，也可以根据所安装版本支持的帮助信息确认配置检查参数：

```bash
mihomo -h
```

不同客户端对内核文件名、配置目录和命令行参数的封装可能不同，因此不要直接复制其他客户端的启动命令。若使用云小狐提供的订阅服务，建议先在客户端内完成订阅更新，再把本地手写字段放入独立配置或覆写配置中，避免直接修改自动生成的订阅文件。

### 4. 启动后只验证一个结果

本次迁移的第一项验证目标应是“配置能否正常加载”，而不是同时测试所有网站、节点和规则。可以按以下顺序观察：

1. 客户端是否能正常启动；
2. 配置页面是否还能显示代理组；
3. 日志中是否出现 YAML 解析错误或 GEO 数据加载错误；
4. 需要时再验证依赖 GEO 数据的规则是否按预期工作。

如果启动失败，先恢复 `config.yaml.bak`，再只改动一个字段。不要在错误配置上继续叠加 DNS、TUN 或规则调整。

## 五、一个可回滚的完整示例

下面是一段只包含相关字段的示例，适合用来对照现有配置：

```yaml
# GEO 数据格式：false 使用 mmdb，true 使用 dat
geodata-mode: false

# 加载模式：standard 或 memconservative
geodata-loader: memconservative

# 确认基础配置无误后，再按需启用自动更新
geo-auto-update: true
geo-update-interval: 24
```

如果修改后出现问题，可以先回滚为最小版本：

```yaml
geodata-mode: false
geodata-loader: memconservative
```

如果最小版本可以启动，再逐项加入 `geo-auto-update` 和 `geo-update-interval`。这样可以判断问题来自格式选择、加载器，还是自动更新设置。

## 常见问题

### 1. `geodata-mode: true` 是不是代表开启 GEO？

不是。它表示选择 `dat` 格式的 GEO 数据；`false` 表示选择 `mmdb` 格式。两者都不是“开启/关闭”开关。是否使用 GEO 相关规则，还要看具体规则类型和配置内容。

### 2. 能不能把 `geodata-mode` 改成 `geodata-loader`？

不能直接替换。前者负责选择 `dat` 或 `mmdb`，后者负责选择加载方式。正确写法是分别配置，例如：

```yaml
geodata-mode: true
geodata-loader: standard
```

### 3. `geodata-loader` 写成 `memory` 或 `low-memory` 可以吗？

不建议。当前文档列出的加载模式是 `standard` 和 `memconservative`。没有资料证明其他名称是有效值，应该使用文档中的字段值，或先以当前客户端实际支持的配置说明为准。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/general/?h=un))

### 4. 改成 `mmdb` 后规则仍然不命中，说明迁移失败吗？

不一定。规则不命中还可能与规则集文件、规则顺序、目标域名、客户端缓存或配置没有真正重新加载有关。建议先确认配置已经成功载入，再单独验证一条规则，不要把 GEO 字段迁移和全部规则排查混在一起。

### 5. 是否应该直接下载最新 Alpha 版本来解决字段问题？

不建议仅因为配置字段迁移就使用预发布版本。官方仓库将稳定发布和 `Prerelease-Alpha` 分开列出；如果没有明确的功能需求，优先使用客户端支持的稳定版本，并以其实际文档为准。([github.com](https://github.com/MetaCubeX/mihomo/releases?ref=zerolab.net))

### 6. 可以同时写多个 `geodata-mode` 让客户端自动选择吗？

不可以。配置文件中同一个键应只保留一处，并明确写出 `true` 或 `false`。如果希望测试两种格式，应分别保存两份配置，逐份检查，而不是在同一个 YAML 文件中重复定义字段。

## 参考资料

- [Mihomo General configuration](https://wiki.metacubex.one/en/config/general/)
- [Mihomo Quick configuration](https://wiki.metacubex.one/en/example/conf/)
- [MetaCubeX/mihomo Releases](https://github.com/MetaCubeX/mihomo/releases)

## 关于 云小狐

提供客户端使用教程、订阅导入说明与常见连接问题排查建议，帮助用户按步骤完成配置检查。官网：https://www.yunxiaohu.com/