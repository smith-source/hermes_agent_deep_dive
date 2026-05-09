# 20 — 代理支撑模块: Insights 分析/Display 渲染/Shell Hooks/Usage 计费/OAuth/Onboarding/ACP 集成

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

## 1. InsightsEngine — 会话洞察分析

`源码位置: agent/insights.py:93-930`

### 1.1 核心架构

InsightsEngine 直接持有 `SessionDB._conn`，通过预编译 SQL 查询获取数据，在内存中完成聚合计算。

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

`源码位置: agent/insights.py:180-184`

SQL 查询使用预计算常量避免运行时字符串拼接注入：

```python
_SESSION_COLS = ("id, source, model, started_at, ended_at, "
                 "message_count, tool_call_count, input_tokens, output_tokens, ...")
_GET_SESSIONS_ALL = f"SELECT {_SESSION_COLS} FROM sessions WHERE started_at >= ?"
```

### 1.2 双源工具计数

`源码位置: agent/insights.py:207-297`

工具使用统计从两个 SQL 源合并，取每工具的最大计数值避免重复计数：

| 源 | SQL 目标 | 覆盖场景 |
|----|---------|---------|
| Source 1: `tool_name` column | `messages WHERE role='tool'` | Gateway 会话（网关写入 tool_name） |
| Source 2: `tool_calls` JSON | `messages WHERE role='assistant'` | CLI 会话（tool_name 为 NULL） |

### 1.3 七维计算管线

```
generate() → _compute_overview()        # 总量/成本/时长
           → _compute_model_breakdown() # 按模型分桶
           → _compute_platform_breakdown() # 按来源分桶
           → _compute_tool_breakdown()  # 工具使用排行
           → _compute_skill_breakdown() # 技能使用排行
           → _compute_activity_patterns() # 日/时分布 + 连续天数
           → _compute_top_sessions()    # 最长/最多消息/最多 token
```

`源码位置: agent/insights.py:606-662`

活动模式分析包括连续使用天数计算（streak）：

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

### 1.4 双格式输出

| 格式 | 方法 | 目标 | 特点 |
|------|------|------|------|
| Terminal | `format_terminal()` | CLI `/insights` 命令 | Unicode box drawing, bar chart, full detail |
| Gateway | `format_gateway()` | Discord/Telegram | Markdown, top-5/8 截断, 紧凑 |

---

## 2. Display — 终端渲染引擎

`源码位置: agent/display.py:1-1002`

### 2.1 KawaiiSpinner

`源码位置: agent/display.py:573-798`

9 种内置旋转动画 + kawaii 表情系统：

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

皮肤引擎集成：优先从 `skin.spinner.waiting_faces` / `thinking_faces` / `thinking_verbs` 读取，失败时回退到硬编码值。

三重输出策略：

| 环境 | 行为 |
|------|------|
| 真实 TTY | `\r` 覆写动画，0.12s 间隔 |
| `prompt_toolkit` StdoutProxy | 跳过动画（避免换行错乱），由 TUI widget 接管 |
| 非 TTY (pipe/Docker) | 仅输出一次 `[tool] {message}`，避免日志膨胀 |

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

`源码位置: agent/display.py:703-742`

### 2.2 LocalEditSnapshot — 本地 diff 预览

`源码位置: agent/display.py:90-433`

写入工具执行前捕获文件快照，执行后生成 unified diff：

```
capture_local_edit_snapshot(tool_name, args)  # 写入前
    ↓
LocalEditSnapshot(paths=[...], before={"path": "old content"})
    ↓
extract_edit_diff(tool_name, result, snapshot=snapshot)  # 写入后
    ↓
render_edit_diff_with_delta(...)  # 渲染到终端
```

支持三种写入工具的路径解析：

| 工具 | 路径提取逻辑 |
|------|-------------|
| `write_file` | `args["path"]` |
| `patch` | `args["path"]` |
| `skill_manage` | 根据 action 类型（create/edit/delete）解析 `SKILLS_DIR` |

diff 渲染使用皮肤感知的 ANSI 颜色，带文件数/行数上限（6 文件 / 80 行）。

### 2.3 工具完成行

`源码位置: agent/display.py:837-996`

`get_cute_tool_message()` 为每个工具生成一行摘要，格式：`┊ {emoji} {verb:9} {detail}  {duration}`

30+ 工具的专属格式化 + 通用 fallback：

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

失败检测：终端工具检查 `exit_code != 0`，其他工具检查 JSON 中的 `"error"` / `"failed"` 关键词。失败行追加红色前缀和错误信息。

