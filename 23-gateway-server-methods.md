# Gateway Server Methods — Deep Dive

> **Gateway server methods** are the heart of the OpenClaw WebSocket protocol. Every method is a typed handler registered in `src/gateway/server-methods/` that receives a `GatewayRequestHandlerOptions` containing `req`, `params`, `client`, `respond`, and `context`. Handlers respond via `respond(ok, payload, error, meta)`.

## Protocol Layer

All handlers live under `src/gateway/server-methods/` and export a `GatewayRequestHandlers` object (a `Record<string, GatewayRequestHandler>`). The `GatewayRequestHandler` type is:

```typescript
type GatewayRequestHandler = (opts: GatewayRequestHandlerOptions) => Promise<void> | void;

type GatewayRequestHandlerOptions = {
  req: RequestFrame;
  params: Record<string, unknown>;
  client: GatewayClient | null;
  isWebchatConnect: (params: ConnectParams | null | undefined) => boolean;
  respond: RespondFn;  // respond(ok, payload?, error?, meta?)
  context: GatewayRequestContext;  // 60+ fields: deps, cron, broadcast, nodeRegistry, dedupe maps, etc.
};
```

Validation is via AJV schema validators generated from protocol schemas in `src/gateway/protocol/index.ts`. The shared validation helper is `assertValidParams<T>(params, validator)` in `validation.ts`.

---

## 1. Connection — `connect.ts`, `disconnect.ts`

### `connect` (rejected)

**File:** `connect.ts`

```typescript
connect: ({ respond }) => { respond(false, undefined, errorShape(ErrorCodes.INVALID_REQUEST, "connect is only valid as the first request")); }
```

`connect` is only valid as the very first WebSocket frame. Any subsequent `connect` call is rejected with `INVALID_REQUEST`. The actual connection handshake happens at the transport layer before handlers are invoked.

### `disconnect` (implicit)

There is no explicit `disconnect` handler. Client disconnection is handled by the WebSocket server runtime. The Gateway tracks connected clients via `connId`.

---

## 2. Health & Status — `health.ts`, `logs.ts`

### `health`

**File:** `health.ts`

```typescript
health: ({ context, respond }) => {
  const cached = context.getHealthCache();
  if (cached) { respond(true, cached); return; }
  context.refreshHealthSnapshot({ probe: true }).then(s => respond(true, s)).catch(err => respond(false, undefined, errorShape(...)));
}
```

Returns a cached health snapshot if available, otherwise probes all subsystems asynchronously. The cache is populated by `refreshHealthSnapshot`.

### `status` (alias of `health`)

Same implementation as `health` — returns the same cached or freshly-probed health summary.

### `logs.tail`

**File:** `logs.ts`

```typescript
logs.tail: async ({ params, context, respond }) => {
  assertValidParams(params, validateLogsTailParams);
  const { path: logPath, lines = 100 } = params;
  // streams last N lines from the log file via execFile('tail')
  respond(true, { lines: [...], nextCursor?: string });
}
```

Tails the last `N` lines from a log file. The `path` must be under an allowed log directory.

---

## 3. Sessions — `sessions.ts`

Sessions are the core abstraction for ongoing conversations. All session state is stored in JSONL transcript files via `SessionManager` from `@mariozechner/pi-coding-agent`.

### `sessions.list`

```typescript
sessions.list: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateSessionsListParams);
  // Returns all sessions visible to the authenticated client
  // Filters by agentId, limit, cursor pagination
  respond(true, { sessions: SessionsPreviewEntry[], nextCursor?: string });
}
```

Loads the combined session store, filters by ownership/auth scope, returns paginated preview entries (session key, agentId, lastMessageAt, model, messageCount).

### `sessions.get`

```typescript
sessions.get: async ({ params, client, context, respond }) => {
  // Loads full session entry by sessionKey
  respond(true, { session: SessionEntry });
}
```

### `sessions.create`

```typescript
sessions.create: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateSessionsCreateParams);
  // agentId, model?, title?
  // Resolves default agent if not specified
  respond(true, { sessionKey, session: SessionEntry });
}
```

### `sessions.send`

