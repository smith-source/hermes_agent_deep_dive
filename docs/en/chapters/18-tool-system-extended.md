# 18 — Tool System Extended: Cross-Platform Messaging / Web Search / Voice / Vision / File / Smart Home — Eight Core Tools

[← 05 — Tool System](/en/chapters/05-tool-system) | [19 — Hermes CLI Subsystem →](/en/chapters/19-hermes-cli-subsystem)

---

## Source Files

### Eight Core Tools

| File | Lines | Core Responsibility |
|------|------|----------|
| `tools/send_message_tool.py` | 1780 | Cross-platform message routing — 15+ platform adapters, channel resolution, media attachments, message chunking |
| `tools/web_tools.py` | 2223 | Web search + content extraction — 5 backends (Exa/Firecrawl/Parallel/Tavily/SearXNG), LLM summarization |
| `tools/tts_tool.py` | 2185 | Text-to-speech — 11+ providers (Edge/ElevenLabs/OpenAI/MiniMax/Mistral/Gemini/xAI/NeuTTS/KittenTTS/Piper/Command), streaming playback |
| `tools/vision_tools.py` | 1168 | Image/video analysis — multi-provider vision routing, Base64 encoding, SSRF protection, auto-resizing |
| `tools/file_tools.py` | 1143 | File operations — read/write/patch/search/dedup loop detection, path safety guards |
| `tools/transcription_tools.py` | 911 | Audio transcription — 6 providers (local Whisper/Groq/OpenAI/Mistral/xAI/local command), language detection |
| `tools/session_search_tool.py` | 605 | Session search — FTS5 full-text retrieval, LLM summarization, parallel concurrency control |
| `tools/homeassistant_tool.py` | 513 | Smart home control — HA REST API, entity discovery, service domain safety filtering |

### Auxiliary Tools

| File | Lines | Core Responsibility |
|------|------|----------|
| `tools/code_execution_tool.py` | 1621 | Code execution sandbox — UDS RPC + file RPC dual transport, hermes_tools.py stub generation |
| `tools/image_generation_tool.py` | 1002 | Image generation — FAL.ai multi-model catalog, Clarity Upscaler post-processing |
| `tools/voice_mode.py` | 1017 | Voice mode — push recording (sounddevice/Termux), TTS playback, hallucination filtering |
| `tools/kanban_tools.py` | 855 | Kanban collaboration — show/complete/block/heartbeat/comment/create/link 7 tools |
| `tools/mixture_of_agents_tool.py` | 541 | Mixture of agents — multi-frontier-model parallel reasoning + aggregation synthesis |
| `tools/memory_tool.py` | 586 | Persistent memory — MEMORY.md/USER.md dual storage, frozen snapshots, injection scanning |

**Total: 14 files, 16,150 lines**

## One-Line Summary

The eight extended tools cover Hermes's cross-platform message delivery (15+ platforms), web search and extraction (5 backends), speech synthesis (11+ providers) and transcription (6 providers), vision analysis, file operations (multi-layer safety guards), session recall (FTS5 + LLM summarization), and smart home control (HA REST API), supplemented by six auxiliary tools: code sandbox, image generation, voice mode, kanban collaboration, mixture-of-agents reasoning, and persistent memory.

---

## Architecture Overview

```
                          ┌──────────────────────────────────────────┐
                          │           AIAgent Tool-Calling Loop       │
                          └─────────────────┬────────────────────────┘
                                            │
              ┌─────────────────────────────┼──────────────────────────────┐
              │                             │                              │
    ┌─────────▼─────────┐       ┌──────────▼──────────┐       ┌──────────▼──────────┐
    │   send_message    │       │     web_tools       │       │   tts_tool          │
    │  ┌─────────────┐  │       │  ┌───────────────┐  │       │  ┌───────────────┐  │
    │  │ target parse│  │       │  │ backend select│  │       │  │ provider select│ │
    │  │ platform    │  │       │  │ Exa/FC/Par/   │  │       │  │ Edge/11L/OAI/  │ │
    │  │  adapters   │  │       │  │ Tavily/SXNG   │  │       │  │ MM/Mist/Gem/   │ │
    │  │ media split │  │       │  └───────┬───────┘  │       │  │ xAI/Piper/Cmd  │ │
    │  │ msg chunk   │  │       │          │          │       │  └───────┬───────┘  │
    │  │ mirror back │  │       │  ┌───────▼───────┐  │       │          │          │
    │  └─────────────┘  │       │  │ LLM summarize │  │       │  ┌───────▼───────┐  │
    └───────────────────┘       │  │ (Gemini Flash)│  │       │  │ Opus/MP3 out  │  │
                                │  └───────────────┘  │       │  │ stream to spk │  │
                                └────────────────────┘       │  └───────────────┘  │
                                                             └────────────────────┘

    ┌───────────────────┐       ┌────────────────────┐       ┌────────────────────┐
    │  vision_tools     │       │    file_tools      │       │ transcription_tools│
    │ ┌───────────────┐ │       │ ┌────────────────┐ │       │ ┌────────────────┐ │
    │ │ download+base64│ │       │ │ path resolve   │ │       │ │ provider select│ │
    │ │ SSRF guard    │ │       │ │ device guard   │ │       │ │ local Whisper  │ │
    │ │ auto-resize   │ │       │ │ /etc/ /boot/   │ │       │ │ Groq/OpenAI/   │ │
    │ │ aux vision LLM│ │       │ │ dedup + loop   │ │       │ │ Mistral/xAI    │ │
    │ └───────────────┘ │       │ │ read cap 100K  │ │       │ │ cmd template   │ │
    └───────────────────┘       │ └────────────────┘ │       │ └────────────────┘ │
                                └────────────────────┘       └────────────────────┘

    ┌───────────────────┐       ┌────────────────────┐       ┌────────────────────┐
    │ session_search    │       │ homeassistant_tool │       │  Auxiliary Tools   │
    │ ┌───────────────┐ │       │ ┌────────────────┐ │       │ ┌────────────────┐ │
    │ │ FTS5 query    │ │       │ │ HA REST API    │ │       │ │ code_execution │ │
    │ │ group+dedup   │ │       │ │ entity filter  │ │       │ │ image_generate │ │
    │ │ truncate match│ │       │ │ service call   │ │       │ │ voice_mode     │ │
    │ │ LLM summarize │ │       │ │ domain block   │ │       │ │ kanban_tools   │ │
    │ └───────────────┘ │       │ └────────────────┘ │       │ │ mixture_agents │ │
    └───────────────────┘       └────────────────────┘       │ │ memory_tool    │ │
                                                             │ └────────────────┘ │
                                                             └────────────────────┘
```

