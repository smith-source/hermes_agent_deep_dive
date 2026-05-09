# 20 — Agent Support Modules: Insights Analysis/Display Rendering/Shell Hooks/Usage Billing/OAuth/Onboarding/ACP Integration

## Source Files

| File | Lines | Core Abstraction |
|------|------:|------------------|
| `agent/insights.py` | 930 | `InsightsEngine` |
| `agent/display.py` | 1002 | `KawaiiSpinner`, `LocalEditSnapshot` |
| `agent/shell_hooks.py` | 836 | `ShellHookSpec` |
| `agent/usage_pricing.py` | 721 | `CanonicalUsage`, `CostResult` |
| `agent/subdirectory_hints.py` | 224 | `SubdirectoryHintTracker` |
| `agent/onboarding.py` | 193 | First-touch hint flags |
| `agent/google_oauth.py` | 1061 | `GoogleCredentials`, PKCE flow |
| `agent/copilot_acp_client.py` | 646 | `CopilotACPClient` |
| `agent/skill_commands.py` | 501 | Slash-command scanner/builder |
| `agent/account_usage.py` | 326 | `AccountUsageSnapshot` |
| `agent/title_generator.py` | 171 | `maybe_auto_title()` |
| `agent/rate_limit_tracker.py` | 246 | `RateLimitState`, `RateLimitBucket` |
| `agent/i18n.py` | 233 | `t()` translation function |
| `agent/nous_rate_guard.py` | 325 | Cross-session rate-limit breaker |
| `agent/manual_compression_feedback.py` | 49 | `summarize_manual_compression()` |
| `agent/think_scrubber.py` | 386 | `StreamingThinkScrubber` |
| `agent/redact.py` | 400 | `RedactingFormatter`, `redact_sensitive_text()` |
| `agent/file_safety.py` | 111 | `is_write_denied()`, `get_read_block_error()` |
| `agent/retry_utils.py` | 57 | `jittered_backoff()` |
| `agent/prompt_caching.py` | 72 | `apply_anthropic_cache_control()` |

**Total: 8,490 lines across 20 modules**

---

## One-Liner

Twenty supporting modules that provide insights analytics, terminal display, shell automation, cost estimation, OAuth authentication, editor integration, rate-limit protection, secret redaction, and i18n — everything AIAgent needs beyond the core inference loop.

---

## Architecture Overview

```
                          ┌─────────────────────┐
                          │      AIAgent        │
                          │   (core loop)       │
                          └──────┬──────────────┘
                                 │
         ┌───────────────────────┼───────────────────────────┐
         │                       │                           │
    ┌────▼─────┐          ┌─────▼──────┐            ┌───────▼──────┐
    │ Analysis  │          │  Display   │            │   Safety     │
    │ Layer     │          │  Layer     │            │   Layer      │
    ├──────────┤          ├───────────┤            ├─────────────┤
    │ insights  │          │ display   │            │ redact      │
    │ usage_    │          │ onboarding│            │ file_safety │
    │  pricing  │          │ i18n      │            │ think_      │
    │ account_  │          │ title_    │            │  scrubber   │
    │  usage    │          │  generator│            │ retry_utils │
    │ rate_     │          └───────────┘            └─────────────┘
    │  limit_   │
    │  tracker  │    ┌──────────────┐     ┌──────────────────┐
    │ nous_     │    │  Automation  │     │   Integration    │
    │  rate_    │    │  Layer       │     │   Layer          │
    │  guard    │    ├─────────────┤     ├──────────────────┤
    └──────────┘    │ shell_hooks │     │ google_oauth     │
                    │ skill_      │     │ copilot_acp_     │
                    │  commands   │     │  client          │
                    └─────────────┘     └──────────────────┘

    ┌──────────────────┐    ┌──────────────────────────────┐
    │   Context Layer  │    │   Cache Layer                │
    ├──────────────────┤    ├──────────────────────────────┤
    │ subdirectory_    │    │ prompt_caching               │
    │  hints           │    │ manual_compression_feedback  │
    └──────────────────┘    └──────────────────────────────┘
```

---

## TL;DR

These 20 modules form the "shoulders" of AIAgent — everything the core loop delegates out to keep `agent.py` focused on orchestration. **InsightsEngine** queries SessionDB to produce usage reports with cost estimation; **KawaiiSpinner** and the diff renderer give CLI users rich visual feedback. **ShellHookSpec** bridges YAML-declared shell scripts into the Python plugin system with consent gating. **usage_pricing** maintains a 28-model pricing table and normalizes three API usage shapes into one `CanonicalUsage`. **SubdirectoryHintTracker** lazily discovers AGENTS.md files as the agent navigates. **google_oauth** implements full PKCE OAuth2 with cross-process locking; **CopilotACPClient** wraps the ACP JSON-RPC protocol as an OpenAI-compatible client. The safety chain — **think_scrubber** (streaming state machine), **redact** (13 secret patterns), and **file_safety** (write denylist) — ensures sensitive data never leaks to logs or the user.

---

## 1. InsightsEngine — Session Insights Analysis

`Source location: agent/insights.py:93-930`

### 1.1 Core Architecture

InsightsEngine directly holds `SessionDB._conn`, retrieves data through pre-compiled SQL queries, and completes aggregation calculations in memory.

```python
class InsightsEngine:
    def __init__(self, db):
        self.db = db
        self._conn = db._conn

    def generate(self, days: int = 30, source: str = None) -> Dict[str, Any]:
        cutoff = time.time() - (days * 86400)
        sessions = self._get_sessions(cutoff, source)
        tool_usage = self._get_tool_usage(cutoff, source)
        skill_usage = self._get_skill_usage(cutoff, source)
        message_stats = self._get_message_stats(cutoff, source)
        # ... compute breakdowns ...
```

`Source location: agent/insights.py:180-184`

SQL queries use pre-computed constants to avoid runtime string concatenation injection:

```python
_SESSION_COLS = ("id, source, model, started_at, ended_at, "
                 "message_count, tool_call_count, input_tokens, output_tokens, ...")
_GET_SESSIONS_ALL = f"SELECT {_SESSION_COLS} FROM sessions WHERE started_at >= ?"
```

