# 07 — 前端交互界面: 四种前端共享同一引擎核心

[← 06-模型路由与推理](/zh-CN/chapters/06-gateway-system) | [→ 08-状态与持久化](/zh-CN/chapters/08-state-persistence)

---

## 源码文件

| 文件 | 行数 | 说明 |
|------|------|------|
| `cli.py` | 12,508 | CLI REPL 主入口, prompt_toolkit + ASCII branding + slash commands |
| `hermes_cli/web_server.py` | 4,063 | Web Dashboard FastAPI 后端, 50+ REST/WebSocket 端点 |
| `acp_adapter/server.py` | 1,428 | ACP Agent 服务器, JSON-RPC over stdio |
| `acp_adapter/session.py` | — | ACP 会话管理器, 映射 ACP session 到 AIAgent 实例 |
| `acp_adapter/entry.py` | — | ACP 入口, 环境加载 + 日志配置 |
| `acp_adapter/tools.py` | — | ACP 工具事件构建 |
| `acp_adapter/events.py` | — | ACP 事件回调工厂 |
| `acp_adapter/auth.py` | — | ACP 认证检测 |
| `acp_adapter/permissions.py` | — | ACP 审批回调 |
| `ui-tui/src/` | ~165 TS/TSX | TUI Ink 前端, React/Ink + JSON-RPC over stdio/WS |
| `ui-tui/src/gatewayClient.ts` | — | TUI Gateway 客户端, spawn + JSON-RPC |
| `ui-tui/src/app.tsx` | 25 | TUI App 组件, GatewayProvider + AppLayout |
| `ui-tui/src/entry.tsx` | — | TUI 入口, GatewayClient 启动 + 内存监控 |
| `ui-tui/src/app/` | ~50 模块 | TUI 应用层: hooks, stores, 组件 |
| `ui-tui/src/protocol/` | 2 文件 | TUI 协议层: interpolation, paste |
| `web/src/` | 69 TS/TSX | Web Dashboard SPA, Vite+React |
| `web/src/pages/` | 11 页面 | Analytics, Chat, Config, Cron, Docs, Env, Logs, Models, Plugins, Profiles, Sessions, Skills |

---

## 一句话总结

Hermes 提供四种前端界面 (CLI REPL, TUI Ink, Web Dashboard, ACP Editor), 全部共享同一个 `AIAgent` 引擎核心, 通过不同传输层 (终端 ANSI, JSON-RPC over stdio/WS, HTTP REST/WebSocket, ACP 协议) 向用户暴露交互能力。

---

## Architecture Overview

```
                    ┌─────────────────────┐
                    │     AIAgent Core    │
                    │ (run_agent.py)      │
                    │  - ContextEngine    │
                    │  - MemoryManager    │
                    │  - Tool Registry    │
                    │  - PromptBuilder    │
                    └─────────┬───────────┘
                              │
           ┌──────────────────┼───────────────────┐
           │                  │                    │
    ┌──────▼───────┐  ┌──────▼───────┐  ┌────────▼────────┐  ┌──────────────▼──────────────┐
    │  CLI REPL    │  │  TUI Ink     │  │  Web Dashboard  │  │      ACP Editor            │
    │  cli.py      │  │  ui-tui/src  │  │  web_server.py  │  │  acp_adapter/server.py     │
    │              │  │              │  │  + web/src SPA  │  │                            │
    │ prompt_      │  │  @hermes/ink │  │  FastAPI +      │  │  acp.Agent subclass        │
    │ toolkit      │  │  React/Ink   │  │  Vite/React     │  │  JSON-RPC over stdio       │
    │              │  │              │  │                 │  │                            │
    │ ANSI out     │  │  JSON-RPC    │  │  REST + WS      │  │  VS Code / Zed / JetBrains │
    │ stdin        │  │  over stdio  │  │  over HTTP      │  │  via ACP protocol           │
    └──────────────┘  │  or WS       │  └────────────────┘  └─────────────────────────────┘
                      └──────────────┘
```

---

## TL;DR

