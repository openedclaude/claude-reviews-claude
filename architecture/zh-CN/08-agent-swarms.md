# 08 — Swarm 智能体：多智能体团队协调

> **范围**: `tools/TeamCreateTool/`、`tools/SendMessageTool/`、`tools/shared/spawnMultiAgent.ts`、`utils/swarm/`（~30 个文件，~6.8 千行）、`utils/teammateMailbox.ts`（1,184 行）
>
> **一句话概括**: Claude Code 如何在 tmux 窗格、iTerm2 分屏或进程内生成并行队友 —— 全部通过基于文件的邮箱系统配合锁文件并发控制进行协调。

---

## 架构概览

```mermaid
graph TB
    subgraph Leader["👑 团队领导"]
        TC["TeamCreate<br/>创建团队 + config.json"]
        SPAWN["spawnMultiAgent<br/>生成队友"]
        SEND["SendMessage<br/>私信 / 广播 / 关闭"]
        INBOX_POLL["useInboxPoller<br/>轮询消息"]
    end

    subgraph Backend["🖥️ 后端"]
        TMUX["TmuxBackend<br/>分屏 / 独立窗口"]
        ITERM["ITermBackend<br/>原生 iTerm2 窗格"]
        INPROC["InProcessBackend<br/>同进程，独立查询循环"]
    end

    subgraph FS["📁 文件系统 (~/.claude/teams/)"]
        CONFIG["config.json<br/>团队清单"]
        MBOX["inboxes/{name}.json<br/>邮箱（加锁）"]
        TASKS["tasks/{team}/"]
    end

    subgraph Workers["🔧 队友"]
        W1["teammate-1<br/>researcher"]
        W2["teammate-2<br/>test-runner"]
        W3["teammate-3<br/>implementer"]
    end

    TC --> CONFIG
    SPAWN --> Backend
    TMUX --> W1
    ITERM --> W2
    INPROC --> W3
    SEND --> MBOX
    W1 --> MBOX
    W2 --> MBOX
    W3 --> MBOX
    INBOX_POLL --> MBOX
```

---

## 1. 团队生命周期

### 创建团队

`TeamCreateTool` 初始化团队基础设施：

1. 生成唯一团队名（冲突时使用随机词组合）
2. 创建团队领导条目，确定性 agent ID：`team-lead@{teamName}`
3. 将 `config.json` 写入 `~/.claude/teams/{team-name}/`
4. 注册会话清理（退出时自动删除，除非已显式删除）
5. 重置任务列表目录，从编号 1 开始

### 生成队友

`spawnMultiAgent.ts`（1,094 行）处理完整的生成流程：

1. **解析模型**：`inherit` → 领导的模型；`undefined` → 硬编码回退
2. **生成唯一名称**：检查现有成员，追加 `-2`、`-3` 等
3. **检测后端**：tmux > iTerm2 > 进程内（见 §3）
4. **创建窗格/进程**：后端特定的生成逻辑
5. **构建 CLI 参数**：传播 `--agent-id`、`--team-name`、`--agent-color`、`--permission-mode`
6. **注册到团队文件**：将成员条目添加到 `config.json`
7. **发送初始消息**：将提示词写入队友的邮箱
8. **注册后台任务**：用于 UI 任务标识显示

---

## 2. 邮箱系统

智能体间通信的骨干是**基于文件的邮箱**，配合锁文件并发控制。

### 目录结构

```
~/.claude/teams/{team-name}/
├── config.json              # 团队清单
└── inboxes/
    ├── team-lead.json       # 领导的收件箱
    ├── researcher.json      # 队友收件箱
    └── test-runner.json     # 队友收件箱
```

### 消息类型

| 类型 | 方向 | 用途 |
|------|------|------|
| 纯文本私信 | 任意 → 任意 | 直接消息 |
| 广播（`to: "*"`） | 领导 → 全体 | 团队公告 |
| `idle_notification` | 工人 → 领导 | "我完成了/被阻塞了/失败了" |
| `permission_request` | 工人 → 领导 | 工具权限委托 |
| `permission_response` | 领导 → 工人 | 权限授予/拒绝 |
| `sandbox_permission_request` | 工人 → 领导 | 网络访问审批 |
| `plan_approval_request` | 工人 → 领导 | 计划审查（planModeRequired） |
| `shutdown_request` | 领导 → 工人 | 优雅关闭 |
| `shutdown_approved/rejected` | 工人 → 领导 | 关闭确认 |

### 并发控制

多个 Claude 实例可以并发写入 —— 锁文件通过指数退避重试（10 次重试，5-100ms 超时）序列化访问。

---

## 3. 后端检测与执行

三种后端决定队友的物理运行方式：