---

## 3. ShellHookSpec — Shell 钩子桥接

`源码位置: agent/shell_hooks.py:106-837`

### 3.1 声明式配置到运行时回调

用户在 `cli-config.yaml` 中声明钩子，模块解析后注册到 Python plugin manager：

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

解析为 `ShellHookSpec` 数据类：

```python
@dataclass
class ShellHookSpec:
    event: str
    command: str
    matcher: Optional[str] = None       # regex, only for pre/post_tool_call
    timeout: int = DEFAULT_TIMEOUT_SECONDS  # 60s, max 300s
    compiled_matcher: Optional[re.Pattern] = field(default=None, repr=False)
```

`源码位置: agent/shell_hooks.py:106-131`

### 3.2 子进程执行管线

```
invoke_hook(event, **kwargs)
    ↓
_make_callback(spec)(**kwargs)
    ↓
1. matcher 门控 (仅 tool 事件)
2. _serialize_payload(event, kwargs) → JSON stdin
3. _spawn(spec, stdin_json)
   ├── shlex.split(os.path.expanduser(command))
   ├── subprocess.run(argv, input=stdin_json, shell=False)
   └── timeout / FileNotFoundError / PermissionError 处理
4. _parse_response(event, stdout) → Hermes wire-shape
```

`源码位置: agent/shell_hooks.py:421-462`

**关键安全设计**：`shell=False` 防止 shell 注入；用户需要管道/重定向时必须包装在脚本中。

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

Stdout (JSON, 两种兼容格式):

```json
// Claude-Code-style (block):
{"decision": "block", "reason": "Forbidden command"}
// Hermes-canonical (block):
{"action": "block", "message": "Forbidden command"}
// Context injection (pre_llm_call):
{"context": "Today is Friday"}
```

`_parse_response` 将 `decision: "block"` 统一翻译为 `action: "block"`——这是整个模块最重要的正确性不变量。

### 3.4 三层同意机制

| 层级 | 来源 | 优先级 |
|------|------|--------|
| CLI flag | `--accept-hooks` | 最高 |
| 环境变量 | `HERMES_ACCEPT_HOOKS=1` | 中 |
| 配置文件 | `hooks_auto_accept: true` | 最低 |

Allowlist 持久化到 `~/.hermes/shell-hooks-allowlist.json`，使用 `fcntl.flock` 跨进程序列化写操作。记录 `(event, command)` 对及审批时间、脚本 mtime。

---

## 4. UsagePricing — 令牌计费引擎

`源码位置: agent/usage_pricing.py:1-721`

### 4.1 数据模型

三层不可变 dataclass 链：

```
CanonicalUsage (token buckets)
    ↓ + model_name
BillingRoute (provider + billing_mode)
    ↓
PricingEntry (per-million rates + source)
    ↓
CostResult (amount_usd + status + label)
```

`源码位置: agent/usage_pricing.py:28-77`

`CanonicalUsage` 归一化四种令牌桶：

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

### 4.2 三源定价解析

`源码位置: agent/usage_pricing.py:486-513`

```
get_pricing_entry(model, provider, base_url)
    ↓
1. resolve_billing_route() → BillingRoute
    ├── openai-codex → subscription_included (零费率)
    ├── openrouter → _openrouter_pricing_entry() (models API)
    ├── anthropic/openai → _lookup_official_docs_pricing() (硬编码快照)
    └── custom/localhost → unknown
2. 对于有 base_url 的路由 → fetch_endpoint_model_metadata() (OpenAI-compatible /models)
3. Fallback → _OFFICIAL_DOCS_PRICING dict lookup
```

### 4.3 官方定价快照

`源码位置: agent/usage_pricing.py:84-381`

`_OFFICIAL_DOCS_PRICING` 字典覆盖 28+ 模型，key 为 `(provider, model)` 元组：

| Provider | Models | Sample Rate (input/M) |
|----------|--------|----------------------|
| Anthropic | claude-opus-4, sonnet-4, 3.5-sonnet, 3-opus, 3-haiku | $0.25 – $15.00 |
| OpenAI | gpt-4o, gpt-4o-mini, gpt-4.1, gpt-4.1-mini/nano, o3, o3-mini | $0.10 – $10.00 |
| DeepSeek | deepseek-chat, deepseek-reasoner | $0.14 – $0.55 |
| Google | gemini-2.5-pro, gemini-2.5-flash, gemini-2.0-flash | $0.10 – $1.25 |
| AWS Bedrock | claude-opus-4-6, sonnet-4-6, nova-pro/lite/micro | $0.035 – $15.00 |
| MiniMax | minimax-m2.7 | $0.30 |

