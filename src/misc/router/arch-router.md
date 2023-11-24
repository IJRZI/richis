# Arch as Router
## 背景
硬件是倍控 G31(Intel Gold 7505)，本来想参考 PVE 跑一堆虚拟机的方案，但尝试过 PVE + openwrt + ikuai + debian(跑docker) 的组合之后，发现了一些问题：
1. 最大的硬伤是这个机子有不错的核显，但不能显卡直通，因此一旦采用 PVE 的方案，基本核显就浪费了
2. ikuai 做主路由的时候，IPv6 配置支持不全，如果用 openwrt 做主路由，又因为固件太多软件包太杂，导致配着配着就崩了（主要是因为不熟悉 openwrt），而且因为不熟悉，导致不是所有东西都 under control

理论上大部分 linux 都可以通过配置来当路由用，因为无非就是要拨号，转发，提供 DHCP/DNS 等服务罢了，区别就在于是不是开箱即用。但与其用一个开箱即用的、不太熟悉的 OS，不如用一个不开箱即用的、熟悉一点的 OS，至少出了问题知道怎么排查

看了一些文章之后，基本断定 Arch 是可以拿来做路由的，刚好自己也比较熟了，Arch Wiki 还有专门的 Router 页面，就决定用 Arch 了

## 安装系统
参考 [archlinux 基础安装](https://arch.icekylin.online/rookie/basic-install.html) 即可，我按照其推荐的做了 btrfs，内核选了 linux-zen

可以再配一下 ssh，之后总会用到的，但路由器可能对外暴露，所以安全问题需要自行注意

## 配置路由功能
> 主要参考 
> - [Arch Wiki Router](https://wiki.archlinux.org/title/router)
> - [Arch based home router](https://eldon.me/arch-linux-based-home-router/)
> - [Arch is the best router](https://yhndnzj.com/2021/08/13/arch-is-the-best-router/)
> - [用 Arch Linux 做软路由](https://nyac.at/posts/archlinux-router)

我有四张网卡，其中第一张用来当 WAN，其余的用来当 LAN，WAN 约定称为 extern0, LAN 约定称为 intern0,1,2

网络管理通过 systemd-networkd 进行，DHCP server 用 dnsmasq 提供，DNS server 用 dnsmasq + smartdns 提供，防火墙和网络转发(NAT)用 firewalld 提供

### 1. 重命名接口
> 参考 [Arch is the best router](https://yhndnzj.com/2021/08/13/arch-is-the-best-router/)
将网卡名字改成刚才约定的，之后配置方便一点

修改
```
# /etc/udev/rules.d/10-network.rules
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="aa:bb:cc:dd:ee:ff", NAME="extern0"
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="ff:ee:dd:cc:bb:aa", NAME="intern0"
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="ff:ee:dd:cc:bb:ab", NAME="intern1"
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="ff:ee:dd:cc:bb:ab", NAME="intern2"
```

重新加载
```bash
udevadm control --reload
udevadm trigger
```

然后 `ip l` 应该可以看到接口名

### 2. 配置各接口
#### extern
对于 extern0, 我们希望它能拨号或者从上游 DHCP 服务器获取 IP，由于目前用的校园网，所以按后者配置
```
# /etc/systemd/network/20-wired-external-dhcp.network
[Match]
Name=extern0

[Network]
DHCP=yes
IPv6AcceptRA=yes
IPv6PrivacyExtensions=yes
```

#### intern
对于 intern0,1,2，我们希望它们能被桥接到一起，这样使用任何一个接口都没有区别

```
# /etc/systemd/network/br_lan.netdev
[NetDev]
Name=br_lan
Kind=bridge
```

```
# /etc/systemd/network/10-bind-br_lan.network
[Match]
Name=intern*

[Network]
Bridge=br_lan
```

然后对于 `br_lan` 进行网络配置
```
# /etc/systemd/network/21-wired-internal.network
[Match]
Name=br_lan

[Link]
Multicast=yes

[Network]
Address=10.0.0.1/24 # router 的内网 IP 及网段
#MulticastDNS=yes # 打算用dnsmasq替代
#IPMasquerade=both # 如果启用，将会与 firewalld 冲突，因为它们都会修改 nftables
```

### 3. 配置 dnsmasq
安装
```bash
pacman -S dnsmasq
```

配置
```
# /etc/dnsmasq.conf
except-interface=extern0 # 排除extern0
expand-hosts      # 为 /etc/hosts 中的主机名添加一个域名
domain=foo.bar    # 允许DHCP主机的完全限定域名（需要启用“expand-hosts”）
dhcp-range=10.0.0.2,10.0.0.255,255.255.255.0,1h # 定义局域网中DHCP地址范围：从 10.0.0.2 至10.0.0.255，子网掩码为 255.255.255.0，DHCP 租期为 1 小时 （可按需修改）
port=0 # 禁用 dns 服务，如果不打算用 smartdns，可以将这个 port 设为默认的 53，然后添加诸如 server=8.8.8.8 的规则以指定 dnsmasq 的上游 DNS
dhcp-option=6,10.0.0.1 # 但是在 DHCP 时通告本机为 DNS server
# 设置默认网关
dhcp-option=3,10.0.0.1
```

启用
```bash
systemctl enable --now systemd-networkd
```

### 4. 配置 SmartDNS
参考 [SmartDNS](https://pymumu.github.io/smartdns/) 即可

### 5. 网络转发
先启用内核的网络转发（需要重启）：
```
# /etc/sysctl.d/30-ipforward.conf
net.ipv4.ip_forward=1
net.ipv6.ip_forward=1
```

然后安装 firewalld 并启用
```bash
pacman -S firewalld
systemctl enable --now firewalld
```

配置 NAT 规则 (参考 [Arch Wiki Internet Sharing](https://wiki.archlinux.org/title/Internet_sharing#With_firewalld))
```bash
firewall-cmd --zone=external --change-interface=extern0 --permanent
firewall-cmd --zone=internal --change-interface=br_lan --permanent

firewall-cmd --permanent --new-policy int2ext
firewall-cmd --permanent --policy int2ext --add-ingress-zone internal
firewall-cmd --permanent --policy int2ext --add-egress-zone external
firewall-cmd --permanent --policy int2ext --set-target ACCEPT

# 重要：wiki里没有手动允许 dns, 导致dnsmasq无法响应请求, 需要手动添加
# 		 可能还需要添加 dhcp 等，因为它默认连内网的包都过滤，所以如果发现内网一些服务不通，先检查 firewalld
firewall-cmd --add-service=dns --zone=internal --permanent
firewall-cmd --add-service=dhcp --zone=internal --permanent

firewall-cmd --reload
```

这一步做完之后如果没问题那就没问题了()，其他设备连网口应该能正常上网了，dnsmasq 会响应设备的 DHCP 请求并分配 IP，设备 DNS 会走 Arch 的 smartdns，网络包会由 firewalld (底层是 nftables) NAT

## 其他配置
### 1. Auththu
校园网需要认证，通过 goauthing 实现，参考 https://github.com/z4yx/GoAuthing

### 2. 备份
参考 [利用 Snapper 实现 btrfs 自动定时备份](https://kasei.im/2017/03/use-snapper-for-btrfs-snapshot-and-backup.html)
btrfs 可以用 snapper， grub-btrfs 可以从快照启动，方便恢复系统
```bash
pacman -S snapper snap-pac grub-btrfs
systemctl enable --now grub-btrfsd
grub-mkconfig -o /boot/grub/grub.cfg
```

配置不容易，多备份，每次要改配置前手动快照一下

### 3. BBR
一个 TCP 拥塞控制算法
参考 [Gist - Enabling BBR On Arch Linux 4.13+](https://gist.github.com/epyonavenger/a7d0bdcdb64169c4b0031391e10ff203)

### 4. DDNS
我是 aliyun 的域名 + aliyun 的 ddns，定时跑这个脚本就行 [Github - SNBQT/aliyunddns](https://github.com/SNBQT/aliyunddns)

### 5. 静态 IP 租约
查看当前租约
```bash
cat /var/lib/misc/dnsmasq.leases
```

分配
```
# /etc/dnsmasq.conf
# 如果要让 dnsmasq 将固定 IP 分配给某些客户端，请绑定 LAN 计算机的 NIC MAC 地址：
dhcp-host=aa:bb:cc:dd:ee:ff,192.168.111.50
dhcp-host=aa:bb:cc:ff:dd:ee,192.168.111.51
```

### 6. UPS
断电了通知关闭 router 防止意外断电

[Arch Wiki APC_UPS](https://wiki.archlinux.org/title/APC_UPS)

兼容山特的 UPS (串口-USB通信)

### 7. cron
Arch 默认不带 cron，可以安装 `cronie`

### 8. 其他防火墙配置
端口开放及转发
```bash
firewall-cmd --zone=external --add-port=2222/tcp --permanent
firewall-cmd --zone=external --add-forward-port=port=2222:proto=tcp:toport=22:toaddr=127.0.0.1 --permanent
```

### 9. qBittorrent
安装配置参考 [archlinux-install-qbittorrent-nox-setup-webui](https://blog.raw.pm/en/archlinux-install-qbittorrent-nox-setup-webui/)

### 10. Plex
安装可以用 snap

需要配置文件权限才可以添加资料库，见 [plex-wont-enter-my-home-directory-or-other-partitions](https://askubuntu.com/questions/150909/plex-wont-enter-my-home-directory-or-other-partitions)

### 11. Clash
见 [Arch Linux Clash 安装配置记录](https://blog.linioi.com/posts/clash-on-arch/)

## Trouble Shooting
遇到的最大问题就是，一开始 NAT 用了 networkd 自带的 IPMasquerade=both，然后想改成 firewalld，配置了很多遍都没成功，需要注意的点有：
- 先关闭 firewalld
- 关闭 networkd 的 IPMasquerade=both 之后，需要 restart networkd
- 需要 nft flush ruleset 以清空规则，networkd 会在 nft 里写入 io.systemd.nat 这个规则表
- 然后启动 firewalld 的服务，配置 zone 和 policy
- 记住启用 internal zone 的 DNS/DHCP 等 service

