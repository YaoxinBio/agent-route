# 客户端证据与可选检查入口

本页保存具体版本的来源与历史观察，供执行 Agent 按需查证，不是通用行动准则，也不是要求复刻的操作方案。项目目标和判定以 [AGENTS](../AGENTS.md)、[CLIENTS](CLIENTS.md)、[DNS-WEBRTC](DNS-WEBRTC.md) 为准。一次客户端修复的成功不能推成其他版本、设备或完整流程的保证。

## 客户端案例

以下证据用于解释为什么要检查规则最终顺序、配置来源、DNS 全阶段路径和实际运行结果。它们不会自动扩大本次设备操作授权；过期时重新核对官方资料，而不是继续追加兼容性补丁。

| 范围 | 易错点 | 依据 |
| --- | --- | --- |
| Surge | 加密 DNS 默认直连，不经过规则。开启 `encrypted-dns-follow-outbound-mode` 后，命中的节点地址如果是域名，这条连接仍回退直连 | [手册](https://manual.nssurge.com/dns/encrypted-dns.html) |
| Surge | 模块里的规则只能用 DIRECT、REJECT 等内置策略，代理规则放不进去 | [手册](https://manual.nssurge.com/profile/module.html) |
| Mihomo | `respect-rules` 要配 `proxy-server-nameserver`，否则节点解析可能依赖节点自身。设置 `direct-nameserver` 后，直连的二次解析默认不遵循 `nameserver-policy`，内网名可能被发给它 | [文档](https://wiki.metacubex.one/config/dns/) |
| Clash Verge Rev | v2.5.5 起，扩展配置只替换 `dns` 下写出的子字段，未写的保留；写出 `nameserver-policy` 会替换整个策略表，不逐条合并。`ipv6`、`tun.enable` 等设置页接管的字段会在末尾回写 | [文档](https://www.clashverge.dev/guide/extend.html)、[v2.5.10 合并代码](https://github.com/clash-verge-rev/clash-verge-rev/blob/c83f7ad529c7c9cb3d2d54a0955ef8c1b22a411b/src-tauri/src/enhance/merge.rs#L19) |
| Clash Verge Rev | v2.5.10 先处理规则编辑器，再执行全局扩展和订阅扩展；后续脚本直接 prepend 托管块，会把它放到编辑器中的用户例外前。可在同一“编辑规则”入口按“用户例外 → 托管规则”排列 prepend，再核对后续扩展及最终生成配置 | 2026-11 核对 [v2.5.10 执行顺序](https://github.com/clash-verge-rev/clash-verge-rev/blob/c83f7ad529c7c9cb3d2d54a0955ef8c1b22a411b/src-tauri/src/enhance/mod.rs#L916) |
| Clash Verge Rev | v2.5.10 开启 DNS 覆写时，实际接管的 DNS 字段和 hosts 会在扩展之后恢复。2026-11-06 官方网页对此与该版本源码描述不同，不能只凭“扩展后执行”认定能覆盖设置；按实际版本核对最终配置 | [v2.5.10 回写代码与测试](https://github.com/clash-verge-rev/clash-verge-rev/blob/c83f7ad529c7c9cb3d2d54a0955ef8c1b22a411b/src-tauri/src/enhance/mod.rs#L1439)，未经本项目实机验证 |
| Clash Verge Rev | v2.5.7 优先读取当前订阅 UID 对应的 `profile_dns_settings`，没有该项才回退到 `enable_dns_settings`。不能只看旧全局字段判断 DNS 覆写是否生效；核对当前订阅的实际开关及最终生成配置，不默认关闭覆写 | [v2.5.7 源码](https://github.com/clash-verge-rev/clash-verge-rev/blob/ea509b82363a40c3c32e951d7ce9d66d66da411f/src-tauri/src/config/dns.rs#L116)；2026-11 [用户实机补充](https://github.com/ryuu-restart/agent-route/issues/11#issuecomment-6678341203)，仅限所述环境，未独立复现 |
| FlClash | “DNS覆写”“追加系统DNS”会改变订阅里的 DNS 策略 | 2026-10 实测，[代码](https://github.com/chen08209/FlClash/blob/main/lib/common/task.dart) |
| FlClash | “自动设置系统DNS”把 `223.5.5.5` 写进系统设置。网络的 DNS 来自 DHCP 时，系统里就只剩这个公网地址，Mihomo 的 `system` 随之指向公网，内网名会发给它 | 2026-11 本机核对，[代码](https://github.com/chen08209/FlClash/blob/v0.9.2/lib/common/system_dns.dart) |
| FlClash | “查找进程”关闭时 `find-process-mode` 为 off，会覆盖订阅里的值，PROCESS 规则不生效 | 2026-11 本机核对 |
| sing-box | 路由规则不会替域名解析（除非用 `resolve`），直接传入 SOCKS / HTTP 的未知域名走默认出站。没有 `sniff` 时，只有 IP 的流量匹配不到域名规则 | [文档](https://sing-box.sagernet.org/configuration/route/rule_action/)，2026-10 隔离实测 |
| FakeIP | 常用的 FakeIP IPv6 段在 `fc00::/7` 内，这条 LAN 规则排在前面时，FakeIP 流量可能被当成内网直连 | [项目模板](../examples/full/mihomo.yaml)的规则顺序和测试 |
| Shadowrocket | 普通 DoH 地址不经过节点，要加 `#proxy`。`proxy-dns-server` 只解析节点域名。`[Host]` 没写能引用规则集，内网域名要逐条加 | [社区手册](https://github.com/LOWERTOP/Shadowrocket)（非官方，未经真机验证），2026-10 用户反馈 |
| Chrome / Chromium | 策略为 `disable_non_proxied_udp` 时 ICE 候选可能为 0，这不证明安全。TUN 下不装扩展，srflx 也可能已是节点出口 | 2026-11 对照：无扩展临时配置档与原配置档比较 |
| 广告名单 | 会拦一些遥测和功能开关域名，如 `statsig.anthropic.com`、`featuregates.org`。AI 工具行为异常时先查这里，实际影响未实测 | 2026-11 名单抽查 |

## 系统 DNS 的只读入口示例

命令由执行 Agent 依据现场系统选择。系统当前值、接口下发值和核心实际使用的解析器是不同层次；无法获得某项证据时如实记录，不能由另一层的标签推断。

| 系统 | 只读检查与边界 |
| --- | --- |
| macOS | `scutil --dns` 查看当前解析器；`ipconfig getpacket <实际接口>` 查看该接口的 DHCPv4 数据。将占位符换成当前物理接口，不假定是 `en0`；无 DHCP 租约时不能据此取得下发值，IPv6 另查 |
| Windows | [Get-DnsClientServerAddress](https://learn.microsoft.com/en-us/powershell/module/dnsclient/get-dnsclientserveraddress) 查看各接口的 IPv4 / IPv6 DNS，物理与 TUN 接口都要辨认；`netsh interface ipv4 show dnsservers` 辅助查看接口 DNS 配置（[说明](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netsh-interface)） |
| Windows IPv4 来源线索 | 注册表 `HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\{GUID}` 的 `NameServer` / `DhcpNameServer` 仅作静态配置与 DHCP 来源线索，先对应当前接口；可能缺失或残留，不能判断由谁修改，也不能证明当前查询路径 |
| Windows 系统 DoH | [Get-DnsClientDohServerAddress](https://learn.microsoft.com/en-us/powershell/module/dnsclient/get-dnsclientdohserveraddress) 查看已知服务器的 DoH 配置，核对 `AutoUpgrade`、`AllowFallbackToUdp` 与当前实际使用的解析器；列出服务器不代表正在使用 DoH，也不代表浏览器独立安全 DNS 或查询出站已验证 |

### 应用接管不等于核心内网解析已核实

[#12 的补充反馈](https://github.com/ryuu-restart/agent-route/issues/12#issuecomment-6679926933)提供了一个案例：普通应用查询被接管，但核心的 `system` 另有读取来源。已核对该报告对应的 [Mihomo 读取代码](https://github.com/MetaCubeX/mihomo/blob/88dcbf7f1614a67c3b36b848ee3592dfa92ada36/dns/system_posix.go#L13)及 [UDP 客户端构造](https://github.com/MetaCubeX/mihomo/blob/88dcbf7f1614a67c3b36b848ee3592dfa92ada36/dns/system_common.go#L15)，以及 [GUI 设置占位地址的条件](https://github.com/clash-verge-rev/clash-verge-rev/blob/4337bd85758d54d7b9044b2da3c9ac1176eae1f3/src-tauri/src/utils/resolve/dns.rs#L13)。这支持“分别核对应用侧与核心侧实际路径”的通用检查要求；报告者的设备与网络观察未由本项目独立复现，不提供关闭客户端开关或改系统 DNS 的通用处方。
