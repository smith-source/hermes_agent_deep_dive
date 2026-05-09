# 09 — 上下文与记忆引擎: 从压缩摘要到跨会话记忆的多层上下文管理

[← 08-状态与持久化](/zh-CN/chapters/08-state-persistence) | [→ 10-安全体系](/zh-CN/chapters/10-security)

---

## 源码文件

| 文件 | 行数 | 说明 |
|------|------|------|
| `agent/context_engine.py` | 206 | ContextEngine ABC — 可插拔上下文引擎的抽象基类 |
| `agent/context_compressor.py` | 1,481 | ContextCompressor — 中间轮次摘要压缩实现 |
| `agent/prompt_builder.py` | 1,186 | PromptBuilder — 系统提示组装: identity + platform + skills + memory + tools + guardrails |
| `agent/curator.py` | 1,674 | Curator — 后台技能自审查理器 (forked review agent) |
| `agent/memory_provider.py` | 279 | MemoryProvider ABC — 可插拔记忆后端基类 |
| `agent/memory_manager.py` | 555 | MemoryManager — 记忆编排器, 限制一个外部 provider |
| `agent/think_scrubber.py` | 386 | StreamingThinkScrubber — 流式 reasoning 标签状态机 |
| `agent/context_references.py` | 518 | ContextReference — @file/url/diff/git 引用解析与展开 |

---

## 一句话总结

Hermes 的上下文管理由三层引擎构成: 上下文压缩层 (ContextEngine ABC → ContextCompressor, 中间轮次摘要) 控制对话窗口; 提示组装层 (PromptBuilder, 组合 identity/platform/skills/memory/tools/guardrails) 构建系统提示; 记忆层 (MemoryProvider ABC → MemoryManager, Honcho/local/hybrid backends) 跨会话持久化, 加上 Curator 自审查理和 StreamingThinkScrubber 流式标签清洗。

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

Hermes 的上下文与记忆引擎由六个核心模块协同工作。ContextEngine ABC (206 行) 定义了可插拔上下文引擎的接口 — update_from_response, should_compress, compress 是三个核心方法。ContextCompressor (1,481 行) 实现了五步压缩算法: prune old tool results -> protect head -> protect tail -> summarize middle -> iterative update, 带 anti-thrashing 保护。PromptBuilder (1,186 行) 组装系统提示的六大层: identity, platform hints, skills index, memory context, tools, guardrails, 含两层 skill 缓存 (in-process LRU + disk snapshot)。Curator (1,674 行) 是后台技能自审查理器, fork 一个 review agent 进行 pin/archive/consolidate/patch 决策。MemoryProvider ABC (279 行) 定义了 10+ 生命周期方法 (initialize, prefetch, sync_turn, on_session_end, on_pre_compress 等), 限制一个外部 provider。StreamingThinkScrubber (386 行) 是流式 reasoning 标签的增量状态机, 处理 delta 边界的 partial tag hold-back。

---

## 1. ContextEngine ABC — 可插拔上下文引擎

### 1.1 核心接口定义

ContextEngine 定义了所有上下文引擎必须实现的三个核心方法 — `name`, `update_from_response`, `should_compress`, `compress` — 以及一系列可选方法:

源码位置: agent/context_engine.py:32-207

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

### 1.2 生命周期

引擎的完整生命周期:
1. Engine instantiated and registered (plugin `register()` or default)
2. `on_session_start()` called when conversation begins
3. `update_from_response()` called after each API response with usage data
4. `should_compress()` checked after each turn
5. `compress()` called when `should_compress()` returns True
6. `on_session_end()` called at real session boundaries (CLI exit, `/reset`, gateway session expiry)

配置选择: `context.engine` in `config.yaml`, 默认 `"compressor"` (内置), 仅一个引擎可激活。

---

## 2. ContextCompressor — 中间轮次摘要压缩

### 2.1 五步压缩算法

ContextCompressor 继承 ContextEngine, 实现了经典的 "head + tail + summarized middle" 算法:

源码位置: agent/context_compressor.py:322-351

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

### 2.2 初始化与 Token Budget 计算

ContextCompressor 的初始化推导三层 token budget — threshold_tokens (触发点), tail_token_budget (保护尾部), max_summary_tokens (摘要上限):

源码位置: agent/context_compressor.py:380-462

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

### 2.3 Anti-Thrashing 保护

连续两次压缩各省不到 10% 时, 停止压缩以避免无限循环 (每次只删 1-2 条消息):

源码位置: agent/context_compressor.py:469-495

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