每个条目包含 `source_url`、`pricing_version`、`fetched_at`，支持审计溯源。

### 4.4 三种 API 用量格式归一化

`源码位置: agent/usage_pricing.py:516-586`

`normalize_usage()` 处理三种 API 响应形状：

| API 模式 | 总 input 字段 | 缓存分离 | 逻辑 |
|----------|-------------|---------|------|
| `anthropic_messages` | `input_tokens` (已排除缓存) | `cache_read_input_tokens` + `cache_creation_input_tokens` 顶层字段 | 直接映射 |
| `codex_responses` | `input_tokens` (含缓存) | `input_tokens_details.cached_tokens` | `input = total - cache_read - cache_write` |
| OpenAI (default) | `prompt_tokens` (含缓存) | `prompt_tokens_details.cached_tokens` | `input = total - cache_read - cache_write` |

OpenAI 兼容模式还有回退逻辑：当 `prompt_tokens_details` 无缓存数据时，检查 Anthropic 风格的顶层字段（某些代理如 OpenRouter/Vercel AI Gateway 暴露这些字段）。

---

## 5. SubdirectoryHintTracker — 子目录上下文注入

`源码位置: agent/subdirectory_hints.py:48-224`

### 5.1 惰性发现机制

不同于启动时 PromptBuilder 只加载 CWD 的上下文，SubdirectoryHintTracker 在工具调用参数中检测路径并按需加载：

```python
class SubdirectoryHintTracker:
    def __init__(self, working_dir=None):
        self.working_dir = Path(working_dir or os.getcwd()).resolve()
        self._loaded_dirs: Set[Path] = set()
        self._loaded_dirs.add(self.working_dir)  # CWD 已在启动时加载

    def check_tool_call(self, tool_name, tool_args) -> Optional[str]:
        dirs = self._extract_directories(tool_name, tool_args)
        all_hints = [self._load_hints_for_directory(d) for d in dirs]
        return "\n\n".join(all_hints) if all_hints else None
```

### 5.2 路径提取与祖先遍历

`源码位置: agent/subdirectory_hints.py:91-158`

从工具参数提取路径的两种方式：

1. **直接路径参数**: `path`, `file_path`, `workdir` 键
2. **Shell 命令解析**: `terminal` 工具的 `command` 字段，`shlex.split` 后提取路径类 token

祖先向上遍历（最多 5 层）确保读取深层文件时发现上层 AGENTS.md：

```python
def _add_path_candidate(self, raw_path, candidates):
    for _ in range(_MAX_ANCESTOR_WALK):
        if p in self._loaded_dirs:
            break  # 已加载的祖先，停止
        if self._is_valid_subdir(p):
            candidates.add(p)
        p = p.parent
```

### 5.3 查找文件优先级

`源码位置: agent/subdirectory_hints.py:29-33`

```python
_HINT_FILENAMES = ["AGENTS.md", "agents.md", "CLAUDE.md", "claude.md", ".cursorrules"]
```

每个目录只加载第一个匹配文件（与启动时逻辑一致），内容截断到 8000 字符。注入到工具结果尾部，**不修改 system prompt**，保护提示缓存。

---

## 6. Onboarding — 首次使用提示

`源码位置: agent/onboarding.py:1-193`

### 6.1 行为分叉点设计

三个提示标记，每个仅在用户首次遇到特定行为时显示一次：

| Flag | 触发时机 | 提示内容 |
|------|---------|---------|
| `busy_input_prompt` | 用户在 Agent 忙碌时发消息 | 解释 queue/interrupt/steer 三种模式 |
| `tool_progress_prompt` | 首次长时间工具运行 | 提示 `/verbose` 切换进度显示 |
| `openclaw_residue_cleanup` | 检测到 `~/.openclaw/` 目录 | 提示 `hermes claw migrate` 迁移 |

### 6.2 持久化与幂等性

```python
def is_seen(config, flag):
    return bool(_get_seen_dict(config).get(flag))

def mark_seen(config_path, flag):
    # 原子 YAML 写入，使用 atomic_yaml_write
    seen[flag] = True
    atomic_yaml_write(config_path, cfg)
```

存储在 `config.yaml` 的 `onboarding.seen.<flag>` 路径下。

---

## 7. GoogleOAuth — Google OAuth2 PKCE 流程

