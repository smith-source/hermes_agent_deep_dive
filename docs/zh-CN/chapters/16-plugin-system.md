# 16 — Plugin System: 动态发现与声明式注册的可扩展架构


---

## Source Files

| File | Lines | Role |
|------|-------|------|
| `hermes_cli/plugins.py` | 1362 | PluginManager 核心 + manifest 解析 + 加载 |
| `hermes_cli/plugins_cmd.py` | 1544 | CLI 命令：install/update/remove/enable/disable/list |
| `plugins/disk-cleanup/plugin.yaml` | 8 | standalone 磁盘清理插件 manifest |
| `plugins/spotify/plugin.yaml` | 14 | backend Spotify 插件 manifest |
| `plugins/model-providers/` (28 sub-plugins) | — | model-provider 类插件 |
| `plugins/memory/` (8 sub-plugins) | — | exclusive 记忆插件 |
| `plugins/image_gen/` (3 sub-plugins) | — | backend 图片生成插件 |
| `plugins/platforms/` (irc, teams) | — | platform 平台适配插件 |
| `plugins/observability/` (langfuse) | — | standalone 可观测性插件 |
| `plugins/kanban/` | — | standalone Kanban 插件 |
| `plugins/google_meet/` | — | standalone Google Meet 插件 |
| `plugins/hermes-achievements/` | — | standalone 成就系统插件 |
| `plugins/strike-freedom-cockpit/` | — | standalone 状态仪表盘插件 |
| `plugins/example-dashboard/` | — | example 示例仪表盘插件 |
| `web/src/plugins/` | — | Web Dashboard Plugin SDK |

**One-liner:** 插件系统通过 PluginManager 实现 4 源扫描发现（bundled + user + project + pip entry-point），使用 PluginManifest 声明式注册（5 种 kind：standalone/backend/exclusive/platform/model-provider），PluginContext 提供 register_tool/register_hook/register_command 注册接口，13+ 内置插件涵盖 memory（8 backend）、model-providers（28 sub）、image_gen（3 backend）、spotify（backend）、platforms（2 adapter）、disk-cleanup、kanban、observability 等，CLI 命令支持 install/update/remove/enable/disable/list 全生命周期管理。

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

插件系统以 PluginManager 为核心，从 4 个来源扫描发现插件（bundled repo 内、user ~/.hermes/plugins/、project .hermes/plugins/、pip entry-point），使用 PluginManifest YAML 声明式注册。5 种 plugin kind 决定加载策略：standalone（opt-in via `plugins.enabled`）、backend（核心工具的可插拔后端）、exclusive（同类别仅一个活跃，如 memory）、platform（gateway 消息平台适配器）、model-provider（推理 provider 可插拔后端）。PluginContext facade 提供 5 种注册方法（tool、hook、command、platform、context_engine）。13+ 内置插件涵盖 memory（8 backend）、model-providers（28 sub）、image_gen（3 backend）、spotify（backend）、platforms（2 adapter）和多个 standalone 插件。CLI 通过 plugins_cmd.py 提供 install/update/remove/enable/disable/list 全生命周期管理，Web Dashboard 有独立的 Plugin SDK。

---

## 1. PluginManifest 声明式注册

### 1.1 Manifest 数据结构

PluginManifest 是 YAML manifest 的解析后表示，包含 name、version、kind、provides_tools、provides_hooks 等元数据。

**源码位置: hermes_cli/plugins.py:180-214**

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
# 源码位置: plugins/disk-cleanup/plugin.yaml
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
# 源码位置: plugins/spotify/plugin.yaml
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
# 源码位置: plugins/model-providers/openrouter/plugin.yaml
name: openrouter-provider
kind: model-provider
version: 1.0.0
description: OpenRouter aggregator
author: Nous Research
```

**mem0 (exclusive/memory):**
```yaml
# 源码位置: plugins/memory/mem0/plugin.yaml
name: mem0
version: 1.0.0
description: "Mem0 — server-side LLM fact extraction..."
pip_dependencies:
  - mem0ai
