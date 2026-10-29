# QQ云助手 v1.2.7

## 更新内容

- 新增控制台“系统 → 在线更新”：支持 GitHub Release 和自定义更新清单，稳定版/预发布通道。
- 托管模式通过 `managed.Load()` 识别是否运行在萌卡内；控制台登录与在线更新使用萌卡管理员账号、密码和图形验证码，保留本地管理令牌回退。
- 在线更新下载后校验插件 ID、版本、压缩包 SHA256（如有）以及 `SHA256SUMS` 中程序入口哈希；校验失败不导入。
- 托管环境通过萌卡 `plugins/managed/import` 导入新版本，权限变化时二次确认；独立/控制台模式只检查、下载，不替换程序。
- 默认更新源指向 `haircv/Mengka-QQ-Cloud-Assistant`，发布资产统一命名为 `*-managed-native-<platform>`。
- 新增 GitHub Actions Release 工作流：推送 `v*` 标签或手动触发后，自动测试、构建三平台安装包并上传 Release。

## 升级说明

1. 在萌卡“插件 → 插件导入”上传对应系统的 `qq-cloud-assistant-1.2.7-managed-native-*` 安装包，保留原数据目录。
2. 首次升级后打开控制台“系统 → 在线更新”，确认 GitHub 仓库为 `haircv/Mengka-QQ-Cloud-Assistant`。
3. 后续推送 `v*` 标签即可自动发布；控制台检查更新后按提示登录萌卡管理员并导入。
4. 独立控制台模式只支持检查和下载，更新仍需在萌卡中导入安装包。

## 产物

- `qq-cloud-assistant-1.2.7-managed-native-windows-amd64.zip`
- `qq-cloud-assistant-1.2.7-managed-native-linux-amd64.tar.gz`
- `qq-cloud-assistant-1.2.7-managed-native-linux-arm64.tar.gz`
- `SHA256SUMS`
