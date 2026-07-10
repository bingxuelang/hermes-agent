# 会话生命周期

> **目标读者：** 网关开发者和维护者
> **源文件：** `gateway/session.py`（约 1444 行），`gateway/run.py`（约 16800 行），`gateway/config.py`
> **最后更新：** 2026-06-16

## 概述

**会话（session）** 表示代理与一个或多个用户在消息平台上的持续对话。会话生命周期管理对话何时持久化、何时重置、如何在网关重启后存活，以及在并发操作期间消息如何排队。

会话系统主要存在于两个模块中：

- `gateway/session.py` — 数据模型（`SessionSource`、`SessionEntry`、`SessionContext`）、密钥生成（`build_session_key`）以及主存储（`SessionStore`）。
- `gateway/run.py` — 网关运行器（`GatewayRunner`），将会话接入消息处理流水线：会话过期监视、代理缓存、重启恢复以及消息排队。

---

## 1. SessionSource — 消息来源描述符

`SessionSource` 是一条不可变记录，描述*消息从何处来*。它被附加到每个传入的 `MessageEvent` 上，用于路由、隔离和上下文注入。

### 字段

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `platform` | `Platform` | *(必填)* | 标识消息平台的枚举（telegram、discord、slack、signal、whatsapp、matrix、local 等）。 |
| `chat_id` | `str` | *(必填)* | 平台级别的聊天/群组/频道标识符。通过适配器的 `chat_id_key` 转换进行路由。 |
| `chat_name` | `Optional[str]` | `None` | 聊天或群组的人类可读名称。 |
| `chat_type` | `str` | `"dm"` | 取值为 `"dm"`、`"group"`、`"channel"`、`"thread"` 之一。控制会话密钥生成与隔离。 |
| `user_id` | `Optional[str]` | `None` | 平台特定的用户标识符。用于授权和按用户隔离会话。 |
| `user_name` | `Optional[str]` | `None` | 消息作者的显示名称。注入到系统提示词中。 |
| `thread_id` | `Optional[str]` | `None` | 论坛话题 / Discord 子线程 / Slack 子线程标识符。用于区分线程化对话。 |
| `chat_topic` | `Optional[str]` | `None` | 频道主题或描述（Discord 频道主题、Slack 频道用途）。 |
| `user_id_alt` | `Optional[str]` | `None` | 平台特定的稳定替代 ID（Signal UUID、Feishu union_id）。当 `user_id` 为临时值时使用。 |
| `chat_id_alt` | `Optional[str]` | `None` | Signal 群组内部 ID —— 将 Signal 群组 V2 标识符映射为其规范形式。 |
| `is_bot` | `bool` | `False` | 当消息作者是机器人或 webhook 时为 True（Discord 机器人）。 |
| `guild_id` | `Optional[str]` | `None` | Discord 公会 / Slack 工作区 / Matrix 服务器作用域标识符。 |
| `parent_chat_id` | `Optional[str]` | `None` | 当 `chat_id` 指向某个线程时的父频道。 |
| `message_id` | `Optional[str]` | `None` | 触发消息的 ID。用于置顶/回复/回应操作以及 Discord ID 注入。 |
| `role_authorized` | `bool` | `False` | 当适配器通过平台角色（而非单个用户 ID）授予访问权限时为 True。 |

### 关键方法

- **`description`**（属性：`str`）— 人类可读的摘要，例如 `"DM with Alice"`、`"group: My Group, thread: 12345"`。
- **`to_dict()` / `from_dict()`** — 序列化往返，用于持久化到 `sessions.json`。

---

## 2. SessionEntry — 活动会话记录

`SessionEntry` 是每个会话的元数据记录，存储在内存中并持久化到 `{sessions_dir}/sessions.json`。每条记录将一个 `session_key` 映射到其当前的 `session_id`。

