# 06 — Gateway 消息平台: 21+ 平台适配器 + 流式消费 + 会话管理 + API Server 的统一消息网关

[05 — 工具体系](/zh-CN/chapters/05-tool-system) | [04 — LLM 提供商体系](/zh-CN/chapters/04-model-providers)

---

## 源码文件

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `gateway/run.py` | 15046 | GatewayRunner 主入口与生命周期管理 |
| `gateway/session.py` | 1387 | SessionStore 会话上下文与持久化 |
| `gateway/stream_consumer.py` | 1018 | GatewayStreamConsumer 流式 delta→平台消息桥接 |
| `gateway/config.py` | 1636 | GatewayConfig 配置管理与 Platform enum |
| `gateway/platforms/base.py` | 3390 | BasePlatformAdapter ABC 与消息截断 |
| `gateway/platforms/telegram.py` | 3661 | Telegram Bot API 适配器 |
| `gateway/platforms/discord.py` | 4900 | Discord Bot 适配器 (最大) |
| `gateway/platforms/signal.py` | 1516 | Signal 适配器 |
| `gateway/platforms/matrix.py` | 2676 | Matrix 适配器 |
| `gateway/platforms/slack.py` | 2926 | Slack Bolt 适配器 |
| `gateway/platforms/email.py` | 748 | SMTP/IMAP 邮件适配器 |
| `gateway/platforms/feishu.py` | 4831 | 飞书/Lark 适配器 |
| `gateway/platforms/dingtalk.py` | 1366 | 钉钉适配器 |
| `gateway/platforms/wecom.py` | 1609 | 企业微信适配器 |
| `gateway/platforms/weixin.py` | 2110 | 微信公众号适配器 |
| `gateway/platforms/yuanbao.py` | 4756 | 腾讯元宝/QQBot 适配器 |
| `gateway/platforms/yuanbao_proto.py` | 1209 | 元宝/QQBot 协议层 (GrpcBridge + 解码) |
| `gateway/platforms/yuanbao_media.py` | 645 | 元宝/QQBot 媒体处理 (图片/文件/语音) |
| `gateway/platforms/yuanbao_sticker.py` | 558 | 元宝/QQBot 表情系统 (自定义表情/商店) |
| `gateway/platforms/whatsapp.py` | 1104 | WhatsApp 适配器 |
| `gateway/platforms/homeassistant.py` | 449 | Home Assistant 适配器 |
| `gateway/platforms/mattermost.py` | 832 | Mattermost 适配器 |
| `gateway/platforms/bluebubbles.py` | 937 | BlueBubbles (iMessage) 适配器 |
| `gateway/platforms/sms.py` | 377 | SMS 适配器 |
| `gateway/platforms/webhook.py` | 771 | Webhook 通用适配器 |
| `gateway/platforms/api_server.py` | 3172 | OpenAI-compatible API Server |
| `gateway/delivery.py` | 258 | DeliveryRouter cron 输出路由 |
| `gateway/pairing.py` | 310 | PairingStore DM 授权码审批 |
| `gateway/hooks.py` | 210 | HookRegistry 事件钩子系统 |
| `gateway/status.py` | 861 | GatewayStatus 状态监控与报告 |
| `gateway/channel_directory.py` | 357 | ChannelDirectory 频道目录管理 |
| `gateway/platform_registry.py` | 212 | PlatformRegistry 动态插件平台注册 |
| `gateway/display_config.py` | 196 | DisplayConfig TUI 显示布局配置 |
| `gateway/mirror.py` | 178 | MirrorStore 频道镜像同步 |
| `gateway/whatsapp_identity.py` | 155 | WhatsAppIdentity WA 号码身份解析 |
| `gateway/session_context.py` | 154 | SessionContext ContextVars per-job 会话状态 |
| `gateway/runtime_footer.py` | 150 | RuntimeFooter Gateway 进程信息脚注 |
| `gateway/sticker_cache.py` | 111 | StickerCache 表情文件缓存管理 |

## 一句话总结

