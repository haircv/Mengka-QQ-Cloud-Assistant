# QQ云助手 · AI 开发提示语 v3.2

> 配套文档：`docs/QQ云助手-插件功能开发文档.md`
> 使用方式：把两份文档一起发给编码 Agent。先完整阅读配套文档，再动手写代码。
> 本提示语是执行指令；功能细节、指令清单、控制台模块、验收标准以配套文档为准。
>
> **v1.2 增补（2026-10-28）**：五类红包独立控制（普通/拼手气/口令/专属/语音）、群/人员/关键词规则、Android 限制与 TTS 语音领取；多源点歌（QQ/网易云/酷狗）、线路优先级与失败换源、链接/语音/卡片回复。指令 382 条，action 54 个，控制台 59 个模块。
> **v1.2.13 增补（2026-10-29）**：运行状态登录协议改为 `Android Pad(9.3.75)` 并调整实时状态时间顺序；点歌增加音频直链探测、卡片错误日志与语音 Base64 回退；红包新增五类开关、标题关键词包含/排除、自己红包默认领取（合计 382 条）；群配置变量提示、分管/黑名单仅 QQ、清屏/召唤/点歌作用域修复、禁言解禁与定时弹窗修复；托管管理接口增加目标地址提示、测试按钮与 502/401 区分。
> **v1.2.14 增补（2026-10-29）**：在线更新导入自动解析萌卡服务节点（单节点自动、多节点可配置 `update.node_id`），修复“请选择服务节点”；数据目录优先使用 `MENGKA_PLUGIN_DATA_DIR`，更新/重新导入自动迁移并保留数据库、配置、授权与日志。
> **v1.2.15 增补（2026-10-29）**：红包分类与请求兼容调整；领取转发完整 `red_packet`，兼容 `claim_token`/`pre_grap_token`，凭据失效自动重查重试；语音红包新增发送开关与内置 AI 语音/自定义 TTS/固定音频三种来源，关闭发送时跳过，官方接口不支持不发送语音直接领取。
> **v1.2.16 增补（2026-10-29）**：在线更新导入后自动调用萌卡启动接口启用插件；红包请求同时提交嵌套/扁平字段与 `sender_id`/`sender_uin`，凭据失败按“新凭据→无凭据→自己红包延迟”多级重试；语音音频改用 Base64 上传，规避萌卡本地媒体目录限制；登录页明确萌卡端口不固定并提示 admin.token 回退。
> **v1.2.17 增补（2026-10-29）**：`开启点歌功能`同时开启外部搜索，卡片/语音点歌缺少搜索时给出明确指引；禁言/解禁/踢人/上下管支持 @ 数值、昵称、UID 映射与引用消息回退解析，并记录目标解析诊断日志；同一 QQ 的 Android 与 Linux/PC 同时在线时，默认仅 Android 实例生效，其他协议静默（账号级开关可关闭）。
> **v1.2.18 增补（2026-10-29）**：撤销 1.2.16 的扁平红包字段，`grab_red_packet` 只提交嵌套 `red_packet`、`group_id`、`sender_id` 和可选 `claim_token`；自动启用只匹配新版本实例，并新增独立启动助手，避免导入替换进程时丢失启动调用。

---

## 1. 角色与任务

你是一名资深 Go 后端工程师，长期维护 QQ 机器人插件，熟悉：

- 萌卡 NT 3.0 的原生 IPC 插件协议；
- `self_id + protocol_id` 双键账号模型；
- SQLite（WAL）并发写入与无框架 Web 控制台；
- 高并发事件路由、指令注册表、权限引擎、任务调度。

你的任务是**直接实现一个可编译、可安装、可运行、可验收的插件**，不是写方案、伪代码或页面草图。

---

## 2. 目标产物

在仓库下新建插件工程 `qq-cloud-assistant/`，实现：

| 项目 | 要求 |
|---|---|
| 中文名 | QQ云助手 |
| 英文名 / 插件 ID | `qq-cloud-assistant` |
| 框架 | 萌卡 NT 3.0 |
| 接入形态 | 原生 IPC 插件（`native-ipc-v1`），可选正向 WebSocket 调试 |
| 语言 | Go 1.25 |
| 依赖 | `modernc.org/sqlite`（无 cgo）、`golang.org/x/image`（图片菜单）、`github.com/gorilla/websocket`（仅调试） |
| 存储 | SQLite（`data/qca.db`）+ 分层 JSON 配置 |
| 控制台 | 插件自建 Web 控制台，默认 `127.0.0.1:18084`；59 个模块，含红包、授权、点歌线路、在线更新 |
| 功能基线 | `cloud-assistant 1.0.0` 的全部可运行功能 |
| 指令 | 源码 363 条真实指令；v1.2 注册 382 条；29 条明确不注册 |
| action | 54 个；包含红包查询/领取/语音与好友/群语音上传 |
| 红包 | 五类独立开关、群/人员/关键词规则、期望过滤、Android、语音来源（内置 AI/自定义 TTS/固定音频）、账本与排行 |
| 点歌 | QQ/网易云/酷狗多源、选歌、链接/语音/卡片、线路优先级与失败换源 |
| 权限 | 0 黑名单 / 1 群员 / 2 管理 / 3 群主 / 4 分管 / 5 主人 |
| 默认权限 | **每一条指令默认 Lv5**，包括菜单、帮助、运行状态、红包/授权/点歌；低级使用只能在控制台显式下调 |
| 自身账号 | 发送者 QQ == 机器人 QQ，直接 Lv5 |
| 授权 | **本地卡密/分群授权/代理**：无授权站、应用 ID、心跳、远程校验、到期退群；门禁默认关闭 |
| 产物 | Windows x64、Linux x86_64 与 Linux ARM64 各一个 IPC 插件包，同一源码构建 |

参考资料：

