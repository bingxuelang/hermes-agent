# Relay ↔ Connector 接口合同（v1，实验性）

> **状态：** 实验性（EXPERIMENTAL）。在至少两个真实的 Class-1 平台（Discord
> + Telegram）对其完成验证之前，本合同可能在无弃用周期的情况下发生变更。
> 实验阶段的演进为**仅追加（additive-only）**，由 `contract_version` 控制。
> 任何破坏性变更会同步更新两个代码仓库。

本文档是 **Hermes 网关**（Python，`gateway/relay/`）与 **connector**（Node/TypeScript，
`NousResearch/gateway-gateway`）之间的正式接口。connector 实现者的第一个动作就是
阅读本文件。

网关运行一个通用的 `RelayAdapter`，它向 connector 发起**出站**连接，在握手时接收一个
`CapabilityDescriptor`，随后在一个按会话（per-turn）的双向 WebSocket 上交换归一化的
`MessageEvent`（入站）与动作（出站）。网关永远不会知道是哪一个具体平台在前端承载它；
所有平台特定的 socket/身份逻辑都由 connector 持有。

---

## 1. 握手（Handshake）

1. 网关打开传输层（`connect`）。
2. 网关调用 `handshake()`；connector 返回一个 `CapabilityDescriptor`
   （见第 2 节），描述此 adapter 实例所承载的平台。
3. 网关依据 descriptor 配置 adapter（字符上限、长度单位、
   草稿/编辑/线程/markdown 能力），并注册入站处理器。
4. 随后 connector 流式推送入站事件，并接受出站动作。

`contract_version`（当前为 `1`）在 descriptor 中承载。网关会忽略未知的 descriptor
字段（前向兼容），并从默认值填充缺失的可选字段。

---

## 2. CapabilityDescriptor（握手载荷）

JSON 对象。事实来源（source of truth）：`gateway/relay/descriptor.py`。

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `contract_version` | int | 是 | 合同版本（同一版本内仅追加）。 |
| `platform` | string | 是 | 平台名称（例如 `"discord"`、`"telegram"`）。 |
| `label` | string | 是 | 人类可读的标签。 |
| `max_message_length` | int | 是 | 字符上限；网关以 `MAX_MESSAGE_LENGTH` 暴露。0 → 按 4096 处理。 |
| `supports_draft_streaming` | bool | 是 | 是否原生支持草稿流式预览。 |
| `supports_edit` | bool | 是 | 是否可基于编辑进行流式推送；若为 false，消费方降级为每个分段一条消息。 |
| `supports_threads` | bool | 是 | 是否具备 `create_handoff_thread` 能力。 |
| `markdown_dialect` | string | 是 | `"plain"`、`"markdown_v2"`、`"discord"`、……（驱动 `supports_code_blocks`）。 |
| `len_unit` | string | 是 | `"chars"`（内建 len）或 `"utf16"`（Telegram 的 UTF-16 码元）。 |
| `emoji` | string | 否 | 展示用 emoji（默认 🔌）。 |
| `platform_hint` | string | 否 | 系统提示中的平台提示。 |
| `pii_safe` | bool | 否 | 在会话描述中脱敏 PII。 |

大多数字段是网关既有 `PlatformEntry` 的一个投影；运行时字段（`len_unit`、
`supports_*`、`markdown_dialect`）来自当前活跃平台 adapter 的能力方法。

---

## 3. 入站：`MessageEvent` 信封

connector 将每个平台线路事件归一化为一个 `MessageEvent`
（`gateway/platforms/base.py`）并交付给网关。**入站通过网关的出站 `/relay`
WebSocket 交付**（见下方传输说明）——connector 沿网关已拨号的 socket 向下推送一个
`inbound` 帧。网关通过 `build_session_key()` 从内嵌的 `SessionSource` 生成会话键——
因此填充正确的判别字段是 connector 在正确性方面最高的单一职责。

### 入站传输（WS 回传通道，非 HTTP）

网关向 connector 的 `/relay` WebSocket 发起**出站**连接，用于握手 + 出站动作
（§4）+ 自身的 `/stop` 出口（§5）。入站沿**同一 socket** 反向传输：connector 沿网关
的出站 WS 向下推送 `inbound` 帧（以及 §5 的 `interrupt_inbound`）。**网关侧不存在入站
HTTP 端点**——网关无需（且在被托管时也无法）暴露任何入站端口；一切都沿它主动发起的
连接流动。

