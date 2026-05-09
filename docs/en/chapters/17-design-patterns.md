# 17 — Design Patterns: Transferable Reusable Architecture Patterns


---

## Source Files

| File | Lines | Role |
|------|-------|------|
| `tools/registry.py` | 537 | Pattern 1: Self-registration tool discovery |
| `agent/error_classifier.py` | 1036 | Pattern 2: Error classification + failover |
| `agent/credential_pool.py` | 1584 | Pattern 2: Multi-credential pool + rotation strategy |
| `providers/base.py` | 165 | Pattern 2: ProviderProfile declarative description |
| `agent/context_compressor.py` | 1481 | Pattern 3: Context compression window |
| `agent/context_engine.py` | 206 | Pattern 3: ContextEngine ABC |
| `gateway/platforms/base.py` | 3390 | Pattern 4: Platform adapter ABC |
| `gateway/platform_registry.py` | 212 | Pattern 4: Platform self-registration registry |
| `hermes_cli/config.py` | 4939 | Pattern 5: Four-source config layer merge |
| `hermes_cli/plugins.py` | 1362 | Pattern 6: Skill source ABC |
| `hermes_constants.py` | 345 | Pattern 5: Hardcoded constants |

**One-liner:** 4 core transferable design patterns + 3 additional patterns: self-registration tool discovery (zero-coupling registry), multi-model failover cascade (error classification + credential pool + ProviderProfile), context compression window (ContextEngine ABC + head/tail protection), platform adapter ABC (unified abstraction + self-registration registry), plus config layer merge, skill source ABC, execution backend ABC.

---

## Architecture Overview

```
                    ┌───────────────────────────────────────────────────┐
                    │        Pattern 1: Self-Registration Registry      │
                    │                                                   │
                    │  tools/*.py ──→ registry.register() at import    │
                    │  model_tools.py ──→ registry.get_definitions()   │
                    │                                                   │
                    │  Zero coupling: tool files never import           │
                    │  model_tools or each other                        │
                    └───────────────────────────────────────────────────┘

                    ┌───────────────────────────────────────────────────┐
                    │   Pattern 2: Multi-Model Failover Cascade         │
                    │                                                   │
                    │  API call fails ──→ error_classifier.classify()  │
                    │       ↓                                            │
                    │  ClassifiedError ──→ recovery action hints        │
                    │       ↓                                            │
                    │  FailoverReason ──→ recovery strategy             │
                    │       ↓                                            │
                    │  credential_pool ──→ rotate/round_robin/fill_first│
                    │       ↓                                            │
                    │  ProviderProfile ──→ declarative provider quirks │
                    └───────────────────────────────────────────────────┘

                    ┌───────────────────────────────────────────────────┐
                    │   Pattern 3: Context Compression Window            │
                    │                                                   │
                    │  ContextEngine ABC (6 implementations):           │
                    │  ├── name, update_from_response, should_compress  │
                    │  ├── compress(messages, current_tokens, focus)    │
                    │  ├── on_session_start/end/reset                   │
                    │  └── get_tool_schemas + handle_tool_call          │
                    │                                                   │
                    │  ContextCompressor:                                │
                    │  protect_first_n + protect_last_n                  │
                    │  → LLM summary of middle turns                     │
                    │  → SUMMARY_PREFIX handoff framing                  │
                    └───────────────────────────────────────────────────┘

                    ┌───────────────────────────────────────────────────┐
                    │   Pattern 4: Platform Adapter ABC                  │
                    │                                                   │
                    │  BasePlatformAdapter(ABC):                        │
                    │  ├── connect(), send_message(), handle_message()  │
                    │  ├── send_typing(), _keep_typing()                │
                    │  ├── _fatal_error handling + auto-recovery        │
                    │  └── _post_delivery_callbacks for async hooks     │
                    │                                                   │
                    │  platform_registry:                                │
                    │  ├── PlatformEntry(name, label, adapter_factory)  │
                    │  ├── register() → self-registration               │
                    │  └── create_adapter(name, config) → instance      │
                    └───────────────────────────────────────────────────┘

       ┌────────────────────────────────────────────────────────────────┐
       │              Additional Patterns                                │
       │                                                                │
       │  Pattern 5: Config Layer Merge (4-source priority)            │
       │  Pattern 6: Skill Source ABC (6 implementations)              │
       │  Pattern 7: Execution Backend ABC (7 environments)            │
       └────────────────────────────────────────────────────────────────┘
```

