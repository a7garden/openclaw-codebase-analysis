# Bundled Plugins (extensions/) Reference

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

The `extensions/` directory contains **~130 bundled plugins** that are bundled into the OpenClaw installation. They follow the same boundary rules as third-party plugins — they MUST NOT import core `src/**` or another extension's `src/**`, and can only use `openclaw/plugin-sdk/*` and their own local barrels.

---

## 2. Complete Plugin Directory

### 2.1 Provider Plugins (Text Inference)

| Extension | Provider | Notes |
|-----------|----------|-------|
| `openai` | OpenAI | Full-featured: text, CLI, speech, transcription, voice, media, image, video |
| `anthropic` | Anthropic | Text, CLI, media understanding |
| `anthropic-vertex` | Anthropic via Vertex AI | Text |
| `google` | Google Gemini | Text, web search, image, music, video |
| `amazon-bedrock` | AWS Bedrock | Text, embedding |
| `amazon-bedrock-mantle` | Bedrock Mantle | Text |
| `alibaba` | Alibaba/Qwen | Video generation |
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
| `llmstudio` | LM Studio | Text (local) |
| `ollama` | Ollama | Text (local) |
| `fireworks` | Fireworks AI | Text |
| `sglang` | SGLang | Text |
| `vllm` | vLLM | Text |
| `cloudflare-ai-gateway` | Cloudflare AI Gateway | Text |
| `vercel-ai-gateway` | Vercel AI Gateway | Text |
| `litellm` | LiteLLM | Text (proxy) |
| `acpx` | ACPX | ACP protocol backend (embedded runtime) |

### 2.2 Channel Plugins (Messaging / Social)

| Extension | Platform | Notes |
|-----------|----------|-------|
| `telegram` | Telegram | Full messaging + streaming + security |
| `discord` | Discord | Bot + subagent hooks |
| `slack` | Slack | HTTP routes |
| `msteams` | Microsoft Teams | |
| `mattermost` | Mattermost | |
| `matrix` | Matrix | |
| `irc` | IRC | |
| `feishu` | Feishu/Lark | |
| `nextcloud-talk` | Nextcloud Talk | |
| `googlechat` | Google Chat | |
| `signal` | Signal | |
| `twitch` | Twitch | Chat |
| `line` | LINE | |
| `qqbot` | QQ | |
| `zalo` | Zalo | |
| `whatsapp` | WhatsApp | |
| `bluebubbles` | BlueBubbles | iMessage |
| `imessage` | iMessage | |
| `synology-chat` | Synology Chat | |
| `nostr` | Nostr | |

### 2.3 Tool / Capability Plugins

| Extension | Type | Description |
|-----------|------|-------------|
| `browser` | Tool | Default browser automation tool |
| `acpx` | Capability | ACP runtime with session/transport |
| `memory-core` | Memory | File-backed memory + search tools |
| `memory-lancedb` | Memory | LanceDB vector backend |
| `memory-wiki` | Memory | Wiki-style memory |
| `image-generation-core` | Capability | Image generation abstractions |
| `video-generation-core` | Capability | Video generation abstractions |
| `media-understanding-core` | Capability | Media understanding |
| `speech-core` | Capability | Speech/TTS abstractions |
| `llm-task` | Tool | Structured subtask execution |
| `device-pair` | Capability | Device pairing |
| `thread-ownership` | Capability | Thread ownership management |
| `webhooks` | Tool | Webhook ingress/targets |
| `diffs` | Tool | Diff utilities |
| `codex` | Capability | Codex integration |
| `github-copilot` | Capability | GitHub Copilot login/token |
| `brave` | Search | Brave search provider |
| `duckduckgo` | Search | DuckDuckGo search |
| `exa` | Search | Exa web search |
| `tavily` | Search | Tavily search |
| `searxng` | Search | SearXNG meta-search |
| `firecrawl` | Fetch | Firecrawl web fetch |
| `elevenlabs` | Speech | ElevenLabs TTS |
| `deepgram` | Transcription | DeepGram |
| `fal` | Image | Fal.ai image generation |
| `runway` | Video | Runway video |
| `stepfun` | Provider | StepFun |
| `comfy` | Image | ComfyUI |
| `arcee` | Provider | Arcee |
| `chutes` | Provider | Chutes |
| `open-prose` | Capability | OpenProse |
| `openshell` | Tool | OpenShell |
| `opencode` | Tool | OpenCode |
| `opencode-go` | Tool | OpenCode Go |
| `kilocode` | Tool | KiloCode |
| `lobster` | Tool | Lobster |
| `phone-control` | Capability | Phone control |
| `voice-call` | Capability | Voice call |
| `talk-voice` | Capability | Talk voice |
| `qa-channel` | Protocol | QA channel |
| `qa-lab` | Tool | QA lab runner |
| `qa-matrix` | Tool | QA matrix runner |
| `diagnostics-otel` | Capability | OpenTelemetry |
| `copilot-proxy` | Capability | Copilot proxy |
| `microsoft-foundry` | Provider | Microsoft Foundry |
| `byteplus` | Provider | BytePlus |
| `active-memory` | Memory | Active memory |

