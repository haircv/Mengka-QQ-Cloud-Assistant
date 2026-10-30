# QQ云助手 v1.2.43

## 更新内容

- 重新启用萌卡插件卡片“管理”：
  - 三个平台清单恢复 `"runtime.admin": true`。
  - `/api/managed/health` 同时返回官方要求的 `ready` 和 `configured`，并提供顶层、`data` 嵌套及 `Ready/Configured/isReady/isConfigured` 别名。
  - 响应合并 IPC ready 身份字段（`type/api_version/framework_api/plugin_id/service/service_id/transport/node_id/instance_id`），并附带 `X-Mengka-Ready`、`X-Mengka-Configured` 等响应头。
  - 保留 18084 控制台；如果管理端仍未通过，仍可直接使用 18084。
- 1.2.42 的 `runtime.admin=false` 回退仍然保留为历史版本，可随时用于恢复启动。

## 升级说明

- 重新导入 1.2.43 Windows 包并启动；如萌卡提示 `capability_change`，确认后重试。
- 点击插件卡片“管理”验证；启动日志会记录一次健康探活请求。
- 若仍提示“插件管理端未就绪”，插件仍可使用 18084 控制台。请提供萌卡框架控制台报错；SDK 没有暴露管理端健康实现，继续修复需要框架侧错误信息。
