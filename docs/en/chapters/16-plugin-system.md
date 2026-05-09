# 16 — Plugin System: Extensible Architecture with Dynamic Discovery and Declarative Registration


---

## Source Files

| File | Lines | Role |
|------|-------|------|
| `hermes_cli/plugins.py` | 1362 | PluginManager core + manifest parsing + loading |
| `hermes_cli/plugins_cmd.py` | 1544 | CLI commands: install/update/remove/enable/disable/list |
| `plugins/disk-cleanup/plugin.yaml` | 8 | standalone disk cleanup plugin manifest |
| `plugins/spotify/plugin.yaml` | 14 | backend Spotify plugin manifest |
| `plugins/model-providers/` (28 sub-plugins) | — | model-provider kind plugins |
| `plugins/memory/` (8 sub-plugins) | — | exclusive memory plugins |
| `plugins/image_gen/` (3 sub-plugins) | — | backend image generation plugins |
| `plugins/platforms/` (irc, teams) | — | platform adapter plugins |
| `plugins/observability/` (langfuse) | — | standalone observability plugin |
| `plugins/kanban/` | — | standalone Kanban plugin |
| `plugins/google_meet/` | — | standalone Google Meet plugin |
| `plugins/hermes-achievements/` | — | standalone achievement system plugin |
| `plugins/strike-freedom-cockpit/` | — | standalone status dashboard plugin |
| `plugins/example-dashboard/` | — | example dashboard plugin |
| `web/src/plugins/` | — | Web Dashboard Plugin SDK |

**One-liner:** The plugin system uses PluginManager for 4-source scan discovery (bundled + user + project + pip entry-point), declarative registration via PluginManifest (5 kinds: standalone/backend/exclusive/platform/model-provider), PluginContext provides register_tool/register_hook/register_command registration interfaces, 13+ built-in plugins cover memory (8 backend), model-providers (28 sub), image_gen (3 backend), spotify (backend), platforms (2 adapter), disk-cleanup, kanban, observability, etc., CLI commands support install/update/remove/enable/disable/list full lifecycle management.

---

## Architecture Overview