### 1.2 Dual-Source Tool Counting

`Source location: agent/insights.py:207-297`

Tool usage statistics are merged from two SQL sources, taking the maximum count per tool to avoid double-counting:

| Source | SQL Target | Coverage Scenario |
|--------|-----------|-------------------|
| Source 1: `tool_name` column | `messages WHERE role='tool'` | Gateway sessions (gateway writes tool_name) |
| Source 2: `tool_calls` JSON | `messages WHERE role='assistant'` | CLI sessions (tool_name is NULL) |

### 1.3 Seven-Dimension Computation Pipeline

```
generate() → _compute_overview()        # Totals/cost/duration
           → _compute_model_breakdown() # Bucket by model
           → _compute_platform_breakdown() # Bucket by source
           → _compute_tool_breakdown()  # Tool usage ranking
           → _compute_skill_breakdown() # Skill usage ranking
           → _compute_activity_patterns() # Day/hour distribution + streak days
           → _compute_top_sessions()    # Longest/most messages/most tokens
```

`Source location: agent/insights.py:606-662`

Activity pattern analysis includes consecutive usage day (streak) calculation:

```python
# Streak calculation
all_dates = sorted(daily_counts.keys())
current_streak = 1
max_streak = 1
for i in range(1, len(all_dates)):
    d1 = datetime.strptime(all_dates[i - 1], "%Y-%m-%d")
    d2 = datetime.strptime(all_dates[i], "%Y-%m-%d")
    if (d2 - d1).days == 1:
        current_streak += 1
        max_streak = max(max_streak, current_streak)
    else:
        current_streak = 1
```

### 1.4 Dual-Format Output

| Format | Method | Target | Features |
|--------|--------|--------|----------|
| Terminal | `format_terminal()` | CLI `/insights` command | Unicode box drawing, bar chart, full detail |
| Gateway | `format_gateway()` | Discord/Telegram | Markdown, top-5/8 truncation, compact |

---

## 2. Display — Terminal Rendering Engine

`Source location: agent/display.py:1-1002`

### 2.1 KawaiiSpinner

`Source location: agent/display.py:573-798`

9 built-in spinner animations + kawaii expression system:

```python
SPINNERS = {
    'dots': ['⠋', '⠙', '⠹', '⠸', '⠼', '⠴', '⠦', '⠧', '⠇', '⠏'],
    'brain': ['🧠', '💭', '💡', '✨', '💫', '🌟', '💡', '💭'],
    'moon': ['🌑', '🌒', '🌓', '🌔', '🌕', '🌖', '🌗', '🌘'],
    # ... 6 more
}
KAWAII_WAITING = ["(｡◕‿◕｡)", "(◕‿◕✿)", "٩(◕‿◕｡)۶", ...]
KAWAII_THINKING = ["(｡•́︿•̀｡)", "(◔_◔)", "(¬‿¬)", ...]
```

Skin engine integration: reads from `skin.spinner.waiting_faces` / `thinking_faces` / `thinking_verbs` first, falls back to hardcoded values on failure.

Triple output strategy:

| Environment | Behavior |
|-------------|----------|
| Real TTY | `\r` overwrite animation, 0.12s interval |
| `prompt_toolkit` StdoutProxy | Skip animation (avoid line-break chaos), TUI widget takes over |
| Non-TTY (pipe/Docker) | Output `[tool] {message}` once only, avoid log bloat |

```python
def _animate(self):
    if not self._is_tty:
        self._write(f"  [tool] {self.message}", flush=True)
        while self.running:
            time.sleep(0.5)
        return
    if self._is_patch_stdout_proxy():
        while self.running:
            time.sleep(0.1)
        return
    # ... TTY animation loop ...
```

`Source location: agent/display.py:703-742`

### 2.2 LocalEditSnapshot — Local Diff Preview

`Source location: agent/display.py:90-433`

Captures file snapshot before write tool execution, generates unified diff after execution:

```
capture_local_edit_snapshot(tool_name, args)  # Before write
    ↓
LocalEditSnapshot(paths=[...], before={"path": "old content"})
    ↓
extract_edit_diff(tool_name, result, snapshot=snapshot)  # After write
    ↓
render_edit_diff_with_delta(...)  # Render to terminal
```

Supports path resolution for three write tools:

| Tool | Path Extraction Logic |
|------|-----------------------|
| `write_file` | `args["path"]` |
| `patch` | `args["path"]` |
| `skill_manage` | Parse `SKILLS_DIR` based on action type (create/edit/delete) |

Diff rendering uses skin-aware ANSI colors, with file/line count limits (6 files / 80 lines).

### 2.3 Tool Completion Line

`Source location: agent/display.py:837-996`

`get_cute_tool_message()` generates a one-line summary for each tool, format: `┊ {emoji} {verb:9} {detail}  {duration}`

30+ tool-specific formatters + generic fallback:

```python
if tool_name == "terminal":
    return _wrap(f"┊ 💻 $         {_trunc(args.get('command', ''), 42)}  {dur}")
if tool_name == "web_search":
    return _wrap(f"┊ 🔍 search    {_trunc(args.get('query', ''), 42)}  {dur}")
# ... 28 more specialized formatters ...
# Fallback:
preview = build_tool_preview(tool_name, args) or ""
return _wrap(f"┊ ⚡ {tool_name[:9]:9} {_trunc(preview, 35)}  {dur}")
```

Failure detection: terminal tools check `exit_code != 0`, other tools check for `"error"` / `"failed"` keywords in JSON. Failed lines get a red prefix and error message appended.

---

## 3. ShellHookSpec — Shell Hook Bridge

`Source location: agent/shell_hooks.py:106-837`

### 3.1 Declarative Configuration to Runtime Callbacks

Users declare hooks in `cli-config.yaml`, and the module parses and registers them with the Python plugin manager:

```yaml
hooks:
  pre_tool_call:
    - command: /usr/local/bin/check-command.sh
      matcher: "terminal"
      timeout: 30
  post_tool_call:
    - command: /usr/local/bin/log-result.sh
  before_send:
    - command: /usr/local/bin/inject-context.sh
```

