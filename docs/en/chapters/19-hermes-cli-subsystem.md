# 19 — Hermes CLI Subsystem: 56K-Line Command-Line Infrastructure — Dispatch / Auth / Config / Gateway Management / Model Selection / Diagnostics / Kanban

## Source File Overview

| File | Lines | Core Responsibility |
|------|------|---------------------|
| `hermes_cli/main.py` | 10876 | CLI entry point, command dispatch, model/provider selection flow, update mechanism |
| `hermes_cli/auth.py` | 5157 | Multi-provider authentication, OAuth device code/API Key, token refresh |
| `hermes_cli/config.py` | 4939 | Configuration read/write, YAML persistence, environment variable parsing |
| `hermes_cli/gateway.py` | 4810 | Gateway process management, systemd/launchd services, platform configuration |
| `hermes_cli/kanban_db.py` | 4087 | SQLite kanban database, task CRUD, dispatcher, circuit breaker |
| `hermes_cli/web_server.py` | 4063 | Web Dashboard server |
| `hermes_cli/models.py` | 3556 | Model catalog, pricing, alias resolution, provider detection |
| `hermes_cli/setup.py` | 3449 | Interactive setup wizard, staged configuration, OpenClaw migration |
| `hermes_cli/tools_config.py` | 2556 | Toolset configuration, platform-level enable/disable, image generation providers |
| `hermes_cli/kanban.py` | 2036 | Kanban CLI commands, kanban creation/task management |
| `hermes_cli/model_switch.py` | 1754 | Runtime model switching, alias resolution, provider hot-swap |
| `hermes_cli/commands.py` | 1713 | Slash command registration, completion, Telegram/Discord/Slack command mapping |
| `hermes_cli/skills_hub.py` | 1594 | Skills marketplace/Hub interaction |
| `hermes_cli/plugins_cmd.py` | 1544 | Plugin CLI commands (install/update/remove/enable/disable/list) |
| `hermes_cli/doctor.py` | 1520 | Diagnostics tool, dependency checks, connectivity testing |
| `hermes_cli/plugins.py` | 1362 | Plugin discovery/loading/lifecycle management, PluginManager |
| `hermes_cli/runtime_provider.py` | 1340 | Runtime provider resolution |
| `hermes_cli/profiles.py` | 1241 | Multi-profile management, create/switch/export/import |
| `hermes_cli/backup.py` | 939 | Configuration backup and restore |
| `hermes_cli/skin_engine.py` | 892 | Skin/theme system, color/spinner/branding customization |
| `hermes_cli/voice.py` | 846 | Voice synthesis (TTS) integration |
| `hermes_cli/claw.py` | 803 | Claw tool integration |
| `hermes_cli/nous_subscription.py` | 799 | Nous subscription feature detection |
| `hermes_cli/mcp_config.py` | 778 | MCP server configuration management |
| `hermes_cli/debug.py` | 746 | Debug output and tracing |
| `hermes_cli/auth_commands.py` | 718 | Authentication subcommands (login/logout/status) |
| `hermes_cli/providers.py` | 701 | Provider registry and extension interface |
| `hermes_cli/kanban_diagnostics.py` | 649 | Kanban diagnostics and health checks |
| `hermes_cli/banner.py` | 635 | ASCII welcome banner, version/update status |
| `hermes_cli/curator.py` | 560 | Content curation tool |
| `hermes_cli/status.py` | 551 | System status display |
| `hermes_cli/goals.py` | 535 | Goal management |
| `hermes_cli/tips.py` | 487 | Random tips corpus |
| `hermes_cli/uninstall.py` | 481 | Uninstall and cleanup |
| `hermes_cli/clipboard.py` | 481 | Clipboard operations |
| `hermes_cli/model_normalize.py` | 473 | Model name normalization |
| `hermes_cli/curses_ui.py` | 472 | curses terminal UI |
| `hermes_cli/memory_setup.py` | 457 | Memory system configuration wizard |
| `hermes_cli/copilot_auth.py` | 392 | GitHub Copilot authentication |
| `hermes_cli/logs.py` | 390 | Log viewing and management |
| `hermes_cli/hooks.py` | 385 | Hook registration and execution |
| `hermes_cli/_parser.py` | 373 | argparse top-level parser construction |
| `hermes_cli/fallback_cmd.py` | 361 | Fallback provider chain management |
| `hermes_cli/oneshot.py` | 331 | One-shot execution mode |
| `hermes_cli/model_catalog.py` | 329 | models.dev catalog fetching |
| `hermes_cli/dump.py` | 325 | Configuration/state dump |
| `hermes_cli/completion.py` | 315 | Shell completion generation |
| `hermes_cli/cron.py` | 307 | Scheduled task management |
| `hermes_cli/azure_detect.py` | 300 | Azure environment detection |
| `hermes_cli/dingtalk_auth.py` | 293 | DingTalk authentication |
| `hermes_cli/webhook.py` | 274 | Webhook management |
| `hermes_cli/checkpoints.py` | 244 | File system checkpoints |
| `hermes_cli/callbacks.py` | 242 | Callback function registration |
| `hermes_cli/pty_bridge.py` | 234 | PTY bridge |
| `hermes_cli/skills_config.py` | 177 | Skills configuration |
| `hermes_cli/codex_models.py` | 177 | Codex model list |
| `hermes_cli/env_loader.py` | 175 | Environment variable loader |
| `hermes_cli/slack_cli.py` | 153 | Slack CLI integration |
| `hermes_cli/relaunch.py` | 148 | Process restart |
| `hermes_cli/browser_connect.py` | 138 | Browser CDP connection |
| `hermes_cli/pairing.py` | 97 | Device pairing |
| `hermes_cli/platforms.py` | 83 | Platform detection helpers |
| `hermes_cli/timeouts.py` | 82 | Timeout configuration |
| `hermes_cli/cli_output.py` | 78 | CLI output helper functions |
| `hermes_cli/vercel_auth.py` | 70 | Vercel authentication |
| `hermes_cli/__init__.py` | 47 | Package initialization, version number |
| `hermes_cli/colors.py` | 38 | ANSI color constants |
| `hermes_cli/default_soul.py` | 11 | Default soul file path |

**Total: 77,169 lines** (67 Python files)

## One-Line Summary

hermes_cli is the 77K-line command-line infrastructure for Hermes Agent, integrating multi-provider authentication, gateway service management, model catalog and switching, interactive configuration wizard, kanban SQLite engine, plugin system, slash command registration, diagnostics tool, and theme engine — constituting the complete user interface layer from the `hermes` command entry point to the agent runtime.

## Architecture Overview

