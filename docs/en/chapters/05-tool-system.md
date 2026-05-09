# 05 — Tool System: Self-registering Registry + 29 Self-registering Tools (69 Tool Files) + 7 Execution Backends + Multi-layer Protection System

[04 — LLM Provider System](/en/chapters/04-model-providers) | [06 — Gateway Messaging Platform](/en/chapters/06-gateway-system)

---

## Source Files

| File | Lines | Core Responsibility |
|------|------|----------|
| `tools/registry.py` | 537 | ToolRegistry singleton + self-registering discovery |
| `toolsets.py` | ~786 | Toolset definition and grouping |
| `tools/terminal_tool.py` | 2342 | Terminal tool + 7 execution backend dispatching |
| `tools/browser_tool.py` | 3476 | Browser tool + CDP/cloud automation |
| `tools/delegate_tool.py` | 2562 | Delegate sub-agent + MAX_DEPTH=1 |
| `tools/mcp_tool.py` | 3175 | MCP protocol client (stdio/HTTP) |
| `tools/approval.py` | 1258 | Dangerous command approval + 3 modes |
| `agent/tool_guardrails.py` | 455 | Tool call loop detection and protection |
| `tools/environments/base.py` | 786 | Execution environment base class (spawn-per-call) |
| `tools/environments/local.py` | 524 | Local execution backend |
| `tools/environments/docker.py` | 645 | Docker container backend |
| `tools/environments/modal.py` | 460 | Modal cloud sandbox backend |
| `tools/environments/managed_modal.py` | 282 | Modal gateway managed mode |
| `tools/environments/ssh.py` | 295 | SSH remote execution backend |
| `tools/environments/singularity.py` | 262 | Singularity container backend |
| `tools/environments/vercel_sandbox.py` | 638 | Vercel Sandbox backend |
| `tools/environments/daytona.py` | 259 | Daytona cloud development backend |
| `tools/environments/file_sync.py` | 399 | Cross-backend file synchronization |
| `tools/environments/modal_utils.py` | 199 | Modal shared utility functions |
| `tools/web_tools.py` | 2,223 | Web search and extraction tools |
| `tools/send_message_tool.py` | 1,780 | Cross-platform message sending tool |
| `tools/tts_tool.py` | 2,185 | TTS voice synthesis tool |
| `tools/code_execution_tool.py` | 1,621 | Code execution (sandbox) tool |
| `tools/vision_tools.py` | 1,168 | Image visual analysis tool |
| `tools/voice_mode.py` | — | Voice mode control tool |
| `tools/homeassistant_tool.py` | — | HomeAssistant smart home tool |
| `tools/session_search_tool.py` | — | Session search tool |
| `tools/transcription_tools.py` | — | Audio transcription tool |
| `tools/feishu_doc_tool.py` | — | Feishu document operation tool |
| `tools/feishu_drive_tool.py` | — | Feishu cloud drive operation tool |
| `tools/discord_tool.py` | — | Discord platform interaction tool |
| `tools/clarify_tool.py` | — | User interaction clarification tool |
| `tools/todo_tool.py` | — | Todo list management tool |
| `tools/process_registry.py` | — | Process registration and management tool |
| `tools/mixture_of_agents_tool.py` | — | Multi-agent mixed reasoning tool |
| `tools/yuanbao_tools.py` | — | Tencent Yuanbao platform tool |

## One-Sentence Summary

Hermes's tool system aggregates 29 self-registering tools through the ToolRegistry self-registration pattern (69 tool-related .py files in the tools/ directory, of which 29 call registry.register() at module level). The Terminal tool supports 7 execution backends (local/docker/modal/ssh/singularity/vercel/daytona), the Browser tool implements CDP headless automation, the Delegate tool spawns sub-agents with MAX_DEPTH=1, the MCP tool connects to external tool servers via stdio/HTTP transport, the Approval system provides hardline/interactive/smart three-layer protection, and ToolGuardrails detects loop calls with progressive blocking.

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

