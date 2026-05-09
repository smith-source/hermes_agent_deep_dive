# 05 — 工具体系: 自注册 Registry + 29 自注册工具 (69 工具文件) + 7 执行后端 + 多层防护体系

[04 — LLM 提供商体系](/zh-CN/chapters/04-model-providers) | [06 — Gateway 消息平台](/zh-CN/chapters/06-gateway-system)

---

## 源码文件

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `tools/registry.py` | 537 | ToolRegistry 单例 + 自注册发现 |
| `toolsets.py` | ~786 | 工具集定义与分组 |
| `tools/terminal_tool.py` | 2342 | Terminal 工具 + 7 执行后端调度 |
| `tools/browser_tool.py` | 3476 | Browser 工具 + CDP/云端自动化 |
| `tools/delegate_tool.py` | 2562 | Delegate 子代理 + MAX_DEPTH=1 |
| `tools/mcp_tool.py` | 3175 | MCP 协议客户端 (stdio/HTTP) |
| `tools/approval.py` | 1258 | 危险命令审批 + 3 种模式 |
| `agent/tool_guardrails.py` | 455 | 工具调用循环检测与防护 |
| `tools/environments/base.py` | 786 | 执行环境基类 (spawn-per-call) |
| `tools/environments/local.py` | 524 | 本地执行后端 |
| `tools/environments/docker.py` | 645 | Docker 容器后端 |
| `tools/environments/modal.py` | 460 | Modal 云沙箱后端 |
| `tools/environments/managed_modal.py` | 282 | Modal 网关托管模式 |
| `tools/environments/ssh.py` | 295 | SSH 远程执行后端 |
| `tools/environments/singularity.py` | 262 | Singularity 容器后端 |
| `tools/environments/vercel_sandbox.py` | 638 | Vercel Sandbox 后端 |
| `tools/environments/daytona.py` | 259 | Daytona 云开发后端 |
| `tools/environments/file_sync.py` | 399 | 跨后端文件同步 |
| `tools/environments/modal_utils.py` | 199 | Modal 共享工具函数 |
| `tools/web_tools.py` | 2,223 | 网页搜索与提取工具 |
| `tools/send_message_tool.py` | 1,780 | 跨平台消息发送工具 |
| `tools/tts_tool.py` | 2,185 | TTS 语音合成工具 |
| `tools/code_execution_tool.py` | 1,621 | 代码执行 (sandbox) 工具 |
| `tools/vision_tools.py` | 1,168 | 图像视觉分析工具 |
| `tools/voice_mode.py` | — | 语音模式控制工具 |
| `tools/homeassistant_tool.py` | — | HomeAssistant 智能家居工具 |
| `tools/session_search_tool.py` | — | 会话搜索工具 |
| `tools/transcription_tools.py` | — | 音频转录工具 |
| `tools/feishu_doc_tool.py` | — | 飞书文档操作工具 |
| `tools/feishu_drive_tool.py` | — | 飞书云盘操作工具 |
| `tools/discord_tool.py` | — | Discord 平台交互工具 |
| `tools/clarify_tool.py` | — | 用户交互澄清工具 |
| `tools/todo_tool.py` | — | Todo 列表管理工具 |
| `tools/process_registry.py` | — | 进程注册与管理工具 |
| `tools/mixture_of_agents_tool.py` | — | 多代理混合推理工具 |
| `tools/yuanbao_tools.py` | — | 腾讯元宝平台工具 |

## 一句话总结

Hermes 工具体系通过 ToolRegistry 自注册模式聚合 29 个自注册工具（tools/ 目录共 69 个工具相关 .py 文件，其中 29 个在模块级调用 registry.register()），Terminal 工具支持 7 种执行后端（local/docker/modal/ssh/singularity/vercel/daytona），Browser 工具实现 CDP 无头自动化，Delegate 工具以 MAX_DEPTH=1 生成子代理，MCP 工具通过 stdio/HTTP 传输接入外部工具服务器，Approval 系统提供 hardline/interactive/smart 三层防护，ToolGuardrails 检测循环调用并渐进阻断。

