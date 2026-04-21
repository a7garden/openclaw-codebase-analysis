# Plugin System Architecture

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

The plugin system is OpenClaw's primary extension mechanism. It is **manifest-first** — plugin metadata (stored in `openclaw.plugin.json`) is read and processed before any plugin code executes. This enables config validation, activation planning, and setup workflows without loading the plugin runtime.

There are ~130 bundled plugins in `extensions/`, and the architecture supports third-party plugins via the same contracts.

---

## 2. Plugin Manifest

**Manifest file:** `openclaw.plugin.json` (defined by `PLUGIN_MANIFEST_FILENAME` in `src/plugins/manifest.ts`)

The manifest is the **control plane entry point** for every plugin. It is read and validated before the plugin's TypeScript/JavaScript code is ever loaded.

### 2.1 Key Manifest Fields

```typescript
// From src/plugins/manifest-types.ts
interface PluginManifest {
  id: string
  configSchema: OpenClawPluginConfigSchema   // Zod-like validation
  providers?: string[]                        // Provider IDs owned by this plugin
  modelSupport?: {
    modelPrefixes?: string[]                   // Cheap prefix-based model ownership
  }
  providerEndpoints?: ProviderEndpoint[]       // hostname/baseUrl matching
  activation?: PluginManifestActivation        // Lazy activation triggers
  setup?: PluginSetupMetadata                  // Cheap setup metadata
  secrets?: PluginSecretSpec                   // Secret input paths
  configContracts?: PluginConfigContract[]     // Dangerous flags
  uiHints?: UiHintsPerField                    // UI labels, help text, advanced markers
}
```

### 2.2 Config Schema

The config schema uses a Zod-like structure with:
- `safeParse()` / `validate()` methods
- `uiHints` — per-field labels, help text, advanced markers
- `jsonSchema` — JSON Schema export for tooling

Example field schema entry:
```typescript
{
  key: "apiKey",
  type: "secret",
  label: "API Key",
  description: "Your OpenAI API key",
  required: true,
  advanced: false
}
```

### 2.3 Activation Triggers

```typescript
type PluginManifestActivation =
  | { kind: "provider", provider: string }
  | { kind: "channel", channel: string }
  | { kind: "capability", capability: "provider" | "channel" | "memory" | "tool" }
  | { kind: "onRoutes" }
  | { kind: "onCommands" }
  | { kind: "onAgentHarnesses" }
```

The activation planner (`src/plugins/activation-planner.ts`) reads these triggers and decides which plugins to load — **no runtime execution needed**.

---

## 3. Plugin Loading

### 3.1 Loading Flow

```
loadPlugins(PluginLoadOptions)
  │
  ├── discoverOpenClawPlugins()
  │     Searches directories for openclaw.plugin.json files
  │     Handles multiple formats: openclaw, codex, claude, cursor bundles
  │
  ├── loadPluginManifestRegistry()
  │     Parses all manifest files
  │     Validates config schemas
  │     Builds manifest registry
  │
  ├── createPluginRegistry()
  │     Builds registry with activation state
  │
  ├── buildPluginApi()
  │     Constructs OpenClawPluginApi for each plugin
  │     Injects runtime helpers
  │
  └── plugin.register() / plugin.activate()
        Plugin code runs and registers capabilities
```

### 3.2 Bundled Plugin Directory Resolution

From `src/plugins/bundled-dir.ts:193`:

```typescript
resolveBundledPluginsDir()
  1. Check OPENCLAW_BUNDLED_PLUGINS_DIR env override
  2. Try dist-runtime/extensions (local source checkout staging)
  3. Try dist/extensions (built output)
  4. Try extensions (source for dev)
  5. For bun --compile: sibling extensions next to executable
  6. Walk up from this module's location
```

### 3.3 Bundle Format Support

The bundle manifest handler (`src/plugins/bundle-manifest.ts:150+`) handles:
- **Native OpenClaw:** `openclaw.plugin.json`
- **Codex bundles:** `.codex-plugin/plugin.json`
- **Claude bundles:** `.claude-plugin/plugin.json`
- **Cursor bundles:** `.cursor-plugin/plugin.json`

### 3.4 Lazy Loading Architecture

Channel plugins use **four separate lazy module references**:

```typescript
// From src/plugin-sdk/channel-entry-contract.ts:522
defineBundledChannelEntry({
  id: "telegram",
  name: "Telegram",
  description: "Telegram channel plugin",
  importMetaUrl: import.meta.url,
  plugin: { specifier: "./channel-plugin-api.js", exportName: "telegramPlugin" },
  secrets: { specifier: "./secret-contract-api.js", exportName: "channelSecrets" },
  runtime: { specifier: "./runtime-api.js", exportName: "setTelegramRuntime" },
  accountInspect: { specifier: "./account-inspect-api.js", exportName: "inspectTelegramReadOnlyAccount" },
})
```

