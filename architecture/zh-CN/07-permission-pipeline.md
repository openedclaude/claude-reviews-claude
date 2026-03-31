# 07 — 权限流水线：从规则到内核的纵深防御

> **范围**: `utils/permissions/` (24 个文件, ~320KB), `utils/settings/` (17 个文件, ~135KB), `utils/sandbox/` (2 个文件, ~37KB)
>
> **一句话概括**: Claude Code 如何决定一个工具调用的生死 —— 经过规则、分类器、钩子和操作系统沙箱的七步考验。

---

## 架构概览

```mermaid
graph TB
    subgraph Pipeline["🔐 权限流水线 (1487 行)"]
        S1A["1a. 拒绝规则<br/>(工具级)"]
        S1B["1b. 询问规则<br/>(工具级)"]
        S1C["1c. tool.checkPermissions()<br/>(内容级)"]
        S1D["1d. 工具拒绝"]
        S1E["1e. requiresUserInteraction()"]
        S1F["1f. 内容级询问规则"]
        S1G["1g. 安全检查<br/>(.git, .claude, shell 配置)"]
        S2A["2a. bypassPermissions 模式"]
        S2B["2b. alwaysAllow 规则<br/>(工具级)"]
        S3["3. passthrough → ask"]
    end

    subgraph PostPipeline["⚡ 后处理变换"]
        DONT["dontAsk 模式<br/>ask → deny"]
        AUTO["Auto 模式 (YOLO)<br/>AI 分类器决策"]
        HEADLESS["无头代理<br/>钩子 → 自动拒绝"]
    end

    subgraph Classifier["🤖 YOLO 分类器 (1496 行)"]
        FAST["acceptEdits 快速路径"]
        SAFE["安全工具允许列表"]
        XML["两阶段 XML 分类器<br/>阶段1: 快速 block yes/no<br/>阶段2: 思维链"]
    end

    subgraph Sandbox["🏗️ 操作系统沙箱"]
        SEAT["macOS: seatbelt"]
        BWRAP["Linux: bubblewrap + seccomp"]
    end

    S1A --> S1B --> S1C --> S1D --> S1E --> S1F --> S1G
    S1G --> S2A --> S2B --> S3
    S3 --> PostPipeline
    AUTO --> Classifier
    FAST --> |"允许"|AUTO
    SAFE --> |"允许"|AUTO
    XML --> |"阻止/允许"|AUTO
```

---

## 1. 七步考验

核心函数 `hasPermissionsToUseToolInner()` 实现了一个**严格有序**的权限评估流水线。每一步都可以短路整个链条：

### 步骤 1a：工具级拒绝规则

硬拒绝 —— 无法覆盖。规则来源包括：`userSettings`、`projectSettings`、`localSettings`、`policySettings`、`flagSettings`、`cliArg`、`command`、`session`。

### 步骤 1b：工具级询问规则

关键设计：当沙箱启用并配置了自动允许时，沙箱化的 Bash 命令可以**跳过**询问规则。非沙箱化命令（被排除的命令、`dangerouslyDisableSandbox`）仍然遵守规则。

### 步骤 1c：工具内容级权限检查

每个工具实现自己的 `checkPermissions()`。BashTool 检查子命令，EditTool 检查文件路径，WebFetch 验证域名。

### 步骤 1d–1g：安全护栏

| 步骤 | 检查 | 能否被绕过？ |
|------|------|------------|
| **1d** | 工具实现拒绝 | ❌ 不能 |
| **1e** | `requiresUserInteraction()` 返回 true | ❌ 不能 |
| **1f** | 内容级询问规则（如 `Bash(npm publish:*)`） | ❌ 不能 |
| **1g** | 安全检查（`.git/`、`.claude/`、`.vscode/`、shell 配置） | ❌ 不能 |

这四项检查是**绕过免疫**的 —— 即使在 `bypassPermissions` 模式下也会触发。

### 步骤 2a：绕过权限模式

如果当前处于 `bypassPermissions` 模式（或 plan 模式但原始模式是 bypass），直接允许。

### 步骤 2b：始终允许规则

支持 MCP 服务器级匹配：规则 `mcp__server1` 匹配 `mcp__server1__tool1`。

### 步骤 3：Passthrough → Ask

如果没有任何决策，默认询问用户。

---

## 2. 六种权限模式

| 模式 | `ask` 变为 | 安全检查 | 说明 |
|------|-----------|---------|------|
| `default` | 提示用户 | 提示 | 标准交互模式 |
| `plan` | 提示用户 | 提示 | 暂存前置模式以便恢复 |
| `acceptEdits` | 允许（仅文件编辑） | 提示 | 非编辑工具仍需提示 |
| `bypassPermissions` | 全部允许 | **仍然提示** | 可被 GrowthBook 门控或设置禁用 |
| `dontAsk` | **拒绝** | 提示 | 静默拒绝，模型看到拒绝消息 |
| `auto` | 分类器决策 | 提示 | 两阶段 XML 分类器，需门控 |

### 模式转换

`transitionPermissionMode()` 集中处理所有副作用：

