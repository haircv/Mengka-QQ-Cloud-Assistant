# QQ云助手 v1.3.3

## 更新内容

- **修复在线更新「读取萌卡服务节点失败：萌卡没有返回有效的管理接口响应」**：萌卡 3.0.6 起统一为单机模式，已移除服务节点管理。节点接口返回 404 或非 `{code,data}` 信封（前端兜底页）时按无节点继续，托管导入不再提交 `node_id`；旧版多节点框架仍自动选择单节点，多节点时按填写的节点 ID 提交。
- **加速助手界面同步萌卡管理端**：日常任务 / 额外任务 / 会员签到三页签，列表展示全部面板任务（含「开通QQ会员」等不可操作项，1.3.2「控制台只显示可操作项」的口径作废）。左侧勾选框用于「补挂选中」手动执行，右侧开关为自动执行选择并即时保存，「开启全部」一键开启当前页签自动执行；会员签到横幅说明成长值归属对应会员、不计入 QQ 加速天数。「最近执行结果」由表格改为列表（成功/失败徽标 + 红色错误详情），数据来自框架 `state.results` / `state.finished_at`，包含框架定时调度发起的执行。
- **加速天数与图标取值修正**：天数优先 `finished_accelerate_days`（本身已是天数），其次 `speed_days` ÷ 10（0.1 天单位），再兜底副标题「X天」文案；图标按 `icon_url` 关键词归类映射本地 FontAwesome（控制台 CSP `img-src 'self' data:` 不允许外链图片），标题关键词兜底。
- **面板数据源对齐官方接口**：`level_task_management`（operation=panel）的日常/额外任务取 `payload.base_info.base_task_list` / `payload.extra_info.extra_task_list`，存在 `state.supported_tasks` 时只保留支持项；会员签到取 `state.membership_tasks`（含 `membership_group`、`status_text`、`execution_message`，旧版框架回退 `payload.member_info.member_task_list`）；`payload.vip_info.vip_list` 为付费开通推广项，不纳入面板；调度取 `state.settings`（`selected_tasks / schedule_enabled / schedule_time / last_run_date`）。
- **指令回复同步**：`查询加速` 改读同一 Overview（含框架执行记录），任务天数与图标取值口径一致，会员签到展示成长值与归属会员，执行失败标 `[×]`，`failed / unknown` 计入失败统计。指令回复与自动补挂仍按 `Actionable()` 过滤必然失败项；已完成任务 `can_execute=false` 仅表示无需再执行，仍展示并允许开关选择。
- **控制台移动端自适应**：窄屏改为紧凑品牌头部 + 底部导航（概览/账号/加速/红包/日志/更多），账号与群聊选择移到头部下方独立一行；卡片网格、统计条、页头与操作面板按 Tailwind 响应式断点重排，表格与长文本不再横向溢出。
- 指令数量（399）、控制台模块（61）、清单 action（59）、权限体系与既有数据均不变。

## 升级说明

- 已安装 1.3.2 及更早版本可在线更新或手动导入 1.3.3；1.3.0 无法通过萌卡导入校验，请直接导入 1.3.3。
- 已有数据目录的配置、卡密、红包账本与农场/加速设置均保留。
- 萌卡 3.0.6 单机模式下在线更新无需填写服务节点；旧版多节点框架存在多个节点时仍需填写节点 ID。
- 加速助手仅 Android 协议账号可用；Free 会员不可用、Plus 要求 QQ 等级 ≥ 32。
- 控制台为纯静态构建产物，更新后若浏览器缓存旧页面，请强制刷新一次（Ctrl+F5）。