Parsed into `ShellHookSpec` dataclass:

```python
@dataclass
class ShellHookSpec:
    event: str
    command: str
    matcher: Optional[str] = None       # regex, only for pre/post_tool_call
    timeout: int = DEFAULT_TIMEOUT_SECONDS  # 60s, max 300s
    compiled_matcher: Optional[re.Pattern] = field(default=None, repr=False)
```

`Source location: agent/shell_hooks.py:106-131`

### 3.2 Subprocess Execution Pipeline

```
invoke_hook(event, **kwargs)
    ↓
_make_callback(spec)(**kwargs)
    ↓
1. matcher gating (tool events only)
2. _serialize_payload(event, kwargs) → JSON stdin
3. _spawn(spec, stdin_json)
   ├── shlex.split(os.path.expanduser(command))
   ├── subprocess.run(argv, input=stdin_json, shell=False)
   └── timeout / FileNotFoundError / PermissionError handling
4. _parse_response(event, stdout) → Hermes wire-shape
```

`Source location: agent/shell_hooks.py:421-462`

**Key safety design**: `shell=False` prevents shell injection; users who need pipes/redirections must wrap them in a script.

### 3.3 Wire Protocol

Stdin (JSON):

```json
{
    "hook_event_name": "pre_tool_call",
    "tool_name": "terminal",
    "tool_input": {"command": "rm -rf /"},
    "session_id": "sess_abc123",
    "cwd": "/home/user/project",
    "extra": {}
}
```

Stdout (JSON, two compatible formats):

```json
// Claude-Code-style (block):
{"decision": "block", "reason": "Forbidden command"}
// Hermes-canonical (block):
{"action": "block", "message": "Forbidden command"}
// Context injection (pre_llm_call):
{"context": "Today is Friday"}
```

`_parse_response` uniformly translates `decision: "block"` to `action: "block"` — this is the most important correctness invariant of the entire module.

### 3.4 Three-Layer Consent Mechanism

| Layer | Source | Priority |
|-------|--------|----------|
| CLI flag | `--accept-hooks` | Highest |
| Environment variable | `HERMES_ACCEPT_HOOKS=1` | Medium |
| Config file | `hooks_auto_accept: true` | Lowest |

Allowlist is persisted to `~/.hermes/shell-hooks-allowlist.json`, using `fcntl.flock` for cross-process serialized writes. It records `(event, command)` pairs along with approval timestamp and script mtime.

---

## 4. UsagePricing — Token Billing Engine

`Source location: agent/usage_pricing.py:1-721`

### 4.1 Data Model

Three-layer immutable dataclass chain:

```
CanonicalUsage (token buckets)
    ↓ + model_name
BillingRoute (provider + billing_mode)
    ↓
PricingEntry (per-million rates + source)
    ↓
CostResult (amount_usd + status + label)
```

`Source location: agent/usage_pricing.py:28-77`

`CanonicalUsage` normalizes four token buckets:

```python
@dataclass(frozen=True)
class CanonicalUsage:
    input_tokens: int = 0
    output_tokens: int = 0
    cache_read_tokens: int = 0
    cache_write_tokens: int = 0
    reasoning_tokens: int = 0
    request_count: int = 1
```

### 4.2 Three-Source Pricing Resolution

`Source location: agent/usage_pricing.py:486-513`

```
get_pricing_entry(model, provider, base_url)
    ↓
1. resolve_billing_route() → BillingRoute
    ├── openai-codex → subscription_included (zero rate)
    ├── openrouter → _openrouter_pricing_entry() (models API)
    ├── anthropic/openai → _lookup_official_docs_pricing() (hardcoded snapshot)
    └── custom/localhost → unknown
2. For routes with base_url → fetch_endpoint_model_metadata() (OpenAI-compatible /models)
3. Fallback → _OFFICIAL_DOCS_PRICING dict lookup
```

### 4.3 Official Pricing Snapshot

`Source location: agent/usage_pricing.py:84-381`

The `_OFFICIAL_DOCS_PRICING` dictionary covers 28+ models, keyed by `(provider, model)` tuple:

| Provider | Models | Sample Rate (input/M) |
|----------|--------|----------------------|
| Anthropic | claude-opus-4, sonnet-4, 3.5-sonnet, 3-opus, 3-haiku | $0.25 – $15.00 |
| OpenAI | gpt-4o, gpt-4o-mini, gpt-4.1, gpt-4.1-mini/nano, o3, o3-mini | $0.10 – $10.00 |
| DeepSeek | deepseek-chat, deepseek-reasoner | $0.14 – $0.55 |
| Google | gemini-2.5-pro, gemini-2.5-flash, gemini-2.0-flash | $0.10 – $1.25 |
| AWS Bedrock | claude-opus-4-6, sonnet-4-6, nova-pro/lite/micro | $0.035 – $15.00 |
| MiniMax | minimax-m2.7 | $0.30 |

Each entry includes `source_url`, `pricing_version`, `fetched_at` for audit traceability.

### 4.4 Three API Usage Format Normalization

`Source location: agent/usage_pricing.py:516-586`

`normalize_usage()` handles three API response shapes:

| API Mode | Total Input Field | Cache Separation | Logic |
|----------|-------------------|-----------------|-------|
| `anthropic_messages` | `input_tokens` (cache excluded) | `cache_read_input_tokens` + `cache_creation_input_tokens` top-level fields | Direct mapping |
| `codex_responses` | `input_tokens` (includes cache) | `input_tokens_details.cached_tokens` | `input = total - cache_read - cache_write` |
| OpenAI (default) | `prompt_tokens` (includes cache) | `prompt_tokens_details.cached_tokens` | `input = total - cache_read - cache_write` |

OpenAI compatible mode also has fallback logic: when `prompt_tokens_details` has no cache data, it checks Anthropic-style top-level fields (certain proxies like OpenRouter/Vercel AI Gateway expose these fields).

---

## 5. SubdirectoryHintTracker — Subdirectory Context Injection

`Source location: agent/subdirectory_hints.py:48-224`

### 5.1 Lazy Discovery Mechanism

Unlike the PromptBuilder which only loads CWD context at startup, SubdirectoryHintTracker detects paths in tool call arguments and loads on demand:

```python
class SubdirectoryHintTracker:
    def __init__(self, working_dir=None):
        self.working_dir = Path(working_dir or os.getcwd()).resolve()
        self._loaded_dirs: Set[Path] = set()
        self._loaded_dirs.add(self.working_dir)  # CWD already loaded at startup

    def check_tool_call(self, tool_name, tool_args) -> Optional[str]:
        dirs = self._extract_directories(tool_name, tool_args)
        all_hints = [self._load_hints_for_directory(d) for d in dirs]
        return "\n\n".join(all_hints) if all_hints else None
```

### 5.2 Path Extraction and Ancestor Traversal

`Source location: agent/subdirectory_hints.py:91-158`

Two methods for extracting paths from tool arguments:

1. **Direct path parameters**: `path`, `file_path`, `workdir` keys
2. **Shell command parsing**: `command` field of `terminal` tool, `shlex.split` then extract path-like tokens

Upward ancestor traversal (up to 5 levels) ensures upper-level AGENTS.md files are discovered when reading deep files:

```python
def _add_path_candidate(self, raw_path, candidates):
    for _ in range(_MAX_ANCESTOR_WALK):
        if p in self._loaded_dirs:
            break  # Already loaded ancestor, stop
        if self._is_valid_subdir(p):
            candidates.add(p)
        p = p.parent
```

### 5.3 Hint File Priority

`Source location: agent/subdirectory_hints.py:29-33`

```python
_HINT_FILENAMES = ["AGENTS.md", "agents.md", "CLAUDE.md", "claude.md", ".cursorrules"]
```

Only the first matching file is loaded per directory (consistent with startup logic), content truncated to 8000 characters. Injected at the end of tool results, **does not modify system prompt**, preserving prompt cache.

---

## 6. Onboarding — First-Use Prompts

`Source location: agent/onboarding.py:1-193`

### 6.1 Behavior Fork Point Design

Three prompt flags, each shown only once when the user first encounters a specific behavior:

| Flag | Trigger Timing | Prompt Content |
|------|---------------|----------------|
| `busy_input_prompt` | User sends a message while Agent is busy | Explains queue/interrupt/steer modes |
| `tool_progress_prompt` | First long-running tool execution | Prompts `/verbose` to toggle progress display |
| `openclaw_residue_cleanup` | `~/.openclaw/` directory detected | Prompts `hermes claw migrate` for migration |

### 6.2 Persistence and Idempotency

```python
def is_seen(config, flag):
    return bool(_get_seen_dict(config).get(flag))

def mark_seen(config_path, flag):
    # Atomic YAML write, using atomic_yaml_write
    seen[flag] = True
    atomic_yaml_write(config_path, cfg)
```

Stored under the `onboarding.seen.<flag>` path in `config.yaml`.

---

## 7. GoogleOAuth — Google OAuth2 PKCE Flow

`Source location: agent/google_oauth.py:1-1061`

### 7.1 Authentication Flow

```
start_oauth_flow()
    ↓
1. Check for existing credentials → skip if present
2. _require_client_id() → get OAuth client (3-level fallback)
3. _generate_pkce_pair() → (verifier, challenge)
4. [Headless?] → _paste_mode_login()
   [GUI?]     → _bind_callback_server() → localhost:8085
5. Browser opens authorization URL
6. Wait for callback → _OAuthCallbackHandler captures code
7. exchange_code(code, verifier, redirect_uri) → token response
8. _persist_token_response() → save_credentials()
```

### 7.2 Client ID Three-Level Fallback

`Source location: agent/google_oauth.py:338-356`

```
_get_client_id()
    ↓
1. HERMES_GEMINI_CLIENT_ID environment variable
2. Hardcoded public client (Google gemini-cli's public OAuth client)
3. _scrape_client_credentials() — extract from locally installed gemini-cli binary
```

### 7.3 RefreshParts — Refresh Token Packed Format

`Source location: agent/google_oauth.py:393-420`

```python
class RefreshParts:
    """Packed refresh token format: refresh_token|project_id|managed_project_id"""
    refresh_token: str
    project_id: str = ""
    managed_project_id: str = ""

    @classmethod
    def parse(cls, packed: str) -> "RefreshParts":
        """Unpack from '|' delimited string."""
        parts = packed.split("|", 2)
        return cls(
            refresh_token=parts[0],
            project_id=parts[1] if len(parts) > 1 else "",
            managed_project_id=parts[2] if len(parts) > 2 else "",
        )

    def format(self) -> str:
        """Pack back to '|' delimited string for storage."""
```

RefreshParts handles the three-segment packed format of Google OAuth refresh tokens (`refresh_token|project_id|managed_project_id`), supporting parsing from and packing back to the storage format. `invalid_grant` errors automatically clear the credentials file, forcing re-login.

### 7.4 Credential Storage and Security

`Source location: agent/google_oauth.py:488-522`

Storage path: `~/.hermes/auth/google_oauth.json` (chmod 0o600)

```python
def save_credentials(creds):
    # O_WRONLY | O_CREAT | O_EXCL + S_IRUSR | S_IWUSR
    # Atomic create temp file → fsync → atomic_replace
    fd = os.open(tmp_path, O_WRONLY | O_CREAT | O_EXCL, S_IRUSR | S_IWUSR)
    with os.fdopen(fd, "w") as fh:
        fh.write(payload)
        fh.flush()
        os.fsync(fh.fileno())
    atomic_replace(tmp_path, path)
```

Parent directory chmod 0o700. Refresh token uses packed format: `refresh_token|project_id|managed_project_id`.

### 7.5 Concurrent Refresh Deduplication

`Source location: agent/google_oauth.py:647-719`

```python
_refresh_inflight: Dict[str, threading.Event] = {}

def get_valid_access_token(*, force_refresh=False):
    with _refresh_inflight_lock:
        event = _refresh_inflight.get(rt)
        if event is None:
            event = threading.Event()
            _refresh_inflight[rt] = event
            owner = True
        else:
            owner = False

    if not owner:
        event.wait(timeout=30)  # Wait for another thread to finish refreshing
        fresh = load_credentials()
        if fresh and not fresh.access_token_expired():
            return fresh.access_token
```