`源码位置: agent/google_oauth.py:1-1061`

### 7.1 认证流程

```
start_oauth_flow()
    ↓
1. 检查已有凭证 → 存在则跳过
2. _require_client_id() → 获取 OAuth client (3 级回退)
3. _generate_pkce_pair() → (verifier, challenge)
4. [Headless?] → _paste_mode_login()
   [GUI?]     → _bind_callback_server() → localhost:8085
5. 浏览器打开授权 URL
6. 等待回调 → _OAuthCallbackHandler 捕获 code
7. exchange_code(code, verifier, redirect_uri) → token response
8. _persist_token_response() → save_credentials()
```

### 7.2 Client ID 三级回退

`源码位置: agent/google_oauth.py:338-356`

```
_get_client_id()
    ↓
1. HERMES_GEMINI_CLIENT_ID 环境变量
2. 硬编码公共客户端 (Google gemini-cli 的公开 OAuth 客户端)
3. _scrape_client_credentials() — 从本地安装的 gemini-cli 二进制文件提取
```

### 7.3 RefreshParts — 刷新令牌打包格式

`源码位置: agent/google_oauth.py:393-420`

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

RefreshParts 处理 Google OAuth 刷新令牌的三段打包格式（`refresh_token|project_id|managed_project_id`），支持从存储格式解析和反向打包。`invalid_grant` 错误自动清除凭证文件，强制重新登录。

### 7.4 凭证存储与安全

`源码位置: agent/google_oauth.py:488-522`

存储路径: `~/.hermes/auth/google_oauth.json` (chmod 0o600)

```python
def save_credentials(creds):
    # O_WRONLY | O_CREAT | O_EXCL + S_IRUSR | S_IWUSR
    # 原子创建临时文件 → fsync → atomic_replace
    fd = os.open(tmp_path, O_WRONLY | O_CREAT | O_EXCL, S_IRUSR | S_IWUSR)
    with os.fdopen(fd, "w") as fh:
        fh.write(payload)
        fh.flush()
        os.fsync(fh.fileno())
    atomic_replace(tmp_path, path)
```

父目录 chmod 0o700。刷新 token 使用 packed 格式：`refresh_token|project_id|managed_project_id`。

### 7.4 并发刷新去重

`源码位置: agent/google_oauth.py:647-719`

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
        event.wait(timeout=30)  # 等待其他线程刷新完成
        fresh = load_credentials()
        if fresh and not fresh.access_token_expired():
            return fresh.access_token
```

`invalid_grant` 错误自动清除凭证文件，强制重新登录。

---

## 8. CopilotACPClient — GitHub Copilot ACP 集成

`源码位置: agent/copilot_acp_client.py:1-646`

### 8.1 OpenAI 兼容封装

将 ACP JSON-RPC 协议适配为 OpenAI SDK 的 `client.chat.completions.create()` 接口：

```python
class CopilotACPClient:
    def __init__(self, *, api_key=None, base_url=None, ...):
        self.chat = _ACPChatNamespace(self)

    # 对外接口与 OpenAI SDK 一致
    client = CopilotACPClient(api_key="copilot-acp")
    response = client.chat.completions.create(
        model="copilot-acp",
        messages=[...],
        tools=[...],
    )
```

### 8.2 请求生命周期

```
_create_chat_completion(messages, model, tools)
    ↓
1. _format_messages_as_prompt() → 将 OpenAI 格式消息转为文本 prompt
2. _run_prompt(prompt_text)
    ↓
    a. subprocess.Popen(["copilot", "--acp", "--stdio"])
    b. JSON-RPC: initialize → session/new → session/prompt
    c. 流式收集: session/update (agent_message_chunk / agent_thought_chunk)
    d. 处理服务端请求: fs/read_text_file, fs/write_text_file, session/request_permission
3. _extract_tool_calls_from_text() → 从文本中提取 ─{...}XMLElement 工具调用
4. 构造 SimpleNamespace 模拟 OpenAI response 对象
```

### 8.3 工具调用提取

`源码位置: agent/copilot_acp_client.py:212-282`

ACP 无原生工具调用支持，从文本中提取两种格式：

1. **XML 标记**: `─{...}XMLElement` 正则匹配
2. **裸 JSON**: 标准 OpenAI function-call JSON 形状

提取后从原文中移除已消耗的 span，返回 `(tool_calls, cleaned_text)`。

### 8.4 文件系统安全

`源码位置: agent/copilot_acp_client.py:286-296`

所有 ACP 文件操作限制在 session CWD 内：

```python
def _ensure_path_within_cwd(path_text, cwd):
    resolved = candidate.resolve()
    root = Path(cwd).resolve()
    resolved.relative_to(root)  # ValueError → PermissionError
