# Gateway Protocol Architecture

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

The gateway is the **client/server communication layer** that exposes OpenClaw's capabilities to frontends (web UI, mobile apps, terminal UI). It is a WebSocket-based protocol (version 3) that handles authentication, session management, and RPC-style method calls between clients and the core daemon.

```
Client (UI / Mobile / TUI)
    │ WebSocket (TLS)
    ▼
┌─────────────────────────────────────┐
│         OpenClaw Gateway            │
│  ┌─────────────────────────────┐   │
│  │   Auth Handler               │   │
│  │   Session Manager            │   │
│  │   Server Methods Registry    │   │
│  └─────────────────────────────┘   │
│           │         │              │
│    Plugin Registry  Agent Runtime   │
└─────────────────────────────────────┘
```

---

## 2. Protocol Design

### 2.1 Wire Format

All communication is JSON frames over a WebSocket connection:

```typescript
// Request (client → server)
{ type: "req", id: string, method: string, params?: unknown }

// Response (server → client)
{ type: "res", id: string, ok: boolean, payload?: unknown, error?: string }

// Server event (server → client)
{ type: "event", event: string, payload?: unknown, seq: number }
```

### 2.2 Protocol Version

```typescript
const GATEWAY_PROTOCOL_VERSION = 3
```

The version is checked during connection handshake to ensure client/server compatibility.

### 2.3 Connection Flow

```
1. Client opens WebSocket connection
2. Server sends { type: "event", event: "connect.challenge", payload: challengeData }
3. Client responds with { type: "req", method: "gateway.connect", params: authParams }
4. Server sends { type: "res", ok: true, payload: helloData }
   helloData includes: session defaults, presence, health, capabilities
5. Bidirectional communication via req/res and event frames
```

---

## 3. Authentication

### 3.1 Auth Modes

From `ui/src/ui/gateway.ts:95-110`, three auth modes are supported:

**Device Identity (Ed25519)**
```typescript
// Key pair stored in localStorage
const deviceIdentity = loadOrCreateDeviceIdentity()
// Challenge signed with private key
```

**Gateway Token**
```typescript
// Explicit token provided by user
{ token: "gw_xxxx" }
```

**Password**
```typescript
// For gateway.auth.password mode
{ password: "user-password" }
```

### 3.2 Auth Parameters

```typescript
interface GatewayConnectParams {
  role: "device" | "operator" | "node"
  scopes?: string[]
  caps?: Capability[]
  commands?: string[]
  permissions?: string[]
  deviceToken?: string        // Device identity token
  sharedToken?: string        // Shared secret token
  bootstrapToken?: string     // Bootstrap token
  password?: string           // Password auth
  challengeResponse?: string   // Ed25519 challenge response
}
```

---

## 4. Server Methods

### 4.1 Method Registration

Server methods are registered via `registerGatewayMethod()` on the plugin API and handled through `src/gateway/server-methods/`:

```typescript
// Registration
api.registerGatewayMethod("chat.send", chatSendHandler)

// Handler signature
type GatewayRequestHandler = (
  handler: GatewayRequestHandlerOptions
) => Promise<GatewayResponse> | GatewayResponse
```

### 4.2 Core Server Methods

| Method | Purpose |
|--------|---------|
| `gateway.connect` | Establish connection with auth |
| `chat.send` | Send message to agent |
| `chat.sendSpatial` | Send with spatial/multi-modal data |
| `sessions.list` | List active sessions |
| `sessions.create` | Create new session |
| `sessions.get` | Get session state |
| `sessions.patch` | Update session |
| `config.get` | Get configuration |
| `config.patch` | Update configuration |
| `config.set` | Set configuration value |
| `cron.jobs.list` | List cron jobs |
| `cron.jobs.create` | Create cron job |
| `cron.jobs.delete` | Delete cron job |
| `agents.create` | Spawn new agent |
| `agents.list` | List agents |
| `agents.get` | Get agent state |
| `nodes.list` | List connected nodes |
| `nodes.invoke` | Invoke node capability |
| `channels.list` | List configured channels |
| `channels.send` | Send via channel |

### 4.3 Schema Validation

From `src/gateway/protocol/index.ts:804`, all method params are validated via JSON Schema (using AJV):

