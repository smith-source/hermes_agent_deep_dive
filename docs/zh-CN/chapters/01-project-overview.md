# 01 — 项目全景与设计哲学: 从 v0.1 到 v0.12 的自进化智能体平台

[上一章](/zh-CN/chapters/overview) | [返回目录](/zh-CN/chapters/overview)

---

## 源码位置与关键文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `README.md` | 182 | 项目主入口文档,设计哲学与快速安装指引 |
| `RELEASE_v0.2.0.md` | 383 | 第一个公开版本,216 PR,63 贡献者 |
| `RELEASE_v0.3.0.md` | 378 | 流式传输+插件架构+Anthropic 原生提供者 |
| `RELEASE_v0.4.0.md` | ~250 | 持久 Shell+ACP+语音模式 |
| `RELEASE_v0.5.0.md` | 349 | 供应链加固+HuggingFace+Telegram Topics |
| `RELEASE_v0.6.0.md` | 250 | 多实例 Profiles+MCP Server+Feishu/WeCom |
| `RELEASE_v0.7.0.md` | ~350 | 可插拔内存+凭证池轮换+Camofox |
| `RELEASE_v0.8.0.md` | ~350 | MiMo v2 Pro+Live Model Switch+Approval Buttons |
| `RELEASE_v0.9.0.md` | ~400 | Web Dashboard+iMessage+WeChat+Android |
| `RELEASE_v0.10.0.md` | 28 | Nous Tool Gateway |
| `RELEASE_v0.11.0.md` | ~400 | Ink TUI 重写+Transport ABC+AWS Bedrock+QQBot |
| `RELEASE_v0.12.0.md` | ~400 | Autonomous Curator+LM Studio+Teams+Spotify |

**一句话概要:** Hermes Agent 是 Nous Research 打造的"自进化智能体"——内置学习闭环,从经验创建技能、使用中自我改进、跨会话持久记忆、搜索历史对话、构建用户模型;在任何基础设施上运行,安全优先。

---

## 架构概览

```
                         ┌─────────────────────────────────────────┐
                         │        Hermes Agent v0.12 Ecosystem     │
                         ├─────────────────────────────────────────┤
                         │                                         │
  ┌──────────┐           │  ┌─────────────┐   ┌──────────────────┐ │
  │ 21+ 平台   │◄─────────┤  │  Gateway    │   │   CLI / TUI      │ │
  │ 消息网关  │           │  │  (15K 行)   │   │  (10.8K 行)      │ │
  ├──────────┤           │  └─────────────┘   └──────────────────┘ │
  │Telegram  │           │         │                  │            │
  │Discord   │           │         ▼                  ▼            │
  │Slack     │           │  ┌───────────────────────────────────┐ │
  │WhatsApp  │           │  │     AIAgent Engine (14.4K 行)     │ │
  │Signal    │           │  │  run_agent.py — 核心对话循环      │ │
  │Matrix    │           │  │  ┌────────┐ ┌────────┐ ┌────────┐ │ │
  │Feishu    │           │  │  │Transport│ │Context │ │Memory  │ │ │
  │WeCom     │           │  │  │  Layer  │ │Compress│ │Plugins │ │ │
  │WeChat    │           │  │  └────────┘ └────────┘ └────────┘ │ │
  │QQBot     │           │  └───────────────────────────────────┘ │
  │iMessage  │           │         │                  │            │
  │Yuanbao   │           │         ▼                  ▼            │
  │DingTalk  │           │  ┌────────────┐    ┌──────────────────┐ │
  │Email     │           │  │ Tool System │    │ Skills Ecosystem │ │
  │SMS       │           │  │ (537+ 行)  │    │ (70+ skills)     │ │
  │HA/Webhook│           │  │ 50+ tools  │    │ Curator (自动)   │ │
  │Teams     │           │  └────────────┘    └──────────────────┘ │
  └──────────┘           │                                         │
                         │  ┌─────────────────────────────────────┐ │
  ┌──────────┐           │  │     28+ Inference Providers         │ │
  │ 7 终端    │           │  │ NousPortal│OpenRouter│Anthropic    │ │
  │ 后端     │           │  │ OpenAI│Bedrock│xAI│Gemini│HuggingFace│ │
  │Local     │           │  │ Ollama│LMStudio│Arcee│Step│NIM│Vercel│ │
  │SSH       │           │  │ CodexOAuth│Kimi│MiniMax│Alibaba│GLM │ │
  │Docker    │           │  │ GMI│Azure│Tokenhub│Lemonade│Fireworks│ │
  │Modal     │           │  └─────────────────────────────────────┘ │
  │Daytona   │           │                                         │
  │Singularity│           │  ┌─────────────────────────────────────┐ │
  │Vercel    │           │  │  Storage Layer                       │ │
  │Sandbox   │           │  │  SessionDB│Config│Memory│SQLite WAL │ │
  └──────────┘           │  └─────────────────────────────────────┘ │
                         └─────────────────────────────────────────┘
```

