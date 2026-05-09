# 02 — System Architecture Overview: Four-Layer Architecture and Core Module Mapping

[Previous Chapter](/en/chapters/01-project-overview) | [Next Chapter](/en/chapters/03-agent-engine)

---

## Source Code Locations and Key Files

| File | Lines | Responsibility |
|------|------|------|
| `run_agent.py` | 14,404 | Core agent engine — AIAgent class, conversation loop, tool dispatch, streaming, failover |
| `gateway/run.py` | 15,046 | Message gateway runner — GatewayRunner, 21+ platform adapters, session cache, lifecycle management |
| `hermes_cli/main.py` | 10,876 | CLI entry — argparse routing, profile override, subcommand dispatch, setup/model/tools/doctor |
| `tools/registry.py` | 537 | Tool registry — ToolRegistry singleton, ToolEntry metadata, check_fn TTL cache, AST self-discovery |
| `hermes_state.py` | 2,669 | Session persistence — SessionDB, SQLite WAL, FTS5 full-text search, message storage and retrieval |
| `hermes_cli/config.py` | 4,939 | Configuration management — config.yaml parsing, .env loading, DEFAULT_CONFIG, credential routing |
| `agent/context_compressor.py` | 1,481 | Context compression engine — conversation history compression, focus_topic-guided compression, session split |
| `agent/curator.py` | 1,674 | Self-review fork agent — fork sub-agent performs self-review and improvement suggestions on output |
| `agent/prompt_builder.py` | 1,186 | System prompt builder — dynamic system prompt assembly, skill/memory/todo injection |
| `agent/tool_guardrails.py` | 455 | Tool call guardrails — loop detection, progressive warn→block→halt escalation |
| `agent/error_classifier.py` | 1,036 | API error classification — 18 FailoverReason categories and recovery path decision |
| `agent/credential_pool.py` | 1,584 | Credential pool management — 4 rotation strategies, exhausted cooldown, OAuth refresh |
| `agent/redact.py` | — | Output sanitization — secret redaction and sanitization in logs/output |

**One-line summary:** Hermes Agent adopts a four-layer architecture (Frontend→CLI→Engine→Tools/Storage), with six core modules each serving their own purpose. The engine layer runs independently of the frontend, and the gateway process serves multiple platforms in parallel. The `agent/` directory (54 files, ~20K lines) contains key supporting subsystems of the engine layer.

---

## Architecture Overview

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
│  │ (10,876 lines)   │                   │    (15,046 lines)        │ │
│  │                   │                   │                           │ │
│  │ • argparse routing│                   │ • GatewayRunner class    │ │
│  │ • profile override│                   │ • 21+ platform adapter mg │ │
│  │ • .env loading    │                   │ • AIAgent instance cache │ │
│  │ • subcommand dispatch│                │ • session lifecycle/timeout│ │
│  │ • config bridging │                   │ • auto-continue recovery │ │
│  └──────┬───────────┘                    └──────────┬────────────────┘ │
│         │                                           │                 │
├─────────┼───────────────────────────────────────────┼─────────────────┤
│         ▼                                           ▼                 │
│                     Layer 3: Agent Engine                            │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                     run_agent.py (14,404 lines)                  │ │
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
│  │ (537 lines)  │  │ (2,669 lines)│  │  (4,939 lines)│            │
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