Hermes's tool system is based on the ToolRegistry self-registration pattern — each tool file calls `registry.register()` at module level, automatically declaring schema, handler, toolset, and availability check upon import. The Terminal tool (2342 lines) is the largest built-in tool, supporting 7 execution backends (local/docker/modal/ssh/singularity/vercel_sandbox/daytona) with a spawn-per-call model. The Browser tool (3476 lines) implements headless automation via agent-browser CLI + CDP accessibility tree, supporting Browser Use/Browserbase/Firecrawl cloud providers. The Delegate tool (2562 lines) spawns sub-AIAgent instances, MAX_DEPTH=1 prevents recursive delegation, sub-agents receive a restricted toolset (5 tools blocked). The MCP tool (3175 lines) connects external tool servers via stdio and HTTP/StreamableHTTP transport. The Approval system (1258 lines) provides hardline (unbypassable), interactive (user confirmation), smart (LLM auto-approval) three-layer protection. ToolGuardrails (455 lines) tracks call fingerprints via ToolCallSignature, progressively blocking loop calls through warn→block→halt.

---

## 1. ToolRegistry — Self-registering Discovery Pattern

Source location: `tools/registry.py:1-537`

ToolRegistry is the central hub of the tool system. Each tool file calls `registry.register()` at module level, and model_tools.py queries the registry rather than maintaining its own parallel data structure.

**AST discovery and auto-import** (Source location: `registry.py:29-74`):

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

**ToolEntry data structure** (Source location: `registry.py:77-98`):

```python
class ToolEntry:
    __slots__ = (
        "name", "toolset", "schema", "handler", "check_fn",
        "requires_env", "is_async", "description", "emoji",
        "max_result_size_chars",
    )
```

**check_fn TTL cache** (Source location: `registry.py:102-140`): Availability checks (Docker daemon, Modal SDK, Playwright binary) probe external state, 30-second TTL cache prevents redundant waste.

**Thread safety and generation counter** (Source location: `registry.py:143-160`): MCP dynamic refresh can concurrently modify the registry. `_lock = threading.RLock()` serializes write operations, `_generation` monotonically incrementing counter enables `get_tool_definitions()` and other external consumers to memoize based on generation.

### Comparison Table: ToolRegistry Discovery Mechanism Comparison

| Discovery Method | Trigger Timing | Scope | Thread Safety |
|----------|----------|--------|----------|
| `discover_builtin_tools()` | Startup import | Built-in tool files | RLock + generation |
| MCP `register()` | Connection/refresh time | Dynamic external tools | RLock + generation |
| `discover_builtin_tools()` AST scan | Each startup | Filter non-registering modules | Stateless |

---

## 2. Terminal Tool — 7 Execution Backends

Source location: `tools/terminal_tool.py:1-2342`

Terminal Tool is Hermes's largest built-in tool, supporting 7 execution environments:

**Environment selection** (Source location: `terminal_tool.py:9-13`):

```
TERMINAL_ENV environment variable:
- "local": Execute directly on host (default, fastest)
- "docker": Docker container isolation (requires Docker)
- "modal": Modal cloud sandbox (direct or managed gateway)
- "ssh": SSH remote execution
- "singularity": Singularity container (HPC environments)
- "vercel_sandbox": Vercel Sandbox cloud sandbox
- "daytona": Daytona cloud development environment
```

**Interrupt response mechanism** (Source location: `terminal_tool.py:57-58`): The global `_interrupt_event` is set by the agent when the user interrupts; the terminal tool polls this event during command execution, immediately terminating long-running subprocesses.

```python
from tools.interrupt import is_interrupted, _interrupt_event
```

**Foreground timeout hard limit** (Source location: `terminal_tool.py:106-111`):

```python
FOREGROUND_MAX_TIMEOUT = _safe_parse_import_env(
    "TERMINAL_MAX_FOREGROUND_TIMEOUT",
    600,     # 10-minute hard limit
    int,
    "integer",
)
```

**Vercel Sandbox validation** (Source location: `terminal_tool.py:120-150`): Supports node24/node22/python3.13 runtimes, disk limit of 51200 MB.

**BaseEnvironment — spawn-per-call model** (Source location: `tools/environments/base.py:1-786`):

All backends inherit from `BaseEnvironment` ABC, using a unified spawn-per-call model: each command spawns a fresh `bash -c` process. A session snapshot (env vars, functions, aliases) is captured once at initialization and re-sourced before each command. CWD persists via stdout markers (remote) or temp files (local).

