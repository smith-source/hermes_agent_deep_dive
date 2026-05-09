# 14 — Configuration System: 四源分层合并与环境管理


---

## Source Files

| File | Lines | Role |
|------|-------|------|
| `hermes_cli/config.py` | 4939 | 配置加载 + 深度合并 + 环境变量展开 + 验证 |
| `hermes_cli/env_loader.py` | 175 | .env 文件加载 + 凭证净化 |
| `hermes_cli/profiles.py` | 1241 | 多 profile 切换 + 克隆 + 别名 |
| `hermes_cli/auth.py` | 5157 | API key 管理 + OAuth + 凭证池读写 |
| `hermes_cli/mcp_config.py` | 778 | MCP server 配置管理 |
| `hermes_cli/tools_config.py` | 2556 | 工具集启用/禁用 + 平台适配 |
| `hermes_cli/setup.py` | 3449 | 交互式 setup wizard |
| `.env.example` | 425 | 全部环境变量参考 |
| `cli-config.yaml.example` | 1039 | 完整 YAML 配置参考 |
| `hermes_constants.py` | 345 | 硬编码常量 + 默认值 |

**One-liner:** 配置系统通过四源优先级分层合并（.env 最高 → config.yaml → cli-config.yaml → hermes_constants.py），提供 `load_config()` 的深合并 + 环境变量展开 + 缓存，配合 env_loader 凭证净化、profiles 多身份切换、auth OAuth+API key 管理、tools_config 工具集过滤，setup wizard 完成首次配置。

---

## Architecture Overview

```
                    ┌───────────────────────────────────────────────────┐
                    │              4-Source Config Hierarchy            │
                    │                                                   │
                    │  Priority (highest → lowest):                     │
                    │  ┌──────────┐                                     │
                    │  │ .env     │ ← ~/.hermes/.env (425+ vars)        │
                    │  │ highest  │   Override stale shell exports       │
                    │  └──────────┘                                     │
                    │  ┌──────────────────┐                             │
                    │  │ config.yaml      │ ← ~/.hermes/config.yaml     │
                    │  │ user overrides   │   Deep-merged onto defaults │
                    │  └──────────────────┘                             │
                    │  ┌─────────────────────┐                          │
                    │  │ cli-config.yaml      │ ← Seeded on first run   │
                    │  │ defaults (1K lines)  │   Full YAML reference   │
                    │  └─────────────────────┘                          │
                    │  ┌──────────────────────────────────────┐        │
                    │  │ hermes_constants.py (hardcoded)      │        │
                    │  │ OPENROUTER_BASE_URL, HERMES_HOME     │        │
                    │  └──────────────────────────────────────┘        │
                    └───────────────────────────────────────────────────┘
                                       │
                    ┌──────────────────▼──────────────────────────┐
                    │       hermes_cli/config.py (4939L)          │
                    │                                              │
                    │  DEFAULT_CONFIG (386L)                       │
                    │  ├── model / providers / fallback_providers  │
                    │  ├── agent (max_turns, timeouts, enforcement)│
                    │  ├── terminal (backend, docker, ssh)         │
                    │  ├── toolsets / skills / context / memory    │
                    │  └────────────────────────────────────────── │
                    │                                              │
                    │  load_config() → _deep_merge → _expand_env_vars│
                    │       │                                      │
                    │       ├── mtime_ns+size cache (stat-based)    │
                    │       ├── _normalize_root_model_keys         │
                    │       ├── _normalize_max_turns_config        │
                    │       └── save_config → atomic_yaml_write  ⚡from utils.py:139 │
                    └──────────────────────────────────────────────┘
                                       │
       ┌───────────────────────────────▼────────────────────────────────┐
       │                    Support Modules                             │
       │                                                                │
       │  ┌─ env_loader.py ───┐  ┌─ profiles.py ────────┐              │
       │  │ load_hermes_dotenv │  │ create_profile()      │              │
       │  │ credential sanitize│  │ clone_from / clone_all│              │
       │  │ corrupted .env fix │  │ wrapper script alias  │              │
       │  └──────────────────┘  └───────────────────────┘              │
       │                                                                │
       │  ┌─ auth.py ────────────────────┐  ┌─ mcp_config.py ────┐    │
       │  │ ProviderConfig (133L)         │  │ MCP server CRUD    │    │
       │  │ OAuth device code flow        │  │ JSON config format │    │
       │  │ API key validation            │  │ stdio/websocket    │    │
       │  │ Credential pool persistence   │  └──────────────────┘    │
       │  │ Auth file locking             │                          │
       │  └──────────────────────────────┘                           │
       │                                                                │
       │  ┌─ tools_config.py ──────────┐  ┌─ setup.py ──────────┐    │
       │  │ Platform toolset filtering  │  │ Interactive wizard   │    │
       │  │ Toolset enable/disable      │  │ Provider setup flows │    │
       │  │ Key validation per toolset  │  │ First-run seeding    │    │
       │  └────────────────────────────┘  └──────────────────────┘    │
       └────────────────────────────────────────────────────────────────┘
```