---

## Architecture Overview

```
                    ┌─────────────────────────────────────────────┐
                    │          AIAgent Tool-Calling Loop           │
                    │                                             │
                    │   ┌───────────────────────────────────┐     │
                    │   │     ToolGuardrailsController       │     │
                    │   │  before_call → after_call → decide │     │
                    │   │  (warn → block → halt progression) │     │
                    │   └───────────────────────────────────┘     │
                    │                    │                        │
                    │                    ▼                        │
                    │   ┌───────────────────────────────────┐     │
                    │   │         ToolRegistry               │     │
                    │   │  29 tools self-register at import │     │
                    │   │  toolsets group + check_fn gating   │     │
                    │   │  generation counter for MCP refresh │     │
                    │   └──────────────────┬────────────────┘     │
                    │                      │                      │
                    │     ┌────────────────┼────────────────┐     │
                    │     │                │                │     │
                    │     ▼                ▼                ▼     │
                    │  ┌─────────┐  ┌───────────┐  ┌──────────┐  │
                    │  │Terminal │  │ Browser    │  │ Delegate  │  │
                    │  │Tool     │  │ Tool       │  │ Tool      │  │
                    │  │         │  │            │  │ MAX_DEPTH │  │
                    │  │ 7 back- │  │ CDP/cloud  │  │ =1 (flat) │  │
                    │  │ ends    │  │ providers  │  │           │  │
                    │  └──┬──────┘  └──┬─────────┘  └──┬───────┘  │
                    │     │            │               │          │
                    │     ▼            ▼               ▼          │
                    │  ┌───────────────────────────────────┐     │
                    │  │      MCPTool (stdio/HTTP)          │     │
                    │  │  External tool servers → registry   │     │
                    │  └───────────────────────────────────┘     │
                    │                                             │
                    │   ┌───────────────────────────────────┐     │
                    │   │       Approval System              │     │
                    │   │  hardline / interactive / smart     │     │
                    │   └───────────────────────────────────┘     │
                    └─────────────────────────────────────────────┘
                                       │
                                       ▼
                    ┌───────────────────────────────────────┐
                    │   tools/environments/ (7 backends)    │
                    │  ┌──────┐ ┌───────┐ ┌──────┐ ┌─────┐ │
                    │  │local │ │docker │ │modal │ │ ssh │ │
                    │  └──┬───┘ └──┬────┘ └──┬───┘ └──┬──┘ │
                    │     │        │        │       │     │
                    │  ┌──────┐ ┌───────┐ ┌──────┐ ┌─────┐ │
                    │  │sing- │ │vercel │ │dayt- │ │file │ │
                    │  │ular  │ │sandb  │ │ona   │ │sync │ │
                    │  └──┬───┘ └──┬────┘ └──┬───┘ └──┬──┘ │
                    │     └────────┴────────┴───────┴──►  │
                    │         BaseEnvironment (ABC)         │
                    │         spawn-per-call model          │
                    └───────────────────────────────────────┘
```

---

## TL;DR

Hermes 的工具体系基于 ToolRegistry 自注册模式——每个工具文件在模块级别调用 `registry.register()`，导入时自动声明 schema、handler、toolset 和可用性检查。Terminal 工具（2342 行）是最大的内置工具，支持 7 种执行后端（local/docker/modal/ssh/singularity/vercel_sandbox/daytona），采用 spawn-per-call 模型。Browser 工具（3476 行）通过 agent-browser CLI + CDP accessibility tree 实现无头自动化，支持 Browser Use/Browserbase/Firecrawl 三种云提供商。Delegate 工具（2562 行）生成子 AIAgent 实例，MAX_DEPTH=1 阻止递归委托，子代理获得受限工具集（5 个工具被阻断）。MCP 工具（3175 行）通过 stdio 和 HTTP/StreamableHTTP 传输接入外部工具服务器。Approval 系统（1258 行）提供 hardline（不可绕过）、interactive（用户确认）、smart（LLM 自动审批）三层防护。ToolGuardrails（455 行）通过 ToolCallSignature 跟踪调用指纹，渐进式 warn→block→halt 阻断循环调用。