Hermes Gateway 是一个 15000+ 行的消息平台网关，通过 BasePlatformAdapter ABC 统一 21+ 消息平台（Telegram/Discord/Signal/Matrix/Slack/飞书/钉钉/企微/微信/元宝/QQBot/WhatsApp/邮件/SMS/Webhook/HomeAssistant/Mattermost/BlueBubbles/iMessage/API Server），GatewayStreamConsumer 将同步 agent delta 流桥接为异步平台消息渐进编辑，SessionStore 管理多平台会话上下文与重置策略，API Server 提供与 OpenAI 兼容的 HTTP 端点，DeliveryRouter 路由 cron 输出，PairingStore 实现 DM 授权码审批，HookRegistry 提供事件钩子系统。

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

Hermes Gateway 以 15000+ 行的 GatewayRunner 为核心，管理 21+ 消息平台适配器的并发连接与生命周期。BasePlatformAdapter ABC（3390 行）定义 send/send_edit/truncate_message 等通用接口，包括 UTF-16 长度计算（Telegram 4096 code-unit 限制）和二分法截断。GatewayStreamConsumer（1018 行）将同步 agent stream delta 通过 queue.Queue 桥接到 asyncio，以 edit-transport 模式渐进编辑平台消息。SessionStore（1387 行）管理多平台会话上下文与 PII 散列。GatewayConfig（1636 行）的 Platform enum 支持 21 个内置成员 + 动态插件扩展。Yuanbao/QQBot 适配器（4756 行主模块 + 1209 行协议层 + 645 行媒体 + 558 行表情）覆盖腾讯元宝和 QQ 机器人双平台。WhatsApp 适配器（1104 行）覆盖 WhatsApp Business API。API Server（3172 行）提供 OpenAI-compatible `/v1/chat/completions` 和 `/v1/responses` 端点。GatewayStatus（861 行）提供状态监控与报告。PairingStore（310 行）实现 OWASP/NIST 指导的 DM 授权码审批流程。HookRegistry（210 行）支持 wildcard 事件匹配。

---

## 1. GatewayRunner — 网关主入口

源码位置: `gateway/run.py:1-15046`

GatewayRunner 是网关的 15000+ 行主模块，管理所有平台适配器的并发连接、agent 缓存、auto-continue、定时任务和生命周期。

**Agent 缓存容量** (源码位置: `run.py:50-53`):

```python
_AGENT_CACHE_MAX_SIZE = 128              # per-session AIAgent cache
_AGENT_CACHE_IDLE_TTL_SECS = 3600.0      # evict agents idle for >1h
_PLATFORM_CONNECT_TIMEOUT_SECS_DEFAULT = 30.0
```

**Auto-continue 新鲜度窗口** (源码位置: `run.py:88-148`): 中断的 gateway turn 仅在新鲜度窗口内自动继续。默认 1 小时，覆盖 `agent.gateway_timeout`（30 分钟）+ 运行时余量。

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

**Telegram 命令名规范化** (源码位置: `run.py:56-73`): Telegram Bot API 命令名仅允许小写字母、数字和下划线。`_telegramize_command_mentions()` 将斜杠命令引用转换为 Telegram-valid 格式。

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

**时间戳强制转换** (源码位置: `run.py:97-128`): `_coerce_gateway_timestamp()` 接受 datetime、epoch seconds/milliseconds、ISO-8601 字符串（含/不含 `Z`），覆盖所有平台的时间戳格式差异。

### 比较表: GatewayRunner 核心参数

| 参数 | 默认值 | 环境变量 | 作用 |
|------|--------|----------|------|
| Agent cache size | 128 | — | per-session AIAgent 最大缓存数 |
| Agent idle TTL | 3600s | — | 空闲 agent 驱逐时间 |
| Platform connect timeout | 30s | — | 平台适配器连接超时 |
| Auto-continue freshness | 3600s | `HERMES_AUTO_CONTINUE_FRESHNESS` | 中断 turn 续行窗口 |
| Agent gateway timeout | 1800s | `HERMES_AGENT_TIMEOUT` | 单 turn 最大运行时间 |