These are loaded **on-demand** via JITI (for `.ts` in dev) or direct `require` (for `.js` in `dist/`).

### 3.5 JITI Caching

JITI is used for loading `.ts` files in development mode, with a Windows-specific `nodeRequire` fast path for `.js` files in the `dist/` output directory.

---

## 4. Plugin API Surface

### 4.1 OpenClawPluginApi

Full interface from `src/plugins/types.ts:1892-2046`:

```typescript
export type OpenClawPluginApi = {
  // Identity
  id: string
  name: string
  version: string
  description: string
  source: "bundled" | "external"
  rootDir: string
  registrationMode: "standard" | "strict"

  // Config
  config: ResolvedPluginConfig
  pluginConfig: PluginConfigResolved

  // Runtime
  runtime: PluginRuntime

  // Registration methods
  registerTool(tool: OpenClawTool): void
  registerHook(hook: string, handler: HookHandler): void
  registerHttpRoute(method, path, handler): void
  registerChannel(channel: ChannelPlugin): void
  registerGatewayMethod(name, handler): void
  registerCli(command: CliCommandDefinition): void
  registerReload(onReload: () => void): void
  registerService(service: ServiceDefinition): void

  // Provider registration
  registerProvider(provider: ProviderPlugin): void
  registerCliBackend(backend: CliBackendPlugin): void
  registerSpeechProvider(provider: SpeechProviderPlugin): void
  registerRealtimeTranscriptionProvider(provider: RealtimeTranscriptionProviderPlugin): void
  registerRealtimeVoiceProvider(provider: RealtimeVoiceProviderPlugin): void
  registerMediaUnderstandingProvider(provider: MediaUnderstandingProviderPlugin): void
  registerImageGenerationProvider(provider: ImageGenerationProviderPlugin): void
  registerMusicGenerationProvider(provider: MusicGenerationProviderPlugin): void
  registerVideoGenerationProvider(provider: VideoGenerationProviderPlugin): void
  registerWebFetchProvider(provider: WebFetchProviderPlugin): void
  registerWebSearchProvider(provider: WebSearchProviderPlugin): void
  registerMemoryEmbeddingProvider(provider: MemoryEmbeddingProviderPlugin): void

  // Memory & agents
  registerMemoryCapability(capability: MemoryCapabilityPlugin): void
  registerAgentHarness(harness: AgentHarnessPlugin): void

  // Context engines
  registerContextEngine(engine: ContextEnginePlugin): void

  // Commands
  registerCommand(command: CommandPlugin): void

  // Hook registration
  on(event: string, handler: HookHandler): void

  // Secrets
  getSecretInputSecret(name: string): Promise<string | null>
}
```

### 4.2 PluginRuntime Interface

From `src/plugins/runtime/types.ts:72`:

```typescript
export type PluginRuntime = {
  // Subagent runtime access
  subagent: SubagentRuntime

  // Channel runtime access
  channel: ChannelRuntime

  // ACP runtime access
  acp: AcpRuntime
}
```

---

## 5. Plugin Shapes

From `docs/plugins/architecture.md:72-85`:

| Shape | Description | Example |
|-------|-------------|---------|
| **plain-capability** | Single capability type | `mistral` (provider-only) |
| **hybrid-capability** | Multiple capability types | `openai` (text + speech + media + image) |
| **hook-only** | Registers only hooks, no capabilities | Some diagnostic plugins |
| **non-capability** | Registers tools/commands/services, no capabilities | `browser` tool |

---

## 6. Provider Plugin Entry

### 6.1 definePluginEntry

For multi-capability plugins, use `definePluginEntry()` from `openclaw/plugin-sdk/plugin-entry`:

```typescript
// From extensions/openai/index.ts:21-52
export default definePluginEntry({
  id: "openai",
  name: "OpenAI Provider",
  description: "Bundled OpenAI provider plugins",
  register(api) {
    api.registerProvider(buildProviderWithPromptContribution(buildOpenAIProvider()))
    api.registerProvider(buildProviderWithPromptContribution(buildOpenAICodexProviderPlugin()))
    api.registerMemoryEmbeddingProvider(openAiMemoryEmbeddingProviderAdapter)
    api.registerImageGenerationProvider(buildOpenAIImageGenerationProvider())
    api.registerRealtimeTranscriptionProvider(buildOpenAIRealtimeTranscriptionProvider())
    api.registerRealtimeVoiceProvider(buildOpenAIRealtimeVoiceProvider())
    api.registerSpeechProvider(buildOpenAISpeechProvider())
    api.registerMediaUnderstandingProvider(openaiMediaUnderstandingProvider)
    api.registerMediaUnderstandingProvider(openaiCodexMediaUnderstandingProvider)
    api.registerVideoGenerationProvider(buildOpenAIVideoGenerationProvider())
  },
})
```