```
                              ┌──────────────────────────────────────────────────┐
                              │                 hermes_cli/main.py               │
                              │            (10876 lines · CLI entry & dispatch)  │
                              │  cmd_chat · cmd_model · cmd_setup · cmd_update  │
                              │  cmd_gateway · cmd_doctor · cmd_kanban · ...    │
                              └──────────┬──────────────────┬───────────────────┘
                                         │                  │
                    ┌────────────────────┘                  └────────────────────┐
                    │                                              │            │
         ┌──────────▼──────────┐                    ┌──────────────▼──────────┐ │
         │    auth.py (5157)   │                    │  gateway.py (4810)      │ │
         │  Auth & Token Mgmt  │                    │  Service Mgmt & Platform│ │
         │  ProviderConfig     │◄───────────────────│  Config                 │ │
         │  OAuth / API Key    │   resolve_provider │  systemd / launchd      │ │
         │  resolve_provider() │                    │  run_gateway()          │ │
         └──────────┬──────────┘                    │  gateway_setup()        │ │
                    │                               └──────────────────────────┘ │
                    │                                              │            │
     ┌──────────────┼──────────────────────────────────────────────┤            │
     │              │                                              │            │
┌────▼────────┐ ┌───▼───────────┐  ┌──────────────────┐  ┌───────▼─────────┐  │
│ models.py   │ │ model_switch  │  │  setup.py (3449) │  │  doctor.py     │  │
│ (3556)      │ │ .py (1754)    │  │  Interactive     │  │  (1520)        │  │
│ Model       │ │ Runtime       │  │  wizard          │  │  Diagnostic    │  │
│ catalog     │ │ switching     │  │  5-stage config  │  │  checks        │  │
│ Pricing/    │ │ ModelIdentity │  │  OpenClaw        │  │  Connectivity  │  │
│ aliases     │ │               │  │  migration       │  │  testing       │  │
└──────┬──────┘ └───────┬───────┘  └──────────────────┘  └────────────────┘  │
       │                │                                                     │
       │    ┌───────────┼──────────────────────────────┐                     │
       │    │           │                              │                     │
┌──────▼────▼──────┐  ┌─▼───────────────┐  ┌──────────▼──────────┐         │
│ tools_config.py  │  │ commands.py     │  │  kanban_db.py (4087)│         │
│ (2556)           │  │ (1713)          │  │  SQLite kanban      │         │
│ Toolset          │  │ Slash command   │  │  engine             │         │
│ configuration    │  │ registration    │  │  Task/Run/Comment   │         │
│ Platform-level   │  │ CommandDef      │  │  claim/complete     │         │
│ enable/disable   │  │ Completion/Help │  │  dispatch_once()    │         │
│ Image gen        │  │                 │  │  Circuit breaker    │         │
└──────────────────┘  └─────────────────┘  └─────────────────────┘         │
                                                                               │
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐       │
  │ profiles.py  │  │ plugins.py   │  │skin_engine.py│  │  banner.py    │       │
  │ (1241)       │  │ (1362)       │  │ (892)        │  │  (635)        │       │
  │ Multi-profile│  │ Plugin       │  │ Theme/Colors │  │  ASCII banner │       │
  │ Create/      │  │ discovery/   │  │ SkinConfig   │  │  Version/     │       │
  │ Switch       │  │ loading      │  │ Built-in +   │  │  Updates      │       │
  │ Export/      │  │ PluginMgr    │  │ Custom       │  │  Skills/      │       │
  │ Import       │  │ 4 kinds      │  │              │  │  Toolsets     │       │
  └──────────────┘  └──────┬───────┘  └──────────────┘  └───────────────┘       │
                           │                                                       │
                  ┌────────▼──────────┐   ┌──────────────┐                        │
                  │ plugins_cmd.py    │   │  tips.py     │                        │
                  │ (1544)            │   │  (487)       │                        │
                  │ install/update/   │   │  Tips corpus │                        │
                  │ remove/enable/    │   │  Random      │                        │
                  │ disable/list      │   │  discovery   │                        │
                  └──────────────────┘   └──────────────┘                        │
                                                                               │
                              ┌──────────────────────────────────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  config.py (4939) │
                    │  Config read/write│
                    │  core             │
                    │  YAML / Env vars  │
                    │  ~/.hermes/       │
                    └───────────────────┘
```

## TL;DR

hermes_cli is the user interface layer for Hermes Agent, comprising 67 Python files totaling 77,169 lines of code. It uses `main.py` (10,876 lines) as the central dispatcher, routing CLI subcommands to various functional modules. The authentication system (`auth.py`) supports 30+ providers, covering four modes — OAuth device code, external OAuth, API Key, and out-of-process authentication — with unified priority resolution via `PROVIDER_REGISTRY` and `resolve_provider()`. Gateway management (`gateway.py`) registers the agent backend as a systemd/launchd service, providing cross-platform process lifecycle management. The kanban engine (`kanban_db.py`) builds a complete SQLite task scheduling system with atomic claiming, circuit breakers, heartbeat detection, and hallucinated card verification. The entire subsystem uses `config.py` to centrally manage YAML configuration and state persistence under `~/.hermes/`.

---

## 1. Authentication System — auth.py (5157 lines)

### 1.1 Architecture Overview

`auth.py` implements Hermes' multi-provider authentication infrastructure, serving as the core bridge connecting users to inference backends. It maintains a `PROVIDER_REGISTRY` dictionary that statically registers authentication configurations for 30+ providers, and implements a runtime provider selection priority chain via `resolve_provider()`. Authentication state is persisted in `~/.hermes/auth.json`, with cross-process file locking to ensure concurrency safety.

### 1.2 ProviderConfig — Provider Registry

```python
# Source location: hermes_cli/auth.py:133-147
@dataclass
class ProviderConfig:
    """Describes a known inference provider."""
    id: str
    name: str
    auth_type: str  # "oauth_device_code", "oauth_external", "oauth_minimax", or "api_key"
    portal_base_url: str = ""
    inference_base_url: str = ""
    client_id: str = ""
    scope: str = ""
    extra: Dict[str, Any] = field(default_factory=dict)
    api_key_env_vars: tuple = ()
    base_url_env_var: str = ""
```

`PROVIDER_REGISTRY` registers all known providers as `id -> ProviderConfig` mappings, covering:

| Provider ID | Name | Auth Type | Description |
|-------------|------|-----------|-------------|
| `nous` | Nous Portal | `oauth_device_code` | Primary OAuth provider |
| `openai-codex` | OpenAI Codex | `oauth_external` | External OAuth flow |
| `qwen-oauth` | Qwen OAuth | `oauth_external` | Tongyi Qianwen OAuth |
| `google-gemini-cli` | Google Gemini (OAuth) | `oauth_external` | Gemini Cloud Code |
| `minimax-oauth` | MiniMax (OAuth) | `oauth_minimax` | Dedicated MiniMax OAuth |
| `anthropic` | Anthropic | `api_key` | Claude API Key |
| `copilot` | GitHub Copilot | `api_key` | GH_TOKEN / COPILOT_GITHUB_TOKEN |
| `deepseek` | DeepSeek | `api_key` | DEEPSEEK_API_KEY |
| `xai` | xAI | `api_key` | XAI_API_KEY |
| `lmstudio` | LM Studio | `api_key` | Local inference, LM_API_KEY |
| `zai` | Z.AI / GLM | `api_key` | GLM_API_KEY |
| `kimi-coding` | Kimi / Moonshot | `api_key` | KIMI_API_KEY |
| `alibaba` | Alibaba Cloud (DashScope) | `api_key` | DASHSCOPE_API_KEY |
| ... | ... | ... | 15+ more providers |

### 1.3 Authentication Type Comparison

| Auth Type | Flow | Token Management | Representative Providers |
|-----------|------|------------------|--------------------------|
| `oauth_device_code` | Device code authorization: POST `/device/code` -> user browser verification -> poll `/token` | access_token + refresh_token auto-refresh | Nous Portal |
| `oauth_external` | Read external CLI tool token files (e.g. `~/.codex/auth.json`) | Proxy-refresh external tokens | OpenAI Codex, Qwen, Gemini |
| `oauth_minimax` | MiniMax-specific user_code authorization flow | access_token periodic refresh | MiniMax OAuth |
| `api_key` | Read API Key from environment variables | No expiry, user manual rotation | Anthropic, DeepSeek, xAI, etc. |
| `external_process` | Call external process to obtain temporary credentials | Process-level lifecycle | Copilot ACP |

### 1.4 resolve_provider() — Provider Selection Priority

```python
# Source location: hermes_cli/auth.py:1298-1313
def resolve_provider(
    requested: Optional[str] = None,
    *,
    explicit_api_key: Optional[str] = None,
    explicit_base_url: Optional[str] = None,
) -> str:
    """
    Determine which inference provider to use.

    Priority (when requested="auto" or None):
    1. active_provider in auth.json with valid credentials
    2. Explicit CLI api_key/base_url -> "openrouter"
    3. OPENAI_API_KEY or OPENROUTER_API_KEY env vars -> "openrouter"
    4. Provider-specific API keys (GLM, Kimi, MiniMax) -> that provider
    5. Fallback: "openrouter"
    """
```

`resolve_provider()` also maintains a `_PROVIDER_ALIASES` local mapping (defined inside the function body, approximately 70+ hardcoded entries + plugin dynamic extensions) that normalizes common user shorthands (e.g. `glm` -> `zai`, `grok` -> `xai`, `claude` -> `anthropic`) to canonical provider IDs. This mapping resides inside the `resolve_provider()` function body (source location: hermes_cli/auth.py:1317-1361), not as a module-level variable.

### 1.5 AuthError — Structured Authentication Error

```python
# Source location: hermes_cli/auth.py:689-729
class AuthError(RuntimeError):
    """Structured auth error with UX mapping hints."""
    def __init__(
        self,
        message: str,
        *,
        provider: str = "",
        code: Optional[str] = None,
        relogin_required: bool = False,
    ) -> None:
        super().__init__(message)
        self.provider = provider
        self.code = code
        self.relogin_required = relogin_required
```

`AuthError` implements fine-grained error mapping through the `code` field:

| Error Code | User Message | Severity |
|-------------|-------------|----------|
| `subscription_required` | "No active paid subscription found on Nous Portal" | Blocking |
| `insufficient_credits` | "Subscription credits are exhausted" | Blocking |
| `temporarily_unavailable` | "Please retry in a few seconds" | Transient |
| `relogin_required=True` | "Run `hermes model` to re-authenticate" | Action Required |

### 1.6 OAuth Device Code Flow

```python
# Source location: hermes_cli/auth.py:2675-2699
def _request_device_code(
    client: httpx.Client,
    portal_base_url: str,
    client_id: str,
    scope: Optional[str],
) -> Dict[str, Any]:
    """POST to the device code endpoint. Returns device_code, user_code, etc."""
    response = client.post(
        f"{portal_base_url}/api/oauth/device/code",
        data={
            "client_id": client_id,
            **({"scope": scope} if scope else {}),
        },
    )
    response.raise_for_status()
    data = response.json()
    required_fields = [
        "device_code", "user_code", "verification_uri",
        "verification_uri_complete", "expires_in", "interval",
    ]
    missing = [f for f in required_fields if f not in data]
    if missing:
        raise ValueError(f"Device code response missing fields: {', '.join(missing)}")
    return data
```

The device code flow follows the RFC 8628 standard: the client POSTs to `/api/oauth/device/code` to obtain a `device_code` and `user_code`, then polls `/api/oauth/token` with exponential backoff in `_poll_for_token()` until the user completes authorization in the browser.

### 1.7 Token Storage and File Locking

Authentication state is persisted in `~/.hermes/auth.json` (`_auth_file_path()`), using `fcntl` (Linux/macOS) or `msvcrt` (Windows) for cross-process mutual exclusion locking, with a lock timeout of 15 seconds (`AUTH_LOCK_TIMEOUT_SECONDS`). Key functions include:

- `_load_auth_store()` — Load the entire auth.json
- `_save_auth_store()` — Atomic write (via `atomic_replace`)
- `_load_provider_state()` / `_save_provider_state()` — Read/write sub-state by provider ID
- `read_credential_pool()` / `write_credential_pool()` — Credential pool management

---

## 2. Gateway Management — gateway.py (4810 lines)

### 2.1 Architecture Overview

`gateway.py` is responsible for running the Hermes agent backend as a system service, supporting both systemd (Linux) and launchd (macOS) service managers. It provides process discovery, PID tracking, health checks, automatic restart, and interactive platform configuration.

### 2.2 GatewayRuntimeSnapshot — Runtime State Snapshot

```python
# Source location: hermes_cli/gateway.py:48-61
@dataclass
class GatewayRuntimeSnapshot:
    manager: str
    service_installed: bool = False
    service_running: bool = False
    gateway_pids: tuple[int, ...] = ()
    service_scope: str | None = None

    @property
    def running(self) -> bool:
        return self.service_running or bool(self.gateway_pids)

    @property
    def has_process_service_mismatch(self) -> bool:
        return self.service_installed and self.running and not self.service_running
```

`GatewayRuntimeSnapshot` provides a unified runtime state view; the `has_process_service_mismatch` property detects the anomalous state of "process running but service not registered."

### 2.3 Service Manager Comparison

