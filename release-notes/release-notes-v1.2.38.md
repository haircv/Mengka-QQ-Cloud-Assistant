# QQ云助手 v1.2.38

## 更新内容

- 使用萌卡 ready 帧中的真实身份修复 `/api/managed/health`：
  - 现场 ready 帧为 `service=QQ云助手`、`transport=native-ipc-v1`、`plugin_id=qq-cloud-assistant`；1.2.37 的身份 JSON 中 `service` 为空，可能导致框架校验失败。
  - 现在保存 ready 帧的 `service`、`transport`、`plugin_id`、`api_version`、`framework_api`、`node_id`，并在健康响应的身份 JSON 中按原样返回（包含 `type:"ready"` 和原始 node_id JSON 类型）。
  - 宽 JSON 同步使用真实身份；响应头继续包含 Plugin-ID/Version/Node-ID/Connected/Running。
- 健康探活仍支持 13 组格式轮换，日志记录 `萌卡健康探活格式 #N`。
- 同步更新 README、开发文档与三平台安装包。

## 升级说明

- 重新导入 1.2.38 Windows 包并启动。
- 日志应出现 `萌卡健康探活格式 #1: json identity`；若框架在格式 #1 后停止轮询，说明身份校验通过。
- 如果仍在约 20 秒后被停止，请提供启动后完整日志及萌卡框架控制台报错。
