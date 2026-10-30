# QQ云助手 v1.2.39

## 更新内容

- 依据萌卡官方 Node SDK 的 IPC `ready` 校验规则重新实现管理端健康响应：
  - 官方要求 `ready.node_id` 必须是安全整数，并与连接文件 `node_id` 一致；此前扩展 JSON 没有按 ready 原值返回。
  - `/api/managed/health` 新增“最小 ready JSON”格式，仅返回 `type/api_version/framework_api/plugin_id/service/transport/node_id`，字段名和值与 IPC ready 帧一致。
  - 保留身份 JSON、扩展 JSON、纯文本、204 等兼容格式轮换；最小 ready JSON 位于第 1 组，ready 帧到达后下一轮即可使用真实 `service=QQ云助手`、`transport=native-ipc-v1` 和数字 `node_id`。
- 同步更新 README、开发文档与三平台安装包。

## 升级说明

- 重新导入 1.2.39 Windows 包并启动。
- 日志应出现 `萌卡健康探活格式 #1: json ready identity`；若框架在此格式后停止轮询，说明健康契约已匹配。
- 如果仍被停止，请提供萌卡框架控制台报错；插件侧已按官方 SDK ready 结构返回。
