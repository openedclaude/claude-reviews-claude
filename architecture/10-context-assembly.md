# Episode 10: Context Assembly — How Claude Code Builds Its Mind Before Every Turn

> **Source files**: `context.ts` (190 lines), `claudemd.ts` (1,480 lines), `systemPrompt.ts` (124 lines), `queryContext.ts` (180 lines), `attachments.ts` (3,998 lines), `prompts.ts` (915 lines), `analyzeContext.ts` (1,383 lines)
>
> **One-liner**: Before every API call, Claude Code assembles a multi-layered context from system prompts, memory files, git state, environment details, tool definitions, and per-turn attachments — each with its own priority, caching strategy, and injection path.

## Architecture Overview

```mermaid
graph TD
    subgraph "System Prompt (Cached)"
        SP["getSystemPrompt()<br/>prompts.ts"] --> INTRO["Identity + Rules"]
        SP --> TOOLS_SEC["Using Your Tools"]
        SP --> STYLE["Tone & Style"]
        SP --> BOUNDARY["── DYNAMIC BOUNDARY ──"]
        SP --> MEMORY["CLAUDE.md Memory"]
        SP --> ENV["Environment Info"]
        SP --> LANG["Language Preference"]
        SP --> MCP_I["MCP Instructions"]
    end

    subgraph "User Context (Memoized)"
        UC["getUserContext()<br/>context.ts"] --> CMD["getClaudeMds()"]
        CMD --> WALK["Directory Walk<br/>CWD → Root"]
        CMD --> RULES["Rules Processing<br/>.claude/rules/*.md"]
    end

    subgraph "System Context (Memoized)"
        SC["getSystemContext()<br/>context.ts"] --> GIT["getGitStatus()"]
        GIT --> BRANCH["Branch + Status + Log"]
    end

    subgraph "Per-Turn Attachments"
        ATT["getAttachments()<br/>attachments.ts"] --> FILES["@mentioned Files"]
        ATT --> IDE["IDE Selections"]
        ATT --> NESTED["Nested Memory"]
        ATT --> DIAG["Diagnostics"]
        ATT --> HOOKS["Hook Responses"]
        ATT --> TASKS["Task/Plan Reminders"]
        ATT --> SKILLS["Skill Discovery"]
        ATT --> MAIL["Teammate Mailbox"]
    end

    SP --> QE["QueryEngine.ask()"]
    UC --> QE
    SC --> QE
    ATT --> QE
    QE --> API["Anthropic API Call"]
```

---

## The Three Context Layers

Claude Code assembles context through three distinct layers, each with different lifetimes and caching strategies:

| Layer | Source | Lifetime | Cache Strategy |
|-------|--------|----------|----------------|
| **System Prompt** | `getSystemPrompt()` in prompts.ts | Per-session | Split at `DYNAMIC_BOUNDARY` — static prefix uses `scope: 'global'`, dynamic suffix per-session |
| **User/System Context** | `getUserContext()` + `getSystemContext()` in context.ts | Per-session (memoized) | `lodash/memoize` — computed once, cached for conversation duration |
| **Attachments** | `getAttachments()` in attachments.ts | Per-turn | Recomputed every turn with 1-second timeout |

---

## Layer 1: System Prompt — The Identity

`getSystemPrompt()` in `prompts.ts` (line 444) builds an array of prompt sections — not a single string, but an ordered list that gets concatenated at the API layer. Here's the assembly order:

### Static Sections (Globally Cacheable)

These sections are identical across all users and sessions:

1. **Identity** — `getSimpleIntroSection()`: "You are an interactive agent..."
2. **System Rules** — `getSimpleSystemSection()`: Tool permissions, system reminders, hooks
3. **Doing Tasks** — `getSimpleDoingTasksSection()`: Code style rules, security warnings, KISS principles
4. **Actions** — `getActionsSection()`: Reversibility analysis, blast radius awareness
5. **Using Tools** — `getUsingYourToolsSection()`: "Use FileRead instead of cat", parallel tool calls
6. **Tone & Style** — `getSimpleToneAndStyleSection()`: No emojis, file:line references
7. **Output Efficiency** — `getOutputEfficiencySection()`: ≤25 words between tool calls (ant-only)

