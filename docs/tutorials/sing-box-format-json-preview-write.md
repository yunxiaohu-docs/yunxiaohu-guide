> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-11-21。

# sing-box format 怎么用：先预览再写回 JSON 配置

一句话结论：整理配置时，先执行不带 `-w` 的 `sing-box format -c config.json` 预览输出，核对差异并备份后，再用 `-w` 写回；不要把格式化理解成只调整缩进。核验的官方源码明确区分了标准输出与写回源文件两条路径。([raw.githubusercontent.com](https://raw.githubusercontent.com/SagerNet/sing-box/v1.14.4/cmd/sing-box/cmd_format.go))

## 适用场景

本文适合已经安装 sing-box 命令行程序、准备整理本地 JSON 配置的用户，重点解决一个问题：怎样先看格式化结果，再决定是否覆盖原文件。

你可以在以下情况下采用本文流程：

- 手动编辑配置后，想先审阅整理结果。
- 准备把配置保存到自己的 Git 仓库，希望控制本次提交的变化。
- 看到其他教程直接使用 `format -w`，但还不清楚写回参数的影响。

本文只讲 `sing-box format`，不扩展到订阅转换、DNS、TUN 或连接测试。

**核验范围：**本次实际打开了官方中文配置文档、官方 Releases 页面、`v1.14.4` 的 `cmd_format.go`，以及官方仓库 issue #4519。发布列表中可以找到 `1.14.4`；下文的命令行为以该标签源码为依据，不使用开发分支实现，也不声称已核验其他版本的格式化源码。其他版本是否一致，需核实对应标签。([sing-box.sagernet.org](https://sing-box.sagernet.org/zh/configuration/))

## 分步骤操作

### 第一步：准备独立副本

建议先通过文件管理器复制配置，把练习文件放进独立目录。不要一开始就对正在使用的唯一原件操作，也不要让备份与工作副本使用同一路径。

如果你正在整理与云小狐相关、已经保存到本地的配置资料，也请先确认文件确实是适用于 sing-box 的 JSON 配置；本文不预设服务提供何种文件格式。

本文使用 `config.json` 作为练习文件名。它不是要求所有设备都采用这个名称，而是要求下面的命令与当前目录中的实际文件保持一致。官方配置文档说明 sing-box 使用 JSON，并在格式化示例中使用了 `-c config.json`。([sing-box.sagernet.org](https://sing-box.sagernet.org/zh/configuration/))

第一次练习可以在独立目录中新建 `config.json`，内容仅为：

```json
{}
```

这个空对象仅用于观察文件处理流程，不是提供代理服务的完整配置。建议先用它熟悉操作，再换成自己的配置副本；不要为了练习加入来源不明的凭证或规则。

### 第二步：先预览，不写回

在能够直接调用 `sing-box` 的终端中，进入练习文件所在目录，执行：

```sh
sing-box format -c config.json
```

关键是先不添加 `-w`。在核验的实现中，写回开关默认关闭；关闭时，程序把整理结果写到标准输出，而不是覆盖源文件。因此，终端出现 JSON 输出并不意味着原文件已经改变。([raw.githubusercontent.com](https://raw.githubusercontent.com/SagerNet/sing-box/v1.14.4/cmd/sing-box/cmd_format.go))

建议保留编辑器中的原文件，将终端结果与它对照阅读。此时不必急着追求“一条命令完成全部操作”：预览阶段的目的，就是给自己留下拒绝变化的机会。

本文没有在你的设备上运行命令，不提供本地执行成功、耗时或截图结论。如果出现报错，应先保留完整提示，不要立即改用带写回参数的命令重试。

### 第三步：核对差异，不只看缩进

核验源码在编码前调用了 `badjson.Omitempty`，随后设置两空格缩进输出。因此，不能把这段实现描述成对原始文本机械地增加空格和换行。([raw.githubusercontent.com](https://raw.githubusercontent.com/SagerNet/sing-box/v1.14.4/cmd/sing-box/cmd_format.go))

建议依次检查：

1. 是否出现自己无法解释的内容增删。
2. 原有列表的顺序是否仍符合预期。
3. 注释是否仍对应原来说明的配置内容。
4. 是否误选了另一份同名配置。

官方仓库 issue #4519 中，报告者针对 `1.14.0` 提供了格式化后字段增删、规则顺序变化的输入与输出。这属于用户报告，不能直接视为维护者确认的通用结论，也不能据此断言 `1.14.4` 必然存在同样问题。这里引用它，只是提醒读者审阅实际差异，不承诺格式化永远只改变外观。([github.com](https://github.com/SagerNet/sing-box/issues/4519))

如果发现不能解释的变化，建议停在预览阶段。记录所用版本、输入文件与输出结果，再查对应版本资料；不要因为终端输出了完整 JSON，就直接接受所有变化。

### 第四步：确认备份后再写回

只有在差异符合预期、原始副本已经保存后，再执行：

```sh
sing-box format -w -c config.json
```

`-w` 是 `--write` 的短形式。核验源码将其定义为把结果写到源文件，而不是标准输出；需要更新内容时，会打开源路径写入结果。这不是“另存为”参数，该写回流程也没有自动创建备份的步骤。([raw.githubusercontent.com](https://raw.githubusercontent.com/SagerNet/sing-box/v1.14.4/cmd/sing-box/cmd_format.go))

建议执行前再确认一次：当前目录正确、目标文件正确、备份可读取。如果这三个条件有任何一个不确定，就先不要写回。

### 第五步：重新打开并决定是否保留

建议写回后重新打开文件，按照第三步的清单再审阅一次。使用版本管理时，先检查差异，再决定是否提交；不用版本管理时，则保留整理前副本，直到确认无需回退。

本次操作的完成标准应是“我知道目标文件发生了什么变化，并决定保留它”，而不是“命令运行过了”。核验的 `format` 实现负责读取、处理、编码和输出配置，没有执行代理连接测试，不能据此给出节点可用或网络连通的结论。([raw.githubusercontent.com](https://raw.githubusercontent.com/SagerNet/sing-box/v1.14.4/cmd/sing-box/cmd_format.go))

## 常见问题

### 1. 为什么终端出现了 JSON，原文件却没变？

先检查是否省略了 `-w`。不带写回参数时，结果进入标准输出，属于核验实现的预期行为；如果你的目的只是预览，无需补上参数。([raw.githubusercontent.com](https://raw.githubusercontent.com/SagerNet/sing-box/v1.14.4/cmd/sing-box/cmd_format.go))

### 2. 能写成 `sing-box format config.json` 吗？

不要采用这种位置参数写法。`v1.14.4` 的 `cmd_format.go` 为该命令设置了 `Args: cobra.NoArgs`，这是“不接受位置参数”的直接源码依据。应使用本文的 `sing-box format -c config.json`。([raw.githubusercontent.com](https://raw.githubusercontent.com/SagerNet/sing-box/v1.14.4/cmd/sing-box/cmd_format.go))

官方配置文档中的示例则用于佐证 `-c config.json` 的写法，不用于证明 `cobra.NoArgs` 这一源码结论。([sing-box.sagernet.org](https://sing-box.sagernet.org/zh/configuration/))

### 3. 格式化成功，是否说明配置一定能正常联网？

不能。核验的格式化命令没有执行节点连接、目标网站访问或测速。请把“文件处理完成”与“实际运行符合预期”分开判断。([raw.githubusercontent.com](https://raw.githubusercontent.com/SagerNet/sing-box/v1.14.4/cmd/sing-box/cmd_format.go))

### 4. 发现内容增删，是否应该直接接受？

建议先不要写回。issue #4519 提供的是特定版本的用户报告，既不能证明你看到的变化都正确，也不能证明它们都有问题。应以自己的输入、输出和对应版本资料逐项核对，而不是统一当成正常优化。([github.com](https://github.com/SagerNet/sing-box/issues/4519))

### 5. 已经写回，能靠这个命令撤销吗？

不要依赖它恢复原始文本。核验的写回流程没有创建备份或提供撤销入口。建议从操作前保存的副本恢复，或使用你已有的版本管理记录回退。([raw.githubusercontent.com](https://raw.githubusercontent.com/SagerNet/sing-box/v1.14.4/cmd/sing-box/cmd_format.go))

## 参考资料

- [sing-box 官方中文配置文档](https://sing-box.sagernet.org/zh/configuration/)
- [sing-box 官方发布列表](https://github.com/SagerNet/sing-box/releases)
- [v1.14.4 格式化命令源码](https://raw.githubusercontent.com/SagerNet/sing-box/v1.14.4/cmd/sing-box/cmd_format.go)
- [官方仓库 issue #4519：格式化结果变化的用户报告](https://github.com/SagerNet/sing-box/issues/4519)

## 关于 云小狐

本站文档仓库整理客户端使用教程、配置操作说明与常见问题，帮助读者理解操作步骤及其边界。平台信息请访问[官网](https://www.yunxiaohu.com/)，使用相关工具与服务时请遵守当地法律法规。