---

## 1. ToolRegistry — 自注册发现模式

源码位置: `tools/registry.py:1-537`

ToolRegistry 是工具体系的中心枢纽。每个工具文件在模块级别调用 `registry.register()`，model_tools.py 查询 registry 而非维护自己的并行数据结构。

**AST 发现与自动导入** (源码位置: `registry.py:29-74`):

```python
def _is_registry_register_call(node: ast.AST) -> bool:
    """Return True when node is a registry.register(...) call expression."""
    if not isinstance(node, ast.Expr) or not isinstance(node.value, ast.Call):
        return False
    func = node.value.func
    return (
        isinstance(func, ast.Attribute)
        and func.attr == "register"
        and isinstance(func.value, ast.Name)
        and func.value.id == "registry"
    )

def discover_builtin_tools(tools_dir=None) -> List[str]:
    """Import built-in self-registering tool modules and return module names."""
    tools_path = Path(tools_dir) if tools_dir else Path(__file__).resolve().parent
    module_names = [
        f"tools.{path.stem}"
        for path in sorted(tools_path.glob("*.py"))
        if path.name not in {"__init__.py", "registry.py", "mcp_tool.py"}
        and _module_registers_tools(path)      # AST scan for register() calls
    ]
    for mod_name in module_names:
        importlib.import_module(mod_name)       # triggers self-registration
    return imported
```

**ToolEntry 数据结构** (源码位置: `registry.py:77-98`):

```python
class ToolEntry:
    __slots__ = (
        "name", "toolset", "schema", "handler", "check_fn",
        "requires_env", "is_async", "description", "emoji",
        "max_result_size_chars",
    )
```

**check_fn TTL 缓存** (源码位置: `registry.py:102-140`): 可用性检查（Docker daemon、Modal SDK、Playwright 二进制）探测外部状态，30 秒 TTL 缓存避免重复浪费。

**线程安全与 generation counter** (源码位置: `registry.py:143-160`): MCP 动态刷新可并发修改 registry。`_lock = threading.RLock()` 串行化写操作，`_generation` 单调递增计数器让 `get_tool_definitions()` 等外部消费者基于 generation memoize。

### 比较表: ToolRegistry 发现机制对比

| 发现方式 | 触发时机 | 作用域 | 线程安全 |
|----------|----------|--------|----------|
| `discover_builtin_tools()` | 启动导入 | 内置工具文件 | RLock + generation |
| MCP `register()` | 连接/刷新时 | 动态外部工具 | RLock + generation |
| `discover_builtin_tools()` AST scan | 每次启动 | 过滤非注册模块 | 无状态 |

---

## 2. Terminal Tool — 7 执行后端

源码位置: `tools/terminal_tool.py:1-2342`

Terminal Tool 是 Hermes 的最大内置工具，支持 7 种执行环境:

**环境选择** (源码位置: `terminal_tool.py:9-13`):

```
TERMINAL_ENV 环境变量:
- "local": 直接在主机执行 (默认, 最快)
- "docker": Docker 容器隔离 (需要 Docker)
- "modal": Modal 云沙箱 (直接或托管网关)
- "ssh": SSH 远程执行
- "singularity": Singularity 容器 (HPC 环境)
- "vercel_sandbox": Vercel Sandbox 云沙箱
- "daytona": Daytona 云开发环境
```

**中断响应机制** (源码位置: `terminal_tool.py:57-58`): 全局 `_interrupt_event` 由 agent 在用户中断时设置，terminal tool 在命令执行中轮询此事件，立即终止长运行子进程。

