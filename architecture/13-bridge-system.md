# Episode 13: Bridge System — The Remote Control Protocol

> **Source files**: `bridge/` directory — 31 files, ~450KB total. Core: `bridgeMain.ts` (3,000 lines), `replBridge.ts` (2,407 lines), `remoteBridgeCore.ts` (1,009 lines), `replBridgeTransport.ts` (371 lines), `sessionRunner.ts` (551 lines), `types.ts` (263 lines)
>
> **One-liner**: The Bridge is Claude Code's remote control wire — a poll-dispatch-heartbeat loop that lets users type on claude.ai while their local machine executes code, with two transport generations (v1 WebSocket/POST, v2 SSE/CCRClient), crash recovery pointers, JWT refresh scheduling, and a 32-session capacity manager.

## Architecture Overview

```mermaid
graph TB
    subgraph "claude.ai / Web UI"
        USER["User types prompt"] --> SERVER["Anthropic Backend"]
    end

    subgraph "Local Machine (claude remote-control)"
        POLL["pollForWork()"] -->|work item| SPAWN["sessionRunner.spawn()"]
        SPAWN --> CHILD["Child claude --print\n--sdk-url --session-id\n--input-format stream-json"]

        subgraph "Transport Layer"
            V1["v1: HybridTransport\nWS reads + POST writes"]
            V2["v2: SSE + CCRClient\nSSETransport reads\nPOST /worker/* writes"]
        end

        POLL --> HB["heartbeatWork()"]
        HB --> SERVER
    end

    SERVER -->|"WorkResponse (JSON)"| POLL
    CHILD -->|NDJSON stdout| V1
    CHILD -->|NDJSON stdout| V2
    V1 -->|WebSocket| SERVER
    V2 -->|SSE + HTTP POST| SERVER
```

---

## The Two Bridge Modes

### 1. Standalone Bridge (`claude remote-control`)

`bridgeMain.ts` (3,000 lines) — the long-running server mode:

```
User runs: claude remote-control
→ registerBridgeEnvironment() → get environment_id + secret
→ Poll loop: pollForWork() every N ms
→ On work: spawn child `claude --print --sdk-url ...`
→ Child streams NDJSON → bridge forwards to server
→ On done: stopWork() → archive session → return to poll
```

**Spawn Modes:**

| Mode | Flag | Behavior |
|------|------|----------|
| `single-session` | (default) | One session, bridge exits when it ends |
| `worktree` | `--worktree` | Each session gets isolated git worktree |
| `same-dir` | `--spawn`, `--capacity` | All sessions share cwd (can stomp) |

**Multi-session capacity:** Up to 32 concurrent sessions (`SPAWN_SESSIONS_DEFAULT = 32`), gated by `tengu_ccr_bridge_multi_session`.

### 2. REPL Bridge (`/remote-control` command in interactive mode)

`replBridge.ts` (2,407 lines) — in-process bridge for interactive sessions:

```
User types /remote-control in REPL
→ initBridgeCore() → register environment → create session
→ Poll loop: wait for web user to type
→ Forward messages bidirectionally via transport
→ History flush: send existing conversation to web UI
```

---

## Transport Generations

### v1: HybridTransport (WebSocket + POST)

```typescript
// Session-Ingress layer
// Reads: WebSocket connection to session-ingress URL
// Writes: HTTP POST to session-ingress URL
// Auth: OAuth access token
```

### v2: SSE + CCRClient

```typescript
// CCR (Claude Code Runtime) layer
// Reads: SSETransport → GET /worker/events/stream
// Writes: CCRClient → POST /worker/events (SerialBatchEventUploader)
// Auth: JWT with session_id claim (NOT OAuth)
// Heartbeat: PUT /worker (CCRClient built-in, 20s default)
```

The v2 path adds:
- **Worker registration**: `registerWorker()` → server assigns epoch number
- **Epoch-based conflict resolution**: 409 on stale epoch → close transport, re-poll
- **Delivery tracking**: `reportDelivery('received' | 'processing' | 'processed')`
- **State reporting**: `reportState('running' | 'idle' | 'requires_action')`

### v3: Env-Less Bridge (`remoteBridgeCore.ts`)

```typescript
// Direct OAuth → worker_jwt exchange, no Environments API
// 1. POST /v1/code/sessions → session.id
// 2. POST /v1/code/sessions/{id}/bridge → {worker_jwt, expires_in, worker_epoch}
// 3. createV2ReplTransport → SSE + CCRClient
// No register/poll/ack/stop/heartbeat/deregister lifecycle
```

