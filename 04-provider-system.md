# Provider System Architecture

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

Providers are AI model integrations — text inference (OpenAI GPT, Anthropic Claude, Google Gemini), speech synthesis, image generation, video generation, etc. The provider system is the mechanism by which OpenClaw connects to AI backends.

There are 30+ bundled provider plugins in `extensions/`, each implementing the provider plugin interface.

---

## 2. Provider Plugin Interface

### 2.1 Main Provider Interface

From `src/plugins/types.ts:1059-1592`, the `ProviderPlugin` interface has ~50 hooks:

```typescript
interface ProviderPlugin {
  id: string
  name: string

  // Model resolution
  resolveModel?(ctx: ProviderCatalogContext): Promise<ResolvedModel>
  resolveModelByName?(name: string): Promise<ResolvedModel | null>

  // Auth
  createAuthMethod?(ctx: CreateAuthMethodContext): Promise<ProviderAuthMethod>
  onAuthRefresh?(ctx: OnAuthRefreshContext): Promise<AuthRefreshResult>

  // Inference
  runInference?(ctx: RunInferenceContext): Promise<InferenceResult>
  runInferenceStream?(ctx: RunInferenceStreamContext): AsyncGenerator<InferenceChunk>

  // Replay policy
  getReplayPolicy?(ctx: ReplayPolicyContext): ReplayPolicy

  // Stream wrapping
  wrapStreamFamily?(family: StreamFamily): StreamFamily

  // Tool compatibility
  getToolSchemaCompat?(ctx: ToolSchemaCompatContext): ToolSchemaCompatResult

  // Token counting
  countTokens?(text: string): Promise<number>
  countTokensForMessages?(messages: Message[]): Promise<number>

  // Context window
  getContextWindow?(model: string): Promise<number>

  // Catalog
  getCatalog?(): Promise<ProviderCatalog>
  resolveModelFromRawRequest?(raw: unknown): ResolvedModel

  // API configuration
  getApiBaseUrl?(): string | undefined
  getApiKey?(): string | undefined
}
```

### 2.2 Provider Capability Types

Providers can register multiple capability types:

| Capability | Registration Method | Description |
|------------|---------------------|-------------|
| Text inference | `api.registerProvider()` | LLM chat/completion |
| CLI backend | `api.registerCliBackend()` | CLI inference (Codex-style) |
| Speech (TTS) | `api.registerSpeechProvider()` | Text-to-speech |
| Realtime transcription | `api.registerRealtimeTranscriptionProvider()` | Audio→text |
| Realtime voice | `api.registerRealtimeVoiceProvider()` | Voice conversation |
| Media understanding | `api.registerMediaUnderstandingProvider()` | Image/audio/video analysis |
| Image generation | `api.registerImageGenerationProvider()` | Image synthesis |
| Music generation | `api.registerMusicGenerationProvider()` | Music synthesis |
| Video generation | `api.registerVideoGenerationProvider()` | Video synthesis |
| Web fetch | `api.registerWebFetchProvider()` | HTTP fetching |
| Web search | `api.registerWebSearchProvider()` | Search engine |
| Memory embedding | `api.registerMemoryEmbeddingProvider()` | Vector embeddings |

---

## 3. Provider Plugin Entry

### 3.1 defineSingleProviderPluginEntry

For single-capability providers, use the sugar from `openclaw/plugin-sdk/provider-entry.ts:168`:

```typescript
export default defineSingleProviderPluginEntry({
  id: "anthropic",
  name: "Anthropic",
  description: "Anthropic model provider",
  provider: buildAnthropicProvider(),
})
```

This automatically wires up:
- Auth method creation and management
- Model catalog building
- Runtime integration

### 3.2 definePluginEntry

For multi-capability providers (like OpenAI), use `definePluginEntry()`:

```typescript
// From extensions/openai/index.ts
export default definePluginEntry({
  id: "openai",
  name: "OpenAI Provider",
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

---

## 4. Provider Auth Methods

### 4.1 Auth Method Types

From `src/plugins/types.ts:500-700`:

```typescript
type ProviderAuthMethod =
  | ApiKeyAuthMethod          // Direct API key
  | OAuthAuthMethod           // OAuth 2.0 flow
  | AzureAuthMethod           // Azure AD token
  | VertexAuthMethod          // GCP OAuth
  | BedrockAuthMethod         // AWS credentials
  | OpenAICompatAuthMethod    // OpenAI-compatible secret
  // ... others
