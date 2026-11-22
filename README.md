# Tailscale 出口节点共享指南

Windows 和 iPhone 都可使用。每个人用自己的 Tailscale 账号。

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

