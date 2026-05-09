# 17 — Design Patterns: 可迁移的可复用架构模式


---

## Source Files

| File | Lines | Role |
|------|-------|------|
| `tools/registry.py` | 537 | Pattern 1: 自注册工具发现 |
| `agent/error_classifier.py` | 1036 | Pattern 2: 错误分类 + 故障转移 |
| `agent/credential_pool.py` | 1584 | Pattern 2: 多凭证池 + 轮转策略 |
| `providers/base.py` | 165 | Pattern 2: ProviderProfile 声明式描述 |
| `agent/context_compressor.py` | 1481 | Pattern 3: 上下文压缩窗口 |
| `agent/context_engine.py` | 206 | Pattern 3: ContextEngine ABC |
| `gateway/platforms/base.py` | 3390 | Pattern 4: 平台适配器 ABC |
| `gateway/platform_registry.py` | 212 | Pattern 4: 平台自注册 registry |
| `hermes_cli/config.py` | 4939 | Pattern 5: 四源配置分层合并 |
| `hermes_cli/plugins.py` | 1362 | Pattern 6: 技能源 ABC |
| `hermes_constants.py` | 345 | Pattern 5: 硬编码常量 |

**One-liner:** 4 个核心可迁移设计模式 + 3 个附加模式：自注册工具发现（零耦合 registry）、多模型故障级联（错误分类 + 凭证池 + ProviderProfile）、上下文压缩窗口（ContextEngine ABC + head/tail 保护）、平台适配器 ABC（统一抽象 + 自注册 registry），附加配置分层合并、技能源 ABC、执行后端 ABC。

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

4 个核心可迁移设计模式解决 Hermes-Agent 中的关键架构问题：Pattern 1 自注册工具发现实现零耦合的 tool → registry → consumer 导入链，消除维护 parallel data structures 的负担；Pattern 2 多模型故障级联通过 ClassifiedError 结构化分类 + FailoverReason 分类 + credential_pool 策略轮转 + ProviderProfile 声明式描述（含 5 个 hook 方法）四层协同实现智能故障恢复；Pattern 3 上下文压缩窗口通过 ContextEngine ABC 抽象压缩策略，保护头尾回合 + LLM 概要中间 + handoff framing；Pattern 4 平台适配器 ABC 统一消息平台接口 + platform_registry 自注册消除 if/elif 链，含致命错误处理和投递后回调集成点。3 个附加模式覆盖配置分层合并、技能源发现和执行后端抽象。

---

## Pattern 1: Self-Registration Tool Discovery (Zero-Coupling)

### Scenario

在一个拥有 40+ 工具的系统中，如何让每个工具自声明其 schema、handler 和可用性，同时不引入任何工具文件之间的耦合？

### Practice

**核心机制：** 每个工具文件在模块级别调用 `registry.register()`，consumer (`model_tools.py`) 只导入 `tools.registry` + 所有工具模块，工具文件之间零依赖。

**源码位置: tools/registry.py:1-15**

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

**源码位置: tools/registry.py:77-98**

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

**源码位置: tools/registry.py:143-159**

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

`discover_builtin_tools()` 使用 AST 解析检测哪些工具文件包含 `registry.register()` 调用，避免盲目导入。

**源码位置: tools/registry.py:29-74**

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

check_fn（如 Docker daemon 检测、SDK 安装检测）结果缓存 30 秒，避免频繁外部状态探测。

**源码位置: tools/registry.py:113-133**

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

任何需要动态发现和注册能力的系统都可以使用此模式：MCP tool 注册、plugin tool 注册、RL environment 发现（AST 扫描 BaseEnv 子类）。

| Aspect | Hermes Usage | Transferable Scenario |
|--------|-------------|---------------------|
| Registration | `tools/*.py → registry.register()` | Plugin system, MCP tools, RL envs |
| Discovery | AST scan for `registry.register()` | Any auto-registration system |
| Caching | check_fn TTL + generation counter | Mutable registry memoization |
| Thread Safety | RLock + snapshot pattern | Concurrent read/mutate registries |