```python
from tools.interrupt import is_interrupted, _interrupt_event
```

**前台超时硬上限** (源码位置: `terminal_tool.py:106-111`):

```python
FOREGROUND_MAX_TIMEOUT = _safe_parse_import_env(
    "TERMINAL_MAX_FOREGROUND_TIMEOUT",
    600,     # 10 分钟硬上限
    int,
    "integer",
)
```

**Vercel Sandbox 验证** (源码位置: `terminal_tool.py:120-150`): 支持 node24/node22/python3.13 三种 runtime，磁盘限制为 51200 MB。

**BaseEnvironment — spawn-per-call 模型** (源码位置: `tools/environments/base.py:1-786`):

所有后端继承 `BaseEnvironment` ABC，采用统一 spawn-per-call 模型: 每次命令生成新的 `bash -c` 进程。会话快照（env vars、functions、aliases）在初始化时捕获一次，每条命令前重新 source。CWD 通过 stdout 标记（远程）或临时文件（本地）持久化。

```python
class BaseEnvironment(ABC):
    """Unified spawn-per-call model:
    every command spawns a fresh bash -c process.
    A session snapshot is captured once at init and
    re-sourced before each command."""
```

**活动回调** (源码位置: `base.py:43-78`): 线程本地 activity callback，长运行 `_wait_for_process` 循环定期向 gateway 报告存活状态。

### 比较表: 7 执行后端特性对比

| 后端 | 行数 | 隔离级别 | 需要 Docker | 云端 | 文件同步 |
|------|------|----------|------------|------|----------|
| local | 524 | 无 | 否 | 否 | 无 |
| docker | 645 | 容器 | 是 | 否 | volume mount |
| modal | 460 | 云沙箱 | 否 | 是 | Modal Volume |
| managed_modal | 282 | 云托管 | 否 | 是 | Modal Volume |
| ssh | 295 | 远程主机 | 否 | 否 | rsync/scp |
| singularity | 262 | HPC 容器 | 否 | 否 | overlay fs |
| vercel_sandbox | 638 | 云沙箱 | 否 | 是 | persistent fs |
| daytona | 259 | 云开发 | 否 | 是 | workspace fs |

### 比较表: Terminal Tool 安全机制对比

| 安全层 | 实现位置 | 作用 | 可绕过 |
|--------|----------|------|--------|
| hardline blocklist | `approval.py:116-150` | 禁止 rm -rf /, mkfs, shutdown 等不可恢复命令 | 否（yolo 也不行） |
| dangerous patterns | `approval.py` | 匹配 rm -rf, chmod 777, curl|sh 等 | yolo 可绕过 |
| approval modes | `approval.py` | interactive/smart/off 三模式 | yolo=off |
| check_fn gating | `registry.py` | 探测后端可用性 | 否 |
| foreground timeout | `terminal_tool.py:106` | 600s 硬上限 | 否 |
| interrupt event | `tools/interrupt` | 用户中断立即终止子进程 | 否 |

---

## 3. Browser Tool — CDP 自动化

源码位置: `tools/browser_tool.py:1-3476`

Browser Tool 使用 agent-browser CLI 的 accessibility tree (ariaSnapshot) 实现文本化页面交互，无需视觉能力即可操作网页。

**三种云提供商 + Camofox 本地反检测** (源码位置: `browser_tool.py:73-94`):

```python
from tools.browser_providers.base import CloudBrowserProvider
from tools.browser_providers.browserbase import BrowserbaseProvider
from tools.browser_providers.browser_use import BrowserUseProvider
from tools.browser_providers.firecrawl import FirecrawlProvider
from tools.tool_backend_helpers import normalize_browser_cloud_provider

# Camofox local anti-detection (optional)
try:
    from tools.browser_camofox import is_camofox_mode as _is_camofox_mode
except ImportError:
    _is_camofox_mode = lambda: False
```