One of the most complex handlers. Sends a user message into a session, triggers the reply pipeline.

```typescript
sessions.send: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateSessionsSendParams);
  // 1. Resolve sessionKey, agentId
  // 2. Check deleted-agent guard
  // 3. Dispatch inbound message via dispatchInboundMessage()
  // 4. Agent run deduplication check
  // 5. Wait for Pi run completion
  // 6. Update session store with last message
  respond(true, { messageId, sentAt });
}
```

Key implementation details:
- Uses `resolveSendPolicy()` to determine delivery (relay vs embedded Pi)
- For embedded Pi runs: calls `dispatchInboundMessage()` which triggers `ReplyDispatcher`
- For relay: forwards to external relay with delivery guarantees
- Abort controller registered in `context.chatAbortControllers` for the session+run
- Transcript entries written via `SessionManager.appendMessage()` — never raw JSONL (critical: `parentId` chain must be preserved)

### `sessions.steer`

Injects a steering directive (system prompt modification, context reset, etc.) into an active or recent session.

### `sessions.preview`

```typescript
sessions.preview: async ({ params, respond }) => {
  assertValidParams(params, validateSessionsPreviewParams);
  // Returns a lightweight preview: last N messages, message count, title
  respond(true, { preview: SessionsPreviewEntry });
}
```

### `sessions.resolve`

Maps partial session keys or aliases to full session keys using the session store.

### `sessions.subscribe` / `sessions.unsubscribe`

```typescript
sessions.subscribe: ({ params, client, context, respond }) => {
  assertValidParams(params, validateSessionsSubscribeParams);
  context.subscribeSessionEvents(connId);  // connId from client
  respond(true, { subscribed: true });
}
```

Subscribes the WebSocket connection to session lifecycle events (created, deleted, updated) for the given session key.

### `sessions.messages.subscribe` / `sessions.messages.unsubscribe`

Subscribes to transcript message events (new messages appended) for a given session. Uses `context.subscribeSessionMessageEvents(connId, sessionKey)`.

### `sessions.compact`

Triggers session compaction — rewriting the transcript file to remove intermediate messages and keep only the significant entries. Uses `compactEmbeddedPiSession()`.

### `sessions.compaction.list`

```typescript
sessions.compaction.list: async ({ params, respond }) => {
  assertValidParams(params, validateSessionsCompactionListParams);
  const checkpoints = listSessionCompactionCheckpoints(sessionKey);
  respond(true, { checkpoints });
}
```

Lists all compaction checkpoints for a session.

### `sessions.compaction.get`

Gets a specific checkpoint's contents.

### `sessions.compaction.branch`

Creates a new session branch from a compaction checkpoint.

### `sessions.compaction.restore`

Restores session state from a compaction checkpoint.

### `sessions.patch`

Patches mutable session fields (title, pinned state, tags) in the session store. Triggers internal hooks (`hasInternalHookListeners`, `triggerInternalHook`).

### `sessions.reset`

Resets a session to a clean state, preserving the session key but clearing transcript. Uses `performGatewaySessionReset()`.

### `sessions.abort`

```typescript
sessions.abort: async ({ params, context, respond }) => {
  assertValidParams(params, validateSessionsAbortParams);
  // Aborts any active Pi run for the session
  abortEmbeddedPiRun(sessionKey);
  respond(true, { aborted: true });
}
```

### `sessions.delete`

Marks a session as deleted in the store (soft delete). Cannot be undone.

### `sessions.usage`

Retrieves usage statistics for a session (token counts, run counts, cost estimates). Delegates to `usage.ts` logic.

---

## 4. Chat — `chat.ts`

Chat handles real-time message sending and history for active conversational sessions. Unlike `sessions.send` (which is a fire-and-forget send), `chat` operates on already-established sessions with more fine-grained control.

### `chat.send`

The primary inbound message handler — the most complex handler in the gateway.

```typescript
chat.send: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateChatSendParams);
  // sessionKey, message (text + attachments), replyConfig?
  // 1. Load session entry
  // 2. Check deleted-agent guard
  // 3. Check chat.abort authorization (client must own the run)
  // 4. Parse message + attachments via parseMessageWithAttachments()
  // 5. Resolve send policy (embedded vs relay)
  // 6. If embedded: dispatchInboundMessage -> ReplyDispatcher -> Pi runner
  // 7. Register abort controller
  // 8. Return sent message ID + timestamp
}
```