```
                    ┌───────────────────────────────────────────────────┐
                    │          PluginManager (1362L)                    │
                    │                                                   │
                    │  discover_and_load():                             │
                    │  ┌─────────────────────────────────────────────┐ │
                    │  │ 1. Bundled plugins (plugins/<name>/)        │ │
                    │  │    skip: memory, context_engine, platforms,  │ │
                    │  │          model-providers (own discovery)     │ │
                    │  │ 2. User plugins (~/.hermes/plugins/)         │ │
                    │  │ 3. Project plugins (.hermes/plugins/)        │ │
                    │  │    (requires HERMES_ENABLE_PROJECT_PLUGINS)  │ │
                    │  │ 4. Pip / entry-point plugins                 │ │
                    │  └─────────────────────────────────────────────┘ │
                    │                                                   │
                    │  PluginManifest → LoadedPlugin → PluginContext    │
                    │                                                   │
                    │  Kind system:                                     │
                    │  ┌─ standalone ──┐ opt-in via plugins.enabled     │
                    │  ┌─ backend ──────┐ pluggable backend for core tool│
                    │  ┌─ exclusive ────┐ exactly one active (memory)   │
                    │  ┌─ platform ─────┐ gateway messaging adapter      │
                    │  ┌─ model-provider┐ inference provider backend      │
                    └───────────────────────────────────────────────────┘
                                       │
       ┌───────────────────────────────▼────────────────────────────────┐
       │            Plugin Lifecycle                                    │
       │                                                                │
       │  plugin.yaml → PluginManifest.parse_yaml()                    │
       │       ↓                                                        │
       │  _check_requires_env() → skip if missing                      │
       │       ↓                                                        │
       │  import module → find register() function                     │
       │       ↓                                                        │
       │  PluginContext(manifest, manager) → register() call           │
       │       ↓                                                        │
       │  ctx.register_tool() → ToolRegistry.register()               │
       │  ctx.register_hook() → PluginManager._hooks[hook_name]       │
       │  ctx.register_command() → PluginManager._plugin_commands     │
       │  ctx.register_platform() → platform_registry.register()      │
       │  ctx.register_context_engine() → PluginManager._context_engine│
       └────────────────────────────────────────────────────────────────┘
                                       │
       ┌───────────────────────────────▼────────────────────────────────┐
       │           13+ Built-in Plugins                                 │
       │                                                                │
       │  ┌─ memory (exclusive) ──────────────────────────────────────┐│
       │  │  8 backends: mem0, byterover, hindsight, holographic,     ││
       │  │  honcho, openviking, retaindb, supermemory               ││
       │  │  Selection via memory.provider config key                  ││
       │  └──────────────────────────────────────────────────────────┘│
       │                                                                │
       │  ┌─ model-providers (28 sub-plugins) ──────────────────────┐ │
       │  │  anthropic, openrouter, copilot, nous, gemini, z.ai,    │ │
       │  │  deepseek, bedrock, azure-foundry, huggingface, nvidia, │ │
       │  │  xai, minimax, alibaba, arcee, ollama-cloud, kimi, ... │ │
       │  └──────────────────────────────────────────────────────────┘ │
       │                                                                │
       │  ┌─ image_gen (backend) ─┐ ┌─ spotify (backend) ─┐ ┌─ platforms ──┐ │
       │  │  openai, xai, codex   │ │  7 Spotify tools    │ │  irc, teams  │ │
       │  └───────────────────────┘ └─────────────────────┘ └──────────────┘ │
       │                                                                │
       │  ┌─ standalone plugins ────────────────────────────────────┐ │
       │  │  disk-cleanup, kanban, google_meet,                     │ │
       │  │  observability/langfuse, hermes-achievements,           │ │
       │  │  strike-freedom-cockpit, example-dashboard              │ │
       │  └──────────────────────────────────────────────────────────┘ │
       └────────────────────────────────────────────────────────────────┘
```

---

## TL;DR

The plugin system uses PluginManager as its core, scanning 4 sources to discover plugins (bundled within repo, user ~/.hermes/plugins/, project .hermes/plugins/, pip entry-point), with PluginManifest YAML for declarative registration. 5 plugin kinds determine loading strategy: standalone (opt-in via `plugins.enabled`), backend (pluggable backend for core tools), exclusive (only one active per category, e.g. memory), platform (gateway messaging platform adapter), model-provider (inference provider pluggable backend). PluginContext facade provides 5 registration methods (tool, hook, command, platform, context_engine). 13+ built-in plugins cover memory (8 backend), model-providers (28 sub), image_gen (3 backend), spotify (backend), platforms (2 adapter) and multiple standalone plugins. CLI via plugins_cmd.py provides install/update/remove/enable/disable/list full lifecycle management, and the Web Dashboard has an independent Plugin SDK.

---

## 1. PluginManifest Declarative Registration

### 1.1 Manifest Data Structure

PluginManifest is the parsed representation of a YAML manifest, containing name, version, kind, provides_tools, provides_hooks and other metadata.

**Source location: hermes_cli/plugins.py:180-214**

```python
class PluginManifest:
    """Parsed representation of a plugin.yaml manifest."""
    name: str
    version: str = ""
    description: str = ""
    author: str = ""
    requires_env: List[Union[str, Dict[str, Any]]] = field(default_factory=list)
    provides_tools: List[str] = field(default_factory=list)
    provides_hooks: List[str] = field(default_factory=list)
    source: str = ""        # "user", "project", or "entrypoint"
    path: Optional[str] = None
    # Plugin kind — controls loading strategy
    kind: str = "standalone"
    #   standalone:      hooks/tools of its own; opt-in via plugins.enabled
    #   backend:          pluggable backend for an existing core tool (e.g. image_gen)
    #   exclusive:        exactly one active provider (memory). Selection via config.
    #   platform:         gateway messaging platform adapter (e.g. IRC)
    #   model-provider:   inference provider backend (28 sub-plugins in model-providers/)
    key: str = ""  # Registry key for plugins.enabled/disabled lookups
```