Hermes 的四种前端界面共享同一个 `AIAgent` 引擎核心, 区别在于传输层和交互范式。CLI REPL (`cli.py`, 12,508 行) 使用 `prompt_toolkit` 构建固定输入区 TUI, 支持 ASCII 品牌图案、slash 命令分发、流式渲染和皮肤系统。TUI Ink (`ui-tui/src/`, ~165 TS 文件) 基于 React/Ink 渲染终端 UI, 通过 `GatewayClient` spawn 子进程并以 JSON-RPC over stdio/WebSocket 通信。Web Dashboard (`web_server.py`, 4,063 行 + `web/src/`, 69 TS/TSX) 是 FastAPI 后端 + Vite/React SPA, 提供 50+ REST 端点管理配置、环境变量、OAuth 认证和会话搜索。ACP Editor (`acp_adapter/server.py`, 1,428 行) 实现 ACP 协议, 通过 `acp.Agent` 子类暴露 JSON-RPC over stdio 接口, 支持 VS Code/Zed/JetBrains 等编辑器嵌入。

---

## 1. CLI REPL — 终端交互的终极形态

### 1.1 整体架构

`HermesCLI` 类 (cli.py:2114) 是整个 CLI 的核心, 它将 `prompt_toolkit` 的固定输入区 TUI 与 `AIAgent` 引擎桥接。初始化参数涵盖模型、toolsets、provider、流式模式等数十个配置维度。

源码位置: cli.py:2114-2233

```python
class HermesCLI:
    """Interactive CLI for the Hermes Agent."""

    def __init__(
        self,
        model: str = None,
        toolsets: List[str] = None,
        provider: str = None,
        api_key: str = None,
        base_url: str = None,
        max_turns: int = None,
        verbose: bool = False,
        compact: bool = False,
        resume: str = None,
        checkpoints: bool = False,
        pass_session_id: bool = False,
        ignore_rules: bool = False,
    ):
        # Initialize Rich console
        self.console = Console()
        self.config = CLI_CONFIG
        self.compact = compact if compact is not None else CLI_CONFIG["display"].get("compact", False)
        # busy_input_mode: "interrupt" | "queue" | "steer"
        _bim = str(CLI_CONFIG["display"].get("busy_input_mode", "interrupt")).strip().lower()
        if _bim == "queue":
            self.busy_input_mode = "queue"
        elif _bim == "steer":
            self.busy_input_mode = "steer"
        else:
            self.busy_input_mode = "interrupt"
```

### 1.2 ASCII 品牌与皮肤系统

CLI 启动时渲染多层 ASCII art — 金色渐变 HERMES_AGENT_LOGO (全宽版) 和 Caduceus 束杖图案 (紧凑版), 配色从 `#FFD700` (金) 到 `#CD7F32` (古铜) 三级渐变。

源码位置: cli.py:1912-1935

```python
HERMES_AGENT_LOGO = """[bold #FFD700]██╗  ██╗███████╗██████╗ ███╗   ███╗███████╗███████╗       █████╗  ██████╗ ███████╗███╗   ██╗████████╗[/]
[bold #FFD700]██║  ██║██╔════╝██╔══██╗████╗ ████║██╔════╝██╔════╝      ██╔══██╗██╔════╝ ██╔════╝████╗  ██║╚══██╔══╝[/]
[#FFBF00]███████║█████╗  ██████╔╝██╔████╔██║█████╗  ███████╗█████╗███████║██║  ███╗█████╗  ██╔██╗ ██║   ██║[/]
...
[#CD7F32]╚═╝  ╚═╝╚══════╝╚═╝  ╚═╝╚═╝     ╚═╝╚══════╝╚══════╝      ╚═╝  ╚═╝ ╚═════╝ ╚══════╝╚═╝  ╚═══╝   ╚═╝[/]"""

HERMES_CADUCEUS = """[#CD7F32]⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣀⡀⠀⣀⣀⠀⢀⣀⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀[/]
[#CD7F32]⠀⠀⠀⠀⠀⠀⢀⣠⣴⣾⣿⣿⣇⠸⣿⣿⠇⣸⣿⣿⣷⣦⣄⡀⠀⠀⠀⠀⠀⠀[/]
[#FFBF00]⠀⢀⣠⣴⣶⠿⠋⣩⡿⣿⡿⠻⣿⡇⢠⡄⢸⣿⠟⢿⣿⢿⣍⠙⠿⣶⣦⣄⡀⠀[/]
...
```

