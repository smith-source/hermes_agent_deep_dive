# 04 — LLM Provider System: 28+ Providers Unified via Declarative Profiles, Credential Pool Rotation, and Intelligent Failure Classification

[05 — Tool System](/en/chapters/05-tool-system) | [06 — Gateway Messaging Platform](/en/chapters/06-gateway-system)

---

## Source Files

| File | Lines | Core Responsibility |
|------|------|----------|
| `providers/base.py` | 165 | ProviderProfile declarative base class |
| `agent/anthropic_adapter.py` | 1970 | Anthropic Messages API adaptation |
| `agent/gemini_native_adapter.py` | 965 | Gemini native REST API adaptation |
| `agent/bedrock_adapter.py` | 1264 | AWS Bedrock Converse API adaptation |
| `agent/codex_responses_adapter.py` | 999 | OpenAI Responses API adaptation |
| `agent/auxiliary_client.py` | 4048 | Auxiliary task LLM routing and fallback |
| `agent/credential_pool.py` | 1584 | Multi-credential pool management and rotation strategies |
| `agent/rate_limit_tracker.py` | 246 | Rate limit state parsing and visualization |
| `agent/error_classifier.py` | 1036 | API error classification and failover decisions |
| `agent/model_metadata.py` | 1483 | Model metadata/context length probing |
| `plugins/model-providers/` (28 subdirectories) | ~2800+ | Provider plugin declarations |

## One-Sentence Summary

Hermes unifies 28+ heterogeneous LLM providers through ProviderProfile declarative descriptions, 6 native API adapters, CredentialPool credential rotation, and ErrorClassifier failure classification into a consistent calling interface, enabling automatic failover and load balancing.

---

## Architecture Overview

```
                    ┌──────────────────────────────────────────────┐
                    │           AIAgent (run_agent.py)             │
                    │  ┌─────────────┐  ┌───────────────────────┐  │
                    │  │ OpenAI SDK  │  │ Native Adapters       │  │
                    │  │ (chat_comp) │  │ anthropic / gemini /  │  │
                    │  │             │  │ bedrock / codex / ... │  │
                    │  └─────┬───────┘  └────────┬──────────────┘  │
                    │        │                   │                  │
                    │        └───────────┬───────┘                  │
                    │                    ▼                          │
                    │         ┌─────────────────────┐              │
                    │         │  ProviderProfile     │              │
                    │         │  (providers/base.py) │              │
                    │         │  name, api_mode,     │              │
                    │         │  auth_type, quirks   │              │
                    │         └──────────┬──────────┘              │
                    │                    │                          │
                    │                    ▼                          │
                    │  ┌───────────────────────────────────────┐   │
                    │  │          CredentialPool               │   │
                    │  │  fill_first / round_robin / random    │   │
                    │  │  least_used + exhausted cooldown       │   │
                    │  └──────────────┬────────────────────────┘   │
                    │                 │                             │
                    │         ┌───────┴───────┐                    │
                    │         ▼               ▼                    │
                    │  ┌─────────────┐  ┌───────────────────┐     │
                    │  │RateLimit    │  │ ErrorClassifier    │     │
                    │  │Tracker      │  │ (failover reason)  │     │
                    │  └─────────────┘  └───────────────────┘     │
                    └──────────────────────────────────────────────┘
                                       │
                                       ▼
                    ┌───────────────────────────────────────┐
                    │    plugins/model-providers/ (28+)     │
                    │  ┌──────┐ ┌──────┐ ┌──────┐ ┌─────┐ │
                    │  │openR │ │anthr │ │gemin │ │deep │ │
                    │  │outer │ │opic  │ │i     │ │seek │ │
                    │  └──────┘ └──────┘ └──────┘ └─────┘ │
                    │  ┌──────┐ ┌──────┐ ┌──────┐ ┌─────┐ │
                    │  │codex │ │copil │ │bedr  │ │xai  │ │
                    │  │      │ │ot    │ │ock   │ │     │ │
                    │  └──────┘ └──────┘ └──────┘ └─────┘ │
                    └───────────────────────────────────────┘
```