---

## 2. BasePlatformAdapter — 平台适配器 ABC

源码位置: `gateway/platforms/base.py:1-3390`

BasePlatformAdapter 定义所有平台适配器的通用接口与辅助函数，是 19+ 适配器的共同基类。

**消息截断 — UTF-16 精确计算** (源码位置: `base.py:66-116`): Telegram 的消息长度限制（4096）以 UTF-16 code unit 计量，不是 Unicode code point。emoji 和 CJK Extension B 字符消耗两个 UTF-16 unit。

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

**音频发送判断** (源码位置: `base.py:29-63`): 不同平台对音频格式支持不同。Telegram Bot API 仅接受 MP3/M4A 的 `sendAudio` 和 Opus/OGG 的 `sendVoice`，其他音频走 document 交付。

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
            return is_voice           # Opus/OGG 仅作为 voice bubble
        return normalized_ext in _TELEGRAM_AUDIO_ATTACHMENT_EXTS
    return True                       # 其他平台: 所有音频走 audio sender
```

**网络可达性检测** (源码位置: `base.py:119-151`): 检测绑定地址是否暴露服务器到 loopback 外。处理 IPv4-mapped IPv6 (`::ffff:127.0.0.1`) 和主机名 DNS 解析。

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

**macOS 系统代理检测** (源码位置: `base.py:154-187`): 通过 `scutil --proxy` 读取 macOS 系统级 HTTP/HTTPS 代理设置。

### 比较表: 平台适配器规模对比

| 平台 | 文件 | 行数 | 认证方式 | 消息模型 |
|------|------|------|----------|----------|
| Discord | `discord.py` | 4900 | Bot token | edit-based streaming |
| 飞书 | `feishu.py` | 4831 | App ID + Secret | AI Card + edit |
| 元宝/QQBot | `yuanbao.py` + proto/media/sticker | 7168 (4756+1209+645+558) | OAuth token | edit-based streaming |
| WhatsApp | `whatsapp.py` | 1104 | WhatsApp Business API token | edit-based streaming |
| Telegram | `telegram.py` | 3661 | Bot token | edit-based streaming |
| Base | `base.py` | 3390 | ABC | truncate + UTF-16 |
| API Server | `api_server.py` | 3172 | Bearer/API key | SSE streaming |
| Slack | `slack.py` | 2926 | Bot token + Socket Mode | edit-based streaming |
| Matrix | `matrix.py` | 2676 | Access token | edit-based streaming |
| 微信 | `weixin.py` | 2110 | AppID + AppSecret | 5s timeout web reply |
| 企微 | `wecom.py` | 1609 | Corp ID + Secret | callback + push |
| Signal | `signal.py` | 1516 | signal-cli REST | edit-based |
| 钉钉 | `dingtalk.py` | 1366 | AppKey + AppSecret | AI Card |
| BlueBubbles | `bluebubbles.py` | 937 | API key + password | REST API |
| Mattermost | `mattermost.py` | 832 | Personal access token | edit-based |
| Email | `email.py` | 748 | SMTP/IMAP creds | plain text |
| Webhook | `webhook.py` | 771 | Secret HMAC | HTTP POST |

---

## 3. GatewayStreamConsumer — 流式桥接

源码位置: `gateway/stream_consumer.py:1-1018`

GatewayStreamConsumer 是 agent 同步 delta 流到平台异步消息编辑的关键桥梁。设计基于 edit-transport 模式: 先发送初始消息，然后逐步 editMessageText。

**核心架构** (源码位置: `stream_consumer.py:15-69`):

```
同步路径 (agent worker thread):
  stream_delta_callback(text) → consumer.on_delta()
                                      │
                                      ▼
                              queue.Queue (thread-safe)
                                      │
                                      ▼
异步路径 (asyncio task):          consumer.run()
  1. 从 queue 取 delta
  2. buffer + rate-limit
  3. 渐进编辑平台消息
  4. finalize 最终编辑