---

## TL;DR

Hermes Agent 是由 Nous Research 开发的开源 MIT 许可自进化 AI 智能体平台,从 v0.1 内部原型到 v0.12 (2026.4.30) 已迭代 11 个正式版本,覆盖 28+ 推理提供者、21+ 消息平台、29 自注册工具（69 工具文件）与 70+ 技能。其核心设计哲学是三层闭环:自改进 (经验→技能→使用→优化)、随处运行 ($5 VPS 到 GPU 集群到 serverless)、安全优先 (命令审批、PII 消除、SSRF 防护、供应链审计)。代码规模达 70 万行 Python + 7 万行 TypeScript,核心模块 `run_agent.py` 单文件 14,404 行承载完整智能体引擎。

---

## 1. 项目历史: v0.2 到 v0.12 的演进

### 1.1 v0.2.0 — 诞生期 (2026.3.12)

> "First tagged release since v0.1.0. In just over two weeks, 216 merged PRs from 63 contributors, resolving 119 issues."

v0.2 是 Hermes Agent 的公开首秀,从一个内部项目在两周内爆发为全功能智能体平台。核心特性包括:

- **多平台消息网关** — Telegram, Discord, Slack, WhatsApp, Signal, Email, Home Assistant 七大平台统一管理
- **MCP 客户端** — stdio + HTTP 传输,重连,资源/提示发现,采样支持
- **技能生态系统** — 70+ 内置和可选技能,15+ 分类,Skills Hub 社区发现
- **集中式提供者路由** — `call_llm()`/`async_call_llm()` 统一 API 替代分散逻辑
- **ACP 服务器** — VS Code, Zed, JetBrains 编辑器集成
- **CLI 皮肤引擎** — 7 内置皮肤 + 自定义 YAML 皮肤
- **Git Worktree 隔离** — `hermes -w` 在 worktree 中启动隔离会话
- **文件系统检查点与回滚** — `/rollback` 恢复到快照
- **3,289 测试** — 从零到全覆盖测试套件

源码位置: `RELEASE_v0.2.0.md:1-383`

### 1.2 v0.3.0 — 流式与插件期 (2026.3.17)

> "The streaming, plugins, and provider release — unified real-time token delivery, first-class plugin architecture, rebuilt provider system."

关键进化:

- **统一流式基础设施** — CLI 和所有网关平台实时逐 token 交付
- **原生 Anthropic 提供者** — Claude Code 凭证自动发现,OAuth PKCE,原生提示缓存
- **智能审批 + /stop 命令** — Codex 启发的审批系统,记住安全偏好
- **Honcho 内存集成** — 异步写入,可配置召回模式,多用户隔离
- **语音模式** — CLI 推送说话,Telegram/Discord 语音笔记,Whisper 转录
- **并发工具执行** — ThreadPoolExecutor 并行执行独立工具调用
- **PII 消除** — `privacy.redact_pii` 自动 scrub 个人信息
- **`/browser connect` via CDP** — Chrome DevTools Protocol 连接活浏览器

源码位置: `RELEASE_v0.3.0.md:1-378`

### 1.3 v0.4.0 — 持久 Shell 与 RL 期 (2026.3.20~22)

关键进化:

- **持久 Shell 模式** — 本地和 SSH 后端跨工具调用维持 shell 状态
- **Agentic OPD** — 新 RL 训练环境,策略蒸馏
- **Tirith 预执行扫描** — 静态分析终端命令安全

### 1.4 v0.5.0 — 加固期 (2026.3.28)

> "The hardening release — Hugging Face provider, /model command overhaul, supply chain audit."

