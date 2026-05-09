# 19 — Hermes CLI 子系统: 56K 行的命令行基础设施——调度/认证/配置/网关管理/模型选择/诊断/看板

## 源文件一览

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `hermes_cli/main.py` | 10876 | CLI 入口、命令调度、模型/提供者选择流程、更新机制 |
| `hermes_cli/auth.py` | 5157 | 多提供者认证、OAuth 设备码/API Key、令牌刷新 |
| `hermes_cli/config.py` | 4939 | 配置读写、YAML 持久化、环境变量解析 |
| `hermes_cli/gateway.py` | 4810 | 网关进程管理、systemd/launchd 服务、平台配置 |
| `hermes_cli/kanban_db.py` | 4087 | SQLite 看板数据库、任务 CRUD、调度器、熔断器 |
| `hermes_cli/web_server.py` | 4063 | Web Dashboard 服务端 |
| `hermes_cli/models.py` | 3556 | 模型目录、定价、别名解析、提供者检测 |
| `hermes_cli/setup.py` | 3449 | 交互式安装向导、分段配置、OpenClaw 迁移 |
| `hermes_cli/tools_config.py` | 2556 | 工具集配置、平台级启用/禁用、图像生成提供者 |
| `hermes_cli/kanban.py` | 2036 | 看板 CLI 命令、看板创建/任务管理 |
| `hermes_cli/model_switch.py` | 1754 | 运行时模型切换、别名解析、提供者热交换 |
| `hermes_cli/commands.py` | 1713 | 斜杠命令注册、补全、Telegram/Discord/Slack 命令映射 |
| `hermes_cli/skills_hub.py` | 1594 | 技能市场/Hub 交互 |
| `hermes_cli/plugins_cmd.py` | 1544 | 插件 CLI 命令（安装/更新/移除/启用/禁用/列表） |
| `hermes_cli/doctor.py` | 1520 | 诊断工具、依赖检查、连通性测试 |
| `hermes_cli/plugins.py` | 1362 | 插件发现/加载/生命周期管理、PluginManager |
| `hermes_cli/runtime_provider.py` | 1340 | 运行时提供者解析 |
| `hermes_cli/profiles.py` | 1241 | 多配置文件管理、创建/切换/导出/导入 |
| `hermes_cli/backup.py` | 939 | 配置备份与恢复 |
| `hermes_cli/skin_engine.py` | 892 | 皮肤/主题系统、颜色/Spinner/品牌定制 |
| `hermes_cli/voice.py` | 846 | 语音合成（TTS）集成 |
| `hermes_cli/claw.py` | 803 | Claw 工具集成 |
| `hermes_cli/nous_subscription.py` | 799 | Nous 订阅功能检测 |
| `hermes_cli/mcp_config.py` | 778 | MCP 服务器配置管理 |
| `hermes_cli/debug.py` | 746 | 调试输出与追踪 |
| `hermes_cli/auth_commands.py` | 718 | 认证子命令（login/logout/status） |
| `hermes_cli/providers.py` | 701 | 提供者注册表与扩展接口 |
| `hermes_cli/kanban_diagnostics.py` | 649 | 看板诊断与健康检查 |
| `hermes_cli/banner.py` | 635 | ASCII 欢迎横幅、版本/更新状态 |
| `hermes_cli/curator.py` | 560 | 内容策展工具 |
| `hermes_cli/status.py` | 551 | 系统状态显示 |
| `hermes_cli/goals.py` | 535 | 目标管理 |
| `hermes_cli/tips.py` | 487 | 随机提示语语料库 |
| `hermes_cli/uninstall.py` | 481 | 卸载与清理 |
| `hermes_cli/clipboard.py` | 481 | 剪贴板操作 |
| `hermes_cli/model_normalize.py` | 473 | 模型名称规范化 |
| `hermes_cli/curses_ui.py` | 472 | curses 终端 UI |
| `hermes_cli/memory_setup.py` | 457 | 记忆系统配置向导 |
| `hermes_cli/copilot_auth.py` | 392 | GitHub Copilot 认证 |
| `hermes_cli/logs.py` | 390 | 日志查看与管理 |
| `hermes_cli/hooks.py` | 385 | 钩子注册与执行 |
| `hermes_cli/_parser.py` | 373 | argparse 顶层解析器构建 |
| `hermes_cli/fallback_cmd.py` | 361 | 回退提供者链管理 |
| `hermes_cli/oneshot.py` | 331 | 单次执行模式 |
| `hermes_cli/model_catalog.py` | 329 | models.dev 目录拉取 |
| `hermes_cli/dump.py` | 325 | 配置/状态转储 |
| `hermes_cli/completion.py` | 315 | Shell 补全生成 |
| `hermes_cli/cron.py` | 307 | 定时任务管理 |
| `hermes_cli/azure_detect.py` | 300 | Azure 环境检测 |
| `hermes_cli/dingtalk_auth.py` | 293 | 钉钉认证 |
| `hermes_cli/webhook.py` | 274 | Webhook 管理 |
| `hermes_cli/checkpoints.py` | 244 | 文件系统检查点 |
| `hermes_cli/callbacks.py` | 242 | 回调函数注册 |
| `hermes_cli/pty_bridge.py` | 234 | PTY 桥接 |
| `hermes_cli/skills_config.py` | 177 | 技能配置 |
| `hermes_cli/codex_models.py` | 177 | Codex 模型列表 |
| `hermes_cli/env_loader.py` | 175 | 环境变量加载器 |
| `hermes_cli/slack_cli.py` | 153 | Slack CLI 集成 |
| `hermes_cli/relaunch.py` | 148 | 进程重启动 |
| `hermes_cli/browser_connect.py` | 138 | 浏览器 CDP 连接 |
| `hermes_cli/pairing.py` | 97 | 设备配对 |
| `hermes_cli/platforms.py` | 83 | 平台检测辅助 |
| `hermes_cli/timeouts.py` | 82 | 超时配置 |
| `hermes_cli/cli_output.py` | 78 | CLI 输出辅助函数 |
| `hermes_cli/vercel_auth.py` | 70 | Vercel 认证 |
| `hermes_cli/__init__.py` | 47 | 包初始化、版本号 |
| `hermes_cli/colors.py` | 38 | ANSI 颜色常量 |
| `hermes_cli/default_soul.py` | 11 | 默认灵魂文件路径 |