```

**StreamConsumerConfig** (源码位置: `stream_consumer.py:41-54`):

```python
@dataclass
class StreamConsumerConfig:
    edit_interval: float = 1.0             # 编辑间隔（秒）
    buffer_threshold: int = 40             # 缓冲字符阈值
    cursor: str = " ▉"                    # 流式光标
    buffer_only: bool = False              # 仅缓冲不发送
    fresh_final_after_seconds: float = 0.0 # 长运行响应重发为新消息的时间阈值
```

**Flood control 保护** (源码位置: `stream_consumer.py:72-74,123-133`): 连续 3 次洪水控制失败后永久禁用渐进编辑，退化为完整响应一次性发送。

```python
_MAX_FLOOD_STRIKES = 3

self._flood_strikes = 0
self._current_edit_interval = self.cfg.edit_interval  # Adaptive backoff
self._adapter_requires_finalize: bool = (
    getattr(adapter, "REQUIRES_EDIT_FINALIZE", False) is True
)                                          # 钉钉 AI Card 需显式 finalize
```

**思考块过滤** (源码位置: `stream_consumer.py:79-87`): 镜像 CLI 的 `_stream_delta` tag 抑制逻辑，过滤 `REASONING_SCRATCHPAD`, `thinking`, `reasoning`, `thought` 等标签。

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

**Fresh-final 逻辑** (源码位置: `stream_consumer.py:51-54,114-116`): 长运行推理模型（如 Opus 4.7）流式速度慢。当原始预览消息存活超过 `fresh_final_after_seconds` 时，最终编辑改为发送一条全新消息，使平台时间戳反映完成时间而非首 token 时间。

### 比较表: StreamConsumer 流式策略对比

| 策略 | 适用平台 | 用户体验 | 实现复杂度 |
|------|----------|----------|------------|
| edit-based streaming | Telegram, Discord, Slack | 实时更新同一消息 | 高（flood control, think block filter） |
| fresh-final resend | 长运行推理场景 | 完成时间戳更准确 | 中 |
| buffer-only | 配置/调试场景 | 一次性发送完整响应 | 低 |
| AI Card finalize | 钉钉 | 结构化卡片呈现 | 中（需显式 finalize call） |

---

## 4. SessionStore — 会话上下文与持久化

源码位置: `gateway/session.py:1-1387`

SessionStore 管理多平台会话上下文追踪、持久化和重置策略。

**SessionSource 数据结构** (源码位置: `session.py:70-150`):

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

**PII 散列** (源码位置: `session.py:34-54`): 所有用户 ID 和 chat ID 在日志中以确定性 SHA-256[:12] 散列呈现，保留平台前缀（`telegram:12345` → `telegram:<hash>`）。

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

**SessionSource.description** (源码位置: `session.py:96-114`): 人类可读的来源描述，用于系统 prompt 注入。

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

### 比较表: SessionSource 字段用途对比

| 字段 | 用途 | 平台特化 | 示例 |
|------|------|----------|------|
| `chat_type` | 系统 prompt 注入 | 所有 | "dm", "group", "channel", "thread" |
| `thread_id` | 消息路由 | Discord/Slack/Matrix | "thread_123" |
| `user_id_alt` | 稳定身份 | Signal (UUID), Feishu (union_id) | "signal-uuid" |
| `guild_id` | workspace scope | Discord/Slack | "guild_456" |
| `message_id` | pin/reply/react | Telegram/Discord | "msg_789" |
| `is_bot` | 过滤 bot 消息 | Discord | True |

---

## 5. GatewayConfig — 配置管理

源码位置: `gateway/config.py:1-1636`

GatewayConfig 管理平台连接配置、home channels、会话重置策略和交付偏好。

**Platform Enum — 动态扩展** (源码位置: `config.py:82-150`): 21 个内置成员 + `_missing_()` 动态创建插件平台 pseudo-member。

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

**配置值 coerce** (源码位置: `config.py:25-66`): `_coerce_bool()`, `_coerce_float()`, `_coerce_int()` 安全转换配置值，防止格式错误崩溃。

**未授权 DM 行为** (源码位置: `config.py:59-74`): `_normalize_unauthorized_dm_behavior()` 限制为 "pair"（发起授权码审批）或 "ignore"（静默忽略）。

```python
def _normalize_unauthorized_dm_behavior(value, default="pair") -> str:
    if isinstance(value, str):
        normalized = value.strip().lower()
        if normalized in {"pair", "ignore"}:
            return normalized
    return default
