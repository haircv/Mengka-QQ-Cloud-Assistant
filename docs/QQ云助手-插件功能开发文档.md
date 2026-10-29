# QQ云助手（qq-cloud-assistant）插件功能开发文档

> 版本：v3.9（2026-10-29，v1.2.18 红包嵌套请求恢复与独立启动助手增补；含 1.2.17 点歌搜索、@ 解析与多协议 Android 优先）
> 目标框架：萌卡 NT 3.0，原生 IPC 插件（`native-ipc-v1`，`mengka-plugin.json` schema 3）
> 功能基线：`cloud-assistant 1.0.0`（源码 43 个注册类目、370 条注册项，其中 7 条为占位）
> 控制台参考：`group_management 1.0.3beta`（仅参考布局、目录驱动、交互与视觉）
> 授权模式：本地卡密 / 分群授权 / 代理，无联网校验；门禁默认关闭


---

## 0.1 v1.1 增补（优先于下文历史口径）

2026-10-28 在 v1.0 基线上补充以下能力；下文涉及“333 条 / 48 个 action / 51 个模块 / 完全免授权 / 不实现红包卡密代理”的历史表述，以本节为准。

| 项目 | v1.1 结论 |
|---|---|
| 指令 | v1.1 为 360 条 = 基线 333 + 授权 16 + 红包 11 |
| action | v1.1 为 50 个 = 基线 48 + 红包 2 |
| 控制台 | v1.1 为 57 个模块；v1.2 见 0.2 |
| 授权 | 本地卡密 `QCA-XXXX-XXXX-XXXX`、分群授权、代理配额；无授权站、应用 ID、心跳、远程下发、到期退群 |
| 门禁 | `license.enforce` 默认关闭；开启后仅拦截群内指令，主人/自身豁免，授权类指令与兑换始终可用 |
| 红包 | v1.1：普通/口令；v1.2：五类独立开关、规则、语音红包；v1.2.15 增加发送开关与内置 AI/自定义 TTS/固定音频三种来源；v1.2.16 增加请求兼容与多级凭据重试 |
| 红包 action | v1.1：查询/领取；v1.2 增加语音匹配、群/好友语音上传、语音领取，共 54 个 action |
| 存储 | v1.1 新增 `license_cards`、`group_authorizations`、`agents`、`red_packet_records` 四张表；schema 版本 4（v1.2 为 5） |
| 数据隔离 | 卡密/代理全局；分群授权、红包配置、红包账本按 `self_id + protocol_id` 隔离 |

详细实现见 5.9（新增指令）、6.5（控制台模块）、6.7（API）、7.1（表结构）、7.2（配置）、8.17–8.19（红包/授权/清理）、12.1（测试）。

红包处理约定：

1. 仅在账号总开关、本群开关、监听范围同时满足时处理；默认 `listen_all=false`，群内发送“开启红包助手”会自动把本群加入监听名单。
2. 收到 `red_packet` 消息段后按 `listid` 去重；查询详情只用于总额/份数与 `claim_token`，不把查询当作领取。
3. 期望值 = 总金额（分）÷ 份数；低于 `min_expected_fen` 记为 `skipped/below_threshold`。
4. 口令红包先发送去空白口令，再用发送结果 `client_seq`、`msg_random` 与原红包数据领取；缺少有效标识时放弃，不使用红包消息的 `msg_seq`。
5. 领取成功必须同时满足 `retcode=0` 与 `state=0`；金额只读 `recv_object.amount`，缺失记未知，默认不回复。
6. 到账回复延迟发送，支持模板变量；失败不自动重试；完整流程耗时与领取响应分别记录。
7. v1.1 语音红包跳过；v1.2 改为生成标题音频 → 匹配 → 群语音 → 专用领取；v1.2.15 默认内置 AI 语音，可切换自定义 TTS 或固定音频；v1.2.16 音频改用 Base64 上传；v1.2.18 增加多协议 Android 优先与 @ 目标回退，见 0.2。
8. 账本默认保留 30 天，后台每 6 小时清理；日志不记录 `authkey`、`claim_token`。

授权约定：

1. 卡密由主人或代理生成；代理配额 0 表示不限，生成时原子校验并累计配额。
2. 群主在群内使用卡密，续期从当前有效到期时间顺延；卡密一旦使用不可禁用/删除。
3. 手动授权、撤销、续期均在控制台完成并写审计日志。
4. 授权门禁不访问网络；未授权提示可配置；红包引擎在门禁开启时只处理已授权群。


## 0.2 v1.2 增补（优先于 v1.1 与下文历史口径）

| 项目 | v1.2 结论 |
|---|---|
| 指令 | 382 条 = v1.1 360 + 红包新增 18 + 语音点歌 1 + 在线更新 3 |
| action | 54 个 = v1.1 50 + `upload_red_packet_voice`、`upload_group_voice`、`upload_friend_voice`、`grab_group_voice_red_packet` |
| 控制台 | 59 个模块 = v1.1 57 + 点歌线路 + 在线更新 |
| 红包类型 | 普通、拼手气、口令、专属、语音五类独立开关；按显式 `type` 优先，其次 `channel`：6/32 口令、65536 语音、1024 专属、2 拼手气，其余普通 |
| 红包规则 | 群/人员/关键词（标题）包含或排除；监听群、只领不回复、每日上限、每分钟限流、去重继续保留 |
| 红包平台 | 领取要求 Android 协议账号；语音红包默认内置 AI 语音，需先匹配并发送语音，关闭发送时跳过 |
| 点歌 | QQ/网易云/酷狗多源；Meting/Meting 兼容/网易直连线路；优先级与失败换源；选歌后链接/语音/卡片回复 |
| 音乐卡片 | 官方 `get_music_ark` 仅支持 `qq`、`163`；酷狗卡片回退链接或语音 |
| 在线更新 | 默认 GitHub `haircv/Mengka-QQ-Cloud-Assistant`；托管模式管理员账号+密码+图形验证码后通过 `plugins/managed/import` 导入并自动启动；包校验 ID/版本/SHA256 |
| 存储 | 新增 `music_lines`、`red_packet_rules`；`red_packet_records.packet_type` 增加 lucky/exclusive；schema 版本 5 |

v1.2 行为约定：

1. 红包类型识别优先使用明确 `channel`（6/32 口令、65536 语音、1024 专属、2 拼手气）；`channel=1/0` 时再读取 `type`/`packet_type`。若上游既不提供明确 channel 也不提供 type，独立开关无法区分普通与拼手气。
2. 专属红包不自行判断接收人；以官方领取接口结果为准，非目标按失败记录。
3. 语音红包流程：准备标题音频（默认内置 AI 语音，可选自定义 TTS 或固定文件/URL），本地/生成音频使用 Base64 上传 → `upload_red_packet_voice` 匹配 → `upload_group_voice` 发送 → `grab_group_voice_red_packet`；关闭发送、音频准备失败或匹配失败不调用领取。
4. 点歌搜索按“首选来源 → 其他来源”尝试启用的线路，单次最多 8 条线路、总耗时上限 20 秒；无结果或全部失败回退搜索链接。
5. 点歌语音优先 `upload_group_voice`；好友/临时会话用 `upload_friend_voice`。语音失败回退卡片，卡片失败回退链接。
6. `开启点歌功能` 同时写入 `music.search_enabled=true`；卡片/语音模式在搜索关闭时直接提示开启搜索，不回退链接。
7. 禁言/解禁/踢人/上下管的目标解析顺序：at 段数字 ID → @ 数值 → UID 映射 → 群名片/昵称 → 引用消息发送者；失败日志仅记录段类型与字段名。

---

## 0. 一页结论

| 项目 | 结论 |
|---|---|
| 中文名 | QQ云助手 |
| 英文名 / 插件 ID | `qq-cloud-assistant` |
| 插件形态 | 萌卡 NT 3.0 IPC 插件；框架启动子进程，stdin/stdout 逐行 JSON |
| 账号模型 | `self_id + protocol_id` 双键；`protocol_id = 0` 是有效值 |
| 功能范围 | 基线 333 条；v1.2 增补 49 条，共 382 条（详见 0.1/0.2） |
| 不可用 | 29 条（字符 5、被戳 6、进群屏蔽/免打扰/群助手 6、上管/下管提示 6、自动退禁言群 4、自更新 2） |
| 权限 | 0 黑名单 / 1 群员 / 2 管理 / 3 群主 / 4 分管 / 5 主人；全部指令默认 Lv5 |
| 自身账号 | 发送者 QQ == 机器人 QQ，直接 Lv5 |
| 授权 | 本地卡密/分群授权/代理；无授权站、应用 ID、心跳、远程校验、到期退群 |
| 存储 | SQLite（业务）+ 分层 JSON（配置）+ `admin.token`（控制台） |
| 控制台 | 插件自建 Web 控制台，默认 `127.0.0.1:18084`，v1.2 共 59 个目录驱动模块（详见 0.2） |
| 产物 | Windows x64、Linux x86_64 与 Linux ARM64 各一个 IPC 插件包，同一源码构建 |

---

## 1. 项目定位

### 1.1 名称与产物

- 中文名：QQ云助手
- 英文名：`qq-cloud-assistant`
- 插件 ID：`qq-cloud-assistant`
- 版本：`1.2.18`
- 语言：Go 1.25
- 依赖：
  - `modernc.org/sqlite`（纯 Go，无 cgo）
  - `golang.org/x/image`（图片菜单文字渲染）
  - `github.com/gorilla/websocket`（仅正向 WS 调试模式，可选）
- 产物：
  - Windows：`qq-cloud-assistant.exe` + `mengka-plugin.json`
  - Linux：`qq-cloud-assistant` + `mengka-plugin.json`
  - 压缩包：ZIP / TAR.GZ / TGZ，根目录必须含清单，最大 512 MiB

### 1.2 设计目标

1. **功能完整**：保留 `cloud-assistant` 的中文指令名、类目结构、菜单习惯与配置思路。
2. **平台原生**：所有 QQ 操作走萌卡 NT 3.0 官方 action，不自行登录、不保存账号密码。
3. **默认安全**：所有指令默认仅 Lv5；自动功能只受开关与名单控制。
4. **可观测**：action 调用、发送失败、权限拒绝、配置变更、任务进度都有日志与统计。
5. **可降级**：平台能力缺失时不注册指令，控制台标明原因，不出现“点了没反应”。
6. **免授权**：启动、运行、退出全程不访问授权服务。
7. **可移植**：同一源码交叉编译 Windows x64、Linux x86_64 与 Linux ARM64；控制台不依赖外部 CDN。

### 1.3 与参考项目的关系

只从 `group_management` 参考以下内容：

- 深色侧栏 + 顶部栏 + 内容区的管理台布局；
- 目录驱动（后端下发模块，前端通用渲染）的 SPA 结构；
- 登录页、表格、表单、弹窗、确认框、Toast、分页与搜索；
- QSign 主题的 CSS 变量与控件尺寸。

不迁移：

- `internal/auth`、授权心跳、应用 ID、机器码、联网校验；
- 支付、订单、财务、收款；
- 到期退群、远程配置下发；
- `plugin-home`、联网授权相关页面与图标。

本地重新实现（不引用其代码）：卡密、分群授权、代理配额与本地授权门禁。

从 `cloud-assistant` 迁移：

- 指令注册表、匹配 DSL、菜单生成、权限引擎、事件路由、消息缓存；
- 36 个真实类目的业务逻辑与中文指令名；
- SQLite 业务表、三层 JSON 配置、Merge Patch、迁移思路。

`cmdstub` 的 7 个占位类目整体不迁移。

### 1.4 术语

| 术语 | 定义 |
|---|---|
| `AccountKey` | `(self_id, protocol_id)`，唯一定位一个账号实例 |
| 会话键 | 群聊 `group:<group_id>`；好友 `friend:<user_id>`；临时会话 `temp:<user_id>` |
| 指令键 | 注册器推导的权限键，如 `全体禁言`、`踢@`、`禁言@` |
| 类目键 | 如 `basic`、`qa`、`timer`；全局权限键写作 `category:basic` |
| 有效级别 | 群覆盖 → 指令覆盖 → 类目覆盖 → 默认 5 |
| 能力快照 | 启动时从 `get_plugin_context` 等接口得到的 action/事件/消息段集合 |

### 1.5 参考资料

官方文档：

- <https://mengka.net/nt/api/>
- <https://mengka.net/nt/reference/plugin-development.html>
- <https://mengka.net/nt/reference/managed-plugins.html>
- <https://mengka.net/nt/reference/protocol.html>
- <https://mengka.net/nt/events/>
- <https://mengka.net/nt/reference/message-segments.html>
- <https://mengka.net/nt/development/webui-sdk.html>

社区资料：

- <https://github.com/shiina0118/MengKa-Docs>
- <https://github.com/Jay1023CN/Mengka-NT-plugins>

本地附件：

- `QbMaxGo云端源码.zip`：功能、指令、配置、数据模型的主要来源。
- `Group-Management.zip`：仅参考 Web 控制台布局、目录驱动、交互与视觉。
- `QbMaxGo群管源码.zip`：可选参考审计、备份、消息缓存、调度实现；不迁移其指令集与群管拦截逻辑。

---

## 2. 萌卡 NT 3.0 接入

### 2.1 接入方式

| 方式 | 是否采用 | 用途 |
|---|---|---|
| 本地 IPC | **主方式** | 插件包安装、随框架启动、stdin/stdout 逐行 JSON |
| 正向 WebSocket | 仅调试 | 本机开发时连接框架，复用同一 action 客户端 |
| HTTP | 不采用 | 只适合外部插件，不能订阅事件 |
| 反向 WebSocket | 不采用 | 插件不需要对框架暴露 WS 服务 |

IPC 形态下，插件不需要 `TOKEN`、`SERVER`、`STORAGE_DIR`，也不需要框架服务令牌。账号范围由安装时选择的节点决定。

### 2.2 `mengka-plugin.json`

```json
{
  "schema": 3,
  "framework_api": "3.0.0",
  "plugin_id": "qq-cloud-assistant",
  "name": "QQ云助手",
  "version": "1.2.18",
  "min_framework": "3.0.0",
  "author": "QQ云助手",
  "notes": "QQ 云端助手，本地卡密，无联网授权",
  "runtime": {
    "entry": "qq-cloud-assistant.exe",
    "transport": "native-ipc-v1",
    "api_version": "1",
    "admin": false,
    "actions": [
      "get_plugin_context",
      "get_version_info",
      "get_bot_list",
      "get_bot_info",
      "get_protocol_list",
      "get_group_list",
      "get_group_member_list",
      "get_friend_list",
      "send_group_msg",
      "send_friend_msg",
      "send_group_temp_msg",
      "recall_group_msg",
      "recall_friend_msg",
      "get_msg",
      "upload_group_image",
      "upload_friend_image",
      "set_msg_emoji_like",
      "set_group_mute",
      "set_group_mute_all",
      "set_group_admin",
      "set_group_special_title",
      "set_group_card",
      "kick_group_member",
      "leave_group",
      "sign_group",
      "get_group_honor_info",
      "get_group_share_link",
      "get_group_at_all_remain",
      "get_group_system_notifications",
      "approve_group_apply",
      "reject_group_apply",
      "get_friend_system_notifications",
      "approve_friend_apply",
      "reject_friend_apply",
      "set_friend_remark",
      "get_qzone_friend_feeds",
      "like_qzone_feed",
      "publish_qzone_feed",
      "comment_qzone_feed",
      "delete_qzone_feed",
      "like_user_profile",
      "get_received_profile_likes",
      "get_sent_profile_likes",
      "set_qq_avatar",
      "profile_operation",
      "get_music_ark",
      "file_transfer",
      "get_contact_ark",
      "get_red_packet_info",
      "grab_red_packet",
      "upload_red_packet_voice",
      "upload_group_voice",
      "upload_friend_voice",
      "grab_group_voice_red_packet"
    ],
    "events": [
      "group_message",
      "friend_message",
      "group_event",
      "friend_event",
      "bot_online",
      "bot_offline"
    ]
  }
}
```

说明：

- Windows 包的 `entry` 指向 `.exe`；Linux 包指向无扩展名二进制。
- `actions` 只列实际调用项，禁止调用未声明接口。
- `events` 是事件权限名，不是 `post_type`：
  - `group_message`：群消息；
  - `friend_message`：好友消息与群临时会话；
  - `group_event`：`group_notice`；
  - `friend_event`：`friend_notice`；
  - `bot_online` / `bot_offline`：账号上下线。
- 不需要 `approve_group_invite`：V1 只自动处理入群申请，不解析好友邀请卡片。
- 不需要 `get_group_msg_history`、`recall_message`、`send_poke`、`unlike_qzone_feed`。
- `get_group_system_notifications` / `get_friend_system_notifications` 仅用于 `bot_online` 后补处理最近 24 小时的待处理申请。

### 2.3 IPC 生命周期

```text
框架启动插件进程
  → 插件 stdout: {"type":"hello","api_version":"1","framework_api":"3.0.0","plugin_id":"qq-cloud-assistant","version":"1.2.18"}
  → 框架 stdin: {"type":"ready"}
  → 插件可发 action
  → 框架推送 event
  → 插件发 action / 收 action_result
  → SIGTERM / SIGINT / stdin 关闭
  → 停止接收 → 排空队列（最多 5 秒）→ 关闭 DB/HTTP → 退出
```

硬性要求：

1. `hello` 必须是第一行 stdout，之前不能有任何输出。
2. `plugin_id`、`version` 必须与清单完全一致。
3. 未收到 `ready` 前不得发 action；10 秒未收到则退出。
4. stdout 只写协议 JSON；日志、panic、调试信息全部写 stderr。
5. 单行 JSON 必须完整，写 stdout 必须加锁。
6. stdin 读取使用可增长缓冲，至少支持 16 MiB 单行。
7. 响应可能乱序，必须按 `id` 配对；不得依赖顺序。
8. 连接关闭时，所有 pending 请求返回“连接已关闭”。

请求帧：

```json
{"type":"action","id":"qca-1","action":"get_group_list","params":{"self_id":123456789,"protocol_id":0}}
```

响应帧：

```json
{"type":"action_result","id":"qca-1","ok":true,"data":{}}
{"type":"action_result","id":"qca-1","ok":false,"error":"账号未在线"}
```

事件帧：

```json
{"type":"event","data":{"self_id":123456789,"protocol_id":0,"post_type":"group_message"}}
```

### 2.4 账号模型

```go
type AccountKey struct {
    SelfID     int64
    ProtocolID int64
}
```

规则：