**总计: 77,169 行** (67 个 Python 文件)

## 一句话总结

hermes_cli 是 Hermes Agent 的 77K 行命令行基础设施，集成了多提供者认证、网关服务管理、模型目录与切换、交互式配置向导、看板 SQLite 引擎、插件系统、斜杠命令注册、诊断工具和主题引擎——构成了从 `hermes` 命令入口到代理运行时之间的完整用户界面层。

## 架构总览

```
                              ┌──────────────────────────────────────────────────┐
                              │                 hermes_cli/main.py               │
                              │            (10876 行 · CLI 入口 & 调度)          │
                              │  cmd_chat · cmd_model · cmd_setup · cmd_update  │
                              │  cmd_gateway · cmd_doctor · cmd_kanban · ...    │
                              └──────────┬──────────────────┬───────────────────┘
                                         │                  │
                    ┌────────────────────┘                  └────────────────────┐
                    │                                              │            │
         ┌──────────▼──────────┐                    ┌──────────────▼──────────┐ │
         │    auth.py (5157)   │                    │  gateway.py (4810)      │ │
         │  认证 & 令牌管理     │                    │  服务管理 & 平台配置     │ │
         │  ProviderConfig     │◄───────────────────│  systemd / launchd      │ │
         │  OAuth / API Key    │   resolve_provider │  run_gateway()          │ │
         │  resolve_provider() │                    │  gateway_setup()        │ │
         └──────────┬──────────┘                    └──────────────────────────┘ │
                    │                                              │            │
     ┌──────────────┼──────────────────────────────────────────────┤            │
     │              │                                              │            │
┌────▼────────┐ ┌───▼───────────┐  ┌──────────────────┐  ┌───────▼─────────┐  │
│ models.py   │ │ model_switch  │  │  setup.py (3449) │  │  doctor.py     │  │
│ (3556)      │ │ .py (1754)    │  │  交互式向导       │  │  (1520)        │  │
│ 模型目录     │ │ 运行时切换     │  │  5段式配置       │  │  诊断检查       │  │
│ 定价/别名    │ │ ModelIdentity │  │  OpenClaw 迁移   │  │  连通性测试     │  │
└──────┬──────┘ └───────┬───────┘  └──────────────────┘  └────────────────┘  │
       │                │                                                     │
       │    ┌───────────┼──────────────────────────────┐                     │
       │    │           │                              │                     │
┌──────▼────▼──────┐  ┌─▼───────────────┐  ┌──────────▼──────────┐         │
│ tools_config.py  │  │ commands.py     │  │  kanban_db.py (4087)│         │
│ (2556)           │  │ (1713)          │  │  SQLite 看板引擎     │         │
│ 工具集配置        │  │ 斜杠命令注册     │  │  Task/Run/Comment   │         │
│ 平台级启用/禁用   │  │ CommandDef      │  │  claim/complete     │         │
│ 图像生成提供者    │  │ 补全/帮助       │  │  dispatch_once()    │         │
└──────────────────┘  └─────────────────┘  │  熔断器             │         │
                                            └─────────────────────┘         │
                                                                                   │
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐       │
  │ profiles.py  │  │ plugins.py   │  │skin_engine.py│  │  banner.py    │       │
  │ (1241)       │  │ (1362)       │  │ (892)        │  │  (635)        │       │
  │ 多配置文件    │  │ 插件发现/加载 │  │ 主题/颜色     │  │  ASCII 横幅   │       │
  │ 创建/切换     │  │ PluginMgr    │  │ SkinConfig   │  │  版本/更新     │       │
  │ 导出/导入     │  │ 4种 kind     │  │ 内置+自定义   │  │  技能/工具集   │       │
  └──────────────┘  └──────┬───────┘  └──────────────┘  └───────────────┘       │
                           │                                                       │
                  ┌────────▼──────────┐   ┌──────────────┐                        │
                  │ plugins_cmd.py    │   │  tips.py     │                        │
                  │ (1544)            │   │  (487)       │                        │
                  │ install/update/   │   │  提示语料库   │                        │
                  │ remove/enable/    │   │  随机发现     │                        │
                  │ disable/list      │   │              │                        │
                  └──────────────────┘   └──────────────┘                        │
                                                                               │
                              ┌──────────────────────────────────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  config.py (4939) │
                    │  配置读写核心       │
                    │  YAML / 环境变量   │
                    │  ~/.hermes/       │
                    └───────────────────┘
```