**多实例路由。** 拥有某平台 socket（因而产生入站事件）的 connector 实例通常**不是**网关
将其出站 WS 拨入的那个实例。产生事件的实例因此将事件发布到 connector 内部的 **relay
总线**（Redis pub/sub；`src/core/relayBus.ts` 中的 `RelayBus`），以租户为键。每个
connector 实例都订阅，并把每条消息路由给该租户在**本地**的会话
（`RelayServer.routeBusMessage`）；实际持有网关 socket 的那一个实例负责交付，
而对该租户没有本地会话的实例则不进行任何操作。因此跨实例交付是集群内的一次 Redis 跳转，
而非一次公网 HTTP 调用。

帧（connector → gateway，沿 WS 传输）：

- `{"type":"inbound", "event": <MessageEvent>, "bufferId"?}`
- `{"type":"interrupt_inbound", "session_key", "chat_id"}`（§5）
- `{"type":"passthrough_forward", "forward": <PassthroughForward>, "bufferId"?}`（§5.1）

`PassthroughForward` 是被转发的 passthrough 平面请求（Class-2/3 webhook——Discord
interactions、Twilio）的线路形式：`{platform, botId, method,
path, headers: [[k,v],…], bodyB64}`。body 经 base64 编码，以便任意字节能经受
newline-delimited-JSON 传输；网关 base64 解码回 connector 转发时的精确字节
（connector 已在边缘验证了 provider 签名，并剥离了任何共享身份凭证——
§6——因此网关重新处理的是一个已脱敏、无 token 的 body，并通过无 token 的
`follow_up` 路径对其执行动作）。见 §3.1。

**信任。** WS 升级使用网关的 per-gateway secret（§6.1）完成认证，因此该通道端到端可信
——入站帧不再单独进行 HMAC 签名（已认证的 socket 已经覆盖了旧 HTTP 路径所需的逐次交付
来源证明）。relay 总线跳转位于 connector 信任域内部（与 lease/buffer/capability 存储
相同）。

> 本合同的早期草案通过一个签名 **HTTP POST** 将入站交付到 `gatewayEndpoint`
> （`HttpGatewayDelivery` + 网关侧的 `inbound_receiver`），使用每租户交付密钥进行
> HMAC 签名。这要求每个网关暴露一个可达的入站 URL——对托管型网关而言不可能，因为它们
> 没有公网 IP。上文的 WS 回传通道取代了它；每租户交付密钥在 provision 时仍保留以保持
> 前向兼容，但已不再用于入站。**passthrough 平面**（Class-2/3 webhook，如 Discord
> interactions / Twilio）历史上仍使用 `gatewayEndpoint` 进行 ACK 之后的转发；
> Phase 5 §5.1 将该转发也迁移到 WS（即上文的 `passthrough_forward` 帧），因此托管型
> 网关无需任何公网入站面，且 `gatewayEndpoint` 在切换落地后即被废弃。

### 3.1 Passthrough 平面转发（§5.1）

passthrough 平面在 connector 边缘（EDGE）应答 provider 对延迟敏感的 ACK
（例如 Discord 在约 3s 内的延迟交互响应），随后对真实请求进行**即发即弃（fire-and-forget）**
转发至网关。该转发无需响应回传（provider 已被满足），因此它通过 `passthrough_forward`
帧沿与 `inbound` 相同的出站 WS 传输，而非通过 HTTP POST。网关通过其正常的 agent 路径
处理解码后的请求（一个 Discord interaction 被解码为 `MessageEvent` 并像消息一样处理；
回复沿出站 / `follow_up` 路径出口）。当转发被缓冲时（Phase 5 §5.3 的 buffered-only
切换），`bufferId` 存在，且网关在持久化移交后对其发送 ack。



### SessionSource 字段（线路接口面）

事实来源：`gateway/session.py` 中的 `SessionSource.to_dict()`。这些是网关在线路上接受的
全部键。`platform`、`chat_id`、`chat_type`、`user_id`、`user_name`、`thread_id`、
`chat_name` 和 `chat_topic` 始终存在（可能为 `null`）；其余仅在设置时包含。