- 任何缓存、配置、Worker、统计、任务都必须带 `AccountKey`。
- `protocol_id` 来自 `get_protocol_list` 的 `id`，不是数组下标；`0` 有效，不能省略。
- `get_bot_list` 返回节点内全部账号实例（含离线），空节点返回 `[]`。
- 一个 QQ 号可能有多个协议实例（手机/平板/Linux），配置与统计必须分开。
- 控制台账号选择器必须同时显示 `self_id` 与 `protocol_id`。

### 2.5 action 客户端

```go
type Result struct {
    OK            bool
    Data          json.RawMessage
    Error         string
    Status        string // restricted / queries_beyond_limit
    Code          string // ACCOUNT_OFFLINE / QQ_LEVEL_TOO_LOW / ...
    ErrCode       int
    RequiredLevel int
    CurrentLevel  int
}
```

规则：

- `id` 格式：`qca-<pid>-<seq>`，进程内单调递增。
- 默认超时 30 秒；上传/头像/空间 120 秒；红包查询 30 秒、文本领取 60 秒；语音匹配 300 秒、群语音上传 120 秒、语音领取 120 秒；点歌搜索总耗时上限 20 秒；批量任务逐条 30 秒。
- 同一连接并发上限 8（官方每连接最多 16）。
- 写操作失败**不自动重试**：发送、撤回、审批、踢人、禁言、点赞、评论、发说说、改资料。
- 读操作失败最多重试 1 次，退避 1 秒/3 秒；`queries_beyond_limit` 不重试，暂停查询 60 秒。
- `ok:false` 必须读取 `error`、`status`、`code`、`errCode`、`requiredLevel`、`currentLevel`，给用户可读提示。
- `status=restricted` 不得当成成功。
- 有业务结果的接口继续检查 `success`、`error_code`、`code`，不能只看外层 `ok`。
- 单条请求 JSON 最大 4 MiB；发送前检查，超限拆分为多条或改用媒体上传。
- 无 `id` 的请求不会收到响应，插件不得使用。

`get_group_member_list`、`get_group_list` 等查询类接口，遇到 `queries_beyond_limit` 时暂停该账号的查询 60 秒，避免连续重试。

### 2.6 能力探测

启动并收到 `ready` 后依次调用：

| action | 用途 |
|---|---|
| `get_plugin_context` | `available_actions`、`message_segments`、`event_permissions`、`temporary_message` |
| `get_version_info` | 框架版本、协议类型（需要在线账号；无账号则跳过） |
| `get_bot_list` | 账号实例快照 |
| `get_protocol_list` | 协议目录与启用状态 |

能力快照：

```json
{
  "framework_version": "3.0.0",
  "api_version": "3.0",
  "node_id": "node-1",
  "actions": ["send_group_msg", "..."],
  "segments": {
    "group": ["text", "at", "image", "voice", "video", "face", "ark", "reply"],
    "friend": ["text", "image", "voice", "video", "face", "ark", "reply"],
    "group_temp": ["text", "image", "voice", "video", "face", "ark", "reply"]
  },
  "events": ["group_message", "friend_message", "group_event", "friend_event", "bot_online", "bot_offline"],
  "temporary_message": true,
  "unavailable": [
    {"feature": "字符功能", "reason": "缺少抽字符接口"},
    {"feature": "被戳处理", "reason": "缺少 poke 通知事件"}
  ]
}
```

处理原则：

- 缺 action：依赖它的指令不注册。
- 缺消息段：对应模式隐藏或回退文字。
- 缺事件：对应自动功能停用。
- 能力快照写日志、落库，并在控制台“能力”页展示。
- `get_plugin_context` 失败时以清单声明为基线，并标记“能力未知”。

### 2.7 事件模型

外层：

```json
{"type":"event","data":{"self_id":123456789,"protocol_id":0,"post_type":"group_message"}}
```

#### 消息事件

| 字段 | 说明 |
|---|---|
| `post_type` | `group_message` / `friend_message` / `group_temp_message` |
| `self_id` / `protocol_id` | 账号与协议 |
| `group_id` / `group_name` | 群消息；临时消息的 `group_id` 可能缺失 |
| `message_id` | 框架消息编号；引用、表情回应、`get_msg` 使用 |
| `msg` | `uid`（十进制字符串）、`seq`、`random`、`time`（Unix 秒） |
| `sender` | `user_id`、`nickname`，可能含 `remark`、`card`、`role`、`title`、`level` |
| `receiver` | 好友/临时消息的接收者 |
| `message` | 结构化消息段数组 |
| `alt_message` | 可读摘要，只作日志/兜底 |
| `source` | 来源，可能省略 |

关键事实：

- `sender.role` 是**数字**：`0` 普通成员、`1` 管理员、`2` 群主；缺失时 `RoleKnown=false`，必须查询成员列表，不能把零值当成员。
- 好友消息没有顶层 `user_id`，触发者取 `sender.user_id`。
- `group_temp_message` 使用 `friend_message` 事件权限；`group_id` 缺失时不做群级权限、不做引用回复。
- 自身消息的 `sender.user_id == self_id`，按 Lv5 处理。
- 名称展示：好友 `remark → nickname → user_id`；群成员 `card → nickname → user_id`。

#### 通知事件

`group_notice` 的 `sub_type`：

| `sub_type` | 用途 |
|---|---|
| `increase` | 入群欢迎、入群验证、权限缓存失效 |
| `decrease` | 退群/被踢提示；有 `operator_id` 视为被踢 |
| `recall` | 防撤回、撤回通知 |
| `mute` | 禁言记录、机器人自身被禁言状态 |
| `apply` | 入群申请自动审批 |

`friend_notice` 的 `sub_type`：

| `sub_type` | 用途 |
|---|---|
| `increase` | 新好友回复、自动备注 |
| `decrease` | 好友关系移除通知 |
| `recall` | 好友防撤回 |
| `like` | 自动回赞 |
| `apply` | 自动同意好友 |

不存在的通知：

- 没有管理员/群主变更事件，因此上管/下管提示不可用；权限缓存只能 TTL 失效。
- 没有群名片变更、群名变更、文件上传、戳一戳事件。
- 没有 poke 通知，因此被戳处理不可用。

撤回事件：

- `body.msg_seq`、`body.msg_time`、`body.msg_random` 用于定位原消息；
- `body.original` 保留原消息 5 分钟；过期时正文为“消息已过期”；
- 缺失 `original` 时用本地消息缓存兜底；
- 两处都没有时只提示“对方撤回了一条消息”，不伪造内容。

#### 系统事件

- `bot_online`：刷新账号状态与群列表；补处理最近 24 小时待审批申请；恢复任务。
- `bot_offline`：标记账号离线，暂停发送与任务；`offline_reason` 记录到日志；不自动重登。
- 断线期间不保证补发事件；重连后主动刷新账号、群列表与待处理申请。

### 2.8 消息段

| 类型 | 群聊 | 好友/临时 | data |
|---|---|---|---|
| `text` | 支持 | 支持 | `text` |
| `at` | 支持 | 不支持 | `user_id` 正整数或 `"all"` |
| `image` | 支持 | 支持 | `file_id` |
| `voice` | 支持 | 支持 | `file_id` |
| `video` | 支持 | 支持 | `file_id` |
| `face` | 支持 | 支持 | `kind` + 表情字段 |
| `ark` | 支持 | 支持 | 完整卡片对象或字符串 |
| `reply` | 支持 | 支持 | `reply_reference_id`；兼容数字 `message_id` |
| `red_packet` | 支持接收 | 不支持 | `title`、`listid`、`authkey`、`channel`、`pay_flag`、`hb_from`、`msg_random`、`client_seq` |

规则：

- 媒体必须先上传，拿到 `file_id` 再发送；`file_id` 有效期 5 分钟。
- `file_transfer` 的 `transfer://` 是临时文件引用，不是 QQ 媒体 `file_id`；仍需调用上传接口。
- 群视频必须单独发送；好友/临时会话的语音、视频必须是唯一消息段。
- 引用优先 `reply_reference_id`；接收引用段里的字符串 `message_id` 是 QQ MsgUID，不能作为发送引用编号。
- At 只接受 `user_id`，不再接受旧字段 `uin`/`uid`。
- 不支持的段必须报错，不能静默丢弃。
- `red_packet` 只用于识别与领取；`authkey`、`claim_token` 是短时凭据，不落日志/账本，`listid` 只存哈希。语音红包走音频准备/匹配/群语音/专用领取。
- 图片菜单上传失败时回退文字菜单。

### 2.9 旧宿主协议 → 萌卡 NT 3.0 映射

| 旧 action/字段 | 新 action/字段 | 说明 |
|---|---|---|
| `set_group_kick` | `kick_group_member` | `user_id` + `user_ids` + `reject_add_request` |
| `set_group_ban` | `set_group_mute` | `duration_sec`，0 为解禁 |
| `set_group_whole_ban` | `set_group_mute_all` | `mute` 布尔 |
| `set_group_admin` | `set_group_admin` | `set_admin` 布尔 |
| `get_group_info` / `get_admin_list` | `get_group_list` / `get_group_member_list` | 角色为数字 |
| `get_group_members` | `get_group_member_list` | 返回 `members[]` |
| `get_group_banned_uids` | 无 | 改为本地 `mute_records`；仅记录本插件操作与 mute 事件 |
| `quit_group` | `leave_group` | 返回 `submitted`，不等于已完成 |
| `group_signin` | `sign_group` | 可能返回业务失败 |
| `delete_msg` | `recall_group_msg` / `recall_friend_msg` | 需要 `msg_seq`/`msg_random`；好友还需 `client_seq`/`msg_time` |
| `group_msg_reaction` | `set_msg_emoji_like` | 使用消息 `message_id`，不是 `msg_seq` |
| `get_qzone_feeds` | `get_qzone_friend_feeds` | 全局好友动态，不再按好友单独拉取 |
| `qzone_like_http` | `like_qzone_feed` | 传完整 feed 对象 |
| `publish_mood` | `publish_qzone_feed` | 文本返回 `feed`，图片返回 `tid` |
| `like_card` | `like_user_profile` | `like_count` |
| `set_nickname` / `set_signature` | `profile_operation` | `update` / `signature.set` |
| `approve_friend_request` | `approve_friend_apply` | 需要事件里的 `request_id`、`msg_type`、`request_type`、`action_info` |
| `handle_group_request` | `approve_group_apply` | 使用 `request_key` 的 `request_id/type/extra` |
| `set_msg_no_disturb` | 无 | 进群屏蔽/免打扰/群助手不可用 |
| `lucky_word_draw` | 无 | 字符功能不可用 |
| `get_group_share_card` | `get_contact_ark` | 群名片群发；失败回退分享链接 |
| `json` 消息段 | `ark` 消息段 | 卡片接口返回的完整对象 |
| `at` 的 `uid` | `at` 的 `user_id` | 直接使用 QQ 号 |
| 引用 `msg_seq/random` | `reply_reference_id` | 使用事件顶层 `message_id` |
| `group_id` 字符串 | `group_id` 数字 | 按官方结构解析 |

### 2.10 运行环境与数据目录

IPC 插件不保证工作目录可写。数据目录解析顺序：

1. `--data-dir` 或环境变量 `QCA_DATA_DIR`；
2. 萌卡注入的 `MENGKA_PLUGIN_DATA_DIR`（托管持久目录，更新/重新导入不改变）；
3. 用户配置目录：Windows `%APPDATA%\qq-cloud-assistant`，Linux `~/.config/qq-cloud-assistant`；
4. 旧版可执行目录 `data/` 或 Windows `%LOCALAPPDATA%\qq-cloud-assistant`。

托管目录为空时自动从用户配置目录或旧版 `data/` 原子迁移数据库、配置、授权与日志，并保留 `.pre-migrate-*` 备份；迁移失败继续使用原目录，避免二次丢失。

启动时检查可写性；失败则向 stderr 输出明确原因并退出，不静默降级。

```text
data/
├── qca.db
├── admin.token
├── global.json
├── permissions.json
├── accounts/<self_id>/<protocol_id>.json
├── groups/<self_id>/<protocol_id>/<group_id>.json
├── media/                 # 菜单图片、头像临时文件
├── backup/                # 迁移/恢复前自动备份
└── logs/                  # 可选运行日志（stderr 同时输出）
```

升级保留 `data/`；卸载默认保留；彻底删除由用户或框架执行。

### 2.11 构建与打包

同一源码交叉编译三个平台，`CGO_ENABLED=0`：

```bash
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build -trimpath   -ldflags "-s -w -X main.version=1.2.18 -X main.commit=$GIT_SHA -X main.buildTime=$BUILD_TIME"   -o dist/windows-amd64/qq-cloud-assistant.exe .

CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath   -ldflags "-s -w -X main.version=1.2.18 -X main.commit=$GIT_SHA -X main.buildTime=$BUILD_TIME"   -o dist/linux-amd64/qq-cloud-assistant .
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -trimpath   -ldflags "-s -w -X main.version=1.2.18 -X main.commit=$GIT_SHA -X main.buildTime=$BUILD_TIME"   -o dist/linux-arm64/qq-cloud-assistant .
```

打包要求：

```text
qq-cloud-assistant-1.2.18-windows-amd64.zip
├── mengka-plugin.json      # entry = qq-cloud-assistant.exe
├── qq-cloud-assistant.exe
├── README.txt
└── SHA256SUMS

qq-cloud-assistant-1.2.18-linux-amd64.tar.gz
qq-cloud-assistant-1.2.18-linux-arm64.tar.gz
├── mengka-plugin.json      # entry = qq-cloud-assistant
├── qq-cloud-assistant      # 0755
├── README.txt
└── SHA256SUMS
```

- 两个包分别使用对应平台的清单，不能共用 `entry`。
- 包根目录必须同时有清单和可执行文件。
- 不包含 `data/`、`admin.token`、数据库、账号信息。
- `scripts/build-all.sh` 与 `scripts/build-all.ps1` 负责构建、打包、生成校验和。
- 打包后校验：解压到空目录，导入框架，首次启动生成 token，控制台可登录。

---

## 3. 功能范围

### 3.1 统计口径

| 指标 | 数量 |
|---|---:|
| 源码注册类目 | 43（含通用 + 7 个占位） |
| 源码注册项 | 370（含 7 条占位） |
| 真实类目 | 38（37 个业务类目 + 通用；v1.1 新增授权、红包） |
| 真实指令 | 363 |
| V1 注册 | 333 |
| v1.1 新增 | 27（授权 16、红包 11） |
| v1.1 注册 | 360 |
| v1.2 新增 | 1（语音点歌） |
| v1.2 注册 | 382 |
| 不可用 | 29 |
| 可注册类目 | 35（基线 33 + 授权、红包；字符、被戳两类整体不注册） |
| 控制台目录项 | v1.2 为 59（基线 51 + 授权 4 + 红包 2 + 点歌线路 1 + 在线更新 1） |

### 3.2 完整支持

- 通用菜单、类目菜单、帮助、运行状态
- 基本群管、召唤、怼人、菜单配置、回复配置
- 防撤回、自身撤回、艾特处理、表情回应
- 点赞、互赞、续火、名片
- 备注、头衔、入群验证
- 问答、高级问答
- 定时、循环、群发
- 账号分组、主人、触发模式
- 自动同意好友、自动同意入群申请（不含邀请卡片）
- 自动回赞、新好友回复
- 自动发说说（文字）
- 秒赞、秒评（升级为真实评论）
- 打卡、一键打卡
- 红包查询/领取（普通、拼手气、口令、专属、语音）、期望过滤、到账回复、账本与排行
- 本地卡密、分群授权、代理配额、可选授权门禁（无联网校验）

### 3.3 降级支持

| 功能 | 降级方式 | 原因 |
|---|---|---|
| 查看禁言列表 | 只显示本插件记录的禁言成员 | 无全量禁言列表接口 |
| 解禁全部 | 遍历成员列表逐个解禁，限 500 人、1 秒/条 | 无批量接口 |
| 清屏 | 改为撤回缓存内最近消息，不再发送零宽字符刷屏 | 合规与风控 |
| 取直链 | 当前/被引用图片的 URL；语音视频仅提示不可直链 | 媒体只提供 `file_id`，URL 有时效 |
| 查共群 | 最多扫描 50 个群，2 秒/群，缓存 10 分钟 | 无专用接口 |
| 群发群名片 | `get_contact_ark`，失败回退群分享链接 | 卡片接口可能受限 |
| 修改头像 | 控制台/QQ 图片 → `file_transfer` → `set_qq_avatar`，5 MiB | 依赖媒体目录与 `file_transfer` |
| 修改昵称/个签 | Android 可写；Linux 只读并提示 | `profile_operation` 平台限制 |
| 打卡 | Android 可用；Linux 待核验 | `sign_group` 平台限制 |
| 秒赞/秒评 | 全局好友动态流过滤白名单，120 秒轮询 | 无按好友拉取接口 |
| 自动同意进群 | 只处理入群申请；好友邀请卡片不自动处理 | 无稳定邀请事件/`msg_seq` |
| 图片菜单 | `golang.org/x/image` 渲染；失败回退文字 | 需要文字渲染依赖 |
| 自动发说说 | 仅文字；图片发布不在 V1 范围 | 文本发布依赖 Android |
| 好友申请处理 | Android 可用；Linux 待核验 | 上游协议限制 |

### 3.4 不可用（29 条，不注册）

| 类目 | 指令 | 数量 | 原因 |
|---|---|---:|---|
| 字符功能 | 抽字符、一键抽群字符、停止抽群字符、一键抽好友字符、停止抽好友字符 | 5 | 官方 API 无抽字符接口 |
| 被戳处理 | 开启/关闭被戳回复、设置被戳冷却、查看/添加/删除被戳回复 | 6 | 官方事件无 poke 通知 |
| 自动处理 | 开启/关闭进群自动屏蔽、开启/关闭进群自动免打扰、开启/关闭进群自动群助手 | 6 | 无账号级群设置接口 |
| 提示功能 | 开启/关闭上管提示、开启/关闭下管提示、设置上管提示、设置下管提示 | 6 | 无管理员变更事件 |
| 个人助手 | 一键退出禁言群、一键退出全禁群、退出少于 N 人禁言群、退出少于 N 人全禁群 | 4 | 无禁言/全禁状态查询接口 |
| 主人菜单 | 检测更新、更新插件 | 2 | 无插件自更新接口 |

处理原则：**不注册、不进菜单、不显示可操作表单**。控制台可以显示“当前框架不可用 + 原因”，不能出现空 handler 或统一回复“暂未开放”。

### 3.5 整体移除