当用户配置的摘要模型 (aux model) 不可用时, ContextCompressor 实现两级回退: 404/not-found 时立即重试主模型; 其他错误时也尝试主模型回退, 但记录 `_last_aux_model_failure_error` 以通知用户配置问题:

源码位置: agent/context_compressor.py:930-981

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

### 2.5 Model Switch 支持

`update_model()` 在模型切换时重新计算所有 token budget, 保持压缩机在不同上下文长度 (如 200K → 32K) 下正确校准:

源码位置: agent/context_compressor.py:352-379

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

## 3. PromptBuilder — 系统提示组装

### 3.1 上下文威胁扫描

PromptBuilder 在注入 AGENTS.md、.cursorrules、SOUL.md 前扫描提示注入攻击模式, 包含 10 种检测规则:

源码位置: agent/prompt_builder.py:36-50

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

还检测隐形 Unicode 字符 (`_CONTEXT_INVISIBLE_CHARS`), 防止零宽字符注入。

### 3.2 Skills 系统提示 — 两层缓存

`build_skills_system_prompt()` 使用两层缓存加速技能索引构建: (1) in-process LRU dict, (2) disk snapshot (`_skills_prompt_snapshot.json`) validated by mtime/size manifest:

源码位置: agent/prompt_builder.py:718-764

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

### 3.3 上下文文件加载

PromptBuilder 从多个来源加载上下文文件 — SOUL.md, AGENTS.md, .claude/CLAUDE.md, .cursorrules, 每个都经过威胁扫描:

源码位置: agent/prompt_builder.py:1034-1117

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

## 4. Curator — 后台技能自审查理

### 4.1 设计理念

Curator 是一个不活跃触发的 (no cron daemon) 后台任务: 当 agent idle 且上次 curator 运行超过 `interval_hours` 时, `maybe_run_curator()` spawn 一个 forked AIAgent 做 review:

源码位置: agent/curator.py:1-60

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

### 4.2 Review Runtime 绑定

Curator 使用 `_ReviewRuntimeBinding` 记录 provider/model/api_key/base_url, 支持 per-slot overrides:

源码位置: agent/curator.py:47-54

```python
class _ReviewRuntimeBinding(NamedTuple):
    """Provider/model for the curator review fork plus optional per-slot overrides."""
    provider: str
    model: str
    explicit_api_key: Optional[str]
    explicit_base_url: Optional[str]
```

### 4.3 自动状态转换

`apply_automatic_transitions()` 根据 skill 的 last-used timestamps 自动推进生命周期状态:

源码位置: agent/curator.py:255-450

```python
def apply_automatic_transitions(now: Optional[datetime] = None) -> Dict[str, int]:
    """Auto-transition lifecycle states based on skill activity timestamps."""
    ...
```

### 4.4 LLM Review 与分类

`run_curator_review()` fork 一个 review agent, 使用 LLM 对技能进行 structured classification (pin/archive/consolidate/patch/keep):

源码位置: agent/curator.py:1278-1496

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

## 5. MemoryProvider ABC — 可插拔记忆后端

### 5.1 核心生命周期方法

MemoryProvider 定义了 10+ 方法, 从 initialize 到 shutdown, 涵盖 session 级和 turn 级两个维度:

源码位置: agent/memory_provider.py:42-280

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

### 5.2 Optional Hooks — 会话/压缩/委派

MemoryProvider 提供 5 个可选 hooks 用于深度集成:

| Hook | 何时调用 | 用途 |
|------|----------|------|
| `on_turn_start(turn, message, **kwargs)` | 每轮开始 | turn-counting, scope management |
| `on_session_end(messages)` | 真正的会话结束 (非每轮) | fact extraction, summarization |
| `on_session_switch(new_id, parent_id, reset)` | session_id 变化 (/resume, /branch, compression) | 更新或重置 per-session 缓存 |
| `on_pre_compress(messages)` | 压缩前 | 从即将丢弃的消息中提取洞察 |
| `on_delegation(task, result, child_id)` | 子代理完成 | 父代理观察委派结果 |
| `on_memory_write(action, target, content, metadata)` | 内置 memory 工具写入 | 镜像写入到外部 backend |
| `get_config_schema()` | memory setup | 返回配置字段定义 |
| `save_config(values, hermes_home)` | memory setup | 写非秘密配置到 provider 原生位置 |

### 5.3 Context Fencing

MemoryProvider 的输出通过 `<memory-context>` 标签围栏, 带有 system note 防止 LLM 混淆记忆和新输入:

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

## 6. MemoryManager — 记忆编排器

### 6.1 One-External-Provider 限制

MemoryManager 强制只允许一个外部 provider, 防止 tool schema bloat 和 backend 冲突:

源码位置: agent/memory_manager.py:190-248

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