### The Dynamic Boundary

```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

Everything before this marker can use `scope: 'global'` for cross-org prompt caching. Everything after is session-specific. Moving a section across this boundary changes caching behavior — the code has explicit warnings about this.

### Dynamic Sections (Per-Session)

After the boundary, sections are resolved through a registry system:

1. **Session Guidance** — Fork agent instructions, skill discovery, verification agent contract
2. **Memory** — `loadMemoryPrompt()`: CLAUDE.md files (see Layer 2)
3. **Environment** — Model name, CWD, platform, shell, git status, knowledge cutoff
4. **Language** — `"Always respond in {language}"`
5. **MCP Instructions** — Server-provided instructions (or delta attachments)
6. **Scratchpad** — Per-session temp directory path
7. **Token Budget** — "+500k" budget instructions (when active)

### System Prompt Priority Chain

```typescript
// In buildEffectiveSystemPrompt (systemPrompt.ts)
if (overrideSystemPrompt)      → [override]           // Loop mode
else if (coordinatorMode)      → [coordinator prompt]  // Swarm lead
else if (agentDefinition)      → [agent prompt]        // Custom agent replaces default
else if (customSystemPrompt)   → [custom]              // --system-prompt flag
else                           → [default sections]    // Normal operation
// appendSystemPrompt is always added at the end (except override)
```

This is a **replacement chain**, not a merge — only one "base" prompt wins.

---

## Layer 2: Memory Files (CLAUDE.md System)

The memory system (`claudemd.ts`, 1,480 lines) is Claude Code's most intricate context subsystem. It discovers, parses, and assembles instructions from multiple sources with strict priority ordering.

### Loading Order (Low → High Priority)

```
1. Managed    /etc/claude-code/CLAUDE.md       ← Organization policy
2. User       ~/.claude/CLAUDE.md              ← Private global rules
3. Project    CLAUDE.md, .claude/CLAUDE.md     ← Checked into repo (CWD → root walk)
4. Local      CLAUDE.local.md                  ← Private project rules (gitignored)
5. AutoMem    ~/.claude/memory/MEMORY.md       ← Auto-memory (agent-managed)
6. TeamMem    Shared team memory               ← Organization-synced (feature-gated)
```

Files loaded **later** have **higher priority** — the model pays more attention to them.

### The Directory Walk

For Project and Local files, the system performs an **upward directory walk** from CWD to filesystem root:

```typescript
let currentDir = originalCwd
while (currentDir !== parse(currentDir).root) {
  dirs.push(currentDir)
  currentDir = dirname(currentDir)
}
// Process from root downward to CWD (reverse order)
for (const dir of dirs.reverse()) {
  // CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md, CLAUDE.local.md
}
```

Directories closer to CWD are loaded **later** (higher priority). Within each directory, the system checks:

- `CLAUDE.md` — Project instructions
- `.claude/CLAUDE.md` — Alternative project instructions
- `.claude/rules/*.md` — Rule files (unconditional + conditional via frontmatter globs)
- `CLAUDE.local.md` — Local private instructions

### @include Directives

Memory files support recursive inclusion:

```markdown
@./relative/path.md
@~/home/path.md
@/absolute/path.md
```

The parser:
1. Lexes with `marked` (GFM disabled to prevent `~/path` becoming strikethrough)
2. Walks text tokens extracting `@path` patterns
3. Resolves to absolute paths with symlink handling
4. Recursively processes up to `MAX_INCLUDE_DEPTH = 5`
5. Tracks `processedPaths` set to prevent circular references

### Conditional Rules (Glob-Gated)

Rules in `.claude/rules/` can have frontmatter that restricts them to specific file paths:

```yaml
---
paths:
  - src/api/**
  - tests/api/**
---
These rules apply only when working with API files.
```

The system uses `picomatch` for glob matching and `ignore` for path filtering. Conditional rules are injected as **nested memory attachments** when a tool touches matching files.

### Content Processing Pipeline

Every memory file goes through:

```
Raw content
  → parseFrontmatter() — Extract paths, strip YAML block
  → stripHtmlComments() — Remove <!-- block comments --> (preserve inline)
  → truncateEntrypointContent() — Cap AutoMem/TeamMem files
  → Track contentDiffersFromDisk — Flag if content was transformed
```

Binary protection: a whitelist of 100+ text extensions (`TEXT_FILE_EXTENSIONS`) prevents loading images, PDFs, etc. into the context.

### Rendering into Context

```typescript
export const getClaudeMds = (memoryFiles: MemoryFileInfo[]): string => {
  const memories: string[] = []
  for (const file of memoryFiles) {
    const description = /* type-specific suffix */
    memories.push(`Contents of ${file.path}${description}:\n\n${content}`)
  }
  return `${MEMORY_INSTRUCTION_PROMPT}\n\n${memories.join('\n\n')}`
}
```

The instruction prompt: *"Codebase and user instructions are shown below. Be sure to adhere to these instructions. IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written."*

---

## Layer 3: Per-Turn Attachments

`getAttachments()` in `attachments.ts` (line 743) assembles context that changes every turn. It runs with a **1-second timeout** via `AbortController` to prevent blocking user input.

### Attachment Types (30+ Types)

The `Attachment` union type spans 700+ lines of type definitions. Major categories:

| Category | Types | Trigger |
|----------|-------|---------|
| **File Content** | `file`, `compact_file_reference`, `pdf_reference`, `already_read_file` | User @mentions a file |
| **IDE Integration** | `selected_lines_in_ide`, `opened_file_in_ide` | IDE sends selection/focus |
| **Memory** | `nested_memory`, `relevant_memories`, `current_session_memory` | Tool touches files beyond CWD |
| **Task Management** | `todo_reminder`, `task_reminder`, `plan_mode`, `verify_plan_reminder` | Periodic (every N turns) |
| **Hook System** | `hook_cancelled`, `hook_success`, `hook_non_blocking_error`, etc. | Hook execution results |
| **Skill System** | `skill_listing`, `skill_discovery`, `invoked_skills`, `dynamic_skill` | Skill matching + invocation |
| **Swarm** | `teammate_mailbox`, `team_context`, `teammate_shutdown_batch` | Multi-agent coordination |
| **Budget** | `token_usage`, `budget_usd`, `output_token_usage` | Token/cost tracking |
| **Tool Deltas** | `deferred_tools_delta`, `agent_listing_delta`, `mcp_instructions_delta` | Tool set changes mid-session |

### Reminder Systems

Several attachment types use turn-based scheduling:

```typescript
export const TODO_REMINDER_CONFIG = {
  TURNS_SINCE_WRITE: 10,    // Remind after 10 turns since last write
  TURNS_BETWEEN_REMINDERS: 10,  // Don't remind more often than 10 turns
}

