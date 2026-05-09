# 01 — Project Overview & Design Philosophy: From v0.1 to v0.12 Self-Evolving Agent Platform

[Previous Chapter](/en/chapters/overview) | [Back to Table of Contents](/en/chapters/overview)

---

## Source Locations & Key Files

| File | Lines | Responsibility |
|------|------|------|
| `README.md` | 182 | Project main entry documentation, design philosophy & quick install guide |
| `RELEASE_v0.2.0.md` | 383 | First public release, 216 PRs, 63 contributors |
| `RELEASE_v0.3.0.md` | 378 | Streaming + plugin architecture + Anthropic native provider |
| `RELEASE_v0.4.0.md` | ~250 | Persistent Shell + ACP + voice mode |
| `RELEASE_v0.5.0.md` | 349 | Supply chain hardening + HuggingFace + Telegram Topics |
| `RELEASE_v0.6.0.md` | 250 | Multi-instance Profiles + MCP Server + Feishu/WeCom |
| `RELEASE_v0.7.0.md` | ~350 | Pluggable memory + credential pool rotation + Camofox |
| `RELEASE_v0.8.0.md` | ~350 | MiMo v2 Pro + Live Model Switch + Approval Buttons |
| `RELEASE_v0.9.0.md` | ~400 | Web Dashboard + iMessage + WeChat + Android |
| `RELEASE_v0.10.0.md` | 28 | Nous Tool Gateway |
| `RELEASE_v0.11.0.md` | ~400 | Ink TUI rewrite + Transport ABC + AWS Bedrock + QQBot |
| `RELEASE_v0.12.0.md` | ~400 | Autonomous Curator + LM Studio + Teams + Spotify |

**One-line summary:** Hermes Agent is a "self-evolving agent" built by Nous Research — with a built-in learning loop that creates skills from experience, self-improves through usage, persists memory across sessions, searches historical conversations, and builds user models; runs on any infrastructure, security-first.

---

## Architecture Overview

```
                         ┌─────────────────────────────────────────┐
                         │        Hermes Agent v0.12 Ecosystem     │
                         ├─────────────────────────────────────────┤
                         │                                         │
  ┌──────────┐           │  ┌─────────────┐   ┌──────────────────┐ │
  │ 21+      │◄─────────┤  │  Gateway    │   │   CLI / TUI      │ │
  │ Platform │           │  │  (15K loc) │   │  (10.8K loc)     │ │
  │ Message  │           │  └─────────────┘   └──────────────────┘ │
  │ Gateways │           │         │                  │            │
  ├──────────┤           │         ▼                  ▼            │
  │Telegram  │           │  ┌───────────────────────────────────┐ │
  │Discord   │           │  │     AIAgent Engine (14.4K loc)    │ │
  │Slack     │           │  │  run_agent.py — core dialog loop  │ │
  │WhatsApp  │           │  │  ┌────────┐ ┌────────┐ ┌────────┐ │ │
  │Signal    │           │  │  │Transport│ │Context │ │Memory  │ │ │
  │Matrix    │           │  │  │  Layer  │ │Compress│ │Plugins │ │ │
  │Feishu    │           │  │  └────────┘ └────────┘ └────────┘ │ │
  │WeCom     │           │  └───────────────────────────────────┘ │
  │WeChat    │           │         │                  │            │
  │QQBot     │           │         ▼                  ▼            │
  │iMessage  │           │  ┌────────────┐    ┌──────────────────┐ │
  │Yuanbao   │           │  │ Tool System │    │ Skills Ecosystem │ │
  │DingTalk  │           │  │ (537+ loc) │    │ (70+ skills)     │ │
  │Email     │           │  │ 50+ tools  │    │ Curator (auto)   │ │
  │SMS       │           │  └────────────┘    └──────────────────┘ │
  │HA/Webhook│           │                                         │
  │Teams     │           │  ┌─────────────────────────────────────┐ │
  └──────────┘           │  │     28+ Inference Providers         │ │
                         │  │ NousPortal│OpenRouter│Anthropic    │ │
  ┌──────────┐           │  │ OpenAI│Bedrock│xAI│Gemini│HuggingFace│ │
  │ 7        │           │  │ Ollama│LMStudio│Arcee│Step│NIM│Vercel│ │
  │ Terminal │           │  │ CodexOAuth│Kimi│MiniMax│Alibaba│GLM │ │
  │ Backends │           │  │ GMI│Azure│Tokenhub│Lemonade│Fireworks│ │
  │Local     │           │  └─────────────────────────────────────┘ │
  │SSH       │           │                                         │
  │Docker    │           │  ┌─────────────────────────────────────┐ │
  │Modal     │           │  │  Storage Layer                       │ │
  │Daytona   │           │  │  SessionDB│Config│Memory│SQLite WAL │ │
  │Singularity│           │  └─────────────────────────────────────┘ │
  │Vercel    │           │                                         │
  │Sandbox   │           └─────────────────────────────────────────┘
  └──────────┘           │                                         │
                         └─────────────────────────────────────────┘
```

