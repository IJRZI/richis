# Plex
针对 Official 的 Plex App 的配置备忘

- 生成 claim token
- use plex pass
- 在 Truenas 里创建 plex 用户组和 plex 用户，给予他们媒体文件夹的权限，同时给予 apps 用户权限
- Environment Variables for Plex 里添加 PLEX_UID 和 PLEX_GID，值为 Truenas 里对应用户和组的 ID
- Plex Extra Host Path Volumes 里挂载目标文件夹