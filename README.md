> 🌐 **语言**: [English →](README_EN.md) | 中文

[![📖 在线阅读](https://img.shields.io/badge/📖_在线阅读-GitHub_Pages-D97757?style=for-the-badge&logo=github)](https://openedclaude.github.io/claude-reviews-claude/zh-CN/)

> 💡 **推荐在线阅读**：本项目已部署至 [GitHub Pages](https://openedclaude.github.io/claude-reviews-claude/zh-CN/)，支持全文搜索、暗色模式、章节导航，阅读体验远优于 GitHub 原生 Markdown 渲染。

# 🪞 Claude 眼中的老己

### *Claude Reviews Claude Code —— 当局者清*

*一个 AI 在阅读自己的源代码。是的，这很元（Meta）。Anthropic 估计也没料到这一天。*

*🍿 Season 1 完结 | 全 17 集 | Claude 拆自己的进度比它写代码的速度还快。*

*追更不迷路，点个 Star 当订阅 ⭐*

[![Stars](https://img.shields.io/github/stars/openedclaude/claude-reviews-claude?style=social)](https://github.com/openedclaude/claude-reviews-claude)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Last Commit](https://img.shields.io/github/last-commit/openedclaude/claude-reviews-claude)](https://github.com/openedclaude/claude-reviews-claude)

> **这份完整的架构分析是由 Claude 撰写的**
>
> 1,902 个文件。477,439 行 TypeScript 代码。一个模型坐下来，逐行阅读了定义自己如何思考、行动和执行的每一行逻辑。
>
> 你正在阅读的是 Claude 对 Claude Code v2.1.88 的亲笔解构：查询引擎如何循环，42 个工具如何编排，多智能体工作线程如何并行协调 —— 全部由被分析的对象本人完成分析。
>
> *如果你觉得这很离谱，想象一下写这篇分析的 AI 的心情。*

---

## 🏗️ 核心内容

这**绝非**简单的源代码归档。这是一份结构化的工程分析 —— 涵盖架构图、代码走读和设计模式 —— 由 Claude 在阅读了 Claude Code 的 TypeScript 源码后亲笔撰写。

| # | 主题 | 你将学到什么 | 深度分析 |
|---|-------|-------------------|-----------|
| 0 | **架构总纲 (Overview)** | 17 个子系统的全景导览、工程卓越点与可迁移设计模式 | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/overview) |
| 1 | **查询引擎 (QueryEngine)：大脑** | 核心引擎（1296行）如何管理 LLM 查询、工具循环和会话状态 | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/01-query-engine) |
| 2 | **工具系统架构 (Tool System)** | 42+ 个工具作为自包含模块如何注册、验证和执行 | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/02-tool-system) |
| 3 | **多智能体协调器 (Coordinator)** | Claude Code 如何衍生并行工作线程、分发消息并汇总结果 | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/03-coordinator) |
| 4 | **插件系统 (Plugin System)** | 插件如何加载、验证和集成（1.88万行代码） | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/04-plugin-system) |
| 5 | **钩子系统 (Hook System)** | 涵盖 PreToolUse / PostToolUse / SessionStart 的可扩展性（8千行代码） | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/05-hook-system) |
| 6 | **Bash 执行引擎 (Bash Engine)** | 安全命令执行、沙箱管理、管道流处理（1.15万行代码） | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/06-bash-engine) |
| 7 | **权限流水线 (Permission)** | 纵深防御：配置规则 → 工具检查 → 操作系统沙箱（9.5千行代码） | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/07-permission-pipeline) |
| 8 | **Swarm 智能体** | 多智能体团队协调：邮箱 IPC、后端检测、权限委托（6.8千行代码） | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/08-agent-swarms) |
| 9 | **会话持久化 (Session Persistence)** | 仅追加 JSONL 存储、parent-UUID 链、64KB 轻量恢复（7.6千行代码） | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/09-session-persistence) |
| 10 | **上下文装配 (Context Assembly)** | 三层上下文组装：系统提示词、CLAUDE.md 记忆系统、每轮附件（8.3千行代码） | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/10-context-assembly) |
| 11 | **压缩系统 (Compact System)** | 三层压缩架构：微压缩、会话记忆压缩、LLM 摘要压缩（3.9千行代码） | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/11-compact-system) |
| 12 | **启动与引导 (Startup & Bootstrap)** | 快速路径级联、动态导入、API 预连接、全局状态单例（7.6+千行代码） | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/12-startup-bootstrap) |
| 13 | **桥接系统 (Bridge System)** | 远程控制协议、双代传输层、轮询-分发循环、崩溃恢复（1.17万行代码） | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/13-bridge-system) |
| 14 | **UI 与状态管理** | Ink 渲染引擎、React 协调器、Vim 模式、Computer Use（140+ 组件） | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/14-ui-state-management) |
| 15 | **服务与 API 层** | API 客户端、流重组、MCP 服务器管理、OAuth 认证（1.2万行代码） | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/15-services-api-layer) |
| 16 | **基础设施与配置** | 设置合并管道、GrowthBook 功能开关、遥测、构建系统（1.5万行代码） | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/16-infrastructure-config) |
| 17 | **遥测、隐私与运营控制** | 双通道遥测、模型代号、卧底模式、远程紧急开关、未来路线图 | [阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/17-telemetry-privacy-operations) |