---

## TL;DR

Hermes's extended tool system provides eight vertical capability modules on top of the core Registry/Delegate/Browser/MCP foundation. `send_message_tool` is the sole router spanning 15+ messaging platforms, achieving unified addressing through regex-based target string parsing and channel directory resolution. `web_tools` supports 5 search backends and LLM content summarization, using `_FirecrawlProxy` for lazy loading to avoid a 200ms import overhead. `tts_tool` covers 11+ speech providers, from free Edge TTS to local Piper, supporting Opus/MP3 output and streaming playback. `vision_tools` unifies calls through an auxiliary vision router, with built-in SSRF redirect protection and auto-resize retry. `file_tools` implements a three-layer safety guard (device paths / sensitive paths / binary extensions) and dedup loop detection. `transcription_tools` defaults to local faster-whisper, supporting 6 STT providers. `session_search_tool` implements efficient session recall based on FTS5 + LLM summarization. `homeassistant_tool` controls smart home devices via the HA REST API, with a built-in blacklist of 5 dangerous service domains.

---

## 1. send_message_tool — Cross-Platform Message Routing

**Source location:** `tools/send_message_tool.py` (1780 lines)

### 1.1 Design Goal

Deliver messages from the Hermes core to any connected messaging platform — Telegram, Discord, Slack, Signal, WhatsApp, WeChat, Matrix, Feishu, Yuanbao, DingTalk, WeCom, Mattermost, BlueBubbles, QQBot, Email, SMS, and all plugin platforms. The tool receives targets in the unified `platform:target_ref` format and performs platform adaptation internally.

### 1.2 Core Data Flow

```
User/model calls send_message(action='send', target='telegram:-100123:42', message='...')
         │
         ▼
   ┌─ _parse_target_ref() ──────────────────────────────────────────┐
   │  Dispatch by platform_name using regex:                         │
   │  telegram  → _TELEGRAM_TOPIC_TARGET_RE  (-?\d+)(?::(\d+))?    │
   │  discord   → _NUMERIC_TOPIC_RE          (same as telegram)     │
   │  slack     → _SLACK_TARGET_RE           [CGD][A-Z0-9]{8,}     │
   │  feishu   → _FEISHU_TARGET_RE           (oc|ou|on|chat|open)_ │
   │  weixin    → _WEIXIN_TARGET_RE          (wxid|gh|...@chatroom)│
   │  signal/sms/whatsapp → _E164_TARGET_RE  \+(\d{7,15})         │
   │  yuanbao  → _YUANBAO_TARGET_RE          (group|direct):...    │
   │  matrix   → !roomid / @userid check                             │
   └─────────────────────────────────────────────────────────────────┘
         │
         ▼
   resolve_channel_name() → fuzzy name resolution to numeric ID
         │
         ▼
   _send_to_platform() → message chunking → platform-specific _send_*() functions
```

### 1.3 Key Implementation: Target Parsing

Source location: `send_message_tool.py:311-351`

```python
def _parse_target_ref(platform_name: str, target_ref: str):
    """Parse a tool target into chat_id/thread_id and whether it is explicit."""
    if platform_name == "telegram":
        match = _TELEGRAM_TOPIC_TARGET_RE.fullmatch(target_ref)
        if match:
            return match.group(1), match.group(2), True
    if platform_name == "discord":
        match = _NUMERIC_TOPIC_RE.fullmatch(target_ref)
        if match:
            return match.group(1), match.group(2), True
    if platform_name == "slack":
        match = _SLACK_TARGET_RE.fullmatch(target_ref)
        if match:
            return match.group(1), None, True
    # ... more platforms
    if platform_name in _PHONE_PLATFORMS:
        match = _E164_TARGET_RE.fullmatch(target_ref)
        if match:
            return target_ref.strip(), None, True  # preserve + sign — E.164 format
    # ...
    return None, None, False
```

Slack's `U...` (user ID) and `W...` (workspace ID) are intentionally excluded — because `chat.postMessage` only accepts `C/G/D` conversation IDs; sending a user ID directly would silently fail.

### 1.4 Message Chunking and Platform Adaptation

Source location: `send_message_tool.py:440-667`

`_send_to_platform()` is the core routing function. It automatically chunks messages based on each platform's length limit:

```python
_MAX_LENGTHS = {
    Platform.TELEGRAM: TelegramAdapter.MAX_MESSAGE_LENGTH,  # 4096 UTF-16 code units
    Platform.DISCORD: DiscordAdapter.MAX_MESSAGE_LENGTH,     # 2000 chars
    Platform.SLACK: SlackAdapter.MAX_MESSAGE_LENGTH,         # 40000 chars
}
# ...
chunks = BasePlatformAdapter.truncate_message(message, max_len, len_fn=_len_fn)
```

Telegram uses `utf16_len()` instead of character counting — because the Telegram API measures length in UTF-16 code units.