| 字段 | 类型 | 始终发送 | 含义 |
| --- | --- | --- | --- |
| `platform` | string | 是 | 平台名称（与 descriptor 的 `platform` 一致）。 |
| `chat_id` | string | 是 | 主会话 id（channel/chat）。会话键判别字段。 |
| `chat_type` | string | 是 | `dm` / `group` / `channel` / `thread` / `forum`。 |
| `chat_name` | string\|null | 是 | 人类可读的会话名称。 |
| `user_id` | string\|null | 是 | 消息作者 id。会话键判别字段。 |
| `user_name` | string\|null | 是 | 作者显示名。 |
| `thread_id` | string\|null | 是 | 处于线程中时的 thread/forum-topic id。会话键判别字段。 |
| `chat_topic` | string\|null | 是 | 频道主题/描述（Discord、Slack）。 |
| `user_id_alt` | string | 否 | 平台特定的稳定备用 id（Signal UUID、Feishu union_id）。 |
| `chat_id_alt` | string | 否 | 备用会话 id（例如 Signal 群组内部 id）。 |
| `scope_id` | string | 否 | 平台无关的 **scope** 判别字段：Discord guild / Slack workspace / Matrix server。**Discord/Slack 的 scope 隔离所必需。** 会话键判别字段。（自 D-Q2.5 线路迁移起的规范名称。） |
| `guild_id` | string | 否 | **遗留别名，connector 不再读取。** 自 D-Q2.5c 起，connector 仅读写 `scope_id`；网关 agent 范围内的 `SessionSource.to_dict()` 仍会（镜像到 `scope_id`）为非 relay 会话持久化而发送 `guild_id`，因此它可能仍出现在线路上，但 connector 会忽略它。不要依赖它。 |
| `parent_chat_id` | string | 否 | 当 `chat_id` 指代一个线程时的父频道。 |
| `message_id` | string | 否 | 触发消息的 id（用于置顶/回复/反应）。 |

> `is_bot`（作者是否为 bot/webhook 的分类）在网关侧的 dataclass 上存在，但在 v1 中
> **刻意不在线路上传输**——它不是 `to_dict()` 的一部分。在首先被添加到这里以及
> `to_dict()`（仅追加 bump）之前，不要将其添加到 connector 的 `SessionSource` 中。

### 各平台的 SessionSource 判别字段

| 平台 | chat_id | chat_type | user_id | thread_id | scope_id |
| --- | --- | --- | --- | --- | --- |
| **Discord** | 频道 id | `dm`/`group`/`thread` | 作者 id | 线程频道 id（线程） | **guild id**（服务器隔离所必需） |
| **Telegram** | chat id | `dm`/`group`/`forum` | from id | 论坛主题 id（论坛） | — |

**若 Discord 的 `guild_id` 取错，两个服务器会合并为一个会话。**
这是 #1 高严重性风险。网关的 `build_session_key()` 是一致性 oracle：对于给定的
`SessionSource`，connector 的归一化必须产生与 Python adapter 相同的键。（Phase-1 的
stub 测试断言已知输入 → 已知键。）

### Bot 身份与租户（单 bot 合并，附录 A）

信封将**发起 bot 身份**作为一个**区别于租户**的字段承载。租户由事件自身的判别字段
（Discord `guild_id`、Telegram `chat_id`、webhook 路径/子域名）解析——**绝不**由哪个
token/socket/进程交付它来解析。这保证了一个共享 bot 能为多个租户承载（Phase 6），
而不会重载既有字段。

### 作者优先解析 + 账号绑定（DM）路径（Phase 7）

Phase 7 新增**用户自助加入共享 bot 的流程**，这改变了对于一条被路由的入站消息，
*哪个*判别字段解析其实例——并新增了一条供用户绑定自己账号的管理路径。

**作者优先解析（多租户 guild 规则，D-7.2）。** 单个 Discord guild 可能容纳**多个**租户
——不同的成员各自关联到自己的 agent。因此对于交付，connector 从**已认证的作者绑定**
（`user_instance_binding`，通过 `resolveByUser` 以 `(tenant, platform, platform_user_id)`
为键）解析目标实例，而**不是**通过 guild→实例路由。具体而言：

- 一条由**已绑定**用户发送的路由消息只会到达**该用户自己的**实例——即使同一 guild
  中第二个已绑定用户由不同的实例服务（各自只到达自己的实例）。
- 一条由**未绑定**用户发送的消息解析为**无**实例并被丢弃（**fail-closed**——
  绝不广播给 guild 中的其他租户）。
- 所使用的作者 id 是**从观察到的事件中获取的真实 `user_id`**，即上文文档化的同一
  `SessionSource.user_id`——绝非由网关断言或由管理帧承载的值。

这是 connector 在 `WsGatewayDelivery` 中强制执行的逐 `user_id` 仅限所有者路由
（网关侧的多租户 guild E2E 驱动 `gateway_multitenant_guild_driver.py` 是跨仓库
oracle）。

