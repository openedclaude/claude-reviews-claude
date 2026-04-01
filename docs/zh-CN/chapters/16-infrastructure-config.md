# 16 — 基础设施与配置：Claude Code 的隐藏骨架

> 🌐 **语言**: 中文 | [English →](/chapters/16-infrastructure-config)
> 📖 **[在线阅读 →](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/16-infrastructure-config)** — 支持侧边栏导航、暗色模式与全文搜索，体验更佳。

> **范围**: `bootstrap/state.ts`（56KB）、`entrypoints/init.ts`、`utils/config.ts`、`utils/settings/`、`utils/secureStorage/`、`utils/tokens.ts`、`utils/claudemd.ts`、`utils/signal.ts`、`utils/git/`、`utils/thinking.ts`、`utils/cleanupRegistry.ts`、`utils/startupProfiler.ts`
>
> **一句话概括**: 那个不起眼的基础设施层——从 1,759 行的全局状态单例到五层设置合并系统——让所有其他子系统得以运转，同时彻底杜绝循环依赖。

---

## 目录

1. [Bootstrap 全局单例模式](#1-bootstrap-全局单例模式)
2. [init.ts 初始化编排器](#2-inits-初始化编排器)
3. [双层配置系统](#3-双层配置系统)
4. [五层设置合并](#4-五层设置合并)
5. [安全存储](#5-安全存储)
6. [Signal 事件原语与 AbortController](#6-signal-事件原语与-abortcontroller)
7. [Git 工具库](#7-git-工具库)
8. [Token 管理与上下文预算](#8-token-管理与上下文预算)
9. [CLAUDE.md 与持久化记忆系统](#9-claudemd-与持久化记忆系统)
10. [Thinking 模式 API 规则](#10-thinking-模式-api-规则)
11. [可迁移的设计模式](#11-可迁移的设计模式)

---

## 1. Bootstrap 全局单例模式

**源码坐标**: `src/bootstrap/state.ts`（1,759 行，56KB——整个项目中导入最少的最大文件）

每个复杂系统都有"上帝对象"问题。Claude Code 的对策是 `state.ts`——一个位于依赖图最底层的**叶模块**，仅导入外部包和仅类型声明。这不是偶然；而是由**自定义 ESLint 规则强制执行**的。

### 1.1 叶模块约束

```typescript
// 源码位置: src/bootstrap/state.ts:17-18
// eslint-disable-next-line custom-rules/bootstrap-isolation
import { randomUUID } from 'src/utils/crypto.js'
```

`custom-rules/bootstrap-isolation` 规则确保 `state.ts` 永远不从 `src/` 的其他位置导入。唯一的例外——通过 `crypto.js` 导入的 `randomUUID`——需要显式的 ESLint 禁用注释，其存在仅因为浏览器 SDK 构建需要平台无关的 `crypto` 垫片。

**为什么重要**: 在一个 100+ 模块的代码库中，任何成为依赖中心的模块都会产生循环导入风险。通过将 `state.ts` 变成叶节点，Claude Code 保证了任何模块都能安全导入它，无需担心依赖循环。这就是对抗循环依赖的**架构免疫系统**。

### 1.2 State 对象：约 100 个字段的会话真相

私有 `STATE` 对象是会话级状态的唯一真相源。分类概览：

```
┌─────────────────────────────────────────────────────────────┐
│                    STATE 对象分类概览                         │
├──────────────────────┬──────────────────────────────────────┤
│ 身份与路径           │ originalCwd, projectRoot, cwd,       │
│                      │ sessionId, parentSessionId            │
├──────────────────────┼──────────────────────────────────────┤
│ 成本与指标           │ totalCostUSD, totalAPIDuration,       │
│                      │ turnHookDurationMs, turnToolCount     │
├──────────────────────┼──────────────────────────────────────┤
│ 模型配置             │ modelUsage, mainLoopModelOverride,    │
│                      │ initialMainLoopModel, modelStrings    │
├──────────────────────┼──────────────────────────────────────┤
│ 遥测（OpenTelemetry）│ meter, sessionCounter, locCounter,    │
│                      │ loggerProvider, tracerProvider         │
├──────────────────────┼──────────────────────────────────────┤
│ 缓存锁存             │ afkModeHeaderLatched,                  │
│（一旦开启不再关闭）   │ fastModeHeaderLatched,                 │
│                      │ promptCache1hEligible,                 │
│                      │ cacheEditingHeaderLatched               │
├──────────────────────┼──────────────────────────────────────┤
│ 会话标志             │ sessionBypassPermissionsMode,          │
│（不持久化）          │ sessionTrustAccepted,                  │
│                      │ scheduledTasksEnabled,                  │
│                      │ sessionCreatedTeams                     │
├──────────────────────┼──────────────────────────────────────┤
│ Skills 与插件        │ invokedSkills, inlinePlugins,          │
│                      │ allowedChannels, hasDevChannels         │
└──────────────────────┴──────────────────────────────────────┘
```

### 1.3 锁存机制：一旦开启，永不关闭

`state.ts` 中最精妙的模式是**粘性开关锁存**——某些 beta header 一旦激活，就会在整个会话生命周期内保持激活：

```typescript
// 源码位置: src/bootstrap/state.ts:226-242
// AFK_MODE_BETA_HEADER 的粘性锁存。一旦 auto 模式首次激活，
// 在会话剩余时间内持续发送该 header，这样 Shift+Tab 切换
// 不会破坏 ~50-70K token 的 prompt cache。
afkModeHeaderLatched: boolean | null   // null = 尚未触发

// 相同模式重复用于:
fastModeHeaderLatched: boolean | null
cacheEditingHeaderLatched: boolean | null
thinkingClearLatched: boolean | null
```

**经济账**: 如果 `Shift+Tab` 切换每次都翻转 prompt cache 控制 header，每次翻转都会使服务器端的 prompt cache（~50–70K token）失效。按 $3/MTok 输入价格计算，每次切换浪费约 $0.15–$0.21。锁存机制将这从"每次切换都花钱"变成了"整个会话只花一次"。

### 1.4 原子会话切换

```typescript
// 源码位置: src/bootstrap/state.ts:468-479
export function switchSession(
  sessionId: SessionId,
  projectDir: string | null = null,
): void {
  STATE.planSlugCache.delete(STATE.sessionId)  // 清理旧会话
  STATE.sessionId = sessionId
  STATE.sessionProjectDir = projectDir
  sessionSwitched.emit(sessionId)  // 通知订阅者
}
```

`sessionId` 和 `sessionProjectDir` 总是一起变化——没有任何一方的独立 setter。注释 `CC-34` 引用了驱动这一设计的 bug：当它们被独立设置时，`/resume` 可能导致两者不同步，导致 transcript 写入错误的目录。

### 1.5 交互时间批处理

一个为终端渲染服务的精巧优化：

```typescript
// 源码位置: src/bootstrap/state.ts:665-689
let interactionTimeDirty = false

export function updateLastInteractionTime(immediate?: boolean): void {
  if (immediate) {
    flushInteractionTime_inner()  // 立即调用 Date.now()
  } else {
    interactionTimeDirty = true   // 延迟到下一个渲染周期
  }
}
```

不在每次按键时都调用 `Date.now()`，而是标记脏位，将实际时间戳更新批量合入 Ink 渲染周期。`immediate` 路径是为 React `useEffect` 回调准备的——它们运行在渲染周期刷新_之后_。

---

## 2. init.ts 初始化编排器

**源码坐标**: `src/entrypoints/init.ts`（341 行）

`init()` 函数——用 `memoize` 包装以确保只执行一次——编排了启动序列。这是**有序初始化与策略性并行**的典范。

### 2.1 初始化序列

```
┌─ 1. enableConfigs()              — 验证并启用配置系统
│
├─ 2. applySafeConfigEnvironmentVariables()
│     ↳ 信任对话框之前仅应用安全变量
│
├─ 3. applyExtraCACertsFromConfig()
│     ↳ 必须在首次 TLS 握手之前
│     ↳ Bun 在启动时通过 BoringSSL 缓存证书存储
│
├─ 4. setupGracefulShutdown()       — 注册清理处理器
│
├─ 5. void Promise.all([...])       — 即发即忘异步初始化
│     ├─ firstPartyEventLogger      — 非阻塞
│     └─ growthbook                  — 特性开关刷新回调
│
├─ 6. void populateOAuthAccountInfoIfNeeded()   — 异步，非阻塞
│  void initJetBrainsDetection()
│  void detectCurrentRepository()
│
├─ 7. configureGlobalMTLS()         — 双向 TLS
│  configureGlobalAgents()          — 代理配置
│
├─ 8. preconnectAnthropicApi()      — TCP+TLS 握手重叠
│     ↳ 在 action-handler 的 ~100ms 工作期间完成
│
├─ 9. registerCleanup(shutdownLspServerManager)
│  registerCleanup(cleanupSessionTeams)
│
└─ 10. ensureScratchpadDir()        — 如果启用了 scratchpad
```

### 2.2 为什么顺序很重要

第 3 步（CA 证书）**必须**在第 8 步（预连接）之前：Bun 使用 BoringSSL，它在启动时缓存证书存储。如果企业设置中的额外 CA 证书在首次 TLS 握手之前没有被应用，它们在整个进程生命周期内都会被忽略。

第 8 步（预连接）**必须**在第 7 步（代理）之后：预连接优化会向 Anthropic API 打开 TCP+TLS 连接，与 action-handler 的 ~100ms 工作重叠。但它必须使用配置好的代理/mTLS 传输，所以代理设置在前。当代理/mTLS/Unix 套接字配置会阻止全局 HTTP 池复用预热连接时，预连接会被完全跳过。

### 2.3 遥测：延迟到信任对话框之后

```typescript
// 源码位置: src/entrypoints/init.ts:305-311
async function setMeterState(): Promise<void> {
  // 延迟加载以推迟 ~400KB 的 OpenTelemetry + protobuf
  const { initializeTelemetry } = await import(
    '../utils/telemetry/instrumentation.js'
  )
  const meter = await initializeTelemetry()
  // ...
}
```

遥测栈——~400KB 的 OpenTelemetry + protobuf，加上进一步的 ~700KB `@grpc/grpc-js` 导出器——仅在**信任对话框被接受后**才加载。这既是性能优化（`--version` 不需要付出导入代价），也是隐私保证（同意之前没有遥测）。

### 2.4 ConfigParseError：优雅的错误对话框

当 `settings.json` 未通过 Zod 验证时，一个基于 React 的 Ink 对话框会出现以展示错误并引导用户修复。但在非交互（SDK/无头）模式下，对话框会破坏 JSON 消费者，所以回退为写入 stderr 并退出。

---

## 3. 双层配置系统

**源码坐标**: `src/utils/config.ts`

Claude Code 将运行时状态和行为配置分开：

| 层级 | 文件 | 用途 |
|-------|------|---------|
| **GlobalConfig** | `~/.claude.json` | 运行时状态：OAuth token、会话历史、使用指标 |
| **ProjectConfig** | `.claude/config.json` | 项目状态：允许的工具、MCP 服务器、信任状态 |
| **SettingsJson** | `settings.json`（多源）| 行为：权限、钩子、模型选择、环境变量 |

### 3.1 重入防护

防止无限递归的精妙防御：

```typescript
// 源码位置: src/utils/config.ts
let insideGetConfig = false

export function getGlobalConfig(): GlobalConfig {
  if (insideGetConfig) {
    return DEFAULT_GLOBAL_CONFIG  // 短路返回默认值
  }
  insideGetConfig = true
  try {
    // ... 实际读取逻辑（可能触发 logEvent → getGlobalConfig）
  } finally {
    insideGetConfig = false
  }
}
```

调用链 `getConfig → logEvent → getGlobalConfig → getConfig` 没有这个防护就会无限递归。修复方法很优雅：重入时返回默认配置。日志事件获取了略微过时的数据，但系统不会崩溃。

---

## 4. 五层设置合并

**源码坐标**: `src/utils/settings/settings.ts`、`src/utils/settings/constants.ts`

### 4.1 五个来源

设置从五个来源加载，后加载的覆盖先加载的：

```typescript
export const SETTING_SOURCES = [
  'userSettings',      // ~/.claude/settings.json — 个人全局
  'projectSettings',   // .claude/settings.json — 项目共享，已提交
  'localSettings',     // .claude/settings.local.json — 项目本地，gitignored
  'flagSettings',      // --settings CLI 参数覆盖
  'policySettings',    // managed-settings.json 或远程 API — 企业管控
] as const
```

### 4.2 企业管控设置：Drop-In 目录

支持 systemd 风格的 drop-in 配置：

```typescript
export function loadManagedFileSettings(): { settings, errors } {
  // 1. 加载基础文件 managed-settings.json（最低优先级）
  // 2. 加载 drop-in 目录 managed-settings.d/*.json
  //    按字母排序，后文件覆盖前文件
  //    例如: 10-otel.json, 20-security.json
}
```

这使 IT 部门能够独立部署配置片段：`10-otel.json` 用于可观察性设置，`20-security.json` 用于权限策略，`30-models.json` 用于批准的模型列表。每个团队可以拥有自己的片段而不会产生合并冲突。

### 4.3 lazySchema：打破 Schema 循环依赖

```typescript
export function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined
  return () => cached ?? (cached = factory())
}
```

这不仅是性能优化——它**打破了 schema 文件之间的循环依赖**。当 `schemas/hooks.ts` 引用 `settings/types.ts` 中的类型，反之亦然时，将 schema 包装在惰性工厂函数中确保在导入时无需完全求值。

---

## 5. 安全存储

**源码坐标**: `src/utils/secureStorage/`

### 5.1 平台适配链

```typescript
export function getSecureStorage(): SecureStorage {
  if (process.platform === 'darwin') {
    return createFallbackStorage(macOsKeychainStorage, plainTextStorage)
  }
  return plainTextStorage  // Linux/Windows: 优雅降级
}
```

### 5.2 macOS Keychain：TTL 缓存 + Stale-While-Error

关键洞察是 **stale-while-error** 策略：当 `security` 子进程失败时（macOS Keychain 服务临时重启、用户切换等），继续使用缓存数据而非返回 null。如果没有这一策略，一次 `security` 进程故障会表现为全局的"未登录"错误，迫使用户重新认证。

### 5.3 异步去重

```typescript
async readAsync(): Promise<SecureStorageData | null> {
  if (keychainCacheState.readInFlight) {
    return keychainCacheState.readInFlight  // 合并并发请求
  }
  // ...
}
```

多个并发的 `readAsync()` 调用共享一个进行中的 Promise，防止对 Keychain 子进程的惊群效应。

---

## 6. Signal 事件原语与 AbortController

**源码坐标**: `src/utils/signal.ts`、`src/utils/abortController.ts`

### 6.1 Signal 原语

Claude Code 用一个可复用原语替换了约 15 处手写的监听器集合：

```typescript
// 源码位置: src/utils/signal.ts
export function createSignal<Args extends unknown[] = []>(): Signal<Args> {
  const listeners = new Set<(...args: Args) => void>()
  return {
    subscribe(listener) {
      listeners.add(listener)
      return () => { listeners.delete(listener) }  // 返回取消订阅函数
    },
    emit(...args) { for (const listener of listeners) listener(...args) },
    clear() { listeners.clear() },
  }
}
```

**Signal vs Store**: Signal 没有快照或 `getState()`——它只说"某事发生了"。Store（第 14 篇）持有状态并在变化时通知。这种区分让 API 表面保持最小化。

### 6.2 带 WeakRef 的父子 AbortController

三重内存安全保证：
1. **WeakRef** 防止父级保持对已废弃子级的强引用
2. **{once: true}** 确保监听器最多触发一次
3. **模块级 `propagateAbort`** 使用 `.bind()` 而非闭包，避免每次调用分配函数对象

---

## 7. Git 工具库

**源码坐标**: `src/utils/git/gitFilesystem.ts`

### 7.1 文件系统级 Git 状态

不为每次状态检查产生 `git` 子进程，而是直接读取 `.git` 文件：

```typescript
export async function resolveGitDir(startPath?: string): Promise<string | null> {
  const gitPath = join(root, '.git')
  const st = await stat(gitPath)
  if (st.isFile()) {
    // Worktree 或 Submodule: .git 是包含 "gitdir: <path>" 的文件
    const content = (await readFile(gitPath, 'utf-8')).trim()
    if (content.startsWith('gitdir:')) {
      return resolve(root, content.slice('gitdir:'.length).trim())
    }
  }
  return gitPath  // 正常仓库: .git 是目录
}
```

透明处理三种情况：正常仓库、Worktree（跟随 gitdir 指针）、Submodule。

### 7.2 Ref 名称安全验证

防止三种攻击向量：路径遍历（`../../../etc/passwd`）、参数注入（前导 `-`）、Shell 元字符注入（反引号、`$`、`;`、`|`、`&`）。使用白名单方式：只允许 ASCII 字母数字 + `/._+-@`。

---

## 8. Token 管理与上下文预算

**源码坐标**: `src/utils/tokens.ts`、`src/utils/context.ts`、`src/utils/tokenBudget.ts`

### 8.1 权威 Token 计数器

`tokenCountWithEstimation()` 是上下文窗口大小的**唯一真相源**。算法：
1. 从消息末尾向前查找最后一个带 `usage` 数据的 API 响应
2. 处理并行工具拆分：如果响应被拆分为多个消息（相同 `message.id`），回溯到第一个
3. 用 API 报告的 token 计数作为基线，然后**估算**该响应之后到达的消息的 token 数

这种混合方法（API 真相 + 估算）避免了昂贵的 tokenization 调用，同时保持足够的精度用于压缩阈值判断。

### 8.2 用户指定 Token 预算

用户可以在消息中直接嵌入预算提示：`+500k fix the login bug` 会被解析为 500,000 输出 token 预算。

---

## 9. CLAUDE.md 与持久化记忆系统

**源码坐标**: `src/utils/claudemd.ts`、`src/memdir/memdir.ts`

### 9.1 加载层级

从低到高（后加载覆盖前加载）：

```
1. /etc/claude-code/CLAUDE.md              — 企业全局指令
2. ~/.claude/CLAUDE.md + ~/.claude/rules/   — 用户全局指令
3. 项目 CLAUDE.md, .claude/CLAUDE.md        — 项目级指令（版本控制内）
4. CLAUDE.local.md                          — 项目本地指令（gitignored）
```

### 9.2 @include 指令系统

支持跨文件包含（`@path`、`@./relative`、`@~/home`），仅在叶文本节点中工作（代码块中的不会被当作 include），有循环引用检测，不存在的文件静默忽略。

### 9.3 自动记忆 (memdir)

`MEMORY.md` 的截断策略是刻意按行感知的：先按行截断（200 行上限），再按字节截断（25,000 字节），且始终在换行符边界切割，不会切断行内容。

---

## 10. Thinking 模式 API 规则

**源码坐标**: `src/utils/thinking.ts`

### 10.1 三种配置类型

- `adaptive`: 模型自主决定（仅 4.6+ 支持）
- `enabled { budgetTokens }`: 固定 token 预算
- `disabled`: 不使用思考块

### 10.2 提供商感知的能力检测

1P 和 Foundry 环境：所有 Claude 4+ 支持 thinking；3P（Bedrock/Vertex）：仅 Opus 4+ 和 Sonnet 4+。自适应思考仅 4.6 版本模型支持。

### 10.3 Ultrathink：构建时 + 运行时双重门控

`feature('ULTRATHINK')` 在构建时由 `bun:bundle` 解析。外部构建中为 `false`，整个函数体包括 GrowthBook 调用都被死代码消除。内部构建中，运行时 GrowthBook 标志提供动态控制。

---

## 可迁移设计模式

> 以下模式从 Claude Code 基础设施层提炼而来，可直接应用于任何复杂 CLI 工具或 Agentic 系统。

### 模式 1: 叶模块隔离

**场景**: 每个其他模块都导入的全局状态模块。
**实践**: 让全局模块成为依赖图叶节点——它**不从项目中导入任何东西**。用自定义 linter 规则强制执行。
**应用**: `bootstrap/state.ts` 通过 `custom-rules/bootstrap-isolation` 阻止从 `src/` 的任何导入。

### 模式 2: 粘性开关锁存（缓存键稳定性）

**场景**: 影响服务器端缓存的 API 请求 header 的布尔开关。
**实践**: header 首次激活后，在会话生命周期内保持激活。使用三态类型（`boolean | null`），其中 `null` 表示"尚未触发"。

### 模式 3: 重入防护

**场景**: 触发日志记录的配置读取器，日志又读取配置。
**实践**: 布尔守卫标志 + 重入时短路返回默认值。

### 模式 4: Stale-While-Error

**场景**: 偶尔失败的外部服务（OS Keychain、远程 API）。
**实践**: 失败时继续使用最近一次成功的缓存响应，而非返回 null。记录异常但不中断用户。

### 模式 5: Drop-In 配置目录

**场景**: 多个团队需要配置不同方面的企业部署。
**实践**: 支持 `settings.d/*.json` 目录，文件按字母排序加载并合并。使用数字前缀确定顺序。

### 模式 6: lazySchema 打破循环 Schema 依赖

**场景**: 互相引用的 Zod schema。
**实践**: 将 schema 构造函数包装在 `lazySchema()` 工厂中，延迟到首次使用时求值，带缓存避免重建。

### 模式 7: 父子事件传播中的 WeakRef

**场景**: 向子级传播取消的父 AbortController。
**实践**: 对父→子引用使用 `WeakRef`，模块级 `.bind()` 处理器而非闭包，`{once: true}` 事件监听器自动清理。

---

## 源码文件参考

| 文件 | 大小 | 角色 |
|------|------|------|
| `bootstrap/state.ts` | 56KB / 1,759 行 | 全局状态单例，叶模块 |
| `entrypoints/init.ts` | 14KB / 341 行 | 初始化编排器 |
| `utils/config.ts` | ~12KB | 双层配置（Global + Project）|
| `utils/settings/settings.ts` | ~15KB | 五层设置合并 + 企业 drop-in |
| `utils/secureStorage/` | ~8KB | 平台自适应凭证存储 |
| `utils/signal.ts` | ~2KB | 轻量级事件原语 |
| `utils/abortController.ts` | ~5KB | 基于 WeakRef 的父子取消 |
| `utils/git/gitFilesystem.ts` | ~7KB | 文件系统级 Git 操作 |
| `utils/tokens.ts` | ~8KB | Token 计数 + 上下文估算 |
| `utils/claudemd.ts` | ~15KB | CLAUDE.md 加载 + @include 系统 |
| `memdir/*.ts` | ~10KB | 自动记忆（MEMORY.md）系统 |
| `utils/thinking.ts` | ~5KB | Thinking 模式配置 + 能力检测 |

---

**上一篇**: [← 15 — 服务层与 API 架构](../architecture/15-services-api-layer.md) · **下一篇**: [00 — 总纲（即将推出）](../architecture/00-overview.md)