| Feature | systemd (Linux) | launchd (macOS) |
|---------|-----------------|-----------------|
| Install | `systemd_install()` writes `.service` unit | `launchd_install()` writes `.plist` |
| Start | `systemd_start()` -> `systemctl start` | `launchd_start()` -> `launchctl load` |
| Stop | `systemd_stop()` -> `systemctl stop` | `launchd_stop()` -> `launchctl unload` |
| Restart | `systemd_restart()` with start-limit protection | `launchd_restart()` with exit wait |
| Status | `systemd_status(deep=)` | `launchd_status(deep=)` |
| Scope | user / system dual scope | User scope |
| PID Tracking | `MainPID` attribute + `_get_service_pids()` | plist PID read |
| Auto-restart | exit code 75 -> `Restart=always` | `KeepAlive=true` |

### 2.4 get_gateway_runtime_snapshot() — Unified State Detection

```python
# Source location: hermes_cli/gateway.py:747-787
def get_gateway_runtime_snapshot(system: bool = False) -> GatewayRuntimeSnapshot:
    """Return a unified view of gateway liveness for the current profile."""
    gateway_pids = tuple(find_gateway_pids())
    if is_termux():
        return GatewayRuntimeSnapshot(
            manager="Termux / manual process",
            gateway_pids=gateway_pids,
        )
    if is_linux() and is_container():
        return GatewayRuntimeSnapshot(
            manager="docker (foreground)",
            gateway_pids=gateway_pids,
        )
    if supports_systemd_services():
        selected_system, service_running = _probe_systemd_service_running(system=system)
        scope_label = _service_scope_label(selected_system)
        return GatewayRuntimeSnapshot(
            manager=f"systemd ({scope_label})",
            service_installed=get_systemd_unit_path(system=selected_system).exists(),
            service_running=service_running,
            gateway_pids=gateway_pids,
            service_scope=scope_label,
        )
    if is_macos():
        return GatewayRuntimeSnapshot(
            manager="launchd",
            service_installed=get_launchd_plist_path().exists(),
            service_running=_probe_launchd_service_running(),
            gateway_pids=gateway_pids,
            service_scope="launchd",
        )
    return GatewayRuntimeSnapshot(
        manager="manual process",
        gateway_pids=gateway_pids,
    )
```

This function detects by environment priority: Termux -> Docker container -> systemd -> launchd -> manual process.

### 2.5 run_gateway() — Foreground Execution

```python
# Source location: hermes_cli/gateway.py:2733-2778
def run_gateway(verbose: int = 0, quiet: bool = False, replace: bool = False):
    """Run the gateway in foreground."""
    # Refresh systemd unit on every boot for updated restart settings
    if supports_systemd_services():
        try:
            refresh_systemd_unit_if_needed(system=False)
        except Exception:
            pass  # best-effort; don't block gateway startup

    from gateway.run import start_gateway
    # Exit with code 1 if gateway fails to connect any platform,
    # so systemd Restart=always will retry on transient errors
    try:
        success = asyncio.run(start_gateway(replace=replace, verbosity=verbosity))
    except KeyboardInterrupt:
        print("\nGateway stopped.")
        return
    if not success:
        sys.exit(1)
```

When the gateway runs in the foreground, exit code 1 triggers systemd `Restart=always` automatic restart.

### 2.6 Gateway Platform Configuration

`gateway_setup()` and the `_PLATFORMS` dictionary define the interactive configuration flow for messaging platforms. Each platform entry includes:

| Platform | Configuration Items | Emoji |
|----------|---------------------|-------|
| Telegram | `TELEGRAM_BOT_TOKEN` | :iphone: |
| Discord | `DISCORD_BOT_TOKEN`, `DISCORD_GUILD_IDS` | :video_game: |
| Slack | `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN` | :briefcase: |
| Matrix | `MATRIX_HOMESERVER`, `MATRIX_ACCESS_TOKEN` | :speech_balloon: |
| Mattermost | `MATTERMOST_TOKEN` | :speech_balloon: |
| WeChat/WeCom | `WEWORK_WEBHOOK_URL` / `WEIXIN_TOKEN` | :envelope: |
| Feishu (Lark) | `FEISHU_APP_ID`, `FEISHU_APP_SECRET` | :envelope: |
| DingTalk | `DINGTALK_TOKEN` | :envelope: |
| QQ Bot | `QQBOT_APPID`, `QQBOT_TOKEN` | :envelope: |
| Signal | `SIGNAL_PHONE_NUMBER` | :signal_strength: |
| WhatsApp | `WHATSAPP_JID`, `WHATSAPP_PASSWORD` | :phone: |

---

## 3. Model Catalog — models.py (3556 lines)

### 3.1 Architecture Overview

`models.py` manages the model catalog, pricing information, provider detection, and alias resolution. It maintains a static model list as an offline fallback while pulling real-time catalogs from OpenRouter, Anthropic, GitHub Copilot, LM Studio, and other endpoints.

### 3.2 Model Catalog Comparison

| Catalog Source | Function | Cache | Refresh Mechanism |
|----------------|----------|-------|-------------------|
| OpenRouter | `fetch_openrouter_models()` | `_openrouter_catalog_cache` | 8s timeout, `force_refresh` |
| Anthropic | `_fetch_anthropic_models()` | Inline cache | 5s timeout |
| GitHub Copilot | `fetch_github_model_catalog()` | None | Real-time fetch each time |
| LM Studio | `fetch_lmstudio_models()` | None | `probe_lmstudio_models()` detection |
| AI Gateway (Vercel) | `fetch_ai_gateway_models()` | Inline cache | `force_refresh` |
| Nous Portal | `fetch_nous_recommended_models()` | Inline cache | `force_refresh` |

### 3.3 ProviderEntry — Provider Catalog Entry

```python
# Source location: hermes_cli/models.py:772
class ProviderEntry(NamedTuple):
```

`ProviderEntry` structures each provider's catalog information as a `NamedTuple`, used by `provider_model_ids()` to aggregate model lists by provider.

### 3.4 detect_provider_for_model() — Automatic Provider Detection

```python
# Source location: hermes_cli/models.py:1626-1662
def detect_provider_for_model(
    model_name: str,
    current_provider: str,
) -> Optional[tuple[str, str]]:
    """Auto-detect the best provider for a model name.
    Priority:
    0. Bare provider name -> switch to that provider's default model
    1. Direct provider static catalog match
    2. OpenRouter catalog match
    """
```

Detection priority chain: bare provider name -> static catalog match -> OpenRouter catalog match.

### 3.5 Model Pricing and Free Model Detection