---

## TL;DR

Hermes Agent is an open-source MIT-licensed self-evolving AI agent platform developed by Nous Research, iterating from v0.1 internal prototype through 11 official releases to v0.12 (2026.4.30), covering 28+ inference providers, 21+ messaging platforms, 29 self-registered tools (69 tool files) and 70+ skills. Its core design philosophy is a three-layer closed loop: self-improvement (experience → skill → usage → optimization), runs anywhere ($5 VPS to GPU clusters to serverless), security-first (command approval, PII elimination, SSRF protection, supply chain audit). The codebase spans 700K lines Python + 70K lines TypeScript, with the core module `run_agent.py` at 14,404 lines in a single file carrying the complete agent engine.

---

## 1. Project History: Evolution from v0.2 to v0.12

### 1.1 v0.2.0 — Birth Phase (2026.3.12)

> "First tagged release since v0.1.0. In just over two weeks, 216 merged PRs from 63 contributors, resolving 119 issues."

v0.2 was Hermes Agent's public debut, bursting from an internal project to a full-featured agent platform within two weeks. Core features include:

- **Multi-platform messaging gateway** — Telegram, Discord, Slack, WhatsApp, Signal, Email, Home Assistant unified management across seven platforms
- **MCP client** — stdio + HTTP transport, reconnection, resource/prompt discovery, sampling support
- **Skills ecosystem** — 70+ built-in and optional skills, 15+ categories, Skills Hub community discovery
- **Centralized provider routing** — `call_llm()`/`async_call_llm()` unified API replacing scattered logic
- **ACP server** — VS Code, Zed, JetBrains editor integration
- **CLI skin engine** — 7 built-in skins + custom YAML skins
- **Git Worktree isolation** — `hermes -w` launches isolated sessions in worktree
- **Filesystem checkpoints & rollback** — `/rollback` restores to snapshot
- **3,289 tests** — from zero to full coverage test suite

Source location: `RELEASE_v0.2.0.md:1-383`

### 1.2 v0.3.0 — Streaming & Plugin Phase (2026.3.17)

> "The streaming, plugins, and provider release — unified real-time token delivery, first-class plugin architecture, rebuilt provider system."

Key evolution:

- **Unified streaming infrastructure** — CLI and all gateway platforms real-time per-token delivery
- **Native Anthropic provider** — Claude Code credential auto-discovery, OAuth PKCE, native prompt caching
- **Smart approval + /stop command** — Codex-inspired approval system, remembers safety preferences
- **Honcho memory integration** — async writes, configurable recall modes, multi-user isolation
- **Voice mode** — CLI push-to-speak, Telegram/Discord voice notes, Whisper transcription
- **Concurrent tool execution** — ThreadPoolExecutor parallel execution of independent tool calls
- **PII elimination** — `privacy.redact_pii` auto-scrub personal information
- **`/browser connect` via CDP** — Chrome DevTools Protocol connection to live browser

Source location: `RELEASE_v0.3.0.md:1-378`

### 1.3 v0.4.0 — Persistent Shell & RL Phase (2026.3.20~22)

Key evolution:

- **Persistent Shell mode** — local and SSH backends maintain shell state across tool calls
- **Agentic OPD** — new RL training environment, policy distillation
- **Tirith pre-execution scanning** — static analysis of terminal command safety

### 1.4 v0.5.0 — Hardening Phase (2026.3.28)

> "The hardening release — Hugging Face provider, /model command overhaul, supply chain audit."

Key evolution:

- **HuggingFace first-class inference provider** — HF Inference API integration, curated agentic model picker
- **Telegram Private Chat Topics** — project-level conversations, each Topic bound to functional skills
- **Native Modal SDK** — replaces swe-rex, eliminates tunnel dependencies
- **Plugin lifecycle hooks** — `pre_llm_call`, `post_llm_call`, `on_session_start`, `on_session_end`
- **Supply chain hardening** — removed compromised `litellm`, locked version ranges, hash-verified lock files
- **Anthropic output limit fix** — per-model native output limits replacing hardcoded 16K `max_tokens`

Source location: `RELEASE_v0.5.0.md:1-349`

### 1.5 v0.6.0 — Multi-instance Phase (2026.3.30)

> "The multi-instance release — Profiles, MCP server mode, Docker container, fallback provider chains."

Key evolution:

- **Profiles multi-instance** — same installation running multiple isolated Hermes instances, each with its own config/memory/sessions/skills/gateways
- **MCP Server mode** — `hermes mcp serve` exposes conversations and sessions to MCP clients
- **Docker container** — official Dockerfile, CLI and gateway dual mode
- **Ordered fallback provider chains** — automatic switch to next in chain when primary provider fails
- **Feishu/Lark platform** — event subscriptions, message cards, group chat, interactive callbacks
- **WeCom (Enterprise WeChat)** — text/image/voice messages, group chat, callback verification
- **Slack multi-workspace OAuth** — single gateway connecting multiple Slack workspaces

Source location: `RELEASE_v0.6.0.md:1-250`

### 1.6 v0.7.0 — Resilience Phase (2026.4.3)

> "The resilience release — pluggable memory, credential pool rotation, Camofox anti-detection, inline diff previews."

Key evolution:

- **Pluggable memory provider interface** — ABC plugin system, Honcho restored full integration
- **Same-provider credential pool** — multi API key auto-rotation, `least_used` strategy, 401 failure auto-switch
- **Camofox anti-detection browser** — stealth browsing, VNC URL discovery, SSRF bypass
- **Inline Diff preview** — file write and patch operations show line-level differences
- **Secret exfiltration blocking** — browser URL and LLM response scan for secret patterns

Source location: `RELEASE_v0.7.0.md:1-350`

### 1.7 v0.8.0 — Intelligence Phase (2026.4.8)

> "The intelligence release — background task auto-notifications, free MiMo v2 Pro, live model switching."

Key evolution:

- **Background task auto-notification** — `notify_on_complete` auto-notifies agent of task completion
- **Free MiMo v2 Pro** — Nous Portal free tier for auxiliary tasks (compression/vision/summary)
- **Live Model Switching** — `/model` mid-conversation model and provider switching on any platform
- **Self-optimizing GPT/Codex tool usage guidance** — auto behavioral benchmarking diagnosing and patching 5 failure modes
- **Google AI Studio native provider** — Gemini direct API access
- **Inactivity timeout** — based on actual tool activity rather than wall-clock time
- **Approval buttons** — Slack and Telegram native button approval for dangerous commands
- **MCP OAuth 2.1 PKCE + OSV malware scanning**

Source location: `RELEASE_v0.8.0.md:1-350`

### 1.8 v0.9.0 — Everywhere Phase (2026.4.13)

> "The everywhere release — mobile, iMessage, WeChat, Fast Mode, web dashboard, 16 platforms."

Key evolution:

- **Local Web Dashboard** — browser management interface, config/sessions/skills/gateways all-in-one
- **Fast Mode (`/fast`)** — OpenAI Priority Processing + Anthropic fast tier
- **iMessage via BlueBubbles** — Apple messaging ecosystem integration
- **WeChat native support** — iLink Bot API, streaming cursor, media upload
- **Termux/Android support** — native running on Android
- **Background process monitoring** — `watch_patterns` real-time output pattern match notifications
- **Pluggable context engine** — custom context management plugins
- **Unified proxy support** — SOCKS proxy and system proxy auto-detection
- **16 supported platforms** — BlueBubbles and WeChat added
- **`hermes backup` & `hermes import`** — config/sessions/skills/memory backup and restore

Source location: `RELEASE_v0.9.0.md:1-400`

### 1.9 v0.10.0 — Tool Gateway Phase (2026.4.16)

> "Paid Nous Portal subscribers can now use web search, image generation, TTS, and browser automation through their existing subscription."

Key evolution:

