# Agent System Architecture

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

The agent system (`src/agents/`) is the runtime that executes AI agents — managing model selection, provider connectivity, tool registration, and the inference loop that processes messages and produces responses.

Agents are the core product of OpenClaw: they connect to messaging channels (so users can talk to them) and to AI model providers (so they can think/reason).

---

## 2. Agent Runtime Entry Points

### 2.1 Embedded Runner

**Directory:** `src/agents/pi-embedded-runner/`

The embedded runner runs agents in-process. This is the primary agent runtime — agents don't run as separate processes but are loaded into the same Node.js process as the core.

### 2.2 Spawn Management

**File:** `src/agents/acp-spawn.ts` (200+ lines)

ACP subagent spawning — creates subagents that can be used for parallel task execution, tool use, or delegation.

```typescript
spawnAcpSubagent(ctx: SpawnContext): Promise<SubagentHandle>
```

Subagents are spawned with their own configuration, provider, and toolset, but share the ACP session infrastructure.

### 2.3 Harness Registry

**Directory:** `src/agents/harness/`

Agent harnesses are the pluggable execution engines for different agent types. The registry manages which harnesses are available and how to instantiate them.

---

## 3. Agent Configuration

### 3.1 Runtime Config

**File:** `src/agents/agent-runtime-config.ts` (150+ lines)

```typescript
interface AgentRuntimeConfig {
  model: string                      // Model ID (e.g., "claude-3-5-sonnet-20241022")
  provider: string                   // Provider ID (e.g., "anthropic")
  maxTokens?: number
  temperature?: number
  topP?: number
  systemPrompt?: string
  authProfile?: string               // Per-agent auth override
  // ... more
}
```

### 3.2 API Key Rotation

**File:** `src/agents/api-key-rotation.ts` (100+ lines)

Handles automatic API key rotation for providers that support it:

```typescript
// Rotation strategies
rotateApiKey(provider: string): Promise<string>
// With fallback chains when primary key fails
```

---

## 4. Transport Streams

### 4.1 Anthropic Transport

**File:** `src/agents/anthropic-transport-stream.ts` (200+ lines)

Connects to **Anthropic's API** directly:
- Request signing (Authorization header with API key)
- Streaming response parsing (SSE)
- Error handling and retry with backoff
- Rate limiting

### 4.2 Anthropic Vertex Transport

**File:** `src/agents/anthropic-vertex-stream.ts` (150+ lines)

Connects to Anthropic via **Google Vertex AI**:
- GCP authentication (OAuth or service account)
- Vertex-specific endpoint handling
- Same streaming/SSE handling as direct Anthropic

### 4.3 Transport Interface

Both transports implement a common interface:

```typescript
interface TransportStream {
  send(messages: Message[], options: InferenceOptions): Promise<InferenceResult>
  sendStream(messages: Message[], options: InferenceOptions): AsyncGenerator<InferenceChunk>
  close(): void
}
```

---

## 5. Tool System

### 5.1 Tool Registration

**Directory:** `src/agents/tools/`

Tools are capabilities that agents can call during inference (like `browser.navigate`, `memory.search`, etc.). Tools are registered via the plugin API:

```typescript
api.registerTool({
  name: "browser_navigate",
  description: "Navigate to a URL in the browser",
  schema: z.object({ url: z.string() }),
  handler: async (params) => { /* ... */ }
})
```

### 5.2 Tool Schema Compatibility

From `src/plugins/types.ts` — `getToolSchemaCompat()`:

Some providers have tool schema requirements that differ from the canonical OpenClaw tool schema. The compatibility layer transforms schemas to match provider expectations.

### 5.3 Tool Execution

Tool execution is handled through the agent runtime:

```
Agent decides to call tool
  → Runtime validates params against schema
  → Executes tool handler
  → Returns result to agent
  → Agent continues inference
```

---

## 6. Model Catalog

### 6.1 Catalog Types

**File:** `src/agents/model-catalog.types.ts` (100+ lines)

```typescript
interface ResolvedModel {
  model: string
  provider: string
  family: string
  displayName: string
  contextWindow: number
  supportsImages: boolean
  supportsTools: boolean
  supportsSystemPrompt: boolean
  inputCost?: number
  outputCost?: number
}

interface ProviderCatalog {
  models: ModelDefinition[]
  defaultModel?: string
  streamingDefaultModel?: string
}
```

### 6.2 Model Resolution

The system resolves model names to provider-specific model IDs:

```
"claude-3-5-sonnet" → { model: "claude-3-5-sonnet-20241022", provider: "anthropic" }
"gpt-4o" → { model: "gpt-4o-2024-08-06", provider: "openai" }
```

---

## 7. Auth Profiles

### 7.1 Per-Agent Auth

**File:** `src/agents/auth-profiles.ts` (100+ lines)

Auth profiles allow per-agent credential override:

```typescript
// ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
{
  "anthropic": {
    "apiKey": "sk-ant-..."
  },
  "openai": {
    "apiKey": "sk-..."
  }
}
```

If a provider supports per-agent auth, the agent uses its own credentials instead of the global provider credentials.

### 7.2 Auth Method Creation

From `src/plugins/types.ts` — `createAuthMethod()`:

Providers can create auth methods through the plugin API, supporting various auth flows (API key, OAuth, etc.).

---

## 8. Agent Command Handling

**File:** `src/agents/agent-command.ts` (100+ lines)

Handles agent-relative commands:
- `/new` — start new conversation
- `/reset` — reset conversation
- `/model <model>` — switch model
- `/provider <provider>` — switch provider

---

## 9. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/agents/pi-embedded-runner/` | ~500 | Embedded agent runner |
| `src/agents/acp-spawn.ts` | 200+ | ACP subagent spawning |
| `src/agents/harness/` | ~200 | Harness registry |
| `src/agents/tools/` | ~300 | Tool system |
| `src/agents/model-catalog.types.ts` | 100+ | Model resolution types |
| `src/agents/agent-runtime-config.ts` | 150+ | Runtime config management |
| `src/agents/api-key-rotation.ts` | 100+ | API key rotation |
| `src/agents/anthropic-transport-stream.ts` | 200+ | Anthropic API transport |
| `src/agents/anthropic-vertex-stream.ts` | 150+ | Vertex AI transport |
| `src/agents/auth-profiles.ts` | 100+ | Per-agent auth profiles |
| `src/agents/agent-command.ts` | 100+ | Agent command handling |