皮肤系统通过 `_SkinAwareAnsi` (cli.py:1167) 实现, 从 `skin_engine` 获取活动皮肤, 动态替换品牌边框、标题等颜色:

源码位置: cli.py:1155-1206

```python
class _SkinAwareAnsi:
    def __init__(self, skin_key: str, fallback_hex: str = "#FFD700", *, bold: bool = False):
        ...
    def __str__(self) -> str:
        # 从 skin_engine 动态获取颜色
```

### 1.3 Slash 命令分发

CLI 通过 `hermes_cli.commands.resolve_command` 注册和解析 slash 命令。命令分发在 `_handle_command` 方法中, 先用 `resolve_command` 将别名映射为 canonical 名称, 再走 if/elif 链路由到具体 handler:

源码位置: cli.py:6580-6690

```python
from hermes_cli.commands import resolve_command as _resolve_cmd
_base_word = cmd_lower.split()[0].lstrip("/")
_cmd_def = _resolve_cmd(_base_word)
canonical = _cmd_def.name if _cmd_def else _base_word

if canonical in ("quit", "exit"):
    return False
elif canonical == "help":
    self.show_help()
elif canonical == "profile":
    self._handle_profile_command()
elif canonical == "tools":
    self._handle_tools_command(cmd_original)
elif canonical == "toolsets":
    self.show_toolsets()
elif canonical == "config":
    self.show_config()
elif canonical == "redraw":
    self._force_full_redraw()
elif canonical == "clear":
    self.new_session(silent=True)
    ...
```

插件命令通过 `_get_plugin_cmd_handler_names()` (cli.py:2018) 扩展, 支持动态注册。

### 1.4 Reasoning 标签剥离

CLI 在渲染前剥离所有 reasoning/thinking 标签, 覆盖 `<think>`, `<thinking>`, `<reasoning>`, `<REASONING_SCRATCHPAD>`, `<thought>` (Gemma 4) 以及 leaked tool-call XML:

源码位置: cli.py:96-174

```python
_REASONING_TAGS = ("REASONING_SCRATCHPAD", "think", "thinking", "reasoning", "thought")

def _strip_reasoning_tags(text: str) -> str:
    """Remove reasoning/thinking blocks from displayed text."""
    cleaned = text
    for tag in _REASONING_TAGS:
        # Closed pair — case-insensitive
        cleaned = re.sub(rf"<{tag}>.*?</{tag}>\s*", "", cleaned, flags=re.DOTALL | re.IGNORECASE)
        # Unterminated open tag — strip from tag to end of text
        cleaned = re.sub(rf"<{tag}>.*$", "", cleaned, flags=re.DOTALL | re.IGNORECASE)
        # Stray orphan close tag
        cleaned = re.sub(rf"</{tag}>\s*", "", cleaned, flags=re.IGNORECASE)
    # Tool-call XML blocks
    for tc_tag in ("tool_call", "tool_calls", "tool_result", "function_call", "function_calls"):
        cleaned = re.sub(rf"<{tc_tag}\b[^>]*>.*?</{tc_tag}>\s*", "", cleaned, flags=re.DOTALL | re.IGNORECASE)
    ...
```

### 1.5 ChatConsole — Rich 到 prompt_toolkit 的桥接

`ChatConsole` (cli.py:1870) 是一个 Rich Console 适配器, 将 Rich 的 ANSI 输出路由到 `_cprint`, 在 prompt_toolkit 的 patch_stdout 上下文中正确渲染颜色和 markup:

源码位置: cli.py:1870-1910

