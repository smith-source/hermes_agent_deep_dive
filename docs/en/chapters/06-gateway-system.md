# 06 — Gateway Messaging Platform: 21+ Platform Adapters + Streaming Consumer + Session Management + API Server Unified Messaging Gateway

[05 — Tool System](/en/chapters/05-tool-system) | [04 — LLM Provider System](/en/chapters/04-model-providers)

---

## Source Files

| File | Lines | Core Responsibility |
|------|------|----------|
| `gateway/run.py` | 15046 | GatewayRunner main entry point and lifecycle management |
| `gateway/session.py` | 1387 | SessionStore session context and persistence |
| `gateway/stream_consumer.py` | 1018 | GatewayStreamConsumer stream delta-to-platform message bridging |
| `gateway/config.py` | 1636 | GatewayConfig configuration management and Platform enum |
| `gateway/platforms/base.py` | 3390 | BasePlatformAdapter ABC and message truncation |
| `gateway/platforms/telegram.py` | 3661 | Telegram Bot API adapter |
| `gateway/platforms/discord.py` | 4900 | Discord Bot adapter (largest) |
| `gateway/platforms/signal.py` | 1516 | Signal adapter |
| `gateway/platforms/matrix.py` | 2676 | Matrix adapter |
| `gateway/platforms/slack.py` | 2926 | Slack Bolt adapter |
| `gateway/platforms/email.py` | 748 | SMTP/IMAP email adapter |
| `gateway/platforms/feishu.py` | 4831 | Feishu/Lark adapter |
| `gateway/platforms/dingtalk.py` | 1366 | DingTalk adapter |
| `gateway/platforms/wecom.py` | 1609 | WeCom (Enterprise WeChat) adapter |
| `gateway/platforms/weixin.py` | 2110 | WeChat Official Account adapter |
| `gateway/platforms/yuanbao.py` | 4756 | Tencent Yuanbao/QQBot adapter |
| `gateway/platforms/yuanbao_proto.py` | 1209 | Yuanbao/QQBot protocol layer (GrpcBridge + decoding) |
| `gateway/platforms/yuanbao_media.py` | 645 | Yuanbao/QQBot media processing (image/file/voice) |
| `gateway/platforms/yuanbao_sticker.py` | 558 | Yuanbao/QQBot sticker system (custom stickers/shop) |
| `gateway/platforms/whatsapp.py` | 1104 | WhatsApp adapter |
| `gateway/platforms/homeassistant.py` | 449 | Home Assistant adapter |
| `gateway/platforms/mattermost.py` | 832 | Mattermost adapter |
| `gateway/platforms/bluebubbles.py` | 937 | BlueBubbles (iMessage) adapter |
| `gateway/platforms/sms.py` | 377 | SMS adapter |
| `gateway/platforms/webhook.py` | 771 | Webhook generic adapter |
| `gateway/platforms/api_server.py` | 3172 | OpenAI-compatible API Server |
| `gateway/delivery.py` | 258 | DeliveryRouter cron output routing |
| `gateway/pairing.py` | 310 | PairingStore DM authorization code approval |
| `gateway/hooks.py` | 210 | HookRegistry event hook system |
| `gateway/status.py` | 861 | GatewayStatus status monitoring and reporting |
| `gateway/channel_directory.py` | 357 | ChannelDirectory channel directory management |
| `gateway/platform_registry.py` | 212 | PlatformRegistry dynamic plugin platform registration |
| `gateway/display_config.py` | 196 | DisplayConfig TUI display layout configuration |
| `gateway/mirror.py` | 178 | MirrorStore channel mirror synchronization |
| `gateway/whatsapp_identity.py` | 155 | WhatsAppIdentity WA number identity resolution |
| `gateway/session_context.py` | 154 | SessionContext ContextVars per-job session state |
| `gateway/runtime_footer.py` | 150 | RuntimeFooter Gateway process info footer |
| `gateway/sticker_cache.py` | 111 | StickerCache sticker file cache management |

## One-Line Summary

Hermes Gateway is a 15000+ line messaging platform gateway that unifies 21+ messaging platforms (Telegram/Discord/Signal/Matrix/Slack/Feishu/DingTalk/WeCom/WeChat/Yuanbao/QQBot/WhatsApp/Email/SMS/Webhook/HomeAssistant/Mattermost/BlueBubbles/iMessage/API Server) through the BasePlatformAdapter ABC, bridges synchronous agent delta streams to asynchronous platform message progressive edits via GatewayStreamConsumer, manages multi-platform session context and reset strategies via SessionStore, provides OpenAI-compatible HTTP endpoints via API Server, routes cron output via DeliveryRouter, implements DM authorization code approval via PairingStore, and provides an event hook system via HookRegistry.

---

## Architecture Overview