**账号绑定（DM）路径。** 用户通过一次性验证码将其账号绑定到某实例，验证码通过向共享
bot 发送 DM 兑换：

1. 所有者从 Portal（或自托管 CLI）触发一次绑定。connector 为**已认证的**
   实例签发一个短时**link code**（`POST /manage/link`；instanceId 来自调用方的 principal——
   一个 NAS 签名的 `aud=agent:{instanceId}` token，或该实例自身的 per-gateway
   secret——**绝不是**请求体）。
2. 用户从其想绑定的账号，以**直接消息**形式向共享 bot 发送 `/link <code>`。
3. connector 的入站观察器**消费**该 DM（它不会被路由到任何 agent），并使用从观察到的
   DM 事件中获取的**真实 `user_id`** 写入 `user_instance_binding`。从此刻起，作者优先
   解析会将该用户的消息路由到所绑定的实例。

**Opt-out 由 connector 权威裁决。** 对某实例执行 deprovision
（`POST /manage/deprovision`）会删除其作者绑定（从而其用户不再解析到它），**并**撤销
其 per-gateway secret（从而其 socket 无法再认证——下一次 WS 升级会被关闭，返回
**4401**）。若网关在**先前握手成功之后**收到 **4401 关闭**，则视为终局撤销：它停止
重连并将该 relay 平台报告为**disabled**（不可重试错误）。若 4401 发生在*任何*成功握手
*之前*，则仍可重试（冷启动 / 尚未 provision 的竞态，而非撤销）。

### 3.2 进入空闲 / buffered-flip 原语（§5.3）

一个可缩容至零（scale-to-zero）的**原语**（PRIMITIVE）（并非行为——这里不决定何时休眠
或挂起机器；后续工作流会消费这些帧）。它让网关在不丢失其离开期间到达的入站的前提下
进入排空/空闲过渡，方式是让 connector 为该实例缓冲并在重连时回放。

三个帧（全部以连接的**已认证** per-instance id 为键——在 WS 升级时从存储的 secret
记录中读取，绝不在帧中断言）：

- `{"type":"going_idle"}`（gateway → connector）——作为网关**既有**排空过渡的一部分
  发出（adapter 在拆除 socket 前发送它）。请求 connector 将此实例切换为
  **buffered-only**。
- `{"type":"going_idle_ack"}`（connector → gateway）——connector 已完成切换：
  实时交付已停止，随后该实例的入站被持久化缓冲。网关**在收到此 ack 之前持续服务**
  （因此落在切换窗口内的事件被实时交付，而非丢失——与总线相同的
  SUBSCRIBE-before-serve 顺序约束）。只有在收到 ack 之后才可安全关闭。
- `{"type":"inbound_ack", "bufferId"}`（gateway → connector）——对重连时回放的
  被缓冲 `inbound` 交付（携带其 `bufferId`）的持久化回执。connector 仅在此之后才 ack
  该缓冲条目，从而在**交付段**上实现排空且不重复：一个在排空中途死掉的实例只重发未
  ack 的尾部；已 ack 的条目绝不重发。

**缓冲 + 排空。** 切换后，connector 将入站追加到一个持久的 per-instance 交付段缓冲
（`delivery:<instanceId>`），而非实时推送。在网关**重连**时（一个全新的重连循环在
意外关闭后重新拨号 + 重新握手），新的握手会触发 connector 沿新 socket **按序、以 ack
为门控**地排空积压，随后清除切换以便实时交付恢复。这复用了与 Discord→connector 摄取段
相同的 `drainWithoutDup` 机制，应用于 connector→gateway 交付段。全程由 connector 权威
裁决：网关只能切换/排空**自己的**实例。

> 不在范围内（推迟的行为）：决定何时排空的自主空闲计时器、实际的机器挂起，
> 以及 NAS 的 suspended-health 模型。该原语是"当网关排空时，relay 切换为缓冲 + 在
> 重连时回放，且无丢失/无重复"；至于*什么*触发排空，不在范围内。

### 3.3 唤醒 poke（§5.2）

睡眠/唤醒循环的另一半：一个被挂起的网关如何得知它有待处理的缓冲工作。这是一个
**原语**——这里不挂起机器；它接通的是唤醒**信号**，以便未来的可缩容至零行为层可以依赖
"被缓冲 ⇒ 收到唤醒 poke"。