| 类目 | 原因 |
|---|---|
| 抢米功能、抢米设置、收款功能 | 涉及资金/支付 |
| 伪造聊天 | 伪造聊天记录，合规风险 |
| 监控功能（闪照监控） | 隐私风险 |

同时不实现：支付、订单、收款、提现、OCR、转图、二维码识别、绕过平台风控的批量协议包。卡密、分群授权、代理改为本地管理；红包改用官方 `red_packet` 消息段与官方领取接口。

### 3.6 能力矩阵

| 功能 | 状态 | 依赖 | 备注 |
|---|---|---|---|
| 群管基础 | 完整 | `set_group_mute`、`set_group_mute_all`、`set_group_admin`、`kick_group_member` | 需要机器人为管理 |
| 踢人/上下管/头衔/名片 | 完整 | 对应 action | 目标需在群 |
| 禁言列表 | 降级 | 本地 `mute_records` | 仅本插件记录 |
| 群星/群信息 | 完整/降级 | `get_group_honor_info` + `get_group_list` + `get_group_member_list` | 优先荣誉榜，失败回退群信息 |
| 取码 | 完整 | 事件 `msg.seq` / `msg.random` | 保留原语义 |
| 加群链接 | 完整 | `get_group_share_link` | 受群设置限制 |
| 主动退群 | 完整 | `leave_group` | 控制台操作，需二次确认；返回 submitted 不等于已完成 |
| 召唤 | 完整 | `send_group_msg` + `at` | 全体次数受 QQ 限制 |
| 清屏 | 降级 | `recall_group_msg` + 缓存 | 只撤回缓存内消息 |
| 怼人 | 完整 | `send_group_msg` + `at` | 高频风险，限时限量 |
| 查共群 | 降级 | 群列表 + 成员列表 | 限频 |
| 防撤 | 完整 | recall 事件 + `body.original` | 原文仅 5 分钟 |
| 自身撤回 | 完整 | 发送结果 + recall | 群/好友，不含临时会话 |
| 艾特处理 | 完整 | 群消息 + `at` | 好友不支持 At |
| 被戳回复 | 不可用 | 无 poke 事件 | 整类下线 |
| 秒赞/秒评 | 降级 | 空间动态 actions | Android，白名单 + 低频 |
| 点赞/互赞 | 完整 | `like_user_profile` | 次数受平台限制 |
| 表情回应 | 完整 | `set_msg_emoji_like` | 需消息 `message_id` |
| 字符 | 不可用 | 无接口 | 整类下线 |
| 续火 | 完整 | 发送 action | 注意频率 |
| 自动说说 | 降级 | `publish_qzone_feed` / `delete_qzone_feed` | 仅文字，Android |
| 自动回赞 | 完整 | `friend_notice.like` + `like_user_profile` | 事件可能缺失 |
| 新好友回复/备注 | 完整 | `friend_notice.increase` + `send_friend_msg` / `set_friend_remark` | 字段缺失则跳过 |
| 自动同意好友 | 完整 | `approve_friend_apply` | 需要 `action_info` |
| 自动同意进群 | 降级 | `approve_group_apply` | 仅申请，不含邀请卡片 |
| 进群屏蔽/免打扰/群助手 | 不可用 | 无接口 | 6 条下线 |
| 打卡 | 降级 | `sign_group` | Android |
| 昵称/个签 | 降级 | `profile_operation` | Linux 只读 |
| 头像 | 降级 | `file_transfer` + `set_qq_avatar` | 5 MiB |
| 备注 | 完整 | `set_friend_remark` | 仅好友 |
| 入群验证 | 完整 | 消息 + 群管 action | 重启会话丢失 |
| 问答 | 完整 | 消息事件 | 自身消息默认排除 |
| 定时/循环/群发 | 完整 | 发送 action | 受在线与风控限制 |
| 点歌（链接） | 完整 | 外部搜索线路 | 搜索失败回退搜索链接 |
| 点歌（卡片） | 完整/降级 | `get_music_ark` + 外部搜索 | 仅 QQ/网易云卡片；酷狗回退 |
| 点歌（语音） | 降级 | `upload_group_voice` / `upload_friend_voice` | 依赖外部音频直链 |
| 取直链 | 降级 | 事件图片 `url` / `get_msg` | 仅当条/引用 |
| 自更新 | 不可用 | 无接口 | 仅展示版本 |
| 红包查询/领取 | 完整 | `red_packet` 段 + 查询/领取/语音接口 | 五类独立开关；Android；语音需音频匹配并发送 |
| 本地卡密/分群授权/代理 | 完整 | 本地 SQLite | 无联网校验；门禁默认关闭 |

---

## 4. 权限系统

### 4.1 六级定义

| 级别 | 名称 | 来源 | 效果 |
|---|---|---|---|
| 0 | 黑名单 | 全局/群黑名单 | 不响应任何指令与交互式自动回复 |
| 1 | 群员 | 默认 | 仅响应下调到 Lv1 的指令 |
| 2 | 管理 | 事件 `sender.role=1` 或成员列表 | 响应 Lv2 及以下 |
| 3 | 群主 | 事件 `sender.role=2` 或成员列表 | 响应 Lv3 及以下 |
| 4 | 分管 | 群配置 `managers`（本地） | 响应 Lv4 及以下 |
| 5 | 主人 | 全局主人 / 账号主人 / 机器人自身 | 响应全部 |

### 4.2 判定顺序

```go
const (
    LevelUnset     = -1
    LevelBlacklist = 0
    LevelMember    = 1
    LevelAdmin     = 2
    LevelOwner     = 3
    LevelSubAdmin  = 4
    LevelMaster    = 5
)

func (e *Engine) Level(ev *Event) int {
    // 1. 机器人自身 = 主人，最高优先，免疫黑名单
    if ev.SenderID != 0 && ev.SenderID == ev.SelfID {
        return LevelMaster
    }
    // 2. 全局主人 / 当前账号主人
    if e.cfg.IsMaster(ev.AccountKey(), ev.SenderID, ev.SenderUID) {
        return LevelMaster
    }
    // 3. 全局黑名单
    if e.cfg.InGlobalBlacklist(ev.SenderID, ev.SenderUID) {
        return LevelBlacklist
    }
    // 4. 群黑名单 / 群分管
    if ev.GroupID != 0 {
        if eff, err := e.cfg.Effective(ev.AccountKey(), ev.GroupID); err == nil {
            if eff.Blacklist.Match(ev.SenderID, ev.SenderUID) {
                return LevelBlacklist
            }
            if eff.Managers.Match(ev.SenderID, ev.SenderUID) {
                return LevelSubAdmin
            }
        }
    }
    // 5. 私聊/临时会话非主人 = 群员
    if ev.GroupID == 0 {
        return LevelMember
    }
    // 6. 事件自带角色（数字；缺失时 RoleKnown=false，不能把零值当成员）
    if ev.RoleKnown {
        switch ev.SenderRole {
        case 2:
            return LevelOwner
        case 1:
            return LevelAdmin
        case 0:
            return LevelMember
        }
    }
    // 7. 查询成员列表；5 分钟缓存；失败 fail-closed 为 Lv1
    return e.memberLevel(ev.AccountKey(), ev.GroupID, ev.SenderID)
}
```

要点：

- `user_id` 与 `uid` 双轨匹配；事件缺 `user_id` 时用 `uin_map` 反查。
- 角色查询使用 `get_group_member_list`，按 `(AccountKey, group_id)` 缓存 5 分钟。
- 查询失败按 Lv1 处理，不误放行。
- 没有管理员变更事件，缓存主要靠 TTL 失效；成员增减事件可立即失效。
- 私聊/临时会话没有群角色，非主人一律 Lv1。
- 黑名单只拦指令与交互式自动回复，不拦入群验证、欢迎、退群、撤回通知。

### 4.3 默认仅主人

- 注册器内部用 `LevelUnset = -1` 表示“未设置”，注册时归一化为 `LevelMaster`。
- **禁止**用 `0` 表示未设置；`0` 是黑名单。
- 可配置的指令级别范围是 **1–5**；`0` 不能分配给指令，否则黑名单用户会通过 `Level >= MinLevel` 判定。
- 子指令未设置继承父类目；父类目未设置则为 5。
- 菜单、帮助、运行状态同样受权限控制。
- 不提供 `AlwaysAvailable` 权限旁路。
- 控制台首次启动时 `command_permissions` 为空，全部按 5 处理。
- 所有指令注册默认 Lv5；如需群主/代理使用，由主人在控制台下调对应指令级别。红包/授权/点歌等新增指令不预设低级默认。

### 4.4 覆盖优先级

```text
群级覆盖 > 全局指令 > 全局类目 > 注册默认值 5
```

示例 `data/permissions.json`：

```json
{
  "菜单": 1,
  "帮助": 1,
  "运行状态": 3,
  "全体禁言": 2,
  "本群打卡": 1,
  "申请头衔": 1,
  "加入互赞": 1,
  "退出互赞": 1,
  "点歌": 1,
  "category:qa": 2,
  "category:checkin": 1,
}
```

- 控制台可按类目批量调整，也可单条覆盖。
- 群配置页可按群覆盖。
- 所有修改写 `audit_log`。
- 权限缓存 30 秒失效；配置写入后立即刷新。

### 4.5 推荐权限预设

为保证“默认仅主人”又不影响成员功能，控制台提供三档预设：

| 预设 | 规则 |
|---|---|
| 严格 | 全部 5 |
| 推荐 | 菜单/帮助 1；运行状态 3；本群打卡 1；申请头衔 1；加入/退出互赞 1；点歌 1；查共群 3；查询群星 3；其余 5；红包/授权/点歌新增指令保持 5 |
| 自定义 | 用户逐条调整 |

预设只写全局指令/类目权限，不写群覆盖；执行前二次确认，执行后写审计。

### 4.6 自动功能不受指令权限限制

以下逻辑由“功能开关 + 名单”驱动，普通群员也能触发其效果：

- 入群欢迎、退群/被踢提示；
- 入群验证；
- 防撤回；
- 问答自动回复；
- 艾特处理、表情回应；
- 秒赞/秒评、自动回赞；
- 定时/循环/群发；
- 打卡、续火、自动说说；
- 红包自动领取（受账号/群开关、监听名单与授权门禁控制，不受指令级别控制）；
- 自动同意好友、自动同意入群申请（不含邀请卡片）。

但“配置这些功能”的指令仍受六级权限控制。

### 4.7 自身消息与环路防护

- 若框架推送机器人自身消息，按 Lv5 处理，不丢弃。
- 自身消息可以触发指令，但若与最近 10 秒内本插件发送内容指纹一致，则整条忽略，避免菜单/帮助被回显后再触发。
- 自身消息与普通消息共用类目 Fallback：发送一级类目名（如 `通用功能`、`点歌功能`）返回对应子菜单，不受自身消息提前返回影响；无可见指令时静默，不回“该类目暂无可用指令”。
- 自身消息**不得**进入问答、艾特回复、表情回应、防撤转发、自动欢迎、自动回赞、秒赞/秒评。
- 红包自动领取例外：自身消息通过 `RegisterSelfMessageHook` 进入红包 Hook；`red_packet.grab_self` 默认 true 时会领取自己发送的群红包，关闭后记录“跳过·自己发送”。
- 控制台提供 `self_trigger_auto`，默认关闭；开启时：
  - 3 秒内相同内容去重；
  - 输出不得包含触发词；
  - 每群每分钟最多 5 条自动回复。
- 群发、循环、定时、续火、说说发出的消息带内部任务标记，不触发自动功能。

### 4.8 缓存与失效

- 角色缓存：`(AccountKey, group_id)` → 群主/管理员集合，TTL 5 分钟。
- 成员增减事件立即失效；禁言事件不影响角色。
- 手动“刷新成员”可从控制台触发失效。
- 群列表缓存 5 分钟；控制台刷新按钮强制拉取。
- 配置快照按文件 mtime 失效；写入后立即失效。

---

## 5. 指令清单

> 说明：`@` 表示一个 At 段，`N` / `秒` / `A` 表示参数位，`X` 表示任意文本。
> 所有指令默认 Lv5；可在控制台下调到 1–5。
> 下表“注册”为 V1 实际注册数量。

### 5.1 通用功能（8）

| 指令 | 说明 |
|---|---|
| 菜单 / 功能 / menu | 功能菜单索引，按权限过滤 |
| 菜单 <类目名> | 类目下全部有权指令 |
| 帮助 | 用法提示 |
| 帮助 <指令> | 指令用法、所需权限、分类 |
| 运行状态 / 状态 / status | 在线账号、好友数、群数、版本、运行时长、收发统计 |
| 在线更新 | 当前版本、最新版本、状态、更新源与控制台入口 |
| 检查更新 | 检查 GitHub/清单最新版本并显示更新说明 |
| 立即更新 | 托管环境尝试安装；需管理员验证码时提示打开控制台 |

### 5.2 群管与互动（39）

| 类目 | 注册 | 指令 |
|---|---:|---|
| 基本群管 basic | 14 | `踢 @`、`解禁 @成员` / `解禁 QQ`、`禁言 @成员 秒` / `禁言 QQ 秒`、`上管 @`、`下管 @`、`全体禁言`、`全体解禁`、`解禁全部`、`查看禁言列表`、`查询群星`、`取码`、`复读 <文本>`、`取直链`、`生成加群链接-<群号>` |
| 召唤功能 summon | 4 | `召唤全体`、`召唤管理`、`#单个召唤 <QQ>`、`设置召唤人数 <N>` |
| 清屏功能 clearscreen | 6 | `开启清屏功能`、`关闭清屏功能`、`定海神针`、`清洗屏幕`、`花样清屏`、`轰炸清屏` |
| 怼人功能 duel | 6 | `怼他 @`、`停止怼他`、`停止怼他 @`、`开启怼人艾特`、`关闭怼人艾特`、`设置怼人间隔 <秒>` |
| 个人助手 assistant | 1 | `查共群 <QQ>`（其余 4 条不可用） |
| 菜单配置 menu_config | 4 | `设置菜单头 <X>`、`设置菜单尾 <X>`、`设置召唤命令 <X>`、`查看菜单配置` |
| 回复配置 reply_config | 4 | `切换文字模式`、`切换图片模式`、`开启静默模式`、`关闭静默模式` |

要点：

- `查询群星` 优先调用 `get_group_honor_info` 展示荣誉榜；失败时回退群信息（群号、群名、人数、群主）。
- `查看禁言列表` 只显示本插件记录的禁言成员。
- `解禁全部` 遍历成员列表逐个解禁，最多 500 人，间隔 1 秒。
- `清屏` 为撤回缓存内最近消息；不再发送零宽字符。
- `怼他` 默认 3 秒间隔，单次最多 10 分钟/200 条；`停止怼他` 立即停止。
- `设置召唤人数` 限制 `召唤管理` 最多 At 的管理员数量。
- `切换图片模式` 将菜单索引/类目菜单渲染为 PNG；上传失败回退文字。
- 控制台 `basic` 模块提供手动退群操作；QQ 指令不注册退群，避免误操作。

### 5.3 提示功能（注册 15 / 总 21）

| 类型 | 指令 |
|---|---|
| 启用/停用 | `开启/关闭入群提示`、`开启/关闭退群提示`、`开启/关闭被踢提示`、`开启/关闭被删通知`、`开启/关闭被踢通知` |
| 模板 | `设置入群提示 X`、`设置退群提示 X`、`设置被踢提示 X`、`设置被删通知 X`、`设置被踢通知 X` |
| 不可用 | `开启/关闭上管提示`、`开启/关闭下管提示`、`设置上管提示 X`、`设置下管提示 X` |

模板变量：`{nick}`、`{uin}`、`{uid}`、`{group}`、`{group_name}`、`{operator}`、`{target}`、`{self_id}`、`{protocol_id}`、`{time}`。

被删通知只表示“好友关系被移除”，不能判断是谁删除。

### 5.4 自动化（注册 144 / 总 161）

| 类目 | 注册 | 指令 |
|---|---:|---|
| 防撤功能 anti_recall | 11 | `防撤加白 <群号>`、`防撤删白 <群号>`、`查看防撤白名单`、`清空防撤白名单`、`开启/关闭本群防撤`、`开启/关闭全局防撤`、`开启/关闭好友防撤`、`查看防撤配置` |
| 自身撤回 self_recall | 4 | `开启自身撤回`、`关闭自身撤回`、`设置撤回时间 <秒>`、`查看撤回配置` |
| 艾特处理 at_handler | 20 | `查看艾特回复`、`删除艾特回复 <序号>`、`添加艾特回复 <内容>`、`查看艾特监控`、`清空艾特监控`、`删除艾特监控 <群号>`、`添加艾特监控 <群号>`、`设置艾特回复间隔 <秒>`、`设置艾特冷却时间 <秒>`、`设置艾特禁言时间 <秒>`、`开启/关闭艾特我回复`、`开启/关闭艾特我禁言`、`开启/关闭艾特几率回复`、`开启/关闭全局艾特监控`、`开启/关闭艾特私聊通知` |
| 秒赞秒评 auto_like | 11 | `开启/关闭秒赞`、`开启/关闭秒评`、`查看秒评内容`、`删除秒评内容 <编号>`、`添加秒评内容 <内容>`、`秒赞加白 <QQ>`、`秒赞删白 <QQ>`、`查看秒赞白名单`、`清空秒赞白名单` |
| 点赞功能 like_card | 4 | `一键好友全赞`、`设置点赞数量 <N>`、`设置点赞好友间隔 <毫秒>`、`设置点赞好友数量 <N>` |
| 互赞功能 mutual_like | 3 | `一键互赞`、`加入互赞`、`退出互赞` |
| 回应表情 emoji | 12 | `开启/关闭回应表情`、`开启/关闭回应指定群`、`查看/清空/添加/删除回应白名单`、`查看/清空/添加/删除回应指定群` |
| 续火功能 keep_fire | 17 | `一键续火`、`开启/关闭续火功能`、`设置续火间隔 <分钟>`、`设置续火时间 <HH.MM>`、`查看/清空/添加/删除续火内容`、`查看/清空/添加/删除续火群号`、`查看/清空/添加/删除续火好友` |
| 自动说说 mood | 8 | `查看说说内容`、`删除说说内容 <编号>`、`添加说说内容 <内容>`、`设置发布间隔 <分钟>`、`开启/关闭删除上一条`、`开启/关闭自动发说说` |
| 自动处理 auto_process | 13 | `开启/关闭自动回赞`、`开启/关闭回赞回复`、`开启/关闭新好友回复`、`开启/关闭自动同意好友`、`开启/关闭自动同意进群`、`设置回赞回复 <内容>`、`设置新好友回复 <内容>`、`查看自动处理` |
| 名片功能 card | 2 | `设置修改间隔 <秒>`、`一键修改群名片 <名片>` |
| 打卡功能 checkin | 9 | `本群打卡`、`一键群打卡`、`开启/关闭0点打卡`、`打卡加白 <QQ>`、`打卡删白 <QQ>`、`查看打卡白名单`、`清空打卡白名单`、`设置打卡间隔 <毫秒>` |
| 资料设置 profile | 3 | `修改昵称 <昵称>`、`修改个签 <签名>`、`修改头像` |
| 备注功能 remark | 4 | `开启/关闭自动备注`、`设置备注回复 <模板>`、`查看备注配置` |
| 头衔功能 title | 11 | `置头衔 @ A`、`申请头衔 <内容>`、`开启/关闭自助头衔`、`开启/关闭无头衔撤回`、`查看/添加/删除头衔违禁词`、`清空头衔次数 @`、`设置自助头衔次数 <N>` |
| 入群验证 verify | 12 | `开启/关闭入群验证`、`切换数字/字母/混合/发言验证`、`切换未验证踢出/禁言/不处理`、`设置验证时间 <秒>`、`设置未验证禁言 <分钟>`、`查看验证配置` |
| 被戳处理 poke | 0 | 6 条不可用 |
| 字符功能 lucky_word | 0 | 5 条不可用 |

