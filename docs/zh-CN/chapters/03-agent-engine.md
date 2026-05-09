# 03 — 核心代理引擎 AIAgent: 对话循环、流式传输、故障转移与回调系统

[上一章](/zh-CN/chapters/02-architecture) | [返回目录](/zh-CN/chapters/overview)

---

## 源码位置与关键文件

| 文件 | 行数 | 关键区域 |
|------|------|----------|
| `run_agent.py` | 14,404 | AIAgent 类 (L885-14,404), ~13,500 行 |
| `run_agent.py` | — | IterationBudget (L272-311) |
| `run_agent.py` | — | `__init__` (L908-2,228) |
| `run_agent.py` | — | `run_conversation` (L10,573-14,404) |
| `run_agent.py` | — | `_interruptible_api_call` (L6,460-6,700) |
| `run_agent.py` | — | `_try_activate_fallback` (L7,633-7,700) |
| `run_agent.py` | — | `_restore_primary_runtime` (L7,831-7,900) |
| `run_agent.py` | — | `_compress_context` (L9,221-9,400) |
| `run_agent.py` | — | `_run_tool` (L9,716-9,800) |

**一句话概要:** AIAgent 是一个 14,404 行的单文件智能体引擎,包含完整对话循环、11 种回调类型、有序故障转移链、turn-scoped 主运行时恢复、阈值触发上下文压缩,以及基于 ThreadPoolExecutor 的并发工具执行。

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AIAgent Engine                                  │
│                    run_agent.py (14,404 行)                              │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                     AIAgent.__init__ (L908-2,228)                  │ │
│  │                                                                    │ │
│  │  推理配置: base_url, api_key, provider, api_mode, model           │ │
│  │  迭代控制: max_iterations (90), iteration_budget, tool_delay      │ │
│  │  11 回调:  tool_progress, tool_start, tool_complete,              │ │
│  │           thinking, reasoning, clarify, step,                     │ │
│  │           stream_delta, interim_assistant, tool_gen, status        │ │
│  │  故障转移: fallback_model, credential_pool                         │ │
│  │  安全:     checkpoints_enabled, pass_session_id                   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │              run_conversation (L10,573-14,404)                     │ │
│  │                                                                    │ │
│  │  ┌─────────────────┐  ┌──────────────────────────────────────┐   │ │
│  │  │ Pre-turn 复位   │  │    主循环 (while iteration)          │   │ │
│  │  │                 │  │                                    │   │ │
│  │  │ • 重置 retry    │  │  ┌──────────┐                      │   │ │
│  │  │ • restore prim  │  │  │ API Call │                      │   │ │
│  │  │ • sanitize      │  │  │ (interruptible)                  │   │ │
│  │  │ • health check  │  │  └──────────┘                      │   │ │
│  │  │ • log context   │  │       │                             │   │ │
│  │  └─────────────────┘  │       ▼                             │   │ │
│  │                       │  ┌──────────────────────────┐      │   │ │
│  │                       │  │ 响应解析                  │      │   │ │
│  │                       │  │  • tool_calls?            │      │   │ │
│  │                       │  │  • empty? (retry nudge)   │      │   │ │
│  │                       │  │  • thinking-only?         │      │   │ │
│  │                       │  │  • stream deltas          │      │   │ │
│  │                       │  └──────────────────────────┘      │   │ │
│  │                       │       │                             │   │ │
│  │                       │  ┌────┼────────────────────┐      │   │ │
│  │                       │  │    ▼                    ▼      │   │ │
│  │                       │  │  Tool Calls          Text Resp │   │ │
│  │                       │  │  (ThreadPool)       (final)    │   │ │
│  │                       │  │    │                    │      │   │ │
│  │                       │  │    ▼                    │      │   │ │
│  │                       │  │  _run_tool()          │      │   │ │
│  │                       │  │  • fan-out interrupt  │      │   │ │
│  │                       │  │  • approval callback  │      │   │ │
│  │                       │  │  • _invoke_tool       │      │   │ │
│  │                       │  │  • guardrails check   │      │   │ │
│  │                       │  └──────────────────────────────┘   │ │ │
│  │                       │       │                             │   │ │
│  │                       │       ▼                             │   │ │
│  │                       │  ┌──────────────────────────┐      │   │ │
│  │                       │  │ Post-iteration checks    │      │   │ │
│  │                       │  │  • context pressure?     │      │   │ │
│  │                       │  │  • compress_context()    │      │   │ │
│  │                       │  │  • memory nudge?         │      │   │ │
│  │                       │  │  • skill nudge?          │      │   │ │
│  │                       │  │  • background review?    │      │   │ │
│  │                       │  └──────────────────────────┘      │   │ │
│  │                       │       │                             │   │ │
│  │                       │       ▼ continue?                  │   │ │
│  │                       └────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌─────────────────┐ ┌─────────────────┐ ┌────────────────────────────┐ │
│  │ Failover Chain  │ │ IterationBudget │ │ Compressor                │ │
│  │ (L7,633+)      │ │ (L272+)         │ │ (L9,221+)                │ │
│  │                 │ │                 │ │                            │ │
│  │ primary → fb1  │ │ max_total: 90   │ │ threshold trigger         │ │
│  │ → fb2 → fb3   │ │ thread-safe     │ │ focus_topic guided        │ │
│  │ → exhausted   │ │ consume/refund  │ │ SQLite session split      │ │
│  │                 │ │                 │ │ auxiliary model fallback  │ │
│  └─────────────────┘ └─────────────────┘ └────────────────────────────┘ │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                     Streaming Pipeline                             │ │
│  │                                                                    │ │
│  │  stream_delta_callback ──► UI/TTS incremental text               │ │
│  │  _stream_callback ──► TTS pre-generation audio                    │ │
│  │  thinking_callback ──► reasoning preview box                      │ │
│  │  reasoning_callback ──► extended reasoning display                │ │
│  │  tool_gen_callback ──► "preparing terminal..." progress           │ │
│  │  status_callback ──► lifecycle events (retry/fallback/compress)   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## TL;DR

