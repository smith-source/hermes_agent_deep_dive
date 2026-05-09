# 07 — Frontend Interaction Interfaces: Four Frontends Sharing the Same Engine Core

[← 06-Model Routing & Inference](/en/chapters/06-gateway-system) | [→ 08-State & Persistence](/en/chapters/08-state-persistence)

---

## Source Files

| File | Lines | Description |
|------|------|------|
| `cli.py` | 12,508 | CLI REPL main entry, prompt_toolkit + ASCII branding + slash commands |
| `hermes_cli/web_server.py` | 4,063 | Web Dashboard FastAPI backend, 50+ REST/WebSocket endpoints |
| `acp_adapter/server.py` | 1,428 | ACP Agent server, JSON-RPC over stdio |
| `acp_adapter/session.py` | — | ACP session manager, maps ACP session to AIAgent instance |
| `acp_adapter/entry.py` | — | ACP entry point, environment loading + log configuration |
| `acp_adapter/tools.py` | — | ACP tool event construction |
| `acp_adapter/events.py` | — | ACP event callback factory |
| `acp_adapter/auth.py` | — | ACP authentication detection |
| `acp_adapter/permissions.py` | — | ACP approval callback |
| `ui-tui/src/` | ~165 TS/TSX | TUI Ink frontend, React/Ink + JSON-RPC over stdio/WS |
| `ui-tui/src/gatewayClient.ts` | — | TUI Gateway client, spawn + JSON-RPC |
| `ui-tui/src/app.tsx` | 25 | TUI App component, GatewayProvider + AppLayout |
| `ui-tui/src/entry.tsx` | — | TUI entry point, GatewayClient startup + memory monitoring |
| `ui-tui/src/app/` | ~50 modules | TUI application layer: hooks, stores, components |
| `ui-tui/src/protocol/` | 2 files | TUI protocol layer: interpolation, paste |
| `web/src/` | 69 TS/TSX | Web Dashboard SPA, Vite+React |
| `web/src/pages/` | 11 pages | Analytics, Chat, Config, Cron, Docs, Env, Logs, Models, Plugins, Profiles, Sessions, Skills |

---

## One-Line Summary

Hermes provides four frontend interfaces (CLI REPL, TUI Ink, Web Dashboard, ACP Editor), all sharing the same `AIAgent` engine core, exposing interaction capabilities to users through different transport layers (terminal ANSI, JSON-RPC over stdio/WS, HTTP REST/WebSocket, ACP protocol).

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

Hermes' four frontend interfaces share the same `AIAgent` engine core, differing in transport layer and interaction paradigm. CLI REPL (`cli.py`, 12,508 lines) uses `prompt_toolkit` to build a fixed-input-area TUI, supporting ASCII brand art, slash command dispatch, streaming rendering, and a skin system. TUI Ink (`ui-tui/src/`, ~165 TS files) renders terminal UI based on React/Ink, communicating via `GatewayClient` which spawns a subprocess and uses JSON-RPC over stdio/WebSocket. Web Dashboard (`web_server.py`, 4,063 lines + `web/src/`, 69 TS/TSX) is a FastAPI backend + Vite/React SPA, providing 50+ REST endpoints managing configuration, environment variables, OAuth authentication, and session search. ACP Editor (`acp_adapter/server.py`, 1,428 lines) implements the ACP protocol, exposing a JSON-RPC over stdio interface through an `acp.Agent` subclass, supporting embedding in editors like VS Code/Zed/JetBrains.

---

## 1. CLI REPL — The Ultimate Form of Terminal Interaction

### 1.1 Overall Architecture

The `HermesCLI` class (cli.py:2114) is the core of the entire CLI, bridging `prompt_toolkit`'s fixed-input-area TUI with the `AIAgent` engine. Initialization parameters cover dozens of configuration dimensions including model, toolsets, provider, streaming mode, etc.

Source location: cli.py:2114-2233

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

