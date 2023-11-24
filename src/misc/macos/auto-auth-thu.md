# 自动认证 Tsinghua WiFi

> 在 Mac 连接到 Tsinghua/Tsinghua-5G 等需要认证的无线网时自动认证而无需经 Web 认证

## 0. 准备

下载
- [WifiWatch.app](https://github.com/p2/WifiWatch/releases/tag/1.0.0)
- [auth-thu.macos.arm64](https://github.com/z4yx/GoAuthing/releases/tag/v2.2)
    - arm64 或 x86_64 看情况, arm64 为例


## 1. auth-thu

用于从命令行认证校园网

- 给予可执行权限: `chmod +x auth-thu.macos.arm64`
- 放到 $PATH 下并重命名为 `auth-thu`: `cp auth-thu.macos.arm64 /usr/local/bin/auth-thu`
- 在 HOME 目录下创建其配置: `touch ~/.auth-thu`
- 在配置内写入校园网账号密码 (这里只能明文): `vim ~/.auth-thu`
    ```json
    {
        "username": "username",
        "password": "password"
    }
    ```
- 尝试认证看正不正常
    ```
    > auth-thu
    2022-11-23 17:08:41 INFO auth-thu main.go:308 Currently online!
    ```

## 2. WifiWatch.app

直接运行 .app 即可，这个程序会在后台，当连接/断开 WiFi 时执行特定脚本
- 连接时执行 `~/.wifiConnected`
- 断开时执行 `~/.wifiDisconnected` (本示例用不到)

可以在系统设置里把这个 .app 加到开机启动项

## 3. .wifiConnected

连接 WiFi 时执行的脚本

写入如下内容 `vim ~/.wifiConnected`
```bash
#!/bin/bash
# arg1: SSID of network
# arg2: SSID of old network, if any

log=/tmp/auth-thu
if [[ "$1" =~ ^(Tsinghua|Tsinghua-5G)$ ]]; then
	for i in 4 3 2 1; do
		sleep $i
		connected=$(/usr/local/bin/auth-thu 2>&1 | tee -a $log | grep -E "online|Successfully" && echo 0 || echo 1)
		if [[ $connected == 0 ]]; then
			sleep $i
			/usr/local/bin/auth-thu 2>&1 | tee -a $log
		else
			break
		fi
	done
fi
```

然后给予可执行权限 `chmod g+x ~/.wifiConnected`

## 4. 尝试

- 退出认证
- 断开 WiFi 然后连接 Tsinghua/Tsinghua-5G
- 检查 /tmp/auth-thu 里是否有 log
- 检查 WiFi 是否正常认证

## 5. Trouble Shooting

WiFi 名目前是字符串匹配且只有 Tsinghua/Tsinghua-5G，如果需要其他的，加在 `.wifiConnected` 里

## 6. Reference
- https://apple.stackexchange.com/questions/139267/run-program-if-connected-to-specific-wifi
- https://github.com/p2/WifiWatch
- https://github.com/z4yx/GoAuthing