```
                    ┌───────────────────────────────────────────────────────────┐
                    │                    GatewayRunner                          │
                    │                  (gateway/run.py:15046)                    │
                    │                                                         │
                    │   ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
                    │   │ SessionStore │  │GatewayConfig │  │ DeliveryRouter│  │
                    │   │ (session.py) │  │ (config.py)  │  │ (delivery.py) │  │
                    │   │ 1387 lines   │  │ 1636 lines   │  │ 258 lines    │  │
                    │   └──────────────┘  └──────────────┘  └───────────────┘  │
                    │                                                         │
                    │   ┌──────────────────┐  ┌───────────────────────────┐    │
                    │   │  StreamConsumer  │  │     PairingStore          │    │
                    │   │ (stream_consumer)│  │     (pairing.py)          │    │
                    │   │ 1018 lines       │  │     310 lines             │    │
                    │   └──────────────────┘  └───────────────────────────┘    │
                    │                                                         │
                    │   ┌──────────────────┐                                   │
                    │   │  HookRegistry    │                                   │
                    │   │ (hooks.py:210)   │                                   │
                    │   └──────────────────┘                                   │
                    │                                                         │
                    │        ┌─────────────────────────────────┐               │
                    │        │   BasePlatformAdapter (ABC)      │               │
                    │        │   (platforms/base.py:3390)       │               │
                    │        │   send / send_edit / truncate    │               │
                    │        └──────────────────┬───────────────┘               │
                    │                           │                               │
                    │    ┌──────────────────────┼───────────────────────┐       │
                    │    │         │         │         │         │      │       │
                    │    ▼         ▼         ▼         ▼         ▼      │       │
                    │  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐ │       │
                    │  │TG  │  │DC  │  │SG  │  │MT  │  │SL  │  │EM  │ │       │
                    │  │3661│  │4900│  │1516│  │2676│  │2926│  │748│ │       │
                    │  └────┘  └────┘  └────┘  └────┘  └────┘  └────┘ │       │
                    │  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐ │       │
                    │  │FS  │  │DT  │  │WC  │  │WX  │  │YB  │  │HA  │ │       │
                    │  │4831│  │1366│  │1609│  │2110│  │7168│  │449│ │       │
                    │  └────┘  └────┘  └────┘  └────┘  └────┘  └────┘ │       │
                    │  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐ │       │
                    │  │MM  │  │BB  │  │WA  │  │SMS │  │WH  │  │API │ │       │
                    │  │832 │  │937 │  │1104│  │377 │  │771 │  │3172│ │       │
                    │  └────┘  └────┘  └────┘  └────┘  └────┘  └────┘ │       │
                    └───────────────────────────────────────────────────────────┘
```

---

## TL;DR

Hermes Gateway centers on the 15000+ line GatewayRunner, managing concurrent connections and lifecycles of 21+ messaging platform adapters. BasePlatformAdapter ABC (3390 lines) defines common interfaces like send/send_edit/truncate_message, including UTF-16 length calculation (Telegram 4096 code-unit limit) and binary search truncation. GatewayStreamConsumer (1018 lines) bridges synchronous agent stream deltas to asyncio via queue.Queue, progressively editing platform messages in edit-transport mode. SessionStore (1387 lines) manages multi-platform session context and PII hashing. GatewayConfig (1636 lines) Platform enum supports 21 built-in members + dynamic plugin extensions. The Yuanbao/QQBot adapter (4756 lines main module + 1209 lines protocol layer + 645 lines media + 558 lines stickers) covers both Tencent Yuanbao and QQ Bot platforms. The WhatsApp adapter (1104 lines) covers the WhatsApp Business API. API Server (3172 lines) provides OpenAI-compatible `/v1/chat/completions` and `/v1/responses` endpoints. GatewayStatus (861 lines) provides status monitoring and reporting. PairingStore (310 lines) implements a DM authorization code approval flow guided by OWASP/NIST. HookRegistry (210 lines) supports wildcard event matching.

---

## 1. GatewayRunner — Gateway Main Entry Point

Source location: `gateway/run.py:1-15046`

GatewayRunner is the gateway's 15000+ line main module, managing concurrent connections, agent caching, auto-continue, scheduled tasks, and lifecycle for all platform adapters.

**Agent Cache Capacity** (Source location: `run.py:50-53`):

```python
_AGENT_CACHE_MAX_SIZE = 128              # per-session AIAgent cache
_AGENT_CACHE_IDLE_TTL_SECS = 3600.0      # evict agents idle for >1h
_PLATFORM_CONNECT_TIMEOUT_SECS_DEFAULT = 30.0
```

**Auto-Continue Freshness Window** (Source location: `run.py:88-148`): Interrupted gateway turns only auto-continue within the freshness window. Default 1 hour, overriding `agent.gateway_timeout` (30 minutes) + runtime margin.

```python
_AUTO_CONTINUE_FRESHNESS_SECS_DEFAULT = 60 * 60

def _auto_continue_freshness_window() -> float:
    """Read HERMES_AUTO_CONTINUE_FRESHNESS env var. Non-positive values disable."""
    raw = os.environ.get("HERMES_AUTO_CONTINUE_FRESHNESS")
    if raw is None or raw == "":
        return float(_AUTO_CONTINUE_FRESHNESS_SECS_DEFAULT)
    try:
        return float(raw)
    except (TypeError, ValueError):
        return float(_AUTO_CONTINUE_FRESHNESS_SECS_DEFAULT)
```