关键进化:

- **HuggingFace 一级推理提供者** — HF Inference API 集成,curated agentic model picker
- **Telegram Private Chat Topics** — 项目级对话,每个 Topic 绑定功能技能
- **原生 Modal SDK** — 替换 swe-rex,消除隧道依赖
- **插件生命周期钩子** — `pre_llm_call`, `post_llm_call`, `on_session_start`, `on_session_end`
- **供应链加固** — 移除 compromised `litellm`,锁定版本范围,hash 校验锁文件
- **Anthropic 输出限制修复** — 每模型原生输出限制替代硬编码 16K `max_tokens`

源码位置: `RELEASE_v0.5.0.md:1-349`

### 1.5 v0.6.0 — 多实例期 (2026.3.30)

> "The multi-instance release — Profiles, MCP server mode, Docker container, fallback provider chains."

关键进化:

- **Profiles 多实例** — 同一安装运行多个隔离 Hermes 实例,各自配置/内存/会话/技能/网关
- **MCP Server 模式** — `hermes mcp serve` 向 MCP 客户端暴露对话和会话
- **Docker 容器** — 官方 Dockerfile,CLI 和网关双模式
- **有序回退提供者链** — 主提供者故障时自动切换链中的下一个
- **飞书/Lark 平台** — 事件订阅,消息卡片,群聊,交互回调
- **企业微信 (WeCom)** — 文本/图像/语音消息,群聊,回调验证
- **Slack 多工作区 OAuth** — 单网关连接多个 Slack workspace

源码位置: `RELEASE_v0.6.0.md:1-250`

### 1.6 v0.7.0 — 韧性期 (2026.4.3)

> "The resilience release — pluggable memory, credential pool rotation, Camofox anti-detection, inline diff previews."

关键进化:

- **可插拔内存提供者接口** — ABC 插件系统,Honcho 恢复完整集成
- **同提供者凭证池** — 多 API key 自动轮换,`least_used` 策略,401 失败自动切换
- **Camofox 反检测浏览器** — 隐匿浏览,VNC URL 发现,SSRF 绕过
- **内联 Diff 预览** — 文件写入和 patch 操作显示行级差异
- **Secret 渗出阻止** — 浏览器 URL 和 LLM 响应扫描秘密模式

源码位置: `RELEASE_v0.7.0.md:1-350`

### 1.7 v0.8.0 — 智能期 (2026.4.8)

> "The intelligence release — background task auto-notifications, free MiMo v2 Pro, live model switching."

关键进化:

- **后台任务自动通知** — `notify_on_complete` 自动通知智能体任务完成
- **免费 MiMo v2 Pro** — Nous Portal 免费层辅助任务 (压缩/视觉/摘要)
- **Live Model Switching** — `/model` 在任何平台中途切换模型和提供者
- **自优化 GPT/Codex 工具使用指导** — 自动行为基准测试诊断和修补 5 种故障模式
- **Google AI Studio 原生提供者** — Gemini 直接 API 访问
- **不活跃超时** — 基于实际工具活动而非挂钟时间
- **审批按钮** — Slack 和 Telegram 原生按钮审批危险命令
- **MCP OAuth 2.1 PKCE + OSV 恶意软件扫描**

源码位置: `RELEASE_v0.8.0.md:1-350`

### 1.8 v0.9.0 — 全平台期 (2026.4.13)

> "The everywhere release — mobile, iMessage, WeChat, Fast Mode, web dashboard, 16 platforms."

关键进化:

- **本地 Web Dashboard** — 浏览器管理界面,配置/会话/技能/网关一站式
- **Fast Mode (`/fast`)** — OpenAI Priority Processing + Anthropic fast tier
- **iMessage via BlueBubbles** — Apple 消息生态系统集成
- **微信原生支持** — iLink Bot API,流式光标,媒体上传
- **Termux/Android 支持** — 在 Android 上原生运行
- **后台进程监控** — `watch_patterns` 实时输出模式匹配通知
- **可插拔上下文引擎** — 自定义上下文管理插件
- **统一代理支持** — SOCKS 代理和系统代理自动检测
- **16 支持平台** — BlueBubbles 和 WeChat 加入
- **`hermes backup` & `hermes import`** — 配置/会话/技能/内存备份恢复

