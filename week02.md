# W2 · 进程、信号与 systemd

## 我做了什么

把 `~/bin/heartbeat.sh`（每 10 秒记录内存和负载）托管成 systemd 服务，
并用四组实验验证了 Restart 策略、信号语义、enable/start 的区别。

## 核心结论

### 1. unit 文件三段的分工

- `[Unit]`：描述 + 启动顺序（`After=` / `Requires=`）
- `[Service]`：怎么跑。`ExecStart` **必须绝对路径**；`Restart` 决定退出后是否拉起；`StandardOutput=journal` 让日志进 journald
- `[Install]`：**只影响 `enable`**，决定挂到哪个 target

### 2. enable 和 start 是两个维度

| 命令 | 管什么 | 影响当前进程 |
|---|---|---|
| start / stop / restart | 本次运行 | 是 |
| enable / disable | 下次开机 | 否 |

实测：`disable` 之后 `is-enabled=disabled`，但 `is-active=active`。

`enable` 的物理实现 = 在 `/etc/systemd/system/multi-user.target.wants/` 下建符号链接。

### 3. Restart 策略

- `no`：不重启
- `on-failure`：异常退出才重启（多数服务的最佳选择）
- `always`：任何退出都重启

实测：`kill` 之后 `Restart=always` 会在 `RestartSec` 秒后拉起，`NRestarts` +1，
journal 记录 `Scheduled restart job, restart counter is at N`。

### 4. SIGTERM 优先于 SIGKILL

- SIGTERM(15)：可被 `trap` 捕获，程序能清理资源 → systemd 记为 `Deactivated successfully`
- SIGKILL(9)：不可捕获，内核强杀 → systemd 记为 `Failed with result 'signal'`，`status=9/KILL`

实测：脚本里加 `trap cleanup TERM` 后，SIGTERM 时能看到两行 `[trap]` 日志；SIGKILL 时一行都没有。

正确顺序：`kill`（TERM）→ 等几秒 → 还不退再 `kill -9`。
生产系统的关机流程 = `systemctl stop` → 等 `TimeoutStopSec`（默认 90 秒）→ 补 SIGKILL。

### 5. CrashLoopBackOff 的原型

`Restart=always` + 启动即崩 → restart counter 一直涨 → 超过 `StartLimitBurst` 就进入
`start-limit-hit`，不再重启。K8s 的 CrashLoopBackOff 就是这套逻辑的分布式版本。

## 踩坑

### journalctl 报 No entries

不是没日志，是权限问题。服务以 root 运行，日志属于系统日志，
普通用户需要 `adm` 或 `systemd-journal` 组才能读。

解决：`sudo usermod -aG adm mxy`，然后重启 WSL 会话（`woff` + `wr`）让组生效。

排查顺序：先 `sudo journalctl -u xxx` 确认日志存在，再改权限。

## 常用命令速查

- `systemctl status <svc> --no-pager`
- `systemctl start|stop|restart <svc>`
- `systemctl enable --now <svc>`
- `systemctl is-enabled <svc>` / `systemctl is-active <svc>`
- `systemctl show <svc> -p Restart -p NRestarts`
- `journalctl -u <svc> -n 20 --no-pager`
- `journalctl -u <svc> -f`
- `journalctl -u <svc> --since "10 min ago" --no-pager`
- `systemd-analyze verify /etc/systemd/system/<svc>.service`