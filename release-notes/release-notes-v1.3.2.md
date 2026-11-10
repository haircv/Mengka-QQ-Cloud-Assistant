# QQ云助手 v1.3.2

## 更新内容

- **修复加速助手面板报错「加速助手不可用：level_task_management panel:」**：等级任务执行器仅支持 Android，同 QQ 存在 Linux/PC 实例时 `protocol_id=0` 会路由到错误实例导致 panel 调用失败。新增 `AndroidKey`/`DefaultKey` 解析安卓实例完整账号键（含真实 `protocol_id`，优先在线实例），控制台与指令统一改用安卓键；控制台读取账号前先刷新账号快照。
- **加速助手指令回复只呈现可操作项**：新增 `Task.Actionable()` 判定，过滤未开通会员、无法领取奖励、未建立师徒等必然失败项；回复按日常/额外/会员签到分组，会员签到展示成长值而非天数；任务开关与选择均按可操作项索引，不再列出「开通QQ会员」等无法操作的项。控制台加速助手页面同步只显示可操作任务。
- **控制台 UI 全部重写为 Vue 3 + Tailwind**：61 个模块全部迁移，按 `components / views / api / stores / utils` 目录分工，不再用单个 index.html 堆砌。Vite 预编译 SFC（无 eval）、Vue 与 FontAwesome 本地打包、Tailwind 编译为静态 CSS，满足严格 CSP（无 CDN、无 unsafe-eval），仍由 Go embed 离线服务；支持 8 种主题色切换与响应式布局。前端构建产物已提交到 `internal/webadmin/static/`，Go 三平台构建无需 Node。
- 指令数量（399）、控制台模块（61）、清单 action（59）、权限体系与既有数据均不变。

## 升级说明

- 已安装 1.3.1 及更早版本可在线更新或手动导入 1.3.2；1.3.0 无法通过萌卡导入校验，请直接导入 1.3.2。
- 已有数据目录的配置、卡密、红包账本与农场/加速设置均保留。
- 加速助手仅 Android 协议账号可用；Free 会员不可用、Plus 要求 QQ 等级 ≥ 32。
- 控制台为纯静态构建产物，更新后若浏览器缓存旧页面，请强制刷新一次（Ctrl+F5）。