- **注册。** 网关在 enroll/provision 时注册一个**唤醒 URL**——任何 connector 可以 GET
  以唤醒它的可达 URL（一个 Fly autostart 主机名、一个 dashboard 主机）。自托管：
  `hermes gateway enroll --wake-url <url>`（或 `GATEWAY_RELAY_WAKE_URL` /
  `gateway.relay_wake_url`）。托管/NAS：与 `GATEWAY_RELAY_URL` 一起写入容器环境。在
  `/relay/provision` 请求体中作为 `wakeUrl` 转发，并按实例存储在 connector 的 secret
  记录上（由网关断言但安全地限定作用域——与 `instanceId` 相同的姿态；
  org/tenant 仍由 token 验证，因此网关只能为其**自己的**实例注册唤醒目标）。与已废弃
  的 `gatewayEndpoint` 不同：它是一个**poke 目标**，而非交付目标。
- **poke。** 当一个 buffered-only（going-idle）目标收到其**第一个**被缓冲事件时，
  connector 向该实例已注册的 `wakeUrl` 发起一次**无载荷、无签名**的 GET，**直接**
  发起（非 NAS 中介——relay 保持 NAS 无关）。它不携带任何租户数据，也不携带任何入站：
  它只是说"你有缓冲工作，请重连"。租户权限在网关重新拨号时（已认证的 WS 升级）按正常
  方式重新建立，因此一个泄露/被猜到的唤醒 URL 至多只能导致**自己的**实例被误触发重连。
  按实例限速（每个冷却窗口一次 poke，而非每个事件一次），且是尽力而为——失败的 poke
  被吞掉；网关仍会在它下次自行重连时进行排空。没有新增帧：唤醒是一个带外 HTTP GET，
  而非 relay-WS 消息（socket 已关闭——这正是重点）。

> 不在范围内（推迟的行为）：实际的机器挂起（Fly `autostop:"suspend"`）以及决定何时
> 睡眠的自主空闲计时器。该原语是"给一个睡眠实例的缓冲事件 ⇒ 其 wakeUrl 被 poke"；
> 至于*什么*让实例睡眠（以及唤醒后服务），属于行为层。

### 3.4 对未来可缩容至零行为层的义务

§3.2 与 §3.3 提供的是**原语**；本节是**一个独立可缩容至零行为工作流为安全消费它们所必须
遵守的合同。** 该工作流拥有*决定*挂起、实际机器挂起以及平台/健康模型——这些都不在此处
——但它必须保证以下条件，这些条件是原语所假设的：

1. **在该实例可能被挂起之前注册一个 `wakeUrl`。** 一个未注册 `wakeUrl` 的挂起实例是
   一个黑洞——被缓冲的入站永远不会触发 poke，因此它会睡过自己的流量，直到有别的东西
   让它重连。行为层必须确保注册了一个可达的唤醒目标（自托管：`--wake-url`；托管：
   写入环境）作为允许挂起的前提条件。一个在机器挂起时不可达的唤醒 URL（例如指向被
   挂起机器自身，且前方无平台 autostart）等同于没有。
2. **通过 `going_idle` 排空 → 在拆除 socket 或挂起之前等待 `going_idle_ack`。** 绝不
   在有未 ack 的切换在途时挂起。该 ack 是 connector 对该实例交付现已切换为 buffered-only
   的确认；一台在发送 `going_idle` 之后、ack 之前挂起的机器可能丢失与切换竞态的入站。
   网关已将 socket 拆除门控在 ack 上（Q-5.3c）；挂起步骤必须位于一次干净排空完成*之后*，
   而非与之竞态。
3. **保持全新的重连循环持续活跃，作为挂起的前提条件。** 唤醒→排空合同是"poke ⇒
   网关重新拨号 ⇒ connector 在重连握手上排空"。若重连循环被禁用，poke 落在一台永不
   重新拨号的机器上，缓冲会滞留。行为层不得挂起一个其 relay 传输在唤醒时不会重连的
   实例。
4. **在健康模型中将挂起 ≠ 宕机（Q-5.3b）。** 一个挂起的实例是健康睡眠，而非失败。
   健康/监控层必须区分两者（例如通过平台机器状态），以便挂起的实例不会被重启、告警或
   被当作不健康而回收——那会破坏挂起并可能竞态唤醒/排空。
5. **唤醒 poke 是尽力而为且限速的——不要假设恰好一次或即时唤醒。** 每个实例每个冷却
   窗口至多一次 poke，且失败的 poke 被吞掉。行为层不得将 poke 视为有保证/及时的信号；
   正确性仍依赖于"网关在下次重连时排空"。一个双保险唤醒（例如一个也会重连的定时任务）
   是行为层的选择，而非原语的。