---

## TL;DR

配置系统采用四级优先级分层合并：`.env`（最高优先级，覆盖陈旧 shell 导出）→ `config.yaml`（用户深度合并覆盖）→ `cli-config.yaml`（首次运行种子默认值）→ `hermes_constants.py`（硬编码常量）。`load_config()` 通过 stat-based 缓存（mtime_ns + size）避免重复 YAML 解析，内部流程为 `DEFAULT_CONFIG → _deep_merge(user_yaml) → _normalize → _expand_env_vars(${VAR})`。`env_loader.py` 在加载 .env 后净化非 ASCII 凭证值（PDF 复制损坏防护），`profiles.py` 支持多身份切换和克隆配置，`auth.py` 管理 28+ provider 的 OAuth/API key 认证和持久凭证池，`tools_config.py` 按平台过滤工具集，`setup.py` 提供交互式首次配置 wizard。

---

## 1. Config Hierarchy: 四源优先级合并

### 1.1 优先级规则

配置值的最终决定遵循四级优先级，从最高到最低：

| Priority | Source | Behavior |
|----------|--------|----------|
| 1 (最高) | `~/.hermes/.env` | 覆盖陈旧 shell 导出值，Override=True |
| 2 | `~/.hermes/config.yaml` | Deep merge onto DEFAULT_CONFIG |
| 3 | `cli-config.yaml.example` | 首次运行时复制为 config.yaml |
| 4 (最低) | `hermes_constants.py` | OPENROUTER_BASE_URL 等硬编码常量 |

### 1.2 DEFAULT_CONFIG 定义

DEFAULT_CONFIG 包含 model、providers、agent、terminal、toolsets 等全部默认值，是合并的基础。

**源码位置: hermes_cli/config.py:386-505**

```python
DEFAULT_CONFIG = {
    "model": "",
    "providers": {},
    "fallback_providers": [],
    "credential_pool_strategies": {},
    "toolsets": ["hermes-cli"],
    "agent": {
        "max_turns": 90,
        "gateway_timeout": 1800,
        "restart_drain_timeout": 180,
        "api_max_retries": 3,
        "tool_use_enforcement": "auto",
        "gateway_timeout_warning": 900,
        "gateway_notify_interval": 180,
        "gateway_auto_continue_freshness": 3600,
        "image_input_mode": "auto",
        "disabled_toolsets": [],
    },
    "terminal": {
        "backend": "local",
        "timeout": 180,
        "docker_image": "nikolaik/python-nodejs:python3.11-nodejs20",
        ...
    },
    ...
}
```

### 1.3 Deep Merge Algorithm

`_deep_merge()` 递归合并 override 到 base，保留嵌套默认值。用户只需覆盖 `tts.elevenlabs.voice_id` 就能保留 `tts.elevenlabs.model_id`。

**源码位置: hermes_cli/config.py:3675-3692**

