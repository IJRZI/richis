# APP 持续 deploying 问题

最开始是 Plex 的 TrueCharts 版部署持续在 Deploying 状态, 以为是这个 APP 本身的问题, 后来发现 nextcloud 的 TC 版也有一样问题

最后通过检查 k3s 的 pod 状态发现, 这两个 APP 部署时均需要创建 PVC, 而创建 PVC 依赖容器的镜像一直 ImagePullBackOff, describe 查看日志后发现是网络问题 (k8s.gcr.io 不通)

在路由上增加代理之后可以解决, 另一个成功的方法是, 局域网设备开 clash(allow lan) 或者之类的代理, Truenas 网络配置 http proxy (http://addr:port), 然后重新创建 APP, 在拉镜像时可能仍然会失败, 但这时只要手动查看是哪个域名的镜像没拉成功 (以 k8s.gcr.io 为例), 然后在终端 http_proxy=http://addr:port curl k8s.gcr.io 即可