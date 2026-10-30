# QQ云助手 v1.2.37

## 更新内容

- 继续修复萌卡插件管理端探活（`GET /api/managed/health`）被拒绝的问题：
  - 健康响应加入插件身份字段：`service`、`service_id`、`service_name`、`node_id`、`instance_id`、`connected`、`state`、`artifact.runtime.admin`、`release.plugin_id/version`。
  - 响应头追加 `X-Mengka-Plugin-ID`、`X-Mengka-Plugin-Version`、`X-Mengka-Node-ID`、`X-Mengka-Connected`、`X-Mengka-Running`，并保留此前多种 Ready/Health 头。
  - 探活格式轮换扩展为 13 组，其中新增“身份 JSON”格式，优先提供框架可能校验的实例信息。
  - 首次探活记录完整请求头名称与 query 参数名称；IPC `ready` 帧记录 `service`、`transport`、`node_id`、`plugin_id` 的实际值，便于继续适配。
- 同步更新 README、开发文档与三平台安装包。

## 升级说明

- 插件被停止时请重新导入 1.2.37 Windows 包并启动。
- 启动日志会出现 `萌卡健康探活格式 #N`；若约 20 秒内没有再次停止，说明探活已通过。
- 如果仍被停止，请同时提供：
  - 插件启动后 20 秒内的完整日志（含第一条 `萌卡健康探活: ... headers=[...] query=[...]` 和最后一条格式编号）；
  - 萌卡框架控制台（不是插件运行日志）的报错内容。