```

### 比较表: Platform Enum 扩展机制对比

| 扩展方式 | 触发条件 | 生命周期 | 身份稳定性 |
|----------|----------|----------|------------|
| 内置成员 | 模块导入时 | 永久 | `Platform.TELEGRAM is Platform.TELEGRAM` |
| 文件系统扫描 pseudo-member | `_missing_()` 首次遇到插件名 | 永久（缓存在 `_value2member_map_`） | `Platform("irc") is Platform("irc")` |
| 运行时注册 pseudo-member | `platform_registry.is_registered()` | 进程生命周期 | 同上 |

---

## 6. API Server — OpenAI-compatible 端点

源码位置: `gateway/platforms/api_server.py:1-3172`

API Server 提供与 OpenAI 兼容的 HTTP 端点，允许任何 OpenAI 兼容前端（Open WebUI, LobeChat, LibreChat, AnythingLLM, NextChat, ChatBox 等）通过 `http://localhost:8642/v1` 连接 Hermes。

**端点列表** (源码位置: `api_server.py:5-16`):

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

**默认设置** (源码位置: `api_server.py:56-63`):

```python
DEFAULT_HOST = "127.0.0.1"
DEFAULT_PORT = 8642
MAX_STORED_RESPONSES = 100
MAX_REQUEST_BYTES = 10_000_000            # 10 MB
MAX_NORMALIZED_TEXT_LENGTH = 65_536       # 64 KB cap
CHAT_COMPLETIONS_SSE_KEEPALIVE_SECONDS = 30.0
```

**内容规范化** (源码位置: `api_server.py:73-128`): `_normalize_chat_content()` 将 OpenAI Chat 格式的多模态内容数组扁平化为纯文本字符串。防御性限制: 递归深度上限 10、列表大小上限 1000、输出长度上限 64KB。

```python
_TEXT_PART_TYPES = frozenset({"text", "input_text", "output_text"})
_IMAGE_PART_TYPES = frozenset({"image_url", "input_image"})
_FILE_PART_TYPES = frozenset({"file", "input_file"})     # unsupported → ValueError
```

**会话持续性** (源码位置: `api_server.py:5`): `X-Hermes-Session-Id` header 提供 opt-in 会话连续性（chat completions 模式），`X-Hermes-Session-Key` header 提供长期内存 scope。

### 比较表: API Server 端点对比

| 端点 | 方法 | 状态 | 流式支持 | 用途 |
|------|------|------|----------|------|
| `/v1/chat/completions` | POST | 无状态（opt-in session） | SSE | OpenAI 兼容前端 |
| `/v1/responses` | POST | 有状态（previous_response_id） | SSE | OpenAI Responses API |
| `/v1/models` | GET | 无状态 | 无 | 模型列表 |
| `/v1/runs` | POST | 异步 | 无 | 异步运行（返回 run_id） |
| `/v1/runs/{id}/events` | GET | SSE | SSE | 生命周期事件流 |
| `/health/detailed` | GET | 无状态 | 无 | 跨容器 dashboard |

---

## 7. DeliveryRouter — Cron 输出路由

源码位置: `gateway/delivery.py:1-258`

DeliveryRouter 路由 cron 输出和代理响应到正确的目标平台。

**DeliveryTarget 解析** (源码位置: `delivery.py:29-107`):

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

**输出截断保护** (源码位置: `delivery.py:21-23,242-249`): `MAX_PLATFORM_OUTPUT = 4000` 字符，超出部分截断并保存到磁盘文件，附带文件路径引用。

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

### 比较表: Delivery 路由类型对比