Key implementation details:
- **Authorization for abort**: `chat.abort` only works if the requesting client matches the original sender's auth context
- **Attachment handling**: Images/media are offloaded to storage, references kept in message payload
- **Canvas blocks**: If canvas content detected in reply, augmented into the transcript
- **Transcript sanitization**: All messages pass through `stripEnvelopeFromMessage` before storage
- **Deduplication**: Uses `setGatewayDedupeEntry()` to prevent duplicate processing

### `chat.history`

```typescript
chat.history: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateChatHistoryParams);
  // sessionKey, before?, limit?, after?
  // Reads transcript file, augments with canvas blocks
  // Sanitizes messages (strips internal envelope fields)
  respond(true, { messages: ChatMessage[], hasMore: boolean });
}
```

Reads from the Pi transcript file and augments with canvas block data. Messages are sanitized to remove internal fields.

### `chat.abort`

```typescript
chat.abort: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateChatAbortParams);
  // sessionKey, runId
  // Checks abort authorization (ownership verification)
  // Calls abortChatRunById(sessionKey, runId)
  respond(true, { aborted: true });
}
```

Authorization is checked by comparing `client.connect.authId` vs the stored run's auth context. Only the original sender or admin can abort.

### `chat.inject`

Injects an assistant message directly into the session transcript (bypassing the normal reply pipeline). Used for server-side interventions.

---

## 5. Agent — `agent.ts`, `agents.ts`, `agent-job.ts`

### `agent` (primary)

The core agent run dispatcher.

```typescript
agent: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateAgentParams);
  // sessionKey, message, replyConfig?, spawnedBy?
  // 1. Deduplication: check context.dedupe for in-flight run
  // 2. Resolve delivery plan (embedded Pi, relay, or best-effort)
  // 3. Resolve ingress workspace override (spawned context)
  // 4. Handle attachments: normalizeRpcAttachmentsToChatAttachments()
  // 5. Inject timestamp via injectTimestamp()
  // 6. Register tool event recipient (for tool use callbacks)
  // 7. Dispatch to Pi runner or relay
  // 8. Set deduplication entry with TTL
  respond(true, { runId, deliveryInfo });
}
```

Key details:
- **Deduplication**: Uses `context.dedupe` Map keyed by `${sessionKey}:${canonicalMessageHash}`. Prevents duplicate runs from concurrent send requests.
- **Delivery planning**: `resolveAgentDeliveryPlan()` decides between embedded Pi, relay, or best-effort based on session type and config
- **Spawned runs**: `spawnedBy` field propagated from parent agent context via `resolveIngressWorkspaceOverrideForSpawnedRun()`
- **Attachment normalization**: RPC attachment format converted to chat attachment format
- **Timestamp injection**: Ensures server-side timestamp is recorded for the run

### `agent.identity.get`

Resolves the assistant identity (name, avatar, model) for the agent in a given session.

### `agent.wait`

```typescript
agent.wait: async ({ params, client, context, respond }) => {
  // sessionKey, runId, timeout?
  // Uses waitForAgentJob() + deduplication-aware caching
  // Returns terminal snapshot (last message, run status)
}
```

Waits for an agent run to complete. Uses deduplication-aware caching to avoid redundant waits.

### `agents.list`

```typescript
agents.list: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateAgentsListParams);
  const agentIds = listAgentIds();
  respond(true, { agents: AgentSummary[] });
}
```

### `agents.create`

Creates a new agent definition (workspace + config). Returns the new agent's ID and summary.

### `agents.update`

Updates mutable agent fields (name, description, instructions, model selection).

### `agents.delete`

Soft-deletes an agent. Sessions using the agent are marked as "deleted agent" and cannot receive new messages.

### `agents.files.list`

Lists files in an agent's workspace.

### `agents.files.get`

Reads a specific file from an agent's workspace.

### `agents.files.set`

