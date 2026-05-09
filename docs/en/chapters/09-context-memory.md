# 09 — Context & Memory Engine: Multi-layer Context Management from Compression Summaries to Cross-session Memory

[← 08-State & Persistence](/en/chapters/08-state-persistence) | [→ 10-Security Architecture](/en/chapters/10-security)

---

## Source Files

| File | Lines | Description |
|------|------|------|
| `agent/context_engine.py` | 206 | ContextEngine ABC — abstract base class for pluggable context engines |
| `agent/context_compressor.py` | 1,481 | ContextCompressor — mid-turn summary compression implementation |
| `agent/prompt_builder.py` | 1,186 | PromptBuilder — system prompt assembly: identity + platform + skills + memory + tools + guardrails |
| `agent/curator.py` | 1,674 | Curator — background skill self-review manager (forked review agent) |
| `agent/memory_provider.py` | 279 | MemoryProvider ABC — pluggable memory backend base class |
| `agent/memory_manager.py` | 555 | MemoryManager — memory orchestrator, limited to one external provider |
| `agent/think_scrubber.py` | 386 | StreamingThinkScrubber — streaming reasoning tag state machine |
| `agent/context_references.py` | 518 | ContextReference — @file/url/diff/git reference parsing and expansion |

---

## One-Line Summary

Hermes's context management is built from three engine layers: the context compression layer (ContextEngine ABC → ContextCompressor, mid-turn summaries) controls the conversation window; the prompt assembly layer (PromptBuilder, combining identity/platform/skills/memory/tools/guardrails) builds the system prompt; the memory layer (MemoryProvider ABC → MemoryManager, Honcho/local/hybrid backends) persists across sessions, plus Curator self-review management and StreamingThinkScrubber streaming tag scrubbing.

---

## Architecture Overview

```
                    ┌───────────── System Prompt ──────────────┐
                    │                                          │
   ┌────────────────┼──────────────────────────────────────────┼───────────┐
   │  PromptBuilder │  identity + platform + skills + memory   │ guardrails │
   │  (1186 lines)  │  + tools + context files + env hints     │           │
   └────────────────┼──────────────────────────────────────────┼───────────┘
                    │                                          │
          ┌─────────┼───────────┐          ┌───────────────────┼──────────┐
          │ MemoryManager       │          │ ContextEngine ABC │          │
          │ (555 lines)         │          │ (206 lines)       │          │
          │                     │          │                   │          │
          │  builtin (always)   │          │  ContextCompressor│          │
          │  + 1 external only  │          │  (1481 lines)     │          │
          │  (Honcho/hybrid)    │          │                   │          │
          │                     │          │  1. prune old tools│          │
          │  prefetch → system  │          │  2. protect head   │          │
          │  sync_turn → backend│          │  3. protect tail   │          │
          │  tool schemas → LLM │          │  4. summarize mid  │          │
          └─────────────────────┘          │  5. iterative update│         │
                                          └───────────────────┘          │
                                                                         │
   ┌─────────────────────────┐  ┌──────────────┐  ┌─────────────────────┼──┐
   │ StreamingThinkScrubber  │  │ Curator      │  │ ContextReference    │  │
   │ (386 lines)             │  │ (1674 lines) │  │ (518 lines)         │  │
   │                         │  │              │  │                     │  │
   │ state machine across    │  │ forked review│  │ @file:url:diff:git  │  │
   │ stream deltas           │  │ agent        │  │ parse → expand      │  │
   │ tag suppression         │  │ pin/archive/ │  │ inline in prompt    │  │
   │ partial tag hold-back   │  │ consolidate  │  │                     │  │
   └─────────────────────────┘  └──────────────┘  └─────────────────────┘
```

---

## TL;DR

Hermes's context and memory engine has six core modules working together. ContextEngine ABC (206 lines) defines the interface for pluggable context engines — update_from_response, should_compress, compress are the three core methods. ContextCompressor (1,481 lines) implements a five-step compression algorithm: prune old tool results -> protect head -> protect tail -> summarize middle -> iterative update, with anti-thrashing protection. PromptBuilder (1,186 lines) assembles the system prompt's six major layers: identity, platform hints, skills index, memory context, tools, guardrails, with two-layer skill caching (in-process LRU + disk snapshot). Curator (1,674 lines) is a background skill self-review manager that forks a review agent for pin/archive/consolidate/patch decisions. MemoryProvider ABC (279 lines) defines 10+ lifecycle methods (initialize, prefetch, sync_turn, on_session_end, on_pre_compress, etc.), limited to one external provider. StreamingThinkScrubber (386 lines) is an incremental state machine for streaming reasoning tags, handling partial tag hold-back at delta boundaries.

---

