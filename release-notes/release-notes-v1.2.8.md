# QQ云助手 v1.2.8

## 更新内容

- 修复 GitHub Release 在线更新使用的下载地址：优先使用 `browser_download_url`，不再误用需要认证头的 API asset 地址。
- 保留 API `digest` 校验与 `SHA256SUMS` 程序入口校验，下载后仍会验证插件 ID 和版本。
- 默认更新源继续指向 `haircv/Mengka-QQ-Cloud-Assistant`。

## 升级说明

在控制台“系统 → 在线更新”检查到 1.2.8 后，登录萌卡管理员并导入即可。也可在萌卡“插件 → 插件导入”上传对应系统的 `qq-cloud-assistant-1.2.8-managed-native-*` 安装包，保留原数据目录。

## 产物

- `qq-cloud-assistant-1.2.8-managed-native-windows-amd64.zip`
- `qq-cloud-assistant-1.2.8-managed-native-linux-amd64.tar.gz`
- `qq-cloud-assistant-1.2.8-managed-native-linux-arm64.tar.gz`
- `SHA256SUMS`