```python
class ChatConsole:
    """Rich Console adapter for prompt_toolkit's patch_stdout context."""

    def __init__(self):
        from io import StringIO
        self._buffer = StringIO()
        self._inner = Console(
            file=self._buffer,
            force_terminal=True,
            color_system="truecolor",
            highlight=False,
        )

    def print(self, *args, **kwargs):
        self._buffer.seek(0)
        self._buffer.truncate()
        self._inner.width = shutil.get_terminal_size((80, 24)).columns
        self._inner.print(*args, **kwargs)
        output = self._buffer.getvalue()
        for line in output.rstrip("\n").split("\n"):
            _cprint(line)

    @contextmanager
    def status(self, *_args, **_kwargs):
        """No-op Rich-compatible status context."""
        yield self
```

---

## 2. TUI Ink — 终端内的 React UI

### 2.1 入口与 GatewayClient

TUI 入口 (`entry.tsx`) 以 `--max-old-space-size=8192` 启动 Node 进程, 创建 `GatewayClient` 实例, spawn Hermes 子进程并通过 stdio JSON-RPC 通信:

源码位置: ui-tui/src/entry.tsx

```typescript
// node --max-old-space-size=8192 --expose-gc
import './lib/forceTruecolor.js'
import { GatewayClient } from './gatewayClient.js'
import { setupGracefulExit } from './lib/gracefulExit.js'

if (!process.stdin.isTTY) {
  console.log('hermes-tui: no TTY')
  process.exit(0)
}
resetTerminalModes()
const gw = new GatewayClient()
gw.start()
```

`GatewayClient` (gatewayClient.ts) 负责发现 Python 解释器 (venv, .venv, system), spawn 子进程, 建立 JSON-RPC 连接, 并提供请求超时管理 (120s 默认):

```typescript
const resolvePython = (root: string) => {
  const configured = process.env.HERMES_PYTHON?.trim() || process.env.PYTHON?.trim()
  if (configured) return configured
  const venv = process.env.VIRTUAL_ENV?.trim()
  const hit = [
    venv && resolve(venv, 'bin/python'),
    venv && resolve(venv, 'Scripts/python.exe'),
    resolve(root, '.venv/bin/python'),
    ...
  ].find(p => p && existsSync(p))
  return hit || (process.platform === 'win32' ? 'python' : 'python3')
}
```

### 2.2 App 组件架构

`App.tsx` 使用 nanostores (`@nanostores/react`) 管理全局状态, 通过 `useMainApp` hook 组合各子模块:

源码位置: ui-tui/src/app.tsx

```typescript
import { useStore } from '@nanostores/react'
import { GatewayProvider } from './app/gatewayContext.js'
import { $uiState } from './app/uiStore.js'
import { useMainApp } from './app/useMainApp.js'
import { AppLayout } from './components/appLayout.js'

export function App({ gw }: { gw: GatewayClient }) {
  const { appActions, appComposer, appProgress, appStatus, appTranscript, gateway } = useMainApp(gw)
  const { mouseTracking } = useStore($uiState)
  return (
    <GatewayProvider value={gateway}>
      <AppLayout
        actions={appActions}
        composer={appComposer}
        mouseTracking={mouseTracking}
        progress={appProgress}
        status={appStatus}
        transcript={appTranscript}
      />
    </GatewayProvider>
  )
}
```

### 2.3 TUI 应用层模块

TUI 的 `app/` 目录包含大量 hooks 和 stores, 模块化地管理不同关注点:

| 模块 | 职责 |
|------|------|
| `gatewayContext.tsx` | React Context 提供 GatewayClient |
| `uiStore.ts` | nanostore 全局 UI 状态 |
| `turnStore.ts` | 轮次状态管理 |
| `turnController.ts` | 轮次控制器 |
| `useMainApp.ts` | 主应用 hook, 组合所有子模块 |
| `useSessionLifecycle.ts` | 会话生命周期管理 |
| `useComposerState.ts` | 输入框状态 |
| `useSubmission.ts` | 提交处理 |
| `useInputHandlers.ts` | 输入事件处理 |
| `useConfigSync.ts` | 配置同步 |
| `useLongRunToolCharms.ts` | 长运行工具动画 |
| `createGatewayEventHandler.ts` | Gateway 事件处理器工厂 |
| `createSlashHandler.ts` | Slash 命令处理器工厂 |
| `delegationStore.ts` | 子代理委派状态 |
| `overlayStore.ts` | Overlay 状态 |
| `inputSelectionStore.ts` | 输入选择状态 |
| `spawnHistoryStore.ts` | Spawn 历史追踪 |
| `setupHandoff.ts` | 会话交接设置 |