### 字段

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `session_key` | `str` | *(必填)* | 标识对话通道的确定性密钥（见第 4 节）。 |
| `session_id` | `str` | *(必填)* | 此特定对话实例的唯一标识符。格式：`YYYYMMDD_HHMMSS_<8hex>`。 |
| `created_at` | `datetime` | *(必填)* | 此会话实例的创建时间。 |
| `updated_at` | `datetime` | *(必填)* | 最后活动时间戳。用于空闲超时和过期检查。 |
| `origin` | `Optional[SessionSource]` | `None` | 创建此会话的来源，用于投递路由。 |
| `display_name` | `Optional[str]` | `None` | 聊天显示名称（取自 `SessionSource.chat_name`）。 |
| `platform` | `Optional[Platform]` | `None` | 平台枚举，持久化以支持跨重启的过期策略查找。 |
| `chat_type` | `str` | `"dm"` | 聊天类型，同样持久化以支持策略查找。 |
| `input_tokens` | `int` | `0` | 累计 LLM 输入（提示词）token 数。 |
| `output_tokens` | `int` | `0` | 累计 LLM 输出（补全）token 数。 |
| `cache_read_tokens` | `int` | `0` | 累计提示词缓存读取 token 数。 |
| `cache_write_tokens` | `int` | `0` | 累计提示词缓存写入 token 数。 |
| `total_tokens` | `int` | `0` | 所有轮次的 token 总数。 |
| `estimated_cost_usd` | `float` | `0.0` | 预估累计美元成本。 |
| `cost_status` | `str` | `"unknown"` | 成本追踪状态标签。 |
| `last_prompt_tokens` | `int` | `0` | 上一次 API 上报的提示词 token 数。用于精确的压缩预检查。 |

### 布尔标志（状态机）

SessionEntry 有若干布尔标志，构成一个简单状态机，管理会话在下一次访问时的行为。

| 标志 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `was_auto_reset` | `bool` | `False` | 当会话因策略过期（空闲/每日）而被自动重置时设置。仅消费一次以注入上下文通知。 |
| `auto_reset_reason` | `Optional[str]` | `None` | `"idle"` 或 `"daily"` —— 上一个会话被自动重置的原因。 |
| `reset_had_activity` | `bool` | `False` | 过期的会话是否曾有任何消息（`total_tokens > 0`）。 |
| `is_fresh_reset` | `bool` | `False` | 由显式 `/new` 或 `/reset` 设置。在首条消息时触发话题/频道技能重新注入。与 `was_auto_reset` 区分，以避免产生误导性的"会话已过期"通知。 |
| `expiry_finalized` | `bool` | `False` | 由后台过期监视器在调用 `on_session_finalize` 钩子、清理工具资源并驱逐缓存的代理后设置。防止跨重启的重复终结。 |
| `suspended` | `bool` | `False` | 强制硬擦除信号。由 `/stop` 或卡死循环升级（连续 3 次以上重启失败）设置。在下一次 `get_or_create_session()` 时，无论 `resume_pending` 如何，都强制生成新的 `session_id`。 |
| `resume_pending` | `bool` | `False` | 软恢复标记。由 `suspend_recently_active()`（崩溃恢复）或排空超时设置。在下一次访问时，保留现有 `session_id` —— 用户在同一个对话记录上继续。在下一次成功轮次完成后清除。 |
| `resume_reason` | `Optional[str]` | `None` | 标记恢复的原因：`"restart_timeout"`、`"shutdown_timeout"`、`"restart_interrupted"`。 |
| `last_resume_marked_at` | `Optional[datetime]` | `None` | 上一次标记 resume-pending 的时间戳。 |

### 状态转换逻辑（get_or_create_session）

```
                    ┌──────────┐
                    │  Incoming │
                    │  Message  │
                    └────┬─────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  session_key exists  │──── No ──► Create fresh SessionEntry
              │  AND !force_new      │
              └──────────┬───────────┘
                         │ Yes
                         ▼
              ┌──────────────────────┐
              │  entry.suspended?    │──── Yes ──► Auto-reset: new session_id
              └──────────┬───────────┘           (reason="suspended")
                         │ No
                         ▼
              ┌──────────────────────┐
              │ entry.resume_pending?│──── Yes ──► Return existing entry
              └──────────┬───────────┘           (preserve session_id)
                         │ No                     Clear flag on next successful turn
                         ▼
              ┌──────────────────────┐
              │   Policy says reset? │──── Yes ──► Auto-reset: new session_id
              └──────────┬───────────┘           (reason="idle"/"daily")
                         │ No
                         ▼
              ┌──────────────────────┐
              │  Return existing     │
              │  entry, bump         │
              │  updated_at          │
              └──────────────────────┘
```

