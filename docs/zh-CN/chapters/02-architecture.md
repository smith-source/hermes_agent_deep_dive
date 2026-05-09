# 02 — 系统架构总览: 四层架构与核心模块映射

[上一章](/zh-CN/chapters/01-project-overview) | [下一章](/zh-CN/chapters/03-agent-engine)

---

## 源码位置与关键文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `run_agent.py` | 14,404 | 核心智能体引擎 — AIAgent 类,对话循环,工具调度,流式传输,故障转移 |
| `gateway/run.py` | 15,046 | 消息网关运行器 — GatewayRunner,21+ 平台适配器,会话缓存,生命周期管理 |
| `hermes_cli/main.py` | 10,876 | CLI 入口 — argparse 路由,profile 覆盖,子命令分发,setup/model/tools/doctor |
| `tools/registry.py` | 537 | 工具注册表 — ToolRegistry 单例,ToolEntry 元数据,check_fn TTL 缓存,AST 自发现 |
| `hermes_state.py` | 2,669 | 会话持久化 — SessionDB,SQLite WAL,FTS5 全文搜索,消息存储与检索 |
| `hermes_cli/config.py` | 4,939 | 配置管理 — config.yaml 解析,.env 加载,DEFAULT_CONFIG,凭证路由 |
| `agent/context_compressor.py` | 1,481 | 上下文压缩引擎 — 对话历史压缩,focus_topic 导向压缩,session split |
| `agent/curator.py` | 1,674 | 自审 fork agent — fork 子代理对输出进行自审与改进建议 |
| `agent/prompt_builder.py` | 1,186 | 系统提示词构建 — 动态 system prompt 组装,skill/memory/todo 注入 |
| `agent/tool_guardrails.py` | 455 | 工具调用防护 — 循环检测,渐进式 warn→block→halt 阻断 |
| `agent/error_classifier.py` | 1,036 | API 错误分类 — 18 种 FailoverReason 分类与恢复路径决策 |
| `agent/credential_pool.py` | 1,584 | 凭据池管理 — 4 种轮转策略,exhausted 冷却,OAuth refresh |
| `agent/redact.py` | — | 输出脱敏 — 日志/输出中的 secret redaction 与 sanitization |

**一句话概要:** Hermes Agent 采用四层架构 (前端→CLI→引擎→工具/存储),六核心模块各行其职,引擎层独立于前端运行,网关进程并行服务多平台。agent/ 目录 (54 文件, ~20K 行) 包含引擎层关键支撑子系统。

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Layer 1: Frontend                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Ink/React TUI│  │ curses TUI   │  │ Web Dashboard│  │ ACP (IDE) │ │
│  │ (ui-tui/)    │  │ (classic)    │  │ (web/)       │  │ (VS Code/ │ │
│  │              │  │              │  │              │  │  Zed/JetBrains)│ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘ │
│         │                  │                  │                │       │
├─────────┼──────────────────┼──────────────────┼────────────────┼───────┤
│         ▼                  ▼                  ▼                ▼       │
│                        Layer 2: CLI / Gateway Entry                   │
│  ┌──────────────────┐                    ┌───────────────────────────┐ │
│  │ hermes_cli/main.py│                   │    gateway/run.py         │ │
│  │ (10,876 行)       │                   │    (15,046 行)            │ │
│  │                   │                   │                           │ │
│  │ • argparse 路由   │                   │ • GatewayRunner 类       │ │
│  │ • profile 覆盖   │                   │ • 21+ 平台适配器管理       │ │
│  │ • .env 加载       │                   │ • AIAgent 实例缓存       │ │
│  │ • 子命令分发      │                   │ • 会话生命周期/超时       │ │
│  │ • config 桥接     │                   │ • auto-continue 恢复     │ │
│  └──────┬───────────┘                    └──────────┬────────────────┘ │
│         │                                           │                 │
├─────────┼───────────────────────────────────────────┼─────────────────┤
│         ▼                                           ▼                 │
│                     Layer 3: Agent Engine                            │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                     run_agent.py (14,404 行)                    │ │
│  │                                                                 │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │ │
│  │  │ AIAgent  │  │Iteration │  │ Transport │  │   Context    │  │ │
│  │  │ (L885+)  │  │ Budget   │  │  Layer    │  │  Compressor  │  │ │
│  │  │          │  │ (L272+)  │  │(v0.11+)  │  │  (L9221+)    │  │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │ │
│  │                                                                 │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │ │
│  │  │ Failover │  │ Callback │  │Tool Guard │  │   Memory     │  │ │
│  │  │ (L7633+) │  │ System   │  │ rails     │  │  Manager     │  │ │
│  │  │          │  │(11 cbks) │  │          │  │  (pluggable) │  │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│         │                                                           │
├─────────┼───────────────────────────────────────────────────────────┤
│         ▼                                                           │
│                     Layer 4: Tools / Storage                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │ ToolRegistry │  │  SessionDB   │  │   Config     │             │
│  │ (537 行)     │  │ (2,669 行)   │  │  (4,939 行) │             │
│  │              │  │              │  │              │             │
│  │ • register() │  │ • SQLite WAL │  │ • YAML parse │             │
│  │ • ToolEntry  │  │ • FTS5 search│  │ • .env merge │             │
│  │ • discover   │  │ • msg store  │  │ • DEFAULT_CFG│             │
│  │ • TTL cache  │  │ • session mg │  │ • credential │             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│         │                  │                  │                     │
│         ▼                  ▼                  ▼                     │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │              50+ Tools │ 7 Terminal Backends │ MCP Client      │ │
│  │  Browser │ Vision │ Terminal │ File │ Cron │ Delegate │ Skill │ │
│  │  Search │ Memory │ Patch │ Chat │ Audio │ Compress │ Code     │ │
│  └──────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## TL;DR