### 2.4 TUI 组件层

TUI 的 `components/` 目录包含纯展示/交互组件:

| 组件 | 职责 |
|------|------|
| `appLayout.tsx` | 主布局框架 |
| `appChrome.tsx` | 应用 chrome (边框/标题) |
| `appOverlays.tsx` | Overlay 容器 |
| `branding.tsx` | 品牌标识渲染 |
| `streamingAssistant.tsx` | 流式助手消息渲染 |
| `streamingMarkdown.tsx` | 流式 Markdown 渲染 |
| `textInput.tsx` | 文本输入组件 |
| `modelPicker.tsx` | 模型选择器 |
| `sessionPicker.tsx` | 会话选择器 |
| `todoPanel.tsx` | Todo 面板 |
| `skillsHub.tsx` | Skills Hub 面板 |
| `thinking.tsx` | 思考/推理展示 |
| `markdown.tsx` | Markdown 渲染 |
| `maskedPrompt.tsx` | 隐码提示展示 |
| `helpHint.tsx` | 帮助提示 |
| `fpsOverlay.tsx` | FPS 性能监控 |
| `overlayControls.tsx` | Overlay 控件 |
| `prompts.tsx` | 提示展示 |
| `queuedMessages.tsx` | 队列消息展示 |
| `themed.tsx` | 主题化组件 |

---

## 3. Web Dashboard — 浏览器内的管理控制台

### 3.1 FastAPI 后端

`web_server.py` 基于 FastAPI 构建, 启动时生成一次性的 session token 保护敏感端点, 并限制 CORS 到 localhost:

源码位置: web_server.py:67-100

```python
app = FastAPI(title="Hermes Agent", version=__version__)
_SESSION_TOKEN = secrets.token_urlsafe(32)
_SESSION_HEADER_NAME = "X-Hermes-Session-Token"

app.add_middleware(
    CORSMiddleware,
    allow_origin_regex=r"^https?://(localhost|127\.0\.0\.1)(:\d+)?$",
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 3.2 认证中间件

两层中间件保护所有 `/api/` 端点 — host header 检查阻止 DNS 重绑定攻击, session token 检查阻止跨站请求:

源码位置: web_server.py:113-225

```python
def _has_valid_session_token(request: Request) -> bool:
    """Check if the request has a valid session token."""
    ...

def _require_token(request: Request) -> None:
    """Raise 403 if the session token is missing or invalid."""
    ...

@app.middleware("http")
async def host_header_middleware(request: Request, call_next):
    """Block requests with mismatched Host headers (DNS rebinding defense)."""
    ...

@app.middleware("http")
async def auth_middleware(request: Request, call_next):
    """Gate /api/ endpoints with session token unless whitelisted."""
    ...
```

### 3.3 REST 端点分类

Web Dashboard 提供 50+ REST 端点, 涵盖多个管理域:

| 端点类别 | 关键路由 | 行数范围 | 职责 |
|----------|----------|----------|------|
| 状态/系统 | `/api/status`, `/api/gateway/restart`, `/api/hermes/update` | 523-730 | 系统状态查询、gateway 重启、版本更新 |
| 会话管理 | `/api/sessions`, `/api/sessions/search` | 761-824 | 会话列表和全文搜索 |
| 配置管理 | `/api/config`, `/api/config/defaults`, `/api/config/schema`, `/api/config` (PUT) | 848-1206 | 配置读写, schema 描述 |
| 环境变量 | `/api/env` (GET/PUT/DELETE), `/api/env/reveal` | 1216-1300 | .env 管理, 密钥安全揭示 |
| 模型管理 | `/api/model/info`, `/api/model/options`, `/api/model/set` | 875-1065 | 模型信息、选项查询、模型切换 |
| OAuth 认证 | `/api/providers/oauth`, `/api/providers/oauth/{id}/start/submit/poll` | 1523-2143 | OAuth 流程管理 (Anthropic PKCE, device code) |
| Actions | `/api/actions/{name}/status` | 732-760 | 长运行操作状态追踪 |

### 3.4 OAuth 流程

Web Dashboard 实现了完整的 OAuth 登录流程, 支持 Anthropic PKCE 和 device code 两种模式:

源码位置: web_server.py:1729-1963

```python
def _start_anthropic_pkce() -> Dict[str, Any]:
    """Start Anthropic OAuth with PKCE flow."""
    ...

