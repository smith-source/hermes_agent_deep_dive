# 🏛️ Hermes Agent Deep Dive

> **A deep analysis series of a self-evolving AI agent — 20 chapters covering every subsystem**
>
> From architecture to security, from core engine to plugin system — every layer dissected, every module analyzed.

---

## 🌐 Language / 语言

| 🇨🇳 [中文版](/zh-CN/) | 🇺🇸 [English](/en/) |
|:---:|:---:|
| 完整的 20 篇中文深度分析 | Complete 20-chapter English deep dive |

---

## 📊 Project at a Glance

| Dimension | Scale | Highlights |
|-----------|-------|------------|
| **Python Core** | 347,240 lines | Self-evolving agent engine |
| **TypeScript Frontend** | 91,223 lines | 4 interfaces sharing one engine |
| **LLM Providers** | 28+ | OpenRouter / Anthropic / Gemini / Bedrock / Alibaba |
| **Messaging Platforms** | 21+ | Telegram / Discord / WhatsApp / WeChat / Slack / Feishu |
| **Self-Registering Tools** | 29 (69 files) | Terminal / Browser / Web / MCP / Delegate |
| **Built-in Skills** | 25 | Creative / Dev / Research / MLOps |
| **Execution Backends** | 7 | Local / Docker / Modal / SSH / Singularity |

---

## 📚 Series Structure

### Core Architecture & Subsystems (01-17)

| Chapter | Focus Area |
|:-------:|-----------|
| [01](/en/chapters/01-project-overview) | Project Overview & Design Philosophy |
| [02](/en/chapters/02-architecture) | System Architecture Overview |
| [03](/en/chapters/03-agent-engine) | Core Agent Engine — AIAgent |
| [04](/en/chapters/04-model-providers) | LLM Provider System |
| [05](/en/chapters/05-tool-system) | Tool System |
| [06](/en/chapters/06-gateway-system) | Gateway Messaging Platform |
| [07](/en/chapters/07-frontend-interfaces) | Frontend Interfaces |
| [08](/en/chapters/08-state-persistence) | State & Persistence |
| [09](/en/chapters/09-context-memory) | Context & Memory Engine |
| [10](/en/chapters/10-security) | Security System |
| [11](/en/chapters/11-skills-self-improvement) | Skills & Self-Evolution |
| [12](/en/chapters/12-cron-scheduling) | Scheduled Tasks & Cron |
| [13](/en/chapters/13-rl-training) | RL Training Pipeline |
| [14](/en/chapters/14-configuration-system) | Configuration System |
| [15](/en/chapters/15-infrastructure-deployment) | Infrastructure & Deployment |
| [16](/en/chapters/16-plugin-system) | Plugin System |
| [17](/en/chapters/17-design-patterns) | Transferable Design Patterns |

### Supplementary Deep Analysis (18-20)

| Chapter | Focus Area |
|:-------:|-----------|
| [18](/en/chapters/18-tool-system-extended) | Tool System Extended |
| [19](/en/chapters/19-hermes-cli-subsystem) | Hermes CLI Subsystem |
| [20](/en/chapters/20-agent-support-modules) | Agent Support Modules |

---

## 🗺️ Reading Paths

Pick your path based on what you want to understand:

| Path | Sequence | Best For |
|------|----------|----------|
| 🔁 **Core Loop** | 02→03→04→05→18→08→09 | How the system works end-to-end |
| 🛡️ **Security & Compliance** | 10→03→05→18→06→14 | Defense, SSRF, credentials, audit |
| 📡 **Platform & Deploy** | 06→07→19→15→14→12 | Integrating & deploying to production |
| 🧬 **Self-Evolution** | 11→16→05→18→09→17 | Building custom skills & plugins |
| 🧪 **RL Research** | 13→03→09→08→17 | Training trajectories & SWE-bench |
| ⌨️ **CLI & Ops** | 19→07→20→14→15 | CLI dispatch, auth, diagnostics |
| ⚡ **Quick Overview** | 01→02→03→10→17 | 30-minute panorama for newcomers |

---

## 🏗️ Architecture at 30,000 Feet

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend Layer                           │
│   CLI REPL  ·  TUI (Ink)  ·  Web SPA (Vite)  ·  ACP Editor    │
├─────────────────────────────────────────────────────────────────┤
│                     Dispatch & CLI Layer                        │
│              hermes_cli  ·  config  ·  auth  ·  setup          │
├─────────────────────────────────────────────────────────────────┤
│                      Agent Engine Layer                         │
│          AIAgent.run()  ·  context  ·  memory  ·  skills      │
├─────────────────────────────────────────────────────────────────┤
│                    Tool & Storage Layer                         │
│    29 tools  ·  SessionDB  ·  7 backends  ·  PluginManager    │
├─────────────────────────────────────────────────────────────────┤
│              Gateway (parallel, independent)                    │
│    21+ adapters  ·  StreamConsumer  ·  DeliveryRouter          │
└─────────────────────────────────────────────────────────────────┤
│         28+ LLM Providers (failover cascade)                    │
│    OpenRouter  Anthropic  Gemini  Bedrock  Alibaba  ...        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔗 Resources

- **Source Code**: [Hermes Agent](https://github.com/smith-source/hermes_agent_deep_dive)
- **License**: MIT
- **Versions Covered**: v0.2 → v0.12 (4-year evolution)

---

> 💡 **Tip**: Use the search bar (top right) to find specific topics across all 20 chapters.