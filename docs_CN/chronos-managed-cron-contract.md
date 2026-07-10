# Chronos 托管 cron — agent ↔ NAS 线上合同规范

**状态：** Chronos cron provider 的权威线上规范（wire spec）。
**目标读者：** `agent-cron` endpoints（`nous-account-service`）的 NAS 侧实现者，
以及任何在调试托管 cron 链路的人。

Chronos 让托管的 Hermes gateway 能够在空闲时 **缩容到零**，同时仍能触发
cron 作业。agent 不再使用进程内的 60 秒 ticker，而是请求 NAS 为每个作业
在其实际的下次触发时间点 **精确地布防一个外部一次性触发（one-shot）**。NAS
在触发时通过一个经过认证的 webhook 回调 agent；agent 执行该作业并重新布防
下一个 one-shot。两次触发之间，agent 进程可以被完全停止——它只在真正触发时
才被唤醒。

NAS 用来实现这些 one-shot 的外部调度器，是一个 **NAS 内部实现细节**。
agent 从不与它通信，从不持有它的凭据，也从不命名它。agent 只知道下文这三个
NAS endpoints。

```
create/update/pause/resume/remove a cron job (agent side)
  │
  ▼
ChronosCronScheduler.reconcile()        ── agent computes next_run_at
  │  POST {portal}/api/agent-cron/provision   (auth: agent's Nous access token)
  ▼
NAS arms a one-shot for fire_at         ── NAS owns the scheduler + its creds
  │
  ⏰ at fire_at
  ▼
scheduler → POST {portal}/api/agent-cron/relay   (auth: scheduler signature, NAS-verified)
  │
  ▼
NAS mints a short-lived agent-audience JWT (purpose=cron_fire)
  │  POST {agent_callback_url}/api/cron/fire        (auth: that JWT)
  ▼
agent verifies the NAS JWT → store CAS claim → run_one_job → re-arm next one-shot
```

## 信任模型（请先阅读本节）

| 跳数 | 谁调用谁 | 认证机制 | 由谁验证 |
|---|---|---|---|
| 1 | agent → NAS（`provision`/`cancel`/`list`） | agent 既有的 **Nous Portal access token**（Bearer）—— 对托管 agent 而，这是 NAS 植入 `auth.json` 的 **bootstrap-session token**（client `hermes-cli-vps`），**不是** `agent:*` client token | NAS（走其常规的 agent-token 路径） |
| 2 | scheduler → NAS（`relay`） | 调度器请求的 **签名（signature）** | NAS（走它已有的签名验证路径） |
| 3 | NAS → agent（`/api/cron/fire`） | 一个 **NAS 签发的短时 JWT**（`aud=agent:{instance_id}`，`purpose=cron_fire`） | agent（用 PyJWT 校对 NAS JWKS） |

> **究竟用哪个 token（跳数 1）。** 托管 agent 从不持有
> `agent:{instance_id}` OAuth client 凭据——这种形态只能由交互式仪表盘的
> auth-code grant（浏览器用户）签发。对所有自有的出站 portal 调用，
> agent 都使用 **bootstrap-session access token**（`resolve_nous_access_token`），
> 它是在仅 bootstrap 用的 client `hermes-cli-vps` 下签发，并在容器首次启动时
> 注入容器。因此 NAS 必须从 **EITHER** 一个 `agent:{id}` client
>（自托管/仪表盘调用方）**OR** —— 对 bootstrap token 而言 —— 从与该 token 的
> session id（`sid`）匹配的 `AgentInstance.bootstrapSessionId`（按 org 作用域）
> 解析出调用 agent 的 instance id。无论如何，跳数 3 签发的 fire JWT 仍带
> `aud=agent:{instance_id}`。（如果仅凭 `agent:*` client 对跳数 1 做门禁，
> 会让每一个真实的托管 agent provision 都返回 403 —— 见
> `src/server/agent-cron/instance-auth.ts`。）

为什么是 NAS 居间，而不是 scheduler→agent 直连：调度器使用 **NAS 的** 密钥
签名，而 agent 并不（也不应）持有这些密钥。agent 只能验证一个
**NAS 签发的** token——这是它已有的一条信任路径。这样就把所有调度器凭据都
留在 NAS 内部。（完整理由见该计划的 DQ-4。）