### 1.2 Kind-specific Manifest Examples

**disk-cleanup (standalone):**
```yaml
# Source location: plugins/disk-cleanup/plugin.yaml
name: disk-cleanup
version: 2.0.0
description: "Auto-track and clean up ephemeral files..."
author: "@LVT382009 (original), NousResearch (plugin port)"
hooks:
  - post_tool_call
  - on_session_end
```

**spotify (backend):**
```yaml
# Source location: plugins/spotify/plugin.yaml
name: spotify
version: 1.0.0
description: "Native Spotify integration — 7 tools..."
author: NousResearch
kind: backend
provides_tools:
  - spotify_playback
  - spotify_devices
  - spotify_queue
  - spotify_search
  - spotify_playlists
  - spotify_albums
  - spotify_library
```

**openrouter-provider (model-provider):**
```yaml
# Source location: plugins/model-providers/openrouter/plugin.yaml
name: openrouter-provider
kind: model-provider
version: 1.0.0
description: OpenRouter aggregator
author: Nous Research
```

**mem0 (exclusive/memory):**
```yaml
# Source location: plugins/memory/mem0/plugin.yaml
name: mem0
version: 1.0.0
description: "Mem0 — server-side LLM fact extraction..."
pip_dependencies:
  - mem0ai
```

---

## 2. LoadedPlugin Runtime State

### 2.1 LoadedPlugin Data Structure

LoadedPlugin tracks the runtime state of a single plugin: loaded module, registered tools/hooks/commands, enabled status, and error information.

**Source location: hermes_cli/plugins.py:217-227**

```python
@dataclass
class LoadedPlugin:
    """Runtime state for a single loaded plugin."""
    manifest: PluginManifest
    module: Optional[types.ModuleType] = None
    tools_registered: List[str] = field(default_factory=list)
    hooks_registered: List[str] = field(default_factory=list)
    commands_registered: List[str] = field(default_factory=list)
    enabled: bool = False
    error: Optional[str] = None
```

---

## 3. PluginContext Registration Facade

### 3.1 Five Registration Methods

PluginContext is a facade provided to each plugin's `register()` function, supporting 5 registration operations.

**Source location: hermes_cli/plugins.py:233-260**

```python
class PluginContext:
    """Facade given to plugins so they can register tools and hooks."""

    def __init__(self, manifest: PluginManifest, manager: "PluginManager"):
        self.manifest = manifest
        self._manager = manager

    def register_tool(
        self,
        name: str,
        toolset: str,
        schema: dict,
        handler: Callable,
        check_fn: Callable | None = None,
        requires_env: list | None = None,
        is_async: bool = False,
        description: str = "",
        emoji: str = "",
    ) -> None:
        """Register a tool in the global registry and track it as plugin-provided."""
        from tools.registry import registry
        registry.register(
            name=name,
            toolset=toolset,
            ...
        )
```

### 3.2 Registration Method List

| Method | Target Registry | Purpose | Example |
|--------|----------------|---------|---------|
| `register_tool()` | `tools.registry` | Register tool to global registry | spotify_register_tool |
| `register_hook()` | `PluginManager._hooks` | Register lifecycle hook | disk-cleanup post_tool_call |
| `register_command()` | `PluginManager._plugin_commands` | Register CLI slash command | /spotify, /kanban |
| `register_platform()` | `platform_registry` | Register platform adapter | irc, teams |
| `register_context_engine()` | `PluginManager._context_engine` | Register context engine | lcm |

---

## 4. PluginManager Discovery and Loading

### 4.1 Four-Source Scan Discovery

**Source location: hermes_cli/plugins.py:617-675**

