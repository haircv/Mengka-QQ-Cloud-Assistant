# QQ云助手（qq-cloud-assistant）

QQ云助手是面向萌卡 NT 3.0 的 QQ 云端助手插件，以原生 IPC 插件运行，并自带 Web 控制台。

- 插件 ID：`qq-cloud-assistant`
- 当前版本：**1.2.9**
- 平台：Windows x64 / Linux x86_64 / Linux ARM64
- 更新说明：[release-notes](release-notes/README.md)

## 下载

从 [v1.2.9 Release](https://github.com/haircv/Mengka-QQ-Cloud-Assistant/releases/tag/v1.2.9) 下载对应系统的安装包：

| 系统 | 安装包 |
| --- | --- |
| Windows AMD64 | `qq-cloud-assistant-1.2.9-managed-native-windows-amd64.zip` |
| Linux AMD64 | `qq-cloud-assistant-1.2.9-managed-native-linux-amd64.tar.gz` |
| Linux ARM64 | `qq-cloud-assistant-1.2.9-managed-native-linux-arm64.tar.gz` |

每个 Release 同时提供 `SHA256SUMS`。

## 安装与更新

1. 在萌卡 NT「插件 → 插件导入」上传对应系统的安装包，选择节点并启动。
2. 打开插件控制台。运行在萌卡内时，默认使用萌卡管理员账号、密码和图形验证码登录；也可以切换到本地管理令牌。
3. 在「系统 → 在线更新」检查新版本。默认更新源为本仓库的 GitHub Release。
4. 发现新版本后，按页面提示登录萌卡管理员，插件会校验安装包并调用萌卡托管导入完成更新；权限变化时需要二次确认。
5. 旧版本无法启动时，可直接导入新版安装包并保留原数据，无需先卸载。

## 功能概览

- 群管、提示、自动化、问答、定时/循环/群发、点歌与搜歌。
- 五类红包助手：普通、拼手气、口令、专属、语音，支持群/人员/关键词规则与 Android 领取。
- 本地卡密、分群授权、代理与可选授权门禁。
- 响应式 Web 控制台：群号联想下拉、主人 QQ 管理、在线更新与托管登录。
- 运行状态显示账号、登录协议、在线时长与框架信息。

## 更新日志

版本说明统一放在 [release-notes](release-notes/README.md)。

## 说明

- 本仓库用于发布 QQ云助手安装包与更新说明。
- 在线更新会校验 `plugin_id`、版本与 `SHA256SUMS`，非本插件安装包不会被导入。
