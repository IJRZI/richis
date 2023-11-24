# Auth THU

THU 宿舍校园网需要网页认证, 不过有同学开发了 [GoAuthing](https://github.com/z4yx/GoAuthing) 这个认证程序, 在常规的 Linux 下可以直接开 crontab/service 定时认证, 但 QNAP 的 crontab 似乎有点问题, 我的解决方案是开了一个 Docker 容器跑认证程序

Dockerfile如下
```
FROM alpine:latest

COPY ./auth-thu /usr/local/bin/auth-thu
COPY ./.auth-thu /root/.auth-thu

# 这里输出 log 是为了之后验证 auth-thu 是否正确执行以 debug, 如果不需要 debug 则使用:
# RUN echo '*/1 * * * * /usr/local/bin/auth-thu auth' > /etc/crontabs/root
RUN echo '*/1 * * * * /usr/local/bin/auth-thu auth >> /var/log/auth-thu.log 2>&1' > /etc/crontabs/root

CMD crond -l 2 -f
```

用了 alpine 这个非常小的 linux 镜像, 使用方式为
1. 创建 Dockerfile 并添加上面的内容
2. 在 Dockerfile 同目录下下载 GoAuthing 的可执行文件并改名为 auth-thu, 同时赋以可执行权限
3. 在 Dockerfile 同目录下创建 .auth-thu 文件, 填入 GoAuthing 的相关配置, 如用户名密码
4. 在该目录下用 Dockerfile 创建镜像, 例如 `docker create -t auththu:latest .`
5. 用创建好的镜像创建容器, 并设置其为自动重启(即随 docker 启动) `docker run -d --name auththu --net=host --restart=always`