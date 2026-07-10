# Profile Builder — 仪表盘原生、功能完备的 Profile 创建器

Status: design proposal (not yet implemented)
Author: drafted for Teknium
Supersedes: PR #31781 (prompt_toolkit `hermes profile wizard`)

## 为什么选择这个方案，而不是 CLI 向导

PR #31781 添加了一个键盘驱动的终端版 `hermes profile wizard`。
我们的决定是 **不** 在 CLI 中构建 profile 创建体验。
仪表盘已经为 profile 所需的每个元素都拥有成熟、独立的页面，而一个 profile
本质上就是一个 HERMES_HOME 目录——因此仪表盘才是功能完备 builder 的正确归处，
而且它可以复用所有已经存在的东西。

一个 profile = 一个完整的 `~/.hermes/profiles/<name>/` 目录，包含其自己的：
- `config.yaml` — 保存 `model`/`provider`、`mcp_servers`、已启用的 skills
- `skills/` — 物理的 SKILL.md 文件（内置 seed + optional + hub installs）
- `.env` — 密钥
- `SOUL.md` / `USER.md` — 身份信息

因此，针对 Model、MCPs 和 Skills 的按 profile 作用域隔离是 **原生的** ——
无需任何数据模型变更。缺口纯粹在 UX 层面：如今的创建流程只是一个简陋的模态框
（name + clone + model + description），你只能在 profile 创建 *之后*，
通过访问其他页面并记得去设置作用域，才能组合 skills/MCPs。

## 已存在的能力（复用，不要重建）

| 元素 | 已有页面 | 已有 API | 可按 profile 作用域隔离？ |
|---|---|---|---|
| 名称 / 描述 | ProfilesPage create modal | `POST /api/profiles` (`create_profile`) | 是（通过参数） |
| Model + Provider | ModelsPage | `_write_profile_model(profile_dir, …)` | 是 — HERMES_HOME override，已接入 create endpoint |
| MCPs | McpPage | `mcp_config._save_mcp_server` + `/api/mcp/catalog` | 是 — 通过 HERMES_HOME override 包装 |
| Skills（内置/可选） | SkillsPage | `GET /api/skills`, `/api/skills/toggle` | 是 — 通过写 config |
| Skills（hub） | SkillsPage | `/api/skills/hub/search`, `/api/skills/hub/install` | **只能通过子进程** — 见 seam #1 |

## 在为该设计落地时发现的两个架构接缝

这些是承重项——它们改变的是实现方式，而不仅是润色。

### Seam #1 — hub-skill 安装无法使用 HERMES_HOME override

`tools/skills_hub.py` 在 **模块导入时** 绑定了
`SKILLS_DIR = HERMES_HOME / "skills"`。context-local 的
`set_hermes_home_override()` 交换（它使 `_write_profile_model` 和 MCP 写入落到
目标 profile 中）并 **不会** 回溯性地重新绑定那个已经导入的模块全局变量。
因此，对 hub 安装做一层数据层包装，会写入仪表盘 *自己* 的活动 profile，
而不是新建的那个。

正确的机制是既有的子进程路径：`_spawn_hermes_action` 会运行
`python -m hermes_cli.main <subcommand>`，而 `_apply_profile_override()`
在新的子进程中会在导入时重新读取 `sys.argv`。前置 `-p <profile>`：

```python
_spawn_hermes_action(["-p", profile, "skills", "install", identifier], "skills-install")
```

一个全新的子进程会重新导入 `skills_hub`，从一开始就以该 profile 的
HERMES_HOME 为绑定，所以 `SKILLS_DIR` 解析为 `<profile>/skills/`。
从构造上就是正确的。

### Seam #2 — hub 安装是异步的，所以 create 无法做到完全原子化

内置/可选 skill 的启用以及 MCP 写入都是 **同步的 config 操作**，
可以纳入 create 调用。Hub 安装则是长时间运行的 git fetch，以分离方式派生
（`_spawn_hermes_action` 立即返回一个 PID）。因此 create 流程是：