AIAgent 类 (14,404 行) 是 Hermes Agent 的全部智能体逻辑。对话循环 `run_conversation` 在每个 turn 开始时重置所有 retry 计数器和恢复主提供者,然后在 while 循环中:调用 `_interruptible_api_call` (后台线程,支持中途中断),解析响应 (tool_calls/空响应/thinking-only),并发执行工具 (`_run_tool` ThreadPoolExecutor worker),检查上下文压力和压缩触发。故障转移通过有序链 `_try_activate_fallback` 在同一 AIAgent 实例上原地替换 client/model/provider,下一 turn 自动恢复主运行时。11 种回调覆盖从 streaming delta 到 lifecycle event 的全部通知路径。IterationBudget 使用线程安全 Lock 控制迭代上限 (父 90,子 50)。

---

## 1. AIAgent 类定义

### 1.1 类结构概览

AIAgent 类从 L885 开始,是整个 `run_agent.py` 的核心。类定义包含一个 corruption marker 常量、一个 `base_url` property (setter 自动维护 `_base_url_lower` 和 `_base_url_hostname`)、50+ 参数的 `__init__` 方法,以及约 50+ 个方法覆盖对话循环、流式传输、故障转移、压缩、工具调度等全部智能体行为。

```python
# 源码位置: run_agent.py:885-896

class AIAgent:
    """
    AI Agent with tool calling capabilities.

    This class manages the conversation flow, tool execution, and response handling
    for AI models that support function calling.
    """

    _TOOL_CALL_ARGUMENTS_CORRUPTION_MARKER = (
        "[hermes-agent: tool call arguments were corrupted in this session and "
        "have been dropped to keep the conversation alive. See issue #15236.]"
    )
```

`_TOOL_CALL_ARGUMENTS_CORRUPTION_MARKER` 是一个安全标记:当 tool call arguments 在会话中被损坏时,用此标记替代损坏的参数以保持对话存活,而不是让整个会话崩溃。

### 1.2 __init__ 参数体系

`__init__` 接收超过 50 个参数,按功能分组:

**推理配置 (Inference):**

```python
# 源码位置: run_agent.py:910-918
base_url: str = None,
api_key: str = None,
provider: str = None,
api_mode: str = None,        # "chat_completions" or "codex_responses"
acp_command: str = None,
model: str = "",              # 默认 "anthropic/claude-opus-4.6"
max_tokens: int = None,
reasoning_config: Dict[str, Any] = None,
service_tier: str = None,
```

**迭代控制 (Iteration):**

```python
# 源码位置: run_agent.py:919-920
max_iterations: int = 90,    # 默认工具调用迭代上限 (共享)
tool_delay: float = 1.0,     # 工具调用间延迟
iteration_budget: "IterationBudget" = None,  # 父创建,子继承
```

**工具配置 (Tools):**

```python
# 源码位置: run_agent.py:921-922
enabled_toolsets: List[str] = None,
disabled_toolsets: List[str] = None,
```

**11 种回调 (Callbacks):**

```python
# 源码位置: run_agent.py:936-946
tool_progress_callback: callable = None,   # (tool_name, args_preview)
tool_start_callback: callable = None,      # 工具开始通知
tool_complete_callback: callable = None,   # 工具完成通知
thinking_callback: callable = None,        # 思考块预览
reasoning_callback: callable = None,       # 扩展推理显示
clarify_callback: callable = None,         # (question, choices) → str
step_callback: callable = None,            # 步骤通知
stream_delta_callback: callable = None,    # 流式增量文本
interim_assistant_callback: callable = None, # 中间助手响应
tool_gen_callback: callable = None,        # 工具参数生成进度
status_callback: callable = None,          # lifecycle events
```

**平台上下文 (Platform Context):**

```python
# 源码位置: run_agent.py:952-959
platform: str = None,          # "cli", "telegram", "discord", etc.
user_id: str = None,
user_name: str = None,
chat_id: str = None,
chat_name: str = None,
chat_type: str = None,
thread_id: str = None,
gateway_session_key: str = None,
```

**故障转移 (Failover):**

