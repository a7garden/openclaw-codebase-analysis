# Packages Architecture

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

Three private npm packages are published from `packages/`:
- `@openclaw/plugin-sdk` — public plugin SDK
- `@openclaw/memory-host-sdk` — memory backend SDK
- `@openclaw/plugin-package-contract` — external plugin validation

---

## 2. @openclaw/plugin-sdk

**Location:** `packages/plugin-sdk/`
**Purpose:** The canonical public SDK for building OpenClaw plugins (both bundled and third-party)

### 2.1 Exports (40+ paths)

```json
// package.json exports
{
  "name": "@openclaw/plugin-sdk",
  "exports": {
    ".": "./dist/index.js",
    "./plugin-entry": "./dist/plugin-entry.js",
    "./provider-entry": "./dist/provider-entry.js",
    "./channel-entry-contract": "./dist/channel-entry-contract.js",
    "./channel-contract": "./dist/channel-contract.js",
    "./core": "./dist/core.js",
    "./provider-auth": "./dist/provider-auth.js",
    "./provider-catalog-shared": "./dist/provider-catalog-shared.js",
    "./provider-http": "./dist/provider-http.js",
    "./provider-model-types": "./dist/provider-model-types.js",
    "./provider-stream-family": "./dist/provider-stream-family.js",
    "./provider-tools": "./dist/provider-tools.js",
    "./provider-model-shared": "./dist/provider-model-shared.js",
    "./runtime": "./dist/runtime.js",
    "./agent-runtime": "./dist/agent-runtime.js",
    "./channel-runtime": "./dist/channel-runtime.js",
    "./acp-runtime": "./dist/acp-runtime.js",
    "./image-generation": "./dist/image-generation.js",
    "./music-generation": "./dist/music-generation.js",
    "./video-generation": "./dist/video-generation.js",
    "./speech": "./dist/speech.js",
    "./realtime-transcription": "./dist/realtime-transcription.js",
    "./realtime-voice": "./dist/realtime-voice.js",
    "./media-understanding": "./dist/media-understanding.js",
    "./memory-host-*": "./dist/memory-host-*.js",
    // ... 295 total subpaths
  }
}
```

### 2.2 Plugin Entry

```typescript
// packages/plugin-sdk/src/plugin-entry.ts (208 lines)
export function definePluginEntry(config: PluginEntryConfig): PluginEntry {
  return {
    id: config.id,
    name: config.name,
    register: config.register,
    // ...
  }
}
```

### 2.3 Provider Entry

```typescript
// packages/plugin-sdk/src/provider-entry.ts (168 lines)
export function defineSingleProviderPluginEntry(config: ProviderPluginConfig): PluginEntry {
  // Sugar for single-provider plugins
  // Automatically wires auth + catalog
}
```

### 2.4 Channel Entry Contract

```typescript
// packages/plugin-sdk/src/channel-entry-contract.ts (522 lines)
export function defineBundledChannelEntry(config: ChannelEntryConfig): ChannelEntry {
  // Lazy-loading channel entry
  // JITI for .ts loading in dev
  // Direct require for .js in dist/
}
```

### 2.5 Re-exports from Core

The package re-exports from `src/plugin-sdk/` in the monorepo:

```typescript
// packages/plugin-sdk/src/core.ts
export * from "../../../src/plugin-sdk/plugin-entry.js"
export * from "../../../src/plugin-sdk/provider-entry.js"
export * from "../../../src/plugin-sdk/channel-entry-contract.js"
export * from "../../../src/plugin-sdk/channel-contract.js"
// ... all public types
```

---

## 3. @openclaw/memory-host-sdk

**Location:** `packages/memory-host-sdk/`
**Purpose:** SDK for building memory/embedding backend implementations

### 3.1 Purpose

Plugin authors who want to build memory backends (like LanceDB, file-based, wiki-style) use this SDK to implement:
- Embedding computation
- Storage engines
- Batch processing
- Query handling

### 3.2 Exports

```json
// package.json
{
  "name": "@openclaw/memory-host-sdk",
  "exports": {
    "./runtime": "./dist/runtime.js",
    "./runtime-core": "./dist/runtime-core.js",
    "./runtime-cli": "./dist/runtime-cli.js",
    "./runtime-files": "./dist/runtime-files.js",
    "./engine": "./dist/engine.js",
    "./engine-foundation": "./dist/engine-foundation.js",
    "./engine-storage": "./dist/engine-storage.js",
    "./engine-embeddings": "./dist/engine-embeddings.js",
    "./engine-qmd": "./dist/engine-qmd.js",
    "./host/batch-runner": "./dist/host/batch-runner.js",
    "./host/batch-http": "./dist/host/batch-http.js",
    "./host/batch-status": "./dist/host/batch-status.js",
    "./host/embeddings": "./dist/host/embeddings.js",
    "./host/embeddings-remote-client": "./dist/host/embeddings-remote-client.js",
    "./host/embeddings-remote-fetch": "./dist/host/embeddings-remote-fetch.js"
  }
}
```