关键行为：

- 防撤白名单是**豁免名单**：名单内群不执行防撤，避免刷屏。
- 全局防撤对所有非豁免群生效；本群开关优先于全局开关。
- 好友防撤只处理好友撤回事件，原文优先 `body.original`。
- 自身撤回只支持群/好友，不含临时会话；失败只记日志。
- 艾特处理仅在群聊生效；`@全体` 不算艾特机器人。
- 秒赞/秒评轮询全局好友动态，按白名单 `user_id` 过滤，同一 `feed_id` 去重。
- 回应表情使用 `set_msg_emoji_like`，表情 ID 池可在控制台配置。
- 续火时间与间隔二选一；设置了时间后按每日时间执行。
- 自动说说只发文字；`删除说说内容 <编号>` 删除内容池条目，删除已发布动态走“删除上一条”或控制台指定 `tid`。
- 自动同意好友必须带回事件里的 `action_info`；自动同意进群只处理申请。
- 无头衔撤回：群消息发送者 `title` 为空且开关开启时撤回该消息。
- 申请头衔受自助开关、违禁词、次数配额限制。

### 5.5 问答（56）

| 类目 | 注册 | 指令 |
|---|---:|---|
| 问答功能 qa | 30 | `开启/关闭本群问答`、`开启/关闭好友问答`、`开启/关闭全局问答`；本群/好友/全局 × 精准/模糊 的 `添加/删除/查看/清空`，共 24 条 |
| 高级问答 qa_advanced | 26 | 本群/好友问答白名单 8 条；本群/全局/好友问答延迟 6 条；随机回复条数 6 条；精准/模糊问冷却 6 条 |

匹配规则：

- 同一作用域精准优先于模糊；
- 群消息：本群规则 → 全局规则；私聊：好友规则 → 全局规则；
- 回复从池中随机，支持模板变量；
- 冷却按 `(AccountKey, 作用域, group_id, keyword)` 计；
- 自身消息默认不触发；黑名单不触发；
- 指令匹配优先于问答兜底。

### 5.6 任务、群发与点歌（54）

| 类目 | 注册 | 指令 |
|---|---:|---|
| 定时功能 timer | 15 | `开启/关闭定时任务`、`清空定时任务`、`查看定时任务`、`查看定时详情 <ID>`、`删除定时任务 <ID>`、`设置任务时间 #ID#时间`、`设置任务对象 #ID#对象`、`设置任务内容 #ID#内容`、`设置任务间隔 #ID#间隔`、`添加定时禁言 #时间#群号`、`添加定时解禁 #时间#群号`、`添加定时群打卡 #时间#群号`、`添加定时群发群 #时间#群号#内容`、`添加定时群发好友 #时间#账号#内容` |
| 循环功能 loop | 14 | `开启/关闭循环任务`、`开启/关闭循环通知`、`清空循环任务`、`删除循环任务 <ID>`、`查看循环任务`、`查看循环详情 <ID>`、`设置循环间隔 #ID#分钟`、`设置循环群号 #ID#群号`、`设置循环内容 #ID#内容`、`设置循环分类 #ID#分类`、`设置循环类型 #ID#类型`、`添加循环任务 #间隔#群号#内容` |
| 群发功能 broadcast | 14 | `加白本群`、`停止群发 <ID>`、`开始群发群 <内容>`、`开始群发好友 <内容>`、`开始群发群名片 <群号>`、`设置群发间隔 <秒>`、`加白群/删白群/查看群白名单/清空群白名单`、`加白好友/删白好友/查看好友白名单/清空好友白名单` |
| 点歌功能 music | 11 | `点歌 [QQ/网易云/酷狗] <歌名>`（别名 `搜歌`）、`切换链接点歌`、`切换卡片点歌`、`切换语音点歌`、`切换QQ音乐`、`切换酷狗音乐`、`切换网易云音乐`、`开启/关闭点歌功能`、`开启/关闭点歌选歌` |

要点：

- 定时“禁言/解禁”是**全体禁言/解禁**，不是单人禁言。
- 时间格式 `HH.MM`；间隔单位为分钟；目标多个用 `*` 分隔（兼容 `,`）。
- 循环分类/类型是任务元数据，用于控制台筛选与分组，不改变发送行为。
- 群发群目标为全部群，排除 `mass_group_wl`；群发好友只发 `mass_friend_wl`。
- 群发群名片使用 `get_contact_ark`，失败回退分享链接。
- `点歌功能` 是类目菜单，类目名优先于指令前缀，不会被当成“点歌 功能”搜索；搜索用 `点歌 <歌名>` 或 `搜歌 <歌名>`。
- 点歌先按线路表搜索；链接/卡片/语音三种回复模式；酷狗无官方卡片，卡片失败回退链接；语音失败回退卡片再回退链接。
- 点歌选歌开启时，返回最多 5 条并等待 60 秒数字选择；关闭时发送第一条。

### 5.7 账号与主人（注册 21 / 总 23）

| 类目 | 注册 | 指令 |
|---|---:|---|
| 主人菜单 owner | 11 | `删除异常`、`重载插件`、`恢复默认菜单`、`查看主群列表`、`添加主群 <群号>`、`删除主群 <群号>`、`开启/关闭调试模式`、`查看全局主人`、`添加全局主人 <QQ>`、`删除全局主人 <QQ>`（检测更新/更新插件 2 条不可用） |
| 触发模式 trigger | 5 | `查看主人账号`、`添加主人 <QQ>`、`删除主人 <QQ>`、`开启/关闭主人触发` |
| 账号分组 account_group | 5 | `查看账号分组`、`删除账号分组 <名称>`、`查看分组账号 <名称>`、`添加分组账号 <名称> <QQ>`、`删除分组账号 <名称> <QQ>` |

要点：

- `删除异常`：清理运行时错误计数、过期权限/群缓存，并写审计。
- `重载插件`：重载配置、权限、能力快照，不重启进程。
- 主群作为主人通知的默认落点。
- 账号分组用于控制台批量选择与任务筛选。
- `关闭主人触发` 后，账号主人不再自动获得 Lv5；全局主人与机器人自身不受影响。

### 5.8 指令统计

| 类目 | 注册 | 类目 | 注册 |
|---|---:|---|---:|
| common | 5 | qa | 30 |
| basic | 14 | qa_advanced | 26 |
| summon | 4 | timer | 15 |
| clearscreen | 6 | loop | 14 |
| duel | 6 | broadcast | 14 |
| assistant | 1 | music | 11 |
| notice | 15 | owner | 11 |
| menu_config | 4 | trigger | 5 |
| reply_config | 4 | account_group | 5 |
| anti_recall | 11 | auto_like | 11 |
| self_recall | 4 | like_card | 4 |
| at_handler | 20 | mutual_like | 3 |
| emoji | 12 | keep_fire | 17 |
| mood | 8 | auto_process | 13 |
| card | 2 | checkin | 9 |
| profile | 3 | remark | 4 |
| title | 11 | verify | 12 |
| **合计** | **333** | | |

> `poke`、`lucky_word` 注册 0，未在表中列出；上表为 v1.0 基线 333 条。v1.2 另增授权 16 条、红包 29 条、语音点歌 1 条、在线更新 3 条，合计 382 条。

### 5.9 v1.1 新增指令（27）

新增两个类目：`license`（授权功能，16 条）与 `redpacket`（红包功能，27 条）。**全部默认 Lv5**；如需群主兑换或代理生成，由主人在控制台下调级别。处理器仍会校验主人/代理身份。

#### 授权功能（16）

| 指令 | 级别 | 说明 |
|---|---|---|
| `生成卡密 <天数> <数量> [备注]` | Lv5 | 主人或代理生成；代理消耗配额 |
| `查询卡密 [卡密]` | Lv5 | 留空查询最近卡密；代理只能看自己的 |
| `删除卡密 <卡密>` | Lv5 | 仅未使用卡密可删 |
| `禁用卡密 <卡密>` | Lv5 | 仅未使用卡密可禁用 |
| `启用卡密 <卡密>` | Lv5 | 恢复被禁用卡密 |
| `使用卡密 <卡密>`（别名 `兑换卡密`、`授权本群`） | Lv5 | 仅群聊；为本群开通/续期 |
| `查询授权 [群号]` | Lv5 | 群主只能查本群；主人可指定群 |
| `授权群 <群号> <天数> [备注]` | Lv5 | 手动授权 |
| `撤销授权 <群号>` | Lv5 | 撤销后不删除历史 |
| `授权列表` | Lv5 | 当前账号分群授权 |
| `添加代理 <QQ> <名称> [联系方式]` | Lv5 | 新增/更新代理 |
| `代理列表` | Lv5 | 配额、生成、兑换统计 |
| `删除代理 <QQ>` | Lv5 | 不删除其历史卡密 |
| `设置代理配额 <QQ> <数量>` | Lv5 | 0 表示不限 |
| `启用代理 <QQ>` | Lv5 | 恢复生成权限 |
| `停用代理 <QQ>` | Lv5 | 暂停生成权限 |

#### 红包功能（29）

所有红包指令带 `Requires: action:grab_red_packet`；节点未提供该 action 时整组不注册。

| 指令 | 级别 | 说明 |
|---|---|---|
| `开启红包助手` | Lv5 | 开启本群并自动加入监听名单；主人可同时打开账号总开关 |
| `关闭红包助手` | Lv5 | 关闭本群自动领取，保留监听名单 |
| `监听本群红包` | Lv5 | 加入红包监听名单 |
| `取消监听本群红包` | Lv5 | 移出监听名单 |
| `设置红包延迟 <毫秒>` | Lv5 | 0–10000，本群覆盖 |
| `设置红包门槛 <分>` | Lv5 | 0–100000，按每份期望过滤 |
| `设置红包回复 <文本>` | Lv5 | 最长 200 字，本群覆盖 |
| `红包状态` | Lv5 | 开关、监听、延迟、门槛、回复、今日数据、授权状态 |
| `红包排行 [今日/7天/30天/全部]` | Lv5 | 按群检测数量 |
| `查询红包记录 [数量]` | Lv5 | 1–30 条，展示结果与耗时 |
| `清空红包记录` | Lv5 | 清空当前账号账本，写审计 |
| `开启/关闭普通红包` | Lv5 | 账号级普通红包类型开关 |
| `开启/关闭拼手气红包` | Lv5 | 账号级拼手气红包类型开关 |
| `开启/关闭口令红包` | Lv5 | 账号级口令红包类型开关 |
| `开启/关闭专属红包` | Lv5 | 账号级专属红包类型开关 |
| `开启/关闭语音红包` | Lv5 | 账号级语音红包类型开关；默认内置 AI 语音，关闭发送时跳过 |
| `开启/关闭领取自己红包` | Lv5 | 默认开启；自己发送的红包也会进入领取流程 |
| `添加红包关键词 <关键词>` | Lv5 | 标题必须包含任意包含词才领取 |
| `删除红包关键词 <关键词>` | Lv5 | 删除标题包含规则 |
| `添加红包排除词 <关键词>` | Lv5 | 标题命中排除词时跳过，防止误抢 |
| `删除红包排除词 <关键词>` | Lv5 | 删除标题排除规则 |
| `查看红包关键词` | Lv5 | 查看包含/排除规则 |
| `清空红包关键词` | Lv5 | 删除全部标题关键词规则 |

控制台特殊操作（生成卡密、卡密状态、手动授权/撤销）不计入指令数，直接在本地数据库执行并写审计。

---

## 6. Web 控制台

### 6.1 形态

- 插件自带 HTTP 服务，默认 `127.0.0.1:18084`。
- 端口被占用时依次尝试 18085–18093，并把实际端口写入 `data/global.json` 的 `web.port`，同时输出到 stderr。
- 前端为原生 HTML/CSS/JS，`embed.FS` 内嵌，不使用构建工具、不使用外部 CDN。
- 框架令牌/服务令牌只保存在后端；前端不接触任何框架凭据。
- 控制台自建登录、会话、CSRF、权限校验。
- 支持 `--console-only` 启动：不接 IPC，只启动控制台，用假数据/只读配置做 UI 开发与验收。

### 6.2 布局与视觉

参考 `group_management` 的 QSign 主题：

```text
┌──────────────┬────────────────────────────────────────────┐
│  QQ云助手     │  面包屑                     admin  [退出] │
│              ├────────────────────────────────────────────┤
│  概览         │  标签页                                     │
│  机器人       │                                            │
│  群管理       │  卡片 / 表单 / 表格 / 操作按钮              │
│  自动化       │                                            │
│  问答         │                                            │
│  任务与群发   │                                            │
│  权限         │                                            │
│  系统         │                                            │
└──────────────┴────────────────────────────────────────────┘
```

视觉令牌（与参考控制台一致）：

| 变量 | 值 |
|---|---|
| 侧栏背景 `--qsign-bg` | `hsl(222.34 10.43% 12.27%)` |
| 工作区背景 `--qsign-bg-deep` | `hsl(220 13.06% 9%)` |
| 前景 `--qsign-foreground` | `hsl(0 0% 95%)` |
| 次要文字 `--qsign-muted` | `hsl(240 5% 64.9%)` |
| 边框 `--qsign-border` | `hsl(240 3.7% 22%)` |
| 强调底 `--qsign-accent` | `hsl(216 5% 19%)` |
| 主色 `--qsign-primary` | `hsl(212 100% 45%)` |
| 主色悬停 | `hsl(212 100% 51%)` |
| 主色按下 | `hsl(212 100% 39%)` |
| 成功 `--qsign-success` | `hsl(144 57% 58%)` |
| 警告 `--qsign-warning` | `hsl(42 84% 61%)` |
| 危险 `--qsign-danger` | `hsl(348 100% 61%)` |
| 圆角 | `8px` |
| 控件高度 | `32px` |
| 侧栏宽度 | 展开 `224px` / 收起 `64px`；移动端抽屉 `min(280px, 86vw)` |
| 顶栏高度 | `50px` |
| 标签栏高度 | `38px` |
| 基础字号 | `14px` |

响应式与交互：

- 桌面侧栏支持 224px ↔ 64px 平滑收起/展开，收起后显示首字图标与悬浮标题；
- ≤900px 侧栏改为抽屉 + 半透明遮罩，菜单按钮控制打开/关闭，点击导航或遮罩自动收起；
- 头部操作区允许换行，页面主体紧随其后，窗口缩放不遮挡内容；
- 表格在窄屏保持表头不换行并在容器内横向滚动，卡片、表单、指标网格自动降列；
- 滚动条使用透明轨道 + 低对比圆角滑块，支持 WebKit 与 Firefox。

登录页：

- 左侧品牌区：Logo、QQ云助手、说明、能力标签（Windows/Linux、IPC、本地卡密）；
- 右侧登录卡：固定用户名 `admin`、管理令牌、登录按钮、错误提示；
- 首次启动提示“令牌保存在数据目录 admin.token，仅显示一次”；
- 不出现联网授权码、应用 ID、机器码；卡密仅在“授权 → 卡密管理”本地生成与展示。

### 6.3 账号/协议/群选择器

这是多账号多协议控制台的关键：

- 顶栏或页面工具栏提供账号选择器：`昵称（QQ / 协议ID）`。
- 群级模块提供群选择器：`群名（群号）`。
- 表格/弹窗中的群号输入使用可联想下拉：选项来自当前选中账号的群列表，支持按群名或群号搜索；循环任务支持多选，保存为 `*` 分隔。单群字段也允许在群列表缺失时回填数字群号。
- 主人/账号主人表格仅显示与填写 QQ：后台统一保存为 `{"uin": QQ}`，不再要求 uid、note。
- 选择结果写入 `sessionStorage`，刷新后保持。
- 每个 API 请求显式带 `self_id`、`protocol_id`、`group_id`。
- 同一 QQ 的 Android 与 Linux/PC 同时在线时，默认仅 Android 实例处理事件；Linux/PC 实例静默，Android 离线后自动恢复。账号级开关 `multi_protocol_android_only` 默认开启。
- 账号离线时仍可查看配置；群列表显示最近缓存。

### 6.4 目录协议

`GET /api/admin/catalog`：

```json
{
  "code": 0,
  "data": {
    "sections": [
      {
        "key": "automation",
        "title": "自动化",
        "modules": [
          {
            "id": "anti-recall",
            "title": "防撤功能",
            "description": "本群/全局/好友防撤回，支持白名单与原文转发",
            "scope": "group",
            "kind": "mixed",
            "available": true,
            "fields": [
              {"key": "group_enabled", "label": "本群防撤", "type": "switch", "default": false},
              {"key": "global_enabled", "label": "全局防撤", "type": "switch", "default": false},
              {"key": "friend_enabled", "label": "好友防撤", "type": "switch", "default": false},
              {"key": "repost", "label": "转发被撤回原文", "type": "switch", "default": true}
            ],
            "tables": [
              {"key": "whitelist", "title": "白名单", "columns": ["target", "note", "created_at"]}
            ],
            "actions": []
          },
          {
            "id": "poke",
            "title": "被戳处理",
            "description": "当前框架未提供 poke 通知事件",
            "scope": "account",
            "kind": "disabled",
            "available": false,
            "reason": "缺少 poke 事件",
            "fields": [],
            "tables": [],
            "actions": []
          }
        ]
      }
    ]
  }
}
```

