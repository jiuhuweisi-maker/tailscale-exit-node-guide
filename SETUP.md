# 从零搭建 Tailscale 出口节点

这份教程用一台可以正常访问互联网的 Linux VPS 搭建出口节点，再让 Windows、iPhone 或好友接入。需要 VPS 管理权限，以及可以在 Tailscale 后台批准出口路由的账号。

## 1. 在 Linux VPS 上接入 Tailscale

按发行版使用官方安装方式。下面是官方安装脚本的示例，已装过 Tailscale 的机器可跳过第一行。

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale status
```

打开命令给出的登录地址并用自己的账号完成注册。确认 VPS 已在线。保留现有 SSH 会话，另开终端操作。

[Linux 安装说明](https://tailscale.com/docs/install/linux)

## 2. 启用转发并广播出口节点

以下适用于常见的 Linux 内核网络模式。检查现有 sysctl 配置后，新建 `/etc/sysctl.d/99-tailscale-exit-node.conf`，写入以下两项。

```ini
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

```bash
sudo sysctl -p /etc/sysctl.d/99-tailscale-exit-node.conf
sudo tailscale set --advertise-exit-node
```

如果使用的是 `--tun=userspace-networking`，转发由 Tailscale 实现。按官方文档核对其协议与性能差异。

[配置出口节点](https://tailscale.com/docs/features/exit-nodes/how-to/setup) · [内核与用户态模式](https://tailscale.com/docs/reference/kernel-vs-userspace-routers)

## 3. 批准出口，并从客户端验证

1. 在 Tailscale 后台进入 **Machines**，找到 VPS，点 **Edit route settings**，启用 **Use as exit node**。已经自动批准时可跳过。
2. Windows 或 iPhone 安装 Tailscale，登录同一网络，在 **Exit Node** 选择 VPS。测试时先暂停会接管同一流量的其他代理。
3. 访问网页并查看公网出口 IP，与 VPS 上直接访问公网时的结果比较。单独保留并验证 SSH 管理路径。
4. Windows 可运行 `tailscale ping <节点名>` 检查与 VPS 的连接。结果里的 `direct` 表示直连，`DERP` 表示走中继。

需要停用时，先把客户端的 **Exit Node** 设为 **None**，再从 VPS 执行 `sudo tailscale set --advertise-exit-node=false`。如果还要撤销系统转发，先确认没有其他服务依赖这两个设置。

[客户端操作](https://tailscale.com/docs/features/exit-nodes/how-to/setup) · [Tailscale CLI](https://tailscale.com/docs/reference/tailscale-cli)

## 4. 和好友共享

在 **Machines → Share** 共享这台设备，勾选 **Allow use as an exit node**。每位好友用自己的账号接受邀请。分享前核对访问策略，确认 SSH 和管理服务权限符合预期。邀请链接单独发送。

[官方共享说明](https://tailscale.com/docs/features/sharing) · [好友连接指南](README.md)

## 可选：Windows 大陆分流

适用于需要大陆直连、其他流量走 VPS，且愿意使用 FlClash 或 Mihomo 的 Windows 用户。Tailscale 保持连接，**Exit Node 选 None**。先在电脑上开启只监听本机的 SOCKS 通道，把 `USER` 和 `TAILNET_HOST` 换成你自己的 SSH 用户和尾网节点地址。首次连接时核对服务器主机密钥。

```powershell
ssh -N -D 127.0.0.1:17897 USER@TAILNET_HOST
```

把以下配置保存为 YAML，在 FlClash 导入并选中。使用**规则模式**，开启系统代理，关闭 FlClash TUN。若 FlClash 的全局 DNS 覆写强制使用 Fake-IP，可改用 `redir-host` 后复测。

```yaml
mode: rule
allow-lan: false
proxies:
  - name: Tailnet-SOCKS
    type: socks5
    server: 127.0.0.1
    port: 17897
    udp: false
rule-providers:
  cn-domain:
    type: http
    behavior: domain
    format: mrs
    url: https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/cn.mrs
    path: ./ruleset/cn-domain.mrs
    proxy: Tailnet-SOCKS
    interval: 86400
  non-cn-domain:
    type: http
    behavior: domain
    format: mrs
    url: https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/geolocation-!cn.mrs
    path: ./ruleset/non-cn-domain.mrs
    proxy: Tailnet-SOCKS
    interval: 86400
  cn-ip:
    type: http
    behavior: ipcidr
    format: mrs
    url: https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geoip/cn.mrs
    path: ./ruleset/cn-ip.mrs
    proxy: Tailnet-SOCKS
    interval: 86400
rules:
  - RULE-SET,cn-domain,DIRECT
  - RULE-SET,non-cn-domain,Tailnet-SOCKS
  - RULE-SET,cn-ip,DIRECT,no-resolve
  - MATCH,Tailnet-SOCKS
```

分别验证大陆网站和境外网站的实际出口。这个例子只分流遵循系统代理的应用，SSH SOCKS 不承载 UDP。iPhone 原生 Tailscale 使用选中的出口节点，无需导入上面的 YAML。

[Mihomo 规则集文档](https://wiki.metacubex.one/en/config/rule-providers/) · [MetaCubeX 规则源](https://github.com/MetaCubeX/meta-rules-dat)

欢迎通过 Issue 或 Pull Request 补充其他系统的步骤或更正说明。提交示例时请用占位符代替真实 IP、账号、密钥和邀请地址。