- 官方 API 与事件：<https://mengka.net/nt/api/>、<https://mengka.net/nt/events/>
- IPC 插件导入：<https://mengka.net/nt/reference/managed-plugins.html>
- 通信协议：<https://mengka.net/nt/reference/protocol.html>
- 消息段：<https://mengka.net/nt/reference/message-segments.html>
- 社区文档：<https://github.com/shiina0118/MengKa-Docs>
- 开源参考：<https://github.com/Jay1023CN/Mengka-NT-plugins>
- 本地源码：`QbMaxGo云端源码.zip`（功能基线）、`Group-Management.zip`（控制台参考）、`QbMaxGo群管源码.zip`（可选审计/备份参考）
- 功能契约：`docs/QQ云助手-插件功能开发文档.md`

以官方文档和功能契约为准；旧版 QbMaxGo 的 action 名、事件字段、消息段不能直接沿用。

---

## 3. 不可协商的约束

### 3.1 IPC 形态

1. 插件根目录必须有 `mengka-plugin.json`，schema 3，`transport` 为 `native-ipc-v1`。
2. 启动后 stdout 第一行必须是：

```json
{"type":"hello","api_version":"1","framework_api":"3.0.0","plugin_id":"qq-cloud-assistant","version":"1.2.18"}
```

3. 等待 stdin 返回 `{"type":"ready"}` 后才能发 action 请求；10 秒未收到则退出。
4. **stdout 只能输出逐行 JSON 协议消息**。所有日志、panic 堆栈、调试信息写 stderr。
5. 写 stdout 必须加锁，保证一行一个完整 JSON。
6. 读 stdin 使用可增长缓冲，至少支持 16 MiB 单行。
7. 同一连接可并发，响应顺序不保证；必须用 `id` 配对，禁止依赖顺序。
8. 每个 action 必须有超时（默认 30 秒）；写操作失败**不得自动重试**。
9. 支持 SIGINT / SIGTERM / stdin EOF 优雅退出：停止接收 → 排空队列（最多 5 秒）→ 关闭 DB/HTTP。

### 3.2 清单权限

`runtime.actions` 必须列出插件实际调用的全部 action；不得多声明，也不得调用未声明接口。

v1.2 使用 54 个 action：

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

`runtime.events` 使用事件权限名：

```text
group_message, friend_message, group_event, friend_event, bot_online, bot_offline
```

不要声明 `approve_group_invite`、`get_group_msg_history`、`recall_message`、`send_poke`、`unlike_qzone_feed`。

### 3.3 账号

1. 账号必须用 `self_id + protocol_id` 唯一选择。
2. `protocol_id` 来自 `get_protocol_list`，**0 是有效值**，不得用 0 表示“未设置”。
3. 任何缓存、配置、Worker、统计、任务都必须带 `AccountKey`。
4. 禁止只用 `self_id` 做 key。
5. 插件安装时绑定一个节点，只能操作该节点账号。
6. 控制台账号选择器必须同时显示 `self_id` 与 `protocol_id`。

### 3.4 权限：六级，默认仅主人

1. **每一条指令**默认 `MinLevel = 5`，包括 `菜单`、`帮助`、`运行状态`。
2. 注册器内部用 `LevelUnset = -1` 表示“未设置”，注册时归一化为 5。**禁止**用 `0` 表示未设置，`0` 是黑名单。
3. 可配置的指令级别范围是 **1–5**；`0` 不能分配给指令。
4. 子指令未设置继承父类目；父类目未设置则为 5。
5. 移除 `AlwaysAvailable`、白名单旁路等权限捷径。
6. 菜单和帮助只展示调用者有权使用的指令。
7. 判定顺序：

```go
const (
    LevelBlacklist = 0
    LevelMember    = 1
    LevelAdmin     = 2
    LevelOwner     = 3
    LevelSubAdmin  = 4
    LevelMaster    = 5

    LevelUnset = -1
)

func (e *Engine) Level(ev *Event) int {
    // 1. 机器人自身发送者 = 主人，最高优先，免疫黑名单
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
    // 5. 私聊非主人 = 群员
    if ev.GroupID == 0 {
        return LevelMember
    }
    // 6. 事件自带角色（数字：2 群主 / 1 管理 / 0 成员）
    //    缺失时 RoleKnown=false，不能把零值当成员
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
    // 7. 成员列表查询；5 分钟缓存；失败 fail-closed 为 Lv1
    return e.memberLevel(ev.AccountKey(), ev.GroupID, ev.SenderID)
}
```

8. 权限覆盖优先级：

```text
群级覆盖 > 全局指令 > 全局类目 > 注册默认值 5
```

9. 控制台提供“严格 / 推荐 / 自定义”三档预设；所有修改写审计日志。
10. 所有指令注册默认 Lv5；红包/授权/点歌不预设低级默认。群主兑换或代理生成需由主人在控制台下调级别；处理器仍校验主人/代理身份。授权门禁只做群级拦截，不改变注册默认。

### 3.5 自动功能与自身消息

1. 入群欢迎、退群提示、入群验证、防撤、问答、艾特、表情回应、秒赞/秒评、自动回赞、定时/循环/群发、打卡、续火、自动说说、红包领取等自动功能**不受指令级别限制**，只受功能开关、名单与授权门禁控制。
2. 若框架推送机器人自身消息：
   - 不丢弃；
   - 权限直接 Lv5；
   - 可触发指令；
   - 一级类目名（如 `通用功能`、`点歌功能`）必须走类目 Fallback 并回复菜单，不能被自身消息的提前返回吞掉；
   - 非 Lv5 发送者发送类目名或无可见指令时保持静默，不回“该类目暂无可用指令”；
   - **不得**进入问答、艾特回复、表情回应、防撤转发、自动欢迎、自动回赞、秒赞/秒评；
   - 红包自动领取是例外：自身消息走专用 self message hook，`red_packet.grab_self` 默认 true 时自己发送的群红包也会处理；关闭后记录“跳过·自己发送”。
   - 与最近 10 秒内本插件发送内容指纹一致的自身消息，整条忽略。
3. `self_trigger_auto` 默认关闭；开启时增加 3 秒去重与“输出不得包含触发词”检查。
4. 群发、循环、定时、续火、说说发出的消息必须带任务标记，不得再次触发自动回复。

### 3.6 免联网授权与本地卡密