```python
class BaseEnvironment(ABC):
    """Unified spawn-per-call model:
    every command spawns a fresh bash -c process.
    A session snapshot is captured once at init and
    re-sourced before each command."""
```

**Activity callback** (Source location: `base.py:43-78`): Thread-local activity callback, long-running `_wait_for_process` loops periodically report liveness to the gateway.

### Comparison Table: 7 Execution Backend Feature Comparison

| Backend | Lines | Isolation Level | Requires Docker | Cloud | File Sync |
|------|------|----------|------------|------|----------|
| local | 524 | None | No | No | None |
| docker | 645 | Container | Yes | No | volume mount |
| modal | 460 | Cloud sandbox | No | Yes | Modal Volume |
| managed_modal | 282 | Cloud managed | No | Yes | Modal Volume |
| ssh | 295 | Remote host | No | No | rsync/scp |
| singularity | 262 | HPC container | No | No | overlay fs |
| vercel_sandbox | 638 | Cloud sandbox | No | Yes | persistent fs |
| daytona | 259 | Cloud development | No | Yes | workspace fs |

### Comparison Table: Terminal Tool Security Mechanism Comparison

| Security Layer | Implementation Location | Effect | Bypassable |
|--------|----------|------|--------|
| hardline blocklist | `approval.py:116-150` | Blocks unrecoverable commands like rm -rf /, mkfs, shutdown | No (not even with yolo) |
| dangerous patterns | `approval.py` | Matches rm -rf, chmod 777, curl|sh, etc. | Bypassable with yolo |
| approval modes | `approval.py` | interactive/smart/off three modes | yolo=off |
| check_fn gating | `registry.py` | Probes backend availability | No |
| foreground timeout | `terminal_tool.py:106` | 600s hard limit | No |
| interrupt event | `tools/interrupt` | User interrupt immediately terminates subprocess | No |

---

## 3. Browser Tool — CDP Automation

Source location: `tools/browser_tool.py:1-3476`

Browser Tool uses the agent-browser CLI's accessibility tree (ariaSnapshot) to implement text-based page interaction, enabling web operation without visual capabilities.

**Three cloud providers + Camofox local anti-detection** (Source location: `browser_tool.py:73-94`):

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

**PATH discovery mechanism** (Source location: `browser_tool.py:98-144`): Standard PATH supplementation (including Termux, macOS Homebrew), and dynamic discovery of Homebrew versioned Node.js bin directories (node@20, node@24, etc.).

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

**Website safety check** (Source location: `browser_tool.py:74-81`): The `website_policy` module checks access permissions, and the `url_safety` module validates URL safety.

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

### Comparison Table: Browser Tool Backend Comparison

| Backend | Authentication | Location | Stealth | Session Isolation |
|------|------|------|---------|----------|
| Local Chromium | None | Local | Camofox (optional) | per task_id |
| Browser Use | BROWSER_USE_API_KEY | Cloud | Built-in | per task_id |
| Browserbase | BROWSERBASE_API_KEY + PROJECT_ID | Cloud | Advanced Stealth (Scale Plan) | per task_id |
| Firecrawl | FIRECRAWL_API_KEY | Cloud | None | Stateless |

---

## 4. Delegate Tool — Sub-agent Architecture

Source location: `tools/delegate_tool.py:1-2562`

Delegate Tool spawns sub-AIAgent instances with isolated context, restricted toolsets, and independent terminal sessions.

**Core restrictions** (Source location: `delegate_tool.py:39-49`):

```python
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",    # No recursive delegation
    "clarify",          # No user interaction
    "memory",           # No writing to shared MEMORY.md
    "send_message",     # No cross-platform side effects
    "execute_code",     # Sub-agents should reason step-by-step, not write scripts
])

MAX_DEPTH = 1  # flat: parent(0) -> child(1); grandchild rejected
_MIN_SPAWN_DEPTH = 1
_MAX_SPAWN_DEPTH_CAP = 3
```