### 1.2 ASCII Branding and Skin System

On startup, the CLI renders multi-layer ASCII art — the golden gradient HERMES_AGENT_LOGO (full-width version) and the Caduceus staff pattern (compact version), with a three-level gradient from `#FFD700` (gold) to `#CD7F32` (bronze).

Source location: cli.py:1912-1935

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

The skin system is implemented via `_SkinAwareAnsi` (cli.py:1167), which retrieves the active skin from `skin_engine` and dynamically replaces colors for brand borders, headings, etc:

Source location: cli.py:1155-1206

```python
class _SkinAwareAnsi:
    def __init__(self, skin_key: str, fallback_hex: str = "#FFD700", *, bold: bool = False):
        ...
    def __str__(self) -> str:
        # Dynamically retrieve color from skin_engine
```

### 1.3 Slash Command Dispatch

The CLI registers and resolves slash commands via `hermes_cli.commands.resolve_command`. Command dispatch occurs in the `_handle_command` method, which first maps aliases to canonical names using `resolve_command`, then routes to specific handlers through an if/elif chain:

Source location: cli.py:6580-6690

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

Plugin commands are extended via `_get_plugin_cmd_handler_names()` (cli.py:2018), supporting dynamic registration.

### 1.4 Reasoning Tag Stripping

The CLI strips all reasoning/thinking tags before rendering, covering `<thinking>`, `<reasoning>`, `<REASONING_SCRATCHPAD>`, `<thought>` (Gemma 4) as well as leaked tool-call XML:

Source location: cli.py:96-174

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

### 1.5 ChatConsole — Bridging Rich to prompt_toolkit

`ChatConsole` (cli.py:1870) is a Rich Console adapter that routes Rich's ANSI output to `_cprint`, rendering colors and markup correctly within prompt_toolkit's patch_stdout context:

Source location: cli.py:1870-1910

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

## 2. TUI Ink — React UI Within the Terminal

### 2.1 Entry Point and GatewayClient

The TUI entry point (`entry.tsx`) starts a Node process with `--max-old-space-size=8192`, creates a `GatewayClient` instance, spawns a Hermes subprocess, and communicates via stdio JSON-RPC:

Source location: ui-tui/src/entry.tsx

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

`GatewayClient` (gatewayClient.ts) handles Python interpreter discovery (venv, .venv, system), subprocess spawning, JSON-RPC connection establishment, and request timeout management (120s default):

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

### 2.2 App Component Architecture

`App.tsx` uses nanostores (`@nanostores/react`) for global state management, composing sub-modules through the `useMainApp` hook:

Source location: ui-tui/src/app.tsx

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

### 2.3 TUI Application Layer Modules

The TUI's `app/` directory contains numerous hooks and stores, modularly managing different concerns:

| Module | Responsibility |
|------|------|
| `gatewayContext.tsx` | React Context providing GatewayClient |
| `uiStore.ts` | nanostore global UI state |
| `turnStore.ts` | Turn state management |
| `turnController.ts` | Turn controller |
| `useMainApp.ts` | Main app hook, composing all sub-modules |
| `useSessionLifecycle.ts` | Session lifecycle management |
| `useComposerState.ts` | Input box state |
| `useSubmission.ts` | Submission handling |
| `useInputHandlers.ts` | Input event handling |
| `useConfigSync.ts` | Configuration synchronization |
| `useLongRunToolCharms.ts` | Long-running tool animations |
| `createGatewayEventHandler.ts` | Gateway event handler factory |
| `createSlashHandler.ts` | Slash command handler factory |
| `delegationStore.ts` | Sub-agent delegation state |
| `overlayStore.ts` | Overlay state |
| `inputSelectionStore.ts` | Input selection state |
| `spawnHistoryStore.ts` | Spawn history tracking |
| `setupHandoff.ts` | Session handoff setup |

### 2.4 TUI Component Layer

The TUI's `components/` directory contains pure display/interaction components:

| Component | Responsibility |
|------|------|
| `appLayout.tsx` | Main layout framework |
| `appChrome.tsx` | App chrome (borders/heading) |
| `appOverlays.tsx` | Overlay container |
| `branding.tsx` | Brand identity rendering |
| `streamingAssistant.tsx` | Streaming assistant message rendering |
| `streamingMarkdown.tsx` | Streaming Markdown rendering |
| `textInput.tsx` | Text input component |
| `modelPicker.tsx` | Model picker |
| `sessionPicker.tsx` | Session picker |
| `todoPanel.tsx` | Todo panel |
| `skillsHub.tsx` | Skills Hub panel |
| `thinking.tsx` | Thinking/reasoning display |
| `markdown.tsx` | Markdown rendering |
| `maskedPrompt.tsx` | Masked prompt display |
| `helpHint.tsx` | Help hint |
| `fpsOverlay.tsx` | FPS performance monitoring |
| `overlayControls.tsx` | Overlay controls |
| `prompts.tsx` | Prompt display |
| `queuedMessages.tsx` | Queued message display |
| `themed.tsx` | Themed components |

---

## 3. Web Dashboard — Management Console in the Browser

### 3.1 FastAPI Backend

`web_server.py` is built on FastAPI, generating a one-time session token on startup to protect sensitive endpoints, and restricting CORS to localhost:

Source location: web_server.py:67-100

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

### 3.2 Authentication Middleware

Two middleware layers protect all `/api/` endpoints — host header checking prevents DNS rebinding attacks, and session token checking prevents cross-site requests:

Source location: web_server.py:113-225

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

### 3.3 REST Endpoint Categories

The Web Dashboard provides 50+ REST endpoints covering multiple management domains:

| Endpoint Category | Key Routes | Line Range | Responsibility |
|----------|----------|----------|------|
| Status/System | `/api/status`, `/api/gateway/restart`, `/api/hermes/update` | 523-730 | System status query, gateway restart, version update |
| Session Management | `/api/sessions`, `/api/sessions/search` | 761-824 | Session list and full-text search |
| Configuration Management | `/api/config`, `/api/config/defaults`, `/api/config/schema`, `/api/config` (PUT) | 848-1206 | Configuration read/write, schema description |
| Environment Variables | `/api/env` (GET/PUT/DELETE), `/api/env/reveal` | 1216-1300 | .env management, secret safe reveal |
| Model Management | `/api/model/info`, `/api/model/options`, `/api/model/set` | 875-1065 | Model info, options query, model switching |
| OAuth Authentication | `/api/providers/oauth`, `/api/providers/oauth/{id}/start/submit/poll` | 1523-2143 | OAuth flow management (Anthropic PKCE, device code) |
| Actions | `/api/actions/{name}/status` | 732-760 | Long-running operation status tracking |

### 3.4 OAuth Flow

The Web Dashboard implements a complete OAuth login flow, supporting both Anthropic PKCE and device code modes:

Source location: web_server.py:1729-1963

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

### 3.5 Web SPA Frontend

The Web frontend (`web/src/`) is a Vite + React SPA, containing 11 page components and corresponding hooks:

| Page | File | Responsibility |
|------|------|------|
| Analytics | `AnalyticsPage.tsx` | Usage analytics and statistics |
| Chat | `ChatPage.tsx` | Embedded chat (optional) |
| Config | `ConfigPage.tsx` | Configuration management interface |
| Cron | `CronPage.tsx` | Scheduled task management |
| Docs | `DocsPage.tsx` | Documentation viewer |
| Env | `EnvPage.tsx` | Environment variable management |
| Logs | `LogsPage.tsx` | Log viewer |
| Models | `ModelsPage.tsx` | Model selection and management |
| Plugins | `PluginsPage.tsx` | Plugin management |
| Profiles | `ProfilesPage.tsx` | Profile switching |
| Sessions | `SessionsPage.tsx` | Session search and browsing |
| Skills | `SkillsPage.tsx` | Skills management |

