# 18 — 工具体系扩展: 跨平台消息/Web 搜索/语音/视觉/文件/智能家居八大核心工具

[← 05 — 工具体系](/zh-CN/chapters/05-tool-system) | [19 — Hermes CLI 子系统 →](/zh-CN/chapters/19-hermes-cli-subsystem)

---

## 源码文件

### 八大核心工具

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `tools/send_message_tool.py` | 1780 | 跨平台消息路由 — 15+ 平台适配、频道解析、媒体附件、消息分块 |
| `tools/web_tools.py` | 2223 | Web 搜索 + 内容提取 — 5 后端(Exa/Firecrawl/Parallel/Tavily/SearXNG)、LLM 摘要 |
| `tools/tts_tool.py` | 2185 | 文本转语音 — 11+ 提供商(Edge/ElevenLabs/OpenAI/MiniMax/Mistral/Gemini/xAI/NeuTTS/KittenTTS/Piper/Command)、流式播放 |
| `tools/vision_tools.py` | 1168 | 图像/视频分析 — 多提供商视觉路由、Base64 编码、SSRF 防护、自动缩放 |
| `tools/file_tools.py` | 1143 | 文件操作 — 读写/补丁/搜索/去重循环检测、路径安全守卫 |
| `tools/transcription_tools.py` | 911 | 音频转写 — 6 提供商(本地 Whisper/Groq/OpenAI/Mistral/xAI/本地命令)、语言检测 |
| `tools/session_search_tool.py` | 605 | 会话搜索 — FTS5 全文检索、LLM 摘要、并行并发控制 |
| `tools/homeassistant_tool.py` | 513 | 智能家居控制 — HA REST API、实体发现、服务域安全过滤 |

### 辅助工具

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `tools/code_execution_tool.py` | 1621 | 代码执行沙箱 — UDS RPC + 文件 RPC 双传输、hermes_tools.py 存根生成 |
| `tools/image_generation_tool.py` | 1002 | 图像生成 — FAL.ai 多模型目录、Clarity Upscaler 后处理 |
| `tools/voice_mode.py` | 1017 | 语音模式 — 推送录音(sounddevice/Termux)、TTS 播放、幻觉过滤 |
| `tools/kanban_tools.py` | 855 | 看板协作 — show/complete/block/heartbeat/comment/create/link 7 工具 |
| `tools/mixture_of_agents_tool.py` | 541 | 混合代理 — 多前沿模型并行推理 + 聚合合成 |
| `tools/memory_tool.py` | 586 | 持久记忆 — MEMORY.md/USER.md 双存储、冻结快照、注入扫描 |

**合计: 14 文件, 16,150 行**

## 一句话总结

八大扩展工具覆盖 Hermes 的跨平台消息投递(15+ 平台)、Web 搜索与提取(5 后端)、语音合成(11+ 提供商)与转写(6 提供商)、视觉分析、文件操作(多层安全守卫)、会话回忆(FTS5 + LLM 摘要)和智能家居控制(HA REST API)，辅以代码沙箱、图像生成、语音模式、看板协作、混合代理推理和持久记忆六大工具。

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

Hermes 的扩展工具体系在核心 Registry/Delegate/Browser/MCP 之上提供了八大垂直能力模块。`send_message_tool` 是唯一跨 15+ 消息平台的路由器，通过正则解析 target 字符串和频道目录解析实现统一寻址。`web_tools` 支持 5 种搜索后端和 LLM 内容摘要，通过 `_FirecrawlProxy` 实现懒加载避免 200ms 导入开销。`tts_tool` 覆盖 11+ 语音提供商，从免费 Edge TTS 到本地 Piper，支持 Opus/MP3 输出和流式播放。`vision_tools` 通过辅助视觉路由器统一调用，内建 SSRF 重定向防护和自动缩放重试。`file_tools` 实现了三层安全守卫(设备路径/敏感路径/二进制扩展)和去重循环检测。`transcription_tools` 以本地 faster-whisper 为默认，支持 6 种 STT 提供商。`session_search_tool` 基于 FTS5 + LLM 摘要实现高效会话回忆。`homeassistant_tool` 通过 HA REST API 控制智能家居，内建 5 个危险服务域黑名单。

---

## 1. send_message_tool — 跨平台消息路由

**源码位置:** `tools/send_message_tool.py` (1780 行)

### 1.1 设计目标

将消息从 Hermes 核心投递到任何已连接的消息平台——Telegram、Discord、Slack、Signal、WhatsApp、WeChat(微信)、Matrix、Feishu(飞书)、Yuanbao、DingTalk、WeCom、Mattermost、BlueBubbles、QQBot、Email、SMS 及所有插件平台。工具以统一的 `platform:target_ref` 格式接收目标，内部完成平台适配。

### 1.2 核心数据流

