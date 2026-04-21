# Channel System Architecture

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

Channels are messaging platform integrations — Telegram bots, Discord bots, Slack apps, Matrix clients, etc. Each channel plugin implements the `ChannelPlugin` interface, which is one of the most structurally complex plugin types due to the wide variety of optional capability adapters.

The channel system lives in `src/channels/` with plugins in `extensions/*` (e.g., `extensions/telegram/`, `extensions/discord/`).

---

## 2. ChannelPlugin Interface

From `src/channels/plugins/types.plugin.ts:97`:

```typescript
export type ChannelPlugin<
  ResolvedAccount,
  Probe,
  Audit
> = {
  id: ChannelId
  meta: ChannelMeta
  capabilities: ChannelCapabilities
  config: ChannelConfigAdapter<ResolvedAccount>
  setup?: ChannelSetupAdapter
  pairing?: ChannelPairingAdapter
  security?: ChannelSecurityAdapter<ResolvedAccount>
  auth?: ChannelAuthAdapter
  approvalCapability?: ChannelApprovalCapability
  outbound?: ChannelOutboundAdapter
  status?: ChannelStatusAdapter<ResolvedAccount, Probe, Audit>
  gateway?: ChannelGatewayAdapter<ResolvedAccount>
  messaging?: ChannelMessagingAdapter
  streaming?: ChannelStreamingAdapter
  threading?: ChannelThreadingAdapter
  groups?: ChannelGroupsAdapter
  mentions?: ChannelMentionsAdapter
  heartbeat?: ChannelHeartbeatAdapter
  actions?: ChannelActionsAdapter
  resolver?: ChannelResolverAdapter
  secrets?: ChannelSecretsAdapter
  allowlist?: ChannelAllowlistAdapter
  doctor?: ChannelDoctorAdapter
  bindings?: ChannelBindingsAdapter
  directory?: ChannelDirectoryAdapter
  commands?: ChannelCommandsAdapter
}
```

All capability adapters are **optional** — a channel plugin only needs to implement the capabilities it actually supports.

---

## 3. Channel Capability Adapters

### 3.1 Messaging Adapter

```typescript
interface ChannelMessagingAdapter {
  sendMessage(opts: SendMessageOptions): Promise<SendMessageResult>
  sendMedia?(opts: SendMediaOptions): Promise<SendMessageResult>
  sendReaction?(opts: SendReactionOptions): Promise<void>
  onInboundMessage(handler: InboundMessageHandler): void
  // ...
}
```

Core send/receive interface. `sendMessage` is the primary method for outbound messages. `onInboundMessage` registers a handler for incoming messages from the channel.

### 3.2 Streaming Adapter

```typescript
interface ChannelStreamingAdapter {
  sendMessageStream(opts: StreamSendMessageOptions): AsyncGenerator<StreamChunk, void, unknown>
}
```

For channels that support streaming responses — used when the agent produces a long response and the channel can stream it to the user in real-time.

### 3.3 Outbound Adapter

```typescript
interface ChannelOutboundAdapter {
  sendPushNotification?(opts: PushNotificationOptions): Promise<void>
  // Used for proactive bot-initiated messages, not just reactive responses
}
```

For push notification capabilities — channels that support the bot initiating conversations.

### 3.4 Security Adapter

```typescript
interface ChannelSecurityAdapter<ResolvedAccount> {
  validateSecuritySignature?(opts: ValidateSignatureOptions): Promise<boolean>
  resolveAccount?(opts: ResolveAccountOptions): Promise<ResolvedAccount>
  // Handles auth token validation, webhook signature verification, etc.
}
```

Handles authentication/authorization for the channel.

### 3.5 Auth Adapter

```typescript
interface ChannelAuthAdapter {
  getAuthUrl?(): Promise<string>
  exchangeCode?(code: string): Promise<AuthResult>
  refreshToken?(refreshToken: string): Promise<AuthResult>
  // Channel-specific OAuth flows (Telegram login, Discord OAuth, etc.)
}
```

Channel-specific authentication flows separate from the core security adapter.

### 3.6 Setup Adapter

```typescript
interface ChannelSetupAdapter {
  validateConfig?(config: unknown): ValidateResult
  inspectAccount?(config: unknown): Promise<AccountInspection>
  healthProbe?(config: unknown): Promise<HealthResult>
}
```

Channel configuration validation, account inspection, and health checks.

### 3.7 Pairing Adapter

```typescript
interface ChannelPairingAdapter {
  startPairing?(opts: StartPairingOptions): Promise<PairingResult>
  completePairing?(token: string): Promise<void>
  // For channels that support device pairing (QR code, token exchange, etc.)
}
```

Device pairing for channels that support it.

### 3.8 Approval Capability

```typescript
interface ChannelApprovalCapability {
  requestApproval?(opts: ApprovalRequestOptions): Promise<ApprovalResult>
  onApprovalResult?(handler: ApprovalResultHandler): void
  // For channels where certain actions require user approval
}
```

Handles approval request flows (e.g., "User asked to send an email — approve?"). Used by the `execApproval` system.

### 3.9 Gateway Adapter

```typescript
interface ChannelGatewayAdapter<ResolvedAccount> {
  registerGatewayMethod?(method: string, handler: GatewayMethodHandler): void
  // Allows channels to expose custom gateway RPC methods
}
```