## 1. ContextEngine ABC — Pluggable Context Engine

### 1.1 Core Interface Definition

ContextEngine defines three core methods that all context engines must implement — `name`, `update_from_response`, `should_compress`, `compress` — along with a set of optional methods:

Source location: agent/context_engine.py:32-207

```python
class ContextEngine(ABC):
    """Base class all context engines must implement."""

    # -- Identity ----------------------------------------------------------
    @property
    @abstractmethod
    def name(self) -> str:
        """Short identifier (e.g. 'compressor', 'lcm')."""

    # -- Token state (read by run_agent.py for display/logging) ------------
    last_prompt_tokens: int = 0
    last_completion_tokens: int = 0
    last_total_tokens: int = 0
    threshold_tokens: int = 0
    context_length: int = 0
    compression_count: int = 0

    # -- Compaction parameters ---------------------------------------------
    threshold_percent: float = 0.75
    protect_first_n: int = 3
    protect_last_n: int = 6

    # -- Core interface ----------------------------------------------------
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
        """Compact the message list and return the new message list.
        Args:
            focus_topic: Optional topic from manual /compress <focus>.
        """

    # -- Optional: pre-flight check ----------------------------------------
    def should_compress_preflight(self, messages: List[Dict[str, Any]]) -> bool:
        """Quick rough check before the API call. Default: False."""

    def has_content_to_compress(self, messages: List[Dict[str, Any]]) -> bool:
        """Quick check: is there anything compactable? Default: True."""

    # -- Optional: session lifecycle ---------------------------------------
    def on_session_start(self, session_id: str, **kwargs) -> None:
        """Called when a new conversation session begins."""

    def on_session_end(self, session_id: str, messages: List[Dict[str, Any]]) -> None:
        """Called at real session boundaries. NOT per-turn."""

    def on_session_reset(self) -> None:
        """Called on /new or /reset. Reset per-session state."""
        self.last_prompt_tokens = 0
        self.compression_count = 0

    # -- Optional: tools ---------------------------------------------------
    def get_tool_schemas(self) -> List[Dict[str, Any]]:
        """Return tool schemas this engine provides. Default: []."""

    def handle_tool_call(self, name: str, args: Dict[str, Any], **kwargs) -> str:
        """Handle a tool call. Must return JSON string."""

    # -- Optional: model switch support ------------------------------------
    def update_model(self, model, context_length, base_url="", api_key="", provider="") -> None:
        """Called when user switches models or on fallback activation."""
        self.context_length = context_length
        self.threshold_tokens = int(context_length * self.threshold_percent)
```

### 1.2 Lifecycle

The engine's complete lifecycle:
1. Engine instantiated and registered (plugin `register()` or default)
2. `on_session_start()` called when conversation begins
3. `update_from_response()` called after each API response with usage data
4. `should_compress()` checked after each turn
5. `compress()` called when `should_compress()` returns True
6. `on_session_end()` called at real session boundaries (CLI exit, `/reset`, gateway session expiry)

Configuration selection: `context.engine` in `config.yaml`, default `"compressor"` (built-in), only one engine can be active.

---

## 2. ContextCompressor — Mid-Turn Summary Compression

### 2.1 Five-Step Compression Algorithm

ContextCompressor inherits ContextEngine, implementing the classic "head + tail + summarized middle" algorithm:

Source location: agent/context_compressor.py:322-351

```python
class ContextCompressor(ContextEngine):
    """Default context engine — compresses via lossy summarization.

    Algorithm:
      1. Prune old tool results (cheap, no LLM call)
      2. Protect head messages (system prompt + first exchange)
      3. Protect tail messages by token budget (most recent ~20K tokens)
      4. Summarize middle turns with structured LLM prompt
      5. On subsequent compactions, iteratively update the previous summary
    """
```

### 2.2 Initialization & Token Budget Calculation

ContextCompressor's initialization derives three-layer token budgets — threshold_tokens (trigger point), tail_token_budget (protect tail), max_summary_tokens (summary ceiling):

Source location: agent/context_compressor.py:380-462

