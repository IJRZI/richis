# LEDE 旁路由

## 背景
PS5 想翻墙, 想在 nas 里装软路由然后装个 clash 给 PS5 当旁路由

## 步骤
### 安装 LEDE
1. 参考 [威联通Docker教程 篇十：威联通NAS安装LEDE软路由，保姆级教程，手把手教您虚拟机安装openwrt旁路由](https://post.smzdm.com/p/amx3023k/) 安装 lede, 这一步没什么坑
2. 装好后, 由于 453bmini 有两块网卡, 第一块网卡我直接连了校园网做主网关, 第二块网卡连路由器给局域网用, 因此需要创建一个连接第二块网卡和 lede 虚拟机接口的虚拟交换机, 然后在 virtualization station 里配置虚拟机的网络为该虚拟交换机

### 安装 clash
1. 使用的 clash 版本为 [Koolshare-Clash-hack](https://github.com/houzi-/Koolshare-Clash-hack), 下载 release 的 tar 包用 lede 自带的离线安装
2. 直接离线安装会提示非法关键字, 无法安装, 需要用[这里](https://zhuanlan.zhihu.com/p/402240863)提到的 hack 禁用关键字扫描规则, 核心是 ssh 上 lede 然后跑这个命令
    ```
    sed -i 's/\tdetect_package/\t# detect_package/g' /koolshare/scripts/ks_tar_install.sh
    ```

### 使用 clash
1. 配置订阅链接, 启用 clash
2. 需要在 clash 里开启需要走代理的设备的 IP
3. 在 PS5 里修改网关为 lede 的 IP