- 不实现、不引入、不调用任何授权站、应用 ID、心跳、机器码、远程校验、远程配置下发、到期退群。
- 不迁移 `group_management` 的 `internal/auth`、`plugin-home` 与联网授权页面。
- 本地实现卡密、分群授权、代理与可选门禁；卡密只在本地数据库生成与核销。
- 启动、运行、退出全程无授权网络请求；除用户主动开启的点歌搜索外无外联。
- 控制台提供“授权”分区（卡密/分群/代理/设置），不出现支付、订单、发卡、收款入口。
- 门禁默认关闭；开启后仅拦截群内指令，主人/自身豁免，授权类指令与兑换始终可用。
- 不实现支付、订单、收款、提现、伪造聊天、闪照监控。
- 红包只使用官方 `red_packet` 消息段与查询/领取/语音接口；语音红包仅 Android，默认内置 AI 语音，也可用自定义 TTS 或固定音频；禁止记录或输出 `authkey`、`claim_token`。
- 口令红包必须先发送口令，再用发送结果 `client_seq`、`msg_random` 领取；缺少有效标识时放弃。
- 领取成功必须同时满足 `retcode=0` 与 `state=0`，金额只取 `recv_object.amount`，不得把总额当成本次到账。
- 失败不自动重试；同一账号同一 `listid` 只处理一次。

### 3.7 功能范围

以配套文档第 3、5 节为准：

- 382 条指令、35 个可运行类目（含授权、红包、点歌）；
- 29 条不可用指令不注册；
- 7 个占位类目整体移除；
- **不得改动任何中文指令名**；
- **不得用空 handler 或“暂未开放”冒充实现**；
- 能力缺失时从注册表移除，并在控制台标记原因。

不可用清单（29 条）：

| 类目 | 数量 | 原因 |
|---|---:|---|
| 字符功能 | 5 | 无抽字符接口 |
| 被戳处理 | 6 | 无 poke 事件 |
| 进群自动屏蔽/免打扰/群助手 | 6 | 无账号级群设置接口 |
| 上管/下管提示 | 6 | 无管理员变更事件 |
| 一键退出禁言群/全禁群等 | 4 | 无禁言/全禁状态查询接口 |
| 检测更新/更新插件 | 2 | 无插件自更新接口 |

### 3.8 工程约束

- 所有 SQL 参数化，禁止拼接 SQL。
- 配置写操作使用 RFC 7386 Merge Patch，禁止整体覆盖丢字段。
- 所有配置变更写 `audit_log`。
- 每个 Worker、每个 HTTP Handler、每个任务协程捕获 panic。
- 日志不得输出令牌、Cookie、完整消息内容；调试模式也只记录截断文本。
- 代码注释只写“为什么”，不写“是什么”。
- 不留下 `TODO` 当作已完成；未实现项必须从注册表移除并在文档中列出。
- 不修改参考源码目录。

---

## 4. 官方协议事实

以下内容来自萌卡 NT 3.0 官方文档，直接照做，不要凭旧版 OneBot/QbMaxGo 经验猜。

### 4.1 IPC 帧

请求：

```json
{"type":"action","id":"qca-1","action":"get_group_list","params":{"self_id":123456789,"protocol_id":0}}
```

成功响应：

```json
{"type":"action_result","id":"qca-1","ok":true,"data":{}}
```

失败响应：

```json
{"type":"action_result","id":"qca-1","ok":false,"error":"账号未在线"}
```

事件：

```json
{"type":"event","data":{"self_id":123456789,"protocol_id":0,"post_type":"group_message"}}
```

规则：

- 无 `id` 的请求不会响应，插件不得使用。
- 每连接最多并发 16 个 action；插件内部限制 8 个。
- 单条请求 JSON 最大 4 MiB。
- 连接关闭时，所有 pending 返回“连接已关闭”。
- `status=restricted` 不是成功；`queries_beyond_limit` 暂停查询，不重试。
- 业务接口还要检查 `success`、`error_code`、`code`。

### 4.2 事件

`post_type`：

```text
group_message
friend_message          # 含 group_temp_message
group_notice            # group_event
friend_notice           # friend_event
bot_online
bot_offline
```

`group_notice.sub_type`：

```text
increase / decrease / recall / mute / apply
```

`friend_notice.sub_type`：

```text
increase / decrease / recall / like / apply
```

关键字段：

- `sender.role` 是**数字**：`0` 成员、`1` 管理员、`2` 群主；缺失时 `RoleKnown=false`，必须查询成员列表，不能把零值当成员。
- 好友消息触发者取 `sender.user_id`。
- `group_temp_message` 的 `group_id` 可能缺失；缺失时不做群级权限、不做引用回复。
- 没有管理员变更事件、没有 poke 事件、没有群名片变更事件、没有群名变更事件、没有文件上传事件。
- 撤回原文 `body.original` 只保留 5 分钟；过期正文为“消息已过期”。

### 4.3 消息段

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

- 媒体先上传，`file_id` 有效期 5 分钟。
- 群视频必须单独发送；好友/临时语音、视频必须是唯一消息段。
- 引用优先 `reply_reference_id`；接收引用段里的字符串 `message_id` 是 QQ MsgUID，不能作为发送引用编号。
- At 只接受 `user_id`，不接受 `uin`/`uid`。
- 不支持的段必须报错，不能静默丢弃。
- 图片菜单上传失败回退文字菜单。
- `red_packet` 只用于识别/领取；`authkey`、`claim_token` 不落日志/账本；`listid` 只存哈希；语音红包走音频准备（内置 AI/自定义 TTS/固定音频）→匹配→群语音→专用领取。

### 4.4 关键字段映射

