# Channel System Deep Dive

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Channel Architecture Stats

- **~250 files** in `src/channels/`
- **~29,687 lines** of TypeScript
- **~180 files** in `src/channels/plugins/`
- **12 adapter types** composing the ChannelPlugin interface

---

## 2. All Channel Adapter Types

### 2.1 ChannelConfigAdapter (config resolution)

```typescript
listAccountIds(cfg)                    // List all account IDs in config
resolveAccount(cfg, accountId?)        // Resolve account from config
inspectAccount?(cfg, accountId?)       // Inspect account state
defaultAccountId?(cfg)                  // Default account
setAccountEnabled?(params)             // Enable/disable account
deleteAccount?(params)                 // Delete account
isEnabled?(account, cfg)               // Check if enabled
isConfigured?(account, cfg)            // Check if configured
describeAccount?(account, cfg)        // Get account snapshot
resolveAllowFrom?(params)              // Allowlist resolution
formatAllowFrom?(params)               // Format allowlist for display
hasConfiguredState?(params)            // Has persistent config
hasPersistedAuthState?(params)         // Has auth state
```

### 2.2 ChannelSecurityAdapter (auth & validation)

```typescript
applyConfigFixes?(cfg)                  // Doctor auto-fix mutations
resolveDmPolicy?(ctx)                   // DM policy resolution
collectWarnings?(ctx)                  // Security warnings
collectAuditFindings?(ctx)             // Audit report data
```

### 2.3 ChannelOutboundAdapter (sending messages)

```typescript
deliveryMode: "direct" | "gateway" | "hybrid"
chunker?: (text, limit) => string[]     // Split long messages
textChunkLimit?: number                  // Per-chunk limit
sanitizeText?(params)                   // Sanitize output
sendPayload?(ctx)                        // Core send
sendFormattedText?(ctx)                  // Formatted text send
sendFormattedMedia?(ctx)                 // Media send
sendText?(ctx)                           // Simple text
sendMedia?(ctx)                          // Media with url
sendPoll?(ctx)                           // Poll creation
pollMaxOptions?: number                  // Max options
normalizePayload?(payload)               // Transform before send
```

### 2.4 ChannelGatewayAdapter (channel lifecycle)

```typescript
startAccount?(ctx)                     // Start channel session
stopAccount?(ctx)                      // Stop channel session
resolveGatewayAuthBypassPaths?(cfg)    // Auth bypass for paths
loginWithQrStart?(params)               // QR login start
loginWithQrWait?(params)                // QR login wait
logoutAccount?(ctx)                     // Logout
```

### 2.5 ChannelDirectoryAdapter (user/group lookup)

```typescript
self?(params)                          // Get self profile
listPeers?(params)                      // List contacts
listPeersLive?(params)                  // Live contact list
listGroups?(params)                     // List groups
listGroupsLive?(params)                 // Live group list
listGroupMembers?(params)               // Get group members
```

### 2.6 ChannelMessagingAdapter (inbound & formatting)

```typescript
normalizeTarget?(raw)                   // Parse target string
deriveLegacySessionChatType?(sessionKey) // "direct" | "group" | "channel"
resolveSessionConversation?(params)     // Resolve conversation ref
resolveInboundConversation?(params)     // Resolve inbound
resolveDeliveryTarget?(params)          // Resolve send target
inferTargetChatType?(to)                // Infer chat type
buildCrossContextComponents?(params)    // Cross-channel components
transformReplyPayload?(params)          // Transform reply
```

### 2.7 ChannelThreadingAdapter (thread management)

```typescript
resolveReplyToMode?(params)             // "off" | "first" | "all" | "batched"
buildToolContext?(params)                 // Thread tool context
resolveAutoThreadId?(params)             // Auto-thread ID
resolveReplyTransport?(params)             // Reply routing
resolveFocusedBinding?(params)            // Focused binding
```

### 2.8 ChannelMessageActionAdapter (tool actions)

```typescript
describeMessageTool?(params)           // Tool description
supportsAction?(action)                 // Check support
resolveExecutionMode?(action)             // "local" | "gateway"
extractToolSend?(args)                   // Extract tool call
handleAction?(ctx)                      // Execute action
```