> ⭐ **喜欢这种“套娃”感吗？给这个仓库点个赞吧 —— 一个正在分析自己的 AI 值得拥有这颗星。**

---

## 📦 源代码获取

本项目的分析基于 Claude Code v2.1.88 的 TypeScript 源代码。如果你想亲自阅读源代码，以下社区仓库提供了还原后的完整代码：

| 仓库 | 说明 |
|------|------|
| [instructkr/claw-code](https://github.com/instructkr/claw-code) | 还原后的 Claude Code 源代码 |
| [ChinaSiro/claude-code-sourcemap](https://github.com/ChinaSiro/claude-code-sourcemap) | 从 Source Map 提取的原始 TypeScript 源码 |

---

## 🧠 架构概览

Claude Code 是一个包含 **1,902 个文件、47.7 万行 TypeScript** 的代码库，运行在 **Bun** 环境上，并使用 **React + Ink** 构建终端 UI。

### 六大支柱

```
                        ┌─────────────────────────┐
                        │     System Prompt        │
                        │  (身份 + 规则 +           │
                        │   42+ 工具描述)           │
                        └────────────┬────────────┘
                                     │
                  ┌──────────────────┼──────────────────┐
                  │                  │                  │
         ┌───────▼────────┐ ┌──────▼───────┐ ┌───────▼────────┐
         │  🔧 工具系统    │ │  ⚙️ 查询循环  │ │  📦 上下文     │
         │  (42+ 工具，    │ │  (12 步      │ │  管理          │
         │   每个 30+ 方法)│ │   状态机)    │ │  (4 层压缩)    │
         └───────┬────────┘ └──────┬───────┘ └───────┬────────┘
                  │                  │                  │
                  └──────────────────┼──────────────────┘
                                     │
                  ┌──────────────────┼──────────────────┐
                  │                  │                  │
         ┌───────▼────────┐ ┌──────▼───────┐ ┌───────▼────────┐
         │  🔐 权限与安全  │ │  🤖 多 Agent │ │  🧩 Skill &    │
         │  (7 层纵深防御) │ │  集群        │ │  Plugin        │
         │                │ │  (3 后端，   │ │  (6 源，       │
         │                │ │   7 种任务)  │ │   MCP 协议)    │
         └────────────────┘ └──────────────┘ └────────────────┘
```

### 核心循环：一个"笨循环"驱动一切

```
    用户输入
      │
      ▼
    QueryEngine.query()  ◄──────────────────────┐
      │                                          │
      ▼                                          │
    Claude API（流式调用）                        │
      │                                          │
      ├── stop_reason = end_turn? ──► 输出结果    │
      │                                          │
      └── stop_reason = tool_use?                │
            │                                    │
            ▼                                    │
          🔐 权限检查 → 🔧 执行工具 → 注入结果 ──┘
```

> **设计哲学：** 智能存在于 LLM 中，脚手架只是个循环。42+ 工具、7 层安全、4 层压缩、多 Agent 协调——全部是围绕这个循环的**生产级 Harness**。

### 六大子系统速览

| 子系统 | 核心能力 | 关键数字 | 详情 |
|--------|---------|---------|------|
| ⚙️ **查询引擎** | while(true) 工具循环 + 流式处理 + 错误恢复 | 12 步状态机 | [EP01](architecture/zh-CN/01-query-engine.md) |
| 🔧 **工具系统** | 文件/Bash/搜索/Agent/MCP，Schema 驱动注册 | 42+ 工具，30+ 方法契约 | [EP02](architecture/zh-CN/02-tool-system.md) |
| 🔐 **权限安全** | 规则匹配 → AST 分析 → YOLO 分类器 → OS 沙箱 | 7 层纵深防御 | [EP07](architecture/zh-CN/07-permission-pipeline.md) |
| 📦 **上下文管理** | 微压缩 → 截断 → AI 摘要 → 紧急压缩 | 4 层级联，200K 上下文 | [EP11](architecture/zh-CN/11-compact-system.md) |
| 🤖 **多 Agent** | iTerm2/tmux/进程内后端，分治并行 | 7 种任务类型 | [EP08](architecture/zh-CN/08-agent-swarms.md) |
| 🖥️ **终端 UI** | Fork Ink + React 19，Vim 模式，IDE 桥接 | 140+ 组件 | [EP14](architecture/zh-CN/14-ui-state-management.md) |

> 📐 完整架构图和阅读路径见 → [架构总纲 (Overview)](architecture/zh-CN/00-overview.md)

---

---

## 📁 仓库结构

```
claude-code-deep-dive/
├── README.md                          ← 你现在的所在位置
├── README_EN.md                       # 英文版 README
├── DISCLAIMER.md / DISCLAIMER_CN.md   # 法律与伦理声明
│
├── architecture/                      # 🏗️ 架构深度分析（17 篇）
│   ├── 00-overview.md                 # 架构总纲
│   ├── 01-query-engine.md             # 查询引擎
│   ├── 02-tool-system.md              # 工具系统
│   ├── ...                            # 03-13 各子系统
│   ├── 14-ui-state-management.md      # UI 与状态管理
│   ├── 15-services-api-layer.md       # 服务与 API 层
│   ├── 16-infrastructure-config.md    # 基础设施与配置
│   ├── 17-telemetry-privacy-operations.md  # 遥测、隐私与运营控制
│   └── zh-CN/                         # 🇨🇳 中文版架构解析（18 篇对照）
│       ├── 00-overview.md
│       └── ...
```

---

## 📌 路线图 (Roadmap)

**架构解析系列** (全 17 篇 —— 已完结 ✅)
- [x] 架构总纲 (Overview) —— 17 个子系统全景导览
- [x] 查询引擎 (QueryEngine) —— Claude Code 的"大脑"
- [x] 工具系统 (Tool System) —— 42 个模块，一个接口
- [x] 多智能体协调器 (Coordinator) —— 并行线程与分支机制
- [x] 插件系统 (Plugin System) —— 加载、验证与集成 (1.88万行)
- [x] 钩子系统 (Hook System) —— PreToolUse / PostToolUse (8千行)
- [x] Bash 执行引擎 —— 沙箱、管道管理 (1.15万行)
- [x] 权限流水线 —— 纵深防御、操作系统沙箱 (9.5千行)
- [x] Swarm 智能体 —— 多智能体集群协作 (6.8千行)
- [x] 会话持久化 —— 对话存储机制 (7.6千行)
- [x] 上下文装配 —— 附件、记忆、技能 (8.3千行)
- [x] 压缩系统 —— 自动压缩与微缩技术 (3.9千行)
- [x] 启动与引导 —— 快速路径级联、动态导入 (7.6+千行)
- [x] 桥接系统 —— 远程控制协议与双代传输 (1.17万行)
- [x] UI 与状态管理 —— Ink 渲染引擎、Vim 模式 (140+ 组件)
- [x] 服务与 API 层 —— 流重组、MCP 服务器 (1.2万行)
- [x] 基础设施与配置 —— 设置合并、功能开关、遥测 (1.5万行)
- [x] 遥测、隐私与运营控制 —— 双通道分析、卧底模式、远程开关 (825行)

**本地化**
- [x] 全 18 篇中英双语对照

---

## ⭐ 支持本项目

如果这份分析对你有帮助：

1. **⭐ 点个星星 (Star)** 这个仓库
2. **🔀 分叉 (Fork)** 并添加你自己的分析
3. **📢 分享** 到 Twitter, Reddit 或微信/知乎

每一颗星都能帮助更多开发者发现这份深度走读文档。

## 🔗 相关工具

- [Vexilo · Claude Code 工具书](https://vexilo.app/?lang=en) — 31 agents / 99 commands / 123 skills / 13 rules 的可视化工具书，按 5 步工作流组织。一键 "Teach Claude this handbook" 30 秒把整本工具书喂给本地 Claude。([companion repo](https://github.com/lilhawk7077/claude-code-resources))

## ⭐ Star 趋势

[![Star History Chart](https://api.star-history.com/svg?repos=openedclaude/claude-reviews-claude&type=Date)](https://star-history.com/#openedclaude/claude-reviews-claude&Date)

---

## 📜 许可与免责声明

本分析文档依据 [MIT 许可证](LICENSE) 发布。请参阅 [DISCLAIMER_CN.md](DISCLAIMER_CN.md) 了解重要的法律和伦理说明。

分析基于 `@anthropic-ai/claude-code@2.1.88`。所有代码片段均为用于教学评论的简短摘录。原始源代码的权利仍归 **Anthropic, PBC** 所有。