---

## 4. ACP Editor — Editor Embedding Protocol

### 4.1 HermesACPAgent Class

`HermesACPAgent` (acp_adapter/server.py:160) inherits from `acp.Agent`, implementing the full ACP protocol lifecycle — initialize, authenticate, new_session, load_session, resume_session, prompt, cancel, fork_session, list_sessions:

Source location: acp_adapter/server.py:160-217

```python
class HermesACPAgent(acp.Agent):
    """ACP agent implementation for Hermes."""

    def __init__(self, session_manager: SessionManager | None = None):
        self.session_manager = session_manager or SessionManager()
        ...
```

### 4.2 Initialize — Capability Declaration

The ACP server declares its full capability set in the `initialize()` method — session management, fork, resume, streaming, model switching, command update, etc:

Source location: acp_adapter/server.py:452-494

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

### 4.3 Prompt — Core Interaction Loop

The `prompt()` method is the ACP's core interaction entry point. It handles slash command interception, /steer rewriting (both idle and interrupted scenarios), concurrent queuing, streaming event relay, and other complex flows:

Source location: acp_adapter/server.py:786-933

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

### 4.4 Session Management

ACP's `SessionManager` (session.py) maps ACP session IDs to Hermes `AIAgent` instances, persisting to `SessionDB` (`~/.hermes/state.db`) to support session recovery after editor restarts. It also handles WSL path translation, converting Windows drive paths to `/mnt/<drive>/...` format:

Source location: acp_adapter/session.py:29-80

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

### 4.5 Entry Point and Log Filtering

The ACP entry point (`entry.py`) routes all logs to stderr (stdout is reserved for JSON-RPC), and filters ACP protocol's benign probe methods (ping/health), preventing periodic liveness checks from producing spurious tracebacks:

Source location: acp_adapter/entry.py:14-60

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

### Table 1: Four Frontend Transport Layer Comparison

| Dimension | CLI REPL | TUI Ink | Web Dashboard | ACP Editor |
|------|----------|---------|---------------|------------|
| Transport Protocol | ANSI over terminal | JSON-RPC over stdio/WS | HTTP REST + WebSocket | JSON-RPC over stdio |
| Rendering Engine | prompt_toolkit + Rich ANSI | React/Ink (@hermes/ink fork) | Vite/React SPA | Editor native rendering |
| Process Model | Direct execution | spawn subprocess + IPC | uvicorn server process | spawn by editor, stdio pipe |
| Input Handling | prompt_toolkit TextArea | Ink TextInput + hooks | HTML form + fetch | ACP prompt method |
| Streaming Support | _stream_buf + ANSI | JSON-RPC delta events | WebSocket events | ACP AgentMessageChunk |
| Authentication | None (local terminal) | None (subprocess IPC) | Session token + CORS | ACP authenticate |
| Configuration Management | config.yaml + .env | Pass-through to subprocess | REST API read/write | ACP set_session_config |
| Cross-Platform | All terminals | Node + terminal | Browser | VS Code/Zed/JetBrains |

### Table 2: Slash Command Support Comparison

| Command | CLI REPL | TUI Ink | Web Dashboard | ACP Editor |
|------|----------|---------|---------------|------------|
| /help | show_help() | createSlashHandler | Help page | _cmd_help |
| /clear | new_session + clear screen | session lifecycle | N/A | N/A |
| /reset | new_session(silent=True) | session lifecycle | N/A | N/A |
| /model | Model switching | modelPicker | /api/model/set | _cmd_model |
| /tools | show_toolsets() | tools display | Config page | _cmd_tools |
| /compress | Manual compression | Pass-through to gateway | N/A | N/A |
| /steer | busy_input_mode="steer" | Pass-through to gateway | N/A | idle rewrite |
| /config | show_config() | config sync | Config page | set_session_config |
| /quit | return False | gracefulExit | N/A | N/A |
| /redraw | _force_full_redraw | N/A | N/A | N/A |
| Plugin commands | _get_plugin_cmd_handler_names | Pass-through | Pass-through | Pass-through |