`invalid_grant` errors automatically clear the credentials file, forcing re-login.

---

## 8. CopilotACPClient — GitHub Copilot ACP Integration

`Source location: agent/copilot_acp_client.py:1-646`

### 8.1 OpenAI-Compatible Wrapper

Adapts the ACP JSON-RPC protocol to the OpenAI SDK's `client.chat.completions.create()` interface:

```python
class CopilotACPClient:
    def __init__(self, *, api_key=None, base_url=None, ...):
        self.chat = _ACPChatNamespace(self)

    # External interface consistent with OpenAI SDK
    client = CopilotACPClient(api_key="copilot-acp")
    response = client.chat.completions.create(
        model="copilot-acp",
        messages=[...],
        tools=[...],
    )
```

### 8.2 Request Lifecycle

```
_create_chat_completion(messages, model, tools)
    ↓
1. _format_messages_as_prompt() → convert OpenAI format messages to text prompt
2. _run_prompt(prompt_text)
    ↓
    a. subprocess.Popen(["copilot", "--acp", "--stdio"])
    b. JSON-RPC: initialize → session/new → session/prompt
    c. Stream collection: session/update (agent_message_chunk / agent_thought_chunk)
    d. Handle server requests: fs/read_text_file, fs/write_text_file, session/request_permission
3. _extract_tool_calls_from_text() → extract ─{...}XMLElement tool calls from text
4. Construct SimpleNamespace simulating OpenAI response object
```

### 8.3 Tool Call Extraction

`Source location: agent/copilot_acp_client.py:212-282`

ACP has no native tool call support; two formats are extracted from text:

1. **XML markers**: `─{...}XMLElement` regex matching
2. **Bare JSON**: standard OpenAI function-call JSON shape

After extraction, consumed spans are removed from the original text, returning `(tool_calls, cleaned_text)`.

### 8.4 File System Security

`Source location: agent/copilot_acp_client.py:286-296`

All ACP file operations are restricted within the session CWD:

```python
def _ensure_path_within_cwd(path_text, cwd):
    resolved = candidate.resolve()
    root = Path(cwd).resolve()
    resolved.relative_to(root)  # ValueError → PermissionError
```

Before reading, `get_read_block_error()` checks the blocklist; before writing, `is_write_denied()` checks protected paths. Read content is desensitized through `redact_sensitive_text(force=True)`.

---

## 9. SkillCommands — Skill Slash Commands

`Source location: agent/skill_commands.py:1-501`

### 9.1 Command Discovery and Registration

```
scan_skill_commands()
    ↓
1. Iterate SKILLS_DIR + get_external_skills_dirs()
2. For each SKILL.md: parse frontmatter → check platform compatibility → check disabled list
3. Name normalization: lowercase + spaces/underscores → hyphens + remove non-alnum characters
4. Register to _skill_commands["/skill-name"] = {name, description, skill_md_path, skill_dir}
```

Platform-aware caching: automatically rescans when `HERMES_PLATFORM` or `HERMES_SESSION_PLATFORM` changes.

### 9.2 Skill Message Construction

`Source location: agent/skill_commands.py:138-238`

`_build_skill_message()` assembles the complete skill content:

1. Template variable substitution (`_substitute_template_vars`)
2. Inline shell execution (`_expand_inline_shell`, configurable timeout)
3. Inject skill directory absolute path
4. Inject parsed configuration values (`_inject_skill_config`)
5. Inject setup prompts (skipped/needed/gateway_hint)
6. List support files (references/templates/scripts/assets)
7. Append user instructions and runtime notes

### 9.3 Cache Invalidation Logic

`Source location: agent/skill_commands.py:82-100`

`scan_skill_commands()` uses platform-aware caching — automatically rescans when `HERMES_PLATFORM` or `HERMES_SESSION_PLATFORM` environment variables change, ensuring only platform-compatible skills are loaded under different platforms (CLI/gateway). The cache key is a `(platform, hermes_home)` tuple; environment variable changes trigger a full rescan.

---

## 10. AccountUsage — Account Limit Queries

`Source location: agent/account_usage.py:1-326`

### 10.1 Three-Provider Adapter

```python
def fetch_account_usage(provider, *, base_url=None, api_key=None):
    if provider == "openai-codex":    return _fetch_codex_account_usage()
    if provider == "anthropic":       return _fetch_anthropic_account_usage()
    if provider == "openrouter":      return _fetch_openrouter_account_usage(base_url, api_key)
```

| Provider | API | Returned Data |
|----------|-----|---------------|
| OpenAI Codex | `chatgpt.com/backend-api/codex/wham/usage` | Session + Weekly window + Credits balance |
| Anthropic | `api.anthropic.com/api/oauth/usage` | 5h/7d/7d-opus/7d-sonnet windows + Extra usage |
| OpenRouter | `openrouter.ai/api/v1/credits` + `/key` | Credits balance + Key quota/usage |

### 10.2 Data Model

```python
@dataclass(frozen=True)
class AccountUsageWindow:
    label: str                    # "Session", "Weekly", "Opus week"
    used_percent: Optional[float] # 0-100
    reset_at: Optional[datetime]  # Reset time
    detail: Optional[str]         # Supplementary information

@dataclass(frozen=True)
class AccountUsageSnapshot:
    provider: str
    windows: tuple[AccountUsageWindow, ...]
    details: tuple[str, ...]
    unavailable_reason: Optional[str]
```

---

## 11. TitleGenerator — Auto Session Title Generation

`Source location: agent/title_generator.py:1-171`

Uses a helper model in a background thread to generate 3-7 word session titles:

```python
_TITLE_PROMPT = (
    "Generate a short, descriptive title (3-7 words) for a conversation "
    "that starts with the following exchange. Return ONLY the title text."
)

def maybe_auto_title(session_db, session_id, user_message, assistant_response,
                     conversation_history, ...):
    # Only triggers on first 2 user messages
    user_msg_count = sum(1 for m in history if m.get("role") == "user")
    if user_msg_count > 2:
        return

    thread = threading.Thread(target=auto_title_session, daemon=True, name="auto-title")
    thread.start()
```

