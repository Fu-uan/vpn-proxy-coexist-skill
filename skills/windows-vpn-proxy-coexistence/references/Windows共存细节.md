# Windows 共存细节

本文覆盖 iOA 的 WFP 模式、Clash Verge Rev 2.4.7、Mihomo v1.19.21 与 sing-tun v0.4.16 的特定行为。其他版本应重新核查源码和运行结果。它是排障参考，不能作为整份配置导入。

## 1. 平台与配置优先级

macOS 的 `utun` 出口绑定方案不能直接用于 Windows WFP。WFP 可能依赖原生连接的重定向；不存在可供照抄的 iOA `utunN`。不要从别人的模板复制企业网关、fake IP 段或关闭 IPv6 的设置。

最终配置可能受活动订阅的增强映射、全局配置、UI 的 TUN 开关和独立 DNS 设置共同影响。核对当前配置映射后再改文件，避免编辑已停用的增强文件。

在 2.4.7 中，启用独立 DNS 设置且其文件有效时，增强链末尾通过 `dns_config.yaml` 覆盖整个 DNS 段，并非逐字段合并。因此只改 merge 中的 DNS 不一定有效。经 UI 重载后检查运行配置与端口实际监听；不能只读落盘 YAML 就宣布生效。

## 2. DNS 端口与解析链

当 iOA 的 DNS 服务需要回环 `:53`，而 Clash 已占用冲突监听时，可将 Clash DNS **候选监听**改为一个已确认空闲的回环高端口，例如：

```yaml
dns:
  listen: 127.0.0.1:1053
```

这不是完整配置；保留其他 DNS 设置。TUN 中原有 `dns-hijack: [any:53]` 属于内部接管，不能因为监听改成 1053 就机械替换为 1053。

公司 DNS policy 只能指向已确认可用、由正确进程监听的 DNS。若 `127.0.0.1:53` 实际仍是 Clash，转发到它可能递归。保留实际需要的公司 DNS，不能用公共解析器替换私有解析链。

公司域名的 fake IP 过滤和 DNS policy 要覆盖根域及所需子域。Mihomo 的 `+.`、`*.` 与 Windows 代理绕过的通配语法不同，不要混用。

## 3. DIRECT 不等于绕过 WFP 冲突

Mihomo 在 Windows 上可通过 `IP_UNICAST_IF` 选择出口接口。启用 `auto-detect-interface` 的默认接口发现可能影响 DIRECT 出站。

在一个已验证的 WFP 案例中，同一企业目的地址的普通连接成功，使用与 Mihomo 相同的接口选择方式立即返回 `WinError 10049`。这支持出口绑定冲突；单独 `socket.bind(本机地址)` 并不是相同实验，也不能仅凭错误码推广到所有环境。

证据支持这一机制时，可让企业流量走 Windows 原生路径：系统代理绕过公司域名；在 TUN 中排除实际公司目标段；公司域名使用正确 DNS 并过滤 Clash fake IP。需要一并验证，单加 DIRECT 不够。

不要一刀切关闭 `auto-detect-interface`：代理的外层连接可能重新进入 TUN。强制使用显式代理的应用也可能忽略系统绕过，应单独配置和验收。

## 4. 系统代理绕过：格式与缓存

Windows 实际绕过列表通常用分号。Clash Verge 的 UI 有自己的编辑和校验方式；按当前版本界面要求输入并保存，再核对系统结果。根域和子域显式列出。

下面只示范语法；`corp.example.invalid` 是保留的虚构域名。应用前将其替换为实际域名，并与用户**完整已有名单**合并：

```text
localhost;127.*;<local>;corp.example.invalid;*.corp.example.invalid
```

不要用这个短示例覆盖现有私网名单。不要为省事扩大为 `172.*`，因为其中包含公网地址。提供文本时使用代码块，避免 `<local>` 被渲染为 HTML 或混入换行。这里也不使用 Mihomo CIDR 规则语法。

2.4.7 中，默认与非空自定义绕过启用时，后端用逗号连接两份列表，Windows UI 则使用分号。可在 UI 中关闭“使用默认绕过”，再保存包含**完整原默认名单**和企业域名的自定义列表。对应持久化字段为 `use_default_bypass`、`system_proxy_bypass`，格式以当前客户端为准。

运行中的 UI 缓存可能把手工改过的 `verge.yaml` 整体写回。磁盘与注册表一致并不代表 UI 已接受保存。优先经 UI 保存；格式无效时修正格式，不要绕过校验。比较名单时可拆分成集合，避免仅因顺序或分隔符差异误判。

## 5. fake IP 被更具体的物理路由截走

以下使用**等价替换后的示例地址**说明该版本的布局，不保留原环境地址：

| 项目 | 值 |
| --- | --- |
| Clash fake IP 池 | `198.18.0.1/16` |
| TUN 服务接口 | `198.18.0.1/30` |
| TUN DNS | `198.18.0.2` |
| 分配给应用的某地址 | `198.18.0.73` |
| 故障时动态路由 | `198.18.0.0/24` 指向 WLAN 物理出口 |

这些是机制说明使用的基准测试地址，不是公司地址或通用推荐配置。必须在故障时用实际选路核查：更长的 `/24` 会胜过 TUN 较宽路由，接口跃点更小也无法改变此前缀优先关系。仅看地址相似不够。

该案例中，显式代理 HTTPS 可达，直接 TUN 超时；错误路由消失时，本机桥接自行恢复。没有证据确认该动态路由的写入者，不能断言来自 iOA。