### 3.10 Other Adapters

| Adapter | Purpose |
|---------|---------|
| `threading` | Thread/topic management |
| `groups` | Group chat support |
| `mentions` | Mention detection (`@bot`, `#channel`) |
| `heartbeat` | Keepalive/ping-pong |
| `status` | Health/status reporting |
| `secrets` | Credential storage |
| `allowlist` | Sender allowlisting |
| `doctor` | Diagnostic support |
| `bindings` | Conversation bindings |
| `directory` | Channel directory lookup |
| `resolver` | Target resolution |
| `actions` | Message actions |

---

## 4. Channel Entry Contract

### 4.1 defineBundledChannelEntry

From `src/plugin-sdk/channel-entry-contract.ts:522`, the channel entry uses lazy loading:

```typescript
export function defineBundledChannelEntry({
  id,
  name,
  description,
  importMetaUrl,
  plugin: { specifier: pluginSpecifier, exportName: pluginExportName },
  secrets: { specifier: secretsSpecifier, exportName: secretsExportName },
  runtime: { specifier: runtimeSpecifier, exportName: runtimeExportName },
  accountInspect: { specifier: accountInspectSpecifier, exportName: accountInspectExportName },
})
```

Four separate lazy module references:
1. **`plugin`** — the main `ChannelPlugin` implementation
2. **`secrets`** — secret contract API (optional)
3. **`runtime`** — runtime setter for injecting `PluginRuntime`
4. **`accountInspect`** — read-only account inspection (optional)

### 4.2 Channel Plugin API (plugin side)

Each channel plugin's `channel-plugin-api.js` exports a `ChannelPlugin` instance:

```typescript
// extensions/telegram/src/channel-plugin-api.ts (simplified)
export const telegramPlugin: ChannelPlugin<
  TelegramAccount,
  TelegramProbe,
  TelegramAudit
> = {
  id: "telegram",
  meta: { name: "Telegram", icon: "..." },
  capabilities: ["messaging", "streaming", "security", "setup"],
  config: { validateConfig, inspectAccount, healthProbe },
  security: { validateSecuritySignature, resolveAccount },
  messaging: { sendMessage, sendMedia, sendReaction, onInboundMessage },
  streaming: { sendMessageStream },
  // ...
}
```

### 4.3 Runtime API (plugin side)

The `runtime-api.js` provides a setter for injecting the core `PluginRuntime`:

```typescript
// extensions/telegram/src/runtime-api.ts (simplified)
export function setTelegramRuntime(runtime: PluginRuntime) {
  // Store runtime for use in plugin implementation
  // Access to: runtime.subagent, runtime.channel, runtime.acp
}
```

---

## 5. Hot Path Rules

Per `src/channels/AGENTS.md`, the hot path rules are:

1. **Keep async-only surfaces off hot entrypoints** — `send`, `monitor`, `probe`, `setup/login` should not be on hot code paths (i.e., module-level executes at import time)
2. **Use small local seams** like `channel-api.ts` or `*.runtime.ts` for heavy code — do not mix static initialization with async operations
3. **Prefer lightweight bundled-plugin artifacts** before falling back to full channel loading

---

## 6. Channel Core Types

Key type files:
- `src/channels/plugins/types.plugin.ts` — `ChannelPlugin` interface (97 lines)
- `src/channels/plugins/types.core.ts` — core channel types: `ChannelMessagingAdapter`, `ChannelOutboundAdapter`, etc.
- `src/channels/plugins/types.adapters.ts` — adapter interfaces for each capability

---

## 7. Bundled Channel Plugins

Complete list of channel plugins in `extensions/`:

| Extension | Platform | Notes |
|----------|----------|-------|
| `telegram` | Telegram | Full messaging + streaming + security |
| `discord` | Discord | Bot channel + subagent hooks |
| `slack` | Slack | HTTP routes |
| `msteams` | Microsoft Teams | |
| `mattermost` | Mattermost | |
| `matrix` | Matrix protocol | |
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

---

## 8. Channel-to-Agent Flow

```
User sends message on Telegram
  → Telegram channel plugin receives via onInboundMessage
  → Routes to core messaging handler
  → Core looks up agent session
  → Agent processes message (calls provider)
  → Agent produces response
  → Core sends via channel's sendMessage
  → Telegram bot sends reply
```

The channel plugin handles platform-specific transport (polling webhooks, WebSocket connections, etc.) but the message routing and agent invocation is core.

---

## 9. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/channels/plugins/types.plugin.ts` | 97 | `ChannelPlugin` interface |
| `src/channels/plugins/types.core.ts` | ~200 | Core channel types |
| `src/channels/plugins/types.adapters.ts` | ~300 | All adapter interfaces |
| `src/plugin-sdk/channel-entry-contract.ts` | 522 | `defineBundledChannelEntry()` lazy loading |
| `src/plugin-sdk/channel-contract.ts` | 39 | Pure channel contract types |
| `extensions/telegram/src/channel-plugin-api.ts` | ~200 | Telegram plugin implementation |
| `extensions/telegram/src/runtime-api.ts` | ~50 | Telegram runtime setter |
| `src/channels/AGENTS.md` | ~60 | Channel architecture rules |