agent 侧不引入任何新密钥：跳数 1 复用 agent 已经用于 portal 的 token，
跳数 3 复用 agent 已经在执行的 NAS-JWT 校验。

---

## Endpoint 1 — `POST /api/agent-cron/provision`  (agent → NAS)

为某个作业布防（或幂等地重新布防）恰好一个 one-shot。

- **认证：** `Authorization: Bearer <agent Nous access token>`。NAS 通过其
  常规的 agent-token 路径校验，并将该行按调用 agent/org 作用域隔离。
- **请求体：**
  ```json
  {
    "job_id": "ab12cd34",
    "fire_at": "2026-06-18T12:34:56+00:00",
    "agent_callback_url": "https://agent-xyz.fly.dev",
    "dedup_key": "ab12cd34:2026-06-18T12:34:56+00:00"
  }
  ```
  - `fire_at` — ISO 8601，**由 agent 计算**。可以是未来不到一分钟的某个时刻；
    NAS 必须支持秒级粒度（时间由 agent 拥有，因此不存在 1 分钟的调度器下限）。
  - `agent_callback_url` — agent 自己的、可公开访问的 base URL。NAS 会在
    触发时 POST `{agent_callback_url}/api/cron/fire`。
  - `dedup_key` — `"{job_id}:{fire_at}"`。NAS **按 `(agent_id, job_id)` 做
    upsert**，因此对同一 fire 重新布防是幂等的（不会产生重复 one-shot）。
    对同一 `job_id` 提交新的 `fire_at` 会替换之前的布防。
- **动作：** 布防一个在 `fire_at` 触发的 one-shot，目的地为 NAS 的
  **relay** 路由（Endpoint 3）—— 而不是直连 agent，这样 NAS 仍留在链路中
  以签发 agent JWT。持久化 `(agent_id, job_id, schedule_id,
  agent_callback_url)`。
- **响应：** `200 {"schedule_id": "<opaque>"}`。

## Endpoint 2 — `POST /api/agent-cron/cancel`  (agent → NAS)

- **认证：** 同 Endpoint 1。
- **请求体：** `{"job_id": "ab12cd34"}`。
- **动作：** 取消 `(agent_id, job_id)` 对应的已布防 one-shot 并删除该行。
  幂等——取消一个未知的作业返回 200 no-op。
- **响应：** `200 {"ok": true}`。

## Endpoint 3 — `POST /api/agent-cron/relay`  (scheduler → NAS，触发中继)

- **认证：** 调度器请求的 **签名（signature）**，由 NAS 用它已有的签名路径
  校验。这是触发的信任边界——一个伪造的 relay 调用必须在此被拒绝。
- **动作：**
  1. 从持久化的行中查询 `(agent_id, job_id) → agent_callback_url`。
  2. 签发一个 **短时** JWT：`aud = "agent:{instance_id}"`、
     `iss = {portal_url}`、`purpose = "cron_fire"`，较小的 `exp`（≈60–120s），
     用 NAS 常规的非对称签名密钥签名（通过 JWKS 发布）。
  3. `POST {agent_callback_url}/api/cron/fire`，带
     `Authorization: Bearer <that JWT>`，body 为
     `{"job_id": "...", "fire_at": "..."}`。
  4. 将 agent 的非 2xx 响应视为 **可重试** 失败（让调度器重试 relay）。
     agent 的 store CAS 会对二次触发做去重，因此重试是安全的。
- **对调度器的响应：** 一旦 agent POST 被接受即返回 2xx（202），
  这样调度器不会重试一个已送达的 fire。

---

## 入站 `POST /api/cron/fire`  (NAS → agent) — agent 侧，已实现

这是 NAS 在 Endpoint 3 第 3 步调用的 agent endpoint。由 **dashboard app**
（`hermes_cli/web_server.py`）提供——它是托管部署中 agent 始终可公开访问的
HTTP 入口（gateway 可能处于空闲/缩容状态）；它被列入 `PUBLIC_API_PATHS`，
因此 dashboard 的 cookie 门禁会让这个 bearer-JWT 回调直通到校验器。（同时
也注册在可选的 `APIServerAdapter` 上，用于自托管的 API-server 部署。）
校验器是 `plugins/cron/chronos/verify.py`。