export const PLAN_MODE_ATTACHMENT_CONFIG = {
  TURNS_BETWEEN_ATTACHMENTS: 5,
  FULL_REMINDER_EVERY_N_ATTACHMENTS: 5,  // Full reminder every 5th, sparse otherwise
}
```

### Relevant Memories (Auto-Memory Surfacing)

When AutoMem is enabled, `findRelevantMemories()` surfaces stored memories based on the current context:

```typescript
export const RELEVANT_MEMORIES_CONFIG = {
  MAX_SESSION_BYTES: 60 * 1024,  // 60KB cumulative cap per session
}
const MAX_MEMORY_LINES = 200      // Per-file line cap
const MAX_MEMORY_BYTES = 4096     // Per-file byte cap (5 × 4KB = 20KB/turn)
```

The memory surfacer pre-computes headers at attachment-creation time to avoid prompt cache busting from changing timestamps ("saved 3 days ago" → "saved 4 days ago").

---

## The Assembly Pipeline

When `QueryEngine.ask()` fires, context assembly follows this sequence:

```
1. fetchSystemPromptParts()  — Parallel: getSystemPrompt() + getUserContext() + getSystemContext()
2. buildEffectiveSystemPrompt()  — Apply priority chain (override > coordinator > agent > custom > default)
3. getAttachments()  — Parallel attachment computation with 1s timeout
4. normalizeMessagesForAPI()  — Convert messages + attachments to Anthropic format
5. microcompactMessages()  — Optional: compact old tool results (FRC)
6. API call  — system[]: prompt parts, messages[]: normalized messages
```

### The Cache Architecture

```
┌──────────────────────────────────┐
│  scope: 'global'                 │  ← Static prompt sections
│  (Cross-org, shared by all)      │     Identity, rules, tools guidance
├──────── DYNAMIC BOUNDARY ────────┤
│  scope: 'session'                │  ← Dynamic prompt sections
│  (Per-user, memoized)            │     Memory, env, language, MCP
├──────────────────────────────────┤
│  Ephemeral (per-turn)            │  ← Attachments
│  (Recomputed each turn)          │     Files, diagnostics, reminders
└──────────────────────────────────┘
```

---

## The /context Visualization

`analyzeContext.ts` (1,383 lines) powers the `/context` command — a real-time breakdown of what's in the context window. It counts tokens for each category:

- System prompt (per-section breakdown)
- Memory files (per-file token count)
- Built-in tools (always-loaded vs deferred)
- MCP tools (loaded vs deferred, per-server)
- Skills (frontmatter token estimates)
- Messages (tool calls vs results vs text)
- Autocompact buffer reservation

The total is compared against `getEffectiveContextWindowSize()` to show percentage utilization and a visual grid.

---

## Design Insights

### Why Memoize, Not Cache?

`getUserContext()` and `getSystemContext()` use `lodash/memoize` — computed once per session, never recomputed. This means:
- Git status is a **snapshot** from session start ("this status will not update during the conversation")
- Memory files are loaded once... unless explicitly cleared via `resetGetMemoryFilesCache('compact')` during compaction
- The cache is cleared on worktree enter/exit, settings sync, and `/memory` dialog

### The Attachment Timeout

```typescript
const abortController = createAbortController()
const timeoutId = setTimeout(ac => ac.abort(), 1000, abortController)
```

If attachment computation takes >1 second, it's aborted. This prevents slow file reads or MCP queries from blocking the user. Each attachment source is wrapped in a `maybe()` helper that catches errors and logs them silently.

### Memory File Change Detection

The `contentDiffersFromDisk` flag on `MemoryFileInfo` enables a clever optimization: when a file's injected content differs from disk (due to comment stripping, frontmatter removal, or truncation), the raw content is preserved alongside. This lets the file state cache track changes without triggering unnecessary re-reads.

---

## Component Summary

| Component | Lines | Role |
|-----------|-------|------|
| `prompts.ts` | 915 | System prompt assembly — static sections, dynamic registry, boundary marker |
| `claudemd.ts` | 1,480 | Memory file discovery, parsing, @include resolution, glob-gated rules |
| `attachments.ts` | 3,998 | Per-turn attachment computation — 30+ types, reminder scheduling |
| `context.ts` | 190 | Memoized git status + user context entry points |
| `systemPrompt.ts` | 124 | Priority chain: override > coordinator > agent > custom > default |
| `queryContext.ts` | 180 | Shared helpers for assembling cache-key prefix |
| `analyzeContext.ts` | 1,383 | /context command — token counting, category breakdown, grid visualization |

---

*Next: [Episode 11 — Compaction System →](11-compact-system.md)*

[← Episode 09 — Session Persistence](09-session-persistence.md) | [Episode 11 →](11-compact-system.md)