### 6.2 StreamingContextScrubber — 流式记忆围栏清洗

`StreamingContextScrubber` (memory_manager.py:62-170) 是有状态的增量状态机, 处理流式 delta 中的 `<memory-context>` 围栏。对于跨 delta 边界的 split 标签, 它 hold-back partial tag tail 并在下一个 `feed()` 或 `flush()` 时解决:

源码位置: agent/memory_manager.py:62-140

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

## 7. StreamingThinkScrubber — 流式 Reasoning 标签清洗

### 7.1 设计动机

`_strip_think_blocks` 的 regex 方法对完整字符串正确, 但在 per-delta 模式下会破坏下游状态机 — MiniMax-M2.7 流式 `<think>`...`</think>` 时, per-delta regex 在 delta 边界处抹除 open tag, 导致 reasoning 泄漏到用户。StreamingThinkScrubber 在上游层面集中处理, 所有 stream_delta_callback 看到的文本已经去除了 reasoning blocks:

源码位置: agent/think_scrubber.py:1-50

```python
"""Stateful scrubber for reasoning/thinking blocks in streamed assistant text.

...when MiniMax-M2.7 streams
    delta1 = "<think>"
    delta2 = "Let me check their config"
    delta3 = "</think>"
the per-delta regex erases delta1 entirely, so the downstream state machine
never sees the open tag, treats delta2 as regular content, and leaks reasoning
to the user.

This module centralises the tag-suppression state machine at the upstream
layer so every stream_delta_callback sees text that has already had reasoning
blocks removed.

Tag variants handled (case-insensitive):
  <think>, <thinking>, <reasoning>, <thought>, <REASONING_SCRATCHPAD>.
"""
```

### 7.2 Block-Boundary 规则

Opening tag 只在以下位置被视为 reasoning-block opener: stream 开始、换行后 (可跟空白)、或当前行只有空白。这防止 prose 中的 `<think>` 被 误判:

源码位置: agent/think_scrubber.py:46-50

```
Block-boundary rule for opens: an opening tag is only treated as a
reasoning-block opener when it appears at the start of the stream,
after a newline (optionally followed by whitespace), or when only
whitespace has been emitted on the current line.  This prevents prose
```

---

## 8. ContextReference — @引用解析与展开

### 8.1 数据模型

`ContextReference` 是 frozen dataclass, 记录引用的原始文本、类型、目标、位置:

源码位置: agent/context_references.py:40-50

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

### 8.2 引用解析

`parse_context_references()` 使用 regex 模式识别 @file:, @folder:, @git:, @url:, @diff, @staged 引用:

源码位置: agent/context_references.py:16-19, 62

```python
_QUOTED_REFERENCE_VALUE = r'(?:`[^`\n]+`|"[^"\n]+"|\'[^\'\n]+\')'
REFERENCE_PATTERN = re.compile(
    rf"(?<![\w/])@(?:(?P<simple>diff|staged)\b|(?P<kind>file|folder|git|url):(?P<value>{_QUOTED_REFERENCE_VALUE}(?::\d+(?:-\d+)?)?|\S+))"
)

def parse_context_references(message: str) -> list[ContextReference]:
    """Parse @-references from a user message."""
    ...
