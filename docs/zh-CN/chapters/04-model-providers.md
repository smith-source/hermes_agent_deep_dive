# 04 — LLM 提供商体系: 28+ 提供商通过声明式 Profile 统一适配、凭据池轮转与智能故障分类

[05 — 工具体系](/zh-CN/chapters/05-tool-system) | [06 — Gateway 消息平台](/zh-CN/chapters/06-gateway-system)

---

## 源码文件

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `providers/base.py` | 165 | ProviderProfile 声明式基类 |
| `agent/anthropic_adapter.py` | 1970 | Anthropic Messages API 适配 |
| `agent/gemini_native_adapter.py` | 965 | Gemini 原生 REST API 适配 |
| `agent/bedrock_adapter.py` | 1264 | AWS Bedrock Converse API 适配 |
| `agent/codex_responses_adapter.py` | 999 | OpenAI Responses API 适配 |
| `agent/auxiliary_client.py` | 4048 | 辅助任务 LLM 路由与 fallback |
| `agent/credential_pool.py` | 1584 | 多凭据池管理与轮转策略 |
| `agent/rate_limit_tracker.py` | 246 | 速率限制状态解析与可视化 |
| `agent/error_classifier.py` | 1036 | API 错误分类与 failover 决策 |
| `agent/model_metadata.py` | 1483 | 模型元数据/上下文长度探测 |
| `plugins/model-providers/` (28 子目录) | ~2800+ | 提供商插件声明 |

## 一句话总结

Hermes 通过 ProviderProfile 声明式描述、6 个原生 API 适配器、CredentialPool 凭据轮转和 ErrorClassifier 故障分类，将 28+ 异构 LLM 提供商统一为一致的调用界面，实现自动 failover 与负载均衡。

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

Hermes 的 LLM 提供商体系将 28+ 异构提供商（从 OpenRouter 聚合器到 Anthropic 原生 API、AWS Bedrock Converse、Gemini 原生 REST）统一为单一调用界面。ProviderProfile 以纯声明式 dataclass 描述每个提供商的身份、认证、端点和请求怪癖，使传输层无需 20+ boolean flag。6 个原生适配器隔离了 Anthropic Messages、Gemini generateContent、Bedrock Converse、OpenAI Responses 等协议差异。CredentialPool 支持 fill_first/round_robin/random/least_used 四种凭据轮转策略，自动冷却 exhausted 凭据。ErrorClassifier 将 API 错误分为 18 种 FailoverReason，驱动 retry/rotate/compress/fallback/abort 五种恢复路径。

---

## 1. ProviderProfile — 声明式提供商描述

`ProviderProfile` 是整个提供商体系的基石。它以纯声明式 dataclass 封装提供商的全部行为特征，替代了旧架构中散落在各处的 20+ boolean flag。

源码位置: `providers/base.py:25-166`

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

核心设计原则: Profile 只描述行为，不拥有客户端构造、凭据轮转或流式处理。那些职责留在 AIAgent 上。Profile 提供可覆写的钩子方法:

```python
# 源码位置: providers/base.py:80-165

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

`OMIT_TEMPERATURE` 哨兵值处理 Kimi/Moonshot 等服务商端管理 temperature 的场景——发送任何值（即使"正确"）都会与网关侧模式选择冲突。

### 比较表: ProviderProfile 字段分类

| 类别 | 字段 | 类型 | 用途示例 |
|------|------|------|----------|
| Identity | `name`, `api_mode`, `aliases` | str/tuple | "openrouter", "chat_completions", ("or",) |
| Auth | `env_vars`, `base_url`, `auth_type` | tuple/str | ("OPENROUTER_API_KEY",), "api_key" |
| Catalog | `fallback_models`, `hostname` | tuple/str | ("gpt-4o", "claude-3.5-sonnet") |
| Request quirks | `fixed_temperature`, `default_max_tokens` | Any/int/None | OMIT_TEMPERATURE, 4096 |
| Hooks | `prepare_messages()`, `build_extra_body()` | method | Anthropic message rewrite, Kimi reasoning |

### 比较表: auth_type 认证模式

| auth_type | 提供商 | 认证机制 | 代码位置 |
|-----------|--------|----------|----------|
| `api_key` | OpenRouter, DeepSeek, xAI | Bearer header from env var | `credential_pool.py:55` |
| `oauth_device_code` | Nous Portal | OAuth device code flow → JWT | `hermes_cli/auth.py` |
| `oauth_external` | Copilot, MiniMax-OAuth | External OAuth → token exchange | `hermes_cli/auth.py` |
| `copilot` | GitHub Copilot | Copilot API token via OAuth | `agent/copilot_adapter.py` |
| `aws_sdk` | AWS Bedrock | IAM role / SSO / instance metadata | `bedrock_adapter.py:48-58` |

---

## 2. 原生适配器深度解析

### 2.1 Anthropic Adapter

源码位置: `agent/anthropic_adapter.py:1-1970`

Anthropic 适配器是最大的原生适配器（1970 行），处理 Anthropic Messages API 的全部协议差异。关键特性:

**思考模式映射** (源码位置: `anthropic_adapter.py:47-79`):

```python
THINKING_BUDGET = {"xhigh": 32000, "high": 16000, "medium": 8000, "low": 4000}

# Claude 4.7 新增 xhigh 级别 (介于 high 和 max 之间)
ADAPTIVE_EFFORT_MAP = {
    "max": "max", "xhigh": "xhigh", "high": "high",
    "medium": "medium", "low": "low", "minimal": "low",
}

# 4.7+ 模型接受 xhigh; 4.6 模型只有 low/medium/high/max
_XHIGH_EFFORT_SUBSTRINGS = ("4-7", "4.7")
_ADAPTIVE_THINKING_SUBSTRINGS = ("4-6", "4.6", "4-7", "4.7")
_NO_SAMPLING_PARAMS_SUBSTRINGS = ("4-7", "4.7")  # 4.7 禁止 temperature/top_p/top_k
```

**max_tokens 分级表** (源码位置: `anthropic_adapter.py:86-117`):

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

**认证支持三种路径**: (1) 标准 API key (`sk-ant-api*`) → `x-api-key` header; (2) OAuth setup-token (`sk-ant-oat*`) → Bearer auth + beta header; (3) Claude Code credentials (`~/.claude.json`) → Bearer auth。

**延迟 SDK 导入** (源码位置: `anthropic_adapter.py:31-43`): anthropic SDK 导入约 220ms，使用哨兵+缓存模式延迟到首次调用。

### 2.2 Gemini Native Adapter

源码位置: `agent/gemini_native_adapter.py:1-965`

Gemini 适配器绕过 Google 的 OpenAI 兼容端点（该端点在多轮工具调用场景下不稳定），直接对接 `models/{model}:generateContent` 原生 REST API。

**免费层探测** (源码位置: `gemini_native_adapter.py:47-118`):

```python
def probe_gemini_tier(api_key, base_url=DEFAULT_GEMINI_BASE_URL, *,
                      model="gemini-2.5-flash", timeout=10.0) -> str:
    """Probe API key and return tier: 'free', 'paid', or 'unknown'.
    Free tier: <= 250 req/day for flash, unusable with Hermes agent loop."""