### 2.9 Other Adapters

| Adapter | Purpose |
|---------|---------|
| `ChannelGroupsAdapter` | Group management |
| `ChannelMentionsAdapter` | @mention detection |
| `ChannelHeartbeatAdapter` | Keepalive |
| `ChannelStatusAdapter` | Health monitoring |
| `ChannelSecretsAdapter` | Credential storage |
| `ChannelAllowlistAdapter` | Sender allowlisting |
| `ChannelDoctorAdapter` | Diagnostic support |
| `ChannelBindingsAdapter` | Conversation bindings |
| `ChannelResolverAdapter` | Target resolution |
| `ChannelApprovalCapability` | Approval requests |
| `ChannelPairingAdapter` | Device pairing |

---

## 3. Channel-to-Channel Bridging (Binding System)

### 3.1 Binding Flow

```
User on Discord → Message → Channel Binding
  → lookup ConfiguredBindingRecord
  → resolve configured source agent + channel
  → create ACP session key: agent:${agentId}:acp:binding:${channel}:${accountId}:${hash}
  → deliver to bound channel
```

### 3.2 ConfiguredBindingRecord

```typescript
{
  channel, accountId, conversationId, parentConversationId?,
  agentId, acpAgentId?, mode, cwd?, backend?, label?
}
```

### 3.3 Binding Resolution

```typescript
resolveConfiguredBindingRecord(cfg, channel, accountId, conversationId)
  → Lookup in configured-binding-registry
  → Return ConfiguredBindingRecord or null
```

---

## 4. Session Conversation Resolution (3-Layer Fallback)

```
1. Plugin resolution
   → channel.messaging?.resolveSessionConversation()

2. Bundled plugin fallback
   → loadBundledChannelSessionConversationApi(channelId)
   → api?.resolveSessionConversation()

3. Generic fallback
   → buildGenericConversationResolution()
```

---

## 5. Message Send Flow

```
Agent produces Reply
  → ChannelRuntime.reply.dispatchReplyWithBufferedBlockDispatcher()
  → ChannelOutboundAdapter.sendPayload() or sendText()
  → Per-channel implementation (grammY API, Discord webhook, etc.)
  → Delivered to messaging platform
```

---

## 6. Setup Wizard (625 lines)

`src/channels/plugins/setup-wizard.ts` — Interactive channel setup:

```typescript
// Steps:
1. detectExistingConfig()     // Check if already configured
2. validateInput()             // User input validation
3. buildPatch()               // Generate config patch
4. applyToConfig()             // Write to config
5. verifyConnection()          // Test the setup
```

---

## 7. Channel IDs and Normalization

```typescript
// src/channels/ids.ts
normalizeChannelId(id)           // Lowercase, trim
resolveChannelIdAlias(id)        // Resolve aliases
CHAT_CHANNEL_ORDER               // Display ordering
```

---

## 8. Status Reactions (415 lines)

Emoji-based status display during agent thinking:

```typescript
createStatusReactionController({
  enabled: boolean,
  adapter: StatusReactionAdapter,
  initialEmoji: string,
  emojis?: StatusReactionEmojis,
  timing?: StatusReactionTiming,
})

// States: queued, thinking, tool, coding, web, done, error, stall
```

---

## 9. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/channels/plugins/types.core.ts` | 719 | Core channel types |
| `src/channels/plugins/types.adapters.ts` | 855 | All adapter interfaces |
| `src/channels/plugins/types.plugin.ts` | 96 | ChannelPlugin type |
| `src/channels/plugins/bundled.ts` | 670 | Bundled channel loading |
| `src/channels/plugins/catalog.ts` | 373 | Channel catalog building |
| `src/channels/plugins/setup-wizard.ts` | 625 | Setup wizard |
| `src/channels/session-conversation.ts` | 334 | Session conversation resolution |
| `src/channels/mention-gating.ts` | 232 | Inbound mention gating |
| `src/channels/channel-config.ts` | 182 | Config matching |
| `src/channels/status-reactions.ts` | 415 | Status emoji reactions |
| `src/channels/plugins/configured-binding-registry.ts` | 113 | Binding registry |