---

## 3. Standard Extension Layout

### 3.1 Provider Plugin Layout

```
extensions/<id>/
├── index.ts                    # Entry point (definePluginEntry or defineSingleProviderPluginEntry)
├── openclaw.plugin.json       # Manifest (required)
├── package.json
├── api.ts                    # Public API surface (optional)
├── runtime-api.ts            # Runtime seam (optional)
├── setup-api.ts              # Setup-only surface (optional)
├── tsconfig.json
└── src/                      # Private implementation (not imported by core)
    ├── provider.ts           # Main provider implementation
    ├── auth.ts               # Auth handling
    ├── catalog.ts            # Model catalog
    └── ... (more)
```

### 3.2 Channel Plugin Layout

```
extensions/<id>/
├── index.ts                    # Entry point (defineBundledChannelEntry)
├── openclaw.plugin.json       # Manifest
├── package.json
├── channel-plugin-api.js     # Main plugin export (lazy-loaded)
├── secret-contract-api.js    # Secrets export (lazy-loaded)
├── runtime-api.js             # Runtime setter (lazy-loaded)
├── account-inspect-api.js     # Account inspector (lazy-loaded)
├── tsconfig.json
└── src/                      # Private implementation
    ├── channel-plugin.ts      # Main ChannelPlugin implementation
    ├── messaging.ts          # Messaging adapter
    ├── auth.ts               # Auth handling
    ├── webhook.ts            # Webhook handling
    └── ... (more)
```

---

## 4. Entry Point Patterns

### 4.1 Provider Entry (Single)

```typescript
// extensions/mistral/index.ts
import { defineSingleProviderPluginEntry } from "openclaw/plugin-sdk/provider-entry"
import { buildMistralProvider } from "./src/provider"

export default defineSingleProviderPluginEntry({
  id: "mistral",
  name: "Mistral",
  description: "Mistral AI provider",
  provider: buildMistralProvider(),
})
```

### 4.2 Provider Entry (Multi)

```typescript
// extensions/openai/index.ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry"

export default definePluginEntry({
  id: "openai",
  name: "OpenAI Provider",
  register(api) {
    api.registerProvider(buildProviderWithPromptContribution(buildOpenAIProvider()))
    api.registerProvider(buildProviderWithPromptContribution(buildOpenAICodexProviderPlugin()))
    // ... more registrations
  },
})
```

### 4.3 Channel Entry