```

读取前调用 `get_read_block_error()` 检查黑名单，写入前调用 `is_write_denied()` 检查保护路径。读取内容经过 `redact_sensitive_text(force=True)` 脱敏。

---

## 9. SkillCommands — 技能斜杠命令

`源码位置: agent/skill_commands.py:1-501`

### 9.1 命令发现与注册

```
scan_skill_commands()
    ↓
1. 遍历 SKILLS_DIR + get_external_skills_dirs()
2. 对每个 SKILL.md: 解析 frontmatter → 检查平台兼容性 → 检查 disabled 名单
3. 名称规范化: 小写 + 空格/下划线 → 连字符 + 去除非 alnum 字符
4. 注册到 _skill_commands["/skill-name"] = {name, description, skill_md_path, skill_dir}
```

平台感知缓存：当 `HERMES_PLATFORM` 或 `HERMES_SESSION_PLATFORM` 变化时自动重新扫描。

### 9.2 技能消息构建

`源码位置: agent/skill_commands.py:138-238`

`_build_skill_message()` 组装完整技能内容：

1. 模板变量替换 (`_substitute_template_vars`)
2. 内联 Shell 执行 (`_expand_inline_shell`, 可配置超时)
3. 注入技能目录绝对路径
4. 注入解析后的配置值 (`_inject_skill_config`)
5. 注入 setup 提示 (skipped/needed/gateway_hint)
6. 列出支持文件 (references/templates/scripts/assets)
7. 附加用户指令和运行时备注

### 9.3 缓存失效逻辑

`源码位置: agent/skill_commands.py:82-100`

`scan_skill_commands()` 使用平台感知缓存——当 `HERMES_PLATFORM` 或 `HERMES_SESSION_PLATFORM` 环境变量变化时自动重新扫描，确保不同平台（CLI/gateway）下只加载平台兼容的技能。缓存键为 `(platform, hermes_home)` 元组，环境变量变更触发完整重新扫描。

---

## 10. AccountUsage — 账户限额查询

`源码位置: agent/account_usage.py:1-326`

### 10.1 三供应商适配

```python
def fetch_account_usage(provider, *, base_url=None, api_key=None):
    if provider == "openai-codex":    return _fetch_codex_account_usage()
    if provider == "anthropic":       return _fetch_anthropic_account_usage()
    if provider == "openrouter":      return _fetch_openrouter_account_usage(base_url, api_key)
```

| Provider | API | 返回数据 |
|----------|-----|---------|
| OpenAI Codex | `chatgpt.com/backend-api/codex/wham/usage` | Session + Weekly 窗口 + Credits 余额 |
| Anthropic | `api.anthropic.com/api/oauth/usage` | 5h/7d/7d-opus/7d-sonnet 窗口 + Extra usage |
| OpenRouter | `openrouter.ai/api/v1/credits` + `/key` | Credits 余额 + Key quota/usage |

### 10.2 数据模型

```python
@dataclass(frozen=True)
class AccountUsageWindow:
    label: str                    # "Session", "Weekly", "Opus week"
    used_percent: Optional[float] # 0-100
    reset_at: Optional[datetime]  # 重置时间
    detail: Optional[str]         # 补充信息

@dataclass(frozen=True)
class AccountUsageSnapshot:
    provider: str
    windows: tuple[AccountUsageWindow, ...]
    details: tuple[str, ...]
    unavailable_reason: Optional[str]
```

---

## 11. TitleGenerator — 会话标题自动生成

`源码位置: agent/title_generator.py:1-171`

使用辅助模型在后台线程中生成 3-7 词的会话标题：

```python
_TITLE_PROMPT = (
    "Generate a short, descriptive title (3-7 words) for a conversation "
    "that starts with the following exchange. Return ONLY the title text."
)

def maybe_auto_title(session_db, session_id, user_message, assistant_response,
                     conversation_history, ...):
    # 仅在前 2 次用户消息时触发
    user_msg_count = sum(1 for m in history if m.get("role") == "user")
    if user_msg_count > 2:
        return

    thread = threading.Thread(target=auto_title_session, daemon=True, name="auto-title")
    thread.start()
