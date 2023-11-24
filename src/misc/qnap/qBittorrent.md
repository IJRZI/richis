# qBittorrent

## 安装
1. App Center 添加软件源 `https://www.qnapclub.eu/repo.xml`
2. 搜索 qBittorrent 安装

## 配置
### 账号
默认端口 6363，默认账号 admin，默认密码 adminadmin

进 设置-WebUI 修改端口、账号密码、语言等为想要的

### HTTPS
在 设置-WebUI 选择使用 HTTPS 而不是 HTTP，证书和密钥设为域名证书的 `.pem` 和 `.key` 文件的绝对路径

### 下载排队
在 设置-BitTorrent 启用或关闭 Torrent 排队和做种限制

### 校园网 IPv6 (北邮人 PT)
- 设置-高级-qBittorrent 相关
    - **网络接口** 改为有校园网 IPv6 的接口
    - **绑定到可选的 IP 地址** 改为 所有 IPv6 地址
    - 如果 Tracker 显示 SSL 证书错误，则取消勾选 **验证 HTTPS tracker 证书**
