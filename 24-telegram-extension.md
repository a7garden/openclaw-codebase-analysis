# Telegram Extension Deep-Dive

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Entry Point & Plugin Structure

### Plugin Registration (`extensions/telegram/index.ts`)

The Telegram extension uses the OpenClaw plugin SDK's `defineBundledChannelEntry` to register itself:

```typescript
export default defineBundledChannelEntry({
  id: "telegram",
  name: "Telegram",
  description: "Telegram channel plugin",
  importMetaUrl: import.meta.url,
  plugin: {
    specifier: "./channel-plugin-api.js",
    exportName: "telegramPlugin",
  },
  secrets: {
    specifier: "./secret-contract-api.js",
    exportName: "channelSecrets",
  },
  runtime: {
    specifier: "./runtime-api.js",
    exportName: "setTelegramRuntime",
  },
  accountInspect: {
    specifier: "./account-inspect-api.js",
    exportName: "inspectTelegramReadOnlyAccount",
  },
});
```

**Key registration points:**
- **`plugin`**: Exports `telegramPlugin` from `channel-plugin-api.ts` — the main plugin instance
- **`secrets`**: Exports `channelSecrets` for bot token management
- **`runtime`**: Exports `setTelegramRuntime` for runtime store access
- **`accountInspect`**: Exports account inspection functionality

### Manifest (`openclaw.plugin.json`)