| 路由类型 | 格式 | 平台选择 | chat_id 选择 |
|----------|------|----------|-------------|
| origin | `"origin"` | 消息来源平台 | 消息来源 chat |
| local | `"local"` | LOCAL | 无（保存到文件） |
| home channel | `"telegram"` | 指定平台 | config home channel |
| explicit chat | `"telegram:123456"` | 指定平台 | 指定 chat_id |
| explicit thread | `"telegram:123456:thread"` | 指定平台 | 指定 chat_id + thread_id |

---

## 8. PairingStore — DM 授权码审批

源码位置: `gateway/pairing.py:1-310`

PairingStore 实现基于 OWASP + NIST SP 800-63-4 指导的 DM 授权码审批流程，替代静态用户 ID allowlist。

**安全参数** (源码位置: `pairing.py:34-46`):

```python
ALPHABET = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"  # 32-char 无歧义字母（排除 0/O/1/I）
CODE_LENGTH = 8                                # 8 字符授权码
CODE_TTL_SECONDS = 3600                        # 1 小时过期
RATE_LIMIT_SECONDS = 600                       # 1 次/用户/10 分钟
LOCKOUT_SECONDS = 3600                         # 5 次失败后锁定 1 小时
MAX_PENDING_PER_PLATFORM = 3                   # 每平台最多 3 个待审批码
MAX_FAILED_ATTEMPTS = 5                         # 5 次失败触发锁定
```

**安全文件写入** (源码位置: `pairing.py:50-73`): `_secure_write()` 使用 temp-file + atomic rename + `chmod 0600`，确保读者只看到完整文件（旧或新），不会读到部分写入。

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

**存储结构** (源码位置: `pairing.py:76-100`): 每平台三个 JSON 文件: `{platform}-pending.json` (待审批), `{platform}-approved.json` (已授权), `_rate_limits.json` (限速追踪)。所有操作通过 `threading.RLock` 串行化（多平台适配器并发运行共享一个 PairingStore）。

### 比较表: Pairing 安全参数对比

| 参数 | 值 | 标准 | 作用 |
|------|------|------|------|
| Code alphabet | 32 chars (无歧义) | OWASP | 防止 0/O/1/I 混淆 |
| Code length | 8 chars | NIST SP 800-63-4 | ~2^40 组合空间 |
| Code TTL | 3600s (1h) | OWASP | 过期失效 |
| Rate limit | 600s (10min) | OWASP | 防止滥用请求 |
| Lockout | 3600s after 5 fails | NIST | 防暴力破解 |
| File permissions | chmod 0600 | OWASP | 仅 owner 可读写 |

---

## 9. HookRegistry — 事件钩子系统

源码位置: `gateway/hooks.py:1-210`

HookRegistry 提供轻量级事件驱动系统，在关键生命周期点触发用户自定义处理器。

**支持的事件** (源码位置: `hooks.py:9-18`):

```
gateway:startup     — Gateway 进程启动
session:start       — 新会话创建
session:end         — 会话结束
session:reset       — 会话重置完成
agent:start         — Agent 开始处理消息
agent:step          — 工具调用循环每一步
agent:end           — Agent 完成处理
command:*           — 任何斜杠命令（wildcard 匹配）
```

**Hook 目录结构** (源码位置: `hooks.py:64-140`): 每个 hook 目录必须包含 `HOOK.yaml`（元数据: name, description, events）和 `handler.py`（顶层 `handle` 函数，sync 或 async）。

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

**Wildcard 匹配** (源码位置: `hooks.py:145-156`): `command:*` 匹配所有 `command:reset`, `command:status` 等子事件。精确匹配先触发，wildcard 后追加。

**emit_collect 模式** (源码位置: `hooks.py:183-210`): 返回处理器非 None 返回值列表，用于决策式钩子（如命令 allow/deny/rewrite 策略）。

### 比较表: Hook 事件触发对比