---

## TL;DR

Hermes's LLM provider system unifies 28+ heterogeneous providers (from OpenRouter aggregator to Anthropic native API, AWS Bedrock Converse, Gemini native REST) into a single calling interface. ProviderProfile uses a pure declarative dataclass to describe each provider's identity, authentication, endpoints, and request quirks, eliminating the need for 20+ boolean flags in the transport layer. 6 native adapters isolate protocol differences for Anthropic Messages, Gemini generateContent, Bedrock Converse, OpenAI Responses, and more. CredentialPool supports fill_first/round_robin/random/least_used credential rotation strategies, automatically cooling exhausted credentials. ErrorClassifier categorizes API errors into 18 FailoverReason types, driving retry/rotate/compress/fallback/abort recovery paths.

---

## 1. ProviderProfile — Declarative Provider Description

`ProviderProfile` is the cornerstone of the entire provider system. It encapsulates all behavioral characteristics of a provider in a pure declarative dataclass, replacing the 20+ boolean flags scattered across the old architecture.

Source location: `providers/base.py:25-166`

```python
@dataclass
class ProviderProfile:
    """Base provider profile — subclass or instantiate with overrides."""

    # ── Identity ─────────────────────────────────────────────
    name: str
    api_mode: str = "chat_completions"       # chat_completions | responses
    aliases: tuple = ()

    # ── Auth & endpoints ─────────────────────────────────────
    env_vars: tuple = ()                     # e.g. ("OPENROUTER_API_KEY",)
    base_url: str = ""                       # e.g. "https://openrouter.ai/api/v1"
    models_url: str = ""                     # explicit models endpoint override
    auth_type: str = "api_key"               # api_key|oauth_device_code|oauth_external|copilot|aws_sdk

    # ── Request-level quirks ─────────────────────────────────
    fixed_temperature: Any = None            # None=caller default, OMIT_TEMPERATURE=omit
    default_max_tokens: int | None = None
    default_aux_model: str = ""              # cheap model for auxiliary tasks
```

Core design principle: Profile only describes behavior, it does not own client construction, credential rotation, or streaming processing. Those responsibilities remain with AIAgent. Profile provides overridable hook methods:

```python
# Source location: providers/base.py:80-165

def prepare_messages(self, messages: list[dict[str, Any]]) -> list[dict[str, Any]]:
    """Provider-specific message preprocessing. Default: pass-through."""
    return messages

def build_extra_body(self, *, session_id: str | None = None, **context: Any) -> dict[str, Any]:
    """Provider-specific extra_body fields merged into API kwargs."""
    return {}

def build_api_kwargs_extras(
    self, *, reasoning_config: dict | None = None, **context: Any,
) -> tuple[dict[str, Any], dict[str, Any]]:
    """Split reasoning config between extra_body and top-level api_kwargs.
    Returns (extra_body_additions, top_level_kwargs)."""
    return {}, {}

def fetch_models(self, *, api_key: str | None = None, timeout: float = 8.0) -> list[str] | None:
    """Fetch live model list from provider's models endpoint.
    Resolution: self.models_url → self.base_url + "/models" → None"""
```

The `OMIT_TEMPERATURE` sentinel value handles providers like Kimi/Moonshot that manage temperature server-side — sending any value (even a "correct" one) conflicts with the gateway-side mode selection.

### Comparison Table: ProviderProfile Field Categories

| Category | Fields | Type | Usage Example |
|------|------|------|----------|
| Identity | `name`, `api_mode`, `aliases` | str/tuple | "openrouter", "chat_completions", ("or",) |
| Auth | `env_vars`, `base_url`, `auth_type` | tuple/str | ("OPENROUTER_API_KEY",), "api_key" |
| Catalog | `fallback_models`, `hostname` | tuple/str | ("gpt-4o", "claude-3.5-sonnet") |
| Request quirks | `fixed_temperature`, `default_max_tokens` | Any/int/None | OMIT_TEMPERATURE, 4096 |
| Hooks | `prepare_messages()`, `build_extra_body()` | method | Anthropic message rewrite, Kimi reasoning |