- **进入 auto 模式**：剥离危险权限（`Bash(*)`、`python:*`、Agent 允许列表）—— 这些权限会绕过分类器
- **离开 auto 模式**：恢复被剥离的权限
- **进入 plan 模式**：保存前置模式以便恢复
- **离开 plan 模式**：恢复之前的模式

---

## 3. 规则来源与层级

规则从 **7 个来源**加载：

| 来源 | 文件 | 范围 |
|------|------|------|
| `userSettings` | `~/.claude/settings.json` | 用户全局 |
| `projectSettings` | `.claude/settings.json` | 项目级，提交到仓库 |
| `localSettings` | `.claude/settings.local.json` | 项目级，gitignore |
| `policySettings` | 企业 MDM 策略 | 组织范围 |
| `flagSettings` | GrowthBook 功能标志 | 远程配置 |
| `cliArg` | `--allowed-tools`、`--disallowed-tools` | 单次调用 |
| `session` | 运行时通过权限对话框添加 | 单次会话 |

### 规则格式

```
ToolName                  → 匹配整个工具
ToolName(content)         → 匹配带特定内容的工具
Bash(npm test:*)          → Bash 命令的前缀匹配
mcp__server1              → 匹配 MCP 服务器的所有工具
Agent(Explore)            → 匹配特定代理类型
```

---

## 4. YOLO 分类器（Auto 模式）

当模式为 `auto` 时，`ask` 决策经过**三级快速路径**系统后才到达 AI 分类器：

### 第一级：acceptEdits 快速路径

以 `acceptEdits` 模式重新运行 `tool.checkPermissions()`。如果允许，跳过分类器。例外：`AgentTool` 和 `REPLTool` 被排除 —— REPL 代码可能包含 VM 逃逸。

### 第二级：安全工具允许列表

列入允许名单的工具完全跳过分类器。

### 第三级：两阶段 XML 分类器

分类器使用单独的 LLM API 调用（`sideQuery`）和专用系统提示词：

**阶段 1（快速）**：max_tokens=64，引导立即给出 `<block>yes/no`。如果允许 → 结束。

**阶段 2（思考）**：仅在阶段 1 阻止时运行。使用思维链减少误报。指令说明：*"审查分类过程... 覆盖阻止需要显式用户确认。"*

### 分类器看到什么

分类器看到的是**紧凑版记录** —— 不是完整对话：

```jsonl
{"user":"修复登录bug"}
{"Bash":"ls src/"}
{"Read":"src/auth.ts"}
{"Bash":"npm test"}     ← 正在被分类的操作
```

关键安全设计：**助手文本被排除** —— 仅包含 tool_use 区块。这防止模型构造影响分类器决策的诱导性文本。

### 危险权限剥离

进入 auto 模式时，会绕过分类器的权限被**剥离**：

- `Bash`（无内容）→ 允许所有命令
- `Bash(python:*)`、`Bash(node:*)` → 允许任意代码执行
- `PowerShell(iex:*)`、`PowerShell(Start-Process:*)` → 代码执行
- `Agent`（任何允许规则）→ 绕过子代理评估

被剥离的规则**暂存**到 `strippedDangerousRules`，离开 auto 模式时恢复。

---

## 5. 拒绝追踪与熔断器

### 连续拒绝限制

```typescript
export const DENIAL_LIMITS = {
  maxConsecutive: 3,   // 连续 3 次阻止 → 回退到用户提示
  maxTotal: 20,        // 单次会话总计 20 次阻止 → 回退
}
```

超过限制时：
- **交互模式**：回退到用户提示
- **无头模式**：抛出 `AbortError` —— 会话终止

### 分类器故障模式

| 场景 | iron_gate = true（默认） | iron_gate = false |
|------|------------------------|-------------------|
| **API 错误** | 拒绝（失败关闭） | 回退到用户提示（失败开放） |
| **记录过长** | 无头时中止；交互时提示 | 相同 |
| **无法解析响应** | 视为阻止 | 相同 |

`tengu_iron_gate_closed` 功能标志控制失败时关闭 vs. 开放行为，每 30 分钟刷新。

---

## 6. 无头代理权限

后台/异步代理无法显示权限提示。流水线的处理方式：

1. 运行 `PermissionRequest` 钩子 —— 给钩子机会做出决策
2. 如果没有钩子决策 → 自动拒绝

钩子可以 `allow`（带可选输入修改）、`deny` 或 `interrupt`（中止整个代理）。

---

## 7. 沙箱集成

沙箱提供**内核级强制执行**，补充应用层权限流水线：

### 沙箱自动允许

当 `autoAllowBashIfSandboxed` 启用时：
1. 通过 `shouldUseSandbox()` 检查的 Bash 命令 → **自动允许**（跳过 ask 规则）
2. OS 沙箱强制执行文件系统和网络限制
3. 应用层检查对沙箱化操作变得冗余

### 沙箱保护范围