**PATH 发现机制** (源码位置: `browser_tool.py:98-144`): 标准 PATH 补充（包括 Termux、macOS Homebrew），以及 Homebrew versioned Node.js bin 目录动态发现（node@20, node@24 等）。

```python
_SANE_PATH_DIRS = (
    "/data/data/com.termux/files/usr/bin",  # Android/Termux
    "/opt/homebrew/bin",                     # macOS Homebrew
    "/usr/local/sbin", "/usr/local/bin",
    "/usr/sbin", "/usr/bin", "/sbin", "/bin",
)

@functools.lru_cache(maxsize=1)
def _discover_homebrew_node_dirs() -> tuple[str, ...]:
    """Find Homebrew versioned Node.js bin directories (node@20, node@24)."""
```

**网站安全检查** (源码位置: `browser_tool.py:74-81`): `website_policy` 模块检查访问权限，`url_safety` 模块验证 URL 安全性。

```python
try:
    from tools.website_policy import check_website_access
except Exception:
    check_website_access = lambda url: None  # fail-open

try:
    from tools.url_safety import is_safe_url as _is_safe_url
except Exception:
    _is_safe_url = lambda url: False  # fail-closed: block all if unavailable
```

### 比较表: Browser Tool 后端对比

| 后端 | 认证 | 位置 | stealth | 会话隔离 |
|------|------|------|---------|----------|
| Local Chromium | 无 | 本地 | Camofox (可选) | per task_id |
| Browser Use | BROWSER_USE_API_KEY | 云 | 内置 | per task_id |
| Browserbase | BROWSERBASE_API_KEY + PROJECT_ID | 云 | Advanced Stealth (Scale Plan) | per task_id |
| Firecrawl | FIRECRAWL_API_KEY | 云 | 无 | 无状态 |

---

## 4. Delegate Tool — 子代理架构

源码位置: `tools/delegate_tool.py:1-2562`

Delegate Tool 生成子 AIAgent 实例，拥有隔离上下文、受限工具集和独立终端会话。

**核心限制** (源码位置: `delegate_tool.py:39-49`):

```python
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",    # 无递归委托
    "clarify",          # 无用户交互
    "memory",           # 无写入共享 MEMORY.md
    "send_message",     # 无跨平台副作用
    "execute_code",     # 子代理应逐步推理，不写脚本
])

MAX_DEPTH = 1  # flat: parent(0) -> child(1); grandchild rejected
_MIN_SPAWN_DEPTH = 1
_MAX_SPAWN_DEPTH_CAP = 3
```

**子代理审批回调** (源码位置: `delegate_tool.py:60-106`): 子代理在 ThreadPoolExecutor worker 中运行，不继承 CLI 的交互审批回调。默认 `_subagent_auto_deny`（安全），可选 `_subagent_auto_approve`（YOLO，需 config `delegation.subagent_auto_approve: true`）。

```python
def _subagent_auto_deny(command, description, **kwargs) -> str:
    """Auto-deny dangerous commands in subagent threads (safe default).
    Returns 'deny' so the subagent sees a refusal, never calls input()."""
    logger.warning("Subagent auto-denied dangerous command: %s (%s).", command, description)
    return "deny"

def _subagent_auto_approve(command, description, **kwargs) -> str:
    """Auto-approve dangerous commands in subagent threads (opt-in YOLO)."""
    logger.warning("Subagent auto-approved dangerous command: %s (%s)", command, description)
    return "once"
```

**运行时暂停与活跃注册** (源码位置: `delegate_tool.py:144-150`): `_spawn_pause_lock` + `_spawn_paused` 暂停标志，`_active_subagents` 注册表供 TUI 可观察层和 gateway RPC 查询。

```python
_spawn_pause_lock = threading.Lock()
_spawn_paused: bool = False

_active_subagents_lock = threading.Lock()
_active_subagents: Dict[str, Dict[str, Any]] = {}
```