**`get_or_create_session()` 中的优先级顺序：**
1. `suspended=True` → 总是强制重置（硬擦除）
2. `resume_pending=True` → 保留 session_id（软恢复）
3. 策略过期（空闲/每日）→ 自动重置
4. 无触发条件 → 返回现有记录（更新 `updated_at`）

---

## 3. SessionStore — 存储与操作

`SessionStore` 是主存储层。它维护一个内存字典（`_entries`），持久化到 `sessions.json`，并以 SQLite（`SessionDB`）作为会话元数据和消息对话记录的规范存储。

### 构造函数

```python
SessionStore(sessions_dir: Path, config: GatewayConfig, has_active_processes_fn=None)
```

- `sessions_dir` — `sessions.json` 所在的目录。
- `config` — `GatewayConfig` 实例，用于重置策略查找。
- `has_active_processes_fn` — 可选的回调，以 `session_key` 为键检查是否有正在运行的后台进程。具有活动进程的会话永远不会被过期或清理。

### 操作（方法）

| 方法 | 说明 |
|---|---|
| `get_or_create_session(source, force_new=False)` | 核心入口。返回现有记录或创建新的 `SessionEntry`。评估 `suspended`、`resume_pending` 和重置策略。创建/结束 SQLite 记录。 |
| `update_session(session_key, last_prompt_tokens=None)` | 一次交互后的轻量级元数据更新。更新 `updated_at`，可选地记录 `last_prompt_tokens`。 |
| `reset_session(session_key, display_name=None)` | 显式重置（来自 `/new` 或 `/reset`）。创建新的 `session_id`，设置 `is_fresh_reset=True`。结束旧的 SQLite 会话，创建新的。 |
| `switch_session(session_key, target_session_id)` | 切换到另一个已存在的会话 ID（来自 `/resume`）。结束当前 SQLite 会话，重新打开目标会话。 |
| `suspend_session(session_key)` | 将会话标记为 `suspended=True`（来自 `/stop`）。在下一次访问时强制自动重置。 |
| `mark_resume_pending(session_key, reason)` | 将会话标记为 `resume_pending=True`（来自排空超时）。在下一次访问时保留 session_id。不会覆盖 `suspended=True`。 |
| `clear_resume_pending(session_key)` | 在一次成功的恢复轮次后清除 `resume_pending`。由网关在 `run_conversation()` 返回后调用。 |
| `suspend_recently_active(max_age_seconds=120)` | 崩溃恢复：将最近活动的会话标记为 `resume_pending=True`。跳过已处于 pending 和已挂起的记录。在非正常关闭后的启动时调用。 |
| `prune_old_entries(max_age_days)` | 丢弃早于 `max_age_days` 的记录（基于 `updated_at`）。跳过 `suspended` 记录以及具有活动进程的会话。 |
| `list_sessions(active_minutes=None)` | 返回所有会话，可选按近期活动过滤。按 `updated_at` 降序排列。 |
| `lookup_by_session_id(session_id)` | 查找某个已持久化会话 ID 对应的活动 `SessionEntry`。 |
| `has_any_sessions()` | 检查是否曾经创建过任何会话（使用 SQLite 历史，而非仅内存字典）。 |
| `append_to_transcript(session_id, message, skip_db=False)` | 向 SQLite 对话记录追加一条消息。`skip_db=True` 可在代理已经持久化时避免重复写入。 |
| `rewrite_transcript(session_id, messages)` | 完全替换会话对话记录（由 `/retry`、`/undo`、`/compress` 使用）。 |
| `load_transcript(session_id)` | 从会话的 SQLite 对话记录加载所有消息。 |
| `rewind_session(session_id, n=1)` | 通过软删除回退 `n` 个用户轮次（保留审计轨迹）。返回 `{rewound_count, turns_undone, target_text}`。 |

### 内部辅助方法

- `_ensure_loaded()` / `_ensure_loaded_locked()` — 将 `sessions.json` 加载到 `_entries` 字典。
- `_save()` — 通过临时文件 + `atomic_replace` 原子写入 `sessions.json`。
- `_generate_session_key(source)` — 委托给 `build_session_key()` 并传入配置参数。
- `_is_session_expired(entry)` — 仅凭记录进行的策略检查（无需 source）。由后台过期监视器使用。
- `_should_reset(entry, source)` — 策略检查，返回 `"idle"`、`"daily"` 或 `None`。

### 存储布局