## TL;DR

hermes_cli 是 Hermes Agent 的用户界面层，包含 67 个 Python 文件共 77,169 行代码。它以 `main.py`（10,876 行）为中心调度器，将 CLI 子命令路由到各个功能模块。认证系统（`auth.py`）支持 30+ 提供者，涵盖 OAuth 设备码、外部 OAuth、API Key 和进程外认证四种模式，通过 `PROVIDER_REGISTRY` 和 `resolve_provider()` 实现统一优先级解析。网关管理（`gateway.py`）将代理后台服务注册为 systemd/launchd 服务，提供跨平台进程生命周期管理。看板引擎（`kanban_db.py`）构建了完整的 SQLite 任务调度系统，含原子认领、熔断器、心跳检测和幻觉卡验证。整个子系统通过 `config.py` 统一管理 `~/.hermes/` 下的 YAML 配置与状态持久化。

---

## 1. 认证系统 — auth.py (5157 行)

### 1.1 架构概述

`auth.py` 实现了 Hermes 的多提供者认证基础设施，是连接用户与推理后端的核心桥梁。它维护一个 `PROVIDER_REGISTRY` 字典，将 30+ 提供者的认证配置静态注册，并通过 `resolve_provider()` 实现运行时的提供者选择优先级链。认证状态持久化于 `~/.hermes/auth.json`，使用跨进程文件锁保证并发安全。

### 1.2 ProviderConfig — 提供者注册表

```python
# 源码位置: hermes_cli/auth.py:133-147
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

`PROVIDER_REGISTRY` 以 `id -> ProviderConfig` 映射注册所有已知提供者，涵盖：

| 提供者 ID | 名称 | 认证类型 | 说明 |
|-----------|------|----------|------|
| `nous` | Nous Portal | `oauth_device_code` | 主 OAuth 提供者 |
| `openai-codex` | OpenAI Codex | `oauth_external` | 外部 OAuth 流 |
| `qwen-oauth` | Qwen OAuth | `oauth_external` | 通义千问 OAuth |
| `google-gemini-cli` | Google Gemini (OAuth) | `oauth_external` | Gemini Cloud Code |
| `minimax-oauth` | MiniMax (OAuth) | `oauth_minimax` | 专用 MiniMax OAuth |
| `anthropic` | Anthropic | `api_key` | Claude API Key |
| `copilot` | GitHub Copilot | `api_key` | GH_TOKEN / COPILOT_GITHUB_TOKEN |
| `deepseek` | DeepSeek | `api_key` | DEEPSEEK_API_KEY |
| `xai` | xAI | `api_key` | XAI_API_KEY |
| `lmstudio` | LM Studio | `api_key` | 本地推理，LM_API_KEY |
| `zai` | Z.AI / GLM | `api_key` | GLM_API_KEY |
| `kimi-coding` | Kimi / Moonshot | `api_key` | KIMI_API_KEY |
| `alibaba` | Alibaba Cloud (DashScope) | `api_key` | DASHSCOPE_API_KEY |
| ... | ... | ... | 还有 15+ 提供者 |

### 1.3 认证类型对比

| 认证类型 | 流程 | 令牌管理 | 代表提供者 |
|----------|------|----------|------------|
| `oauth_device_code` | 设备码授权：POST `/device/code` -> 用户浏览器验证 -> 轮询 `/token` | access_token + refresh_token 自动刷新 | Nous Portal |
| `oauth_external` | 读取外部 CLI 工具的令牌文件（如 `~/.codex/auth.json`） | 代理刷新外部令牌 | OpenAI Codex, Qwen, Gemini |
| `oauth_minimax` | MiniMax 专用 user_code 授权流 | access_token 定时刷新 | MiniMax OAuth |
| `api_key` | 从环境变量读取 API Key | 无过期，用户手动轮换 | Anthropic, DeepSeek, xAI 等 |
| `external_process` | 调用外部进程获取临时凭证 | 进程级生命周期 | Copilot ACP |

### 1.4 resolve_provider() — 提供者选择优先级

```python
# 源码位置: hermes_cli/auth.py:1298-1313
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

`resolve_provider()` 还维护了一个 `_PROVIDER_ALIASES` 局部映射（定义在函数内部，约 70+ 硬编码条目 + 插件动态扩展），将用户常见的简写（如 `glm` -> `zai`, `grok` -> `xai`, `claude` -> `anthropic`）归一化到规范提供者 ID。该映射位于 `resolve_provider()` 函数体内部（源码位置: hermes_cli/auth.py:1317-1361），而非模块级别变量。

### 1.5 AuthError — 结构化认证错误

```python
# 源码位置: hermes_cli/auth.py:689-729
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

`AuthError` 通过 `code` 字段实现细粒度的错误映射：

| 错误码 | 用户提示 | 严重程度 |
|--------|----------|----------|
| `subscription_required` | "No active paid subscription found on Nous Portal" | 阻断 |
| `insufficient_credits` | "Subscription credits are exhausted" | 阻断 |
| `temporarily_unavailable` | "Please retry in a few seconds" | 暂时 |
| `relogin_required=True` | "Run `hermes model` to re-authenticate" | 需操作 |

### 1.6 OAuth 设备码流程

```python
# 源码位置: hermes_cli/auth.py:2675-2699
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

