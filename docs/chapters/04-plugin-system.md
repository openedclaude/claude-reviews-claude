> 🌐 **Language**: English | [中文版 →](/zh-CN/chapters/04-plugin-system)
> 📖 **[Read Online →](https://openedclaude.github.io/claude-reviews-claude/chapters/04-plugin-system)** — Sidebar nav, dark mode & full-text search. Better than raw GitHub.


# Plugin System: 44 Files, Full Lifecycle Management

> **Source files**: `utils/plugins/` (44 files, 18,856 lines), `components/ManagePlugins.tsx` (2,089 lines)

## TL;DR

Claude Code has a full plugin ecosystem hidden under the hood — with a marketplace, dependency resolution, auto-update, blocklisting, zip caching, and hot-reload. It's far more sophisticated than the official docs suggest. The plugin loader alone (`pluginLoader.ts`) is 113K bytes — larger than most entire npm packages.

---

## 1. Plugin Architecture at a Glance

![04 plugin system 1](../assets/04-plugin-system-1.svg)

---

## 2. The Plugin Manifest

Every plugin must have a `.claude-plugin/plugin.json` manifest:

```typescript
// From schemas.ts (60K — the largest schema file in the codebase)
{
  "name": "my-plugin",
  "description": "What this plugin does",
  "version": "1.0.0",
  "commands": ["commands/*.md"],
  "agents": ["agents/*.md"],
  "hooks": { ... },
  "mcpServers": { ... },
  "skills": ["skills/*/SKILL.md"],
  "outputStyle": "styles/custom.md"
}
```

The schema validation in `schemas.ts` is **60,595 bytes** — more than most entire plugins. It validates everything from command frontmatter to MCP server configurations.

---

## 3. Key Files in the Plugin System

| File | Size | Purpose |
|------|------|---------|
| **`pluginLoader.ts`** | **113K** | Core loader — discovers, reads, validates, and registers all plugins |
| **`marketplaceManager.ts`** | **96K** | Marketplace browsing, search, installation from official catalog |
| **`schemas.ts`** | **61K** | Zod validation schemas for all plugin manifest formats |
| **`installedPluginsManager.ts`** | **43K** | Manages installed plugin state, activation, deactivation |
| **`loadPluginCommands.ts`** | **31K** | Parses markdown command files with YAML frontmatter |
| **`mcpbHandler.ts`** | **32K** | MCP bridge handler for plugin-provided MCP servers |
| **`validatePlugin.ts`** | **29K** | Multi-pass plugin validation before activation |
| **`pluginInstallationHelpers.ts`** | **21K** | Git clone, npm install, dependency resolution |
| **`mcpPluginIntegration.ts`** | **21K** | Integrates plugin-declared MCP servers into the tool pool |
| **`marketplaceHelpers.ts`** | **19K** | Helper functions for marketplace operations |
| **`dependencyResolver.ts`** | **12K** | Resolves plugin dependency graphs |
| **`zipCache.ts`** | **14K** | Caches downloaded plugins as zip files for offline use |

---

## 4. Plugin Lifecycle

### 4.1 Discovery

Plugins are discovered from multiple locations:

```
1. Built-in plugins    — bundled with the binary
2. Project plugins     — .claude/ directory in the project
3. User plugins        — ~/.config/claude-code/plugins/
4. Marketplace plugins — official GCS bucket catalog
```

### 4.2 Installation

<p align="center">
  <img src="../assets/04-plugin-system-2.svg" width="240">
</p>

### 4.3 Validation

`validatePlugin.ts` (29K) runs extensive checks:
- Manifest schema validation (Zod)
- Command file syntax validation
- Hook command safety checks
- MCP server configuration validation
- Circular dependency detection
- Version compatibility checks

### 4.4 Loading

`pluginLoader.ts` (113K — the second largest file in the codebase) handles:
- Parallel loading of all plugin components
- Hook registration
- Agent definition merging
- Command registration with deduplication
- MCP server startup
- Skill directory registration
- Error isolation (one plugin failing doesn't crash others)

---

## 5. Marketplace System

### Official Marketplace (`officialMarketplaceGcs.ts`)

The marketplace is a GCS (Google Cloud Storage) bucket serving a plugin catalog:

```typescript
// officialMarketplace.ts
const OFFICIAL_MARKETPLACE_NAMES = new Set([
  'official',     // Production marketplace
  'staging',      // Staging environment
])
```

Features:
- **Startup check** (`officialMarketplaceStartupCheck.ts`, 15.6K) — checks for plugin updates on launch
- **Auto-update** (`pluginAutoupdate.ts`, 9.8K) — background auto-update mechanism
- **Blocklist** (`pluginBlocklist.ts`, 4.5K) — remotely disable compromised plugins
- **Flagging** (`pluginFlagging.ts`, 5.8K) — flag plugins for review
- **Install counts** (`installCounts.ts`, 8.6K) — telemetry for marketplace popularity

### Zip Cache System

Downloaded plugins are cached as zip files to avoid re-downloading:

```typescript
// zipCache.ts — 14K
// Plugins are cached in ~/.config/claude-code/plugin-cache/
// Keyed by content hash for deduplication
// zipCacheAdapters.ts — 5.5K  
// Adapters for different storage backends
```

---

## 6. What Plugins Can Provide

### Agents

```typescript
// loadPluginAgents.ts — 12.8K
// Plugins define agents as markdown files:
// agents/code-reviewer.md → becomes available as subagent_type: "code-reviewer"
```

### Commands (Slash Commands)

```typescript
// loadPluginCommands.ts — 31K
// Markdown files with YAML frontmatter:
// commands/review-pr.md → becomes /review-pr
// Frontmatter supports: allowed_tools, model, description
```

### Hooks

```typescript
// loadPluginHooks.ts — 10.4K
// Plugin hooks.json → registered as PreToolUse/PostToolUse/etc hooks
// Plugin hooks run with workspace trust checks
```

### MCP Servers

```typescript
// mcpPluginIntegration.ts — 21K
// Plugins can declare MCP servers that auto-start
// mcpbHandler.ts — 32K  
// MCP bridge: proxies between plugin MCP servers and Claude Code
```

### Skills

```typescript
// Skills are markdown files (SKILL.md) with trigger patterns
// Auto-discovered from plugin skill directories
```

### Output Styles

```typescript
// loadPluginOutputStyles.ts — 5.9K
// Custom output formatting (explanatory, learning, etc.)
```

---

## 7. Security & Trust Model

### Plugin Policy

```typescript
// pluginPolicy.ts
// Plugins from untrusted sources require explicit user approval
// Managed plugins (from official marketplace) have elevated trust
```

### Blocklist

```typescript
// pluginBlocklist.ts
// Remote blocklist can disable plugins by ID
// Checked on every startup and plugin load
```

### Orphan Filter

```typescript
// orphanedPluginFilter.ts — 4K
// Detects plugins whose source repo was deleted
// Prevents dangling references
```

---

## 8. Skill System Architecture

**Source coordinates**: `src/skills/`, `src/commands/`

Skills are Claude Code's "prompt as code" system — each Skill is a Markdown file with YAML frontmatter that defines when and how a capability should be activated.

### 8.1 Six Source Layers

```typescript
// 源码位置: src/skills/types.ts
export type LoadedFrom =
  | 'commands_DEPRECATED'  // Legacy commands/ directory (migration path)
  | 'skills'               // .claude/skills/ directory
  | 'plugin'               // Installed via plugin
  | 'managed'              // Enterprise managed configuration
  | 'bundled'              // CLI built-in skills
  | 'mcp'                  // Provided by MCP server
```

Search paths by priority:

```
policySettings → /path/to/managed/.claude/skills/
userSettings   → ~/.claude/skills/
projectSettings → .claude/skills/
plugin         → from plugin installation
bundled        → compiled into the binary
mcp            → discovered at runtime from MCP servers
```

### 8.2 Skill Frontmatter

Every Skill is a Markdown file parsed through YAML frontmatter:

```typescript
export function parseSkillFrontmatterFields(frontmatter, markdownContent, resolvedName): {
  displayName: string | undefined
  description: string
  allowedTools: string[]        // Restrict available tools
  argumentHint: string          // Argument prompt hint
  argumentNames: string[]       // Named argument list
  whenToUse: string             // Model trigger condition
  version: string
  model: string                 // Can specify 'inherit'
  disableModelInvocation: boolean  // Manual-only trigger
  userInvocable: boolean         // Show in /skill list
  hooks: HooksSettings           // Associated hooks
  executionContext: 'fork'       // Run in isolated context
  agent: string                  // Bind to specific agent type
  effort: EffortValue            // Reasoning effort level
  shell: FrontmatterShell        // Shell execution config
}
```

### 8.3 Bundled Skills Registration

Built-in skills are registered programmatically at startup:

```typescript
// 源码位置: src/skills/initBundledSkills.ts
export function initBundledSkills(): void {
  registerUpdateConfigSkill()   // /update-config
  registerKeybindingsSkill()    // /keybindings
  registerVerifySkill()         // /verify
  registerDebugSkill()          // /debug
  registerSimplifySkill()       // /simplify
  registerBatchSkill()          // /batch
  registerStuckSkill()          // /stuck
  // Feature-gated:
  if (feature('AGENT_TRIGGERS')) registerLoopSkill()       // /loop
  if (feature('AGENT_TRIGGERS_REMOTE')) registerScheduleRemoteAgentsSkill()
  if (feature('BUILDING_CLAUDE_APPS')) registerClaudeApiSkill()  // /claude-api
}
```

### 8.4 Inline vs Fork Execution Context

Skills can execute in two contexts:

| Context | Behavior | Use Case |
|---------|----------|----------|
| **inline** | Injects prompt into current conversation | Simple commands, config changes |
| **fork** | Runs in an isolated sub-agent with independent token budget | Complex tasks, multi-step operations |

Fork execution creates a fresh query loop with its own message history, preventing Skill execution from polluting the main conversation context.

### 8.5 Token Budget Management

Skill listings consume ~1% of the context window:

```typescript
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01
export const MAX_LISTING_DESC_CHARS = 250  // Per-skill description limit

export function formatCommandsWithinBudget(commands, contextWindowTokens?): string {
  // 1. Try full descriptions
  // 2. Over budget? Bundled skills keep full descriptions, others truncated
  // 3. Extreme case? Non-bundled show only names
}
```

This three-tier degradation ensures that even with hundreds of installed Skills, the listing never overwhelms the context window.

### 8.6 Bundled Skill File Safety

When a bundled Skill carries reference files, extraction uses OS-level security:

```typescript
// O_NOFOLLOW | O_EXCL prevents symlink attacks
const SAFE_WRITE_FLAGS = fsConstants.O_WRONLY | O_CREAT | O_EXCL | O_NOFOLLOW

// Path traversal validation
function resolveSkillFilePath(baseDir: string, relPath: string): string {
  const normalized = normalize(relPath)
  if (isAbsolute(normalized) || normalized.split(pathSep).includes('..')) {
    throw new Error(`bundled skill file path escapes skill dir: ${relPath}`)
  }
  return join(baseDir, normalized)
}
```

---

## 9. Built-in Plugin Registration

**Source coordinates**: `src/plugins/`

### 9.1 Plugin Definition Type

```typescript
export type BuiltinPluginDefinition = {
  name: string
  description: string
  version: string
  defaultEnabled?: boolean
  isAvailable?: () => boolean   // Environment detection
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: McpServerConfig[]
}
```

Each built-in plugin is a **triple** of skills + hooks + MCP servers. The `isAvailable` function enables environment-aware activation (e.g., a JetBrains-specific plugin only activates when JetBrains IDE is detected).

### 9.2 Enable/Disable Logic

```typescript
export function getBuiltinPlugins(): { enabled: LoadedPlugin[], disabled: LoadedPlugin[] } {
  for (const [name, definition] of BUILTIN_PLUGINS) {
    // 1. Skip if environment check fails
    if (definition.isAvailable && !definition.isAvailable()) continue

    // 2. Check user setting override
    const userSetting = settings?.enabledPlugins?.[pluginId]

    // 3. Resolve: explicit setting > defaultEnabled > true
    const isEnabled = userSetting !== undefined
      ? userSetting === true
      : (definition.defaultEnabled ?? true)
  }
}
```

### 9.3 Plugin → Skill → Command Conversion

```typescript
// Plugin-provided skills become slash commands
export function getBuiltinPluginSkillCommands(): Command[] {
  const { enabled } = getBuiltinPlugins()
  for (const plugin of enabled) {
    const definition = BUILTIN_PLUGINS.get(plugin.name)
    if (!definition?.skills) continue
    for (const skill of definition.skills) {
      commands.push(skillDefinitionToCommand(skill))
    }
  }
}
```

---

## 10. MCP Integration Deep Dive

**Source coordinates**: `src/services/mcp/client.ts`, `src/utils/plugins/mcpPluginIntegration.ts`

### 10.1 Six Transport Types

MCP servers connect through six transport mechanisms:

| Transport | Protocol | Use Case |
|-----------|----------|----------|
| `stdio` | stdin/stdout | Local CLI tools |
| `sse` | Server-Sent Events | Remote HTTP servers |
| `http` | HTTP POST (streamable) | Stateless API servers |
| `ws` | WebSocket | Bidirectional streaming |
| `sdk` | In-process SDK | Same-process tools |
| `sse-ide` | SSE via IDE proxy | IDE-bridged servers |

### 10.2 Tool Discovery → Tool Object Conversion

When an MCP server registers tools, they are converted into Claude Code's internal `Tool` interface:

```
MCP Server Tool Discovery
  ↓
listTools() response
  ↓
Convert each MCP tool → Tool object:
  - name: "mcp__{serverName}__{toolName}"
  - inputSchema: from MCP tool schema
  - checkPermissions: checks mcp permission rules
  - call(): proxied to MCP server
  ↓
Injected into assembleToolPool() with stable ordering
```

### 10.3 MCPConnectionManager

A React context component that manages MCP server lifecycle:

```
Session start → discover MCP servers from settings + plugins
  ↓
Parallel connection → health check → tool registration
  ↓
Mid-session changes → incremental reconnect
  ↓
Session end → graceful disconnect all
```

---

## 11. Background Installation Manager

**Source coordinates**: `src/utils/plugins/pluginInstallationManager.ts`

The `PluginInstallationManager` reconciles declared plugins with actual installations:

```
Declared plugins (from settings + marketplace)
  ↓
diff against actually installed plugins
  ↓
Missing? → Install (git clone/zip download)
Extra? → Prompt user for cleanup
Updated? → Auto-update if marketplace source
  ↓
Report progress via React state
```

Key design:
- **Reconciliation model**: Instead of imperative "install this," the system declaratively states "these plugins should exist" and the manager handles the delta
- **Progress feedback**: Each installation step emits progress events consumed by the ManagePlugins UI
- **Error isolation**: One plugin failing to install doesn't block others

---

## Transferable Design Patterns

> The following patterns from the Plugin System can be directly applied to any extensible application architecture.

### Pattern 1: Isolation by Default

Each plugin loads in isolation. One plugin crashing doesn't affect others. The loader wraps each plugin load in try/catch and reports failures individually.

### Pattern 2: Multi-Source Discovery with Priority

Plugins from different sources (built-in, project, user, marketplace) are merged with clear priority rules. Project-local plugins override user-level ones.

### Pattern 3: Content-Addressed Caching

The zip cache uses content hashes as keys, enabling deduplication across versions and users.

### Pattern 4: Prompt as Code (Skills)

Skills turn Markdown files with YAML frontmatter into executable capabilities. The frontmatter provides structured metadata (triggers, tools, models) while the Markdown body provides the prompt content. This makes Skills version-controllable, shareable, and composable.

### Pattern 5: Declarative Reconciliation

The `PluginInstallationManager` follows a declarative "desired state → actual state → diff → reconcile" model, borrowed from Kubernetes operators. This is more robust than imperative install/uninstall sequences because it's self-healing.

---

## Summary

| Aspect | Detail |
|--------|--------|
| **Total code** | 44 files, 18,856 lines in `utils/plugins/` alone |
| **Largest files** | `pluginLoader.ts` (113K), `marketplaceManager.ts` (96K) |
| **Plugin sources** | Built-in, project, user, official marketplace (GCS) |
| **Components** | Agents, Commands, Hooks, Skills, MCP Servers, Output Styles |
| **Marketplace** | GCS bucket catalog, auto-update, blocklist, install counts |
| **Security** | Workspace trust, validation, blocklist, orphan detection |
| **Caching** | Content-addressed zip cache for offline use |

---

**Previous**: [← 03 — Multi-Agent Coordinator](./03-coordinator)
**Next**: [→ 05 — Hook System](./05-hook-system)