```python
def _deep_merge(base: dict, override: dict) -> dict:
    """Recursively merge override into base, preserving nested defaults."""
    result = base.copy()
    for key, value in override.items():
        if (
            key in result
            and isinstance(result[key], dict)
            and isinstance(value, dict)
        ):
            result[key] = _deep_merge(result[key], value)
        else:
            result[key] = value
    return result
```

### 1.4 Environment Variable Expansion

`_expand_env_vars()` 递归展开 `${VAR}` 引用，未解析的保留原样。

**源码位置: hermes_cli/config.py:3695-3712**

```python
def _expand_env_vars(obj):
    """Recursively expand ${VAR} references in config values."""
    if isinstance(obj, str):
        return re.sub(
            r"\${([^}]+)}",
            lambda m: os.environ.get(m.group(1), m.group(0)),
            obj,
        )
    if isinstance(obj, dict):
        return {k: _expand_env_vars(v) for k, v in obj.items()}
    if isinstance(obj, list):
        return [_expand_env_vars(item) for item in obj]
    return obj
```

### 1.5 Template Preservation on Save

`_preserve_env_ref_templates()` 在保存时恢复原始 `${VAR}` 模板，避免将展开的明文密钥写入 config.yaml。

**源码位置: hermes_cli/config.py:3730-3749**

```python
def _preserve_env_ref_templates(current, raw, loaded_expanded=None):
    """Restore raw ${VAR} templates when a value is otherwise unchanged."""
    if isinstance(current, str) and isinstance(raw, str) and re.search(r"\${[^}]+}", raw):
        if current == raw:
            return raw
        if isinstance(loaded_expanded, str) and current == loaded_expanded:
            return raw
        if _expand_env_vars(raw) == current:
            return raw
        return current  # caller-owned once value diverges
```

### 1.6 Stat-based Config Cache

`load_config()` 使用 mtime_ns + size 缓存，跳过 YAML 解析 + 深合并 + 环境展开（约 13ms/call）。

**源码位置: hermes_cli/config.py:3923-3972**

```python
def load_config() -> Dict[str, Any]:
    """Load configuration from ~/.hermes/config.yaml.
    Cached on the config file's (mtime_ns, size)."""
    ensure_hermes_home()
    config_path = get_config_path()
    path_key = str(config_path)

    try:
        st = config_path.stat()
        cache_key = (st.st_mtime_ns, st.st_size)
    except FileNotFoundError:
        cache_key = None

    cached = _LOAD_CONFIG_CACHE.get(path_key)
    if cached is not None and cache_key is not None and cached[:2] == cache_key:
        return copy.deepcopy(cached[2])  # Cache hit

    config = copy.deepcopy(DEFAULT_CONFIG)
    if cache_key is not None:
        user_config = yaml.safe_load(f) or {}
        config = _deep_merge(config, user_config)

    normalized = _normalize_root_model_keys(_normalize_max_turns_config(config))
    expanded = _expand_env_vars(normalized)
    # Cache for future reads
    _LOAD_CONFIG_CACHE[path_key] = (cache_key[0], cache_key[1], copy.deepcopy(expanded))
    return expanded
```

---

## 2. Environment Loader (`hermes_cli/env_loader.py`, 175 lines)

### 2.1 双路径加载

`load_hermes_dotenv()` 优先加载 `~/.hermes/.env`（override=True），然后加载 project `.env`（仅当无用户 env 时才 override）。

**源码位置: hermes_cli/env_loader.py:142-175**

```python
def load_hermes_dotenv(
    *,
    hermes_home: str | os.PathLike | None = None,
    project_env: str | os.PathLike | None = None,
) -> list[Path]:
    """Load Hermes environment files with user config taking precedence."""
    loaded: list[Path] = []
    home_path = Path(hermes_home or os.getenv("HERMES_HOME", Path.home() / ".hermes"))
    user_env = home_path / ".env"

    # Fix corrupted .env files before python-dotenv parses them
    _sanitize_env_file_if_needed(user_env)

    if user_env.exists():
        _load_dotenv_with_fallback(user_env, override=True)
        loaded.append(user_env)

    if project_env_path and project_env_path.exists():
        _load_dotenv_with_fallback(project_env_path, override=not loaded)
        loaded.append(project_env_path)

    return loaded
```