```python
# Source location: hermes_cli/models.py:928-960
def _openrouter_model_is_free(pricing: Any) -> bool:
    """Return True when both prompt and completion pricing are zero."""

def _openrouter_model_supports_tools(item: Any) -> bool:
    """Return True when the model's supported_parameters advertise tool calling."""
```

| Filter | Logic | Strictness |
|--------|-------|------------|
| `_openrouter_model_is_free` | prompt price = 0 AND completion price = 0 | Exact |
| `_openrouter_model_supports_tools` | `supported_parameters` contains `"tools"`; defaults to allowed when field is missing | Lenient |
| `_is_model_free` | Generic pricing field check | General |

---

## 4. Interactive Setup Wizard — setup.py (3449 lines)

### 4.1 Architecture Overview

`setup.py` implements Hermes' first-run and reconfiguration wizard, using a modular staged design where each section can run independently. It also includes the OpenClaw migration tool, supporting smooth migration from the legacy agent.

### 4.2 Five-Stage Wizard Structure

| Stage | Entry Function | Config Domain | Content |
|-------|---------------|---------------|---------|
| 1. Model & Provider | `setup_model_provider()` | `model` | Select AI provider and model |
| 2. Terminal Backend | `setup_terminal_backend()` | `terminal` | Docker/local/container execution backend |
| 3. Agent Settings | `setup_agent_settings()` | `agent` | Iteration count, compression, session reset |
| 4. Messaging Platforms | `setup_gateway()` | `gateway` | Telegram/Discord/Slack and other messaging platforms |
| 5. Tools | `setup_tools()` | `tools` | TTS/Web search/image generation and other tools |

### 4.3 Interactive Helper Functions

```python
# Source location: hermes_cli/setup.py:197-278
def prompt(question: str, default: str = None, password: bool = False) -> str:
def prompt_choice(question: str, choices: list, default: int = 0, ...) -> int:
def prompt_yes_no(question: str, default: bool = True) -> bool:
def prompt_checklist(title: str, items: list, pre_selected: list = None) -> list:
```

| Function | Interaction Mode | Purpose |
|----------|-----------------|---------|
| `prompt()` | Free text input | API Key, URL, name |
| `prompt_choice()` | Numeric selection | Provider/model selection |
| `prompt_yes_no()` | Y/n confirmation | Feature enable/skip |
| `prompt_checklist()` | Multi-select list | Toolset/platform selection |

### 4.4 Configured Section Skip Mechanism

```python
# Source location: hermes_cli/setup.py:2726-2738
def _skip_configured_section(
    config: dict, section_key: str, label: str
) -> bool:
    """Show an already-configured section summary and offer to skip.
    Returns True if the user chose to skip, False if the section should run.
    """
    summary = _get_section_config_summary(config, section_key)
    if not summary:
        return False
    print()
    print_success(f"  {label}: {summary}")
    return not prompt_yes_no(f"  Reconfigure {label.lower()}?", default=False)
```

### 4.5 OpenClaw Migration

`_load_openclaw_migration_module()` dynamically loads `optional-skills/migration/openclaw-migration/scripts/openclaw_to_hermes.py` as a module, performs a dry-run preview before deciding whether to migrate. High-impact items (gateway tokens, configuration values, soul files) are flagged as warnings.

### 4.6 TTS Provider Configuration

`setup_tts()` supports configuration for three TTS backends:

| TTS Backend | Dependency Installation | Configuration |
|-------------|------------------------|---------------|
| NeuTTS | espeak-ng + Python packages | `TTS_PROVIDER=neutts` |
| KittenTTS | Dedicated Python packages | `TTS_PROVIDER=kittentts` |
| System TTS | No additional dependencies | `TTS_PROVIDER=system` |

---

## 5. Diagnostics Tool — doctor.py (1520 lines)

### 5.1 Architecture Overview

`doctor.py` is Hermes' health check tool, performing Python environment verification, dependency checks, configuration consistency, provider connectivity, and gateway service status checks. It supports a `--fix` parameter to automatically repair certain issues.

### 5.2 Check Categories

| Check Category | Check Content | Auto-Fixable |
|----------------|---------------|--------------|
| Python Environment | Version >= 3.10, bytecode cache | Yes |
| Dependencies | Required Python package installation status | Partial |
| Configuration Files | `~/.hermes/config.yaml` integrity | Yes |
| API Key Providers | Connectivity testing for 30+ providers | No |
| Gateway Service | systemd/launchd service status | Partial |
| Tool Availability | Dependency checks for each toolset | Partial |

### 5.3 API Key Provider Health Check

```python
# Source location: hermes_cli/doctor.py:195-268
def _build_apikey_providers_list() -> list:
    """Build the API-key provider health-check list once and cache it.
    Tuple format: (name, env_vars, default_url, base_env, supports_models_endpoint)
    """
    _static = [
        ("Z.AI / GLM",      ("GLM_API_KEY", "ZAI_API_KEY", "Z_AI_API_KEY"), ...),
        ("Kimi / Moonshot",  ("KIMI_API_KEY",), ...),
        ("DeepSeek",         ("DEEPSEEK_API_KEY",), ...),
        # ... 15+ more providers
    ]
```

The static provider list is dynamically extended by plugin providers registered in `providers.py`.

### 5.4 Diagnostic Output Format

```python
# Source location: hermes_cli/doctor.py:146-157
def check_ok(text: str, detail: str = ""):    # ✓ Green
def check_warn(text: str, detail: str = ""):   # ⚠ Yellow
def check_fail(text: str, detail: str = ""):   # ✗ Red
def check_info(text: str):                      # ℹ Blue
```

| Level | Symbol | Meaning | Auto-Fix |
|-------|--------|---------|----------|
| `check_ok` | ✓ | Check passed | N/A |
| `check_warn` | ⚠ | Non-critical issue | Conditional |
| `check_fail` | ✗ | Critical issue | `--fix` attempt |
| `check_info` | ℹ | Informational hint | N/A |

---

## 6. Kanban Database — kanban_db.py (4087 lines)

### 6.1 Architecture Overview

`kanban_db.py` builds a complete SQLite kanban engine, managing the full lifecycle of Tasks, Runs, Comments, and Events. It implements atomic claiming, heartbeat detection, circuit breakers, and hallucinated card verification, serving as the core infrastructure for Hermes multi-agent collaboration.

### 6.2 Data Model

