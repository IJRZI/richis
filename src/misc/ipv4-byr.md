# 实现校外 IPv4 访问北邮人
为之后毕业离校做打算，用 SSR + 海外 IPv6 服务器做代理访问 byr

## 搭建服务器
1. Vultr 买个最便宜的机器，为了稳定系统选了 CentOS 7，记得 enable ipv6
2. 参考 [SSR一键安装](https://ebvtech.gitbook.io/wiki/ssr-basic/ssr-one-click-install) 安装  ssr 服务端
    ```bash
    wget --no-check-certificate -O shadowsocks-all.sh https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks-all.sh
    chmod +x shadowsocks-all.sh
    ./shadowsocks-all.sh 2>&1 | tee shadowsocks-all.log
    ```
    
    配置选择
    ```
    Your Protocol         :  auth_aes128_sha1
    Your obfs             :  tls1.2_ticket_auth
    Your Encryption Method:  aes-256-cfb
    ```

    相关命令
    ```
    启动SSR：
    /etc/init.d/shadowsocks-r start
    退出SSR：
    /etc/init.d/shadowsocks-r stop
    重启SSR：
    /etc/init.d/shadowsocks-r restart
    SSR状态：
    /etc/init.d/shadowsocks-r status
    卸载SSR：
    ./shadowsocks-all.sh uninstall
    ```

    配置文件: /etc/shadowsocks-r/config.json，可以检查一下里面有没有 `server_ipv6`，以及需要将 `dns_ipv6` 改为 `true`
## 客户端
1. 安装 ssr 客户端
    - 对于 osx，可以用 `brew install shadowsocksx-ng-r --cask`，配置连接
2. 在 ssr 客户端里添加 byr 的规则以对相关域名使用代理

## 下载
参考 [如何在普通网络环境下上北邮人](https://www.codewoody.com/posts/54288/)

以 qBittorrent 为例，需要配置
1. Connection - Proxy 填入 ssr 的 local 代理地址/端口
2. 关闭 Advanced 里的验证 tracker 证书

## Reference
- [在Vultr上搭建SSR服务](https://lccurious.github.io/2018/03/03/vlutr-ssr/)
- [实现校园网IPv6免流量上网与科学上网 | 安装SSR与开启BBR加速](https://www.jixing.one/vps/ssr-bbr/)
- [Byr 校外代理下载](https://byr.pt/forums.php?action=viewtopic&forumid=9&topicid=10862)
    - 这个帖子是 byr 内部的，说是需要将代理改为 http 而非 socks5 才能下载有速度