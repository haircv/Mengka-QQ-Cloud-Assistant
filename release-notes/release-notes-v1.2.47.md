# QQ云助手 v1.2.47

## 更新内容

- 按可用实现 `mengkaNT-qq-farm-bridge` 修正管理端启动闸门：
  - `/api/managed/health` 返回 `{ready:true, connected:true, configured:true, active:true}`。
  - 新增 `/api/managed/control`，GET/POST 返回 `{active:true}`，接受框架激活/控制请求。
  - 健康与控制接口位于回环管理端口，无需浏览器登录；浏览器 API 仍校验管理令牌。
  - 保留 `runtime.admin=true`、`runtime.args` 与管理端反代能力。
- 同步更新 README、开发文档与三平台安装包。

## 升级说明

- 重新导入 1.2.47 Windows 包并启动；如萌卡提示 `capability_change`，确认后重试。
- 启动后从插件卡片点击“管理”验证。
- 如果仍提示未就绪，请使用 18084 控制台，并提供萌卡框架 `-debug` 日志；1.2.45 回退版可用于恢复启动。