**Telegram Command Name Normalization** (Source location: `run.py:56-73`): Telegram Bot API command names only allow lowercase letters, digits, and underscores. `_telegramize_command_mentions()` converts slash command references to Telegram-valid format.

```python
_TELEGRAM_COMMAND_MENTION_RE = re.compile(r"(?<![\w:/])/([A-Za-z0-9][A-Za-z0-9_-]*)")

def _telegramize_command_mentions(text: str, platform: Any) -> str:
    """Rewrite slash-command mentions to Telegram-valid command names."""
    platform_value = getattr(platform, "value", platform)
    if platform_value != "telegram":
        return text
    from hermes_cli.commands import _sanitize_telegram_name
    def _replace(match):
        sanitized = _sanitize_telegram_name(match.group(1))
        return f"/{sanitized}" if sanitized else match.group(0)
    return _TELEGRAM_COMMAND_MENTION_RE.sub(_replace, text)
```

**Timestamp Coercion** (Source location: `run.py:97-128`): `_coerce_gateway_timestamp()` accepts datetime, epoch seconds/milliseconds, ISO-8601 strings (with or without `Z`), covering timestamp format differences across all platforms.

### Comparison Table: GatewayRunner Core Parameters

| Parameter | Default | Environment Variable | Effect |
|------|--------|----------|------|
| Agent cache size | 128 | — | per-session AIAgent max cache count |
| Agent idle TTL | 3600s | — | Idle agent eviction time |
| Platform connect timeout | 30s | — | Platform adapter connection timeout |
| Auto-continue freshness | 3600s | `HERMES_AUTO_CONTINUE_FRESHNESS` | Interrupted turn continuation window |
| Agent gateway timeout | 1800s | `HERMES_AGENT_TIMEOUT` | Max runtime per single turn |

---

## 2. BasePlatformAdapter — Platform Adapter ABC

Source location: `gateway/platforms/base.py:1-3390`

BasePlatformAdapter defines the common interface and helper functions for all platform adapters, serving as the shared base class for 19+ adapters.

**Message Truncation — Precise UTF-16 Calculation** (Source location: `base.py:66-116`): Telegram's message length limit (4096) is measured in UTF-16 code units, not Unicode code points. Emoji and CJK Extension B characters consume two UTF-16 units each.

```python
def utf16_len(s: str) -> int:
    """Count UTF-16 code units in s.
    Telegram's message-length limit (4096) is measured in UTF-16 code units,
    not Unicode code-points. Characters outside BMP consume two units each."""
    return len(s.encode("utf-16-le")) // 2

def _prefix_within_utf16_limit(s: str, limit: int) -> str:
    """Return longest prefix of s whose UTF-16 length <= limit.
    Binary search respects surrogate-pair boundaries."""
    if utf16_len(s) <= limit:
        return s
    lo, hi = 0, len(s)
    while lo < hi:
        mid = (lo + hi + 1) // 2
        if utf16_len(s[:mid]) <= limit:
            lo = mid
        else:
            hi = mid - 1
    return s[:lo]
```

**Audio Send Decision** (Source location: `base.py:29-63`): Different platforms support different audio formats. Telegram Bot API only accepts MP3/M4A via `sendAudio` and Opus/OGG via `sendVoice`; other audio formats are delivered as documents.

```python
_AUDIO_EXTS = frozenset({'.ogg', '.opus', '.mp3', '.wav', '.m4a', '.flac'})
_TELEGRAM_AUDIO_ATTACHMENT_EXTS = frozenset({'.mp3', '.m4a'})
_TELEGRAM_VOICE_EXTS = frozenset({'.ogg', '.opus'})

def should_send_media_as_audio(platform, ext, is_voice=False) -> bool:
    normalized_ext = (ext or "").lower()
    if normalized_ext not in _AUDIO_EXTS:
        return False
    if _platform_name(platform) == "telegram":
        if normalized_ext in _TELEGRAM_VOICE_EXTS:
            return is_voice           # Opus/OGG only as voice bubble
        return normalized_ext in _TELEGRAM_AUDIO_ATTACHMENT_EXTS
    return True                       # other platforms: all audio goes through audio sender
```

**Network Accessibility Detection** (Source location: `base.py:119-151`): Detects whether a bound address exposes the server beyond loopback. Handles IPv4-mapped IPv6 (`::ffff:127.0.0.1`) and hostname DNS resolution.

```python
def is_network_accessible(host: str) -> bool:
    """Return True if host would expose the server beyond loopback."""
    try:
        addr = ipaddress.ip_address(host)
        if addr.is_loopback:
            return False
        if getattr(addr, "ipv4_mapped", None) and addr.ipv4_mapped.is_loopback:
            return False
        return True
    except ValueError:
        # hostname: resolve DNS, fail closed
        resolved = _socket.getaddrinfo(host, None, ...)
        for ... in resolved:
            addr = ipaddress.ip_address(sockaddr[0])
            if not addr.is_loopback:
                return True
        return False
```