Writes or updates a file in an agent's workspace.

---

## 6. Config — `config.ts`

### `config.get`

```typescript
config.get: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateConfigGetParams);
  const snapshot = readConfigFileSnapshot();
  // Returns full config (with sensitive fields redacted via redactConfigObject)
  respond(true, { config: OpenClawConfig, baseHash: string });
}
```

Returns the current config snapshot plus a `baseHash` for optimistic locking on writes.

### `config.set`

Full config replacement with base hash validation. Rejected if base hash mismatch (concurrent modification).

```typescript
config.set: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateConfigSetParams);
  // Requires baseHash match
  // Validates with plugins via validateConfigObjectWithPlugins()
  // Writes config file
  // Broadcasts config changed event
  respond(true, { config, baseHash });
}
```

### `config.patch`

Partial config update using JSON merge patch (RFC 7396). Only changed fields are written. More efficient than `set` for small changes.

### `config.apply`

Applies a config change with restart coordination via restart sentinels. If restart is required, writes a restart sentinel file and coordinates graceful restart.

### `config.schema`

Returns the full runtime config schema (all known config fields, types, defaults, validation rules).

### `config.schema.lookup`

Looks up a specific schema node by path (e.g., `config.schema.lookup(["auth", "providers", 0, "apiKey"])`).

### `config.openFile`

Opens a config file in the editor configured by the user (`$EDITOR` or similar).

---

## 7. Channels — `channels.ts`

### `channels.status`

```typescript
channels.status: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateChannelsStatusParams);
  // Probes all configured channels for connection status
  // Returns per-channel status: connected, disconnected, error, lastErrorAt
  // For each channel: checks runtime, credential validity, recent errors
  respond(true, { channels: ChannelStatus[] });
}
```

Probes all channel plugins (Slack, Discord, Teams, etc.) to get real-time connection status.

### `channels.start`

Starts a specific channel plugin (initialize runtime, connect to provider). Used for on-demand channel activation.

### `channels.logout`

Logs out of a channel (clears credentials, marks channel as logged-out in runtime).

---

## 8. Commands — `commands.ts`

### `commands.list`

```typescript
commands.list: async ({ params, respond }) => {
  assertValidParams(params, validateCommandsListParams);
  // Returns available slash commands: name, description, params
  respond(true, { commands: CommandEntry[] });
}
```

---

## 9. Cron — `cron.ts`

Cron jobs are scheduled via `CronServiceContract` in the context. Jobs persist to a store file.

### `wake`

Used by the cron scheduler to wake the gateway and dispatch due cron jobs. Called by the background scheduler loop.

### `cron.list`

```typescript
cron.list: async ({ params, respond }) => {
  assertValidParams(params, validateCronListParams);
  const jobs = cron.list();  // from CronServiceContract
  respond(true, { jobs: CronJob[] });
}
```

### `cron.status`

```typescript
cron.status: async ({ params, respond }) => {
  // Returns runtime status of the cron scheduler (enabled, next run, last run, errors)
  respond(true, { status: CronStatus });
}
```

### `cron.add`

Adds a new cron job. Params: `name`, `pattern` (cron expression), `action`, `params`, `enabled?`.

### `cron.update`

Updates an existing cron job's fields.

### `cron.remove`

Deletes a cron job by ID.

### `cron.run`

Manually triggers a cron job run immediately (bypassing the schedule).

### `cron.runs`

Lists recent cron job run history (last N executions with status, duration, output/error).

---

## 10. Devices — `devices.ts`

Device management for mobile app pairing and authentication.

### `device.pair.list`

```typescript
device.pair.list: async ({ params, respond }) => {
  assertValidParams(params, validateDevicePairListParams);
  const pairings = listDevicePairing();
  respond(true, { pairings: DevicePairingEntry[] });
}
```

### `device.pair.approve`

Approves a pending device pairing request. After approval, the device can complete the auth flow.

### `device.pair.reject`

Rejects a pending device pairing request.

### `device.pair.remove`

Removes a paired device (revokes its tokens).

### `device.token.rotate`

Rotates a device's authentication token (used during re-auth flow).

### `device.token.revoke`