6. **仅在真正空闲时挂起——且空闲由 connector 可观察，而非网关猜测。** *什么*算作空闲
   （无在途 turn + N 分钟无入站）是行为层的策略，但它必须与既有的排空机制
   （`gateway_state` running→draining）组合，而非引入一条并行的、仅 relay 的空闲路径
   ——与 §3.2 对 `going_idle` 施加的同一集成约束。

这些是行为层**欠**原语的保证；原语欠行为层的仅是 §3.2/§3.3 已指定的内容
（一个基于 going_idle 的切换、一个持久的 per-instance 缓冲 + 以 ack 为门控的重连排空，
以及对一个已切换实例的第一个被缓冲事件发起一次 poke）。

---

## 4. 出站：动作集

网关以动作字典调用传输层。事实来源：
`gateway/relay/transport.py` + `gateway/relay/adapter.py`。

| `op` | 字段 | 结果 |
| --- | --- | --- |
| `send` | `chat_id`, `content`, `reply_to?`, `metadata?` | `{success: bool, message_id?, error?}` |
| `edit` | `chat_id`, `message_id`, `content`, `metadata?` | `{success: bool, error?}` |
| `typing` | `chat_id` | `{success: bool}` |
| `follow_up` | `session_key`, `kind`, `content`, `metadata?` | `{success: bool, message_id?, error?}` |

`get_chat_info(chat_id)` 是一个独立的代理调用，至少返回 `{name, type}`。媒体动作遵循
相同的信封形状（推迟到后续合同修订；仅追加）。

**`follow_up`（A2 capability 动作）。** 某些入站载荷携带一个作用于**共享** bot 身份的
凭证（例如 Discord interaction follow-up token）。依据 §6，connector 在边缘剥离它
并以其会话为键绑定到其 capability vault 中；它**永远不会到达网关**。要使用它，网关发出
`follow_up`，命名**它已处于的会话**（`session_key`）以及 capability `kind`（例如
`discord.interaction_token`）——**绝不是 token**。connector 从其 vault 解析真实值，
强制执行租户匹配（租户 B 永远不能使用租户 A 的 capability），然后出口。当 capability
不存在/已过期或租户不匹配时返回 `success: false`——按设计网关无任何可重试之物
（一台泄露的网关持零凭证材料）。事实来源：
`gateway/relay/transport.py`（`send_follow_up`）+ `gateway/relay/adapter.py`。

---

## 5. 中断（`/stop`）路由

- **Gateway → connector：** `send_interrupt(session_key, reason?)` 通过出站 WS 出口一次
  会话中途的 `/stop`。connector 必须将其转发给运行该 `session_key` 的网关实例
  （路由不变式）。
- **Connector → gateway：** 一个针对某 `session_key` 的入站中断以 `interrupt_inbound`
  帧沿网关的出站 WS 下发（§3 传输说明）——通过 relay 总线跨实例路由到持有该 socket 的
  实例——并由 adapter 的 `on_interrupt(session_key, chat_id)` 桥接进既有的逐会话中断
  机制，精确取消该 turn（兄弟 turn 不受影响）。

两个方向都沿网关的出站 WS 传输：网关→connector 的 `/stop` 经其出口，connector→gateway
的中断则作为归一化事件沿同一 `inbound` 回传通道传输。

---

## 6. 信任边界与签名 body 处理（A2）

**connector 是唯一的加密/身份边界。网关不重新验证任何东西。**

Webhook 签名（Discord ed25519、Twilio HMAC、WeCom BizMsgCrypt）基于精确原始字节计算，
且某些载荷用共享密钥*加密*。connector 为多个租户承载一个**共享** bot，并持有每个租户
的平台密钥，因此它：

- **在边缘验证/解密**（密钥仅存于此），
- 将载荷**归一化**为一个租户作用域的 `MessageEvent`（§3），
- **从载荷中剥离任何共享身份 capability** 并以会话为键绑定到其 capability vault
  （见 §4 `follow_up`），
- **仅转发已脱敏的 `MessageEvent`**——绝不转发原始签名 body。

因此网关在 relay 路径上**不**执行任何平台签名/加密验证；它信任归一化后的事件。这是
网关侧的一条受强制执行的不变式（`tests/gateway/relay/test_relay_sheds_crypto.py`：
relay 包不导入/调用任何平台加密）。