```
用户/模型调用 send_message(action='send', target='telegram:-100123:42', message='...')
         │
         ▼
   ┌─ _parse_target_ref() ──────────────────────────────────────────┐
   │  按 platform_name 分发正则:                                      │
   │  telegram  → _TELEGRAM_TOPIC_TARGET_RE  (-?\d+)(?::(\d+))?    │
   │  discord   → _NUMERIC_TOPIC_RE          (同 telegram 格式)      │
   │  slack     → _SLACK_TARGET_RE           [CGD][A-Z0-9]{8,}     │
   │  feishu   → _FEISHU_TARGET_RE           (oc|ou|on|chat|open)_ │
   │  weixin    → _WEIXIN_TARGET_RE          (wxid|gh|...@chatroom)│
   │  signal/sms/whatsapp → _E164_TARGET_RE  \+(\d{7,15})         │
   │  yuanbao  → _YUANBAO_TARGET_RE          (group|direct):...    │
   │  matrix   → !roomid / @userid 检查                              │
   └─────────────────────────────────────────────────────────────────┘
         │
         ▼
   resolve_channel_name() → 模糊名称解析为数字 ID
         │
         ▼
   _send_to_platform() → 消息分块 → 平台特定 _send_*() 函数
```

### 1.3 关键实现: 目标解析

源码位置: `send_message_tool.py:311-351`

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
    # ... 更多平台
    if platform_name in _PHONE_PLATFORMS:
        match = _E164_TARGET_RE.fullmatch(target_ref)
        if match:
            return target_ref.strip(), None, True  # 保留 + 号 — E.164 格式
    # ...
    return None, None, False
```

Slack 的 `U...` (用户 ID) 和 `W...` (工作区 ID) 被故意排除——因为 `chat.postMessage` 只接受 `C/G/D` 会话 ID，直接发送用户 ID 会静默失败。

### 1.4 消息分块与平台适配

源码位置: `send_message_tool.py:440-667`

`_send_to_platform()` 是核心路由函数。它根据平台消息长度上限自动分块:

```python
_MAX_LENGTHS = {
    Platform.TELEGRAM: TelegramAdapter.MAX_MESSAGE_LENGTH,  # 4096 UTF-16 code units
    Platform.DISCORD: DiscordAdapter.MAX_MESSAGE_LENGTH,     # 2000 chars
    Platform.SLACK: SlackAdapter.MAX_MESSAGE_LENGTH,         # 40000 chars
}
# ...
chunks = BasePlatformAdapter.truncate_message(message, max_len, len_fn=_len_fn)
```

Telegram 使用 `utf16_len()` 而非字符计数——因为 Telegram API 以 UTF-16 码元衡量长度。

### 1.5 媒体附件处理

源码位置: `send_message_tool.py:22-49`

消息中的 `MEDIA:<path>` 标记被 `BasePlatformAdapter.extract_media()` 提取，然后按平台路由:

- **Telegram**: 按扩展名分发到 `send_photo/send_video/send_voice/send_audio/send_document`
- **Discord**: multipart/form-data 上传，论坛频道(类型 15)创建帖子+附件
- **Matrix/Signal/Yuanbao/Feishu/WeChat**: 各有原生媒体适配
- **不支持媒体的平台**: 返回警告但文本仍然发送

### 1.6 Telegram 重试策略

源码位置: `send_message_tool.py:73-113`

```python
def _telegram_retry_delay(exc: Exception, attempt: int) -> float | None:
    retry_after = getattr(exc, "retry_after", None)
    if retry_after is not None:
        return max(float(retry_after), 0.0)
    # 429/502/503/504 → 指数退避
    if "429" in text or "too many requests" in text:
        return float(2 ** attempt)
```

### 1.7 Cron 去重

源码位置: `send_message_tool.py:388-416`

当 Cron 调度器已设定自动投递目标时，`send_message` 会检测到目标重复并跳过，避免同一条消息被发送两次:

```python
def _maybe_skip_cron_duplicate_send(platform_name, chat_id, thread_id):
    auto_target = _get_cron_auto_delivery_target()
    if auto_target and same_target:
        return {"success": True, "skipped": True, "reason": "cron_auto_delivery_duplicate_target"}
```

### 1.8 错误净化

源码位置: `send_message_tool.py:60-65`

所有错误文本在返回给模型前经过双层净化——先 `redact_sensitive_text()` 脱敏，再正则擦除 URL 查询参数和赋值语句中的 token:

```python
def _sanitize_error_text(text) -> str:
    redacted = redact_sensitive_text(text)
    redacted = _URL_SECRET_QUERY_RE.sub(lambda m: f"{m.group(1)}***", redacted)
    redacted = _GENERIC_SECRET_ASSIGN_RE.sub(lambda m: f"{m.group(1)}=***", redacted)
    return redacted
```

### 1.9 Discord 论坛频道

源码位置: `send_message_tool.py:823-1017`

Discord 论坛频道(类型 15)拒绝 `POST /messages`。`_send_discord()` 实现三层检测:

1. **频道目录缓存** → `lookup_channel_type("discord", chat_id)`
2. **进程本地探测缓存** → `_probe_is_forum_cached(chat_id)`
3. **实时 GET /channels/{id} 探测** → 结果 memoize

论坛频道自动创建线程帖子，线程名从消息首行派生(上限 100 字符)。

---

## 2. web_tools — Web 搜索 + 内容提取

**源码位置:** `tools/web_tools.py` (2223 行)

### 2.1 设计目标

提供统一的 Web 搜索和内容提取接口，支持 5 种后端自动选择，并通过 LLM 摘要压缩大页面内容以节省 token。

### 2.2 后端选择架构

源码位置: `web_tools.py:121-184`

```
配置优先级:
  1. config.yaml → web.search_backend / web.extract_backend (per-capability override)
  2. config.yaml → web.backend (shared fallback)
  3. 环境变量自动检测 → firecrawl > parallel > tavily > exa > searxng