```

通过 `x-ratelimit-limit-requests-per-day` header 区分免费层（<= 1000）和付费层（> 1000）。免费层密钥直接被拒绝——Hermes 每轮 3-10 次 API 调用，免费层仅能支撑几条消息。

**多模态转换** (源码位置: `gemini_native_adapter.py:159-200`): 将 OpenAI 格式 `image_url` (data: URI) 转换为 Gemini `inlineData` (base64 + mimeType)。

### 2.3 Bedrock Adapter

源码位置: `agent/bedrock_adapter.py:1-1264`

Bedrock 适配器使用 AWS Converse API 统一接口，支持 Claude、Nova、Llama、Mistral 等所有 Bedrock 模型，完全绕过 OpenAI 兼容端点。

**惰性 boto3 导入与缓存** (源码位置: `bedrock_adapter.py:44-87`):

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

**陈旧连接检测** (源码位置: `bedrock_adapter.py:125-196`): 检测 boto3 缓存 HTTPS 连接池中的死连接（`ConnectionClosedError`, `ProtocolError`, 库内 `AssertionError`），并驱逐区域缓存客户端以重建连接池。

```python
def is_stale_connection_error(exc: BaseException) -> bool:
    """Match botocore ConnectionError, urllib3 ProtocolError,
    and library-internal AssertionError from urllib3/botocore/boto3 frames."""
```

### 2.4 Codex Responses Adapter

源码位置: `agent/codex_responses_adapter.py:1-999`

处理 OpenAI Responses API 格式（Codex、xAI、GitHub Models 使用）。核心功能:

**确定性 call_id** (源码位置: `codex_responses_adapter.py:143-152`): 避免 UUID 导致 OpenAI prompt cache 失效。

```python
def _deterministic_call_id(fn_name: str, arguments: str, index: int = 0) -> str:
    """Deterministic ID from tool call content. Prevents cache invalidation."""
    seed = f"{fn_name}:{arguments}:{index}"
    digest = hashlib.sha256(seed.encode()).hexdigest()[:12]
    return f"call_{digest}"
```

**工具调用泄漏过滤** (源码位置: `codex_responses_adapter.py:37-40`): 匹配 Codex/Harmony 序列化泄漏模式 `to=functions.<name>`。

### 比较表: 原生适配器特性对比

| 适配器 | 协议 | 认证 | 思考支持 | 行数 |
|--------|------|------|----------|------|
| Anthropic | Messages API | API key + OAuth + Claude Code | Adaptive thinking (4 levels + xhigh on 4.7) | 1970 |
| Gemini | generateContent REST | API key query param | Free-tier probe, paid-only enforcement | 965 |
| Bedrock | Converse API | AWS IAM chain (role/SSO/instance) | Stale-connection detection + client eviction | 1264 |
| Codex Responses | Responses API | OAuth device code | Deterministic call_id for cache | 999 |

---

## 3. CredentialPool — 凭据池管理与轮转

源码位置: `agent/credential_pool.py:1-1584`

CredentialPool 是 Hermes 多凭据 failover 的核心。它为每个提供商维护一组 `PooledCredential`，支持四种轮转策略和自动冷却。

**凭据数据结构** (源码位置: `credential_pool.py:82-171`):

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

**四种轮转策略** (源码位置: `credential_pool.py:59-68`):

```python
STRATEGY_FILL_FIRST = "fill_first"      # 用完一个再换下一个
STRATEGY_ROUND_ROBIN = "round_robin"    # 均匀轮转
STRATEGY_RANDOM = "random"              # 随机选择
STRATEGY_LEAST_USED = "least_used"      # 最少使用优先
```

**Exhausted 冷却机制** (源码位置: `credential_pool.py:71-74,191-195`):

```python
EXHAUSTED_TTL_429_SECONDS = 60 * 60      # 429 rate-limit: 1h cooldown
EXHAUSTED_TTL_DEFAULT_SECONDS = 60 * 60  # default: 1h cooldown

def _exhausted_ttl(error_code: Optional[int]) -> int:
    if error_code == 429:
        return EXHAUSTED_TTL_429_SECONDS
    return EXHAUSTED_TTL_DEFAULT_SECONDS