```python
# 源码位置: run_agent.py:966-967
fallback_model: Dict[str, Any] = None,
credential_pool=None,
```

**安全 (Security):**

```python
# 源码位置: run_agent.py:968-972
checkpoints_enabled: bool = False,
checkpoint_max_snapshots: int = 20,
checkpoint_max_total_size_mb: int = 500,
checkpoint_max_file_size_mb: int = 10,
pass_session_id: bool = False,
```

---

## 2. 对话循环: run_conversation

### 2.1 方法签名与核心职责

```python
# 源码位置: run_agent.py:10573-10600

def run_conversation(
    self,
    user_message: str,
    system_message: str = None,
    conversation_history: List[Dict[str, Any]] = None,
    task_id: str = None,
    stream_callback: Optional[callable] = None,
    persist_user_message: Optional[str] = None,
) -> Dict[str, Any]:
    """
    Run a complete conversation with tool calling until completion.

    Args:
        user_message (str): The user's message/question
        system_message (str): Custom system message
        conversation_history (List[Dict]): Previous conversation messages
        task_id (str): Unique identifier for this task
        stream_callback: Optional callback invoked with each text delta
        persist_user_message: Optional clean user message to store in transcripts

    Returns:
        Dict: Complete conversation result with final response and message history
    """
```

`run_conversation` 是 AIAgent 的主入口方法,接收用户消息后执行完整的工具调用循环直到模型返回纯文本响应或迭代预算耗尽。

### 2.2 Pre-turn 复位 (Turn-scoped 状态管理)

每个 turn 开始时,`run_conversation` 执行一系列关键复位操作:

```python
# 源码位置: run_agent.py:10601-10684

# Guard stdio against OSError from broken pipes
_install_safe_stdio()

self._ensure_db_session()

# Tag all log records with session ID
from hermes_logging import set_session_context
set_session_context(self.session_id)

# Bind skill write-origin ContextVar
from tools.skill_provenance import set_current_write_origin
set_current_write_origin(getattr(self, "_memory_write_origin", "assistant_tool"))

# Restore primary runtime (turn-scoped fallback restoration)
self._restore_primary_runtime()

# Sanitize surrogate characters from user input
if isinstance(user_message, str):
    user_message = _sanitize_surrogates(user_message)

# Store stream callback for _interruptible_api_call to pick up
self._stream_callback = stream_callback

# Reset retry counters
self._invalid_tool_retries = 0
self._invalid_json_retries = 0
self._empty_content_retries = 0
self._incomplete_scratchpad_retries = 0
self._codex_incomplete_retries = 0
self._thinking_prefill_retries = 0
self._post_tool_empty_retried = False
self._unicode_sanitization_passes = 0
self._tool_guardrails.reset_for_turn()

# Pre-turn connection health check
if self.api_mode != "anthropic_messages":
    try:
        if self._cleanup_dead_connections():
            self._emit_status("Detected stale connections — cleaned up.")
    except Exception:
        pass

# Reset iteration budget for this turn
self.iteration_budget = IterationBudget(self.max_iterations)
```

这些复位确保:
- **turn-scoped 故障转移**: 上一 turn 的 fallback 不影响当前 turn
- **retry 计数器隔离**: 每个 turn 重新开始 retry,不累积历史负担
- **死连接清理**: 检测并关闭上次 provider outage 留下的僵尸 TCP 连接
- **surrogate 消除**: 防止 Google Docs/Word 剪贴板粘贴的孤 surrogate 字符导致 JSON 序列化崩溃
- **迭代预算重置**: 每个 turn 获得全新的 IterationBudget

### 2.3 主循环结构

主循环的核心逻辑 (简化):

```text
while iteration_budget.consume():
    1. 构建 API 调用参数 (messages, tools, system prompt)
    2. _interruptible_api_call(api_kwargs)
       → 后台线程执行,主线程可检测 interrupt
    3. 解析响应:
       a. tool_calls? → 并发执行 (_run_tool workers)
       b. empty response? → retry with nudge (3 attempts)
       c. thinking-only? → accept as "(empty)" or retry
       d. normal text → final response, break loop
    4. Post-iteration:
       a. context pressure check → compress if threshold exceeded
       b. memory nudge check → suggest saving memories
       c. skill nudge check → suggest creating skills
       d. background review fork → self-improvement loop
    5. Continue or break
```

---

## 3. 流式传输机制

### 3.1 双通道流式架构

AIAgent 使用双通道流式传输架构:

| 通道 | 回调 | 目的 | 消费者 |
|------|------|------|--------|
| Display | `stream_delta_callback` | 增量文本显示 | CLI TUI, Gateway platforms |
| Audio | `_stream_callback` | TTS 预生成音频 | TTS pipeline |

```python
# 源码位置: run_agent.py:6632-6646

# Streaming delta callbacks (双通道)
callbacks = [cb for cb in (self.stream_delta_callback, self._stream_callback) if cb is not None]
```

两个回调在流式消费中同时触发,`stream_delta_callback` 负责增量文本显示 (TUI/平台消息),`_stream_callback` 负责提前启动 TTS 音频生成。

