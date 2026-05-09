# 03 — Core Agent Engine AIAgent: Conversation Loop, Streaming, Failover & Callback System

[Previous Chapter](/en/chapters/02-architecture) | [Back to Table of Contents](/en/chapters/overview)

---

## Source Code Location & Key Files

| File | Lines | Key Areas |
|------|------|----------|
| `run_agent.py` | 14,404 | AIAgent class (L885-14,404), ~13,500 lines |
| `run_agent.py` | — | IterationBudget (L272-311) |
| `run_agent.py` | — | `__init__` (L908-2,228) |
| `run_agent.py` | — | `run_conversation` (L10,573-14,404) |
| `run_agent.py` | — | `_interruptible_api_call` (L6,460-6,700) |
| `run_agent.py` | — | `_try_activate_fallback` (L7,633-7,700) |
| `run_agent.py` | — | `_restore_primary_runtime` (L7,831-7,900) |
| `run_agent.py` | — | `_compress_context` (L9,221-9,400) |
| `run_agent.py` | — | `_run_tool` (L9,716-9,800) |

**One-line summary:** AIAgent is a 14,404-line single-file agent engine, encompassing a complete conversation loop, 11 callback types, an ordered failover chain, turn-scoped primary runtime restoration, threshold-triggered context compression, and ThreadPoolExecutor-based concurrent tool execution.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AIAgent Engine                                  │
│                    run_agent.py (14,404 lines)                          │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                     AIAgent.__init__ (L908-2,228)                  │ │
│  │                                                                    │ │
│  │  Inference config: base_url, api_key, provider, api_mode, model   │ │
│  │  Iteration control: max_iterations (90), iteration_budget,        │ │
│  │                     tool_delay                                     │ │
│  │  11 callbacks: tool_progress, tool_start, tool_complete,          │ │
│  │           thinking, reasoning, clarify, step,                     │ │
│  │           stream_delta, interim_assistant, tool_gen, status        │ │
│  │  Failover: fallback_model, credential_pool                         │ │
│  │  Security:  checkpoints_enabled, pass_session_id                   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │              run_conversation (L10,573-14,404)                     │ │
│  │                                                                    │ │
│  │  ┌─────────────────┐  ┌──────────────────────────────────────┐   │ │
│  │  │ Pre-turn reset   │  │    Main loop (while iteration)      │   │ │
│  │  │                 │  │                                    │   │ │
│  │  │ • reset retry    │  │  ┌──────────┐                      │   │ │
│  │  │ • restore prim  │  │  │ API Call │                      │   │ │
│  │  │ • sanitize      │  │  │ (interruptible)                  │   │ │
│  │  │ • health check  │  │  └──────────┘                      │   │ │
│  │  │ • log context   │  │       │                             │   │ │
│  │  └─────────────────┘  │       ▼                             │   │ │
│  │                       │  ┌──────────────────────────┐      │   │ │
│  │                       │  │ Response parsing          │      │   │ │
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

The AIAgent class (14,404 lines) encompasses all of Hermes Agent's agent logic. The conversation loop `run_conversation` resets all retry counters and restores the primary provider at the start of each turn, then in a while loop: calls `_interruptible_api_call` (background thread, supports mid-call interruption), parses the response (tool_calls / empty response / thinking-only), executes tools concurrently (`_run_tool` ThreadPoolExecutor workers), checks context pressure and triggers compression. Failover via the ordered chain `_try_activate_fallback` in-place swaps the client/model/provider on the same AIAgent instance, with the next turn automatically restoring the primary runtime. The 11 callbacks cover all notification paths from streaming deltas to lifecycle events. IterationBudget uses a thread-safe Lock to control iteration limits (parent 90, subagent 50).

---

## 1. AIAgent Class Definition

### 1.1 Class Structure Overview

The AIAgent class starts at L885 and is the core of the entire `run_agent.py`. The class definition includes a corruption marker constant, a `base_url` property (setter automatically maintains `_base_url_lower` and `_base_url_hostname`), a `__init__` method with 50+ parameters, and approximately 50+ methods covering conversation loop, streaming, failover, compression, tool dispatch, and all other agent behaviors.