```

Provider-supplied `reset_at` 时间戳覆盖默认冷却期。

**时间戳解析** (源码位置: `credential_pool.py:198-238`): 支持 epoch seconds、epoch milliseconds、ISO-8601 字符串，以及嵌入式 `retryDelay` 文本解析。

### 比较表: 轮转策略对比

| 策略 | 选择逻辑 | 适用场景 | 简单度 |
|------|----------|----------|--------|
| `fill_first` | 优先级排序，用完再换 | 单主凭据 + 备用 | 最高 |
| `round_robin` | 均匀轮流 | 多凭据负载均衡 | 中 |
| `random` | 随机抽取 | 无状态分发 | 高 |
| `least_used` | `request_count` 最少 | 均匀磨损 | 低 |

---

## 4. ErrorClassifier — 智能故障分类与 Failover

源码位置: `agent/error_classifier.py:1-1036`

ErrorClassifier 提供 18 种 FailoverReason 分类，驱动 retry/rotate/compress/fallback/abort 五种恢复路径。

**故障分类体系** (源码位置: `error_classifier.py:24-61`, 定义而非导入):

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

**分类结果带恢复提示** (源码位置: `error_classifier.py:67-87`):

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

**多级模式匹配** (源码位置: `error_classifier.py:93-300`): Classifier 维护 10+ 组正则模式，从 billing exhaustion（"insufficient credits", "top up your credits"）到 rate limiting（"rate limit", "resource_exhausted", "throttlingexception"），再到 context overflow（"context length", "max_model_len", "超过最大长度"），以及 OpenRouter 策略拦截（"no endpoints available matching your guardrail"）。

### 比较表: FailoverReason 到恢复路径映射 (18 种,按类别分组)

| 类别 | FailoverReason | HTTP 状态码 | 恢复路径 | retryable |
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

## 5. AuxiliaryClient — 辅助任务 LLM 路由

源码位置: `agent/auxiliary_client.py:1-4048`

AuxiliaryClient 为侧任务（上下文压缩、视觉分析、网页提取、会话搜索）提供统一的 LLM 路由与 fallback 链。

**文本任务自动检测链** (源码位置: `auxiliary_client.py:7-16`):

```
1. User's main provider + main model (任何提供商类型)
2. OpenRouter (OPENROUTER_API_KEY)
3. Nous Portal (~/.hermes/auth.json)
4. Custom endpoint (config.yaml + OPENAI_API_KEY)
5. Native Anthropic
6. Direct API-key providers (z.ai, Kimi, MiniMax)
7. None (无可用后端)
```

**提供商别名系统** (源码位置: `auxiliary_client.py:131-162`): 30+ 别名映射，如 `"google"→"gemini"`, `"grok"→"xai"`, `"kimi"→"kimi-coding"`, `"claude"→"anthropic"`。

**OpenAI SDK 延迟代理** (源码位置: `auxiliary_client.py:69-100`): OpenAI SDK 导入约 240ms，使用 `_OpenAIProxy` 类实现模块级延迟代理，支持 `isinstance` 检查和 patch mock。

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

## 6. RateLimitTracker — 速率限制可视化

源码位置: `agent/rate_limit_tracker.py:1-246`

RateLimitTracker 解析 `x-ratelimit-*` 响应头（12 个 header），提供终端可视化和 compact 状态栏两种输出格式。

**数据模型** (源码位置: `rate_limit_tracker.py:30-68`):

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

**ASCII 进度条输出** (源码位置: `rate_limit_tracker.py:159-246`):

```
Nous Portal Rate Limits (captured just now):

  Requests/min   [████████░░░░░░░░░░░░] 40.0%  320/800 used  (480 left, resets in 42s)
  Requests/hr    [██░░░░░░░░░░░░░░░░░░] 10.5%  800/7600 used  (6.8K left, resets in 58m 14s)

  Tokens/min     [████████████░░░░░░░░] 60.0%  300K/500K used  (200K left, resets in 42s)
  Tokens/hr      [██░░░░░░░░░░░░░░░░░░]  8.0%  2.0M/25M used  (23.0M left, resets in 58m 14s)