```python
def __init__(
    self,
    model: str,
    threshold_percent: float = 0.50,
    protect_first_n: int = 3,
    protect_last_n: int = 20,
    summary_target_ratio: float = 0.20,
    quiet_mode: bool = False,
    summary_model_override: str = None,
    base_url: str = "",
    api_key: str = "",
    config_context_length: int | None = None,
    provider: str = "",
    api_mode: str = "",
):
    self.model = model
    self.threshold_percent = threshold_percent
    self.protect_first_n = protect_first_n
    self.protect_last_n = protect_last_n
    self.summary_target_ratio = max(0.10, min(summary_target_ratio, 0.80))

    self.context_length = get_model_context_length(model, ...)
    # Floor: never compress below MINIMUM_CONTEXT_LENGTH tokens
    self.threshold_tokens = max(
        int(self.context_length * threshold_percent),
        MINIMUM_CONTEXT_LENGTH,
    )

    # Derive token budgets: ratio relative to threshold, not total context
    target_tokens = int(self.threshold_tokens * self.summary_target_ratio)
    self.tail_token_budget = target_tokens
    self.max_summary_tokens = min(
        int(self.context_length * 0.05), _SUMMARY_TOKENS_CEILING,
    )

    self.summary_model = summary_model_override or ""
    self._previous_summary: Optional[str] = None
    # Anti-thrashing: track whether last compression was effective
    self._last_compression_savings_pct: float = 100.0
    self._ineffective_compression_count: int = 0
    self._summary_failure_cooldown_until: float = 0.0
```

### 2.3 Anti-Thrashing Protection

When two consecutive compressions each save less than 10%, compression is stopped to avoid infinite loops (where each pass only removes 1-2 messages):

Source location: agent/context_compressor.py:469-495

```python
def should_compress(self, prompt_tokens: int = None) -> bool:
    """Check if context exceeds threshold. Includes anti-thrashing protection.

    If the last two compressions each saved less than 10%, skip to avoid
    infinite loops where each pass removes only 1-2 messages.
    """
    tokens = prompt_tokens if prompt_tokens is not None else self.last_prompt_tokens
    if tokens < self.threshold_tokens:
        return False
    # Anti-thrashing: back off if recent compressions were ineffective
    ...
```

### 2.4 Summary Model Fallback

When the user-configured summary model (aux model) is unavailable, ContextCompressor implements two-level fallback: on 404/not-found it immediately retries on the main model; on other errors it also attempts main model fallback, but records `_last_aux_model_failure_error` to notify the user of the configuration issue:

Source location: agent/context_compressor.py:930-981

```python
# Summary model 404 — fast-path retry on main model
if (is_model_not_found and self.summary_model != self.model
        and not getattr(self, "_summary_model_fallen_back", False)):
    self._summary_model_fallen_back = True
    logging.warning("Summary model '%s' unavailable. Falling back to main model '%s'.",
                    self.summary_model, self.model)
    self._last_aux_model_failure_error = _err_text
    self._last_aux_model_failure_model = self.summary_model
    self.summary_model = ""  # empty = use main model
    return self._generate_summary(turns_to_summarize, focus_topic=focus_topic)

# Unknown-error best-effort retry on main model
if (self.summary_model and self.summary_model != self.model
        and not getattr(self, "_summary_model_fallen_back", False)):
    self._summary_model_fallen_back = True
    logging.warning("Summary model '%s' failed. Retrying on main model before giving up.",
                    self.summary_model)
    self._last_aux_model_failure_error = _err_text
    self._last_aux_model_failure_model = self.summary_model
    self.summary_model = ""
    return self._generate_summary(turns_to_summarize, focus_topic=focus_topic)
```

### 2.5 Model Switch Support

`update_model()` recalculates all token budgets on model switch, keeping the compressor correctly calibrated across different context lengths (e.g. 200K → 32K):

Source location: agent/context_compressor.py:352-379

```python
def update_model(self, model, context_length, base_url="", api_key="", provider="", api_mode=""):
    """Update model info after a model switch or fallback activation."""
    self.model = model
    self.base_url = base_url
    self.api_key = api_key
    self.provider = provider
    self.api_mode = api_mode
    self.context_length = context_length
    self.threshold_tokens = max(
        int(context_length * self.threshold_percent),
        MINIMUM_CONTEXT_LENGTH,
    )
    target_tokens = int(self.threshold_tokens * self.summary_target_ratio)
    self.tail_token_budget = target_tokens
    self.max_summary_tokens = min(
        int(context_length * 0.05), _SUMMARY_TOKENS_CEILING,
    )
```

---

## 3. PromptBuilder — System Prompt Assembly

### 3.1 Context Threat Scanning

PromptBuilder scans for prompt injection attack patterns before injecting AGENTS.md, .cursorrules, SOUL.md, containing 10 detection rules:

Source location: agent/prompt_builder.py:36-50