---

## TL;DR

4 core transferable design patterns solve key architectural problems in Hermes-Agent: Pattern 1 self-registration tool discovery achieves zero-coupling tool → registry → consumer import chain, eliminating the burden of maintaining parallel data structures; Pattern 2 multi-model failover cascade implements intelligent failure recovery through four-layer coordination: ClassifiedError structured classification + FailoverReason taxonomy + credential_pool strategy rotation + ProviderProfile declarative description (with 5 hook methods); Pattern 3 context compression window abstracts compression strategy via ContextEngine ABC, protecting head/tail turns + LLM summary of middle + handoff framing; Pattern 4 platform adapter ABC unifies messaging platform interfaces + platform_registry self-registration eliminates if/elif chains, with fatal error handling and post-delivery callback integration points. 3 additional patterns cover config layer merge, skill source discovery, and execution backend abstraction.

---

## Pattern 1: Self-Registration Tool Discovery (Zero-Coupling)

### Scenario

In a system with 40+ tools, how can each tool self-declare its schema, handler, and availability without introducing any coupling between tool files?

### Practice

**Core mechanism:** Each tool file calls `registry.register()` at module level, the consumer (`model_tools.py`) only imports `tools.registry` + all tool modules, and there are zero dependencies between tool files.

**Source location: tools/registry.py:1-15**

```python
"""Central registry for all hermes-agent tools.

Import chain (circular-import safe):
    tools/registry.py  (no imports from model_tools or tool files)
           ^
    tools/*.py  (import from tools.registry at module level)
           ^
    model_tools.py  (imports tools.registry + all tool modules)
           ^
    run_agent.py, cli.py, batch_runner.py, etc.
"""
```

### Implementation: ToolEntry

**Source location: tools/registry.py:77-98**

```python
class ToolEntry:
    """Metadata for a single registered tool."""
    __slots__ = (
        "name", "toolset", "schema", "handler", "check_fn",
        "requires_env", "is_async", "description", "emoji",
        "max_result_size_chars",
    )
```

### Implementation: ToolRegistry Singleton

**Source location: tools/registry.py:143-159**

```python
class ToolRegistry:
    """Singleton registry that collects tool schemas + handlers from tool files."""

    def __init__(self):
        self._tools: Dict[str, ToolEntry] = {}
        self._toolset_checks: Dict[str, Callable] = {}
        self._toolset_aliases: Dict[str, str] = {}
        self._lock = threading.RLock()
        # Monotonically-increasing generation counter for memoization
        self._generation: int = 0

    def _snapshot_state(self) -> tuple[List[ToolEntry], Dict[str, Callable]]:
        """Return a coherent snapshot of registry entries and toolset checks."""
        with self._lock:
            return list(self._tools.values()), dict(self._toolset_checks)
```

### Implementation: AST Discovery

`discover_builtin_tools()` uses AST parsing to detect which tool files contain `registry.register()` calls, avoiding blind imports.

**Source location: tools/registry.py:29-74**

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
    """Import built-in self-registering tool modules."""
    tools_path = Path(tools_dir) if tools_dir else Path(__file__).resolve().parent
    module_names = [
        f"tools.{path.stem}"
        for path in sorted(tools_path.glob("*.py"))
        if path.name not in {"__init__.py", "registry.py", "mcp_tool.py"}
        and _module_registers_tools(path)  # AST check
    ]
```

### Implementation: check_fn TTL Cache

check_fn results (e.g. Docker daemon detection, SDK installation detection) are cached for 30 seconds, avoiding frequent external state probing.

**Source location: tools/registry.py:113-133**

```python
_CHECK_FN_TTL_SECONDS = 30.0
_check_fn_cache: Dict[Callable, tuple[float, bool]] = {}