```

### 8.3 引用展开

每个引用类型有独立的展开函数:

| 函数 | 行数 | 职责 |
|------|------|------|
| `_expand_file_reference()` | 236-262 | 读取文件内容, 支持行范围 (:10-20) |
| `_expand_folder_reference()` | 263-280 | 列出目录内容, 限 200 条 |
| `_expand_git_reference()` | 280-305 | git show/diff 输出 |
| `_fetch_url_content()` | 305-328 | HTTP 获取 URL 内容 |
| `_build_folder_listing()` | 430-446 | 目录树构建 |
| `_file_metadata()` | 494-504 | 文件元信息 (size, mtime) |
| `_code_fence_language()` | 504-... | 根据扩展名推断代码围栏语言 |
| `_is_binary_file()` | 420-430 | 二进制文件检测 |

### 8.4 安全限制

ContextReference 实现敏感路径保护 — `.ssh`, `.aws`, `.gnupg`, `.kube`, `.docker`, `.azure`, `.config/gh` 等目录下的文件被拒绝展开:

源码位置: agent/context_references.py:21-37

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

### 表 1: 上下文引擎接口对比 (ContextEngine ABC vs 实现)

| 方法 | ContextEngine ABC | ContextCompressor |
|------|-------------------|-------------------|
| `name` | abstract | "compressor" |
| `update_from_response` | abstract | token tracking |
| `should_compress` | abstract | threshold + anti-thrashing |
| `compress` | abstract | 5-step summarize |
| `should_compress_preflight` | default False | 未 override |
| `has_content_to_compress` | default True | override (check boundaries) |
| `on_session_start` | default no-op | 未 override |
| `on_session_end` | default no-op | 未 override |
| `on_session_reset` | reset tokens | reset + clear summary |
| `get_tool_schemas` | default [] | [] |
| `handle_tool_call` | default error | N/A |
| `update_model` | update threshold | recalc all budgets |

### 表 2: 系统提示组成层对比

| 层 | 来源 | PromptBuilder 函数 | 可缓存 |
|----|------|--------------------|---------|
| identity | SOUL.md | `load_soul_md()` | 否 (per-project) |
| platform hints | runtime | `build_environment_hints()` | 否 (dynamic) |
| skills index | ~/.hermes/skills/ + external | `build_skills_system_prompt()` | 两层 LRU + disk snapshot |
| memory context | MemoryProvider.prefetch() | `build_memory_context_block()` | provider-managed |
| tools | tools.registry | get_tool_definitions() | 否 (dynamic toolset) |
| context files | AGENTS.md, CLAUDE.md, .cursorrules | `build_context_files_prompt()` | 否 (per-project) |
| guardrails | threat patterns | `_scan_context_content()` | 否 (security scan) |
| nous subscription | runtime | `build_nous_subscription_prompt()` | 否 (dynamic) |

### 表 3: MemoryProvider 生命周期方法对比

| 方法 | builtin provider | Honcho provider | local/hybrid |
|------|-----------------|-----------------|-------------|
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

### 表 4: StreamingScrubber 对比 (Think vs Context)

| 维度 | StreamingThinkScrubber | StreamingContextScrubber |
|------|------------------------|--------------------------|
| 位置 | agent/think_scrubber.py | agent/memory_manager.py |
| 标签 | `<think>`, `<thinking>`, `<reasoning>`, `<thought>`, `<REASONING_SCRATCHPAD>` | `<memory-context>`, `</memory-context>` |
| 目的 | 防止 reasoning 泄漏到用户 | 防止 memory context 泄漏到 LLM 输出 |
| 状态 | `_in_block: bool`, `_buf: str` | `_in_span: bool`, `_buf: str` |
| feed 逻辑 | 检测 open/close tags, hold-back partial | 检测 open/close tags, hold-back partial |
| flush 逻辑 | discard unterminated, emit held | discard unterminated span, emit held |
| boundary 规则 | open tag 只在新行/stream start | 无特殊 boundary 规则 |
| reset | per-turn | per-turn |
| 使用场景 | run_agent.py stream delta | memory manager stream output |

### 表 5: ContextReference 引用类型对比

| 引用类型 | 语法 | 展开行为 | 安全限制 |
|----------|------|----------|----------|
| @file:path | 读取文件内容 | 支持 :line-range | sensitive dirs/files blocked |
| @folder:path | 列出目录树 | limit 200 entries | same as file |
| @git:ref | git show/diff | commit hash validated | none |
| @url:URL | HTTP fetch | content extraction | none |
| @diff | staged diff | git diff HEAD | none |
| @staged | staged files list | git status --short | none |

---

## Summary Table

| 组件名 | 行数 | 职责 |
|--------|------|------|
| `agent/context_engine.py` ContextEngine ABC | 206 | 可插拔上下文引擎基类, 3 core + 8 optional 方法, token state, lifecycle |
| `agent/context_compressor.py` ContextCompressor | 1,481 | 5-step 压缩算法, anti-thrashing, aux model fallback, model switch recalc |
| `agent/prompt_builder.py` PromptBuilder | 1,186 | 系统提示组装, 10 种 threat pattern, 2-layer skill cache, context file load |
| `agent/curator.py` Curator | 1,674 | 后台技能自审查, forked review agent, lifecycle transition, LLM classification |
| `agent/memory_provider.py` MemoryProvider ABC | 279 | 可插拔记忆后端基类, 10+ lifecycle methods, 5 optional hooks, config schema |
| `agent/memory_manager.py` MemoryManager | 555 | 记忆编排器, one-external-provider 限制, StreamingContextScrubber, tool routing |
| `agent/think_scrubber.py` StreamingThinkScrubber | 386 | 流式 reasoning 标签状态机, partial tag hold-back, boundary rules |
| `agent/context_references.py` ContextReference | 518 | @引用解析与展开, 6 种引用类型, sensitive path protection |

---

[← 08-状态与持久化](/zh-CN/chapters/08-state-persistence) | [→ 10-安全体系](/zh-CN/chapters/10-security)