```json
{
  "id": "telegram",
  "channels": ["telegram"],
  "channelEnvVars": {
    "telegram": ["TELEGRAM_BOT_TOKEN"]
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

The manifest declares:
- Channel ID: `telegram`
- Supported channels array (for multi-bot setups)
- Environment variable: `TELEGRAM_BOT_TOKEN`
- Empty config schema (schema validation delegated to plugin code)

### Main Plugin Export (`channel-plugin-api.ts`)

```typescript
export { telegramPlugin } from "./src/channel.js";
export { telegramSetupPlugin } from "./src/channel.setup.js";
```

The plugin uses `createChatChannelPlugin` from the plugin SDK — a comprehensive adapter that wires together:
- Messaging (normalizeTarget, resolveInboundConversation, resolveOutboundSessionRoute)
- Bindings (configured bindings, thread bindings, conversation bindings)
- Security (DM policy, group allowlist)
- Lifecycle (account start/stop, legacy migrations)
- Status/Audit (probe, group membership audit)
- Gateway (startAccount, logoutAccount)
- Threading (top-level reply-to mode)
- Outbound (base adapter, attached results)

---

## 2. grammY Integration

### Polling vs Webhook Modes

The Telegram plugin supports both polling and webhook modes, selected at account start time based on configuration:

**Polling** (`monitor-polling.runtime.ts` / `polling-session.ts`):
- Uses grammY's `run()` runner with `TelegramPollingSession` class
- Custom `allowed_updates` filter to minimize unnecessary API calls
- Persistent `update_id` offset tracking via `update-offset-store.ts`
- Stall detection (120s default) with automatic restart
- IPv4/IPv6 transport fallback support
- Exponential backoff retry (initial 2s, max 30s, 1.8x factor, 0.25 jitter)

**Webhook** (`webhook.ts`):
- Uses grammY's `webhookCallback()` with HMAC secret validation
- Internal HTTP server (default `127.0.0.1:8787`)
- Per-source rate limiting before secret validation
- Client IP resolution via X-Forwarded-For with trusted proxy support
- Health endpoint at `/healthz`

### Bot Token Management (`token.ts`)

Token resolution follows a priority cascade:
1. Per-account `tokenFile` (secret file path)
2. Per-account `botToken` (direct config value)
3. Channel-level `tokenFile`
4. Channel-level `botToken`
5. Environment variable `TELEGRAM_BOT_TOKEN`
6. Result: `{ token: "", source: "none" }`

Multi-account behavior:
- With explicit `accounts` section: unknown account IDs refuse channel-level fallback
- Without `accounts` section (single-bot): allow channel-level fallback

### Update Handling

**Supported update types** (via `allowed-updates.ts`):
- `message`, `edited_message`, `channel_post`, `edited_channel_post`
- `callback_query`, `message_reaction`, `message_reaction_count`
- `inline_query` (not currently used but available)

**Update type routing** in `bot-handlers.runtime.ts`:

| Handler | grammY Event | Purpose |
|---------|-------------|---------|
| `bot.on("message")` | `message` | Primary inbound messages |
| `bot.on("channel_post")` | `channel_post` | Bot-to-bot channel comms |
| `bot.on("callback_query")` | `callback_query` | Inline button interactions |
| `bot.on("message_reaction")` | `message_reaction` | Emoji reaction events |
| `bot.on("message:migrate_to_chat_id")` | `migrate_to_chat_id` | Group migration handling |

---

## 3. Message Processing Pipeline

### Stage 1: Inbound Debouncing (`registerTelegramHandlers`)

**Text Fragment Coalescing** (lines 938-997):
- Buffers near-limit messages (~4096 char Telegram limit)
- Groups consecutive messages within 1 message ID gap and 1.5s time window
- Maximum 12 parts / 50,000 total chars before forcing flush
- Debounce key: `text:${chatId}:${threadId}:${senderId}` with "default" or "forward" lane

**Media Group Coalescing** (lines 1000-1032):
- 500ms timeout after last message before processing
- Sorts messages by `message_id`
- Uses first message with caption/text as primary entry
- Recovers from partial fetch failures gracefully

**Forward Burst Debouncing**:
- Forwarded messages use 80ms debounce window (vs default debounceMs)
- Allows media+text in forwarded bursts to coalesce

### Stage 2: Message Context Building (`buildTelegramMessageContext`)

Returns a `TelegramMessageContext` containing:

```typescript
export type TelegramMessageContext = {
  ctxPayload: TelegramMessageContextPayload;
  primaryCtx: BuildTelegramMessageContextParams["primaryCtx"];
  msg: Message;
  chatId: number;
  isGroup: boolean;
  groupConfig?: TelegramGroupConfig;
  topicConfig?: TelegramTopicConfig;
  resolvedThreadId?: number;
  threadSpec: TelegramThreadSpec;
  replyThreadId?: number;
  isForum: boolean;
  historyKey?: string;
  historyLimit: number;
  groupHistories: Map<string, HistoryEntry[]>;
  route: ResolvedAgentRoute;
  skillFilter: string[] | undefined;
  sendTyping: () => Promise<void>;
  sendRecordVoice: () => Promise<void>;
  ackReactionPromise: Promise<boolean> | null;
  reactionApi: TelegramReactionApi | null;
  removeAckAfterReply: boolean;
  statusReactionController: TelegramStatusReactionController | null;
  accountId: string;
};
```

**Key build steps**:
1. **Forum Detection**: Queries `getChat` API if `is_forum` not cached
2. **Thread Spec Resolution**: Determines DM vs forum topic scope
3. **Access Control**: Evaluates `groupConfig`, `topicConfig`, `allowFrom` overrides
4. **Route Resolution**: Uses `resolveTelegramConversationRoute()` to determine agent/session
5. **Session Key**: Handles per-DM-topic isolation with thread suffix
6. **Body Resolution**: Calls `resolveTelegramInboundBody()` for text normalization
7. **Status Reactions**: Optional ACK reaction via `setMessageReaction` API

### Stage 3: Body Normalization (`bot-message-context.body.ts`)

The `resolveTelegramInboundBody()` function:

1. **Sticker Vision**: Checks cache for sticker description; runs vision model if needed
2. **Audio Transcription**: Preflight transcription for voice messages (if no text and mention required)
3. **Location Extraction**: Converts location data to text format
4. **Text Expansion**: Expands `text_link` entities into inline text
5. **Mention Detection**: Evaluates explicit @mention and implicit reply-to-bot scenarios
6. **Command Authorization**: Uses `resolveControlCommandGate()` for native command access control
7. **Group Policy Enforcement**: Blocks messages when `requireMention=true` and no valid mention, unless `ingest=true` (fires internal hook for async processing)

**Return shape**:
```typescript
type TelegramInboundBodyResult = {
  bodyText: string;
  rawBody: string;
  historyKey?: string;
  commandAuthorized: boolean;
  effectiveWasMentioned: boolean;
  canDetectMention: boolean;
  shouldBypassMention: boolean;
  stickerCacheHit: boolean;
  locationData?: NormalizedLocation;
};
```

### Stage 4: Context Payload Building (`bot-message-context.session.ts`)

The `buildTelegramInboundContextPayload()` function builds the final `ctxPayload` for the agent:

1. **Envelope Formatting**: Uses `formatInboundEnvelope()` for structured message encoding
2. **Reply/Forward Context**: Extracts and includes quoted replies and forwarded origins
3. **Group History**: Prepends recent message context (configurable limit)
4. **Session Recording**: Updates session store with `recordInboundSession()`
5. **Last Route Persistence**: Updates routing metadata for quick reconnections
6. **DM Topic Pinning**: Resolves `pinnedMainDmOwnerFromAllowlist` for DM topic sessions

### Stage 5: Dispatch to Agent (`bot-message-dispatch.ts`)

The `dispatchTelegramMessage()` function orchestrates the response flow:

**Abort Fence System**:
```typescript
// Prevents older replies from racing newer ones in the same session
const telegramAbortFenceByKey = new Map<string, TelegramAbortFenceState>();
// Key: SessionKey or telegram:${chatId}:${threadScope}:${threadId}
// Generation increments on supersede (abort command)
// Blocks active dispatches until generation changes
```

**Streaming Lane Architecture**:
- `answer` lane: Primary response text with `sendMessage` / `editMessageText`
- `reasoning` lane: Separate reasoning stream (when `reasoningLevel: "stream"`)
- Drafts use `createTelegramDraftStream()` with throttling (default 1s)

**Reasoning Split** (`reasoning-lane-coordinator.ts`):
```typescript
splitTelegramReasoningText(text)
// Extracts <think>... tags outside code regions
// Formats reasoning: "Reasoning:\n<reasoning>"
// Answer: remaining text after stripping tags
```

**Delivery State Tracking** (`lane-delivery-state.ts`):
- Tracks `delivered`, `skippedNonSilent`, `failedNonSilent` counts
- Provides `snapshot()` for final delivery summary

---

## 4. Telegram-Specific Features

### Streaming Response Handling (`draft-stream.ts`)

**`createTelegramDraftStream()`** creates a streaming preview that:

1. **Throttles Updates**: Minimum 250ms between sends (configurable)
2. **Respects Telegram Limits**: 4096 char max per message
3. **Uses sendMessageDraft When Available**: Telegram's native draft API (avoids edit conflicts)
4. **Falls Back to sendMessage/editMessageText**: For older clients or draft-unsupported chats

**Transport Selection**:
```typescript
const prefersDraftTransport = params.thread?.scope === "dm"; // DM uses draft by default
// Falls back to "message" transport (edit-based) for groups
```

**Generation System**:
- `forceNewMessage()` increments generation to abandon old preview
- Superseded previews tracked via `onSupersededPreview` callback
- Cleanup deletes superseded previews after final delivery

**Materialization**: Converts draft to permanent message on final delivery

### Inline Keyboard Handling

**Callback Data Types** (`interactive-dispatch.ts`):
- `Exec Approval`: `approve:${id}:accept|reject`
- `Plugin Binding`: `plugin-binding:${approvalId}:accept|reject`
- `Commands Pagination`: `commands_page_${page}:${agentId}`
- `Model Selection`: `mdl_prov|mdl_list_${provider}_${page}|mdl_sel_${provider}_${model}|mdl_back`
- `Native Commands`: Encoded via `parseTelegramNativeCommandCallbackData`

**Button Builder** (`inline-keyboard.ts`):
```typescript
buildInlineKeyboard(buttons)
// Maps to Telegram's inline_keyboard format
// Supports style hints (danger/success/primary) via data-* prefix
```

### Media Handling

**Resolution** (`bot/delivery.resolve-media.ts`):
- Fetches file via `getFile` API
- Caches to local temp storage
- Extracts metadata (dimensions, duration, MIME type)
- Handles sticker metadata extraction

**Send Logic** (`send.ts`):
- **Images**: Validates dimensions (< 10,000px sum, < 20:1 aspect ratio), sends as photo or document
- **Video**: Supports video note (square, looped) via `asVideoNote` option
- **Audio**: Supports voice note via `asVoice` option
- **Animation**: GIF sent as animation
- **Document**: Everything else, with `forceDocument` option

**Caption Splitting** (`caption.ts`):
- Telegram caption limit ~1024 chars
- Splits into caption + follow-up text for longer content

### Group Chat Handling

**Access Evaluation** (`group-access.ts`):
```typescript
evaluateTelegramGroupBaseAccess()
// Checks: group enabled, topic enabled, allowFrom override authorization
// Returns: { allowed: true } | { allowed: false, reason: "group-disabled"|"topic-disabled"|"group-override-unauthorized" }