- **Nous Tool Gateway** — paid subscribers use Firecrawl/FAL/OpenAI TTS/Browser Use without extra API keys

Source location: `RELEASE_v0.10.0.md:1-28`

### 1.10 v0.11.0 — Interface Phase (2026.4.23)

> "The Interface release — Ink TUI rewrite, Transport ABC, AWS Bedrock, 17th platform (QQBot)."

Key evolution:

- **Ink-based TUI** — React/Ink rewrite of interactive CLI, JSON-RPC Python backend
- **Transport ABC** — format conversion and HTTP transport extracted from `run_agent.py` into `agent/transports/` layer
- **Native AWS Bedrock** — Converse API transport layer
- **Five new inference paths** — NVIDIA NIM, Arcee AI, Step Plan, Gemini CLI OAuth, Vercel ai-gateway
- **GPT-5.5 over Codex OAuth** — ChatGPT Codex OAuth inference
- **QQBot — 17th platform** — QQ Official API v2, QR scan configuration
- **Plugin surface expansion** — slash commands, tool dispatch, tool veto, result rewrite, terminal output transform, dashboard tabs
- **`/steer` — mid-run agent steering** — inject prompts without interrupting turn or breaking prompt cache

Source location: `RELEASE_v0.11.0.md:1-400`

### 1.11 v0.12.0 — Curator Phase (2026.4.30)

> "The Curator release — Hermes Agent now maintains itself. Autonomous background Curator grades, prunes, and consolidates your skill library."

Key evolution:

- **Autonomous Curator** — background agent with 7-day cycle auto-grading/merging/pruning skill library
- **Self-improvement loop major upgrade** — class-based review prompts, active update preferences, live runtime inheritance
- **ComfyUI v5 and TouchDesigner-MCP** — elevated from optional to built-in default
- **LM Studio first-class provider** — upgraded from custom endpoint alias to native provider
- **Four new inference providers** — GMI Cloud, Azure AI Foundry, MiniMax OAuth, Tencent Tokenhub
- **Pluggable gateway platforms + Microsoft Teams** — gateway becomes plugin host, Teams is first plugin platform
- **Tencent Yuanbao — 18th messaging platform**
- **Spotify native tool + skill** — PKCE OAuth, 7 tools, interactive configuration wizard
- **Google Meet plugin** — join call/transcribe/speak/follow-up
- **`hermes -z` one-shot mode** — non-interactive single query
- **TUI cold start ~57% speedup** — lazy agent init, mtime-cached config, memoized tool definitions
- **Secret redaction default off** — patched patch-corruption long-standing issue

Source location: `RELEASE_v0.12.0.md:1-400`

---

## 2. Design Philosophy

### 2.1 Self-Improving (Self-Evolving Closed Loop)

Hermes Agent's core differentiator is a built-in learning loop, rather than a simple "dialog + tools" model:

```text
┌──────────────────────────────────────────────────────────────┐
│                    Self-Improving Loop                       │
│                                                              │
│  Experience ──► Skill Creation ──► Usage Load ──► Active ──► Curator │
│    │               │              │          Improvement      Prune │
│    ▼               ▼              ▼               │            │    │
│  Memory      │ agentskills.io │ Skill  │ rubric    │ Merge/  │ │
│  Persistence │ open standard │ View   │ grading   │ Prune   │ │
│  Cross-session│              │        │           │         │ │
│            │                │        │           │         │ │
│  ──◄── Search historical conversations (FTS5 + LLM summary) ──◄── Honcho user model ──│
└──────────────────────────────────────────────────────────────┘
```

Source location: `README.md:22` (closed loop description), `RELEASE_v0.12.0.md:12-14` (Curator + self-improvement loop upgrade)

### 2.2 Runs Anywhere

```text
┌─────────────┐  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐
│ $5 VPS      │  │ GPU Cluster │  │ Serverless   │  │ Android/Termux│
│ (local/SSH) │  │ (Modal/Daytona)│ │ (Vercel/Modal)│ │ (Termux adapted)│
└─────────────┘  └─────────────┘  └──────────────┘  └───────────────┘
        │                │                │                 │
        ▼                ▼                ▼                 ▼
┌─────────────────────────────────────────────────────────────┐
│  7 Terminal Backends:                                       │
│  local │ Docker │ SSH │ Singularity │ Modal │ Daytona │ Vercel│
│  Sandbox                                                    │
│                                                             │
│  Modal/Daytona serverless persistence:                      │
│  Near-zero cost when dormant, pay-on-demand when awakened   │
└─────────────────────────────────────────────────────────────┘
```

