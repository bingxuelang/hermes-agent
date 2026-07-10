# Hermes 中间件

Hermes 中间件是观察者钩子的、可改变行为的伴生组件。观察者钩子报告发生了什么。中间件可以通过在执行前重写请求或通过包装执行回调本身来改变发生的事情。

该合同被有意设计为后端中立。插件可以将其用于本地策略、请求塑形、追踪、自适应路由、缓存控制、沙盒选择，或交接到诸如 NeMo Relay 这类运行时，而无需改变 Hermes 的规划器、模型提供者适配器、工具注册表、记忆或 CLI UX。

在启用中间件的情况下，插件可以：

- 在 Hermes 调用提供者之前重写 LLM 提供者请求 kwargs。
- 在护栏、审批检查、钩子和工具执行看到工具参数之前重写它们。
- 包装实际的 LLM 执行回调，同时保留 Hermes 的重试、流式传输、中断和钩子行为。
- 包装实际的工具执行回调，同时保留 Hermes 的护栏、审批、工具后钩子和工具结果转换。

## 合同

插件从 `register(ctx)` 中注册中间件：

```python
def register(ctx):
    ctx.register_middleware("llm_request", on_llm_request)
    ctx.register_middleware("llm_execution", on_llm_execution)
    ctx.register_middleware("tool_request", on_tool_request)
    ctx.register_middleware("tool_execution", on_tool_execution)
```

每个中间件回调接收：

- `telemetry_schema_version`：当前为 `hermes.observer.v1`
- `middleware_schema_version`：当前为 `hermes.middleware.v1`
- 运行时上下文，例如在适用时的 `session_id`、`task_id`、`turn_id`、`api_request_id`、`provider`、`model`、`api_mode`、`tool_name` 和 `tool_call_id`。

受支持的中间件种类：

| 种类 | 载荷 | 返回形态 | 用途 |
| --- | --- | --- | --- |
| `llm_request` | `request`、`original_request` | `{"request": {...}}` | 在提供者执行之前替换有效的提供者 kwargs。 |
| `tool_request` | `tool_name`、`args`、`original_args` | `{"args": {...}}` | 在钩子、护栏、审批和执行之前替换有效的工具参数。 |
| `llm_execution` | `request`、`original_request`、`next_call` | 任意提供者响应 | 包装或替换实际的提供者调用。 |
| `tool_execution` | `tool_name`、`args`、`original_args`、`next_call` | 任意工具结果 | 包装或替换实际的工具调用。 |

请求中间件可以返回可选的追踪字段：

```python
return {
    "request": updated_request,
    "source": "my-plugin",
    "reason": "selected fallback model",
}
```

Hermes 将这些追踪条目存储在后续的观察者钩子载荷中，作为 `middleware_trace`。

执行中间件接收一个 `next_call` 回调。调用它以继续链条：

```python
def on_tool_execution(**kwargs):
    result = kwargs["next_call"](kwargs["args"])
    return result
```

如果多个插件注册了同一种执行中间件，Hermes 会按注册顺序将它们作为嵌套链条运行。中间件失败采用失败放行策略：Hermes 记录警告，并继续执行下一个中间件或基础运行时路径。

## 执行顺序

### LLM 调用

对于每个提供者请求，Hermes 按以下顺序应用中间件：

1. 从当前会话构建提供者 kwargs。
2. 应用 `llm_request` 中间件。
3. 以有效请求触发 `pre_api_request` 观察者钩子。
4. 通过 `llm_execution` 中间件运行提供者执行。
5. 触发 `post_api_request` 或 `api_request_error` 观察者钩子。

请求中间件能看到完整的提供者 kwargs，包括 `messages` 或 Responses API 的 `input`、模型设置、工具定义、流选项以及提供者专有选项。执行中间件接收相同的有效请求，外加 `next_call`。

### 工具调用

对于每个工具调用，Hermes 按以下顺序应用中间件：

1. 解析并强制转换模型提供的工具参数。
2. 应用 `tool_request` 中间件。
3. 针对有效参数运行常规的 Hermes 执行前路径：工具可用性检查、观察者阻止指令、护栏和审批检查。
4. 通过 `tool_execution` 中间件运行工具执行。
5. 触发 `post_tool_call` 观察者钩子。
6. 在结果被追加回会话上下文之前应用 `transform_tool_result` 钩子。

工具请求中间件在审批检查之前运行。请谨慎使用：被重写的路径、命令或 URL 正是下游策略将要评估的值。