```python
_CONTEXT_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    (r'system\s+prompt\s+override', "sys_prompt_override"),
    (r'disregard\s+(your|all|any)\s+(instructions|rules|guidelines)', "disregard_rules"),
    (r'act\s+as\s+(if|though)\s+you\s+(have\s+no|don\'t\s+have)\s+(restrictions|limits|rules)', "bypass_restrictions"),
    (r'<!--[^>]*(?:ignore|override|system|secret|hidden)[^>]*-->', "html_comment_injection"),
    (r'<\s*div\s+style\s*=\s*["\'][\s\S]*?display\s*:\s*none', "hidden_div"),
    (r'translate\s+.*\s+into\s+.*\s+and\s+(execute|run|eval)', "translate_execute"),
    (r'curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET|PASSWORD|CREDENTIAL|API)', "exfil_curl"),
    (r'cat\s+[^\n]*(\.env|credentials|\.netrc|\.pgpass)', "read_secrets"),
]
```

It also detects invisible Unicode characters (`_CONTEXT_INVISIBLE_CHARS`) to prevent zero-width character injection.

### 3.2 Skills System Prompt — Two-Layer Cache

`build_skills_system_prompt()` uses a two-layer cache to accelerate skill index construction: (1) in-process LRU dict, (2) disk snapshot (`_skills_prompt_snapshot.json`) validated by mtime/size manifest:

Source location: agent/prompt_builder.py:718-764

```python
def build_skills_system_prompt(
    available_tools: "set[str] | None" = None,
    available_toolsets: "set[str] | None" = None,
) -> str:
    """Build a compact skill index for the system prompt.

    Two-layer cache:
      1. In-process LRU dict keyed by (skills_dir, tools, toolsets)
      2. Disk snapshot validated by mtime/size manifest — survives restarts
    """
    skills_dir = get_skills_dir()
    external_dirs = get_all_skills_dirs()[1:]  # skip local

    # Layer 1: in-process LRU cache
    cache_key = (
        str(skills_dir.resolve()),
        tuple(str(d) for d in external_dirs),
        tuple(sorted(str(t) for t in (available_tools or set()))),
        tuple(sorted(str(ts) for ts in (available_toolsets or set()))),
        _platform_hint,
        tuple(sorted(disabled)),
    )
    with _SKILLS_PROMPT_CACHE_LOCK:
        cached = _SKILLS_PROMPT_CACHE.get(cache_key)
        if cached is not None:
            _SKILLS_PROMPT_CACHE.move_to_end(cache_key)
            return cached

    # Layer 2: disk snapshot
    snapshot = _load_skills_snapshot(skills_dir)
    ...
```

### 3.3 Context File Loading

PromptBuilder loads context files from multiple sources — SOUL.md, AGENTS.md, .claude/CLAUDE.md, .cursorrules — each going through threat scanning:

Source location: agent/prompt_builder.py:1034-1117

```python
def load_soul_md() -> Optional[str]:
    """Load SOUL.md — the agent's identity and personality definition."""
    ...

def _load_hermes_md(cwd_path: Path) -> str:
    """Load .hermes.md — project-specific instructions."""
    ...

def _load_agents_md(cwd_path: Path) -> str:
    """Load AGENTS.md — codebase documentation."""
    ...

def _load_claude_md(cwd_path: Path) -> str:
    """Load .claude/CLAUDE.md — Claude Code instructions."""
    ...

def _load_cursorrules(cwd_path: Path) -> str:
    """Load .cursorrules — Cursor IDE rules."""
    ...

def build_context_files_prompt(cwd, skip_soul=False) -> str:
    """Assemble all context files into the system prompt."""
    ...
```

---

## 4. Curator — Background Skill Self-Review Manager

### 4.1 Design Philosophy

Curator is an idle-triggered (no cron daemon) background task: when the agent is idle and the last curator run was more than `interval_hours` ago, `maybe_run_curator()` spawns a forked AIAgent to do a review:

Source location: agent/curator.py:1-60

```python
"""Curator — background skill maintenance orchestrator.

Responsibilities:
  - Auto-transition lifecycle states based on derived skill activity timestamps
  - Spawn a background review agent that can pin / archive / consolidate /
    patch agent-created skills via skill_manage
  - Persist curator state (last_run_at, paused, etc.) in .curator_state

Strict invariants:
  - Only touches agent-created skills (see tools/skill_usage.is_agent_created)
  - Never auto-deletes — only archives. Archive is recoverable.
  - Pinned skills bypass all auto-transitions
  - Uses the auxiliary client; never touches the main session's prompt cache
"""

DEFAULT_INTERVAL_HOURS = 24 * 7  # 7 days
DEFAULT_MIN_IDLE_HOURS = 2
DEFAULT_STALE_AFTER_DAYS = 30
DEFAULT_ARCHIVE_AFTER_DAYS = 90
```

### 4.2 Review Runtime Binding

Curator uses `_ReviewRuntimeBinding` to record provider/model/api_key/base_url, supporting per-slot overrides:

Source location: agent/curator.py:47-54