Hermes Agent 采用严格的四层架构:前端层 (Ink TUI / curses TUI / Web Dashboard / ACP) 通过 CLI 或网关入口层接入核心引擎层 (`run_agent.py` 的 AIAgent),引擎层调用工具/存储层 (ToolRegistry / SessionDB / Config)。引擎独立于前端运行——CLI 模式下 `hermes_cli/main.py` 构建一个 AIAgent 实例处理单用户交互,网关模式下 `gateway/run.py` 的 GatewayRunner 维护一个 LRU 缓存池为多平台多用户并行服务。六核心模块合计 48,471 行,引擎层单文件 14,404 行承载全部智能体逻辑。

---

## 1. 四层架构详解

### 1.1 Layer 1: Frontend (前端层)

前端层提供用户交互界面,四种前端各自独立,通过不同的入口接入引擎:

| 前端 | 技术栈 | 入口 | 特性 |
|------|--------|------|------|
| Ink/React TUI | React, Ink, TypeScript | `ui-tui/` + `tui_gateway/` (JSON-RPC) | Sticky composer,OSC-92 clipboard,LaTeX,light-theme |
| curses TUI | Python prompt_toolkit, curses | `hermes_cli/main.py` | 经典 CLI,slash commands,autocomplete |
| Web Dashboard | React, Vite | `web/` + `hermes_api_server.py` | i18n,mobile-responsive,plugin tabs,live theme |
| ACP (IDE) | Agent Communication Protocol | `hermes acp` | VS Code, Zed, JetBrains slash commands |

**前端与引擎的交互模式:**

```text
CLI 前端:
  hermes_cli/main.py → 构建单个 AIAgent → run_conversation() → 回调驱动 UI 更新

Gateway 前端:
  gateway/run.py → GatewayRunner._agent_cache → 每消息取或建 AIAgent → run_conversation()

ACP 前端:
  hermes acp → ACPAdapter → 共享 AIAgent 实例 → run_conversation() via SSE

TUI 前端:
  ui-tui/ (Node) → tui_gateway (Python JSON-RPC) → AIAgent.run_conversation()
```

### 1.2 Layer 2: CLI / Gateway Entry (入口层)

#### hermes_cli/main.py (10,876 行)

CLI 入口层的核心职责是参数解析和环境初始化。最关键的操作在模块导入之前发生:

```python
# 源码位置: hermes_cli/main.py:101-176

def _apply_profile_override() -> None:
    """Pre-parse --profile/-p and set HERMES_HOME before module imports."""
    argv = sys.argv[1:]
    profile_name = None
    consume = 0

    # 1. Check for explicit -p / --profile flag
    for i, arg in enumerate(argv):
        if arg in ("--profile", "-p") and i + 1 < len(argv):
            profile_name = argv[i + 1]
            consume = 2
            break
        elif arg.startswith("--profile="):
            profile_name = arg.split("=", 1)[1]
            consume = 1
            break
```

Profile 覆盖必须在所有 Hermes 模块导入之前执行,因为许多模块在导入时缓存 `HERMES_HOME` (模块级常量)。`_apply_profile_override()` 从 `sys.argv` 中拦截 `--profile/-p` 并设置 `HERMES_HOME` 环境变量,然后从 argv 中剥离该标志以避免 argparse 冲突。

