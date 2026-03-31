# Episode 12: Startup & Bootstrap — From `claude` to First Prompt

> **Source files**: `cli.tsx` (303 lines), `init.ts` (341 lines), `setup.ts` (478 lines), `main.tsx` (4,500+ lines), `bootstrap/state.ts` (1,759 lines), `startupProfiler.ts` (195 lines), `apiPreconnect.ts` (72 lines)
>
> **One-liner**: Claude Code's startup is a carefully orchestrated race — a cascade of fast-paths, dynamic imports, parallel prefetches, and API preconnection designed to get the user typing as fast as possible while deferring 400KB+ of OpenTelemetry, plugins, and analytics to the background.

## Architecture Overview

```mermaid
graph LR
    subgraph "Phase 0: CLI Entry (cli.tsx)"
        CLI["process.argv"] --> VER{"--version?"}
        VER -->|Yes| PRINT["Print & exit<br/>Zero imports"]
        VER -->|No| FAST{"Fast-path?<br/>--daemon-worker<br/>remote-control<br/>ps/logs/attach<br/>--bg<br/>new/list/reply"}
        FAST -->|Yes| FP["Minimal import path<br/>Skip full CLI"]
        FAST -->|No| MAIN["Dynamic import main.tsx"]
    end

    subgraph "Phase 1: Module Eval (main.tsx)"
        MAIN --> IMPORTS["200+ static imports<br/>profileCheckpoint: main_tsx_imports_loaded"]
        IMPORTS --> SETTINGS["eagerLoadSettings()<br/>Read settings.json, apply env vars"]
    end

    subgraph "Phase 2: Init (init.ts)"
        SETTINGS --> CONFIGS["enableConfigs()"]
        CONFIGS --> TLS["CA certs + mTLS + proxy"]
        TLS --> PRECONN["preconnectAnthropicApi()<br/>Fire-and-forget HEAD request"]
        TLS --> OTEL["Lazy: OpenTelemetry (~400KB)"]
        TLS --> OAUTH["populateOAuthAccountInfoIfNeeded()"]
        TLS --> JETBRAINS["initJetBrainsDetection()"]
        TLS --> REPO["detectCurrentRepository()"]
    end

    subgraph "Phase 3: Setup (setup.ts)"
        CONFIGS --> HOOKS["captureHooksConfigSnapshot()"]
        HOOKS --> WORKTREE{"--worktree?"}
        WORKTREE -->|Yes| WT["createWorktreeForSession()"]
        WORKTREE -->|No| BG["Background jobs"]
        BG --> SM["initSessionMemory()"]
        BG --> CMDS["void getCommands() // prefetch"]
        BG --> PLUGINS["loadPluginHooks() // prefetch"]
        BG --> ATTR["registerAttributionHooks() // setImmediate"]
    end

    subgraph "Phase 4: Action Handler (main.tsx)"
        BG --> MCP["connectMcpServers()"]
        MCP --> RENDER["Ink render REPL"]
    end
```

---

## Phase 0: The Fast-Path Cascade

`cli.tsx` (303 lines) is the true entry point. Its design principle: **never load more than you need**.

### Zero-Import Fast-Path

```typescript
// --version: Zero module loading needed
if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
  console.log(`${MACRO.VERSION} (Claude Code)`)  // Build-time constant
  return
}
```

`MACRO.VERSION` is inlined at build time — no imports, no config, no disk I/O.

### The Fast-Path Hierarchy

| Fast-Path | Trigger | What Loads | What Skips |
|-----------|---------|------------|------------|
| `--version` | `-v`, `--version` | Nothing | Everything |
| `--dump-system-prompt` | Ant-only flag | `config.js`, `prompts.js` | UI, auth, analytics |
| `--daemon-worker` | Supervisored spawn | Worker-specific modules | Config, analytics, sinks |
| `remote-control` | `rc`, `remote`, `bridge` | Bridge + auth + policy | Full CLI, UI |
| `daemon` | `daemon` subcommand | Config + sinks + daemon | Full CLI, UI |
| `ps/logs/attach/kill` | Background sessions | Config + bg module | Full CLI, UI |
| `new/list/reply` | Templates | Template handler | Full CLI |
| `--worktree --tmux` | Combined flags | Config + worktree | May skip full CLI |
| *(default)* | Normal startup | `main.tsx` (everything) | Nothing |

