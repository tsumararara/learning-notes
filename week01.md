# W1 · 环境配置

## 做了什么
- WSL2 + Ubuntu 24.04，装在 E 盘
- apt 换清华源
- 装了 30 个命令行工具
- zsh + oh-my-zsh + tmux
- Git + SSH 打通

## 踩过的坑
- `wsl --install` 报 403 → 改用 msixbundle 手动安装
- `ssh.github.com:443` 连接超时 → 改用 `github.com:22`

## VS Code + Remote-WSL
- VS Code 1.141.2，winget 安装必须加 `--source winget`（msstore 源 403）
- 扩展：`ms-vscode-remote.remote-wsl`
- 验证：WSL 里 `code .`，左下角显示 `WSL: Ubuntu-24.04`

## 坑：WSL 的 DNS 代理不稳定
- 现象：`ssh -T git@github.com` 报 `Temporary failure in name resolution`
- 排查：`dig github.com +short` 超时两次才出结果；`dig @223.5.5.5 github.com +short` 立即返回
- 根因：`/etc/resolv.conf` 指向 `10.255.255.254`（WSL 虚拟 DNS 代理），转发到 Windows DNS 时超时
- 修复：`/etc/wsl.conf` 加 `[network] generateResolvConf = false`，手写 `223.5.5.5` / `119.29.29.29`，`chattr +i` 防覆盖，`wsl --shutdown` 重启生效