```python
# Source location: hermes_cli/kanban_db.py:556-635
class Task:
    """In-memory view of a row from the tasks table."""
    id: str
    title: str
    body: Optional[str]
    assignee: Optional[str]
    status: str              # todo -> ready -> running -> done/blocked
    priority: int
    claim_lock: Optional[str]
    claim_expires: Optional[int]
    consecutive_failures: int = 0    # Circuit breaker counter
    worker_pid: Optional[int] = None
    last_failure_error: Optional[str] = None
    max_runtime_seconds: Optional[int] = None
    skills: Optional[list] = None    # Force-loaded skills

# Source location: hermes_cli/kanban_db.py:663-713
class Run:
    """In-memory view of a task_runs row."""
    id: int
    task_id: str
    profile: Optional[str]
    status: str
    outcome: Optional[str]   # completed, timed_out, crashed, spawn_failure, reclaimed
    summary: Optional[str]
    metadata: Optional[dict]
    error: Optional[str]
```

| Model | Table | Core Fields | Purpose |
|-------|-------|-------------|---------|
| `Task` | `tasks` | id, title, status, assignee, consecutive_failures | Work items |
| `Run` | `task_runs` | task_id, profile, outcome, summary | Execution attempts |
| `Comment` | `comments` | task_id, author, body | Human comments |
| `Event` | `events` | task_id, kind, payload (JSON) | Audit trail |

### 6.3 Task State Machine

```
  todo ──(parents done)──> ready ──(claim)──> running ──(complete)──> done
                              │                    │
                              │                    ├──(block)──> blocked
                              │                    ├──(crash/timeout)──> ready (reclaim)
                              │                    └──(spawn failure)──> ready (retry)
                              │
                              └──(archive)──> archived
```

| Transition | Trigger Function | Condition |
|------------|-----------------|-----------|
| `todo -> ready` | `recompute_ready()` | All parent tasks `done` |
| `ready -> running` | `claim_task()` | CAS lock-free + create Run |
| `running -> done` | `complete_task()` | Includes hallucinated card verification |
| `running -> blocked` | `block_task()` | Circuit breaker triggered or manual |
| `running -> ready` | `release_stale_claims()` | TTL expired |
| `ready/blocked -> ready` | `unblock_task()` | Manual unblock |

### 6.4 claim_task() — Atomic Claiming

```python
# Source location: hermes_cli/kanban_db.py:1742-1790
def claim_task(
    conn: sqlite3.Connection,
    task_id: str,
    *,
    ttl_seconds: int = DEFAULT_CLAIM_TTL_SECONDS,
    claimer: Optional[str] = None,
) -> Optional[Task]:
    """Atomically transition ready -> running."""
    now = int(time.time())
    lock = claimer or _claimer_id()
    expires = now + int(ttl_seconds)
    with write_txn(conn):
        # Defensive: close leaked prior runs as 'reclaimed'
        stale = conn.execute(
            "SELECT current_run_id FROM tasks WHERE id = ? AND status = 'ready'",
            (task_id,),
        ).fetchone()
        if stale and stale["current_run_id"]:
            conn.execute(
                "UPDATE task_runs SET status = 'reclaimed', outcome = 'reclaimed', ...",
                (now, int(stale["current_run_id"])),
            )
        cur = conn.execute(
            """UPDATE tasks SET status = 'running', claim_lock = ?,
               claim_expires = ?, started_at = COALESCE(started_at, ?)
               WHERE id = ? AND status = 'ready' AND claim_lock IS NULL""",
            (lock, expires, now, task_id),
        )
        if cur.rowcount != 1:
            return None
```

The CAS (Compare-And-Swap) pattern ensures multiple workers cannot claim the same task simultaneously.

### 6.5 Circuit Breaker — consecutive_failures

Each `Task`'s `consecutive_failures` field is incremented on the following events:
- Spawn failure
- Timeout (worker exceeds `max_runtime_seconds`)
- Crash (worker PID disappears)

It is only reset to 0 on successful completion. `dispatch_once()` automatically blocks a task after `failure_limit` consecutive failures.

### 6.6 Hallucinated Card Verification

```python
# Source location: hermes_cli/kanban_db.py:2120-2179
def complete_task(
    conn: sqlite3.Connection,
    task_id: str,
    *,
    result: Optional[str] = None,
    summary: Optional[str] = None,
    created_cards: Optional[Iterable[str]] = None,
    ...
) -> bool:
    """
    created_cards: list of task ids the completing worker claims to have created.
    If any id is phantom (does not exist or was not created by this worker's
    assignee profile), completion is blocked with a HallucinatedCardsError.
    """
```

| Verification Layer | Timing | Behavior |
|--------------------|--------|----------|
| `created_cards` pre-verification | complete_task entry | Hallucinated card -> `HallucinatedCardsError` blocks completion |
| `suspected_hallucinated_references` post-scan | After successful completion | Ghost references in result/summary -> log event only, no blocking |

### 6.7 dispatch_once() — Dispatcher Single Tick

```python
# Source location: hermes_cli/kanban_db.py:3124-3173
def dispatch_once(
    conn: sqlite3.Connection,
    *,
    spawn_fn=None,
    ttl_seconds: int = DEFAULT_CLAIM_TTL_SECONDS,
    dry_run: bool = False,
    max_spawn: Optional[int] = None,
    failure_limit: int = DEFAULT_SPAWN_FAILURE_LIMIT,
    board: Optional[str] = None,
) -> DispatchResult:
    """Run one dispatcher tick.
    Steps:
      1. Reclaim stale running tasks (TTL expired).
      2. Reclaim crashed running tasks (host-local PID no longer alive).
      3. Promote todo -> ready where all parents are done.
      4. For each ready task with an assignee, atomically claim and call spawn_fn.
    """
```

| Step | Operation | Function |
|------|-----------|----------|
| 1 | Reclaim expired tasks | `release_stale_claims()` |
| 2 | Detect crashed workers | `detect_crashed_workers()` |
| 3 | Enforce timeout | `enforce_max_runtime()` |
| 4 | Promote to ready | `recompute_ready()` |
| 5 | Claim and dispatch | CAS claim -> `spawn_fn()` |

---

## 7. Tool Configuration — tools_config.py (2556 lines)

### 7.1 Architecture Overview

`tools_config.py` manages Hermes' toolset configuration, supporting per-platform (CLI/gateway/Telegram, etc.) independent enable/disable of toolsets, and providing interactive configuration for tools requiring API Keys.

### 7.2 Toolset Registry

```python
# Source location: hermes_cli/tools_config.py:52-82
CONFIGURABLE_TOOLSETS = [
    ("web",             "Web Search & Scraping",    "web_search, web_extract"),
    ("browser",         "Browser Automation",       "navigate, click, type, scroll"),
    ("terminal",        "Terminal & Processes",      "terminal, process"),
    ("file",            "File Operations",           "read, write, patch, search"),
    ("code_execution",  "Code Execution",            "execute_code"),
    ("vision",          "Vision / Image Analysis",   "vision_analyze"),
    ("video",           "Video Analysis",            "video_analyze"),
    ("image_gen",       "Image Generation",          "image_generate"),
    # ... more toolsets
]
```