```

关键: 搜索和提取可以使用不同后端。例如 SearXNG(自托管)搜索 + Firecrawl(高质量)提取:

```python
def _get_capability_backend(capability: str) -> str:
    cfg = _load_web_config()
    specific = (cfg.get(f"{capability}_backend") or "").lower().strip()
    if specific and _is_backend_available(specific):
        return specific
    return _get_backend()
```

### 2.3 Firecrawl 懒加载代理

源码位置: `web_tools.py:72-87`

Firecrawl SDK 导入耗时 ~200ms(拉入 httpcore + 类型树)。`_FirecrawlProxy` 在模块级别替换 `Firecrawl` 名字，仅在首次调用时真正导入:

```python
class _FirecrawlProxy:
    __slots__ = ()
    def __call__(self, *args, **kwargs):
        return _load_firecrawl_cls()(*args, **kwargs)
    def __instancecheck__(self, obj):
        return isinstance(obj, _load_firecrawl_cls())

Firecrawl = _FirecrawlProxy()  # 模块级替换
```

这确保了 `patch("tools.web_tools.Firecrawl", ...)` 的测试仍然正常工作。

### 2.4 web_providers 模块分离

`tools/web_tools.py` 将搜索/提取后端抽象为独立的 provider 类系统，位于 `tools/web_providers/` 子模块：

| Provider 类 | 源码位置 | 能力 |
|-------------|---------|------|
| `SearXNGSearchProvider` | tools/web_providers/searxng.py | 仅搜索（search-only），不支持内容提取或爬取 |
| `ExaSearchProvider` | tools/web_tools.py (内联) | 搜索 + 提取 |
| `Firecrawl` (懒加载代理) | tools/web_tools.py:72-87 | 搜索 + 提取 + 爬取 |

**重要运营区别**：SearXNG 是 search-only 后端——它只能执行搜索查询，不能提取 URL 内容或爬取页面。当配置 SearXNG 为搜索后端时，提取操作会自动回退到其他可用后端（如 Firecrawl）。

源码位置: `tools/web_providers/searxng.py`, `tools/web_providers/base.py`

### 2.5 web_search_tool

源码位置: `web_tools.py:1117-1260`

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

返回统一格式: `{"success": bool, "data": {"web": [{title, url, description, position}]}}`

### 2.6 web_extract_tool — LLM 摘要

源码位置: `web_tools.py:1263-1597`

提取工具的核心是 `process_content_with_llm()`——当原始内容超过 `min_length`(默认 5000 字符)时，使用 Gemini 3 Flash Preview 通过 OpenRouter 生成摘要:

```python
async def process_content_with_llm(
    content: str, url: str, query: Optional[str], ...
) -> Dict[str, Any]:
    # ... 超大内容走 _process_large_content_chunked() 分块处理
```

安全特性: URL 内嵌密钥检测——在获取前对每个 URL 做 `_PREFIX_RE` 扫描，同时检查原始和 URL 解码形式:

```python
from agent.redact import _PREFIX_RE
from urllib.parse import unquote
for _url in urls:
    if _PREFIX_RE.search(_url) or _PREFIX_RE.search(unquote(_url)):
        return json.dumps({"success": False, "error": "Blocked: URL contains API key or token"})