`scope`：`global` / `account` / `group`。

`kind`：

| kind | 渲染 |
|---|---|
| `form` | 字段分组表单 + 保存 |
| `table` | 工具栏 + 表格 + 分页 + 行编辑 |
| `mixed` | 上方表单 + 下方一个或多个表格 |
| `actions` | 按钮网格，危险操作二次确认 |
| `accounts` | 账号卡片/表格 + 刷新 |
| `logs` | 日志表格 + 筛选 + 导出 |
| `dashboard` | 指标卡片 + 图表 |
| `redpacket` | 红包统计卡 + 运行概况 + 群排行 + 处理记录 + 设置 + 名单表格 |
| `disabled` | 只读说明 + 能力缺失原因 |

字段类型：`text`、`textarea`、`number`、`switch`、`select`、`tags`、`password`、`url`、`time`、`image`、`json`、`qq`、`duration`。

字段属性：`key`、`label`、`type`、`placeholder`、`options`、`min`、`max`、`rows`、`required`、`help`、`default`。

### 6.5 模块清单（59）

#### 概览（2）

| 模块 | 类型 | 内容 |
|---|---|---|
| dashboard | dashboard | 账号卡片、好友/群数逐账号 `get_bot_info` 刷新、今日消息/指令/自动回复/失败、近 7 日趋势、最近操作 |
| runtime | dashboard | 运行时长、Worker 数、队列积压、action 延迟、错误数、框架版本、协议版本 |

#### 机器人（5）

| 模块 | 类型 | 内容 |
|---|---|---|
| bots | accounts | `self_id`、`protocol_id`、协议平台、昵称、在线状态、群数、好友数、今日消息；刷新 |
| bot-settings | form | 菜单头尾、召唤命令、调试模式、自身触发自动回复、权限拒绝提示、发送限速 |
| trigger | form + table | 主人触发开关、账号主人列表（仅填写 QQ） |
| main-groups | table | 主群列表（群号联想选择） |
| account-groups | table | 账号分组（名称、账号键列表） |

#### 群管理（13）

| 模块 | 类型 | 内容 |
|---|---|---|
| groups | table + 详情 | 群号、群名、总开关、跟随默认、成员数、今日消息 |
| group-config | form（分群） | 基础开关、欢迎、退群、审批、提示模板、验证、权限覆盖；顶部展示可用模板变量 |
| managers | table | Lv4 分管（仅 QQ） |
| blacklist | table | 全局/本群黑名单（仅 QQ） |
| whitelist | table | 各功能白名单，按 kind 过滤 |
| notice | form | 5 类提示开关 + 5 个模板；上管/下管只读说明 |
| menu-config | form + preview | 菜单头、菜单尾、召唤命令、实时预览 |
| reply-config | form | 文字/图片模式、静默模式 |
| basic | actions | 踢人、禁言、解禁、上下管、全体禁言、解禁全部、取码、复读、加群链接、退群 |
| summon | form + actions | 召唤人数、召唤全体/管理 |
| clearscreen | form + actions | 清屏开关、撤回条数、清洗/花样/轰炸 |
| duel | form + actions | 怼人艾特开关、间隔、开始/停止 |
| assistant | actions | 查共群；自动退群按钮只读说明 |

#### 自动化（18）

| 模块 | 类型 | 关键内容 |
|---|---|---|
| anti-recall | mixed | 本群/全局/好友开关、原文转发、白名单 |
| self-recall | form | 开关、延迟秒数 |
| at-handler | mixed | 回复/禁言/几率回复开关、冷却、间隔、禁言时长、回复池、监控群、私聊通知、几率 |
| auto-like | mixed | 秒赞/秒评开关、白名单、轮询间隔、评论池 |
| like-card | form + action | 点赞数量、好友间隔、好友数量、一键全赞 |
| mutual-like | actions | 加入/退出/一键互赞 |
| emoji | mixed | 开关、白名单、指定群、表情 ID 池、概率 |
| keep-fire | mixed | 开关、间隔/时间、内容池、群名单、好友名单、一键续火 |
| mood | mixed | 开关、间隔、删除上一条、内容池、立即发布、删除指定动态 |
| auto-process | form | 回赞、回赞回复、新好友回复、自动同意好友/进群、两段模板；3 组不可用开关只读 |
| card | form + action | 修改间隔、一键改名片 |
| checkin | mixed | 0 点打卡、间隔、白名单、本群打卡、一键群打卡 |
| profile | actions | 修改昵称、修改个签、上传头像 |
| remark | form | 开关、备注模板 |
| title | mixed | 自助头衔、无头衔撤回、违禁词、次数、清空次数 |
| verify | form | 开关、模式、超时、未验证处理、禁言时长、当前验证会话 |
| poke | disabled | 平台无 poke 通知，只读说明 |
| lucky-word | disabled | 平台无抽字符接口，只读说明 |

#### 问答（2）

| 模块 | 类型 | 关键内容 |
|---|---|---|
| qa | table | 作用域、类型、关键词、回复、冷却、启停 |
| qa-advanced | mixed | 白名单、本群/全局/好友延迟、随机条数、关键词冷却 |

#### 任务与群发（5）

| 模块 | 类型 | 关键内容 |
|---|---|---|
| timer | table | 类型、时间、对象、内容、间隔、下次执行、启停 |
| loop | table | 分类、类型、间隔、群号、内容、循环通知、启停 |
| broadcast | table + actions | 模式、目标、内容、间隔、进度、开始/停止、群/好友白名单 |
| music | form | 开关、模式（链接/卡片/语音）、默认来源、音质、搜索、选歌、失败换源、结果数 |
| music-lines | table | 线路名称、来源、格式、接口、来源参数、音质、优先级、启停、备注 |

#### 红包（2）

| 模块 | 类型 | 关键内容 |
|---|---|---|
| red-packet | redpacket | 五类独立开关、Android 限制、每分钟限流、语音发送开关、音频来源、延迟/抖动、期望门槛、查询策略、到账回复、每日上限、异常通知、账本保留；统计卡、运行概况、群排行、记录、监听群、只领不回复、抢包规则 |
| red-packet-group | form | 本群启用、延迟/抖动、门槛、领取前查询、查询失败策略、到账回复/延迟、每日上限；按群覆盖账号默认 |

#### 授权（4）

| 模块 | 类型 | 关键内容 |
|---|---|---|
| license-cards | mixed + actions | 卡密列表（卡密/套餐/天数/状态/代理/使用群/到期）、生成卡密、禁用、启用、删除 |
| group-authorizations | mixed + actions | 分群授权列表（群号/群名/状态/累计天数/到期/操作人/来源）、手动授权、撤销 |
| agents | table | 代理 QQ、名称、联系方式、状态、配额/已用、生成/兑换、备注；新增/编辑/删除 |
| license-settings | form | 授权门禁开关、默认套餐、未授权提示 |

#### 权限（2）

| 模块 | 类型 | 关键内容 |
|---|---|---|
| masters | table | 全局主人、账号主人（仅填写 QQ） |
| command-permissions | table | 类目/指令、默认级别、全局级别、群覆盖数、启停、预设 |

#### 系统（6）

| 模块 | 类型 | 关键内容 |
|---|---|---|
| logs | logs | 时间、级别、账号、群、操作者、动作、详情、筛选与导出 |
| backup | actions | 导出 ZIP（配置+业务数据，不含 token）、导入恢复（输入 `RESTORE`） |
| update | update | 当前/最新版本、更新说明、检查更新、托管模式萌卡管理员验证码登录与导入 |
| about | dashboard | 版本、构建时间、功能来源、开源许可、能力矩阵 |
| web-settings | form | 监听地址、端口、会话时长、重置管理令牌 |
| capabilities | dashboard | action/事件/消息段矩阵、缺失项与原因、快照时间 |

### 6.6 交互规范

- 保存：`PUT` 局部更新，Toast “已保存”，不刷新整页。
- 删除：确认弹窗，显示对象描述；危险操作输入确认词。
- 表格：默认 50 条/页，支持搜索、排序；仅安全模块支持批量删除。
- 表单：离开未保存提示；数字前后端双重校验。
- 一键操作：返回 `job_id`，前端 2 秒轮询进度，完成 Toast 汇总。
- 错误：右上角红色 Toast，展示后端 `message`，不暴露堆栈。
- 权限：所有写接口要求登录 + CSRF；只读接口也要求登录（`/api/health` 除外）。
- 能力缺失：模块显示 `disabled`，不提供可操作控件。
- 响应式：≥1200px 三栏，768–1199px 两栏，<768px 侧边栏抽屉。
- 所有危险操作（退群、全体禁言、解禁全部、群发、怼人、恢复备份、重置令牌）二次确认。

### 6.7 后端 API

统一约定：

- 除 `/api/health` 外均需登录。
- 响应统一 `{"code":0,"message":"","data":...}`；错误为非 2xx + `{"code":400,"message":"中文描述"}`。
- 写操作需 `X-CSRF-Token`。
- 请求体上限 1 MiB；上传接口单独 6 MiB。
- 所有变更写 `audit_log`。

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/admin/login` | 本地令牌登录：`{"username":"admin","token":"..."}`，成功设置 `qca_session` |
| GET | `/api/admin/login/mode` | 返回 `managed/mode/origin`；托管模式登录页使用萌卡管理员 |
| GET | `/api/admin/login/captcha` | 获取萌卡图形验证码（无需本地会话） |
| POST | `/api/admin/login/mengka` | 萌卡管理员账号、密码、图形验证码登录 |
| POST | `/api/admin/logout` | 退出 |
| GET | `/api/admin/profile` | 当前用户、CSRF Token、会话过期时间、实际端口 |
| GET | `/api/admin/catalog` | 模块目录 |
| GET | `/api/admin/dashboard` | 概览数据 |
| GET | `/api/admin/modules/{id}` | 模块元数据 + `settings` |
| PUT | `/api/admin/modules/{id}/settings` | 局部更新设置 |
| GET | `/api/admin/modules/{id}/items?table=&page=&size=&q=&self_id=&protocol_id=&group_id=` | 表格数据 |
| POST | `/api/admin/modules/{id}/items?table=...` | 新增 |
| PUT | `/api/admin/modules/{id}/items/{item_id}?table=...` | 修改 |
| DELETE | `/api/admin/modules/{id}/items/{item_id}?table=...` | 删除 |
| POST | `/api/admin/modules/{id}/actions/{action}` | 一键操作，返回 `job_id` |
| GET | `/api/admin/jobs/{job_id}` | 任务进度 |
| POST | `/api/admin/jobs/{job_id}/cancel` | 取消任务 |
| GET | `/api/admin/accounts` | 账号列表（含协议） |
| POST | `/api/admin/accounts/refresh` | 刷新 `get_bot_list` / `get_protocol_list` |
| GET | `/api/admin/groups?self_id=&protocol_id=` | 账号群列表（离线时返回缓存） |
| GET | `/api/admin/groups/{group_id}?self_id=&protocol_id=` | 群配置 |
| PUT | `/api/admin/groups/{group_id}?self_id=&protocol_id=` | 更新群配置（Merge Patch） |
| GET | `/api/admin/groups/{group_id}/members?self_id=&protocol_id=` | 成员与角色（只读） |
| GET | `/api/admin/permissions` | 全部指令/类目级别 |
| PUT | `/api/admin/permissions` | 批量更新 `{"items":[{"key":"全体禁言","min_level":2}]}` |
| POST | `/api/admin/permissions/preset` | `{"preset":"recommended"}` |
| GET | `/api/admin/masters` | 全局主人 + 账号主人 |
| PUT | `/api/admin/masters` | 覆盖式更新（带审计） |
| GET | `/api/admin/logs?page=&size=&level=&self_id=&protocol_id=&group_id=&q=` | 审计/运行日志 |
| GET | `/api/admin/logs/export` | CSV 导出 |
| GET | `/api/admin/backup` | 下载 ZIP（不含 token） |
| POST | `/api/admin/restore` | 上传 ZIP，需 `{"confirm":"RESTORE"}` |
| POST | `/api/admin/uploads` | 上传头像/图片，返回 `upload_id` |
| GET | `/api/admin/capabilities` | 能力快照与缺失原因 |
| GET | `/api/admin/redpacket/summary?self_id=&protocol_id=&range=` | 红包统计、运行状态、监听群数、账号状态 |
| GET | `/api/admin/redpacket/ranking?self_id=&protocol_id=&range=&sort=` | 群红包排行；sort=count/amount/grabbed |
| GET | `/api/admin/redpacket/records?self_id=&protocol_id=&range=&group_id=&result=&q=&page=&size=` | 红包处理记录 |
| POST | `/api/admin/modules/license-cards/actions/generate` | 控制台生成卡密，返回 `codes` |
| POST | `/api/admin/modules/license-cards/actions/{action}` | 卡密状态操作，action 为 disable/enable/delete |
| POST | `/api/admin/modules/group-authorizations/actions/grant` | 控制台手动授权 |
| POST | `/api/admin/modules/group-authorizations/actions/revoke` | 控制台撤销授权 |
| GET | `/api/admin/update` | 在线更新状态、版本、更新源、托管状态 |
| POST | `/api/admin/update/check` | 检查 GitHub/更新清单最新版本 |
| POST | `/api/admin/update/install` | 下载校验并导入；可返回登录/权限确认要求 |
| GET | `/api/admin/update/captcha` | 获取更新流程的萌卡图形验证码 |
| GET | `/api/admin/update/admin-check` | 测试萌卡管理接口可达性与会话状态（不消耗验证码） |
| POST | `/api/admin/update/login` | 萌卡管理员账号、密码、验证码登录 |
| GET | `/api/admin/settings/web` | 控制台设置 |
| PUT | `/api/admin/settings/web` | 修改监听地址/端口/会话时长 |
| POST | `/api/admin/settings/token/reset` | 重置管理令牌，返回新令牌一次 |
| GET | `/api/health` | `{"status":"ok","version":"1.2.18"}` |

- 新增表格 kind：`license_cards`、`group_authorizations`、`agents`、`music_lines`、`red_packet_rules`；红包记录由专用 `/api/admin/redpacket/records` 提供筛选与分页。

## 6.8 控制台安全

- 默认仅监听 `127.0.0.1`；改为 `0.0.0.0` 时启动告警并要求已设置令牌。
- `admin.token` 32 字节随机十六进制，文件权限 0600；首次启动生成并打印一次。
- 登录恒定时间比较；5 次失败锁定 5 分钟（按 IP）。
- 会话 Cookie：`qca_session`，HttpOnly、SameSite=Lax、默认 12 小时；重启失效。
- 写接口校验 `X-CSRF-Token`；CSRF Token 每会话随机 32 字节。
- 请求体上限 1 MiB；上传接口 6 MiB；静态文件防目录穿越。
- 安全头：CSP、`X-Frame-Options: DENY`、`X-Content-Type-Options: nosniff`、`Referrer-Policy: no-referrer`。
- 恢复备份需输入 `RESTORE`。
- 备份不包含 token；恢复后保留当前 token。
- 日志脱敏：不记录令牌、Cookie、完整消息内容；调试模式也只记录截断文本。
- 重置令牌后旧会话全部失效。

### 6.9 首次使用引导

1. 启动插件，查看 stderr 中的管理令牌与控制台地址。
2. 登录控制台。
3. 在“账号”页确认节点账号与协议实例。
4. 在“主人”页添加主人 QQ（或直接用机器人自身 QQ 发指令）。
5. 在“权限”页选择“推荐”预设，或手动下调菜单/帮助/打卡/申请头衔/互赞/点歌。
6. 在“群管理”页选择群，开启需要的自动化功能。
7. 发送“菜单”验证。

### 6.10 托管模式与在线更新

- 启动后 `managed.Load()` 判断运行环境：原生 IPC 握手成功即 `managed=true`，并读取 `get_plugin_context.admin_base_url`；也兼容 `MENGKA_PLUGIN_ADMIN_ORIGIN`、`MENGKA_PLUGIN_ADMIN_HOST/PORT` 与 `MENGKA_MANAGED_V1=1`。控制台模式标记为独立模式。
- 托管模式下登录页默认切换为“萌卡管理员登录”：管理员账号、密码、图形验证码；验证码通过插件后端代理获取，密码只用于本次登录，不写配置、不写日志。登录页显示实际管理地址，并明确端口不固定（不一定是 7891）；无法连接时使用数据目录 `admin.token` 回退登录。
- 默认更新源为 `haircv/Mengka-QQ-Cloud-Assistant` 的 GitHub Release；也可改为其他 GitHub 仓库（`owner/repo`，支持 stable/prerelease）或自定义更新清单 URL。
- `.github/workflows/release.yml` 在推送 `v*` 标签或手动触发时测试、构建三平台短名称包（`qq-cloud-assistant-<version>-<platform>`）并上传 Release；本机可用 `scripts/publish-release.sh` / `.ps1`。
- 自定义清单格式：`{"version":"1.2.18","notes":"...","assets":{"windows-amd64":{"url":"...","sha256":"...","size":123},"linux-amd64":{...},"linux-arm64":{...}}}`；也接受 GitHub Release JSON 的 `tag_name/body/assets[]`。
- 下载后必须校验：压缩包大小/可选 SHA256；`mengka-plugin.json` 的 `plugin_id` 必须为 `qq-cloud-assistant` 且版本匹配；`SHA256SUMS` 中必须存在并匹配程序入口哈希。任何一项失败都不导入。
- “萌卡管理接口”填写 **Mengka 管理端 Web 地址**（例如 `http://127.0.0.1:7891` 或反向代理域名），不是本插件控制台 18084；登录弹窗会显示实际目标地址。
- 托管环境通过萌卡管理接口登录后调用 `POST /api/v1/plugins/managed/import` 上传安装包；如返回 `capability_change`，控制台展示确认，携带 `approval_token` 重试。导入前会复制当前程序并启动独立助手，导入后由助手轮询 `/api/v1/plugins/managed`，只匹配 `release.plugin_id`、目标版本与节点，再 POST `/{id}/action`（`start`）；进程内也会做同样的版本匹配作为兜底。启动失败保留更新结果并返回 `start_error`。
- `grab_red_packet` 只提交 `group_id`、`sender_id`、嵌套 `red_packet` 和可选 `claim_token`；顶层 `channel/title` 等字段会被当前萌卡接口以“已移除”拒绝。
- 导入前读取 `/api/v1/nodes/options` 解析服务节点：优先匹配萌卡托管节点，单节点自动选择；多节点可在“在线更新设置 → 服务节点 ID”（`update.node_id`）指定。表单固定提交 `node_id` 与 `trusted_source_ack=true`，未传节点时萌卡会拒绝为“请选择服务节点”。
- 1.2.13 及更早版本不提交 `node_id`，需要先手动导入一次 1.2.14 或更高版本；升级后的版本可继续在线更新。
- 独立启动助手从 1.2.18 开始生效；从 1.2.17 或更早版本升级到 1.2.18 时，首次仍需手动启动一次，后续更新由助手自动启用。
- 控制台提供“测试萌卡接口”，只检查可达性与会话状态，**不会消耗验证码**（此前的接口测试会生成新验证码，导致登录页验证码失效）；管理接口地址错误返回 502 并提示不要填写插件控制台，账号/验证码错误保持 401。
- 非托管环境只允许检查/下载，提示手动导入，不替换当前程序。
- 自动检查由 `update.auto_check` 控制，启动 3 秒后检查一次，之后每 6 小时一次；只检查，不自动安装。