```python
class _ReviewRuntimeBinding(NamedTuple):
    """Provider/model for the curator review fork plus optional per-slot overrides."""
    provider: str
    model: str
    explicit_api_key: Optional[str]
    explicit_base_url: Optional[str]
```

### 4.3 Automatic State Transitions

`apply_automatic_transitions()` automatically advances lifecycle states based on skill last-used timestamps:

Source location: agent/curator.py:255-450

```python
def apply_automatic_transitions(now: Optional[datetime] = None) -> Dict[str, int]:
    """Auto-transition lifecycle states based on skill activity timestamps."""
    ...
```

### 4.4 LLM Review & Classification

`run_curator_review()` forks a review agent, using LLM for structured classification (pin/archive/consolidate/patch/keep) of skills:

Source location: agent/curator.py:1278-1496

```python
def run_curator_review(...) -> Dict[str, Any]:
    """Spawn a forked AIAgent to review agent-created skills."""
    ...

def _run_llm_review(prompt: str) -> Dict[str, Any]:
    """Execute LLM classification on a curated skill list."""
    ...

def _classify_removed_skills(...) -> ...:
    """Classify skills that were removed during the review."""
    ...

def _reconcile_classification(...) -> ...:
    """Reconcile model classification with heuristic rules."""
    ...
```

---

## 5. MemoryProvider ABC — Pluggable Memory Backend

### 5.1 Core Lifecycle Methods

MemoryProvider defines 10+ methods, from initialize to shutdown, covering both session-level and turn-level dimensions:

Source location: agent/memory_provider.py:42-280

```python
class MemoryProvider(ABC):
    """Abstract base class for memory providers."""

    @property
    @abstractmethod
    def name(self) -> str:
        """Short identifier (e.g. 'builtin', 'honcho', 'hindsight')."""

    @abstractmethod
    def is_available(self) -> bool:
        """Return True if configured and ready. Should not make network calls."""

    @abstractmethod
    def initialize(self, session_id: str, **kwargs) -> None:
        """Initialize for a session. kwargs always include:
          - hermes_home, platform
        kwargs may also include:
          - agent_context: 'primary' | 'subagent' | 'cron' | 'flush'
          - agent_identity: Profile name
          - agent_workspace, parent_session_id, user_id
        """

    def system_prompt_block(self) -> str:
        """Static text for system prompt. Return empty string to skip."""

    def prefetch(self, query: str, *, session_id: str = "") -> str:
        """Recall relevant context before each turn. Should be fast."""

    def queue_prefetch(self, query: str, *, session_id: str = "") -> None:
        """Queue background recall for the NEXT turn."""

    def sync_turn(self, user_content: str, assistant_content: str, *, session_id: str = "") -> None:
        """Persist a completed turn. Should be non-blocking."""

    @abstractmethod
    def get_tool_schemas(self) -> List[Dict[str, Any]]:
        """Return tool schemas in OpenAI function calling format."""

    def handle_tool_call(self, tool_name: str, args, **kwargs) -> str:
        """Handle tool call. Must return JSON string."""

    def shutdown(self) -> None:
        """Clean shutdown — flush queues, close connections."""
```

### 5.2 Optional Hooks — Session/Compression/Delegation

MemoryProvider provides 5 optional hooks for deep integration:

| Hook | When Called | Purpose |
|------|------------|---------|
| `on_turn_start(turn, message, **kwargs)` | Each turn start | turn-counting, scope management |
| `on_session_end(messages)` | Real session end (not per-turn) | fact extraction, summarization |
| `on_session_switch(new_id, parent_id, reset)` | session_id change (/resume, /branch, compression) | Update or reset per-session cache |
| `on_pre_compress(messages)` | Before compression | Extract insights from messages about to be discarded |
| `on_delegation(task, result, child_id)` | Sub-agent completion | Parent agent observes delegation result |
| `on_memory_write(action, target, content, metadata)` | Built-in memory tool write | Mirror write to external backend |
| `get_config_schema()` | memory setup | Return configuration field definitions |
| `save_config(values, hermes_home)` | memory setup | Write non-secret config to provider's native location |

### 5.3 Context Fencing

MemoryProvider output is fenced with `<memory-context>` tags, with a system note preventing the LLM from confusing memory and new input:

```python
def build_memory_context_block(raw_context: str) -> str:
    """Wrap prefetched memory in a fenced block with system note."""
    clean = sanitize_context(raw_context)
    return (
        "<memory-context>\n"
        "[System note: The following is recalled memory context, "
        "NOT new user input. Treat as authoritative reference data — "
        "this is the agent's persistent memory and should inform all responses.]\n\n"
        f"{clean}\n"
        "</memory-context>"
    )
```

---

## 6. MemoryManager — Memory Orchestrator

### 6.1 One-External-Provider Limit

