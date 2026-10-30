# QQ云助手 v1.2.33

## 更新内容

- 完善萌卡插件卡片“管理”的反代兼容：
  - 支持 `MENGKA_PLUGIN_ADMIN_BASE_PATH` 挂载前缀，自动剥离反代路径并在访问裸前缀时 307 补斜杠。
  - 管理令牌同时接受 `X-Mengka-Managed-Token`、`X-Mengka-Admin-Token`、`Authorization: Bearer` 与 URL `token` 参数；仍校验回环地址与 `X-Mengka-Managed-Origin`。
  - 支持 `MENGKA_PLUGIN_ADMIN_EMBEDDED=1` 且令牌文件为空的内嵌模式：只信任回环反代请求，并生成独立 CSRF。
  - 前端从 URL `token` 参数读取管理令牌，附加到全部 API、上传、备份与日志导出请求，解决部分萌卡版本只通过 URL 传递令牌时无法自动登录的问题。
  - 启动日志新增实际挂载前缀，便于排查反代路径不一致。
- 保持完整 `/api/v1/plugins/managed/{id}/web/...`、去掉 `/api/v1`、仅 `/web/...` 与已剥离前缀四种转发兼容。
- 同步更新 README、开发文档与三平台安装包。

## 升级说明

- 在“系统 → 在线更新”安装 1.2.33；若萌卡提示 `capability_change`（新增 `admin` 能力），按控制台提示确认后会携带审批令牌重试。
- 安装后从萌卡插件卡片点击“管理”，应直接进入控制台；若仍显示登录页，请查看插件启动日志中的“萌卡管理端已监听”和“萌卡管理端挂载前缀”。
- 局域网直连仍使用原控制台端口（默认 `0.0.0.0:18084`），需要放行防火墙。