**macOS System Proxy Detection** (Source location: `base.py:154-187`): Reads macOS system-level HTTP/HTTPS proxy settings via `scutil --proxy`.

### Comparison Table: Platform Adapter Scale Comparison

| Platform | File | Lines | Authentication Method | Message Model |
|------|------|------|----------|----------|
| Discord | `discord.py` | 4900 | Bot token | edit-based streaming |
| Feishu | `feishu.py` | 4831 | App ID + Secret | AI Card + edit |
| Yuanbao/QQBot | `yuanbao.py` + proto/media/sticker | 7168 (4756+1209+645+558) | OAuth token | edit-based streaming |
| WhatsApp | `whatsapp.py` | 1104 | WhatsApp Business API token | edit-based streaming |
| Telegram | `telegram.py` | 3661 | Bot token | edit-based streaming |
| Base | `base.py` | 3390 | ABC | truncate + UTF-16 |
| API Server | `api_server.py` | 3172 | Bearer/API key | SSE streaming |
| Slack | `slack.py` | 2926 | Bot token + Socket Mode | edit-based streaming |
| Matrix | `matrix.py` | 2676 | Access token | edit-based streaming |
| WeChat | `weixin.py` | 2110 | AppID + AppSecret | 5s timeout web reply |
| WeCom | `wecom.py` | 1609 | Corp ID + Secret | callback + push |
| Signal | `signal.py` | 1516 | signal-cli REST | edit-based |
| DingTalk | `dingtalk.py` | 1366 | AppKey + AppSecret | AI Card |
| BlueBubbles | `bluebubbles.py` | 937 | API key + password | REST API |
| Mattermost | `mattermost.py` | 832 | Personal access token | edit-based |
| Email | `email.py` | 748 | SMTP/IMAP creds | plain text |
| Webhook | `webhook.py` | 771 | Secret HMAC | HTTP POST |

---

## 3. GatewayStreamConsumer — Stream Bridging

Source location: `gateway/stream_consumer.py:1-1018`

GatewayStreamConsumer is the key bridge from synchronous agent delta streams to asynchronous platform message editing. The design is based on the edit-transport pattern: first send an initial message, then progressively editMessageText.

**Core Architecture** (Source location: `stream_consumer.py:15-69`):

```
Synchronous path (agent worker thread):
  stream_delta_callback(text) → consumer.on_delta()
                                      │
                                      ▼
                              queue.Queue (thread-safe)
                                      │
                                      ▼
Asynchronous path (asyncio task):          consumer.run()
  1. Take delta from queue
  2. Buffer + rate-limit
  3. Progressive edit of platform message
  4. Finalize final edit
```

**StreamConsumerConfig** (Source location: `stream_consumer.py:41-54`):

```python
@dataclass
class StreamConsumerConfig:
    edit_interval: float = 1.0             # Edit interval (seconds)
    buffer_threshold: int = 40             # Buffer character threshold
    cursor: str = " ▉"                    # Streaming cursor
    buffer_only: bool = False              # Buffer only, no sending
    fresh_final_after_seconds: float = 0.0 # Time threshold for re-sending long-running responses as new messages
```

**Flood Control Protection** (Source location: `stream_consumer.py:72-74,123-133`): After 3 consecutive flood control failures, progressive editing is permanently disabled, degrading to one-shot delivery of the complete response.

```python
_MAX_FLOOD_STRIKES = 3

self._flood_strikes = 0
self._current_edit_interval = self.cfg.edit_interval  # Adaptive backoff
self._adapter_requires_finalize: bool = (
    getattr(adapter, "REQUIRES_EDIT_FINALIZE", False) is True
)                                          # DingTalk AI Card requires explicit finalize
```

**Thinking Block Filtering** (Source location: `stream_consumer.py:79-87`): Mirrors the CLI's `_stream_delta` tag suppression logic, filtering tags like `REASONING_SCRATCHPAD`, `thinking`, `reasoning`, `thought`.

```python
_OPEN_THINK_TAGS = (
    "<REASONING_SCRATCHPAD>", "ibaba", "<reasoning>",
    "<THINKING>", "<thinking>", "<thought>",
)
_CLOSE_THINK_TAGS = (
    "</REASONING_SCRATCHPAD>", "ku", "</reasoning>",
    "</THINKING>", "</thinking>", "</thought>",
)
```

**Fresh-Final Logic** (Source location: `stream_consumer.py:51-54,114-116`): Long-running inference models (like Opus 4.7) stream slowly. When the original preview message has lived longer than `fresh_final_after_seconds`, the final edit is replaced by sending a brand-new message, so the platform timestamp reflects completion time rather than first-token time.

### Comparison Table: StreamConsumer Streaming Strategy Comparison