```

---

## 2. LoadedPlugin 运行时状态

### 2.1 LoadedPlugin 数据结构

LoadedPlugin 跟踪单个插件的运行时状态：加载的模块、注册的工具/hooks/commands、启用状态和错误信息。

**源码位置: hermes_cli/plugins.py:217-227**

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

## 3. PluginContext 注册 Facade

### 3.1 五种注册方法

PluginContext 是提供给每个插件 `register()` 函数的 facade，支持 5 种注册操作。

**源码位置: hermes_cli/plugins.py:233-260**

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

### 3.2 注册方法清单

| Method | Target Registry | Purpose | Example |
|--------|----------------|---------|---------|
| `register_tool()` | `tools.registry` | 注册工具到全局 registry | spotify_register_tool |
| `register_hook()` | `PluginManager._hooks` | 注册生命周期钩子 | disk-cleanup post_tool_call |
| `register_command()` | `PluginManager._plugin_commands` | 注册 CLI slash 命令 | /spotify, /kanban |
| `register_platform()` | `platform_registry` | 注册平台适配器 | irc, teams |
| `register_context_engine()` | `PluginManager._context_engine` | 注册上下文引擎 | lcm |

---

## 4. PluginManager 发现与加载

### 4.1 四源扫描发现

**源码位置: hermes_cli/plugins.py:617-675**

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

### 4.2 加载流程

对每个发现的 manifest：检查环境变量依赖 → 导入模块 → 查找 `register()` 函数 → 创建 PluginContext → 调用 `register(ctx)` → 收集注册结果。

---

## 5. Plugin CLI 命令 (`hermes_cli/plugins_cmd.py`, 1544 lines)

### 5.1 命令清单

| Command | Function | Purpose |
|---------|----------|---------|
| `hermes plugins install <url>` | `cmd_install()` | 从 Git URL/pip 安装插件 |
| `hermes plugins update <name>` | `cmd_update()` | 更新已安装插件 |
| `hermes plugins remove <name>` | `cmd_remove()` | 移除插件 |
| `hermes plugins enable <name>` | `_save_enabled_set()` | 启用 standalone 插件 |
| `hermes plugins disable <name>` | `_save_disabled_set()` | 禁用插件 |
| `hermes plugins list` | `cmd_list()` | 列出所有插件 + 状态 |

### 5.2 安装核心逻辑

**源码位置: hermes_cli/plugins_cmd.py:309-397**

```python
def _install_plugin_core(identifier: str, *, force: bool) -> tuple[Path, dict, str]:
    """Core install logic — clone/download, validate manifest, pip install deps."""
```

### 5.3 Manifest 解析与验证

**源码位置: hermes_cli/plugins_cmd.py:122-136**

```python
def _read_manifest(plugin_dir: Path) -> dict:
    """Read and parse a plugin.yaml manifest."""
```

---

## 6. 内置插件清单

### 6.1 按分类统计

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

Spotify 音乐集成 — 7 工具 (playback, devices, queue, search, playlists, albums, library)。源码位置: plugins/spotify/plugin.yaml, kind: backend。

### 6.6 platforms (2 adapters)

irc, teams

### 6.7 standalone plugins

disk-cleanup, kanban, google_meet, observability/langfuse, hermes-achievements, strike-freedom-cockpit, example-dashboard

---

## 7. Web Dashboard Plugin SDK (`web/src/plugins/`)

Web Dashboard 有独立的 Plugin SDK，允许前端插件注册 UI 组件和交互逻辑。

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
| `hermes_cli/plugins.py` | 1362 | PluginManager 发现/加载/生命周期 + manifest/loadedplugin/plugincontext |
| `hermes_cli/plugins_cmd.py` | 1544 | CLI 命令 install/update/remove/enable/disable/list |
| `plugins/model-providers/` | 28 manifests | 推理 provider 可插拔后端 (model-provider kind) |
| `plugins/memory/` | 8 manifests | 记忆系统 exclusive 选择 |
| `plugins/image_gen/` | 3 manifests | 图片生成 backend |
| `plugins/spotify/` | 1 manifest | Spotify 音乐集成 (backend kind) |
| `plugins/platforms/` | 2 manifests | Gateway 消息平台适配 |
| `plugins/disk-cleanup/` | 1 manifest | 临时文件自动清理 |
| `plugins/kanban/` | 1 manifest | Kanban 任务管理 |
| `plugins/google_meet/` | 1 manifest | Google Meet 集成 |
| `plugins/observability/` | 1 manifest | Langfuse 可观测性 |
| `plugins/hermes-achievements/` | 1 manifest | 成就系统 |
| `plugins/strike-freedom-cockpit/` | 1 manifest | 状态仪表盘 |
| `plugins/example-dashboard/` | 1 manifest | 示例仪表盘 |
| `web/src/plugins/` | — | Web Dashboard Plugin SDK |

---

[<< 15 — Infrastructure & Deployment](/zh-CN/chapters/15-infrastructure-deployment) | [17 — Design Patterns >>](/zh-CN/chapters/17-design-patterns)