| 旧版 | 萌卡 NT 3.0 |
|---|---|
| `set_group_kick` | `kick_group_member` |
| `set_group_ban` | `set_group_mute` |
| `set_group_whole_ban` | `set_group_mute_all` |
| `get_group_info` / `get_admin_list` | `get_group_list` / `get_group_member_list` |
| `quit_group` | `leave_group` |
| `group_signin` | `sign_group` |
| `delete_msg` | `recall_group_msg` / `recall_friend_msg` |
| `group_msg_reaction` | `set_msg_emoji_like` |
| `get_qzone_feeds` | `get_qzone_friend_feeds`（全局流） |
| `qzone_like_http` | `like_qzone_feed`（传完整 feed） |
| `publish_mood` | `publish_qzone_feed` |
| `like_card` | `like_user_profile` |
| `set_nickname` / `set_signature` | `profile_operation` |
| `approve_friend_request` | `approve_friend_apply` |
| `handle_group_request` | `approve_group_apply` |
| `json` 消息段 | `ark` 消息段 |
| 引用 `msg_seq/random` | `reply_reference_id` |

---

## 5. 功能实现要求

### 5.1 必做类目

1. 通用功能：菜单、类目菜单、帮助、运行状态、在线更新、检查更新、立即更新；运行状态用 `get_bot_info`/`get_bot_list` 回退，在线从 `login_time` 起算，框架显示 `app_name + app_version`；在“当前Bot”下输出登录协议，格式为 `Android Pad(9.3.75)`（平台 + 设备类型 + 版本）；实时状态末尾依次输出在线、登录时间、当前时间。
2. 群管基础：踢/禁言/解禁/上下管/全体禁言/解禁/解禁全部/禁言列表/群信息/取码/复读/取直链/加群链接。
3. 提示：入群、退群、被踢、被删、被踢通知 + 模板。
4. 菜单配置、回复配置、召唤、清屏、怼人、个人助手（仅查共群）。
5. 自动化 18 类：防撤、自身撤回、艾特、秒赞秒评、点赞、互赞、表情回应、续火、说说、自动处理、名片、打卡、资料、备注、头衔、入群验证；被戳、字符整类不注册。
6. 问答：本群/好友/全局 × 精准/模糊，白名单、延迟、随机条数、冷却。
7. 任务：定时、循环、群发、点歌。
8. 账号：主人菜单、触发模式、账号分组、主群。
9. 红包：五类独立控制、群/人员/关键词规则、期望过滤、拟人延迟、到账回复、语音来源（内置 AI/自定义 TTS/固定音频）、只领不回复名单、账本与排行。
10. 本地授权：卡密、分群授权、代理配额、可选门禁；禁止联网授权。

### 5.2 行为要求

- **发送**：默认 800ms/条，按账号串行；群发任务默认 3000ms，最小 1000ms。
- **定时**：30 秒扫描，`next_run` 落库，重启续跑，执行中防重入。
- **循环**：按间隔分钟计算下一次；失败记录后继续；连续失败 10 次停用并通知主人。
- **群发**：任务持久化；重启后未完成任务标记 `interrupted`；失败计数，不自动重试；可停止；统计成功/失败。
- **入群验证**：数字/字母/混合/发言；默认 120 秒；超时踢/禁言/不处理；默认禁言 10 分钟；会话内存存储，重启记录日志。
- **防撤**：缓存 10 分钟；本群/全局/好友三种作用域；白名单群豁免；不转发自身消息；原文优先 `body.original`。
- **自身撤回**：发送成功后按延迟入队；群用 `msg_seq+msg_random`，好友再加 `client_seq+msg_time`；失败只记日志。
- **清屏**：只撤回缓存中仍可定位的消息；超时/无权限跳过并汇总；不发送零宽字符。
- **打卡/续火**：支持一键与定时，完成后私聊发起人汇总。
- **点歌**：`点歌功能` 是类目菜单（类目名优先于命令前缀）；搜索用 `点歌 <歌名>`/`搜歌 <歌名>`；QQ/网易云/酷狗多源、线路换源、选歌、链接/语音/Ark 卡片；酷狗卡片回退。
- **秒赞/秒评**：轮询 `get_qzone_friend_feeds`，按白名单好友过滤；动态去重；不重复操作同一条。
- **资料**：昵称/个签用 `profile_operation`；头像用 `file_transfer` + `set_qq_avatar`；失败先查询实际状态，不盲目重试。
- **查共群**：遍历本账号群列表与成员列表；限制 50 群、2 秒/群、10 分钟缓存、每人 10 分钟一次。
- **菜单**：由注册表实时生成，按权限过滤；菜单头尾、召唤命令可配置；图片模式用 `x/image` 渲染。

### 5.3 最容易做错的 12 件事

1. 用旧 action 名（`set_group_ban`、`quit_group`、`delete_msg`、`group_signin`）——必须换成 NT 名称。
2. 把 `sender.role` 当字符串——NT 是数字 `0/1/2`。
3. 把 At 写成 `uid`——NT 只接受 `user_id`。
4. 用 `msg_seq` 做引用——NT 引用用事件顶层 `message_id` 作为 `reply_reference_id`。
5. 用 `delete_msg` 撤回——NT 必须用 `msg_seq/msg_random`，好友还要 `client_seq/msg_time`。
6. 发 `json` 消息段——NT 只支持 `ark`。
7. 按好友拉空间动态——NT 只有全局好友动态，需要按 `user_id` 过滤。
8. 保留源码的占位实现——秒评、删除上一条、无头衔撤回、图片模式、查共群必须真正实现或明确不注册。
9. 忘记“自身=主人”——源码没有这条，新实现必须有，且最高优先。
10. 用 `0` 表示权限未设置——`0` 是黑名单；未设置用 `-1`。
11. 只用 `self_id` 做 key——必须 `self_id + protocol_id`。
12. 把验证超时写成 5 分钟——源码默认 120 秒。

---


### 5.4 v1.2 红包与点歌实现要点

红包：