**Sub-agent approval callback** (Source location: `delegate_tool.py:60-106`): Sub-agents run in ThreadPoolExecutor workers and do not inherit the CLI's interactive approval callback. Default is `_subagent_auto_deny` (safe), optional `_subagent_auto_approve` (YOLO, requires config `delegation.subagent_auto_approve: true`).

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

**Runtime pause and active registration** (Source location: `delegate_tool.py:144-150`): `_spawn_pause_lock` + `_spawn_paused` pause flag, `_active_subagents` registry for TUI observability layer and gateway RPC queries.

```python
_spawn_pause_lock = threading.Lock()
_spawn_paused: bool = False

_active_subagents_lock = threading.Lock()
_active_subagents: Dict[str, Dict[str, Any]] = {}
```

### Comparison Table: Delegate Tool Sub-agent Isolation Comparison

| Aspect | Parent Agent | Sub-agent | Reason |
|------|--------|--------|------|
| Conversation history | Full multi-turn | Only objective + context prompt | Prevent context bloat |
| Toolset | All available | Restricted (5 tools blocked) | Security and functional boundary |
| Approval mode | interactive | auto_deny/auto_approve | Prevent worker thread input() deadlock |
| Delegation depth | Unlimited | MAX_DEPTH=1 (flat) | Prevent recursive delegation |
| terminal session | Independent task_id | Independent task_id | Session isolation |

---

## 5. MCP Tool — Model Context Protocol Client

Source location: `tools/mcp_tool.py:1-3175`

MCP Tool connects to external tool servers via stdio and HTTP/StreamableHTTP transport, discovers their tools, and registers them into the Hermes ToolRegistry.

**Configuration format** (Source location: `mcp_tool.py:13-43`):

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

**Architecture** (Source location: `mcp_tool.py:55-69`): A dedicated background event loop `_mcp_loop` runs in a daemon thread. Each MCP server runs as a long-lived asyncio Task on this loop, keeping transport contexts alive. Tool call coroutines are dispatched to the loop via `run_coroutine_threadsafe()`.

**Stdio stderr redirection** (Source location: `mcp_tool.py:91-143`): The MCP SDK defaults to outputting subprocess stderr to the user TTY, which breaks prompt_toolkit/Rich TUI rendering. Hermes redirects all stdio MCP subprocess stderr to a shared log file `~/.hermes/logs/mcp-stderr.log`, tagged with server names.

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

**Thread safety** (Source location: `mcp_tool.py:66-69`): `_servers` and `_mcp_loop/_mcp_thread` are accessed concurrently from the MCP background thread and calling threads. All modifications are protected by `_lock`, safe regardless of GIL presence (Python 3.13+ free-threading).

### Comparison Table: MCP Transport Mode Comparison

| Transport | Startup Method | Lifecycle | Applicable Scenario | stderr Handling |
|------|----------|----------|----------|------------|
| stdio | subprocess (command + args) | Long-lived Task | Local tool servers | Redirected to mcp-stderr.log |
| HTTP/StreamableHTTP | HTTP POST (url + headers) | Long-lived Task | Remote API services | No subprocess stderr |

---

## 6. Approval System — Dangerous Command Approval

Source location: `tools/approval.py:1-1258`

The Approval system is the single source of truth for Hermes's security protection, providing three-layer protection.

**Hardline unconditional blocklist** (Source location: `approval.py:116-150`): Unbypassable — even yolo mode does not allow execution. Only lists commands with no recovery path: filesystem root-level destruction (`rm -rf /`), block device overwrite (`mkfs`), kernel shutdown (`shutdown`, `reboot`), host denial-of-service commands.

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

**Three approval modes**:

| Mode | Interaction | Applicable Scenario | LLM Involvement |
|------|------|----------|----------|
| interactive | Manual user confirmation | CLI/TUI default | None |
| smart | LLM-assisted auto-approval for low-risk commands | Long-running tasks | auxiliary_client.call_llm |
| off (yolo) | Auto-pass (hardline still blocks) | Trusted agent | None |

**Session context safety** (Source location: `approval.py:30-84`): Gateway runs agent turns concurrently, using `contextvars.ContextVar` instead of global env vars to prevent races.

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