### 7.3 Toolset Configuration Flow Comparison

| Flow | Entry | Purpose |
|------|-------|---------|
| First Install | `setup_tools(first_install=True)` | Simplified selection |
| Reconfiguration | `_reconfigure_tool()` | Modify existing configuration |
| Enable/Disable | `tools_disable_enable_command()` | Quick toggle |
| MCP Configuration | `_configure_mcp_tools_interactive()` | MCP server management |

### 7.4 Image Generation Providers

`_configure_imagegen_model()` supports multiple image generation backends including fal.ai, DALL-E, etc., and extends plugin-provided backends via `_plugin_image_gen_providers()`.

---

## 8. Slash Command Registration — commands.py (1713 lines)

### 8.1 Architecture Overview

`commands.py` defines the metadata registry for all Hermes slash commands, supporting command aliases, categories, tab completion, and cross-platform command mapping (Telegram/Discord/Slack).

### 8.2 CommandDef — Command Definition

```python
# Source location: hermes_cli/commands.py:46-58
class CommandDef:
    """Definition of a single slash command."""
    name: str                          # canonical name without slash: "background"
    description: str                   # human-readable description
    category: str                      # "Session", "Configuration", etc.
    aliases: tuple[str, ...] = ()      # alternative names: ("bg",)
    args_hint: str = ""                # argument placeholder: "<prompt>"
    subcommands: tuple[str, ...] = ()  # tab-completable subcommands
    cli_only: bool = False             # only available in CLI
    gateway_only: bool = False         # only available in gateway/messaging
    gateway_config_gate: str | None = None  # config dotpath for gateway visibility
```

### 8.3 Command Categories

| Category | Command Examples | Count |
|----------|-----------------|-------|
| Session | new, clear, retry, undo, branch, compress, rollback | ~20 |
| Configuration | model, config, personality, statusbar, verbose | ~10 |
| Tools | tools, browser, reload-mcp | ~5 |
| Agent | agents, goal, steer, queue, background | ~5 |
| Info | help, profile, usage, insights, gquota | ~5 |
| Admin | kanban, cron, plugins, doctor, update | ~10 |

### 8.4 Platform Command Mapping

| Platform | Mapping Function | Limitation |
|----------|-----------------|------------|
| Telegram | `telegram_menu_commands()` | Max 100 commands, name 1-32 characters |
| Discord | `discord_skill_commands_by_category()` | No hard limit |
| Slack | `slack_native_slashes()` | Requires manifest registration |

---

## 9. Runtime Model Switching — model_switch.py (1754 lines)

### 9.1 Architecture Overview

`model_switch.py` implements the core logic for runtime model switching, resolving user-input model names (which may be aliases, partial names, or provider names) into concrete `(model_id, provider, base_url, api_key)` quadruples.

### 9.2 ModelIdentity — Model Identity

```python
# Source location: hermes_cli/model_switch.py:99-153
class ModelIdentity(NamedTuple):
    """Vendor slug and family prefix used for catalog resolution."""
    vendor: str
    family: str

MODEL_ALIASES: dict[str, ModelIdentity] = {
    "sonnet":    ModelIdentity("anthropic", "claude-sonnet"),
    "opus":      ModelIdentity("anthropic", "claude-opus"),
    "haiku":     ModelIdentity("anthropic", "claude-haiku"),
    "gpt5":      ModelIdentity("openai", "gpt-5"),
    "gemini":    ModelIdentity("google", "gemini"),
    "deepseek":  ModelIdentity("deepseek", "deepseek-chat"),
    "grok":      ModelIdentity("x-ai", "grok"),
    "kimi":      ModelIdentity("moonshotai", "kimi"),
    "glm":       ModelIdentity("z-ai", "glm"),
    # ... 15+ more aliases
}
```

### 9.3 DirectAlias — Exact Mapping

```python
# Source location: hermes_cli/model_switch.py:165-176
class DirectAlias(NamedTuple):
    """Exact model mapping that bypasses catalog resolution."""
    model: str
    provider: str
    base_url: str
```

`DirectAlias` takes priority over `MODEL_ALIASES`, supporting user-defined mappings loaded from the `model_aliases:` section of `config.yaml`.

### 9.4 Resolution Priority Comparison

| Priority | Resolution Layer | Example |
|----------|-----------------|---------|
| 1 | DirectAlias (config file + built-in) | `ollama-cloud` -> exact mapping |
| 2 | MODEL_ALIASES (vendor+family) | `sonnet` -> `anthropic/claude-sonnet*` |
| 3 | Provider static catalog match | `claude-opus-4.7` -> `anthropic` |
| 4 | OpenRouter catalog fuzzy match | `deepseek-chat` -> `deepseek/deepseek-chat` |

### 9.5 ModelSwitchResult

```python
# Source location: hermes_cli/model_switch.py:263-279
class ModelSwitchResult:
    """Result of a model switch attempt."""
    success: bool
    new_model: str = ""
    target_provider: str = ""
    provider_changed: bool = False
    api_key: str = ""
    base_url: str = ""
    api_mode: str = ""
    error_message: str = ""
    warning_message: str = ""
    provider_label: str = ""
    resolved_via_alias: str = ""
    capabilities: Optional[ModelCapabilities] = None
    model_info: Optional[ModelInfo] = None
    is_global: bool = False
```

---

## 10. Multi-Profile Management — profiles.py (1241 lines)

### 10.1 Architecture Overview

`profiles.py` implements Hermes' multi-profile system, where each profile has its own `~/.hermes-<name>/` directory, configuration, gateway service, skills, and credentials. It supports create, switch, export, import, rename, and delete operations.

### 10.2 ProfileInfo — Profile Summary

```python
# Source location: hermes_cli/profiles.py:321-331
class ProfileInfo:
    """Summary information about a profile."""
    name: str
    path: Path
    is_default: bool
    gateway_running: bool
    model: Optional[str] = None
    provider: Optional[str] = None
    has_env: bool = False
    skill_count: int = 0
    alias_path: Optional[Path] = None
```

### 10.3 Profile Operation Comparison

| Operation | Function | Impact |
|-----------|----------|--------|
| List | `list_profiles()` | Read-only |
| Create | `create_profile()` | Create directory + seed skills |
| Delete | `delete_profile()` | Stop gateway + archive |
| Switch | `set_active_profile()` | Update symlink |
| Export | `export_profile()` | Package as tar.gz |
| Import | `import_profile()` | Restore from tar.gz |
| Rename | `rename_profile()` | Migrate directory + update services |

### 10.4 Wrapper Scripts

