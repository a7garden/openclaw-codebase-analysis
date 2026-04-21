# ACP (Agent Communication Protocol) Deep Dive

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

ACP bridges OpenClaw's internal agent runtime with **external harness clients** — IDE plugins (VS Code/IDX), CLI tools (Codex CLI), and other external agents. It translates the ACP protocol (from `@agentclientprotocol/sdk`) to OpenClaw's Gateway WebSocket protocol and back.

```
ACP Client (VS Code/IDX/Codex CLI)
  → stdin/stdout ndJson streams
  → AcpGatewayAgent (translator.ts)
  → Gateway WebSocket API
  → OpenClaw Core
```

---

## 2. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/acp/translator.ts` | 1419 | **Central bridge** — AcpGatewayAgent class |
| `src/acp/control-plane/manager.core.ts` | 2221 | **Core orchestrator** — AcpSessionManager |
| `src/acp/session.ts` | 191 | Session store (in-memory, 5000 max, 24hr TTL) |
| `src/acp/runtime/types.ts` | 152 | AcpRuntime interface contract |
| `src/acp/event-mapper.ts` | 409 | Content block transformation |
| `src/acp/approval-classifier.ts` | 227 | Tool permission classification |
| `src/acp/server.ts` | 254 | ACP server entry point |
| `src/acp/client.ts` | 464 | ACP client for spawning server |

---

## 3. Session Store

### 3.1 In-Memory Store

```typescript
createInMemorySessionStore({
  maxSessions: 5000,
  idleTtlMs: 24 * 60 * 60 * 1000, // 24 hours
})
```

### 3.2 Session Eviction

When at capacity:
1. Sort sessions by `lastTouchedAt` ascending
2. Evict oldest idle session (no activeRunId, no abortController)
3. Create new session

### 3.3 Session Structure

```typescript
interface AcpSession {
  sessionId: string        // Local UUID
  sessionKey: string       // Gateway session key: agent:${agentId}:acp:${uuid}
  cwd: string
  createdAt: Date
  lastTouchedAt: Date
  abortController: AbortController
  activeRunId: string | null
}
```

---

## 4. Translator (AcpGatewayAgent)

### 4.1 Core Responsibilities

```typescript
class AcpGatewayAgent implements Agent {
  // Session lifecycle
  newSession(params)     // → gateway sessions.create
  loadSession(params)    // → gateway sessions.get
  unstable_listSessions()

  // Prompt dispatch
  prompt(params)         // → gateway chat.send

  // Session controls
  setSessionMode(params)  // → gateway sessions.patch
  setSessionConfigOption(params) // → gateway sessions.patch

  // Events
  handleGatewayEvent(event)  // Process gateway chat/agent events
}
```

### 4.2 Pending Prompt Tracking

```typescript
PendingPrompt = {
  sessionId, sessionKey, idempotencyKey,
  sentTextLength, sentThoughtLength,
  toolCalls: Map<toolCallId, PendingToolCall>,
  disconnectContext?,
  resolve, reject
}
```

### 4.3 Disconnect Handling

1. Gateway disconnects → `handleGatewayDisconnect()`
2. `armDisconnectTimer(5s)` grace period
3. `reconcilePendingPrompts()` → calls `agent.wait` for each pending
4. If timeout → reject pending with "Gateway disconnected"

### 4.4 Provenance Modes

| Mode | Behavior |
|------|----------|
| `off` | No provenance metadata |
| `meta` | Adds `systemInputProvenance: { kind: "external_user", originSessionId, sourceChannel: "acp" }` |
| `meta+receipt` | Above + hostname, cwd, session IDs |

---

## 5. Control Plane (AcpSessionManager)

### 5.1 Session Actor Queue

Per-session serialization via `KeyedAsyncQueue`:

```typescript
// All operations on same sessionKey are serialized
withSessionActor(sessionKey, async (actor) => {
  // Only one operation runs at a time per session
})
```

### 5.2 Runtime Cache

```typescript
RuntimeCache: Map<actorKey, CachedRuntimeState>
// Cached handles include: runtime, backend, agent, mode, cwd
// Idle eviction when TTL exceeded
```

### 5.3 Turn Execution Flow

```
runTurn(input)
  → canonicalizeSessionKey()
  → evictIdleRuntimeHandles()
  → withSessionActor(sessionKey)
    → resolveSession() → requireReadySessionMeta()
    → ensureRuntimeHandle()
    → applyRuntimeControls()
    → setActiveRun()
    → runtime.runTurn() → AsyncIterable events
    → awaitTurnWithTimeout(timeoutMs + 1s grace)
    → recordTurnCompletion()
    → reconcileRuntimeSessionIdentifiers()
```

### 5.4 Turn Timeout

- Default from `cfg.acp.runtime.timeoutSeconds`
- 1s grace period on top
- On timeout: `eventGate.open = false` stops event processing

---

## 6. Runtime Interface (AcpRuntime Contract)

```typescript
interface AcpRuntime {
  ensureSession(input: AcpRuntimeEnsureInput): Promise<AcpRuntimeHandle>
  runTurn(input: AcpRuntimeTurnInput): AsyncIterable<AcpRuntimeEvent>
  getCapabilities?(input): Promise<AcpRuntimeCapabilities>
  getStatus?(input): Promise<AcpRuntimeStatus>
  setMode?(input): Promise<void>
  setConfigOption?(input): Promise<void>
  doctor?(): Promise<AcpRuntimeDoctorReport>
  prepareFreshSession?(input): Promise<void>
  cancel(input): Promise<void>
  close(input): Promise<void>
}

type AcpRuntimeEvent =
  | { type: "text_delta", delta: string }
  | { type: "status", status: SessionStatus }
  | { type: "tool_call", name: string, toolCallId: string, args: unknown }
  | { type: "done", reason: "stop"|"complete"|"error"|"cancelled"|"timeout" }
  | { type: "error", error: string }
```

---

## 7. Approval Classification

```typescript
classifyAcpToolApproval(toolName, toolArgs, cwd)
// → "readonly_scoped"   — read tool scoped to cwd
// → "readonly_search"   — safe search tools
// → "mutating"          — general mutating tools
// → "exec_capable"      — exec/spawn/shell/bash
// → "control_plane"      — sessions_spawn/sessions_send
// → "interactive"        — interactive tools
// → "other"             — unknown
```

Auto-approval policy based on classification.

---

## 8. Subagent Spawning via ACP

```typescript
spawnAcpSubagent(ctx)
  → sessions.patch(sessionKey)     // Create gateway session
  → manager.initializeSession()     // Create ACP session
  → runtime.ensureSession()         // Get/create runtime handle
  → createRunningTaskRun()           // Track as background task
```

Binding session key format: `agent:${agentId}:acp:binding:${channel}:${accountId}:${hash}`

---

## 9. ACP Error Codes

```typescript
ACP_BACKEND_MISSING
ACP_BACKEND_UNAVAILABLE
ACP_BACKEND_UNSUPPORTED_CONTROL
ACP_DISPATCH_DISABLED
ACP_INVALID_RUNTIME_OPTION
ACP_SESSION_INIT_FAILED
ACP_TURN_FAILED
```