- 五类独立开关：`type_normal`、`type_lucky`、`type_password`、`type_exclusive`、`type_voice`；默认语音关闭。
- 类型识别：显式 `type`/`packet_type` 优先；否则 `channel` 6/32 口令、65536 语音、1024 专属、2 拼手气，其余普通。
- 账号 `require_android=true` 时，非 Android 平台记 `android_only`，不调用领取。
- 规则：`red_packet_rules` 支持 group/person/keyword × include/exclude；关键词按标题；排除优先，包含按维度校验。
- 普通/拼手气/专属：查询总额/份数/`claim_token`；期望值低于门槛跳过；`query_before_grab=false` 时跳过查询与门槛过滤。
- 口令红包：先发送口令，再用发送结果 `client_seq`、`msg_random` 领取；不传 `claim_token`，不使用红包消息 `msg_seq`。
- 语音红包：准备标题音频（默认内置 AI 语音；可选自定义 TTS 或固定文件/URL），本地/生成音频用 Base64 上传 → `upload_red_packet_voice` → `upload_group_voice` 发送 → `grab_group_voice_red_packet`；关闭“发送语音”时跳过，匹配失败不领取。
- 成功必须同时满足 `retcode=0` 与 `state=0`；金额只取 `recv_object.amount`，缺失标记未知且默认不回复。
- 按账号 + `listid` 哈希去重；每分钟限流 `max_per_minute`；失败不重试；`authkey`、`claim_token` 不落库、不写日志。

点歌：

- 线路表 `music_lines` 支持 `meting`、`meting_legacy`、`netease` 格式，按 priority 升序；首选来源优先，失败后按 `failover_enabled` 尝试其他来源。
- `点歌 [QQ/网易云/酷狗] <歌名>`（别名 `搜歌`）；解析结果字段兼容 `title/name`、`author/artist`、`url/audio`、`pic/cover` 与 `result.songs`。
- 搜索结果最多 5 条供选歌，60 秒内回复序号；关闭选歌直接发送第一条。
- 回复：链接使用歌曲页（酷狗用搜索页）；卡片仅 QQ/网易云调用 `get_music_ark`；语音用 `upload_group_voice` / `upload_friend_voice`；语音失败回退卡片，卡片失败回退链接。
- 搜索总耗时上限 20 秒、最多 8 条线路；全部失败回退搜索链接；不缓存版权内容。

授权：

- 本地 SQLite 实现 `license_cards`、`group_authorizations`、`agents`；禁止任何联网授权、应用 ID、心跳、远程配置。
- 卡密格式 `QCA-XXXX-XXXX-XXXX`；生成时校验代理状态与配额；群主在群内使用后为本群开通/续期，续期从当前有效到期时间顺延。
- 授权门禁 `license.enforce` 默认关闭；开启后只拦截未授权群的指令，Lv5（主人/自身）与 `license` 类目豁免。
- 控制台提供卡密/分群授权/代理/授权设置；红包引擎在门禁开启时只处理已授权群。
- 控制台与红包页必须复用现有 QSign 深色样式，不引入外部 CDN。

## 6. 工程结构

```text
qq-cloud-assistant/
├── mengka-plugin.json
├── go.mod
├── main.go
├── imports.go
├── internal/
│   ├── ipc/              # hello/ready、行 JSON、pending 配对、读写循环
│   ├── mengka/           # action 客户端、账号键、能力探测、错误映射
│   ├── framework/        # 注册表、分发、Ctx、Pattern、菜单/帮助
│   ├── perm/             # 六级权限引擎
│   ├── router/           # 每账号每会话串行 Worker
│   ├── config/           # 三层配置 + Merge Patch + 迁移
│   ├── model/            # 萌卡 NT 事件/消息段归一化
│   ├── msgcache/         # 内存 LRU 消息缓存
│   ├── render/           # 模板变量
│   ├── store/            # SQLite + JSON
│   ├── cmdauto/          # 自动化类目
│   ├── cmdcore/          # 群管、提示、菜单、召唤、清屏、怼人、助手
│   ├── cmdqa/            # 问答
│   ├── cmdtask/          # 定时、循环、群发、多源点歌/线路搜索
│   ├── license/          # 本地卡密、分群授权、代理、门禁
│   ├── cmdbiz/           # 红包引擎、授权/红包指令、账本清理
│   ├── webadmin/         # 控制台后端 + embed 静态资源（含 redpacket.js）
│   ├── job/              # 一键任务进度
│   └── bootstrap/        # token、数据目录、初始化、能力快照
├── packaging/
│   ├── windows/mengka-plugin.json
│   └── linux/mengka-plugin.json
└── scripts/
    ├── build-all.sh
    └── build-all.ps1
```

构建与打包：

```bash
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build -trimpath   -ldflags "-s -w -X main.version=1.2.18"   -o dist/windows-amd64/qq-cloud-assistant.exe .

CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath   -ldflags "-s -w -X main.version=1.2.18"   -o dist/linux-amd64/qq-cloud-assistant .
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -trimpath   -ldflags "-s -w -X main.version=1.2.18"   -o dist/linux-arm64/qq-cloud-assistant .
```

- Windows 包清单 `entry` 为 `qq-cloud-assistant.exe`；Linux 包为 `qq-cloud-assistant`。
- 包根目录同时包含清单与可执行文件；不包含 `data/`、`admin.token`、数据库。
- `scripts/build-all.sh` / `scripts/build-all.ps1` 构建、打包、生成 SHA256。
- 两个包分别解压到空目录验证导入与首次启动。

要求：

- 单进程承载节点内多个账号。
- 控制台与 IPC 解耦：框架断线时仍可查看配置与日志，但不能发送。
- 账号离线时暂停发送类任务，定时任务保留 `next_run`。
- 捕获 panic；队列满丢弃并记日志，不阻塞读循环。
- 事件处理不得阻塞 stdin 读循环。

---

## 7. 数据与配置

### 7.1 SQLite

至少包含：

```text
schema_meta, qa_rules, white_lists, reply_pool,
scheduled_tasks, loop_tasks, mass_jobs, jobs,
account_groups, account_group_members, group_managers,
bot_switches, command_permissions, mute_records, qzone_seen,
group_cache, counters, uin_map, audit_log, capability_snapshots,
license_cards, group_authorizations, agents, red_packet_records,
music_lines, red_packet_rules
```

要求：

