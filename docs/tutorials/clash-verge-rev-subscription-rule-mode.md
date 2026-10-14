> 本文由 [云小狐](https://www.yunxiaohu.com/) 整理，更新于 2026-10-14。

# Clash Verge Rev 导入订阅与规则模式设置教程

**一句话结论：**在官方渠道安装 Clash Verge Rev 后，通过 Profiles（配置）页面导入 Clash/Mihomo 订阅链接，完成配置更新、节点选择和系统代理设置，再根据需要使用 Rule（规则）模式即可完成首次连接。

## 适用场景

本文适用于第一次使用 Clash Verge Rev 的 Windows、macOS 和 Linux 用户，也适合遇到“订阅已经添加但无法连接”“更新失败”“规则模式和全局模式不知道怎么选”等问题的用户。Clash Verge Rev 是一个基于 Tauri 的桌面客户端，项目官方仓库说明它内置 mihomo（Clash.Meta）内核，并提供配置管理、系统代理、TUN 虚拟网卡等功能；下载安装包时，应优先使用项目官方 GitHub 仓库及 Releases 页面，而不是来源不明的第三方安装包。([github.com](https://github.com/Clash-Verge-rev/clash-verge-rev))

需要注意的是，Clash Verge Rev 客户端本身不等于订阅服务。客户端负责读取配置、管理节点并转发本机流量，订阅服务负责提供订阅链接和配置内容。使用网络工具时，请遵守当地法律法规，并妥善保护订阅链接中的账号标识或访问令牌。

## 分步骤设置 Clash Verge Rev

### 1. 从官方渠道下载并安装客户端

打开 Clash Verge Rev 的官方 GitHub 仓库，进入 Releases 页面，根据操作系统和处理器架构下载对应文件：

- Windows：根据电脑架构选择 x64 或 ARM64 安装包。
- macOS：Apple 芯片设备选择 Apple Silicon 版本，Intel 设备选择 Intel 版本。
- Linux：根据发行版和架构选择 DEB、RPM 或其他项目提供的安装包。

官方发布页会列出每个版本的安装文件及校验信息。不要把搜索结果中名称相近、但不属于 `clash-verge-rev/clash-verge-rev` 仓库的下载站当作官方来源。安装完成后启动应用；如果系统安全提示要求确认，请核对文件来源和签名后再继续。([github.com](https://github.com/Clash-Verge-rev/clash-verge-rev/releases))

### 2. 准备可用的订阅链接

登录你的订阅服务后台，复制完整的 Clash、Clash Meta 或 Mihomo 订阅 URL。通常链接以 `http://`/ 或 `https://`/ 开头，可能包含问号、参数或较长的令牌。

如果你使用云小狐提供的订阅服务，应从对应账户页面复制订阅链接，不要手动删除链接末尾的参数，也不要在公开截图、工单或聊天记录中暴露完整 URL。订阅链接往往相当于访问凭证，泄露后可能导致配置被他人使用或账户信息暴露。

复制前建议检查以下事项：

1. 链接是否完整，尤其是 `?` 后面的参数是否保留。
2. 链接前后是否误带空格、换行或中文标点。
3. 订阅是否仍在有效期内，账户是否有可用流量或权限。
4. 服务商提供的格式是否明确支持 Clash、Clash Meta 或 Mihomo。

Clash Verge Rev 的远程配置功能要求订阅响应内容符合客户端能够解析的配置格式；普通网页地址、登录页面地址或只返回文本说明的地址，不能直接当作 Clash 配置使用。([github.com](https://github.com/clash-verge-rev/clash-verge-rev.github.io/blob/main/docs/guide/profile.md))

### 3. 在 Profiles 页面导入订阅

打开 Clash Verge Rev，进入 **Profiles** 页面。不同版本或语言翻译可能将其显示为“配置”“配置文件”或类似名称。

按照以下流程操作：

1. 点击添加远程配置或导入订阅的入口。
2. 将完整订阅链接粘贴到 URL 输入框。
3. 按需填写一个便于识别的名称，例如“主订阅”或“个人订阅”。
4. 保存或点击导入。
5. 等待客户端下载并解析配置。
6. 导入成功后，点击该配置卡片，使其成为当前使用的配置。

官方文档列出了远程订阅链接导入、订阅链接配置和 URL Scheme 等方式。对于首次使用者，直接在 Profiles 页面粘贴 URL 是最容易排查的一种方式。如果直接导入提示 `client error(Connect)`，可以改用订阅链接配置方式，并检查是否存在证书或网络连接问题。([github.com](https://github.com/clash-verge-rev/clash-verge-rev.github.io/blob/main/docs/guide/profile.md))

### 4. 手动更新订阅并确认更新时间

订阅卡片出现并不一定代表配置已经成功下载。导入后，建议在对应卡片的菜单中点击 **Update** 或“更新”，等待操作完成，再查看最近更新时间或状态提示。

确认更新成功时，重点观察三项：

- 配置卡片没有显示错误状态。
- 最近更新时间发生变化。
- 进入 Proxies（代理）页面后，可以看到代理组和节点列表。

如果配置卡片存在但节点列表为空，通常说明订阅响应没有被正确解析、返回的内容不是 Clash/Mihomo 配置，或者订阅本身已经失效。不要仅凭“卡片名称出现”判断导入完成。

如果客户端提供自动更新间隔，可以根据订阅服务的建议设置。更新间隔只是控制客户端多久重新请求一次订阅地址，不会直接提高节点速度；过于频繁的请求还可能触发服务端限制。Clash Verge Rev 的配置编辑页面也提供了更新间隔等选项。([github.com](https://github.com/clash-verge-rev/clash-verge-rev/blob/dev/src/locales/en/profiles.json))

### 5. 选择代理组和节点

进入 **Proxies** 页面，展开主要代理组，选择一个可用节点或策略组。订阅中显示的分组名称、节点名称和测速按钮可能因服务商配置而不同，客户端不会替你判断某个节点一定适合所有网站或应用。

建议按以下顺序测试：

1. 先选择一个明确可用的节点。
2. 如果代理组支持延迟测试，执行一次测试作为参考。
3. 选择延迟较低且状态正常的节点。
4. 切换节点后等待几秒，再进行网页或应用测试。

延迟测试只说明测试目标能够响应，不等于所有网站、所有协议或所有应用都能正常访问。如果测速成功但浏览器打不开网页，还需要继续检查系统代理、DNS、TUN 设置以及浏览器自身的代理配置。

### 6. 开启系统代理

打开设置页面，找到 **System Proxy** 或“系统代理”开关并启用。系统代理开启后，遵循操作系统代理设置的应用通常会把 HTTP 或 HTTPS 请求交给 Clash Verge Rev 处理；但某些应用可能使用独立网络栈、忽略系统代理，或者需要额外启用 TUN 模式。

首次排查时建议只开启系统代理，不要同时修改多个高级选项。这样可以先确认最基础的链路是否成立：

> 订阅配置已更新 → 节点已选择 → Clash Verge Rev 正在运行 → 系统代理已开启 → 浏览器重新加载页面。

如果浏览器此前配置过手动代理，可能与客户端的系统代理设置互相覆盖。遇到行为异常时，先恢复浏览器的默认代理设置，再只保留 Clash Verge Rev 的系统代理开关。

### 7. 选择 Rule、Global 或 Direct 模式

Clash Verge Rev 使用的 mihomo 内核提供三种常见运行模式：

- **Rule（规则模式）**：根据配置文件中的 `rules` 从上到下匹配请求，并将请求交给对应的代理组、直连或拒绝策略。
- **Global（全局模式）**：所有请求都交给当前选中的代理或代理策略组，通常还需要在 `GLOBAL` 组中选择具体出口。
- **Direct（直连模式）**：所有请求直接连接目标，不经过代理。

mihomo 官方文档明确列出了这三种模式，并说明规则模式是默认模式。规则匹配通常按配置中的顺序执行，靠前的规则优先级更高，最后常见的 `MATCH` 规则用于处理没有被前面规则匹配到的请求。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/general/))

日常使用通常可以先选择 **Rule**：国内或无需代理的请求按配置直连，需要代理的请求交给相应策略组。遇到某个应用在规则模式下行为不符合预期时，可以临时切换到 **Global** 做对比测试；测试完成后，建议切回 Rule，避免所有流量都经过同一个代理出口。只需要确认本机是否可以直连时，可以选择 Direct。

### 8. 通过连接记录定位问题

如果网页仍然无法打开，进入 **Connections** 或“连接”页面，观察请求是否出现，以及请求最终匹配到了哪个策略。这个步骤可以区分不同问题：

- 没有任何连接记录：应用可能没有使用系统代理，或请求被其他代理软件接管。
- 有连接记录但全部为 Direct：当前规则将请求判定为直连，或模式处于 Direct。
- 有连接记录但节点连接失败：重点检查节点状态、网络环境和订阅有效性。
- 只有部分域名失败：可能与规则、DNS、IPv6 或目标站点自身状态有关。

修改配置或模式后，重新打开页面并观察新的连接记录，不要只看之前残留的日志。

## 常见问题

### 1. 导入订阅时提示 `client error(Connect)`，怎么办？

先确认电脑当前网络可以访问订阅域名，并检查 URL 是否完整。然后尝试在 Profiles 页面使用另一种远程订阅导入方式；官方文档也建议在该错误出现时尝试其他导入方式。如果订阅服务提供了“通过系统代理更新”或“通过 Clash 代理更新”的选项，可以在确认代理链路已经可用后再尝试。不要一开始就勾选“允许无效证书”，除非你明确知道订阅服务的证书问题来源，因为该选项会降低 TLS 证书校验的安全性。([github.com](https://github.com/clash-verge-rev/clash-verge-rev.github.io/blob/main/docs/guide/profile.md))

### 2. 出现 401 或 403，是否说明客户端坏了？

通常不是。401 表示服务端要求认证或当前凭证无效，403 表示服务端拒绝当前请求。请回到订阅服务后台检查登录状态、订阅权限、链接是否过期，以及是否复制了完整的新链接。不要反复删除并重新导入同一个无效 URL；如果服务商已经生成新订阅链接，应编辑原配置的 URL 或新增配置后再测试。

### 3. 配置已经导入，但 Proxies 页面没有节点，怎么办？

先手动更新配置并查看错误提示。随后确认订阅返回的是 Clash/Mihomo 配置，而不是 HTML 登录页、JSON 错误信息或普通文本。也可以将订阅链接交给服务商核对格式。不要直接把下载后的配置文件发布到公共平台，因为其中可能包含节点地址、密码或其他敏感信息。

### 4. 节点测速成功，但浏览器仍然打不开网页，怎么办？

依次检查：当前配置是否被选中、节点或代理组是否已选择、系统代理是否开启、浏览器是否设置了其他手动代理，以及当前模式是否误设为 Direct。若只有特定域名失败，再查看 Connections 页面中的匹配规则和错误日志。必要时可暂时使用 Global 模式进行对比，但这只是排查手段，不代表所有请求都必须全局代理。

### 5. 规则模式和全局模式应该怎么选？

规则模式适合日常使用，因为它按照订阅配置中的路由规则分别处理请求；全局模式适合临时验证“是否必须经过代理才能访问”的问题，但会让所有请求都使用当前代理组。Direct 模式则用于确认直连状态。模式切换不会自动修复失效订阅、错误节点或 DNS 问题，因此仍应结合连接记录和日志判断原因。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/general/))

### 6. 是否必须开启 TUN 模式？

不一定。系统代理可以覆盖遵循系统代理设置的应用；TUN 是虚拟网络接口方案，适合处理不遵循系统代理的程序，但通常涉及更高权限、路由和 DNS 设置。建议先用系统代理完成基础验证，只有确认目标应用不支持系统代理时，再根据客户端和操作系统文档配置 TUN。mihomo 官方文档将代理端口、TUN 和监听器列为不同的流量接入方式，具体是否需要取决于应用和网络环境。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/inbound/))

## 参考资料

- [Clash Verge Rev 官方 GitHub 仓库](https://github.com/Clash-Verge-rev/clash-verge-rev)
- [Clash Verge Rev 官方 Releases](https://github.com/Clash-Verge-rev/clash-verge-rev/releases)
- [Clash Verge Rev 官方文档：Profile 配置与订阅导入](https://github.com/clash-verge-rev/clash-verge-rev.github.io/blob/main/docs/guide/profile.md)
- [Clash Verge Rev 官方文档：URL Schemes 与订阅响应头](https://www.clash-verge-rev.org/guide/url_schemes)
- [mihomo 官方文档：全局配置与运行模式](https://wiki.metacubex.one/config/general/)
- [mihomo 官方文档：路由规则](https://wiki.metacubex.one/config/rules/)
- [mihomo 官方文档：入站配置](https://wiki.metacubex.one/config/inbound/)

## 关于 云小狐

云小狐提供网络加速相关服务及使用指引，本文仅介绍 Clash Verge Rev 客户端的通用配置流程，不代表对具体节点、速度或可用性的承诺。更多服务信息请访问[云小狐官网](https://www.yunxiaohu.com/)。