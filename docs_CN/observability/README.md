# Hermes 观察者钩子

Hermes 观察者钩子是为需要在不改变运行时行为的前提下重建代理执行的插件提供的只读遥测合同（telemetry contract）。该合同支持 trace（追踪）、metrics（指标）、audit（审计）、replay（回放）以及导出集成，例如 Langfuse、OpenTelemetry 风格的采集器和 NeMo Relay。

观察者钩子被有意设计为后端中立（backend-neutral）。它们暴露稳定的生命周期事件、关联 ID、已脱敏的载荷、计时、状态和错误字段。它们不替代 Hermes 的规划器、模型提供者、记忆、工具注册表、审批 UX、CLI、网关行为或执行语义。

改变行为的请求或执行包装器不属于此观察者合同范畴。观察者钩子应当报告发生了什么；它们不应替换提供者请求、工具参数或执行回调。

## 合同

插件从 `register(ctx)` 中注册观察者回调：

```python
def register(ctx):
    ctx.register_hook("pre_api_request", on_pre_api_request)
    ctx.register_hook("post_api_request", on_post_api_request)
    ctx.register_hook("pre_tool_call", on_pre_tool_call)
    ctx.register_hook("post_tool_call", on_post_tool_call)
```

每个钩子回调接收关键字参数。插件应当接受 `**kwargs`，以便新增字段保持向后兼容：

```python
def on_post_tool_call(**kwargs):
    tool_name = kwargs.get("tool_name")
    status = kwargs.get("status")
    result = kwargs.get("result")
```

插件管理器向每个钩子载荷注入该字段：

```text
telemetry_schema_version = "hermes.observer.v1"
```

钩子回调采用 fail-open（失败放行）策略。Hermes 会捕获回调异常、记录警告，并保持代理循环继续运行。

大多数观察者钩子的返回值会被忽略。例外的是较早的、影响行为的钩子：

| 钩子 | 返回行为 |
| --- | --- |
| `pre_llm_call` | 可返回字符串或 `{"context": "..."}`，以将临时上下文注入当前用户消息。 |
| `pre_tool_call` | 可返回 `{"action": "block", "message": "..."}`，以在执行前阻止工具。 |
| `transform_tool_result` | 可在 `post_tool_call` 之后返回替换用的工具结果字符串。 |
| `transform_llm_output` | 可返回替换用的最终助手文本字符串。 |

遥测插件应当将这些影响行为的返回视为可选的兼容特性，而非可观测性要求。

## 关联 ID

观察者载荷使用稳定的 ID，以便插件能够在不单靠回调顺序的情况下关联事件。

| 字段 | 含义 |
| --- | --- |
| `session_id` | 会话/会话身份标识。 |
| `task_id` | 任务身份标识，对子代理和隔离执行尤其有用。 |
| `turn_id` | 用户轮次身份标识，由一轮中的 API 尝试和工具调用共享。 |
| `api_request_id` | 不透明的提供者尝试身份标识。不要解析其字符串格式。 |
| `api_call_count` | 代理循环内的数值型 API 尝试计数。 |
| `tool_call_id` | 可用时的提供者提供的工具调用 ID。 |
| `parent_session_id` / `child_session_id` | 用于委派子代理的会话链接。 |
| `parent_subagent_id` / `child_subagent_id` | 可用时的子代理链接。 |
| `parent_turn_id` | 触发委派工作的父轮次。 |

消费者应当优先使用显式字段，而非解析复合 ID。特别是，`api_request_id` 是一个不透明的关联值。

## 事件族

### 会话生命周期

会话钩子描述会话边界与重置：

| 钩子 | 触发时机 |
| --- | --- |
| `on_session_start` | 在系统提示词构建完成后，全新会话启动。 |
| `on_session_end` | 一次 `run_conversation` 调用结束，包括被中断或未完成的轮次。 |
| `on_session_finalize` | CLI 或网关拆除一个活动的会话身份标识。 |
| `on_session_reset` | CLI 或网关从旧的会话身份标识切换到新的会话身份标识。 |

在可用时，常见字段包括 `session_id`、`completed`、`interrupted`、`reason`、`old_session_id` 和 `new_session_id`。