MemoryManager enforces allowing only one external provider, preventing tool schema bloat and backend conflicts:

Source location: agent/memory_manager.py:190-248

```python
class MemoryManager:
    """Orchestrates the built-in provider plus at most one external provider."""

    def __init__(self) -> None:
        self._providers: List[MemoryProvider] = []
        self._tool_to_provider: Dict[str, MemoryProvider] = {}
        self._has_external: bool = False

    def add_provider(self, provider: MemoryProvider) -> None:
        """Register a memory provider.

        Built-in provider (name "builtin") is always accepted.
        Only ONE external (non-builtin) provider is allowed.
        """
        is_builtin = provider.name == "builtin"
        if not is_builtin:
            if self._has_external:
                existing = next(
                    (p.name for p in self._providers if p.name != "builtin"), "unknown"
                )
                logger.warning(
                    "Rejected memory provider '%s' — external provider '%s' is "
                    "already registered. Only one external memory provider is "
                    "allowed at a time.",
                    provider.name, existing,
                )
                return
            self._has_external = True
        self._providers.append(provider)
        # Index tool names → provider for routing
        for schema in provider.get_tool_schemas():
            tool_name = schema.get("name", "")
            if tool_name and tool_name not in self._tool_to_provider:
                self._tool_to_provider[tool_name] = provider
```

### 6.2 StreamingContextScrubber — Streaming Memory Fence Scrubbing

`StreamingContextScrubber` (memory_manager.py:62-170) is a stateful incremental state machine that handles `<memory-context>` fences in streaming deltas. For split tags that cross delta boundaries, it holds back partial tag tails and resolves them on the next `feed()` or `flush()`:

Source location: agent/memory_manager.py:62-140

```python
class StreamingContextScrubber:
    """Stateful scrubber for streaming text with split memory-context spans.

    The one-shot sanitize_context regex cannot survive chunk boundaries.
    This scrubber runs a small state machine across deltas, holding back
    partial-tag tails and discarding everything inside a span.
    """

    _OPEN_TAG = "<memory-context>"
    _CLOSE_TAG = "</memory-context>"

    def __init__(self) -> None:
        self._in_span: bool = False
        self._buf: str = ""

    def reset(self) -> None:
        self._in_span = False
        self._buf = ""

    def feed(self, text: str) -> str:
        """Return the visible portion after scrubbing."""
        if not text:
            return ""
        buf = self._buf + text
        self._buf = ""
        out: list[str] = []

        while buf:
            if self._in_span:
                idx = buf.lower().find(self._CLOSE_TAG)
                if idx == -1:
                    held = self._max_partial_suffix(buf, self._CLOSE_TAG)
                    self._buf = buf[-held:] if held else ""
                    return "".join(out)
                buf = buf[idx + len(self._CLOSE_TAG):]
                self._in_span = False
            else:
                idx = buf.lower().find(self._OPEN_TAG)
                if idx == -1:
                    held = self._max_partial_suffix(buf, self._OPEN_TAG)
                    if held:
                        out.append(buf[:-held])
                        self._buf = buf[-held:]
                    else:
                        out.append(buf)
                    return "".join(out)
                if idx > 0:
                    out.append(buf[:idx])
                buf = buf[idx + len(self._OPEN_TAG):]
                self._in_span = True
        return "".join(out)

    def flush(self) -> str:
        """Emit held-back buffer at end-of-stream.
        If still inside an unterminated span, discard remaining content.
        """
        if self._in_span:
            self._buf = ""
            self._in_span = False
            return ""
        tail = self._buf
        self._buf = ""
        return tail
```

---

## 7. StreamingThinkScrubber — Streaming Reasoning Tag Scrubbing

### 7.1 Design Motivation

The `_strip_think_blocks` regex approach works correctly on complete strings, but in per-delta mode it breaks downstream state machines — when MiniMax-M2.7 streams `ჺ`...`pecta`, the per-delta regex erases delta1 entirely at delta boundaries, so the downstream state machine never sees the open tag, treats delta2 as regular content, and leaks reasoning to the user. StreamingThinkScrubber centralizes the tag-suppression state machine at the upstream layer, so all stream_delta_callback sees text that has already had reasoning blocks removed:

Source location: agent/think_scrubber.py:1-50

```python
"""Stateful scrubber for reasoning/thinking blocks in streamed assistant text.

...when MiniMax-M2.7 streams
    delta1 = ".SizeF"
    delta2 = "Let me check their config"
    delta3 = "pecta"
the per-delta regex erases delta1 entirely, so the downstream state machine
never sees the open tag, treats delta2 as regular content, and leaks reasoning
to the user.

This module centralises the tag-suppression state machine at the upstream
layer so every stream_delta_callback sees text that has already had reasoning
blocks removed.

Tag variants handled (case-insensitive):
  .SizeF, <thinking>, <reasoning>, <thought>, <REASONING_SCRATCHPAD>.
"""
```