| Strategy | Applicable Platforms | User Experience | Implementation Complexity |
|------|----------|----------|------------|
| edit-based streaming | Telegram, Discord, Slack | Real-time updates to the same message | High (flood control, think block filter) |
| fresh-final resend | Long-running inference scenarios | More accurate completion timestamp | Medium |
| buffer-only | Configuration/debug scenarios | One-shot delivery of complete response | Low |
| AI Card finalize | DingTalk | Structured card rendering | Medium (requires explicit finalize call) |

---

## 4. SessionStore — Session Context and Persistence

Source location: `gateway/session.py:1-1387`

SessionStore manages multi-platform session context tracking, persistence, and reset strategies.

**SessionSource Data Structure** (Source location: `session.py:70-150`):

```python
@dataclass
class SessionSource:
    """Describes where a message originated from."""
    platform: Platform
    chat_id: str
    chat_name: Optional[str] = None
    chat_type: str = "dm"                  # "dm", "group", "channel", "thread"
    user_id: Optional[str] = None
    user_name: Optional[str] = None
    thread_id: Optional[str] = None        # forum topics, Discord threads
    chat_topic: Optional[str] = None       # channel topic/description
    user_id_alt: Optional[str] = None      # Signal UUID, Feishu union_id
    chat_id_alt: Optional[str] = None      # Signal group internal ID
    is_bot: bool = False                   # message author is a bot/webhook
    guild_id: Optional[str] = None         # Discord guild / Slack workspace
    parent_chat_id: Optional[str] = None   # parent channel for threads
    message_id: Optional[str] = None       # for pin/reply/react
```

**PII Hashing** (Source location: `session.py:34-54`): All user IDs and chat IDs are presented in logs as deterministic SHA-256[:12] hashes, preserving platform prefixes (`telegram:12345` → `telegram:<hash>`).

```python
def _hash_id(value: str) -> str:
    """Deterministic 12-char hex hash of an identifier."""
    return hashlib.sha256(value.encode("utf-8")).hexdigest()[:12]

def _hash_sender_id(value: str) -> str:
    return f"user_{_hash_id(value)}"

def _hash_chat_id(value: str) -> str:
    """Hash numeric portion, preserving platform prefix."""
    colon = value.find(":")
    if colon > 0:
        prefix = value[:colon]
        return f"{prefix}:{_hash_id(value[colon + 1:])}"
    return _hash_id(value)
```

**SessionSource.description** (Source location: `session.py:96-114`): Human-readable source description, used for system prompt injection.

```python
@property
def description(self) -> str:
    if self.platform == Platform.LOCAL:
        return "CLI terminal"
    parts = []
    if self.chat_type == "dm":
        parts.append(f"DM with {self.user_name or self.user_id or 'user'}")
    elif self.chat_type == "group":
        parts.append(f"group: {self.chat_name or self.chat_id}")
    elif self.chat_type == "channel":
        parts.append(f"channel: {self.chat_name or self.chat_id}")
    if self.thread_id:
        parts.append(f"thread: {self.thread_id}")
    return ", ".join(parts)
```

### Comparison Table: SessionSource Field Usage Comparison

| Field | Purpose | Platform Specificity | Example |
|------|------|----------|------|
| `chat_type` | System prompt injection | All | "dm", "group", "channel", "thread" |
| `thread_id` | Message routing | Discord/Slack/Matrix | "thread_123" |
| `user_id_alt` | Stable identity | Signal (UUID), Feishu (union_id) | "signal-uuid" |
| `guild_id` | Workspace scope | Discord/Slack | "guild_456" |
| `message_id` | pin/reply/react | Telegram/Discord | "msg_789" |
| `is_bot` | Filter bot messages | Discord | True |

---

## 5. GatewayConfig — Configuration Management

Source location: `gateway/config.py:1-1636`

GatewayConfig manages platform connection configuration, home channels, session reset strategies, and delivery preferences.

**Platform Enum — Dynamic Extension** (Source location: `config.py:82-150`): 21 built-in members + `_missing_()` dynamically creates plugin platform pseudo-members.

```python
class Platform(Enum):
    """Supported messaging platforms. Built-in + dynamic plugin members."""
    LOCAL = "local"
    TELEGRAM = "telegram"
    DISCORD = "discord"
    WHATSAPP = "whatsapp"
    SLACK = "slack"
    SIGNAL = "signal"
    MATTERMOST = "mattermost"
    MATRIX = "matrix"
    HOMEASSISTANT = "homeassistant"
    EMAIL = "email"
    SMS = "sms"
    DINGTALK = "dingtalk"
    API_SERVER = "api_server"
    WEBHOOK = "webhook"
    FEISHU = "feishu"
    WECOM = "wecom"
    WECOM_CALLBACK = "wecom_callback"
    WEIXIN = "weixin"
    BLUEBUBBLES = "bluebubbles"
    QQBOT = "qqbot"
    YUANBAO = "yuanbao"

    @classmethod
    def _missing_(cls, value):
        """Accept unknown platform names only for known plugin adapters.
        Creates identity-stable pseudo-members cached in _value2member_map_."""
        # filesystem scan for bundled plugin platforms
        # + runtime-registered plugin platforms
        # arbitrary strings rejected to prevent enum pollution
```