设备码流程遵循 RFC 8628 标准：客户端 POST 到 `/api/oauth/device/code` 获取 `device_code` 和 `user_code`，然后在 `_poll_for_token()` 中以指数退避方式轮询 `/api/oauth/token` 直到用户在浏览器中完成授权。

### 1.7 令牌存储与文件锁

认证状态持久化于 `~/.hermes/auth.json`（`_auth_file_path()`），使用 `fcntl`（Linux/macOS）或 `msvcrt`（Windows）实现跨进程互斥锁，锁超时为 15 秒（`AUTH_LOCK_TIMEOUT_SECONDS`）。关键函数包括：

- `_load_auth_store()` — 加载整个 auth.json
- `_save_auth_store()` — 原子写入（通过 `atomic_replace`）
- `_load_provider_state()` / `_save_provider_state()` — 按提供者 ID 读写子状态
- `read_credential_pool()` / `write_credential_pool()` — 凭证池管理

---

## 2. 网关管理 — gateway.py (4810 行)

### 2.1 架构概述

`gateway.py` 负责将 Hermes 代理后台作为系统服务运行，支持 systemd（Linux）和 launchd（macOS）两种服务管理器。它提供进程发现、PID 跟踪、健康检查、自动重启和交互式平台配置。

### 2.2 GatewayRuntimeSnapshot — 运行时状态快照

```python
# 源码位置: hermes_cli/gateway.py:48-61
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

`GatewayRuntimeSnapshot` 提供统一的运行时状态视图，`has_process_service_mismatch` 属性检测"进程运行但服务未注册"的异常状态。

### 2.3 服务管理器对比

| 特性 | systemd (Linux) | launchd (macOS) |
|------|-----------------|-----------------|
| 安装 | `systemd_install()` 写 `.service` 单元 | `launchd_install()` 写 `.plist` |
| 启动 | `systemd_start()` -> `systemctl start` | `launchd_start()` -> `launchctl load` |
| 停止 | `systemd_stop()` -> `systemctl stop` | `launchd_stop()` -> `launchctl unload` |
| 重启 | `systemd_restart()` 含 start-limit 保护 | `launchd_restart()` 含退出等待 |
| 状态 | `systemd_status(deep=)` | `launchd_status(deep=)` |
| 作用域 | user / system 双作用域 | 用户作用域 |
| PID 追踪 | `MainPID` 属性 + `_get_service_pids()` | plist PID 读取 |
| 自重启 | exit code 75 -> `Restart=always` | `KeepAlive=true` |

### 2.4 get_gateway_runtime_snapshot() — 统一状态检测

```python
# 源码位置: hermes_cli/gateway.py:747-787
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

此函数按环境优先级检测：Termux -> Docker 容器 -> systemd -> launchd -> 手动进程。

### 2.5 run_gateway() — 前台运行

```python
# 源码位置: hermes_cli/gateway.py:2733-2778
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

网关前台运行时退出码 1 触发 systemd `Restart=always` 自动重启。

### 2.6 网关平台配置

`gateway_setup()` 和 `_PLATFORMS` 字典定义了消息平台的交互式配置流程，每个平台条目包含：

| 平台 | 配置项 | Emoji |
|------|--------|-------|
| Telegram | `TELEGRAM_BOT_TOKEN` | :iphone: |
| Discord | `DISCORD_BOT_TOKEN`, `DISCORD_GUILD_IDS` | :video_game: |
| Slack | `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN` | :briefcase: |
| Matrix | `MATRIX_HOMESERVER`, `MATRIX_ACCESS_TOKEN` | :speech_balloon: |
| Mattermost | `MATTERMOST_TOKEN` | :speech_balloon: |
| WeChat/企业微信 | `WEWORK_WEBHOOK_URL` / `WEIXIN_TOKEN` | :envelope: |
| 飞书 | `FEISHU_APP_ID`, `FEISHU_APP_SECRET` | :envelope: |
| 钉钉 | `DINGTALK_TOKEN` | :envelope: |
| QQ Bot | `QQBOT_APPID`, `QQBOT_TOKEN` | :envelope: |
| Signal | `SIGNAL_PHONE_NUMBER` | :signal_strength: |
| WhatsApp | `WHATSAPP_JID`, `WHATSAPP_PASSWORD` | :phone: |

---

## 3. 模型目录 — models.py (3556 行)

### 3.1 架构概述

`models.py` 管理模型目录、定价信息、提供者检测和别名解析。它维护静态模型列表作为离线后备，同时从 OpenRouter、Anthropic、GitHub Copilot、LM Studio 等端点拉取实时目录。

### 3.2 模型目录对比

| 目录源 | 函数 | 缓存 | 刷新机制 |
|--------|------|------|----------|
| OpenRouter | `fetch_openrouter_models()` | `_openrouter_catalog_cache` | 8s 超时，`force_refresh` |
| Anthropic | `_fetch_anthropic_models()` | 内联缓存 | 5s 超时 |
| GitHub Copilot | `fetch_github_model_catalog()` | 无 | 每次实时拉取 |
| LM Studio | `fetch_lmstudio_models()` | 无 | `probe_lmstudio_models()` 检测 |
| AI Gateway (Vercel) | `fetch_ai_gateway_models()` | 内联缓存 | `force_refresh` |
| Nous Portal | `fetch_nous_recommended_models()` | 内联缓存 | `force_refresh` |

### 3.3 ProviderEntry — 提供者目录条目

```python
# 源码位置: hermes_cli/models.py:772
class ProviderEntry(NamedTuple):
```

`ProviderEntry` 以 `NamedTuple` 形式结构化每个提供者的目录信息，被 `provider_model_ids()` 用于按提供者聚合模型列表。

### 3.4 detect_provider_for_model() — 自动提供者检测

```python
# 源码位置: hermes_cli/models.py:1626-1662
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

