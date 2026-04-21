# Frontend Architecture (Web UI)

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

The web UI (`ui/`) is built with **Lit 3.3.2** — a lightweight Web Components library using tagged template literals, NOT React. It connects to the OpenClaw core via the WebSocket gateway protocol.

---

## 2. Framework: Lit

### 2.1 Why Lit?

- **Web Components** — native browser components, no virtual DOM
- **Tagged template literals** — HTML-in-JS without JSX
- **Lightweight** — ~15KB gzipped vs React's ~40KB+
- **Native** — works with any framework or vanilla JS

### 2.2 Main Component

**File:** `ui/src/ui/app.ts:132` (824 lines)

```typescript
@customElement("openclaw-app")
export class OpenClawApp extends LitElement {
  @state() private state: AppState = { /* ... */ }

  render() {
    return html`
      <div class="app">
        ${this.renderHeader()}
        ${this.renderContent()}
        ${this.renderSidebar()}
      </div>
    `
  }
}
```

---

## 3. Directory Structure

```
ui/src/
├── main.ts                     # Bootstrap entry (2 lines)
├── styles.css                  # Main CSS import
├── local-storage.ts            # LocalStorage wrapper

├── ui/
│   ├── app.ts                  # Main <openclaw-app> (824 lines)
│   ├── app-gateway.ts          # Gateway connection (610 lines)
│   ├── app-chat.ts             # Chat send/queue (567 lines)
│   ├── app-render.ts          # Main render (~2400 lines)
│   ├── app-lifecycle.ts        # Connected/updated/disconnected
│   ├── app-settings.ts        # Theme, URL, tab settings
│   ├── app-scroll.ts          # Auto-scroll management
│   ├── app-polling.ts         # Polling for logs/nodes/debug
│   ├── app-events.ts          # Event log buffer
│   ├── app-tool-stream.ts     # Tool streaming state
│   ├── app-channels.ts        # Channel config handlers
│   ├── app-defaults.ts        # Default values
│   ├── gateway.ts             # GatewayBrowserClient (645 lines)
│   ├── navigation.ts          # Tab routing
│   ├── storage.ts             # UiSettings persistence

│   ├── controllers/           # Data loading/management
│   │   ├── chat.ts
│   │   ├── sessions.ts
│   │   ├── agents.ts
│   │   ├── channels.ts
│   │   ├── config.ts
│   │   ├── cron.ts
│   │   ├── nodes.ts
│   │   ├── logs.ts
│   │   ├── devices.ts
│   │   ├── exec-approvals.ts
│   │   ├── dreaming.ts
│   │   ├── skills.ts
│   │   ├── presence.ts
│   │   ├── debug.ts
│   │   └── ... (more)

│   ├── views/                 # Lazy-loaded view components
│   │   ├── chat.ts
│   │   ├── agents.ts
│   │   ├── channels.ts
│   │   ├── sessions.ts
│   │   ├── cron.ts
│   │   ├── skills.ts
│   │   ├── nodes.ts
│   │   ├── logs.ts
│   │   ├── config.ts
│   │   ├── dreaming.ts
│   │   ├── command-palette.ts
│   │   ├── overview.ts
│   │   ├── debug.ts
│   │   └── usage.ts

│   ├── chat/                  # Chat-specific rendering
│   │   ├── build-chat-items.ts
│   │   ├── message-normalizer.ts
│   │   ├── grouped-render.ts
│   │   ├── tool-cards.ts
│   │   ├── slash-commands.ts
│   │   ├── speech.ts
│   │   ├── export.ts
│   │   └── ...

│   └── components/
│       ├── dashboard-header.ts
│       └── resizable-divider.ts

├── i18n/
│   ├── index.ts
│   ├── locales/
│   │   ├── en.ts, de.ts, fr.ts, ... (11 locales)
│   └── lib/
│       ├── registry.ts
│       ├── translate.ts
│       ├── types.ts
│       └── lit-controller.ts
```

---

## 4. Gateway Connection

### 4.1 GatewayBrowserClient

**File:** `ui/src/ui/gateway.ts:289` (645 lines)

```typescript
export class GatewayBrowserClient {
  private ws: WebSocket | null = null

  start() {
    this.ws = new WebSocket(this.opts.url)
    this.ws.onmessage = (e) => this.handleMessage(JSON.parse(e.data))
    this.ws.onopen = () => this.onConnected()
    this.ws.onerror = (e) => this.onError(e)
    this.ws.onclose = () => this.onDisconnected()
  }

  request<T>(method: string, params?: unknown): Promise<T> {
    // Sends: { type: "req", id, method, params }
    // Returns: Promise that resolves when { type: "res" } arrives
  }

  onEvent(handler: (event: string, payload: unknown) => void) {
    // Registers handler for { type: "event" } messages
  }
}
```

### 4.2 Connection Flow

**File:** `ui/src/ui/app-gateway.ts:251-366`

```
1. User enters gateway URL + optional token/password
2. connectGateway() creates GatewayBrowserClient
3. Client opens WebSocket
4. Server sends connect.challenge event
5. Client responds with gateway.connect method
6. Server sends hello with snapshot data (session defaults, presence, health)
7. UI sets up event subscriptions
```

