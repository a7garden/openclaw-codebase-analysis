# Memory System Architecture

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

The memory system provides persistent memory and embedding infrastructure for agents. It consists of:
- **Memory plugins** (`extensions/memory-*`) — file-backed, LanceDB, wiki-style
- **Memory-host-sdk** (`packages/memory-host-sdk/`) — SDK for embedding/compute backends
- **Memory capability** registration via the plugin API

---

## 2. Memory Plugins

### 2.1 memory-core

**Location:** `extensions/memory-core/`

File-backed memory with search tools:
- `memory_search` — search memory for relevant context
- `memory_get` — retrieve specific memory entries
- `memory_save` — save information to memory
- `memory_delete` — delete memory entries

Uses file-based storage (not vector DB) — good for structured/text memories.

### 2.2 memory-lancedb

**Location:** `extensions/memory-lancedb/`

LanceDB vector storage backend:
- Stores embeddings in LanceDB
- Supports semantic similarity search
- Better for large-scale unstructured data

### 2.3 memory-wiki

**Location:** `extensions/memory-wiki/`

Wiki-style memory with search — good for structured knowledge bases with hierarchical organization.

---

## 3. Memory Capability Registration

### 3.1 Registering Memory Capability

```typescript
// From extensions/memory-core/index.ts
export default definePluginEntry({
  id: "memory-core",
  register(api) {
    api.registerMemoryCapability({
      id: "memory-core",
      tools: [memorySearchTool, memoryGetTool, memorySaveTool, memoryDeleteTool],
      storage: fileBasedStorage,
      searchIndex: textSearchIndex,
    })
  }
})
```

### 3.2 MemoryCapabilityPlugin Interface

```typescript
interface MemoryCapabilityPlugin {
  id: string
  name: string
  tools: OpenClawTool[]
  storage: MemoryStorage
  searchIndex: MemorySearchIndex
}
```

---

## 4. Memory-Host SDK

**Location:** `packages/memory-host-sdk/`

SDK for memory/embeddings backend implementors.

### 4.1 Runtime Interfaces

```
runtime.ts          — Main runtime interface
runtime-core.ts     — Core runtime (embedding computation)
runtime-cli.ts      — CLI integration
runtime-files.ts    — File-based runtime
```

### 4.2 Engine Interfaces

```
engine.ts                    — Base engine interface
engine-foundation.ts        — Foundation engine
engine-storage.ts           — Storage engine
engine-embeddings.ts         — Embedding computation
engine-qmd.ts               — Query metadata
```

### 4.3 Batch Processing

```
host/batch-runner.ts   — Batch processing runner
host/batch-http.ts     — HTTP batch client
host/batch-status.ts   — Batch status tracking
```

### 4.4 Embeddings

```
host/embeddings.ts               — Embedding interface
host/embeddings-remote-client.ts — Remote embedding service client
host/embeddings-remote-fetch.ts  — Fetch-based remote client
```

---

## 5. Embedding Providers

### 5.1 Memory Embedding Provider

From `src/plugins/types.ts` — `registerMemoryEmbeddingProvider()`:

```typescript
api.registerMemoryEmbeddingProvider({
  id: "openai-embedding",
  computeEmbedding(text: string): Promise<number[]>
  computeEmbeddings(texts: string[]): Promise<number[][]>
})
```

### 5.2 Bundled Embedding Providers

| Provider | Plugin |
|----------|--------|
| OpenAI `text-embedding-3-large` | `extensions/openai/` |
| Voyage AI | `extensions/voyage/` |
| Cohere | `extensions/cohere/` |
| Amazon Bedrock embeddings | `extensions/amazon-bedrock/` |

---

## 6. Memory Tools

### 6.1 Tool Schema

```typescript
// memory_search tool
{
  name: "memory_search",
  description: "Search memory for relevant context",
  schema: {
    query: z.string(),
    limit: z.number().optional(),
    type: z.enum(["all", "episodic", "semantic"]).optional()
  }
}

// memory_get tool
{
  name: "memory_get",
  description: "Retrieve specific memory entries",
  schema: {
    id: z.string()
  }
}
```

### 6.2 Tool Execution Flow

```
Agent calls memory_search tool
  → Validate params against schema
  → Query search index (vector or text)
  → Return top-k results to agent
  → Agent uses results in context
```

---

## 7. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `extensions/memory-core/index.ts` | ~100 | File-backed memory plugin |
| `extensions/memory-lancedb/index.ts` | ~100 | LanceDB vector memory |
| `extensions/memory-wiki/index.ts` | ~100 | Wiki memory |
| `packages/memory-host-sdk/runtime.ts` | ~100 | Memory host runtime |
| `packages/memory-host-sdk/engine-embeddings.ts` | ~150 | Embedding engine |
| `src/plugins/types.ts` | ~1230 | MemoryEmbeddingProvider interface |