检测优先级链：裸提供者名 -> 静态目录匹配 -> OpenRouter 目录匹配。

### 3.5 模型定价与免费模型检测

```python
# 源码位置: hermes_cli/models.py:928-960
def _openrouter_model_is_free(pricing: Any) -> bool:
    """Return True when both prompt and completion pricing are zero."""

def _openrouter_model_supports_tools(item: Any) -> bool:
    """Return True when the model's supported_parameters advertise tool calling."""
```

| 过滤器 | 逻辑 | 严格度 |
|--------|------|--------|
| `_openrouter_model_is_free` | prompt 价格 = 0 且 completion 价格 = 0 | 精确 |
| `_openrouter_model_supports_tools` | `supported_parameters` 包含 `"tools"`；字段缺失时默认允许 | 宽松 |
| `_is_model_free` | 通用定价字段检查 | 通用 |

---

## 4. 交互式设置向导 — setup.py (3449 行)

### 4.1 架构概述

`setup.py` 实现了 Hermes 的首次运行和重新配置向导，采用模块化分段设计，每个部分可独立运行。它还包含 OpenClaw 迁移工具，支持从旧版代理平滑迁移。

### 4.2 五段式向导结构

| 段 | 入口函数 | 配置域 | 内容 |
|----|----------|--------|------|
| 1. Model & Provider | `setup_model_provider()` | `model` | 选择 AI 提供者和模型 |
| 2. Terminal Backend | `setup_terminal_backend()` | `terminal` | Docker/本地/容器执行后端 |
| 3. Agent Settings | `setup_agent_settings()` | `agent` | 迭代次数、压缩、会话重置 |
| 4. Messaging Platforms | `setup_gateway()` | `gateway` | Telegram/Discord/Slack 等消息平台 |
| 5. Tools | `setup_tools()` | `tools` | TTS/Web 搜索/图像生成等工具 |

### 4.3 交互式辅助函数

```python
# 源码位置: hermes_cli/setup.py:197-278
def prompt(question: str, default: str = None, password: bool = False) -> str:
def prompt_choice(question: str, choices: list, default: int = 0, ...) -> int:
def prompt_yes_no(question: str, default: bool = True) -> bool:
def prompt_checklist(title: str, items: list, pre_selected: list = None) -> list:
```

| 函数 | 交互模式 | 用途 |
|------|----------|------|
| `prompt()` | 自由文本输入 | API Key、URL、名称 |
| `prompt_choice()` | 数字选择 | 提供者/模型选择 |
| `prompt_yes_no()` | Y/n 确认 | 功能启用/跳过 |
| `prompt_checklist()` | 多选列表 | 工具集/平台选择 |

### 4.4 已配置段跳过机制

```python
# 源码位置: hermes_cli/setup.py:2726-2738
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

### 4.5 OpenClaw 迁移

`_load_openclaw_migration_module()` 动态加载 `optional-skills/migration/openclaw-migration/scripts/openclaw_to_hermes.py` 作为模块，执行干运行预览后决定是否迁移。高影响项（网关令牌、配置值、灵魂文件）会被标记为警告。

### 4.6 TTS 提供者配置

`setup_tts()` 支持三种 TTS 后端的配置：

| TTS 后端 | 依赖安装 | 配置项 |
|----------|----------|--------|
| NeuTTS | espeak-ng + Python 包 | `TTS_PROVIDER=neutts` |
| KittenTTS | 专用 Python 包 | `TTS_PROVIDER=kittentts` |
| 系统 TTS | 无额外依赖 | `TTS_PROVIDER=system` |

---

## 5. 诊断工具 — doctor.py (1520 行)

### 5.1 架构概述

`doctor.py` 是 Hermes 的健康检查工具，执行 Python 环境验证、依赖检查、配置一致性、提供者连通性和网关服务状态检查。支持 `--fix` 参数自动修复部分问题。

### 5.2 检查类别

| 检查类别 | 检查内容 | 可自动修复 |
|----------|----------|------------|
| Python 环境 | 版本 >= 3.10，字节码缓存 | 是 |
| 依赖项 | 必要 Python 包安装状态 | 部分 |
| 配置文件 | `~/.hermes/config.yaml` 完整性 | 是 |
| API Key 提供者 | 30+ 提供者的连通性测试 | 否 |
| 网关服务 | systemd/launchd 服务状态 | 部分 |
| 工具可用性 | 各工具集的依赖检查 | 部分 |

### 5.3 API Key 提供者健康检查

```python
# 源码位置: hermes_cli/doctor.py:195-268
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

静态提供者列表会被 `providers.py` 中注册的插件提供者动态扩展。

### 5.4 诊断输出格式