Fallback chain: main runtime model → helper LLM client (cheapest available model) → silent failure (does not block the user). Messages truncated to 500 characters to keep requests small.

---

## 12. RateLimitTracker — Rate Limit Tracking

`Source location: agent/rate_limit_tracker.py:1-246`

### 12.1 Four-Bucket State Model

```python
@dataclass
class RateLimitState:
    requests_min: RateLimitBucket   # RPM
    requests_hour: RateLimitBucket  # RPH
    tokens_min: RateLimitBucket     # TPM
    tokens_hour: RateLimitBucket    # TPH
    captured_at: float              # Capture time
    provider: str
```

Each `RateLimitBucket` contains `limit`, `remaining`, `reset_seconds`, and computed properties `used`, `usage_pct`, `remaining_seconds_now` (deducting elapsed time).

### 12.2 Header Parsing

`Source location: agent/rate_limit_tracker.py:92-129`

Supports 12 standard x-ratelimit headers:

```
x-ratelimit-limit-requests / remaining / reset
x-ratelimit-limit-requests-1h / remaining / reset
x-ratelimit-limit-tokens / remaining / reset
x-ratelimit-limit-tokens-1h / remaining / reset
```

Case-insensitive (RFC 7230 compliant).

### 12.3 Visualization

```
Anthropic Rate Limits (captured 12s ago):

  Requests/min   [████████░░░░░░░░░░░░] 40.0%  20/50 used  (30 left, resets in 42s)
  Requests/hr    [██░░░░░░░░░░░░░░░░░░] 12.0%  120/1000 used  (880 left, resets in 2h 14m)

  Tokens/min     [██████████████████░░] 92.0%  92K/100K used  (8K left, resets in 15s)
  Tokens/hr      [████░░░░░░░░░░░░░░░░] 22.0%  220K/1.0M used  (780K left, resets in 58m)

  ⚠ tokens/min at 92% — resets in 15s
```

---

## 13. I18n — Internationalization System

`Source location: agent/i18n.py:1-233`

### 13.1 Language Resolution Chain

```
t(key, lang=None, **kwargs)
    ↓
1. Explicit lang= parameter
2. HERMES_LANGUAGE environment variable
3. config.yaml display.language
4. "en" (default)
```

8 supported languages: `en`, `zh`, `ja`, `de`, `es`, `fr`, `tr`, `uk`

Alias system handles common spelling variants:

```python
_LANGUAGE_ALIASES = {
    "chinese": "zh", "mandarin": "zh", "zh-cn": "zh", "zh-tw": "zh",
    "japanese": "ja", "jp": "ja",
    "german": "de", "deutsch": "de",
    "ukrainian": "uk", "українська": "uk",
    # ... 20+ aliases
}
```

### 13.2 Catalog Loading

YAML files can be nested; `_flatten_into()` recursively flattens into dotted-key dictionary:

```yaml
approval:
  choose: "请选择"
  choose_long: "请选择一个选项"
gateway:
  draining: "正在排空 {count} 个请求"
```

→ `{"approval.choose": "请选择", "approval.choose_long": "请选择一个选项", "gateway.draining": "正在排空 {count} 个请求"}`

Missing keys fall back to English; if English is also missing, the key path itself is returned (never crashes).

---

## 14. NousRateGuard — Nous Rate Limit Circuit Breaker

`Source location: agent/nous_rate_guard.py:1-325`

### 14.1 Cross-Session State Sharing

```
record_nous_rate_limit(headers, error_context)
    ↓ write to ~/.hermes/rate_limits/nous.json
    {"reset_at": 1744848000, "recorded_at": 1744847700, "reset_seconds": 300}

nous_rate_limit_remaining()
    ↓ read state file
    → remaining seconds or None (not limited)
```

Atomic write (tempfile + atomic_replace), all sessions share the same state file.

### 14.2 True vs. False 429 Determination

`Source location: agent/nous_rate_guard.py:192-244`

Nous Portal reuses multiple upstream providers behind the scenes. A 429 can be:

- **(a) Genuine limit exhaustion**: The caller's own RPM/RPH bucket is empty → trigger circuit breaker
- **(b) Upstream capacity insufficient**: Specific model temporarily unavailable → do not trigger circuit breaker

Determination logic:

```python
def is_genuine_nous_rate_limit(headers, last_known_state):
    # Signal 1: remaining == 0 in 429 response headers and reset >= 60s
    if _has_exhausted_bucket(_parse_buckets_from_headers(headers)):
        return True
    # Signal 2: bucket from last successful response is near exhaustion
    if _has_exhausted_bucket_in_object(last_known_state):
        return True
    return False  # Treated as (b), do not trigger circuit breaker
```

`_MIN_RESET_FOR_BREAKER_SECONDS = 60.0` — reset windows shorter than 60s are treated as transient jitter, not triggering the circuit breaker.

---

## 15. ManualCompressionFeedback — Compression Feedback

`Source location: agent/manual_compression_feedback.py:1-49`

```python
def summarize_manual_compression(before_messages, after_messages, before_tokens, after_tokens):
    noop = list(after_messages) == list(before_messages)
    if noop:
        headline = f"No changes from compression: {before_count} messages"
    else:
        headline = f"Compressed: {before_count} → {after_count} messages"

    # Smart tip: fewer messages but more tokens (compression summaries are denser)
    if not noop and after_count < before_count and after_tokens > before_tokens:
        note = "Note: fewer messages can still raise this estimate when "
               "compression rewrites the transcript into denser summaries."
```

---

## 16. ThinkScrubber — Streaming Reasoning Block Scrubbing

`Source location: agent/think_scrubber.py:64-386`

### 16.1 State Machine Design

```
                    ┌──────────────────────┐
                    │                      │
    feed(text)      │   _in_block=False    │   feed(text)
    ──────────►     │   (normal emit)      │   ──────────►
                    │                      │
                    │  detected <think>    │
                    │  at block boundary   │
                    └──────┬───────────────┘
                           │
                           ▼
                    ┌──────────────────────┐
                    │                      │
    feed(text)      │   _in_block=True     │   feed(text)
    ──────────►     │   (discard content)  │   ──────────►
                    │                      │
                    │  detected </think>   │
                    │                      │
                    └──────┬───────────────┘
                           │
                           ▼
                    back to _in_block=False
```