### 1.5 Media Attachment Handling

Source location: `send_message_tool.py:22-49`

`MEDIA:<path>` markers in messages are extracted by `BasePlatformAdapter.extract_media()`, then routed per platform:

- **Telegram**: dispatched by extension to `send_photo/send_video/send_voice/send_audio/send_document`
- **Discord**: multipart/form-data upload; forum channels (type 15) create a thread + attachment
- **Matrix/Signal/Yuanbao/Feishu/WeChat**: each has native media adapters
- **Platforms without media support**: returns a warning but text is still sent

### 1.6 Telegram Retry Strategy

Source location: `send_message_tool.py:73-113`

```python
def _telegram_retry_delay(exc: Exception, attempt: int) -> float | None:
    retry_after = getattr(exc, "retry_after", None)
    if retry_after is not None:
        return max(float(retry_after), 0.0)
    # 429/502/503/504 → exponential backoff
    if "429" in text or "too many requests" in text:
        return float(2 ** attempt)
```

### 1.7 Cron Deduplication

Source location: `send_message_tool.py:388-416`

When the Cron scheduler has already set up auto-delivery to a target, `send_message` detects the duplicate target and skips it, preventing the same message from being sent twice:

```python
def _maybe_skip_cron_duplicate_send(platform_name, chat_id, thread_id):
    auto_target = _get_cron_auto_delivery_target()
    if auto_target and same_target:
        return {"success": True, "skipped": True, "reason": "cron_auto_delivery_duplicate_target"}
```

### 1.8 Error Sanitization

Source location: `send_message_tool.py:60-65`

All error text undergoes dual-layer sanitization before being returned to the model — first `redact_sensitive_text()` for desensitization, then regex erasure of tokens in URL query parameters and assignment statements:

```python
def _sanitize_error_text(text) -> str:
    redacted = redact_sensitive_text(text)
    redacted = _URL_SECRET_QUERY_RE.sub(lambda m: f"{m.group(1)}***", redacted)
    redacted = _GENERIC_SECRET_ASSIGN_RE.sub(lambda m: f"{m.group(1)}=***", redacted)
    return redacted
```

### 1.9 Discord Forum Channels

Source location: `send_message_tool.py:823-1017`

Discord forum channels (type 15) reject `POST /messages`. `_send_discord()` implements three-layer detection:

1. **Channel directory cache** → `lookup_channel_type("discord", chat_id)`
2. **Process-local probe cache** → `_probe_is_forum_cached(chat_id)`
3. **Live GET /channels/{id} probe** → result memoized

Forum channels automatically create thread posts, with the thread name derived from the first line of the message (capped at 100 characters).

---

## 2. web_tools — Web Search + Content Extraction

**Source location:** `tools/web_tools.py` (2223 lines)

### 2.1 Design Goal

Provide a unified web search and content extraction interface, supporting 5 backend auto-selection, and compressing large page content through LLM summarization to save tokens.

### 2.2 Backend Selection Architecture

Source location: `web_tools.py:121-184`

```
Configuration priority:
  1. config.yaml → web.search_backend / web.extract_backend (per-capability override)
  2. config.yaml → web.backend (shared fallback)
  3. Environment variable auto-detection → firecrawl > parallel > tavily > exa > searxng
```

Key: Search and extraction can use different backends. For example, SearXNG (self-hosted) for search + Firecrawl (high quality) for extraction:

```python
def _get_capability_backend(capability: str) -> str:
    cfg = _load_web_config()
    specific = (cfg.get(f"{capability}_backend") or "").lower().strip()
    if specific and _is_backend_available(specific):
        return specific
    return _get_backend()
```

### 2.3 Firecrawl Lazy-Loading Proxy

Source location: `web_tools.py:72-87`

The Firecrawl SDK import takes ~200ms (pulling in httpcore + type tree). `_FirecrawlProxy` replaces the `Firecrawl` name at the module level, only performing the actual import on first call:

```python
class _FirecrawlProxy:
    __slots__ = ()
    def __call__(self, *args, **kwargs):
        return _load_firecrawl_cls()(*args, **kwargs)
    def __instancecheck__(self, obj):
        return isinstance(obj, _load_firecrawl_cls())

Firecrawl = _FirecrawlProxy()  # module-level replacement
```

This ensures that `patch("tools.web_tools.Firecrawl", ...)` tests still work correctly.

### 2.4 web_providers Module Separation

`tools/web_tools.py` abstracts search/extraction backends into an independent provider class system, located in the `tools/web_providers/` submodule:

| Provider Class | Source Location | Capability |
|-------------|---------|------|
| `SearXNGSearchProvider` | tools/web_providers/searxng.py | Search-only; does not support content extraction or crawling |
| `ExaSearchProvider` | tools/web_tools.py (inline) | Search + extraction |
| `Firecrawl` (lazy-loading proxy) | tools/web_tools.py:72-87 | Search + extraction + crawling |

**Important operational distinction**: SearXNG is a search-only backend — it can only execute search queries, not extract URL content or crawl pages. When SearXNG is configured as the search backend, extraction operations automatically fall back to another available backend (such as Firecrawl).

Source location: `tools/web_providers/searxng.py`, `tools/web_providers/base.py`

### 2.5 web_search_tool

Source location: `web_tools.py:1117-1260`

```python
def web_search_tool(query: str, limit: int = 5) -> str:
    backend = _get_search_backend()
    if backend == "parallel":
        response_data = _parallel_search(query, limit)
    elif backend == "exa":
        response_data = _exa_search(query, limit)
    elif backend == "searxng":
        response_data = SearXNGSearchProvider().search(query, limit)
    elif backend == "tavily":
        raw = _tavily_request("search", {"query": query, "max_results": min(limit, 20)})
        response_data = _normalize_tavily_search_results(raw)
    else:  # firecrawl
        response = _get_firecrawl_client().search(query=query, limit=limit)
        web_results = _extract_web_search_results(response)
```