### 6.2 defineSingleProviderPluginEntry

For single-provider plugins, use the sugar from `openclaw/plugin-sdk/provider-entry`:

```typescript
export default defineSingleProviderPluginEntry({
  id: "anthropic",
  name: "Anthropic",
  description: "Anthropic model provider",
  provider: buildAnthropicProvider(),
})
```

This provides auth + catalog wiring automatically.

---

## 7. Channel Plugin Entry

For channel plugins, use `defineBundledChannelEntry()` from `openclaw/plugin-sdk/channel-entry-contract.ts:522`:

```typescript
// From extensions/telegram/index.ts
export default defineBundledChannelEntry({
  id: "telegram",
  name: "Telegram",
  description: "Telegram channel plugin",
  importMetaUrl: import.meta.url,
  plugin: { specifier: "./channel-plugin-api.js", exportName: "telegramPlugin" },
  secrets: { specifier: "./secret-contract-api.js", exportName: "channelSecrets" },
  runtime: { specifier: "./runtime-api.js", exportName: "setTelegramRuntime" },
  accountInspect: { specifier: "./account-inspect-api.js", exportName: "inspectTelegramReadOnlyAccount" },
})
```

---

## 8. SDK Boundary Rules

### 8.1 What Extensions Can Import

```
✓ openclaw/plugin-sdk/*           (public SDK only)
✓ ./api.ts                        (extension's own public barrel)
✓ ./runtime-api.ts                (extension's own runtime seam)
✓ Relative paths within extension

✗ src/**                          (core internals)
✗ src/channels/**                 (channel internals)
✗ src/plugin-sdk-internal/**      (internal SDK)
✗ extensions/*/src/**             (another extension's internals)
✗ Paths outside extension root    (except openclaw/plugin-sdk/*)
```

### 8.2 What Core Can Import

```
✗ extensions/*/src/**            (plugin internals)
✗ Extensions' onboard.js          (plugin-specific)
✓ Plugin api.ts / public SDK      (generic contract)
✓ Plugin manifest metadata        (control plane)
```

---

## 9. Plugin Discovery

### 9.1 Discovery Process

From `src/plugins/loader.ts:400+`:

1. **Scan directories** — walk plugin directories looking for `openclaw.plugin.json`
2. **Parse manifests** — extract metadata, validate schema structure
3. **Build index** — map plugin IDs to manifest locations

### 9.2 Public Surface Runtime

From `src/plugins/public-surface-runtime.ts:80+`:

Collects public surface artifacts from plugin directories:
- Top-level `.ts`, `.mts`, `.js`, `.mjs` files
- Excludes: tests, `.d.ts`, `config-api.*`

---

## 10. Hook System

Plugins can register hooks via `api.on(event, handler)` or `api.registerHook()`:

```typescript
// From src/plugins/hook-types.ts
type PluginHookName =
  | "onAgentStart"
  | "onAgentStop"
  | "onMessageSend"
  | "onMessageReceive"
  | "onApprovalRequest"
  // ... extensive list
```

Internal hooks are defined in `src/hooks/` with hook runners in `src/hooks/` as well.

---

## 11. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/plugins/types.ts` | ~1230 | Full plugin API surface, provider/channel/memory type definitions |
| `src/plugins/manifest.ts` | ~200+ | `PLUGIN_MANIFEST_FILENAME`, manifest parsing |
| `src/plugins/manifest-types.ts` | ~230 | PluginFormat, PluginBundleFormat, config schema types |
| `src/plugins/loader.ts` | 400+ | Main plugin loading orchestration |
| `src/plugins/activation-planner.ts` | 80+ | Manifest-first activation planning |
| `src/plugins/bundled-dir.ts` | 193 | Bundled plugins directory resolution |
| `src/plugins/bundle-manifest.ts` | 150+ | Multi-format bundle manifest handling |
| `src/plugins/public-surface-runtime.ts` | 80+ | Public surface artifact collection |
| `src/plugin-sdk/plugin-entry.ts` | 208 | `definePluginEntry()` |
| `src/plugin-sdk/provider-entry.ts` | 168 | `defineSingleProviderPluginEntry()` |
| `src/plugin-sdk/channel-entry-contract.ts` | 522 | `defineBundledChannelEntry()` with lazy loading |
| `src/plugin-sdk/core.ts` | 150+ | Full SDK barrel re-export |
| `src/plugins/runtime/types.ts` | 72 | `PluginRuntime` interface |
| `src/plugins/hook-types.ts` | ~150 | Hook type definitions |