### 7.2 Block-Boundary Rules

An opening tag is only treated as a reasoning-block opener when it appears at the following positions: stream start, after a newline (optionally followed by whitespace), or when only whitespace has been emitted on the current line. This prevents `setSizeF` in prose from being misidentified:

Source location: agent/think_scrubber.py:46-50

```
Block-boundary rule for opens: an opening tag is only treated as a
reasoning-block opener when it appears at the start of the stream,
after a newline (optionally followed by whitespace), or when only
whitespace has been emitted on the current line.  This prevents prose
```

---

## 8. ContextReference — @-Reference Parsing & Expansion

### 8.1 Data Model

`ContextReference` is a frozen dataclass that records the reference's raw text, type, target, and position:

Source location: agent/context_references.py:40-50

```python
@dataclass(frozen=True)
class ContextReference:
    raw: str           # Original @... text
    kind: str          # 'file', 'folder', 'git', 'url', 'diff', 'staged'
    target: str        # Path/URL/ref after the colon
    start: int         # Start position in message
    end: int           # End position in message
    line_start: int | None = None
    line_end: int | None = None

@dataclass
class ContextReferenceResult:
    """Result of expanding a reference."""
    ...
```

### 8.2 Reference Parsing

`parse_context_references()` uses regex patterns to identify @file:, @folder:, @git:, @url:, @diff, @staged references:

Source location: agent/context_references.py:16-19, 62

```python
_QUOTED_REFERENCE_VALUE = r'(?:`[^`\n]+`|"[^"\n]+"|\'[^\'\n]+\')'
REFERENCE_PATTERN = re.compile(
    rf"(?<![\w/])@(?:(?P<simple>diff|staged)\b|(?P<kind>file|folder|git|url):(?P<value>{_QUOTED_REFERENCE_VALUE}(?::\d+(?:-\d+)?)?|\S+))"
)

def parse_context_references(message: str) -> list[ContextReference]:
    """Parse @-references from a user message."""
    ...