```python
# 源码位置: hermes_cli/doctor.py:146-157
def check_ok(text: str, detail: str = ""):    # ✓ 绿色
def check_warn(text: str, detail: str = ""):   # ⚠ 黄色
def check_fail(text: str, detail: str = ""):   # ✗ 红色
def check_info(text: str):                      # ℹ 蓝色
```

| 级别 | 符号 | 含义 | 自动修复 |
|------|------|------|----------|
| `check_ok` | ✓ | 检查通过 | N/A |
| `check_warn` | ⚠ | 非关键问题 | 视情况 |
| `check_fail` | ✗ | 关键问题 | `--fix` 尝试 |
| `check_info` | ℹ | 信息性提示 | N/A |

---

## 6. 看板数据库 — kanban_db.py (4087 行)

### 6.1 架构概述

`kanban_db.py` 构建了完整的 SQLite 看板引擎，管理任务（Task）、运行（Run）、评论（Comment）和事件（Event）的完整生命周期。它实现了原子认领、心跳检测、熔断器和幻觉卡验证，是 Hermes 多代理协作的核心基础设施。

### 6.2 数据模型

```python
# 源码位置: hermes_cli/kanban_db.py:556-635
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
    consecutive_failures: int = 0    # 熔断器计数器
    worker_pid: Optional[int] = None
    last_failure_error: Optional[str] = None
    max_runtime_seconds: Optional[int] = None
    skills: Optional[list] = None    # 强制加载的技能

# 源码位置: hermes_cli/kanban_db.py:663-713
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

| 模型 | 表 | 核心字段 | 用途 |
|------|-----|----------|------|
| `Task` | `tasks` | id, title, status, assignee, consecutive_failures | 工作项 |
| `Run` | `task_runs` | task_id, profile, outcome, summary | 执行尝试 |
| `Comment` | `comments` | task_id, author, body | 人工评论 |
| `Event` | `events` | task_id, kind, payload (JSON) | 审计追踪 |

### 6.3 任务状态机

```
  todo ──(parents done)──> ready ──(claim)──> running ──(complete)──> done
                              │                    │
                              │                    ├──(block)──> blocked
                              │                    ├──(crash/timeout)──> ready (reclaim)
                              │                    └──(spawn failure)──> ready (retry)
                              │
                              └──(archive)──> archived
```

| 转换 | 触发函数 | 条件 |
|------|----------|------|
| `todo -> ready` | `recompute_ready()` | 所有父任务 `done` |
| `ready -> running` | `claim_task()` | CAS 无锁 + 创建 Run |
| `running -> done` | `complete_task()` | 含幻觉卡验证 |
| `running -> blocked` | `block_task()` | 熔断器触发或手动 |
| `running -> ready` | `release_stale_claims()` | TTL 过期 |
| `ready/blocked -> ready` | `unblock_task()` | 手动解除阻塞 |

### 6.4 claim_task() — 原子认领

```python
# 源码位置: hermes_cli/kanban_db.py:1742-1790
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

CAS（Compare-And-Swap）模式确保多 worker 不会同时认领同一任务。

### 6.5 熔断器 — consecutive_failures

每个 `Task` 的 `consecutive_failures` 字段在以下事件递增：
- 派发失败（spawn failure）
- 超时（worker 超过 `max_runtime_seconds`）
- 崩溃（worker PID 消失）

仅在成功完成时重置为 0。`dispatch_once()` 在 `failure_limit` 次连续失败后自动阻塞任务。

### 6.6 幻觉卡验证

```python
# 源码位置: hermes_cli/kanban_db.py:2120-2179
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

| 验证层 | 时机 | 行为 |
|--------|------|------|
| `created_cards` 前置验证 | complete_task 入口 | 幻觉卡 -> `HallucinatedCardsError` 阻断完成 |
| `suspected_hallucinated_references` 后置扫描 | 完成成功后 | 结果/摘要中的幽灵引用 -> 仅记录事件，不阻断 |

### 6.7 dispatch_once() — 调度器单次滴答

```python
# 源码位置: hermes_cli/kanban_db.py:3124-3173
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

| 步骤 | 操作 | 函数 |
|------|------|------|
| 1 | 回收过期任务 | `release_stale_claims()` |
| 2 | 检测崩溃 worker | `detect_crashed_workers()` |
| 3 | 强制超时 | `enforce_max_runtime()` |
| 4 | 提升就绪 | `recompute_ready()` |
| 5 | 认领并派发 | CAS claim -> `spawn_fn()` |

---

## 7. 工具配置 — tools_config.py (2556 行)

### 7.1 架构概述

`tools_config.py` 管理 Hermes 的工具集配置，支持按平台（CLI/gateway/Telegram 等）独立启用/禁用工具集，并为需要 API Key 的工具提供交互式配置。

### 7.2 工具集注册表

```python
# 源码位置: hermes_cli/tools_config.py:52-82
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

### 7.3 工具集配置流程对比

| 流程 | 入口 | 用途 |
|------|------|------|
| 首次安装 | `setup_tools(first_install=True)` | 简化选择 |
| 重新配置 | `_reconfigure_tool()` | 修改已有配置 |
| 启用/禁用 | `tools_disable_enable_command()` | 快速切换 |
| MCP 配置 | `_configure_mcp_tools_interactive()` | MCP 服务器管理 |

### 7.4 图像生成提供者

`_configure_imagegen_model()` 支持多个图像生成后端，包括 fal.ai、DALL-E 等，通过 `_plugin_image_gen_providers()` 扩展插件提供的后端。

---

## 8. 斜杠命令注册 — commands.py (1713 行)

### 8.1 架构概述

`commands.py` 定义了 Hermes 所有斜杠命令的元数据注册表，支持命令别名、分类、Tab 补全和跨平台命令映射（Telegram/Discord/Slack）。

### 8.2 CommandDef — 命令定义

```python
# 源码位置: hermes_cli/commands.py:46-58
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