### 比较表: Delegate Tool 子代理隔离对比

| 方面 | 父代理 | 子代理 | 原因 |
|------|--------|--------|------|
| 对话历史 | 完整多轮 | 仅目标+上下文 prompt | 防止上下文膨胀 |
| 工具集 | 全部可用 | 受限（5 工具阻断） | 安全与功能边界 |
| 审批模式 | interactive | auto_deny/auto_approve | 防止 worker thread input() 死锁 |
| 委托深度 | 无限 | MAX_DEPTH=1 (flat) | 防止递归委托 |
| terminal 会话 | 独立 task_id | 独立 task_id | 会话隔离 |

---

## 5. MCP Tool — Model Context Protocol 客户端

源码位置: `tools/mcp_tool.py:1-3175`

MCP Tool 通过 stdio 和 HTTP/StreamableHTTP 传输连接外部工具服务器，发现其工具并注册到 Hermes ToolRegistry。

**配置格式** (源码位置: `mcp_tool.py:13-43`):

```yaml
mcp_servers:
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
    timeout: 120         # per-tool-call timeout (default: 120)
    connect_timeout: 60  # initial connection timeout (default: 60)
  remote_api:
    url: "https://my-mcp-server.example.com/mcp"
    headers:
      Authorization: "Bearer sk-..."
    timeout: 180
  analysis:
    command: "npx"
    args: ["-y", "analysis-server"]
    sampling:                    # server-initiated LLM requests
      enabled: true
      model: "gemini-3-flash"
      max_tokens_cap: 4096
      timeout: 30
      max_rpm: 10
      allowed_models: []
      max_tool_rounds: 5
      log_level: "info"
```

**架构** (源码位置: `mcp_tool.py:55-69`): 专用后台事件循环 `_mcp_loop` 在 daemon 线程中运行。每个 MCP 服务器作为长期 asyncio Task 在此循环上运行，保持传输上下文活跃。工具调用协程通过 `run_coroutine_threadsafe()` 调度到循环上。

**Stdio stderr 重定向** (源码位置: `mcp_tool.py:91-143`): MCP SDK 默认将子进程 stderr 输出到用户 TTY，破坏 prompt_toolkit/Rich TUI 渲染。Hermes 重定向所有 stdio MCP 子进程 stderr 到共享日志文件 `~/.hermes/logs/mcp-stderr.log`，标记服务器名称。

```python
def _get_mcp_stderr_log() -> Any:
    """Return shared append-mode file handle for MCP subprocess stderr.
    Opened once per process, reused for every stdio server."""
    global _mcp_stderr_log_fh
    with _mcp_stderr_log_lock:
        if _mcp_stderr_log_fh is not None:
            return _mcp_stderr_log_fh
        log_path = log_dir / "mcp-stderr.log"
        fh = open(log_path, "a", encoding="utf-8", errors="replace", buffering=1)
        fh.fileno()  # sanity-check real fd available
        _mcp_stderr_log_fh = fh
```

**线程安全** (源码位置: `mcp_tool.py:66-69`): `_servers` 和 `_mcp_loop/_mcp_thread` 从 MCP 后台线程和调用线程同时访问。所有修改通过 `_lock` 保护，安全于 GIL 存在与否（Python 3.13+ free-threading）。

### 比较表: MCP 传输模式对比

| 传输 | 启动方式 | 生命周期 | 适用场景 | stderr 处理 |
|------|----------|----------|----------|------------|
| stdio | subprocess (command + args) | 长期 Task | 本地工具服务器 | 重定向到 mcp-stderr.log |
| HTTP/StreamableHTTP | HTTP POST (url + headers) | 长期 Task | 远程 API 服务 | 无子进程 stderr |

---

## 6. Approval System — 危险命令审批

源码位置: `tools/approval.py:1-1258`

Approval 系统是 Hermes 安全防护的单一真相来源，提供三层防护。