```
{sessions_dir}/
  sessions.json          # In-memory _entries dict, persisted as JSON
                           Maps session_key → SessionEntry (metadata only)
  {session_id}.jsonl     # (Legacy, removed in spec 002)
```

规范对话记录存储是通过 `SessionDB`（来自 `hermes_state`）的 SQLite。`sessions.json` 文件持久化 `session_key → session_id` 的映射和记录元数据（标志、时间戳、token 计数）。如果 SQLite 不可用，存储会回退到 JSONL，但这是一种降级路径。

---

## 4. SessionKey 生成规则

会话密钥是标识对话通道的确定性字符串。它们由 `build_session_key(source, group_sessions_per_user, thread_sessions_per_user)` 生成。

### 密钥格式

```
agent:main:{platform}:{chat_type}[:{chat_id}][:{thread_id}][:{participant_id}]
```

### 私聊规则

| 场景 | 密钥 |
|---|---|
| 带 chat_id 的私聊 | `agent:main:telegram:dm:12345` |
| 带 chat_id + 线程的私聊 | `agent:main:telegram:dm:12345:thread_678` |
| 不带 chat_id、带 participant_id 的私聊 | `agent:main:signal:dm:user_abc` |
| 不带 chat_id 或 participant_id 的私聊 | `agent:main:telegram:dm` |
| WhatsApp 私聊（已规范化） | `agent:main:whatsapp:dm:{canonical_number}` |

- 私聊在 chat_id 存在时总是包含 chat_id，从而隔离每个私聊会话。
- `thread_id` 进一步区分同一私聊中的线程化私聊。
- 在没有 chat_id 时，回退到 `user_id_alt` 或 `user_id` 作为 participant_id。
- 在没有任何标识符时，该平台上的所有私聊合并为一个共享会话。

### 群组/频道规则

| 场景 | 密钥 |
|---|---|
| 群组聊天 | `agent:main:telegram:group:-10012345` |
| 群组聊天，按用户隔离 | `agent:main:telegram:group:-10012345:user_abc` |
| 群组中的线程，共享 | `agent:main:discord:group:12345:thread_678` |
| 群组中的线程，按用户 | `agent:main:discord:group:12345:thread_678:user_abc` |
| 频道 | `agent:main:slack:channel:C12345` |
| WhatsApp 群组（已规范化） | `agent:main:whatsapp:group:{canonical_id}:{participant}` |

- `chat_id` 标识父群组/频道。
- `thread_id` 区分该父级中的线程。
- **按用户隔离**（追加 `participant_id`）由以下配置控制：
  - `group_sessions_per_user`（默认：`True`）— 群组/频道会话按用户隔离。
  - `thread_sessions_per_user`（默认：`False`）— 线程默认为**共享**
    （Telegram 论坛话题、Discord 子线程、Slack 子线程都按线程共享一个会话）。
- `participant_id` = `user_id_alt` 或 `user_id`（按此优先级）。
- WhatsApp 标识符经过规范化处理，以应对 JID/LID 别名翻转。

### 特殊情况：WhatsApp

WhatsApp 电话号码经过 `canonical_whatsapp_identifier()` 处理，该函数去除 `@s.whatsapp.net` 后缀并规范化为 E.164 格式。这可以防止当桥接返回同一电话号码的不同别名形式时出现会话碎片化。

---

## 5. 多用户隔离策略

多用户隔离决定同一聊天中的多个用户是共享一个对话，还是各自拥有独立的私有会话。

### 决策逻辑（`is_shared_multi_user_session`）

```python
def is_shared_multi_user_session(source, *, group_sessions_per_user, thread_sessions_per_user):
    if source.chat_type == "dm":
        return False  # DMs are always private
    if source.thread_id:
        return not thread_sessions_per_user  # Threads: shared unless per-user
    return not group_sessions_per_user       # Groups: isolated unless shared
```

### 汇总

| 聊天类型 | 默认行为 | 配置控制 |
|---|---|---|
| 私聊 | 私有（从不共享） | 不适用 |
| 群组/频道 | 按用户隔离 | `group_sessions_per_user`（默认：True） |
| 线程（论坛、Discord） | 共享（所有参与者看到相同上下文） | `thread_sessions_per_user`（默认：False） |

### 对系统提示词的影响

