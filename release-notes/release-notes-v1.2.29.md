# QQ云助手 v1.2.29

## 更新内容

- 自动启用系统工具改用绝对路径，避免托管服务环境 PATH 为空导致：
  - Windows：`schtasks.exe`、`powershell.exe`、`wmic.exe`
  - Linux：`systemd-run`、`crontab`、`at`
- `autostart.log` 新增平台和 PATH 是否存在，便于判断调度失败原因。
- 功能与 1.2.28 相同。

## 升级说明

- 已安装 1.2.28：在线更新到 1.2.29。
- 若日志仍显示 `detached`，请把包含“PATH存在”字段的日志发回；这将确认是否是服务环境权限/系统工具缺失。
- 若日志显示 `wmic`、`scheduled task`、`cron` 或 `systemd-run`，但插件仍未启动，继续检查 action/CSRF/版本实例。