```

容错链：主运行时模型 → 辅助 LLM 客户端（最便宜可用模型）→ 静默失败（不阻塞用户）。消息截断到 500 字符保持请求小型化。

---

## 12. RateLimitTracker — 速率限制追踪

`源码位置: agent/rate_limit_tracker.py:1-246`

### 12.1 四桶状态模型

```python
@dataclass
class RateLimitState:
    requests_min: RateLimitBucket   # RPM
    requests_hour: RateLimitBucket  # RPH
    tokens_min: RateLimitBucket     # TPM
    tokens_hour: RateLimitBucket    # TPH
    captured_at: float              # 捕获时间
    provider: str
```

每个 `RateLimitBucket` 包含 `limit`、`remaining`、`reset_seconds`，以及计算属性 `used`、`usage_pct`、`remaining_seconds_now`（扣除已流逝时间）。

### 12.2 Header 解析

`源码位置: agent/rate_limit_tracker.py:92-129`

支持 12 个标准 x-ratelimit 头：

```
x-ratelimit-limit-requests / remaining / reset
x-ratelimit-limit-requests-1h / remaining / reset
x-ratelimit-limit-tokens / remaining / reset
x-ratelimit-limit-tokens-1h / remaining / reset
```

大小写不敏感（RFC 7230 合规）。

### 12.3 可视化

```
Anthropic Rate Limits (captured 12s ago):

  Requests/min   [████████░░░░░░░░░░░░] 40.0%  20/50 used  (30 left, resets in 42s)
  Requests/hr    [██░░░░░░░░░░░░░░░░░░] 12.0%  120/1000 used  (880 left, resets in 2h 14m)

  Tokens/min     [██████████████████░░] 92.0%  92K/100K used  (8K left, resets in 15s)
  Tokens/hr      [████░░░░░░░░░░░░░░░░] 22.0%  220K/1.0M used  (780K left, resets in 58m)

  ⚠ tokens/min at 92% — resets in 15s
```

---

## 13. I18n — 国际化系统

`源码位置: agent/i18n.py:1-233`

### 13.1 语言解析链

```
t(key, lang=None, **kwargs)
    ↓
1. 显式 lang= 参数
2. HERMES_LANGUAGE 环境变量
3. config.yaml display.language
4. "en" (默认)
```

8 种支持语言：`en`, `zh`, `ja`, `de`, `es`, `fr`, `tr`, `uk`

别名系统处理常见拼写变体：

```python
_LANGUAGE_ALIASES = {
    "chinese": "zh", "mandarin": "zh", "zh-cn": "zh", "zh-tw": "zh",
    "japanese": "ja", "jp": "ja",
    "german": "de", "deutsch": "de",
    "ukrainian": "uk", "українська": "uk",
    # ... 20+ aliases
}
```

### 13.2 目录加载

YAML 文件可嵌套，`_flatten_into()` 递归展平为 dotted-key 字典：

```yaml
approval:
  choose: "请选择"
  choose_long: "请选择一个选项"
gateway:
  draining: "正在排空 {count} 个请求"
```

→ `{"approval.choose": "请选择", "approval.choose_long": "请选择一个选项", "gateway.draining": "正在排空 {count} 个请求"}`

缺失 key 回退到英语；英语也缺失时返回 key 路径本身（永不崩溃）。

---

## 14. NousRateGuard — Nous 速率限制断路器

`源码位置: agent/nous_rate_guard.py:1-325`

### 14.1 跨会话状态共享

```
record_nous_rate_limit(headers, error_context)
    ↓ 写入 ~/.hermes/rate_limits/nous.json
    {"reset_at": 1744848000, "recorded_at": 1744847700, "reset_seconds": 300}

nous_rate_limit_remaining()
    ↓ 读取状态文件
    → remaining seconds or None (not limited)
```

原子写入（tempfile + atomic_replace），所有会话共享同一状态文件。

### 14.2 真/假 429 判定

`源码位置: agent/nous_rate_guard.py:192-244`

Nous Portal 背后复用多个上游提供商。429 可能是：

- **(a) 真限额耗尽**: 调用者自己的 RPM/RPH 桶空了 → 触发断路器
- **(b) 上游容量不足**: 特定模型暂时不可用 → 不触发断路器

判定逻辑：

```python
def is_genuine_nous_rate_limit(headers, last_known_state):
    # 信号 1: 429 响应头中 remaining == 0 且 reset >= 60s
    if _has_exhausted_bucket(_parse_buckets_from_headers(headers)):
        return True
    # 信号 2: 上次成功响应的桶已接近耗尽
    if _has_exhausted_bucket_in_object(last_known_state):
        return True
    return False  # 视为 (b)，不触发断路器
