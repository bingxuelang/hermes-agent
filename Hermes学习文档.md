# Hermes Agent 学习文档 —— 从入门到精通

> 基于 Hermes Agent v0.18.2 源码深度分析整理
> Hermes 是 [Nous Research](https://nousresearch.com) 出品的自我改进型 AI 智能体

---

# 第一篇：入门篇（零基础启动）

## 第 1 章 认识 Hermes Agent

### 1.1 什么是 Hermes Agent

Hermes Agent 是一个**自我改进的 AI 代理**。它不只是和 LLM 对话——它能：

- **执行工具**：读写文件、运行终端命令、浏览网页、操作浏览器
- **自我学习**：自动从经验中创建技能（Skills），跨会话记忆
- **多平台接入**：通过 Telegram、Discord、Slack 等 20+ 消息平台与它交互
- **调度自动化**：内置 cron 调度器，定时执行任务
- **并行委托**：生成隔离子代理处理并行工作流

### 1.2 核心概念速览

| 概念 | 说明 |
|------|------|
| **Agent** | AI 代理，核心对话循环，位于 [run_agent.py](file:///d:/project/test/hermes-agent/run_agent.py) 的 `AIAgent` 类 |
| **Tool** | 工具，Agent 可调用的函数（如 `read_file`、`terminal`、`web_search`） |
| **Toolset** | 工具集，工具的逻辑分组（如 `terminal`、`web`、`browser`） |
| **Skill** | 技能，Markdown 文档形式的可复用知识（`SKILL.md`） |
| **Session** | 会话，一次连续对话，存储在 SQLite 中 |
| **Gateway** | 网关，连接消息平台和 Agent 的桥梁 |
| **Profile** | 配置文件，多个完全隔离的 Hermes 实例 |
| **HERMES_HOME** | Hermes 的主目录，默认 `~/.hermes/` |

### 1.3 技术栈

```
后端：     Python 3.11+（最高 3.13）
TUI：      TypeScript + Ink（React 终端框架）
桌面应用： Electron + React
Web 控制台：React + Vite
文档站点： Docusaurus
打包：     setuptools + uv
```

---

## 第 2 章 安装与初次使用

### 2.1 安装

#### Linux / macOS / WSL2

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

#### Windows 原生（PowerShell）

```powershell
iex (irm https://hermes-agent.nousresearch.com/install.ps1)
```

安装器自动处理：`uv`（Python 包管理器）、Python 3.11、Node.js、ripgrep、ffmpeg，以及一个便携的 Git Bash。

#### 安装后

```bash
source ~/.bashrc        # 重新加载 shell
hermes                  # 启动！
```

### 2.2 首次运行

```bash
hermes              # 交互式 CLI 对话
```

首次运行会引导你选择 LLM 提供商和模型。最简单的方式是使用 Nous Portal：

```bash
hermes setup --portal    # 通过 OAuth 一键配置
```

### 2.3 验证安装

```bash
hermes doctor       # 检查配置和依赖
hermes status       # 查看所有组件状态
hermes version      # 查看版本
```

### 2.4 基本对话

启动 `hermes` 后进入交互模式：

```
> 你好，请介绍一下你自己
> 帮我在当前目录创建一个 hello.py 文件
> /new          # 开始新对话
> /model        # 切换模型
> /quit         # 退出
```

---

## 第 3 章 目录结构认知

### 3.1 HERMES_HOME 目录

```
~/.hermes/
├── config.yaml          # 主配置文件
├── .env                 # API 密钥（仅密钥！）
├── logs/                # 日志目录
│   ├── agent.log        # INFO+ 级别
│   ├── errors.log       # WARNING+ 级别
│   └── gateway.log      # 网关日志
├── sessions/            # 会话数据库（SQLite）
├── skills/              # 用户技能
├── SOUL.md              # Agent 人格定义
├── USER.md              # 用户画像
├── MEMORY.md            # 持久记忆
└── plugins/             # 用户插件
```

### 3.2 项目源码目录

```
hermes-agent/
├── run_agent.py            # AIAgent 类——核心对话循环
├── model_tools.py          # 工具编排层
├── toolsets.py             # 工具集定义
├── cli.py                  # 交互式 CLI 编排器
├── hermes_state.py         # SQLite 会话存储
├── hermes_constants.py     # 路径工具
├── agent/                  # Agent 内部模块
│   ├── prompt_builder.py   # 系统提示构建
│   ├── context_compressor.py # 上下文压缩
│   ├── memory_manager.py   # 记忆管理
│   └── ...
├── tools/                  # 工具实现
│   ├── registry.py         # 工具注册中心
│   ├── file_tools.py       # 文件操作
│   ├── terminal_tool.py    # 终端执行
│   ├── web_tools.py        # 网络搜索
│   ├── browser_tool.py     # 浏览器自动化
│   ├── delegate_tool.py    # 子代理委托
│   └── environments/       # 终端后端
├── hermes_cli/             # CLI 子命令
├── gateway/                # 消息网关
│   ├── run.py              # 网关运行器
│   ├── session.py          # 会话管理
│   └── platforms/          # 各平台适配器
├── skills/                 # 内置技能
├── plugins/                # 插件系统
└── tests/                  # 测试套件
```

---

# 第二篇：基础篇（日常使用）

## 第 4 章 配置系统

### 4.1 两个配置文件

| 文件 | 用途 | 位置 |
|------|------|------|
| `config.yaml` | 所有设置（模型、工具、显示等） | `~/.hermes/config.yaml` |
| `.env` | **仅** API 密钥、token、密码 | `~/.hermes/.env` |

> **重要**：不要把 API 密钥放到 `config.yaml`，不要把配置项放到 `.env`。

### 4.2 config.yaml 主要章节

```yaml
model:
  provider: openrouter           # LLM 提供商
  name: anthropic/claude-3.5-sonnet  # 模型名

terminal:
  backend: local                 # local | docker | ssh | modal | daytona
  cwd: .
  timeout: 60

compression:
  enabled: true                  # 上下文压缩
  threshold: 0.8                 # 80% 时触发

display:
  skin: default                  # default | ares | mono | slate
  interface: cli                 # cli | tui

memory:
  provider: local                # local | honcho | mem0 | ...

delegation:
  max_concurrent_children: 3     # 并行子代理上限
  max_spawn_depth: 2             # 委托嵌套深度

curator:
  enabled: true                  # 技能自动维护
  interval_hours: 6              # 检查间隔
```

### 4.3 配置命令

```bash
hermes config          # 显示当前配置
hermes config edit     # 在编辑器中打开
hermes config set      # 设置特定值
hermes config wizard   # 重新运行设置向导
```

### 4.4 .env 变量示例

```bash
# LLM 提供商密钥
OPENROUTER_API_KEY=sk-or-v1-xxxxx
GOOGLE_API_KEY=AIzaxxxxx

# 工具密钥
EXA_API_KEY=xxxxx              # Exa 网络搜索
FAL_KEY=xxxxx                  # FAL.ai 图像生成

# 消息平台
TELEGRAM_BOT_TOKEN=123456:ABC-DEF
TELEGRAM_ALLOWED_USERS=123456789,987654321
```

### 4.5 三种配置加载器

理解这一点对后续开发很重要：

| 加载器 | 使用场景 | 源码位置 |
|--------|---------|---------|
| `load_cli_config()` | CLI 交互模式 | [cli.py](file:///d:/project/test/hermes-agent/cli.py) |
| `load_config()` | `hermes tools`、`hermes setup` 等子命令 | [hermes_cli/config.py](file:///d:/project/test/hermes-agent/hermes_cli/config.py) |
| 直接 YAML 读取 | 网关运行时 | [gateway/run.py](file:///d:/project/test/hermes-agent/gateway/run.py) |

---

## 第 5 章 CLI 交互指南

### 5.1 主要命令

```bash
hermes              # 交互式对话（默认）
hermes chat         # 同上
hermes --tui        # 使用 TUI 界面（Ink/React）
hermes gateway      # 启动消息网关
hermes setup        # 设置向导
hermes tools        # 配置工具
hermes model        # 选择模型
hermes doctor       # 诊断问题
hermes status       # 组件状态
hermes cron list    # 定时任务
hermes update       # 更新
hermes sessions browse  # 会话浏览器
```

### 5.2 会话内斜杠命令

在对话中输入 `/` 开头的命令：

| 命令 | 说明 |
|------|------|
| `/new` 或 `/reset` | 开始新对话 |
| `/model [provider:model]` | 切换模型 |
| `/personality [name]` | 设置人格 |
| `/retry` | 重试上一次 |
| `/undo` | 撤销上一次 |
| `/compress` | 手动压缩上下文 |
| `/usage` | 查看 token 使用量 |
| `/skills` | 浏览可用技能 |
| `/skin [name]` | 切换皮肤 |
| `/cron` | 管理定时任务 |
| `/insights` | 查看洞察 |
| `/quit` | 退出 |

### 5.3 单次查询模式

```bash
hermes -c "用 Python 写一个快速排序"
hermes --query "解释这段代码的作用"
```

### 5.4 恢复会话

```bash
hermes --resume           # 恢复最近的会话
hermes sessions browse    # 交互式选择历史会话
```

---

## 第 6 章 工具系统使用

### 6.1 什么是工具

工具是 Agent 可以调用的函数。例如：

```
用户：帮我读取 config.yaml 文件
Agent → 调用 read_file(path="config.yaml")
工具返回文件内容
Agent → 总结内容返回给用户
```

### 6.2 查看与管理工具

```bash
hermes tools              # curses UI 管理工具（推荐）
```

工具按**工具集（Toolset）**组织。每个平台（CLI、Telegram、Discord 等）可以选择启用/禁用哪些工具集。

### 6.3 核心工具一览

| 工具集 | 工具 | 说明 |
|--------|------|------|
| `terminal` | `terminal`, `process` | 执行终端命令、管理进程 |
| `file` | `read_file`, `write_file`, `patch`, `search_files` | 文件读写、补丁、搜索 |
| `web` | `web_search`, `web_extract` | 网络搜索、网页提取 |
| `browser` | `browser_navigate`, `browser_click`, ... | 浏览器自动化 |
| `vision` | `vision_analyze` | 图像分析 |
| `image_gen` | `image_generate` | 图像生成 |
| `skills` | `skills_list`, `skill_view`, `skill_manage` | 技能管理 |
| `todo` | `todo` | 任务规划 |
| `memory` | `memory` | 持久记忆 |
| `delegation` | `delegate_task` | 子代理委托 |
| `code_execution` | `execute_code` | Python 脚本执行 |
| `cronjob` | `cronjob` | 定时任务管理 |
| `tts` | `text_to_speech` | 文字转语音 |
| `session_search` | `session_search` | 历史会话搜索 |
| `clarify` | `clarify` | 向用户提问 |
| `kanban` | `kanban_show`, `kanban_list`, ... | 多代理看板 |

### 6.4 工具的安全机制

- **危险命令审批**：`terminal` 工具执行危险命令前会请求用户批准
- **工具可用性检查**：`check_fn` 在运行时检查依赖是否满足（如 API 密钥是否配置）
- **Webhook 安全工具集**：来自不可信源的消息只能使用受限工具集

---

## 第 7 章 技能系统

### 7.1 什么是技能

技能（Skill）是 Markdown 文档形式的可复用知识。它告诉 Agent 如何完成特定任务。

```
skills/
├── github/
│   └── SKILL.md          # 如何使用 GitHub
├── mlops/
│   └── SKILL.md          # 机器学习运维指南
└── my-custom-skill/
    ├── SKILL.md          # 主文档
    ├── references/       # 参考资料
    ├── templates/        # 模板
    └── scripts/          # 脚本
```

### 7.2 使用技能

在对话中：

```
> /skills                    # 列出所有技能
> /github                    # 加载 github 技能
> 帮我创建一个 PR             # Agent 使用技能中的知识
```

### 7.3 安装技能

```bash
hermes skills search <关键词>      # 搜索技能
hermes skills install <name>       # 安装技能
hermes skills publish              # 发布自己的技能
```

### 7.4 创建技能

创建 `~/.hermes/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: 简短描述（≤60字符，单句，以句号结尾）
version: 1.0.0
author: Your Name
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [category1, category2]
    category: productivity
---

# My Skill

## When to Use
描述何时使用此技能。

## Prerequisites
前置条件。

## How to Run
执行步骤。

## Procedure
1. 步骤一
2. 步骤二

## Pitfalls
常见陷阱。

## Verification
如何验证结果。
```

### 7.5 Curator（技能自动维护）

Curator 是后台技能维护系统，自动归档陈旧技能：

```bash
hermes curator status     # 查看状态
hermes curator run        # 立即运行
hermes curator pin <skill>   # 固定技能（豁免自动归档）
```

> 技能不会被删除，只是归档到 `~/.hermes/skills/.archive/`，可以恢复。

---

# 第三篇：进阶篇（深入理解）

## 第 8 章 Agent 核心循环

### 8.1 AIAgent 类

核心位于 [run_agent.py](file:///d:/project/test/hermes-agent/run_agent.py) 的 `AIAgent` 类。

```python
from run_agent import AIAgent

agent = AIAgent(
    base_url="https://openrouter.ai/api/v1",
    api_key="sk-or-v1-xxx",
    model="anthropic/claude-3.5-sonnet",
    max_iterations=90,           # 工具调用迭代上限
    enabled_toolsets=["terminal", "file", "web"],
)

# 简单接口
response = agent.chat("你好")

# 完整接口
result = agent.run_conversation(
    user_message="帮我创建一个文件",
    system_message=None,         # 使用默认系统提示
    conversation_history=None,   # 传入历史消息
)
```

### 8.2 对话循环原理

```
用户消息
    │
    ▼
构建系统提示（身份 + 工具 schema + 技能 + 记忆 + 上下文文件）
    │
    ▼
┌─────────────────────────────────────┐
│  循环（最多 max_iterations 次）       │
│                                     │
│  1. 调用 LLM API（带 tools schema）  │
│  2. LLM 返回响应                     │
│     ├── 有 tool_calls → 执行工具     │
│     │   ├── 并行执行（如果安全）      │
│     │   ├── 审批检查（如果危险）      │
│     │   └── 将结果加入消息历史        │
│     │   → 继续循环                   │
│     └── 无 tool_calls → 最终响应     │
│         → 退出循环                   │
│                                     │
└─────────────────────────────────────┘
    │
    ▼
返回最终响应 + 完整消息历史
```

### 8.3 消息格式

遵循 OpenAI 格式：

```python
messages = [
    {"role": "system", "content": "你是 Hermes Agent..."},
    {"role": "user", "content": "帮我读文件"},
    {"role": "assistant", "content": None, "tool_calls": [
        {"id": "call_xxx", "type": "function",
         "function": {"name": "read_file", "arguments": '{"path": "test.py"}'}}
    ]},
    {"role": "tool", "tool_call_id": "call_xxx", "content": '{"content": "file content..."}'},
    {"role": "assistant", "content": "这是文件内容..."},
]
```

### 8.4 工具调用并行执行

当 LLM 一次返回多个工具调用时，Agent 会判断是否可以并行执行：

```python
# 位于 agent/tool_dispatch_helpers.py
_should_parallelize_tool_batch(tool_calls)
```

判断逻辑：
- 命令是否破坏性（`_is_destructive_command`）
- 文件路径是否重叠（`_paths_overlap`）
- 是否是只读操作

### 8.5 迭代预算

```python
from agent.iteration_budget import IterationBudget

budget = IterationBudget(total=90)
# 每次工具调用消耗 1 点
# 预算耗尽时退出循环
# 有宽限调用（grace call）机制防止突然中断
```

---

## 第 9 章 工具注册系统深入

### 9.1 注册中心架构

工具注册中心位于 [tools/registry.py](file:///d:/project/test/hermes-agent/tools/registry.py)，是一个单例模式：

```python
from tools.registry import registry

# 全局单例
registry = ToolRegistry()
```

### 9.2 工具注册流程

每个工具文件在**模块导入时**自动注册：

```python
# tools/my_tool.py
import json
from tools.registry import registry, tool_error, tool_result

def check_requirements() -> bool:
    """检查依赖是否满足"""
    return bool(os.getenv("MY_API_KEY"))

def my_tool(param: str, task_id: str = None) -> str:
    """工具实现——必须返回 JSON 字符串"""
    try:
        result = do_something(param)
        return tool_result(success=True, data=result)
    except Exception as e:
        return tool_error(str(e))

# 模块级别注册——导入时自动执行
registry.register(
    name="my_tool",
    toolset="my_toolset",
    schema={
        "name": "my_tool",
        "description": "做什么的工具",
        "parameters": {
            "type": "object",
            "properties": {
                "param": {"type": "string", "description": "参数说明"}
            },
            "required": ["param"]
        }
    },
    handler=lambda args, **kw: my_tool(param=args.get("param", "")),
    check_fn=check_requirements,        # 可选：可用性检查
    requires_env=["MY_API_KEY"],        # 可选：所需环境变量
    is_async=False,                     # 可选：是否异步
    emoji="🔧",                         # 可选：显示图标
)
```

### 9.3 工具发现机制

```python
# tools/registry.py
def discover_builtin_tools(tools_dir=None):
    """扫描 tools/ 目录，自动导入包含 registry.register() 调用的模块"""
    tools_path = Path(tools_dir) or Path(__file__).parent
    module_names = [
        f"tools.{path.stem}"
        for path in sorted(tools_path.glob("*.py"))
        if path.name not in {"__init__.py", "registry.py", "mcp_tool.py"}
        and _module_registers_tools(path)  # AST 检查是否有顶层 register 调用
    ]
    for mod_name in module_names:
        importlib.import_module(mod_name)
```

关键点：
- 使用 AST 分析判断文件是否包含 `registry.register()` 调用
- 只检查**模块顶层语句**，不检查函数内的调用
- 导入失败只警告不崩溃

### 9.4 check_fn 缓存机制

`check_fn` 结果会被缓存 30 秒（`_CHECK_FN_TTL_SECONDS`），避免频繁探测外部状态：

```python
# 缓存逻辑
_CHECK_FN_TTL_SECONDS = 30.0           # 正常缓存时间
_CHECK_FN_FAILURE_GRACE_SECONDS = 60.0  # 失败宽限窗口

# 如果最近成功过，短时间内失败被视为瞬时故障
# 返回上次的 True 值，避免工具被误移除
```

### 9.5 工具集组合

工具集可以嵌套组合（位于 [toolsets.py](file:///d:/project/test/hermes-agent/toolsets.py)）：

```python
TOOLSETS = {
    "debugging": {
        "description": "调试工具集",
        "tools": ["terminal", "process"],
        "includes": ["web", "file"]  # 包含其他工具集
    },
    "safe": {
        "description": "安全工具集（无终端）",
        "tools": [],
        "includes": ["web", "vision", "image_gen"]
    }
}

# 递归解析
def resolve_toolset(name, visited=None):
    """递归解析工具集，返回所有工具名"""
    toolset = TOOLSETS.get(name)
    tools = set(toolset.get("tools", []))
    for included in toolset.get("includes", []):
        tools.update(resolve_toolset(included, visited))
    return sorted(tools)
```

### 9.6 平台工具集

每个消息平台有对应的工具集：

```python
_HERMES_CORE_TOOLS = [
    "web_search", "web_extract",
    "terminal", "process",
    "read_file", "write_file", "patch", "search_files",
    "vision_analyze", "image_generate",
    "skills_list", "skill_view", "skill_manage",
    "browser_navigate", "browser_snapshot", ...
]

TOOLSETS = {
    "hermes-cli": {"tools": _HERMES_CORE_TOOLS, ...},
    "hermes-telegram": {"tools": _HERMES_CORE_TOOLS, ...},
    "hermes-discord": {"tools": _HERMES_CORE_TOOLS + ["discord", "discord_admin"], ...},
    # ... 20+ 平台
}
```

### 9.7 dispatch 方法

工具执行的核心入口：

```python
def dispatch(self, name: str, args: dict, **kwargs) -> str:
    """执行工具，返回 JSON 字符串"""
    entry = self.get_entry(name)
    if not entry:
        return json.dumps({"error": f"Unknown tool: {name}"})
    try:
        if entry.is_async:
            return _run_async(entry.handler(args, **kwargs))
        return entry.handler(args, **kwargs)
    except Exception as e:
        return json.dumps({"error": f"Tool execution failed: {e}"})
```

> **关键约束**：所有工具 handler 必须返回 JSON 字符串。

---

## 第 10 章 系统提示构建

### 10.1 提示组装流程

位于 [agent/prompt_builder.py](file:///d:/project/test/hermes-agent/agent/prompt_builder.py)：

```
系统提示 = 身份定义（SOUL.md）
         + 平台提示（来自 GatewayConfig）
         + 工具使用指导
         + 技能索引（skills_list 元数据）
         + 上下文文件（AGENTS.md, .hermes.md, .cursorrules）
         + 环境提示（OS, shell, cwd）
         + 订阅信息
         + 记忆（MEMORY.md, USER.md）
         + 会话上下文（当前平台、连接的平台、交付渠道）
```

### 10.2 上下文文件发现

Agent 会自动发现并加载以下文件：

| 文件 | 说明 |
|------|------|
| `SOUL.md` | Agent 人格定义，位于 HERMES_HOME |
| `AGENTS.md` | 项目级指导，位于工作目录 |
| `.hermes.md` / `HERMES.md` | Hermes 专用项目指导 |
| `.cursorrules` | Cursor 编辑器规则（兼容） |
| `USER.md` | 用户画像 |

搜索顺序：工作目录 → 逐级向上到 git 根目录。

### 10.3 提示注入防护

上下文文件在注入系统提示前会进行安全扫描：

```python
from tools.threat_patterns import scan_for_threats

def _scan_context_content(content: str, filename: str) -> str:
    """扫描上下文文件内容，返回净化后的内容"""
    findings = scan_for_threats(content, scope="context")
    if findings:
        # 阻止加载，返回占位符
        return f"[BLOCKED: {filename} contained potential prompt injection]"
    return content
```

### 10.4 Prompt 缓存

**最重要的性能优化**：系统提示在会话中保持不变，以利用 LLM 提供商的 prompt 缓存。

**禁止的操作**（会破坏缓存）：
- 在会话中改变过去的上下文
- 在会话中切换工具集
- 在会话中重建系统提示

**唯一例外**：上下文压缩（`/compress`）。

---

## 第 11 章 上下文压缩

### 11.1 为什么需要压缩

长对话会超出模型的上下文窗口。压缩系统在接近上限时自动触发。

### 11.2 压缩原理

位于 [agent/context_compressor.py](file:///d:/project/test/hermes-agent/agent/context_compressor.py)：

```
原始消息历史：
[系统提示] [用户1] [助手1] [用户2] [助手2] ... [用户N] [助手N] [用户N+1]
                                              ↑
                                         压缩到这里

压缩后：
[系统提示] [压缩摘要] [用户N] [助手N] [用户N+1]
```

### 11.3 压缩策略

1. **保护头部和尾部**：系统提示和最近的消息不被压缩
2. **工具输出修剪**：先修剪大型工具输出（廉价预处理）
3. **LLM 摘要**：使用辅助模型（便宜/快速）对中间内容摘要
4. **结构化模板**：摘要包含已解决问题、待处理问题、进行中工作
5. **迭代摘要**：多次压缩时保留之前摘要的信息

### 11.4 摘要前缀

摘要消息有明确的前缀，告诉模型这是参考材料而非当前指令：

```
[CONTEXT COMPACTION — REFERENCE ONLY] Earlier turns were compacted
into the summary below. This is a handoff from a previous context
window — treat it as background reference, NOT as active instructions.
...
```

### 11.5 手动压缩

```bash
> /compress      # 手动触发压缩
> /usage         # 查看 token 使用情况
```

---

## 第 12 章 会话存储

### 12.1 SessionDB

位于 [hermes_state.py](file:///d:/project/test/hermes-agent/hermes_state.py)，使用 SQLite + FTS5 全文搜索：

```python
class SessionDB:
    """SQLite 会话存储
    
    特性：
    - WAL 模式：支持并发读取 + 单写入
    - FTS5 虚拟表：跨所有消息的快速文本搜索
    - 压缩触发的会话链：parent_session_id
    - 会话来源标记：'cli', 'telegram', 'discord' 等
    """
```

### 12.2 会话结构

```
sessions/
├── sessions.db          # SQLite 数据库
│   ├── sessions 表       # 会话元数据
│   ├── messages 表       # 消息历史
│   └── messages_fts 表   # FTS5 全文搜索索引
└── sessions.json        # 网关会话映射（内存快照）
```

### 12.3 会话生命周期

```
创建 → 活跃 → [压缩] → [重置] → 结束
                ↓
          新会话（parent_session_id 链接）
```

### 12.4 会话搜索

```bash
# CLI 中
> /search 关键词

# 命令行
hermes sessions browse    # 交互式浏览器
```

---

## 第 13 章 记忆系统

### 13.1 三层记忆

| 层 | 文件 | 说明 |
|---|------|------|
| **持久记忆** | `MEMORY.md` | Agent 主动记录的笔记 |
| **用户画像** | `USER.md` | 关于用户的信息 |
| **会话记忆** | SessionDB | 当前会话的完整历史 |

### 13.2 记忆工具

Agent 通过 `memory` 工具读写记忆：

```
用户：记住我喜欢用 Python 3.12
Agent → 调用 memory(action="append", content="用户偏好 Python 3.12")
→ 写入 MEMORY.md
```

### 13.3 记忆提供商

通过 [agent/memory_manager.py](file:///d:/project/test/hermes-agent/agent/memory_manager.py) 编排：

| 提供商 | 说明 |
|--------|------|
| `local` | 本地文件（默认） |
| `honcho` | 跨会话辩证用户建模 |
| `mem0` | Mem0 记忆平台 |
| `supermemory` | Supermemory |
| `byterover` | Byterover |

### 13.4 Honcho 集成

```bash
hermes honcho setup              # 配置 Honcho
hermes honcho mode hybrid        # 混合模式（默认）
hermes honcho peer --user Alice  # 设置用户名
hermes honcho tokens --context 4000  # 上下文 token 上限
```

---

# 第四篇：高级篇（平台与自动化）

## 第 14 章 消息网关

### 14.1 网关架构

```
                    ┌─────────────┐
  Telegram  ────────│             │
  Discord   ────────│  Gateway    │──── AIAgent ──── LLM API
  Slack     ────────│  Runner     │
  WhatsApp  ────────│             │
  Email     ────────│             │
  ...       ────────│             │
                    └─────────────┘
```

### 14.2 启动网关

```bash
hermes gateway setup     # 配置平台
hermes gateway start     # 启动为后台服务
hermes gateway stop      # 停止
hermes gateway status    # 查看状态
hermes gateway           # 前台运行（调试用）
hermes gateway install   # 安装为系统服务
```

### 14.3 支持的平台

| 平台 | 配置变量 |
|------|---------|
| Telegram | `TELEGRAM_BOT_TOKEN` + `TELEGRAM_ALLOWED_USERS` |
| Discord | （通过 messaging extra） |
| Slack | `SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` |
| WhatsApp | `WHATSAPP_ENABLED` |
| Signal | （通过 platforms/signal.py） |
| Email | `EMAIL_ADDRESS` + `EMAIL_PASSWORD` |
| Microsoft Teams | `TEAMS_CLIENT_ID` + `TEAMS_CLIENT_SECRET` |
| Google Chat | `GOOGLE_CHAT_PROJECT_ID` |
| Matrix | （端到端加密） |
| DingTalk / WeCom / WeChat / Feishu / QQbot | 各自配置 |
| Webhook | 通用 HTTP 端点 |

### 14.4 会话管理

网关的会话系统位于 [gateway/session.py](file:///d:/project/test/hermes-agent/gateway/session.py)：

```
SessionSource（消息来源）
    │
    ▼
build_session_key() 生成会话密钥
    │
    ▼
SessionStore.get_or_create_session()
    │
    ├── 会话已存在 → 返回现有会话
    ├── 会话已挂起 → 自动重置
    ├── 会话待恢复 → 保留 session_id
    └── 策略要求重置 → 创建新会话
```

### 14.5 会话密钥格式

```
agent:main:{platform}:{chat_type}[:{chat_id}][:{thread_id}][:{participant_id}]

示例：
agent:main:telegram:dm:12345           # Telegram 私聊
agent:main:telegram:group:-10012345     # Telegram 群组
agent:main:discord:group:12345:user_abc # Discord 群组（按用户隔离）
```

### 14.6 重启恢复

网关崩溃后能恢复进行中的会话：

1. 检查 `.clean_shutdown` 标记
2. 如果不存在（崩溃）：`suspend_recently_active()` 标记最近活跃的会话为待恢复
3. 卡循环检测：连续 3 次重启仍活跃的会话被自动挂起
4. 排水超时标记：被中断的会话标记为待恢复

### 14.7 后台进程通知

```yaml
display.background_process_notifications:
  # all    — 运行输出 + 最终消息（默认）
  # result — 仅最终完成消息
  # error  — 仅退出码 != 0 的最终消息
  # off    — 无通知
```

---

## 第 15 章 定时任务（Cron）

### 15.1 创建任务

```bash
hermes cron add
# 或在对话中：
> /cron
```

### 15.2 调度格式

```bash
# 持续时间
"30m"           # 每 30 分钟
"2h"            # 每 2 小时
"1d"            # 每天

# "every" 短语
"every 2h"
"every monday 9am"

# 5 字段 cron 表达式
"0 9 * * *"     # 每天 9 点

# 一次性 ISO 时间戳
"2026-06-01T09:00:00Z"
```

### 15.3 任务字段

```yaml
- name: "每日报告"
  schedule: "0 9 * * *"
  prompt: "生成本周工作总结"
  skills: ["github"]           # 加载特定技能
  model: "anthropic/claude-3.5-sonnet"  # 模型覆盖
  platforms:                    # 交付到多个平台
    - telegram:12345
    - discord:67890
  workdir: "/home/user/project"  # 工作目录
  context_from: "task-a"        # 链式：使用任务 A 的输出
```

### 15.4 Chronos 托管 Cron

对于托管部署，Hermes 支持**缩容到零**的 cron：

```
agent 请求 NAS 布防一次性触发
    → NAS 在触发时间回调 agent
    → agent 执行任务
    → agent 重新布防下一个触发
```

见 [docs_CN/chronos-managed-cron-contract.md](file:///d:/project/test/hermes-agent/docs_CN/chronos-managed-cron-contract.md)。

---

## 第 16 章 委托与并行子代理

### 16.1 委托机制

位于 [tools/delegate_tool.py](file:///d:/project/test/hermes-agent/tools/delegate_tool.py)：

```python
# 单任务委托
delegate_task(
    goal="分析这个 GitHub 仓库的代码质量",
    context="仓库地址：https://github.com/...",
    toolsets=["web", "terminal", "file"]
)

# 批量并行委托
delegate_task(
    tasks=[
        {"goal": "搜索相关论文", "toolsets": ["web"]},
        {"goal": "分析本地数据", "toolsets": ["file", "terminal"]},
        {"goal": "生成可视化", "toolsets": ["code_execution"]},
    ]
)
```

### 16.2 子代理隔离

每个子代理获得：
- **全新的对话**（无父代理历史）
- **独立的 task_id**（独立的终端会话、文件操作缓存）
- **受限的工具集**（可配置）
- **聚焦的系统提示**（基于委托目标）

### 16.3 被阻止的工具

子代理**永远不能**访问的工具：

```python
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",   # 不能递归委托
    "clarify",         # 不能与用户交互
    "memory",          # 不能写共享记忆
    "send_message",    # 不能跨平台发消息
    "execute_code",    # 应该逐步推理
    "cronjob",         # 不能调度更多任务
])
```

### 16.4 角色

| 角色 | 能力 |
|------|------|
| `leaf`（默认） | 聚焦工作者，不能委托 |
| `orchestrator` | 可以生成自己的工作者（受 `max_spawn_depth` 限制） |

### 16.5 配置

```yaml
delegation:
  max_concurrent_children: 3    # 并行子代理上限
  max_spawn_depth: 2            # 委托嵌套深度
  orchestrator_enabled: true    # 允许编排者角色
  subagent_auto_approve: false  # 子代理自动批准危险命令
```

---

## 第 17 章 看板系统（Kanban）

### 17.1 概念

持久 SQLite 支持的看板，允许多个 profile/worker 协作。

### 17.2 CLI 命令

```bash
hermes kanban init           # 初始化
hermes kanban create         # 创建任务
hermes kanban list           # 列出任务
hermes kanban show <id>      # 查看详情
hermes kanban assign <id>    # 分配
hermes kanban complete <id>  # 完成
hermes kanban block <id>     # 阻塞
hermes kanban comment <id>   # 评论
hermes kanban archive <id>   # 归档
hermes kanban watch          # 监视
hermes kanban stats          # 统计
```

### 17.3 多网关部署

```yaml
# 在拥有 dispatch 职责的 gateway 上（通常是 default profile）
kanban:
  dispatch_in_gateway: true   # 默认

# 在其余 profile 的 gateway 上
kanban:
  dispatch_in_gateway: false
```

---

## 第 18 章 终端后端

### 18.1 六种后端

| 后端 | 配置值 | 适用场景 |
|------|--------|---------|
| 本地 | `local` | 本机开发 |
| Docker | `docker` | 容器隔离 |
| SSH | `ssh` | 远程服务器（安全沙箱） |
| Singularity | `singularity` | HPC 环境 |
| Modal | `modal` | 无服务器，空闲休眠 |
| Daytona | `daytona` | 无服务器 |

### 18.2 配置

```yaml
terminal:
  backend: ssh
  cwd: /home/user/workspace
  timeout: 120
```

或环境变量：

```bash
TERMINAL_ENV=ssh
TERMINAL_SSH_HOST=user@server
TERMINAL_SSH_KEY=~/.ssh/id_rsa
```

### 18.3 SSH 后端的安全优势

- 代理无法读取 `.env` 文件（API 密钥受保护）
- 代理无法修改自身代码
- 远程服务器作为隔离沙箱
- 可安全配置无密码 sudo

---

# 第五篇：精通篇（架构与扩展）

## 第 19 章 插件系统

### 19.1 插件架构

位于 [hermes_cli/plugins.py](file:///d:/project/test/hermes-agent/hermes_cli/plugins.py)：

```
PluginManager
    ├── 从 ~/.hermes/plugins/ 发现
    ├── 从 ./.hermes/plugins/ 发现
    └── 从 pip entry points 发现
```

### 19.2 插件能力

```python
def register(ctx: PluginContext):
    # 1. 注册工具
    ctx.register_tool(name="my_tool", schema={...}, handler=...)

    # 2. 注册 CLI 子命令
    ctx.register_cli_command(name="mycommand", ...)

    # 3. 生命周期钩子（观察者）
    ctx.register_hook("pre_tool_call", on_pre_tool_call)
    ctx.register_hook("post_tool_call", on_post_tool_call)
    ctx.register_hook("pre_api_request", on_pre_api_request)
    ctx.register_hook("post_api_request", on_post_api_request)
    ctx.register_hook("on_session_start", on_session_start)

    # 4. 中间件（行为修改）
    ctx.register_middleware("llm_request", on_llm_request)
    ctx.register_middleware("tool_request", on_tool_request)
    ctx.register_middleware("llm_execution", on_llm_execution)
    ctx.register_middleware("tool_execution", on_tool_execution)
```

### 19.3 观察者钩子 vs 中间件

| 特性 | 观察者钩子 | 中间件 |
|------|-----------|--------|
| 能否改变行为 | 不能（只读） | 能 |
| 用途 | 遥测、日志、审计 | 请求改写、执行包装 |
| 返回值 | 大多被忽略 | 可以替换请求/结果 |

### 19.4 编写插件

```python
# ~/.hermes/plugins/myplugin/__init__.py

def register(ctx):
    # 注册一个工具
    ctx.register_tool(
        name="my_tool",
        toolset="my_toolset",
        schema={
            "name": "my_tool",
            "description": "我的工具",
            "parameters": {"type": "object", "properties": {}}
        },
        handler=lambda args, **kw: '{"success": true}',
    )

    # 注册观察者
    @ctx.hook("post_tool_call")
    def log_tool_call(**kwargs):
        print(f"工具 {kwargs.get('tool_name')} 完成，状态：{kwargs.get('status')}")

    # 注册中间件
    ctx.register_middleware("llm_request", tag_requests)

def tag_requests(**kwargs):
    request = dict(kwargs["request"])
    request.setdefault("extra_body", {}).setdefault("metadata", {})["plugin"] = "myplugin"
    return {"request": request, "source": "myplugin", "reason": "tagged"}
```

### 19.5 插件目录

| 目录 | 用途 |
|------|------|
| `plugins/memory/` | 记忆提供商插件 |
| `plugins/context_engine/` | 上下文引擎插件 |
| `plugins/model-providers/` | 推理后端插件 |
| `plugins/kanban/` | 多代理看板 |
| `plugins/observability/` | 可观测性（Langfuse、NeMo Relay） |
| `plugins/image_gen/` | 图像生成提供商 |
| `plugins/web/` | 网络搜索提供商 |

### 19.6 重要政策

- **插件不得修改核心文件**（`run_agent.py`、`cli.py` 等）
- **不再接受新的内置记忆提供商**——必须作为独立插件仓库
- **不再接受第三方产品插件**——observability、SaaS 连接器等必须独立发布

---

## 第 20 章 MCP 服务器集成

### 20.1 什么是 MCP

MCP（Model Context Protocol）是一个开放标准，允许 Agent 连接外部工具服务器。

### 20.2 配置 MCP 服务器

```yaml
# config.yaml
mcp_servers:
  filesystem:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"]
  
  github:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_TOKEN: ghp_xxxxx
```

### 20.3 MCP 工具动态发现

MCP 服务器的工具在运行时动态注册到工具注册中心：

```python
# tools/mcp_tool.py
# MCP 工具集前缀为 "mcp-"
# 支持动态刷新（notifications/tools/list_changed）
# 工具注册时使用 mcp- 前缀，豁免覆盖检查
```

---

## 第 21 章 Profile 多实例管理

### 21.1 概念

Profile 是完全隔离的 Hermes 实例，每个有自己的 HERMES_HOME：

```
~/.hermes/                    # 默认 profile
~/.hermes/profiles/coder/     # coder profile
~/.hermes/profiles/writer/    # writer profile
~/.hermes/profiles/admin/     # admin profile
```

### 21.2 使用 Profile

```bash
hermes -p coder           # 使用 coder profile
hermes -p coder profile list   # 列出所有 profile
```

### 21.3 代码规范

**必须使用 `get_hermes_home()`**，不要硬编码路径：

```python
# 正确
from hermes_constants import get_hermes_home, display_hermes_home
config_path = get_hermes_home() / "config.yaml"
print(f"配置保存到 {display_hermes_home()}/config.yaml")

# 错误——破坏 profile 隔离
config_path = Path.home() / ".hermes" / "config.yaml"
```

---

## 第 22 章 皮肤系统

### 22.1 内置皮肤

| 皮肤 | 风格 |
|------|------|
| `default` | 经典 Hermes 金色 |
| `ares` | 深红/青铜战神主题 |
| `mono` | 干净灰度单色 |
| `slate` | 冷蓝色开发者主题 |

### 22.2 自定义皮肤

创建 `~/.hermes/skins/my-theme.yaml`：

```yaml
name: my-theme
description: 我的自定义主题

colors:
  banner_border: "#FF00FF"
  banner_title: "#00FFFF"
  response_border: "#FF1493"

spinner:
  thinking_verbs: ["思考中", "处理中", "计算中"]
  wings:
    - ["⟨⚡", "⚡⟩"]

branding:
  agent_name: "我的助手"
  response_label: " ⚡ 助手 "

tool_emojis:
  terminal: "💻"
  read_file: "📄"
  web_search: "🔍"
```

激活：`/skin my-theme` 或在 config.yaml 设置 `display.skin: my-theme`。

---

## 第 23 章 TUI 与桌面应用

### 23.1 TUI（Ink/React）

```bash
hermes --tui        # 强制使用 TUI
hermes --cli        # 强制使用 CLI
```

进程模型：

```
hermes --tui
  └─ Node (Ink)  ──stdio JSON-RPC──  Python (tui_gateway)
       │                                  └─ AIAgent + tools + sessions
       └─ 渲染转录、输入、提示、活动
```

### 23.2 桌面应用

Electron + React 应用，通过 JSON-RPC 与 `tui_gateway` 通信：

```bash
cd apps/desktop
npm install
npm run dev       # 开发模式
npm run build     # 构建
```

### 23.3 Web Dashboard

```bash
hermes dashboard    # 启动 Web 控制台
```

嵌入式 TUI——通过 xterm.js 在浏览器中渲染真实的 `hermes --tui`。

---

## 第 24 章 ACP 编辑器集成

### 24.1 什么是 ACP

ACP（Agent Communication Protocol）允许 Hermes 集成到编辑器中：

```bash
hermes acp    # 作为 ACP 服务器运行
```

支持：VS Code、Zed、JetBrains。

### 24.2 工具集

ACP 使用 `hermes-acp` 工具集——编码聚焦，无消息/音频/clarify UI 工具。

---

## 第 25 章 安全模型

### 25.1 信任边界

```
┌──────────────────────────────────────────────┐
│  受信任区域（HERMES_HOME）                     │
│  ├── .env（API 密钥）                         │
│  ├── config.yaml                              │
│  └── SOUL.md                                  │
├──────────────────────────────────────────────┤
│  Agent 执行区域                                │
│  ├── 终端后端（local/docker/ssh）              │
│  ├── 工具执行                                  │
│  └── 文件操作                                  │
├──────────────────────────────────────────────┤
│  不可信外部区域                                │
│  ├── 网页内容                                  │
│  ├── 消息平台输入                              │
│  └── 工具返回结果                              │
└──────────────────────────────────────────────┘
```

### 25.2 提示注入防护

- **上下文文件扫描**：`AGENTS.md`、`.cursorrules` 等在注入前扫描
- **工具结果扫描**：工具返回的内容扫描威胁模式
- **Webhook 安全工具集**：来自不可信源的消息使用受限工具

### 25.3 网络出口隔离

对于 Docker 部署，可以隔离 Agent 的网络访问：

```yaml
# docker-compose.override.yml
networks:
  internal:
    driver: bridge
    internal: true          # 无互联网
  egress:
    driver: bridge

services:
  gateway:
    networks: [internal, egress]
    environment:
      - HTTP_PROXY=http://egress-proxy:3128
```

见 [docs_CN/security/network-egress-isolation.md](file:///d:/project/test/hermes-agent/docs_CN/security/network-egress-isolation.md)。

---

## 第 26 章 开发与贡献

### 26.1 开发环境

```bash
# 使用 uv
curl -LsSf https://astral.sh/uv/install.sh | sh
uv venv ~/.hermes/venvs/hermes-dev --python 3.11
source ~/.hermes/venvs/hermes-dev/bin/activate
uv pip install -e ".[all,dev]"
```

### 26.2 测试

**始终使用 `scripts/run_tests.sh`**：

```bash
scripts/run_tests.sh                                  # 完整套件
scripts/run_tests.sh tests/gateway/                   # 一个目录
scripts/run_tests.sh tests/agent/test_foo.py::test_x  # 一个测试
```

每个测试文件在独立子进程中运行（subprocess-per-test-file 隔离）。

### 26.3 测试规则

- **不要写变化检测器测试**——不要断言会随数据更新失败的快照
- 写**行为契约**测试，而非快照
- 测试不得写入 `~/.hermes/`（自动重定向到临时目录）

### 26.4 添加新能力——Footprint Ladder

选择**最高（最小足迹）**能解决问题的层级：

1. **扩展现有代码**——零新表面
2. **CLI 命令 + 技能**——零模型工具足迹
3. **服务门控工具（`check_fn`）**——前置条件未配置时零足迹
4. **插件**——在 `~/.hermes/plugins/` 中
5. **MCP 服务器**——零永久核心 schema 足迹
6. **新核心工具**——最后手段

### 26.5 添加新工具

```python
# 1. 创建 tools/my_tool.py
import json
from tools.registry import registry, tool_error, tool_result

def check_requirements() -> bool:
    return bool(os.getenv("MY_API_KEY"))

def my_tool(param: str, task_id: str = None) -> str:
    return tool_result(success=True, data="result")

registry.register(
    name="my_tool",
    toolset="my_toolset",
    schema={"name": "my_tool", "description": "...", "parameters": {...}},
    handler=lambda args, **kw: my_tool(param=args.get("param", "")),
    check_fn=check_requirements,
    requires_env=["MY_API_KEY"],
)

# 2. 在 toolsets.py 注册
TOOLSETS["my_toolset"] = {
    "description": "我的工具集",
    "tools": ["my_tool"],
    "includes": []
}

# 3. 添加到 _HERMES_CORE_TOOLS（如果应该是核心工具）
```

### 26.6 添加斜杠命令

1. 在 [hermes_cli/commands.py](file:///d:/project/test/hermes-agent/hermes_cli/commands.py) 添加：

```python
CommandDef("mycommand", "描述", "Session",
           aliases=("mc",), args_hint="[arg]")
```

2. 在 [cli.py](file:///d:/project/test/hermes-agent/cli.py) 的 `process_command()` 添加处理逻辑

3. 如需在网关使用，在 [gateway/run.py](file:///d:/project/test/hermes-agent/gateway/run.py) 添加处理

### 26.7 添加配置项

1. 添加到 [hermes_cli/config.py](file:///d:/project/test/hermes-agent/hermes_cli/config.py) 的 `DEFAULT_CONFIG`
2. 新增密钥添加到 `OPTIONAL_ENV_VARS`：

```python
"NEW_API_KEY": {
    "description": "用途说明",
    "prompt": "显示名",
    "url": "https://...",
    "password": True,
    "category": "tool",
}
```

### 26.8 已知陷阱

- **不要硬编码 `~/.hermes` 路径**——使用 `get_hermes_home()`
- **不要使用 `simple_term_menu`**——使用 `hermes_cli/curses_ui.py`
- **不要在 spinner 代码中使用 `\033[K`**——在 prompt_toolkit 下泄漏
- **`_last_resolved_tool_names` 是进程全局**——委托时需保存/恢复
- **不要在 schema 描述中硬编码跨工具引用**

### 26.9 依赖固定政策

所有依赖必须有上界：

```toml
# pyproject.toml
dependencies = [
    "httpx>=0.28.1,<1",        # PyPI: >=floor,<next_major
    "openai>=1.0,<2",
]
```

---

## 第 27 章 调试与排查

### 27.1 诊断工具

```bash
hermes doctor       # 检查配置和依赖
hermes status       # 组件状态
hermes logs --follow  # 跟踪日志
```

### 27.2 日志级别

```bash
hermes logs --level WARNING         # 级别过滤
hermes logs --session <session_id>  # 会话过滤
```

日志文件：
- `~/.hermes/logs/agent.log` — INFO+
- `~/.hermes/logs/errors.log` — WARNING+
- `~/.hermes/logs/gateway.log` — 网关日志

### 27.3 常见问题

#### SSL CA 证书问题

```
FileNotFoundError: SSL_CERT_FILE points to a missing CA bundle
```

修复：

```bash
python -m pip install --force-reinstall certifi openai httpx
```

#### Prompt 缓存失效

如果对话成本异常高，检查是否在会话中：
- 改变了过去上下文
- 切换了工具集
- 重建了系统提示

这些都是禁止的。

#### 工具不可用

```bash
hermes tools    # 检查工具状态
hermes doctor   # 检查依赖
```

工具的 `check_fn` 可能返回 False（缺少 API 密钥、Docker 未运行等）。

---

## 第 28 章 学习路径总结

### 28.1 推荐学习顺序

```
入门（1-3 章）
  ├── 安装 Hermes
  ├── 首次对话
  └── 理解目录结构
       │
       ▼
基础（4-7 章）
  ├── 配置系统和 .env
  ├── CLI 交互和斜杠命令
  ├── 工具系统使用
  └── 技能系统
       │
       ▼
进阶（8-13 章）
  ├── Agent 核心循环源码
  ├── 工具注册系统源码
  ├── 系统提示构建
  ├── 上下文压缩
  ├── 会话存储
  └── 记忆系统
       │
       ▼
高级（14-18 章）
  ├── 消息网关
  ├── 定时任务
  ├── 委托与并行子代理
  ├── 看板系统
  └── 终端后端
       │
       ▼
精通（19-27 章）
  ├── 插件系统开发
  ├── MCP 集成
  ├── Profile 管理
  ├── 皮肤定制
  ├── TUI/桌面应用
  ├── 安全模型
  ├── 开发贡献
  └── 调试排查
```

### 28.2 关键源码文件索引

| 文件 | 行数 | 说明 | 学习优先级 |
|------|------|------|-----------|
| [run_agent.py](file:///d:/project/test/hermes-agent/run_agent.py) | ~12k | AIAgent 类，核心对话循环 | ★★★★★ |
| [tools/registry.py](file:///d:/project/test/hermes-agent/tools/registry.py) | ~760 | 工具注册中心 | ★★★★★ |
| [toolsets.py](file:///d:/project/test/hermes-agent/toolsets.py) | ~960 | 工具集定义 | ★★★★ |
| [model_tools.py](file:///d:/project/test/hermes-agent/model_tools.py) | — | 工具编排层 | ★★★★ |
| [cli.py](file:///d:/project/test/hermes-agent/cli.py) | ~11k | CLI 编排器 | ★★★ |
| [gateway/run.py](file:///d:/project/test/hermes-agent/gateway/run.py) | ~16.8k | 网关运行器 | ★★★ |
| [gateway/session.py](file:///d:/project/test/hermes-agent/gateway/session.py) | ~1.4k | 会话管理 | ★★★ |
| [hermes_state.py](file:///d:/project/test/hermes-agent/hermes_state.py) | — | SQLite 会话存储 | ★★★ |
| [agent/prompt_builder.py](file:///d:/project/test/hermes-agent/agent/prompt_builder.py) | — | 系统提示构建 | ★★★★ |
| [agent/context_compressor.py](file:///d:/project/test/hermes-agent/agent/context_compressor.py) | — | 上下文压缩 | ★★★ |
| [tools/delegate_tool.py](file:///d:/project/test/hermes-agent/tools/delegate_tool.py) | — | 子代理委托 | ★★★ |
| [hermes_cli/config.py](file:///d:/project/test/hermes-agent/hermes_cli/config.py) | — | 配置加载 | ★★★ |
| [hermes_cli/commands.py](file:///d:/project/test/hermes-agent/hermes_cli/commands.py) | — | 斜杠命令注册表 | ★★ |
| [hermes_cli/plugins.py](file:///d:/project/test/hermes-agent/hermes_cli/plugins.py) | — | 插件管理器 | ★★★ |

### 28.3 实践项目建议

#### 入门级

1. **配置 Hermes**：安装、配置模型、完成第一次对话
2. **使用工具**：让 Agent 读写文件、执行命令、搜索网页
3. **创建技能**：为自己常用的工作流创建一个 SKILL.md

#### 进阶级

4. **接入 Telegram**：配置网关，通过 Telegram 与 Agent 对话
5. **定时任务**：创建一个每日报告的 cron 任务
6. **使用委托**：让 Agent 并行处理多个子任务
7. **自定义皮肤**：创建符合自己审美的 CLI 主题

#### 高级

8. **开发插件**：编写一个简单的观察者插件记录工具调用
9. **添加工具**：实现一个新的核心工具并注册
10. **MCP 集成**：配置一个 MCP 服务器扩展 Agent 能力
11. **Docker 部署**：使用 Docker + 网络出口隔离部署生产环境

#### 精通

12. **贡献代码**：修复一个 GitHub issue 并提交 PR
13. **多 Profile 部署**：配置多个隔离的 Hermes 实例协同工作
14. **看板多代理**：使用 Kanban 系统编排多个 Agent 协作完成大型项目
15. **自定义终端后端**：实现一个自定义的终端后端

---

## 附录

### A. 在线资源

- **文档站**：https://hermes-agent.nousresearch.com/docs/
- **Discord**：https://discord.gg/NousResearch
- **Skills Hub**：https://agentskills.io
- **GitHub**：https://github.com/NousResearch/hermes-agent
- **Nous Portal**：https://portal.nousresearch.com

### B. 相关文档

- [项目分析文档.md](file:///d:/project/test/hermes-agent/项目分析文档.md) — 项目整体分析
- [docs_CN/](file:///d:/project/test/hermes-agent/docs_CN) — 已翻译的中文技术文档
- [AGENTS.md](file:///d:/project/test/hermes-agent/AGENTS.md) — 开发指南
- [CONTRIBUTING.md](file:///d:/project/test/hermes-agent/CONTRIBUTING.md) — 贡献指南
- [.env.example](file:///d:/project/test/hermes-agent/.env.example) — 环境变量示例

### C. 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| 代理 | Agent | AI 智能体 |
| 工具 | Tool | Agent 可调用的函数 |
| 工具集 | Toolset | 工具的逻辑分组 |
| 技能 | Skill | 可复用的知识文档 |
| 会话 | Session | 一次连续对话 |
| 网关 | Gateway | 消息平台连接器 |
| 配置文件 | Profile | 隔离的 Hermes 实例 |
| 提示注入 | Prompt Injection | 安全攻击手段 |
| 上下文压缩 | Context Compression | 压缩长对话以适应上下文窗口 |
| 委托 | Delegation | 生成子代理处理子任务 |
| 看板 | Kanban | 多代理任务协调系统 |
| 终端后端 | Terminal Backend | 命令执行环境 |
| 插件 | Plugin | 可扩展的功能模块 |
| 中间件 | Middleware | 修改请求/执行行为的插件 |
| 观察者钩子 | Observer Hook | 只读遥测回调 |
| 足迹阶梯 | Footprint Ladder | 新能力决策框架 |

---

*本文档基于 Hermes Agent v0.18.2 源码深度分析整理。如有出入，以 [官方文档](https://hermes-agent.nousresearch.com/docs/) 为准。*