源码位置: `RELEASE_v0.9.0.md:1-400`

### 1.9 v0.10.0 — Tool Gateway 期 (2026.4.16)

> "Paid Nous Portal subscribers can now use web search, image generation, TTS, and browser automation through their existing subscription."

关键进化:

- **Nous Tool Gateway** — 付费订阅者无需额外 API key,直接使用 Firecrawl/FAL/OpenAI TTS/Browser Use

源码位置: `RELEASE_v0.10.0.md:1-28`

### 1.10 v0.11.0 — Interface 期 (2026.4.23)

> "The Interface release — Ink TUI rewrite, Transport ABC, AWS Bedrock, 17th platform (QQBot)."

关键进化:

- **Ink-based TUI** — React/Ink 重写交互式 CLI,JSON-RPC Python 后端
- **Transport ABC** — 格式转换和 HTTP 传输从 `run_agent.py` 抽取到 `agent/transports/` 层
- **原生 AWS Bedrock** — Converse API 运输层
- **五个新推理路径** — NVIDIA NIM, Arcee AI, Step Plan, Gemini CLI OAuth, Vercel ai-gateway
- **GPT-5.5 over Codex OAuth** — ChatGPT Codex OAuth 推理
- **QQBot — 第 17 个平台** — QQ Official API v2,QR 扫码配置
- **插件面扩展** — slash commands, tool dispatch, tool veto, result rewrite, terminal output transform, dashboard tabs
- **`/steer` — 中途智能体引导** — 注入提示,不中断 turn 或破坏 prompt cache

源码位置: `RELEASE_v0.11.0.md:1-400`

### 1.11 v0.12.0 — Curator 期 (2026.4.30)

> "The Curator release — Hermes Agent now maintains itself. Autonomous background Curator grades, prunes, and consolidates your skill library."

关键进化:

- **自治 Curator** — 后台智能体 7 天周期自动分级/合并/修剪技能库
- **自改进环大幅升级** — 基于类的评审提示,活跃更新偏好,实时运行时继承
- **ComfyUI v5 和 TouchDesigner-MCP** — 从可选提升为内置默认
- **LM Studio 一级提供者** — 从自定义端点别名升级为原生提供者
- **四个新推理提供者** — GMI Cloud, Azure AI Foundry, MiniMax OAuth, Tencent Tokenhub
- **可插拔网关平台 + Microsoft Teams** — 网关成为插件主机,Teams 是第一个插件平台
- **腾讯元宝 — 第 18 个消息平台**
- **Spotify 原生工具 + 技能** — PKCE OAuth,7 工具,交互式配置向导
- **Google Meet 插件** — 加入通话/转录/说话/跟进
- **`hermes -z` one-shot 模式** — 非交互单查询
- **TUI 冷启动 ~57% 加速** — lazy agent init, mtime-cached config, memoized tool definitions
- **秘密消除默认关闭** — 修补 patch-corruption 长期问题

源码位置: `RELEASE_v0.12.0.md:1-400`

---

## 2. 设计哲学

### 2.1 Self-Improving (自进化闭环)

Hermes Agent 的核心差异在于内置学习闭环,而非简单的"对话+工具"模式:

```text
┌──────────────────────────────────────────────────────────────┐
│                    Self-Improving Loop                       │
│                                                              │
│  经验 ──► 技能创建 ──► 使用加载 ──► 活跃改进 ──► Curator 修剪 │
│    │           │            │            │            │      │
│    ▼           ▼            ▼            ▼            ▼      │
│  记忆持久  │ agentskills.io │ 技能  │ rubric    │ 合并/  │ │
│  跨会话    │ 开放标准       │ 查看  │ grading   │ 修剪   │ │
│            │                │        │           │        │ │
│  ──◄── 搜索历史对话 (FTS5 + LLM 摘要) ──◄── Honcho 用户模型 ──│
└──────────────────────────────────────────────────────────────┘
```

源码位置: `README.md:22` (闭环描述), `RELEASE_v0.12.0.md:12-14` (Curator + 自改进环升级)

### 2.2 Runs Anywhere (随处运行)