| 特性 | Tmux | iTerm2 | 进程内 |
|------|------|--------|--------|
| 隔离性 | 独立进程 | 独立进程 | 同进程，独立查询循环 |
| UI 可见性 | 带彩色边框的窗格 | 原生 iTerm2 窗格 | 后台任务标识 |
| 前置条件 | tmux 已安装 | `it2` CLI 已安装 | 无 |
| 非交互模式（`-p`） | ❌ | ❌ | ✅（强制） |
| Socket 隔离 | PID 作用域：`claude-swarm-{pid}` | N/A | N/A |

检测优先级：**tmux 内部 → iTerm2 原生 → tmux 外部 → 进程内回退**。

---

## 4. 权限委托

队友没有交互终端 —— 它们将权限决策委托给领导：

1. 工人需要权限 → 创建 `permission_request` 消息
2. 写入领导的邮箱
3. 领导的 `useInboxPoller` 拾取请求
4. 领导向用户显示权限提示
5. 领导发送 `permission_response` 回工人的邮箱
6. 工人轮询收件箱，获取响应，继续或中止

### Plan Mode Required

带 `plan_mode_required: true` 生成的队友：
- 必须进入 plan 模式并创建计划
- 计划作为 `plan_approval_request` 发送给领导
- 领导审核后发送 `plan_approval_response`
- 批准时，领导的权限模式被继承（`plan` 映射为 `default`）

---

## 5. 智能体身份系统

```
格式：{name}@{teamName}
示例：researcher@my-project、team-lead@my-project
```

### CLI 标志传播

生成队友时，领导传播以下标志：

| 标志 | 条件 | 用途 |
|------|------|------|
| `--dangerously-skip-permissions` | bypass 模式 + 非 planModeRequired | 继承权限绕过 |
| `--permission-mode auto` | auto 模式 | 继承分类器 |
| `--model {model}` | 显式模型覆盖 | 使用领导的模型 |
| `--settings {path}` | CLI 设置路径 | 共享设置 |
| `--plugin-dir {dir}` | 内联插件 | 共享插件 |
| `--parent-session-id {id}` | 始终 | 血统追踪 |

---

## 6. 团队清理

### 优雅关闭

`SendMessage(type: shutdown_request)` → 队友回应 `shutdown_approved/rejected`：
- **批准**：进程内队友中止查询循环；窗格队友调用 `gracefulShutdown(0)`
- **拒绝**：队友提供原因，继续工作

### 会话清理

`cleanupSessionTeams()` 在领导退出时运行：
1. 终止孤立的队友窗格
2. 删除团队目录：`~/.claude/teams/{team-name}/`
3. 删除任务目录：`~/.claude/tasks/{team-name}/`
4. 销毁为隔离队友创建的 git worktree

---

## 7. 设计洞察

### 为什么用文件邮箱？

邮箱系统使用纯 JSON 文件 + 锁文件，而非 IPC、WebSocket 或共享内存：
- **跨进程**：tmux 窗格是独立进程，没有共享内存
- **崩溃安全**：消息持久化在磁盘上，即使队友崩溃也不丢失
- **可调试**：`cat ~/.claude/teams/my-team/inboxes/researcher.json`
- **简单**：无守护进程，无端口分配，无服务发现

### 一个领导，多个工人

架构强制执行严格的领导-工人层级：
- 每个领导会话只能有一个团队
- 工人不能创建团队或批准自己的计划
- 关闭始终由领导发起，工人确认
- 权限委托始终是 工人 → 领导 → 工人

---

## 组件总结

| 组件 | 行数 | 角色 |
|------|------|------|
| `spawnMultiAgent.ts` | 1,094 | 统一的队友生成逻辑 |
| `teammateMailbox.ts` | 1,184 | 基于文件的邮箱 + 锁文件并发 |
| `teamHelpers.ts` | 684 | 团队文件 CRUD、清理、worktree 管理 |
| `SendMessageTool.ts` | 918 | 私信、广播、关闭、计划审批 |
| `TeamCreateTool.ts` | 241 | 团队初始化 |
| `backends/registry.ts` | 465 | 后端检测：tmux > iTerm2 > 进程内 |
| `teammateLayoutManager.ts` | ~400 | 窗格创建、颜色分配、边框状态 |

Swarm 系统是 Claude Code 操作最复杂的功能 —— 它将进程管理、基于文件的 IPC、终端多路复用和分布式权限委托融合为一个多智能体框架。文件邮箱设计优先考虑简单性和可调试性而非性能，这在"分布式系统"实际上是共享同一文件系统的多个 AI 智能体时是正确的权衡。

---

**上一篇**: [← 07 — 权限流水线](07-permission-pipeline.md)