Source location: `README.md:25` (runs anywhere description)

### 2.3 Security-First

Hermes Agent has undergone deep security hardening in every release:

| Version | Security Features |
|------|----------|
| v0.2 | Path traversal fix, shell injection prevention, dangerous command detection, 0600/0700 file permissions, atomic writes |
| v0.3 | PII elimination, Tirith pre-execution scanning, subprocess environment variable stripping, Docker cwd explicit opt-in |
| v0.5 | SSRF protection, zip-slip prevention, shell injection `_expand_path`, supply chain audit CI |
| v0.7 | Secret exfiltration blocking (URL/base64/prompt injection), credential directory expansion |
| v0.8 | MCP OAuth 2.1 PKCE, OSV malware scanning, SSRF reassembly prevention, tar traversal |
| v0.9 | Path traversal in checkpoint, shell injection neutralization, Twilio webhook signature verification, git argument injection |
| v0.11 | SSRF in browser_navigate, restriction of subagent toolsets |
| v0.12 | Secret redaction default off (patched patch-corruption), skill pinning prevents Curator overwrite |

Source location: Security & Reliability sections of each RELEASE file

---

## 3. Scale Statistics

### 3.1 Code Scale

| Dimension | Value |
|------|------|
| Total Python lines | ~704K |
| Total TypeScript lines | ~73K |
| `run_agent.py` (core engine) | 14,404 |
| `gateway/run.py` (message gateway) | 15,046 |
| `hermes_cli/main.py` (CLI entry) | 10,876 |
| `hermes_cli/config.py` (config system) | 4,939 |
| `hermes_state.py` (session database) | 2,669 |
| `tools/registry.py` (tool registry) | 537 |

### 3.2 Ecosystem Scale

| Dimension | Value | Source Version |
|------|------|----------|
| Inference providers | 28+ | v0.12 (LM Studio, GMI Cloud, Azure, Tokenhub added) |
| Messaging platforms | 19+ | v0.12 (Yuanbao + Teams plugin) |
| Built-in + optional skills | 70+ | since v0.2, continuously growing |
| Tools | 50+ | README + each version |
| Tests | 3,289+ | since v0.2, continuously growing |
| Community contributors | 63+ (v0.2) → 213+ (v0.12) | Contributors stats from each version |
| License | MIT | `LICENSE` |

### 3.3 Version Iteration Scale

| Version | PRs | Issues | Contributors |
|------|-------|----------|----------|
| v0.2 | 216 | 119 | 63 |
| v0.3 | ~220+ | ~50+ | 15 |
| v0.5 | ~157+ | ~30+ | 5 |
| v0.6 | 95 | 16 | 4 |
| v0.7 | 168 | 46 | ~10 |
| v0.8 | 209 | 82 | ~15 |
| v0.9 | 269 | 167 | 24 |
| v0.11 | 761 | ~100+ | 29 (290 incl. co-authors) |
| v0.12 | 550 | ~50+ | 213 |

---

## 4. Target Users (6 Roles)

| Role | Use Cases | Key Features |
|------|----------|----------|
| **Developers** | Code writing, debugging, project management | Git worktree isolation, checkpoint/rollback, patch tools, skill self-creation |
| **Researchers** | RL training, trajectory generation, model evaluation | Atropos environment, OPD distillation, batch runner, trajectory compression |
| **Ops/Admins** | Automated operations, scheduled reports, monitoring | Cron scheduler, background process monitor, `watch_patterns` |
| **AI Enthusiasts** | Agent interaction, skill exploration, community contribution | Skills Hub, skin themes, TUI interaction, voice mode |
| **Security Professionals** | Penetration testing, vulnerability analysis, red team exercises | Tirith pre-scanning, dangerous command detection, SSRF protection, secret redaction |
| **Enterprise Users** | Multi-platform customer service, knowledge base, internal automation | Profiles multi-instance, credential pool, WeCom/Feishu/WeChat, approval buttons |

Source location: `README.md:19-27` (feature table indirectly describes 6 role scenarios)

---

## 5. Key Milestone Comparison

### 5.1 Version Comparison: Core Architecture Changes