### 8.3 命令分类

| 分类 | 命令示例 | 数量 |
|------|----------|------|
| Session | new, clear, retry, undo, branch, compress, rollback | ~20 |
| Configuration | model, config, personality, statusbar, verbose | ~10 |
| Tools | tools, browser, reload-mcp | ~5 |
| Agent | agents, goal, steer, queue, background | ~5 |
| Info | help, profile, usage, insights, gquota | ~5 |
| Admin | kanban, cron, plugins, doctor, update | ~10 |

### 8.4 平台命令映射

| 平台 | 映射函数 | 限制 |
|------|----------|------|
| Telegram | `telegram_menu_commands()` | 最多 100 条，名称 1-32 字符 |
| Discord | `discord_skill_commands_by_category()` | 无硬限制 |
| Slack | `slack_native_slashes()` | 需 manifest 注册 |

---

## 9. 运行时模型切换 — model_switch.py (1754 行)

### 9.1 架构概述

`model_switch.py` 实现了运行时模型切换的核心逻辑，将用户输入的模型名（可能是别名、部分名、提供者名）解析为具体的 `(model_id, provider, base_url, api_key)` 四元组。

### 9.2 ModelIdentity — 模型身份

```python
# 源码位置: hermes_cli/model_switch.py:99-153
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

### 9.3 DirectAlias — 精确映射

```python
# 源码位置: hermes_cli/model_switch.py:165-176
class DirectAlias(NamedTuple):
    """Exact model mapping that bypasses catalog resolution."""
    model: str
    provider: str
    base_url: str
```

`DirectAlias` 优先于 `MODEL_ALIASES`，支持从 `config.yaml` 的 `model_aliases:` 节加载用户自定义映射。

### 9.4 解析优先级对比

| 优先级 | 解析层 | 示例 |
|--------|--------|------|
| 1 | DirectAlias（配置文件 + 内置） | `ollama-cloud` -> 精确映射 |
| 2 | MODEL_ALIASES（vendor+family） | `sonnet` -> `anthropic/claude-sonnet*` |
| 3 | 提供者静态目录匹配 | `claude-opus-4.7` -> `anthropic` |
| 4 | OpenRouter 目录模糊匹配 | `deepseek-chat` -> `deepseek/deepseek-chat` |

### 9.5 ModelSwitchResult

```python
# 源码位置: hermes_cli/model_switch.py:263-279
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

## 10. 多配置文件管理 — profiles.py (1241 行)

### 10.1 架构概述

`profiles.py` 实现了 Hermes 的多配置文件系统，每个配置文件拥有独立的 `~/.hermes-<name>/` 目录、配置、网关服务、技能和凭证。支持创建、切换、导出、导入、重命名和删除操作。

### 10.2 ProfileInfo — 配置文件摘要