### 只在前提全部成立时考虑保留段拆分

- 当前版本的 fake IP 分配从网络地址 `.4` 开始，`.0` 到 `.3` 是服务保留段。
- TUN 已有覆盖服务地址 `.1/.2` 的 on-link `/30` 路由。
- 实测冲突正是同池首个 `/24` 被物理路由抢走。
- Windows 当前版本确实从地址集合减法生成更具体路由，且该方法不会影响现有企业排除项。
- 有断连后仍可执行的恢复方式，并能立即检查 DNS 服务地址与实际目标选路。

核实依赖可查对应 Mihomo tag 的 `go.mod`（[v1.19.21 固定使用 sing-tun v0.4.16](https://github.com/MetaCubeX/mihomo/blob/v1.19.21/go.mod#L36)），再核对下方该依赖版本的集合减法源码。定制或重新编译的内核仍需核实实际构建信息，不能仅凭客户端版本推定。

对应本节示例的修复，是在**保留原列表**的基础上，向 `tun.route-exclude-address` 追加 `198.18.0.0/30`。这是列表追加操作，不要把下面片段当完整 merge 文件覆盖原列表：

```yaml
# 仅展示追加的一个元素；必须保留全部现有排除项
route-exclude-address:
  - 198.18.0.0/30
```

此处“排除”并非让可分配 fake IP 走物理网卡。sing-tun 将保留服务段从自动路由集合减去后，在此版本生成覆盖剩余地址的更具体前缀，包括：

```text
198.18.0.4/30
198.18.0.8/29
198.18.0.16/28
198.18.0.32/27
198.18.0.64/26
198.18.0.128/25
```

这些前缀覆盖首 `/24` 的全部可分配地址，且比错误物理 `/24` 更具体。本节示例中的 `.73` 应选中 TUN `/26`，服务 `.2` 仍由 on-link `/30` 到达；完整 `/16` fake IP 池不需要更换。原案例已按同一机制核对实际选路与服务可达性。

**任一前提不成立，就不应用这个固定 `/30`。** 特别是服务地址布局、池起点、冲突前缀或操作系统不同的环境。另一 VPN 自己的地址池也不应套入此补丁。

不要把这种行为包装成所有路由冲突的通用解法。不要直接删除企业路由、反复更换 fake IP 池，或使用隐藏常驻的第二个 TUN 来掩盖根因。

### 此版本容易误用的字段

- Mihomo v1.19.21 的全局 `tun.inet4-address` 在原始配置结构中被注释，不是可用 YAML 参数。API 出现内部同名值，也不证明手写配置生效。
- 自定义 `listeners: type: tun` 与常规 UI TUN 开关并非同一控制路径，可能出现双 TUN；不能作为透明替代。
- `route-exclude-address-set` 不是 Windows 的 CIDR 列表替代项。Windows 这里核查的是 `route-exclude-address`。

## 6. 验证层级与已知边界

静态 YAML 校验 → UI 保存和运行配置 → DNS/路由/传输 → 应用业务 → dot 本机只读执行，各层不能互相替代。普通对话与传统 CLI 远控注册也不能替代 dot 本机连接验收。

该脱敏案例在同一“三开”组合下通过企业页面、Codex 对话、dot 在线与实际本机只读操作，并完成一次短时无新断连观察。重启恢复、长期运行和其他版本未验证，不据此承诺普适性。

## 公开依据

以下链接固定到对应开源版本，供复核，不复制上游源码。上游许可适用于各自项目；本仓库 MIT 仅覆盖本仓库原创内容。

- [Clash Verge Rev v2.4.7：增强链与 DNS 覆盖](https://github.com/clash-verge-rev/clash-verge-rev/blob/v2.4.7/src-tauri/src/enhance/mod.rs#L562-L585)
- [Clash Verge Rev v2.4.7：系统代理与绕过合并](https://github.com/clash-verge-rev/clash-verge-rev/blob/v2.4.7/src-tauri/src/core/sysopt.rs#L37-L63)
- [Clash Verge Rev v2.4.7：绕过编辑、校验和保存](https://github.com/clash-verge-rev/clash-verge-rev/blob/v2.4.7/src/components/setting/mods/sysproxy-viewer.tsx#L61-L87)
- [Clash Verge Rev v2.4.7：UI TUN 覆盖](https://github.com/clash-verge-rev/clash-verge-rev/blob/v2.4.7/src-tauri/src/enhance/tun.rs)
- [Mihomo v1.19.21：Windows 接口绑定](https://github.com/MetaCubeX/mihomo/blob/v1.19.21/component/dialer/bind_windows.go#L15-L35)
- [Mihomo v1.19.21：fake IP 分配](https://github.com/MetaCubeX/mihomo/blob/v1.19.21/component/fakeip/pool.go#L158-L175)
- [Mihomo v1.19.21：配置字段](https://github.com/MetaCubeX/mihomo/blob/v1.19.21/config/config.go#L267-L279)与 [TUN 地址派生](https://github.com/MetaCubeX/mihomo/blob/v1.19.21/config/config.go#L1638-L1656)
- [sing-tun v0.4.16：路由集合减法](https://github.com/MetaCubeX/sing-tun/blob/v0.4.16/tun_rules.go#L104-L147)
- [sing-tun v0.4.16：Windows 路由生成](https://github.com/MetaCubeX/sing-tun/blob/v0.4.16/tun_windows.go#L119-L130)
- [OpenAI：为 dot 连接电脑和应用](https://learn.chatgpt.com/docs/dots/computers-and-apps)