Revokes a device token entirely (logout from that device).

---

## 11. Nodes — `nodes.ts`

Node pairing and remote invocation for companion apps (iOS, Android, macOS).

### `node.pair.request`

Initiates pairing between gateway and a companion node.

```typescript
node.pair.request: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateNodePairRequestParams);
  // Stores pairing request, generates pending token
  // Notifies operator via broadcast
  respond(true, { pairingId, expiresAt });
}
```

### `node.pair.list`

Lists all node pairing requests (pending + paired nodes).

### `node.pair.approve`

Approves a node pairing request. The node receives an approval notification via push (APNs) and completes the handshake.

### `node.pair.reject`

Rejects a node pairing request.

### `node.pair.verify`

Verifies the node's token during pairing handshake.

### `node.rename`

Renames a paired node (updates the catalog).

### `node.list`

```typescript
node.list: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateNodeListParams);
  const nodes = listKnownNodes();
  respond(true, { nodes: KnownNode[] });
}
```

### `node.describe`

Gets detailed information about a specific node (capabilities, last seen, connection state).

### `node.canvas.capability.refresh`

Refreshes the canvas capability token for a node (extends validity, may rotate key).

### `node.pending.pull`

Pulls pending actions queued for a node (actions dispatched when node was offline).

### `node.pending.ack`

Acknowledges receipt of pending actions. Nodes ack after processing.

### `node.invoke`

Invokes a command on a companion node, waking it via APNs push if necessary.

```typescript
node.invoke: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateNodeInvokeParams);
  // nodeId, command, paramsJSON, idempotencyKey
  // 1. Look up node in nodeRegistry
  // 2. If offline: send APNs wake (sendApnsBackgroundWake)
  // 3. Queue action (PendingNodeAction) with TTL
  // 4. If node reconnects within wait window: action forwarded
  // 5. Respond with invokeId for later result correlation
}
```

**Wake flow**: If the node is offline, `sendApnsBackgroundWake()` is called with a wake payload. The node reconnects, pulls pending actions, and executes. The gateway waits for `node.invoke.result` within the timeout window.

**Wake throttling**: Nodes are not woken more than once per 15 seconds (`NODE_WAKE_THROTTLE_MS`). Nudge (non-wake) notifications throttled to once per 10 minutes.

### `node.invoke.result`

Receives the result of a node invoke command from the node. The node calls this when it has completed the command.

### `node.event`

Allows a node to push an event to the gateway (asynchronous notification from node to gateway).

### `node.pending.drain` / `node.pending.enqueue`

Manually drain or enqueue pending actions for a node (admin/debug operations).

---

## 12. Talk (TTS) — `talk.ts`

Talk controls text-to-speech settings and playback.

### `talk.config`

Gets the current TTS configuration (provider, voice, rate, pitch).

### `talk.speak`

Triggers TTS playback of a text string on the client device. The text is synthesized via the configured TTS provider and played on the client.

### `talk.mode`

Sets or gets the current talk mode (enabled/disabled/automatic).

---

## 13. Skills — `skills.ts`

### `skills.status`

Gets the current status of all installed skills (enabled/disabled, version, last updated).

### `skills.bins`

Lists all skill bins (grouped skill collections).

### `skills.search`

Searches available skills (from clawhub or local bins).

### `skills.detail`

Gets detailed information about a specific skill.

### `skills.install`

Installs a skill from clawhub or a URL.

### `skills.update`

Updates an installed skill to a newer version.

---

## 14. Tools — `tools-effective.ts`, `tools-catalog.ts`

### `tools.effective`

```typescript
tools.effective: async ({ params, client, context, respond }) => {
  // sessionKey, agentId?
  // Returns the effective tool list for the session/agent
  // Resolves from: agent config + global config + runtime overrides
  respond(true, { tools: ToolDefinition[] });
}
```

### `tools.catalog`

Returns the full catalog of available tools (all tool plugins registered in the system).

---

## 15. Models — `models.ts`, `models-auth-status.ts`

### `models.list`

```typescript
models.list: async ({ params, respond }) => {
  assertValidParams(params, validateModelsListParams);
  const catalog = context.loadGatewayModelCatalog();
  respond(true, { models: ModelCatalogEntry[] });
}
```