### 3.2 流式消费流程

```text
SSE 流 ──► 逐 chunk 解析
             │
     ┌───────┼────────────────┐
     │       │                │
     ▼       ▼                ▼
  文本 delta  reasoning delta  tool_call delta
     │       │                │
     ▼       ▼                ▼
  stream_delta thinking_     tool_gen_
  callback     callback     callback
     │       │                │
     ▼       ▼                ▼
  TUI/平台   推理预览框       "preparing..." 提示
  更新       渲染             工具参数生成进度
```

关键设计:
- **reasoning preview buffering** (v0.5: #3013) — 推理预览块缓冲以防止重复显示
- **reasoning box anti-triple-render** (v0.5: #3405) — 防止推理框在 tool-calling 循环中渲染 3 次
- **thinking/reasoning 分离** — `<think>` 块和 `reasoning_content` 分开处理和回调
- **tool_gen_callback** — 流式显示工具参数生成进度 ("preparing terminal...")

---

## 4. 故障转移逻辑

### 4.1 有序回退提供者链

Hermes Agent 实现有序回退提供者链:当主提供者持续失败时,在同一 AIAgent 实例上原地替换 client, model, provider, api_mode 等运行时状态,使 retry 循环可以继续使用新后端。

```python
# 源码位置: run_agent.py:7633-7644

def _try_activate_fallback(self, reason: "FailoverReason | None" = None) -> bool:
    """Switch to the next fallback model/provider in the chain.

    Called when the current model is failing after retries.  Swaps the
    OpenAI client, model slug, and provider in-place so the retry loop
    can continue with the new backend.  Advances through the chain on
    each call; returns False when exhausted.

    Uses the centralized provider router (resolve_provider_client) for
    auth resolution and client construction — no duplicated provider→key
    mappings.
    """
```

回退链的激活流程:

```text
主提供者失败 ──► retry with backoff ──► 仍然失败
       │
       ▼
_try_activate_fallback(reason)
       │
       ├─ reason=rate_limit/billing → 设置 60s cooldown
       │
       ▼
检查 _fallback_index < len(_fallback_chain)
       │
       ▼
取 fb = _fallback_chain[_fallback_index]
       │
       ▼
resolve_provider_client(fb_provider, fb_model)
       │
       ├─ client=None → skip, try next in chain
       │
       ▼
原地替换:
  self.model = fb_model
  self.provider = fb_provider
  self.client = fb_client
  self.api_mode = fb_api_mode
  self.base_url = fb_base_url
       │
       ▼
标记 _fallback_activated = True
保存 _primary_runtime (当前运行时快照)
       │
       ▼
retry loop 继续使用新后端
```

**Ollama Cloud 安全**: 对于 Ollama Cloud 端点,回退配置没有显式 key 时从环境变量取 `OLLAMA_API_KEY`,使用 host 匹配 (非 substring) 防止安全绕过 (GHSA-76xc-57q6-vm5m)。

```python
# 源码位置: run_agent.py:7681-7682

if fb_base_url_hint and base_url_host_matches(fb_base_url_hint, "ollama.com") and not fb_api_key_hint:
    fb_api_key_hint = os.getenv("OLLAMA_API_KEY") or None
```

### 4.2 Turn-scoped 主运行时恢复

关键设计:故障转移是 **turn-scoped** 的,不是 session-scoped 的。下一个 `run_conversation` 调用时自动恢复主提供者:

```python
# 源码位置: run_agent.py:7831-7843

def _restore_primary_runtime(self) -> bool:
    """Restore the primary runtime at the start of a new turn.

    In long-lived CLI sessions a single AIAgent instance spans multiple
    turns.  Without restoration, one transient failure pins the session
    to the fallback provider for every subsequent turn.  Calling this at
    the top of ``run_conversation()`` makes fallback turn-scoped.

    The gateway caches agents across messages (``_agent_cache`` in
    ``gateway/run.py``), so this restoration IS needed there too.
    """
    if not self._fallback_activated:
        return False

    if getattr(self, "_rate_limited_until", 0) > time.monotonic():
        return False  # primary still in rate-limit cooldown
```

恢复操作包括:
- 核心运行时状态: `model`, `provider`, `base_url`, `api_mode`, `api_key`, `_client_kwargs`
- Transport 缓存清空: `_transport_cache.clear()`
- Prompt caching 状态: `_use_prompt_caching`, `_use_native_cache_layout`
- 客户端重建: Anthropic 用 `build_anthropic_client`, OpenAI-compatible 用 `_create_openai_client`
- 上下文压缩器状态恢复: `context_compressor.update_model()`

```python
# 源码位置: run_agent.py:7850-7866

rt = self._primary_runtime
try:
    # ── Core runtime state ──
    self.model = rt["model"]
    self.provider = rt["provider"]
    self.base_url = rt["base_url"]           # setter updates _base_url_lower
    self.api_mode = rt["api_mode"]
    if hasattr(self, "_transport_cache"):
        self._transport_cache.clear()
    self.api_key = rt["api_key"]
    self._client_kwargs = dict(rt["client_kwargs"])
    self._use_prompt_caching = rt["use_prompt_caching"]
    self._use_native_cache_layout = rt.get(
        "use_native_cache_layout",
        self.api_mode == "anthropic_messages" and self.provider == "anthropic",
    )
```

---

## 5. 回调系统

### 5.1 11 种回调类型

AIAgent 支持 11 种回调,覆盖从流式增量到 lifecycle 事件的全部通知路径:

| 回调 | 签名 | 触发时机 | 消费者 |
|------|------|----------|--------|
| `tool_progress_callback` | `(tool_name, args_preview)` | 工具开始执行 | CLI spinner, gateway progress |
| `tool_start_callback` | `(tool_name, args)` | 工具调用开始 | TUI widget |
| `tool_complete_callback` | `(tool_name, result)` | 工具调用完成 | TUI widget |
| `thinking_callback` | `(thinking_text)` | `<think>` 块增量 | CLI reasoning box |
| `reasoning_callback` | `(reasoning_content)` | `reasoning_content` 增量 | CLI reasoning display |
| `clarify_callback` | `(question, choices) → str` | 用户交互澄清 | CLI input, gateway buttons |
| `step_callback` | `(step_info)` | 对话步骤通知 | observability |
| `stream_delta_callback` | `(delta_text)` | 流式文本增量 | TUI, platform messages |
| `interim_assistant_callback` | `(partial_response)` | 中间助手响应 | TTS preview |
| `tool_gen_callback` | `(progress_text)` | 工具参数生成进度 | "preparing terminal..." |
| `status_callback` | `(status_message)` | lifecycle events | retry/fallback/compress 通知 |

源码位置: `run_agent.py:1157-1168` (回调赋值到 self)

### 5.2 回调在引擎中的注入点

```text
run_conversation 主循环:
  │
  ├── Pre-turn: status_callback ("Detected stale connections...")
  │
  ├── API Call:
  │   ├── stream_delta_callback (每个文本 delta)
  │   ├── _stream_callback (每个文本 delta, 用于 TTS)
  │   ├── thinking_callback (每个 thinking delta)
  │   ├── reasoning_callback (每个 reasoning delta)
  │   ├── tool_gen_callback (工具参数生成 delta)
  │   ├── status_callback (retry/fallback/compress events)
  │   └── interim_assistant_callback (中间响应)
  │
  ├── Tool Execution:
  │   ├── tool_progress_callback (工具开始)
  │   ├── tool_start_callback (工具调用开始)
  │   ├── tool_complete_callback (工具调用完成)
  │   └── clarify_callback (交互澄清)
  │
  ├── Post-iteration:
  │   ├── step_callback (步骤完成)
  │   └── status_callback (compress/fallback 通知)
```

---

## 6. IterationBudget 与迭代控制

### 6.1 IterationBudget 类

IterationBudget (L272) 是线程安全的迭代计数器,控制智能体循环的最大迭代次数:

```python
# 源码位置: run_agent.py:272-310

class IterationBudget:
    """Thread-safe iteration counter for an agent.

    Each agent (parent or subagent) gets its own ``IterationBudget``.
    The parent's budget is capped at ``max_iterations`` (default 90).
    Each subagent gets an independent budget capped at
    ``delegation.max_iterations`` (default 50) — this means total
    iterations across parent + subagents can exceed the parent's cap.

    ``execute_code`` (programmatic tool calling) iterations are refunded via
    :meth:`refund` so they don't eat into the budget.
    """

    def __init__(self, max_total: int):
        self.max_total = max_total
        self._used = 0
        self._lock = threading.Lock()

    def consume(self) -> bool:
        """Try to consume one iteration.  Returns True if allowed."""
        with self._lock:
            if self._used >= self.max_total:
                return False
            self._used += 1
            return True

    def refund(self) -> None:
        """Give back one iteration (e.g. for execute_code turns)."""
        with self._lock:
            if self._used > 0:
                self._used -= 1

    @property
    def used(self) -> int:
        with self._lock:
            return self._used
```

关键设计:
- **子智能体独立预算**: 子智能体获得独立预算 (default 50),不共享父的 90 上限 (v0.5: #3004 修复了共享预算导致子智能体过早耗尽的问题)
- **execute_code 退款**: execute_code 的程序化工具调用迭代通过 `refund()` 返还,不吞噬预算
- **线程安全**: 使用 `threading.Lock` 保护 `_used` 计数

### 6.2 迭代控制演进

| 版本 | 变化 | 问题修复 |
|------|------|----------|
| v0.2 | 共享 parent+subagent 预算 | — |
| v0.2 | Iteration budget pressure via tool result injection | — |
| v0.5 | 子智能体独立预算 (delegation.max_iterations) | 子智能体过早耗尽 (#3004) |
| v0.7 | 重置 retry 计数器在 run_conversation 开始 | 防止跨 turn 累积 (#607) |
| v0.8 | Jittered retry backoff | — |
| v0.12 | IterationBudget 在每个 turn 重置 | 确保每个 turn 有完整预算 |

---

## 7. 上下文压缩触发

### 7.1 _compress_context 方法

当上下文 token 数超过阈值时,`_compress_context` 自动压缩对话历史并分割会话:

```python
# 源码位置: run_agent.py:9221-9238

def _compress_context(self, messages: list, system_message: str, *, approx_tokens: int = None, task_id: str = "default", focus_topic: str = None) -> tuple:
    """Compress conversation context and split the session in SQLite.

    Args:
        focus_topic: Optional focus string for guided compression — the
            summariser will prioritise preserving information related to
            this topic.  Inspired by Claude Code's ``/compact <focus>``.

    Returns:
        (compressed_messages, new_system_prompt) tuple
    """
    _pre_msg_count = len(messages)
    logger.info(
        "context compression started: session=%s messages=%d tokens=~%s model=%s focus=%r",
        self.session_id or "none", _pre_msg_count,
        f"{approx_tokens:,}" if approx_tokens else "unknown", self.model,
        focus_topic,
    )
```

压缩流程:

```text
1. 预通知内存提供者 (on_pre_compress)
2. 调用 context_compressor.compress(messages, tokens, focus_topic)
   │
   ├─ auxiliary model 失败? → fallback 到 main model
   │  → emit warning: "compression model '{model}' failed, recovered using main"
   │
3. Todo store 格式化注入 (preserve todo items)
4. Session 分割: SQLite session split
   │
   ├─ 压缩前消息保存为 "old session"
   ├─ 压缩后消息成为 "new session" 的初始上下文
   │
5. 压缩后更新上下文长度和压力阈值
6. 通过 status_callback 通知用户
```

### 7.2 压缩演进对比

| 维度 | v0.2 | v0.5 | v0.7 | v0.9 | v0.12 |
|------|------|------|------|------|-------|
| 触发机制 | 阈值 50% hygiene | ratio-based scaling | threshold configurable | tiered warnings | threshold + degradation |
| 目标 | `summary_target_tokens` | `compression.target_ratio` | — | `protect_last_n` | ratio + hard cap 12K |
| Focus | 无 | 无 | 无 | `/compress <focus>` | guided compression |
| 辅助模型 | 无 | 可配置 | — | fallback to main | fallback + warning |
| 死亡螺旋 | 无保护 | 无保护 | 防止压缩死亡螺旋 | 防止压缩死亡螺旋 | 防止 + auto-reset |
| Session split | 无 | SQLite split | SQLite split | preserve transcript | preserve transcript |

---

## 8. 并发工具执行

### 8.1 _run_tool 工作线程

当模型返回多个 tool_calls 时,AIAgent 使用 ThreadPoolExecutor 并发执行:

```python
# 源码位置: run_agent.py:9716-9789

def _run_tool(index, tool_call, function_name, function_args):
    """Worker function executed in a thread."""
    # Register this worker tid for interrupt fan-out
    _worker_tid = threading.current_thread().ident
    with self._tool_worker_threads_lock:
        self._tool_worker_threads.add(_worker_tid)

    # Race: if the agent was interrupted between fan-out and registration,
    # apply the interrupt to our own tid now
    if self._interrupt_requested:
        try:
            _set_interrupt(True, _worker_tid)
        except Exception:
            pass

    # Set activity callback on THIS worker thread for heartbeats
    try:
        from tools.environments.base import set_activity_callback
        set_activity_callback(self._touch_activity)
    except Exception:
        pass

    # Propagate approval/sudo callbacks to this worker thread
    if _parent_approval_cb is not None:
        try:
            _set_approval_callback(_parent_approval_cb)
        except Exception:
            pass
    if _parent_sudo_cb is not None:
        try:
            _set_sudo_password_callback(_parent_sudo_cb)
        except Exception:
            pass

    start = time.time()
    try:
        result = self._invoke_tool(
            function_name,
            function_args,
            effective_task_id,
            tool_call.id,
            messages=messages,
            pre_tool_block_checked=True,
        )
    except Exception as tool_error:
        result = f"Error executing tool '{function_name}': {tool_error}"
```

每个工作线程的关键设计:
- **Interrupt fan-out**: 注册 worker tid 到 `_tool_worker_threads`,使 `AIAgent.interrupt()` 可以向所有活跃 worker 广播中断信号
- **Activity callback**: 线程局部设置 `set_activity_callback`,使 `_wait_for_process` (终端命令) 可以发送心跳
- **Approval callback 传播**: 父线程的 approval/sudo callback 通过 `_set_approval_callback` 和 `_set_sudo_password_callback` 传播到每个 worker (GHSA-qg5c-hvr5-hjgr 安全修复)
- **Worker tid 清理**: finally 块中从 `_tool_worker_threads` 移除,清除 interrupt bit,清除线程局部 callbacks

---

## 9. _interruptible_api_call

### 9.1 后台线程 API 调用

`_interruptible_api_call` 在后台线程中执行 API 调用,主线程可以检测中断而无需等待完整 HTTP round-trip:

```python
# 源码位置: run_agent.py:6460-6473

def _interruptible_api_call(self, api_kwargs: dict):
    """
    Run the API call in a background thread so the main conversation loop
    can detect interrupts without waiting for the full HTTP round-trip.

    Each worker thread gets its own OpenAI client instance. Interrupts only
    close that worker-local client, so retries and other requests never
    inherit a closed transport.

    Includes a stale-call detector: if no response arrives within the
    configured timeout, the connection is killed and an error raised so
    the main retry loop can try again with backoff / credential rotation /
    provider fallback.
    """
```

API 调用路由:

```python
# 源码位置: run_agent.py:6478-6519

def _call():
    try:
        if self.api_mode == "codex_responses":
            request_client_holder["client"] = self._create_request_openai_client(
                reason="codex_stream_request",
                api_kwargs=api_kwargs,
            )
            result["response"] = self._run_codex_stream(api_kwargs, client=request_client_holder["client"])
        elif self.api_mode == "anthropic_messages":
            result["response"] = self._anthropic_messages_create(api_kwargs)
        elif self.api_mode == "bedrock_converse":
            from agent.bedrock_adapter import (
                _get_bedrock_runtime_client,
                normalize_converse_response,
            )
            raw_response = client.converse(**api_kwargs)
            result["response"] = normalize_converse_response(raw_response)
        else:
            request_client_holder["client"] = self._create_request_openai_client(
                reason="chat_completion_request",
                api_kwargs=api_kwargs,
            )
            result["response"] = request_client_holder["client"].chat.completions.create(**api_kwargs)
```

四种 API 调用路径:
- `codex_responses` — OpenAI Responses API (Codex), 使用 `_run_codex_stream` 流式消费
- `anthropic_messages` — Anthropic Messages API, 使用 `_anthropic_messages_create`
- `bedrock_converse` — AWS Bedrock Converse API, 使用 boto3 直接调用 + `normalize_converse_response`
- `chat_completions` (default) — OpenAI Chat Completions API, 标准 `client.chat.completions.create`

---

## 10. 模型适配器连接

### 10.1 Transport 层 (v0.11+)

v0.11 将格式转换和 HTTP 传输从 `run_agent.py` 抽取到 `agent/transports/` 层,形成四种可插拔 Transport:

| Transport | 提供者 | 核心方法 | 格式转换 |
|-----------|--------|----------|----------|
| AnthropicTransport | Anthropic | `_anthropic_messages_create` | Anthropic → OpenAI-compatible |
| ChatCompletionsTransport | OpenAI-compatible | `client.chat.completions.create` | 原生 |
| ResponsesApiTransport | Codex | `_run_codex_stream` | Codex Responses → OpenAI-compatible |
| BedrockTransport | AWS Bedrock | `client.converse` | Bedrock Converse → OpenAI-compatible |

### 10.2 提供者路由流程

```text
run_conversation ──► _interruptible_api_call(api_kwargs)
                          │
                          ▼
                    api_mode 检查
                          │
     ┌────────────────────┼────────────────────┐
     │                    │                    │
     ▼                    ▼                    ▼
anthropic_messages   codex_responses    chat_completions
     │                    │                    │
     ▼                    ▼                    ▼
_anthropic_messages  _run_codex_stream  client.chat.completions
_create()            ()                  .create()
     │                    │                    │
     ▼                    ▼                    ▼
AnthropicTransport  ResponsesApiTransport  ChatCompletionsTransport
     │                    │                    │
     ▼                    ▼                    ▼
build_anthropic_    _create_request_    _create_request_openai_
client()            openai_client()      client()
```

---

## 11. 工具调度与 Guardrails

### 11.1 工具调度流程

```text
tool_calls ──► 预检查 (tool_guardrails)
                    │
                    ├─ block verdict? → halt, return safety message
                    │
                    ▼
              并发执行 (_run_tool workers)
                    │
     ┌──────────────┼──────────────┐
     │              │              │
     ▼              ▼              ▼
  _invoke_tool  _invoke_tool  _invoke_tool
  (tool_1)      (tool_2)      (tool_3)
     │              │              │
     ▼              ▼              ▼
  tool_handler  tool_handler  tool_handler
  (from registry) (from registry) (from registry)
     │              │              │
     └──────────────┼──────────────┘
                    │
                    ▼
              结果收集 → 附加到 messages
                    │
                    ▼
              继续循环 (下一个 API 调用)
```

### 11.2 Tool Guardrails

Tool guardrails 在每个 turn 重置 (`_tool_guardrails.reset_for_turn()`),提供以下安全检查:

| 检查 | 目的 | 版本 |
|------|------|------|
| Dangerous command detection | 阻止风险 shell 命令 | v0.2+ |
| Path traversal prevention | 阻止 `../` 在 skill category | v0.6 |
| Shell injection prevention | 阻止 sudo password piping | v0.2 |
| Tirith pre-exec scan | 静态分析终端命令 | v0.4 |
| File tool path guards | 阻止写 `/etc/`, `/boot/`, docker.sock | v0.6 |
| Subagent toolset restriction | 子智能体只能用父启用的工具集 | v0.5 |

---

## 12. 模型对比表

### 12.1 故障转移行为对比

| 场景 | 行为 | 恢复时机 | 通知 |
|------|------|----------|------|
| 429 rate limit | 60s cooldown + activate fallback | cooldown 结束后下一 turn | status_callback |
| 401 auth failure | 凭证池轮换 (credential_pool) | 池中下一个 key | status_callback |
| 500 server error | retry with jittered backoff | 下一个 retry | status_callback |
| Empty response | 3 retry nudges | nudge 成功或耗尽 | status_callback |
| Thinking-only response | accept as "(empty)" or retry | retry 或 accept | status_callback |
| Context exceeded | compress_context then retry | 压缩后 retry | status_callback + warning |
| Compression failure | fallback compression model | main model 恢复 | warning |
| All retries exhausted | graceful return, no crash | — | status_callback |

### 12.2 API 调用路径对比

| api_mode | Transport | 流式支持 | Thinking 处理 | 工具调用格式 |
|----------|-----------|----------|---------------|-------------|
| `anthropic_messages` | AnthropicTransport | SSE | thinking block signatures | Anthropic native |
| `chat_completions` | ChatCompletionsTransport | SSE | `reasoning_content` field | OpenAI function calling |
| `codex_responses` | ResponsesApiTransport | SSE stream | reasoning items | Codex tool use |
| `bedrock_converse` | BedrockTransport | 否 | — | Bedrock native |

### 12.3 回调消费者对比

| 回调 | CLI TUI | Gateway | Web Dashboard | ACP |
|------|---------|---------|---------------|-----|
| stream_delta | Ink text render | platform send_message | SSE push | SSE push |
| thinking | reasoning box | platform send_message | SSE push | SSE push |
| reasoning | reasoning display | platform send_message | SSE push | SSE push |
| tool_progress | KawaiiSpinner | platform progress msg | progress bar | SSE push |
| clarify | input() prompt | platform buttons | modal dialog | — |
| status | banner line | platform message | status toast | SSE push |
| tool_gen | "preparing..." text | platform message | progress text | SSE push |

---

## 13. agent/ 目录关键支撑模块

AIAgent 引擎的运行依赖于 `agent/` 目录下的多个支撑子系统 (54 文件, ~20K 行),这些模块虽独立于 `run_agent.py` 但为引擎提供压缩、自审、提示构建、防护、错误分类和凭据管理等关键能力:

| 子模块 | 行数 | 核心职责 | 引擎依赖关系 |
|--------|------|----------|-------------|
| `agent/context_compressor.py` | 1,481 | 对话历史压缩,focus_topic 导向,session split | `_compress_context` 调用其 `compress()` 方法 |
| `agent/curator.py` | 1,674 | fork 子代理对输出进行自审与改进建议 | 后台 review fork 在 post-iteration 阶段调用 |
| `agent/prompt_builder.py` | 1,186 | 动态 system prompt 组装,skill/memory/todo 注入 | `run_conversation` 每轮构建 API kwargs 时调用 |
| `agent/tool_guardrails.py` | 455 | 工具调用循环检测,渐进式 warn→block→halt | `_run_tool` 前置检查,每 turn `reset_for_turn()` |
| `agent/error_classifier.py` | 1,036 | 18 种 FailoverReason 分类与恢复路径决策 | retry loop 中 `classify_api_error()` 查询恢复策略 |
| `agent/credential_pool.py` | 1,584 | 4 种轮转策略,exhausted 冷却,OAuth refresh | `_try_activate_fallback` 调用其轮转逻辑 |

---

## 总结表

| 组件 | 行数范围 | 职责 |
|------|----------|------|
| AIAgent 类 | L885-14,404 (~13,500) | 智能体主类,全部对话/工具/流式/故障转移/压缩逻辑 |
| IterationBudget | L272-311 | 线程安全迭代计数器,consume/refund,max_total=90(父)/50(子) |
| `__init__` | L908-2,228 | 50+ 参数初始化,11 回调赋值,凭证池,故障转移链构建 |
| `run_conversation` | L10,573-14,404 (~3,800) | 完整对话循环:pre-turn 复位→API 调用→解析→工具→压缩 |
| `_interruptible_api_call` | L6,460-6,700 | 后台线程 API 调用,中断检测,4 种 api_mode 路由 |
| `_try_activate_fallback` | L7,633-7,700 | 有序回退链激活,原地替换运行时,rate-limit cooldown |
| `_restore_primary_runtime` | L7,831-7,900 | Turn-scoped 主运行时恢复,client 重建,压缩器状态 |
| `_compress_context` | L9,221-9,400 | 上下文压缩,辅助模型 fallback,focus_topic,SQLite split |
| `_run_tool` | L9,716-9,800 | 并发工具执行 worker,interrupt fan-out,approval propagation |
| 11 Callbacks | L936-946, L1,157-1,168 | 全通知路径覆盖:streaming/thinking/tool/lifecycle/status |

---

[上一章: 02 — 系统架构总览](/zh-CN/chapters/02-architecture) | [返回目录](/zh-CN/chapters/overview)