| 事件 | 触发时机 | 返回值 | 常见用途 |
|------|----------|--------|----------|
| `gateway:startup` | 进程启动 | 无 | 初始化外部连接 |
| `session:start` | 首条消息 | 无 | 设置会话环境 |
| `agent:start` | 开始处理 | 无 | 记录开始时间 |
| `agent:step` | 工具调用 | 无 | 进度追踪 |
| `command:*` | 斜杠命令 | allow/deny/rewrite | 命令策略拦截 |
| `session:reset` | 重置完成 | 无 | 清理会话状态 |

---

## Summary Table

| 组件 | 文件 | 行数 | 核心职责 |
|------|------|------|----------|
| GatewayRunner | `gateway/run.py` | 15046 | 网关主入口 + agent 缓存 + auto-continue |
| SessionStore | `gateway/session.py` | 1387 | 多平台会话上下文 + PII 散列 + 持久化 |
| StreamConsumer | `gateway/stream_consumer.py` | 1018 | 同步 delta→异步平台消息渐进编辑桥接 |
| GatewayConfig | `gateway/config.py` | 1636 | Platform enum (21 内置 + 动态插件) + 配置管理 |
| BasePlatformAdapter | `gateway/platforms/base.py` | 3390 | ABC + UTF-16 截断 + 音频/网络检测 |
| Telegram Adapter | `gateway/platforms/telegram.py` | 3661 | Telegram Bot API 适配 |
| Discord Adapter | `gateway/platforms/discord.py` | 4900 | Discord Bot 适配（最大适配器） |
| Signal Adapter | `gateway/platforms/signal.py` | 1516 | signal-cli REST 适配 |
| Matrix Adapter | `gateway/platforms/matrix.py` | 2676 | Matrix 协议适配 |
| Slack Adapter | `gateway/platforms/slack.py` | 2926 | Slack Bolt 适配 |
| Feishu Adapter | `gateway/platforms/feishu.py` | 4831 | 飞书/Lark AI Card 适配 |
| DingTalk Adapter | `gateway/platforms/dingtalk.py` | 1366 | 钉钉 AI Card 适配 |
| WeCom Adapter | `gateway/platforms/wecom.py` | 1609 | 企业微信回调适配 |
| Weixin Adapter | `gateway/platforms/weixin.py` | 2110 | 微信公众号适配 |
| Yuanbao/QQBot Adapter | `gateway/platforms/yuanbao.py` + proto/media/sticker | 7168 | 腾讯元宝+QQBot 双平台适配 (协议+媒体+表情) |
| WhatsApp Adapter | `gateway/platforms/whatsapp.py` | 1104 | WhatsApp Business API 适配 |
| API Server | `gateway/platforms/api_server.py` | 3172 | OpenAI-compatible HTTP 端点 |
| DeliveryRouter | `gateway/delivery.py` | 258 | Cron 输出路由到平台/文件 |
| PairingStore | `gateway/pairing.py` | 310 | DM 授权码审批 (OWASP/NIST) |
| HookRegistry | `gateway/hooks.py` | 210 | 事件钩子 + wildcard 匹配 |
| GatewayStatus | `gateway/status.py` | 861 | 状态监控 + 平台连接健康报告 |
| ChannelDirectory | `gateway/channel_directory.py` | 357 | 频道目录管理 + 搜索索引 |
| PlatformRegistry | `gateway/platform_registry.py` | 212 | 动态插件平台注册 + 伪成员缓存 |
| DisplayConfig | `gateway/display_config.py` | 196 | TUI 显示布局配置 + 脚注模板 |
| MirrorStore | `gateway/mirror.py` | 178 | 频道镜像同步 + 双向转发 |
| WhatsAppIdentity | `gateway/whatsapp_identity.py` | 155 | WhatsApp 号码身份解析 |
| SessionContext | `gateway/session_context.py` | 154 | ContextVars per-job 会话/交付状态 |
| RuntimeFooter | `gateway/runtime_footer.py` | 150 | Gateway 进程信息脚注 (PID/版本/平台) |
| StickerCache | `gateway/sticker_cache.py` | 111 | 表情文件缓存 + LRU 淘汰 |

---

[05 — 工具体系](/zh-CN/chapters/05-tool-system) | [04 — LLM 提供商体系](/zh-CN/chapters/04-model-providers)