`on_session_end` 是按轮次/运行划定作用域的。它不一定是聊天身份标识的最终生命周期边界。对于必须每个会话身份标识发生一次的生命周期清理，请使用 `on_session_finalize` 和 `on_session_reset`。

### 轮次作用域的 LLM 钩子

这些钩子界定用户轮次，而非单个提供者 API 尝试：

| 钩子 | 触发时机 |
| --- | --- |
| `pre_llm_call` | 在用户轮次的工具循环开始之前。 |
| `post_llm_call` | 在轮次以最终助手输出完成之后。 |

常见的 `pre_llm_call` 字段包括 `session_id`、`turn_id`、`user_message`、`conversation_history`、`is_first_turn`、`model`、`platform` 和 `sender_id`。

常见的 `post_llm_call` 字段包括 `session_id`、`turn_id`、`user_message`、`assistant_response`、`conversation_history`、`model` 和 `platform`。

对于 LLM span 遥测，请使用请求作用域的 API 钩子。对于轮次级别的上下文、兼容性和最终轮次摘要，请使用 `pre_llm_call` 和 `post_llm_call`。

### 请求作用域的 API 钩子

API 钩子描述代理循环内的提供者尝试：

| 钩子 | 触发时机 |
| --- | --- |
| `pre_api_request` | 紧接着在提供者 API 请求之前。 |
| `post_api_request` | 在提供者成功响应之后。 |
| `api_request_error` | 在提供者请求失败或可重试的错误路径之后。 |

`pre_api_request` 包括：

- 身份标识：`session_id`、`task_id`、`turn_id`、`api_request_id`
- 运行时：`platform`、`model`、`provider`、`base_url`、`api_mode`
- 尝试元数据：`api_call_count`、`message_count`、`tool_count`、`approx_input_tokens`、`request_char_count`、`max_tokens`
- 计时：`started_at`
- 已脱敏的请求载荷：`request`

`post_api_request` 包括相同的身份标识/运行时字段，外加：

- `api_duration`、`started_at`、`ended_at`
- `finish_reason`、`message_count`、`response_model`
- `usage`
- `assistant_content_chars`、`assistant_tool_call_count`
- 已脱敏的响应载荷：`response`
- 兼容性对象：`assistant_message`

`api_request_error` 包括相同的身份标识/运行时字段，外加：

- `api_duration`、`started_at`、`ended_at`
- `status_code`、`retry_count`、`max_retries`、`retryable`、`reason`
- 结构化的 `error = {"type": ..., "message": ...}`
- 已脱敏的失败请求载荷：`request`

已脱敏的 `request`、`response` 和 `error` 字段是新消费者的规范化观察者输入。

### 工具生命周期

工具钩子描述单个工具调用：

| 钩子 | 触发时机 |
| --- | --- |
| `pre_tool_call` | 在经护栏批准的工具分派之前。 |
| `post_tool_call` | 在工具分派、取消、阻止或错误完成之后。 |
| `transform_tool_result` | 在 `post_tool_call` 之后、结果被追加到模型上下文之前。 |

`pre_tool_call` 包括 `tool_name`、`args`、`task_id`、`session_id`、`tool_call_id`、`turn_id` 和 `api_request_id`。

`post_tool_call` 包括相同的身份标识字段，外加 `result`、`duration_ms`、`status`、`error_type` 和 `error_message`。

`status` 是观察者级别的生命周期结果。常见值包括：

| 状态 | 含义 |
| --- | --- |
| `ok` | 工具正常完成。 |
| `error` | 工具运行并返回或抛出了错误结果。 |
| `blocked` | 一个 `pre_tool_call` 钩子阻止了执行。 |
| `cancelled` | 执行在正常完成之前被取消。 |

`post_tool_call` 也会在阻止和取消路径上被触发，以便遥测插件能够干净地关闭 span。

### 审批生命周期

审批钩子描述危险命令的审批提示：

| 钩子 | 触发时机 |
| --- | --- |
| `pre_approval_request` | 在审批请求被展示或发送之前。 |
| `post_approval_response` | 在用户响应或请求超时之后。 |

常见字段包括 `command`、`description`、`pattern_key`、`pattern_keys`、`session_key` 和 `surface`。