Hermes Agent adopts a strict four-layer architecture: the frontend layer (Ink TUI / curses TUI / Web Dashboard / ACP) connects through the CLI or gateway entry layer to the core engine layer (`run_agent.py`'s AIAgent), and the engine layer calls the tools/storage layer (ToolRegistry / SessionDB / Config). The engine runs independently of the frontend — in CLI mode, `hermes_cli/main.py` builds a single AIAgent instance for single-user interaction, while in gateway mode, `gateway/run.py`'s GatewayRunner maintains an LRU cache pool serving multiple platforms and users in parallel. The six core modules total 48,471 lines, with the engine layer's single file of 14,404 lines carrying all agent logic.

---

## 1. Four-Layer Architecture in Detail

### 1.1 Layer 1: Frontend

The frontend layer provides user interaction interfaces. The four frontends are each independent, connecting to the engine through different entry points:

| Frontend | Tech Stack | Entry Point | Features |
|------|-------- |------|------|
| Ink/React TUI | React, Ink, TypeScript | `ui-tui/` + `tui_gateway/` (JSON-RPC) | Sticky composer, OSC-92 clipboard, LaTeX, light-theme |
| curses TUI | Python prompt_toolkit, curses | `hermes_cli/main.py` | Classic CLI, slash commands, autocomplete |
| Web Dashboard | React, Vite | `web/` + `hermes_api_server.py` | i18n, mobile-responsive, plugin tabs, live theme |
| ACP (IDE) | Agent Communication Protocol | `hermes acp` | VS Code, Zed, JetBrains slash commands |

**Frontend-to-engine interaction patterns:**

```text
CLI frontend:
  hermes_cli/main.py → build single AIAgent → run_conversation() → callback-driven UI updates

Gateway frontend:
  gateway/run.py → GatewayRunner._agent_cache → get or build AIAgent per message → run_conversation()

ACP frontend:
  hermes acp → ACPAdapter → shared AIAgent instance → run_conversation() via SSE

TUI frontend:
  ui-tui/ (Node) → tui_gateway (Python JSON-RPC) → AIAgent.run_conversation()
```

### 1.2 Layer 2: CLI / Gateway Entry

#### hermes_cli/main.py (10,876 lines)

The CLI entry layer's core responsibility is argument parsing and environment initialization. The most critical operation happens before module imports:

```python
# Source location: hermes_cli/main.py:101-176

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

Profile override must execute before all Hermes module imports, because many modules cache `HERMES_HOME` at import time (module-level constants). `_apply_profile_override()` intercepts `--profile/-p` from `sys.argv` and sets the `HERMES_HOME` environment variable, then strips the flag from argv to avoid argparse conflicts.

Next, .env is loaded and `config.yaml`'s `security.redact_secrets` is bridged to the environment variable:

```python
# Source location: hermes_cli/main.py:179-200

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

CLI subcommand routing uses argparse:

```python
# Source location: hermes_cli/main.py:1-44 (usage docstring)

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

#### gateway/run.py (15,046 lines)

The gateway entry layer is the core manager when Hermes Agent runs in message platform mode. The GatewayRunner class (L988) maintains the following key state:

```text
GatewayRunner core data structures:
  • _agent_cache: Dict[str, AIAgent] — LRU cache pool (max 128, idle TTL 1h)
  • _session_store: SessionStore — thread-safe session mapping
  • _platforms: Dict[str, BasePlatformAdapter] — 21+ message platform adapter instances
  • _running_agents: Dict[str, threading.Thread] — active agent thread tracking
  • _background_tasks: List — background task reference tracking
  • _gateway_task_refs: Set — asyncio task references to prevent GC
```

Source location: `gateway/run.py:1-200` (module header, helper functions, timestamp parsing)

**Gateway's independent parallel execution mechanism:**

```text
Gateway process architecture:
┌──────────────────────────────────────────────────┐
│              GatewayRunner (main thread)          │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Telegram │  │ Discord  │  │ 17 other      │  │
│  │ adapter  │  │ adapter  │  │ adapters      │  │
│  └──────────┘  └──────────┘  └──────────────┘  │
│        │             │              │            │
│        └► message events ──► _handle_message() ──►  │
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
│  │ Parallel: each platform adapter runs independent asyncio loop │
│  │ Shared: SessionDB (SQLite WAL)          │   │
│  │ Cached: AIAgent instances (LRU + idle TTL) │   │
│  └──────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

Gateway's key design features:

- **AIAgent cache pool**: LRU + idle TTL eviction, upper limit of 128 instances. Cache signature includes the full auth token fingerprint, ensuring automatic rebuild when credentials change
- **Auto-continue**: Automatically resumes interrupted sessions after gateway restart, identifying turns that need continuation through `tool-tail` and `resume_pending` markers, with a freshness window (default 1h)
- **Agent cache cap**: `_enforce_agent_cache_cap()` and `_session_expiry_watcher()` evict cached instances from both the LRU order and idle TTL dimensions respectively

Source location: `gateway/run.py:46-53` (cache tuning parameters), `gateway/run.py:76-94` (auto-continue mechanism)

### 1.3 Layer 3: Agent Engine

The engine layer is the core of Hermes Agent, entirely contained within `run_agent.py`'s AIAgent class. For detailed analysis, see [Next Chapter](/en/chapters/03-agent-engine).

Key features:

| Subsystem | Location | Line Range | Responsibility |
|--------|------|----------|------|
| AIAgent class | L885 | 885-14,404 | Main agent class, ~13,500 lines |
| IterationBudget | L272 | 272-311 | Thread-safe iteration counter, shared budget |
| run_conversation | L10573 | 10573-14,404 | Full conversation loop entry point |
| _interruptible_api_call | L6460 | 6460-6700 | Background thread API call + interrupt detection |
| _try_activate_fallback | L7633 | 7633-7700 | Ordered fallback provider chain activation |
| _restore_primary_runtime | L7831 | 7831-7900 | Turn-scoped primary runtime restoration |
| _compress_context | L9221 | 9221-9400 | Context compression + SQLite session split |
| _run_tool | L9716 | 9716-9800 | Concurrent tool execution worker thread |

**agent/ directory key supporting modules (54 files, ~20K lines):**

| Submodule | Lines | Responsibility |
|--------|------|------|
| `agent/context_compressor.py` | 1,481 | Conversation history compression, focus_topic-guided, session split |
| `agent/curator.py` | 1,674 | Fork sub-agent self-review and improvement |
| `agent/prompt_builder.py` | 1,186 | Dynamic system prompt assembly |
| `agent/tool_guardrails.py` | 455 | Tool call loop detection and progressive blocking |
| `agent/error_classifier.py` | 1,036 | 18 FailoverReason categories and recovery paths |
| `agent/credential_pool.py` | 1,584 | Credential rotation (4 strategies), exhausted cooldown |
| `agent/redact.py` | — | Output/log secret redaction |

### 1.4 Layer 4: Tools / Storage

#### ToolRegistry (tools/registry.py, 537 lines)

The tool registry adopts a self-registration pattern: each tool file calls `registry.register()` at the module level to declare its schema, handler, toolset, and availability check. `model_tools.py` queries the registry rather than maintaining its own parallel data structure.

```python
# Source location: tools/registry.py:57-74

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

Key innovation: AST detection — `_module_registers_tools()` uses `ast.parse` to check if the module body has top-level `registry.register(...)` calls, avoiding importing registration calls that happen to exist in helper modules. `check_fn` TTL cache (30s) prevents long-lifetime processes from repeatedly probing external state (Docker daemon, Modal SDK, playwright binary).

```python
# Source location: tools/registry.py:143-160

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

ToolEntry uses `__slots__` for compact storage, containing fields such as name, toolset, schema, handler, check_fn, requires_env, is_async, description, emoji, max_result_size_chars, etc.

#### SessionDB (hermes_state.py, 2,669 lines)

SessionDB (L159) uses SQLite WAL mode for session persistence, supporting FTS5 full-text search:

```text
SessionDB core capabilities:
  • SQLite WAL write-lock contention fix (v0.5: #3385)
  • FTS5 full-text search across session content
  • Message storage and retrieval (unified across CLI/Gateway/ACP)
  • Session naming and lineage tracking
  • Reasoning persistence (v0.5: schema v6 new columns)
  • Thread-safe concurrent access
```

Source location: `hermes_state.py:159` (SessionDB class definition)

#### Config System (hermes_cli/config.py, 4,939 lines)

The configuration system manages config.yaml parsing, .env loading, and DEFAULT_CONFIG definition:

```text
Config core capabilities:
  • config.yaml as single source of truth for endpoint URLs (v0.7: #4165)
  • .env loading takes precedence over shell exports (v0.3: reload .env over stale shell overrides)
  • Config migration system (currently v7)
  • YAML null value protection (v0.5: #3377)
  • Atomic writes to prevent data loss
  • Environment variable bridging (security.redact_secrets → HERMES_REDACT_SECRETS)
```

Source location: `hermes_cli/config.py:1-4,939`

---

## 2. Module Dependency Graph

```text
                            ┌──────────────┐
                            │  README.md   │
                            │  (user entry)│
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
  │ (29 tools│        │              │              │
  │  self-registering) │              │              │
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

Dependency rules:

1. `run_agent.py` does not import `hermes_cli/main.py` or `gateway/run.py` (engine layer is independent)
2. `tools/*.py` imports `tools/registry.py` at module level (self-registration, no circular dependencies)
3. `model_tools.py` imports `tools.registry` + all tool modules (bridge layer)
4. `hermes_state.py` does not import `run_agent.py` (storage layer is independent)
5. `hermes_cli/config.py` does not import `run_agent.py` (configuration layer is independent)
6. `agent/transports/` (v0.11+) was extracted from `run_agent.py`, forming a pluggable transport layer

Source location: `tools/registry.py:7-14` (import chain documentation)

---

## 3. Six Core Modules in Detail

### 3.1 run_agent.py (14,404 lines) — Core Agent Engine

`run_agent.py` is the heart of the entire system, a single file carrying the complete agent logic:

```python
# Source location: run_agent.py:885-973

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

AIAgent.__init__ accepts 50+ parameters, including inference configuration (base_url, api_key, provider, api_mode, model, max_iterations, max_tokens, reasoning_config), tool configuration (enabled_toolsets, disabled_toolsets), 11 callbacks (tool_progress_callback, tool_start_callback, tool_complete_callback, thinking_callback, reasoning_callback, clarify_callback, step_callback, stream_delta_callback, interim_assistant_callback, tool_gen_callback, status_callback), platform context (platform, user_id, chat_id, thread_id), session management (session_db, parent_session_id, iteration_budget), failover (fallback_model, credential_pool), and security configuration (checkpoints_enabled).

### 3.2 gateway/run.py (15,046 lines) — Message Gateway Runner

GatewayRunner (L988) is the core management class for gateway mode:

```text
GatewayRunner core methods:
  • start() — launch all configured platform adapters
  • _handle_message() — message event → AIAgent.run_conversation()
  • _get_or_create_agent() — LRU cache lookup/build AIAgent
  • _enforce_agent_cache_cap() — LRU + idle TTL eviction
  • _session_expiry_watcher() — background thread detecting idle sessions
  • _auto_continue_interrupted_turn() — resume interrupted sessions after restart
```

The gateway's key design is **cache signature**: full auth token fingerprint ensures automatic rebuild of AIAgent instances when credentials change (v0.5: #3247).

### 3.3 hermes_cli/main.py (10,876 lines) — CLI Entry

The CLI entry's unique aspect is that **profile override must execute before module imports**:

```python
# Source location: hermes_cli/main.py:92-99

# Profile override — MUST happen before any hermes module import.
#
# Many modules cache HERMES_HOME at import time (module-level constants).
# We intercept --profile/-p from sys.argv here and set the env var so that
# every subsequent ``os.getenv("HERMES_HOME", ...)`` resolves correctly.
# The flag is stripped from sys.argv so argparse never sees it.
```

### 3.4 tools/registry.py (537 lines) — Tool Registry

The tool registry's core innovations:

| Mechanism | Implementation | Significance |
|------|------|------|
| AST self-discovery | `_module_registers_tools()` | Only imports modules containing top-level `registry.register()` |
| check_fn TTL cache | 30s TTL, thread-safe | Long-lifetime processes don't repeatedly probe external state |
| Generation counter | `_generation: int` | Cache memoization key, bumped on mutation |
| Thread-safe RLock | `threading.RLock` | MCP dynamic refresh can concurrently read/write the registry |
| Snapshot state | `_snapshot_state()` | Read consistency, external code gets a stable snapshot |

### 3.5 hermes_state.py (2,669 lines) — Session Persistence

SessionDB (L159) uses SQLite WAL mode to resolve concurrent write conflicts:

```text
SessionDB key fix history:
  v0.2: Message duplication 3-4x (token bloat) → fixed transcript offset
  v0.5: WAL write-lock contention (15-20s TUI freeze) → fixed
  v0.5: Schema v6 added reasoning, reasoning_details, codex_reasoning_items columns
  v0.9: Thread-safe SessionStore (threading.Lock protecting _entries)
```

### 3.6 hermes_cli/config.py (4,939 lines) — Configuration Management

The configuration system's key designs:

| Design | Implementation | Version |
|------|------|------|
| config.yaml single source of truth | Endpoint URLs no longer conflict between env var and config.yaml | v0.7 |
| .env priority | `load_hermes_dotenv()` prioritizes user .env | v0.3 |
| Config migration | v7 migration system, auto-upgrades old formats | v0.2+ |
| YAML null protection | `config.get()` doesn't crash on YAML null | v0.5 |
| Atomic writes | config.yaml uses atomic writes to prevent data loss | v0.6 |
| Env bridging | security.redact_secrets → HERMES_REDACT_SECRETS | various versions |

---

## 4. Data Flow: From Message to Response

### 4.1 CLI Mode Data Flow

```text
User input ──► prompt_toolkit readline ──► slash command check
  │                                        │
  │ (slash command)                        │ (normal message)
  │                                        │
  ▼                                        ▼
Command routing ──► built-in handling       AIAgent.run_conversation(user_message)
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
                                     Response parsing → tool_calls?
                                              │
                              ┌───────────────┼───────────────┐
                              │               │               │
                              ▼               ▼               ▼
                         Text response    Tool calls → concurrent execution    Empty response → retry
                         (return to user) (ThreadPoolExecutor)                   (nudge + retry)
                                              │
                                              ▼
                                     Tool results → append to messages
                                              │
                                              ▼
                                     Continue loop (until no tool_calls
                                     or max_iterations exhausted)
```

### 4.2 Gateway Mode Data Flow

```text
Platform message ──► BasePlatformAdapter.receive() ──► GatewayRunner._handle_message()
                                                    │
                                                    ▼
                                          _get_or_create_agent(session_key)
                                                    │
                                    ┌───────────────┼───────────────┐
                                    │               │               │
                                    ▼               ▼               ▼
                              Cache hit:         Cache miss:        Idle TTL:
                              Return existing    Build new instance  Evict old instance
                              instance (LRU)     (credential parse)  (reclaim resources)
                                                    │
                                                    ▼
                                          AIAgent.run_conversation(msg)
                                                    │
                                          (same as CLI data flow, but)
                                                    │
                              ┌───────────────┼───────────────┐
                              │               │               │
                              ▼               ▼               ▼
                         Callback-driven     status_callback   stream_delta_
                         UI updates          platform msg send  callback streaming
                                                    │
                                                    ▼
                                          Platform adapter.send_message()
                                          (Telegram/Discord/Slack/...)
```

---

## 5. Gateway Independent Parallel Execution

### 5.1 Parallel Relationship Between Gateway and CLI

```text
┌────────────────────────────────────────────────────────────┐
│                    Running Mode Comparison                  │
│                                                            │
│  CLI mode:                                                 │
│  hermes ──► hermes_cli/main.py ──► single AIAgent ──► single user │
│  (foreground process, blocking interaction)                │
│                                                            │
│  Gateway mode:                                             │
│  hermes gateway start ──► systemd service ──► 21+ platforms parallel │
│  (background daemon, event-driven, multi-user multi-platform) │
│                                                            │
│  Key differences:                                          │
│  • CLI: 1 AIAgent, 1 session, foreground interaction      │
│  • Gateway: LRU cache pool, multi-session, background daemon │
│  • Gateway: auto-continue restart recovery                 │
│  • Gateway: platform adapters each run independent asyncio loop │
│  • CLI: callbacks directly drive UI                       │
│  • Gateway: callbacks drive platform message sending      │
└────────────────────────────────────────────────────────────┘
```

### 5.2 Gateway Parallel Architecture

Each of Gateway's platform adapters runs in an independent asyncio loop, coordinating through the shared GatewayRunner and SessionDB:

| Dimension | CLI Mode | Gateway Mode |
|------|----------|--------------|
| AIAgent instances | Single instance, rebuilt each time | LRU cache pool (max 128) |
| User count | 1 | Multiple (per-session) |
| Concurrency model | Synchronous blocking | Async event-driven |
| Session management | In-memory + SessionDB | SessionDB (SQLite WAL) |
| Lifecycle | Destroyed on process exit | systemd daemon, restart recovery |
| Message delivery | Direct stdout | Platform adapter.send_message() |
| Approval mechanism | CLI input prompt | Platform buttons (Slack/Telegram) |
| Timeout management | Unlimited | Inactivity-based (tool activity tracking) |
| Cache signature | None | Auth token fingerprint |

---

## 6. Module Comparison Table

### 6.1 Core Module Responsibility Comparison

| Module | Core Class | Thread-safe | State Persistence | Concurrency Model | Cache Mechanism |
|------|--------|----------|----------|----------|----------|
| `run_agent.py` | AIAgent | IterationBudget (Lock) | SessionDB | ThreadPool (tool) | transport + client |
| `gateway/run.py` | GatewayRunner | SessionStore (Lock) | SQLite WAL | asyncio per platform | LRU + idle TTL |
| `hermes_cli/main.py` | argparse | No | config.yaml/.env | Synchronous | No |
| `tools/registry.py` | ToolRegistry | RLock | No | thread-safe mutations | check_fn TTL 30s |
| `hermes_state.py` | SessionDB | WAL lock | SQLite | WAL concurrent | No |
| `hermes_cli/config.py` | DEFAULT_CONFIG | No | config.yaml | Synchronous | mtime-cached (v0.12) |

### 6.2 Frontend Mode Comparison

| Frontend | Transport Protocol | Message Format | Streaming Support | Session Model | Independent Process |
|------|----------|----------|----------|----------|----------|
| Ink TUI | JSON-RPC | JSON | OSC-92 clipboard | per-thread | Node.js subprocess |
| curses TUI | Direct Python | In-memory | patch_stdout | per-process | No |
| Web Dashboard | HTTP SSE | JSON | SSE events | per-request | FastAPI server |
| ACP | SSE + HTTP | ACP protocol | SSE | per-session | No |
| Gateway | platform-specific | Markdown/HTML | stream_delta | per-session-key | systemd daemon |

### 6.3 Transport Layer Comparison (v0.11+)

| Transport | Provider | API Format | Format Conversion | Streaming | Usage |
|-----------|--------|----------|----------|------|------|
| AnthropicTransport | Anthropic | Messages API | Anthropic→OpenAI | SSE | Claude direct |
| ChatCompletionsTransport | OpenAI-compatible | Chat Completions | Native | SSE | Default path |
| ResponsesApiTransport | Codex | Responses API | Codex→OpenAI | SSE | GPT/Codex |
| BedrockTransport | AWS | Converse API | Bedrock→OpenAI | No | AWS Bedrock |

---

## 7. Transport Layer Evolution

Before v0.11, all format conversion and HTTP transport logic was embedded in `run_agent.py`. v0.11 extracted these into a pluggable Transport ABC:

```text
run_agent.py v0.10 (embedded logic):
  ┌─────────────────────────────────────────┐
  │ AIAgent                                 │
  │  • Anthropic format conversion           │
  │  • OpenAI chat.completions call          │
  │  • Codex responses.stream() call         │
  │  • Bedrock Converse call (embedded when added) │
  │  • Stream consumption + SSE parsing      │
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

## Summary Table

| Component | Lines | Responsibility |
|------|------|------|
| `run_agent.py` | 14,404 | Core agent engine (AIAgent, conversation loop, tool dispatch, streaming, failover, compression) |
| `gateway/run.py` | 15,046 | Message gateway runner (GatewayRunner, 21+ platform adapters, LRU cache, auto-continue) |
| `hermes_cli/main.py` | 10,876 | CLI entry (argparse routing, profile override, .env loading, subcommand dispatch) |
| `tools/registry.py` | 537 | Tool registry (ToolRegistry singleton, AST self-discovery, check_fn TTL, generation) |
| `hermes_state.py` | 2,669 | Session persistence (SessionDB, SQLite WAL, FTS5 search, reasoning persistence) |
| `hermes_cli/config.py` | 4,939 | Configuration management (config.yaml parsing, .env loading, migration, atomic writes) |
| `agent/context_compressor.py` | 1,481 | Context compression engine (conversation compression, focus_topic, session split) |
| `agent/curator.py` | 1,674 | Self-review fork agent (output self-review and improvement suggestions) |
| `agent/prompt_builder.py` | 1,186 | System prompt builder (dynamic prompt assembly) |
| `agent/tool_guardrails.py` | 455 | Tool call guardrails (loop detection, progressive blocking) |
| `agent/error_classifier.py` | 1,036 | API error classification (18 FailoverReason, recovery paths) |
| `agent/credential_pool.py` | 1,584 | Credential pool management (4 rotation strategies, exhausted cooldown) |
| `agent/redact.py` | — | Output sanitization (secret redaction/sanitization) |
| Core modules total | 48,471 | All core logic of the four-layer architecture |

---

[Previous Chapter: 01 — Project Overview and Design Philosophy](/en/chapters/01-project-overview) | [Next Chapter: 03 — Core Agent Engine AIAgent](/en/chapters/03-agent-engine)