```python
# 源码位置: hermes_cli/profiles.py:321-331
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

### 10.3 配置文件操作对比

| 操作 | 函数 | 影响 |
|------|------|------|
| 列出 | `list_profiles()` | 只读 |
| 创建 | `create_profile()` | 创建目录 + 种子技能 |
| 删除 | `delete_profile()` | 停止网关 + 归档 |
| 切换 | `set_active_profile()` | 更新符号链接 |
| 导出 | `export_profile()` | 打包为 tar.gz |
| 导入 | `import_profile()` | 从 tar.gz 还原 |
| 重命名 | `rename_profile()` | 迁移目录 + 更新服务 |

### 10.4 Wrapper 脚本

`create_wrapper_script()` 为每个配置文件生成 shell 包装脚本（`hermes-<name>`），使得可以直接以 `hermes-<name> chat` 形式调用特定配置文件，无需 `-p <name>` 参数。

---

## 11. 补充模块

### 11.0 web_server.py (4063 行) — Web Dashboard 服务端

`hermes_cli/web_server.py` 提供 Web Dashboard REST API 服务端，基于 aiohttp 构建。主要功能：

- REST API 端点：会话管理、配置读写、模型切换、插件状态查询
- WebSocket 实时推送：会话消息流、工具执行进度、状态变更通知
- 静态文件服务：前端 Dashboard UI 资源
- 认证中间件：JWT token 验证 + session 绑定

源码位置: hermes_cli/web_server.py

### 11.0a kanban.py (2036 行) — 看板 CLI 命令

`hermes_cli/kanban.py` 提供看板系统的 CLI 子命令入口，桥接 `kanban_db.py` 的 SQLite 引擎到用户交互层。主要命令：

| 命令 | 功能 |
|------|------|
| `hermes kanban create` | 创建看板和任务 |
| `hermes kanban show` | 显示看板状态 |
| `hermes kanban list` | 列出任务 |
| `hermes kanban claim` | 认领任务（CAS 原子认领） |
| `hermes kanban complete` | 完成任务（含幻觉卡验证） |

与 `kanban_db.py` 配合实现 CAS 原子认领 + 熔断器模式。源码位置: hermes_cli/kanban.py

### 11.0b skills_hub.py (1594 行) — 技能市场交互

`hermes_cli/skills_hub.py` 提供技能 Hub 的 CLI 交互功能，支持技能浏览、安装和搜索：

| 命令 | 功能 |
|------|------|
| `hermes skills browse` | 浏览可用技能目录 |
| `hermes skills install <name>` | 安装技能到本地 |
| `hermes skills search <query>` | 搜索技能 |
| `hermes skills list` | 列出已安装技能 |

源码位置: hermes_cli/skills_hub.py

### 11.0c config.py (4939 行) — 配置读写核心

`hermes_cli/config.py` 是整个 CLI 子系统的配置核心（4939 行），已在 [14 — Configuration System](/zh-CN/chapters/14-configuration-system) 详细覆盖。在 CLI 子系统中的主要职责：

- `load_config()` — stat-based 缓存的配置加载入口（被几乎所有 CLI 模块调用）
- `save_config()` — 原子 YAML 写入（通过 `atomic_yaml_write` from utils.py:139）
- `get_config_path()` / `ensure_hermes_home()` — 路径管理
- `DEFAULT_CONFIG` — 386 行默认值定义（model, providers, agent, terminal, toolsets 等）
- `_deep_merge()` / `_expand_env_vars()` / `_preserve_env_ref_templates()` — 配置合并管线

源码位置: hermes_cli/config.py

### 11.1 main.py (10876 行) — CLI 调度中心

`main.py` 是整个 CLI 的入口，`main()` 函数构建 argparse 解析器并注册所有子命令。它包含了模型选择流程（`_model_flow_openrouter`, `_model_flow_anthropic`, `_model_flow_copilot` 等 15+ 个提供者特定的流程函数）、更新机制（`cmd_update` 含 zip/git 两种更新路径）和 TUI/Dashboard 启动逻辑。

### 11.2 skin_engine.py (892 行) — 主题引擎

```python
# 源码位置: hermes_cli/skin_engine.py:130-158
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

内置皮肤包括 `default`（金色主题）、`ares`、`mono`、`slate`、`poseidon`、`charizard` 等，支持从 `~/.hermes/skins/` 加载自定义 YAML 皮肤。

### 11.3 banner.py (635 行) — ASCII 横幅

`build_welcome_banner()` 使用 Rich Table 网格布局，左侧显示 Caduceus ASCII art，右侧显示模型、工具集、会话 ID 和上下文长度。`check_for_updates()` 在后台预取更新状态。

### 11.4 tips.py (487 行) — 提示语料库

包含 100+ 条随机提示，覆盖斜杠命令、@ 上下文引用、CLI 标志、工具、网关、技能和工作流技巧。`get_random_tip()` 在每次会话启动时随机选择一条。

### 11.5 plugins.py (1362 行) — 插件管理器

```python
# 源码位置: hermes_cli/plugins.py:180-213
class PluginManifest:
    name: str
    version: str = ""
    kind: str = "standalone"  # standalone, backend, exclusive, platform
    key: str = ""
```

| 插件类型 | 加载时机 | 典型用途 |
|----------|----------|----------|
| `standalone` | `plugins.enabled` 中列出时 | 自定义工具/钩子 |
| `backend` | 内置后端自动加载 | image_gen 后端 |
| `exclusive` | 配置选择时加载 | memory provider |
| `platform` | 内置平台自动加载 | IRC 等消息平台 |

### 11.6 plugins_cmd.py (1544 行) — 插件 CLI

提供 `hermes plugins install/update/remove/enable/disable/list/toggle` 子命令，支持 Git URL 和本地路径安装，自动运行 `post_setup` 钩子和环境变量配置。

---

## 总结表

| 组件 | 行数 | 核心职责 |
|------|------|----------|
| main.py | 10876 | CLI 入口、命令调度、模型选择流程、更新 |
| auth.py | 5157 | 多提供者认证、OAuth/API Key、令牌管理 |
| config.py | 4939 | 配置读写、YAML 持久化 |
| gateway.py | 4810 | systemd/launchd 服务、进程管理、平台配置 |
| kanban_db.py | 4087 | SQLite 看板、任务 CRUD、调度器、熔断器 |
| models.py | 3556 | 模型目录、定价、别名、提供者检测 |
| setup.py | 3449 | 交互式向导、分段配置、迁移 |
| tools_config.py | 2556 | 工具集配置、平台级启用/禁用 |
| model_switch.py | 1754 | 运行时模型切换、别名解析 |
| commands.py | 1713 | 斜杠命令注册、补全、跨平台映射 |
| plugins_cmd.py | 1544 | 插件 CLI 命令 |
| doctor.py | 1520 | 诊断检查、依赖验证 |
| plugins.py | 1362 | 插件发现/加载/生命周期 |
| profiles.py | 1241 | 多配置文件管理 |
| skin_engine.py | 892 | 主题/颜色/品牌定制 |
| banner.py | 635 | ASCII 欢迎横幅 |
| tips.py | 487 | 随机提示语料库 |

---

[← 18 — 工具体系扩展](/zh-CN/chapters/18-tool-system-extended) | [→ 20 — 代理支撑模块](/zh-CN/chapters/20-agent-support-modules)
