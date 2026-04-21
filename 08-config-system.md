# Configuration System Architecture

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

The configuration system (`src/config/`) handles all configuration for OpenClaw — from the user-facing config file (`~/.openclaw/config.json`) to plugin-specific configuration schemas and validation.

It is a **manifest-first** system: plugin config schemas are declared in `openclaw.plugin.json` and validated before plugin runtime loads.

---

## 2. Config File Structure

### 2.1 Main Config File

Location: `~/.openclaw/config.json`

```typescript
interface OpenClawConfig {
  version: number
  agents?: Record<AgentId, AgentConfig>
  channels?: Record<ChannelId, ChannelConfig>
  providers?: Record<ProviderId, ProviderConfig>
  plugins?: Record<PluginId, PluginConfig>
  gateway?: GatewayConfig
  logging?: LoggingConfig
  // ... more
}
```

### 2.2 Agent Config

```typescript
interface AgentConfig {
  model?: string           // e.g., "claude-3-5-sonnet-20241022"
  provider?: string        // e.g., "anthropic"
  systemPrompt?: string
  authProfile?: string
  maxTokens?: number
  temperature?: number
  // ... more
}
```

---

## 3. Config Schema

### 3.1 Schema Definition

From `src/plugins/manifest-types.ts` — `OpenClawPluginConfigSchema`:

```typescript
interface OpenClawPluginConfigSchema {
  safeParse(value: unknown): ParseResult
  validate(value: unknown): ValidateResult
  uiHints?: UiHintsPerField
  jsonSchema?: object      // JSON Schema export for tooling
}
```

### 3.2 Field Schema

```typescript
interface ConfigFieldSchema {
  key: string
  type: "string" | "number" | "boolean" | "secret" | "object" | "array"
  label?: string
  description?: string
  required?: boolean
  default?: unknown
  advanced?: boolean       // Marked as "advanced" in UI
  secret?: boolean         // Masked in UI
  validate?: (value: unknown) => ValidationResult
}
```

### 3.3 UI Hints

Per-field UI metadata for config forms:

```typescript
interface UiHintsPerField {
  [fieldKey: string]: {
    label?: string
    description?: string
    helpText?: string
    advanced?: boolean
    secret?: boolean
    group?: string         // Field grouping in UI
    order?: number         // Display order
  }
}
```

---

## 4. Config Loading

### 4.1 Loading Process

```
Config load sequence:
  1. Read ~/.openclaw/config.json
  2. Validate against base schema
  3. Load plugin manifests
  4. Validate plugin configs against manifest schemas
  5. Normalize values (apply defaults, coerce types)
  6. Resolve secrets (fetch from ~/.openclaw/credentials/)
  7. Provide to plugin runtime
```

### 4.2 Config Normalization

From `src/config/` — config values are normalized:
- Apply default values
- Coerce string values to correct types
- Validate against schema
- Report errors for invalid config

### 4.3 Secrets Resolution

Plugin config can reference secrets stored separately:

```typescript
// In config.json
{
  "plugins": {
    "openai": {
      "apiKeyRef": "file:openai.apiKey"
    }
  }
}

// Resolved from ~/.openclaw/credentials/openai.apiKey
```

---

## 5. Plugin Config Contracts

### 5.1 Dangerous Flags

From `src/plugins/manifest-types.ts`:

```typescript
interface PluginConfigContract {
  key: string
  dangerous: boolean
  description: string
}
```

Some config fields are marked as "dangerous" (e.g., `allowUnsafeOldCryptography`) and require explicit user acknowledgment.

### 5.2 Secret Input Paths

```typescript
interface PluginSecretSpec {
  fields: string[]
  description: string
}
```

Declares which config fields contain secrets and should be masked/resolved from credentials store.

---

## 6. Config Migration

### 6.1 Legacy Config Repair

From `AGENTS.md` — legacy config repair prefers doctor/fix paths over startup/load-time core migrations:

```typescript
// On startup, detect old config format
// Run migration via openclaw doctor --fix
// Do not silently migrate at startup
```

### 6.2 Doctor Integration

`openclaw doctor` command detects and fixes:
- Deprecated config keys
- Invalid plugin configs
- Missing required fields
- Schema mismatches

---

## 7. Config API (Gateway)

### 7.1 Gateway Config Methods

| Method | Purpose |
|--------|---------|
| `config.get` | Get full config or specific section |
| `config.patch` | Partial update (merge) |
| `config.set` | Set specific value |
| `config.validate` | Validate without applying |

### 7.2 Config Hash

Gateway config operations include a `baseHash` parameter for optimistic locking:

```typescript
config.patch({ raw: {...}, baseHash: "abc123" })
// If baseHash doesn't match current config, reject
```

---

## 8. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/config/` | ~400 | Config types, schema, loading, normalization |
| `src/plugins/manifest-types.ts` | ~230 | Plugin config schema types, uiHints, contracts |
| `src/plugins/manifest.ts` | ~200 | Manifest parsing, `PLUGIN_MANIFEST_FILENAME` |
| `src/gateway/server-methods/config.ts` | ~150 | Gateway config methods |