### Comparison Table: auth_type Authentication Modes

| auth_type | Provider | Authentication Mechanism | Code Location |
|-----------|--------|----------|----------|
| `api_key` | OpenRouter, DeepSeek, xAI | Bearer header from env var | `credential_pool.py:55` |
| `oauth_device_code` | Nous Portal | OAuth device code flow → JWT | `hermes_cli/auth.py` |
| `oauth_external` | Copilot, MiniMax-OAuth | External OAuth → token exchange | `hermes_cli/auth.py` |
| `copilot` | GitHub Copilot | Copilot API token via OAuth | `agent/copilot_adapter.py` |
| `aws_sdk` | AWS Bedrock | IAM role / SSO / instance metadata | `bedrock_adapter.py:48-58` |

---

## 2. Native Adapters Deep Dive

### 2.1 Anthropic Adapter

Source location: `agent/anthropic_adapter.py:1-1970`

The Anthropic adapter is the largest native adapter (1970 lines), handling all protocol differences for the Anthropic Messages API. Key features:

**Thinking mode mapping** (Source location: `anthropic_adapter.py:47-79`):

```python
THINKING_BUDGET = {"xhigh": 32000, "high": 16000, "medium": 8000, "low": 4000}

# Claude 4.7 adds xhigh level (between high and max)
ADAPTIVE_EFFORT_MAP = {
    "max": "max", "xhigh": "xhigh", "high": "high",
    "medium": "medium", "low": "low", "minimal": "low",
}

# 4.7+ models accept xhigh; 4.6 models only have low/medium/high/max
_XHIGH_EFFORT_SUBSTRINGS = ("4-7", "4.7")
_ADAPTIVE_THINKING_SUBSTRINGS = ("4-6", "4.6", "4-7", "4.7")
_NO_SAMPLING_PARAMS_SUBSTRINGS = ("4-7", "4.7")  # 4.7 forbids temperature/top_p/top_k
```

**max_tokens tier table** (Source location: `anthropic_adapter.py:86-117`):

```python
_ANTHROPIC_OUTPUT_LIMITS = {
    "claude-opus-4-7":   128_000,   # Claude 4.7 Opus
    "claude-opus-4-6":   128_000,   # Claude 4.6 Opus
    "claude-sonnet-4-6":  64_000,   # Claude 4.6 Sonnet
    "claude-3-7-sonnet": 128_000,   # Claude 3.7 Sonnet
    "claude-3-5-sonnet":   8_192,   # Claude 3.5 Sonnet
    "claude-3-opus":       4_096,   # Claude 3 Opus
}
_ANTHROPIC_DEFAULT_OUTPUT_LIMIT = 128_000
```

**Authentication supports three paths**: (1) Standard API key (`sk-ant-api*`) → `x-api-key` header; (2) OAuth setup-token (`sk-ant-oat*`) → Bearer auth + beta header; (3) Claude Code credentials (`~/.claude.json`) → Bearer auth.

**Lazy SDK import** (Source location: `anthropic_adapter.py:31-43`): The anthropic SDK import takes ~220ms, using a sentinel + cache pattern to defer until the first call.

### 2.2 Gemini Native Adapter

Source location: `agent/gemini_native_adapter.py:1-965`

The Gemini adapter bypasses Google's OpenAI-compatible endpoint (which is unstable under multi-turn tool calling scenarios) and directly interfaces with the `models/{model}:generateContent` native REST API.

**Free-tier probing** (Source location: `gemini_native_adapter.py:47-118`):