def _submit_anthropic_pkce(session_id: str, code_input: str) -> Dict[str, Any]:
    """Submit the authorization code for the PKCE flow."""
    ...

def _start_device_code_flow(provider_id: str) -> Dict[str, Any]:
    """Start a device code flow for Nous/other providers."""
    ...

def _nous_poller(session_id: str) -> None:
    """Background thread polling for device code completion."""
    ...
```

### 3.5 Web SPA 前端

Web 前端 (`web/src/`) 是 Vite + React SPA, 包含 11 个页面组件和对应的 hooks:

| 页面 | 文件 | 职责 |
|------|------|------|
| Analytics | `AnalyticsPage.tsx` | 使用分析和统计 |
| Chat | `ChatPage.tsx` | 内嵌聊天 (可选) |
| Config | `ConfigPage.tsx` | 配置管理界面 |
| Cron | `CronPage.tsx` | 定时任务管理 |
| Docs | `DocsPage.tsx` | 文档查看 |
| Env | `EnvPage.tsx` | 环境变量管理 |
| Logs | `LogsPage.tsx` | 日志查看 |
| Models | `ModelsPage.tsx` | 模型选择和管理 |
| Plugins | `PluginsPage.tsx` | 插件管理 |
| Profiles | `ProfilesPage.tsx` | Profile 切换 |
| Sessions | `SessionsPage.tsx` | 会话搜索和浏览 |
| Skills | `SkillsPage.tsx` | Skills 管理 |

---

## 4. ACP Editor — 编辑器嵌入协议

### 4.1 HermesACPAgent 类

`HermesACPAgent` (acp_adapter/server.py:160) 继承 `acp.Agent`, 实现 ACP 协议的完整生命周期 — initialize, authenticate, new_session, load_session, resume_session, prompt, cancel, fork_session, list_sessions:

源码位置: acp_adapter/server.py:160-217

```python
class HermesACPAgent(acp.Agent):
    """ACP agent implementation for Hermes."""

    def __init__(self, session_manager: SessionManager | None = None):
        self.session_manager = session_manager or SessionManager()
        ...
```

### 4.2 Initialize — 能力声明

ACP 服务器在 `initialize()` 方法中声明完整能力集 — 会话管理、fork、resume、streaming、模型切换、命令更新等:

源码位置: acp_adapter/server.py:452-494

```python
async def initialize(self, conn: acp.Client) -> InitializeResponse:
    """Declare capabilities and server info."""
    return InitializeResponse(
        capabilities=AgentCapabilities(
            sessions=SessionCapabilities(
                new=NewSessionCapabilities(supported=True),
                fork=SessionForkCapabilities(supported=True, ...),
                resume=SessionResumeCapabilities(supported=True),
                list=SessionListCapabilities(supported=True),
            ),
            prompt=PromptCapabilities(supported=True, streaming=True, ...),
            ...
        ),
        ...
    )
```

### 4.3 Prompt — 核心交互循环

`prompt()` 方法是 ACP 的核心交互入口。它处理 slash 命令拦截、/steer 重写 (idle 和 interrupted 两种场景)、并发排队、streaming 事件回传等复杂流程:

源码位置: acp_adapter/server.py:786-933

```python
async def prompt(self, prompt, session_id, **kwargs) -> PromptResponse:
    """Run Hermes on the user's prompt and stream events back to the editor."""
    state = self.session_manager.get_session(session_id)

    # /steer rewrite for idle sessions
    if isinstance(user_content, str) and user_text.startswith("/steer"):
        steer_text = user_text.split(maxsplit=1)[1].strip() ...
        if interrupted_prompt:
            user_text = f"{interrupted_prompt}\n\nUser correction/guidance after interrupt: {steer_text}"
        elif rewrite_idle:
            user_text = steer_text

    # Slash command interception
    if isinstance(user_content, str) and user_text.startswith("/"):
        response_text = self._handle_slash_command(user_text, state)
        if response_text is not None:
            ...
            return PromptResponse(stop_reason="end_turn")

    # Queue concurrent prompts while session is running
    with state.runtime_lock:
        if state.is_running:
            state.queued_prompts.append(queued_text)
            ...
            return PromptResponse(stop_reason="end_turn")
        state.is_running = True