```text
┌─────────────┐  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐
│ $5 VPS      │  │ GPU 集群     │  │ Serverless   │  │ Android/Termux│
│ (local/SSH) │  │ (Modal/Daytona)│ │ (Vercel/Modal)│ │ (Termux 适配) │
└─────────────┘  └─────────────┘  └──────────────┘  └───────────────┘
        │                │                │                 │
        ▼                ▼                ▼                 ▼
┌─────────────────────────────────────────────────────────────┐
│  7 Terminal Backends:                                       │
│  local │ Docker │ SSH │ Singularity │ Modal │ Daytona │ Vercel│
│  Sandbox                                                    │
│                                                             │
│  Modal/Daytona serverless persistence:                      │
│  休眠时几乎零成本,唤醒时按需付费                              │
└─────────────────────────────────────────────────────────────┘
```

源码位置: `README.md:25` (随处运行描述)

### 2.3 Security-First (安全优先)

Hermes Agent 在每个版本都进行了深层安全加固:

| 版本 | 安全特性 |
|------|----------|
| v0.2 | Path traversal 修复,Shell 注入防护,危险命令检测,0600/0700 文件权限,原子写入 |
| v0.3 | PII 消除,Tirith 预执行扫描,subprocess 环境变量剥离,Docker cwd 显式 opt-in |
| v0.5 | SSRF 保护,zip-slip 防护,shell 注入 `_expand_path`,供应链审计 CI |
| v0.7 | Secret 渗出阻止 (URL/base64/prompt injection),credential 目录扩展 |
| v0.8 | MCP OAuth 2.1 PKCE,OSV 恶意软件扫描,SSRF 重组防护,tar traversal |
| v0.9 | Path traversal in checkpoint,shell injection neutralization,Twilio webhook 签名验证,git argument injection |
| v0.11 | SSRF in browser_navigate,restriction of subagent toolsets |
| v0.12 | Secret redaction 默认关闭 (修补 patch-corruption),skill pinning 阻止 Curator 覆写 |

源码位置: 各 RELEASE 文件的 Security & Reliability 部分

---

## 3. 规模统计

### 3.1 代码规模

| 维度 | 数值 |
|------|------|
| Python 总行数 | ~704K |
| TypeScript 总行数 | ~73K |
| `run_agent.py` (核心引擎) | 14,404 |
| `gateway/run.py` (消息网关) | 15,046 |
| `hermes_cli/main.py` (CLI 入口) | 10,876 |
| `hermes_cli/config.py` (配置系统) | 4,939 |
| `hermes_state.py` (会话数据库) | 2,669 |
| `tools/registry.py` (工具注册表) | 537 |

### 3.2 生态系统规模

| 维度 | 数值 | 来源版本 |
|------|------|----------|
| 推理提供者 | 28+ | v0.12 (LM Studio, GMI Cloud, Azure, Tokenhub 加入) |
| 消息平台 | 19+ | v0.12 (Yuanbao + Teams plugin) |
| 内置+可选技能 | 70+ | v0.2 起,持续增长 |
| 工具 | 50+ | README + 各版本 |
| 测试 | 3,289+ | v0.2 起,持续增长 |
| 社区贡献者 | 63+ (v0.2) → 213+ (v0.12) | 各版本 Contributors 统计 |
| 许可证 | MIT | `LICENSE` |

### 3.3 版本迭代规模

| 版本 | PR 数 | Issue 数 | 贡献者数 |
|------|-------|----------|----------|
| v0.2 | 216 | 119 | 63 |
| v0.3 | ~220+ | ~50+ | 15 |
| v0.5 | ~157+ | ~30+ | 5 |
| v0.6 | 95 | 16 | 4 |
| v0.7 | 168 | 46 | ~10 |
| v0.8 | 209 | 82 | ~15 |
| v0.9 | 269 | 167 | 24 |
| v0.11 | 761 | ~100+ | 29 (290 含 co-authors) |
| v0.12 | 550 | ~50+ | 213 |

---

## 4. 目标用户 (6 种角色)