| Dimension | v0.2 | v0.5 | v0.8 | v0.11 | v0.12 |
|------|------|------|------|-------|-------|
| Provider routing | `call_llm()` centralized routing | `/model` overhaul | Live model switching | Transport ABC (4 transports) | +LM Studio native |
| Memory system | MEMORY.md + Honcho | — | Supermemory plugin | — | Curator autonomous |
| Tool execution | Sequential | — | Concurrent (v0.3+) | `/steer` mid-run | ComfyUI/TouchDesigner bundled |
| Messaging platforms | 7 | +Feishu/WeCom | +Matrix Tier 1 | +QQBot (17th) | +Yuanbao/Teams (19th) |
| Terminal backends | 5 | +Modal native | +Vercel Sandbox | — | +Vercel Sandbox |
| CLI | curses TUI | — | Status bar | Ink/React TUI rewrite | ~57% cold start cut |

### 5.2 Version Comparison: Security Capability Evolution

| Security Capability | v0.2 | v0.5 | v0.8 | v0.9 | v0.12 |
|----------|------|------|------|------|-------|
| Command approval | Basic detection | — | Slack/Telegram buttons | git argument injection | — |
| PII elimination | — | v0.3: `redact_pii` | — | — | Redaction default off |
| SSRF protection | — | browser/vision/web | — | Twilio webhook | — |
| Supply chain | — | Removed litellm, hash lock | OSV scanning | — | — |
| Secret protection | Basic patterns | — | — | Exfiltration blocking | Patch-corruption fix |

### 5.3 Version Comparison: Platform Coverage Evolution

| Messaging Platform | v0.2 | v0.6 | v0.9 | v0.11 | v0.12 |
|----------|------|------|------|-------|-------|
| Telegram | yes | Webhook mode | — | — | — |
| Discord | yes | Processing reactions | — | Allowed channels | — |
| Slack | yes | Multi-workspace OAuth | — | — | — |
| WhatsApp | yes | Persistent aiohttp | — | — | — |
| Signal | yes | — | — | — | — |
| Email | yes | — | — | — | — |
| Home Assistant | yes | — | — | — | — |
| Matrix | — | — | v0.8: Tier 1 | — | — |
| Feishu/Lark | — | yes | — | — | — |
| WeCom | — | yes | Callback mode | — | — |
| iMessage | — | — | yes | — | — |
| WeChat | — | — | yes | — | — |
| DingTalk | — | — | — | — | — |
| QQBot | — | — | — | yes | — |
| Yuanbao | — | — | — | — | yes |
| Teams | — | — | — | — | yes (plugin) |

---

## 6. License & Authors

| Dimension | Details |
|------|------|
| License | MIT — `LICENSE` |
| Authors | [Nous Research](https://nousresearch.com) |
| Project Lead | @teknium1 |
| Documentation site | [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/) |
| Community | [Discord](https://discord.gg/NousResearch), [Skills Hub](https://agentskills.io) |
| Repository | [github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) |

Source location: `README.md:10` (License badge), `README.md:181` ("Built by Nous Research")

---

## Summary Table

| Component | Lines | Responsibility |
|------|------|------|
| `run_agent.py` | 14,404 | Core agent engine (AIAgent class, dialog loop, tool dispatch, streaming, failover) |
| `gateway/run.py` | 15,046 | Message gateway runner (GatewayRunner, 21+ platform adapters, session cache, lifecycle) |
| `hermes_cli/main.py` | 10,876 | CLI entry (argparse, profile override, subcommand routing, setup/model/tools/doctor) |
| `hermes_cli/config.py` | 4,939 | Config management (config.yaml parsing, env loading, provider routing, DEFAULT_CONFIG) |
| `hermes_state.py` | 2,669 | Session persistence (SessionDB, SQLite WAL, FTS5 search, message storage) |
| `tools/registry.py` | 537 | Tool registry (ToolRegistry singleton, ToolEntry, check_fn TTL cache, self-discovery) |
| Total Python | ~704K | Full project Python code |
| Total TypeScript | ~73K | TUI (React/Ink) + Web Dashboard |
| RELEASE files | 11 | v0.2 to v0.12 release notes |

---

[Next Chapter: 02 — System Architecture Overview](/en/chapters/02-architecture) | [Back to Table of Contents](/en/chapters/overview)