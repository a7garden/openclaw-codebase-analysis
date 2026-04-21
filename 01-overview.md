# OpenClaw Codebase Architecture Analysis

> **Project:** OpenClaw — AI Agent Platform with Multi-Channel, Multi-Provider Support
> **Repository:** https://github.com/openclaw/openclaw
> **Generated:** 2026-04-21
> **Scope:** Full monorepo including `src/`, `extensions/`, `ui/`, `packages/`, `apps/`, `docs/`
> **Total Source Size:** ~1.35M+ lines of TypeScript + Swift + Kotlin + other languages

---

## Table of Contents

1. [System Overview](#1-system-overview) — What OpenClaw is and how it fits together
2. [Architecture Principles](#2-architecture-principles) — Core rules that govern the design
3. [Directory Structure](#3-directory-structure) — Top-level layout and mapping
4. [Entry Points & Boot Chain](#4-entry-points--boot-chain) — How the application starts
5. [Plugin System](#5-plugin-system) — The extension architecture
6. [Channel System](#6-channel-system) — Messaging channel plugins
7. [Provider System](#7-provider-system) — Model inference providers
8. [Gateway Protocol](#8-gateway-protocol) — WebSocket-based client/server communication
9. [Agent System](#9-agent-system) — Agent runtime and subagent management
10. [ACP (Agent Communication Protocol)](#10-acp-agent-communication-protocol) — Inter-agent messaging
11. [Configuration System](#11-configuration-system) — Config schema, loading, normalization
12. [Memory System](#12-memory-system) — Memory plugins and embedding infrastructure
13. [Frontend Architecture (UI)](#13-frontend-architecture-ui) — Lit web components, routing, gateway client
14. [Terminal UI (TUI)](#14-terminal-ui-tui) — CLI terminal interface
15. [Mobile Apps](#15-mobile-apps) — iOS, Android, macOS, Swabble
16. [Bundled Plugins (extensions/)](#16-bundled-plugins-extensions) — 100+ built-in plugins
17. [ACP Deep Dive](#17-acp-deep-dive) — Translator, session store, control-plane manager, runtime interface
18. [Infrastructure, Hooks & Tasks](#18-infrastructure-hooks--tasks) — Retry, heartbeat, events, hook system, SQLite tasks
19. [Vendor Code & Python Skills](#19-vendor-code--python-skills) — Google A2UI, Python skills ecosystem, baileys patches
20. [Plugin Loader Deep Dive](#20-plugin-loader-deep-dive) — Full 12-stage loading sequence, lazy runtime proxy
21. [Channel System Deep Dive](#21-channel-system-deep-dive) — ~250 files, 12 adapter types, binding system
22. [Embedded Runner Deep Dive](#22-embedded-runner-deep-dive) — pi-embedded-runner, 12-stage stream wrapper, auth rotation
23. [Gateway Server Methods](#23-gateway-server-methods) — ~110 methods, 28 functional categories
24. [Telegram Extension](#24-telegram-extension) — grammY polling/webhook, streaming lanes, dedupe, media handling
25. [Packages](#25-packages) — plugin-sdk, memory-host-sdk, plugin-package-contract
26. [CLI & Daemon](#26-cli--daemon) — Command structure and daemon lifecycle
27. [Testing Strategy](#27-testing-strategy) — Test layout and patterns
28. [Key Files Reference](#28-key-files-reference) — Essential files by topic

---

## 1. System Overview

OpenClaw is a **platform for running AI agents** that can connect to multiple messaging channels (Telegram, Discord, Slack, etc.) and multiple AI model providers (OpenAI, Anthropic, Google, etc.), exposing those agents through a WebSocket-based gateway to various frontends (web UI, mobile apps, terminal UI).

```
                    ┌──────────────────────────────────────────────────────┐
                    │                    OpenClaw Core                     │
                    │  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐ │
                    │  │   Gateway   │  │   Plugin     │  │    Agent    │ │
                    │  │  Protocol   │  │   Loader     │  │   Runtime   │ │
                    │  └─────────────┘  └──────────────┘  └─────────────┘ │
                    └──────────────────────┬───────────────────────────────┘
                                             │
           ┌─────────────────────────────────┼─────────────────────────────────┐
           │                                 │                                 │
     ┌─────┴─────┐                    ┌─────┴─────┐                    ┌─────┴─────┐
     │  Mobile   │                    │    Web    │                    │   TUI     │
     │   Apps    │                    │     UI    │                    │ (Terminal)│
     │ iOS/Andr  │                    │   (Lit)   │                    │           │
     └───────────┘                    └───────────┘                    └───────────┘
                                             │
              ┌──────────────────────────────┴──────────────────────────────┐
              │                        Channels                              │
              │  Telegram  Discord  Slack  Matrix  IRC  Feishu  WhatsApp ...  │
              └──────────────────────────────────────────────────────────────┘
                                             │
              ┌──────────────────────────────┴──────────────────────────────┐
              │                      Providers                               │
              │  OpenAI  Anthropic  Google  Amazon  Mistral  Groq  Ollama ... │
              └──────────────────────────────────────────────────────────────┘
```

**Core responsibilities:**
- Load and manage 100+ bundled plugins (providers, channels, tools, memory backends)
- Expose a WebSocket gateway protocol for clients (UI, mobile apps)
- Run AI agents that connect providers to channels
- Manage agent sessions, configuration, secrets, and tool registration

**NOT core responsibilities (stays in extensions):**
- Channel-specific auth flows (Telegram login, Discord OAuth, etc.)
- Provider-specific API key management
- Provider model catalog/fallback strategies
- Onboarding flows per channel/provider

---

## 2. Architecture Principles

### 2.1 Extension-Agnostic Core

Core must stay extension-agnostic. **No core special cases** for bundled plugin/provider/channel IDs when manifest/registry/capability contracts can express it.

> "Extensions cross into core only via `openclaw/plugin-sdk/*`, manifest metadata, injected runtime helpers, and documented local barrels (`api.ts`, `runtime-api.ts`)."

### 2.2 Strict Boundary Enforcement

```
Extension production code MUST NOT:
  - Import core `src/**`, `src/plugin-sdk-internal/**`
  - Import another extension's `src/**`
  - Use relative paths outside its package root

Core code/tests MUST NOT:
  - Deep-import plugin internals (`extensions/*/src/**`, `onboard.js`)
  - Use plugin `api.ts` / public SDK facade / generic contract
```

### 2.3 Manifest-First Control Plane

Plugin discovery, config validation, activation planning, and setup work from **metadata before plugin runtime executes**. This keeps startup fast and allows early validation without loading plugin code.

### 2.4 Lazy Loading Everywhere

- Channel plugins use separate lazy module references for plugin/secrets/runtime/accountInspect
- Provider plugins defer heavy surfaces behind async imports
- JITI caching for `.ts` file loading in dev, `.js` fast path for `dist/`
- `*.runtime.ts` lazy boundaries for expensive modules

### 2.5 Separation of Concerns

| Layer | Owned by |
|-------|----------|
| Auth/catalog/onboarding | Plugin-owned |
| Transport/replay/tool compat families | Shared helpers |
| Registry/runtime code | Core |
| Channel internals | Not exposed to plugin authors |

### 2.6 Versioned Protocol Contracts

Gateway protocol uses `PROTOCOL_VERSION`. Additive changes preferred; incompatible changes require versioning, docs, and client follow-through.

### 2.7 Deterministic Ordering for Prompt Cache

Maps/sets/registries/plugin lists/files/network results must be deterministically ordered **before** model/tool payloads. Old transcript bytes must be preserved when possible.

---

## 3. Directory Structure

```
openclaw/
├── src/                    # Core TypeScript (~1.3M lines)
│   ├── index.ts            # Package entry / library exports
│   ├── entry.ts            # Process entry with respawn handling
│   ├── cli/                # CLI bootstrapping and command registration
│   │   └── run-main.ts     # Main CLI orchestration (292 lines)
│   ├── acp/                # Agent Communication Protocol (session, translator, control-plane)
│   ├── agents/             # Agent runtime, embedded runner, spawn, auth profiles
│   ├── channels/           # Channel implementations, adapters, security
│   ├── config/             # Configuration types, schema, loading, normalization
│   ├── gateway/            # Gateway server (boot, client, auth, call, sessions)
│   │   ├── protocol/       # Wire protocol (schema, validators, types)
│   │   └── server-methods/ # Server RPC method handlers
│   ├── hooks/              # Internal hook types and runners
│   ├── infra/              # Environment, networking, errors, secrets, filesystem
│   ├── plugins/            # Plugin loader, registry, discovery, bundled management
│   ├── plugin-sdk/         # Public SDK (plugin-entry, provider-entry, channel-entry-contract)
│   ├── runtime/            # Main entry point orchestration
│   ├── tasks/              # Task registry, detached task runtime
│   ├── tui/                # Terminal UI (chat log, markdown, overlays, gateway-chat)
│   └── wizard/             # Interactive setup wizard (config, plugin config, secrets)
│
├── extensions/             # ~130 bundled plugins
│   ├── openai/             # OpenAI provider + Codex + image/speech/realtime/media
│   ├── anthropic/          # Anthropic provider + CLI backend
│   ├── telegram/           # Telegram channel
│   ├── discord/            # Discord channel
│   ├── memory-core/        # File-backed memory + search tools
│   ├── memory-lancedb/     # LanceDB vector memory
│   ├── browser/            # Default browser automation tool
│   ├── ... (120+ more)
│   └── CLAUDE.md           # Extension boundary rules
│
├── ui/                     # Web UI (Lit web components)
│   ├── index.html          # Entry HTML
│   ├── package.json
│   └── src/
│       ├── main.ts         # Bootstrap entry
│       ├── styles.css
│       ├── ui/
│       │   ├── app.ts       # Main <openclaw-app> component (824 lines)
│       │   ├── app-gateway.ts    # Gateway WebSocket client (610 lines)
│       │   ├── app-chat.ts       # Chat send/queue logic (567 lines)
│       │   ├── app-render.ts    # Main render (~2400 lines)
│       │   ├── gateway.ts       # GatewayBrowserClient WebSocket class (645 lines)
│       │   ├── navigation.ts   # Tab routing
│       │   ├── controllers/    # Data loading/management
│       │   ├── views/           # Lazy-loaded view components
│       │   └── chat/            # Chat-specific rendering
│       └── i18n/            # Internationalization (11 locales)
│
├── packages/               # Private npm packages
│   ├── plugin-sdk/         # @openclaw/plugin-sdk — public plugin SDK
│   ├── memory-host-sdk/    # @openclaw/memory-host-sdk — memory backend SDK
│   └── plugin-package-contract/  # External plugin package.json validation
│
├── apps/                   # Mobile/desktop applications
│   ├── ios/                # iOS app (Swift 6, SwiftUI)
│   ├── android/            # Android app (Kotlin, Jetpack Compose)
│   ├── macos/              # macOS app (Swift 6, SwiftUI)
│   └── shared/OpenClawKit/ # Shared Swift protocol library
│
├── Swabble/                # macOS speech wake-word daemon (Swift 6.2)
│
├── packages/               # (see above)
│
├── docs/                  # Documentation
│   ├── guides/             # User guides
│   ├── reference/          # Technical reference
│   └── AGENTS.md           # Docs-specific rules
│
├── test/                  # Test helpers and fixtures
│   └── helpers/           # Shared test utilities
│
├── scripts/               # Build/dev scripts
├── vendor/                # Vendored third-party code
└── patches/               # pnpm patch files
```

---

## 4. Entry Points & Boot Chain

### 4.1 Process Entry Chain

```
Node.js process starts
  → src/index.ts:85        runLegacyCliEntry()   [legacy direct entry]
  → src/entry.ts:182       runMainOrRootHelp()
    → import("./cli/run-main.js")
    → src/cli/run-main.ts:150  runCli()
      → command registration (plugin commands, etc.)
      → program.parse() via cacaca CLI builder
```

### 4.2 Main CLI Commands

From `src/cli/run-main.ts:150-292`, the CLI supports:
- `openclaw dev` — development mode
- `openclaw gateway` — start gateway server
- `openclaw tui` — terminal UI
- `openclaw wizard` — setup wizard
- `openclaw plugins` — plugin management
- `openclaw config` — config inspection/mutation
- `openclaw doctor` — diagnostics
- And plugin-registered commands via `registerCli()`

### 4.3 Gateway Boot Chain

```
gateway/boot.ts
  → loads plugin registry
  → sets up WebSocket server
  → registers server methods
  → auth handler setup
  → client connection handling
```

### 4.4 Plugin Loading Sequence

```
loadPlugins(PluginLoadOptions)
  → discoverOpenClawPlugins()    [finds plugins via manifest scan]
  → loadPluginManifestRegistry() [parses manifest files]
  → createPluginRegistry()      [builds registry with activation state]
  → buildPluginApi()            [constructs OpenClawPluginApi per plugin]
  → plugin.register() / plugin.activate()
```

The activation planner (`src/plugins/activation-planner.ts`) uses manifest metadata to decide what to activate — **no runtime execution required** for activation decisions.

---

## 5. Plugin System

*Detailed in: [02-plugin-system.md](./02-plugin-system.md)*

**Key concept:** OpenClaw has a **manifest-first** plugin system where `openclaw.plugin.json` metadata is read before any plugin code executes. This enables config validation, activation planning, and setup without loading plugin runtime.

**Plugin API (`OpenClawPluginApi`):**
- `registerTool()`, `registerHook()`, `registerHttpRoute()`, `registerChannel()`
- `registerGatewayMethod()`, `registerCli()`, `registerService()`
- `registerProvider()`, `registerSpeechProvider()`, `registerRealtimeTranscriptionProvider()`
- `registerRealtimeVoiceProvider()`, `registerMemoryCapability()`, `registerAgentHarness()`
- ...40+ methods total

**Plugin shapes:**
- **plain-capability** — single capability type (e.g., `mistral` provider-only)
- **hybrid-capability** — multiple capability types (e.g., `openai` has text inference + speech + media + image generation)
- **hook-only** — registers only hooks, no capabilities
- **non-capability** — registers tools/commands/services but no capabilities

**SDK boundary:** Extensions MUST use `openclaw/plugin-sdk/*` only. They cannot import `src/**` or another extension's `src/**`.

---

## 6. Channel System

*Detailed in: [03-channel-system.md](./03-channel-system.md)*

Channels are messaging platform integrations (Telegram, Discord, Slack, etc.). Each channel plugin implements the `ChannelPlugin` interface with optional capability adapters:

- **messaging** — send/receive messages
- **streaming** — stream responses
- **outbound** — push notifications
- **security** — authentication/authorization
- **approvalCapability** — approval request handling
- **setup** — channel configuration
- **pairing** — device pairing
- **threading** — thread/topic handling
- **groups** — group chat support
- **status** — health/status reporting
- **secrets** — credential storage
- **gatewayMethods** — gateway RPC methods

**Channel entry contract:** `defineBundledChannelEntry()` in `openclaw/plugin-sdk/channel-entry-contract.ts:522` provides lazy-loading wiring with four separate module references (plugin, secrets, runtime, accountInspect).

---

## 7. Provider System

*Detailed in: [04-provider-system.md](./04-provider-system.md)*

Providers are AI model integrations (OpenAI, Anthropic, Google, etc.). The provider system handles:
- Model resolution and selection
- Auth method management (API keys, OAuth, etc.)
- Replay policy and stream wrapping
- Tool schema compatibility
- Catalog building and model metadata

**Provider plugin interface:** Uses `definePluginEntry()` or `defineSingleProviderPluginEntry()`. Can register:
- Text inference (`registerProvider`)
- CLI backend (`registerCliBackend`)
- Speech (`registerSpeechProvider`)
- Realtime transcription (`registerRealtimeTranscriptionProvider`)
- Realtime voice (`registerRealtimeVoiceProvider`)
- Media understanding (`registerMediaUnderstandingProvider`)
- Image generation (`registerImageGenerationProvider`)
- Music generation (`registerMusicGenerationProvider`)
- Video generation (`registerVideoGenerationProvider`)
- Web fetch (`registerWebFetchProvider`)
- Web search (`registerWebSearchProvider`)
- Memory embedding (`registerMemoryEmbeddingProvider`)

---

## 8. Gateway Protocol

*Detailed in: [05-gateway-protocol.md](./05-gateway-protocol.md)*

The gateway is a WebSocket-based client/server protocol (version 3) that exposes OpenClaw's capabilities to frontends.

**Protocol:** JSON frames over WebSocket:
```
{ type: "req", id, method, params }   // request
{ type: "res", id, ok, payload/error } // response
{ type: "event", event, payload, seq } // server->client event
```

**Auth modes:**
- Device identity (Ed25519 key pair stored in localStorage)
- Gateway token (explicit)
- Password (for `gateway.auth.password` mode)

**Key server methods:**
- `gateway.connect` — establish connection with auth
- `chat.send` — send message
- `sessions.list` / `sessions.create` / `sessions.get`
- `config.patch` — update config
- `cron.jobs.list` / `cron.jobs.create`
- `agents.create` — spawn agent

**Client libraries:**
- `GatewayBrowserClient` in `ui/src/ui/gateway.ts` (645 lines)
- `GatewayChannel` in `apps/shared/OpenClawKit/` (Swift)
- `GatewaySession` in `apps/android/` (Kotlin)

---

## 9. Agent System

*Detailed in: [06-agent-system.md](./06-agent-system.md)*

The agent runtime (`src/agents/`) handles:
- **Embedded runner** (`pi-embedded-runner/`) — runs agents in-process
- **Spawn management** (`acp-spawn.ts`) — ACP subagent spawning
- **Auth profiles** (`auth-profiles.ts`) — per-agent auth configuration
- **Transport streams** (`anthropic-transport-stream.ts`, `anthropic-vertex-stream.ts`) — provider connectivity
- **Tool registration** (`tools/`) — tool schema and execution
- **Model catalog** (`model-catalog.types.ts`) — model metadata

**Agent runtime config** (`agent-runtime-config.ts`): manages model selection, provider config, API key rotation (`api-key-rotation.ts`).

---

## 10. ACP (Agent Communication Protocol)

*Detailed in: [07-acp.md](./07-acp.md)*

ACP is the inter-agent communication protocol (`src/acp/`):
- **Session management** (`session.ts`) — `AcpSessionStore` with 5000 max sessions, 24hr idle TTL
- **Session actor queue** — per-session message queue
- **Translator** — message translation between agents
- **Control-plane** — ACP control plane logic
- **Runtime adapters** — adapter between ACP and actual agent runtimes

**Session store interface:**
```typescript
export type AcpSessionStore = {
  createSession, hasSession, getSession, getSessionByRunId
  setActiveRun, clearActiveRun, cancelActiveRun
}
```

---

## 11. Configuration System

*Detailed in: [08-config-system.md](./08-config-system.md)*

The config system (`src/config/`) handles:
- **Types** — config type definitions
- **Schema** — Zod-like validation schema with `safeParse`, `validate`, `uiHints`, `jsonSchema`
- **Loading** — config file loading from `~/.openclaw/`
- **Normalization** — config value normalization and defaults

**Manifest config schema** (`src/plugins/manifest-types.ts`): `OpenClawPluginConfigSchema` with per-field UI hints, advanced markers, secret input paths.

---

## 12. Memory System

*Detailed in: [09-memory-system.md](./09-memory-system.md)*

Memory plugins (`extensions/memory-*`) provide persistent memory/embedding infrastructure:
- `memory-core` — file-backed memory with search tools (`memory_search`, `memory_get`)
- `memory-lancedb` — LanceDB vector storage backend
- `memory-wiki` — wiki-style memory with search

**Memory host SDK** (`packages/memory-host-sdk/`): exposes engine interfaces for:
- `engine-foundation.ts` — base memory engine
- `engine-embeddings.ts` — embedding computation
- `engine-storage.ts` — persistent storage
- `engine-qmd.ts` — query metadata
- `host/batch-runner.ts` — batch processing
- `host/embeddings-remote-client.ts` — remote embedding services

---

## 13. Frontend Architecture (UI)

*Detailed in: [10-ui-frontend.md](./10-ui-frontend.md)*

**Framework:** Lit 3.3.2 (Web Components with tagged template literals), NOT React.

**Main component:** `<openclaw-app>` in `ui/src/ui/app.ts:132` (824 lines)

**State management:** Lit's `@state()` decorators, centralized in `OpenClawApp`

**Routing:** Hash/path-based tab routing via `navigation.ts`, synced with `popstate`. Tabs grouped: chat, control, agent, settings.

**Gateway connection:** `GatewayBrowserClient` in `ui/src/ui/gateway.ts:289` — WebSocket client with device identity auth, reconnect with backoff.

**Views:** Lazy-loaded via `createLazy()` in `app-render.ts:159-174`.

**i18n:** 11 locales in `ui/src/i18n/locales/`, generated from `en.ts`.

---

## 14. Terminal UI (TUI)

*Detailed in: [11-tui.md](./11-tui.md)*

The TUI lives in `src/tui/` (NOT in `ui/`), using `@mariozechner/pi-tui`.

**Main entry:** `src/tui/tui.ts:1`

**Key components:**
- `gateway-chat.ts` — Gateway client for TUI (WebSocket, similar to UI)
- `tui-command-handlers.ts` — Slash command handlers (`/new`, `/reset`, `/model`, etc.)
- `components/chat-log.ts` — Chat log rendering
- `components/markdown-message.ts` — Markdown rendering in terminal
- `components/custom-editor.ts` — Custom input editor
- `theme/theme.ts` — Terminal ANSI color theme

**CLI entry:** `src/cli/tui-cli.ts`

---

## 15. Mobile Apps

*Detailed in: [12-mobile-apps.md](./12-mobile-apps.md)*

| Platform | Location | Framework | Build |
|----------|----------|-----------|-------|
| iOS | `apps/ios/` | Swift 6 / SwiftUI | Xcode (project.yml) |
| watchOS | `apps/ios/WatchApp`, `WatchExtension` | Swift 6 | Xcode |
| Android | `apps/android/` | Kotlin / Jetpack Compose | Gradle |
| macOS | `apps/macos/` | Swift 6 / SwiftUI | SPM |
| Swabble | `Swabble/` | Swift 6.2 | SPM |

**Shared protocol library:** `apps/shared/OpenClawKit/` — Swift library with:
- `OpenClawProtocol/GatewayModels.swift` — cross-platform types
- `OpenClawKit/GatewayChannel.swift` — WebSocket gateway protocol
- `OpenClawChatUI/` — shared chat UI components

**Gateway protocol version:** `GATEWAY_PROTOCOL_VERSION = 3`

---

## 16. Bundled Plugins (extensions/)

*Detailed in: [13-bundled-plugins.md](./13-bundled-plugins.md)*

~130 bundled plugins across categories:

**Provider plugins (30+):** openai, anthropic, anthropic-vertex, google, amazon-bedrock, mistral, groq, openrouter, together, perplexity, cohere, deepseek, moonshot, minimax, kimi-coding, xai, volcengine, qianfan, voyage, ollama, fireworks, sglang, vllm, cloudflare-ai-gateway, vercel-ai-gateway, litellm, acpx, and more.

**Channel plugins (20+):** telegram, discord, slack, msteams, mattermost, matrix, irc, feishu, nextcloud-talk, googlechat, signal, twitch, line, qqbot, zalo, whatsapp, bluebubbles, imessage, synology-chat, nostr, and more.

**Tool/Capability plugins:** browser, acpx, memory-core, memory-lancedb, memory-wiki, image-generation-core, video-generation-core, speech-core, llm-task, device-pair, webhooks, diffs, codex, github-copilot, brave, duckduckgo, exa, tavily, searxng, firecrawl, elevenlabs, deepgram, fal, runway, and more.

---

## 17. ACP Deep Dive

*Detailed in: [17-acp-deep-dive.md](./17-acp-deep-dive.md)*

ACP bridges OpenClaw's internal agent runtime with **external harness clients** — IDE plugins (VS Code/IDX), CLI tools (Codex CLI), and other external agents. It translates the ACP protocol (from `@agentclientprotocol/sdk`) to OpenClaw's Gateway WebSocket protocol and back.

**Key files:**
- `src/acp/translator.ts` — 1,419-line central bridge (AcpGatewayAgent)
- `src/acp/control-plane/manager.core.ts` — 2,221-line orchestrator (AcpSessionManager)
- `src/acp/session.ts` — 191-line session store (in-memory, 5000 max, 24hr TTL)

**Core concepts:** Session actor queue (per-session serialization), runtime cache with idle eviction, turn timeout with 1s grace period, pending prompt tracking, disconnect handling with 5s arm timer.

---

## 18. Infrastructure, Hooks & Tasks

*Detailed in: [18-infra-hooks-tasks.md](./18-infra-hooks-tasks.md)*

**Infrastructure (`src/infra/`):**
- Error handling with nested `.cause` chain extraction
- Retry logic with cryptographic jitter (`generateSecureFraction()`)
- Deduplication cache (TTL-based, global singleton)
- Heartbeat runner (1,500+ lines) — periodic agent heartbeat scheduler
- Agent events (global event bus) + system events (ephemeral in-memory queue)
- Safe file operations: `O_NOFOLLOW`, `0o600` permissions, atomic writes

**Hooks (`src/hooks/`):** 30 hook event types, global singleton runner, bundled hooks (session-memory saves context on `/new` or `/reset`), hook installation from npm/archive/path.

**Tasks (`src/tasks/`):** SQLite persistence at `~/.openclaw/tasks.db`, 5000-session in-memory store, 24hr TTL, task flows with CAS revision control, delivery/notification policies.

---

## 19. Vendor Code & Python Skills

*Detailed in: [19-vendor-skills-python.md](./19-vendor-skills-python.md)*

**Google A2UI (`vendor/a2ui/`):** Agent-to-User Interface — open-source (Apache 2.0) protocol for agents to generate rich declarative UIs via JSON specification. Versions v0.8 and v0.9. OpenClaw vendors Lit web components renderer and Angular renderer for canvas/control UI.

**Python Skills (`skills/`):** 50+ Python-based skill packages (productivity, communication, media, dev, AI, IoT, utilities). Each skill contains `SKILL.md`, Python scripts, and pytest tests. Linted via ruff and tested via pytest.

**Baileys Patch:** `patches/@whiskeysockets__baileys@7.0.0-rc.9.patch` fixes file write race condition and dispatcher type guard for undici compatibility.

---

## 20. Plugin Loader Deep Dive

*Detailed in: [20-plugin-loader-deep-dive.md](./20-plugin-loader-deep-dive.md)*

Full 12-stage loading sequence from `src/plugins/loader.ts` (line 1397+):
1. Resolve plugin load cache context
2. Check cache for existing registry
3. Throw if load already in-flight
4. Clear prior plugin state
5. Create JITI loader
6. Set up LAZY PROXY for runtime
7. Create empty registry + API builder
8. Discover all plugin candidates
9. Parse manifest registry
10. Sort by precedence (config > global > bundled > workspace)
11. Per-candidate processing (validate, load, register/activate)
12. Activate plugin registry

**Key features:** Lazy runtime proxy (accessing any lazy key triggers module load), bundle format detection (codex/claude/cursor), rollback on error clearing commands/handlers/hooks/context engines.

---

## 21. Channel System Deep Dive

*Detailed in: [21-channel-deep-dive.md](./21-channel-deep-dive.md)*

**Scale:** ~250 files in `src/channels/`, ~29,687 lines of TypeScript, ~180 files in `src/channels/plugins/`, 12 adapter types composing ChannelPlugin interface.

**12 Adapter Types:** ChannelConfigAdapter, ChannelSecurityAdapter, ChannelOutboundAdapter, ChannelGatewayAdapter, ChannelDirectoryAdapter, ChannelMessagingAdapter, ChannelThreadingAdapter, ChannelMessageActionAdapter, + 5 others (Groups, Mentions, Heartbeat, Status, Secrets, Allowlist, Doctor, Bindings, Resolver, Approval, Pairing).

**Binding System:** Channel-to-channel bridging via `ConfiguredBindingRecord`. Resolution flow: User on source channel → lookup binding → resolve agent + channel → create ACP session key → deliver to bound channel.

**3-Layer Session Conversation Fallback:** (1) Plugin resolution → (2) Bundled plugin fallback via `loadBundledChannelSessionConversationApi()` → (3) Generic fallback via `buildGenericConversationResolution()`.

---

## 22. Embedded Runner Deep Dive

*Detailed in: [22-embedded-runner-deep-dive.md](./22-embedded-runner-deep-dive.md)*

**pi-embedded-runner** (`src/agents/pi-embedded-runner/`) is the core agent execution engine.

**Two-layer execution model:**
- **Outer (`runEmbeddedPiAgent`)**: Retry loop orchestrator managing auth rotation, compaction decisions, and retry policy
- **Inner (`runEmbeddedAttempt`)**: Single-shot per-attempt execution with its own session lifecycle

**12-stage stream wrapper chain** (applied to `activeSession.agent.streamFn`):
1. `cacheTrace.wrapStreamFn()` — Observability
2. `dropThinkingBlocks()` — Strip thinking for Anthropic
3. `sanitizeReplayToolCallIds()` — Normalize tool call IDs
4. `downgradeOpenAIReasoningBlocks()` — OpenAI Responses API compat
5. `handleSessionsYieldAbort()` — Yield interruption handling
6. `sanitizeMalformedToolCalls()` — Fix malformed tool names
7. `trimToolCallNames()` + `unknownToolGuard` — Whitespace trimming + hallucination guard
8. `repairMalformedAnthropicToolArgs()` — Fix JSON in tool arguments
9. `decodeXaiHtmlEntities()` — xAI-specific decoding
10. `anthropicPayloadLogger.wrap()` — Debug logging
11. `handleSensitiveStopReason()` — Provider-specific recovery
12. `streamWithIdleTimeout()` — LLM responsiveness detection

**Key files:**
- `run.ts` (~2641 lines) — Main orchestrator with retry loop
- `run/attempt.ts` (~2642 lines) — Single-attempt execution engine
- `run/backend.ts` (9 lines) — Delegates to harness
- `run/auth-controller.ts` (527 lines) — Auth profile lifecycle
- `compact.ts` (~1400 lines) — Session compaction
- `model.ts` (~855 lines) — Model resolution

---

## 23. Gateway Server Methods

*Detailed in: [23-gateway-server-methods.md](./23-gateway-server-methods.md)*

**~110 gateway server methods** organized into 28 functional categories.

**28 functional areas:** Connection, Health, Sessions (~18 methods), Chat (4), Agent (7), Config (7), Channels (3), Commands, Cron (7), Devices (6), Nodes (~12), Talk, Skills (6), Tools (2), Models (2), Usage (4), Exec Approvals (7), Plugin Approvals (4), System (5), Update, TTS (5), Doctor (7), Push, Secrets, Web, Wizard, Voice Wake

**Helper files:** `agent-job.ts`, `agent-wait-dedupe.ts`, `attachment-normalize.ts`, `base-hash.ts`, `nodes.helpers.ts`, and 9 others

**8 key architectural notes:**
- `parentId` chain rule for transcript references
- Deduplication as first-class concept
- Optimistic locking for config
- Node invoke wake/throttle/queue pattern
- Broadcast-first design
- Session isolation in heartbeat delivery
- Tool permission classification
- ACP binding session key format

---

## 24. Telegram Extension

*Detailed in: [24-telegram-extension.md](./24-telegram-extension.md)*

The Telegram extension is one of the most sophisticated channel plugins, with both **polling** and **webhook** modes.

**grammY Integration:**
- `TelegramPollingSession` with persistent `update_id` offset tracking
- Stall detection (120s default) with automatic restart
- IPv4/IPv6 transport fallback, exponential backoff retry

**Message Processing Pipeline (5 stages):**
1. Inbound debouncing (text fragments coalescing, media group handling)
2. Message context building (access control, route resolution, body normalization)
3. Body normalization (sticker vision, audio transcription, mention detection)
4. Context payload building (envelope formatting, group history, session recording)
5. Dispatch to agent (abort fence, streaming lanes, reasoning/answer splitting)

**Key capabilities:** Streaming preview via `createTelegramDraftStream()`, inline keyboard handling, media send with dimension validation, forum topic support with auto-labeling, reaction notifications, status reactions (lifecycle indicators).

**Deduplication:** Persistent offset store, 5-minute TTL dedupe cache, media group buffering (500ms flush).

---

## 25. Packages

*Detailed in: [14-packages.md](./14-packages.md)*

Three private npm packages:

**`@openclaw/plugin-sdk`** (`packages/plugin-sdk/`): The public SDK for building plugins. 40+ export paths covering plugin-entry, provider-entry, channel-entry-contract, runtime interfaces, tool schemas, HTTP client, etc.

**`@openclaw/memory-host-sdk`** (`packages/memory-host-sdk/`): SDK for memory/embeddings backend. Exposes runtime interfaces, engine interfaces, batch processing, remote embeddings.

**`@openclaw/plugin-package-contract`** (`packages/plugin-package-contract/`): Validates external plugin `package.json` structure.

---

## 26. CLI & Daemon

*Detailed in: [15-cli-daemon.md](./15-cli-daemon.md)*

The CLI (`src/cli/`) handles:
- Command registration and program building
- Daemon lifecycle management
- Config resolution
- Plugin command registration (via `registerCli()`)

**Daemon mode:** `openclaw gateway` runs the gateway as a long-lived daemon process.

**Key files:**
- `run-main.ts` (292 lines) — main CLI orchestration
- `progress.ts` — CLI progress display
- `tui-cli.ts` — TUI launch entry

---

## 27. Testing Strategy

*Detailed in: [16-testing.md](./16-testing-strategy.md)*

**Framework:** Vitest

**Layout:**
- Tests colocated `*.test.ts` next to source
- e2e tests `*.e2e.test.ts`
- Test helpers in `test/helpers/`

**Key patterns:**
- Per-test `vi.resetModules()` avoided in hot tests — prefer static or `beforeAll` imports
- Mock expensive runtime seams directly (scanners, manifests, package registries, filesystem, provider SDKs)
- Prefer injected deps over module mocks
- Shard timing artifact at `.artifacts/vitest-shard-timings.json`
- `OPENCLAW_LIVE_TEST=1 pnpm test:live` for live testing
- `pnpm test:perf:imports <file>` for import performance profiling

**Commands:**
- `pnpm test` — full test suite
- `pnpm test:changed` — changed tests only
- `pnpm test:extensions` — extension test shards
- `pnpm test:coverage` — coverage report

---

## 28. Key Files Reference

| File | Lines | Purpose |
|------|-------|---------|
| `src/plugins/loader.ts` | 12-stage | Plugin loading orchestration |
| `src/agents/pi-embedded-runner/run/backend.ts` | 2,640 | Single-attempt agent execution engine |
| `src/gateway/server-methods/` | 40 files | Gateway WebSocket handler methods |
| `src/acp/translator.ts` | 1,419 | ACP-to-Gateway bridge (AcpGatewayAgent) |
| `src/acp/control-plane/manager.core.ts` | 2,221 | ACP session orchestrator (AcpSessionManager) |
| `src/channels/plugins/types.adapters.ts` | 855 | All 12 channel adapter interface definitions |
| `src/channels/plugins/setup-wizard.ts` | 625 | Interactive channel setup wizard |
| `src/infra/heartbeat-runner.ts` | 1,500+ | Periodic heartbeat scheduler |
| `src/hooks/internal-hooks.ts` | ~500 | Hook registration and triggering |
| `src/tasks/task-registry.ts` | 2,000+ | In-memory task store with CRUD |
| `src/plugin-sdk/plugin-entry.ts` | 208 | `definePluginEntry()` for non-channel plugins |
| `src/plugin-sdk/channel-entry-contract.ts` | 522 | `defineBundledChannelEntry()` lazy loading |
| `ui/src/ui/app.ts` | 824 | Main `<openclaw-app>` Lit component |
| `ui/src/ui/gateway.ts` | 645 | `GatewayBrowserClient` WebSocket client |

---

## Document Index

| # | Document | Contents |
|---|----------|----------|
| 01 | `01-overview.md` (this file) | System overview, principles, directory structure, entry points |
| 02 | `02-plugin-system.md` | Plugin loader, registry, manifest, activation planning, SDK surface |
| 03 | `03-channel-system.md` | Channel plugin interface, capability adapters, channel implementations |
| 04 | `04-provider-system.md` | Provider plugins, auth methods, model catalog, capability registration |
| 05 | `05-gateway-protocol.md` | WebSocket protocol, server methods, auth modes, client implementations |
| 06 | `06-agent-system.md` | Agent runtime, embedded runner, spawn, tools, transport streams |
| 07 | `07-acp.md` | ACP session, translator, control-plane, runtime adapters |
| 08 | `08-config-system.md` | Config types, schema, loading, normalization |
| 09 | `09-memory-system.md` | Memory plugins, embedding engines, memory-host-sdk |
| 10 | `10-ui-frontend.md` | Lit components, routing, gateway client, controllers, views, i18n |
| 11 | `11-tui.md` | Terminal UI components, gateway-chat, command handlers, theme |
| 12 | `12-mobile-apps.md` | iOS, Android, macOS, Swabble, OpenClawKit shared library |
| 13 | `13-bundled-plugins.md` | Complete plugin directory, categories, and entry patterns |
| 14 | `14-packages.md` | plugin-sdk, memory-host-sdk, plugin-package-contract |
| 15 | `15-cli-daemon.md` | CLI structure, commands, daemon lifecycle, config resolution |
| 16 | `16-testing.md` | Testing strategy, patterns, commands, performance profiling |
| 17 | `17-acp-deep-dive.md` | ACP translator (1419 lines), session store (5000 max), control-plane manager (2221 lines) |
| 18 | `18-infra-hooks-tasks.md` | Retry, heartbeat-runner (1500+ lines), 30 hook events, SQLite tasks with CAS |
| 19 | `19-vendor-skills-python.md` | Google A2UI vendoring, Python skills (50+ packages), baileys patches |
| 20 | `20-plugin-loader-deep-dive.md` | Full 12-stage loading sequence, lazy runtime proxy, bundle format detection |
| 21 | `21-channel-deep-dive.md` | ~250 files, 12 adapter types, channel binding, 3-layer session fallback |
| 22 | `22-embedded-runner-deep-dive.md` | pi-embedded-runner (2640+ lines), 12-stage stream wrapper, auth rotation, compaction |
| 23 | `23-gateway-server-methods.md` | ~110 gateway methods, 28 functional categories, all handler details |
| 24 | `24-telegram-extension.md` | Telegram grammY polling/webhook, streaming lanes, dedupe, media/group/forum handling |
| 25 | `14-packages.md` | plugin-sdk (295 subpaths), memory-host-sdk, plugin-package-contract |
| 26 | `15-cli-daemon.md` | CLI structure, commands, daemon lifecycle, config resolution |
| 27 | `16-testing.md` | Testing strategy, patterns, commands, performance profiling |
| 28 | *(in this file)* | Essential files reference by topic |