**Configuration Value Coercion** (Source location: `config.py:25-66`): `_coerce_bool()`, `_coerce_float()`, `_coerce_int()` safely convert configuration values, preventing format-error crashes.

**Unauthorized DM Behavior** (Source location: `config.py:59-74`): `_normalize_unauthorized_dm_behavior()` restricts to "pair" (initiate authorization code approval) or "ignore" (silently ignore).

```python
def _normalize_unauthorized_dm_behavior(value, default="pair") -> str:
    if isinstance(value, str):
        normalized = value.strip().lower()
        if normalized in {"pair", "ignore"}:
            return normalized
    return default
```

### Comparison Table: Platform Enum Extension Mechanism Comparison

| Extension Method | Trigger Condition | Lifecycle | Identity Stability |
|----------|----------|----------|------------|
| Built-in members | Module import time | Permanent | `Platform.TELEGRAM is Platform.TELEGRAM` |
| Filesystem scan pseudo-member | `_missing_()` first encounter of plugin name | Permanent (cached in `_value2member_map_`) | `Platform("irc") is Platform("irc")` |
| Runtime registered pseudo-member | `platform_registry.is_registered()` | Process lifecycle | Same as above |

---

## 6. API Server — OpenAI-Compatible Endpoints

Source location: `gateway/platforms/api_server.py:1-3172`

API Server provides OpenAI-compatible HTTP endpoints, allowing any OpenAI-compatible frontend (Open WebUI, LobeChat, LibreChat, AnythingLLM, NextChat, ChatBox, etc.) to connect to Hermes via `http://localhost:8642/v1`.

**Endpoint List** (Source location: `api_server.py:5-16`):

```
POST /v1/chat/completions          — OpenAI Chat Completions format
POST /v1/responses                 — OpenAI Responses API format
GET  /v1/responses/{response_id}   — Retrieve stored response
DELETE /v1/responses/{response_id} — Delete stored response
GET  /v1/models                    — list hermes-agent as available model
GET  /v1/capabilities              — machine-readable API capabilities
POST /v1/runs                      — start a run (202, returns run_id)
GET  /v1/runs/{run_id}             — retrieve run status
GET  /v1/runs/{run_id}/events      — SSE stream of lifecycle events
POST /v1/runs/{run_id}/stop        — interrupt running agent
GET  /health                       — health check
GET  /health/detailed              — rich status for dashboard probing
```

**Default Settings** (Source location: `api_server.py:56-63`):

```python
DEFAULT_HOST = "127.0.0.1"
DEFAULT_PORT = 8642
MAX_STORED_RESPONSES = 100
MAX_REQUEST_BYTES = 10_000_000            # 10 MB
MAX_NORMALIZED_TEXT_LENGTH = 65_536       # 64 KB cap
CHAT_COMPLETIONS_SSE_KEEPALIVE_SECONDS = 30.0
```

**Content Normalization** (Source location: `api_server.py:73-128`): `_normalize_chat_content()` flattens OpenAI Chat format multimodal content arrays into plain text strings. Defensive limits: recursion depth cap 10, list size cap 1000, output length cap 64KB.

```python
_TEXT_PART_TYPES = frozenset({"text", "input_text", "output_text"})
_IMAGE_PART_TYPES = frozenset({"image_url", "input_image"})
_FILE_PART_TYPES = frozenset({"file", "input_file"})     # unsupported → ValueError
```

**Session Continuity** (Source location: `api_server.py:5`): `X-Hermes-Session-Id` header provides opt-in session continuity (chat completions mode), `X-Hermes-Session-Key` header provides long-term memory scope.

### Comparison Table: API Server Endpoint Comparison

| Endpoint | Method | State | Streaming Support | Purpose |
|------|------|------|----------|------|
| `/v1/chat/completions` | POST | Stateless (opt-in session) | SSE | OpenAI-compatible frontends |
| `/v1/responses` | POST | Stateful (previous_response_id) | SSE | OpenAI Responses API |
| `/v1/models` | GET | Stateless | None | Model list |
| `/v1/runs` | POST | Asynchronous | None | Async run (returns run_id) |
| `/v1/runs/{id}/events` | GET | SSE | SSE | Lifecycle event stream |
| `/health/detailed` | GET | Stateless | None | Cross-container dashboard |

---

## 7. DeliveryRouter — Cron Output Routing

Source location: `gateway/delivery.py:1-258`

DeliveryRouter routes cron output and proxy responses to the correct target platform.

**DeliveryTarget Parsing** (Source location: `delivery.py:29-107`):

```python
@dataclass
class DeliveryTarget:
    """A single delivery target.
    "origin" → back to source
    "local" → save to local files
    "telegram" → Telegram home channel
    "telegram:123456" → specific Telegram chat"""

    platform: Platform
    chat_id: Optional[str] = None         # None = use home channel
    thread_id: Optional[str] = None
    is_origin: bool = False
    is_explicit: bool = False             # True if chat_id explicitly specified

    @classmethod
    def parse(cls, target, origin=None) -> "DeliveryTarget":
        """Formats:
        "origin" → back to source
        "local" → local files only
        "telegram" → home channel
        "telegram:123456" → specific chat
        "telegram:123456:thread_789" → specific thread"""
```

