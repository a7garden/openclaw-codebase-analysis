# ACP (Agent Communication Protocol) Architecture

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

ACP is the **inter-agent communication protocol** that handles:
- Session management between agents and their subagents
- Message routing and queuing
- Control-plane operations
- Runtime adapters between ACP and actual agent runtimes

ACP lives in `src/acp/` and is used extensively by the agent system for subagent management.

---

## 2. Session Store

### 2.1 AcpSessionStore Interface

**File:** `src/acp/session.ts:4-13`

```typescript
export type AcpSessionStore = {
  createSession(config: SessionConfig): Promise<Session>
  hasSession(id: string): boolean
  getSession(id: string): Session | null
  getSessionByRunId(runId: string): Session | null
  setActiveRun(sessionId: string, runId: string): void
  clearActiveRun(sessionId: string): void
  cancelActiveRun(sessionId: string): void
}
```

### 2.2 Session Store Implementation

**File:** `src/acp/session.ts:150+`

In-memory store with:
- **Max sessions:** 5000
- **Idle TTL:** 24 hours
- **Session key derivation** from agent ID, channel ID, thread ID
- **Active run tracking** for ongoing inference
- **Abort controller support** for cancellation

### 2.3 Session Key Derivation

```typescript
// Derives a unique session key from components
deriveSessionKey(agentId, channelId, threadId): string
```

---

## 3. Session Actor Queue

**Directory:** `src/acp/`

The session actor queue handles per-session message queuing and processing:

```
Session receives message
  → Queue actor for processing
  → Agent processes message
  → Response routed back to caller
  → Queue updates state
```

---

## 4. Translator

**File:** `src/acp/translator.ts` (100+ lines)

The ACP translator handles message translation between agents:

```typescript
// Translate between different agent message formats
translateMessage(message: AgentMessage, targetFormat: MessageFormat): AgentMessage
```

Used when subagents from different harnesses need to communicate.

---

## 5. Control Plane

**Directory:** `src/acp/control-plane/`

ACP control-plane handles:
- Session lifecycle (creation, teardown, idle cleanup)
- Run management (starting, aborting, completing runs)
- Capability negotiation between agents

---

## 6. Runtime Adapters

**Directory:** `src/acp/runtime-adapters/`

Runtime adapters bridge ACP's abstract session model to actual agent runtimes:

```typescript
// ACP runtime → actual agent runtime
class AgentRuntimeAdapter implements AcpRuntime {
  constructor(private runner: EmbeddedRunner) {}

  async createSession(config: SessionConfig): Promise<Session>
  async sendMessage(sessionId: string, message: Message): Promise<Response>
  async abortRun(sessionId: string): Promise<void>
}
```

---

## 7. ACP in Agent Spawning

**File:** `src/agents/acp-spawn.ts` (200+ lines)

Subagent spawning uses ACP:

```typescript
async function spawnAcpSubagent(ctx: SpawnContext): Promise<SubagentHandle> {
  // 1. Create ACP session for subagent
  const session = await acpSessionStore.createSession({ ... })

  // 2. Start agent runtime in session
  await agentRunner.start(session.id, { ... })

  // 3. Return handle with send/abort methods
  return {
    session,
    send: (msg) => acpSessionStore.sendMessage(session.id, msg),
    abort: () => acpSessionStore.cancelActiveRun(session.id)
  }
}
```

---

## 8. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/acp/session.ts` | 150+ | AcpSessionStore implementation |
| `src/acp/translator.ts` | 100+ | Message translation |
| `src/acp/control-plane/` | ~200 | ACP control plane logic |
| `src/acp/runtime-adapters/` | ~200 | Runtime adapters |
| `src/agents/acp-spawn.ts` | 200+ | Subagent spawning via ACP |