### 2.2 Credential Sanitization

`_sanitize_loaded_credentials()` 在 .env 加载后剥离非 ASCII 字符，防止 PDF 复制损坏（Unicode lookalike glyphs）。

**源码位置: hermes_cli/env_loader.py:40-81**

```python
def _sanitize_loaded_credentials() -> None:
    """Strip non-ASCII characters from credential env vars."""
    _CREDENTIAL_SUFFIXES = ("_API_KEY", "_TOKEN", "_SECRET", "_KEY")

    for key, value in list(os.environ.items()):
        if not any(key.endswith(suffix) for suffix in _CREDENTIAL_SUFFIXES):
            continue
        try:
            value.encode("ascii")
            continue
        except UnicodeEncodeError:
            pass
        cleaned = value.encode("ascii", errors="ignore").decode("ascii")
        os.environ[key] = cleaned
        # One-time warning about stripped characters
        ...
```

### 2.3 Corrupted .env Fix

`_sanitize_env_file_if_needed()` 在 python-dotenv 解析前修复多 KEY=VALUE 拼接在同一行的损坏文件。

**源码位置: hermes_cli/env_loader.py:97-139**

```python
def _sanitize_env_file_if_needed(path: Path) -> None:
    """Pre-sanitize a .env file before python-dotenv reads it.
    python-dotenv does not handle corrupted lines where multiple
    KEY=VALUE pairs are concatenated on a single line."""
    if not path.exists():
        return
    from hermes_cli.config import _sanitize_env_lines
    with open(path, encoding="utf-8", errors="replace") as f:
        original = f.readlines()
    sanitized = _sanitize_env_lines(original)
    if sanitized != original:
        # Atomic write via temp file + atomic_replace
        ...
```

---

## 3. .env.example (425 lines)

### 3.1 环境变量分类

`.env.example` 覆盖所有环境变量，按功能分区：

| Section | Example Vars | Purpose |
|---------|-------------|---------|
| LLM PROVIDER | `OPENROUTER_API_KEY`, `GOOGLE_API_KEY`, `OLLAMA_API_KEY` | 推理提供者认证 |
| RL Training | `TINKER_API_KEY`, `WANDB_API_KEY` | RL 训练管线 |
| Terminal | `TERMINAL_ENV`, `TERMINAL_SSH_KEY`, `TERMINAL_SSH_PORT` | 终端沙箱配置 |
| Messaging | `DISCORD_HOME_CHANNEL`, `TELEGRAM_HOME_CHANNEL`, `IRC_SERVER` | 平台通道 |
| TTS | `ELEVENLABS_API_KEY`, `OPENAI_TTS_MODEL` | 语音合成 |
| Vision | `FIRECRAWL_API_KEY`, `PLAYWRIGHT_BROWSERS_PATH` | 爬取 + 浏览器 |
| Security | `TIRITH_ENABLED`, `TIRITH_BIN`, `TIRITH_TIMEOUT` | 安全扫描 |

**源码位置: .env.example:1-40**

```bash
# =============================================================================
# LLM PROVIDER (OpenRouter)
# =============================================================================
# OPENROUTER_API_KEY=

# =============================================================================
# LLM PROVIDER (Google AI Studio / Gemini)
# =============================================================================
# GOOGLE_API_KEY=your_google_ai_studio_key_here
# GEMINI_API_KEY=your_gemini_key_here  # alias for GOOGLE_API_KEY

# =============================================================================
# LLM PROVIDER (Ollama Cloud)
# =============================================================================
# OLLAMA_API_KEY=your_ollama_key_here
```

---

## 4. cli-config.yaml.example (1039 lines)

### 4.1 YAML 配置参考

完整的 YAML 配置文件，覆盖 model、providers、agent、terminal、toolsets、skills、context、memory、security 等全部配置维度。

**源码位置: cli-config.yaml.example:1-46**