### `models.authStatus`

```typescript
models.authStatus: async ({ params, client, context, respond }) => {
  // Checks OAuth status for all configured model providers
  // Aggregates multiple provider auth states
  // Caches results with TTL
  // Enriches with usage data if available
  respond(true, { providers: ProviderAuthStatus[], totalUsage?: UsageSummary });
}
```

Key details:
- Caches per-provider auth status to avoid repeated OAuth introspection
- Aggregates status across all providers (Anthropic, OpenAI, Google, etc.)
- Enriches with usage/cost data when available

---

## 16. Usage — `usage.ts`

### `usage.status`

Current overall usage summary (requests, tokens, cost across all models).

### `usage.cost`

Cost breakdown by model and time period.

### `sessions.usage`

Token and request usage for a specific session.

### `sessions.usage.timeseries`

Time-series usage data (hourly/daily cost and request counts).

### `sessions.usage.logs`

Detailed per-request usage logs for a session.

---

## 17. Exec Approvals — `exec-approvals.ts`, `exec-approval.ts`

### `exec.approvals.get`

Gets the current exec approval settings (approval required, approved commands, etc.).

### `exec.approvals.set`

Updates exec approval settings.

### `exec.approvals.node.get/set`

Per-node exec approval overrides.

### `exec.approval.get`

Gets the status of a specific pending exec approval request.

### `exec.approval.list`

Lists all pending exec approval requests.

### `exec.approval.request`

```typescript
exec.approval.request: async ({ params, client, context, respond }) => {
  assertValidParams(params, validateExecApprovalRequestParams);
  // command, args, sessionKey, origin
  // Creates approval request, assigns requestId
  // Notifies approval clients via broadcast
  // Returns requestId for polling
  respond(true, { requestId, expiresAt });
}
```

### `exec.approval.waitDecision`

Blocks until a decision is reached (approved/denied/timeout). Used by operators waiting for approval.

### `exec.approval.resolve`

Resolves a pending approval request (approve or deny). When approved, the command is executed.

Key implementation details:
- Two-phase approval: request is created, operator polls `waitDecision`, resolver approves
- iOS push delivery: approval requests can be pushed to mobile devices via APNs
- Forwarder pattern: for commands that need approval, a `ExecApprovalForwarder` intercepts and holds the command

---

## 18. Plugin Approvals — `plugin-approval.ts`

### `plugin.approval.list`

Lists pending plugin approval requests.

### `plugin.approval.request`

Requests approval to load/use a plugin.

### `plugin.approval.waitDecision`

Blocks until plugin approval is granted or denied.

### `plugin.approval.resolve`

Resolves a pending plugin approval request.

---

## 19. Cron Approvals — (handled in `cron.ts`)

Cron jobs that involve exec approval flow through the same approval manager.

---

## 20. System — `system.ts`

### `gateway.identity.get`

```typescript
gateway.identity.get: ({ client, respond }) => {
  // Returns the gateway's identity: nodeId, gatewayVersion, protocol version
  respond(true, { nodeId, version, protocolVersion });
}
```

### `last-heartbeat`

Returns the last heartbeat timestamp received from this client.

### `set-heartbeats`

Enables or disables heartbeat monitoring for this connection.

### `system-presence`

Sets the operator's presence status (online/away/busy).

### `system-event`

Allows operator to emit a named system event for debugging/tracing purposes.

---

## 21. Update — `update.ts`

### `update.run`

Triggers a gateway self-update. Downloads and applies updates. Coordinate with restart sentinel system.

---

## 22. TTS — `tts.ts`

### `tts.status`

Gets current TTS provider status (enabled, configured provider, voice settings).

### `tts.enable` / `tts.disable`

Enables or disables TTS globally.

### `tts.convert`

Converts text to speech and returns audio data (or saves to a file for playback).

### `tts.setProvider`

Sets the active TTS provider.

### `tts.providers`

Lists available TTS providers and their configuration status.

---

## 23. Doctor (Memory System) — `doctor.ts`

Memory system diagnostic and repair tools.

