# Infrastructure, Hooks & Tasks Deep Dive

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Infrastructure Layer (`src/infra/`)

### 1.1 Error Handling (`src/infra/errors.ts`)

```typescript
formatErrorMessage(error)   // Extracts nested .cause chains
detectErrorKind(error)      // → "refusal" | "timeout" | "rate_limit" | "context_length"
hasErrnoCode(error)          // Node.js errno check
isErrno(error)              // Node.js errno classification
```

Uses `redactSensitiveText()` for security-safe error messages.

### 1.2 Retry Logic (`src/infra/retry.ts`)

```typescript
retryAsync<T>({
  attempts: number,
  delayMs: number,
  backoff?: BackoffPolicy,
  shouldRetry?: (error, attempt) => boolean,
  retryAfterMs?: (error) => number,
  onRetry?: (error, attempt) => void,
})
```

Backoff with cryptographic jitter via `generateSecureFraction()`.

### 1.3 Deduplication Cache (`src/infra/dedupe.ts`)

```typescript
createDedupeCache({ ttlMs, maxSize })
// Global singleton via resolveGlobalDedupeCache()
```

### 1.4 Heartbeat Runner (`src/infra/heartbeat-runner.ts`) — 1500+ lines

```typescript
startHeartbeatRunner()      // Periodic agent heartbeat scheduler
runHeartbeatOnce()          // Single heartbeat execution
// Handles: active hours, session isolation, task batching from HEARTBEAT.md
// Delivery: resolves channel targets, sends outbound payloads
// Deduplication: skips duplicate payloads within 24h window
```

### 1.5 Agent Events (`src/infra/agent-events.ts`)

```typescript
emitAgentEvent({
  runId, seq, stream, ts, data, sessionKey
})
// Streams: lifecycle, tool, assistant, error, item, plan, approval, command_output, patch, compaction, thinking
```

### 1.6 System Events (`src/infra/system-events.ts`)

Lightweight in-memory queue for ephemeral system events prefixed to next prompt:
```typescript
enqueueSystemEvent()
peekSystemEventEntries()
consumeSystemEventEntries()
```

### 1.7 Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/infra/errors.ts` | ~100 | Error formatting, classification |
| `src/infra/retry.ts` | ~150 | Async retry with backoff |
| `src/infra/dedupe.ts` | ~80 | TTL-based deduplication cache |
| `src/infra/json-file.ts` | ~100 | Atomic JSON read/write |
| `src/infra/secret-file.ts` | ~80 | Secure secret handling, 0o600 permissions |
| `src/infra/fs-safe.ts` | ~150 | Boundary-constrained file ops (O_NOFOLLOW) |
| `src/infra/heartbeat-runner.ts` | 1500+ | Periodic heartbeat scheduler |
| `src/infra/agent-events.ts` | ~100 | Global agent event bus |
| `src/infra/system-events.ts` | ~50 | Ephemeral system event queue |
| `src/infra/abort-signal.ts` | ~30 | Promise-based abort signal waiting |
| `src/infra/backoff.ts` | ~50 | Backoff computation |
| `src/infra/archive.ts` | ~150 | Zip/tar extraction with limits |
| `src/infra/bonjour-discovery.ts` | ~100 | mDNS service discovery |
| `src/infra/push-apns.ts` | ~150 | Apple push notifications |

---

## 2. Hooks System (`src/hooks/`)

### 2.1 Hook Types

```typescript
type InternalHookEventType = "command" | "session" | "agent" | "gateway" | "message"
type InternalHookHandler = (event: InternalHookEvent) => Promise<void> | void
```

### 2.2 Hook Registration

```typescript
registerInternalHook(eventKey: string, handler: InternalHookHandler)
unregisterInternalHook(eventKey: string, handler: InternalHookHandler)
triggerInternalHook(event: InternalHookEvent)
hasInternalHookListeners(type, action)
```

Uses global singleton Map sharing across chunks.

### 2.3 Hook Sources

| Source | Location |
|--------|----------|
| `openclaw-bundled` | `src/hooks/bundled/` |
| `openclaw-managed` | `~/.openclaw/hooks/` |
| `openclaw-workspace` | Workspace hooks |
| `openclaw-plugin` | Plugin-registered hooks |

### 2.4 Bundled Hooks (`src/hooks/bundled/`)

**session-memory** — Saves session context to `memory/` on `/new` or `/reset`:
- Generates LLM-based slugs for filenames
- Reads previous session transcripts for context
- Writes markdown files with conversation summaries

**command-logger** — Logs command executions
**boot-md** — Gateway startup hook
**bootstrap-extra-files** — File bootstrap hook

### 2.5 Hook Installation (`src/hooks/install.ts`)

```typescript
installHooksFromArchive()
installHooksFromNpmSpec(spec)   // npm install + HOOK.md validation
installHooksFromPath(path)
resolveHookInstallDir()          // ~/.openclaw/hooks/<hookName>
```

### 2.6 Hook Metadata Schema