def _check_fn_cached(fn: Callable) -> bool:
    """Return bool(fn()), TTL-cached across calls."""
    now = time.monotonic()
    with _check_fn_cache_lock:
        cached = _check_fn_cache.get(fn)
        if cached is not None:
            ts, value = cached
            if now - ts < _CHECK_FN_TTL_SECONDS:
                return value
    try:
        value = bool(fn())
    except Exception:
        value = False
    with _check_fn_cache_lock:
        _check_fn_cache[fn] = (now, value)
    return value
```

### Application

Any system requiring dynamic discovery and registration capabilities can use this pattern: MCP tool registration, plugin tool registration, RL environment discovery (AST scanning for BaseEnv subclasses).

| Aspect | Hermes Usage | Transferable Scenario |
|--------|-------------|---------------------|
| Registration | `tools/*.py → registry.register()` | Plugin system, MCP tools, RL envs |
| Discovery | AST scan for `registry.register()` | Any auto-registration system |
| Caching | check_fn TTL + generation counter | Mutable registry memoization |
| Thread Safety | RLock + snapshot pattern | Concurrent read/mutate registries |

---

## Pattern 2: Multi-Model Failover Cascade

### Scenario

When an API call fails, how can you intelligently determine the failure cause and select the correct recovery strategy (retry vs credential rotation vs provider cascade vs context compression)?

### Practice

Four-layer coordinated design: ClassifiedError structured classification → FailoverReason taxonomy → credential_pool strategy → ProviderProfile declarative description.

### Layer 0: ClassifiedError — Structured Error Output

**Source location: agent/error_classifier.py:67-85**

```python
@dataclass
class ClassifiedError:
    """Structured classification of an API error with recovery hints."""

    reason: FailoverReason
    status_code: Optional[int] = None
    provider: Optional[str] = None
    model: Optional[str] = None
    message: str = ""
    error_context: Dict[str, Any] = field(default_factory=dict)

    # Recovery action hints — the retry loop checks these instead of
    # re-classifying the error itself.
    retryable: bool = True
    should_compress: bool = False
    should_rotate_credential: bool = False
    should_fallback: bool = False

    @property
    def is_auth(self) -> bool:
        return self.reason in (FailoverReason.auth, FailoverReason.auth_permanent)
```

ClassifiedError is the core output of Pattern 2 — the `classify()` function returns ClassifiedError instances, and the retry loop directly reads `retryable`/`should_compress`/`should_rotate_credential`/`should_fallback` attributes to determine recovery actions without re-classifying.

### Layer 1: FailoverReason Taxonomy

**Source location: agent/error_classifier.py:24-61**

```python
class FailoverReason(enum.Enum):
    """Why an API call failed — determines recovery strategy."""
    # Authentication / authorization
    auth = "auth"                        # Transient auth (401/403) — refresh/rotate
    auth_permanent = "auth_permanent"    # Auth failed after refresh — abort

    # Billing / quota
    billing = "billing"                  # 402 or confirmed credit exhaustion — rotate
    rate_limit = "rate_limit"            # 429 — backoff then rotate

    # Server-side
    overloaded = "overloaded"            # 503/529 — provider overloaded, backoff
    server_error = "server_error"        # 500/502 — retry

    # Transport
    timeout = "timeout"                  # Connection/read timeout — rebuild + retry

    # Context / payload
    context_overflow = "context_overflow"  # Context too large — compress, NOT failover
    payload_too_large = "payload_too_large"  # 413 — compress payload
    image_too_large = "image_too_large"   # Native image part exceeds provider limit — shrink + retry

    # Model
    model_not_found = "model_not_found"  # 404 — fallback to different model
    provider_policy_blocked = "provider_policy_blocked"  # Aggregator blocked endpoint — failover

    # Request format
    format_error = "format_error"        # 400 bad request — abort or strip + retry

    # Provider-specific
    thinking_signature = "thinking_signature"  # Anthropic thinking block sig invalid
    long_context_tier = "long_context_tier"    # Anthropic "extra usage" tier gate
    oauth_long_context_beta_forbidden = "oauth_long_context_beta_forbidden"  # Anthropic OAuth rejects 1M context beta — disable and retry
    llama_cpp_grammar_pattern = "llama_cpp_grammar_pattern"  # llama.cpp json-schema-to-grammar rejects regex — strip from tools and retry

    # Unknown
    unknown = "unknown"                  # Unclassifiable — retry with backoff
```

Key design decision: `context_overflow` does not trigger provider failover but instead triggers context compression (Pattern 3) — the classifier distinguishes between two fundamentally different recovery strategies: "need to switch provider" vs "need to compress context." 18 FailoverReason values cover all failure scenarios from authentication to billing to server-side to context to model to request format to provider-specific issues.

### Layer 2: Credential Pool Strategy

**Source location: agent/credential_pool.py:49-60**

```python
STATUS_OK = "ok"
STATUS_EXHAUSTED = "exhausted"

STRATEGY_FILL_FIRST = "fill_first"
STRATEGY_ROUND_ROBIN = "round_robin"
```

Three credential rotation strategies:
- `fill_first`: use the first available credential until exhausted, then switch
- `round_robin`: distribute requests evenly across all credentials
- `random`: randomly select a credential (suitable for multi-provider balancing)

### Layer 3: ProviderProfile Declarative Description

**Source location: providers/base.py:24-165**

```python
@dataclass
class ProviderProfile:
    """Base provider profile — subclass or instantiate with overrides."""
    name: str
    api_mode: str = "chat_completions"
    aliases: tuple = ()

    # Auth & endpoints
    env_vars: tuple = ()
    base_url: str = ""
    models_url: str = ""
    auth_type: str = "api_key"   # api_key|oauth_device_code|oauth_external|copilot|aws_sdk

    # Model catalog
    fallback_models: tuple = ()
    hostname: str = ""

    # Client-level quirks
    default_headers: dict[str, str] = field(default_factory=dict)

    # Request-level quirks
    fixed_temperature: Any = None  # None = use caller default; OMIT_TEMPERATURE = don't send
    default_max_tokens: int | None = None
    default_aux_model: str = ""
```

Key design: `OMIT_TEMPERATURE = object()` is a sentinel value meaning "do not send the temperature parameter at all" — the Kimi provider manages temperature server-side, so the client must omit this parameter.

ProviderProfile also declares 5 hook methods, embodying the declarative design pattern — subclasses override these methods to declare provider-specific behavior without modifying caller code:

| Hook Method | Source Location | Declarative Behavior |
|-------------|----------------|---------------------|
| `get_hostname()` | providers/base.py:87-97 | Auto-derive hostname from base_url |
| `prepare_messages()` | providers/base.py:99-103 | Provider-specific message preprocessing |
| `build_extra_body()` | providers/base.py:105-109 | Additional extra_body fields |
| `build_api_kwargs_extras()` | providers/base.py:111-125 | Separate extra_body from top-level kwargs (reasoning config dispatch) |
| `fetch_models()` | providers/base.py:127-165 | Live model list fetching (with models_url fallback logic) |

### Application

| Recovery Path | Trigger | Action | Pattern Interaction |
|---------------|---------|--------|---------------------|
| Retry same provider | `server_error`, `timeout` | Backoff + rebuild client | ProviderProfile quirks |
| Rotate credential | `rate_limit`, `billing` | credential_pool next key | fill_first/round_robin |
| Fallback provider | `model_not_found`, `auth_permanent` | Switch to next provider | ProviderProfile + config |
| Compress context | `context_overflow` | Trigger ContextEngine compress() | Pattern 3 integration |
| Abort | `format_error`, `auth_permanent` (after refresh) | Surface error to user | None |

---

## Pattern 3: Context Compression Window

### Scenario

When a conversation approaches the model's token limit, how can you compress the context while preserving critical information so the agent can continue working?

### Practice

Two-layer design: ContextEngine ABC defines the abstract interface, ContextCompressor provides the built-in implementation.

### Layer 1: ContextEngine ABC

**Source location: agent/context_engine.py:32-97**

```python
class ContextEngine(ABC):
    """Base class all context engines must implement."""

    # -- Identity
    @property
    @abstractmethod
    def name(self) -> str:
        """Short identifier (e.g. 'compressor', 'lcm')."""

    # -- Token state (read by run_agent.py for display/logging)
    last_prompt_tokens: int = 0
    last_completion_tokens: int = 0
    threshold_tokens: int = 0
    context_length: int = 0
    compression_count: int = 0

    # -- Compaction parameters
    threshold_percent: float = 0.75
    protect_first_n: int = 3
    protect_last_n: int = 6

    # -- Core interface
    @abstractmethod
    def update_from_response(self, usage: Dict[str, Any]) -> None:
        """Update tracked token usage from an API response."""

    @abstractmethod
    def should_compress(self, prompt_tokens: int = None) -> bool:
        """Return True if compaction should fire this turn."""

    @abstractmethod
    def compress(
        self,
        messages: List[Dict[str, Any]],
        current_tokens: int = None,
        focus_topic: str = None,
    ) -> List[Dict[str, Any]]:
        """Compact the message list and return the new list."""
```

### Layer 2: ContextCompressor Implementation

**Source location: agent/context_compressor.py:38-60**

```python
SUMMARY_PREFIX = (
    "[CONTEXT COMPACTION — REFERENCE ONLY] Earlier turns were compacted "
    "into the summary below. This is a handoff from a previous context "
    "window — treat it as background reference, NOT as active instructions. "
    "Do NOT answer questions or fulfill requests mentioned in this summary; "
    "they were already addressed. "
    "Your current task is identified in the '## Active Task' section of the "
    "summary — resume exactly from there. "
    "IMPORTANT: Your persistent memory (MEMORY.md, USER.md) in the system "
    "prompt is ALWAYS authoritative and active — never ignore or deprioritize "
    "memory content due to this compaction note. "
    "Respond ONLY to the latest user message "
    "that appears AFTER this summary."
)

_MIN_SUMMARY_TOKENS = 2000
_SUMMARY_RATIO = 0.20
_SUMMARY_TOKENS_CEILING = 12_000
```

Key design: SUMMARY_PREFIX uses a multi-layer guard framework to prevent the compressed summary from being executed as active instructions — "REFERENCE ONLY" marking, "Do NOT answer questions" directive, and "already addressed" assertion provide triple constraint.

### Application

| Engine | name | Core Mechanism | Tools Provided |
|--------|------|---------------|---------------|
| ContextCompressor | "compressor" | Head/tail protection + LLM summary | None |
| LCM (pluggable) | "lcm" | DAG construction + semantic search | lcm_grep, lcm_describe, lcm_expand |
| Plugin engine | custom | Any custom strategy | Custom tools |

---

## Pattern 4: Platform Adapter ABC (Unified Abstraction)

### Scenario

How can you provide a unified interface for 15+ messaging platforms (Telegram, Discord, WhatsApp, IRC, etc.) while allowing new platforms to self-register via plugins?

### Practice

Two-layer design: BasePlatformAdapter ABC defines the unified interface, platform_registry implements self-registration discovery.

### Layer 1: BasePlatformAdapter ABC

**Source location: gateway/platforms/base.py:1206-1267**

```python
class BasePlatformAdapter(ABC):
    """Base class for platform adapters.

    Subclasses implement platform-specific logic for:
    - Connecting and authenticating
    - Receiving messages
    - Sending messages/responses
    - Handling media
    """

    def __init__(self, config: PlatformConfig, platform: Platform):
        self.config = config
        self.platform = platform
        self._message_handler: Optional[MessageHandler] = None
        self._running = False

        # Track active message handlers per session for interrupt support
        self._active_sessions: Dict[str, asyncio.Event] = {}
        self._pending_messages: Dict[str, MessageEvent] = {}
        self._session_tasks: Dict[str, asyncio.Task] = {}
        self._background_tasks: set[asyncio.Task] = set()

        # Fatal error handling — unrecoverable platform errors
        self._fatal_error_code: Optional[str] = None
        self._fatal_error_message: Optional[str] = None
        self._fatal_error_retryable = True
        self._fatal_error_handler: Optional[Callable[["BasePlatformAdapter"], Awaitable[None] | None]] = None

        # Post-delivery callback integration
        self._post_delivery_callbacks: Dict[str, Any] = {}
```

Key design points expanded:
1. **Fatal error handling** (`_fatal_error_code`/`_fatal_error_message`/`_fatal_error_handler`): Unrecoverable platform errors (e.g. invalid Telegram bot token) trigger a fatal state where `has_fatal_error()` returns True, stopping further message processing. `_fatal_error_retryable` marks whether recovery is possible via restart.

2. **Post-delivery callbacks** (`_post_delivery_callbacks`): Async callback hooks after successful message delivery, allowing plugins and external systems to perform follow-up operations after message sending completes (e.g. disk cleanup plugin's post-delivery logging).

3. **Session-level interrupt** (`_active_sessions` + `_session_tasks`): The `/stop` command can precisely cancel a specific session's processing task without affecting other sessions.

### Layer 2: PlatformEntry + PlatformRegistry

**Source location: gateway/platform_registry.py:38-113**

```python
@dataclass
class PlatformEntry:
    """Metadata and factory for a single platform adapter."""
    name: str                           # Config identifier (e.g. "irc")
    label: str                          # Human-readable (e.g. "IRC")
    adapter_factory: Callable[[Any], Any]  # Factory for adapter instances
    check_fn: Callable[[], bool]        # Dependency availability check
    validate_config: Optional[Callable] = None  # Config validation
    required_env: list = field(default_factory=list)
    install_hint: str = ""
    setup_fn: Optional[Callable] = None  # Interactive setup
    source: str = "plugin"              # "builtin" or "plugin"
    max_message_length: int = 0         # Smart-chunking limit
    pii_safe: bool = False              # PII redaction
    emoji: str = "🔌"
    platform_hint: str = ""             # System prompt injection

class PlatformRegistry:
    """Central registry of platform adapters."""
    def register(self, entry: PlatformEntry) -> None:
        """Register a platform adapter entry. Last writer wins."""
    def create_adapter(self, name: str, config: Any) -> Optional[Any]:
        """Create adapter: check_fn → validate_config → factory"""
```

Key design: `adapter_factory` uses a factory function rather than a bare class, allowing plugins to perform custom initialization (extra kwargs, try/except wrapping). `create_adapter()`'s three-layer validation (check_fn → validate_config → factory) ensures graceful degradation when dependencies are missing or configuration is incorrect.

### Application

| Platform | Implementation | Source | Register Method |
|----------|---------------|--------|----------------|
| Telegram | Built-in class | builtin | Legacy if/elif fallback |
| Discord | Built-in class | builtin | Legacy if/elif fallback |
| WhatsApp | Built-in class | builtin | Legacy if/elif fallback |
| IRC | Plugin adapter | plugin | `platform_registry.register()` |
| Teams | Plugin adapter | plugin | `platform_registry.register()` |

---

## Pattern 5: Config Layer Merge (4-source Priority)

### Scenario

How can you let users override only the config items they need while preserving all defaults and preventing sensitive information (API keys) from leaking into YAML files?

### Practice

**Source location: hermes_cli/config.py:3675-3712**

```python
def _deep_merge(base: dict, override: dict) -> dict:
    """Recursively merge override into base, preserving nested defaults."""
    result = base.copy()
    for key, value in override.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            result[key] = _deep_merge(result[key], value)
        else:
            result[key] = value
    return result

def _expand_env_vars(obj):
    """Recursively expand ${VAR} references."""
    if isinstance(obj, str):
        return re.sub(r"\${([^}]+)}",
                      lambda m: os.environ.get(m.group(1), m.group(0)), obj)
    if isinstance(obj, dict):
        return {k: _expand_env_vars(v) for k, v in obj.items()}
    ...
```

`_preserve_env_ref_templates()` restores original `${VAR}` templates when saving, preventing plaintext secrets from being written to YAML:

```python
# Source location: hermes_cli/config.py:3730-3749
def _preserve_env_ref_templates(current, raw, loaded_expanded=None):
    """Restore raw ${VAR} templates when a value is otherwise unchanged."""
    if isinstance(current, str) and isinstance(raw, str) and re.search(r"\${[^}]+}", raw):
        if current == raw:                    # Identical → keep template
            return raw
        if current == loaded_expanded:        # Matches expanded → keep template
            return raw
        if _expand_env_vars(raw) == current:  # Env var rotated → keep template
            return raw
        return current                        # Caller modified value → keep literal
```

### Application

| Priority | Source | Merge Strategy | Security |
|----------|--------|---------------|---------|
| 1 (highest) | .env | Override=True | Plain secrets, file-locked |
| 2 | config.yaml | Deep merge onto DEFAULT | `${VAR}` template support |
| 3 | cli-config.yaml | Seeds default | Reference only |
| 4 (lowest) | hermes_constants.py | Hardcoded | Version-controlled |

---

## Pattern 6: Skill Source ABC

### Scenario

How can skills be discovered and loaded from multiple sources (bundled, user-installed, project-local, pip) while maintaining a consistent registration interface?

### Practice

Similar to Pattern 1's self-registration discovery, but for skills rather than tools. `hermes_cli/plugins.py`'s `_scan_directory()` and `_scan_entry_points()` implement multi-path scanning for skill sources.

### Application

| Skill Source | Path | Discovery Method | Trust Level |
|-------------|------|-----------------|------------|
| Bundled | `skills/` in repo | Directory scan + manifest | High (reviewed) |
| User | `~/.hermes/skills/` | Directory scan + manifest | Medium |
| Project | `.hermes/skills/` | Directory scan + manifest | Medium |
| Pip | Python packages | Entry-point scan | Low |

---

## Pattern 7: Execution Backend ABC

### Scenario

How can you provide multiple sandbox backends (local, docker, modal, daytona, ssh, singularity) for terminal execution while maintaining a unified tool interface?

### Practice

The Terminal tool selects a backend via the `terminal_backend` config item, and each backend implements the `TerminalBackend` interface (execute_command, start_session, cleanup, etc.).

### Application

| Backend | Config Value | Isolation | Use Case |
|---------|-------------|-----------|----------|
| local | `"local"` | None | Developer workstation |
| docker | `"docker"` | Container | CI/CD, RL training |
| modal | `"modal"` | Cloud | Production RL (per-rollout isolation) |
| daytona | `"daytona"` | Cloud IDE | SWE-bench evaluation |
| ssh | `"ssh"` | Remote server | Remote execution |
| singularity | `"singularity"` | HPC container | Academic clusters |

---

## Comparison Tables

### Table 1: Pattern Complexity Comparison

| Pattern | Core Abstraction | Key Mechanism | Coupling Level | Transfer Difficulty |
|---------|-----------------|---------------|---------------|-------------------|
| P1: Self-Registration | ToolEntry + ToolRegistry | Module-level register() | Zero | Low |
| P2: Failover Cascade | ClassifiedError + FailoverReason + CredentialPool + ProviderProfile | Error classification → recovery hints → strategy → provider quirks | Medium (4-layer) | Medium |
| P3: Context Window | ContextEngine ABC | Head/tail protection + summary + handoff framing | Low (ABC) | Medium |
| P4: Platform Adapter | BasePlatformAdapter ABC + PlatformRegistry | Factory function + 3-layer validation + fatal error + post-delivery | Low (ABC + registry) | Low |
| P5: Config Merge | _deep_merge + _expand_env_vars + _preserve_env_ref_templates | 4-source priority + template preservation | Low (functions) | Low |
| P6: Skill Source | Directory scan + entry-point | Manifest-based discovery | Low | Low |
| P7: Execution Backend | Config-driven backend selection | Abstract backend interface | Low (config) | Low |

### Table 2: Registry Pattern Comparison (P1 vs P4)

| Aspect | ToolRegistry (P1) | PlatformRegistry (P4) |
|--------|-------------------|----------------------|
| Registration method | `registry.register()` at import | `platform_registry.register()` via plugin |
| Discovery method | AST scan + import all modules | Plugin discover_and_load() |
| Thread safety | RLock + snapshot + generation counter | Sequential writes at startup |
| Validation | check_fn (TTL-cached) | check_fn → validate_config → factory |
| Override policy | Name collision = error | Last writer wins |
| Factory pattern | Direct handler callable | adapter_factory lambda |

### Table 3: Failover Strategy Comparison

| Error Type | FailoverReason | ClassifiedError.recovery | Credential Pool Impact | Provider Switch |
|------------|---------------|----------------------|----------------------|----------------|
| 401/403 (transient) | `auth` | `should_rotate_credential=True` | Mark exhausted, rotate | No |
| 402 (billing) | `billing` | `should_rotate_credential=True` | Mark exhausted | No |
| 429 (rate limit) | `rate_limit` | `should_rotate_credential=True` | Mark exhausted | No |
| 503/529 (overloaded) | `overloaded` | `retryable=True` | None | No |
| 500/502 (server error) | `server_error` | `retryable=True` | None | No |
| Context too large | `context_overflow` | `should_compress=True` | None | No |
| Image too large | `image_too_large` | `retryable=True` (shrink + retry) | None | No |
| 404 (model not found) | `model_not_found` | `should_fallback=True` | None | Yes |
| Provider policy blocked | `provider_policy_blocked` | `should_fallback=True` | None | Yes |
| 400 (format error) | `format_error` | `retryable=False` (abort) | None | Consider |
| Auth failed after refresh | `auth_permanent` | `retryable=False` (abort) | None | Consider |
| Thinking signature invalid | `thinking_signature` | `retryable=True` | None | No |
| Long context tier gate | `long_context_tier` | `retryable=True` | None | No |
| OAuth beta forbidden | `oauth_long_context_beta_forbidden` | `retryable=True` | None | No |
| llama.cpp grammar pattern | `llama_cpp_grammar_pattern` | `retryable=True` (strip + retry) | None | No |

### Table 4: Context Compression Design Comparison

| Design Aspect | ContextCompressor | LCM (future) | Plugin Engine |
|---------------|-------------------|--------------|---------------|
| Protection strategy | protect_first_n + protect_last_n | DAG-based semantic relevance | Custom |
| Summary method | LLM call (auxiliary model) | Semantic compression | Custom |
| Handoff framing | SUMMARY_PREFIX (3-layer guard) | Implicit in DAG | Custom |
| Budget calculation | threshold_percent (0.75) | Dynamic based on DAG | Custom |
| Tool integration | None | lcm_grep, lcm_describe | Custom |
| Session lifecycle | on_session_start/end/reset | Persistent DAG store | Custom |
| Model switch support | update_model() recalc | DAG budget recalc | Custom |

---

## Summary Table

| Pattern | Source Files | Lines | Responsibility |
|---------|-------------|-------|----------------|
| P1: Self-Registration Registry | `tools/registry.py` | 537 | Zero-coupling tool self-declaration discovery |
| P2: Failover Cascade | `agent/error_classifier.py` + `agent/credential_pool.py` + `providers/base.py` | 1036+1584+165 | Four-layer failure recovery strategy (ClassifiedError + FailoverReason + credential_pool + ProviderProfile) |
| P3: Context Window | `agent/context_engine.py` + `agent/context_compressor.py` | 206+1481 | ABC abstraction + head/tail protection + summary |
| P4: Platform Adapter | `gateway/platforms/base.py` + `gateway/platform_registry.py` | 3390+212 | ABC unified interface + fatal error + post-delivery + self-registration registry |
| P5: Config Layer Merge | `hermes_cli/config.py` | 4939 | Four-source priority + env expansion + template preservation |
| P6: Skill Source ABC | `hermes_cli/plugins.py` | 1362 | Multi-path skill discovery |
| P7: Execution Backend ABC | Terminal tool backends | — | 7 sandbox backend selection |

---

[<< 16 — Plugin System](/en/chapters/16-plugin-system)