Gated by `tengu_bridge_repl_v2`. Eliminates the entire poll-dispatch layer for REPL sessions.

---

## The Poll-Dispatch Loop

`bridgeMain.ts` implements a sophisticated work-polling loop:

### Poll Interval Configuration (GrowthBook-driven)

```typescript
// Live-tunable via GrowthBook (refreshes every 5 min)
const pollConfig = getPollIntervalConfig()
// Intervals:
//   not_at_capacity: fast polling for new work
//   partial_capacity: moderate polling (some sessions active)
//   at_capacity: slow/heartbeat-only (all slots full)
```

### Heartbeat Modes

When at capacity, the bridge can operate in two heartbeat patterns:

1. **Non-exclusive heartbeat** (`non_exclusive_heartbeat_interval_ms > 0`): Heartbeat loop runs independently of polling, with optional at-capacity polling at a slower rate
2. **At-capacity polling only** (`at_capacity > 0`): Slow poll doubles as liveness signal
3. **At-capacity wake**: When a session completes, `capacityWake.wake()` interrupts the sleep so the bridge can immediately accept new work

### Error Recovery

```typescript
const DEFAULT_BACKOFF: BackoffConfig = {
  connInitialMs: 2_000,
  connCapMs: 120_000,      // 2 minutes max backoff
  connGiveUpMs: 600_000,   // 10 minutes → give up
  generalInitialMs: 500,
  generalCapMs: 30_000,
  generalGiveUpMs: 600_000,
}
```

The bridge distinguishes connection errors (network down) from general errors (server 500s) and applies independent exponential backoff with separate give-up timers.

---

## Session Runner

`sessionRunner.ts` (551 lines) wraps child process management:

```typescript
// Child spawn command:
claude --print \
  --sdk-url <session_url> \
  --session-id <id> \
  --input-format stream-json \
  --output-format stream-json \
  --replay-user-messages
```

### Activity Tracking

The bridge parses child stdout NDJSON to extract real-time activity summaries:

```typescript
const TOOL_VERBS: Record<string, string> = {
  Read: 'Reading', Write: 'Writing', Edit: 'Editing',
  Bash: 'Running', Glob: 'Searching', Grep: 'Searching',
  WebFetch: 'Fetching', Task: 'Running task',
}
```

### Token Refresh via Stdin

```typescript
// Send fresh JWT to child process via stdin
handle.writeStdin(JSON.stringify({
  type: 'update_environment_variables',
  variables: { CLAUDE_CODE_SESSION_ACCESS_TOKEN: token },
}) + '\n')
```

The child's StructuredIO handles `update_environment_variables` by setting `process.env` directly — so `getSessionIngressAuthToken()` picks up the new token on the next API call.

---

## JWT Lifecycle & Crash Recovery

### Token Refresh Scheduling

```typescript
const tokenRefresh = createTokenRefreshScheduler({
  refreshBufferMs: 5 * 60_000,  // 5 min before expiry
  getAccessToken: async () => { /* OAuth refresh */ },
  onRefresh: (sessionId, oauthToken) => {
    // v1: deliver OAuth token directly to child
    // v2: call reconnectSession() to trigger server re-dispatch
  },
})
```

v2's JWT approach introduces a subtle difference: each `/bridge` call bumps the server-side epoch. A JWT-only swap would leave the old CCRClient heartbeating with a stale epoch (→ 409 within 20s), so the entire transport must be rebuilt.

### Crash Recovery Pointer

```typescript
// Written after session creation (crash-recovery trail)
await writeBridgePointer(dir, {
  sessionId: currentSessionId,
  environmentId,
  source: 'repl',  // vs 'standalone'
})
// Cleared on clean teardown (unless perpetual mode)
// Perpetual mode: pointer survives clean exits for daemon restart
```

On restart, the bridge reads the pointer and attempts reconnection:
1. **Strategy 1**: Idempotent re-register with `reuseEnvironmentId` → if same env returned, `reconnectSession()` re-queues existing session
2. **Strategy 2**: If env expired (laptop slept >4h), archive old session → create fresh session on new environment

### Environment Reconnection Budget

```typescript
const MAX_ENVIRONMENT_RECREATIONS = 3
// After 3 consecutive failures, give up
```