```python
# Source location: run_agent.py:885-896

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

`_TOOL_CALL_ARGUMENTS_CORRUPTION_MARKER` is a safety marker: when tool call arguments are corrupted in a session, this marker replaces the corrupted arguments to keep the conversation alive, rather than crashing the entire session.

### 1.2 __init__ Parameter System

`__init__` accepts over 50 parameters, grouped by function:

**Inference Config:**

```python
# Source location: run_agent.py:910-918
base_url: str = None,
api_key: str = None,
provider: str = None,
api_mode: str = None,        # "chat_completions" or "codex_responses"
acp_command: str = None,
model: str = "",              # default "anthropic/claude-opus-4.6"
max_tokens: int = None,
reasoning_config: Dict[str, Any] = None,
service_tier: str = None,
```

**Iteration Control:**

```python
# Source location: run_agent.py:919-920
max_iterations: int = 90,    # default tool call iteration limit (shared)
tool_delay: float = 1.0,     # delay between tool calls
iteration_budget: "IterationBudget" = None,  # parent creates, subagent inherits
```

**Tool Config:**

```python
# Source location: run_agent.py:921-922
enabled_toolsets: List[str] = None,
disabled_toolsets: List[str] = None,
```

**11 Callbacks:**

```python
# Source location: run_agent.py:936-946
tool_progress_callback: callable = None,   # (tool_name, args_preview)
tool_start_callback: callable = None,      # tool start notification
tool_complete_callback: callable = None,   # tool completion notification
thinking_callback: callable = None,        # thinking block preview
reasoning_callback: callable = None,       # extended reasoning display
clarify_callback: callable = None,         # (question, choices) → str
step_callback: callable = None,            # step notification
stream_delta_callback: callable = None,    # streaming incremental text
interim_assistant_callback: callable = None, # interim assistant response
tool_gen_callback: callable = None,        # tool argument generation progress
status_callback: callable = None,          # lifecycle events
```

**Platform Context:**

```python
# Source location: run_agent.py:952-959
platform: str = None,          # "cli", "telegram", "discord", etc.
user_id: str = None,
user_name: str = None,
chat_id: str = None,
chat_name: str = None,
chat_type: str = None,
thread_id: str = None,
gateway_session_key: str = None,
```

**Failover:**

```python
# Source location: run_agent.py:966-967
fallback_model: Dict[str, Any] = None,
credential_pool=None,
```

**Security:**

```python
# Source location: run_agent.py:968-972
checkpoints_enabled: bool = False,
checkpoint_max_snapshots: int = 20,
checkpoint_max_total_size_mb: int = 500,
checkpoint_max_file_size_mb: int = 10,
pass_session_id: bool = False,
```

---

## 2. Conversation Loop: run_conversation

### 2.1 Method Signature & Core Responsibilities

```python
# Source location: run_agent.py:10573-10600

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

`run_conversation` is the main entry point method of AIAgent. After receiving a user message, it executes the full tool-calling loop until the model returns a pure text response or the iteration budget is exhausted.

### 2.2 Pre-turn Reset (Turn-scoped State Management)

At the start of each turn, `run_conversation` performs a series of key reset operations:

```python
# Source location: run_agent.py:10601-10684

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

These resets ensure:
- **Turn-scoped failover**: The previous turn's fallback does not affect the current turn
- **Retry counter isolation**: Each turn starts retries fresh, not accumulating historical burden
- **Dead connection cleanup**: Detects and closes zombie TCP connections left from previous provider outages
- **Surrogate elimination**: Prevents lone surrogate characters from Google Docs/Word clipboard pastes causing JSON serialization crashes
- **Iteration budget reset**: Each turn gets a brand-new IterationBudget

### 2.3 Main Loop Structure

Core logic of the main loop (simplified):

```text
while iteration_budget.consume():
    1. Build API call parameters (messages, tools, system prompt)
    2. _interruptible_api_call(api_kwargs)
       → background thread execution, main thread can detect interrupt
    3. Parse response:
       a. tool_calls? → concurrent execution (_run_tool workers)
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

## 3. Streaming Mechanism

### 3.1 Dual-Channel Streaming Architecture

AIAgent uses a dual-channel streaming architecture:

| Channel | Callback | Purpose | Consumer |
|----------|----------|---------|----------|
| Display | `stream_delta_callback` | Incremental text display | CLI TUI, Gateway platforms |
| Audio | `_stream_callback` | TTS pre-generation audio | TTS pipeline |

```python
# Source location: run_agent.py:6632-6646

# Streaming delta callbacks (dual-channel)
callbacks = [cb for cb in (self.stream_delta_callback, self._stream_callback) if cb is not None]
```

