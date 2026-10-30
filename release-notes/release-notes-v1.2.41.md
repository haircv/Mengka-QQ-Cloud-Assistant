# QQ云助手 v1.2.41

## 更新内容

- 管理端健康接口同时返回官方契约与 ready 身份：
  - 顶层与 `data` 嵌套均包含 `ready:true`、`configured:true`。
  - 追加 `Ready`、`Configured`、`isReady`、`isConfigured` 大小写/别名，兼容不同解析方式。
  - 合并 ready 帧身份字段：`type/api_version/framework_api/plugin_id/service/service_id/service_name/transport/node_id/instance_id/connected/state/version`。
  - 响应头增加 `X-Mengka-Ready`、`X-Mengka-Configured`、`X-Mengka-Health-Ready`、`X-Mengka-Health-Configured`。
- 同步更新 README、开发文档与三平台安装包。

## 升级说明

- 重新导入 1.2.41 Windows 包并启动。
- 启动日志应只出现一次健康探活记录，之后约 20 秒内不应再被停止。
- 如果仍提示未就绪，请提供萌卡框架控制台的报错；插件侧已同时覆盖官方文档字段、嵌套结构、大小写别名和 ready 身份。