### `doctor.memory.status`

Overall health status of the memory system (vector store, memory nodes, embedding status).

### `doctor.memory.dreamDiary`

Gets the dream diary entries for memory dreaming.

### `doctor.memory.backfillDreamDiary`

Triggers a backfill of dream diary entries from recent memory activity.

### `doctor.memory.resetDreamDiary`

Clears the dream diary.

### `doctor.memory.resetGroundedShortTerm`

Resets the grounded short-term memory store.

### `doctor.memory.repairDreamingArtifacts`

Repairs corrupted or inconsistent dreaming artifacts.

### `doctor.memory.dedupeDreamDiary`

Deduplicates dream diary entries.

---

## 24. Push — `push.ts`

### `push.test`

Sends a test push notification to the operator's registered devices. Used to verify push configuration.

---

## 25. Secrets — `secrets.ts`

### `secrets.reload`

Reloads secrets from the secrets store (re-reads from Vault, environment, or secrets file).

### `secrets.resolve`

Resolves a secret reference to its current value (for runtime use). Used by the config system to inject secrets into plugin configs.

---

## 26. Web Login — `web.ts`

### `web.login.start`

Initiates a web-based login flow. Returns an OAuth URL or login page URL for the operator to complete.

### `web.login.wait`

Blocks until the web login flow completes (operator completes auth in browser). Returns the authenticated session token.

---

## 27. Wizard — `wizard.ts`

### `wizard.start`

Starts the onboarding wizard for a new operator.

### `wizard.next`

Advances the wizard to the next step. Handles step validation and data collection.

### `wizard.cancel`

Cancels the onboarding wizard.

### `wizard.status`

Returns the current wizard state (current step, collected data, progress).

---

## 28. Voice Wake — `voicewake.ts`

### `voicewake.get`

```typescript
voicewake.get: async ({ respond }) => {
  const cfg = await loadVoiceWakeConfig();
  respond(true, { triggers: cfg.triggers });
}
```

Returns the current voice wake word triggers.

### `voicewake.set`

```typescript
voicewake.set: async ({ params, context, respond }) => {
  if (!Array.isArray(params.triggers)) { respond(false, ...); return; }
  const triggers = normalizeVoiceWakeTriggers(params.triggers);
  const cfg = await setVoiceWakeTriggers(triggers);
  context.broadcastVoiceWakeChanged(cfg.triggers);  // notify all connected clients
  respond(true, { triggers: cfg.triggers });
}
```

Updates voice wake triggers and broadcasts the change to all connected clients.

---

## Helper Files

### `agent-job.ts`

`waitForAgentJob()` — Core function for waiting on agent run completion. Uses `context.chatAbortControllers` and `context.dedupe` for tracking. Returns when the run reaches a terminal state or times out.

### `agent-wait-dedupe.ts`

Deduplication logic for `agent.wait`. Prevents multiple concurrent waits for the same run from redundant work.

### `agent-timestamp.ts`

`injectTimestamp()` and `timestampOptsFromConfig()` — Injects server-side timestamps into agent runs for latency tracking.

### `attachment-normalize.ts`

`normalizeRpcAttachmentsToChatAttachments()` — Converts RPC attachment format (from network protocol) to internal chat attachment format.

### `base-hash.ts`

`resolveBaseHashParam()` — Extracts and validates the base hash parameter used for optimistic locking in config writes.

### `restart-request.ts`

`parseRestartRequestParams()` — Shared utility for parsing restart request params (used by `config.apply`).

### `nodes.helpers.ts`

Utility functions for node handlers: `respondInvalidParams`, `respondUnavailableOnNodeInvokeError`, `respondUnavailableOnThrow`, `safeParseJson`.

### `nodes.handlers.invoke-result.ts`

`handleNodeInvokeResult()` — Handles the result of a node invoke, correlating it with the pending action and notifying waiters.

### `chat-transcript-inject.ts`

`appendInjectedAssistantMessageToTranscript()` — Appends an injected assistant message to the session transcript.

### `chat-webchat-media.ts`

`buildWebchatAudioContentBlocksFromReplyPayloads()` — Builds webchat-compatible media blocks from reply payloads (for TTS/audio responses).