**Hardline 无条件阻断清单** (源码位置: `approval.py:116-150`): 不可绕过——即使 yolo 模式也不允许执行。仅列出无恢复路径的命令: 文件系统根级销毁 (`rm -rf /`)、块设备覆写 (`mkfs`)、内核关机 (`shutdown`, `reboot`)、主机拒绝服务命令。

```python
# Hardline: commands so catastrophic they should NEVER run, regardless of
# --yolo, /yolo, approvals.mode=off, or cron approve mode.
# Only applies to environments that can damage the host (local, ssh).
# Containerized backends (docker, modal, etc.) bypass this layer.
_CMDPOS = (
    r'(?:^|[;&|\n`]|\$\()'         # start position
    r'\s*'
    r'(?:sudo\s+(?:-[^\s]+\s+)*)?'  # optional sudo with flags
)
```

**三种审批模式**:

| 模式 | 交互 | 适用场景 | LLM 参与 |
|------|------|----------|----------|
| interactive | 用户手动确认 | CLI/TUI 默认 | 无 |
| smart | LLM 辅助自动审批低风险命令 | 长运行任务 | auxiliary_client.call_llm |
| off (yolo) | 自动通过（hardline 仍阻断） | 信任代理 | 无 |

**会话上下文安全** (源码位置: `approval.py:30-84`): Gateway 并发运行 agent turn，使用 `contextvars.ContextVar` 代替全局 env var 防止竞态。

```python
_approval_session_key: contextvars.ContextVar[str] = contextvars.ContextVar(
    "approval_session_key", default="",
)

def get_current_session_key(default="default") -> str:
    """Resolution order:
    1. approval-specific contextvars (set by gateway before agent.run)
    2. session_context contextvars (set by _set_session_env)
    3. os.environ fallback (CLI, cron, tests)"""
    session_key = _approval_session_key.get()
    if session_key:
        return session_key
    from gateway.session_context import get_session_env
    return get_session_env("HERMES_SESSION_KEY", default)
```

**敏感写入目标检测** (源码位置: `approval.py:88-113`): 正则匹配 `$HOME/.ssh`, `.hermes/.env`, `.bashrc/.zshrc`, `.netrc/.pgpass` 等敏感文件路径，即使通过 shell 扩展引用也触发审批。

### 比较表: Approval 三层防护对比

| 层级 | 机制 | 可绕过 | 覆盖范围 |
|------|------|--------|----------|
| hardline | 正则匹配无恢复命令 | 否（yolo 也不行） | local/ssh 主机环境 |
| dangerous | 正则匹配高风险命令 | yolo 可绕过 | 所有非容器后端 |
| interactive/smart | 用户确认或 LLM 自动 | off 模式可绕过 | 所有环境 |

---

## 7. ToolGuardrails — 循环调用检测

源码位置: `agent/tool_guardrails.py:1-455`

ToolGuardrailsController 通过 `ToolCallSignature` 跟踪工具调用指纹，渐进式阻断循环调用。

**工具分类** (源码位置: `tool_guardrails.py:19-59`):

```python
IDEMPOTENT_TOOL_NAMES = frozenset({
    "read_file", "search_files", "web_search", "web_extract",
    "session_search", "browser_snapshot", "browser_console",
    "mcp_filesystem_read_file", ...
})

MUTATING_TOOL_NAMES = frozenset({
    "terminal", "execute_code", "write_file", "patch", "todo",
    "memory", "browser_click", "browser_type", "browser_press",
    "browser_navigate", "send_message", "cronjob", "delegate_task", ...
})
```

**配置阈值** (源码位置: `tool_guardrails.py:63-123`):

```python
@dataclass(frozen=True)
class ToolCallGuardrailConfig:
    warnings_enabled: bool = True         # 警告默认开启
    hard_stop_enabled: bool = False       # 硬停默认关闭（需显式 opt-in）
    exact_failure_warn_after: int = 2     # 同参数失败 2 次警告
    exact_failure_block_after: int = 5    # 同参数失败 5 次阻断
    same_tool_failure_warn_after: int = 3 # 同工具不同参数失败 3 次警告
    same_tool_failure_halt_after: int = 8 # 同工具不同参数失败 8 次停机
    no_progress_warn_after: int = 2       # 幂等工具无进展 2 次警告
    no_progress_block_after: int = 5      # 幂等工具无进展 5 次阻断