```

> 80% 使用率时自动发出警告。

---

## 7. Model Metadata — 上下文长度探测

源码位置: `agent/model_metadata.py:1-1483`

Model metadata 系统维护模型上下文长度分级表、provider prefix stripping 和多层探测 fallback。

**分级探测序列** (源码位置: `model_metadata.py:116-131`):

```python
CONTEXT_PROBE_TIERS = [256_000, 128_000, 64_000, 32_000, 16_000, 8_000]
DEFAULT_FALLBACK_CONTEXT = 256_000
MINIMUM_CONTEXT_LENGTH = 64_000  # 低于此值拒绝运行
```

**上下文长度 fallback 表** (源码位置: `model_metadata.py:137-200`): Claude 4.6+ = 1M, GPT-5.4 = 1.05M, DeepSeek V4 = 1M, Gemini = 1M, Qwen3-Coder-Plus = 1M, Llama = 131K。

**Provider prefix stripping** (源码位置: `model_metadata.py:47-103`): 70+ provider 前缀（"openrouter:", "anthropic:", "local:" 等），但保留 Ollama `model:tag` 格式（"qwen3.5:27b" 不被剥离）。

### 比较表: Model Metadata 探测优先级

| 优先级 | 来源 | 缓存 TTL | 典型覆盖 |
|--------|------|----------|----------|
| 1 | Hardcoded `DEFAULT_CONTEXT_LENGTHS` | 无限 | Claude/GPT/Gemini 主流模型 |
| 2 | models.dev 远程 API | 1h | 新发布模型、Ollama 模型 |
| 3 | OpenRouter `/api/v1/models` | 1h | OpenRouter 路由模型 |
| 4 | Anthropic `/v1/models` | 5min | Anthropic 模型族 |
| 5 | `CONTEXT_PROBE_TIERS` 逐级试探 | 无 | 完全未知模型 |

---

## 8. 提供商插件系统

`plugins/model-providers/` 目录下有 28 个子插件，每个以 `plugin.py` 声明 ProviderProfile 实例。插件系统允许第三方提供商通过目录扫描自动注册，无需修改核心代码。

28 个子插件: ai-gateway, alibaba, alibaba-coding-plan, anthropic, arcee, azure-foundry, bedrock, copilot, copilot-acp, custom, deepseek, gemini, gmi, huggingface, kilocode, kimi-coding, minimax, nous, nvidia, ollama-cloud, openai-codex, opencode-zen, openrouter, qwen-oauth, stepfun, xai, xiaomi, zai。

---

## Summary Table

| 组件 | 文件 | 行数 | 核心职责 |
|------|------|------|----------|
| ProviderProfile | `providers/base.py` | 165 | 声明式提供商描述基类 |
| Anthropic Adapter | `agent/anthropic_adapter.py` | 1970 | Anthropic Messages API 适配与思考模式 |
| Gemini Native Adapter | `agent/gemini_native_adapter.py` | 965 | Gemini 原生 REST API 适配 |
| Bedrock Adapter | `agent/bedrock_adapter.py` | 1264 | AWS Bedrock Converse API 适配 |
| Codex Responses Adapter | `agent/codex_responses_adapter.py` | 999 | OpenAI Responses API 适配 |
| AuxiliaryClient | `agent/auxiliary_client.py` | 4048 | 辅助任务 LLM 路由与 fallback |
| CredentialPool | `agent/credential_pool.py` | 1584 | 多凭据池管理与 4 种轮转策略 |
| RateLimitTracker | `agent/rate_limit_tracker.py` | 246 | 速率限制 header 解析与可视化 |
| ErrorClassifier | `agent/error_classifier.py` | 1036 | 18 种故障分类与恢复路径决策 |
| Model Metadata | `agent/model_metadata.py` | 1483 | 模型上下文长度探测与分级 fallback |
| Provider Plugins | `plugins/model-providers/` | ~2800+ | 28 提供商 Profile 实例声明 |

---

[03 — 上下文与压缩](/zh-CN/chapters/03-agent-engine) | [05 — 工具体系](/zh-CN/chapters/05-tool-system)