`post_approval_response` 还包括 `choice`，其值例如 `once`、`session`、`always`、`deny` 和 `timeout`。

审批钩子是仅观察者使用的。插件不能从这些钩子预先回答或否决审批。要阻止工具进入审批，请使用 `pre_tool_call` 阻止机制。

### 子代理生命周期

子代理钩子描述委派的子代理工作：

| 钩子 | 触发时机 |
| --- | --- |
| `subagent_start` | 一个委派的子代理被创建。 |
| `subagent_stop` | 一个委派的子代理返回或失败。 |

`subagent_start` 字段包括 `parent_session_id`、`parent_turn_id`、`parent_subagent_id`、`child_session_id`、`child_subagent_id`、`child_role` 和 `child_goal`。

`subagent_stop` 字段包括父/子会话 ID、角色/状态字段、`child_summary` 和 `duration_ms`。

观察者可以使用这些钩子对嵌套轨迹建模，同时保持子代理执行与触发它的父轮次相关联。

## 载荷安全

观察者载荷是为遥测消费者设计的，而非用于原始对象访问。新消费者应当使用已脱敏的 API 载荷：

- `pre_api_request.request`
- `post_api_request.response`
- `api_request_error.request`
- `api_request_error.error`

脱敏处理将提供者对象转换为 JSON 兼容的结构，限制大型载荷的规模，抹除敏感键，并避免在已脱敏字段中暴露原始响应对象。

诸如 `request_messages`、`conversation_history` 和 `assistant_message` 这类遗留兼容性字段可能仍为现有插件保留。新的可观测性消费者应当优先使用已脱敏的载荷。

## 性能

默认的未插桩路径应当保持低成本。昂贵的请求/响应载荷构建受到 `has_hook(...)` 的门控，因此 Hermes 仅在至少有一个插件注册了相关钩子时才构建已脱敏的 API 遥测载荷。

插件作者应当保持该特性：

- 仅注册插件实际使用的钩子。
- 避免对已脱敏的载荷进行深拷贝或再次脱敏。
- 保持钩子回调快速且失败放行。
- 在可行时，将网络导出或批量写入卸载到后台。

## 编写观察者插件

最小化观察者插件：

```python
def register(ctx):
    ctx.register_hook("pre_api_request", on_pre_api_request)
    ctx.register_hook("post_api_request", on_post_api_request)
    ctx.register_hook("pre_tool_call", on_pre_tool_call)
    ctx.register_hook("post_tool_call", on_post_tool_call)


def on_pre_api_request(**kwargs):
    start_llm_span(
        request_id=kwargs.get("api_request_id"),
        turn_id=kwargs.get("turn_id"),
        request=kwargs.get("request"),
        model=kwargs.get("model"),
    )


def on_post_api_request(**kwargs):
    finish_llm_span(
        request_id=kwargs.get("api_request_id"),
        response=kwargs.get("response"),
        usage=kwargs.get("usage"),
        duration=kwargs.get("api_duration"),
    )


def on_pre_tool_call(**kwargs):
    start_tool_span(
        call_id=kwargs.get("tool_call_id"),
        name=kwargs.get("tool_name"),
        args=kwargs.get("args"),
    )


def on_post_tool_call(**kwargs):
    finish_tool_span(
        call_id=kwargs.get("tool_call_id"),
        result=kwargs.get("result"),
        status=kwargs.get("status"),
        duration_ms=kwargs.get("duration_ms"),
    )
```

使用 `session_id`、`turn_id`、`api_request_id` 和 `tool_call_id` 进行 span 关联。当导出格式支持嵌套代理工作或安全生命周期事件时，使用子代理和审批钩子。

## 现有消费者

内置的 Langfuse 插件演示了针对轮次、提供者请求和工具调用的、基于钩子的直接可观测性。

内置的 NeMo Relay 插件将同一通用观察者合同映射到 NeMo Relay 作用域、LLM span、工具 span、标记、ATOF 流和 ATIF 导出。NeMo Relay 专有的配置和示例位于 [`plugins/observability/nemo_relay/README.md`](../../plugins/observability/nemo_relay/README.md)。