```python
def probe_gemini_tier(api_key, base_url=DEFAULT_GEMINI_BASE_URL, *,
                      model="gemini-2.5-flash", timeout=10.0) -> str:
    """Probe API key and return tier: 'free', 'paid', or 'unknown'.
    Free tier: <= 250 req/day for flash, unusable with Hermes agent loop."""
```

It distinguishes free tier (<= 1000) from paid tier (> 1000) via the `x-ratelimit-limit-requests-per-day` header. Free-tier keys are rejected outright — Hermes makes 3-10 API calls per turn, and the free tier can only sustain a few messages.

**Multimodal conversion** (Source location: `gemini_native_adapter.py:159-200`): Converts OpenAI format `image_url` (data: URI) to Gemini `inlineData` (base64 + mimeType).

### 2.3 Bedrock Adapter

Source location: `agent/bedrock_adapter.py:1-1264`

The Bedrock adapter uses the AWS Converse API unified interface, supporting all Bedrock models including Claude, Nova, Llama, and Mistral, completely bypassing the OpenAI-compatible endpoint.

**Lazy boto3 import and caching** (Source location: `bedrock_adapter.py:44-87`):

```python
_bedrock_runtime_client_cache: Dict[str, Any] = {}

def _require_boto3():
    """Import boto3, raising clear error if not installed."""
    try:
        import boto3
        return boto3
    except ImportError:
        raise ImportError("pip install boto3 or pip install -e '.[bedrock]'")

def _get_bedrock_runtime_client(region: str):
    """Get or create cached bedrock-runtime client (uses default AWS credential chain)."""
    if region not in _bedrock_runtime_client_cache:
        boto3 = _require_boto3()
        _bedrock_runtime_client_cache[region] = boto3.client("bedrock-runtime", region_name=region)
    return _bedrock_runtime_client_cache[region]
```

**Stale connection detection** (Source location: `bedrock_adapter.py:125-196`): Detects dead connections in the boto3 cached HTTPS connection pool (`ConnectionClosedError`, `ProtocolError`, library-internal `AssertionError`) and evicts the region-cached client to rebuild the connection pool.

```python
def is_stale_connection_error(exc: BaseException) -> bool:
    """Match botocore ConnectionError, urllib3 ProtocolError,
    and library-internal AssertionError from urllib3/botocore/boto3 frames."""
```

### 2.4 Codex Responses Adapter

Source location: `agent/codex_responses_adapter.py:1-999`

Handles the OpenAI Responses API format (used by Codex, xAI, GitHub Models). Core features:

**Deterministic call_id** (Source location: `codex_responses_adapter.py:143-152`): Avoids UUID invalidation of OpenAI prompt cache.

```python
def _deterministic_call_id(fn_name: str, arguments: str, index: int = 0) -> str:
    """Deterministic ID from tool call content. Prevents cache invalidation."""
    seed = f"{fn_name}:{arguments}:{index}"
    digest = hashlib.sha256(seed.encode()).hexdigest()[:12]
    return f"call_{digest}"
```

**Tool call leak filtering** (Source location: `codex_responses_adapter.py:37-40`): Matches Codex/Harmony serialization leak pattern `to=functions.<name>`.

### Comparison Table: Native Adapter Feature Comparison

| Adapter | Protocol | Authentication | Thinking Support | Lines |
|--------|------|------|----------|------|
| Anthropic | Messages API | API key + OAuth + Claude Code | Adaptive thinking (4 levels + xhigh on 4.7) | 1970 |
| Gemini | generateContent REST | API key query param | Free-tier probe, paid-only enforcement | 965 |
| Bedrock | Converse API | AWS IAM chain (role/SSO/instance) | Stale-connection detection + client eviction | 1264 |
| Codex Responses | Responses API | OAuth device code | Deterministic call_id for cache | 999 |

---

## 3. CredentialPool — Credential Pool Management and Rotation

Source location: `agent/credential_pool.py:1-1584`

CredentialPool is the core of Hermes's multi-credential failover. It maintains a set of `PooledCredential` for each provider, supporting four rotation strategies and automatic cooldown.