当 `shared_multi_user_session=True` 时，系统提示词会省略固定的用户名，而是声明：*"Multi-user {thread|session} — messages are prefixed with [sender name]. Multiple users may participate."*（多用户 {线程|会话} —— 消息以 [发送者名称] 为前缀。多个用户可能参与。）网关在运行时将每个用户消息加上独立的发送者名称前缀，从而保留提示词缓存（系统提示词不会逐轮变化）。

---

## 6. 重置策略

重置策略控制会话何时自动丢失上下文（获得新的 `session_id`）。

### 策略模式（`SessionResetPolicy`）

| 模式 | 行为 | 默认配置 |
|---|---|---|
| `"none"` | 从不自动重置。上下文仅由压缩管理。 | — |
| `"idle"` | 从 `updated_at` 起 N 分钟不活动后重置。 | `idle_minutes: 1440`（24 小时） |
| `"daily"` | 每天在特定时刻（本地时间）重置。 | `at_hour: 4`（凌晨 4 点） |
| `"both"` | 取最先触发者 —— 每日边界或空闲超时。 | **（默认）** |

### 策略评估

```python
# Idle check
idle_deadline = entry.updated_at + timedelta(minutes=policy.idle_minutes)
if now > idle_deadline: return "idle"

# Daily check
today_reset = now.replace(hour=policy.at_hour, minute=0, second=0, microsecond=0)
if now.hour < policy.at_hour:
    today_reset -= timedelta(days=1)  # Reset hasn't happened yet today
if entry.updated_at < today_reset: return "daily"
```

### 按平台/按类型策略

重置策略可通过 `config.get_reset_policy()` 按平台和会话类型进行配置。这允许不同平台有不同的过期规则（例如，Telegram 私聊在空闲 24 小时后重置，但 Slack 群组无限期保留）。

### 排除项

具有活动后台进程的会话**永远不会**被过期或重置。`has_active_processes_fn` 回调会在评估策略时检查是否有正在运行的进程。

### 重置影响

当触发重置时：

1. 旧会话在 SQLite 中结束（原因为 `"session_reset"`）。
2. 生成新的 `session_id`（`YYYYMMDD_HHMMSS_<8hex>`）。
3. 创建新的 `SessionEntry`，设置 `was_auto_reset=True` 和重置原因。
4. 如果旧会话曾有任何轮次（`total_tokens > 0`），则设置 `reset_had_activity`。
5. 旧 AIAgent 缓存条目在下一轮过期监视器扫描时被驱逐。
6. 在重置后的首条消息时，注入一条上下文通知："Session expired due to inactivity / daily reset."（会话因不活动/每日重置而过期。）

---

## 7. 重启恢复流程

重启恢复系统确保进行中的会话在网关重启、崩溃和排空超时后得到保留。它是 issue #7536 的解决方案。

### 启动恢复序列

```
Gateway starts
       │
       ▼
┌───────────────────────────────┐
│ Check for .clean_shutdown     │── Exists? ──► Skip suspension (clean exit)
│ marker                        │
└───────────────────────────────┘
       │ Missing
       ▼
┌───────────────────────────────┐
│ session_store                 │── Marks sessions updated within
│ .suspend_recently_active()    │   last 120 seconds as resume_pending
└───────────────────────────────┘
       │
       ▼
┌───────────────────────────────┐
│ _suspend_stuck_loop_sessions()│── Suspends sessions that have been
│                               │   active across 3+ restarts
└───────────────────────────────┘
       │
       ▼
┌───────────────────────────────┐
│ Queue inbound messages while  │
│ startup restore runs          │
│ (_startup_restore_in_progress)│
└───────────────────────────────┘
       │
       ▼
┌───────────────────────────────┐
│ For each adapter, find        │
│ resume_pending sessions →     │
│ synthesize MessageEvent and   │
│ run _handle_message to let    │
│ the agent auto-continue       │
└───────────────────────────────┘
```

### suspend_recently_active(max_age_seconds=120)

在网关启动时，当不存在 `.clean_shutdown` 标记（表明发生了崩溃或意外退出）时调用。对于在过去 120 秒内更新过的每个会话：

- 设置 `resume_pending=True`、`resume_reason="restart_interrupted"`、`last_resume_marked_at=now`。
- 跳过已为 `resume_pending=True` 的记录（不重复标记）。
- 跳过显式为 `suspended=True` 的记录（硬擦除应保持）。

### 卡死循环检测（`_suspend_stuck_loop_sessions`）