1. `create_profile()` — 创建目录（同步）
2. 写入 model（同步，HERMES_HOME override）
3. 写入选中的 MCP servers（同步，HERMES_HOME override）
4. seed/启用的选中内置 + 可选 skills（同步）
5. 为每个 hub skill 派生 `hermes -p <profile> skills install <id>`（异步，返回 PIDs）

步骤 1–4 在响应返回前提交；步骤 5 返回一个 action PID 列表，由
UI 轮询（与今天的 SkillsPage hub install 同一模式）。builder 的
"Review → Create" 返回 `{ok, name, path, hub_installs: [{id, pid}]}`，
最终屏幕为 hub skills 显示实时安装进度。

## 提议的后端变更（小改动，遵循既有模式）

扩展 `ProfileCreate` 和 create endpoint —— 不新增 endpoint，不重写：

```python
class ProfileCreate(BaseModel):
    name: str
    clone_from: Optional[str] = None
    # Backward compatibility for older dashboard/desktop clients.
    clone_from_default: bool = False
    clone_all: bool = False
    no_skills: bool = False
    description: Optional[str] = None
    provider: Optional[str] = None
    model: Optional[str] = None
    # NEW — all optional, all best-effort post-create (profile already exists)
    mcp_servers: List[MCPServerCreate] = []      # synchronous, HERMES_HOME override
    builtin_skills: List[str] = []               # synchronous enable/seed
    hub_skills: List[str] = []                   # async spawn, returns PIDs
```

endpoint 已经在做尽力而为的 post-create 步骤（`seed_profile_skills`、
`_write_profile_model`）。按同一风格再追加两个尽力而为的块（MCP 写入、
hub-skill 派生）——其中任何一个失败都不得让 create 返回 500，
因为 profile 目录已经存在，用户事后可以从相关页面修复。为 MCP 写入
镜像 `_write_profile_model` 的 HERMES_HOME-override helper
（`_write_profile_mcp_servers(profile_dir, servers)`）。

## 提议的前端 —— 专用 builder 页面 `/profiles/new`

一个完整页面（而不是拥挤的模态框），分步进行，每一步都复用既有页面的
组件 + API，并指向新 profile：

```
① Identity   名称 + 描述（+ 可选从既有 profile 克隆）
② Model      Provider + model picker（复用 ModelsPage picker）
③ Skills     Tabs：Built-in · Optional · Hub-search
             多选；"Start from default bundle" 预设按钮
④ MCPs       Tabs：Catalog browse · Manual add（复用 McpPage 表单）
⑤ Review     Blueprint 预览 → Create
             → 异步 hub installs 的进度屏幕
```

在到达 ⑤ 之前不向磁盘写入任何内容。

## 待定的产品决策（需要 Teknium）

1. **Skills seeding 默认行为。** 如今新 profile 会自动 seed 默认 bundle。
   在 builder 中，skill 步骤应该 **替换** 该 bundle（精确挑选你想要的；
   提供 "start from default bundle" 预设）还是 **增量追加**？
   建议：替换 + 预设按钮。

2. **页面 vs 更丰富的模态框。** 专用 `/profiles/new` 页面（有成长空间：
   后续 SOUL 编辑、多 agent 集群）vs ProfilesPage 上更大的 create 模态框。
   建议：专用页面 —— 契合 "full-featured / way more options" 的定位。

## 验证计划（实现完成后）

- 后端 E2E，使用隔离的 HERMES_HOME：POST 一个完整的 create body
  （name + model + 2 MCPs + 3 builtin skills + 1 hub skill），断言新
  profile 目录在 config.yaml 中有 model、两个 MCP servers 都在 config.yaml
  中、内置 skills 已启用、并且为 hub skill 派生了一个 PID。负面用例：
  一个错误的 MCP 条目不得让 create 返回 500。
- `cd web && npm run build`（web/ 中没有 JS 测试套件）。
- 定向测试：`pytest tests/<web_server profile tests> -k profile_create`。