```

### 2.7 异步签名

多个工具使用 async 模式实现核心功能：

| 工具 | 异步函数 | 源码位置 |
|------|----------|---------|
| web_tools | `async def process_content_with_llm()` | web_tools.py:568+ |
| vision_tools | `async def _download_image()` / `async def vision_analyze_tool()` | vision_tools.py:129+, 406+ |
| session_search | `async def _summarize_session()` / `async def _summarize_all()` | session_search_tool.py:198+, 454+ |

这些异步函数通过 `async_call_llm()` 调用辅助 LLM 模型，使用 `asyncio.gather()` 实现并行摘要，通过 `asyncio.Semaphore` 控制并发。

### 2.8 Tavily 结果归一化

源码位置: `web_tools.py:388-442`

不同后端返回不同格式。Tavily 归一化器将 `results: [{title, url, content, score}]` 映射为 Hermes 标准格式，并处理 `failed_results` 和 `failed_urls`:

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

### 2.9 调试模式

设置 `WEB_TOOLS_DEBUG=true` 启用详细日志，生成 `web_tools_debug_UUID.json` 捕获所有调用、结果和压缩指标。

---

## 3. tts_tool — 文本转语音

**源码位置:** `tools/tts_tool.py` (2185 行)

### 3.1 设计目标

将文本转换为语音音频文件，支持 11+ 提供商从云端到本地全覆盖。模型只发送文本，用户配置声音和提供商。在消息平台上返回 `MEDIA:<path>` 标记，由发送管线拦截并作为原生语音消息投递。

### 3.2 提供商矩阵

| 提供商 | 类型 | API Key | 默认声音 | 默认模型 |
|--------|------|---------|----------|----------|
| Edge TTS | 云端(免费) | 无 | en-US-AriaNeural | — |
| ElevenLabs | 云端(付费) | ELEVENLABS_API_KEY | Adam | eleven_multilingual_v2 |
| OpenAI | 云端(付费) | OPENAI_API_KEY | alloy | gpt-4o-mini-tts |
| MiniMax | 云端(付费) | MINIMAX_API_KEY | female-shaonv | speech-01 |
| Mistral (Voxtral) | 云端(付费) | MISTRAL_API_KEY | Paul-Neutral | voxtral-mini-tts-2603 |
| Gemini | 云端(付费) | GEMINI_API_KEY | Kore | gemini-2.5-flash-preview-tts |
| xAI | 云端(付费) | XAI_API_KEY | eve | — |
| NeuTTS | 本地(免费) | 无 | — | espeak-ng |
| KittenTTS | 本地(免费) | 无 | Jasper | kitten-tts-nano-0.8-int8 (25MB) |
| Piper | 本地(免费) | 无 | en_US-lessac-medium | 44 语言 ONNX 模型 |
| Command | 自定义 | 无 | — | 用户 shell 命令模板 |

### 3.3 核心分发逻辑

源码位置: `tts_tool.py:1533-1732`

```python
def text_to_speech_tool(text: str, output_path: Optional[str] = None) -> str:
    provider = _get_provider(tts_config)

    # Command 提供商优先于内置 (防止 tts.providers.openai.command 覆盖真正的 OpenAI)
    command_provider_config = _resolve_command_provider_config(provider, tts_config)

    # 文本长度截断 — 每个 provider 有不同上限
    max_len = _resolve_max_text_length(provider, tts_config)
    if len(text) > max_len:
        text = text[:max_len]

    # 检测平台选择输出格式
    platform = get_session_env("HERMES_SESSION_PLATFORM", "").lower()
    want_opus = (platform == "telegram")  # Telegram 语音气泡需要 Opus

    # 分发到对应 provider
    if command_provider_config is not None:
        file_str = _generate_command_tts(text, file_str, provider, command_provider_config, tts_config)
    elif provider == "elevenlabs":
        _generate_elevenlabs(text, file_str, tts_config)
    elif provider == "openai":
        _generate_openai_tts(text, file_str, tts_config)
    # ... 更多 provider
    else:  # 默认: Edge TTS, 回退 NeuTTS
```

### 3.4 Gemini TTS — PCM 到 WAV 转换

源码位置: `tts_tool.py:1090-1224`

Gemini TTS API 返回原始 PCM(L16, 24kHz, 单声道)。`_generate_gemini_tts()` 将其包装为标准 WAV:

```python
GEMINI_TTS_SAMPLE_RATE = 24000
GEMINI_TTS_CHANNELS = 1
GEMINI_TTS_SAMPLE_WIDTH = 2  # 16-bit PCM (L16)

def _wrap_pcm_as_wav(pcm_data: bytes, ...) -> bytes:
    # 写入 RIFF/WAVE 头 + PCM 数据
```

### 3.5 Command TTS — 自定义 Shell 命令

源码位置: `tts_tool.py:367-678`

用户可在 `~/.hermes/config.yaml` 声明任意数量的命令提供商:

```yaml
tts:
  providers:
    my_local:
      type: command
      command: "piper --model {model} --output_file {output_path}"
```

Hermes 将文本写入临时文件，运行命令，命令必须将音频写入预期路径。支持超时、进程树终止和输出格式对齐。

### 3.6 流式播放

源码位置: `tts_tool.py:1910+`

`stream_tts_to_speaker()` 在 CLI 模式下边生成边播放，使用 `sounddevice` 的回调模式和线程队列。

---

## 4. vision_tools — 图像/视频分析

**源码位置:** `tools/vision_tools.py` (1168 行)

### 4.1 设计目标

接受 URL 或本地文件路径，下载图像，转换为 Base64，通过辅助视觉路由器调用 LLM 进行分析。内建 SSRF 防护、大小限制和自动缩放重试。

### 4.2 图像下载与安全

源码位置: `vision_tools.py:129-231`

```python
async def _download_image(image_url: str, destination: Path, max_retries: int = 3) -> Path:
    # SSRF 重定向防护 — 验证每个 302 目标
    async def _ssrf_redirect_guard(response):
        if response.is_redirect and response.next_request:
            redirect_url = str(response.next_request.url)
            if not is_safe_url(redirect_url):
                raise ValueError(f"Blocked redirect to private/internal address: {redirect_url}")

    async with httpx.AsyncClient(
        follow_redirects=True,
        event_hooks={"response": [_ssrf_redirect_guard]},
    ) as client:
        # Content-Length 预检: 50 MB 上限
        cl = response.headers.get("content-length")
        if cl and int(cl) > _VISION_MAX_DOWNLOAD_BYTES:
            raise ValueError(f"Image too large ({int(cl)} bytes)")