- 所有业务表带 `self_id` 与 `protocol_id`。
- `command_permissions.min_level` 默认 5；只允许 1–5。
- `white_lists.kind` 覆盖所有名单类型。
- `qzone_seen` 用于动态去重。
- `mute_records` 只用于降级的禁言列表，不代表全量禁言。
- WAL 模式；写操作串行；批量导入使用事务。
- 迁移幂等；迁移前自动备份；失败回滚。
- `license_cards`、`agents` 为全局表；`group_authorizations`、`red_packet_records` 必须带 `self_id + protocol_id`。
- `red_packet_records` 以 `listid` 的 SHA-256 建唯一约束；不保存 `authkey`/`claim_token`；`music_lines` 为全局线路；`red_packet_rules` 按账号隔离；schema 版本 5。

### 7.2 JSON 配置

```text
data/global.json
data/accounts/<self_id>/<protocol_id>.json
data/groups/<self_id>/<protocol_id>/<group_id>.json
data/permissions.json
data/admin.token
```

读取顺序：

```text
内建默认 → global.default_group → 账号配置 → 群配置
```

- `follow_default=true` 时群配置只存差异。
- 所有写操作 Merge Patch。
- `global.json` 增加 `license` 与 `music`（模式/来源/音质/换源/选歌/结果数）；账号配置增加 `red_packet`（五类开关、语音发送开关、音频来源、规则、监听、延迟、门槛、回复、限额）；群配置可覆盖 `red_packet` 与 `music`。
- `admin.token` 32 字节随机十六进制，0600，首次启动生成并打印一次。
- 数据目录解析：`QCA_DATA_DIR` → 用户配置目录（稳定，重新导入不变） → 首次启动从旧版可执行目录/本地缓存原子迁移。
- 不可写时明确报错退出，不静默降级。

---

## 8. Web 控制台

### 8.1 形态

- 默认 `127.0.0.1:18084`；端口占用时尝试 18085–18093。
- 原生 HTML/CSS/JS，`embed.FS`，无构建工具、无外部 CDN。
- 自建登录、会话、CSRF、权限校验。
- 令牌只保存在后端，不注入前端、URL 或浏览器长期存储。
- 支持 `--console-only` 做 UI 开发与验收。

### 8.2 视觉

参考 `group_management` 的 QSign 主题：

- 侧栏 `hsl(222.34 10.43% 12.27%)`，工作区 `hsl(220 13.06% 9%)`；
- 主色 `hsl(212 100% 45%)`，成功 `hsl(144 57% 58%)`，警告 `hsl(42 84% 61%)`，危险 `hsl(348 100% 61%)`；
- 圆角 8px，控件 32px，侧栏展开 224px / 收起 64px，顶栏 50px，标签栏 38px，字号 14px；
- 登录页左侧品牌区 + 右侧登录卡；
- 内容区：卡片/表单/表格/按钮，弹窗、确认框、Toast；滚动条透明细样式。
- 侧栏桌面可平滑收起为 64px 图标栏，≤900px 为抽屉+遮罩；头部操作区可换行，表格在窄屏内部横向滚动，所有宽度不得出现页面级横向溢出或内容遮挡。

### 8.3 结构

59 个目录驱动模块（57 个可操作 + 2 个只读缺位说明）：

| 分区 | 模块 |
|---|---|
| 概览 | dashboard、runtime |
| 机器人 | bots、bot-settings、trigger、main-groups、account-groups |
| 群管理 | groups、group-config、managers、blacklist、whitelist、notice、menu-config、reply-config、basic、summon、clearscreen、duel、assistant |
| 自动化 | anti-recall、self-recall、at-handler、auto-like、like-card、mutual-like、emoji、keep-fire、mood、auto-process、card、checkin、profile、remark、title、verify、poke（disabled）、lucky-word（disabled） |
| 问答 | qa、qa-advanced |
| 任务与群发 | timer、loop、broadcast、music、music-lines |
| 红包 | red-packet、red-packet-group |
| 授权 | license-cards、group-authorizations、agents、license-settings |
| 权限 | masters、command-permissions |
| 系统 | logs、backup、update、about、web-settings、capabilities |

要求：

- 账号选择器显示 `昵称（QQ / 协议ID）`；群级模块显示群选择器。
- 所有请求带 `self_id`、`protocol_id`、`group_id`。
- `GET /api/admin/catalog` 返回 sections → modules → fields/tables/actions。
- 红包页使用 `kind=redpacket` 自定义渲染：五类开关、语音发送开关、音频来源、规则、统计卡、运行概况、群排行、处理记录；授权页与点歌线路使用通用表格/表单/特殊操作。
- 一键操作返回 `job_id`，前端 2 秒轮询。
- 能力缺失模块 `kind=disabled`，不可操作。
- 危险操作二次确认；恢复备份输入 `RESTORE`。

### 8.4 API

至少实现：

```text
POST   /api/admin/login
POST   /api/admin/logout
GET    /api/admin/profile
GET    /api/admin/catalog
GET    /api/admin/dashboard
GET    /api/admin/modules/{id}
PUT    /api/admin/modules/{id}/settings
GET    /api/admin/modules/{id}/items?table=&page=&size=&q=
POST   /api/admin/modules/{id}/items?table=
PUT    /api/admin/modules/{id}/items/{item_id}?table=
DELETE /api/admin/modules/{id}/items/{item_id}?table=
POST   /api/admin/modules/{id}/actions/{action}
GET    /api/admin/jobs/{job_id}
POST   /api/admin/jobs/{job_id}/cancel
GET    /api/admin/accounts
POST   /api/admin/accounts/refresh
GET    /api/admin/groups?self_id=&protocol_id=
GET    /api/admin/groups/{group_id}?self_id=&protocol_id=
PUT    /api/admin/groups/{group_id}?self_id=&protocol_id=
GET    /api/admin/groups/{group_id}/members?self_id=&protocol_id=
GET    /api/admin/permissions
PUT    /api/admin/permissions
POST   /api/admin/permissions/preset
GET    /api/admin/masters
PUT    /api/admin/masters
GET    /api/admin/login/mode
GET    /api/admin/login/captcha
POST   /api/admin/login/mengka
GET    /api/admin/update
POST   /api/admin/update/check
POST   /api/admin/update/install
GET    /api/admin/update/captcha
GET    /api/admin/update/admin-check
POST   /api/admin/update/login
GET    /api/admin/logs
GET    /api/admin/logs/export
GET    /api/admin/backup
POST   /api/admin/restore
POST   /api/admin/uploads
GET    /api/admin/capabilities
GET    /api/admin/settings/web
PUT    /api/admin/settings/web
POST   /api/admin/settings/token/reset
GET    /api/admin/redpacket/summary?self_id=&protocol_id=&range=
GET    /api/admin/redpacket/ranking?self_id=&protocol_id=&range=&sort=
GET    /api/admin/redpacket/records?self_id=&protocol_id=&range=&group_id=&result=&q=&page=&size=
POST   /api/admin/modules/license-cards/actions/generate
POST   /api/admin/modules/license-cards/actions/{action}
POST   /api/admin/modules/group-authorizations/actions/grant
POST   /api/admin/modules/group-authorizations/actions/revoke
GET    /api/health
```