**Credential data structure** (Source location: `credential_pool.py:82-171`):

```python
@dataclass
class PooledCredential:
    provider: str
    id: str                         # auto-generated uuid[:6]
    label: str                      # human-readable (email/username)
    auth_type: str                  # api_key | oauth
    priority: int                   # higher = preferred
    source: str                     # manual | oauth | ...
    access_token: str
    refresh_token: Optional[str] = None
    last_status: Optional[str] = None           # ok | exhausted
    last_status_at: Optional[float] = None
    last_error_code: Optional[int] = None       # HTTP status
    last_error_reason: Optional[str] = None
    last_error_message: Optional[str] = None    # provider error text
    last_error_reset_at: Optional[float] = None # provider reset timestamp
    base_url: Optional[str] = None              # per-credential endpoint override
    expires_at: Optional[str] = None            # token expiry (ISO string)
    expires_at_ms: Optional[int] = None         # token expiry (epoch ms)
    last_refresh: Optional[str] = None          # last refresh timestamp
    inference_base_url: Optional[str] = None    # Nous inference endpoint
    agent_key: Optional[str] = None             # Nous agent key (distinct from access_token)
    agent_key_expires_at: Optional[str] = None
    request_count: int = 0
    extra: Dict[str, Any] = None                # type: ignore[assignment]

    def __post_init__(self):
        if self.extra is None:
            self.extra = {}

    # ── extra proxy design pattern ──────────────────────────────
    # Keys in _EXTRA_KEYS (token_type, scope, client_id, portal_base_url,
    # obtained_at, expires_in, agent_key_id, agent_key_expires_in,
    # agent_key_reused, agent_key_obtained_at, tls) are stored in
    # self.extra but accessed as attributes via __getattr__.
    # This keeps the dataclass signature clean while preserving
    # round-trip JSON fields that are not used in logic.

    def __getattr__(self, name: str):
        if name in _EXTRA_KEYS:
            return self.extra.get(name)
        raise AttributeError(f"'{type(self).__name__}' has no attribute {name!r}")

    @property
    def runtime_api_key(self) -> str:
        if self.provider == "nous":
            return str(self.agent_key or self.access_token or "")
        return str(self.access_token or "")

    @property
    def runtime_base_url(self) -> Optional[str]:
        if self.provider == "nous":
            return self.inference_base_url or self.base_url
        return self.base_url
```

**Four rotation strategies** (Source location: `credential_pool.py:59-68`):

```python
STRATEGY_FILL_FIRST = "fill_first"      # Use one until exhausted, then switch to next
STRATEGY_ROUND_ROBIN = "round_robin"    # Even rotation
STRATEGY_RANDOM = "random"              # Random selection
STRATEGY_LEAST_USED = "least_used"      # Least-used priority
```

**Exhausted cooldown mechanism** (Source location: `credential_pool.py:71-74,191-195`):

```python
EXHAUSTED_TTL_429_SECONDS = 60 * 60      # 429 rate-limit: 1h cooldown
EXHAUSTED_TTL_DEFAULT_SECONDS = 60 * 60  # default: 1h cooldown

def _exhausted_ttl(error_code: Optional[int]) -> int:
    if error_code == 429:
        return EXHAUSTED_TTL_429_SECONDS
    return EXHAUSTED_TTL_DEFAULT_SECONDS
```

Provider-supplied `reset_at` timestamps override the default cooldown period.

**Timestamp parsing** (Source location: `credential_pool.py:198-238`): Supports epoch seconds, epoch milliseconds, ISO-8601 strings, and embedded `retryDelay` text parsing.

### Comparison Table: Rotation Strategy Comparison

| Strategy | Selection Logic | Applicable Scenario | Simplicity |
|------|----------|----------|--------|
| `fill_first` | Priority-ordered, switch when exhausted | Single primary credential + backup | Highest |
| `round_robin` | Even轮流 rotation | Multi-credential load balancing | Medium |
| `random` | Random sampling | Stateless distribution | High |
| `least_used` | Lowest `request_count` | Even wear distribution | Low |