Every fast-path uses dynamic `await import()` so the module tree is loaded only when that path is taken.

### Early Input Capture

```typescript
// Before loading main.tsx (which triggers heavy module eval)
const { startCapturingEarlyInput } = await import('../utils/earlyInput.js')
startCapturingEarlyInput()
```

This buffers keystrokes during the ~500ms module evaluation window so the user can start typing before the REPL is ready.

---

## Phase 1: Module Evaluation

When `main.tsx` loads, it triggers a cascade of 200+ static import evaluations. The startup profiler tracks key milestones:

```typescript
// main.tsx top-level
import { profileCheckpoint } from './utils/startupProfiler.js'
profileCheckpoint('main_tsx_entry')  // Before heavy imports

// ... 200+ imports ...

profileCheckpoint('main_tsx_imports_loaded')  // After all imports
```

### The Settings Bootstrap

```typescript
// main.tsx: eagerLoadSettings()
profileCheckpoint('eagerLoadSettings_start')
// Read settings.json, .claude/settings.json, etc.
// Apply environment variables from settings
profileCheckpoint('eagerLoadSettings_end')
```

Settings must load eagerly because they influence module-level constants (e.g., `DISABLE_BACKGROUND_TASKS` captured by BashTool at import time).

---

## Phase 2: Initialization (init.ts)

`init()` (341 lines, memoized — runs exactly once) handles trust-independent setup:

### Execution Order

```
1. enableConfigs()                    — Validate and enable configuration system
2. applySafeConfigEnvironmentVariables() — Apply safe env vars before trust dialog
3. applyExtraCACertsFromConfig()      — Must happen before first TLS handshake
4. setupGracefulShutdown()            — Register SIGINT/SIGTERM handlers
5. initialize1PEventLogging()         — Lazy: OpenTelemetry sdk-logs (deferred)
6. populateOAuthAccountInfoIfNeeded() — Async: fill OAuth cache
7. initJetBrainsDetection()           — Async: detect IDE
8. detectCurrentRepository()          — Async: populate git cache
9. configureGlobalMTLS()              — mTLS certificate configuration
10. configureGlobalAgents()           — HTTP proxy agents
11. preconnectAnthropicApi()          — Fire-and-forget HEAD request
12. setShellIfWindows()               — Git-bash detection on Windows
13. ensureScratchpadDir()             — Create temp directory if enabled
```

### API Preconnection

```typescript
// apiPreconnect.ts — Overlap TCP+TLS handshake with startup work
export function preconnectAnthropicApi(): void {
  // Skip if using proxy/mTLS/unix socket (SDK uses different transport)
  // Skip if using Bedrock/Vertex/Foundry (different endpoints)

  const baseUrl = process.env.ANTHROPIC_BASE_URL || getOauthConfig().BASE_API_URL
  // Fire and forget — 10s timeout, errors silently caught
  void fetch(baseUrl, {
    method: 'HEAD',
    signal: AbortSignal.timeout(10_000),
  }).catch(() => {})
}
```

The TCP+TLS handshake costs ~100-200ms. By firing it during init, the warmed connection is ready by the time the first API call happens. Bun's fetch shares a keep-alive connection pool globally.

### Lazy Telemetry Loading

```typescript
// OpenTelemetry is ~400KB + protobuf modules
// gRPC exporters add another ~700KB via @grpc/grpc-js
// All deferred until telemetry is actually initialized
const { initializeTelemetry } = await import('../utils/telemetry/instrumentation.js')
```

---

## Phase 3: Setup (setup.ts)

`setup()` (478 lines) runs after trust is established. It handles environment preparation:

### Key Operations

1. **UDS Messaging Server** — Unix domain socket for inter-process messaging (deferred if `--bare`)
2. **Teammate Snapshot** — Capture swarm teammate state (deferred if `--bare`)
3. **Terminal Backup Restoration** — Detect interrupted iTerm2/Terminal.app setups
4. **CWD + Hooks** — `setCwd()` must run before anything that depends on path, then `captureHooksConfigSnapshot()`
5. **Worktree Creation** — If `--worktree`, create git worktree and chdir into it
6. **Background Jobs** — Fire-and-forget prefetches