```

`_MIN_RESET_FOR_BREAKER_SECONDS = 60.0` — 重置窗口短于 60s 的视为瞬态抖动，不触发断路器。

---

## 15. ManualCompressionFeedback — 压缩反馈

`源码位置: agent/manual_compression_feedback.py:1-49`

```python
def summarize_manual_compression(before_messages, after_messages, before_tokens, after_tokens):
    noop = list(after_messages) == list(before_messages)
    if noop:
        headline = f"No changes from compression: {before_count} messages"
    else:
        headline = f"Compressed: {before_count} → {after_count} messages"

    # 智能提示：消息减少但 token 增加（压缩摘要更密集）
    if not noop and after_count < before_count and after_tokens > before_tokens:
        note = "Note: fewer messages can still raise this estimate when "
               "compression rewrites the transcript into denser summaries."
```

---

## 16. ThinkScrubber — 流式推理块清洗

`源码位置: agent/think_scrubber.py:64-386`

### 16.1 状态机设计

```
                    ┌──────────────────────┐
                    │                      │
    feed(text)      │   _in_block=False    │   feed(text)
    ──────────►     │   (正常发射)         │   ──────────►
                    │                      │
                    │  检测到 <think>       │
                    │  at block boundary   │
                    └──────┬───────────────┘
                           │
                           ▼
                    ┌──────────────────────┐
                    │                      │
    feed(text)      │   _in_block=True     │   feed(text)
    ──────────►     │   (丢弃内容)         │   ──────────►
                    │                      │
                    │  检测到 </think>      │
                    │                      │
                    └──────┬───────────────┘
                           │
                           ▼
                    回到 _in_block=False
```

### 16.2 Delta 边界处理

关键设计：部分标签跨 delta 时持有不发，直到下一个 delta 解析：

```python
def feed(self, text):
    buf = self._buf + text    # 拼接上次持有的部分
    self._buf = ""

    # ... 查找标签 ...

    # 无可解析标签时，持有可能的标签前缀
    held = self._max_partial_suffix(buf, self._OPEN_TAGS)
    if held:
        emit_text = buf[:-held]
        self._buf = buf[-held:]   # 持有，等下次 feed() 解析
```

### 16.3 Block Boundary 规则

开标签仅在以下位置被视为推理块起始：

1. 流开头（`_last_emitted_ended_newline=True` 初始值）
2. 前一个发射以 `\n` 结束
3. 当前行自上次换行以来只有空白

这防止文本中*提到*标签名（如 `"use <think> tags here"`）被误脱敏。闭标签对（`<think>X</think>`）无此限制——总是被脱敏。

---

## 17. Redact — 密钥脱敏

`源码位置: agent/redact.py:1-400`

### 17.1 脱敏模式覆盖

| 模式类型 | 正则 | 示例 |
|---------|------|------|
| API Key 前缀 | 25+ vendor prefixes | `sk-proj-...`, `ghp_...`, `AIza...` |
| ENV 赋值 | `KEY=VALUE` (KEY 含 SECRET/TOKEN/API_KEY) | `OPENAI_API_KEY=***` |
| JSON 字段 | `"apiKey": "..."` | `"token": "***"` |
| Auth Header | `Authorization: Bearer ...` | `Bearer sk-p...7890` |
| Telegram Bot | `bot<digits>:<token>` | `bot123456:***` |
| Private Key | `-----BEGIN ... PRIVATE KEY-----` | `[REDACTED PRIVATE KEY]` |
| DB 连接串 | `postgres://user:pass@host` | `postgres://user:***@host` |
| JWT | `eyJ...` | `eyJhbG...SflKx` |
| URL Query | `?access_token=...&code=...` | `?access_token=***&code=***` |
| URL Userinfo | `https://user:pass@host` | `https://user:***@host` |
| Form Body | `k=v&k=v` 整体 | `api_key=***&name=test` |
| Discord Mention | `<@snowflake_id>` | `<@***>` |
| Phone (E.164) | `+<country><number>` | `+8613****5678` |

### 17.2 mask_secret 规则

`源码位置: agent/redact.py:187-231`

```python
def mask_secret(value, *, head=4, tail=4, floor=12, placeholder="***"):
    if len(value) < floor:
        return placeholder    # 短于 12 字符完全遮蔽
    return f"{value[:head]}...{value[-tail:]}"  # 保留首 4 尾 4
```

