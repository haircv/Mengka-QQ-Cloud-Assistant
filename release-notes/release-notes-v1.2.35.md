# QQ云助手 v1.2.35

## 更新内容

- 修复萌卡启动后约 20 秒终止插件、提示“插件管理端未就绪”：
  - 萌卡实际轮询 `GET /api/managed/health`；此前该路径被静态处理器当作未知文件返回 `index.html`，框架解析不到健康状态后判定未就绪并关闭插件。
  - 新增专用 JSON 探活接口，返回 `ok`、`status: "ok"`、`ready`、`healthy`、`code: 0` 及嵌套 `data`，并附带 `X-Mengka-Managed-Ready`、`X-Mengka-Admin-Ready`、`X-Mengka-Plugin-Health` 等响应头。
  - 同时支持 `/api/managed/health/`、`/api/managed/healthz`、`/api/managed/ready`、`/api/managed/readyz`、`/api/managed/ping`、`/api/managed/status`。
  - 该高频探活不再写入插件日志，避免每秒刷屏；首次探活记录 `Accept`、`Content-Type`、`User-Agent` 和是否携带令牌，便于继续排查。
- 说明：萌卡管理端显示的 `127.0.0.1:5454` / `bot.qqir.cn` 是框架自身的反代入口；插件使用框架注入的实例管理端口（日志中的 `MENGKA_PLUGIN_ADMIN_PORT`，例如 56178），不需要也不应该手动绑定 5454。
- 同步更新 README、开发文档与三平台安装包。

## 升级说明

- 在“系统 → 在线更新”安装 1.2.35；若插件已被框架停止，可在萌卡插件管理中重新启动或重新导入后启动。
- 启动日志应出现：
  - `萌卡管理端配置: port=... token=set`
  - `萌卡管理端已监听 0.0.0.0:...`
  - 不再出现约 20 秒后的 `正在退出...`
- 然后从插件卡片点击“管理”即可进入控制台。
