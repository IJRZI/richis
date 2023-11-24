# i3wm 切换到 sway

## 复制配置
```
mkdir -p ~/.config/sway
cp ~/.config/i3/config ~/.config/sway/
```

## 重映射 CapsLock 到 Ctrl, 修改键盘重复速率

原来用的是 `setxkbmap`, 但这是针对 xorg 的

映射方法为在 sway config 中加入
```
input "type:keyboard" {
    xkb_options caps:ctrl_modifier
    repeat_delay 150
    repeat_rate 80
}
```

## Deepin Wine Wechat 黑屏
见 [Wechat](./wechat.md)

## 其他
- 使用 wofi 替换 rofi
- 使用 waybar 替换 polybar
- 使用 swaybg 替换 feh