**为什么不"逐字节转发签名 body 让网关重新验证"？** 那个早期模型在不可信、可弃用的租户
网关下是不自洽的：

- 重新验证 Twilio HMAC / WeCom 加密需要把**共享签名密钥**交给网关——而这本身就是泄露，
  在共享 bot 上则是一次*跨租户*泄露。
- WeCom 载荷用共享密钥加密；connector 必须在边缘解密才能路由，因此转发密文同样需要
  把密钥给网关。
- 一个 Discord interaction token 位于签名 JSON body **之内**——你无法既保留字节又剥离
  凭证；它们是同一批字节。

因此逐字节保留被刻意放弃：connector 重新序列化已脱敏的事件，网关信任它。这也统一了
passthrough 与 relay 两个平面——二者都是"在边缘验证 → 发出一个归一化事件"，仅在传输上
不同。完整的 A2 依据与 connector 侧 vault 见 `docs/capability-trust-boundary.md`
（connector 仓库：`gateway-gateway`）。

### 6.1 通道认证（connector⇄gateway 链路本身）

A2 让 connector 成为平台密钥的唯一持有者，而网关可能是**客户托管且暴露在公网**的，
因此 connector⇄gateway 通道本身是经过认证的。网关持有一个由 enroll 或 provision 签发的
**per-gateway secret**（`hermes gateway enroll` → connector `/relay/enroll`，或托管
自 provision → `/relay/provision`），用于认证其出站 WS 升级。它是一个带多密钥轮换验证
列表的 HMAC-SHA256 方案（网关侧：`gateway/relay/auth.py`；connector 侧：
`src/core/relayAuthToken.ts`）。

| 段 | 凭证 | 机制 |
|-----|-----------|-----------|
| Gateway → connector WS 升级 | per-gateway secret | 在 `/relay` 升级上携带 `Authorization` bearer 头。token 为 `base64url(payload:exp:sig)`，其中 `payload = gatewayId`，`sig = HMAC(payload:exp, secret)`。connector 验证并在不匹配/缺失/撤销时拒绝升级（**关闭 4401**）。已认证的租户来自 connector 的存储，绝不来自 `hello` 帧。 |
| Connector → gateway 入站（`inbound` / `interrupt_inbound` 帧） | —（沿已认证 WS） | 入站沿网关已认证的出站 socket 推送（§3），因此无需逐消息签名。一个**每租户交付密钥**仍会在 enroll/provision 时签发并保留以保持前向兼容，但已不再用于对入站签名。 |

这是**通道**认证器——区别于平台加密，后者在 relay 路径上仍被完全剥离（§6）。网关持有
零平台密钥；per-gateway secret 仅认证 connector 链路。完整威胁模型 +
enrollment/rotation/kill-switch 设计：`docs/connector-gateway-auth-design.md`
（connector 仓库）。

---

## 7. 按实例交付与管理平面（Phase 6）

Phase 1–5 将 connector 视为单租户前端：某租户的入站事件扇出到该租户的网关 socket。
**Phase 6 将交付改为按实例（per-INSTANCE）**——一个共享 bot 可以在一个租户内
（一个 Discord guild、一个 Telegram bot）为多个用户/agent 承载，而不会跨租户交付——
并新增一个轻量的**管理平面**，供 agent（或托管 Portal）声明谁看到什么以及什么相关。
所有这些都位于**connector 侧**；网关唯一的新职责是在启动时**声明其相关性策略**
（§7.3）。

### 7.1 交付门控（connector 侧，信息性）

对每个入站事件，connector 通过组合三个以 AND 连接的过滤器来决定哪些实例接收它。网关
不实现这些——它们在 connector 中运行——但它们定义了网关所依赖的交付语义：

| 层 | 问题 | 事实来源 |
| --- | --- | --- |
| **owner / scope ∧ principal** | 此实例*可以*在此看到这位作者吗？ | 逐用户的 `user_id → instance` 绑定（所有者下限）+ 逐实例的 `(guild, channel)` scope 授权 + 一个 `owner-only` / `allow-list` / `any` principal 策略。 |
| **visibility 下限** | 该实例绑定的所有者是否真的能在 Discord 中 `VIEW_CHANNEL`？ | 实时 Discord ACL（有效权限），fail-closed。向下收窄一个过宽的 scope 授权。 |
| **relevance** | *给定*它可以看到，agent 是否应当介入？ | §7.3 中声明的相关性策略（address-gating / free-response / allow-bots）。 |

