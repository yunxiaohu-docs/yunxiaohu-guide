> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-11-08。

# Mihomo include-all 怎么写：将订阅节点加入策略组

## 一句话结论

如果你的 Mihomo 配置已经加载了订阅节点或其他出站代理，只需在目标 `proxy-groups` 项中加入 `include-all: true`，即可让该策略组自动包含所有出站代理和代理集合；它不会自动包含其他策略组。

## 适用场景

本文只介绍一个配置字段：`proxy-groups` 下的 `include-all`。它适合以下情况：

- 订阅节点会定期更新，手动维护 `proxies` 节点列表比较麻烦；
- 使用 `proxy-providers` 加载订阅后，希望策略组自动看到其中的节点；
- 配置中同时存在手写出站代理和订阅代理，希望把它们放进同一个选择组；
- 导入订阅后，客户端界面只显示 `DIRECT`、`REJECT` 或少量固定项，没有显示订阅节点；
- 希望先用一个最小策略组验证节点是否已经被核心加载。

Mihomo 官方文档将 `include-all` 定义为：引入所有出站代理以及代理集合，并按照名称排序；该字段不会包含策略组，其他策略组仍需要单独写入 `proxies`。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/proxy-groups/))

## 分步骤配置

### 第一步：确认配置中已经有节点来源

`include-all` 只负责把已经存在的出站代理和代理集合加入策略组，它不会创建节点，也不会替代订阅配置。Mihomo 的代理集合通常在 `proxy-providers` 中定义，官方示例使用 `http`、`file` 或 `inline` 类型，并由代理集合保存或提供节点内容。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/proxy-providers/))

因此，先确认当前配置至少满足以下一种情况：

1. 配置中存在手写的 `proxies` 出站代理；
2. 配置中存在已经成功加载的 `proxy-providers`；
3. 客户端日志或控制面板能够看到订阅更新成功。

如果订阅本身没有下载成功，单独添加 `include-all` 也不会产生节点。

### 第二步：在目标策略组添加 include-all

在现有配置的 `proxy-groups` 中添加一个最小选择组。下面的写法不依赖具体订阅名称，只要配置中已经存在可用出站代理或代理集合，就可以作为验证用配置：

```yaml
proxy-groups:
  - name: "全部节点"
    type: select
    include-all: true
    proxies:
      - DIRECT
```

这里的关键字段是：

```yaml
include-all: true
```

`type: select` 表示由用户手动选择组内项目；`DIRECT` 是额外加入的直连选项，便于在代理和直连之间切换。`include-all` 会把当前配置中的出站代理以及代理集合一起纳入该组，并按名称排序。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/proxy-groups/))

如果你使用云小狐的订阅链接，请先按照客户端的订阅导入流程完成更新，再把上述字段加入自己的配置或覆写配置中；本文不要求改动订阅内容本身。

### 第三步：把该策略组接入实际流量规则

仅定义策略组并不代表所有流量都会自动使用它。现有配置通常还需要让规则或最终匹配项指向这个策略组。一个最小的规则示例如下：

```yaml
rules:
  - MATCH,全部节点
```

这条规则表示：前面的规则都没有匹配时，将剩余流量交给名为 `全部节点` 的策略组。若你的配置已经有完整规则，只需要把对应的目标策略组名称改成实际使用的组名，不要重复添加多个最终匹配规则。

### 第四步：保存并检查策略组内容

保存配置后，在客户端重新载入配置，观察 `全部节点` 是否出现订阅节点。正常情况下，组内应至少能看到已经加载的出站代理、代理集合以及手动写入的 `DIRECT`。

如果客户端提供配置检查功能，建议先执行检查，再重新载入。例如，部分 Mihomo 命令行发行版支持使用 `-t` 对配置进行测试，但不同客户端封装的启动参数可能不同，具体命令应以该客户端帮助信息或启动说明为准。不要把某个 GUI 客户端的启动参数直接套用到所有版本。

## 最小验证思路