| 角色 | 使用场景 | 关键特性 |
|------|----------|----------|
| **开发者** | 代码编写,调试,项目管理 | Git worktree 隔离,检查点/回滚,patch 工具,skill 自创建 |
| **研究人员** | RL 训练,轨迹生成,模型评估 | Atropos 环境,OPD 蒸馏,batch runner,trajectory compression |
| **运维/管理员** | 自动化运维,定时报告,监控 | Cron scheduler,background process monitor,`watch_patterns` |
| **AI 爱好者** | 智能体交互,技能探索,社区贡献 | Skills Hub,skin themes,TUI 交互,voice mode |
| **安全专业人士** | 渗透测试,漏洞分析,红队演练 | Tirith 预扫描,dangerous command 检测,SSRF 防护,secret redaction |
| **企业用户** | 多平台客服,知识库,内部自动化 | Profiles 多实例,credential pool,WeCom/Feishu/WeChat,approval buttons |

源码位置: `README.md:19-27` (功能表间接描述 6 种角色场景)

---

## 5. 关键里程碑对比

### 5.1 版本对比: 核心架构变迁

| 维度 | v0.2 | v0.5 | v0.8 | v0.11 | v0.12 |
|------|------|------|------|-------|-------|
| 提供者路由 | `call_llm()` 集中路由 | `/model` overhaul | Live model switching | Transport ABC (4 transports) | +LM Studio native |
| 内存系统 | MEMORY.md + Honcho | — | Supermemory plugin | — | Curator autonomous |
| 工具执行 | 顺序执行 | — | 并发执行 (v0.3+) | `/steer` mid-run | ComfyUI/TouchDesigner bundled |
| 消息平台 | 7 | +Feishu/WeCom | +Matrix Tier 1 | +QQBot (17th) | +Yuanbao/Teams (19th) |
| 终端后端 | 5 | +Modal native | +Vercel Sandbox | — | +Vercel Sandbox |
| CLI | curses TUI | — | Status bar | Ink/React TUI rewrite | ~57% cold start cut |

### 5.2 版本对比: 安全能力演进

| 安全能力 | v0.2 | v0.5 | v0.8 | v0.9 | v0.12 |
|----------|------|------|------|------|-------|
| 命令审批 | 基础检测 | — | Slack/Telegram buttons | git argument injection | — |
| PII 消除 | — | v0.3: `redact_pii` | — | — | 默认关闭 redaction |
| SSRF 防护 | — | browser/vision/web | — | Twilio webhook | — |
| 供应链 | — | 移除 litellm,hash lock | OSV scanning | — | — |
| Secret 保护 | 基础模式 | — | — | exfiltration blocking | patch-corruption 修复 |

### 5.3 版本对比: 平台覆盖演进

| 消息平台 | v0.2 | v0.6 | v0.9 | v0.11 | v0.12 |
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

## 6. License 与作者

| 维度 | 详情 |
|------|------|
| 许可证 | MIT — `LICENSE` |
| 作者 | [Nous Research](https://nousresearch.com) |
| 项目负责人 | @teknium1 |
| 文档站点 | [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/) |
| 社区 | [Discord](https://discord.gg/NousResearch), [Skills Hub](https://agentskills.io) |
| 仓库 | [github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) |

源码位置: `README.md:10` (License badge), `README.md:181` ("Built by Nous Research")

---

## 总结表

| 组件 | 行数 | 职责 |
|------|------|------|
| `run_agent.py` | 14,404 | 核心智能体引擎 (AIAgent 类,对话循环,工具调度,流式传输,故障转移) |
| `gateway/run.py` | 15,046 | 消息网关运行器 (GatewayRunner,21+ 平台适配器,会话缓存,生命周期) |
| `hermes_cli/main.py` | 10,876 | CLI 入口 (argparse,profile 覆盖,子命令路由,setup/model/tools/doctor) |
| `hermes_cli/config.py` | 4,939 | 配置管理 (config.yaml 解析,env 加载,提供者路由,DEFAULT_CONFIG) |
| `hermes_state.py` | 2,669 | 会话持久化 (SessionDB,SQLite WAL,FTS5 搜索,消息存储) |
| `tools/registry.py` | 537 | 工具注册表 (ToolRegistry 单例,ToolEntry,check_fn TTL 缓存,自发现) |
| 总 Python | ~704K | 全项目 Python 代码 |
| 总 TypeScript | ~73K | TUI (React/Ink) + Web Dashboard |
| RELEASE 文件 | 11 | v0.2 到 v0.12 版本发布说明 |

---

[下一章: 02 — 系统架构总览](/zh-CN/chapters/02-architecture) | [返回目录](/zh-CN/chapters/overview)