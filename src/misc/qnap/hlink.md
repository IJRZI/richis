# QNAP Hlink Docker 使用

## 环境

- QNAP 453B mini，事先安装好 Container Station，配置 SSH 登录
- 我的 qBittorrent 下载文件均在 `/share/Download` 目录下，希望将里面的文件硬链到 `/share/Media`
    - 硬链目录不能跨盘，`Download` 和 `Media` 两个共享文件夹均在同一块硬盘同一个存储池上

## 部署

直接用 Container Station 部署时，hlink 会报 `必须指定配置文件` 的错，原因不明，因此这里用命令行部署

ssh 到 nas:
```bash
docker run -d --name hlink \
    -e HLINK_HOME=/share/Container/Docker/Hlink \
    -p 9090:9090 \
    -v /share:/share \
    likun7981/hlink:latest
```

将 nas 的 `/share` 挂载到 container 的 `/share` 下，同时指定 `/share/Container/Docker/Hlink` 为 hlink 的家目录。nas 的 9090 端口映射为容器的 9090 端口，即 hlink 的 WebUI

然后访问 :9090，创建配置文件和定时计划即可

## Trouble shooting

目前（2022-10-17）hlink 的 WebUI 不支持账号功能，即一旦暴露到公网，任何人都可以访问该 WebUI，而我的 nas 直连校园网，自带公网 IP，因此需要考虑安全问题

暂时的解决办法是，考虑到一旦配置好 hlink 服务，便不太需要访问其 WebUI，因此可以在完成配置后用 iptable ban 掉 WebUI 的端口
```bash
#!/bin/bash

number=`iptables -t nat --line-numbers --numeric --list | grep dpt:9090 | awk '{print $1}'`
if [ -n "$number" ]; then
echo $number
iptables -t nat -D DOCKER $number
fi
```
qnap 的 iptable 规则每次重启后都会失效，因此可以将上述脚本加入 [crontab](./crontab.md)