---

## Pattern 2: Multi-Model Failover Cascade

### Scenario

当 API 调用失败时，如何智能判断失败原因并选择正确的恢复策略（重试 vs 凭证轮转 vs provider 级联 vs 上下文压缩）？

### Practice

四层协同设计：ClassifiedError 结构化分类 → FailoverReason 分类 → credential_pool 策略 → ProviderProfile 声明式描述。

### Layer 0: ClassifiedError — 结构化错误输出

**源码位置: agent/error_classifier.py:67-85**

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

ClassifiedError 是 Pattern 2 的核心输出——`classify()` 函数返回 ClassifiedError 实例，retry loop 直接读取 `retryable`/`should_compress`/`should_rotate_credential`/`should_fallback` 属性决定恢复动作，无需重新分类。

### Layer 1: FailoverReason Taxonomy

**源码位置: agent/error_classifier.py:24-61**

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

关键设计决策：`context_overflow` 不触发 provider 故障转移，而是触发上下文压缩（Pattern 3）——分类器区分了"需要换 provider"和"需要压缩上下文"两种完全不同的恢复策略。18 个 FailoverReason 值覆盖了从认证到计费到服务端到上下文到模型到请求格式到 provider 特定的全部故障场景。

### Layer 2: Credential Pool Strategy

**源码位置: agent/credential_pool.py:49-60**

```python
STATUS_OK = "ok"
STATUS_EXHAUSTED = "exhausted"

STRATEGY_FILL_FIRST = "fill_first"
STRATEGY_ROUND_ROBIN = "round_robin"
```

三种凭证轮转策略：
- `fill_first`：使用第一个可用凭证直到耗尽，然后切换
- `round_robin`：均匀分配请求到所有凭证
- `random`：随机选择凭证（适合多 provider 均衡）

### Layer 3: ProviderProfile Declarative Description

**源码位置: providers/base.py:24-165**

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

关键设计：`OMIT_TEMPERATURE = object()` 是 sentinel value，表示"完全不发送 temperature 参数"——Kimi provider 服务端管理温度，客户端必须省略该参数。

ProviderProfile 还声明了 5 个 hook 方法，体现声明式（"声明式"）设计模式——子类通过覆盖这些方法声明 provider 特定行为，无需修改调用方代码：

| Hook 方法 | 源码位置 | 声明式行为 |
|-----------|---------|-----------|
| `get_hostname()` | providers/base.py:87-97 | 自动从 base_url 推导 hostname |
| `prepare_messages()` | providers/base.py:99-103 | provider 特定消息预处理 |
| `build_extra_body()` | providers/base.py:105-109 | 附加 extra_body 字段 |
| `build_api_kwargs_extras()` | providers/base.py:111-125 | 分离 extra_body 与顶层 kwargs（reasoning 配置分发） |
| `fetch_models()` | providers/base.py:127-165 | 活模型列表拉取（含 models_url 回退逻辑） |

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

当对话接近模型 token 上限时，如何在保留关键信息的同时压缩上下文，使 agent 能够继续工作？

### Practice

两层设计：ContextEngine ABC 定义抽象接口，ContextCompressor 提供内置实现。

### Layer 1: ContextEngine ABC

**源码位置: agent/context_engine.py:32-97**

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

**源码位置: agent/context_compressor.py:38-60**

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

关键设计：SUMMARY_PREFIX 使用多重防护框架避免压缩后的概要被当作活跃指令执行——"REFERENCE ONLY"标记、"Do NOT answer questions"指令、"already addressed"声明三重约束。

### Application

| Engine | name | Core Mechanism | Tools Provided |
|--------|------|---------------|---------------|
| ContextCompressor | "compressor" | Head/tail protection + LLM summary | None |
| LCM (pluggable) | "lcm" | DAG construction + semantic search | lcm_grep, lcm_describe, lcm_expand |
| Plugin engine | custom | Any custom strategy | Custom tools |