```typescript
validateConnectParams
validateSessionsSendParams
validateChatSendParams
validateAgentsCreateParams
validateConfigSetParams
// ... extensive validator list
```

---

## 5. Sessions

### 5.1 Session Management

From `src/gateway/sessions.ts`:

Sessions track active conversations. Key operations:
- `sessions.list` — list active sessions with filters
- `sessions.create` — create new session
- `sessions.get` — get session state
- `sessions.patch` — update session metadata

### 5.2 Session Key Derivation

Sessions are identified by a derived key that incorporates:
- Agent ID
- Channel ID
- Conversation thread ID

### 5.3 ACP Session Store

From `src/acp/session.ts:150+`:
- Max 5000 sessions
- 24hr idle TTL
- Active run tracking
- Abort controller support

---

## 6. Events (Server → Client)

### 6.1 Event Types

The server pushes events to subscribed clients:

```typescript
// Chat events
{ event: "chat", payload: ChatEvent }
{ event: "chat.side_result", payload: SideResultEvent }
{ event: "session.message", payload: SessionMessageEvent }

// Agent events
{ event: "agent", payload: AgentEvent }

// Presence
{ event: "presence", payload: PresenceEvent }

// System
{ event: "shutdown", payload: ShutdownEvent }
{ event: "exec.approval.requested", payload: ExecApprovalEvent }
```

### 6.2 Event Subscription

Clients subscribe via `onEvent` callback:

```typescript
gateway.onEvent((event, payload) => {
  switch (event) {
    case "chat":
      handleChatEvent(payload)
      break
    case "presence":
      handlePresenceEvent(payload)
      break
  }
})
```

---

## 7. Client Implementations

### 7.1 Browser Client (TypeScript)

**File:** `ui/src/ui/gateway.ts:289` (645 lines)

```typescript
export class GatewayBrowserClient {
  private ws: WebSocket | null = null

  start() {
    this.ws = new WebSocket(this.opts.url)
  }

  request<T>(method: string, params?: unknown): Promise<T> {
    // JSON frame: { type: "req", id, method, params }
  }

  onEvent(handler: (event: string, payload: unknown) => void) {
    // Register event handler
  }
}
```

### 7.2 iOS Client (Swift)

**File:** `apps/shared/OpenClawKit/OpenClawKit/GatewayChannel.swift` (500+ lines)

```swift
protocol WebSocketSessioning {
  func writeMessage(_ message: GatewayMessage) async throws
  func readMessage() async throws -> GatewayMessage
}

class GatewayChannel {
  func connect(options: GatewayConnectOptions) async throws
  func send(method: String, params: [String: Any]) async throws -> [String: Any]
}
```

### 7.3 Android Client (Kotlin)

**File:** `apps/android/app/src/main/java/ai/openclaw/app/gateway/GatewaySession.kt` (500+ lines)

```kotlin
class GatewaySession {
  suspend fun connect(options: GatewayConnectOptions)
  suspend fun send(method: String, params: Map<String, Any>): Map<String, Any>
}
```

---

## 8. Gateway Boot

From `src/gateway/boot.ts`:

```typescript
bootGateway()
  → Load plugin registry
  → Set up WebSocket server (TLS)
  → Register built-in server methods
  → Register plugin gateway methods
  → Set up auth handler
  → Start client connection handler
  → Begin accepting connections
```

---

## 9. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/gateway/protocol/index.ts` | 804 | Protocol validators and schema |
| `src/gateway/server-methods/types.ts` | 118 | Gateway handler types |
| `src/gateway/server-methods/` | ~500+ | Individual method handlers |
| `src/gateway/boot.ts` | ~150 | Gateway startup |
| `src/gateway/client.ts` | ~200 | Client connection handling |
| `src/gateway/auth.ts` | ~150 | Auth surface |
| `src/gateway/call.ts` | ~100 | RPC call handling |
| `src/gateway/sessions.ts` | ~150 | Session management |
| `src/acp/session.ts` | 150+ | ACP session store |
| `ui/src/ui/gateway.ts` | 645 | GatewayBrowserClient |
| `apps/shared/OpenClawKit/OpenClawKit/GatewayChannel.swift` | 500+ | Swift gateway client |
| `apps/android/app/src/main/java/ai/openclaw/app/gateway/GatewaySession.kt` | 500+ | Kotlin gateway client |