---

## 4. ErrorClassifier — Intelligent Failure Classification and Failover

Source location: `agent/error_classifier.py:1-1036`

ErrorClassifier provides 18 FailoverReason classifications, driving retry/rotate/compress/fallback/abort recovery paths.

**Failure classification system** (Source location: `error_classifier.py:24-61`, defined not imported):

```python
class FailoverReason(enum.Enum):
    """Why an API call failed — determines recovery strategy."""
    # Authentication / authorization
    auth = "auth"                        # Transient 401/403 — refresh/rotate
    auth_permanent = "auth_permanent"    # Auth failed after refresh — abort

    # Billing / quota
    billing = "billing"                  # 402 or confirmed credit exhaustion — rotate immediately
    rate_limit = "rate_limit"            # 429 or quota-based throttling — backoff then rotate

    # Server-side
    overloaded = "overloaded"            # 503/529 — provider overloaded, backoff
    server_error = "server_error"        # 500/502 — internal server error, retry

    # Transport
    timeout = "timeout"                  # Connection/read timeout — rebuild client + retry

    # Context / payload
    context_overflow = "context_overflow"  # Context too large — compress, not failover
    payload_too_large = "payload_too_large"  # 413 — compress payload
    image_too_large = "image_too_large"   # Native image exceeds per-image limit — shrink and retry

    # Model
    model_not_found = "model_not_found"  # 404 or invalid model — fallback to different model
    provider_policy_blocked = "provider_policy_blocked"  # OpenRouter privacy gate — abort (user action)

    # Request format
    format_error = "format_error"        # 400 bad request — abort or strip + retry

    # Provider-specific
    thinking_signature = "thinking_signature"  # Anthropic thinking block sig invalid
    long_context_tier = "long_context_tier"    # Anthropic "extra usage" tier gate
    oauth_long_context_beta_forbidden = "oauth_long_context_beta_forbidden"  # Anthropic OAuth rejects 1M context beta — disable beta and retry
    llama_cpp_grammar_pattern = "llama_cpp_grammar_pattern"  # llama.cpp json-schema-to-grammar rejects regex pattern/format — strip from tools and retry

    # Catch-all
    unknown = "unknown"                  # Unclassifiable — retry with backoff
```

**Classification results with recovery hints** (Source location: `error_classifier.py:67-87`):

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

**Multi-level pattern matching** (Source location: `error_classifier.py:93-300`): The Classifier maintains 10+ groups of regex patterns, from billing exhaustion ("insufficient credits", "top up your credits") to rate limiting ("rate limit", "resource_exhausted", "throttlingexception"), to context overflow ("context length", "max_model_len", "exceeded maximum length"), and OpenRouter policy blocking ("no endpoints available matching your guardrail").

### Comparison Table: FailoverReason to Recovery Path Mapping (18 types, grouped by category)

| Category | FailoverReason | HTTP Status Code | Recovery Path | retryable |
|------|----------------|-------------|----------|-----------|
| **Authentication** | `auth` | 401/403 | rotate credential | False (rotate+fallback) |
| | `auth_permanent` | 401 | abort | False |
| **Billing/quota** | `billing` | 402 | rotate immediately | False (rotate+fallback) |
| | `rate_limit` | 429 | backoff + rotate | True |
| **Server-side** | `overloaded` | 503/529 | backoff | True |
| | `server_error` | 500/502 | retry | True |
| **Transport** | `timeout` | — | rebuild client + retry | True |
| **Context/payload** | `context_overflow` | 400 | compress context | True (compress) |
| | `payload_too_large` | 413 | compress payload | True (compress) |
| | `image_too_large` | 400 | shrink image + retry | True |
| **Model** | `model_not_found` | 404 | fallback model | False (fallback) |
| | `provider_policy_blocked` | 404 | abort (user action) | False |
| **Request format** | `format_error` | 400 | abort/strip + retry | depends |
| **Provider-specific** | `thinking_signature` | 400 | strip sig + retry | True |
| | `long_context_tier` | 429 | compress + retry | True |
| | `oauth_long_context_beta_forbidden` | 400 | strip beta header + retry | True |
| | `llama_cpp_grammar_pattern` | 400 | strip pattern/format from tools + retry | True |
| **Catch-all** | `unknown` | — | retry with backoff | True |