Returns a unified format: `{"success": bool, "data": {"web": [{title, url, description, position}]}}`

### 2.6 web_extract_tool — LLM Summarization

Source location: `web_tools.py:1263-1597`

The core of the extraction tool is `process_content_with_llm()` — when raw content exceeds `min_length` (default 5000 characters), it uses Gemini 3 Flash Preview via OpenRouter to generate a summary:

```python
async def process_content_with_llm(
    content: str, url: str, query: Optional[str], ...
) -> Dict[str, Any]:
    # ... oversized content goes through _process_large_content_chunked() for chunked processing
```

Safety feature: URL embedded key detection — before fetching, each URL is scanned with `_PREFIX_RE`, checking both the raw and URL-decoded forms:

```python
from agent.redact import _PREFIX_RE
from urllib.parse import unquote
for _url in urls:
    if _PREFIX_RE.search(_url) or _PREFIX_RE.search(unquote(_url)):
        return json.dumps({"success": False, "error": "Blocked: URL contains API key or token"})
```

### 2.7 Async Signatures

Multiple tools implement core functionality using the async pattern:

| Tool | Async Function | Source Location |
|------|----------|---------|
| web_tools | `async def process_content_with_llm()` | web_tools.py:568+ |
| vision_tools | `async def _download_image()` / `async def vision_analyze_tool()` | vision_tools.py:129+, 406+ |
| session_search | `async def _summarize_session()` / `async def _summarize_all()` | session_search_tool.py:198+, 454+ |

These async functions call auxiliary LLM models via `async_call_llm()`, use `asyncio.gather()` for parallel summarization, and control concurrency through `asyncio.Semaphore`.

### 2.8 Tavily Result Normalization

Source location: `web_tools.py:388-442`

Different backends return different formats. The Tavily normalizer maps `results: [{title, url, content, score}]` to Hermes's standard format, and handles `failed_results` and `failed_urls`:

```python
def _normalize_tavily_search_results(response: dict) -> dict:
    web_results = []
    for i, result in enumerate(response.get("results", [])):
        web_results.append({
            "title": result.get("title", ""),
            "url": result.get("url", ""),
            "description": result.get("content", ""),
            "position": i + 1,
        })
    return {"success": True, "data": {"web": web_results}}
```

### 2.9 Debug Mode

Setting `WEB_TOOLS_DEBUG=true` enables verbose logging, generating `web_tools_debug_UUID.json` capturing all calls, results, and compression metrics.

---

## 3. tts_tool — Text-to-Speech

**Source location:** `tools/tts_tool.py` (2185 lines)

### 3.1 Design Goal

Convert text to speech audio files, supporting 11+ providers covering cloud to local. The model only sends text; users configure voice and provider. On messaging platforms, `MEDIA:<path>` markers are returned, intercepted by the send pipeline and delivered as native voice messages.

### 3.2 Provider Matrix

| Provider | Type | API Key | Default Voice | Default Model |
|--------|------|---------|----------|----------|
| Edge TTS | Cloud (free) | None | en-US-AriaNeural | — |
| ElevenLabs | Cloud (paid) | ELEVENLABS_API_KEY | Adam | eleven_multilingual_v2 |
| OpenAI | Cloud (paid) | OPENAI_API_KEY | alloy | gpt-4o-mini-tts |
| MiniMax | Cloud (paid) | MINIMAX_API_KEY | female-shaonv | speech-01 |
| Mistral (Voxtral) | Cloud (paid) | MISTRAL_API_KEY | Paul-Neutral | voxtral-mini-tts-2603 |
| Gemini | Cloud (paid) | GEMINI_API_KEY | Kore | gemini-2.5-flash-preview-tts |
| xAI | Cloud (paid) | XAI_API_KEY | eve | — |
| NeuTTS | Local (free) | None | — | espeak-ng |
| KittenTTS | Local (free) | None | Jasper | kitten-tts-nano-0.8-int8 (25MB) |
| Piper | Local (free) | None | en_US-lessac-medium | 44-language ONNX models |
| Command | Custom | None | — | User shell command template |

### 3.3 Core Dispatch Logic

Source location: `tts_tool.py:1533-1732`

```python
def text_to_speech_tool(text: str, output_path: Optional[str] = None) -> str:
    provider = _get_provider(tts_config)

    # Command provider takes priority over built-in (prevents tts.providers.openai.command from overriding real OpenAI)
    command_provider_config = _resolve_command_provider_config(provider, tts_config)

    # Text length truncation — each provider has different limits
    max_len = _resolve_max_text_length(provider, tts_config)
    if len(text) > max_len:
        text = text[:max_len]

    # Detect platform to select output format
    platform = get_session_env("HERMES_SESSION_PLATFORM", "").lower()
    want_opus = (platform == "telegram")  # Telegram voice bubbles require Opus

    # Dispatch to corresponding provider
    if command_provider_config is not None:
        file_str = _generate_command_tts(text, file_str, provider, command_provider_config, tts_config)
    elif provider == "elevenlabs":
        _generate_elevenlabs(text, file_str, tts_config)
    elif provider == "openai":
        _generate_openai_tts(text, file_str, tts_config)
    # ... more providers
    else:  # default: Edge TTS, fallback NeuTTS
```

### 3.4 Gemini TTS — PCM to WAV Conversion

Source location: `tts_tool.py:1090-1224`

The Gemini TTS API returns raw PCM (L16, 24kHz, mono). `_generate_gemini_tts()` wraps it into a standard WAV:

```python
GEMINI_TTS_SAMPLE_RATE = 24000
GEMINI_TTS_CHANNELS = 1
GEMINI_TTS_SAMPLE_WIDTH = 2  # 16-bit PCM (L16)

def _wrap_pcm_as_wav(pcm_data: bytes, ...) -> bytes:
    # Write RIFF/WAVE header + PCM data
```

### 3.5 Command TTS — Custom Shell Command

Source location: `tts_tool.py:367-678`

Users can declare any number of command providers in `~/.hermes/config.yaml`:

```yaml
tts:
  providers:
    my_local:
      type: command
      command: "piper --model {model} --output_file {output_path}"
```

Hermes writes the text to a temporary file, runs the command, and the command must write audio to the expected path. Supports timeout, process tree termination, and output format alignment.

### 3.6 Streaming Playback

Source location: `tts_tool.py:1910+`

`stream_tts_to_speaker()` generates and plays audio concurrently in CLI mode, using `sounddevice`'s callback mode and a thread queue.

---

## 4. vision_tools — Image/Video Analysis

**Source location:** `tools/vision_tools.py` (1168 lines)

### 4.1 Design Goal

Accept URL or local file path, download the image, convert to Base64, and call the LLM for analysis through the auxiliary vision router. Built-in SSRF protection, size limits, and auto-resize retry.

### 4.2 Image Download and Security

Source location: `vision_tools.py:129-231`

```python
async def _download_image(image_url: str, destination: Path, max_retries: int = 3) -> Path:
    # SSRF redirect protection — validate each 302 target
    async def _ssrf_redirect_guard(response):
        if response.is_redirect and response.next_request:
            redirect_url = str(response.next_request.url)
            if not is_safe_url(redirect_url):
                raise ValueError(f"Blocked redirect to private/internal address: {redirect_url}")

    async with httpx.AsyncClient(
        follow_redirects=True,
        event_hooks={"response": [_ssrf_redirect_guard]},
    ) as client:
        # Content-Length pre-check: 50 MB limit
        cl = response.headers.get("content-length")
        if cl and int(cl) > _VISION_MAX_DOWNLOAD_BYTES:
            raise ValueError(f"Image too large ({int(cl)} bytes)")
```

### 4.3 MIME Type Detection

Source location: `vision_tools.py:107-126`

Based on file header magic bytes, not relying on extensions:

```python
def _detect_image_mime_type(image_path: Path) -> Optional[str]:
    with image_path.open("rb") as f:
        header = f.read(64)
    if header.startswith(b"\x89PNG\r\n\x1a\n"): return "image/png"
    if header.startswith(b"\xff\xd8\xff"):       return "image/jpeg"
    if header.startswith((b"GIF87a", b"GIF89a")): return "image/gif"
    if len(header) >= 12 and header[:4] == b"RIFF" and header[8:12] == b"WEBP":
        return "image/webp"
    # SVG: read first 4KB to check for <svg> tag
```

### 4.4 Auto-Resize Retry

Source location: `vision_tools.py:406-598`

`vision_analyze_tool()` first sends the image at original size. If the API rejects it (image too large error), it automatically calls `_resize_image_for_vision()` to shrink to ~5MB and retries:

```python
try:
    response = await async_call_llm(**call_kwargs)
except Exception as _api_err:
    if _is_image_size_error(_api_err) and len(image_data_url) > _RESIZE_TARGET_BYTES:
        image_data_url = _resize_image_for_vision(temp_image_path, mime_type=detected_mime_type)
        messages[0]["content"][1]["image_url"]["url"] = image_data_url
        response = await async_call_llm(**call_kwargs)  # retry
```

### 4.5 Video Analysis

Source location: `vision_tools.py:915-1148`

`video_analyze_tool()` supports video file analysis, using a similar Base64 encoding and auxiliary router pattern.

---

## 5. file_tools — File Operations

**Source location:** `tools/file_tools.py` (1143 lines)

### 5.1 Design Goal

Provide safe file read/write/patch/search operations, with built-in multi-layer protection to prevent the model from accidentally damaging the system.

### 5.2 Three-Layer Path Safety Guard

| Layer | Guard | Blocked Targets |
|----|------|----------|
| 1 | Device paths | `/dev/zero`, `/dev/random`, `/dev/stdin`, `/proc/*/fd/0-2` — infinite output or blocking |
| 2 | Sensitive path writes | `/etc/`, `/boot/`, `/usr/lib/systemd/`, `/var/run/docker.sock` — system files |
| 3 | Binary extensions | `.pyc`, `.so`, `.exe`, `.png` etc. — binary content meaningless to the model |

Source location: `file_tools.py:69-174`

```python
_BLOCKED_DEVICE_PATHS = frozenset({
    "/dev/zero", "/dev/random", "/dev/urandom", "/dev/full",
    "/dev/stdin", "/dev/tty", "/dev/console",
    "/dev/stdout", "/dev/stderr",
    "/dev/fd/0", "/dev/fd/1", "/dev/fd/2",
})

_SENSITIVE_PATH_PREFIXES = (
    "/etc/", "/boot/", "/usr/lib/systemd/",
    "/private/etc/", "/private/var/",
)
_SENSITIVE_EXACT_PATHS = {"/var/run/docker.sock", "/run/docker.sock"}
```

Note: Device path checking uses literal paths (no symlink resolution), because resolving `/dev/stdin` via symlink to `/dev/pts/0` would bypass the check.

### 5.3 Deduplication and Loop Detection

Source location: `file_tools.py:447-646`

`read_file_tool()` maintains a per-task read tracker:

- **Dedup**: Same (path, offset, limit) + file unmodified → returns a lightweight stub instead of re-sending content
- **Loop detection**: 4 consecutive reads of the same region → hard block returning an error
- **Stub loop guard**: After 2 stub returns, escalation to hard block
- **Capacity control**: `_cap_read_tracker_data()` prevents long sessions from accumulating massive tracker state

