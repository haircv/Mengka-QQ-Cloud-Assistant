# QQ云助手 v1.2.32

## 更新内容

- 萌卡插件卡片新增“管理”入口：
  - 清单启用 `runtime.admin=true`，萌卡管理端通过 API 反代直接打开 Web 控制台，不再依赖 `18084` 端口或仅本机访问。
  - 插件读取 `MENGKA_PLUGIN_ADMIN_HOST`（仅 `127.0.0.1`）、`MENGKA_PLUGIN_ADMIN_PORT`、`MENGKA_PLUGIN_ADMIN_TOKEN_FILE`、`MENGKA_PLUGIN_ADMIN_ORIGIN`，在回环地址额外监听管理端口；原 `0.0.0.0` 控制台保持不变。
  - 反代请求校验回环地址、`X-Mengka-Managed-Token` 与 `X-Mengka-Managed-Origin`，通过后自动进入控制台，无需再次输入本地管理令牌。
  - 兼容完整 `/api/v1/plugins/managed/{id}/web/...`、去掉 `/api/v1`、仅 `/web/...` 以及已剥离前缀四种转发路径；不带尾斜杠的 `.../web` 自动 307 补斜杠。
  - 前端静态资源与 API 全部改为相对路径，反代下自动获取 CSRF；写请求 CSRF 失败自动刷新并重试一次。
  - 管理端只监听 `127.0.0.1`；令牌文件缺失或为空时仍可用数据目录 `admin.token` 登录，但不会自动登录。
- 同步更新 README、开发文档与三平台安装包。

## 升级说明

- 在“系统 → 在线更新”检查并安装 1.2.32；若萌卡提示 `capability_change`（新增 `admin` 能力），按控制台提示确认后会携带审批令牌重试。
- 安装后重新打开萌卡插件卡片，点击“管理”即可直接进入控制台。
- 如果“管理”打开后显示本地令牌登录页，请检查插件进程是否由萌卡托管启动，以及 `MENGKA_PLUGIN_ADMIN_*` 环境变量和令牌文件是否可读。
- 局域网直连仍使用原控制台端口（默认 `0.0.0.0:18084`），需要放行防火墙。
