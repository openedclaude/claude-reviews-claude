> 🌐 **Language**: English | [中文版 →](/zh-CN/chapters/01-query-engine)
> 📖 **[Read Online →](https://openedclaude.github.io/claude-reviews-claude/chapters/01-query-engine)** — Sidebar nav, dark mode & full-text search. Better than raw GitHub.


# QueryEngine: The Brain of Claude Code

> **Source files**: `QueryEngine.ts` (1,296 lines), `query.ts` (1,730 lines), `query/` directory

## TL;DR

QueryEngine is the central orchestrator of Claude Code's entire lifecycle. It owns the conversation state, manages the LLM query loop, handles streaming, tracks costs, and coordinates everything from user input processing to tool execution. The core loop in `query()` is a deliberately simple `while(true)` AsyncGenerator — all intelligence lives in the LLM, the scaffold is intentionally "dumb."

---

## 1. Two Layers: QueryEngine (Session) + query() (Turn)

The engine is split into two layers with distinct lifetimes:

<p align="center">
  <img src="../assets/01-query-engine-1.svg" width="600">
</p>

| Layer | Lifetime | Responsibility |
|-------|----------|----------------|
| `QueryEngine` | Per conversation | Session state, message history, cumulative usage, file cache |
| `query()` | Per user message | API loop, tool execution, auto-compaction, budget enforcement |

### The QueryEngine Class

```typescript
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]        // Full conversation
  private abortController: AbortController  // Interrupt signal
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage      // Cumulative tokens
  private readFileState: FileStateCache     // File content LRU
  private discoveredSkillNames = new Set<string>()

  async *submitMessage(prompt, options?): AsyncGenerator<SDKMessage> {
    // ... 900+ lines of orchestration
  }
}
```

---

## 2. The submitMessage() Lifecycle

Each call to `submitMessage()` follows a precise sequence:

![01 query engine 2](../assets/01-query-engine-2.svg)

### Phase 1: Input Processing

```typescript
const { messages, shouldQuery, allowedTools, model, resultText } = 
  await processUserInput({
    input: prompt,
    mode: 'prompt',
    context: processUserInputContext,
  })
```

`processUserInput()` handles:
- **Slash commands** (`/compact`, `/clear`, `/model`, `/bug`, etc.)
- **File attachments** (images, documents)
- **Input normalization** (content blocks vs. plain text)
- **Tool allowlisting** from command output

If `shouldQuery` is false (e.g., `/clear`), the result is returned directly without calling the API.

### Phase 2: Context Assembly

Before calling the API, the system prompt is assembled from multiple sources:

```typescript
const { defaultSystemPrompt, userContext, systemContext } = 
  await fetchSystemPromptParts({
    tools: tools,
    mainLoopModel: initialMainLoopModel,
    additionalWorkingDirectories: [...],
    mcpClients: mcpClients,
    customSystemPrompt: customPrompt,
  })

// Inject coordinator context if in coordinator mode
const userContext = {
  ...baseUserContext,
  ...getCoordinatorUserContext(mcpClients, scratchpadDir),
}

// Layer: default/custom + memory mechanics + append
const systemPrompt = asSystemPrompt([
  ...(customPrompt ? [customPrompt] : defaultSystemPrompt),
  ...(memoryMechanicsPrompt ? [memoryMechanicsPrompt] : []),
  ...(appendSystemPrompt ? [appendSystemPrompt] : []),
])
```

### Phase 3: The query() Loop

The core loop is a `while(true)` AsyncGenerator:

```typescript
for await (const message of query({
  messages,
  systemPrompt,
  userContext,
  systemContext,
  canUseTool: wrappedCanUseTool,
  toolUseContext: processUserInputContext,
  maxTurns,
  taskBudget,
})) {
  // Handle each yielded message...
}
```

### Phase 4: Result Extraction

After the loop ends, the engine extracts the final result:

```typescript
const result = messages.findLast(
  m => m.type === 'assistant' || m.type === 'user',
)

if (!isResultSuccessful(result, lastStopReason)) {
  yield { type: 'result', subtype: 'error_during_execution', ... }
} else {
  yield { type: 'result', subtype: 'success', result: textResult, ... }
}
```

---

## 3. The query() Loop: 1,730 Lines of Controlled Chaos

The `query()` function in `query.ts` is the heart of tool execution. Despite being 1,730 lines, its core structure is simple:

```
while (true) {
    1. Pre-process: snip → microcompact → context collapse → autocompact
    2. Call API: stream response
    3. Post-process: execute tools, handle errors
    4. Decision: continue (tool_use) or terminate (end_turn)
}
```

### The Pre-Processing Pipeline

Before each API call, messages pass through a multi-stage compression pipeline:

<p align="center">
  <img src="../assets/01-query-engine-3.svg" width="280">
</p>

| Stage | Purpose | Feature Gate |
|-------|---------|-------------|
| **Tool result budget** | Cap aggregate tool output size, persist to disk | Always |
| **Snip** | Remove stale conversation segments | `HISTORY_SNIP` |
| **Microcompact** | Cache-aware editing of past messages | `CACHED_MICROCOMPACT` |
| **Context collapse** | Archive old turns, project collapsed view | `CONTEXT_COLLAPSE` |
| **Autocompact** | Full conversation summarization when near token limit | Always (configurable) |

### Streaming Tool Execution

Tools can execute concurrently during streaming:

```typescript
const streamingToolExecutor = new StreamingToolExecutor(
  toolUseContext.options.tools,
  canUseTool,
  toolUseContext,
)
```

When a `tool_use` block arrives in the stream, the executor begins execution immediately — before the full response finishes. This overlaps tool execution with API streaming.

### Budget Enforcement

The loop enforces three budget types:

```typescript
// 1. USD budget
if (maxBudgetUsd !== undefined && getTotalCost() >= maxBudgetUsd) {
  yield { type: 'result', subtype: 'error_max_budget_usd', ... }
  return
}

// 2. Turn budget
if (turnCount >= maxTurns) {
  yield { type: 'result', subtype: 'error_max_turns', ... }
  return
}

// 3. Token budget (per-turn output limit)
if (feature('TOKEN_BUDGET')) {
  checkTokenBudget(budgetTracker, ...)
}
```

---

## 4. State Management: The Message Switch

Inside `submitMessage()`, a large `switch` statement routes each message type:

```typescript
switch (message.type) {
  case 'assistant':     // LLM response → push to history, yield to SDK
  case 'user':          // Tool results → push to history, increment turn
  case 'progress':      // Tool progress → push and yield
  case 'stream_event':  // Raw SSE events → usage tracking, partial yields
  case 'attachment':    // Structured output, max_turns, queued commands
  case 'system':        // Compact boundaries, API errors, snip replay
  case 'tombstone':     // Message removal signals
  case 'tool_use_summary': // Aggregated tool summaries
}
```

Key design: **every message type** goes through the same yield pipeline. There are no side channels — the AsyncGenerator is the single communication path.

---

## 5. The ask() Convenience Wrapper

For one-shot usage (headless/SDK mode), `ask()` wraps QueryEngine:

```typescript
export async function* ask({ prompt, tools, ... }) {
  const engine = new QueryEngine({
    cwd, tools, commands, mcpClients,
    initialMessages: mutableMessages,
    readFileCache: cloneFileStateCache(getReadFileCache()),
    // ... 30+ config options
  })

  try {
    yield* engine.submitMessage(prompt, { uuid: promptUuid })
  } finally {
    setReadFileCache(engine.getReadFileState())
  }
}
```

The `try/finally` ensures the file state cache is always saved back — even if the generator is terminated mid-stream.

---

## 6. Error Recovery Mechanisms

The query loop handles several error recovery paths:

### Fallback Model

If the primary model fails, the loop retries with a fallback:

```typescript
try {
  for await (const message of deps.callModel({ ... })) { ... }
} catch (error) {
  if (error instanceof FallbackTriggeredError) {
    // Tombstone orphaned messages from failed attempt
    for (const msg of assistantMessages) {
      yield { type: 'tombstone', message: msg }
    }
    // Reset and retry with fallback model
    assistantMessages.length = 0
    attemptWithFallback = true
  }
}
```

### Max Output Tokens Recovery

When the model hits output token limits, the loop attempts up to 3 retries:

```typescript
const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3
```

### Reactive Compaction

If prompt-too-long errors occur, reactive compaction can compress the conversation on-the-fly.

---

## Transferable Design Patterns

> The following patterns from QueryEngine can be directly applied to any system that orchestrates LLM interactions.

### Pattern 1: AsyncGenerator as Communication Protocol

The entire engine communicates through `yield`. No callbacks, no event emitters, no message buses. The AsyncGenerator provides:
- **Backpressure**: consumer controls pace
- **Cancellation**: `.return()` propagates through `yield*`
- **Type safety**: `AsyncGenerator<SDKMessage, void, unknown>`

### Pattern 2: Mutable State with Immutable Snapshots

```typescript
this.mutableMessages.push(...messagesFromUserInput)
const messages = [...this.mutableMessages]  // Snapshot for query loop
```

The engine maintains a mutable message array (for persistence across turns) but takes immutable snapshots for each query loop iteration.

### Pattern 3: Permission Wrapping via Closure

```typescript
const wrappedCanUseTool: CanUseToolFn = async (tool, input, ...) => {
  const result = await canUseTool(tool, input, ...)
  if (result.behavior !== 'allow') {
    this.permissionDenials.push({
      tool_name: sdkCompatToolName(tool.name),
      tool_use_id: toolUseID,
      tool_input: input,
    })
  }
  return result
}
```

Permission tracking is injected transparently — the query loop doesn't know it's being monitored.

### Pattern 4: Watermark-Based Error Scoping

```typescript
const errorLogWatermark = getInMemoryErrors().at(-1)
// ... later, on error:
const all = getInMemoryErrors()
const start = errorLogWatermark ? all.lastIndexOf(errorLogWatermark) + 1 : 0
errors = all.slice(start).map(_ => _.error)
```

Instead of counting errors (which breaks when the ring buffer rotates), the engine saves a reference to the last error and slices from there. Turn-scoped errors without a counter.

---

## 8. Cost and Token Tracking: Every Cent Accounted For

> Every API call costs real money. The cost tracking system acts as the financial ledger of the entire session — accumulating costs per-model, integrating with OpenTelemetry, and persisting across session resumptions.

// 源码位置: src/cost-tracker.ts:278-323

### The Accumulation Pipeline

Each time the API returns a response, `addToTotalSessionCost()` fires a multi-step accounting process:

```typescript
export function addToTotalSessionCost(
  cost: number, usage: Usage, model: string,
): number {
  // 1. Update per-model usage map (input, output, cache read/write tokens)
  const modelUsage = addToTotalModelUsage(cost, usage, model)
  // 2. Increment global state counters
  addToTotalCostState(cost, modelUsage, model)

  // 3. Push to OpenTelemetry counters (if configured)
  const attrs = isFastModeEnabled() && usage.speed === 'fast'
    ? { model, speed: 'fast' } : { model }
  getCostCounter()?.add(cost, attrs)
  getTokenCounter()?.add(usage.input_tokens, { ...attrs, type: 'input' })
  getTokenCounter()?.add(usage.output_tokens, { ...attrs, type: 'output' })

  // 4. Recursive advisor cost — nested models (e.g., Haiku for classification)
  let totalCost = cost
  for (const advisorUsage of getAdvisorUsage(usage)) {
    const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
    totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
  }
  return totalCost
}
```

### Per-Model Usage Map

// 源码位置: src/cost-tracker.ts:250-276

The system tracks usage at model granularity. Each model gets its own running tally:

| Field | Source |
|-------|--------|
| `inputTokens` | `usage.input_tokens` |
| `outputTokens` | `usage.output_tokens` |
| `cacheReadInputTokens` | `usage.cache_read_input_tokens` |
| `cacheCreationInputTokens` | `usage.cache_creation_input_tokens` |
| `webSearchRequests` | `usage.server_tool_use?.web_search_requests` |
| `costUSD` | Accumulated dollar cost |
| `contextWindow` | Model-specific context window size |
| `maxOutputTokens` | Model-specific max output limit |

### Session Persistence and Restoration

// 源码位置: src/cost-tracker.ts:87-175

Costs survive session resumptions (`--resume`). When a session is saved, all accumulated state is written to the project config. On resume, `restoreCostStateForSession()` re-hydrates the counters — but **only if the session ID matches**. This prevents cross-session cost contamination:

```typescript
if (projectConfig.lastSessionId !== sessionId) {
  return undefined  // Don't mix costs from different sessions
}
```

---

## 9. Error Recovery: The Art of Graceful Degradation

> Error handling in the query loop is not an afterthought — it's the second-largest subsection of `query.ts`. The system implements a multi-layered retry and recovery architecture that turns transient failures into invisible blips.

### The withRetry Engine

// 源码位置: src/services/api/withRetry.ts:170-517

`withRetry()` is an AsyncGenerator that wraps every API call with sophisticated retry logic:

```
API Call → Error?
  ├── 429 (Rate Limited) → Retry with exponential backoff (base 500ms, max 32s)
  ├── 529 (Overloaded)   → Retry if foreground source; bail if background
  ├── 401 (Auth failed)  → Refresh OAuth token, clear API key cache, retry
  ├── 403 (Token revoked) → Force token refresh, retry
  ├── ECONNRESET/EPIPE   → Disable keep-alive, reconnect
  ├── Context overflow    → Calculate safe max_tokens, retry
  └── Other 5xx          → Standard exponential retry
```

### Foreground vs Background: Not All Queries Are Equal

// 源码位置: src/services/api/withRetry.ts:57-89

A critical optimization: **529 errors only retry for foreground queries**. Background tasks (summaries, title generation, classifiers) bail immediately on 529 to avoid amplifying a capacity cascade:

```typescript
// Only these sources retry on 529:
const FOREGROUND_529_RETRY_SOURCES = new Set([
  'repl_main_thread', 'sdk', 'compact',
  'agent:custom', 'agent:default', 'agent:builtin',
  'auto_mode', 'hook_agent', 'hook_prompt', ...
])

// Non-foreground → instant bail
if (is529Error(error) && !shouldRetry529(options.querySource)) {
  throw new CannotRetryError(error, retryContext)
}
```

### Fallback Model Trigger

// 源码位置: src/services/api/withRetry.ts:327-364

After 3 consecutive 529 errors, the system triggers a `FallbackTriggeredError`. The query loop catches this, tombstones orphaned messages, strips thinking signatures (which are model-bound), and retries with the fallback model.

### Persistent Retry Mode (UNATTENDED_RETRY)

// 源码位置: src/services/api/withRetry.ts:91-104, 477-513

For unattended sessions, the system retries 429/529 **indefinitely** with up to 5-minute backoff, chunking the wait into 30-second heartbeat intervals. Each heartbeat yields a `SystemAPIErrorMessage` so the host environment doesn't mark the session as idle:

```typescript
let remaining = delayMs
while (remaining > 0) {
  yield createSystemAPIErrorMessage(error, remaining, attempt, maxRetries)
  const chunk = Math.min(remaining, HEARTBEAT_INTERVAL_MS)  // 30s
  await sleep(chunk, options.signal, { abortError })
  remaining -= chunk
}
```

### max_output_tokens Recovery (Three-Stage Escalation)

// 源码位置: src/query.ts:1188-1256

When the model hits its output token limit, the loop escalates through three strategies:

```
Stage 1: Escalate to 64K tokens (one-shot upgrade)
Stage 2: Inject recovery message (up to 3 attempts)
         → "Output token limit hit. Resume directly — no apology, no recap."
Stage 3: Recovery exhausted → surface the error to the user
```

The recovery message explicitly tells the model to avoid apologizing or recapping — because that wastes more tokens and makes the problem worse.

---

## 10. Streaming Architecture: Avoiding the O(n²) Trap

> The choice between Anthropic's SDK-level `BetaMessageStream` and raw SSE processing isn't academic — it's a performance cliff that matters at scale.

// 源码位置: src/services/api/claude.ts:1266-1280

### Why Raw SSE Over SDK Streams?

The SDK's `BetaMessageStream` accumulates content blocks by rebuilding the entire message object on each delta. For a response with 10,000 characters, this means O(n²) string concatenation. Claude Code instead processes the raw SSE stream directly:

```
SDK BetaMessageStream:  message_start → rebuild full message → delta → rebuild again → ...
Raw SSE Processing:     content_block_delta → append only the delta → track state incrementally
```

The streaming pipeline uses an incremental state machine:
1. `message_start` → Initialize message structure
2. `content_block_start` → Create a new block (text, tool_use, or thinking)
3. `content_block_delta` → Append delta text to the current block (O(1) per delta)
4. `content_block_stop` → Finalize the block
5. `message_stop` → Complete, extract usage stats

### Stream Idle Watchdog

// 源码位置: src/services/api/claude.ts (stream processing)

A 90-second idle watchdog aborts streams that stall without producing data. This prevents the system from hanging indefinitely on a dead connection:

```
Stream data received → Reset timer
90s without data → Abort signal → Retry via withRetry
```

---

## 11. Context Collection: What Claude Knows About Your World

> Before any API call, the system assembles a rich picture of the user's environment. This context is memoized, prefetched during idle windows, and security-gated to prevent untrusted code execution.

// 源码位置: src/context.ts:116-189

### Two Context Sources

| Source | Function | Contents |
|--------|----------|----------|
| **System Context** | `getSystemContext()` | Git status, branch, recent log, user name |
| **User Context** | `getUserContext()` | CLAUDE.md files, current date |

### Git Status: Parallel Collection

// 源码位置: src/context.ts:60-110

Git metadata is collected with `Promise.all()` for maximum parallelism:

```typescript
const [branch, mainBranch, status, log, userName] = await Promise.all([
  getBranch(),
  getDefaultBranch(),
  execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short'], ...),
  execFileNoThrow(gitExe(), ['--no-optional-locks', 'log', '--oneline', '-n', '5'], ...),
  execFileNoThrow(gitExe(), ['config', 'user.name'], ...),
])
```

The `--no-optional-locks` flag is critical: it prevents git from acquiring the index lock, avoiding conflicts with the user's concurrent git operations.

Status output is truncated at 2,000 characters — a dirty repo with hundreds of changed files shouldn't blow up the context window.

### Memoize + Manual Invalidation

Both context functions use lodash `memoize()` for "compute once, cache forever" semantics. When the underlying data changes (e.g., system prompt injection), the cache is manually cleared:

```typescript
export function setSystemPromptInjection(value: string | null): void {
  systemPromptInjection = value
  getUserContext.cache.clear?.()
  getSystemContext.cache.clear?.()
}
```

### Security-First Prefetching

// 源码位置: src/main.tsx:360-380

Git commands can execute arbitrary code through `core.fsmonitor` and `diff.external` hooks. The system only prefetches git context **after trust has been established**:

```typescript
function prefetchSystemContextIfSafe(): void {
  if (isNonInteractiveSession) {
    void getSystemContext()  // Non-interactive: trust is implicit
    return
  }
  const hasTrust = checkHasTrustDialogAccepted()
  if (hasTrust) void getSystemContext()
  // Otherwise: DON'T prefetch — wait for trust
}
```

---

## 12. Message Normalization: The Invisible Translator

> The internal message format and the API wire format are not the same thing. `normalizeMessagesForAPI()` bridges this gap — merging, filtering, and transforming messages into what the API expects, all while preserving prompt cache integrity.

// 源码位置: src/utils/messages.ts:1989+

### What Normalization Does

```
Internal Messages → normalizeMessagesForAPI() → API-ready Messages
  - Merge consecutive same-role messages
  - Filter out progress/system/tombstone messages
  - Strip deferred tool schemas (ToolSearch optimization)
  - Normalize tool inputs for API compatibility
  - Handle thinking block placement constraints
```

### The Immutable Message Principle

// 源码位置: src/query.ts:747-787

Messages sent to the API must remain byte-identical across turns — any change invalidates the prompt cache. The system enforces this through clone-before-yield:

```typescript
// Original message is NEVER mutated — it flows back to the API as-is
let yieldMessage = message
if (message.type === 'assistant') {
  // Only clone if backfill adds NEW fields (not overwrites)
  const addedFields = Object.keys(inputCopy).some(k => !(k in block.input))
  if (addedFields) {
    clonedContent ??= [...message.message.content]
    yieldMessage = { ...message, message: { ...message.message, content: clonedContent } }
  }
}
```

### Thinking Block Constraints

The API enforces three strict rules for thinking blocks that `normalizeMessagesForAPI()` must respect:

| Rule | Constraint | Penalty for Violation |
|------|-----------|----------------------|
| **Thinking requires budget** | Messages containing thinking blocks must appear in queries with `max_thinking_length > 0` | 400 error |
| **Thinking can't be last** | A thinking block must NOT be the final block in a message | 400 error |
| **Thinking persists across tool use** | Thinking blocks must be preserved throughout the assistant's trace — spanning tool_use/tool_result boundaries | Silent corruption |

As the source code comments put it: _"不遵守这些规则的惩罚：一整天的调试和揪头发。"_ (Penalty for violating these rules: a full day of debugging and hair-pulling.)

---

## 13. CLI Bootstrap: From Binary to Query Loop

**Source coordinates**: `src/entrypoints/cli.tsx` (303 lines, 39KB)

Before `QueryEngine` ever runs, the CLI entrypoint decides *what* to run. This is not a simple argument parser — it's a **multi-path dispatcher** optimized for startup latency.

### The Fast-Path Architecture

```typescript
async function main(): Promise<void> {
  const args = process.argv.slice(2)

  // Fast-path 1: --version → zero imports, instant exit
  if (args[0] === '--version') { console.log(MACRO.VERSION); return }

  // Fast-path 2: --dump-system-prompt → minimal imports
  // Fast-path 3: --claude-in-chrome-mcp → MCP server mode
  // Fast-path 4: --computer-use-mcp → Computer Use MCP
  // Fast-path 5: --daemon-worker → lean worker (no configs/analytics)
  // Fast-path 6: remote-control|rc|bridge → Bridge mode
  // Fast-path 7: daemon → long-running supervisor
  // Fast-path 8: ps|logs|attach|kill|--bg → background sessions
  // Fast-path 9: new|list|reply → template jobs
  // Fast-path 10: environment-runner → headless BYOC
  // Fast-path 11: self-hosted-runner → self-hosted polling
  // Fast-path 12: --worktree --tmux → exec into tmux
  // ...
  // Default: load full CLI → cliMain() → QueryEngine
}
```

### Three Design Principles

**1. Dynamic `import()` everywhere**: Every fast-path uses `await import()` instead of top-level imports. The `--version` path has **zero module imports** — it prints `MACRO.VERSION` (build-time inlined) and exits instantly.

**2. `feature()` gates for dead code elimination**: Each fast-path is wrapped in `feature('FLAG')` checks. At build time, Bun's bundler eliminates entire code blocks for disabled features, so external builds never include internal-only paths like `ABLATION_BASELINE` or `DAEMON`.

**3. `profileCheckpoint()` for startup instrumentation**: Every path records its checkpoint — enabling precise startup latency measurement across all entry vectors.

### The Ablation Baseline Easter Egg

```typescript
// Inlined here (not init.ts) because BashTool/AgentTool capture
// module-level consts at import time — init() runs too late
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) {
  for (const k of [
    'CLAUDE_CODE_SIMPLE', 'CLAUDE_CODE_DISABLE_THINKING',
    'DISABLE_INTERLEAVED_THINKING', 'DISABLE_COMPACT',
    'DISABLE_AUTO_COMPACT', 'CLAUDE_CODE_DISABLE_AUTO_MEMORY',
    'CLAUDE_CODE_DISABLE_BACKGROUND_TASKS',
  ]) {
    process.env[k] ??= '1'
  }
}
```

This is a **harness-science L0 ablation** — it strips every intelligent optimization (thinking, compact, auto-memory, background tasks) to measure the raw LLM baseline. It must run before any module evaluation because tools capture env vars into constants at import time.

---

## Summary

| Aspect | Detail |
|--------|--------|
| **QueryEngine** | 1,296 lines, per-conversation lifecycle, owns message history + usage |
| **query()** | 1,730 lines, per-turn `while(true)` loop with tool execution |
| **Communication** | Pure AsyncGenerator — no callbacks, no events |
| **Pre-processing** | 5-stage compression pipeline (budget → snip → MC → collapse → autocompact) |
| **Budget limits** | USD, turns, tokens — all enforced in the loop |
| **Error recovery** | withRetry (429/529/401 matrix), fallback model, max-output escalation (3x), persistent retry |
| **Cost tracking** | Per-model `addToTotalSessionCost()`, OpenTelemetry counters, session-scoped persistence |
| **Streaming** | Raw SSE over SDK streams (avoids O(n²)), 90s idle watchdog |
| **Context** | Memoized git parallel collection, security-gated prefetch, 2K status truncation |
| **Messages** | `normalizeMessagesForAPI()` — immutable originals, clone-before-yield, thinking block rules |
| **Key principle** | "Dumb scaffold, smart model" — the loop is simple, intelligence is in the LLM |

---

**Previous**: [← 00 — Overview](./00-overview)
**Next**: [→ 02 — Tool System](./02-tool-system)