随后加载 .env 并桥接 `config.yaml` 的 `security.redact_secrets` 到环境变量:

```python
# 源码位置: hermes_cli/main.py:179-200

_apply_profile_override()

from hermes_cli.config import get_hermes_home
from hermes_cli.env_loader import load_hermes_dotenv

load_hermes_dotenv(project_env=PROJECT_ROOT / ".env")

# Bridge security.redact_secrets from config.yaml → HERMES_REDACT_SECRETS env
# var BEFORE hermes_logging imports agent.redact (which snapshots the flag at
# module-import time).
try:
    if "HERMES_REDACT_SECRETS" not in os.environ:
        import yaml as _yaml_early
        _cfg_path = get_hermes_home() / "config.yaml"
        if _cfg_path.exists():
            with open(_cfg_path, encoding="utf-8") as _f:
                _early_sec_cfg = (_yaml_early.safe_load(_f) or {}).get("security", {})
```

CLI 的子命令路由使用 argparse:

```python
# 源码位置: hermes_cli/main.py:1-44 (usage docstring)

"""
Usage:
    hermes                     # Interactive chat (default)
    hermes chat                # Interactive chat
    hermes gateway             # Run gateway in foreground
    hermes gateway start       # Start gateway as service
    hermes setup               # Interactive setup wizard
    hermes model               # Choose your LLM provider and model
    hermes tools               # Configure which tools are enabled
    hermes doctor              # Diagnose any issues
    hermes update              # Update to latest version
    hermes sessions browse     # Interactive session picker
"""
```

#### gateway/run.py (15,046 行)

网关入口层是 Hermes Agent 运行在消息平台模式时的核心管理器。GatewayRunner 类 (L988) 维护以下关键状态:

```text
GatewayRunner 核心数据结构:
  • _agent_cache: Dict[str, AIAgent] — LRU 缓存池 (max 128, idle TTL 1h)
  • _session_store: SessionStore — 线程安全的会话映射
  • _platforms: Dict[str, BasePlatformAdapter] — 21+ 消息平台适配器实例
  • _running_agents: Dict[str, threading.Thread] — 活跃智能体线程追踪
  • _background_tasks: List — 后台任务引用追踪
  • _gateway_task_refs: Set — asyncio 任务引用防止 GC
```

源码位置: `gateway/run.py:1-200` (模块头部,辅助函数,时间戳解析)

**Gateway 的独立并行运行机制:**

