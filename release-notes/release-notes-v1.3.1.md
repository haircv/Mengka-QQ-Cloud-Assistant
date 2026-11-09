# QQ云助手 v1.3.1

## 更新内容

- 修复萌卡导入拦截「插件声明的 Action get_qq_farm_code 不属于当前 API」：以官方 API 索引（mengka.net/nt/api/）逐一核对清单，移除全部 12 个当前 API 不存在的 action 与 1 个未使用声明，清单 actions 71 → **59**，全部声明均可在官方文档查证。
- 农场助手：取码只用官方 `get_qq_classic_farm_code`（Android/Linux 均支持，框架按账号 `protocol_id` 路由协议，无需额外参数）；不再调用不存在的 `get_qq_farm_code`。
- 加速助手改用官方接口：
  - 任务面板与调度设置：`level_task_management`（operation=panel/settings/run），框架调度仍是唯一数据源。
  - 等级与加速总览：`get_level_tasks`（overall_info 的 0.1 天/倍率单位已换算）。
  - 执行结果以官方 `results` 中 `status=confirmed` 为确认完成；run 最长 15 分钟，插件与控制台超时放宽到 16–17 分钟。
  - 电脑/手机在线状态改由账号快照判断；`电脑登录` 直接提交电脑在线任务，由框架自动匹配或创建同 QQ 的 Linux 实例。
  - 移除不存在的 `get_level_task_panel`、`get_level_task_status`、`get_level_task_settings`、`update_level_task_settings`、`execute_level_task_selection`、`get_level_task_accounts`、`get_level_task_account`、`get_online_clients`、`set_online_status`、`stop_account_login`、`get_login_info` 调用与声明。
- 指令数量（399）、控制台模块（61）、权限体系与既有数据均不变。

## 升级说明

- **1.3.0 无法通过萌卡导入校验**（清单含 `get_qq_farm_code`），请直接导入 1.3.1；已安装 1.2.51 及更早版本可在线更新或手动导入 1.3.1。
- 已有数据目录的配置、卡密、红包账本与农场/加速设置均保留。
- 加速助手仅 Android 协议账号可用；Free 会员不可用、Plus 要求 QQ 等级 ≥ 32。