`RedactingFormatter` 作为 logging handler 自动脱敏所有日志消息。

`code_file=True` 参数跳过 ENV 赋值和 JSON 字段模式（防止误脱敏源码常量如 `MAX_TOKENS=***`）。

---

## 18. FileSafety — 文件路径安全校验

`源码位置: agent/file_safety.py:1-111`

### 18.1 写入拒绝名单

精确路径拒绝（`build_write_denied_paths`）：

```
~/.ssh/authorized_keys, ~/.ssh/id_rsa, ~/.ssh/id_ed25519, ~/.ssh/config
~/.hermes/.env, ~/.bashrc, ~/.zshrc, ~/.profile, ~/.bash_profile, ~/.zprofile
~/.netrc, ~/.pgpass, ~/.npmrc, ~/.pypirc
/etc/sudoers, /etc/passwd, /etc/shadow
```

前缀路径拒绝（`build_write_denied_prefixes`）：

```
~/.ssh/, ~/.aws/, ~/.gnupg/, ~/.kube/, /etc/sudoers.d/, /etc/systemd/
~/.docker/, ~/.azure/, ~/.config/gh/
```

### 18.2 安全根目录模式

`HERMES_WRITE_SAFE_ROOT` 环境变量限制所有写入操作到指定目录树内。

### 18.3 读取保护

`源码位置: agent/file_safety.py:93-111`

```python
def get_read_block_error(path):
    # 阻止读取 Hermes 内部缓存文件（防止 prompt injection）
    blocked_dirs = [hermes_home / "skills" / ".hub" / "index-cache", ...]
    return "Access denied: ... Use the skills_list or skill_view tools instead."
```

---

## 19. RetryUtils — 抖动退避

`源码位置: agent/retry_utils.py:1-57`

```python
def jittered_backoff(attempt, *, base_delay=5.0, max_delay=120.0, jitter_ratio=0.5):
    delay = min(base_delay * (2 ** (attempt - 1)), max_delay)
    seed = (time.time_ns() ^ (tick * 0x9E3779B9)) & 0xFFFFFFFF  # 黄金比例哈希
    rng = random.Random(seed)   # 确定性但去相关
    jitter = rng.uniform(0, jitter_ratio * delay)
    return delay + jitter
```

单调递增计数器 + 时间戳 XOR 黄金比例哈希确保并发会话的抖动互不相关，防止雷群效应。

---

## 20. PromptCaching — Anthropic 提示缓存预算

`源码位置: agent/prompt_caching.py:1-72`

### 20.1 system_and_3 策略

在 4 个位置放置 `cache_control` 断点（Anthropic 最大值）：

1. System prompt — 所有轮次稳定
2-4. 最后 3 条非 system 消息 — 滚动窗口

```python
def apply_anthropic_cache_control(api_messages, cache_ttl="5m", native_anthropic=False):
    marker = {"type": "ephemeral"}
    if cache_ttl == "1h":
        marker["ttl"] = "1h"

    # 断点 1: system prompt
    if messages[0].get("role") == "system":
        _apply_cache_marker(messages[0], marker)

    # 断点 2-4: 最后 3 条非 system 消息
    non_sys = [i for i in range(len(messages)) if messages[i].get("role") != "system"]
    for idx in non_sys[-remaining:]:
        _apply_cache_marker(messages[idx], marker)
```

返回深拷贝，不修改原始消息列表。`_apply_cache_marker` 处理字符串内容、列表内容、空内容的所有格式变体。

---

## Comparison Tables

### Cross-Reference: Kanban Types from hermes_cli/kanban_db.py

虽然 hermes_cli/kanban_db.py 不在 agent/ 目录中，但以下类型与多代理协作场景密切相关：

| 类型 | 源码位置 | 核心职责 |
|------|---------|---------|
| `HallucinatedCardsError` | hermes_cli/kanban_db.py:2102 | 幻觉卡验证——worker 完成任务时声称创建了子卡，若子卡不存在或非本 worker 创建则阻断完成 |
| `DispatchResult` | hermes_cli/kanban_db.py:2565 | 调度器单次滴答结果——包含 reclaimed/promoted/spawned/skipped/crashed/auto_blocked/timed_out 等详细状态 |

```python
# 源码位置: hermes_cli/kanban_db.py:2565-2580
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

[← 19 — Hermes CLI 子系统](/zh-CN/chapters/19-hermes-cli-subsystem) | [返回目录](/zh-CN/chapters/overview)