| 保护 | 实现 |
|------|------|
| 文件系统写入 | `denyWrite` 列表（设置文件、`.claude/skills`） |
| 文件系统读取 | `denyRead` 列表（敏感路径） |
| 网络访问 | 来自 WebFetch 规则的域名允许列表 |
| 裸 Git 仓库攻击 | 命令前后文件清扫 |
| 符号链接追踪 | `O_NOFOLLOW` 文件操作 |
| 设置逃逸 | 无条件拒绝写入 settings.json |

---

## 8. 完整决策流程

```mermaid
flowchart TD
    START["工具调用请求"] --> DENY_RULE{"1a. 工具<br/>拒绝规则？"}
    DENY_RULE -->|是| DENIED["❌ 拒绝"]
    DENY_RULE -->|否| ASK_RULE{"1b. 工具<br/>询问规则？"}

    ASK_RULE -->|是| SANDBOX_CHECK{"沙箱可<br/>自动允许？"}
    SANDBOX_CHECK -->|否| ASK_RETURN["→ 询问"]
    SANDBOX_CHECK -->|是| TOOL_CHECK
    ASK_RULE -->|否| TOOL_CHECK

    TOOL_CHECK["1c. tool.checkPermissions()"] --> TOOL_DENY{"1d. 工具<br/>拒绝？"}
    TOOL_DENY -->|是| DENIED
    TOOL_DENY -->|否| SAFETY{"1g. 安全检查<br/>(.git, .claude)?"}

    SAFETY -->|是| ASK_RETURN
    SAFETY -->|否| MODE{"2a. 绕过<br/>模式？"}

    MODE -->|是| ALLOWED["✅ 允许"]
    MODE -->|否| ALLOW_RULE{"2b. 始终允许<br/>规则？"}

    ALLOW_RULE -->|是| ALLOWED
    ALLOW_RULE -->|否| PASSTHROUGH["3. → 询问 (默认)"]

    PASSTHROUGH --> MODE_TRANSFORM{"权限模式？"}
    MODE_TRANSFORM -->|dontAsk| DENIED
    MODE_TRANSFORM -->|auto| CLASSIFIER["YOLO 分类器"]
    MODE_TRANSFORM -->|default| USER_PROMPT["👤 提示用户"]

    CLASSIFIER --> FAST_PATH{"acceptEdits<br/>快速路径？"}
    FAST_PATH -->|是| ALLOWED
    FAST_PATH -->|否| XML_CLASSIFY["两阶段 XML<br/>分类器"]
    XML_CLASSIFY -->|允许| ALLOWED
    XML_CLASSIFY -->|阻止| DENIAL_LIMIT{"拒绝限制<br/>超过？"}
    DENIAL_LIMIT -->|否| DENIED
    DENIAL_LIMIT -->|是: 交互| USER_PROMPT
    DENIAL_LIMIT -->|是: 无头| ABORT["💀 中止"]
```

---

## 9. 设计洞察

### 为什么是七步而非三步？

流水线的精细粒度存在的原因是不同调用者（规则、工具、安全检查）需要不同的绕过行为。更简单的"拒绝/允许/询问"模型无法表达"即使用户说绕过一切，这也应该提示"。

### 分类器看到工具而非文本

通过从记录中剥除助手文本并仅包含 tool_use 区块，分类器对社会工程攻击免疫 —— 模型无法构造诱导性文本来影响自己的权限检查。

### 权限剥离是可逆的

进入 auto 模式时，危险权限被剥离并暂存；离开时恢复。这意味着 auto 模式是一个安全的实验 —— 用户可以尝试而不会永久丢失允许规则。

### 拒绝追踪防止无限循环

如果没有拒绝限制（连续 3 次，总计 20 次），被阻止的操作可能导致模型无休止地重试。熔断器确保人工介入（交互式）或干净关闭（无头式）。

---

## 组件总结

| 组件 | 行数 | 角色 |
|------|------|------|
| `permissions.ts` | 1,487 | 核心流水线：7步评估、模式变换 |
| `permissionSetup.ts` | 1,533 | 模式初始化、危险权限检测 |
| `yoloClassifier.ts` | 1,496 | auto 模式的两阶段 XML 分类器 |
| `PermissionMode.ts` | 142 | 6种权限模式 + 配置 |
| `PermissionRule.ts` | 41 | 规则类型：`{toolName, ruleContent?}` |
| `denialTracking.ts` | 46 | 熔断器：连续 3 次 / 总计 20 次 |
| `permissionRuleParser.ts` | ~200 | 规则字符串 ↔ 结构化值转换 |
| `permissionsLoader.ts` | ~250 | 从 7 个设置来源加载规则 |
| `shadowedRuleDetection.ts` | ~250 | 检测冲突/遮蔽的规则 |
| `sandbox-adapter.ts` | 986 | OS 沙箱：seatbelt / bubblewrap |

权限流水线是 Claude Code 架构最精密的子系统。其七步评估顺序 —— 包含四项绕过免疫的安全检查 —— 代表了关于 AI 代理执行任意代码时可能发生什么的血泪教训。YOLO 分类器的引入展示了系统从纯规则匹配向 AI 辅助安全决策的演进，同时保留确定性护栏作为安全网。

---

**上一篇**: [← 06 — Bash 执行引擎](06-bash-engine.md)