```yaml
model:
  default: "anthropic/claude-opus-4.6"
  provider: "auto"
  base_url: "https://openrouter.ai/api/v1"

  # Token limits:
  # context_length: TOTAL context window (input + output)
  # max_tokens: OUTPUT cap per response

  # Fallback providers for automatic failover:
  # fallback_providers:
  #   - provider: openai-codex
  #   - provider: nous
```

---

## 5. Profile Management (`hermes_cli/profiles.py`, 1241 lines)

### 5.1 Profile 目录结构

每个 profile 是 `~/.hermes/profiles/<name>/` 下的独立 HERMES_HOME 目录，包含完整的 config.yaml、.env、SOUL.md 和 skill 列表。

**源码位置: hermes_cli/profiles.py:424-460**

```python
def create_profile(
    name: str,
    clone_from: Optional[str] = None,
    clone_all: bool = False,
    clone_config: bool = False,
    no_alias: bool = False,
) -> Path:
    """Create a new profile directory."""
    canon = normalize_profile_name(name)
    validate_profile_name(canon)

    if canon == "default":
        raise ValueError(
            "Cannot create a profile named 'default' — it is the built-in profile (~/.hermes)."
        )

    profile_dir = get_profile_dir(canon)
    if profile_dir.exists():
        raise FileExistsError(f"Profile '{canon}' already exists at {profile_dir}")

    if clone_all and source_dir:
        # Full copytree of source profile (exclude sibling profiles/)
        shutil.copytree(source_dir, profile_dir, ignore=_clone_all_copytree_ignore(source_dir))
```

### 5.2 Wrapper Script Alias

每个 profile 创建 shell wrapper script，自动设置 HERMES_HOME 环境变量。

**源码位置: hermes_cli/profiles.py:278-300**

```python
def create_wrapper_script(name: str) -> Optional[Path]:
    """Create a shell wrapper script that activates a profile."""
```

### 5.3 ProfileInfo 数据类

**源码位置: hermes_cli/profiles.py:321-333**

```python
class ProfileInfo:
    """Summary info about a profile for listing/display."""
```

---

## 6. Auth System (`hermes_cli/auth.py`, 5157 lines)

### 6.1 ProviderConfig

ProviderConfig 定义了每个认证提供者的完整元数据：auth_type（api_key/oauth/device_code 等）、env_vars、base_url、scopes 等。

**源码位置: hermes_cli/auth.py:133-458**

```python
class ProviderConfig:
    """Configuration for an authentication provider."""
```

### 6.2 Auth Store Persistence

认证数据通过 JSON 文件持久化，使用 file locking 防止并发写入冲突。

**源码位置: hermes_cli/auth.py:857-914**

```python
def _auth_store_lock(timeout_seconds: float = AUTH_LOCK_TIMEOUT_SECONDS):
    """Acquire a lock on the auth store file."""
```

### 6.3 OAuth Device Code Flow

支持 OAuth device code 认证流程（Nous Portal、GitHub Copilot 等），包含 token 自动刷新和过期检测。

---

## 7. MCP Config (`hermes_cli/mcp_config.py`, 778 lines)

MCP server 配置管理，支持 stdio 和 websocket 两种传输模式，提供 CRUD 操作和 JSON 配置格式。

---

## 8. Tools Config (`hermes_cli/tools_config.py`, 2556 lines)

### 8.1 平台工具集过滤

`_platform_toolset_summary()` 按平台过滤可配置的工具集，不同平台有不同的工具集可用性。

**源码位置: hermes_cli/tools_config.py:783-799**

```python
def _platform_toolset_summary(config: dict, platforms: Optional[List[str]] = None) -> Dict[str, Set[str]]:
```

### 8.2 工具集 Key 验证

`_toolset_has_keys()` 检查工具集所需的环境变量是否已设置。

**源码位置: hermes_cli/tools_config.py:1050-1090**

```python
def _toolset_has_keys(ts_key: str, config: dict = None) -> bool:
```

---

## 9. Setup Wizard (`hermes_cli/setup.py`, 3449 lines)