```

### 4.4 会话管理

ACP 的 `SessionManager` (session.py) 映射 ACP session ID 到 Hermes `AIAgent` 实例, 持久化到 `SessionDB` (`~/.hermes/state.db`) 以支持编辑器重启后恢复会话。它还处理 WSL 路径翻译, 将 Windows drive paths 转换为 `/mnt/<drive>/...` 格式:

源码位置: acp_adapter/session.py:29-80

```python
def _win_path_to_wsl(path: str) -> str | None:
    """Convert a Windows drive path to its WSL /mnt/<drive>/... equivalent."""
    match = re.match(r"^([A-Za-z]):[\\/](.*)$", path)
    ...

def _translate_acp_cwd(cwd: str) -> str:
    """Translate Windows ACP cwd values when Hermes itself is running in WSL."""
    if not is_wsl():
        return cwd
    translated = _win_path_to_wsl(str(cwd))
    return translated if translated is not None else cwd
```

### 4.5 入口与日志过滤

ACP 入口 (`entry.py`) 将所有日志路由到 stderr (stdout 留给 JSON-RPC), 并过滤 ACP 协议的 benign probe 方法 (ping/health), 防止周期性探活产生虚假 traceback:

源码位置: acp_adapter/entry.py:14-60

```python
_BENIGN_PROBE_METHODS = frozenset({"ping", "health", "healthcheck"})

class _BenignProbeMethodFilter(logging.Filter):
    """Suppress acp 'Background task failed' tracebacks from liveness probes."""
    def filter(self, record: logging.LogRecord) -> bool:
        if record.getMessage() != "Background task failed":
            return True
        ...
        method = data.get("method") if isinstance(data, dict) else None
        return method not in _BENIGN_PROBE_METHODS