### 4.3 Auth Modes

From `gateway.ts:95-110`:
- **Device identity** — Ed25519 key pair from localStorage
- **Gateway token** — explicit token from user
- **Password** — for `gateway.auth.password` mode

---

## 5. Routing

### 5.1 Tab Groups

**File:** `ui/src/ui/navigation.ts:5-25`

```typescript
export const TAB_GROUPS = [
  { label: "chat", tabs: ["chat"] },
  { label: "control", tabs: ["overview", "channels", "instances", "sessions", "usage", "cron"] },
  { label: "agent", tabs: ["agents", "skills", "nodes", "dreams"] },
  { label: "settings", tabs: ["config", "communications", "appearance", "automation", "infrastructure", "aiAgents", "debug", "logs"] },
]
```

### 5.2 Path Mapping

**File:** `ui/src/ui/navigation.ts:48-68`

```typescript
const TAB_PATHS: Record<Tab, string> = {
  chat: "/chat",
  overview: "/overview",
  channels: "/channels",
  sessions: "/sessions",
  cron: "/cron",
  // ...
}

const PATH_TAB: Record<string, Tab> = {
  "/chat": "chat",
  "/overview": "overview",
  // ...
}
```

### 5.3 URL Sync

```typescript
// From app.ts
connectedCallback() {
  window.addEventListener("popstate", this.onPopState)
}

private onPopState = () => {
  const tab = PATH_TAB[window.location.pathname] ?? "chat"
  this.state = { ...this.state, tab }
}
```

---

## 6. Lazy Views

### 6.1 Lazy Loading

**File:** `ui/src/ui/app-render.ts:159-174`

```typescript
const lazyAgents = createLazy(() => import("./views/agents.ts"))
const lazyChannels = createLazy(() => import("./views/channels.ts"))
const lazyCron = createLazy(() => import("./views/cron.ts"))
// ... more

// In render():
switch (tab) {
  case "agents": return lazyAgents()
  case "channels": return lazyChannels()
  case "cron": return lazyCron()
  // ...
}
```

### 6.2 createLazy Implementation

```typescript
function createLazy<T extends LitElement>(
  importer: () => Promise<{ default: new () => T }>
): () => T {
  let cache: T | null = null
  return () => {
    if (!cache) {
      // First call: trigger import, show placeholder
      // Subsequent calls: return cached instance
    }
    return cache
  }
}
```

---

## 7. Controllers

Controllers manage data loading and state for views:

| Controller | Responsibility |
|------------|---------------|
| `chat.ts` | Chat message list, send queue |
| `sessions.ts` | Session list and state |
| `agents.ts` | Agent configuration |
| `channels.ts` | Channel configuration |
| `config.ts` | Global config |
| `cron.ts` | Cron job management |
| `nodes.ts` | Connected nodes |
| `logs.ts` | Log entries |
| `devices.ts` | Paired devices |
| `exec-approvals.ts` | Approval requests |
| `dreaming.ts` | Dream mode |
| `skills.ts` | Skill management |
| `presence.ts` | Presence state |
| `debug.ts` | Debug info |

---

## 8. i18n (Internationalization)

### 8.1 Locale Structure

**File:** `ui/src/i18n/index.ts`

11 locales in `ui/src/i18n/locales/`:
`en.ts`, `de.ts`, `fr.ts`, `es.ts`, `pt.ts`, `zh.ts`, `ja.ts`, `ko.ts`, `ru.ts`, `ar.ts`, `hi.ts`

### 8.2 Translation System

```typescript
// ui/src/i18n/lib/translate.ts
export function t(key: string, params?: Record<string, string>): string {
  const locale = currentLocale()
  const messages = loadedLocales[locale]
  return messages[key] ?? key
}
```

### 8.3 Lit Controller for i18n

```typescript
// ui/src/i18n/lib/lit-controller.ts
export class I18nMixin(SuperClass: typeof LitElement) {
  @property() locale = "en"

  updated(changedProperties: Map<string, unknown>) {
    if (changedProperties.has("locale")) {
      this.requestUpdate()
    }
  }
}
```

---

## 9. HTML Entry Point

**File:** `ui/index.html:66`

```html
<openclaw-app></openclaw-app>
<script type="module" src="/src/main.ts"></script>
```

---

## 10. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `ui/src/ui/app.ts` | 824 | Main `<openclaw-app>` component |
| `ui/src/ui/gateway.ts` | 645 | GatewayBrowserClient WebSocket client |
| `ui/src/ui/app-gateway.ts` | 610 | Gateway connection management |
| `ui/src/ui/app-chat.ts` | 567 | Chat send/queue logic |
| `ui/src/ui/app-render.ts` | ~2400 | Main render (largest file) |
| `ui/src/ui/navigation.ts` | ~150 | Tab routing |
| `ui/src/main.ts` | 2 | Bootstrap entry |
| `ui/index.html` | ~100 | Entry HTML |