```python
def discover_and_load(self, force: bool = False) -> None:
    """Scan all plugin sources and load each plugin found."""
    if self._discovered and not force:
        return

    manifests: List[PluginManifest] = []

    # 1. Bundled plugins (<repo>/plugins/<name>/)
    repo_plugins = get_bundled_plugins_dir()
    manifests.extend(
        self._scan_directory(
            repo_plugins,
            source="bundled",
            skip_names={"memory", "context_engine", "platforms", "model-providers"},
        )
    )
    manifests.extend(
        self._scan_directory(repo_plugins / "platforms", source="bundled")
    )

    # 2. User plugins (~/.hermes/plugins/)
    user_dir = get_hermes_home() / "plugins"
    manifests.extend(self._scan_directory(user_dir, source="user"))

    # 3. Project plugins (./.hermes/plugins/)
    if _env_enabled("HERMES_ENABLE_PROJECT_PLUGINS"):
        project_dir = Path.cwd() / ".hermes" / "plugins"
        manifests.extend(self._scan_directory(project_dir, source="project"))

    # 4. Pip / entry-point plugins
    manifests.extend(self._scan_entry_points())
```

### 4.2 Loading Process

For each discovered manifest: check environment variable dependencies → import module → find `register()` function → create PluginContext → call `register(ctx)` → collect registration results.

---

## 5. Plugin CLI Commands (`hermes_cli/plugins_cmd.py`, 1544 lines)

### 5.1 Command List

| Command | Function | Purpose |
|---------|----------|---------|
| `hermes plugins install <url>` | `cmd_install()` | Install plugin from Git URL/pip |
| `hermes plugins update <name>` | `cmd_update()` | Update installed plugin |
| `hermes plugins remove <name>` | `cmd_remove()` | Remove plugin |
| `hermes plugins enable <name>` | `_save_enabled_set()` | Enable standalone plugin |
| `hermes plugins disable <name>` | `_save_disabled_set()` | Disable plugin |
| `hermes plugins list` | `cmd_list()` | List all plugins + status |

### 5.2 Install Core Logic

**Source location: hermes_cli/plugins_cmd.py:309-397**

```python
def _install_plugin_core(identifier: str, *, force: bool) -> tuple[Path, dict, str]:
    """Core install logic — clone/download, validate manifest, pip install deps."""
```

### 5.3 Manifest Parsing and Validation

**Source location: hermes_cli/plugins_cmd.py:122-136**

```python
def _read_manifest(plugin_dir: Path) -> dict:
    """Read and parse a plugin.yaml manifest."""
```

---

## 6. Built-in Plugin List

### 6.1 Statistics by Category

| Category | Kind | Count | Sub-plugins | Loading Strategy |
|----------|------|-------|------------|-----------------|
| model-providers | model-provider | 1 | 28 sub | Auto-load (bundled backends) |
| memory | exclusive | 1 | 8 backends | Config-driven (`memory.provider`) |
| image_gen | backend | 1 | 3 backends | Auto-load (bundled backends) |
| spotify | backend | 1 | — | Auto-load (bundled backend) |
| platforms | platform | 1 | 2 (irc, teams) | Auto-load (bundled platforms) |
| standalone | standalone | 7 | — | Opt-in (`plugins.enabled`) |

### 6.2 model-providers (28 sub-plugins)

The `plugins/model-providers/` subdirectory contains 28 model-provider kind plugins providing inference provider backends: ai-gateway, alibaba, alibaba-coding-plan, anthropic, arcee, azure-foundry, bedrock, copilot, copilot-acp, custom, deepseek, gemini, gmi, huggingface, kilocode, kimi-coding, minimax, nous, nvidia, ollama-cloud, openai-codex, opencode-zen, openrouter, qwen-oauth, stepfun, xai, xiaomi, zai

### 6.3 memory (8 backends)

mem0, byterover, hindsight, holographic, honcho, openviking, retaindb, supermemory

### 6.4 image_gen (3 backends)

openai, xai, openai-codex

### 6.5 spotify (backend)

Spotify music integration — 7 tools (playback, devices, queue, search, playlists, albums, library). Source location: plugins/spotify/plugin.yaml, kind: backend.