```

### 4.2 Auth Profile Management

Per-agent auth profiles stored at `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`. Allows per-agent credential override when a provider supports it.

### 4.3 API Key Rotation

From `src/agents/api-key-rotation.ts`: handles automatic API key rotation for providers that support it, with fallback chains.

---

## 5. Model Catalog

### 5.1 Model Resolution

```typescript
// From src/agents/model-catalog.types.ts
interface ResolvedModel {
  model: string                    // Canonical model ID (e.g., "claude-3-5-sonnet-20241022")
  provider: string                 // Provider ID (e.g., "anthropic")
  family: string                   // Model family (e.g., "claude-3-5-sonnet")
  displayName: string              // Human-readable name
  contextWindow: number            // Context window size
  supportsImages: boolean
  supportsTools: boolean
  supportsSystemPrompt: boolean
  inputCost?: number               // Cost per token (input)
  outputCost?: number              // Cost per token (output)
  // ... more metadata
}
```

### 5.2 Catalog Building

From `src/plugins/types.ts` — `getCatalog()` returns a `ProviderCatalog`:

```typescript
interface ProviderCatalog {
  models: ModelDefinition[]
  defaultModel?: string
  streamingDefaultModel?: string
}
```

Model definitions include: context window, tool support, image support, pricing, and other metadata.

---

## 6. Provider Endpoint Matching

From manifest types — `providerEndpoints` for hostname/baseUrl matching:

```typescript
{
  providerEndpoints: [
    { kind: "baseUrl", baseUrl: "https://api.openai.com/v1" },
    { kind: "hostname", hostname: "api.anthropic.com" },
  ]
}
```

Used for transparent model prefix resolution — if a request comes in with model `openai/gpt-4o`, the system can auto-activate the OpenAI plugin.

---

## 7. Replay Policy and Stream Wrapping

### 7.1 Replay Policy

```typescript
interface ReplayPolicy {
  canReplay: boolean
  maxReplayAge?: number            // How old a conversation can be to replay
  replayCooldownMs?: number       // Cooldown between replays
}
```

Some models/providers don't support conversation replay (e.g., some streaming endpoints). The replay policy tells the system how to handle this.

### 7.2 Stream Wrapping Family

```typescript
wrapStreamFamily?(family: StreamFamily): StreamFamily
```

Allows providers to wrap or transform streaming behavior — useful when a provider's native streaming format differs from the standard format.

---

## 8. Transport Streams

From `src/agents/anthropic-transport-stream.ts` and `src/agents/anthropic-vertex-stream.ts`:

- `AnthropicTransportStream` — connects to Anthropic's API
- `AnthropicVertexStream` — connects via Google Vertex AI

These handle:
- Request signing
- Streaming response parsing
- Error handling and retry
- Rate limiting

---

## 9. Bundled Provider Plugins

| Extension | Provider | Capabilities |
|-----------|----------|-------------|
| `openai` | OpenAI | Text, CLI, speech, transcription, voice, media, image, video, embedding |
| `anthropic` | Anthropic | Text, CLI, media |
| `anthropic-vertex` | Anthropic via Vertex | Text |
| `google` | Google Gemini | Text, web search, image, music, video |
| `amazon-bedrock` | AWS Bedrock | Text, embedding |
| `amazon-bedrock-mantle` | Bedrock Mantle | Text |
| `alibaba` | Alibaba/Qwen | Video |
| `mistral` | Mistral AI | Text |
| `groq` | Groq | Text |
| `openrouter` | OpenRouter | Text |
| `together` | Together AI | Text |
| `perplexity` | Perplexity | Text, web search |
| `cohere` | Cohere | Text, embedding |
| `deepseek` | DeepSeek | Text |
| `moonshot` | Moonshot AI | Text |
| `minimax` | MiniMax | Text |
| `kimi-coding` | Kimi Coding | Text |
| `xai` | xAI | Text |
| `volcengine` | Volcano Engine | Text |
| `qianfan` | Qianfan/Baidu | Text |
| `voyage` | Voyage AI | Embedding |
| `ollama` | Ollama | Text (local) |
| `fireworks` | Fireworks AI | Text |
| `sglang` | SGLang | Text |
| `vllm` | vLLM | Text |
| `cloudflare-ai-gateway` | CF AI Gateway | Text |
| `vercel-ai-gateway` | Vercel AI Gateway | Text |
| `litellm` | LiteLLM | Text |
| `acpx` | ACPX | Embedded runtime |

---

## 10. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/plugins/types.ts` | ~1230 | Full provider interface, auth methods, catalog types |
| `src/plugin-sdk/provider-entry.ts` | 168 | `defineSingleProviderPluginEntry()` |
| `src/agents/model-catalog.types.ts` | ~100 | Model resolution types |
| `src/agents/api-key-rotation.ts` | ~100 | API key rotation logic |
| `src/agents/anthropic-transport-stream.ts` | ~200 | Anthropic API transport |
| `src/agents/anthropic-vertex-stream.ts` | ~150 | Vertex AI transport |
| `src/agents/agent-runtime-config.ts` | ~150 | Runtime config management |
| `extensions/openai/index.ts` | ~50 | OpenAI plugin entry |
| `extensions/anthropic/index.ts` | ~50 | Anthropic plugin entry |