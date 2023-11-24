# Arch 休眠到交换文件

参考 [Arch Wiki](https://wiki.archlinux.org/title/Power_management_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)/Suspend_and_hibernate_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)) 及 [Arch简明指南](https://arch.icekylin.online/advanced/optional-cfg-2.html#%E4%BC%91%E7%9C%A0%E5%88%B0-swap-%E5%88%86%E5%8C%BA) 配置系统休眠到 swap file(ext4)，配置完毕后无法正常休眠，问题如下

## 1. KDE 开始菜单不展示休眠选项

尝试手动休眠 `systemctl hibernate`，提示 "Not enough swap space for hibernation"

根据 [Arch BBS](https://bbs.archlinux.org/viewtopic.php?id=259382)，通过 `systemctl edit systemd-logind.service` 并在其中添加

```
# 注意添加位置，必须在文件中注明的两段注释之间，否则不会生效
[Service]       
Environment=SYSTEMD_BYPASS_HIBERNATION_MEMORY_CHECK=1
```

后重启 logind 服务 `systemctl restart systemd-logind  ` 即可

## 2. 休眠后立刻回到登录页面

休眠后查看日志 `journalctl -n 1000`，在其中查找 hibernate 相关记录，发现报了

```
Failed to find location to hibernate to: Function not implemented
```

怀疑是 hibernate 目标交换文件配置有误，检查后发现在获取交换文件 resume_offset 时，用 `sudo filefrag -v /swapfile` 命令查看的偏移如下：

```
Filesystem type is: ef53
File size of /swapfile is 34359738368 (8388608 blocks of 4096 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..    6143:    4114432..   4120575:   6144:            
   1:     6144..   38911:    3997696..   4030463:  32768:    4120576:
   2:    38912..   71679:    3506176..   3538943:  32768:    4030464:
   3:    71680..  104447:    8224768..   8257535:  32768:    3538944:
   ...
```

physical_offset列的第一个值应当是 4114432，而我配置成了 4120575，修改后重新生成 grub.cfg 即可