### 3.3 Runtime Interfaces

```typescript
// packages/memory-host-sdk/src/runtime.ts
export interface MemoryHostRuntime {
  storage: StorageEngine
  embeddings: EmbeddingsEngine
  query: QueryEngine
}

// packages/memory-host-sdk/src/runtime-core.ts
export interface MemoryHostRuntimeCore {
  computeEmbedding(text: string): Promise<number[]>
  computeEmbeddings(texts: string[]): Promise<number[][]>
}
```

### 3.4 Engine Interfaces

```typescript
// packages/memory-host-sdk/src/engine-foundation.ts
export interface MemoryEngineFoundation {
  initialize(): Promise<void>
  store(key: string, value: MemoryEntry): Promise<void>
  retrieve(key: string): Promise<MemoryEntry | null>
  delete(key: string): Promise<void>
  search(query: string, options?: SearchOptions): Promise<SearchResult[]>
}

// packages/memory-host-sdk/src/engine-embeddings.ts
export interface EmbeddingsEngine {
  computeEmbedding(text: string): Promise<number[]>
  computeEmbeddings(texts: string[]): Promise<number[][]>
  findSimilar(embedding: number[], options?: SimilarityOptions): Promise<MemoryEntry[]>
}
```

### 3.5 Batch Processing

```typescript
// packages/memory-host-sdk/src/host/batch-runner.ts
export class BatchRunner {
  constructor(options: BatchRunnerOptions)
  submit(job: BatchJob): Promise<BatchJobResult>
  getStatus(jobId: string): BatchJobStatus
  cancel(jobId: string): Promise<void>
}

// packages/memory-host-sdk/src/host/batch-http.ts
export class BatchHttpClient {
  constructor(baseUrl: string, auth?: AuthOptions)
  submitBatch(job: BatchJob): Promise<BatchSubmitResponse>
  getResults(jobId: string): Promise<BatchResults>
}
```

---

## 4. @openclaw/plugin-package-contract

**Location:** `packages/plugin-package-contract/`
**Purpose:** Validates that external (third-party) plugin `package.json` files conform to the expected structure

### 4.1 Purpose

When loading third-party plugins (not bundled), OpenClaw validates the plugin's `package.json` to ensure:
- Required fields are present (`name`, `version`, `main`, `openclaw`)
- Exports are correctly configured
- Peer dependencies are satisfied
- Manifest file exists

### 4.2 API

```typescript
// packages/plugin-package-contract/src/index.ts (91 lines)
export function validateExternalCodePluginPackageJson(
  packageJson: unknown
): ExternalCodePluginValidationResult {
  // Returns { valid: true } or { valid: false, errors: string[] }
}
```

### 4.3 Validation Rules

```typescript
interface ExternalCodePluginValidationResult {
  valid: boolean
  errors?: string[]
  warnings?: string[]
  metadata?: {
    id?: string
    version?: string
    manifestPath?: string
  }
}
```

---

## 5. Package Build

### 5.1 Build System

Packages are built using the monorepo's build system (`tsdown` or similar), outputting to `dist/` directories with proper ESM/CommonJS separation.

### 5.2 SDK Subpath Entries

**File:** `scripts/lib/plugin-sdk-entrypoints.json` (295 entries)

This JSON file lists all 295 public subpaths under `openclaw/plugin-sdk/`:

```json
[
  "openclaw/plugin-sdk",
  "openclaw/plugin-sdk/core",
  "openclaw/plugin-sdk/plugin-entry",
  "openclaw/plugin-sdk/provider-entry",
  // ... 291 more
]
```

Used by the build system to generate `package.json` `exports` field.

---

## 6. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `packages/plugin-sdk/package.json` | ~50 | Package exports (295 subpaths) |
| `packages/plugin-sdk/src/plugin-entry.ts` | 208 | `definePluginEntry()` |
| `packages/plugin-sdk/src/provider-entry.ts` | 168 | `defineSingleProviderPluginEntry()` |
| `packages/plugin-sdk/src/channel-entry-contract.ts` | 522 | `defineBundledChannelEntry()` |
| `packages/plugin-sdk/src/core.ts` | ~100 | Full SDK barrel re-export |
| `packages/memory-host-sdk/package.json` | ~30 | Memory host SDK exports |
| `packages/memory-host-sdk/src/runtime.ts` | ~100 | Runtime interface |
| `packages/memory-host-sdk/src/engine-embeddings.ts` | ~150 | Embeddings engine |
| `packages/memory-host-sdk/src/host/batch-runner.ts` | ~100 | Batch processing |
| `packages/plugin-package-contract/src/index.ts` | 91 | External plugin validation |
| `scripts/lib/plugin-sdk-entrypoints.json` | 295 entries | All SDK subpaths |