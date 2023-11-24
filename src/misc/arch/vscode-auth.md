# vscode 无法记住登录

装了 `visual-studio-code-bin`, 发现每次启动一个新的实例时都要求登录以同步设置, 即无法记住其他实例的登录

最后参考 [Settings Sync in Visual Studio Code](https://code.visualstudio.com/docs/editor/settings-sync#_linux) 解决了

Linux 上的 VSCode 依赖 gnome-keyring 来保存认证信息, 所以需要正确安装并配置 gnome-keyring, 我之前尝试过装 gnome-keyring 但最后还是没解决, 可能是因为当时装了另一个 keyring 包, 二者冲突了

### 具体步骤

1. 在 `~/.xinitrc` 里添加如下行
    ```bash
    # see https://unix.stackexchange.com/a/295652/332452
    source /etc/X11/xinit/xinitrc.d/50-systemd-user.sh

    # see https://wiki.archlinux.org/title/GNOME/Keyring#xinitrc
    eval $(/usr/bin/gnome-keyring-daemon --start)
    export SSH_AUTH_SOCK

    # see https://github.com/NixOS/nixpkgs/issues/14966#issuecomment-520083836
    mkdir -p "$HOME"/.local/share/keyrings
    ```
    
    由于我在用 swaywm, 所以我把上面这些行加到了 sway 的 config 里, 最后也能 work

2. 重新登录以加载 1. 的配置
3. 安装 gnome-keyring etc., `sudo pacman -S gnome-keyring libsecret libgnome-keyring`
4. 使用任何 gnome-keyring 的管理手段 (比如 `seahorse`), 解锁默认的 keyring 或者创建一个新的没上锁的 keyring. 在我的尝试里, 这一步如果不做, 启动 vscode 并登录账号时会提示新建 keyring, 新建时不要设置密码即可
5. 启动 vscode