为了避免复杂规则、DNS、TUN 或多个策略组同时干扰，可以暂时只保留一个验证组：

```yaml
proxy-groups:
  - name: "检查节点"
    type: select
    include-all: true
    proxies:
      - DIRECT
```

验证时只观察三件事：

1. 配置是否能够成功载入；
2. `检查节点` 是否出现在客户端的策略组列表中；
3. 组内是否出现已经加载的出站代理或代理集合。

如果策略组存在但组内没有订阅节点，问题通常不在 `include-all` 的缩进，而在代理集合没有成功加载、订阅内容为空、客户端没有使用当前配置，或者客户端前端没有把相关项目完整展示出来。官方文档明确说明，`include-all` 的范围是出站代理和代理集合，不包含策略组。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/proxy-groups/))

## 常见问题

### 1. `include-all: true` 写了，但策略组还是空的，怎么办？

先确认订阅或代理集合本身已经成功加载。`include-all` 不负责下载订阅，也不会修复代理集合的 URL、路径、权限或网络错误。可以先在客户端的代理列表、配置日志或代理集合页面确认节点是否存在，再检查当前载入的是否是刚刚修改的配置。

### 2. 为什么 include-all 没有把其他策略组加入进来？

这是字段设计范围。官方文档说明，`include-all` 不包含策略组；它只引入出站代理和代理集合。若需要让一个总选择组同时提供“自动选择”“香港分组”等策略组，应在该总组的 `proxies` 中明确写出这些策略组名称，例如：

```yaml
proxy-groups:
  - name: "总入口"
    type: select
    include-all: true
    proxies:
      - "自动选择"
      - "香港节点"
      - DIRECT
```

前提是这些名称确实已经在 `proxy-groups` 中定义，并且名称完全一致。

### 3. include-all 会不会重复添加节点？

它会按照配置中可见的出站代理和代理集合建立策略组候选项。若你同时在 `proxies` 中手动写入某个已经由其他机制引入的名称，界面表现可能取决于客户端前端和核心处理方式。为了便于排查，建议先使用最小配置验证，确认节点来源后再逐步加入其他策略组和固定选项。

### 4. 为什么节点顺序和订阅文件里的顺序不一样？

官方说明中，`include-all` 引入的出站代理和代理集合会按照名称排序，因此不要把它当作保留订阅原始顺序的功能。如果需要固定的展示顺序，应通过明确的策略组结构维护，而不是依赖自动收集后的顺序。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/proxy-groups/))

### 5. include-all 能不能代替 proxy-providers？

不能。`include-all` 是策略组字段，作用是收集已经存在的出站代理和代理集合；`proxy-providers` 则负责定义代理集合的来源、更新周期、保存路径和健康检查等内容。两者解决的问题不同，不能互相替代。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/proxy-providers/))

### 6. 配置能载入，但客户端界面没有显示所有节点，是否一定是核心错误？

不一定。核心是否已经加载节点、策略组是否包含节点、客户端前端如何展示，是三个不同环节。MetaCubeX 的历史问题记录中也出现过“代理集合节点没有按预期显示在全局列表”的讨论，因此遇到此类情况，应先检查策略组实际内容，再判断是核心配置还是客户端界面行为。([github.com](https://github.com/MetaCubeX/mihomo/issues/787))

## 参考资料

1. https://wiki.metacubex.one/en/config/proxy-groups/
2. https://wiki.metacubex.one/en/config/proxy-providers/
3. https://github.com/MetaCubeX/Meta-Docs/blob/main/docs/config/proxy-groups/index.md
4. https://github.com/MetaCubeX/mihomo/wiki/Configuring-example/bea854d2221517d86d9a45c3aa22ab66d28a6ebf
5. https://github.com/MetaCubeX/mihomo/issues/787

## 关于 云小狐

云小狐提供网络加速服务使用说明、客户端配置教程和常见问题整理，帮助用户理解订阅导入、代理模式与配置排查流程。更多平台信息与使用入口请访问：https://www.yunxiaohu.com/%E3%80%82