```python
if count >= 4:
    return json.dumps({
        "error": f"BLOCKED: You have read this exact file region {count} times in a row. "
                 "STOP re-reading and proceed with your task.",
        "path": path, "already_read": count,
    })
```

### 5.4 Character Count Limit

Source location: `file_tools.py:35-36, 547-569`

Default 100,000 characters (~25-35K tokens), configurable via `file_read_max_chars` in `config.yaml`. When exceeded, it refuses and guides the model to use offset+limit.

### 5.5 File Staleness Detection on Write

Source location: `file_tools.py:762-792`

`_check_file_staleness()` checks before write/patch: if the file has been externally modified since the model's last read (by another agent, concurrent process), it returns a warning. Implemented through the `file_state` module's cross-agent registry.

---

## 6. transcription_tools — Audio Transcription

**Source location:** `tools/transcription_tools.py` (911 lines)

### 6.1 Design Goal

Transcribe voice messages to text, automatically called by the Gateway to handle voice messages sent by users on Telegram/Discord/WhatsApp/Slack/Signal. Defaults to local faster-whisper (free), supporting 5 cloud providers.

### 6.2 Provider Matrix

| Provider | Type | Requirements | Default Model |
|--------|------|------|----------|
| local (faster-whisper) | Local (free) | pip install faster-whisper | base (~150MB) |
| local_command | Local | Custom command template | — |
| groq | Cloud (free tier) | GROQ_API_KEY | whisper-large-v3-turbo |
| openai | Cloud (paid) | VOICE_TOOLS_OPENAI_KEY | whisper-1 |
| mistral | Cloud (paid) | MISTRAL_API_KEY | voxtral-mini-latest |
| xai | Cloud (paid) | XAI_API_KEY | grok-stt (21 languages, ITN, speaker diarization) |

### 6.3 Core Dispatch

Source location: `transcription_tools.py:789-868`

```python
def transcribe_audio(file_path: str, model: Optional[str] = None) -> Dict[str, Any]:
    provider = _get_provider(stt_config)

    if provider == "local":
        model_name = _normalize_local_model(model or local_cfg.get("model", DEFAULT_LOCAL_MODEL))
        return _transcribe_local(file_path, model_name)
    if provider == "groq":
        return _transcribe_groq(file_path, model_name)
    if provider == "openai":
        return _transcribe_openai(file_path, model_name)
    # ...
```

### 6.4 Model Name Normalization

Source location: `transcription_tools.py:175-193`

Cloud model names (e.g., `whisper-1`) are invalid for local faster-whisper (which requires `tiny/base/small/medium/large-v*`). `_normalize_local_model()` automatically maps and emits a warning:

```python
def _normalize_local_model(model_name: Optional[str]) -> str:
    if not model_name or model_name in OPENAI_MODELS or model_name in GROQ_MODELS:
        if model_name and (model_name in OPENAI_MODELS or model_name in GROQ_MODELS):
            logger.warning("STT model '%s' is a cloud-only name, falling back to '%s'", ...)
        return DEFAULT_LOCAL_MODEL
    return model_name
```

### 6.5 Audio Validation

Source location: `transcription_tools.py:298-350`

`_validate_audio_file()` checks:
- File existence and readability
- Extension in supported set: `.mp3/.mp4/.mpeg/.mpga/.m4a/.wav/.webm/.ogg/.aac/.flac`
- File size not exceeding 25 MB
- No CUDA library errors (common local GPU configuration issue)

---

## 7. session_search_tool — Session Search

**Source location:** `tools/session_search_tool.py` (605 lines)

### 7.1 Design Goal

Search transcripts of past sessions, returning focused summaries instead of raw text, keeping the main model's context window clean.

### 7.2 Two-Stage Pipeline

```
Stage 1: FTS5 Search
  db.search_messages(query, role_filter, limit=50)
    → relevance-sorted message list
    → grouped by session_id, take top N unique sessions (default 3)
    → resolve sub-sessions to parent sessions (delegation chain)
    → exclude current session (model already has that context)

Stage 2: LLM Summarization
  For each matching session:
    1. Load full conversation → _format_conversation()
    2. Truncate to ~100K characters → _truncate_around_matches()
    3. Send to auxiliary model → async_call_llm(task="session_search")
    4. Return structured summary
```

### 7.3 Smart Truncation Algorithm

Source location: `session_search_tool.py:113-195`

`_truncate_around_matches()` is not a simple first-N-characters truncation; it selects the window covering the most match positions:

1. **Full phrase search**: Find the entire query as a phrase
2. **Proximity co-occurrence**: Positions where all query terms appear within 200 characters simultaneously
3. **Individual word positions**: Independent positions of each term (last resort)
4. **Window selection**: For each candidate position, calculate how many matches the window (25% before + 75% after) covers, and select the optimal one

```python
for candidate in match_positions:
    ws = max(0, candidate - max_chars // 4)  # 25% before, 75% after
    we = ws + max_chars
    count = sum(1 for p in match_positions if ws <= p < we)
    if count > best_count:
        best_count = count
        best_start = ws
```

### 7.4 Parallel Concurrency Control

Source location: `session_search_tool.py:454-477`

Multiple session summaries execute in parallel, with concurrency controlled through a semaphore (default 3, max 5):

```python
max_concurrency = min(_get_session_search_max_concurrency(), max(1, len(tasks)))
semaphore = asyncio.Semaphore(max_concurrency)

async def _bounded_summary(text, meta):
    async with semaphore:
        return await _summarize_session(text, query, meta)

coros = [_bounded_summary(text, meta) for _, _, text, meta in tasks]
results = await asyncio.gather(*coros, return_exceptions=True)
```