---

## 7. 数据模型

### 7.1 SQLite（`data/qca.db`）

所有业务表均带 `self_id` 与 `protocol_id`（全局表除外）。

```sql
CREATE TABLE IF NOT EXISTS schema_meta (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS qa_rules (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  self_id     INTEGER NOT NULL,
  protocol_id INTEGER NOT NULL,
  scope       TEXT NOT NULL,           -- group / friend / global
  group_id    INTEGER NOT NULL DEFAULT 0,
  rule_type   TEXT NOT NULL,           -- exact / fuzzy
  keyword     TEXT NOT NULL,
  reply       TEXT NOT NULL,
  cooldown_s  INTEGER NOT NULL DEFAULT 0,
  enabled     INTEGER NOT NULL DEFAULT 1,
  created_at  TEXT NOT NULL,
  updated_at  TEXT NOT NULL,
  UNIQUE(self_id, protocol_id, scope, group_id, rule_type, keyword)
);
CREATE INDEX IF NOT EXISTS idx_qa_lookup
  ON qa_rules(self_id, protocol_id, scope, group_id, rule_type);

CREATE TABLE IF NOT EXISTS white_lists (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  self_id     INTEGER NOT NULL,
  protocol_id INTEGER NOT NULL,
  kind        TEXT NOT NULL,           -- anti_recall / at_monitor / emoji / emoji_specific /
                                       -- qa / qa_friend / checkin / auto_like_qzone /
                                       -- fire_group / fire_friend / mass_group_wl /
                                       -- mass_friend_wl / mutual_like / red_packet_group /
                                       -- red_packet_noreply
  group_id    INTEGER NOT NULL DEFAULT 0,
  target_id   INTEGER NOT NULL DEFAULT 0,
  note        TEXT NOT NULL DEFAULT '',
  created_at  TEXT NOT NULL,
  UNIQUE(self_id, protocol_id, kind, group_id, target_id)
);

CREATE TABLE IF NOT EXISTS reply_pool (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  self_id     INTEGER NOT NULL,
  protocol_id INTEGER NOT NULL,
  kind        TEXT NOT NULL,           -- at / comment / fire / mood / like_reply / new_friend
  content     TEXT NOT NULL,
  created_at  TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS scheduled_tasks (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  self_id      INTEGER NOT NULL,
  protocol_id  INTEGER NOT NULL,
  ttype        TEXT NOT NULL,          -- timer_ban / timer_unban / timer_checkin /
                                       -- timer_mass_group / timer_mass_friend
  time_hm      TEXT NOT NULL DEFAULT '',
  target       TEXT NOT NULL DEFAULT '',
  content      TEXT NOT NULL DEFAULT '',
  interval_min INTEGER NOT NULL DEFAULT 0,
  enabled      INTEGER NOT NULL DEFAULT 1,
  next_run     TEXT NOT NULL DEFAULT '',
  last_run     TEXT NOT NULL DEFAULT '',
  fail_count   INTEGER NOT NULL DEFAULT 0,
  created_at   TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS loop_tasks (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  self_id      INTEGER NOT NULL,
  protocol_id  INTEGER NOT NULL,
  category     TEXT NOT NULL DEFAULT '',
  ltype        TEXT NOT NULL DEFAULT '',
  interval_min INTEGER NOT NULL DEFAULT 60,
  groups       TEXT NOT NULL DEFAULT '',
  content      TEXT NOT NULL DEFAULT '',
  notify       INTEGER NOT NULL DEFAULT 0,
  enabled      INTEGER NOT NULL DEFAULT 1,
  next_run     TEXT NOT NULL DEFAULT '',
  last_run     TEXT NOT NULL DEFAULT '',
  fail_count   INTEGER NOT NULL DEFAULT 0,
  created_at   TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS mass_jobs (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  self_id     INTEGER NOT NULL,
  protocol_id INTEGER NOT NULL,
  mode        TEXT NOT NULL,           -- group / friend / card
  targets     TEXT NOT NULL DEFAULT '',
  content     TEXT NOT NULL DEFAULT '',
  interval_s  INTEGER NOT NULL DEFAULT 3,
  status      TEXT NOT NULL,           -- pending / running / completed / stopped / interrupted
  total       INTEGER NOT NULL DEFAULT 0,
  sent        INTEGER NOT NULL DEFAULT 0,
  failed      INTEGER NOT NULL DEFAULT 0,
  created_at  TEXT NOT NULL,
  updated_at  TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS jobs (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  kind        TEXT NOT NULL,
  self_id     INTEGER NOT NULL DEFAULT 0,
  protocol_id INTEGER NOT NULL DEFAULT 0,
  group_id    INTEGER NOT NULL DEFAULT 0,
  status      TEXT NOT NULL,           -- running / done / failed / cancelled
  total       INTEGER NOT NULL DEFAULT 0,
  done        INTEGER NOT NULL DEFAULT 0,
  failed      INTEGER NOT NULL DEFAULT 0,
  message     TEXT NOT NULL DEFAULT '',
  created_at  TEXT NOT NULL,
  updated_at  TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS account_groups (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  name        TEXT NOT NULL UNIQUE,
  created_at  TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS account_group_members (
  group_name  TEXT NOT NULL,
  self_id     INTEGER NOT NULL,
  protocol_id INTEGER NOT NULL,
  PRIMARY KEY(group_name, self_id, protocol_id)
);

CREATE TABLE IF NOT EXISTS group_managers (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  self_id     INTEGER NOT NULL,
  protocol_id INTEGER NOT NULL,
  group_id    INTEGER NOT NULL,
  user_id     INTEGER NOT NULL DEFAULT 0,
  uid         TEXT NOT NULL DEFAULT '',
  note        TEXT NOT NULL DEFAULT '',
  created_at  TEXT NOT NULL,
  UNIQUE(self_id, protocol_id, group_id, user_id, uid)
);

CREATE TABLE IF NOT EXISTS bot_switches (
  self_id     INTEGER NOT NULL,
  protocol_id INTEGER NOT NULL,
  enabled     INTEGER NOT NULL DEFAULT 1,
  note        TEXT NOT NULL DEFAULT '',
  updated_at  TEXT NOT NULL,
  PRIMARY KEY(self_id, protocol_id)
);

CREATE TABLE IF NOT EXISTS command_permissions (
  key        TEXT PRIMARY KEY,         -- 指令键 或 category:xxx
  min_level  INTEGER NOT NULL DEFAULT 5 CHECK(min_level BETWEEN 1 AND 5),
  enabled    INTEGER NOT NULL DEFAULT 1,
  updated_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS mute_records (
  self_id     INTEGER NOT NULL,
  protocol_id INTEGER NOT NULL,
  group_id    INTEGER NOT NULL,
  user_id     INTEGER NOT NULL,
  until_ts    INTEGER NOT NULL DEFAULT 0,
  operator_id INTEGER NOT NULL DEFAULT 0,
  source      TEXT NOT NULL DEFAULT '', -- command / notice
  updated_at  TEXT NOT NULL,
  PRIMARY KEY(self_id, protocol_id, group_id, user_id)
);

CREATE TABLE IF NOT EXISTS qzone_seen (
  self_id     INTEGER NOT NULL,
  protocol_id INTEGER NOT NULL,
  friend_id   INTEGER NOT NULL,
  feed_id     TEXT NOT NULL,
  action      TEXT NOT NULL,           -- like / comment
  created_at  TEXT NOT NULL,
  PRIMARY KEY(self_id, protocol_id, friend_id, feed_id, action)
);

CREATE TABLE IF NOT EXISTS group_cache (
  self_id      INTEGER NOT NULL,
  protocol_id  INTEGER NOT NULL,
  group_id     INTEGER NOT NULL,
  group_name   TEXT NOT NULL DEFAULT '',
  member_count INTEGER NOT NULL DEFAULT 0,
  max_member   INTEGER NOT NULL DEFAULT 0,
  updated_at   TEXT NOT NULL,
  PRIMARY KEY(self_id, protocol_id, group_id)
);

CREATE TABLE IF NOT EXISTS counters (
  self_id     INTEGER NOT NULL,
  protocol_id INTEGER NOT NULL,
  day         TEXT NOT NULL,
  name        TEXT NOT NULL,
  value       INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY(self_id, protocol_id, day, name)
);

CREATE TABLE IF NOT EXISTS uin_map (
  uid        TEXT PRIMARY KEY,
  user_id    INTEGER NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS audit_log (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  ts          TEXT NOT NULL,
  level       TEXT NOT NULL DEFAULT 'info',
  self_id     INTEGER NOT NULL DEFAULT 0,
  protocol_id INTEGER NOT NULL DEFAULT 0,
  group_id    INTEGER NOT NULL DEFAULT 0,
  operator    TEXT NOT NULL DEFAULT '',
  action      TEXT NOT NULL,
  detail      TEXT NOT NULL DEFAULT ''
);
CREATE INDEX IF NOT EXISTS idx_audit_ts ON audit_log(ts DESC);
CREATE INDEX IF NOT EXISTS idx_audit_query ON audit_log(self_id, protocol_id, group_id, ts DESC);

CREATE TABLE IF NOT EXISTS capability_snapshots (
  id                INTEGER PRIMARY KEY AUTOINCREMENT,
  created_at        TEXT NOT NULL,
  framework_version TEXT NOT NULL DEFAULT '',
  payload           TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS license_cards (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  code         TEXT NOT NULL UNIQUE,
  plan         TEXT NOT NULL DEFAULT '',
  days         INTEGER NOT NULL,
  status       TEXT NOT NULL DEFAULT 'unused', -- unused / used / disabled
  batch        TEXT NOT NULL DEFAULT '',
  created_by   TEXT NOT NULL DEFAULT '',
  agent_qq     INTEGER NOT NULL DEFAULT 0,
  used_by      INTEGER NOT NULL DEFAULT 0,
  group_id     INTEGER NOT NULL DEFAULT 0,
  group_name   TEXT NOT NULL DEFAULT '',
  used_at      TEXT NOT NULL DEFAULT '',
  expires_at   TEXT NOT NULL DEFAULT '',
  note         TEXT NOT NULL DEFAULT '',
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_license_cards_status ON license_cards(status, created_at DESC);
CREATE INDEX IF NOT EXISTS idx_license_cards_agent ON license_cards(agent_qq, status);

CREATE TABLE IF NOT EXISTS group_authorizations (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  self_id      INTEGER NOT NULL,
  protocol_id  INTEGER NOT NULL,
  group_id     INTEGER NOT NULL,
  group_name   TEXT NOT NULL DEFAULT '',
  plan         TEXT NOT NULL DEFAULT '',
  days_total   INTEGER NOT NULL DEFAULT 0,
  activated_by INTEGER NOT NULL DEFAULT 0,
  expires_at   TEXT NOT NULL DEFAULT '',
  status       TEXT NOT NULL DEFAULT 'active', -- active / revoked；expired 由到期时间计算
  note         TEXT NOT NULL DEFAULT '',
  source_card  TEXT NOT NULL DEFAULT '',
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL,
  UNIQUE(self_id, protocol_id, group_id)
);
CREATE INDEX IF NOT EXISTS idx_group_auth_expiry ON group_authorizations(expires_at);

CREATE TABLE IF NOT EXISTS agents (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  qq           INTEGER NOT NULL UNIQUE,
  name         TEXT NOT NULL DEFAULT '',
  contact      TEXT NOT NULL DEFAULT '',
  status       TEXT NOT NULL DEFAULT 'active', -- active / disabled
  quota        INTEGER NOT NULL DEFAULT 0,     -- 0 表示不限
  quota_used   INTEGER NOT NULL DEFAULT 0,
  generated    INTEGER NOT NULL DEFAULT 0,
  redeemed     INTEGER NOT NULL DEFAULT 0,
  note         TEXT NOT NULL DEFAULT '',
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS red_packet_records (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  self_id      INTEGER NOT NULL,
  protocol_id  INTEGER NOT NULL,
  group_id     INTEGER NOT NULL,
  group_name   TEXT NOT NULL DEFAULT '',
  sender_id    INTEGER NOT NULL DEFAULT 0,
  sender_name  TEXT NOT NULL DEFAULT '',
  message_id   INTEGER NOT NULL DEFAULT 0,
  listid_hash  TEXT NOT NULL,
  channel      INTEGER NOT NULL DEFAULT 0,
  packet_type  TEXT NOT NULL DEFAULT '',  -- normal / password / voice
  title        TEXT NOT NULL DEFAULT '',
  result       TEXT NOT NULL DEFAULT 'processing',
  reason_code  TEXT NOT NULL DEFAULT '',
  reason       TEXT NOT NULL DEFAULT '',
  total_fen    INTEGER NOT NULL DEFAULT 0,
  total_num    INTEGER NOT NULL DEFAULT 0,
  expected_fen INTEGER NOT NULL DEFAULT 0,
  amount_fen   INTEGER NOT NULL DEFAULT 0,
  amount_known INTEGER NOT NULL DEFAULT 0,
  query_ms     INTEGER NOT NULL DEFAULT 0,
  grab_ms      INTEGER NOT NULL DEFAULT 0,
  total_ms     INTEGER NOT NULL DEFAULT 0,
  reply_status TEXT NOT NULL DEFAULT '',
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL,
  UNIQUE(self_id, protocol_id, listid_hash)
);
CREATE INDEX IF NOT EXISTS idx_red_packet_group ON red_packet_records(self_id, protocol_id, group_id, created_at DESC);
CREATE INDEX IF NOT EXISTS idx_red_packet_created ON red_packet_records(self_id, protocol_id, created_at DESC);
CREATE INDEX IF NOT EXISTS idx_red_packet_result ON red_packet_records(result, created_at DESC);

CREATE TABLE IF NOT EXISTS music_lines (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  name         TEXT NOT NULL DEFAULT '',
  source       TEXT NOT NULL,             -- qq / netease / kugou
  format       TEXT NOT NULL DEFAULT 'meting', -- meting / meting_legacy / netease
  endpoint     TEXT NOT NULL,
  source_param TEXT NOT NULL DEFAULT '',
  quality      TEXT NOT NULL DEFAULT '320',
  priority     INTEGER NOT NULL DEFAULT 100,
  enabled      INTEGER NOT NULL DEFAULT 1,
  note         TEXT NOT NULL DEFAULT '',
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_music_lines_source ON music_lines(source, enabled, priority);

CREATE TABLE IF NOT EXISTS red_packet_rules (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  self_id     INTEGER NOT NULL,
  protocol_id INTEGER NOT NULL,
  kind        TEXT NOT NULL,              -- group / person / keyword
  mode        TEXT NOT NULL,              -- include / exclude
  value       TEXT NOT NULL,
  enabled     INTEGER NOT NULL DEFAULT 1,
  note        TEXT NOT NULL DEFAULT '',
  created_at  TEXT NOT NULL,
  updated_at  TEXT NOT NULL,
  UNIQUE(self_id, protocol_id, kind, mode, value)
);
CREATE INDEX IF NOT EXISTS idx_red_packet_rules_lookup ON red_packet_rules(self_id, protocol_id, enabled);
```

说明：

- 不建 `recall_cache` 表：消息缓存为内存 LRU，10 分钟 TTL，重启丢失，避免持久化聊天内容。
- 不建 `verify_sessions` 表：验证会话内存存储，重启标记过期并记录日志。
- `mute_records` 仅用于降级的禁言列表与自身被禁言状态，不代表全量禁言。
- `group_cache` 让控制台在账号离线时仍能显示群选择器。
- `counters` 按天统计：messages、commands、auto_replies、sends、errors。
- `license_cards` 与 `agents` 为全局表；`group_authorizations` 与 `red_packet_records` 必须带账号双键。
- `red_packet_records.listid_hash` 为 `listid` 的 SHA-256，不保存 `authkey`、`claim_token`；`UNIQUE` 用于同一账号红包去重。
- `red_packet_records.result` 只接受 processing/grabbed/skipped/failed；`packet_type` 支持 normal/lucky/password/exclusive/voice；成功必须由 `retcode=0 && state=0` 判定。
- `music_lines` 为全局线路表；`red_packet_rules` 按账号隔离，支持 group/person/keyword × include/exclude。
- schema 版本为 5；新增表通过 `CREATE TABLE IF NOT EXISTS` 幂等迁移。

### 7.2 JSON 配置

`data/global.json`：

```json
{
  "schema_version": 1,
  "masters": [{"uin": 10001}],
  "default_group": {
    "follow_default": true,
    "welcome": {"enabled": true, "messages": ["欢迎 {nick} 加入 {group_name}"]},
    "leave_notice": {"enabled": true, "text": "{nick} 离开了群聊"},
    "approval": {"enabled": false, "white_words": [], "black_words": [], "miss_policy": "manual"}
  },
  "global_blacklist": [],
  "summon_command": "菜单",
  "menu_header": "",
  "menu_footer": "",
  "debug": false,
  "self_trigger_auto": false,
  "permission_denied_hint": false,
  "send_rate_limit_ms": 800,
  "mass_interval_s": 3,
  "reply_mode": "text",
  "silent_mode": false,
  "license": {
    "enforce": false,
    "default_plan": "标准",
    "enforce_message": "本群尚未授权，请联系管理员使用卡密开通"
  },
  "music": {
    "enabled": true,
    "mode": "link",
    "source": "netease",
    "quality": "320",
    "search_enabled": true,
    "select_enabled": true,
    "failover_enabled": true,
    "search_limit": 5
  },
  "update": {
    "github_repo": "",
    "manifest_url": "",
    "channel": "stable",
    "auto_check": false,
    "admin_base_url": ""
  },
  "web": {"listen": "127.0.0.1", "port": 18084, "session_hours": 12}
}
```

