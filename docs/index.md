---
layout: home
hero:
  name: "Claude Reviews Claude"
  text: "When AI Reads Its Own Source Code"
  tagline: "17 architecture deep dives written by Claude, dissecting Claude Code v2.1.88 — 1,902 files, 477K lines of TypeScript."
  actions:
    - theme: brand
      text: Start Reading →
      link: /overview
    - theme: alt
      text: 简体中文
      link: /zh-CN/
    - theme: alt
      text: GitHub ⭐
      link: https://github.com/openedclaude/claude-reviews-claude

features:
  - icon: ⚙️
    title: Query Engine
    details: The 12-step state machine that powers the core while(true) tool loop — the "brain" of Claude Code.
    link: /chapters/01-query-engine
  - icon: 🔧
    title: Tool System
    details: 42+ tools as self-contained modules — Schema-driven registration, validation, and execution.
    link: /chapters/02-tool-system
  - icon: 🔐
    title: Permission Pipeline
    details: 7-layer defense-in-depth — from config rules to AST analysis to OS sandboxing.
    link: /chapters/07-permission-pipeline
  - icon: 🤖
    title: Agent Swarms
    details: Multi-agent coordination — mailbox IPC, backend detection, and permission delegation.
    link: /chapters/08-agent-swarms
  - icon: 📦
    title: Context Assembly
    details: 3-layer context assembly — system prompts, CLAUDE.md memory, per-turn attachments.
    link: /chapters/10-context-assembly
  - icon: 🖥️
    title: Terminal UI
    details: Forked Ink + React 19, Vim mode, IDE bridge — 140+ components powering the CLI experience.
    link: /chapters/14-ui-state-management
---

<style>
:root {
  --vp-home-hero-name-color: transparent;
  --vp-home-hero-name-background: linear-gradient(135deg, #D97757, #E8A87C, #F0C4A8);
}

.VPHero .name {
  font-size: 56px !important;
  line-height: 1.1 !important;
}

.VPHero .text {
  font-size: 32px !important;
  line-height: 1.3 !important;
}

.VPHero .main {
  max-width: 700px;
}
</style>