### 6.6 platforms (2 adapters)

irc, teams

### 6.7 standalone plugins

disk-cleanup, kanban, google_meet, observability/langfuse, hermes-achievements, strike-freedom-cockpit, example-dashboard

---

## 7. Web Dashboard Plugin SDK (`web/src/plugins/`)

The Web Dashboard has an independent Plugin SDK, allowing frontend plugins to register UI components and interaction logic.

---

## Comparison Tables

### Table 1: Plugin Kind Comparison

| Kind | Auto-Load | Selection Method | Examples | Conflict Resolution |
|------|-----------|-----------------|---------|---------------------|
| standalone | No (opt-in) | `plugins.enabled` config | disk-cleanup, kanban | Multiple can coexist |
| backend | Yes (bundled) | Config-driven per tool | spotify, image_gen/openai | One per tool category |
| exclusive | Config-driven | `<category>.provider` config | memory/mem0 | Exactly one active |
| platform | Yes (bundled) | Gateway auto-connect | irc, teams | One per platform |
| model-provider | Yes (bundled) | `model.provider` config | openrouter, anthropic | Provider selection logic |

### Table 2: Plugin Source Comparison

| Source | Path | Trust Level | Requires Enable | Priority on Collision |
|--------|------|-------------|-----------------|----------------------|
| Bundled (repo) | `plugins/<name>/` | High (reviewed) | kind-dependent | Lowest (overridden by user) |
| User (~/.hermes) | `~/.hermes/plugins/` | Medium | `plugins.enabled` | Higher (overrides bundled) |
| Project (.hermes) | `.hermes/plugins/` | Medium | `HERMES_ENABLE_PROJECT_PLUGINS` | Highest |
| Pip entry-point | Python packages | Low (unreviewed) | `plugins.enabled` | Medium |

### Table 3: Plugin Registration Method Comparison

| Method | Target | Lifecycle | Called By | Example |
|--------|--------|-----------|-----------|---------|
| `register_tool()` | ToolRegistry | Persistent | Plugin register() | spotify_playback |
| `register_hook()` | _hooks dict | Persistent | Plugin register() | disk-cleanup post_tool_call |
| `register_command()` | _plugin_commands | Persistent | Plugin register() | /spotify |
| `register_platform()` | platform_registry | Persistent | Plugin register() | IRC adapter |
| `register_context_engine()` | _context_engine | Singleton | Plugin register() | LCM engine |

---

## Summary Table

| Component | Lines | Responsibility |
|-----------|-------|----------------|
| `hermes_cli/plugins.py` | 1362 | PluginManager discovery/loading/lifecycle + manifest/loadedplugin/plugincontext |
| `hermes_cli/plugins_cmd.py` | 1544 | CLI commands install/update/remove/enable/disable/list |
| `plugins/model-providers/` | 28 manifests | Inference provider pluggable backends (model-provider kind) |
| `plugins/memory/` | 8 manifests | Memory system exclusive selection |
| `plugins/image_gen/` | 3 manifests | Image generation backend |
| `plugins/spotify/` | 1 manifest | Spotify music integration (backend kind) |
| `plugins/platforms/` | 2 manifests | Gateway messaging platform adapters |
| `plugins/disk-cleanup/` | 1 manifest | Ephemeral file auto-cleanup |
| `plugins/kanban/` | 1 manifest | Kanban task management |
| `plugins/google_meet/` | 1 manifest | Google Meet integration |
| `plugins/observability/` | 1 manifest | Langfuse observability |
| `plugins/hermes-achievements/` | 1 manifest | Achievement system |
| `plugins/strike-freedom-cockpit/` | 1 manifest | Status dashboard |
| `plugins/example-dashboard/` | 1 manifest | Example dashboard |
| `web/src/plugins/` | — | Web Dashboard Plugin SDK |

---

[<< 15 — Infrastructure & Deployment](/en/chapters/15-infrastructure-deployment) | [17 — Design Patterns >>](/en/chapters/17-design-patterns)