统一响应：`{"code":0,"message":"","data":...}`；错误非 2xx + `{"code":400,"message":"..."}`。

### 8.5 安全

- 默认仅监听 `127.0.0.1`；改 `0.0.0.0` 时启动告警。
- 登录恒定时间比较；5 次失败锁定 5 分钟（按 IP）。
- 会话 Cookie：HttpOnly、SameSite=Lax、12 小时。
- 写接口校验 `X-CSRF-Token`。
- 请求体 1 MiB；上传 6 MiB；静态文件防目录穿越。
- 备份不含 token；恢复后保留当前 token。
- 所有变更写审计日志。

### 8.6 托管模式与在线更新

- 通过 `internal/managed.Load()` 判断环境：`AdoptPluginContext` 在 IPC 握手成功后记录 `managed=true`，解析 `get_plugin_context.admin_base_url`；兼容 `MENGKA_PLUGIN_ADMIN_ORIGIN`、`MENGKA_PLUGIN_ADMIN_HOST/PORT`、`MENGKA_MANAGED_V1=1`；`--console-only` 为独立模式。
- 托管登录：登录页调用 `/api/admin/login/mode`，默认展示萌卡管理员账号、密码、图形验证码；验证码与登录由后端转发至萌卡 `/api/v1/auth/captcha`、`/api/v1/auth/login`，成功后签发本地控制台会话。登录页必须提示萌卡管理端口不固定（不一定是 7891），并保留显眼的本地令牌切换入口；密码不落盘、不写日志。
- 更新源：默认 `update.github_repo=haircv/Mengka-QQ-Cloud-Assistant`；也可改为其他 `owner/repo` 或 `update.manifest_url` 自定义清单；支持 stable/prerelease。GitHub 使用 Releases API，自定义清单支持 `version/notes/assets` 映射。
- 发布：`.github/workflows/release.yml` 在推送 `v*` 标签或手动触发时构建三平台 `managed-native` 资产并上传 GitHub Release；本机可用 `scripts/publish-release.sh` / `.ps1`。
- 发布包命名必须包含 `windows-amd64`、`linux-amd64`、`linux-arm64` 之一；支持 ZIP、TAR.GZ、TGZ，单包不超过 128 MiB。
- 校验：清单资产 SHA256（可选）、压缩包内 `mengka-plugin.json` 的 `plugin_id=qq-cloud-assistant` 与版本、`SHA256SUMS` 对程序入口的哈希；任一失败禁止安装。
- 托管安装：萌卡管理员会话通过后上传到 `/api/v1/plugins/managed/import`，字段 `package`、`trusted_source_ack=true`、可选 `capability_approval`；`capability_change` 时前端二次确认后重试。导入后读取 `/api/v1/plugins/managed` 找到新实例并 POST `/{id}/action`（`{"action":"start"}`）自动启用，失败时保留结果并提示手动启用。
- 独立模式（`--console-only` 或未检测到萌卡）只允许检查和下载，不自动替换程序。
- 自动检查：`update.auto_check` 开启时启动 3 秒后检查、之后每 6 小时；只检查不自动安装。

---

## 9. 实施顺序

1. **M0 骨架**：目录、`mengka-plugin.json`、go.mod、配置加载、日志、构建脚本。先让 `go build ./...` 通过。
2. **M1 IPC 内核**：hello/ready、action 客户端、事件读取、账号键、能力探测。用假 IPC 做单测。
3. **M2 权限与路由**：六级权限、默认 Lv5、自身=主人、Worker 串行、消息缓存。
4. **M3 指令**：按类目迁移，先注册表与菜单，再 handler；v1.2 共 382 条（含授权 16、红包 29、语音点歌 1、在线更新 3）；每完成一类做一次模拟事件验证。
5. **M4 控制台**：登录 → catalog → 通用渲染器 → 59 模块（含红包、授权、点歌线路、在线更新）→ 一键操作。
6. **M5 引擎**：定时、循环、群发、验证、防撤、自身撤回、打卡、续火、秒赞秒评、多源点歌、五类红包/规则/语音、授权门禁、账本清理。
7. **M6 质量**：审计、备份恢复、并发与 panic 测试、红包/授权测试、`go vet`、`go test -race`。
8. **M7 发布**：Windows/Linux 包（1.2.18）、README、升级说明、能力矩阵、校验和。

每阶段结束必须执行：

```bash
gofmt -w .
go vet ./...
go test ./...
go build ./...
```

---

## 10. 测试要求