```

### 4.3 MIME 类型检测

源码位置: `vision_tools.py:107-126`

基于文件头魔术字节，不依赖扩展名:

```python
def _detect_image_mime_type(image_path: Path) -> Optional[str]:
    with image_path.open("rb") as f:
        header = f.read(64)
    if header.startswith(b"\x89PNG\r\n\x1a\n"): return "image/png"
    if header.startswith(b"\xff\xd8\xff"):       return "image/jpeg"
    if header.startswith((b"GIF87a", b"GIF89a")): return "image/gif"
    if len(header) >= 12 and header[:4] == b"RIFF" and header[8:12] == b"WEBP":
        return "image/webp"
    # SVG: 读取前 4KB 检查 <svg> 标签
```

### 4.4 自动缩放重试

源码位置: `vision_tools.py:406-598`

`vision_analyze_tool()` 先发送原始大小图像。如果 API 拒绝(图像过大错误)，自动调用 `_resize_image_for_vision()` 缩小到 ~5MB 并重试:

```python
try:
    response = await async_call_llm(**call_kwargs)
except Exception as _api_err:
    if _is_image_size_error(_api_err) and len(image_data_url) > _RESIZE_TARGET_BYTES:
        image_data_url = _resize_image_for_vision(temp_image_path, mime_type=detected_mime_type)
        messages[0]["content"][1]["image_url"]["url"] = image_data_url
        response = await async_call_llm(**call_kwargs)  # 重试