---

## 5. AuxiliaryClient — Auxiliary Task LLM Routing

Source location: `agent/auxiliary_client.py:1-4048`

AuxiliaryClient provides unified LLM routing and fallback chains for side tasks (context compression, visual analysis, web extraction, session search).

**Text task auto-detection chain** (Source location: `auxiliary_client.py:7-16`):

```
1. User's main provider + main model (any provider type)
2. OpenRouter (OPENROUTER_API_KEY)
3. Nous Portal (~/.hermes/auth.json)
4. Custom endpoint (config.yaml + OPENAI_API_KEY)
5. Native Anthropic
6. Direct API-key providers (z.ai, Kimi, MiniMax)
7. None (no available backend)
```

**Provider alias system** (Source location: `auxiliary_client.py:131-162`): 30+ alias mappings, such as `"google"→"gemini"`, `"grok"→"xai"`, `"kimi"→"kimi-coding"`, `"claude"→"anthropic"`.

**OpenAI SDK lazy proxy** (Source location: `auxiliary_client.py:69-100`): The OpenAI SDK import takes ~240ms, using the `_OpenAIProxy` class for module-level lazy proxying, supporting `isinstance` checks and patch mocking.

```python
class _OpenAIProxy:
    """Module-level proxy that looks like openai.OpenAI class.
    Forwards calls and isinstance checks to the real SDK lazily."""
    __slots__ = ()
    def __call__(self, *args, **kwargs):
        return _load_openai_cls()(*args, **kwargs)
    def __instancecheck__(self, obj):
        return isinstance(obj, _load_openai_cls())

OpenAI = _OpenAIProxy()  # module-level name
```

---

## 6. RateLimitTracker — Rate Limit Visualization

Source location: `agent/rate_limit_tracker.py:1-246`

RateLimitTracker parses `x-ratelimit-*` response headers (12 headers), providing terminal visualization and compact status bar output formats.

**Data model** (Source location: `rate_limit_tracker.py:30-68`):

```python
@dataclass
class RateLimitBucket:
    """One rate-limit window (e.g. requests per minute)."""
    limit: int = 0
    remaining: int = 0
    reset_seconds: float = 0.0
    captured_at: float = 0.0

    @property
    def used(self) -> int:
        return max(0, self.limit - self.remaining)

    @property
    def usage_pct(self) -> float:
        return (self.used / self.limit) * 100.0 if self.limit > 0 else 0.0

@dataclass
class RateLimitState:
    """Full rate-limit state parsed from response headers."""
    requests_min: RateLimitBucket = field(default_factory=RateLimitBucket)
    requests_hour: RateLimitBucket = field(default_factory=RateLimitBucket)
    tokens_min: RateLimitBucket = field(default_factory=RateLimitBucket)
    tokens_hour: RateLimitBucket = field(default_factory=RateLimitBucket)
```

**ASCII progress bar output** (Source location: `rate_limit_tracker.py:159-246`):

```
Nous Portal Rate Limits (captured just now):

  Requests/min   [████████░░░░░░░░░░░░] 40.0%  320/800 used  (480 left, resets in 42s)
  Requests/hr    [██░░░░░░░░░░░░░░░░░░] 10.5%  800/7600 used  (6.8K left, resets in 58m 14s)

  Tokens/min     [████████████░░░░░░░░] 60.0%  300K/500K used  (200K left, resets in 42s)
  Tokens/hr      [██░░░░░░░░░░░░░░░░░░]  8.0%  2.0M/25M used  (23.0M left, resets in 58m 14s)
```