```text
Gateway 进程架构:
┌──────────────────────────────────────────────────┐
│              GatewayRunner (主线程)               │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Telegram │  │ Discord  │  │ 17 other      │  │
│  │ adapter  │  │ adapter  │  │ adapters      │  │
│  └──────────┘  └──────────┘  └──────────────┘  │
│        │             │              │            │
│        └──► 消息事件 ──► _handle_message() ──►  │
│                        │                         │
│                        ▼                         │
│              ┌─────────────────────┐             │
│              │ _get_or_create_agent│             │
│              │ (LRU cache lookup)  │             │
│              └─────────────────────┘             │
│                        │                         │
│                        ▼                         │
│              ┌─────────────────────┐             │
│              │ AIAgent.run_conversation│         │
│              │ (per-thread, per-session)│        │
│              └─────────────────────┘             │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │ 并行: 每个平台适配器独立 asyncio loop     │   │
│  │ 共享: SessionDB (SQLite WAL)             │   │
│  │ 缓存: AIAgent 实例 (LRU + idle TTL)      │   │
│  └──────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

Gateway 的关键设计特征:

- **AIAgent 缓存池**: LRU + idle TTL 淘汰,上限 128 实例。缓存签名包含完整的 auth token 指纹,确保凭证变更后自动重建
- **Auto-continue**: 网关重启后自动恢复中断的会话,通过 `tool-tail` 和 `resume_pending` 标记识别需要继续的 turn,配有新鲜度窗口 (默认 1h)
- **Agent cache cap**: `_enforce_agent_cache_cap()` 和 `_session_expiry_watcher()` 分别从 LRU 顺序和 idle TTL 两维度淘汰缓存实例

源码位置: `gateway/run.py:46-53` (缓存调优参数), `gateway/run.py:76-94` (auto-continue 机制)

### 1.3 Layer 3: Agent Engine (引擎层)

引擎层是 Hermes Agent 的核心,完全包含在 `run_agent.py` 的 AIAgent 类中。详细分析见 [下一章](/zh-CN/chapters/03-agent-engine)。

关键特征:

| 子系统 | 位置 | 行数范围 | 职责 |
|--------|------|----------|------|
| AIAgent 类 | L885 | 885-14,404 | 智能体主类,~13,500 行 |
| IterationBudget | L272 | 272-311 | 线程安全迭代计数器,共享预算 |
| run_conversation | L10573 | 10573-14,404 | 完整对话循环入口 |
| _interruptible_api_call | L6460 | 6460-6700 | 后台线程 API 调用 + 中断检测 |
| _try_activate_fallback | L7633 | 7633-7700 | 有序回退提供者链激活 |
| _restore_primary_runtime | L7831 | 7831-7900 | Turn-scoped 主运行时恢复 |
| _compress_context | L9221 | 9221-9400 | 上下文压缩 + SQLite session split |
| _run_tool | L9716 | 9716-9800 | 并发工具执行工作线程 |

**agent/ 目录关键支撑模块 (54 文件, ~20K 行):**

| 子模块 | 行数 | 职责 |
|--------|------|------|
| `agent/context_compressor.py` | 1,481 | 对话历史压缩,focus_topic 导向,session split |
| `agent/curator.py` | 1,674 | fork 子代理自审与改进 |
| `agent/prompt_builder.py` | 1,186 | 动态 system prompt 组装 |
| `agent/tool_guardrails.py` | 455 | 工具调用循环检测与渐进阻断 |
| `agent/error_classifier.py` | 1,036 | 18 种 FailoverReason 分类与恢复路径 |
| `agent/credential_pool.py` | 1,584 | 凭据轮转 (4 种策略),exhausted 冷却 |
| `agent/redact.py` | — | 输出/日志 secret redaction |

### 1.4 Layer 4: Tools / Storage (工具/存储层)

#### ToolRegistry (tools/registry.py, 537 行)

工具注册表采用自注册模式:每个工具文件在模块级别调用 `registry.register()` 声明其 schema、handler、toolset 和可用性检查。`model_tools.py` 查询注册表而非维护自己的平行数据结构。

```python
# 源码位置: tools/registry.py:57-74

def discover_builtin_tools(tools_dir: Optional[Path] = None) -> List[str]:
    """Import built-in self-registering tool modules and return their module names."""
    tools_path = Path(tools_dir) if tools_dir is not None else Path(__file__).resolve().parent
    module_names = [
        f"tools.{path.stem}"
        for path in sorted(tools_path.glob("*.py"))
        if path.name not in {"__init__.py", "registry.py", "mcp_tool.py"}
        and _module_registers_tools(path)
    ]
    imported: List[str] = []
    for mod_name in module_names:
        try:
            importlib.import_module(mod_name)
            imported.append(mod_name)
        except Exception as e:
            logger.warning("Could not import tool module %s: %s", mod_name, e)
    return imported
```

关键创新:AST 检测 `_module_registers_tools()` 使用 `ast.parse` 检查模块体是否有顶级 `registry.register(...)` 调用,避免导入 helper 模块中偶然包含的注册调用。`check_fn` TTL 缓存 (30s) 防止长生命周期进程重复探测外部状态 (Docker daemon, Modal SDK, playwright binary)。

```python
# 源码位置: tools/registry.py:143-160

class ToolRegistry:
    """Singleton registry that collects tool schemas + handlers from tool files."""

    def __init__(self):
        self._tools: Dict[str, ToolEntry] = {}
        self._toolset_checks: Dict[str, Callable] = {}
        self._toolset_aliases: Dict[str, str] = {}
        self._lock = threading.RLock()
        # Monotonically-increasing generation counter. Bumped on every
        # mutation (register / deregister / register_toolset_alias / MCP
        # refresh). External callers can memoize against it.
        self._generation: int = 0