- **认证：** `Authorization: Bearer <NAS-minted JWT>`。agent 校验：
  - 签名（对 NAS JWKS，`cron.chronos.nas_jwks_url`），
  - `aud` == `cron.chronos.expected_audience`（本 agent 的
    `agent:{instance_id}`），
  - `iss` == `cron.chronos.portal_url`，
  - `exp` / `nbf`（30 秒宽限），
  - `purpose == "cron_fire"` —— 通用 agent JWT（无 purpose 或其他 purpose）
    会被拒绝，使其无法在此 endpoint 上被重放。
- **请求体：** `{"job_id": "ab12cd34", "fire_at": "..."}`（仅使用 `job_id`）。
- **行为：**
  - 无效/缺失/伪造/过期/aud 错误/purpose 错误的 token → **401**，不执行。
  - 缺失 `job_id` → **400**。
  - 有效 → 立即返回 **202 `{"status": "accepted", "job_id": "..."}`**，
    作业在后台运行。先回 202 再运行，意味着一个耗时的 agent turn 永远不会
    触发 relay 的 HTTP 超时。
- **At-most-once:** agent 在运行作业前，用 store 级别的 compare-and-set
  （`claim_job_for_fire`）来声明该作业。一个在首次触发进行中（或已完成）之后
  到达的 relay/scheduler 重试会失去声明，不会重复运行。

---

## At-most-once 与 re-arm 语义

- **周期性（cron/interval）：** 触发时，agent 在其 store lock 下，作为
  声明的一部分推进 `next_run_at`，运行作业，然后为新的 `next_run_at`
  重新 provision 一个 one-shot。一个针对旧 `fire_at` 的重复 relay 会发现
  声明已被取走 / 时间已推进，从而被丢弃。
- **一次性（`30m`、`+90s` 等）：** 触发一次；`mark_job_run` 将其标记为
  completed。不 re-arm。
- **`repeat.times = N`：** `mark_job_run` 在到达上限时删除作业，因此
  `get_job` 在最终触发之后返回 `None` → agent **不会** re-arm →
  调度干净地停止，不会留下孤立的 one-shot。
- **多副本 agents：** store CAS 使得在共享同一个 `HERMES_HOME` 的 N 个
  gateway 副本之间，触发是 at-most-once 的——恰好一个副本运行每次触发。

## Reconcile（自愈）

agent 在以下时机对 desired（`jobs.json`）与 armed 状态做 reconcile：
- `start()`（gateway 启动 / 唤醒），
- 每次成功的作业变更之后（`on_jobs_changed`），
- 每次触发后附带进行（re-arm）。

Reconcile 会布防缺失/时间已变的作业并取消孤立的作业。一次失败的 provision
（瞬时 NAS 错误）会在下一次 reconcile 时自愈。**不存在** 对休眠 agent 的
周期性唤醒——那样会抵消 scale-to-zero 的意义。

## 配置（agent 侧）

全部为非密钥项（`config.yaml` 中的 `cron.chronos.*`）；agent 不持有调度器
凭据。对托管 agent，NAS 在 provision 时设置这些：

| key | 含义 |
|---|---|
| `cron.provider` | `"chronos"` 表示启用（留空 = 内置 ticker） |
| `cron.chronos.portal_url` | NAS base URL（同时也是期望的 JWT `iss`） |
| `cron.chronos.callback_url` | agent 自己的公开 base URL，用于 NAS→agent 触发 |
| `cron.chronos.expected_audience` | 本 agent 的 JWT `aud`（`agent:{instance_id}`） |
| `cron.chronos.nas_jwks_url` | 用于校验 fire JWT 的 NAS JWKS |

如果 `callback_url` / `portal_url` 为空，或 agent 没有 Nous 登录，
`is_available()` 返回 False，解析器回退到内置的进程内 ticker——cron 永远不会
丢失其触发器。

## 应急逃生口（非默认）

inbound `/api/cron/fire` 的校验器是可插拔的（`get_fire_verifier()`）。
如果经 NAS 的 relay 流量出现过饱和，可以引入一种带每作业 NAS 签发 cron-key
的、scheduler→agent 直连模式来替换 NAS-JWT 校验器，而 **无需改动 webhook
handler**。NAS 居间（本合同）是默认方式。
