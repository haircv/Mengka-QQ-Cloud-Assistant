# QQ云助手 v1.2.27

## 更新内容

- Windows 自动启用增加 `wmic process call create` 兜底，顺序为：
  1. Windows 任务计划
  2. WMI `Invoke-CimMethod`
  3. `wmic`
  4. 独立进程
- 启动助手日志新增平台和启动方式，例如：
  - `启动助手已提交（scheduled task，平台 windows）`
  - `启动助手已提交（wmi，平台 windows）`
  - `启动助手已提交（cron，平台 linux）`
  - `启动助手已提交（detached，平台 windows/linux）`
- `detached` 会额外记录“系统调度不可用”的提示，便于判断是否被萌卡进程一起终止。
- 保留 Linux systemd-run/cron/at、30 天免登录、下载重试、更新后页面自动刷新。

## 升级说明

- 已安装 1.2.26：在线更新到 1.2.27。
- 1.2.26 的助手可能仍使用旧调度方式；本次更新后如仍未自动启用，请手动启动一次 1.2.27，之后更新使用新的双平台兜底。
- 如日志仍显示 `detached`，说明该平台的任务计划/WMI/wmic 或 systemd/cron/at 均不可用；请把带“平台”的日志发回继续定位。