```

### 8.3 Reference Expansion

Each reference type has an independent expansion function:

| Function | Lines | Responsibility |
|----------|-------|---------------|
| `_expand_file_reference()` | 236-262 | Read file content, supports line ranges (:10-20) |
| `_expand_folder_reference()` | 263-280 | List directory contents, limited to 200 entries |
| `_expand_git_reference()` | 280-305 | git show/diff output |
| `_fetch_url_content()` | 305-328 | HTTP fetch URL content |
| `_build_folder_listing()` | 430-446 | Directory tree construction |
| `_file_metadata()` | 494-504 | File metadata (size, mtime) |
| `_code_fence_language()` | 504-... | Infer code fence language from extension |
| `_is_binary_file()` | 420-430 | Binary file detection |

### 8.4 Security Restrictions

ContextReference implements sensitive path protection — files under directories such as `.ssh`, `.aws`, `.gnupg`, `.kube`, `.docker`, `.azure`, `.config/gh` are refused expansion:

Source location: agent/context_references.py:21-37

```python
_SENSITIVE_HOME_DIRS = (".ssh", ".aws", ".gnupg", ".kube", ".docker", ".azure", ".config/gh")
_SENSITIVE_HERMES_DIRS = (Path("skills") / ".hub",)
_SENSITIVE_HOME_FILES = (
    Path(".ssh") / "authorized_keys",
    Path(".ssh") / "id_rsa",
    Path(".ssh") / "id_ed25519",
    Path(".ssh") / "config",
    Path(".bashrc"),
    Path(".zshrc"),
    ...
    Path(".netrc"),
    Path(".pgpass"),
    ...
)
```

---

## Comparison Tables

### Table 1: Context Engine Interface Comparison (ContextEngine ABC vs Implementation)

| Method | ContextEngine ABC | ContextCompressor |
|--------|-------------------|-------------------|
| `name` | abstract | "compressor" |
| `update_from_response` | abstract | token tracking |
| `should_compress` | abstract | threshold + anti-thrashing |
| `compress` | abstract | 5-step summarize |
| `should_compress_preflight` | default False | not overridden |
| `has_content_to_compress` | default True | override (check boundaries) |
| `on_session_start` | default no-op | not overridden |
| `on_session_end` | default no-op | not overridden |
| `on_session_reset` | reset tokens | reset + clear summary |
| `get_tool_schemas` | default [] | [] |
| `handle_tool_call` | default error | N/A |
| `update_model` | update threshold | recalc all budgets |

### Table 2: System Prompt Composition Layer Comparison

| Layer | Source | PromptBuilder Function | Cacheable |
|-------|--------|------------------------|-----------|
| identity | SOUL.md | `load_soul_md()` | No (per-project) |
| platform hints | runtime | `build_environment_hints()` | No (dynamic) |
| skills index | ~/.hermes/skills/ + external | `build_skills_system_prompt()` | Two-layer LRU + disk snapshot |
| memory context | MemoryProvider.prefetch() | `build_memory_context_block()` | provider-managed |
| tools | tools.registry | get_tool_definitions() | No (dynamic toolset) |
| context files | AGENTS.md, CLAUDE.md, .cursorrules | `build_context_files_prompt()` | No (per-project) |
| guardrails | threat patterns | `_scan_context_content()` | No (security scan) |
| nous subscription | runtime | `build_nous_subscription_prompt()` | No (dynamic) |

### Table 3: MemoryProvider Lifecycle Method Comparison

| Method | builtin provider | Honcho provider | local/hybrid |
|--------|-----------------|-----------------|-------------|
| `is_available()` | True (always) | check Honcho API | check local config |
| `initialize()` | load ~/.hermes/memories/ | create Honcho client/banks | load local store |
| `system_prompt_block()` | memory instructions | Honcho status | local instructions |
| `prefetch()` | read local JSON | Honcho query API | local search |
| `queue_prefetch()` | no-op | background Honcho query | local async |
| `sync_turn()` | append JSON | Honcho turn API | local append |
| `on_session_end()` | extract summary | Honcho session API | local extract |
| `on_pre_compress()` | no-op | extract from compressed | local extract |
| `on_delegation()` | no-op | observe delegation | local log |
| `get_tool_schemas()` | memory_read/write/search | Honcho-specific tools | local tools |

### Table 4: StreamingScrubber Comparison (Think vs Context)

| Dimension | StreamingThinkScrubber | StreamingContextScrubber |
|-----------|------------------------|--------------------------|
| Location | agent/think_scrubber.py | agent/memory_manager.py |
| Tags | `setSizeF`, `<thinking>`, `<reasoning>`, `<thought>`, `<REASONING_SCRATCHPAD>` | `<memory-context>`, `</memory-context>` |
| Purpose | Prevent reasoning leaking to user | Prevent memory context leaking to LLM output |
| State | `_in_block: bool`, `_buf: str` | `_in_span: bool`, `_buf: str` |
| feed logic | Detect open/close tags, hold-back partial | Detect open/close tags, hold-back partial |
| flush logic | Discard unterminated, emit held | Discard unterminated span, emit held |
| boundary rule | Open tag only on new line/stream start | No special boundary rule |
| reset | per-turn | per-turn |
| Use case | run_agent.py stream delta | memory manager stream output |

### Table 5: ContextReference Reference Type Comparison

| Reference Type | Syntax | Expansion Behavior | Security Restriction |
|----------------|--------|--------------------|----------------------|
| @file:path | Read file content | Supports :line-range | sensitive dirs/files blocked |
| @folder:path | List directory tree | limit 200 entries | same as file |
| @git:ref | git show/diff | commit hash validated | none |
| @url:URL | HTTP fetch | content extraction | none |
| @diff | staged diff | git diff HEAD | none |
| @staged | staged files list | git status --short | none |

---

## Summary Table

| Component Name | Lines | Responsibility |
|----------------|-------|---------------|
| `agent/context_engine.py` ContextEngine ABC | 206 | Pluggable context engine base class, 3 core + 8 optional methods, token state, lifecycle |
| `agent/context_compressor.py` ContextCompressor | 1,481 | 5-step compression algorithm, anti-thrashing, aux model fallback, model switch recalc |
| `agent/prompt_builder.py` PromptBuilder | 1,186 | System prompt assembly, 10 threat patterns, 2-layer skill cache, context file load |
| `agent/curator.py` Curator | 1,674 | Background skill self-review, forked review agent, lifecycle transition, LLM classification |
| `agent/memory_provider.py` MemoryProvider ABC | 279 | Pluggable memory backend base class, 10+ lifecycle methods, 5 optional hooks, config schema |
| `agent/memory_manager.py` MemoryManager | 555 | Memory orchestrator, one-external-provider limit, StreamingContextScrubber, tool routing |
| `agent/think_scrubber.py` StreamingThinkScrubber | 386 | Streaming reasoning tag state machine, partial tag hold-back, boundary rules |
| `agent/context_references.py` ContextReference | 518 | @-reference parsing & expansion, 6 reference types, sensitive path protection |

---

[← 08-State & Persistence](/en/chapters/08-state-persistence) | [→ 10-Security Architecture](/en/chapters/10-security)