`data/accounts/<self_id>/<protocol_id>.json`：

```json
{
  "enabled": true,
  "masters": [],
  "master_trigger": true,
  "menu_header": "",
  "menu_footer": "",
  "reply_mode": "",
  "silent_mode": null,
  "feature_switches": {},
  "red_packet": {
    "enabled": false,
    "listen_all": false,
    "type_normal": true,
    "type_lucky": true,
    "type_password": true,
    "type_exclusive": true,
    "type_voice": false,
    "require_android": true,
    "delay_ms": 500,
    "jitter_ms": 300,
    "min_expected_fen": 10,
    "query_before_grab": true,
    "on_query_fail": "grab",
    "reply_enabled": true,
    "reply_text": "xxlb",
    "reply_delay_ms": 3000,
    "reply_requires_amount": true,
    "max_per_day": 0,
    "max_per_minute": 10,
    "voice_tts_endpoint": "",
    "voice_tts_format": "mp3",
    "record_days": 30
  }
}
```

`data/groups/<self_id>/<protocol_id>/<group_id>.json`：

```json
{
  "follow_default": true,
  "managers": [{"uin": 10002, "uid": "", "note": "分管"}],
  "blacklist": [{"uin": 10003, "uid": "", "reason": "广告", "ts": 1790000000}],
  "whitelist": [],
  "notice": {},
  "verify": {},
  "anti_recall": {},
  "at_handler": {},
  "title": {},
  "music": {},
  "red_packet": {
    "group_enabled": true,
    "delay_ms": 500,
    "min_expected_fen": 10,
    "reply_text": "xxlb"
  },
  "permission_overrides": {}
}
```

`data/permissions.json`：

```json
{"菜单": 1, "帮助": 1, "本群打卡": 1, "category:qa": 2}
```

`data/admin.token`：

- 32 字节随机十六进制（64 字符）；
- 文件权限 0600；
- 首次启动生成；
- 不进入备份包。

### 7.3 配置合并与迁移

读取顺序：

```text
内建默认 → global.default_group → 账号配置 → 群配置
```

- `follow_default=true` 时群配置只存差异；`false` 时独立，但仍叠加内建默认值保证字段完整。
- 所有写操作使用 RFC 7386 Merge Patch，禁止整体覆盖丢字段。
- 配置原子写：临时文件 + rename。
- `schema_version` 幂等迁移；迁移前自动备份到 `data/backup/pre-migrate-<ts>.zip`。
- 迁移失败回滚并拒绝启动。
- 运行时始终读取生效配置，不直接读群文件原文；控制台“原始差异”视图单独展示。

---

## 8. 运行机制

### 8.1 事件流水线

```text
stdin IPC 帧
  → 解析 event
  → 归一化（AccountKey / sender / role / segments / text）
  → 自身消息指纹过滤
  → Bot 总开关
  → Router 投递（AccountKey + 会话键，队列 256）
  → Worker 串行处理
      → 消息缓存 / 计数
      → 通知 Hook（入群/退群/被踢/撤回/禁言/申请/好友变化/点赞/上下线）
      → 自动功能（验证/防撤/艾特/表情回应/问答）
      → 红包消息 Hook（异步领取，不阻塞 Worker）
      → 指令匹配（Exact → Prefix 最长 → Pattern → Fallback）
      → 权限判定
      → 授权门禁（license.enforce，默认关闭）
      → Handler
  → 发送队列（账号级串行、限速、优先级）
```

规则：

- 同一 `(AccountKey, 会话键)` 严格串行，跨会话/跨账号并行。
- 私聊按好友分 Worker，不只用一个 `group_id=0` 队列，避免不同好友互相阻塞。
- Worker 空闲 30 分钟回收。
- 队列满丢弃并记日志，不阻塞 IPC 读循环。
- 每个 Worker 捕获 panic，单条事件异常不影响后续。
- Bot 总开关关闭时，在 Router 入口直接丢弃事件。
- 事件处理不得阻塞 stdin 读循环；解析后立即入队。
- `bot_online` 后拉取最近 24 小时待处理申请，按 `request_id` 去重后交给自动审批；不处理已处理/过期申请。

### 8.2 队列与限速

- 指令回复/验证/审批：高优先级。
- 自动回复/通知：普通优先级。
- 群发/续火/说说/打卡：低优先级。
- 每个账号一个发送 Worker，默认 800ms/条；高优先级可插队到低优先级之前。
- 群发任务默认 3000ms/条，最小 1000ms。
- 发送队列上限 1000；超过时拒绝新增群发任务并告警。
- 账号离线时暂停发送；恢复在线后继续。
- 红包口令与到账回复走账号级限速；领取调用按账号串行，失败不重试。

### 8.3 消息缓存

- 每个会话内存 LRU，最多 1000 条，TTL 10 分钟。
- Key：`(AccountKey, 会话键, message_id)`。
- 内容：`message_id`、`msg_seq`、`msg_random`、发送者、段、文本、时间。
- 用途：防撤、清屏、引用回复、取直链、自身撤回。
- 不持久化，重启丢失。

### 8.4 防撤回

- 监听 `group_notice.recall` / `friend_notice.recall`。
- 优先 `body.original`；正文为“消息已过期”时查本地缓存。
- 群/全局/好友三种作用域；白名单群豁免。
- 自身消息不转发。
- 原文转发失败不重试，只记日志。
- 不伪造原文；无原文时提示“原文未缓存”。

### 8.5 自身撤回

- 发送成功后保存 `msg_seq`/`msg_random`（群）或 `msg_seq`/`msg_random`/`client_seq`/`msg_time`（好友）。
- 按 `delay_seconds` 入延迟队列。
- 群用 `recall_group_msg`；好友用 `recall_friend_msg`。
- 失败只记日志，不重试。
- 重启后未执行的延迟撤回丢弃并记录。
- 临时会话不支持自身撤回。

### 8.6 清屏

- 只处理当前会话缓存内、仍在撤回时限内的消息。
- 默认最多 20 条，1 秒/条，从新到旧。
- 超出时限、无权限、已被撤回的消息跳过。
- 完成后汇总“成功 X / 跳过 Y / 失败 Z”。
- 不发送零宽字符刷屏，不提供绕过权限的“强清”。

### 8.7 入群验证

- 会话键：`(AccountKey, group_id, user_id)`。
- 模式：数字、字母、混合、发言；默认混合。
- 超时默认 120 秒；动作：踢出/禁言/不处理；禁言默认 10 分钟。
- 未验证用户的消息只接受验证码，不进入指令/问答；可选撤回其消息。
- 通过后发送欢迎语（若开启）并删除会话。
- 重复入群：替换旧会话并停止旧计时器。
- 重启后会话过期，记录日志，不追溯处罚。

### 8.8 定时与循环

- 30 秒扫描，精度 ±30 秒。
- 定时任务 `next_run` 落库，重启续跑；执行中加锁防重入。
- 循环任务按间隔计算下一次；失败记录后继续；连续失败 10 次停用并通知主人。
- 账号离线时暂停，恢复在线后重新计算。
- 时间使用本机时区，控制台统一显示 `YYYY-MM-DD HH:MM:SS`。

### 8.9 群发

- `mass_jobs` 持久化。
- 重启后未完成任务标记 `interrupted`，需手动继续，避免重复发送。
- 逐条发送，默认 3000ms，最小 1000ms。
- 失败计数，不自动重试。
- 支持群发群、群发好友、群发群名片。
- 停止后当前条发送完退出。
- 群发内容带任务标记，不触发自动回复。
- 队列满拒绝新增任务。

### 8.10 问答

- 作用域：本群、好友、全局。
- 类型：精准（全等）、模糊（包含）。
- 精准优先于模糊；本群/好友优先于全局。
- 冷却按关键词与作用域计；延迟 0–N 秒；随机回复 1–N 条。
- 回复池随机，支持模板变量。
- 自身消息默认排除；黑名单不触发。
- 指令匹配优先于问答兜底。

### 8.11 秒赞/秒评

- 每 120 秒轮询 `get_qzone_friend_feeds` 第一页。
- 只处理白名单好友 `user_id` 的动态。
- 动态去重：`(AccountKey, friend_id, feed_id, action)`，落 `qzone_seen`。
- 点赞 `like_qzone_feed`；评论 `comment_qzone_feed`；都传完整 feed 对象。
- 评论内容从池中随机；失败记录，不连续重试同一条。
- `status=restricted` 或 Android 材料缺失时暂停并提示。
- 平台风控风险：默认低频，白名单必填。

### 8.12 打卡与续火

- 打卡：`sign_group`；本群立即、一键遍历、0 点定时。
- 一键打卡白名单非空时只打白名单，否则全部群；默认 1000ms/群；完成后私聊发起人汇总。
- 续火：按群/好友名单发送内容池随机文本；支持间隔与每日时间两种模式。
- 互赞：一键互赞优先使用本地互赞名单；名单为空时读取收到的名片赞并回赞；用送出的赞记录去重，避免重复。
- 一键任务完成私聊发起人汇总；失败逐条记录，不中断。

### 8.13 点歌与搜歌

- 搜索：`点歌 [QQ/网易云/酷狗] <歌名>`（别名 `搜歌`）；按首选来源线路、再按其他来源线路失败换源，单次最多 8 条线路、总耗时 20 秒。
- 线路：`music_lines` 支持 meting / meting_legacy / netease 三种格式，按优先级排序；线路返回音频直链、封面、来源 ID 或页面地址。
- 回复：链接模式发送歌曲页；卡片模式仅对 QQ/网易云调用 `get_music_ark`，酷狗回退链接；语音模式用 `upload_group_voice` / `upload_friend_voice`，失败回退卡片再回退链接。
- 音频解析：发送前探测搜索线路、网易云 outer、injahow 等候选地址，优先使用跟随跳转后的可达直链；卡片使用可达音频与封面，页面地址不带 `#` 片段。
- 语音回退：框架下载外部地址失败时，由插件带 Referer/UA 下载音频并以 Base64 传给上传接口；单文件上限 10 MiB，超限回退卡片/链接。卡片失败日志包含上游 `status/error`，不再只显示 `<nil>`。
- 选歌：返回最多 5 条，60 秒内数字选择；关闭选歌时发送第一条。
- 外部接口失败不重试；搜索全部失败回退搜索链接；不缓存版权内容。

### 8.14 菜单与帮助

- 类目显示统一为“通用功能 / 授权功能 / 红包功能 / 点歌功能”等；旧“通用/授权/红包”不再作为类目入口。
- 运行状态优先调用 `get_bot_info` 获取好友/群数量、QQ 等级、收发包与登录时间；数量缺失时回退 `get_bot_list`，再回退好友/群列表计数。
- 在“当前Bot”下输出登录协议，格式为 `平台 设备类型(版本)`，例如 `Android Pad(9.3.75)`；设备类型来自 `protocol.type`，缺失时用账号状态回退，协议 ID 已包含在“当前Bot”的账号键中。
- 实时状态末尾依次输出：在线、登录时间、当前时间；在线时长从 `login_time`（缺失时 `online_time`）起算。
- 框架显示 `app_name + app_version`（缺失时回退 API 版本）。
- 由注册表实时生成，按权限过滤。
- 类目名可作为 Fallback；匹配优先级低于真实指令。
- 菜单头尾、召唤命令可配置。
- 文字/图片模式；图片上传失败回退文字。
- 帮助只显示有权使用的指令。
- 静默模式抑制文本回复，但动作仍执行。

### 8.15 查共群

- 取 `get_group_list`，最多扫描 50 个群。
- 每群 `get_group_member_list`，单次 2 秒超时。
- 结果缓存 10 分钟；每人 10 分钟最多查询一次。
- 返回共同群数量与最多 20 个群名。
- 频率限制与群数量可配置。

### 8.16 错误与可观测

- 每个 action 记录：账号、action、耗时、结果、错误码；不记录消息正文。
- 发送失败区分协议错误/业务错误/超时；写操作不重试。
- 权限拒绝按配置决定是否回复提示；黑名单始终静默。
- 失败计数进入 `counters`，控制台概览展示。
- 审计日志覆盖配置、权限、主人、备份、令牌、任务启停。
- 运行日志写 stderr，并保留内存环形缓冲供控制台“日志”页查看。

---


### 8.17 红包领取流程（五类）

1. 只处理 `group_message` 中的 `red_packet` 消息段；自身消息、黑名单发送者不处理。
2. 读取账号级开关、群级开关与监听范围；默认 `listen_all=false`，需群在 `red_packet_group` 名单中。群内执行“开启红包助手”会自动加入名单。
3. 以 `listid` 的 SHA-256 去重并写入 `red_packet_records`（初始 `processing`）；重复事件直接丢弃。
4. 类型识别：显式 `type` 优先；否则 `channel` 6/32 口令、65536 语音、1024 专属、2 拼手气，其余普通。类型开关关闭时记 `type_disabled`；平台非 Android 时记 `android_only`。
5. 群/人员/关键词规则先于限流执行；排除命中或包含不满足时记 `rule_excluded`。每分钟限流超限记 `rate_limited`。
6. 查询详情：`get_red_packet_info` 取得总额、份数、`claim_token`；期望值 = 总额分 ÷ 份数，低于 `min_expected_fen` 记为 `skipped/below_threshold`。查询失败按 `on_query_fail` 选择跳过或继续。若 `query_before_grab=false`，跳过查询与门槛过滤，直接进入延迟；门槛只在查询成功且总额/份数有效时生效。
7. 等待 `delay_ms + rand(0..jitter_ms)`；等待期间重新检查开关、账号在线、授权门禁。
8. 口令红包（`channel=6/32`）：先发送去首尾空白的口令，要求发送结果 `client_seq ∈ [1,4294967295]` 且 `msg_random != 0`；把这两个值与完整红包数据一起提交，不传 `claim_token`，不使用红包消息的 `msg_seq`。
9. 普通/拼手气/专属红包：提交完整 `red_packet` 数据与查询返回的 `claim_token`（可省略由接口自动获取）。
10. 领取结果先看 `retcode`，再看 `state`；只有 `retcode=0 && state=0` 才算成功。金额只读 `recv_object.amount`（分），缺失标记未知。
11. 语音红包：准备标题音频（内置 AI 语音/自定义 TTS/固定音频）→ `upload_red_packet_voice` 匹配 → `upload_group_voice` 发送群语音 → `grab_group_voice_red_packet`；关闭发送、音频准备失败或匹配失败不领取。
12. 成功后按 `reply_delay_ms` 延迟回复；金额未知且 `reply_requires_amount=true` 时不回复；命中只领不回复名单时不回复。失败不自动重试。
13. 完整流程写入 `total_ms`，领取调用耗时写入 `grab_ms`；后台按 `record_days` 清理账本。

### 8.18 本地授权与门禁

- 卡密由主人或代理生成：服务生成 `QCA-XXXX-XXXX-XXXX`，在事务中校验代理状态与配额，写入 `license_cards` 并累计代理配额。
- 群主在群内使用卡密：事务内校验卡密未使用/未禁用，读取现有授权；有效期内从原到期时间顺延，已撤销/已过期从当前时间起算；更新卡密、授权、代理兑换计数。
- 手动授权/撤销：控制台直接调用授权服务，写 `audit_log`；不依赖群内指令。
- 授权门禁：`license.enforce=false` 默认关闭；开启后在 `executeCommand` 中拦截未授权群的指令。Lv5（主人/自身）豁免；`license` 类目（卡密/授权操作）始终放行。
- 红包引擎在门禁开启时只处理已授权群；其他自动功能仍按各自开关运行。
- 全程无网络请求；服务器不保存授权站地址、应用 ID、心跳或远程配置。

### 8.19 账本与清理

- `red_packet_records` 按账号隔离，保留 `query_ms`、`grab_ms`、`total_ms`、`reply_status` 与失败原因。
- 后台引擎每 6 小时按账号 `record_days`（默认 30 天）清理；日志只记录清理数量，不记录 `authkey`/`claim_token`。
- 控制台统计直接聚合账本：确认领取、累计到账、成功回复、平均领取耗时、群排行与最近记录。

## 9. 安全、稳定与合规

### 9.1 控制台安全

见 6.8。补充：

- 反向代理部署时由用户自行加 HTTPS；控制台检测到非本机监听时启动告警。
- 不把框架服务令牌写入任何配置文件、前端、URL。
- 备份默认不含 token；恢复后保留当前 token。
- 日志与审计不记录令牌、Cookie、完整消息。

### 9.2 运行稳定

- IPC 读循环、Worker、HTTP Handler、任务协程、定时器均捕获 panic。
- 队列满丢弃并告警，不阻塞读循环。
- action 超时清理 pending，迟到响应丢弃并记日志。
- SQLite WAL、`busy_timeout=5000`、写串行；批量导入使用事务。
- 配置文件原子写；迁移前备份；失败回滚。
- 优雅退出：停止接收 → 排空队列（最多 5 秒）→ 关闭 DB → 关闭 HTTP。
- 单实例锁：数据目录 `.lock` 文件，防止同一数据目录被两个进程打开。

### 9.3 合规边界

- 不实现伪造聊天、收款、支付、订单、发卡、闪照监控；红包仅调用官方接口，卡密/分群授权/代理仅本地管理。
- 点歌搜索与语音红包内置/自定义 TTS、固定音频 URL 仅在用户主动开启/配置后访问外部接口；不采集用户数据。
- 群发、清屏、怼人、秒赞/秒评在控制台显示频率风险提示。
- 怼人限时限量；清屏改为撤回；群发默认低频。
- 不采集、不上传任何用户数据。
- 除用户主动点歌（配置线路）、语音红包内置/自定义 TTS 与固定音频 URL 外，无外联。
- 不实现绕过平台风控的协议包。
- 不提供账号密码登录；账号由萌卡 NT 管理。

---

## 10. 源码复用与 action 映射

### 10.1 包映射

