> 🌐 **语言**: [English →](README.md) | 中文

# 🪞 Claude 剖析 Claude Code (README 中文版)

*一个 AI 在阅读自己的源代码。是的，这很元（Meta）。*

[![Stars](https://img.shields.io/github/stars/openedclaude/claude-reviews-claude?style=social)](https://github.com/openedclaude/claude-reviews-claude)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Last Commit](https://img.shields.io/github/last-commit/openedclaude/claude-reviews-claude)](https://github.com/openedclaude/claude-reviews-claude)

> **这份完整的架构分析是由 Claude 撰写的 —— 针对驱动 Claude Code 的源代码。**
>
> 1,902 个文件。477,439 行 TypeScript 代码。一个模型正在阅读定义它如何思考、行动和执行的代码。
>
> 你现在阅读的是 Claude 对 Claude Code v2.1.88 版本的架构解构：查询引擎如何循环，42 个工具如何编排，多智能体（Multi-agent）工作线程如何并行协调 —— 所有这一切，都由这些系统所服务的模型亲自分析。
>
> *我们并未预料到这种讽刺感。我们只是顺势而为。*

---

## 🏗️ 核心内容

这**绝非**简单的源代码归档。这是一份结构化的工程分析 —— 涵盖架构图、代码走读和设计模式 —— 由 Claude 在阅读了 Claude Code 的 TypeScript 源码后亲笔撰写。

| # | 主题 | 你将学到什么 | 深度分析 |
|---|-------|-------------------|-----------|
| 0 | **架构总纲 (Overview)** | 17 个子系统的全景导览、工程卓越点与可迁移设计模式 | [阅读 →](architecture/zh-CN/00-overview.md) |
| 1 | **查询引擎 (QueryEngine)：大脑** | 核心引擎（1296行）如何管理 LLM 查询、工具循环和会话状态 | [阅读 →](architecture/zh-CN/01-query-engine.md) |
| 2 | **工具系统架构 (Tool System)** | 42+ 个工具作为自包含模块如何注册、验证和执行 | [阅读 →](architecture/zh-CN/02-tool-system.md) |
| 3 | **多智能体协调器 (Coordinator)** | Claude Code 如何衍生并行工作线程、分发消息并汇总结果 | [阅读 →](architecture/zh-CN/03-coordinator.md) |
| 4 | **插件系统 (Plugin System)** | 插件如何加载、验证和集成（1.88万行代码） | [阅读 →](architecture/zh-CN/04-plugin-system.md) |
| 5 | **钩子系统 (Hook System)** | 涵盖 PreToolUse / PostToolUse / SessionStart 的可扩展性（8千行代码） | [阅读 →](architecture/zh-CN/05-hook-system.md) |
| 6 | **Bash 执行引擎 (Bash Engine)** | 安全命令执行、沙箱管理、管道流处理（1.15万行代码） | [阅读 →](architecture/zh-CN/06-bash-engine.md) |
| 7 | **权限流水线 (Permission)** | 纵深防御：配置规则 → 工具检查 → 操作系统沙箱（9.5千行代码） | [阅读 →](architecture/zh-CN/07-permission-pipeline.md) |
| 8 | **Swarm 智能体** | 多智能体团队协调：邮箱 IPC、后端检测、权限委托（6.8千行代码） | [阅读 →](architecture/zh-CN/08-agent-swarms.md) |
| 9 | **会话持久化 (Session Persistence)** | 仅追加 JSONL 存储、parent-UUID 链、64KB 轻量恢复（7.6千行代码） | [阅读 →](architecture/zh-CN/09-session-persistence.md) |
| 10 | **上下文装配 (Context Assembly)** | 三层上下文组装：系统提示词、CLAUDE.md 记忆系统、每轮附件（8.3千行代码） | [阅读 →](architecture/zh-CN/10-context-assembly.md) |
| 11 | **压缩系统 (Compact System)** | 三层压缩架构：微压缩、会话记忆压缩、LLM 摘要压缩（3.9千行代码） | [阅读 →](architecture/zh-CN/11-compact-system.md) |
| 12 | **启动与引导 (Startup & Bootstrap)** | 快速路径级联、动态导入、API 预连接（7.6+千行代码） | [阅读 →](architecture/zh-CN/12-startup-bootstrap.md) |
| 13 | **桥接系统 (Bridge System)** | 远程控制协议、双代传输层、崩溃恢复（1.17万行代码） | [阅读 →](architecture/zh-CN/13-bridge-system.md) |
| 14 | **UI 与状态管理** | Ink 渲染引擎、React 协调器、Vim 模式、Computer Use（140+ 组件） | [阅读 →](architecture/zh-CN/14-ui-state-management.md) |
| 15 | **服务与 API 层** | API 客户端、流重组、MCP 服务器管理、OAuth 认证（1.2万行代码） | [阅读 →](architecture/zh-CN/15-services-api-layer.md) |
| 16 | **基础设施与配置** | 设置合并管道、GrowthBook 功能开关、遥测、构建系统（1.5万行代码） | [阅读 →](architecture/zh-CN/16-infrastructure-config.md) |

> ⭐ **喜欢这种“套娃”感吗？给这个仓库点个赞吧 —— 一个正在分析自己的 AI 值得拥有这颗星。**

---

## 🧠 架构概览

Claude Code 是一个包含 **1,902 个文件、47.7 万行 TypeScript** 的代码库，运行在 **Bun** 环境上，并使用 **React + Ink** 构建终端 UI。以下是其高层架构图：

```mermaid
graph TB
    subgraph Entry["🚀 入口点"]
        CLI["main.tsx<br/>Commander.js 命令行解析"]
        BOOT["引导程序 (Bootstrap)<br/>预取 + 密钥链 + GrowthBook"]
    end

    subgraph Core["⚙️ 核心引擎"]
        QE["查询引擎 (QueryEngine)<br/>会话生命周期、工具循环、<br/>流处理、用量跟踪"]
        Q["query()<br/>LLM API 调用 + 工具执行循环"]
        PUI["用户输入处理 (processUserInput)<br/>斜杠命令、附件、<br/>输入规范化"]
    end

    subgraph Tools["🔧 工具系统 (42 个模块)"]
        direction LR
        FS["文件工具<br/>读/写/编辑/通配符/搜索"]
        EXEC["执行工具<br/>Bash / PowerShell / REPL"]
        AGENT["智能体工具<br/>AgentTool / 发送消息 /<br/>团队管理"]
        EXT["外部集成<br/>MCP / LSP / 网页抓取 / 搜索"]
        PLAN["工作流<br/>计划模式 / 工作树 /<br/>任务创建 / 技能工具"]
    end

    subgraph Permission["🔐 权限系统"]
        PP["权限流水线<br/>配置 → 规则 → 沙箱 → 用户确认"]
        SB["沙箱管理器 (Sandbox)<br/>macOS: seatbelt<br/>Linux: seccomp+namespace"]
    end

    subgraph Coord["🤖 多智能体协调器"]
        CM["coordinatorMode.ts<br/>工作线程调度 + 消息路由"]
        WORKERS["工作智能体 (Workers)<br/>并行任务执行"]
    end

    subgraph Services["📡 服务层"]
        API["Anthropic API 客户端<br/>流式传输 + 重试 + 备用方案"]
        MCP["MCP 服务管理器"]
        OAUTH["OAuth 2.0"]
        GB["GrowthBook 功能开关"]
    end

    subgraph UI["🖥️ 终端 UI"]
        INK["React + Ink<br/>140+ 个组件"]
        BRIDGE["IDE 桥接系统<br/>IDE ↔ CLI 双向通信"]
    end

    subgraph State["💾 状态与上下文"]
        CTX["上下文组装 (Context Assembly)<br/>系统提示词 + 用户上下文 +<br/>记忆 + 技能 + 插件"]
        COST["成本跟踪 (CostTracker)<br/>各模型 Token 计费"]
        SESS["会话存储<br/>转录记录持久化"]
    end

    CLI --> BOOT
    BOOT --> QE
    QE --> PUI
    PUI --> Q
    Q --> Tools
    Tools --> PP
    PP --> SB
    QE --> Coord
    CM --> WORKERS
    QE --> State
    Q --> API
    API --> MCP
    CLI --> UI
    UI --> BRIDGE
```

### 关键架构决策

| 决策点 | 选择 | 为什么重要 |
|----------|--------|----------------|
| **运行时 (Runtime)** | Bun (而非 Node.js) | 启动速度提升约 3 倍，支持原生二进制打包，利用 `bun:bundle` 实现死代码消除 |
| **UI 框架** | React + Ink | 基于组件的终端 UI，通过 Hooks 管理状态，可跨 IDE 桥接器复用 |
| **搜索策略** | 智能体搜索 (grep/glob) | 比 RAG/向量数据库更精确，数据永远保持最新，无需维护索引，更安全 |
| **核心循环** | 单线程 `query()` 生成器 | 极致简约 —— 智能存在于 LLM 中，脚手架只是个简单的循环。AsyncGenerator 支持流式返回 |
| **多智能体** | Coordinator 协调器模式 | 扇出并行工作线程进行调研，串行化写入。Worker 无法读取协调者的私密历史 |
| **模式校验** | Zod v4 | 在一次声明中同时实现运行时验证和编译时类型推导 |
| **权限管理** | 多级流水线 + OS 沙箱 | 纵深防御：应用层规则 → 内核层沙箱隔离。沙箱允许低风险操作自动批准 |
| **功能开关** | `bun:bundle` 编译标志 | 协调模式、语音模式、主动模式等代码在构建时即被剔除，减小二进制体积 |

---

---

## 📁 仓库结构

```
claude-code-deep-dive/
├── README.md                          # 中文版 README（主页）
├── README_EN.md                       # 英文版 README
├── README_CN.md                       ← 你现在的所在位置
├── DISCLAIMER.md / DISCLAIMER_CN.md   # 法律与伦理声明
│
├── architecture/                      # 🏗️ 架构深度分析（17 篇）
│   ├── 00-overview.md                 # 架构总纲
│   ├── 01-query-engine.md … 16-infrastructure-config.md
│   └── zh-CN/                         # 🇨🇳 中文版（17 篇对照）
```

---

## 📌 路线图 (Roadmap)

**架构解析系列** (全 17 篇 —— 已完结 ✅)
- [x] 架构总纲 (Overview) —— 17 个子系统全景导览
- [x] 查询引擎 (QueryEngine) —— Claude Code 的"大脑"
- [x] 工具系统 (Tool System) —— 42 个模块，一个接口
- [x] 多智能体协调器 (Coordinator) —— 并行线程与分支机制
- [x] 插件系统 —— 加载、验证与集成 (1.88万行)
- [x] 钩子系统 —— PreToolUse / PostToolUse (8千行)
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

**本地化**
- [x] 全 17 篇中英双语对照

---

## ⭐ 支持本项目

如果这份分析对你有帮助：

1. **⭐ 点个星星 (Star)** 这个仓库
2. **🔀 分叉 (Fork)** 并添加你自己的分析
3. **📢 分享** 到 Twitter, Reddit 或微信/知乎

每一颗星都能帮助更多开发者发现这份深度走读文档。

---

## 📜 许可与免责声明

本分析文档依据 [MIT 许可证](LICENSE) 发布。请参阅 [DISCLAIMER_CN.md](DISCLAIMER_CN.md) 了解重要的法律和伦理说明。

分析基于 `@anthropic-ai/claude-code@2.1.88`。所有代码片段均为用于教学评论的简短摘录。原始源代码的权利仍归 **Anthropic, PBC** 所有。