```typescript
// extensions/telegram/index.ts
import { defineBundledChannelEntry } from "openclaw/plugin-sdk/channel-entry-contract"

export default defineBundledChannelEntry({
  id: "telegram",
  name: "Telegram",
  importMetaUrl: import.meta.url,
  plugin: { specifier: "./channel-plugin-api.js", exportName: "telegramPlugin" },
  secrets: { specifier: "./secret-contract-api.js", exportName: "channelSecrets" },
  runtime: { specifier: "./runtime-api.js", exportName: "setTelegramRuntime" },
  accountInspect: { specifier: "./account-inspect-api.js", exportName: "inspectTelegramReadOnlyAccount" },
})
```

---

## 5. Manifest Example

```json
{
  "id": "openai",
  "version": "1.0.0",
  "configSchema": {
    "fields": [
      {
        "key": "apiKey",
        "type": "secret",
        "label": "API Key",
        "description": "Your OpenAI API key",
        "required": true
      },
      {
        "key": "organization",
        "type": "string",
        "label": "Organization ID",
        "description": "Optional organization ID",
        "required": false
      }
    ]
  },
  "providers": ["openai", "openai-codex"],
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1-", "o3-"]
  },
  "providerEndpoints": [
    { "kind": "baseUrl", "baseUrl": "https://api.openai.com/v1" }
  ],
  "activation": [
    { "kind": "provider", "provider": "openai" },
    { "kind": "provider", "provider": "openai-codex" }
  ],
  "secrets": {
    "fields": ["apiKey"]
  }
}
```

---

## 6. Extension SDK Subpaths (295 entries)

The Plugin SDK exposes a large surface under `openclaw/plugin-sdk/`:

### Core
- `openclaw/plugin-sdk` — main barrel
- `openclaw/plugin-sdk/core` — full SDK re-export
- `openclaw/plugin-sdk/plugin-entry` — `definePluginEntry()`
- `openclaw/plugin-sdk/provider-entry` — `defineSingleProviderPluginEntry()`
- `openclaw/plugin-sdk/channel-entry-contract` — `defineBundledChannelEntry()`
- `openclaw/plugin-sdk/channel-contract` — channel type definitions

### Provider
- `openclaw/plugin-sdk/provider-auth` — auth method creation
- `openclaw/plugin-sdk/provider-catalog-shared` — shared catalog building
- `openclaw/plugin-sdk/provider-env-vars` — environment variable handling
- `openclaw/plugin-sdk/provider-http` — HTTP client
- `openclaw/plugin-sdk/provider-model-types` — model types
- `openclaw/plugin-sdk/provider-stream-family` — stream wrapper family
- `openclaw/plugin-sdk/provider-tools` — tool schema compat
- `openclaw/plugin-sdk/provider-model-shared` — model cloning

### Runtime
- `openclaw/plugin-sdk/runtime` — main runtime interface
- `openclaw/plugin-sdk/agent-runtime` — agent harness
- `openclaw/plugin-sdk/channel-runtime` — channel runtime
- `openclaw/plugin-sdk/acp-runtime` — ACP runtime

### Capability-Specific
- `openclaw/plugin-sdk/image-generation`
- `openclaw/plugin-sdk/music-generation`
- `openclaw/plugin-sdk/video-generation`
- `openclaw/plugin-sdk/speech`
- `openclaw/plugin-sdk/realtime-transcription`
- `openclaw/plugin-sdk/realtime-voice`
- `openclaw/plugin-sdk/media-understanding`
- `openclaw/plugin-sdk/memory-host-*` (22+ subpaths)

---

## 7. OpenClawKit Swift Exports

From `apps/shared/OpenClawKit/Package.swift`:

```swift
targets: [
  .target(name: "OpenClawProtocol"),
  .target(name: "OpenClawKit"),
  .target(name: "OpenClawChatUI"),
]
```

**OpenClawProtocol:** Gateway wire protocol types (same as TypeScript gateway protocol)

**OpenClawKit:** GatewayChannel (WebSocket), TLS, discovery, command handlers

**OpenClawChatUI:** Reusable chat UI components (ChatComposer, ChatView, ChatMarkdownRenderer)