# Plugin Loader Deep Dive — Full Loading Sequence

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. The Full Loading Sequence

From `src/plugins/loader.ts` (line 1397+):

```
loadOpenClawPlugins(options)
  │
  ├─ resolvePluginLoadCacheContext()        [normalize config, activation source, scope]
  ├─ getCachedPluginRegistry()             [check cache for existing registry]
  ├─ PluginLoadReentryError               [throw if load already in-flight]
  ├─ Clear prior plugin state             [if activating: agents, commands, handlers]
  │
  ├─ createPluginJitiLoader()             [JITI for .ts loading, with caching]
  │
  ├─ LAZY PROXY for runtime               [defer runtime property access]
  │    └─ resolvePluginRuntimeModulePath() [resolve runtime module path]
  │
  ├─ createPluginRegistry()               [create empty registry + API builder]
  │
  ├─ discoverOpenClawPlugins()             [find all candidates from configured paths]
  │
  ├─ loadPluginManifestRegistry()          [parse openclaw.plugin.json for each]
  │
  ├─ buildProvenanceIndex()               [path matching for install tracking]
  │
  ├─ Sort candidates by precedence:         [config > global > bundled > workspace]
  │
  ├─ Per-candidate processing:
  │    ├─ Validate config schema
  │    ├─ ensureBundledPluginRuntimeDeps()  [for bundled plugins]
  │    ├─ Determine registration mode: [full | setup-runtime | setup-only | null]
  │    ├─ For bundle-format: skip runtime, push capabilities
  │    ├─ For memory plugins: apply slot decision
  │    ├─ Load module via JITI
  │    ├─ For channel plugins with setupSource: handle split
  │    └─ Call register(api) or activate(api)
  │
  ├─ Memory slot validation               [warn if configured slot plugin not found]
  │
  ├─ Untracked plugin warning               [loaded but not in allowlist]
  │
  ├─ Store cache                           [registry + all plugin state snapshots]
  │
  └─ activatePluginRegistry()
       └─ setActivePluginRegistry()
       └─ initializeGlobalHookRunner()
```

---

## 2. Discovery

```typescript
discoverOpenClawPlugins()
// → Searches stock/global/workspace roots
// → Returns PluginCandidate[] with: root, dir, manifestPath, origin
// Origin types: "bundled" | "global" | "workspace" | "config"
// Bundled plugins skip ownership checks
```

Discovery roots resolved via `resolvePluginSourceRoots()`:
- **stock**: built-in bundled plugins
- **global**: `~/.openclaw/plugins/`
- **workspace**: project-specific plugins

---

## 3. Manifest Loading

```typescript
loadPluginManifest(rootDir, rejectHardlinks)
  → resolvePluginManifestPath()
  → openBoundaryFileSync()           [hardlink rejection, boundary safety]
  → JSON5.parse()                    [allows comments/trailing commas]
  → Validate: id (string), configSchema (object)
  → Normalize optional fields
  → Return PluginManifestLoadResult
```

**Manifest normalizations:**
- `kind`: string → array of `PluginKind`
- `channels`, `providers`, `cliBackends`: normalized lists
- `activation`: parsed into `PluginManifestActivation`
- `contracts`: parsed into `PluginManifestContracts`
- `channelConfigs`: parsed via `normalizeChannelConfigs()`

---

## 4. Lazy Runtime Proxy

```typescript
// LAZY_RUNTIME_REFLECTION_KEYS defines which properties are lazy
const LAZY_RUNTIME_REFLECTION_KEYS = [
  "subagent", "channel", "acp",
  "config", "agent", "system",
  "media", "tts", "stt", "events",
  "logging", "state", "modelAuth"
]

// Accessing any lazy key triggers module load
runtime.subagent.run(...)  // → loads runtime module first
```

---

## 5. Registration Flow

```typescript
createPluginRegistry()
  → Creates empty PluginRegistry
  → Sets up all registration functions
  → Returns registry

// Each registration function:
registerProvider(provider)
  → validateProvider()
  → check for duplicates
  → push to registry.providers[]
  → push diagnostic on conflict

registerTool(tool)
  → validateTool()
  → check for duplicates
  → push to registry.tools[]

registerChannel(channel)
  → normalizeRegisteredChannelPlugin()
  → push to registry.channels[]

registerHook(event, handler)
  → registerTypedHook(event, handler)
  → push to registry.typedHooks[]
```

---

## 6. Activation Planner

```typescript
resolveManifestActivationPluginIds(params)
  → For each trigger in params.triggers:
       kind: "command"  → checks plugin.activation.onCommands + commandAliases
       kind: "provider" → checks plugin.activation.onProviders + plugin.providers
       kind: "agentHarness" → checks plugin.activation.onAgentHarnesses
       kind: "channel"  → checks plugin.activation.onChannels
       kind: "route"    → checks plugin.activation.onRoutes
       kind: "capability" → checks plugin.activation.onCapabilities
  → Returns sorted plugin IDs
```

---

## 7. Bundle Format Detection

```typescript
detectBundleManifestFormat(rootDir)
  → Check .codex-plugin/plugin.json     → "codex"
  → Check .claude-plugin/plugin.json    → "claude"
  → Check .cursor-plugin/plugin.json    → "cursor"
  → Check for manifestless Claude:
       skills/, commands/, hooks/hooks.json, .mcp.json, .lsp.json
       → "claude" (manifestless)
```

---

## 8. Rollback on Error

```typescript
rollbackPluginGlobalSideEffects(pluginId)
// Clears:
// - Plugin commands
// - Plugin interactive handlers
// - Context engines for plugin:${pluginId}
// - Internal hooks (unregisters, restores previous registrations)
```

---

## 9. Key Plugin Type: PluginRuntime

```typescript
type PluginRuntime = PluginRuntimeCore & {
  subagent: {
    run(params: SubagentRunParams): Promise<SubagentRunResult>
    waitForRun(params: SubagentWaitParams): Promise<SubagentWaitResult>
    getSessionMessages(params): Promise<SubagentGetSessionMessagesResult>
    getSession(params): Promise<SubagentGetSessionResult>
    deleteSession(params): Promise<void>
  }
  channel: PluginRuntimeChannel
}

type SubagentRunParams = {
  sessionKey: string
  message: string
  provider?: string
  model?: string
  extraSystemPrompt?: string
  lane?: string
  deliver?: boolean
  idempotencyKey?: string
}
```