## 启用

中间件仅对已启用的插件运行。对于内置插件：

```bash
hermes plugins enable <plugin-name>
```

对于隔离的本地测试，使用同一个 `HERMES_HOME` 进行插件启用和代理运行：

```bash
export HERMES_HOME=/tmp/hermes-middleware-test
mkdir -p "$HERMES_HOME"
hermes plugins enable <plugin-name>
hermes chat --query 'Reply exactly ok'
```

对于源代码检出，请优先使用 source 命令，以便运行时能够看到工作树中的插件和中间件：

```bash
uv sync
uv run hermes plugins enable <plugin-name>
uv run hermes chat --query 'Reply exactly ok'
```

## 通用插件示例

以下示例被有意保持得很小。它们展示了中间件合同的形态，而不依赖 NeMo Relay。

### LLM 请求中间件

该插件为提供者请求打标，并记录一条中间件追踪条目：

```python
def register(ctx):
    ctx.register_middleware("llm_request", tag_llm_request)


def tag_llm_request(**kwargs):
    request = dict(kwargs["request"])
    extra_body = dict(request.get("extra_body") or {})
    extra_body.setdefault("metadata", {})["hermes_middleware_demo"] = True
    request["extra_body"] = extra_body
    return {
        "request": request,
        "source": "middleware-demo",
        "reason": "tagged provider request",
    }
```

有效请求会被传递给 `pre_api_request`、提供者执行和 `post_api_request`。

### 工具请求中间件

该插件将 `terminal` 调用约束到已知的工作目录：

```python
def register(ctx):
    ctx.register_middleware("tool_request", normalize_terminal_workdir)


def normalize_terminal_workdir(**kwargs):
    if kwargs.get("tool_name") != "terminal":
        return None
    args = dict(kwargs["args"])
    args.setdefault("workdir", "/tmp/hermes-middleware-demo")
    return {
        "args": args,
        "source": "middleware-demo",
        "reason": "defaulted terminal workdir",
    }
```

由于它在钩子和审批之前运行，下游遥测和策略观察到的就是被重写后的 `workdir`。

### LLM 执行中间件

该插件包装提供者调用，并保留原始的提供者响应：

```python
import time


def register(ctx):
    ctx.register_middleware("llm_execution", time_llm_execution)


def time_llm_execution(**kwargs):
    started = time.monotonic()
    response = kwargs["next_call"](kwargs["request"])
    elapsed_ms = int((time.monotonic() - started) * 1000)
    print(f"llm_execution elapsed_ms={elapsed_ms}")
    return response
```

返回 Hermes 期望从提供者适配器得到的相同响应形态。不要将响应包装在插件专有的信封中，除非运行时的其余部分期望该信封。

### 工具执行中间件

该插件包装工具执行，同时保留工具结果：

```python
def register(ctx):
    ctx.register_middleware("tool_execution", annotate_tool_execution)


def annotate_tool_execution(**kwargs):
    result = kwargs["next_call"](kwargs["args"])
    # Metrics, logging, or external routing can happen here.
    return result
```

执行中间件可以调用 `next_call(modified_args)`，以将修改后的载荷传递给后续中间件和基础工具分派器。

插件专有的示例应当与拥有该行为的插件放在一起。关于 NeMo Relay 自适应执行中间件，请参见 [`plugins/observability/nemo_relay/README.md`](../../plugins/observability/nemo_relay/README.md)。

## 安全说明

- 中间件对相同输入应当是确定性的，除非它显式地路由到动态的外部系统。
- 请求中间件应当返回完整的替换载荷，而非部分补丁。
- 执行中间件应当恰好调用 `next_call(...)` 一次，除非它有意短路执行。
- 如果执行中间件在调用 `next_call(...)` 之前抛出异常，Hermes 会将其视为中间件失败，并继续执行剩余的中间件链和基础执行。
- 如果执行中间件成功调用 `next_call(...)` 后在后处理期间抛出异常，Hermes 会保留下游结果，并且不会再次运行提供者或工具。
- 如果下游提供者或工具执行失败，中间件可以让该错误传播或有意地对其进行转换。Hermes 不会将下游失败转换为成功的 `None` 结果。
- 工具请求中间件在审批之前运行。如果它修改了文件路径、命令、URL 或参数，被修改后的值正是护栏和审批将要评估的对象。
- 观察者钩子仍然是只读遥测的正确位置。仅当插件需要改变或包装行为时才使用中间件。