### 16.2 Delta Boundary Handling

Key design: partial tags spanning deltas are held back and not emitted until the next delta is parsed:

```python
def feed(self, text):
    buf = self._buf + text    # Concatenate last held partial
    self._buf = ""

    # ... find tags ...

    # When no parsable tags, hold possible tag prefix
    held = self._max_partial_suffix(buf, self._OPEN_TAGS)
    if held:
        emit_text = buf[:-held]
        self._buf = buf[-held:]   # Hold, wait for next feed() to parse
```

### 16.3 Block Boundary Rules

Opening tags are only treated as reasoning block starts at the following positions:

1. Stream start (`_last_emitted_ended_newline=True` initial value)
2. Previous emission ends with `\n`
3. Current line has only whitespace since the last newline

This prevents text that merely *mentions* the tag name (e.g., `"use <think> tags here"`) from being mistakenly scrubbed. Closing tag pairs (`<think>...</think>`) have no such restriction — they are always scrubbed.

---

## 17. Redact — Secret Redaction

`Source location: agent/redact.py:1-400`

### 17.1 Redaction Pattern Coverage

| Pattern Type | Regex | Example |
|-------------|-------|---------|
| API Key Prefix | 25+ vendor prefixes | `sk-proj-...`, `ghp_...`, `AIza...` |
| ENV Assignment | `KEY=VALUE` (KEY contains SECRET/TOKEN/API_KEY) | `OPENAI_API_KEY=***` |
| JSON Field | `"apiKey": "..."` | `"token": "***"` |
| Auth Header | `Authorization: Bearer ...` | `Bearer sk-p...7890` |
| Telegram Bot | `bot<digits>:<token>` | `bot123456:***` |
| Private Key | `-----BEGIN ... PRIVATE KEY-----` | `[REDACTED PRIVATE KEY]` |
| DB Connection String | `postgres://user:pass@host` | `postgres://user:***@host` |
| JWT | `eyJ...` | `eyJhbG...SflKx` |
| URL Query | `?access_token=...&code=...` | `?access_token=***&code=***` |
| URL Userinfo | `https://user:pass@host` | `https://user:***@host` |
| Form Body | `k=v&k=v` entire body | `api_key=***&name=test` |
| Discord Mention | `<@snowflake_id>` | `<@***>` |
| Phone (E.164) | `+<country><number>` | `+8613****5678` |

### 17.2 mask_secret Rules

`Source location: agent/redact.py:187-231`

```python
def mask_secret(value, *, head=4, tail=4, floor=12, placeholder="***"):
    if len(value) < floor:
        return placeholder    # Shorter than 12 characters: fully masked
    return f"{value[:head]}...{value[-tail:]}"  # Keep first 4 and last 4
```

`RedactingFormatter` serves as a logging handler that automatically redacts all log messages.

`code_file=True` parameter skips ENV assignment and JSON field patterns (preventing false redaction of source code constants like `MAX_TOKENS=***`).

---

## 18. FileSafety — File Path Security Validation

`Source location: agent/file_safety.py:1-111`

### 18.1 Write Denylist

Exact path denial (`build_write_denied_paths`):

```
~/.ssh/authorized_keys, ~/.ssh/id_rsa, ~/.ssh/id_ed25519, ~/.ssh/config
~/.hermes/.env, ~/.bashrc, ~/.zshrc, ~/.profile, ~/.bash_profile, ~/.zprofile
~/.netrc, ~/.pgpass, ~/.npmrc, ~/.pypirc
/etc/sudoers, /etc/passwd, /etc/shadow
```

Prefix path denial (`build_write_denied_prefixes`):

```
~/.ssh/, ~/.aws/, ~/.gnupg/, ~/.kube/, /etc/sudoers.d/, /etc/systemd/
~/.docker/, ~/.azure/, ~/.config/gh/
```

### 18.2 Safe Root Directory Mode

The `HERMES_WRITE_SAFE_ROOT` environment variable restricts all write operations to within the specified directory tree.

### 18.3 Read Protection

`Source location: agent/file_safety.py:93-111`

```python
def get_read_block_error(path):
    # Block reading Hermes internal cache files (prevent prompt injection)
    blocked_dirs = [hermes_home / "skills" / ".hub" / "index-cache", ...]
    return "Access denied: ... Use the skills_list or skill_view tools instead."
```

---

## 19. RetryUtils — Jittered Backoff

`Source location: agent/retry_utils.py:1-57`

```python
def jittered_backoff(attempt, *, base_delay=5.0, max_delay=120.0, jitter_ratio=0.5):
    delay = min(base_delay * (2 ** (attempt - 1)), max_delay)
    seed = (time.time_ns() ^ (tick * 0x9E3779B9)) & 0xFFFFFFFF  # Golden ratio hash
    rng = random.Random(seed)   # Deterministic but decorrelated
    jitter = rng.uniform(0, jitter_ratio * delay)
    return delay + jitter
```

Monotonically increasing counter + timestamp XOR golden ratio hash ensures concurrent sessions have decorrelated jitter, preventing thundering herd effects.

---

## 20. PromptCaching — Anthropic Prompt Cache Budget

`Source location: agent/prompt_caching.py:1-72`

### 20.1 system_and_3 Strategy

Places `cache_control` breakpoints at 4 positions (Anthropic maximum):

1. System prompt — stable across all turns
2-4. Last 3 non-system messages — sliding window

```python
def apply_anthropic_cache_control(api_messages, cache_ttl="5m", native_anthropic=False):
    marker = {"type": "ephemeral"}
    if cache_ttl == "1h":
        marker["ttl"] = "1h"

    # Breakpoint 1: system prompt
    if messages[0].get("role") == "system":
        _apply_cache_marker(messages[0], marker)

    # Breakpoints 2-4: last 3 non-system messages
    non_sys = [i for i in range(len(messages)) if messages[i].get("role") != "system"]
    for idx in non_sys[-remaining:]:
        _apply_cache_marker(messages[idx], marker)
```