evaluateTelegramGroupPolicyAccess()
// Checks: chat-level allowlist, sender authorization
// Policy modes: "open", "disabled", "allowlist"
```

**Group Migration** (`migrate_to_chat_id` handler):
- Detects group ID change
- Migrates configuration from old to new chat ID
- Updates config file persist

### Forum Topics (`bot/helpers.ts`)

**Thread Spec**:
```typescript
resolveTelegramThreadSpec({ isGroup, isForum, messageThreadId })
// Returns: { id?: number, scope: "dm" | "forum" | "none" }
// General forum topic (id=1) must NOT be sent with message_thread_id
```

**Topic Naming** (`auto-topic-label.ts`):
- On first turn in DM topic, generates label from user message
- Uses LLM with configured prompt
- Renames topic via `editForumTopic`

### Reaction Notifications

**Modes** (`cfg.messages?.reactionNotifications`):
- `"own"`: Only react to messages sent by bot (default)
- `"off"`: Ignore all reactions

**Event Processing**:
- Filters bot-authored reactions
- Enqueues system events for added reactions
- Uses `setMessageReaction` API (Telegram v6.4+)

### Status Reactions (Lifecycle Indicators)

**Controller** (`TelegramStatusReactionController`):
```typescript
setQueued()      // Replace ACK emoji with queued indicator
setThinking()    // Show "typing" indicator
setTool(name)    // Show tool execution indicator
setCompacting()  // Show compaction indicator
setError()       // Show error indicator
setDone()        // Restore to final state
restoreInitial() // Remove reaction
cancelPending() // Cancel pending transition
```

**Configuration** (`cfg.messages?.statusReactions`):
- `enabled`: boolean
- `emojis`: override emoji set
- `timing`: hold durations (queuedHoldMs, doneHoldMs, errorHoldMs)

---

## 5. Deduplication & Idempotency

### Update ID Tracking (`update-offset-runtime-api.ts`)

**Persistent Offset Store** (`update-offset-store.ts`):
- Stores last confirmed `update_id` per account
- JSON file storage at plugin-controlled path
- Normalization: rejects unsafe integers (< 0, non-finite)
- Confirm-on-startup: Fetches one update at `lastUpdateId + 1` to confirm validity

**Recovery Flow**:
1. Read persisted offset on startup
2. If invalid (null/non-integer), start fresh (no offset confirmation)
3. If valid, call `getUpdates(offset: lastUpdateId + 1, limit: 1, timeout: 0)` to verify
4. Non-fatal failure: proceed with local tracking only

### Message Deduplication (`bot-updates.ts`)

**Dedupe Cache** (`createTelegramUpdateDedupe()`):
- TTL: 5 minutes
- Max entries: 2000
- Key: `update:${updateId}` or `callback:${callbackId}` or `message:${chatId}:${messageId}`

**Synthetic ID for Unidentified Updates**:
- Messages without `update_id`: composite key from `chatId:messageId`
- Ensures media groups don't produce duplicate processing

### Media Group Handling

**Buffer** (`mediaGroupBuffer` Map):
- Keyed by `media_group_id`
- Timer: 500ms flush timeout
- Sort by `message_id` before processing
- Recoverable errors (network) skip individual photos

### Text Fragment Handling

**Buffer** (`textFragmentBuffer` Map):
- Key: `text:${chatId}:${threadId}:${senderId}`
- Conditions for append: message ID gap ≤ 1, time gap ≤ 1.5s
- Limits: 12 parts max, 50,000 chars max
- Exceeded: flush existing, start new buffer

---

## 6. Outbound Message Flow

### Entry Points

**`sendMessageTelegram(to, text, opts)`** — Primary send function

**`deliverReplies()`** — Multi-payload delivery via `bot-message-dispatch.ts`

### Core Send Pipeline (`send.ts`)

1. **Target Resolution**:
   - Numeric chat ID: use directly
   - Username: lookup via `getChat` API, persist resolution

2. **HTML Rendering** (`format.ts`):
   - Markdown/HTML → Telegram HTML via `renderTelegramHtmlText()`
   - Code blocks, tables (configurable mode), links

3. **Chunking**:
   - Split at 4000 char boundaries
   - Preserve entity integrity where possible
   - Fall back to plain text if HTML parsing fails

4. **Thread Parameters** (`buildTelegramThreadReplyParams`):
   - `message_thread_id`: for forum topics and DM topics
   - General topic (id=1): strip thread_id (Telegram rejects it)
   - `reply_to_message_id`: for reply threading
   - `reply_parameters`: for quote (with text)

5. **Transport Fallback**:
   - Thread-not-found: retry without `message_thread_id`
   - Chat-not-found: enriched error with resolution hints

6. **Error Handling**:
   - Rate limit: retry with exponential backoff
   - Server error (5xx): retry safe
   - Network error: retry safe
   - "Bot not member": fail fast with helpful message
   - "Chat not found": fail fast with migration hints

### Reply Threading

**Top-Level Reply Mode** (`threading.topLevelReplyToMode: "telegram"`):
- Uses Telegram's native `reply_to_message_id`
- Quote support with `reply_parameters`

**Per-Message Threading**:
- Forum topics: `message_thread_id` parameter
- DM topics: `message_thread_id` parameter

### Reply Suppression

**`targetsMatchForReplySuppression`**:
- Same chat ID (case-insensitive)
- Same thread ID (both null OR both equal)
- Prevents duplicate delivery across binding paths

### Lane Delivery (`lane-delivery-text-deliverer.ts`)

**Preview Lifecycle**:
- `"transient"`: Can be edited/deleted on final delivery
- `"complete"`: Retained permanently

**Archived Previews**:
- Superseded answer previews stored for potential final-edit replacement
- Deleted on cleanup if unconsumed

**Final Delivery Strategy**:
1. If archived preview exists: try to edit it (preserves message ID)
2. If no archived preview and can edit: edit active stream
3. If cannot edit (too long, media, error): send as new message
4. Delete transient previews on cleanup

---

## 7. Configuration & Secrets

### Configuration Schema (`config-api.ts` / `config-schema.ts`)

**`TelegramConfigSchema`**:
```typescript
type TelegramAccountConfig = {
  botToken?: SecretRef | string;
  tokenFile?: string;
  webhookUrl?: string;
  webhookSecret?: string;
  webhookPath?: string;
  webhookHost?: string;
  webhookPort?: number;
  webhookCertPath?: string;
  proxy?: string;
  apiRoot?: string;
  network?: {
    autoSelectFamily?: boolean;
    dnsResultOrder?: "ipv4first" | "ipv6first";
  };
  timeoutSeconds?: number;
  retry?: RetryConfig;
  dmPolicy?: "open" | "pairing" | "disabled";
  allowFrom?: AllowFromConfig;
  groupPolicy?: "open" | "disabled" | "allowlist";
  groupAllowFrom?: AllowFromConfig;
  groups?: Record<string, TelegramGroupConfig>;
  reactionNotifications?: "own" | "off";
  statusReactions?: {
    enabled?: boolean;
    emojis?: Record<string, string>;
    timing?: StatusReactionTiming;
  };
  linkPreview?: boolean;
  pollingStallThresholdMs?: number;
  mediaMaxMb?: number;
  autoTopicLabel?: AutoTopicLabelConfig;
  // ACP binding-specific
  bindingMode?: "single" | "multi";
  blockStreamingDefault?: "on" | "off";
  inlineButtonsScope?: "off" | "dm" | "group" | "allowlist";
  // Approval config
  approvals?: ApprovalConfig;
};
```

### Per-Chat Configuration

**Group Config** (`telegramConfigAdapter.formatGroupConfig`):
```typescript
type TelegramGroupConfig = {
  enabled?: boolean;
  requireMention?: boolean;
  allowFrom?: AllowFromConfig;
  dmPolicy?: DmPolicy;
  groupPolicy?: ChannelGroupPolicy;
  topics?: Record<string, TelegramTopicConfig>;
  // ACP binding
  agentId?: string;
  requireTopic?: boolean;
  autoTopicLabel?: AutoTopicLabelConfig;
  disableAudioPreflight?: boolean;
  ingest?: boolean;
};
```

**Topic Config**: Inherits from group with optional overrides

### Secrets Storage

**`channelSecrets`** (via `secret-contract-api.ts`):
- Supports `SecretRef` references (`env:NAME`, `file:PATH`)
- Environment variable: `TELEGRAM_BOT_TOKEN` (via `channelEnvVars` in manifest)
- Token file: `tokenFile` path (absolute or relative to config)

### Webhook Security

**Secret Validation**:
- HMAC-SHA256 via `safeEqualSecret()` (timing-safe comparison)
- Header: `X-Telegram-Bot-Api-Secret-Token`
- Rejects immediately before rate limit counting

**Per-Source Rate Limiting**:
- Fixed window: 60s, 30 requests per window per IP
- Applied before secret validation

### Status/Audit Probes

**`probeTelegram()`** (`probe.ts`):
- Calls `getMe`, `getWebhookInfo`
- Returns bot info, join group flags, inline query support
- Used for account status display

**`auditTelegramGroupMembership()`** (`audit.ts`):
- Batch checks group membership via `getChatMember`
- Reports unresolvable groups and wildcard unmentioned groups

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                        grammY Bot Instance                           │
│  ┌──────────┐  ┌──────────────┐  ┌────────────────┐  ┌───────────┐ │
│  │ on(msg)  │  │ on(callback)  │  │ on(reaction)   │  │ webhook   │ │
│  └────┬─────┘  └──────┬───────┘  └───────┬────────┘  └─────┬─────┘ │
└───────┼───────────────┼─────────────────┼─────────────────┼───────┘
        │               │                 │                 │
        ▼               ▼                 ▼                 ▼
┌───────────────────────────────────────────────────────────────────┐
│                    bot-handlers.runtime.ts                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  registerTelegramHandlers()                                 │   │
│  │  - inboundDebouncer (text fragments, media groups)          │   │
│  │  - processInboundMessage()                                  │   │
│  │  - processMediaGroup()                                       │   │
│  │  - processCallbackQuery()                                    │   │
│  │  - processReaction()                                         │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
└──────────────────────────────┼──────────────────────────────────────┘
                               │
                               ▼
┌───────────────────────────────────────────────────────────────────┐
│                 bot-message-context.ts                            │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │  buildTelegramMessageContext()                              │   │
│  │  - Access evaluation (group/topic allowFrom)                │   │
│  │  - Route resolution (agent, session key)                    │   │
│  │  - resolveTelegramInboundBody()                            │   │
│  │  - buildTelegramInboundContextPayload()                    │   │
│  └──────────────────────────┬─────────────────────────────────┘   │
└──────────────────────────────┼──────────────────────────────────────┘
                               │
                               ▼
┌───────────────────────────────────────────────────────────────────┐
│                 bot-message-dispatch.ts                            │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │  dispatchTelegramMessage()                                  │   │
│  │  - Abort fence management                                   │   │
│  │  - Lane delivery (answer/reasoning)                         │   │
│  │  - createChannelReplyPipeline()                            │   │
│  │  - Streaming preview management                            │   │
│  └──────────────────────────┬─────────────────────────────────┘   │
└──────────────────────────────┼──────────────────────────────────────┘
                               │
                               ▼
┌───────────────────────────────────────────────────────────────────┐
│                      Plugin SDK Core                               │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │  createChatChannelPlugin()                                  │   │
│  │  - Messaging bindings                                       │   │
│  │  - Conversation bindings                                   │   │
│  │  - Security policies                                        │   │
│  │  - Threading management                                     │   │
│  │  - Outbound adapter                                          │   │
│  └────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
```

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `index.ts` | Plugin entry point registration |
| `channel-plugin-api.ts` | Main plugin export |
| `src/channel.ts` | `createChatChannelPlugin()` configuration |
| `src/runtime.ts` | Runtime store (setTelegramRuntime) |
| `src/monitor.ts` | Provider monitor (polling/webhook dispatcher) |
| `src/monitor-polling.runtime.ts` | `TelegramPollingSession` class |
| `src/monitor-webhook.runtime.ts` | Webhook startup |
| `src/webhook.ts` | HTTP server + webhook handling |
| `src/bot-handlers.runtime.ts` | grammY event handlers |
| `src/bot-handlers.ts` | `registerTelegramHandlers()` |
| `src/bot-message-context.ts` | Context building |
| `src/bot-message-context.body.ts` | Body normalization |
| `src/bot-message-context.session.ts` | Session/context payload building |
| `src/bot-message-dispatch.ts` | Response dispatch orchestration |
| `src/send.ts` | Outbound send functions |
| `src/draft-stream.ts` | Streaming preview |
| `src/lane-delivery.ts` | Lane delivery abstractions |
| `src/lane-delivery-text-deliverer.ts` | Lane delivery implementation |
| `src/reasoning-lane-coordinator.ts` | Reasoning/answer splitting |
| `src/conversation-route.ts` | Agent/session routing |
| `src/token.ts` | Bot token resolution |
| `src/dm-access.ts` | DM access control |
| `src/group-access.ts` | Group access control |
| `src/bot/helpers.ts` | Telegram helpers (thread params, etc.) |
| `src/bot/types.ts` | Telegram types |
| `src/update-offset-runtime-api.ts` | Offset store exports |
| `src/update-offset-store.ts` | Persistent offset storage |