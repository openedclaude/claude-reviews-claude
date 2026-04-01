---
layout: home
hero:
  name: "Claude 眼中的老己"
  text: "当 AI 阅读自己的源代码"
  tagline: "17 篇架构深度分析，由 Claude 亲笔解构 Claude Code v2.1.88 — 1,902 个文件，47.7 万行 TypeScript。"
  actions:
    - theme: brand
      text: 开始阅读 →
      link: /zh-CN/overview
    - theme: alt
      text: English
      link: /
    - theme: alt
      text: GitHub ⭐
      link: https://github.com/openedclaude/claude-reviews-claude

features:
  - icon: ⚙️
    title: 查询引擎
    details: 12 步状态机驱动的核心 while(true) 工具循环 — Claude Code 的「大脑」。
    link: /zh-CN/chapters/01-query-engine
  - icon: 🔧
    title: 工具系统
    details: 42+ 个工具作为自包含模块 — Schema 驱动注册、验证和执行。
    link: /zh-CN/chapters/02-tool-system
  - icon: 🔐
    title: 权限流水线
    details: 7 层纵深防御 — 从配置规则到 AST 分析到操作系统沙箱。
    link: /zh-CN/chapters/07-permission-pipeline
  - icon: 🤖
    title: Swarm 智能体
    details: 多智能体协调 — 邮箱 IPC、后端检测、权限委托。
    link: /zh-CN/chapters/08-agent-swarms
  - icon: 📦
    title: 上下文装配
    details: 三层上下文组装 — 系统提示词、CLAUDE.md 记忆系统、每轮附件。
    link: /zh-CN/chapters/10-context-assembly
  - icon: 🖥️
    title: 终端 UI
    details: Fork Ink + React 19，Vim 模式，IDE 桥接 — 140+ 组件驱动 CLI 体验。
    link: /zh-CN/chapters/14-ui-state-management
---

<style>
:root {
  --vp-home-hero-name-color: transparent;
  --vp-home-hero-name-background: linear-gradient(135deg, #D97757, #E8A87C, #F0C4A8);
}
</style>