Both callbacks fire simultaneously during stream consumption: `stream_delta_callback` handles incremental text display (TUI / platform messages), while `_stream_callback` handles early TTS audio generation startup.

### 3.2 Stream Consumption Flow

```text
SSE stream ──► parse per chunk
             │
     ┌───────┼────────────────┐
     │       │                │
     ▼       ▼                ▼
  text delta  reasoning delta  tool_call delta
     │       │                │
     ▼       ▼                ▼
  stream_delta thinking_     tool_gen_
  callback     callback     callback
     │       │                │
     ▼       ▼                ▼
  TUI/platform   reasoning preview    "preparing..." hint
  update         box rendering        tool arg generation progress
```

Key design points:
- **reasoning preview buffering** (v0.5: #3013) — reasoning preview blocks are buffered to prevent duplicate display
- **reasoning box anti-triple-render** (v0.5: #3405) — prevents the reasoning box from rendering 3 times in the tool-calling loop
- **thinking/reasoning separation** — `-thinking-` blocks and `reasoning_content` are processed and callbacked separately
- **tool_gen_callback** — streams tool argument generation progress ("preparing terminal...")

---

## 4. Failover Logic

### 4.1 Ordered Fallback Provider Chain

Hermes Agent implements an ordered fallback provider chain: when the primary provider keeps failing, it in-place swaps the client, model, provider, api_mode and other runtime state on the same AIAgent instance, allowing the retry loop to continue using the new backend.

```python
# Source location: run_agent.py:7633-7644

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

Fallback chain activation flow:

```text
Primary provider fails ──► retry with backoff ──► still fails
       │
       ▼
_try_activate_fallback(reason)
       │
       ├─ reason=rate_limit/billing → set 60s cooldown
       │
       ▼
Check _fallback_index < len(_fallback_chain)
       │
       ▼
Take fb = _fallback_chain[_fallback_index]
       │
       ▼
resolve_provider_client(fb_provider, fb_model)
       │
       ├─ client=None → skip, try next in chain
       │
       ▼
In-place swap:
  self.model = fb_model
  self.provider = fb_provider
  self.client = fb_client
  self.api_mode = fb_api_mode
  self.base_url = fb_base_url
       │
       ▼
Mark _fallback_activated = True
Save _primary_runtime (current runtime snapshot)
       │
       ▼
Retry loop continues using the new backend
```

**Ollama Cloud security**: For Ollama Cloud endpoints, the fallback configuration fetches `OLLAMA_API_KEY` from environment variables when no explicit key is provided, using host matching (not substring) to prevent security bypasses (GHSA-76xc-57q6-vm5m).

```python
# Source location: run_agent.py:7681-7682

if fb_base_url_hint and base_url_host_matches(fb_base_url_hint, "ollama.com") and not fb_api_key_hint:
    fb_api_key_hint = os.getenv("OLLAMA_API_KEY") or None
```

### 4.2 Turn-scoped Primary Runtime Restoration

Key design: failover is **turn-scoped**, not session-scoped. The next `run_conversation` call automatically restores the primary provider:

```python
# Source location: run_agent.py:7831-7843

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

Restoration operations include:
- Core runtime state: `model`, `provider`, `base_url`, `api_mode`, `api_key`, `_client_kwargs`
- Transport cache clearing: `_transport_cache.clear()`
- Prompt caching state: `_use_prompt_caching`, `_use_native_cache_layout`
- Client rebuild: Anthropic uses `build_anthropic_client`, OpenAI-compatible uses `_create_openai_client`
- Context compressor state restoration: `context_compressor.update_model()`

```python
# Source location: run_agent.py:7850-7866

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

## 5. Callback System

### 5.1 11 Callback Types

AIAgent supports 11 callbacks, covering all notification paths from streaming increments to lifecycle events:

| Callback | Signature | Trigger Timing | Consumer |
|----------|-----------|----------------|----------|
| `tool_progress_callback` | `(tool_name, args_preview)` | Tool execution starts | CLI spinner, gateway progress |
| `tool_start_callback` | `(tool_name, args)` | Tool call begins | TUI widget |
| `tool_complete_callback` | `(tool_name, result)` | Tool call completes | TUI widget |
| `thinking_callback` | `(thinking_text)` | `-thinking-` block increment | CLI reasoning box |
| `reasoning_callback` | `(reasoning_content)` | `reasoning_content` increment | CLI reasoning display |
| `clarify_callback` | `(question, choices) → str` | User interaction clarification | CLI input, gateway buttons |
| `step_callback` | `(step_info)` | Conversation step notification | observability |
| `stream_delta_callback` | `(delta_text)` | Streaming text increment | TUI, platform messages |
| `interim_assistant_callback` | `(partial_response)` | Interim assistant response | TTS preview |
| `tool_gen_callback` | `(progress_text)` | Tool argument generation progress | "preparing terminal..." |
| `statusCallback` | `(status_message)` | lifecycle events | retry/fallback/compress notification |

Source location: `run_agent.py:1157-1168` (callbacks assigned to self)

### 5.2 Callback Injection Points in the Engine

```text
run_conversation main loop:
  │
  ├── Pre-turn: status_callback ("Detected stale connections...")
  │
  ├── API Call:
  │   ├── stream_delta_callback (each text delta)
  │   ├── _stream_callback (each text delta, for TTS)
  │   ├── thinking_callback (each thinking delta)
  │   ├── reasoning_callback (each reasoning delta)
  │   ├── tool_gen_callback (tool argument generation delta)
  │   ├── status_callback (retry/fallback/compress events)
  │   └── interim_assistant_callback (interim response)
  │
  ├── Tool Execution:
  │   ├── tool_progress_callback (tool starts)
  │   ├── tool_start_callback (tool call begins)
  │   ├── tool_complete_callback (tool call completes)
  │   └── clarify_callback (interaction clarification)
  │
  ├── Post-iteration:
  │   ├── step_callback (step completion)
  │   └── status_callback (compress/fallback notification)
```

---

## 6. IterationBudget & Iteration Control

### 6.1 IterationBudget Class

IterationBudget (L272) is a thread-safe iteration counter that controls the maximum number of iterations for the agent loop:

```python
# Source location: run_agent.py:272-310

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

Key design points:
- **Subagent independent budget**: Subagents get an independent budget (default 50), not sharing the parent's 90 limit (v0.5: #3004 fixed the shared budget causing subagents to exhaust prematurely)
- **execute_code refund**: execute_code's programmatic tool call iterations are refunded via `refund()`, not consuming budget
- **Thread safety**: Uses `threading.Lock` to protect `_used` count

### 6.2 Iteration Control Evolution

| Version | Change | Issue Fix |
|---------|---------|-----------|
| v0.2 | Shared parent+subagent budget | — |
| v0.2 | Iteration budget pressure via tool result injection | — |
| v0.5 | Subagent independent budget (delegation.max_iterations) | Subagent premature exhaustion (#3004) |
| v0.7 | Reset retry counters at run_conversation start | Prevent cross-turn accumulation (#607) |
| v0.8 | Jittered retry backoff | — |
| v0.12 | IterationBudget reset at each turn | Ensure each turn has full budget |

---

## 7. Context Compression Trigger

### 7.1 _compress_context Method

When context token count exceeds the threshold, `_compress_context` automatically compresses conversation history and splits the session:

```python
# Source location: run_agent.py:9221-9238

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

Compression flow:

```text
1. Pre-notify memory providers (on_pre_compress)
2. Call context_compressor.compress(messages, tokens, focus_topic)
   │
   ├─ auxiliary model fails? → fallback to main model
   │  → emit warning: "compression model '{model}' failed, recovered using main"
   │
3. Todo store format injection (preserve todo items)
4. Session split: SQLite session split
   │
   ├─ Pre-compression messages saved as "old session"
   ├─ Post-compression messages become "new session" initial context
   │
5. Update context length and pressure threshold after compression
6. Notify user via status_callback
```

### 7.2 Compression Evolution Comparison

| Dimension | v0.2 | v0.5 | v0.7 | v0.9 | v0.12 |
|-----------|------|------|------|------|-------|
| Trigger mechanism | threshold 50% hygiene | ratio-based scaling | threshold configurable | tiered warnings | threshold + degradation |
| Target | `summary_target_tokens` | `compression.target_ratio` | — | `protect_last_n` | ratio + hard cap 12K |
| Focus | none | none | none | `/compress <focus>` | guided compression |
| Auxiliary model | none | configurable | — | fallback to main | fallback + warning |
| Death spiral | no protection | no protection | prevent compression death spiral | prevent compression death spiral | prevent + auto-reset |
| Session split | none | SQLite split | SQLite split | preserve transcript | preserve transcript |

---

## 8. Concurrent Tool Execution

### 8.1 _run_tool Worker Thread

When the model returns multiple tool_calls, AIAgent executes them concurrently using ThreadPoolExecutor:

```python
# Source location: run_agent.py:9716-9789

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

Key design per worker thread:
- **Interrupt fan-out**: Registers worker tid to `_tool_worker_threads`, enabling `AIAgent.interrupt()` to broadcast interrupt signals to all active workers
- **Activity callback**: Thread-local `set_activity_callback` enables `_wait_for_process` (terminal commands) to send heartbeats
- **Approval callback propagation**: Parent thread's approval/sudo callbacks propagate to each worker via `_set_approval_callback` and `_set_sudo_password_callback` (GHSA-qg5c-hvr5-hjgr security fix)
- **Worker tid cleanup**: Removes from `_tool_worker_threads` in finally block, clears interrupt bit, clears thread-local callbacks

---

## 9. _interruptible_api_call

### 9.1 Background Thread API Call

`_interruptible_api_call` executes the API call in a background thread, allowing the main thread to detect interrupts without waiting for a full HTTP round-trip:

```python
# Source location: run_agent.py:6460-6473

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

API call routing:

```python
# Source location: run_agent.py:6478-6519

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

Four API call paths:
- `codex_responses` — OpenAI Responses API (Codex), consumed via `_run_codex_stream` streaming
- `anthropic_messages` — Anthropic Messages API, uses `_anthropic_messages_create`
- `bedrock_converse` — AWS Bedrock Converse API, uses boto3 direct call + `normalize_converse_response`
- `chat_completions` (default) — OpenAI Chat Completions API, standard `client.chat.completions.create`

---

## 10. Model Adapter Connection

### 10.1 Transport Layer (v0.11+)

v0.11 extracted format conversion and HTTP transport from `run_agent.py` into the `agent/transports/` layer, forming four pluggable Transports:

| Transport | Provider | Core Method | Format Conversion |
|-----------|----------|-------------|-------------------|
| AnthropicTransport | Anthropic | `_anthropic_messages_create` | Anthropic → OpenAI-compatible |
| ChatCompletionsTransport | OpenAI-compatible | `client.chat.completions.create` | Native |
| ResponsesApiTransport | Codex | `_run_codex_stream` | Codex Responses → OpenAI-compatible |
| BedrockTransport | AWS Bedrock | `client.converse` | Bedrock Converse → OpenAI-compatible |

### 10.2 Provider Routing Flow

```text
run_conversation ──► _interruptible_api_call(api_kwargs)
                          │
                          ▼
                    api_mode check
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

## 11. Tool Dispatch & Guardrails

### 11.1 Tool Dispatch Flow

```text
tool_calls ──► pre-check (tool_guardrails)
                    │
                    ├─ block verdict? → halt, return safety message
                    │
                    ▼
              concurrent execution (_run_tool workers)
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
              Collect results → append to messages
                    │
                    ▼
              Continue loop (next API call)
```

### 11.2 Tool Guardrails

Tool guardrails reset on each turn (`_tool_guardrails.reset_for_turn()`), providing the following safety checks:

| Check | Purpose | Version |
|-------|---------|---------|
| Dangerous command detection | Block risky shell commands | v0.2+ |
| Path traversal prevention | Block `../` in skill category | v0.6 |
| Shell injection prevention | Block sudo password piping | v0.2 |
| Tirith pre-exec scan | Static analysis of terminal commands | v0.4 |
| File tool path guards | Block writing to `/etc/`, `/boot/`, docker.sock | v0.6 |
| Subagent toolset restriction | Subagents can only use parent-enabled toolsets | v0.5 |

---

## 12. Model Comparison Tables

### 12.1 Failover Behavior Comparison

| Scenario | Behavior | Recovery Timing | Notification |
|----------|----------|-----------------|--------------|
| 429 rate limit | 60s cooldown + activate fallback | next turn after cooldown ends | status_callback |
| 401 auth failure | credential pool rotation (credential_pool) | next key in pool | status_callback |
| 500 server error | retry with jittered backoff | next retry | status_callback |
| Empty response | 3 retry nudges | nudge succeeds or exhausted | status_callback |
| Thinking-only response | accept as "(empty)" or retry | retry or accept | status_callback |
| Context exceeded | compress_context then retry | retry after compression | status_callback + warning |
| Compression failure | fallback compression model | main model restores | warning |
| All retries exhausted | graceful return, no crash | — | status_callback |

### 12.2 API Call Path Comparison

| api_mode | Transport | Streaming Support | Thinking Handling | Tool Call Format |
|----------|-----------|-------------------|------------------|-----------------|
| `anthropic_messages` | AnthropicTransport | SSE | thinking block signatures | Anthropic native |
| `chat_completions` | ChatCompletionsTransport | SSE | `reasoning_content` field | OpenAI function calling |
| `codex_responses` | ResponsesApiTransport | SSE stream | reasoning items | Codex tool use |
| `bedrock_converse` | BedrockTransport | No | — | Bedrock native |

### 12.3 Callback Consumer Comparison

| Callback | CLI TUI | Gateway | Web Dashboard | ACP |
|----------|---------|---------|---------------|-----|
| stream_delta | Ink text render | platform send_message | SSE push | SSE push |
| thinking | reasoning box | platform send_message | SSE push | SSE push |
| reasoning | reasoning display | platform send_message | SSE push | SSE push |
| tool_progress | KawaiiSpinner | platform progress msg | progress bar | SSE push |
| clarify | input() prompt | platform buttons | modal dialog | — |
| status | banner line | platform message | status toast | SSE push |
| tool_gen | "preparing..." text | platform message | progress text | SSE push |

---

## 13. Key Support Modules in the agent/ Directory

The AIAgent engine's operation depends on multiple support subsystems in the `agent/` directory (54 files, ~20K lines). These modules are independent from `run_agent.py` but provide critical capabilities such as compression, self-review, prompt building, guardrails, error classification, and credential management:

| Submodule | Lines | Core Responsibility | Engine Dependency |
|-----------|-------|---------------------|-------------------|
| `agent/context_compressor.py` | 1,481 | Conversation history compression, focus_topic guided, session split | `_compress_context` calls its `compress()` method |
| `agent/curator.py` | 1,674 | Fork subagent for self-review and improvement suggestions | Background review fork called in post-iteration phase |
| `agent/prompt_builder.py` | 1,186 | Dynamic system prompt assembly, skill/memory/todo injection | Called by `run_conversation` each round when building API kwargs |
| `agent/tool_guardrails.py` | 455 | Tool call loop detection, progressive warn→block→halt | Pre-check before `_run_tool`, `reset_for_turn()` per turn |
| `agent/error_classifier.py` | 1,036 | 18 FailoverReason classifications and recovery path decisions | `classify_api_error()` queried for recovery strategy in retry loop |
| `agent/credential_pool.py` | 1,584 | 4 rotation strategies, exhausted cooldown, OAuth refresh | `_try_activate_fallback` calls its rotation logic |

---

## Summary Table

| Component | Line Range | Responsibility |
|-----------|------------|----------------|
| AIAgent class | L885-14,404 (~13,500) | Main agent class, all conversation/tool/streaming/failover/compression logic |
| IterationBudget | L272-311 | Thread-safe iteration counter, consume/refund, max_total=90(parent)/50(subagent) |
| `__init__` | L908-2,228 | 50+ parameter initialization, 11 callback assignment, credential pool, failover chain construction |
| `run_conversation` | L10,573-14,404 (~3,800) | Complete conversation loop: pre-turn reset → API call → parse → tools → compression |
| `_interruptible_api_call` | L6,460-6,700 | Background thread API call, interrupt detection, 4 api_mode routing |
| `_try_activate_fallback` | L7,633-7,700 | Ordered fallback chain activation, in-place runtime swap, rate-limit cooldown |
| `_restore_primary_runtime` | L7,831-7,900 | Turn-scoped primary runtime restoration, client rebuild, compressor state |
| `_compress_context` | L9,221-9,400 | Context compression, auxiliary model fallback, focus_topic, SQLite split |
| `_run_tool` | L9,716-9,800 | Concurrent tool execution worker, interrupt fan-out, approval propagation |
| 11 Callbacks | L936-946, L1,157-1,168 | Full notification path coverage: streaming/thinking/tool/lifecycle/status |

---

[Previous Chapter: 02 — System Architecture Overview](/en/chapters/02-architecture) | [Back to Table of Contents](/en/chapters/overview)