**Sensitive write target detection** (Source location: `approval.py:88-113`): Regex matches sensitive file paths like `$HOME/.ssh`, `.hermes/.env`, `.bashrc/.zshrc`, `.netrc/.pgpass`, triggering approval even when referenced through shell expansion.

### Comparison Table: Approval Three-layer Protection Comparison

| Layer | Mechanism | Bypassable | Coverage Scope |
|------|------|--------|----------|
| hardline | Regex matching of unrecoverable commands | No (not even with yolo) | local/ssh host environments |
| dangerous | Regex matching of high-risk commands | Bypassable with yolo | All non-container backends |
| interactive/smart | User confirmation or LLM auto-approval | Bypassable with off mode | All environments |

---

## 7. ToolGuardrails — Loop Call Detection

Source location: `agent/tool_guardrails.py:1-455`

ToolGuardrailsController tracks tool call fingerprints via `ToolCallSignature`, progressively blocking loop calls.

**Tool classification** (Source location: `tool_guardrails.py:19-59`):

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

**Configuration thresholds** (Source location: `tool_guardrails.py:63-123`):

```python
@dataclass(frozen=True)
class ToolCallGuardrailConfig:
    warnings_enabled: bool = True         # Warnings enabled by default
    hard_stop_enabled: bool = False       # Hard stop disabled by default (requires explicit opt-in)
    exact_failure_warn_after: int = 2     # Warn after 2 same-args failures
    exact_failure_block_after: int = 5    # Block after 5 same-args failures
    same_tool_failure_warn_after: int = 3 # Warn after 3 same-tool different-args failures
    same_tool_failure_halt_after: int = 8 # Halt after 8 same-tool different-args failures
    no_progress_warn_after: int = 2       # Warn after 2 idempotent tool no-progress
    no_progress_block_after: int = 5      # Block after 5 idempotent tool no-progress
```

**ToolCallSignature — irreversible fingerprint** (Source location: `tool_guardrails.py:127-141`):

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

**Progressive decision path** (Source location: `tool_guardrails.py:238-380`):

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

### Comparison Table: Guardrail Decision Type Comparison

| Decision | action | allows_execution | should_halt | Effect |
|------|--------|------------------|-------------|------|
| allow | "allow" | True | False | Normal execution |
| warn | "warn" | True | False | Execute + append warning text |
| block | "block" | False | True | Refuse execution, return synthetic result |
| halt | "halt" | False | True | Refuse execution + stop current turn |

---

## Summary Table

| Component | File | Lines | Core Responsibility |
|------|------|------|----------|
| ToolRegistry | `tools/registry.py` | 537 | Self-registering singleton + AST discovery + generation counter |
| Toolsets | `toolsets.py` | ~786 | Toolset grouping definition |
| Terminal Tool | `tools/terminal_tool.py` | 2342 | 7 execution backend dispatch + interrupt response |
| Browser Tool | `tools/browser_tool.py` | 3476 | CDP headless automation + 3 cloud providers |
| Delegate Tool | `tools/delegate_tool.py` | 2562 | Sub-agent spawning + MAX_DEPTH=1 |
| MCP Tool | `tools/mcp_tool.py` | 3175 | stdio/HTTP transport + sampling support |
| Approval | `tools/approval.py` | 1258 | hardline/interactive/smart three-layer protection |
| ToolGuardrails | `agent/tool_guardrails.py` | 455 | Loop call detection + progressive blocking |
| BaseEnvironment | `tools/environments/base.py` | 786 | spawn-per-call model base class |
| Local Backend | `tools/environments/local.py` | 524 | Host direct execution |
| Docker Backend | `tools/environments/docker.py` | 645 | Container-isolated execution |
| Modal Backend | `tools/environments/modal.py` | 460 | Cloud sandbox execution |
| SSH Backend | `tools/environments/ssh.py` | 295 | Remote host execution |
| Singularity Backend | `tools/environments/singularity.py` | 262 | HPC container execution |
| Vercel Sandbox | `tools/environments/vercel_sandbox.py` | 638 | Cloud sandbox execution |
| Daytona Backend | `tools/environments/daytona.py` | 259 | Cloud development environment execution |

---

[04 — LLM Provider System](/en/chapters/04-model-providers) | [06 — Gateway Messaging Platform](/en/chapters/06-gateway-system)