**Output Truncation Protection** (Source location: `delivery.py:21-23,242-249`): `MAX_PLATFORM_OUTPUT = 4000` characters; excess is truncated and saved to a disk file, with the file path reference attached.

```python
MAX_PLATFORM_OUTPUT = 4000
TRUNCATED_VISIBLE = 3800

if len(content) > MAX_PLATFORM_OUTPUT:
    saved_path = self._save_full_output(content, job_id)
    content = (
        content[:TRUNCATED_VISIBLE]
        + f"\n\n... [truncated, full output saved to {saved_path}]"
    )
```

### Comparison Table: Delivery Routing Type Comparison

| Routing Type | Format | Platform Selection | chat_id Selection |
|----------|------|----------|-------------|
| origin | `"origin"` | Message source platform | Message source chat |
| local | `"local"` | LOCAL | None (save to file) |
| home channel | `"telegram"` | Specified platform | Config home channel |
| explicit chat | `"telegram:123456"` | Specified platform | Specified chat_id |
| explicit thread | `"telegram:123456:thread"` | Specified platform | Specified chat_id + thread_id |

---

## 8. PairingStore — DM Authorization Code Approval

Source location: `gateway/pairing.py:1-310`

PairingStore implements a DM authorization code approval flow based on OWASP + NIST SP 800-63-4 guidance, replacing static user ID allowlists.

**Security Parameters** (Source location: `pairing.py:34-46`):

```python
ALPHABET = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"  # 32-char unambiguous alphabet (excludes 0/O/1/I)
CODE_LENGTH = 8                                # 8-character authorization code
CODE_TTL_SECONDS = 3600                        # 1 hour expiry
RATE_LIMIT_SECONDS = 600                       # 1 per user per 10 minutes
LOCKOUT_SECONDS = 3600                         # 1 hour lockout after 5 failures
MAX_PENDING_PER_PLATFORM = 3                   # max 3 pending approval codes per platform
MAX_FAILED_ATTEMPTS = 5                         # 5 failures trigger lockout
```

**Secure File Writing** (Source location: `pairing.py:50-73`): `_secure_write()` uses temp-file + atomic rename + `chmod 0600`, ensuring readers only ever see a complete file (old or new), never a partial write.

```python
def _secure_write(path: Path, data: str) -> None:
    """Write with restrictive permissions (owner read/write only).
    temp-file + atomic rename so readers always see complete file."""
    fd, tmp_path = tempfile.mkstemp(dir=str(path.parent), suffix=".tmp")
    with os.fdopen(fd, "w", encoding="utf-8") as f:
        f.write(data)
        f.flush()
        os.fsync(f.fileno())
    atomic_replace(tmp_path, path)
    os.chmod(path, 0o600)               # Windows: pass on OSError
```

**Storage Structure** (Source location: `pairing.py:76-100`): Three JSON files per platform: `{platform}-pending.json` (pending approval), `{platform}-approved.json` (authorized), `_rate_limits.json` (rate limit tracking). All operations are serialized through `threading.RLock` (multiple platform adapters running concurrently share one PairingStore).

### Comparison Table: Pairing Security Parameters Comparison

| Parameter | Value | Standard | Effect |
|------|------|------|------|
| Code alphabet | 32 chars (unambiguous) | OWASP | Prevent 0/O/1/I confusion |
| Code length | 8 chars | NIST SP 800-63-4 | ~2^40 combination space |
| Code TTL | 3600s (1h) | OWASP | Expired codes become invalid |
| Rate limit | 600s (10min) | OWASP | Prevent abusive requests |
| Lockout | 3600s after 5 fails | NIST | Prevent brute-force attacks |
| File permissions | chmod 0600 | OWASP | Owner read/write only |

---

## 9. HookRegistry — Event Hook System

Source location: `gateway/hooks.py:1-210`

HookRegistry provides a lightweight event-driven system that triggers user-defined handlers at key lifecycle points.

**Supported Events** (Source location: `hooks.py:9-18`):

```
gateway:startup     — Gateway process startup
session:start       — New session creation
session:end         — Session end
session:reset       — Session reset completed
agent:start         — Agent starts processing message
agent:step          — Each step of tool call loop
agent:end           — Agent completes processing
command:*           — Any slash command (wildcard match)
```

**Hook Directory Structure** (Source location: `hooks.py:64-140`): Each hook directory must contain `HOOK.yaml` (metadata: name, description, events) and `handler.py` (top-level `handle` function, sync or async).

