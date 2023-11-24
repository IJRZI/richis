# QNAP Crontab 使用

- crontab 配置文件在 `/etc/config/crontab`
- 应用修改 `# crontab /etc/config/crontab`
- 重启服务 `# /etc/init.d/crond.sh restart`

注意定时任务所用的脚本不能放在根目录下，因为 QNAP 每次重启都会重置根目录，建议放在 `/share/homes/<username>/` 下