Uses `_run_async()` instead of `asyncio.run()` to avoid event loop conflicts with cached AsyncOpenAI/httpx clients.

### 7.5 Sub-Session Resolution

Source location: `session_search_tool.py:385-408`

Compression and delegation create sub-sessions, but users perceive the parent conversation. `_resolve_to_parent()` traces the `parent_session_id` chain back to the root session, ensuring results are deduplicated and all branches of the currently active conversation are excluded.

---

## 8. homeassistant_tool — Smart Home Control

**Source location:** `tools/homeassistant_tool.py` (513 lines)

### 8.1 Design Goal

Control smart home devices through the Home Assistant REST API, providing 4 tools: entity list, state query, service list, service call.

### 8.2 Tool Matrix

| Tool | Schema | Function | HA API Endpoint |
|------|--------|------|-------------|
| `ha_list_entities` | LIST_ENTITIES_SCHEMA | List/filter entities | GET /api/states |
| `ha_get_state` | GET_STATE_SCHEMA | Query single entity details | GET /api/states/{entity_id} |
| `ha_list_services` | LIST_SERVICES_SCHEMA | List available services | GET /api/services |
| `ha_call_service` | CALL_SERVICE_SCHEMA | Call service to control device | POST /api/services/{domain}/{service} |

### 8.3 Safety: Service Domain Blacklist

Source location: `homeassistant_tool.py:53-60`

HA does not provide service-level access control — all security layers must be implemented on our side:

```python
_BLOCKED_DOMAINS = frozenset({
    "shell_command",    # arbitrary shell commands (root within HA container)
    "command_line",     # sensors/switches executing shell commands
    "python_script",    # sandboxed but can escalate via hass.services.call()
    "pyscript",         # broader scripting integration
    "hassio",           # plugin control, host shutdown/restart, container stdin
    "rest_command",     # HTTP requests from HA server (SSRF vector)
})
```

### 8.4 Path Traversal Protection

Source location: `homeassistant_tool.py:38-48`

Domain and service names are directly inserted into the URL `/api/services/{domain}/{service}`. Regex validation only allows lowercase ASCII letters + digits + underscores, blocking `../../api/config` style path traversal:

```python
_ENTITY_ID_RE = re.compile(r"^[a-z_][a-z0-9_]*\.[a-z0-9_]+$")
_SERVICE_NAME_RE = re.compile(r"^[a-z][a-z0-9_]*$")
```

### 8.5 Entity Filtering

Source location: `homeassistant_tool.py:77-102`

`_filter_and_summarize()` supports filtering by domain (`light.`/`switch.`/`climate.`) and area name (fuzzy matching against friendly_name and area attributes), returning a compact summary instead of full state.

---

## Auxiliary Tool Briefs

### 9. code_execution_tool — Code Execution Sandbox

**Source location:** `tools/code_execution_tool.py` (1621 lines)

Lets the LLM write Python scripts, call Hermes tools via RPC, and collapse multi-step tool chains into a single inference.

Dual transport architecture:
- **Local (UDS)**: Unix Domain Socket + RPC listener thread
- **Remote (file RPC)**: request/response files + polling thread, supports Docker/SSH/Modal/Daytona

7 allowed sandbox tools: `web_search`, `web_extract`, `read_file`, `write_file`, `search_files`, `patch`, `terminal`

Resource limits: default 300s timeout, 50 tool calls, 50KB stdout / 10KB stderr.

### 10. image_generation_tool — Image Generation

**Source location:** `tools/image_generation_tool.py` (1002 lines)

Generates images via FAL.ai. The `FAL_MODELS` catalog declares metadata for each model (size family, defaults, supports whitelist, upscaler flag). `_build_fal_payload()` translates unified input (prompt + aspect_ratio) into model-specific payloads, filtering unsupported parameters through the `supports` whitelist. Optional Clarity Upscaler post-processing.

### 11. voice_mode — Voice Mode

**Source location:** `tools/voice_mode.py` (1017 lines)

CLI push-to-record mode. Two implementations: `AudioRecorder` (desktop) and `TermuxAudioRecorder` (Android). After recording, transcription via `transcription_tools.transcribe_audio()`. `is_whisper_hallucination()` filters common Whisper hallucination outputs ("Thank you for watching", "Subtitles by..." etc.).

### 12. kanban_tools — Kanban Collaboration

**Source location:** `tools/kanban_tools.py` (855 lines)

7 tools: `kanban_show`, `kanban_complete`, `kanban_block`, `kanban_heartbeat`, `kanban_comment`, `kanban_create`, `kanban_link`. Only registered when `HERMES_KANBAN_TASK` environment variable is set (scheduler-dispatched worker agents) or the profile includes the `kanban` toolset. Tools directly access `~/.hermes/kanban.db`, bypassing the terminal backend's shell quoting issues.

### 13. mixture_of_agents_tool — Mixture of Agents

**Source location:** `tools/mixture_of_agents_tool.py` (541 lines)

Based on the Mixture-of-Agents method from Wang et al. (arXiv:2406.04692v1):

- **Layer 1**: 4 reference models (Claude Opus 4.6 / Gemini 2.5 Pro / GPT-5.4 Pro / DeepSeek V3.2) generate initial responses in parallel (temp=0.6)
- **Layer 2**: Aggregation model (Claude Opus 4.6) synthesizes the final response (temp=0.4)
- Fault tolerance: minimum 1 reference model success to continue, 6 retries

### 14. memory_tool — Persistent Memory

**Source location:** `tools/memory_tool.py` (586 lines)

Dual storage: `MEMORY.md` (agent notes) + `USER.md` (user preferences). Key design:

- **Frozen snapshots**: Memory in system prompts is frozen at session start, keeping prefix cache stable
- **Immediate persistence**: Tool calls write to disk immediately (atomic replacement via `os.replace()`)
- **Injection scanning**: `_scan_memory_content()` detects prompt injection, role hijacking, key exfiltration, and other threat patterns
- **Invisible character detection**: Blocks zero-width characters (U+200B etc.) and bidirectional control characters (U+202A-U+202E)
- **Character limits**: MEMORY 2200 characters, USER 1375 characters (model-independent)

---

## Comparison Tables

### Table 1: Messaging Platform Support Matrix

| Platform | Target Parsing | Media Attachments | Message Chunking | Format Conversion | Special Handling |
|------|----------|----------|----------|----------|----------|
| Telegram | Regex + topics | photo/video/voice/audio/doc | UTF-16 length | MarkdownV2/HTML auto-switch | 429 retry, topic threads |
| Discord | Regex + topics | multipart upload | 2000 chars | — | Forum channels (type 15) auto-create posts |
| Slack | [CGD] regex | Not supported | 40000 chars | mrkdwn formatting | — |
| Signal | E.164 phone | JSON-RPC attachment | — | — | Rate limit scheduler |
| WhatsApp | E.164 phone | Not supported | — | — | Local bridge HTTP API |
| WeChat | WeChat ID | Native adapter | — | — | .env synthesized config |
| Matrix | !/@ ID | Native adapter | — | — | — |
| Feishu | Feishu ID | Native adapter | — | — | — |
| Email | — | Not supported | — | — | SMTP sending |

### Table 2: TTS Provider Comparison

| Provider | Latency | Cost | Output Format | Installation Required | Voice Selection |
|--------|------|------|----------|--------|----------|
| Edge TTS | Medium | Free | MP3 | pip install edge-tts | 400+ neural voices |
| ElevenLabs | Low | Paid | MP3/Opus | pip install elevenlabs | High quality + streaming |
| OpenAI | Low | Paid | Opus/MP3 | pip install openai | 6 preset voices |
| MiniMax | Low | Paid | MP3 | — | Voice cloning |
| Mistral | Low | Paid | Opus | pip install mistralai | Multilingual |
| Gemini | Medium | Paid | WAV (PCM) | — | 30 preset voices |
| xAI | Low | Paid | MP3 | — | Grok voices |
| NeuTTS | Fast | Free | WAV | espeak-ng + neutts | — |
| KittenTTS | Fast | Free | WAV | 25MB model | 1 voice |
| Piper | Fastest | Free | WAV | pip install piper-tts | 44 languages |
| Command | Variable | Variable | Configurable | — | Configurable |

### Table 3: Web Search Backend Comparison

| Backend | Search | Extract | Crawl | API Key | Features |
|------|------|------|------|---------|------|
| Firecrawl | yes | yes | yes | FIRECRAWL_API_KEY | High quality, Nous gateway support |
| Exa | yes | yes | no | EXA_API_KEY | Semantic search |
| Parallel | yes | yes | no | PARALLEL_API_KEY | — |
| Tavily | yes | yes | yes | TAVILY_API_KEY | Includes failed_results normalization |
| SearXNG | yes | no | no | SEARXNG_URL (self-hosted) | Fully private, no API key |

### Table 4: STT Provider Comparison

| Provider | Latency | Cost | Model | Special Capabilities |
|--------|------|------|------|----------|
| local (faster-whisper) | Medium | Free | base→large-v3 | Offline, optional GPU |
| local_command | Variable | Free | Custom | Flexible |
| Groq | Very low | Free tier | whisper-large-v3-turbo | — |
| OpenAI | Low | Paid | whisper-1 / gpt-4o-transcribe | — |
| Mistral | Low | Paid | voxtral-mini-latest | Multilingual |
| xAI | Low | Paid | grok-stt | 21 languages, ITN, speaker diarization |

---

## Summary Table

| Component Name | Lines | Responsibility |
|--------|------|------|
| send_message_tool | 1780 | Cross-platform message routing for 15+ platforms, target parsing, media attachments, message chunking, Cron dedup |
| web_tools | 2223 | 5-backend web search/extract, LLM summarization, URL safety check, Firecrawl lazy loading |
| tts_tool | 2185 | 11+ TTS providers, Opus/MP3 output, Command custom, streaming playback |
| vision_tools | 1168 | Image/video analysis, SSRF protection, auto-resize retry, auxiliary vision routing |
| file_tools | 1143 | File read/write/patch/search, three-layer path safety guard, dedup loop detection, 100K character limit |
| transcription_tools | 911 | 6 STT providers, audio validation, model name normalization, CUDA error detection |
| session_search_tool | 605 | FTS5 full-text search, smart truncation, parallel LLM summarization, sub-session resolution |
| homeassistant_tool | 513 | HA REST API 4 tools, domain blacklist, path traversal protection, entity filtering |
| code_execution_tool | 1621 | UDS/file dual-transport RPC, hermes_tools.py stubs, 7 sandbox tools |
| image_generation_tool | 1002 | FAL.ai multi-model catalog, supports whitelist, Clarity Upscaler |
| voice_mode | 1017 | Push-to-record (desktop/Termux), Whisper hallucination filtering, TTS playback |
| kanban_tools | 855 | 7 kanban tools, scheduler/orchestrator dual gating, direct DB access |
| mixture_of_agents_tool | 541 | 4 reference models parallel + aggregation synthesis, 6-retry fault tolerance |
| memory_tool | 586 | Dual storage MEMORY/USER, frozen snapshots, injection scanning, atomic replacement |

---

[← 05 — Tool System](/en/chapters/05-tool-system) | [19 — Hermes CLI Subsystem →](/en/chapters/19-hermes-cli-subsystem)