### Table 3: Security Mechanism Comparison

| Security Measure | CLI REPL | TUI Ink | Web Dashboard | ACP Editor |
|----------|----------|---------|---------------|------------|
| Authentication | None (local user) | None (subprocess) | X-Hermes-Session-Token | ACP authenticate |
| CORS | N/A | N/A | localhost-only regex | N/A |
| DNS Rebinding Defense | N/A | N/A | host_header_middleware | N/A |
| Secret Reveal Limit | N/A | N/A | rate limiter (5/30s) | permissions callback |
| Environment Variable Protection | _env_loader | Pass-through | /api/env/reveal + rate limit | N/A |
| Reasoning Tag Stripping | _strip_reasoning_tags | streaming filter | gateway filter | StreamingThinkScrubber |
| Tool Call XML Cleanup | cli.py regex | Pass-through | N/A | Pass-through |

### Table 4: State Management Pattern Comparison

| Dimension | CLI REPL | TUI Ink | Web Dashboard | ACP Editor |
|------|----------|---------|---------------|------------|
| Global State | HermesCLI instance | nanostores ($uiState etc.) | React state + FastAPI | SessionManager + SessionState |
| Turn Management | _busy_command / _stream_buf | turnStore + turnController | /api/sessions | state.is_running + queued_prompts |
| Session Persistence | SessionDB (resume) | Pass-through | SessionDB search | SessionDB load/resume |
| Context Compression | ContextCompressor | Pass-through to gateway | Pass-through | Pass-through to AIAgent |
| Memory Management | MemoryManager | Pass-through | Pass-through | Pass-through |

---

## Summary Table

| Component Name | Lines | Responsibility |
|--------|------|------|
| `cli.py` | 12,508 | CLI REPL main class, prompt_toolkit TUI, ASCII branding, slash commands, streaming rendering, skin system |
| `hermes_cli/web_server.py` | 4,063 | FastAPI backend, 50+ REST endpoints, session token auth, CORS, OAuth flows |
| `acp_adapter/server.py` | 1,428 | ACP Agent subclass, JSON-RPC protocol, slash command intercept, streaming |
| `acp_adapter/session.py` | ~80 | ACP session manager, session-AIAgent mapping, WSL path translation |
| `acp_adapter/entry.py` | ~80 | ACP entry point, log filtering, environment loading |
| `acp_adapter/tools.py` | — | ACP tool start/complete event construction |
| `acp_adapter/events.py` | — | ACP message/step/thinking/tool_progress callback factory |
| `acp_adapter/auth.py` | — | ACP authentication provider detection |
| `acp_adapter/permissions.py` | — | ACP approval callback construction |
| `ui-tui/src/entry.tsx` | ~40 | TUI entry point, GatewayClient spawn, memory monitoring |
| `ui-tui/src/app.tsx` | 25 | TUI App component, GatewayProvider + AppLayout |
| `ui-tui/src/gatewayClient.ts` | ~100 | TUI Gateway client, JSON-RPC, Python discovery |
| `ui-tui/src/app/` | ~50 modules | TUI hooks/stores: turnController, uiStore, useMainApp, etc. |
| `ui-tui/src/components/` | ~20 components | TUI display components: streamingAssistant, textInput, modelPicker, etc. |
| `web/src/pages/` | 11 pages | Web SPA pages: Config, Sessions, Models, Env, Cron, etc. |
| `web/src/hooks/` | 3 hooks | Web hooks: useConfirmDelete, useSidebarStatus, useToast |

---

[← 06-Model Routing & Inference](/en/chapters/06-gateway-system) | [→ 08-State & Persistence](/en/chapters/08-state-persistence)