```

### 4.5 视频分析

源码位置: `vision_tools.py:915-1148`

`video_analyze_tool()` 支持视频文件分析，使用类似的 Base64 编码和辅助路由器模式。

---

## 5. file_tools — 文件操作

**源码位置:** `tools/file_tools.py` (1143 行)

### 5.1 设计目标

提供安全的文件读/写/补丁/搜索操作，内建多层防护防止模型意外损坏系统。

### 5.2 三层路径安全守卫

| 层 | 守卫 | 阻止目标 |
|----|------|----------|
| 1 | 设备路径 | `/dev/zero`, `/dev/random`, `/dev/stdin`, `/proc/*/fd/0-2` — 无限输出或阻塞 |
| 2 | 敏感路径写入 | `/etc/`, `/boot/`, `/usr/lib/systemd/`, `/var/run/docker.sock` — 系统文件 |
| 3 | 二进制扩展 | `.pyc`, `.so`, `.exe`, `.png` 等 — 对模型无意义的二进制内容 |

源码位置: `file_tools.py:69-174`

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

注意: 设备路径检查使用字面路径(不解析符号链接)，因为 `/dev/stdin` 通过符号链接解析为 `/dev/pts/0` 会绕过检查。

### 5.3 去重与循环检测

源码位置: `file_tools.py:447-646`

`read_file_tool()` 维护 per-task 的读追踪器:

- **去重**: 相同 (path, offset, limit) + 文件未修改 → 返回轻量存根而非重新发送内容
- **循环检测**: 连续 4 次读取同一区域 → 硬阻断返回错误
- **存根循环防护**: 2 次存根返回后升级为硬阻断
- **容量控制**: `_cap_read_tracker_data()` 防止长会话累积巨量追踪状态

```python
if count >= 4:
    return json.dumps({
        "error": f"BLOCKED: You have read this exact file region {count} times in a row. "
                 "STOP re-reading and proceed with your task.",
        "path": path, "already_read": count,
    })
```

### 5.4 字符数上限

源码位置: `file_tools.py:35-36, 547-569`

默认 100,000 字符(约 25-35K token)，可通过 `config.yaml` 的 `file_read_max_chars` 配置。超出时拒绝并引导模型使用 offset+limit。

### 5.5 写入时文件过时检测

源码位置: `file_tools.py:762-792`

`_check_file_staleness()` 在写入/补丁前检查: 如果文件在模型上次读取后被外部修改(其他代理、并发进程)，返回警告。通过 `file_state` 模块的跨代理注册表实现。

---

## 6. transcription_tools — 音频转写

**源码位置:** `tools/transcription_tools.py` (911 行)

### 6.1 设计目标

将语音消息转写为文本，由 Gateway 自动调用处理用户在 Telegram/Discord/WhatsApp/Slack/Signal 上发送的语音消息。默认使用本地 faster-whisper(免费)，支持 5 种云端提供商。

### 6.2 提供商矩阵

| 提供商 | 类型 | 需求 | 默认模型 |
|--------|------|------|----------|
| local (faster-whisper) | 本地(免费) | pip install faster-whisper | base (~150MB) |
| local_command | 本地 | 自定义命令模板 | — |
| groq | 云端(免费额度) | GROQ_API_KEY | whisper-large-v3-turbo |
| openai | 云端(付费) | VOICE_TOOLS_OPENAI_KEY | whisper-1 |
| mistral | 云端(付费) | MISTRAL_API_KEY | voxtral-mini-latest |
| xai | 云端(付费) | XAI_API_KEY | grok-stt (21 语言, ITN, 说话人分离) |

### 6.3 核心分发

源码位置: `transcription_tools.py:789-868`

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

### 6.4 模型名称归一化

源码位置: `transcription_tools.py:175-193`

云端模型名(如 `whisper-1`)对本地 faster-whisper 无效(需要 `tiny/base/small/medium/large-v*`)。`_normalize_local_model()` 自动映射并发出警告:

```python
def _normalize_local_model(model_name: Optional[str]) -> str:
    if not model_name or model_name in OPENAI_MODELS or model_name in GROQ_MODELS:
        if model_name and (model_name in OPENAI_MODELS or model_name in GROQ_MODELS):
            logger.warning("STT model '%s' is a cloud-only name, falling back to '%s'", ...)
        return DEFAULT_LOCAL_MODEL
    return model_name
```

### 6.5 音频验证

源码位置: `transcription_tools.py:298-350`

`_validate_audio_file()` 检查:
- 文件存在性和可读性
- 扩展名在支持集合中: `.mp3/.mp4/.mpeg/.mpga/.m4a/.wav/.webm/.ogg/.aac/.flac`
- 文件大小不超过 25 MB
- 非 CUDA 库错误(常见的本地 GPU 配置问题)

---

## 7. session_search_tool — 会话搜索

**源码位置:** `tools/session_search_tool.py` (605 行)

### 7.1 设计目标

搜索过去会话的文字记录，返回聚焦摘要而非原始文本，保持主模型的上下文窗口清洁。

### 7.2 两阶段流水线

```
阶段 1: FTS5 搜索
  db.search_messages(query, role_filter, limit=50)
    → 按相关性排序的消息列表
    → 按 session_id 分组，取前 N 个唯一会话(默认 3)
    → 解析子会话到父会话(delegation 链)
    → 排除当前会话(模型已有该上下文)

阶段 2: LLM 摘要
  对每个匹配会话:
    1. 加载完整对话 → _format_conversation()
    2. 截断到 ~100K 字符 → _truncate_around_matches()
    3. 发送到辅助模型 → async_call_llm(task="session_search")
    4. 返回结构化摘要
```

### 7.3 智能截断算法

源码位置: `session_search_tool.py:113-195`

`_truncate_around_matches()` 不是简单的前 N 字符截断，而是选择覆盖最多匹配位置的窗口:

1. **完整短语搜索**: 将整个查询作为短语查找
2. **邻近共现**: 所有查询词在 200 字符内同时出现的位置
3. **单词位置**: 各词的独立位置(最后手段)
4. **窗口选择**: 对每个候选位置计算窗口(25% 前 + 75% 后)覆盖的匹配数，选最优

```python
for candidate in match_positions:
    ws = max(0, candidate - max_chars // 4)  # 25% 前, 75% 后
    we = ws + max_chars
    count = sum(1 for p in match_positions if ws <= p < we)
    if count > best_count:
        best_count = count
        best_start = ws
```

### 7.4 并行并发控制

源码位置: `session_search_tool.py:454-477`

多个会话摘要并行执行，通过信号量控制并发(默认 3，最大 5):

```python
max_concurrency = min(_get_session_search_max_concurrency(), max(1, len(tasks)))
semaphore = asyncio.Semaphore(max_concurrency)

async def _bounded_summary(text, meta):
    async with semaphore:
        return await _summarize_session(text, query, meta)

coros = [_bounded_summary(text, meta) for _, _, text, meta in tasks]
results = await asyncio.gather(*coros, return_exceptions=True)
```

使用 `_run_async()` 而非 `asyncio.run()` 避免与缓存的 AsyncOpenAI/httpx 客户端的事件循环冲突。

### 7.5 子会话解析

源码位置: `session_search_tool.py:385-408`

压缩和委派创建子会话，但用户感知的是父对话。`_resolve_to_parent()` 沿 `parent_session_id` 链回溯到根会话，确保结果去重且排除当前活跃对话的所有分支。

---

## 8. homeassistant_tool — 智能家居控制

**源码位置:** `tools/homeassistant_tool.py` (513 行)

### 8.1 设计目标

通过 Home Assistant REST API 控制智能家居设备，提供 4 个工具: 实体列表、状态查询、服务列表、服务调用。

### 8.2 工具矩阵

| 工具 | Schema | 功能 | HA API 端点 |
|------|--------|------|-------------|
| `ha_list_entities` | LIST_ENTITIES_SCHEMA | 列出/过滤实体 | GET /api/states |
| `ha_get_state` | GET_STATE_SCHEMA | 查询单实体详情 | GET /api/states/{entity_id} |
| `ha_list_services` | LIST_SERVICES_SCHEMA | 列出可用服务 | GET /api/services |
| `ha_call_service` | CALL_SERVICE_SCHEMA | 调用服务控制设备 | POST /api/services/{domain}/{service} |

### 8.3 安全: 服务域黑名单

源码位置: `homeassistant_tool.py:53-60`

HA 不提供服务级访问控制——所有安全层必须在我们侧实现:

```python
_BLOCKED_DOMAINS = frozenset({
    "shell_command",    # 任意 shell 命令 (HA 容器内 root)
    "command_line",     # 执行 shell 命令的传感器/开关
    "python_script",    # 沙箱但可通过 hass.services.call() 提权
    "pyscript",         # 更广泛的脚本集成
    "hassio",           # 插件控制、主机关机/重启、容器 stdin
    "rest_command",     # 从 HA 服务器发 HTTP 请求 (SSRF 向量)
})
```

### 8.4 路径遍历防护

源码位置: `homeassistant_tool.py:38-48`

域和服务名被直接插入 URL `/api/services/{domain}/{service}`。正则验证只允许小写 ASCII 字母+数字+下划线，阻止 `../../api/config` 类路径遍历:

```python
_ENTITY_ID_RE = re.compile(r"^[a-z_][a-z0-9_]*\.[a-z0-9_]+$")
_SERVICE_NAME_RE = re.compile(r"^[a-z][a-z0-9_]*$")
```

### 8.5 实体过滤

源码位置: `homeassistant_tool.py:77-102`

`_filter_and_summarize()` 支持按域(`light.`/`switch.`/`climate.`)和区域名称(模糊匹配 friendly_name 和 area 属性)过滤，返回紧凑摘要而非完整状态。

---

## 辅助工具简述

### 9. code_execution_tool — 代码执行沙箱

**源码位置:** `tools/code_execution_tool.py` (1621 行)

让 LLM 编写 Python 脚本，通过 RPC 调用 Hermes 工具，将多步工具链折叠为单次推理。

双传输架构:
- **本地(UDS)**: Unix Domain Socket + RPC 监听线程
- **远程(文件 RPC)**: 请求/响应文件 + 轮询线程，支持 Docker/SSH/Modal/Daytona

7 个允许的沙箱工具: `web_search`, `web_extract`, `read_file`, `write_file`, `search_files`, `patch`, `terminal`

资源限制: 默认 300s 超时、50 次工具调用、50KB stdout / 10KB stderr。

### 10. image_generation_tool — 图像生成

**源码位置:** `tools/image_generation_tool.py` (1002 行)

通过 FAL.ai 生成图像。`FAL_MODELS` 目录声明每个模型的元数据(尺寸族、默认值、supports 白名单、upscaler 标志)。`_build_fal_payload()` 将统一输入(prompt + aspect_ratio)翻译为模型特定负载，通过 `supports` 白名单过滤不支持的参数。可选 Clarity Upscaler 后处理。

### 11. voice_mode — 语音模式

**源码位置:** `tools/voice_mode.py` (1017 行)

CLI 推送录音模式。`AudioRecorder`(桌面)和 `TermuxAudioRecorder`(Android)两种实现。录音后通过 `transcription_tools.transcribe_audio()` 转写。`is_whisper_hallucination()` 过滤 Whisper 常见幻觉输出("谢谢观看"、"字幕由..."等)。

### 12. kanban_tools — 看板协作

**源码位置:** `tools/kanban_tools.py` (855 行)

7 个工具: `kanban_show`, `kanban_complete`, `kanban_block`, `kanban_heartbeat`, `kanban_comment`, `kanban_create`, `kanban_link`。仅在 `HERMES_KANBAN_TASK` 环境变量设置(调度器派生的工作代理)或 profile 包含 `kanban` 工具集时注册。工具直接访问 `~/.hermes/kanban.db`，绕过终端后端的 shell 引号问题。

### 13. mixture_of_agents_tool — 混合代理

**源码位置:** `tools/mixture_of_agents_tool.py` (541 行)

基于 Wang et al. (arXiv:2406.04692v1) 的 Mixture-of-Agents 方法:

- **Layer 1**: 4 个参考模型(Claude Opus 4.6 / Gemini 2.5 Pro / GPT-5.4 Pro / DeepSeek V3.2)并行生成初始响应(temp=0.6)
- **Layer 2**: 聚合模型(Claude Opus 4.6)合成最终响应(temp=0.4)
- 容错: 最少 1 个参考模型成功即可继续，6 次重试

### 14. memory_tool — 持久记忆

**源码位置:** `tools/memory_tool.py` (586 行)

双存储: `MEMORY.md`(代理笔记) + `USER.md`(用户偏好)。关键设计:

- **冻结快照**: 系统提示中的记忆在会话开始时冻结，保持前缀缓存稳定
- **即时持久化**: 工具调用立即写盘(原子替换 `os.replace()`)
- **注入扫描**: `_scan_memory_content()` 检测提示注入、角色劫持、密钥外泄等威胁模式
- **不可见字符检测**: 拦截零宽字符(U+200B 等)和双向控制字符(U+202A-U+202E)
- **字符限制**: MEMORY 2200 字符、USER 1375 字符(模型无关)

---

## 比较表

### 表 1: 消息平台支持矩阵

| 平台 | 目标解析 | 媒体附件 | 消息分块 | 格式转换 | 特殊处理 |
|------|----------|----------|----------|----------|----------|
| Telegram | 正则+话题 | photo/video/voice/audio/doc | UTF-16 长度 | MarkdownV2/HTML 自动切换 | 429 重试、话题线程 |
| Discord | 正则+话题 | multipart 上传 | 2000 字符 | — | 论坛频道(类型 15)自动创建帖子 |
| Slack | [CGD] 正则 | 不支持 | 40000 字符 | mrkdwn 格式化 | — |
| Signal | E.164 电话 | JSON-RPC 附件 | — | — | 速率限制调度器 |
| WhatsApp | E.164 电话 | 不支持 | — | — | 本地桥接 HTTP API |
| WeChat | 微信 ID | 原生适配 | — | — | .env 合成配置 |
| Matrix | !/@ ID | 原生适配 | — | — | — |
| Feishu | 飞书 ID | 原生适配 | — | — | — |
| Email | — | 不支持 | — | — | SMTP 发送 |

### 表 2: TTS 提供商对比

| 提供商 | 延迟 | 成本 | 输出格式 | 需安装 | 声音选择 |
|--------|------|------|----------|--------|----------|
| Edge TTS | 中 | 免费 | MP3 | pip install edge-tts | 400+ 神经声音 |
| ElevenLabs | 低 | 付费 | MP3/Opus | pip install elevenlabs | 高质量+流式 |
| OpenAI | 低 | 付费 | Opus/MP3 | pip install openai | 6 预设声音 |
| MiniMax | 低 | 付费 | MP3 | — | 声音克隆 |
| Mistral | 低 | 付费 | Opus | pip install mistralai | 多语言 |
| Gemini | 中 | 付费 | WAV(PCM) | — | 30 预设声音 |
| xAI | 低 | 付费 | MP3 | — | Grok 声音 |
| NeuTTS | 快 | 免费 | WAV | espeak-ng + neutts | — |
| KittenTTS | 快 | 免费 | WAV | 25MB 模型 | 1 声音 |
| Piper | 最快 | 免费 | WAV | pip install piper-tts | 44 语言 |
| Command | 变化 | 变化 | 可配置 | — | 可配置 |

### 表 3: Web 搜索后端对比

| 后端 | 搜索 | 提取 | 爬取 | API Key | 特点 |
|------|------|------|------|---------|------|
| Firecrawl | yes | yes | yes | FIRECRAWL_API_KEY | 高质量、Nous 网关支持 |
| Exa | yes | yes | no | EXA_API_KEY | 语义搜索 |
| Parallel | yes | yes | no | PARALLEL_API_KEY | — |
| Tavily | yes | yes | yes | TAVILY_API_KEY | 含 failed_results 归一化 |
| SearXNG | yes | no | no | SEARXNG_URL (自托管) | 完全私有、无 API Key |

### 表 4: STT 提供商对比

| 提供商 | 延迟 | 成本 | 模型 | 特殊能力 |
|--------|------|------|------|----------|
| local (faster-whisper) | 中 | 免费 | base→large-v3 | 离线、GPU 可选 |
| local_command | 变化 | 免费 | 自定义 | 灵活 |
| Groq | 极低 | 免费额度 | whisper-large-v3-turbo | — |
| OpenAI | 低 | 付费 | whisper-1 / gpt-4o-transcribe | — |
| Mistral | 低 | 付费 | voxtral-mini-latest | 多语言 |
| xAI | 低 | 付费 | grok-stt | 21 语言、ITN、说话人分离 |

---

## 汇总表

| 组件名 | 行数 | 职责 |
|--------|------|------|
| send_message_tool | 1780 | 跨 15+ 平台消息路由，目标解析，媒体附件，消息分块，Cron 去重 |
| web_tools | 2223 | 5 后端 Web 搜索/提取，LLM 摘要，URL 安全检查，Firecrawl 懒加载 |
| tts_tool | 2185 | 11+ TTS 提供商，Opus/MP3 输出，Command 自定义，流式播放 |
| vision_tools | 1168 | 图像/视频分析，SSRF 防护，自动缩放重试，辅助视觉路由 |
| file_tools | 1143 | 文件读写/补丁/搜索，三层路径安全守卫，去重循环检测，100K 字符上限 |
| transcription_tools | 911 | 6 STT 提供商，音频验证，模型名归一化，CUDA 错误检测 |
| session_search_tool | 605 | FTS5 全文搜索，智能截断，并行 LLM 摘要，子会话解析 |
| homeassistant_tool | 513 | HA REST API 4 工具，域黑名单，路径遍历防护，实体过滤 |
| code_execution_tool | 1621 | UDS/文件双传输 RPC，hermes_tools.py 存根，7 沙箱工具 |
| image_generation_tool | 1002 | FAL.ai 多模型目录，supports 白名单，Clarity Upscaler |
| voice_mode | 1017 | 推送录音(桌面/Termux)，Whisper 幻觉过滤，TTS 播放 |
| kanban_tools | 855 | 7 看板工具，调度器/编排器双门控，直接 DB 访问 |
| mixture_of_agents_tool | 541 | 4 参考模型并行 + 聚合合成，6 次重试容错 |
| memory_tool | 586 | 双存储 MEMORY/USER，冻结快照，注入扫描，原子替换 |

---

[← 05 — 工具体系](/zh-CN/chapters/05-tool-system) | [19 — Hermes CLI 子系统 →](/zh-CN/chapters/19-hermes-cli-subsystem)