通过一个 JSON 文件（`{HERMES_HOME}/restart_counts.json`）统计连续重启次数。如果某个会话在连续 3 次以上重启中均处于活动状态，它会被自动挂起，以便用户获得一个干净的开始。

### 排空超时标记

在优雅关闭/重启时，排空系统会对任何在排空超时触发时正处于轮次中间的会话调用 `mark_resume_pending()`。原因包括：

- `"restart_timeout"` — 在重启排空期间被杀死
- `"shutdown_timeout"` — 在关闭排空期间被杀死
- `"restart_interrupted"` — 崩溃恢复（来自 `suspend_recently_active`）

这三种原因都属于 `_AUTO_RESUME_REASONS`，有资格进行启动时自动恢复。

### 下次访问时的自动恢复

当 `get_or_create_session()` 遇到 `resume_pending=True` 时：

1. 它返回现有记录，**不**创建新的 `session_id`。
2. 现有对话记录完整加载。
3. 该标记不会在此处清除 —— 它会一直保留，直到下一次成功轮次完成（`clear_resume_pending()` 由网关在 `run_conversation()` 返回真实响应后调用）。
4. 如果恢复的轮次再次被中断，`resume_pending` 标志保持设置，下一次重启会重试。卡死循环计数器处理终局升级（3 次重试 → 挂起）。

### 干净关闭标记（`.clean_shutdown`）

在优雅关闭结束时写入。在下一次启动时：

- 如果存在：完全跳过 `suspend_recently_active()`。活动代理已经被排空，因此没有会话卡住。
- 然后删除该标记。

这可以防止在 `hermes update`、`hermes gateway restart` 或 `/restart` 之后出现不必要的自动重置。

---

## 8. 消息排队流程

消息排队系统处理两种场景：

1. **中断后续消息** — 当用户在代理处理期间发送多条消息时，后续消息作为单槽位的待处理消息排队。
2. **`/queue` FIFO** — 显式的 `/queue` 命令，每条都必须按顺序产生自己的完整代理轮次，不进行合并。

### 数据结构

```
adapter._pending_messages: Dict[session_key, MessageEvent]
    └── Single "next-up" slot per session. Overwritten on repeat sends
        (burst collapse). Shared with photo-burst follow-ups.

self._queued_events: Dict[session_key, List[MessageEvent]]
    └── Overflow buffer. Each /queue invocation appends here when the
        slot is occupied. Promoted one-at-a-time after each drain.
```

### 入队（`_enqueue_fifo`）

```
_enqueue_fifo(session_key, event, adapter)
       │
       ▼
┌───────────────────────────────────────┐
│ Is slot free?                         │
│ (session_key NOT in _pending_messages)│── Yes ──► Place event in slot
└───────────────────────────────────────┘
       │ No
       ▼
Append to _queued_events[session_key] (overflow tail)
```

### 出队 / 提升（`_promote_queued_event`）

在槽位被消费后于排空点调用。如果存在溢出项：

- 当 `pending_event is None`（槽位为空）时，返回溢出队列头部作为新事件。
- 当 `pending_event` 存在时，将溢出队列头部暂存到槽位，供下一次递归使用。
- 如果没有可用的适配器，推回到 `_queued_events`（不要静默丢弃）。

### 队列深度

`_queue_depth(session_key, adapter)` 返回 `len(overflow) + (1 if slot occupied else 0)`。

### 清理

会话的排队事件在 `/new` 和 `/reset` 时被清理（通过 `_handle_reset_command`）。

### FIFO 不变式

每次 `/queue` 调用都按 FIFO 顺序精确产生一个完整代理轮次，不进行合并。单槽位 `_pending_messages` + 溢出 `_queued_events` 的设计确保在活动轮次期间重复发送不会导致乱序处理。

---

## 9. 会话上下文注入

`SessionContext` 由 `SessionSource` 和 `GatewayConfig` 构建，并注入到代理的系统提示词中。它告知代理：

- 当前消息来自何处
- 连接了哪些平台
- 可以将计划任务输出投递到哪里
- 这是否是一个共享的多用户会话

### 构造（`build_session_context`）

```python
def build_session_context(source, config, session_entry=None) -> SessionContext
```

1. 从配置收集已连接的平台。
2. 收集每个平台的主频道。
3. 通过 `is_shared_multi_user_session()` 判定 `shared_multi_user_session`。
4. 如果提供了 `session_entry`，附加会话元数据（密钥、ID、时间戳）。