---

## Pattern 4: Platform Adapter ABC (Unified Abstraction)

### Scenario

如何为 15+ 消息平台（Telegram、Discord、WhatsApp、IRC 等）提供统一接口，同时允许新平台通过插件自注册？

### Practice

两层设计：BasePlatformAdapter ABC 定义统一接口，platform_registry 实现自注册发现。

### Layer 1: BasePlatformAdapter ABC

**源码位置: gateway/platforms/base.py:1206-1267**

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

关键设计点扩展：
1. **致命错误处理** (`_fatal_error_code`/`_fatal_error_message`/`_fatal_error_handler`): 不可恢复的平台错误（如 Telegram bot token 无效）触发致命状态，`has_fatal_error()` 返回 True，停止进一步消息处理。`_fatal_error_retryable` 标记是否可通过重启恢复。

2. **投递后回调** (`_post_delivery_callbacks`): 消息成功投递后的异步回调钩子，允许插件和外部系统在消息发送完成后执行后续操作（如磁盘清理插件的 post-delivery 日志记录）。

3. **Session 级中断** (`_active_sessions` + `_session_tasks`): `/stop` 命令可以精确取消特定 session 的处理 task，而不会影响其他 session。

### Layer 2: PlatformEntry + PlatformRegistry

**源码位置: gateway/platform_registry.py:38-113**

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

关键设计：`adapter_factory` 使用工厂函数而非裸类，允许插件做自定义初始化（额外 kwargs、try/except 包装）。`create_adapter()` 的三层验证（check_fn → validate_config → factory）确保依赖缺失或配置错误时优雅降级。

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

如何让用户只覆盖需要的配置项，同时保留所有默认值和敏感信息（API key）不泄露到 YAML 文件？

### Practice

**源码位置: hermes_cli/config.py:3675-3712**

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

`_preserve_env_ref_templates()` 在保存时恢复原始 `${VAR}` 模板，避免明文密钥写入 YAML：

```python
# 源码位置: hermes_cli/config.py:3730-3749
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

如何让技能（skills）从多个来源（bundled、user-installed、project-local、pip）被发现和加载，同时保持一致的注册接口？

### Practice

类似 Pattern 1 的自注册发现，但用于 skills 而非 tools。`hermes_cli/plugins.py` 的 `_scan_directory()` 和 `_scan_entry_points()` 实现了技能源的多路径扫描。

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

如何为终端执行提供多种沙箱后端（local、docker、modal、daytona、ssh、singularity），同时保持统一的工具接口？

### Practice

Terminal 工具通过 `terminal_backend` 配置项选择后端，每个后端实现 `TerminalBackend` 接口（execute_command、start_session、cleanup 等）。

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
| P1: Self-Registration Registry | `tools/registry.py` | 537 | 零耦合工具自声明发现 |
| P2: Failover Cascade | `agent/error_classifier.py` + `agent/credential_pool.py` + `providers/base.py` | 1036+1584+165 | 四层故障恢复策略（ClassifiedError + FailoverReason + credential_pool + ProviderProfile） |
| P3: Context Window | `agent/context_engine.py` + `agent/context_compressor.py` | 206+1481 | ABC 抽象 + head/tail 保护 + 概要 |
| P4: Platform Adapter | `gateway/platforms/base.py` + `gateway/platform_registry.py` | 3390+212 | ABC 统一接口 + fatal error + post-delivery + 自注册 registry |
| P5: Config Layer Merge | `hermes_cli/config.py` | 4939 | 四源优先级 + env 展开 + 模板保留 |
| P6: Skill Source ABC | `hermes_cli/plugins.py` | 1362 | 多路径技能发现 |
| P7: Execution Backend ABC | Terminal tool backends | — | 7 种沙箱后端选择 |

---

[<< 16 — Plugin System](/zh-CN/chapters/16-plugin-system)