> Auto-warning at > 80% usage.

---

## 7. Model Metadata — Context Length Probing

Source location: `agent/model_metadata.py:1-1483`

The model metadata system maintains model context length tier tables, provider prefix stripping, and multi-layer probing fallback.

**Tiered probing sequence** (Source location: `model_metadata.py:116-131`):

```python
CONTEXT_PROBE_TIERS = [256_000, 128_000, 64_000, 32_000, 16_000, 8_000]
DEFAULT_FALLBACK_CONTEXT = 256_000
MINIMUM_CONTEXT_LENGTH = 64_000  # Refuse to run below this value
```

**Context length fallback table** (Source location: `model_metadata.py:137-200`): Claude 4.6+ = 1M, GPT-5.4 = 1.05M, DeepSeek V4 = 1M, Gemini = 1M, Qwen3-Coder-Plus = 1M, Llama = 131K.

**Provider prefix stripping** (Source location: `model_metadata.py:47-103`): 70+ provider prefixes ("openrouter:", "anthropic:", "local:", etc.), but preserves Ollama `model:tag` format ("qwen3.5:27b" is not stripped).

### Comparison Table: Model Metadata Probing Priority

| Priority | Source | Cache TTL | Typical Coverage |
|--------|------|----------|----------|
| 1 | Hardcoded `DEFAULT_CONTEXT_LENGTHS` | Infinite | Mainstream Claude/GPT/Gemini models |
| 2 | models.dev remote API | 1h | Newly released models, Ollama models |
| 3 | OpenRouter `/api/v1/models` | 1h | OpenRouter routed models |
| 4 | Anthropic `/v1/models` | 5min | Anthropic model family |
| 5 | `CONTEXT_PROBE_TIERS` tier-by-tier probing | None | Completely unknown models |

---

## 8. Provider Plugin System

The `plugins/model-providers/` directory contains 28 sub-plugins, each declaring a ProviderProfile instance via `plugin.py`. The plugin system allows third-party providers to auto-register through directory scanning without modifying core code.

28 sub-plugins: ai-gateway, alibaba, alibaba-coding-plan, anthropic, arcee, azure-foundry, bedrock, copilot, copilot-acp, custom, deepseek, gemini, gmi, huggingface, kilocode, kimi-coding, minimax, nous, nvidia, ollama-cloud, openai-codex, opencode-zen, openrouter, qwen-oauth, stepfun, xai, xiaomi, zai.

---

## Summary Table

| Component | File | Lines | Core Responsibility |
|------|------|------|----------|
| ProviderProfile | `providers/base.py` | 165 | Declarative provider description base class |
| Anthropic Adapter | `agent/anthropic_adapter.py` | 1970 | Anthropic Messages API adaptation and thinking modes |
| Gemini Native Adapter | `agent/gemini_native_adapter.py` | 965 | Gemini native REST API adaptation |
| Bedrock Adapter | `agent/bedrock_adapter.py` | 1264 | AWS Bedrock Converse API adaptation |
| Codex Responses Adapter | `agent/codex_responses_adapter.py` | 999 | OpenAI Responses API adaptation |
| AuxiliaryClient | `agent/auxiliary_client.py` | 4048 | Auxiliary task LLM routing and fallback |
| CredentialPool | `agent/credential_pool.py` | 1584 | Multi-credential pool management and 4 rotation strategies |
| RateLimitTracker | `agent/rate_limit_tracker.py` | 246 | Rate limit header parsing and visualization |
| ErrorClassifier | `agent/error_classifier.py` | 1036 | 18 failure classifications and recovery path decisions |
| Model Metadata | `agent/model_metadata.py` | 1483 | Model context length probing and tiered fallback |
| Provider Plugins | `plugins/model-providers/` | ~2800+ | 28 provider Profile instance declarations |

---

[03 — Context and Compression](/en/chapters/03-agent-engine) | [05 — Tool System](/en/chapters/05-tool-system)