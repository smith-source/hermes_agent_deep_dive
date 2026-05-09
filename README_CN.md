# Hermes Agent Deep Dive

> **自进化 AI 代理的 20 篇深度分析系列** — 每个子系统逐层解剖，每个模块逐一剖析。

[![Docsify](https://img.shields.io/badge/Powered%20by-Docsify-42b883)](https://docsify.js.org/) [![Languages](https://img.shields.io/badge/Languages-CN%20%7C%20EN-blue)]() [![Chapters](https://img.shields.io/badge/Chapters-20-orange)]() [![License](https://img.shields.io/badge/License-MIT-green)]()

**Wiki**: <!-- TODO: 填入 GitHub Pages 链接 -->

[English](./README.md) | 中文

---

## 项目简介

Hermes Agent 是一个自进化的全能 AI 代理——从经验中创建技能、在使用中持续改进、可运行于任何环境。本项目对它的完整架构进行了 **20 篇深度分析**，覆盖 347,240 行 Python 核心引擎与 91,223 行 TypeScript 前端界面。

| 维度 | 规模 |
|------|------|
| Python 核心代码 | 347,240 行 |
| TypeScript 前端代码 | 91,223 行 |
| LLM 提供商 | 28+ |
| 消息平台适配器 | 21+ |
| 自注册工具 | 29（69 文件） |
| 内置技能 | 25 |
| 执行后端 | 7 |

---

## 系列目录

### 核心架构与子系统（01–17）

| # | 标题 |
|---|------|
| 01 | 项目全景与设计哲学 |
| 02 | 系统架构总览 |
| 03 | 核心代理引擎 AIAgent |
| 04 | LLM 提供商体系 |
| 05 | 工具体系 |
| 06 | Gateway 消息平台 |
| 07 | 前端交互界面 |
| 08 | 状态与持久化 |
| 09 | 上下文与记忆引擎 |
| 10 | 安全体系 |
| 11 | 技能与自进化系统 |
| 12 | 定时任务与调度 |
| 13 | RL 训练管线 |
| 14 | 配置系统 |
| 15 | 基础设施与部署 |
| 16 | 插件系统 |
| 17 | 可迁移设计模式 |

### 补充深度分析（18–20）

| # | 标题 |
|---|------|
| 18 | 工具体系扩展 |
| 19 | Hermes CLI 子系统 |
| 20 | 代理支撑模块 |

---

## 项目结构

```
hermes-agent-reviews/
├── docs/                   # Docsify 文档根目录
│   ├── index.html          # Docsify 配置（路由、搜索、插件）
│   ├── README.md           # 首页（简介、统计、阅读路径）
│   ├── _navbar.md          # 顶部导航（语言切换）
│   ├── zh-CN/
│   │   ├── README.md       # 中文首页
│   │   └── chapters/       # 21 个中文章节文件
│   └── en/
│       ├── README.md       # 英文首页
│       └── chapters/       # 21 个英文章节文件
├── .gitignore
├── README.md               # 英文说明（本项目主文件）
└── README_CN.md            # 中文说明
```

---

## 参与贡献

1. Fork 本仓库
2. 创建功能分支 (`git checkout -b feature/your-chapter`)
3. 提交修改
4. 推送到分支 (`git push origin feature/your-chapter`)
5. 创建 Pull Request

### 翻译规范

- 中文文件路径：`docs/zh-CN/chapters/XX-name.md`
- 英文文件路径：`docs/en/chapters/XX-name.md`
- 导航链接使用 Docsify 绝对路径：
  - 中文：`/zh-CN/chapters/XX-name`
  - 英文：`/en/chapters/XX-name`

---

## 许可证

[MIT](https://opensource.org/licenses/MIT)