交互式首次配置 wizard，覆盖：provider 选择 + API key 输入 + 工具集启用 + 平台连接 + RL 环境配置。

---

## Comparison Tables

### Table 1: Config Source Priority Comparison

| Source | Priority | Override Behavior | Edit Method | Persistence |
|--------|----------|-------------------|-------------|-------------|
| `.env` | 最高 | Override stale shell exports | 手动编辑/`hermes setup` | 文件级持久化 |
| `config.yaml` | 2 | Deep merge onto DEFAULT_CONFIG | `hermes config edit/set` | atomic_yaml_write |
| `cli-config.yaml.example` | 3 | Seeds config.yaml on first run | 首次运行复制 | 只读参考 |
| `hermes_constants.py` | 最低 | 硬编码不可覆盖 | 代码修改 | 版本锁定 |

### Table 2: Auth Provider Type Comparison

| Auth Type | Providers | Token Refresh | Persistence | Locking |
|-----------|-----------|---------------|-------------|---------|
| `api_key` | OpenRouter, Anthropic, Google, z.ai | Manual | .env 文件 | None |
| `oauth_device_code` | Nous Portal, GitHub Copilot | Auto-refresh | auth_store.json | File lock |
| `oauth_external` | Vercel, DingTalk | Callback-based | auth_store.json | File lock |
| `aws_sdk` | AWS Bedrock | IAM rotation | .env + config | None |
| `copilot` | GitHub Copilot | OAuth refresh | auth_store.json | File lock |

### Table 3: Profile Operation Comparison

| Operation | Method | Clone Behavior | Alias | Use Case |
|-----------|--------|---------------|-------|----------|
| Create fresh | `create_profile(name)` | None | Auto wrapper | 新身份 |
| Clone config | `create_profile(name, clone_config=True)` | config.yaml + .env + SOUL.md | Auto wrapper | 共享配置 |
| Clone all | `create_profile(name, clone_all=True)` | Full copytree | Auto wrapper | 完整复制 |
| No alias | `create_profile(name, no_alias=True)` | Any | None | CI/CD 脚本 |

### Table 4: Config Loading Pipeline Comparison

| Step | Function | Input | Output | Cost |
|------|----------|-------|--------|------|
| 1 | `copy.deepcopy(DEFAULT_CONFIG)` | Hardcoded defaults | Base dict | ~0ms |
| 2 | `_deep_merge(config, user_yaml)` | base + user overrides | Merged dict | ~5ms |
| 3 | `_normalize_root_model_keys()` | Merged dict | Normalized dict | ~1ms |
| 4 | `_expand_env_vars()` | Normalized dict | Final config | ~7ms |
| 5 | Cache check (stat) | mtime_ns + size | Deepcopy of cached | ~0ms on hit |

---

## Summary Table

| Component | Lines | Responsibility |
|-----------|-------|----------------|
| `hermes_cli/config.py` | 4939 | 配置加载 + 深度合并 + env 展开 + 验证 + 缓存 |
| `hermes_cli/env_loader.py` | 175 | .env 双路径加载 + 凭证净化 + 损坏修复 |
| `hermes_cli/profiles.py` | 1241 | 多 profile 创建/克隆/切换/别名 |
| `hermes_cli/auth.py` | 5157 | 28+ provider 认证 + OAuth + 凭证池 |
| `hermes_cli/mcp_config.py` | 778 | MCP server 配置 CRUD |
| `hermes_cli/tools_config.py` | 2556 | 工具集启用/禁用 + 平台过滤 |
| `hermes_cli/setup.py` | 3449 | 交互式首次配置 wizard |
| `.env.example` | 425 | 全部环境变量参考 |
| `cli-config.yaml.example` | 1039 | 完整 YAML 配置参考 |
| `hermes_constants.py` | 345 | 硬编码常量 + HERMES_HOME 路径 |

---

[<< 13 — RL Training Pipeline](/zh-CN/chapters/13-rl-training) | [15 — Infrastructure & Deployment >>](/zh-CN/chapters/15-infrastructure-deployment)