Returns a deep copy, does not modify the original message list. `_apply_cache_marker` handles all format variants of string content, list content, and empty content.

---

## Comparison Tables

### Cross-Reference: Kanban Types from hermes_cli/kanban_db.py

Although hermes_cli/kanban_db.py is not in the agent/ directory, the following types are closely related to multi-agent collaboration scenarios:

| Type | Source Location | Core Responsibility |
|------|----------------|---------------------|
| `HallucinatedCardsError` | hermes_cli/kanban_db.py:2102 | Hallucinated card validation — when a worker claims to have created sub-cards upon task completion, blocks completion if sub-cards do not exist or were not created by this worker |
| `DispatchResult` | hermes_cli/kanban_db.py:2565 | Dispatcher single-tick result — includes detailed statuses such as reclaimed/promoted/spawned/skipped/crashed/auto_blocked/timed_out |

```python
# Source location: hermes_cli/kanban_db.py:2565-2580
class DispatchResult:
    """Outcome of a single dispatch pass."""
    reclaimed: int = 0
    promoted: int = 0
    spawned: list[tuple[str, str, str]] = field(default_factory=list)
    skipped_unassigned: list[str] = field(default_factory=list)
    skipped_nonspawnable: list[str] = field(default_factory=list)
    crashed: list[str] = field(default_factory=list)
    auto_blocked: list[str] = field(default_factory=list)
    timed_out: list[str] = field(default_factory=list)
```

### Table 1: Security Filter Chain Comparison

| Module | Filter Target | Trigger Timing | Configurable | Force Mode |
|--------|--------------|----------------|-------------|-----------|
| `think_scrubber` | Reasoning/thinking tags | Per-stream-delta callback | Not disableable | N/A |
| `redact` | API keys/tokens/credentials | Log/tool output | `HERMES_REDACT_SECRETS` | `force=True` |
| `file_safety` | Dangerous file path writes | Before tool execution | `HERMES_WRITE_SAFE_ROOT` | Always on |

### Table 2: Billing/Rate-Limit Module Comparison

| Module | Granularity | Data Source | Cross-Session | Output Format |
|--------|------------|-------------|--------------|--------------|
| `usage_pricing` | Single request | Built-in snapshot + API | No | `CostResult` (USD) |
| `insights` | 30-day aggregate | SessionDB SQL | Yes | terminal/gateway |
| `account_usage` | Account limits | Provider API | No | snapshot + lines |
| `rate_limit_tracker` | Single response | HTTP headers | No | full/compact |
| `nous_rate_guard` | 429 breaker | File + headers | Yes | remaining seconds |

### Table 3: OAuth/Integration Module Comparison

| Module | Protocol | Auth Method | Token Storage | Refresh |
|--------|----------|------------|--------------|---------|
| `google_oauth` | OAuth2 + PKCE | Browser/paste | `~/.hermes/auth/` | Auto + dedup |
| `copilot_acp` | ACP (JSON-RPC) | Subprocess stdio | Stateless | N/A (new session each) |
| `shell_hooks` | Subprocess stdin/stdout | Allowlist consent | `~/.hermes/shell-hooks-allowlist.json` | N/A |

### Table 4: Display/Interaction Module Comparison

| Module | Output Target | Async | Theme-Aware | Localized |
|--------|-------------|-------|------------|----------|
| `display.KawaiiSpinner` | CLI stdout | Thread | Skin engine | No |
| `display.get_cute_tool_message` | CLI quiet mode | No | Skin engine | No |
| `onboarding` | CLI/Gateway | No | No | No |
| `i18n` | All user-facing | No | No | 8 languages |
| `title_generator` | SessionDB | Thread | No | No |

---

## Summary Table

| Component | Lines | Responsibility |
|-----------|------:|---------------|
| `insights.py` | 930 | Session usage analytics: model/platform/tool/skill breakdown, activity patterns, cost estimation |
| `display.py` | 1002 | KawaiiSpinner animation, LocalEditSnapshot diff previews, tool completion lines |
| `shell_hooks.py` | 836 | YAML-declared shell hook registration, subprocess execution, cross-process allowlist |
| `usage_pricing.py` | 721 | 28+ model pricing snapshot, 3-source resolution, API usage normalization, USD estimation |
| `google_oauth.py` | 1061 | Google OAuth2 PKCE flow, refresh dedup, atomic credential storage |
| `copilot_acp_client.py` | 646 | ACP JSON-RPC adapter to OpenAI-compatible interface, tool call extraction |
| `skill_commands.py` | 501 | Skill slash-command scanning, loading, template substitution, config injection |
| `subdirectory_hints.py` | 224 | Lazy subdirectory context discovery and injection into tool results |
| `nous_rate_guard.py` | 325 | Nous Portal 429 cross-session circuit breaker, genuine-vs-transient discrimination |
| `rate_limit_tracker.py` | 246 | Parse x-ratelimit-* response headers, 4-bucket state model, progress bar visualization |
| `account_usage.py` | 326 | 3-provider account limit queries (Codex/Anthropic/OpenRouter) |
| `think_scrubber.py` | 386 | Streaming reasoning block state-machine scrubber, partial tag hold-back across deltas |
| `redact.py` | 400 | 13-category secret redaction patterns, RedactingFormatter log handler |
| `i18n.py` | 233 | 8-language internationalization, YAML catalog loading, dotted-key lookup |
| `title_generator.py` | 171 | Background thread auto-generates session titles from first exchange |
| `onboarding.py` | 193 | First-touch behavior-fork hints, persisted in config.yaml |
| `prompt_caching.py` | 72 | Anthropic system_and_3 cache strategy, 4-breakpoint injection |
| `file_safety.py` | 111 | File write denylist, safe root, cache read protection |
| `retry_utils.py` | 57 | Jittered exponential backoff, golden-ratio hash decorrelation |
| `manual_compression_feedback.py` | 49 | User-facing feedback summary for manual compression operations |

---

[← 19 — Hermes CLI Subsystem](/en/chapters/19-hermes-cli-subsystem) | [Back to Table of Contents](/en/chapters/overview)