| 目标模块 | 来源 | 处理方式 |
|---|---|---|
| `internal/ipc` | 无 | 新增：hello/ready、行 JSON、pending 配对 |
| `internal/mengka` | 无 | 新增：action 客户端、能力探测、错误映射 |
| `internal/framework` | `cloud-assistant/internal/framework` | 迁移注册表/分发/菜单；增加能力门与权限过滤 |
| `internal/perm` | `cloud-assistant/internal/perm` | 增加“自身=主人”、数字角色、成员列表查询 |
| `internal/router` | `cloud-assistant/internal/router` | 改为 `AccountKey + 会话键` |
| `internal/config` | `cloud-assistant/internal/config` | 保留三层合并；移除未用的群管字段；增加 `protocol_id` |
| `internal/store` | `cloud-assistant/internal/store` | 保留业务表；新增权限/任务/能力/去重/卡密/授权/红包账本/点歌线路/红包规则表 |
| `internal/model` | `cloud-assistant/internal/model` | 重写为萌卡 NT 事件/消息段结构 |
| `internal/msgcache` | `cloud-assistant/internal/msgcache` | 保留内存 LRU |
| `internal/render` | `cloud-assistant/internal/render` | 保留模板变量 |
| `internal/cmdauto` | `cloud-assistant/internal/cmdauto` | 迁移 18 类；移除字符/被戳；升级秒评/说说/资料 |
| `internal/cmdcore` | `cloud-assistant/internal/cmdcore` | 迁移；移除取直链旧实现/自更新；增加查共群 |
| `internal/cmdqa` | `cloud-assistant/internal/cmdqa` | 迁移 |
| `internal/cmdtask` | `cloud-assistant/internal/cmdtask` | 迁移；新增多源搜索、线路换源、链接/语音/卡片；修复间隔单位 |
| `internal/license` | 无 | 新增：本地卡密、分群授权、代理、授权门禁 |
| `internal/cmdbiz` | 无 | 新增：红包引擎、授权/红包指令、账本清理 |
| `internal/mengka`（红包） | 无 | 新增：`get_red_packet_info`、`grab_red_packet` 与响应解析 |
| `webadmin/static/redpacket.js` | 无 | 新增：红包统计/排行/记录面板、卡密生成交互 |
| `internal/cmdstub` | `cloud-assistant/internal/cmdstub` | **不迁移** |
| `internal/ws` | `cloud-assistant/internal/ws` | 替换为 `internal/ipc` + `internal/mengka`；保留调试 WS |
| `internal/webadmin` | `group_management/internal/webadmin` | 参考布局与交互，重写数据层 |
| `internal/auth` / 联网授权 | `group_management` | **不迁移**；本地卡密/授权重新实现 |

### 10.2 源码缺陷修复清单

迁移时必须修复以下行为：

1. 权限：源码缺少“自身=主人”；新实现必须最高优先。
2. 权限：源码用 `0` 表示继承；新实现用 `-1`，`0` 保留给黑名单。
3. 账号：源码只用 `self_id`；新实现全部用 `AccountKey`。
4. 权限查询：源码用 `get_group_info`/`get_admin_list`；NT 改为 `get_group_member_list`。
5. 角色：源码按字符串 `owner/admin/member`；NT 是数字 `2/1/0`。
6. At：源码用 UID；NT 用 `user_id`。
7. 引用：源码用 `msg_seq/random/sender_uin`；NT 用 `reply_reference_id`。
8. 撤回：源码 `delete_msg` 不带 seq；NT 必须带 `msg_seq/msg_random`，好友还需 `client_seq/msg_time`。
9. 卡片：源码发 `json` 段；NT 只支持 `ark` 段。
10. 空间：源码按好友拉动态；NT 只有全局好友动态，需要按 `user_id` 过滤。
11. 秒评：源码占位；新实现用 `comment_qzone_feed`。
12. 删除上一条说说：源码占位；新实现用 `delete_qzone_feed`。
13. 无头衔撤回：源码只写开关；新实现必须真正撤回无头衔成员消息。
14. 图片模式：源码占位；新实现用 `x/image` 渲染并上传。
15. 查共群：源码占位；新实现遍历群/成员列表。
16. 续火间隔：源码把毫秒当分钟；新实现统一分钟。
17. 循环分类/类型：源码 no-op；新实现存为元数据并在控制台筛选。
18. 召唤人数：源码未使用；新实现限制召唤管理人数。
19. 艾特回复间隔：源码未使用；新实现为群级最小回复间隔。
20. 定时禁言：源码实际是全体禁言；新实现按“全体禁言/解禁”命名与帮助。
21. 验证默认：源码 120 秒/混合/踢出；文档不得写成 5 分钟。
22. 打卡 0 点：源码内存去重，重启会重复；新实现按日期落库。
23. 群发重启：源码自动续跑可能重复；新实现标记 interrupted。
24. 防撤白名单：源码是豁免名单；帮助与文档必须写明。
25. 静默模式：源码抑制所有 Reply；新实现保持一致并文档化。
26. 红包响应：源码没有该链路；新实现必须同时校验 `retcode=0` 与 `state=0`，金额只取 `recv_object.amount`。
27. 口令红包：源码无实现；新实现必须先发口令，再用 `client_seq`/`msg_random` 领取，不能传 `claim_token`。
28. 红包去重：源码无实现；新实现按账号 + `listid` 哈希唯一约束，失败不重试。
29. 授权：源码的联网授权不得迁移；新实现使用本地卡密/分群授权/代理，门禁默认关闭。

---

## 11. 里程碑

| 阶段 | 交付 | 验收 |
|---|---|---|
| M0 骨架 | 目录、`mengka-plugin.json`、go.mod、配置加载、日志、构建脚本 | `go build ./...` 通过 |
| M1 IPC 内核 | hello/ready、action 客户端、事件读取、账号键、能力探测 | 假 IPC 单测通过；stdout 仅 JSON |
| M2 模型与权限 | 事件归一化、六级权限、自身=主人、Worker 串行、消息缓存 | 权限矩阵单测通过 |
| M3 指令 | 382 条指令、35 个类目、能力门、菜单/帮助 | 菜单可列出全部有权指令 |
| M4 控制台 | 登录、catalog、通用渲染器、59 模块、CRUD、任务进度、红包/授权/点歌线路/在线更新 | 浏览器完成全流程 |
| M5 引擎 | 定时、循环、群发、验证、防撤、自身撤回、打卡、续火、秒赞秒评、多源点歌、五类红包/规则/语音、授权门禁 | 重启后任务恢复，失败可观测 |
| M6 质量 | 审计、备份恢复、迁移、并发与 panic、红包/授权测试、`go vet`、`go test -race` | 关键路径有测试，覆盖率 ≥60% |
| M7 发布 | Windows/Linux 包、README、升级说明、能力矩阵、校验和 | 全新节点首次安装可用 |

每阶段结束必须执行：

```bash
gofmt -w .
go vet ./...
go test ./...
go build ./...
```

---

## 12. 验收清单

- [ ] `go build ./...`、`go vet ./...`、`go test ./...` 通过。
- [ ] `mengka-plugin.json` 可被萌卡 NT 3.0 导入，hello/ready 正常。
- [ ] stdout 第一行是 hello，后续每行都是合法 JSON；无日志混入。
- [ ] 所有 action 均在清单中声明，未调用未声明接口。
- [ ] 账号全部使用 `self_id + protocol_id`，`protocol_id=0` 正常工作。
- [ ] 多账号/多协议配置、缓存、Worker、统计互不串扰。
- [ ] 首次启动生成 `data/admin.token`，控制台可登录。
- [ ] 382 条指令注册（基线 333 + 授权 16 + 红包 29 + 语音点歌 1 + 在线更新 3）；29 条不可用项不出现在菜单；7 个占位类目不存在。
- [ ] 所有指令默认 Lv5；可配置范围 1–5；`0` 不能分配。
- [ ] 机器人自身 QQ 发指令可直接触发。
- [ ] 群员/管理/群主/分管/黑名单按级别生效。
- [ ] 菜单/帮助/类目菜单按权限过滤。
- [ ] 能力缺失时对应指令不注册，控制台显示原因。
- [ ] 控制台 59 个模块可打开、可保存、可增删改查；红包页含五类/语音来源/规则，授权页含卡密/分群/代理，任务页含点歌线路，系统页含在线更新。
- [ ] 控制台账号/协议/群选择器工作正常。
- [ ] 任务进度 `job_id` 轮询、取消正常。
- [ ] 定时/循环/群发重启后可恢复；群发重启标记 interrupted。
- [ ] 写操作失败不自动重试。
- [ ] 审计日志覆盖所有配置、权限、主人、备份、令牌变更。
- [ ] 备份导出后可在空目录恢复（token 保留当前值）。
- [ ] 普通/拼手气/口令/专属五种文本红包、语音红包、低门槛、规则过滤与类型关闭均有测试；成功必须 `retcode=0 && state=0`。
- [ ] 点歌搜索换源、选歌、链接/卡片/语音回复均有测试；酷狗卡片回退链接。
- [ ] 口令红包先发口令，领取使用发送结果的 `client_seq`/`msg_random`，不传 `claim_token`。
- [ ] 同一账号同一 `listid` 只处理一次；失败不自动重试；`authkey`/`claim_token` 不出现在日志和账本。
- [ ] 卡密生成、兑换、续期、撤销、代理配额与门禁行为有测试；未授权群被门禁拦截，Lv5 豁免。
- [ ] 控制台红包页统计/排行/记录/设置可用；授权页卡密/分群/代理/设置可用。
- [ ] 启动与运行期间无授权服务器请求；卡密/分群/代理完全本地。
- [ ] Windows x64、Linux x86_64 与 Linux ARM64 包均可安装运行。
- [ ] README 包含安装、升级、卸载、已知限制。

---

### 12.1 v1.1 测试与验证

已完成的自动化测试：

| 文件 | 覆盖内容 |
|---|---|
| `internal/store/business_test.go` | 卡密生成/激活/续期/撤销、代理配额、红包账本去重/汇总/排行/清理 |
| `internal/store/music_rules_test.go` | 点歌线路默认种子/CRUD、红包规则 CRUD |
| `internal/license/service_test.go` | 卡密格式、代理配额、授权门禁默认关闭与开启、Lv5 豁免 |
| `internal/mengka/redpacket_test.go` | 红包类型、`retcode`/`state`/金额解析、`state` 缺失不误判成功 |
| `internal/cmdbiz/redpacket_engine_test.go` | 回复模板、数据清洗、账号/群配置覆盖 |
| `internal/cmdbiz/redpacket_rules_test.go` | 群/人员/关键词包含排除、每分钟限流 |
| `internal/cmdbiz/redpacket_voice_test.go` | TTS 音频/JSON 地址、固定音频下载、本地路径与临时文件清理 |
| `internal/cmdtask/music_search_test.go` | Meting/网易解析、来源别名、跨线路失败换源 |
| `internal/framework/license_gate_test.go` | 未授权群指令拦截、授权后放行 |
| `internal/cmdcount/count_test.go` | 382 条指令注册计数 |

已完成的集成验证：

- 假 IPC 节点回放 `group_message` + `red_packet`：普通/拼手气/专属成功领取、低门槛跳过、规则过滤、类型关闭、口令红包先发口令后领取；语音红包经音频准备→匹配→群语音→领取。
- 假 IPC 节点回放点歌：坏线路失败换源、搜歌选歌、QQ/网易云卡片、语音发送、酷狗/失败回退链接。
- 红包账本验证：同一 `listid` 不重复处理；成功金额取 `recv_object.amount`；完整流程与领取响应分别记录。
- 控制台浏览器验证：59 个模块；红包五类/规则/语音来源、点歌线路、在线更新正常；卡密生成、禁用/启用/删除、手动授权/撤销正常；无 JS 报错。
- 打包验证：Windows/Linux 包清单 54 个 action；Windows 包 hello 版本 `1.2.18`；控制台登录与静态资源加载正常。

---

## 13. 风险与决策

| 风险 | 影响 | 决策 |
|---|---|---|
| 平台接口/事件缺失 | 字符、被戳、进群屏蔽等不可用 | 能力探测 + 不注册 + 控制台说明 |
| 默认 Lv5 导致新用户“无反应” | 首次使用门槛高 | 控制台“推荐”预设 + 首次引导 |
| 自身消息不被框架回推 | 自身触发无法实测 | 规则保留；不依赖自身触发做自动化 |
| 自身消息回显 | 自激循环 | 发送指纹过滤 + 自动功能跳过自身 |
| `protocol_id` 误用 | 多协议串号 | 全链路双键；启动校验；控制台选择器 |
| 群发/怼人/秒赞触发风控 | 账号受限 | 默认低频、限量、白名单、风险提示 |
| 管理员变更无事件 | 权限缓存滞后 | TTL 5 分钟 + 手动刷新 |
| 禁言状态无接口 | 禁言列表/自动退群不可靠 | 本地记录 + 明确降级/下线 |
| 媒体 file_id 5 分钟 | 转发/清屏失败 | 立即发送；过期跳过并汇总 |
| Android-only 接口 | Linux 账号功能缺失 | 按协议类型能力门 + 明确提示 |
| Qzone 接口受限 | 秒赞/秒评失败 | 低频 + 去重 + 暂停而非重试 |
| SQLite 并发写 | 高峰期写放大 | WAL + 写串行 + 事务 |
| 控制台暴露公网 | 未授权访问 | 默认 127.0.0.1 + 令牌 + 会话 + CSRF + 限速 |
| 备份包含敏感数据 | 泄露 | 默认不含 token；不含消息缓存 |
| 配置迁移失败 | 启动异常 | 迁移前备份、幂等、失败回滚 |
| 外部点歌搜索 | 隐私/可用性 | 仅用户主动点歌时访问配置线路；搜索可关闭；失败回退链接 |
| 红包接口/协议差异 | 无法领取或金额误报 | 能力门 + `retcode/state` 双校验 + 金额只取 `recv_object.amount` |
| 口令红包发送失败 | 无法领取 | 缺 `client_seq`/`msg_random` 时放弃并记录，不猜测 |
| 红包重复/并发 | 重复领取或串号 | 账号 + `listid` 哈希唯一约束 + 账号级领取锁 |
| `authkey`/`claim_token` 泄露 | 凭据风险 | 不落库、不写日志；记录仅存 `listid` 哈希 |
| 语音红包 | 需要音频匹配与发送链路 | 默认内置 AI 语音；关闭发送、音频准备失败或匹配失败不领；仅 Android；记录原因 |
| 五类识别 | 上游 channel 可能合并普通/拼手气 | 优先显式 type，其次 channel；文档说明限制 |
| 点歌线路失效 | 搜索/音频不可用 | 优先级换源 + 回退链接；不自动重试 |
| 酷狗卡片 | 官方仅 QQ/163 卡片 | 酷狗回退链接或语音，不伪造卡片 |
| 授权门禁影响老用户 | 升级后指令不可用 | 默认关闭；控制台可配置提示；主人/自身豁免 |
| 本地卡密泄露 | 被恶意兑换 | 卡密仅管理员/代理可见；操作写审计；可禁用 |

---

## 附录 A：清单 actions / events

actions（54）：

```text
get_plugin_context, get_version_info, get_bot_list, get_bot_info, get_protocol_list,
get_group_list, get_group_member_list, get_friend_list,
send_group_msg, send_friend_msg, send_group_temp_msg,
recall_group_msg, recall_friend_msg, get_msg,
upload_group_image, upload_friend_image, set_msg_emoji_like,
set_group_mute, set_group_mute_all, set_group_admin, set_group_special_title,
set_group_card, kick_group_member, leave_group, sign_group,
get_group_honor_info, get_group_share_link, get_group_at_all_remain,
get_group_system_notifications, approve_group_apply, reject_group_apply,
get_friend_system_notifications, approve_friend_apply, reject_friend_apply,
set_friend_remark,
get_qzone_friend_feeds, like_qzone_feed, publish_qzone_feed,
comment_qzone_feed, delete_qzone_feed,
like_user_profile, get_received_profile_likes, get_sent_profile_likes,
set_qq_avatar, profile_operation,
get_music_ark, file_transfer, get_contact_ark,
get_red_packet_info, grab_red_packet,
upload_red_packet_voice, upload_group_voice, upload_friend_voice, grab_group_voice_red_packet
```

events（6）：

```text
group_message, friend_message, group_event, friend_event, bot_online, bot_offline
```

## 附录 B：事件 → 功能

| 事件 | 功能 |
|---|---|
| `group_message` | 指令、问答、艾特、表情回应、验证、防撤缓存、清屏缓存、无头衔撤回、红包识别与异步领取 |
| `friend_message` | 好友指令、好友问答、好友防撤缓存、临时会话指令 |
| `group_notice.increase` | 入群欢迎、入群验证、权限缓存失效 |
| `group_notice.decrease` | 退群/被踢提示、权限缓存失效 |
| `group_notice.recall` | 防撤回、撤回通知 |
| `group_notice.mute` | 禁言记录、自身被禁言状态 |
| `group_notice.apply` | 自动同意入群申请 |
| `friend_notice.increase` | 新好友回复、自动备注 |
| `friend_notice.decrease` | 被删通知 |
| `friend_notice.recall` | 好友防撤 |
| `friend_notice.like` | 自动回赞 |
| `friend_notice.apply` | 自动同意好友 |
| `bot_online` | 刷新账号/群列表；补处理待审批；恢复任务 |
| `bot_offline` | 暂停发送与任务；记录离线原因 |

## 附录 C：配置项索引

| 配置文件 | 关键项 |
|---|---|
| `global.json` | masters、default_group、global_blacklist、summon_command、menu_header、menu_footer、debug、self_trigger_auto、permission_denied_hint、send_rate_limit_ms、mass_interval_s、reply_mode、silent_mode、license、music、web |
| `accounts/<self>/<protocol>.json` | enabled、masters、master_trigger、菜单覆盖、reply_mode、silent_mode、feature_switches、red_packet（五类/语音发送与来源/规则）、permission_overrides |
| `groups/<self>/<protocol>/<group>.json` | follow_default、managers、blacklist、whitelist、notice、verify、anti_recall、at_handler、title、music（模式/源/音质/换源）、red_packet、permission_overrides |
| `permissions.json` | 指令键 / `category:xxx` → min_level（1–5） |
| `admin.token` | 控制台登录令牌 |

## 附录 D：指令键与 Pattern DSL

- `Exact`：全文相等。
- `Prefix`：最长前缀匹配，剩余文本进 `Args.Rest`。
- `Pattern`：
  - 字面量：必须逐字出现；
  - `@`：消费一个 At 段；
  - `N`：整数；
  - `秒`：整数秒；
  - `A`：剩余全部文本，必须放最后。
- 匹配优先级：Exact > Prefix（最长）> Pattern > Fallback。
- 权限键推导：
  - Exact → 原文；
  - Prefix → 去掉尾部空白；
  - Pattern → 字面量 + `@` 拼接，如 `禁言@`。
- 同一触发词重复注册，启动直接失败退出。
- 别名共享同一 handler 与权限。
- 类目名可作为 Fallback，但不得覆盖同名真实指令。