该组合只会**收窄**交付（`deliver ⇔ authorized ∧ visible
∧ relevant`）；**所有者下限绕过相关性层**（作者自己的消息总是到达其自己的实例——
你不会 @mention 你自己的 agent）。由未绑定用户发送的消息不会到达任何实例
（fail-closed）。完整设计与不变式位于 connector 仓库（`NousResearch/gateway-gateway`）；
本节是面向网关的摘要。

### 7.2 管理路由（connector 侧，已认证）

connector 挂载已认证的管理路由。它们共享与 WS 升级**相同的双重认证**：要么是一个托管、
NAS 签名的 `aud=agent:{instanceId}` RS256 JWT，**要么**是网关自身的 per-gateway
secret bearer（§6.1 `make_upgrade_token`）。在两种情况下，connector 都从其**存储的**
记录中解析权威的 `{tenant, instanceId}`——**绝不**来自请求体（请求体中断言的
`instanceId` 被忽略）。

| 路由 | 用途 |
| --- | --- |
| `POST /manage/link` | 为已认证实例签发一个短时验证码以绑定一个平台账号（`/link <code>` 流程；connector 从入站事件读取真实 `user_id`）。 |
| `POST /manage/scope`, `/manage/scope/release` | 为已认证实例声明/释放一个 `(guild, channel)` scope。一个频道至多由一个实例拥有（不重叠是 PK 约束）。 |
| `POST /manage/principal` | 设置实例的 principal 策略（`owner-only` \| `allow-list` \| `any`）。 |
| `POST /manage/dm-default` | 设置用户的 DM-default 实例（当用户绑定了多个实例时的 DM 仲裁）。 |
| `POST /relay/policy` | 声明实例的**相关性策略**（§7.3）。 |

这些由 connector 拥有（管理平面不是网关 agent 路径的一部分）；网关仅调用
`POST /relay/policy`（§7.3）。其余由托管 Portal / `hermes` CLI 驱动。

### 7.3 相关性策略声明（网关的职责）

相关性层（§7.1）是网关自身行为开关（`require_mention`、`free_response_channels`、
`{PLATFORM}_ALLOW_BOTS`）的逐租户对等物。因此**同一**行为支配 relay 交付，网关将这些
开关投影为一个**平台无关**的策略，并在启动时（在其 per-gateway secret 解析完成后）
POST 到 `POST /relay/policy`。

请求体（`gateway/relay/__init__.py` 的 `relay_relevance_policy()` → `send_relay_policy()`）：

| 字段 | 类型 | 投影自 | 含义 |
| --- | --- | --- | --- |
| `platform` | string | 所承载的平台（`relay_platform_identity`） | 此策略应用于哪个平台。 |
| `requireAddress` | bool | `require_mention` | 一条非所有者消息必须 @mention / 回复 bot 才被视为相关。 |
| `freeResponseScopes` | string[] | `free_response_channels` | `requireAddress` 被豁免的 scope（频道）id。与 §7.1 的 scope 授权使用相同的 scope 词汇表。 |
| `allowOtherBots` | bool | `{PLATFORM}_ALLOW_BOTS ∈ {mentions, all}` | 是否允许 bot 发送的消息（默认关闭）。 |

认证使用 per-gateway 升级 token（§6.1），因此 connector 将策略附加到已认证实例。网关是
**事实来源**，并在**每次启动时**重新声明（一次全量替换，镜像 provision 时的
`routeKeys` upsert——自愈）。当投影出的策略全为默认值时，网关不发送任何内容
（connector 的 absent-row 默认值已经匹配）。该 POST 是**fail-soft**：失败仅记录日志并
继续启动——相关性是叠加在授权门控（§7.1）之上的优化层，绝非启动依赖。**没有新增的网关
入站面**，也**没有新增凭证**——它复用 per-gateway secret 以及与
`/relay/provision` 相同的主机。

> 相关性丢弃发生在 connector 唤醒一个可缩容至零的 agent（Phase 5）**之前**，因此被
> 排除的闲聊绝不会把 agent 拉起——相关性既是可缩容至零的主要杠杆，也是一个正确性
> 过滤器。

---

## 8. 版本化策略

- `contract_version` 是一个 int；在实验阶段**仅**为追加性变更（新增可选字段、新增
  `op`）而 bump。
- 一次破坏性变更（字段被重命名/移除、语义改变）需要对两个仓库进行协调更新并 bump
  版本。
- connector 的第一个 PR 需引用其所实现的本文件的 commit SHA。