```

---

## Comparison Tables

### 表 1: 四种前端传输层对比

| 维度 | CLI REPL | TUI Ink | Web Dashboard | ACP Editor |
|------|----------|---------|---------------|------------|
| 传输协议 | ANSI over terminal | JSON-RPC over stdio/WS | HTTP REST + WebSocket | JSON-RPC over stdio |
| 渲染引擎 | prompt_toolkit + Rich ANSI | React/Ink (@hermes/ink fork) | Vite/React SPA | 编辑器原生渲染 |
| 进程模型 | 直接运行 | spawn 子进程 + IPC | uvicorn 服务进程 | spawn by editor, stdio pipe |
| 输入处理 | prompt_toolkit TextArea | Ink TextInput + hooks | HTML form + fetch | ACP prompt 方法 |
| 流式支持 | _stream_buf + ANSI | JSON-RPC delta events | WebSocket events | ACP AgentMessageChunk |
| 认证方式 | 无 (本地终端) | 无 (子进程 IPC) | Session token + CORS | ACP authenticate |
| 配置管理 | config.yaml + .env | 透传到子进程 | REST API读写 | ACP set_session_config |
| 跨平台 | 全终端 | Node + terminal | 浏览器 | VS Code/Zed/JetBrains |

### 表 2: Slash 命令支持对比

| 命令 | CLI REPL | TUI Ink | Web Dashboard | ACP Editor |
|------|----------|---------|---------------|------------|
| /help | show_help() | createSlashHandler | Help page | _cmd_help |
| /clear | new_session + clear screen | session lifecycle | N/A | N/A |
| /reset | new_session(silent=True) | session lifecycle | N/A | N/A |
| /model | 模型切换 | modelPicker | /api/model/set | _cmd_model |
| /tools | show_toolsets() | tools display | Config page | _cmd_tools |
| /compress | 手动压缩 | 透传到 gateway | N/A | N/A |
| /steer | busy_input_mode="steer" | 透传到 gateway | N/A | idle rewrite |
| /config | show_config() | config sync | Config page | set_session_config |
| /quit | return False | gracefulExit | N/A | N/A |
| /redraw | _force_full_redraw | N/A | N/A | N/A |
| 插件命令 | _get_plugin_cmd_handler_names | 透传 | 透传 | 透传 |

### 表 3: 安全机制对比

| 安全措施 | CLI REPL | TUI Ink | Web Dashboard | ACP Editor |
|----------|----------|---------|---------------|------------|
| 认证 | 无 (本地用户) | 无 (子进程) | X-Hermes-Session-Token | ACP authenticate |
| CORS | N/A | N/A | localhost-only regex | N/A |
| DNS 重绑定防护 | N/A | N/A | host_header_middleware | N/A |
| 密钥揭示限制 | N/A | N/A | rate limiter (5/30s) | permissions callback |
| 环境变量保护 | _env_loader | 透传 | /api/env/reveal + rate limit | N/A |
| 理由标签剥离 | _strip_reasoning_tags | streaming filter | gateway filter | StreamingThinkScrubber |
| 工具调用 XML 清理 | cli.py regex | 透传 | N/A | 透传 |

### 表 4: 状态管理模式对比

| 维度 | CLI REPL | TUI Ink | Web Dashboard | ACP Editor |
|------|----------|---------|---------------|------------|
| 全局状态 | HermesCLI instance | nanostores ($uiState etc.) | React state + FastAPI | SessionManager + SessionState |
| 轮次管理 | _busy_command / _stream_buf | turnStore + turnController | /api/sessions | state.is_running + queued_prompts |
| 会话持久化 | SessionDB (resume) | 透传 | SessionDB search | SessionDB load/resume |
| 上下文压缩 | ContextCompressor | 透传到 gateway | 透传 | 透传到 AIAgent |
| 内存管理 | MemoryManager | 透传 | 透传 | 透传 |

---

## Summary Table

| 组件名 | 行数 | 职责 |
|--------|------|------|
| `cli.py` | 12,508 | CLI REPL 主类, prompt_toolkit TUI, ASCII branding, slash commands, 流式渲染, 皮肤系统 |
| `hermes_cli/web_server.py` | 4,063 | FastAPI 后端, 50+ REST 端点, session token auth, CORS, OAuth flows |
| `acp_adapter/server.py` | 1,428 | ACP Agent 子类, JSON-RPC 协议, slash command intercept, streaming |
| `acp_adapter/session.py` | ~80 | ACP 会话管理器, session-AIAgent 映射, WSL path translation |
| `acp_adapter/entry.py` | ~80 | ACP 入口, 日志过滤, environment loading |
| `acp_adapter/tools.py` | — | ACP tool start/complete 事件构建 |
| `acp_adapter/events.py` | — | ACP message/step/thinking/tool_progress 回调工厂 |
| `acp_adapter/auth.py` | — | ACP 认证 provider detection |
| `acp_adapter/permissions.py` | — | ACP approval callback 构建 |
| `ui-tui/src/entry.tsx` | ~40 | TUI 入口, GatewayClient spawn, 内存监控 |
| `ui-tui/src/app.tsx` | 25 | TUI App 组件, GatewayProvider + AppLayout |
| `ui-tui/src/gatewayClient.ts` | ~100 | TUI Gateway 客户端, JSON-RPC, Python 发现 |
| `ui-tui/src/app/` | ~50 模块 | TUI hooks/stores: turnController, uiStore, useMainApp 等 |
| `ui-tui/src/components/` | ~20 组件 | TUI 展示组件: streamingAssistant, textInput, modelPicker 等 |
| `web/src/pages/` | 11 页面 | Web SPA 页面: Config, Sessions, Models, Env, Cron 等 |
| `web/src/hooks/` | 3 hooks | Web hooks: useConfirmDelete, useSidebarStatus, useToast |

---

[← 06-模型路由与推理](/zh-CN/chapters/06-gateway-system) | [→ 08-状态与持久化](/zh-CN/chapters/08-state-persistence)