### Background Prefetch Strategy

```typescript
// Background jobs - only critical registrations before first query
initSessionMemory()     // Synchronous - registers hook, gate check lazy
void getCommands()      // Prefetch commands (parallel with user input)
void loadPluginHooks()  // Pre-load plugin hooks for SessionStart

// Deferred to next tick so git subprocess doesn't block first render
setImmediate(() => {
  void registerAttributionHooks()
})

void prefetchApiKeyFromApiKeyHelperIfSafe()

// Await only if release notes exist (interactive only)
if (!isBareMode()) {
  const { hasReleaseNotes } = await checkForReleaseNotes(...)
}
```

### The `--bare` Mode

The `--bare` flag (called "SIMPLE" internally) short-circuits massive amounts of startup:

| Skipped in `--bare` | Reason |
|---------------------|--------|
| UDS messaging server | Scripted calls don't receive injected messages |
| Teammate snapshot | No swarm in bare mode |
| Terminal backup checks | Non-interactive |
| Plugin prefetch | `executeHooks` early-returns under `--bare` |
| Attribution hooks | Scripted calls don't commit code |
| Session file access hooks | No usage metrics needed |
| Team memory watcher | No team memory in scripted mode |
| Release notes | Non-interactive |
| Recent activity | Non-interactive |

---

## The Bootstrap State Singleton

`bootstrap/state.ts` (1,759 lines) is the global state store — the **only** place session-wide mutable state lives.

### Design Constraints

```typescript
// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE
// ALSO HERE - THINK THRICE BEFORE MODIFYING
// AND ESPECIALLY HERE
```

The file contains stern warnings because it's the DAG leaf — every module can import it, but it imports almost nothing.

### Key State Categories (80+ fields)

| Category | Examples | Lifetime |
|----------|----------|----------|
| **Identity** | `sessionId`, `originalCwd`, `projectRoot`, `cwd` | Session |
| **Cost Tracking** | `totalCostUSD`, `totalAPIDuration`, `modelUsage` | Session |
| **Turn Metrics** | `turnToolDurationMs`, `turnHookCount`, `turnClassifierCount` | Per-turn (reset each query) |
| **Telemetry** | `meter`, `sessionCounter`, `loggerProvider`, `tracerProvider` | Lazy init |
| **API State** | `lastAPIRequest`, `lastMainRequestId`, `lastApiCompletionTimestamp` | Rolling |
| **Cache Latches** | `afkModeHeaderLatched`, `fastModeHeaderLatched`, `cacheEditingHeaderLatched` | Sticky-on (never unset) |
| **Feature State** | `invokedSkills`, `planSlugCache`, `systemPromptSectionCache` | Session |

### Sticky Latches for Cache Preservation

```typescript
// Once auto mode is activated, keep sending the header forever
// So Shift+Tab toggles don't bust the prompt cache
afkModeHeaderLatched: boolean | null  // null = not yet, true = latched

// Once fast mode is enabled, keep sending the header
// So cooldown enter/exit doesn't double-bust the cache
fastModeHeaderLatched: boolean | null

// Once cache editing is enabled, keep the header
// So mid-session GrowthBook toggles don't bust the cache
cacheEditingHeaderLatched: boolean | null
```

These are "sticky-on" — once set to `true`, they never go back to `false`. This pattern prevents prompt cache busting from feature toggles during a session.

### Scroll Drain Suspension

```typescript
let scrollDraining = false
const SCROLL_DRAIN_IDLE_MS = 150

export function markScrollActivity(): void {
  scrollDraining = true
  // Background intervals check getIsScrollDraining() and skip work
  // so they don't compete with scroll frames for the event loop
}
```

During active scrolling, background intervals (analytics, file watchers) voluntarily yield to keep the UI responsive.

---

## The Startup Profiler

`startupProfiler.ts` (195 lines) provides performance instrumentation for the entire startup path.

### Two Modes