```

ToolEntry 使用 `__slots__` 紧凑存储,包含 name, toolset, schema, handler, check_fn, requires_env, is_async, description, emoji, max_result_size_chars 等字段。

#### SessionDB (hermes_state.py, 2,669 行)

SessionDB (L159) 使用 SQLite WAL 模式提供会话持久化,支持 FTS5 全文搜索:

```text
SessionDB 核心能力:
  • SQLite WAL write-lock contention 修复 (v0.5: #3385)
  • FTS5 全文搜索跨会话内容
  • 消息存储与检索 (跨 CLI/Gateway/ACP 统一)
  • 会话命名与 lineage 追踪
  • Reasoning 持久化 (v0.5: schema v6 新增列)
  • 线程安全并发访问
```

源码位置: `hermes_state.py:159` (SessionDB 类定义)

#### Config System (hermes_cli/config.py, 4,939 行)

配置系统管理 config.yaml 解析、.env 加载和 DEFAULT_CONFIG 定义:

```text
Config 核心能力:
  • config.yaml 作为端点 URL 的单一真相来源 (v0.7: #4165)
  • .env 加载优先于 shell exports (v0.3: reload .env over stale shell overrides)
  • Config migration system (currently v7)
  • YAML null 值保护 (v0.5: #3377)
  • Atomic writes 防止数据丢失
  • 环境变量桥接 (security.redact_secrets → HERMES_REDACT_SECRETS)
```

源码位置: `hermes_cli/config.py:1-4,939`

---

## 2. 模块依赖图

```text
                            ┌──────────────┐
                            │  README.md   │
                            │  (用户入口)  │
                            └──────┬───────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
                    ▼              ▼              ▼
            ┌───────────┐  ┌───────────┐  ┌───────────┐
            │ hermes_cli│  │ gateway/  │  │ hermes acp│
            │ /main.py  │  │ /run.py   │  │           │
            │ (10,876)  │  │ (15,046)  │  │           │
            └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
                  │              │              │
                  └──────┬───────┘              │
                         │                      │
                         ▼                      │
                  ┌──────────────┐              │
                  │ run_agent.py │◄─────────────┘
                  │ (14,404)     │
                  │ AIAgent      │
                  └──────┬───────┘
                         │
          ┌──────────────┼──────────────┬──────────────┐
          │              │              │              │
          ▼              ▼              ▼              ▼
  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
  │ tools/    │  │ hermes_   │  │ hermes_   │  │ agent/    │
  │ registry  │  │ state.py  │  │ cli/      │  │ transports│
  │ (537)     │  │ (2,669)   │  │ config.py │  │ (v0.11+)  │
  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
        │              │              │              │
        ▼              │              │              │
  ┌───────────┐        │              │              │
  │ tools/*.py│        │              │              │
  │ (29 工具 │        │              │              │
  │  自注册)  │        │              │              │
  └───────────┘        │              │              │
                       │              │              │
         ┌─────────────┼──────────────┼──────────────┘
         │             │              │
         ▼             ▼              ▼
  ┌───────────┐  ┌───────────┐  ┌──────────────────┐
  │ SQLite    │  │ config.   │  │ Memory Providers  │
  │ WAL DB   │  │ yaml/.env │  │ (Honcho/Super/    │
  │           │  │           │  │  mem0/Hindsight/  │
  │           │  │           │  │  local/builtin)   │
  └───────────┘  └───────────┘  └──────────────────┘
```

依赖规则:

1. `run_agent.py` 不导入 `hermes_cli/main.py` 或 `gateway/run.py` (引擎层独立)
2. `tools/*.py` 在模块级导入 `tools/registry.py` (自注册,无循环依赖)
3. `model_tools.py` 导入 `tools.registry` + 所有工具模块 (桥接层)
4. `hermes_state.py` 不导入 `run_agent.py` (存储层独立)
5. `hermes_cli/config.py` 不导入 `run_agent.py` (配置层独立)
6. `agent/transports/` (v0.11+) 从 `run_agent.py` 中抽取,形成插件化传输层

源码位置: `tools/registry.py:7-14` (import chain 文档)

---

## 3. 六核心模块详解

### 3.1 run_agent.py (14,404 行) — 核心智能体引擎

`run_agent.py` 是整个系统的心脏,单文件承载完整智能体逻辑:

```python
# 源码位置: run_agent.py:885-973

class AIAgent:
    """
    AI Agent with tool calling capabilities.

    This class manages the conversation flow, tool execution, and response handling
    for AI models that support function calling.
    """

    def __init__(
        self,
        base_url: str = None,
        api_key: str = None,
        provider: str = None,
        api_mode: str = None,
        model: str = "",
        max_iterations: int = 90,
        ...
    ):
```

AIAgent.__init__ 接收 50+ 参数,包括推理配置 (base_url, api_key, provider, api_mode, model, max_iterations, max_tokens, reasoning_config)、工具配置 (enabled_toolsets, disabled_toolsets)、11 个回调 (tool_progress_callback, tool_start_callback, tool_complete_callback, thinking_callback, reasoning_callback, clarify_callback, step_callback, stream_delta_callback, interim_assistant_callback, tool_gen_callback, status_callback)、平台上下文 (platform, user_id, chat_id, thread_id)、会话管理 (session_db, parent_session_id, iteration_budget)、故障转移 (fallback_model, credential_pool) 和安全配置 (checkpoints_enabled)。

### 3.2 gateway/run.py (15,046 行) — 消息网关运行器

GatewayRunner (L988) 是网关模式的核心管理类:

```text
GatewayRunner 核心方法:
  • start() — 启动所有配置平台适配器
  • _handle_message() — 消息事件 → AIAgent.run_conversation()
  • _get_or_create_agent() — LRU 缓存查找/构建 AIAgent
  • _enforce_agent_cache_cap() — LRU + idle TTL 淘汰
  • _session_expiry_watcher() — 后台线程检测空闲会话
  • _auto_continue_interrupted_turn() — 重启后恢复中断会话
```

网关的关键设计是 **缓存签名**:完整 auth token 指纹确保凭证变更时自动重建 AIAgent 实例 (v0.5: #3247)。

### 3.3 hermes_cli/main.py (10,876 行) — CLI 入口

CLI 入口的独特之处在于 **profile 覆盖必须在模块导入之前执行**:

```python
# 源码位置: hermes_cli/main.py:92-99

# Profile override — MUST happen before any hermes module import.
#
# Many modules cache HERMES_HOME at import time (module-level constants).
# We intercept --profile/-p from sys.argv here and set the env var so that
# every subsequent ``os.getenv("HERMES_HOME", ...)`` resolves correctly.
# The flag is stripped from sys.argv so argparse never sees it.
```

### 3.4 tools/registry.py (537 行) — 工具注册表

工具注册表的核心创新:

| 机制 | 实现 | 意义 |
|------|------|------|
| AST 自发现 | `_module_registers_tools()` | 只导入包含顶级 `registry.register()` 的模块 |
| check_fn TTL 缓存 | 30s TTL, thread-safe | 长生命周期进程不重复探测外部状态 |
| Generation counter | `_generation: int` | 缓存 memoization key,mutation 时 bump |
| Thread-safe RLock | `threading.RLock` | MCP 动态刷新可并发读写注册表 |
| Snapshot state | `_snapshot_state()` | 读写一致性,外部代码拿到稳定快照 |

### 3.5 hermes_state.py (2,669 行) — 会话持久化

SessionDB (L159) 使用 SQLite WAL 模式解决并发写入冲突:

```text
SessionDB 关键修复历史:
  v0.2: 消息重复 3-4x (token 膨胀) → 修复 transcript offset
  v0.5: WAL write-lock contention (15-20s TUI freeze) → 修复
  v0.5: Schema v6 新增 reasoning, reasoning_details, codex_reasoning_items 列
  v0.9: 线程安全 SessionStore (threading.Lock 保护 _entries)
```

### 3.6 hermes_cli/config.py (4,939 行) — 配置管理

配置系统的关键设计:

| 设计 | 实现 | 版本 |
|------|------|------|
| config.yaml 单一真相 | 端点 URL 不再 env var vs config.yaml 冲突 | v0.7 |
| .env 优先 | `load_hermes_dotenv()` 优先加载用户 .env | v0.3 |
| Config migration | v7 migration system,自动升级旧格式 | v0.2+ |
| YAML null 保护 | `config.get()` 不 crash on YAML null | v0.5 |
| Atomic writes | config.yaml 使用 atomic 写防止数据丢失 | v0.6 |
| Env 桥接 | security.redact_secrets → HERMES_REDACT_SECRETS | 各版本 |

---

## 4. 数据流: 从消息到响应

### 4.1 CLI 模式数据流

```text
用户输入 ──► prompt_toolkit readline ──► slash command 检查
  │                                        │
  │ (slash command)                        │ (普通消息)
  │                                        │
  ▼                                        ▼
命令路由 ──► 内置处理           AIAgent.run_conversation(user_message)
                                              │
                                              ▼
                                     _interruptible_api_call()
                                              │
                              ┌───────────────┼───────────────┐
                              │               │               │
                              ▼               ▼               ▼
                         Anthropic     ChatCompletions    Bedrock/Codex
                         Messages API  API              Responses API
                              │               │               │
                              └───────────────┼───────────────┘
                                              │
                                              ▼
                                     响应解析 → tool_calls?
                                              │
                              ┌───────────────┼───────────────┐
                              │               │               │
                              ▼               ▼               ▼
                         文本响应        工具调用 → 并发执行    空响应 → retry
                         (返回给用户)    (ThreadPoolExecutor)   (nudge + retry)
                                              │
                                              ▼
                                     工具结果 → 附加到 messages
                                              │
                                              ▼
                                     继续循环 (直到无 tool_calls
                                     或 max_iterations 耗尽)
```

### 4.2 Gateway 模式数据流

```text
平台消息 ──► BasePlatformAdapter.receive() ──► GatewayRunner._handle_message()
                                                    │
                                                    ▼
                                          _get_or_create_agent(session_key)
                                                    │
                                    ┌───────────────┼───────────────┐
                                    │               │               │
                                    ▼               ▼               ▼
                              缓存命中:         缓存缺失:        idle TTL:
                              返回现有实例      构建新实例        淘汰旧实例
                              (LRU 顺序)       (凭证解析)       (回收资源)
                                                    │
                                                    ▼
                                          AIAgent.run_conversation(msg)
                                                    │
                                          (同 CLI 数据流, 但)
                                                    │
                              ┌───────────────┼───────────────┐
                              │               │               │
                              ▼               ▼               ▼
                         回调驱动         status_callback   stream_delta_
                         UI 更新          平台消息发送      callback 流式
                                                    │
                                                    ▼
                                          平台适配器.send_message()
                                          (Telegram/Discord/Slack/...)
```

---

## 5. Gateway 独立并行运行

### 5.1 Gateway 与 CLI 的并行关系

```text
┌────────────────────────────────────────────────────────────┐
│                    运行模式对比                             │
│                                                            │
│  CLI 模式:                                                 │
│  hermes ──► hermes_cli/main.py ──► 单 AIAgent ──► 单用户  │
│  (前台进程,阻塞式交互)                                      │
│                                                            │
│  Gateway 模式:                                              │
│  hermes gateway start ──► systemd service ──► 21+ 平台并行  │
│  (后台守护进程,事件驱动,多用户多平台)                        │
│                                                            │
│  关键差异:                                                 │
│  • CLI: 1 AIAgent, 1 session, 前台交互                     │
│  • Gateway: LRU 缓存池, 多 session, 后台守护               │
│  • Gateway: auto-continue 重启恢复                          │
│  • Gateway: 平台适配器各自独立 asyncio loop                  │
│  • CLI: 回调直接驱动 UI                                    │
│  • Gateway: 回调驱动平台消息发送                            │
└────────────────────────────────────────────────────────────┘
```

### 5.2 Gateway 的并行架构

Gateway 的每个平台适配器在独立的 asyncio loop 中运行,通过共享的 GatewayRunner 和 SessionDB 协调:

| 维度 | CLI 模式 | Gateway 模式 |
|------|----------|--------------|
| AIAgent 实例 | 单实例,每次重建 | LRU 缓存池 (max 128) |
| 用户数 | 1 | 多 (per-session) |
| 并发模型 | 同步阻塞 | 异步事件驱动 |
| 会话管理 | 内存 + SessionDB | SessionDB (SQLite WAL) |
| 生命周期 | 进程退出时销毁 | systemd daemon,重启恢复 |
| 消息交付 | 直接 stdout | 平台适配器.send_message() |
| 审批机制 | CLI input prompt | 平台按钮 (Slack/Telegram) |
| 超时管理 | 无限制 | inactivity-based (工具活动追踪) |
| 缓存签名 | 无 | auth token 指纹 |

---

## 6. 模块对比表

### 6.1 核心模块职责对比

| 模块 | 核心类 | 线程安全 | 状态持久 | 并发模型 | 缓存机制 |
|------|--------|----------|----------|----------|----------|
| `run_agent.py` | AIAgent | IterationBudget (Lock) | SessionDB | ThreadPool (tool) | transport + client |
| `gateway/run.py` | GatewayRunner | SessionStore (Lock) | SQLite WAL | asyncio per platform | LRU + idle TTL |
| `hermes_cli/main.py` | argparse | 否 | config.yaml/.env | 同步 | 否 |
| `tools/registry.py` | ToolRegistry | RLock | 否 | thread-safe mutations | check_fn TTL 30s |
| `hermes_state.py` | SessionDB | WAL lock | SQLite | WAL concurrent | 否 |
| `hermes_cli/config.py` | DEFAULT_CONFIG | 否 | config.yaml | 同步 | mtime-cached (v0.12) |

### 6.2 前端模式对比

| 前端 | 传输协议 | 消息格式 | 流式支持 | 会话模型 | 独立进程 |
|------|----------|----------|----------|----------|----------|
| Ink TUI | JSON-RPC | JSON | OSC-92 clipboard | per-thread | Node.js 子进程 |
| curses TUI | 直接 Python | 内存 | patch_stdout | per-process | 否 |
| Web Dashboard | HTTP SSE | JSON | SSE events | per-request | FastAPI 服务 |
| ACP | SSE + HTTP | ACP protocol | SSE | per-session | 否 |
| Gateway | platform-specific | Markdown/HTML | stream_delta | per-session-key | systemd daemon |

### 6.3 传输层对比 (v0.11+)

| Transport | 提供者 | API 格式 | 格式转换 | 流式 | 用途 |
|-----------|--------|----------|----------|------|------|
| AnthropicTransport | Anthropic | Messages API | Anthropic→OpenAI | SSE | Claude 直接 |
| ChatCompletionsTransport | OpenAI-compatible | Chat Completions | 原生 | SSE | 默认路径 |
| ResponsesApiTransport | Codex | Responses API | Codex→OpenAI | SSE | GPT/Codex |
| BedrockTransport | AWS | Converse API | Bedrock→OpenAI | 否 | AWS Bedrock |

---

## 7. Transport 层演进

v0.11 之前,所有格式转换和 HTTP 传输逻辑内嵌在 `run_agent.py` 中。v0.11 将这些抽取为可插拔 Transport ABC:

```text
run_agent.py v0.10 (内嵌逻辑):
  ┌─────────────────────────────────────────┐
  │ AIAgent                                 │
  │  • Anthropic format conversion           │
  │  • OpenAI chat.completions call          │
  │  • Codex responses.stream() call         │
  │  • Bedrock Converse call (新增时内嵌)    │
  │  • 流式消费 + SSE 解析                   │
  └─────────────────────────────────────────┘

run_agent.py v0.11+ (Transport ABC):
  ┌─────────────────────────────────────────┐
  │ AIAgent                                 │
  │  • _interruptible_api_call()             │
  │  • transport.dispatch(api_kwargs)        │
  │                                         │
  │  ┌─────────────────────────────────┐   │
  │  │ agent/transports/               │   │
  │  │  • AnthropicTransport           │   │
  │  │  • ChatCompletionsTransport     │   │
  │  │  • ResponsesApiTransport        │   │
  │  │  • BedrockTransport             │   │
  │  └─────────────────────────────────┘   │
  └─────────────────────────────────────────┘
```

---

## 总结表

| 组件 | 行数 | 职责 |
|------|------|------|
| `run_agent.py` | 14,404 | 核心智能体引擎 (AIAgent,对话循环,工具调度,流式,故障转移,压缩) |
| `gateway/run.py` | 15,046 | 消息网关运行器 (GatewayRunner,21+ 平台适配器,LRU 缓存,auto-continue) |
| `hermes_cli/main.py` | 10,876 | CLI 入口 (argparse 路由,profile 覆盖,.env 加载,子命令分发) |
| `tools/registry.py` | 537 | 工具注册表 (ToolRegistry 单例,AST 自发现,check_fn TTL,generation) |
| `hermes_state.py` | 2,669 | 会话持久化 (SessionDB,SQLite WAL,FTS5 搜索,reasoning 持久化) |
| `hermes_cli/config.py` | 4,939 | 配置管理 (config.yaml 解析,.env 加载,migration,atomic writes) |
| `agent/context_compressor.py` | 1,481 | 上下文压缩引擎 (对话压缩,focus_topic,session split) |
| `agent/curator.py` | 1,674 | 自审 fork agent (输出自审与改进建议) |
| `agent/prompt_builder.py` | 1,186 | 系统提示词构建 (动态 prompt 组装) |
| `agent/tool_guardrails.py` | 455 | 工具调用防护 (循环检测,渐进阻断) |
| `agent/error_classifier.py` | 1,036 | API 错误分类 (18 种 FailoverReason,恢复路径) |
| `agent/credential_pool.py` | 1,584 | 凭据池管理 (4 种轮转策略,exhausted 冷却) |
| `agent/redact.py` | — | 输出脱敏 (secret redaction/sanitization) |
| 总计核心模块 | 48,471 | 四层架构的全部核心逻辑 |

---

[上一章: 01 — 项目全景与设计哲学](/zh-CN/chapters/01-project-overview) | [下一章: 03 — 核心代理引擎 AIAgent](/zh-CN/chapters/03-agent-engine)