### `shared-types.ts`

Re-exports from `types.ts` — provides `GatewayRequestHandlers` and related types to handler files.

### `validation.ts`

`assertValidParams<T>()` — The central AJV validation helper used by all handlers.

### `sessions.runtime.ts`

Lazy-loaded module for session runtime operations. Loaded on demand to avoid bundling session runtime in server tests.

---

## Summary: Method Count by Category

| Category | Methods |
|---|---|
| Connection | 1 (connect rejected) |
| Health/Status | 2 (health, logs.tail) |
| Sessions | ~18 (list, get, create, send, steer, preview, resolve, subscribe/unsubscribe, messages.subscribe/unsubscribe, compact, compaction.*, patch, reset, abort, delete, usage) |
| Chat | 4 (send, history, abort, inject) |
| Agent | 7 (agent, identity.get, wait, agents.list/create/update/delete, agents.files.list/get/set) |
| Config | 7 (get, set, patch, apply, schema, schema.lookup, openFile) |
| Channels | 3 (status, start, logout) |
| Commands | 1 (list) |
| Cron | 7 (wake, list, status, add, update, remove, run, runs) |
| Devices | 6 (pair.list/approve/reject/remove, token.rotate/revoke) |
| Nodes | ~12 (pair.request/list/approve/reject/verify, rename, list, describe, canvas.capability.refresh, pending.pull/ack, invoke, invoke.result, event, pending.drain/enqueue) |
| Talk | 3 (config, speak, mode) |
| Skills | 6 (status, bins, search, detail, install, update) |
| Tools | 2 (effective, catalog) |
| Models | 2 (list, authStatus) |
| Usage | 4 (status, cost, sessions.usage, sessions.usage.timeseries, sessions.usage.logs) |
| Exec Approvals | 7 (approvals.get/set, approvals.node.get/set, approval.get/list/request/waitDecision/resolve) |
| Plugin Approvals | 4 (list, request, waitDecision, resolve) |
| System | 5 (identity.get, last-heartbeat, set-heartbeats, system-presence, system-event) |
| Update | 1 (run) |
| TTS | 5 (status, enable, disable, convert, setProvider, providers) |
| Doctor | 7 (memory.status, dreamDiary, backfillDreamDiary, resetDreamDiary, resetGroundedShortTerm, repairDreamingArtifacts, dedupeDreamDiary) |
| Push | 1 (test) |
| Secrets | 2 (reload, resolve) |
| Web | 2 (login.start, login.wait) |
| Wizard | 4 (start, next, cancel, status) |
| Voice Wake | 2 (get, set) |
| **Total** | **~110+** |

---

## Key Architectural Notes

1. **All handlers are data-first and acyclic**: Protocol exports do not route back through heavier gateway runtime helpers at import time. This keeps the contract surface cheap and order-independent.

2. **Session transcripts are parentId chains/DAGs**: Always write via `SessionManager.appendMessage()` (or `appendInjectedAssistantMessageToTranscript`). Never append via raw JSONL — missing `parentId` can sever the leaf path and break compaction/history.

3. **Deduplication is first-class**: Agent runs, chat sends, and agent waits all use the `context.dedupe` map keyed by `${sessionKey}:${canonicalHash}`. This prevents duplicate work from concurrent requests.

4. **Abort is scoped and authorized**: Chat abort checks that the requesting client matches the original sender. Sessions abort checks the abort controller entry.

5. **Config writes use optimistic locking**: The `baseHash` parameter is validated against the current config snapshot hash before any write proceeds. Mismatch → `INVALID_REQUEST`.

6. **Node invoke uses a wake/throttle/queue pattern**: Offline nodes are woken via APNs, actions are queued with TTL and per-node limits, and nodes poll for pending actions on reconnect.

7. **Broadcast is first-class**: All state changes (config changed, voice wake triggers changed, session events) are broadcast to all relevant connected clients via `context.broadcast` and `context.broadcastToConnIds`.

8. **Handler tests should reuse suite-level infrastructure**: Connect/auth handshake should not be restarted per test case. Use `context` fixtures and explicit state reset.