```typescript
type OpenClawHookMetadata = {
  always?: boolean
  hookKey?: string
  events: string[]  // ["command:new", "session:start"]
  export?: string  // export name, default "default"
  os?: string[]
  requires?: { bins?: string[]; anyBins?: string[]; env?: string[]; config?: string[] }
  install?: HookInstallSpec[]
}
```

### 2.7 Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/hooks/internal-hook-types.ts` | ~100 | Hook event/handler types |
| `src/hooks/internal-hooks.ts` | ~500 | Hook registration and triggering |
| `src/hooks/loader.ts` | ~200 | Dynamic hook loading from directories |
| `src/hooks/install.ts` | ~200 | Hook installation from npm/archive/path |
| `src/hooks/types.ts` | ~100 | Hook metadata schema |
| `src/hooks/module-loader.ts` | ~100 | Handler module resolution |
| `src/hooks/bundled/session-memory/handler.ts` | ~200 | Session memory preservation |

---

## 3. Tasks System (`src/tasks/`)

### 3.1 Task vs Agent

| Aspect | Task | Agent |
|--------|------|-------|
| Lifecycle | Background, detached | Interactive, session-bound |
| Delivery | Delivery semantics with retry | Real-time conversation |
| Parent-child | Flow orchestration with `parentFlowId` | No parent-child |
| Notification | `notifyPolicy` controls visibility | Always visible |
| Runtimes | `subagent`, `acp`, `cli`, `cron` | Interactive gateway sessions |

### 3.2 Task Registry

```typescript
// In-memory store with SQLite persistence at ~/.openclaw/tasks.db
type TaskStatus =
  | "queued" | "running" | "succeeded" | "failed"
  | "timed_out" | "cancelled" | "lost"

type TaskRuntime = "subagent" | "acp" | "cli" | "cron"
type TaskDeliveryStatus = "pending" | "delivered" | "session_queued" | "failed" | "parent_missing"
type TaskNotifyPolicy = "done_only" | "state_changes" | "silent"

// Indexes: taskIdsByRunId, taskIdsByOwnerKey, taskIdsByParentFlowId, taskIdsByRelatedSessionKey
```

### 3.3 Task Flow Registry

```typescript
// Parent containers for related tasks
type TaskFlowStatus = "queued" | "running" | "waiting" | "blocked" | "succeeded" | "failed" | "cancelled" | "lost"
type TaskFlowSyncMode = "managed" | "task_mirrored"

// CAS-style revision-based concurrency control
updateFlowRecordByIdExpectedRevision(revision, updates)
```

### 3.4 Task Executor

```typescript
createQueuedTaskRun(params)      // Create new task
startTaskRunByRunId(params)      // Start running
recordTaskRunProgressByRunId(params)
completeTaskRunByRunId(params)
failTaskRunByRunId(params)
cancelTaskById(id)
```

### 3.5 Delivery Policy

```typescript
shouldAutoDeliverTaskTerminalUpdate(policy, taskRecord)
shouldAutoDeliverTaskStateChange(policy, taskRecord)
formatTaskTerminalMessage(taskRecord)
formatTaskStateChangeMessage(taskRecord)
shouldSuppressDuplicateTerminalDelivery()
```

### 3.6 Task Registry Store (SQLite)

```typescript
~/.openclaw/tasks.db
// upsertTask, deleteTask, loadSnapshot, saveSnapshot
// Observer pattern for event notifications
```

### 3.7 Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/tasks/task-registry.ts` | 2000+ | In-memory task store with CRUD |
| `src/tasks/task-registry.types.ts` | ~200 | Task/status/delivery types |
| `src/tasks/task-executor.ts` | ~300 | Task lifecycle operations |
| `src/tasks/task-flow-registry.ts` | 680+ | Parent flow management |
| `src/tasks/detached-task-runtime.ts` | ~100 | Plugin task runtime registration |
| `src/tasks/task-executor-policy.ts` | ~100 | Delivery and notification policies |
| `src/tasks/task-registry.store.sqlite.ts` | ~200 | SQLite persistence |
| `src/tasks/task-registry.store.ts` | ~100 | Store interface |

---

## 4. Key Architectural Insights

### 4.1 Global Singletons

Both hooks and tasks use global singletons for cross-chunk compatibility:
- `resolveGlobalSingleton()` for hook runner
- `resolveGlobalDedupeCache()` for deduplication

### 4.2 Safe File Operations

All file operations use:
- `O_NOFOLLOW` flag to prevent symlink attacks
- `0o600` permissions for secrets
- Atomic write with temp file + rename
- Boundary-constrained paths (stays within root)

### 4.3 Retry Strategy

Retry uses **cryptographic jitter** (`generateSecureFraction()`) rather than predictable random, preventing thundering herd on shared failures.

### 4.4 Heartbeat-Driven Automation

The heartbeat runner (`heartbeat-runner.ts:1500+`) is the backbone of periodic automation:
- Resolves channel targets from `HEARTBEAT.md`
- Deduplicates payloads within 24h window
- Sessions isolated per heartbeat delivery

### 4.5 Task Flow Revision Control

Task flows use **CAS (compare-and-swap) revision** to prevent lost updates when multiple processes update the same flow simultaneously.