`create_wrapper_script()` generates a shell wrapper script (`hermes-<name>`) for each profile, enabling direct invocation as `hermes-<name> chat` for a specific profile without the `-p <name>` parameter.

---

## 11. Supplementary Modules

### 11.0 web_server.py (4063 lines) — Web Dashboard Server

`hermes_cli/web_server.py` provides the Web Dashboard REST API server, built on aiohttp. Key features:

- REST API endpoints: session management, configuration read/write, model switching, plugin status queries
- WebSocket real-time push: session message streams, tool execution progress, status change notifications
- Static file serving: frontend Dashboard UI assets
- Authentication middleware: JWT token verification + session binding

Source location: hermes_cli/web_server.py

### 11.0a kanban.py (2036 lines) — Kanban CLI Commands

`hermes_cli/kanban.py` provides the CLI subcommand entry point for the kanban system, bridging `kanban_db.py`'s SQLite engine to the user interaction layer. Main commands:

| Command | Function |
|---------|----------|
| `hermes kanban create` | Create kanban and tasks |
| `hermes kanban show` | Display kanban status |
| `hermes kanban list` | List tasks |
| `hermes kanban claim` | Claim task (CAS atomic claim) |
| `hermes kanban complete` | Complete task (with hallucinated card verification) |

Works with `kanban_db.py` to implement the CAS atomic claim + circuit breaker pattern. Source location: hermes_cli/kanban.py

### 11.0b skills_hub.py (1594 lines) — Skills Marketplace Interaction

`hermes_cli/skills_hub.py` provides CLI interaction functionality for the Skills Hub, supporting skill browsing, installation, and search:

| Command | Function |
|---------|----------|
| `hermes skills browse` | Browse available skills catalog |
| `hermes skills install <name>` | Install skill locally |
| `hermes skills search <query>` | Search skills |
| `hermes skills list` | List installed skills |

Source location: hermes_cli/skills_hub.py

### 11.0c config.py (4939 lines) — Configuration Read/Write Core

`hermes_cli/config.py` is the configuration core (4939 lines) for the entire CLI subsystem, already covered in detail in [14 — Configuration System](/en/chapters/14-configuration-system). Key responsibilities within the CLI subsystem:

- `load_config()` — stat-based cached configuration loading entry point (called by nearly all CLI modules)
- `save_config()` — Atomic YAML write (via `atomic_yaml_write` from utils.py:139)
- `get_config_path()` / `ensure_hermes_home()` — Path management
- `DEFAULT_CONFIG` — 386-line default value definition (model, providers, agent, terminal, toolsets, etc.)
- `_deep_merge()` / `_expand_env_vars()` / `_preserve_env_ref_templates()` — Configuration merge pipeline

Source location: hermes_cli/config.py

### 11.1 main.py (10876 lines) — CLI Dispatch Center

`main.py` is the entry point for the entire CLI; the `main()` function builds the argparse parser and registers all subcommands. It includes model selection flows (`_model_flow_openrouter`, `_model_flow_anthropic`, `_model_flow_copilot`, and 15+ provider-specific flow functions), update mechanisms (`cmd_update` with zip/git update paths), and TUI/Dashboard launch logic.

### 11.2 skin_engine.py (892 lines) — Theme Engine

```python
# Source location: hermes_cli/skin_engine.py:130-158
class SkinConfig:
    """Complete skin configuration."""
    name: str
    description: str = ""
    colors: Dict[str, str] = field(default_factory=dict)
    spinner: Dict[str, Any] = field(default_factory=dict)
    branding: Dict[str, str] = field(default_factory=dict)
    tool_prefix: str = "┊"
    tool_emojis: Dict[str, str] = field(default_factory=dict)
    banner_logo: str = ""
    banner_hero: str = ""
```

Built-in skins include `default` (gold theme), `ares`, `mono`, `slate`, `poseidon`, `charizard`, etc., with support for loading custom YAML skins from `~/.hermes/skins/`.

### 11.3 banner.py (635 lines) — ASCII Banner

`build_welcome_banner()` uses a Rich Table grid layout, displaying the Caduceus ASCII art on the left and model, toolsets, session ID, and context length on the right. `check_for_updates()` prefetches update status in the background.

### 11.4 tips.py (487 lines) — Tips Corpus

Contains 100+ random tips covering slash commands, @ context references, CLI flags, tools, gateway, skills, and workflow tips. `get_random_tip()` randomly selects one at each session start.

### 11.5 plugins.py (1362 lines) — Plugin Manager

```python
# Source location: hermes_cli/plugins.py:180-213
class PluginManifest:
    name: str
    version: str = ""
    kind: str = "standalone"  # standalone, backend, exclusive, platform
    key: str = ""
```

| Plugin Type | Loading Timing | Typical Use |
|-------------|---------------|-------------|
| `standalone` | Listed in `plugins.enabled` | Custom tools/hooks |
| `backend` | Built-in backend auto-load | image_gen backend |
| `exclusive` | Loaded on config selection | memory provider |
| `platform` | Built-in platform auto-load | IRC and other messaging platforms |

### 11.6 plugins_cmd.py (1544 lines) — Plugin CLI

Provides `hermes plugins install/update/remove/enable/disable/list/toggle` subcommands, supports Git URL and local path installation, and automatically runs `post_setup` hooks and environment variable configuration.

---

## Summary Table

| Component | Lines | Core Responsibility |
|-----------|-------|---------------------|
| main.py | 10876 | CLI entry point, command dispatch, model selection flow, updates |
| auth.py | 5157 | Multi-provider authentication, OAuth/API Key, token management |
| config.py | 4939 | Configuration read/write, YAML persistence |
| gateway.py | 4810 | systemd/launchd services, process management, platform configuration |
| kanban_db.py | 4087 | SQLite kanban, task CRUD, dispatcher, circuit breaker |
| models.py | 3556 | Model catalog, pricing, aliases, provider detection |
| setup.py | 3449 | Interactive wizard, staged configuration, migration |
| tools_config.py | 2556 | Toolset configuration, platform-level enable/disable |
| model_switch.py | 1754 | Runtime model switching, alias resolution |
| commands.py | 1713 | Slash command registration, completion, cross-platform mapping |
| plugins_cmd.py | 1544 | Plugin CLI commands |
| doctor.py | 1520 | Diagnostic checks, dependency verification |
| plugins.py | 1362 | Plugin discovery/loading/lifecycle |
| profiles.py | 1241 | Multi-profile management |
| skin_engine.py | 892 | Theme/color/branding customization |
| banner.py | 635 | ASCII welcome banner |
| tips.py | 487 | Random tips corpus |

---

[← 18 — Tool System Extended](/en/chapters/18-tool-system-extended) | [→ 20 — Agent Support Modules](/en/chapters/20-agent-support-modules)
