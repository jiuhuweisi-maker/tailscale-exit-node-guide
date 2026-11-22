# Tailscale 出口节点：使用、排障与迁移

适用于已有或准备搭建 Linux VPS 出口节点的个人和小团队。搭建命令见 [SETUP.md](SETUP.md)，发给好友的简明流程见 [README](README.md)。

## 1. 什么时候适合用

- **设备经常变化**：Windows、iPhone 可以用自己的 Tailscale 身份登录。一台 VPS 可以给多台获准设备提供出口。
- **直连线路不稳定**：Tailscale 优先建立直连，必要时可以走中继。中继增加一段路径，延迟和吞吐量可能比直连差。先在实际宽带、蜂窝网络上测，再决定是否迁移。
- **已有 VPS 业务**：出口节点与站点反代是两件事，启用出口节点不需要更改网站 DNS。保留现有 SSH，先验证新管理通道再退役旧代理。
- **需要细粒度规则或更高吞吐**：原生 Exit Node 以整机出网为主。需要国内外分流时，用具备规则引擎的客户端，或者让原生出口只在需要整机转发时启用。

Cloudflare 支持 WebSocket，但自助订阅协议第 2.2.1(j) 条对利用其服务提供 VPN 或类似代理有限制，除非得到 Cloudflare 明确书面许可。通用代理和网站加速要分别选型。

