# QQ云助手 v1.2.46

## 更新内容

- 参考可用的 B 站综合插件（Bilibili-Utility-Suite）原生 IPC 实现重新启用管理端：
  - 清单加入 `runtime.args`：`--connection {connection} --config {data}/config.json`，与 B 站插件一致。
  - 主程序接受 `--connection` / `--config` 参数（当前只用于兼容启动，数据目录仍优先读取 `MENGKA_PLUGIN_DATA_DIR`）。
  - 不再注册 `/api/managed/health` 健康接口，该路径返回 404；B 站插件同样不提供此健康路由，框架仍可从插件卡片打开管理端。
  - 保留管理端监听、令牌校验、相对路径和反代兼容代码。
- 同步更新 README、开发文档与三平台安装包。

## 升级说明

- 重新导入 1.2.46 Windows 包并启动；如萌卡提示 `capability_change`，确认后重试。
- 启动后从插件卡片点击“管理”验证。
- 如果仍提示未就绪，请使用 18084 控制台，并提供萌卡框架 `-debug` 日志；1.2.45 回退版可用于恢复启动。