### PII 脱敏（`build_session_context_prompt`）

动态系统提示词部分（`## Current Session Context`）可选择在发送给 LLM 之前对个人身份信息（PII）进行脱敏：

- 用户 ID → `user_<12hex>`（SHA-256 前缀）
- 聊天 ID → `<platform>:<12hex>` 或仅 `<12hex>`
- 不参与脱敏的平台：Discord（需要原始 ID 以支持 `@mentions`），以及任何未标记为 `pii_safe` 的插件注册平台。

脱敏仅适用于系统提示词文本。路由、会话密钥和适配器操作始终使用原始值。

---

## 10. 后台过期监视器

`_session_expiry_watcher` 任务在网关事件循环中每 300 秒（5 分钟）运行一次。

### 职责

1. **终结过期会话** — 对于每个 `_is_session_expired()` 返回 True 且 `expiry_finalized` 为 False 的记录：
   - 调用 `on_session_finalize` 插件钩子（清理、通知）。
   - 清理缓存的 AIAgent 资源（关闭工具资源、关闭内存提供程序）。
   - 驱逐缓存的代理条目。
   - 清除按会话的覆盖（`_session_model_overrides`、推理覆盖等）。
   - 标记 `expiry_finalized=True` 并持久化。

2. **清扫空闲缓存代理** — 调用 `_sweep_idle_cached_agents()` 驱逐空闲时间超过 `_AGENT_CACHE_IDLE_TTL_SECS`（3600 秒 / 1 小时）的代理，与会话重置策略无关。这可防止具有长生命周期会话的网关出现无界内存增长。

3. **清理过期记录** — 每小时基于 `config.session_store_max_age_days` 调用 `session_store.prune_old_entries()`。防止 `sessions.json` 无界增长。

### 故障处理

- 按会话的重试计数：每次失败的终结最多连续重试 3 次。
- 3 次失败后，该记录被强制标记为 `expiry_finalized=True`，以防止无限重试循环。

---

## 11. 代理缓存

网关维护一个以 `session_key` 为键的 `AIAgent` 实例 LRU 缓存，以在轮次间保留提示词缓存。

### 缓存属性

- **最大容量：** 128 个条目（`_AGENT_CACHE_MAX_SIZE`）。
- **驱逐策略：** 最近最少使用（通过 `OrderedDict` 实现 LRU）。
- **空闲 TTL：** 3600 秒（1 小时）— 由 `_session_expiry_watcher` 强制执行。
- **锁：** `_agent_cache_lock`（线程锁），用于线程安全。

### 缓存生命周期

```
Message arrives
    │
    ▼
get_or_create_session()  →  session_key obtained
    │
    ▼
Lookup _agent_cache[session_key]
    │
    ├── Hit → move_to_end(), reuse AIAgent (preserves prompt cache)
    │
    └── Miss → create new AIAgent, store in cache
                (if at capacity, popitem(last=False) evicts LRU entry)
    │
    ▼
run_conversation()  →  agent processes message
    │
    ▼
Session expiry watcher evicts agent when session finalizes
```

### 清理流程

当会话过期时：
1. `_cleanup_agent_resources(agent)` — 关闭内存提供程序，关闭工具资源。
2. `_evict_cached_agent(key)` — 从 `_agent_cache` 中移除，以便代理可被 GC 回收。

---

## 附录：关键配置

| 配置键 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `group_sessions_per_user` | `bool` | `true` | 按用户隔离群组/频道会话 |
| `thread_sessions_per_user` | `bool` | `false` | 按用户隔离线程会话 |
| `session_store_max_age_days` | `int` | `0` | 清理超过 N 天的会话（0=禁用） |
| `agent.gateway_auto_continue_freshness` | `int` | `3600` | 恢复新鲜度窗口的秒数 |
| `agent.gateway_timeout` | `int` | `1800` | 代理轮次超时（默认 30 分钟） |

### 重置策略（按平台/类型，位于 config.yaml）

```yaml
session_reset:
  mode: both            # none | idle | daily | both
  at_hour: 4            # daily reset hour (local time)
  idle_minutes: 1440    # idle timeout (24h)
  notify: true          # notify user on auto-reset
```

平台特定的覆盖可在 `platforms.<name>.session_reset` 下设置。
