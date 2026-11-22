# Tailscale 出口节点共享指南

Windows 和 iPhone 都可使用。每个人用自己的 Tailscale 账号。

## 怎么创建 Tailscale 出口节点（Linux VPS）

用自己的 Ubuntu 或其他 Linux VPS 搭建 **Tailscale Exit Node**，让 Windows 和 iPhone 通过 VPS 上网，再按需把出口节点共享给好友。

1. 按 [Tailscale Linux 安装文档](https://tailscale.com/docs/install/linux) 安装 Tailscale。在 VPS 上运行 `sudo tailscale up`，按提示登录并加入自己的 tailnet。
2. 使用 Linux 内核网络模式时，在 `/etc/sysctl.d/99-tailscale-exit-node.conf` 写入 `net.ipv4.ip_forward = 1` 和 `net.ipv6.conf.all.forwarding = 1`，运行 `sudo sysctl -p /etc/sysctl.d/99-tailscale-exit-node.conf` 应用配置。用户态网络模式由 Tailscale 自行转发。
3. 在 VPS 运行 `sudo tailscale set --advertise-exit-node`。打开 Tailscale 后台 **Machines → 你的 VPS → Edit route settings**，勾选 **Use as exit node** 并保存。
4. 在 Windows 或 iPhone 的 Tailscale 应用中选中该 **Exit Node**。打开网页，核对客户端出口 IP 与 VPS 公网出口是否一致。确认可用后，再按下文生成好友邀请。

[完整搭建教程](SETUP.md)包含安装命令、停用方法、直连与 DERP 检查，以及基于 FlClash/Mihomo 的 Windows 大陆分流示例。

## 好友怎么连接

1. 安装 Tailscale 并登录自己的账号。向节点所有者获取私人邀请，接受后即可看到共享节点。
2. Windows 点击系统托盘的 Tailscale 图标，在 **Exit nodes** 选择共享节点。iPhone 打开 Tailscale 应用，在 **Exit Node** 选择共享节点。
3. 如果另一款代理或 VPN 正在接管流量，先暂停它。打开网页确认连接可用。
4. 用完后，把 **Exit Node** 设为 **None**，恢复原来的互联网出口。

选择出口节点后，互联网流量默认通过共享节点。仍需访问本地局域网时，可开启 **Allow LAN access**。

## 节点所有者怎么分享

1. 在 Tailscale 管理后台确认出口节点已批准并在线。
2. 在 **Machines** 找到设备，选择 **Share**，勾选 **Allow use as an exit node**，把邀请单独发给好友。
3. 核对访问策略，确认共享对象只获得所需权限。邀请链接与管理凭据应私下保管。

- [Tailscale 官方下载](https://tailscale.com/download)
- [使用出口节点](https://tailscale.com/docs/features/exit-nodes/how-to/setup)
- [共享设备与出口节点](https://tailscale.com/docs/features/sharing)