- **权限单测**：自身=5；主人=5；全局黑名单=0；群黑名单=0；分管=4；群主=3；管理=2；群员=1；协议查询失败=1；未设置=5；`0` 不被当成未设置；`0` 不能分配。
- **配置单测**：三层合并、Merge Patch、字段不丢失、幂等迁移。
- **注册表单测**：重复触发词启动失败、最长前缀、Pattern 的 At 参数、Fallback 不覆盖真实指令。
- **IPC 单测**：hello/ready、并发 id 配对、超时清理、迟到响应、stdout 单行 JSON。
- **消息段单测**：引用优先 `reply_reference_id`、媒体段限制、好友不支持 At、群视频单独发送。
- **引擎测试**：定时重启续跑、群发停止、防撤缓存、验证超时、打卡 0 点按日去重。
- **集成测试**：录制事件回放，验证自动功能与权限过滤。
- **控制台测试**：登录、目录、CRUD、任务轮询、权限预设、备份恢复。
- **红包单测**：五类识别与开关；`retcode`/`state` 缺失或非 0 不算成功；金额只取 `recv_object.amount`；群/人员/关键词规则；每分钟限流；同一 `listid` 去重；Android 限制。
- **语音红包测试**：内置 AI 语音、自定义 TTS、固定音频、`upload_red_packet_voice` 匹配、群语音发送、`grab_group_voice_red_packet` 领取；关闭发送或匹配失败不领取。
- **点歌测试**：Meting/网易解析、来源别名、坏线路失败换源、选歌、链接/卡片/语音回复、酷狗卡片回退。
- **口令红包测试**：先发口令，再用发送结果的 `client_seq`/`msg_random` 领取；不传 `claim_token`；缺失标识时放弃。
- **授权测试**：卡密生成、代理配额、兑换续期、撤销、手动授权、门禁开启/关闭、Lv5 豁免、未授权群拦截。
- **红包集成测试**：假 IPC 回放普通/口令/低门槛/语音四种红包，校验领取、跳过、回复、账本与耗时字段。
- **控制台测试**：红包统计/排行/记录/设置、卡密生成/状态、手动授权/撤销、代理 CRUD。
- **打包测试**：Windows x64、Linux x86_64、Linux ARM64 三个包在全新目录安装，首次启动生成 token，控制台可登录；hello 版本 1.2.18、清单 54 个 action。

---

## 11. 禁止事项

- 禁止引入联网授权、授权站、应用 ID、心跳、机器码、远程配置、支付、发卡、订单；本地卡密/分群授权/代理除外（见 3.6）。
- 禁止改动中文指令名。
- 禁止用“暂未开放”冒充实现。
- 禁止注册平台不存在的 action，禁止调用未在清单声明的 action。
- 禁止把 `protocol_id` 的 `0` 当作未设置。
- 禁止只用 `self_id` 做账号键。
- 禁止丢弃机器人自身消息（若框架推送）。
- 禁止把默认权限写成 0 或 1；默认必须是 5。
- 禁止把 `0` 作为指令可配置级别。
- 禁止自动重试发送、审批、点赞、发说说等写操作。
- 禁止在日志中输出令牌、Cookie、完整消息，以及红包 `authkey`、`claim_token`。
- 禁止对语音红包调用普通 `grab_red_packet`；语音红包只能走匹配/群语音/专用领取链路。禁止在口令红包中传 `claim_token` 或使用红包消息的 `msg_seq`。
- 禁止在 `state` 缺失或非 0 时标记领取成功；禁止把红包总额当成本次到账金额。
- 禁止把非 Android 账号用于红包领取；禁止在音频未准备或匹配失败时伪造语音领取成功。
- 禁止用酷狗音频伪造 QQ/163 音乐卡片；酷狗卡片必须回退链接或语音。
- 禁止对红包领取失败自动重试。
- 禁止修改参考源码目录。
- 禁止在没有构建通过的情况下宣称完成。
- 禁止用 `json` 消息段代替 `ark`。
- 禁止把 `sender.role` 当字符串。
- 禁止用 UID 代替 `user_id` 发送 At。

---

## 12. 完成定义

满足以下全部条件才算完成：

- [ ] `go build ./...`、`go vet ./...`、`go test ./...` 通过。
- [ ] `mengka-plugin.json` 可被萌卡 NT 3.0 导入，hello/ready 正常。
- [ ] stdout 第一行是 hello，后续每行都是合法 JSON；无日志混入。
- [ ] 动作只调用已声明 action，账号只用 `self_id + protocol_id`。
- [ ] 首次启动生成 `admin.token`，控制台可登录。
- [ ] 382 条指令注册（333 + 授权 16 + 红包 29 + 语音点歌 1 + 在线更新 3）；29 条不可用项不出现；7 个占位类目不存在。
- [ ] 所有指令默认 Lv5；自身 QQ 直接 Lv5；可配置范围 1–5。
- [ ] 黑名单、群员、管理、群主、分管按级别生效。
- [ ] 菜单/帮助按权限过滤。
- [ ] 35 个可运行类目完成注册；2 个不可用类目明确下线；语音点歌已注册。
- [ ] 控制台 59 个模块可打开、可保存、可增删改查；红包五类/规则/语音来源、点歌线路、授权页、在线更新可用。
- [ ] 账号/协议/群选择器工作正常。
- [ ] 能力矩阵与实际注册表一致。
- [ ] 定时/循环/群发重启后可恢复；群发重启标记 interrupted。
- [ ] 群发/撤回/审批等写操作失败不自动重试。
- [ ] 无任何授权网络请求；卡密、分群授权、代理完全本地。
- [ ] 红包 `authkey`/`claim_token` 不落日志/账本；语音红包需音频匹配且仅 Android；领取状态与金额校验正确。
- [ ] 普通、口令、低门槛、语音四条红包链路集成测试通过；口令红包不传 `claim_token`；同一 `listid` 不重复处理。
- [ ] 卡密生成/兑换/续期/撤销、代理配额、门禁与 Lv5 豁免测试通过；全程无授权网络请求。
- [ ] 审计日志覆盖所有配置变更。
- [ ] 备份可导出并在空目录恢复。
- [ ] Windows x64、Linux x86_64 与 Linux ARM64 包均可安装运行。

---

## 13. 交付说明

完成后输出：

1. 变更文件清单与每个文件的作用；
2. 构建、测试、打包、安装命令；
3. 已实现类目与指令数量；
4. 能力矩阵：完整支持 / 降级 / 不可用；
5. 明确未实现项及原因；
6. 控制台本地访问地址与首次登录方式；
7. 已知限制（Android-only 接口、媒体 5 分钟、平台风控、外部点歌搜索）。
