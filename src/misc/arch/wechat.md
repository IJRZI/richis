# Wechat

使用打包好的 [Deepin Wine Wechat Arch](https://github.com/vufa/deepin-wine-wechat-arch)，仓库中已经给出了详细的安装方法、字体更换等

## Sway
从 i3 迁移到 sway 后, Deepin Wine Wechat 的微信窗口黑屏, 暂时没找到解决办法, 改用 wine-for-wechat 后解决

安装流程
1. 安装 wine-for-wechat, wine-wechat 包, 均在 archlinuxcn 源上
2. 微信官网下载 `.exe` 安装包
3. 命令行 `wechat -i /path/to/wechat_setup.exe` 安装微信
4. 安装完成后就能使用, 如果字体有问题, 参考 [Ubuntu20.04 Wine 6.0 微信中文显示方块/方框](https://gist.github.com/qin-yu/bfd799f2380c875045e7c8b918d02f36), 本质上为如下几步
  - winetricks 安装所有字体和所需 dll
  - 改注册表

不过仍然没解决 emoji 为方块的问题

## Trouble shotting
### 窗口阴影
换到 i3wm 后，混成器使用了 picom，此时使用 Deepin Wine Wechat 会发现整个窗口被灰色遮罩，且有弧形的黑色阴影，关闭 picom 后上述情况消失，因此判断是 picom 的问题

查阅 arch wiki 上 picom 条目发现，picom 可以针对窗口禁用半透明、阴影等特性，其规则可以细化到匹配窗口名称

首先通过 `xprop` 查询微信的窗口名:
```
# xprop
WM_NAME(STRING) = "WeChat"
...
WM_CLASS(STRING) = "wechat.exe", "Wine"
```

然后在 `picom.conf` 里添加如下规则: 
```conf
~/.config/picom/picom.conf
# Specify a list of conditions of windows that should have no shadow.
#
# examples:
#   shadow-exclude = "n:e:Notification";
#
# shadow-exclude = []
shadow-exclude = [
  # ...
  "name = 'WeChat'",
  "class_g = 'wechat.exe'",
  "class_g = 'Wine'"
];
```

重启 picom 即可

### 字体发虚
打开 wine 设置
```
# /opt/apps/com.qq.weixin.deepin/files/run.sh winecfg
```
graphics 选项卡里调高DPI即可

### 部分 Emoji 为方块
尚未解决