---

## Message Protocol

### Outbound (CLI → Server)

Messages flow as NDJSON lines on the child's stdout, parsed by the bridge and forwarded via the transport layer:

| Message Type | Meaning |
|-------------|---------|
| `assistant` | Model response (text blocks, tool_use blocks) |
| `result` | Turn completed (success/error) |
| `control_request` | Permission prompt (can_use_tool) |

### Inbound (Server → CLI)

```typescript
// handleIngressMessage() processes incoming messages
// - Echo dedup: recentPostedUUIDs (BoundedUUIDSet, cap=2000)
// - Inbound dedup: recentInboundUUIDs (BoundedUUIDSet, cap=2000)
// - Control requests: model changes, permission mode, interrupt
```

### FlushGate

```typescript
// Queue live writes during history flush to preserve ordering
const flushGate = new FlushGate<Message>()
// Start: queue new messages
// End: drain queued → send, then resume direct writes
// Drop: discard queue (transport dead)
```

---

## The Permission Pipeline

When the child CLI needs tool approval:

```
Child stdout → { type: 'control_request', subtype: 'can_use_tool', tool_name, input }
→ Bridge forwards via transport → Server → claude.ai shows approval UI
→ User clicks approve/deny
→ Server sends control_response → Bridge transport → child stdin
→ reportState('running') // Clear the "waiting for input" indicator
```

The bridge also handles server-initiated control requests:
- `set_model`: Change the model mid-session
- `set_max_thinking_tokens`: Adjust thinking budget
- `set_permission_mode`: Switch between permission modes (auto, bypassPermissions)
- `api_key_interrupt`: Interrupt the current turn

---

## Design Insights

### Why Two Transport Generations?

v1 (WebSocket) was the original design. v2 (SSE+CCRClient) was introduced to:
1. **Eliminate long-lived WS connections** — SSE reconnects transparently
2. **Add delivery tracking** — `received/processing/processed` states for reliability
3. **Enable epoch-based conflict resolution** — multiple workers can't collide
4. **Reduce server load** — batched POST writes vs per-message WS frames

### The CapacityWake Pattern

```typescript
// Problem: bridge is sleeping during at-capacity wait
// Solution: AbortController-based wake signal
capacityWake.wake()  // Session ended → interrupt sleep immediately
```

Without this, the bridge would wait up to `atCapMs` (potentially minutes) before noticing a slot opened.

### Why Bootstrap-Isolation Matters Here

The bridge code carefully avoids importing from the main REPL tree. Dependencies like `auth.ts`, `config.ts`, and `commands.ts` are injected as callbacks:

```typescript
// Instead of: import { createBridgeSession } from './createSession.ts'
// Injected as: createSession: (opts) => Promise<string | null>
// Reason: createSession.ts lazy-loads auth/model/oauth which pull
//         the entire REPL tree into the Agent SDK bundle
```

---

## Component Summary

| Component | Lines | Role |
|-----------|-------|------|
| `bridgeMain.ts` | 3,000 | Standalone bridge: poll loop, multi-session, worktree, backoff |
| `replBridge.ts` | 2,407 | REPL bridge: env registration, session create, transport management |
| `remoteBridgeCore.ts` | 1,009 | Env-less bridge: direct OAuth→JWT, no poll-dispatch layer |
| `sessionRunner.ts` | 551 | Child process spawning, NDJSON parsing, activity tracking |
| `replBridgeTransport.ts` | 371 | Transport abstraction: v1 (WS+POST) vs v2 (SSE+CCRClient) |
| `types.ts` | 263 | Protocol types: WorkResponse, SessionHandle, BridgeConfig |
| `bridgeApi.ts` | ~500 | HTTP client: register, poll, ack, stop, heartbeat, reconnect |
| `bridgeMessaging.ts` | ~430 | Message handling: ingress/egress, dedup, control requests |
| `bridgeUI.ts` | ~460 | Live terminal display: status, QR codes, multi-session bullets |
| `jwtUtils.ts` | ~260 | Token refresh scheduler: proactive 5min-before-expiry refresh |
| `workSecret.ts` | ~130 | Work secret decoding: JWT extraction, SDK URL construction |

**Total bridge surface: ~11,700 lines of protocol orchestration.**

---

*Series finale. Thanks for reading the entire deep dive.*

[← Episode 12 — Startup & Bootstrap](12-startup-bootstrap.md)