| Mode | Trigger | Sampling | Output |
|------|---------|----------|--------|
| **Sampled** | Always (100% ant, 0.5% external) | Per-session random | Statsig `tengu_startup_perf` event |
| **Detailed** | `CLAUDE_CODE_PROFILE_STARTUP=1` | 100% | Full report with memory snapshots to `~/.claude/startup-perf/` |

### Phase Definitions

```typescript
const PHASE_DEFINITIONS = {
  import_time: ['cli_entry', 'main_tsx_imports_loaded'],
  init_time: ['init_function_start', 'init_function_end'],
  settings_time: ['eagerLoadSettings_start', 'eagerLoadSettings_end'],
  total_time: ['cli_entry', 'main_after_run'],
}
```

### Checkpoint Timeline (Normal Startup)

```
cli_entry                          →  t=0ms
cli_before_main_import             →  ~5ms (early input buffer set up)
main_tsx_entry                     →  ~10ms
main_tsx_imports_loaded            →  ~200-500ms (200+ modules evaluated)
eagerLoadSettings_start            →  ~500ms
eagerLoadSettings_end              →  ~520ms
main_function_start                →  ~520ms
init_function_start                →  ~525ms
init_configs_enabled               →  ~530ms
init_network_configured            →  ~540ms (mTLS + proxy)
init_function_end                  →  ~550ms
setup_before_prefetch              →  ~600ms
setup_after_prefetch               →  ~610ms
action_handler_start               →  ~650ms
action_tools_loaded                →  ~700ms
action_after_setup                 →  ~750ms
action_commands_loaded             →  ~800ms
action_mcp_configs_loaded          →  ~850ms
action_after_plugins_init          →  ~900ms (if plugins exist)
run_before_parse                   →  ~950ms
run_after_parse                    →  ~960ms
main_after_run                     →  ~1000ms (REPL ready)
```

---

## Design Insights

### Why Dynamic Imports Everywhere?

CLI tools have a startup budget of ~1 second. Claude Code's full module tree is 200+ files. Dynamic imports let each fast-path load only what it needs:

- `--version`: 0 imports, ~5ms
- `--daemon-worker`: ~10 imports, ~50ms
- Normal interactive: 200+ imports, ~500-1000ms

### The DAG Leaf Pattern

`bootstrap/state.ts` is the **leaf** of the module dependency graph — it imports almost nothing (`crypto`, `lodash`, `process`). This is enforced by an ESLint rule (`bootstrap-isolation`) that prevents it from importing modules that would create circular dependencies.

### Preconnection vs Prefetch

Two different warm-up strategies:
- **Preconnection** (`apiPreconnect.ts`): TCP+TLS handshake — saves ~100-200ms on first API call
- **Prefetch** (`setup.ts`): Commands, plugins, hooks — saves ~200ms on first user interaction

Both use fire-and-forget patterns so they don't block the startup path.

### The Ablation Baseline

```typescript
// cli.tsx: Science L0 ablation baseline
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) {
  // Disable: thinking, compact, auto-compact, auto-memory, background tasks
  // This creates a "baseline" for measuring the impact of each feature
}
```

This environment variable disables all advanced features at once, creating a controlled baseline for A/B testing individual features' impact on user experience.

---

## Component Summary

| Component | Lines | Role |
|-----------|-------|------|
| `main.tsx` | 4,500+ | Full CLI: commander setup, action handlers, REPL orchestration |
| `bootstrap/state.ts` | 1,759 | Global state singleton: 80+ fields, DAG leaf, sticky latches |
| `setup.ts` | 478 | Post-trust environment: worktrees, hooks, background prefetches |
| `init.ts` | 341 | Trust-independent init: configs, TLS, proxy, preconnect |
| `cli.tsx` | 303 | Entry point: fast-path cascade, dynamic import dispatch |
| `startupProfiler.ts` | 195 | Startup performance: checkpoints, phases, memory snapshots |
| `apiPreconnect.ts` | 72 | TCP+TLS warm-up: fire-and-forget HEAD to Anthropic API |

---

*Next: [Episode 13 — Bridge System →](13-bridge-system.md)*

[← Episode 11 — Compact System](11-compact-system.md) | [Episode 13 →](13-bridge-system.md)