```

**ToolCallSignature — 不可逆指纹** (源码位置: `tool_guardrails.py:127-141`):

```python
@dataclass(frozen=True)
class ToolCallSignature:
    """Stable, non-reversible identity for a tool name plus canonical args."""
    tool_name: str
    args_hash: str                # SHA-256 of canonical sorted JSON

    @classmethod
    def from_call(cls, tool_name, args):
        canonical = canonical_tool_args(args or {})
        return cls(tool_name=tool_name, args_hash=_sha256(canonical))
```

**渐进式决策路径** (源码位置: `tool_guardrails.py:238-380`):

```
before_call:
  - exact_failure >= block_after → block (repeated_exact_failure_block)
  - idempotent no_progress >= block_after → block (idempotent_no_progress_block)
  - else → allow

after_call:
  - failed:
    - exact_failure count → warn after 2, block after 5
    - same_tool_failure count → warn after 3, halt after 8
  - success:
    - clear failure counters
    - idempotent tool: track result hash → warn no_progress after 2, block after 5
```

### 比较表: Guardrail 决策类型对比

| 决策 | action | allows_execution | should_halt | 效果 |
|------|--------|------------------|-------------|------|
| allow | "allow" | True | False | 正常执行 |
| warn | "warn" | True | False | 执行 + 附加警告文本 |
| block | "block" | False | True | 拒绝执行，返回 synthetic result |
| halt | "halt" | False | True | 拒绝执行 + 停止当前 turn |

---

## Summary Table

| 组件 | 文件 | 行数 | 核心职责 |
|------|------|------|----------|
| ToolRegistry | `tools/registry.py` | 537 | 自注册单例 + AST 发现 + generation counter |
| Toolsets | `toolsets.py` | ~786 | 工具集分组定义 |
| Terminal Tool | `tools/terminal_tool.py` | 2342 | 7 执行后端调度 + 中断响应 |
| Browser Tool | `tools/browser_tool.py` | 3476 | CDP 无头自动化 + 3 云提供商 |
| Delegate Tool | `tools/delegate_tool.py` | 2562 | 子代理生成 + MAX_DEPTH=1 |
| MCP Tool | `tools/mcp_tool.py` | 3175 | stdio/HTTP 传输 + 采样支持 |
| Approval | `tools/approval.py` | 1258 | hardline/interactive/smart 三层防护 |
| ToolGuardrails | `agent/tool_guardrails.py` | 455 | 循环调用检测 + 渐进阻断 |
| BaseEnvironment | `tools/environments/base.py` | 786 | spawn-per-call 模型基类 |
| Local Backend | `tools/environments/local.py` | 524 | 主机直接执行 |
| Docker Backend | `tools/environments/docker.py` | 645 | 容器隔离执行 |
| Modal Backend | `tools/environments/modal.py` | 460 | 云沙箱执行 |
| SSH Backend | `tools/environments/ssh.py` | 295 | 远程主机执行 |
| Singularity Backend | `tools/environments/singularity.py` | 262 | HPC 容器执行 |
| Vercel Sandbox | `tools/environments/vercel_sandbox.py` | 638 | 云沙箱执行 |
| Daytona Backend | `tools/environments/daytona.py` | 259 | 云开发环境执行 |

---

[04 — LLM 提供商体系](/zh-CN/chapters/04-model-providers) | [06 — Gateway 消息平台](/zh-CN/chapters/06-gateway-system)