[出口节点](https://tailscale.com/docs/features/exit-nodes) · [连接类型](https://tailscale.com/docs/reference/connection-types) · [Cloudflare 自助订阅协议](https://www.cloudflare.com/terms/)

## 2. 三种线路怎么走

| 模式 | 数据路径 | 适合场景 |
| --- | --- | --- |
| 原生 Exit Node | 设备 → Tailscale 直连或中继 → VPS → 公网 | 整机共用 VPS 出口，适用于 Windows、iPhone |
| Windows 规则模式 | 大陆域名/IP 直接出网；其余经 FlClash → 本机 SSH SOCKS → 尾网 VPS | 按规则分流，使用遵循系统代理的应用 |
| 保持本地默认线路 | 默认按原来的规则出网，按需连接尾网并启动 VPS 通道 | 偶尔才需要某个远程出口 |

在 Windows 分流模式里，Tailscale 要连上尾网，同时把 **Exit Node 设为 None**。如果先让 Tailscale 接管系统默认出口，FlClash 标记的 `DIRECT` 仍可能经该出口。SSH 动态转发只提供 TCP；原生 Exit Node 的 UDP 能力与之不同。用来连接尾网 VPS 的普通 SSH 需要服务器允许转发，并用服务器自己的 SSH 身份认证。

规则先匹配大陆域名，再匹配已知境外域名，最后检查大陆 IP 并用 `MATCH` 兜底。`no-resolve` 适合匹配已有 IP 的请求，可避免为这条规则额外解析域名。首次加载规则集时，先让 SOCKS 通道可用再通过它拉取规则。两套 TUN 同时接管网络时，先用一个 TUN 验证，避免各自捕获对方的控制流量。

[FlClash/Mihomo 例子](SETUP.md#可选windows-大陆分流) · [Mihomo 路由规则](https://wiki.metacubex.one/en/config/rules/) · [MetaCubeX 规则源](https://github.com/MetaCubeX/meta-rules-dat)

## 3. Windows 启动与按需切换

Windows 的 Tailscale GUI 自启动、Tailscale 是否连接、FlClash 是否自动运行，以及 SSH SOCKS 是否在监听，是四项独立状态。操作前在托盘检查 Tailscale 的连接与 Exit Node 选择，在 FlClash 检查选中的配置、规则模式和系统代理。

**常用线路默认启动、VPS 偶尔用**

1. 保留 FlClash 常用配置。Windows 登录时先选择默认配置，再启动 FlClash。若客户端会记住上次配置，需在启动流程里明确选择默认配置。避免两个启动项同时启动同一个客户端。
2. 登录时可用 `tailscale down` 保持尾网断开。将切换流程绑定到桌面快捷方式，双击后连接 Tailscale、确认 Exit Node 为 None、启动本机 SSH SOCKS 并检测出口，再选择对应的 FlClash 配置。
3. 用完先切回常用配置，再断开尾网。只有需要整机从 VPS 出网时，才在 Tailscale 选择 Exit Node。

**分流作为日常默认**

让登录启动器先等 Tailscale 和 VPS 在线，再启动 `ssh -N -D`。检查本机 SOCKS 监听端口，已有通道时跳过重复启动。启动器只应在本机监听，失败应有退避重试，日志保留错误类型即可。先手动触发启动器实测，再通过实际注销或重启确认启动顺序。

## 4. 按顺序验收和定位故障

| 检查 | 具体做法 | 能证明什么 |
| --- | --- | --- |
| 节点注册与路由批准 | 后台查看节点在线，确认 **Use as exit node** 已批准 | 已注册且可供获准客户端选择 |
| 尾网连通和路径 | 客户端执行 `tailscale ping <节点名>` | 能到达节点，以及当前走 `direct` 还是 `DERP` |
| 管理通道 | 新建 SSH 会话，登录尾网地址并执行只读命令 | 在保留旧管理通道时，已验证另一条可用 SSH 路径 |
| 实际出网 | 用客户端访问外部 HTTPS 服务并核对出口 IP | 已验证从应用经目标线路访问公网 |
| 大陆分流 | 比较大陆网站经规则访问与本地直连的出口，再核对境外请求的 VPS 出口 | 两条规则分支分别生效 |
| 服务与资源 | 切换前后比较现有服务健康检查、`free -m`、`vmstat 1 3` | 判断变更是否影响业务或引起持续换页 |

端口能连、Tailscale 能 ping 通和浏览器能从 VPS 出网，是三项不同的验收。只测一次也无法证明长期稳定。重要场景应在目标网络上测试重连、休眠唤醒、Wi‑Fi 与蜂窝网络切换。

常见问题

- **节点在线但下拉里看不到**：先看出口是否获批，再看客户端账号、共享邀请和访问策略。
- **Windows 一直显示正在启动**：先确认当前 Tailscale 账号及配置，再暂时排除其他 TUN 与系统代理冲突。区分能否访问协调服务器与能否通过尾网传输数据。服务确实卡住时，由 Windows 管理员执行 `Restart-Service Tailscale` 后再查 `tailscale status`。
- **DERP 能通却较慢**：记录客户端到节点的中继延迟，并用同一网络复测。实际速度还受 VPS 带宽、上游路径和并发数影响。
- **SOCKS 能出网，FlClash 打开网页却失败**：检查选中的配置、系统代理、规则集是否已加载、DNS 是否被全局覆写。仅关闭 TUN 很可能不会改变全局 Fake-IP 设定；改成 `redir-host` 后再验证两个分流方向。
- **iPhone 代理互相顶掉**：试用原生 Exit Node 时先关闭另一款 VPN/代理客户端，再用 Safari 核对出口。Windows 的 Clash YAML 属于 Windows 分流例子。

[Tailscale CLI](https://tailscale.com/docs/reference/tailscale-cli) · [连接类型](https://tailscale.com/docs/reference/connection-types)

## 5. 浏览器显示的 IP 和 WebRTC 泄漏

浏览器页面显示的是 HTTP(S) 请求的出口；WebRTC 的 ICE/STUN 另行探测网络路径。在系统代理加 SSH SOCKS 的模式下，网页可能走 VPS，STUN 仍可能发现本地公网 IP。DNS 规则无法阻止这类 UDP 直连。

在 Edge 或 Chromium 可限制非代理 UDP。以 WebRTC Network Limiter 为例，安装后进入扩展选项，选择 **Use my proxy server (if present)**。关闭原检测标签，再新开一页重测。网页仍应显示预期出口，STUN 可以无结果，但不应再显示本地公网 IP。此设置可能影响网页语音、视频；需要保留 WebRTC UDP 时，优先实测原生 Exit Node 的整机路由。

[Edge WebRTC 策略](https://learn.microsoft.com/en-us/deployedge/microsoft-edge-policies/webrtclocalhostiphandling) · [WebRTC Network Limiter](https://chromewebstore.google.com/detail/webrtc-network-limiter/npeicpdbkakmehahjeeohfdhnlpdklia)

## 6. 低内存 VPS 上如何安全迁移

1. **检查依赖**。用 `systemctl`、`ss -lntup` 和现有站点反代配置确认监听归属。旧代理面板可能还负责订阅、其他入站，反代也可能依赖它。不要单看代理协议名就停整个进程。
2. **先试跑**。将试跑与现有业务的进程、控制套接字和状态分开。公共 SSH 保持可用。试跑如果使用临时节点身份，切到常驻节点时应计划重新注册、批准、验证，并保护状态文件。
3. **备份后切换**。备份配置、数据库、相关服务定义和待改防火墙规则。确认新节点对实际客户端可用、独立 SSH 可用、原业务健康后，优先禁用废弃入站；确认进程无其他作用才考虑停服务和取消自启。
4. **只清理独占资源**。端口可能由多种协议共用，例如同一端口的 TCP 与 UDP。只删除已确认归旧线路独占的防火墙规则。暂停服务后记录恢复命令，保留数据与备份。
5. **比较实际开销**。看可用 RAM、Swap 使用量及 `vmstat` 的 `si/so`，并在客户端有流量时复测。已经占用 Swap 不代表正在持续换页；禁用共享进程中的两个入站也不能把整个进程的内存算成节省。

回滚时优先恢复独立管理连接和旧路由，再恢复应用流量。遇到异常先停止清理，验证现有网站、反代和自动化服务；不要靠重启无关服务试错。

## 7. 好友共享的权限边界

共享单个出口节点，收件人用自己的账号接受邀请。指南链接只用于阅读；邀请才授予相应访问。评估是否需要限制对节点自身的 SSH、管理面板及其他监听的访问。离开共享关系时，可在后台取消共享。多个用户共用同一 VPS 时也会共用它的带宽，应在并发使用的情况下测试实际体验。

发布教程时，用占位符替换真实公网 IP、尾网地址、账户、设备名和电脑路径。请将邀请链接、SSH 私钥（包括服务器主机私钥）、管理令牌、状态文件和私人代理配置保存在受限位置。

[设备和出口节点共享](https://tailscale.com/docs/features/sharing)

欢迎补充其他系统的验收方法、失败案例或改进建议。提交前请清除日志和截图里的个人信息。
