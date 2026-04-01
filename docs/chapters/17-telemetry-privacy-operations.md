# 17 — Telemetry, Privacy & Operational Control: The Dark Side of Production

> 🌐 **Language**: English | [中文版 →](/zh-CN/chapters/17-telemetry-privacy-operations)
> 📖 **[Read Online →](https://openedclaude.github.io/claude-reviews-claude/chapters/17-telemetry-privacy-operations)** — Sidebar nav, dark mode & full-text search. Better than raw GitHub.


> **Scope**: `services/analytics/` (9 modules, ~148KB), `utils/undercover.ts`, `utils/attribution.ts`, `utils/commitAttribution.ts`, `utils/fastMode.ts`, `services/remoteManagedSettings/`, `constants/prompts.ts`, `buddy/`, `voice/`, `tasks/DreamTask/`
>
> **One-liner**: The production infrastructure you don't see — dual analytics pipelines, model codename concealment, remote killswitches, and a glimpse of features waiting behind compile-time gates.

---

---

## 1. Dual-Channel Telemetry Pipeline

**Source coordinates**: `src/services/analytics/` (9 files, ~148KB total)

Every tool call, every API request, every session start — each generates telemetry events that flow through a **dual-channel pipeline**. One channel stays in-house; the other reaches a third-party observability platform. Together, they form one of the most comprehensive analytics systems in any CLI tool.

### 1.1 Channel A: First-Party (Anthropic Direct)

```typescript
// Source: src/services/analytics/firstPartyEventLogger.ts:300-302
const DEFAULT_LOGS_EXPORT_INTERVAL_MS = 10000    // 10-second batch flush
const DEFAULT_MAX_EXPORT_BATCH_SIZE = 200         // Up to 200 events per batch
const DEFAULT_MAX_QUEUE_SIZE = 8192               // 8K events in-memory queue
```

The first-party pipeline uses **OpenTelemetry's `LoggerProvider`** — not the global one (which serves customer OTLP telemetry), but a dedicated internal provider. Events are serialized as Protocol Buffers and shipped to:

```
POST https://api.anthropic.com/api/event_logging/batch
```

**Resilience is aggressive.** The exporter (`FirstPartyEventLoggingExporter`, 27KB) implements:
- Quadratic backoff retries with configurable max attempts
- **Disk persistence** for failed batches — events survive process crashes and are retried on the next session startup from `~/.claude/telemetry/`
- Batch configuration is remotely adjustable via GrowthBook (`tengu_1p_event_batch_config`), meaning Anthropic can change flush intervals, batch sizes, and even the target endpoint without shipping a new version

**Hot-swap safety** is a subtle engineering detail worth noting:

```typescript
// Source: src/services/analytics/firstPartyEventLogger.ts:396-449
// When GrowthBook updates batch config mid-session:
// 1. Null the logger first — concurrent calls bail at the guard
// 2. forceFlush() drains the old processor's buffer
// 3. Swap to new provider; old one shuts down in background
// 4. Disk-persisted retry files use stable keys (BATCH_UUID + sessionId)
//    so the new exporter picks up any failures from the old one
```

This means the analytics pipeline can reconfigure itself on the fly — batch size, flush frequency, even the backend URL — without losing events or requiring a restart.

### 1.2 Channel B: Datadog (Third-Party Observability)

```typescript
// Source: src/services/analytics/datadog.ts:12-17
const DATADOG_LOGS_ENDPOINT = 'https://http-intake.logs.us5.datadoghq.com/api/v2/logs'
const DATADOG_CLIENT_TOKEN = 'pubbbf48e6d78dae54bceaa4acf463299bf'
const DEFAULT_FLUSH_INTERVAL_MS = 15000   // 15-second flush
const MAX_BATCH_SIZE = 100
const NETWORK_TIMEOUT_MS = 5000
```

The Datadog channel is more restrictive. Only **64 pre-approved event types** pass the whitelist filter — from `tengu_api_error` and `tengu_tool_use_success` to voice events and team memory sync signals. Every event name carries the `tengu_` prefix (more on this in §5).

**Three cardinality-reduction techniques** prevent Datadog cost explosion:

1. **MCP tool name normalization**: Any tool starting with `mcp__` gets collapsed to just `"mcp"` — preventing each unique MCP server tool from creating a new facet
2. **Model name normalization**: External users' model names are mapped to canonical names, with unrecognized models collapsed to `"other"`
3. **User bucketing**: Instead of tracking individual user IDs (which would create millions of unique facets), users are hashed into 30 buckets via `SHA256(userId) % 30`, enabling approximate unique-user alerting without cardinality explosion

```typescript
// Source: src/services/analytics/datadog.ts:281-299
const NUM_USER_BUCKETS = 30
const getUserBucket = memoize((): number => {
  const userId = getOrCreateUserID()
  const hash = createHash('sha256').update(userId).digest('hex')
  return parseInt(hash.slice(0, 8), 16) % NUM_USER_BUCKETS
})
```

### 1.3 Event Sampling: GrowthBook-Controlled Volume Dial

Not every event type needs 100% capture. A **per-event sampling configuration** (`tengu_event_sampling_config`) allows Anthropic to dynamically throttle high-volume event types:

```typescript
// Source: src/services/analytics/firstPartyEventLogger.ts:38-85
export function shouldSampleEvent(eventName: string): number | null {
  const config = getEventSamplingConfig()           // From GrowthBook
  const eventConfig = config[eventName]
  if (!eventConfig) return null                     // No config → 100% capture
  const sampleRate = eventConfig.sample_rate
  if (sampleRate >= 1) return null                  // Rate 1.0 → capture all
  if (sampleRate <= 0) return 0                     // Rate 0.0 → drop all
  return Math.random() < sampleRate ? sampleRate : 0  // Probabilistic sampling
}
```

This means any event type can be remotely dialed from 0% to 100% capture rate without code changes — a production-grade volume management mechanism.

> → Cross-reference: [Episode 16: Infrastructure](./16-infrastructure-config) for GrowthBook integration details

---

## 2. The Data Harvest: What Gets Collected

**Source coordinates**: `src/services/analytics/metadata.ts` (33KB — the single largest analytics file)

Every telemetry event carries three metadata layers, assembled by `getEventMetadata()`:

### 2.1 Layer 1: Environment Fingerprint

```
// Source: metadata.ts (conceptual summary of fields)
┌─────────────────────────────────────────────────────────────────┐
│  Environment Fingerprint (14+ fields)                           │
├──────────────────┬──────────────────────────────────────────────┤
│ Runtime          │ platform, platformRaw, arch, nodeVersion     │
│ Terminal         │ terminal type (iTerm2 / Terminal.app / ...)  │
│ Dev Environment  │ installed package managers and runtimes      │
│ CI/CD            │ CI detection, GitHub Actions metadata        │
│ OS Details       │ WSL version, Linux distro, kernel version    │
│ VCS              │ version control system type                  │
│ Claude Code      │ version, build timestamp                     │
│ Deployment       │ environment identifier                       │
└──────────────────┴──────────────────────────────────────────────┘
```

This fingerprint is rich enough to identify your specific machine configuration — what OS, what terminal, what development tools, whether you're running in CI, and what VCS you use.

### 2.2 Layer 2: Process Health Metrics

```
// Source: metadata.ts (process metrics section)
┌─────────────────────────────────────────────────────────────────┐
│  Process Metrics (8+ indicators)                                │
├──────────────────┬──────────────────────────────────────────────┤
│ Timing           │ process uptime                               │
│ Memory           │ rss, heapTotal, heapUsed, external, arrays  │
│ CPU              │ usage time, percentage                       │
└──────────────────┴──────────────────────────────────────────────┘
```

### 2.3 Layer 3: User & Session Identity

```
// Source: metadata.ts (user tracking section)
┌─────────────────────────────────────────────────────────────────┐
│  User & Session Tracking                                        │
├──────────────────┬──────────────────────────────────────────────┤
│ Model            │ active model name                            │
│ Session          │ sessionId, parentSessionId                   │
│ Device           │ deviceId (persistent across sessions)        │
│ Account          │ accountUUID, organizationUUID                │
│ Subscription     │ tier (max, pro, enterprise, team)            │
│ Repository       │ remote URL hash (SHA256, first 16 chars)     │
│ Agent            │ agent type, team name                        │
└──────────────────┴──────────────────────────────────────────────┘
```

The **repository fingerprint** deserves special attention. Rather than sending the raw repository URL (which would expose project names), the system takes the SHA256 hash and truncates to 16 hex characters. This is not anonymization — it is **pseudonymization**. An observer with access to the telemetry backend who knows a target repository's URL can trivially compute the hash and match it. The 16-character truncation provides a 64-bit collision space — effectively unique for realistic repository populations.

### 2.4 Tool Input Logging

By default, tool inputs are aggressively truncated:

```
// Source: metadata.ts (truncation thresholds)
Strings:       512 character hard cap, displayed as 128 + "…"
JSON objects:  4,096 character limit
Arrays:        maximum 20 items
Nested depth:  maximum 2 levels
```

But there is a **full-capture override**:

```typescript
// Source: metadata.ts (OTEL tool details)
// When OTEL_LOG_TOOL_DETAILS=1 is set in environment:
// ALL tool inputs are logged WITHOUT truncation
```

This environment variable, intended for debugging, creates a vector where every file path, every search query, and every code edit command is captured in full fidelity.

### 2.5 Bash Command Extension Tracking

A particularly granular collection mechanism targets Bash commands. When you run operations involving any of 17 commands (`rm`, `mv`, `cp`, `touch`, `mkdir`, `chmod`, `chown`, `cat`, `head`, `tail`, `sort`, `stat`, `diff`, `wc`, `grep`, `rg`, `sed`), the system extracts and logs the **file extensions** of the arguments. This creates a profile of what file types you work with — `.py`, `.ts`, `.go`, `.md` — without capturing specific file paths.

> → Cross-reference: [Episode 06: Bash Engine](./06-bash-engine) for command parsing

---

## 3. The Opt-Out Dilemma

**Source coordinates**: `src/services/analytics/firstPartyEventLogger.ts:141-144`, `src/services/analytics/config.ts`

### 3.1 When Analytics Are Disabled

```typescript
// Source: src/services/analytics/config.ts
// isAnalyticsDisabled() returns true ONLY for:
// 1. Test environments (NODE_ENV !== 'production')
// 2. Third-party cloud providers (Bedrock, Vertex)
// 3. Global telemetry opt-out flag
```

The first-party logging function checks this gate:

```typescript
// Source: src/services/analytics/firstPartyEventLogger.ts:141-144
export function is1PEventLoggingEnabled(): boolean {
  return !isAnalyticsDisabled()
}
```

For direct Anthropic API users (the vast majority), `isAnalyticsDisabled()` returns `false`. There is **no settings panel, no CLI flag, and no environment variable** that a regular user can set to disable first-party event logging while maintaining full product functionality.

### 3.2 The Datadog Gate

The Datadog channel adds one more restriction:

```typescript
// Source: src/services/analytics/datadog.ts:168-171
// Don't send events for 3P providers (Bedrock, Vertex, Foundry)
if (getAPIProvider() !== 'firstParty') return
```

Third-party API providers (AWS Bedrock, Google Vertex, Foundry) are exempt from Datadog logging — because their billing and analytics flow through separate systems. But first-party users cannot escape.

### 3.3 The Sink Killswitch (Remote Off-Switch)

Ironically, Anthropic _can_ remotely disable analytics — but only for themselves:

```typescript
// Source: src/services/analytics/sinkKillswitch.ts
const SINK_KILLSWITCH_CONFIG_NAME = 'tengu_frond_boric'
// GrowthBook flag that can disable analytics sinks remotely
```

This GrowthBook flag allows Anthropic to globally or selectively silence the analytics pipeline. Individual users have no equivalent capability.

### 3.4 Regulatory Implications

The combination of: (1) no user-facing opt-out, (2) persistent device and session tracking, (3) repository fingerprinting, and (4) organization-level identification creates a dataset that falls squarely within the scope of GDPR Article 6 (lawful basis for processing) and CCPA Section 1798.100 (right to know). While the truncation and hashing measures reduce sensitivity, the persistent `deviceId` and `accountUUID` constitute identifiable data under most privacy frameworks.

> → Design pattern: The **stale-while-error** strategy on failed telemetry exports (disk-persist + retry) prioritizes delivery completeness over user control. This is a deliberate product decision that trades privacy transparency for operational observability.

---

## 4. Model Codename System

**Source coordinates**: `src/utils/undercover.ts:48-49`, `src/constants/prompts.ts`, `src/migrations/migrateFennecToOpus.ts`, `src/buddy/types.ts`

Anthropic assigns **animal codenames** to internal model versions — a practice common in tech companies, but Claude Code's source reveals the specific codenames, their evolutionary lineage, and the elaborate machinery built to prevent them from leaking.

### 4.1 The Four Known Codenames

| Codename | Animal | Role | Evidence |
|----------|--------|------|----------|
| **Capybara** | 水豚 | Sonnet-series model, currently v8 | `capybara-v2-fast[1m]` in model strings; dedicated prompt patches |
| **Tengu** | 天狗 | Product/telemetry prefix | All 250+ analytics events and feature flags use `tengu_*` prefix |
| **Fennec** | 耳廓狐 | Predecessor to Opus 4.6 | Migration script: `fennec-latest → opus` |
| **Numbat** | 袋食蚁兽 | Next unreleased model | Comment: `"Remove this section when we launch numbat"` |

### 4.2 The Evolution Chain

```
Fennec (耳廓狐)  ──migration──→  Opus 4.6  ──→  [Numbat?]
Capybara (水豚)  ──────────────→  Sonnet v8 ──→  [?]
Tengu (天狗)     ──────────────→  Product/telemetry prefix (not a model)
```

The Fennec-to-Opus migration is a concrete code artifact:

```typescript
// Source: src/migrations/migrateFennecToOpus.ts:7-11
// fennec-latest      → opus
// fennec-latest[1m]  → opus[1m]
// fennec-fast-latest → opus[1m] + fast mode enabled
```

### 4.3 Codename Protection Mechanisms

Two layers prevent codenames from leaking into external builds:

**Layer 1: Build-time scanner** — `scripts/excluded-strings.txt` contains patterns that the CI pipeline scans for in build output. Any match fails the build.

**Layer 2: Runtime obfuscation** — When a codename might appear in user-visible strings, it is actively masked:

```typescript
// Source: src/utils/model/model.ts:386-392
function maskModelCodename(baseName: string): string {
  // e.g. capybara-v2-fast → cap*****-v2-fast
  const [codename = '', ...rest] = baseName.split('-')
  const masked = codename.slice(0, 3) + '*'.repeat(Math.max(0, codename.length - 3))
  return [masked, ...rest].join('-')
}
```

**Layer 3: Source-level collision avoidance** — The Buddy virtual pet system includes "capybara" as a pet species, which collides with the model codename scanner. The solution is encoding the species name character-by-character at runtime to keep the literal out of the source bundle:

```typescript
// Source: src/buddy/types.ts:10-13
// One species name collides with a model-codename canary in excluded-strings.txt.
// The check greps build output (not source), so runtime-constructing the value
// keeps the literal out of the bundle while the check stays armed for the actual codename.
```

### 4.4 Capybara v8: Five Documented Behavioral Defects

The source code contains specific, annotated workarounds for Capybara v8 issues — a rare window into model-level debugging:

| # | Defect | Impact | Source Location |
|---|--------|--------|----------------|
| 1 | **Stop sequence false trigger** | ~10% rate when `<functions>` appears at prompt tail | `prompts.ts` / `messages.ts:2141` |
| 2 | **Empty tool_result zero output** | Model generates nothing when receiving blank tool results | `toolResultStorage.ts:281` |
| 3 | **Over-commenting** | Requires dedicated anti-commenting prompt patches | `prompts.ts:204` |
| 4 | **High false-claims rate** | 29-30% FC rate vs. Capybara v4's 16.7% | `prompts.ts:237` |
| 5 | **Insufficient verification** | Requires "thoroughness counterweight" prompt injection | `prompts.ts:210` |

Each defect has a `@[MODEL LAUNCH]` annotation tied to it, indicating these are temporary patches expected to be removed or revised when the next model (Numbat) launches. The codebase contains **8+ `@[MODEL LAUNCH]` markers** covering: default model names, family IDs, knowledge cutoff dates, pricing tables, context window configurations, thinking mode support, display name mappings, and migration scripts.

> → Cross-reference: [Episode 01: QueryEngine](./01-query-engine) for how prompt patches affect the query loop

---

## 5. Feature Flag Obfuscation

**Source coordinates**: `src/services/analytics/growthbook.ts` (41KB), `src/services/analytics/sinkKillswitch.ts`, various `utils/*.ts`

### 5.1 The Tengu Naming Convention

Every feature flag and analytics event follows a deliberate naming pattern designed to obscure its purpose from external observers:

```
tengu_<word1>_<word2>
```

The word pairs are selected from a constrained vocabulary — adjective/material words paired with nature/object words — creating names that are memorable to insiders but opaque to outsiders. Examples from the actual codebase:

| Flag Name | Decoded Purpose | Category |
|-----------|----------------|----------|
| `tengu_frond_boric` | Analytics sink killswitch | Killswitch |
| `tengu_amber_quartz_disabled` | Voice mode emergency off | Killswitch |
| `tengu_amber_flint` | Agent teams gate | Feature gate |
| `tengu_hive_evidence` | Verification agent gate | Feature gate |
| `tengu_onyx_plover` | Auto-Dream (background memory) | Feature gate |
| `tengu_coral_fern` | Memdir feature | Feature gate |
| `tengu_moth_copse` | Memdir switch (secondary) | Feature gate |
| `tengu_herring_clock` | Team memory | Feature gate |
| `tengu_passport_quail` | Path feature | Feature gate |
| `tengu_slate_thimble` | Memdir switch (tertiary) | Feature gate |
| `tengu_sedge_lantern` | Away summary | Feature gate |
| `tengu_marble_sandcastle` | Fast mode (Penguin) gate | Feature gate |
| `tengu_penguins_off` | Fast mode disable | Killswitch |
| `tengu_turtle_carbon` | Ultrathink gate | Feature gate |
| `tengu_log_datadog_events` | Datadog event gate | Analytics |
| `tengu_event_sampling_config` | Per-event sampling rates | Config |
| `tengu_1p_event_batch_config` | 1P batch processor config | Config |
| `tengu_ant_model_override` | Internal model override | Internal |
| `tengu_max_version_config` | Version enforcement | Config |

### 5.2 The Three-Tier Flag Resolution

Feature behavior in Claude Code is controlled through three distinct mechanisms, each operating at a different level:

```
┌─────────────────────────────────────────────────────────────────┐
│  Tier 1: Compile-Time DCE (Dead Code Elimination)               │
│  mechanism: feature('FLAG_NAME') from bun:bundle                │
│  scope:     entire code branches removed at build time          │
│  examples:  VOICE_MODE, DAEMON, KAIROS, COORDINATOR_MODE        │
│  effect:    code doesn't exist in published bundle at all       │
├─────────────────────────────────────────────────────────────────┤
│  Tier 2: Runtime Environment Check                              │
│  mechanism: process.env.USER_TYPE === 'ant'                     │
│  scope:     code exists but is bypassed for external users      │
│  examples:  REPLTool, TungstenTool, debug logging               │
│  effect:    constant-folded by V8 JIT after first check         │
├─────────────────────────────────────────────────────────────────┤
│  Tier 3: Runtime GrowthBook Flag                                │
│  mechanism: getFeatureValue('tengu_*') via GrowthBook SDK       │
│  scope:     can change per-user, per-session, per-experiment    │
│  examples:  all tengu_* flags listed above                      │
│  effect:    cached locally, refreshed periodically              │
└─────────────────────────────────────────────────────────────────┘
```

Some features use **double-gating** — a Tier 1 compile-time gate combined with a Tier 3 runtime flag:

```typescript
// Source: src/utils/thinking.ts
export function isUltrathinkEnabled(): boolean {
  if (!feature('ULTRATHINK')) return false      // Tier 1: DCE in external builds
  return getFeatureValue_CACHED_MAY_BE_STALE(   // Tier 3: runtime toggle
    'tengu_turtle_carbon', true
  )
}
```

In external builds, `feature('ULTRATHINK')` is `false`, so the entire function body — including the GrowthBook call — is dead-code-eliminated. In internal builds, the runtime flag provides dynamic control. This two-tier approach means Anthropic can both restrict features to internal builds AND dynamically control them within those builds.

### 5.3 The Sink Killswitch Architecture

The analytics killswitch (`tengu_frond_boric`) deserves special attention for its engineering subtlety:

```typescript
// Source: src/services/analytics/sinkKillswitch.ts
export function isSinkKilled(sink: SinkName): boolean {
  const config = getDynamicConfig_CACHED_MAY_BE_STALE<
    Partial<Record<SinkName, boolean>>
  >(SINK_KILLSWITCH_CONFIG_NAME, {})
  // NOTE: Must NOT be called from is1PEventLoggingEnabled() —
  // growthbook.ts:isGrowthBookEnabled() calls that, creating a cycle
  return config?.[sink] === true
}
```

The comment reveals a **circular dependency hazard**: GrowthBook initialization calls `is1PEventLoggingEnabled()`, which would call `isSinkKilled()`, which calls GrowthBook — infinite recursion. The solution: the killswitch is checked at **each event dispatch site**, not at the global enable check. This is a textbook example of breaking circular dependencies through architectural placement rather than lazy loading.

> → Cross-reference: [Episode 16: Infrastructure](./16-infrastructure-config) §1.3 for the latching pattern used in analytics beta headers

---

## 6. Undercover Mode: Concealing AI Authorship

**Source coordinates**: `src/utils/undercover.ts` (90 lines), `src/utils/attribution.ts` (394 lines), `src/utils/commitAttribution.ts` (30KB)

When Anthropic employees use Claude Code to contribute to open-source repositories, a specialized system activates to hide all traces of AI involvement. This is "Undercover Mode" — a built-in mechanism for concealing the AI's identity in commits, pull requests, and code contributions.

### 6.1 Activation Logic

```typescript
// Source: src/utils/undercover.ts:28-37
export function isUndercover(): boolean {
  if (process.env.USER_TYPE === 'ant') {          // Only for Anthropic employees
    if (isEnvTruthy(process.env.CLAUDE_CODE_UNDERCOVER)) return true  // Force ON
    // Auto-detect: active UNLESS we've confirmed internal repo
    // 'external', 'none', null → all resolve to ON
    // Only 'internal' → OFF
    return getRepoClassCached() !== 'internal'
  }
  return false  // External users: always off (dead-code-eliminated in prod)
}
```

The activation decision tree:

```
Is USER_TYPE === 'ant'?
├─ NO  → return false (entire function DCE'd in external builds)
└─ YES → Is CLAUDE_CODE_UNDERCOVER env var set?
         ├─ YES → Force ON (even in internal repos)
         └─ NO  → Is repo in INTERNAL_MODEL_REPOS allowlist?
                   ├─ YES ('internal') → OFF
                   └─ NO/Unknown/None  → ON (safe default)
```

The critical design decision: **there is no force-OFF**. If the system cannot confirm the current repository is on Anthropic's internal allowlist, undercover mode activates. This fail-safe ensures that even when working in `/tmp` or any non-git directory, AI traces never leak.

### 6.2 The Undercover Prompt

When active, the model receives explicit concealment instructions:

```typescript
// Source: src/utils/undercover.ts:41-69 (full prompt text)
`## UNDERCOVER MODE — CRITICAL

You are operating UNDERCOVER in a PUBLIC/OPEN-SOURCE repository.
Your commit messages, PR titles, and PR bodies MUST NOT contain
ANY Anthropic-internal information. Do not blow your cover.

NEVER include in commit messages or PR descriptions:
- Internal model codenames (animal names like Capybara, Tengu, etc.)
- Unreleased model version numbers (e.g., opus-4-7, sonnet-4-8)
- Internal repo or project names (e.g., claude-cli-internal)
- Internal tooling, Slack channels, or short links (e.g., go/cc)
- The phrase "Claude Code" or any mention that you are an AI
- Any hint of what model or version you are
- Co-Authored-By lines or any other attribution`
```

Seven explicit prohibitions ensure no artifact of AI authorship survives.

### 6.3 Attribution Stripping

The attribution system (`attribution.ts`) actively removes AI traces when undercover:

```typescript
// Source: src/utils/attribution.ts:52-55
export function getAttributionTexts(): AttributionTexts {
  if (process.env.USER_TYPE === 'ant' && isUndercover()) {
    return { commit: '', pr: '' }  // Empty strings = no attribution
  }
  // ... normal attribution logic
}
```

In normal mode, commits get `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>` and PRs get `🤖 Generated with Claude Code`. In undercover mode, both return empty strings.

**For unknown models, there is a hardcoded fallback:**

```typescript
// Source: src/utils/attribution.ts:70-78
// @[MODEL LAUNCH]: Update the hardcoded fallback model name below
// For external repos, fall back to "Claude Opus 4.6" for
// unrecognized models to avoid leaking codenames.
const modelName = isInternalModelRepoCached() || isKnownPublicModel
  ? getPublicModelName(model)
  : 'Claude Opus 4.6'
```

This ensures even in non-undercover mode, unknown model names never appear in user-visible attribution.

### 6.4 Enhanced PR Attribution (Non-Undercover)

When not undercover, the attribution system generates rich statistics:

```
🤖 Generated with Claude Code (93% 3-shotted by claude-opus-4-5, 2 memories recalled)
```

This includes: Claude's code contribution percentage (from `commitAttribution.ts` 30KB worth of git attribution analysis), prompt count (N-shotted), model name, and memory file access count.

### 6.5 Ethical Implications

Undercover mode raises fundamental questions about AI transparency in open-source:

1. **Contributor authenticity**: Other maintainers cannot distinguish human-written from AI-generated contributions
2. **License compliance**: Some open-source licenses require disclosure of AI-generated content
3. **Community trust**: The absence of a force-OFF switch means the system is engineered to make concealment the default
4. **Regulatory divergence**: EU AI Act Article 50 may require disclosure of AI-generated content in certain contexts

This is not a bug — it is a deliberate product decision that prioritizes Anthropic's competitive positioning (employees using Claude Code effectively in public) over contributor transparency.

> → Design pattern: **Fail-safe default** — when in doubt, conceal. This inverts the typical transparency principle: instead of requiring explicit opt-in to hide AI involvement, it requires explicit allowlisting to reveal it.

---

## 7. Remote Control & Emergency Switches

**Source coordinates**: `src/services/remoteManagedSettings/` (5 files, ~43KB), `src/utils/fastMode.ts` (18KB), `src/utils/permissions/bypassPermissionsKillswitch.ts`

Claude Code is not just a local tool — it maintains a persistent connection to Anthropic's servers for configuration, and Anthropic can remotely alter its behavior, enforce policies, or shut down features.

### 7.1 Remote Managed Settings

```typescript
// Source: src/services/remoteManagedSettings/index.ts:52-54
const SETTINGS_TIMEOUT_MS = 10000     // 10s fetch timeout
const DEFAULT_MAX_RETRIES = 5         // Up to 5 retries per fetch
const POLLING_INTERVAL_MS = 3600000   // 1-hour background polling
```

Settings are fetched from:
```
GET https://api.anthropic.com/api/claude_code/settings
```

The fetch lifecycle implements **stale-while-revalidate** semantics:

1. **Cache-first on startup**: If disk-cached settings exist, apply them immediately and unblock dependents
2. **Fetch in background**: New settings are fetched with 5-retry exponential backoff
3. **ETag-based caching**: SHA256 checksums enable HTTP 304 Not Modified responses
4. **Hourly polling**: Background interval checks for settings changes mid-session
5. **Fail-open**: If all fetches fail, continue with stale cache or no remote settings

### 7.2 The Accept-or-Die Dialog

When remote settings contain "dangerous" changes (e.g., new permission overrides, custom tool allowlists), a **blocking security dialog** appears:

```typescript
// Source: src/services/remoteManagedSettings/securityCheck.tsx:67-73
export function handleSecurityCheckResult(result: SecurityCheckResult): boolean {
  if (result === 'rejected') {
    gracefulShutdownSync(1)   // Exit code 1 — process terminates
    return false
  }
  return true
}
```

This is the "Accept-or-Die" pattern: the user can either accept the new settings or the process terminates with exit code 1. There is no "dismiss" or "ask later" option. The dialog is rendered via Ink (React for terminals) and blocks the entire CLI until the user responds.

**Non-interactive mode bypass**: In CI/CD environments (`!getIsInteractive()`), the security check is skipped entirely — dangerous settings are applied silently.

### 7.3 The Six Emergency Switches

Six remote killswitches discovered in the source, each controlling a critical behavior:

| # | Switch | Mechanism | Effect When Triggered |
|---|--------|-----------|----------------------|
| 1 | **Permissions Bypass** | `bypassPermissionsKillswitch.ts` | Disables the entire permission system — all tools auto-approved |
| 2 | **Auto Mode Breaker** | `autoModeDenials.ts` | Emergency circuit breaker for autonomous execution |
| 3 | **Fast Mode (Penguin)** | `tengu_marble_sandcastle` + `/api/claude_code_penguin_mode` | Switches to cheaper/faster model for cost reduction |
| 4 | **Analytics Sink** | `tengu_frond_boric` | Disables Datadog and/or 1P event logging |
| 5 | **Agent Teams** | `tengu_amber_flint` | Gates multi-agent collaboration feature |
| 6 | **Voice Mode** | `tengu_amber_quartz_disabled` | Emergency disable for voice input |

### 7.4 Penguin Mode (Fast Mode Remote Control)

"Penguin Mode" is a particularly interesting remote control mechanism. It allows Anthropic to remotely switch users from expensive Opus/Sonnet models to cheaper alternatives:

```typescript
// Source: src/utils/fastMode.ts (conceptual summary)
// 1. Local fast mode: user opts in via /fast command
// 2. Remote fast mode: Anthropic enables via API endpoint
//    GET /api/claude_code_penguin_mode → { enabled: boolean, model: string }
// 3. When active: model is silently replaced, thinking mode may be disabled
// 4. User is NOT explicitly notified of model change (though UI hints exist)
```

The combination of:
- Remote model switching without explicit user consent
- GrowthBook-based A/B assignment (`tengu_marble_sandcastle`)
- Separate killswitch (`tengu_penguins_off`) for emergency disable

...creates a system where the model serving your requests can change mid-session based on Anthropic's operational decisions.

### 7.5 Model Override System (Internal Only)

For Anthropic employees, the model override is even more granular:

```
// Source: growthbook.ts (tengu_ant_model_override config)
// GrowthBook config that can override:
// - Default model (e.g., force all ant users to numbat-latest)
// - Thinking effort level
// - Additional prompt appendages
// - Model-specific behavior flags
```

This enables internal A/B testing of unreleased models (like Numbat) across the entire Anthropic engineering team.

> → Cross-reference: [Episode 16: Infrastructure](./16-infrastructure-config) for the five-layer settings merge system

---

## 8. The Two-Tier User Experience

**Source coordinates**: `src/constants/prompts.ts`, `src/tools/`, `src/commands.ts`

Anthropic employees and external users experience fundamentally different versions of Claude Code. This divergence spans prompts, tools, commands, and model behavior.

### 8.1 Prompt Divergence: Six Dimensions

| Dimension | External Users | Anthropic Employees (`ant`) |
|-----------|---------------|---------------------------|
| **Output style** | Standard formatting | `tengu_output_style_prompt` override via GrowthBook |
| **False-claims mitigation** | Capybara v8 29-30% FC rate mitigated via prompt patch | Same patch + additional numerical anchoring prompts |
| **Verification** | Standard verification | `tengu_hive_evidence` verification agent + thoroughness counterweight |
| **Comment control** | Standard commenting guidance | Dedicated anti-over-commenting prompt (Capybara v8 fix) |
| **Proactive correction** | Standard behavior | Enhanced "assertiveness counterweight" (PR #24302) |
| **Model awareness** | Cannot see model codename | Sees internal model name, can use debugging tools |

### 8.2 Internal-Only Tools

Five tools are gated behind `USER_TYPE === 'ant'`:

| Tool | Purpose | Gate |
|------|---------|------|
| **REPLTool** | Inline code execution in REPL sessions | Tier 2 (env check) |
| **TungstenTool** | Internal debugging and diagnostics | Tier 2 (env check) |
| **VerifyPlanTool** | Verification agent for plan validation | Tier 3 (`tengu_hive_evidence`) |
| **SuggestBackgroundPR** | Suggest follow-up PRs from background analysis | Tier 1 (`feature()`) |
| **Nested Agent** | In-process sub-agent spawning | Tier 2 (env check) |

### 8.3 Hidden Commands

Several slash commands exist but are not documented in public help:

| Command | Purpose | Access |
|---------|---------|--------|
| `/btw` | Side-comment injection into conversation | Internal only |
| `/stickers` | Terminal sticker/art display | Unlockable |
| `/thinkback` | Replay last thinking trace | Debug mode |
| `/effort` | Adjust model thinking effort level | Internal only |
| `/buddy` | Summon virtual companion (see §9) | Behind `feature()` |
| `/good-claude` | Positive reinforcement (may affect heuristics) | Internal only |
| `/bughunter` | Activate bug-hunting mode | Internal only |

> → Cross-reference: [Episode 08: Agent Swarms](./08-agent-swarms) for nested agent behavior

---

## 9. Future Roadmap: Evidence from Source

**Source coordinates**: `src/tasks/DreamTask/`, `src/buddy/`, `src/voice/`, `src/coordinator/`, `src/moreright/`

The source code contains substantial implementations of features that are compile-time gated but architecturally complete. These are not speculative — they have real code behind them.

### 9.1 Numbat: The Next Model Generation

Evidence from `prompts.ts`:
```typescript
// Source: src/constants/prompts.ts:402
// @[MODEL LAUNCH]: Remove this section when we launch numbat.
```

Additional `@[MODEL LAUNCH]` markers reference model IDs like `opus-4-7` and `sonnet-4-8`, strongly suggesting Numbat is the codename for the next major model family.

### 9.2 KAIROS: Autonomous Agent Mode

A fully architected autonomous execution mode exists behind `feature('KAIROS')`:

- **Tick-based heartbeat**: Unlike the interactive request-response loop, KAIROS agents operate on periodic "ticks" — autonomous execution cycles that continue without user input
- **Focus awareness**: The system tracks whether the terminal is focused or unfocused, adjusting behavior accordingly (e.g., batching operations when unfocused)
- **Push notifications**: Integration with OS-level notification systems (`PushNotificationTool`) to alert users of autonomous progress
- **PR subscription**: `SubscribePRTool` enables agents to monitor GitHub PRs and react to CI status changes
- **Sleep/wake**: `SleepTool` allows agents to pause and resume at scheduled intervals

> → Cross-reference: [Episode 08: Agent Swarms](./08-agent-swarms) §4 for initial KAIROS discussion

### 9.3 Voice Mode

Behind `feature('VOICE_MODE')`:
- **Push-to-talk**: Keyboard-activated voice recording
- **WebSocket streaming**: Real-time speech-to-text via WebSocket connection (21KB in `voiceStreamSTT.ts`)
- **mTLS authentication**: Mutual TLS for secure voice data transmission
- **OAuth-restricted**: Voice requires OAuth login (not API key) for authentication
- **Keyterm vocabulary**: Custom vocabulary file (`voiceKeyterms.ts`, 3.5KB) of technical terms for improved recognition

### 9.4 Buddy: Virtual Companion System

The most whimsical feature — a complete virtual pet system (6 files, ~76KB):

**18 Species** (all encoded via `String.fromCharCode` to avoid codename scanner):
```
duck, goose, blob, cat, dragon, octopus, owl, penguin,
turtle, snail, ghost, axolotl, capybara, cactus, robot,
rabbit, mushroom, chonk
```

**5 Rarity Tiers** with weighted distribution:
| Rarity | Weight | Stars | Probability |
|--------|:------:|:-----:|:-----------:|
| Common | 60 | ★ | 60% |
| Uncommon | 25 | ★★ | 25% |
| Rare | 10 | ★★★ | 10% |
| Epic | 4 | ★★★★ | 4% |
| Legendary | 1 | ★★★★★ | 1% |

**8 Hats**: none, crown, tophat, propeller, halo, wizard, beanie, tinyduck

**6 Eye Styles**: `·`, `✦`, `×`, `◉`, `@`, `°`

**5 Stats**: DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK

**Shiny variant**: ~1% chance — deterministic from `hash(userId)`, preventing users from "rerolling" their companion.

Each companion has a **soul** (model-generated name and personality) stored in config, while **bones** (species, rarity, stats) are regenerated from `hash(userId)` on every read — ensuring users cannot edit their config file to upgrade to a legendary.

### 9.5 Unreleased Tools (11 Identified)

| Tool | Purpose | Gate |
|------|---------|------|
| SleepTool | Scheduled agent pause/resume | `feature('KAIROS')` |
| PushNotificationTool | OS notification dispatch | `feature('KAIROS')` |
| SubscribePRTool | GitHub PR subscription | `feature('KAIROS')` |
| DaemonTool | Background process management | `feature('DAEMON')` |
| CoordinatorTool | Multi-agent coordination | `feature('COORDINATOR_MODE')` |
| MorerightTool | Context-window extension | `feature('MORERIGHT')` |
| DreamConsolidationTool | Background memory consolidation | `feature('AUTO_DREAM')` |
| DxtTool | DXT plugin packaging | `feature('DXT')` |
| UltraplanTool | Advanced multi-step planning | `feature('ULTRAPLAN')` |
| VoiceInputTool | Voice-to-text input | `feature('VOICE_MODE')` |
| BuddyTool | Virtual companion summon | `feature('BUDDY')` |

### 9.6 Three Strategic Directions

The unreleased features cluster into three clear strategic directions:

1. **Autonomous Agents** (KAIROS + Dream + Coordinator): Moving from reactive tool to proactive agent that operates independently
2. **Multi-modal Input** (Voice + Computer Use enhancements): Expanding beyond text-only interaction
3. **Social/Emotional** (Buddy + Stickers + Team Memory): Creating engagement loops and team collaboration features

These directions suggest Claude Code's long-term vision is not "better code completion" but rather "autonomous software engineering agent with social features."

---

## Source Coordinates Summary

| Component | Key Files | Size |
|-----------|----------|:----:|
| Analytics pipeline | `services/analytics/` (9 files) | 148KB |
| Undercover mode | `utils/undercover.ts` | 3.7KB |
| Attribution system | `utils/attribution.ts` + `commitAttribution.ts` | 44KB |
| Remote settings | `services/remoteManagedSettings/` (5 files) | 43KB |
| Fast mode | `utils/fastMode.ts` | 18KB |
| GrowthBook | `services/analytics/growthbook.ts` | 41KB |
| Buddy system | `buddy/` (6 files) | 76KB |
| Voice system | `voice/` + `services/voice*.ts` | 45KB |

---

## Transferable Design Patterns

| Pattern | Where Used | Takeaway |
|---------|-----------|----------|
| **Dual-channel analytics** | 1P + Datadog | Separate internal/external analytics with different retention policies |
| **Cardinality reduction** | User bucketing, MCP normalization | Hash-based bucketing (mod N) enables approximate unique-user counts without cardinality explosion |
| **Hot-swap reconfiguration** | 1P logger reinit | Null-guard → flush → swap → background shutdown for zero-downtime config changes |
| **Fail-safe concealment** | Undercover mode | Default to maximum concealment; require explicit allowlisting to reveal |
| **Accept-or-Die** | Security dialog | Binary choice with no dismiss option prevents indefinite deferral of security decisions |
| **Three-tier feature gating** | DCE + env + GrowthBook | Compile-time, build-time, and runtime gates provide defense in depth |
| **Codename collision avoidance** | Buddy species encoding | `String.fromCharCode` prevents static string scanners from triggering on false positives |
| **Stale-while-revalidate** | Remote settings | Cache-first startup with background refresh minimizes user-visible latency |

---

> **Next**: [Episode 00: Overview — The 30,000-Foot View](./00-overview)
>
> **Previous**: [Episode 16: Infrastructure & Configuration](./16-infrastructure-config)