```python
def discover_and_load(self) -> None:
    """Scan hooks directory for HOOK.yaml + handler.py pairs."""
    for hook_dir in sorted(HOOKS_DIR.iterdir()):
        manifest_path = hook_dir / "HOOK.yaml"
        handler_path = hook_dir / "handler.py"
        # ... load manifest, import handler module, register for events
        module_name = f"hermes_hook_{hook_name}"
        spec = importlib.util.spec_from_file_location(module_name, handler_path)
        module = importlib.util.module_from_spec(spec)
        sys.modules[module_name] = module          # Pydantic/dataclass forward-ref support
        spec.loader.exec_module(module)
        handle_fn = getattr(module, "handle", None)
        for event in events:
            self._handlers.setdefault(event, []).append(handle_fn)
```

**Wildcard Matching** (Source location: `hooks.py:145-156`): `command:*` matches all sub-events like `command:reset`, `command:status`, etc. Exact matches fire first; wildcard handlers are appended after.

**emit_collect Mode** (Source location: `hooks.py:183-210`): Returns a list of non-None return values from handlers, used for decision-based hooks (e.g., command allow/deny/rewrite policies).

### Comparison Table: Hook Event Trigger Comparison

| Event | Trigger Timing | Return Value | Common Usage |
|------|----------|--------|----------|
| `gateway:startup` | Process startup | None | Initialize external connections |
| `session:start` | First message | None | Set up session environment |
| `agent:start` | Start processing | None | Record start time |
| `agent:step` | Tool call | None | Progress tracking |
| `command:*` | Slash command | allow/deny/rewrite | Command policy interception |
| `session:reset` | Reset completed | None | Clean up session state |

---

## Summary Table

| Component | File | Lines | Core Responsibility |
|------|------|------|----------|
| GatewayRunner | `gateway/run.py` | 15046 | Gateway main entry point + agent cache + auto-continue |
| SessionStore | `gateway/session.py` | 1387 | Multi-platform session context + PII hashing + persistence |
| StreamConsumer | `gateway/stream_consumer.py` | 1018 | Sync delta→async platform message progressive edit bridging |
| GatewayConfig | `gateway/config.py` | 1636 | Platform enum (21 built-in + dynamic plugins) + configuration management |
| BasePlatformAdapter | `gateway/platforms/base.py` | 3390 | ABC + UTF-16 truncation + audio/network detection |
| Telegram Adapter | `gateway/platforms/telegram.py` | 3661 | Telegram Bot API adaptation |
| Discord Adapter | `gateway/platforms/discord.py` | 4900 | Discord Bot adaptation (largest adapter) |
| Signal Adapter | `gateway/platforms/signal.py` | 1516 | signal-cli REST adaptation |
| Matrix Adapter | `gateway/platforms/matrix.py` | 2676 | Matrix protocol adaptation |
| Slack Adapter | `gateway/platforms/slack.py` | 2926 | Slack Bolt adaptation |
| Feishu Adapter | `gateway/platforms/feishu.py` | 4831 | Feishu/Lark AI Card adaptation |
| DingTalk Adapter | `gateway/platforms/dingtalk.py` | 1366 | DingTalk AI Card adaptation |
| WeCom Adapter | `gateway/platforms/wecom.py` | 1609 | WeCom (Enterprise WeChat) callback adaptation |
| Weixin Adapter | `gateway/platforms/weixin.py` | 2110 | WeChat Official Account adaptation |
| Yuanbao/QQBot Adapter | `gateway/platforms/yuanbao.py` + proto/media/sticker | 7168 | Tencent Yuanbao + QQBot dual-platform adaptation (protocol + media + stickers) |
| WhatsApp Adapter | `gateway/platforms/whatsapp.py` | 1104 | WhatsApp Business API adaptation |
| API Server | `gateway/platforms/api_server.py` | 3172 | OpenAI-compatible HTTP endpoints |
| DeliveryRouter | `gateway/delivery.py` | 258 | Cron output routing to platforms/files |
| PairingStore | `gateway/pairing.py` | 310 | DM authorization code approval (OWASP/NIST) |
| HookRegistry | `gateway/hooks.py` | 210 | Event hooks + wildcard matching |
| GatewayStatus | `gateway/status.py` | 861 | Status monitoring + platform connection health reporting |
| ChannelDirectory | `gateway/channel_directory.py` | 357 | Channel directory management + search index |
| PlatformRegistry | `gateway/platform_registry.py` | 212 | Dynamic plugin platform registration + pseudo-member caching |
| DisplayConfig | `gateway/display_config.py` | 196 | TUI display layout configuration + footer templates |
| MirrorStore | `gateway/mirror.py` | 178 | Channel mirror synchronization + bidirectional forwarding |
| WhatsAppIdentity | `gateway/whatsapp_identity.py` | 155 | WhatsApp number identity resolution |
| SessionContext | `gateway/session_context.py` | 154 | ContextVars per-job session/delivery state |
| RuntimeFooter | `gateway/runtime_footer.py` | 150 | Gateway process info footer (PID/version/platform) |
| StickerCache | `gateway/sticker_cache.py` | 111 | Sticker file cache + LRU eviction |

---

[05 — Tool System](/en/chapters/05-tool-system) | [04 — LLM Provider System](/en/chapters/04-model-providers)