# Terminal UI (TUI) Architecture

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

The Terminal UI (`src/tui/`) is a CLI-based interface for interacting with OpenClaw, built with `@mariozechner/pi-tui`. It is **separate from `ui/`** (the web UI) — they share the gateway protocol client but use completely different rendering systems.

---

## 2. Framework

**Library:** `@mariozechner/pi-tui`

A third-party TUI library providing:
- `Container` — layout container
- `Text` — text rendering
- `ProcessTerminal` — terminal control
- `Key` — keyboard input
- `Loader` — loading indicators
- `CombinedAutocompleteProvider` — autocomplete

---

## 3. Directory Structure

```
src/tui/
├── tui.ts                      # Main TUI setup (entry point)
├── tui-types.ts               # TypeScript types
├── tui-launch.ts              # Launch logic
├── gateway-chat.ts            # Gateway client (WebSocket)
├── tui-command-handlers.ts    # Slash command handlers
├── tui-event-handlers.ts      # Keyboard/mouse event handling
├── tui-overlays.ts            # Modal overlays
├── tui-local-shell.ts         # Local shell execution
├── components/
│   ├── chat-log.ts            # Chat log rendering
│   ├── markdown-message.ts    # Markdown rendering
│   └── custom-editor.ts       # Custom input editor
└── theme/
    └── theme.ts               # Terminal ANSI colors
```

---

## 4. Main Entry

**File:** `src/tui/tui.ts:1`

```typescript
import { CombinedAutocompleteProvider, Container, Key, Loader, ProcessTerminal, Text, TUI } from "@mariozechner/pi-tui"

export async function runTui(opts: TuiOptions) {
  const terminal = new ProcessTerminal()
  const container = new Container()
  const tui = new TUI(terminal, container)

  // Set up session key resolution
  // Set up initial agent resolution
  // Set up gateway connection
  // Start rendering loop
}
```

---

## 5. Gateway Connection (TUI)

### 5.1 TUI Gateway Client

**File:** `src/tui/gateway-chat.ts`

Similar to `ui/src/ui/gateway.ts` (GatewayBrowserClient) but adapted for TUI:

```typescript
export class TuiGatewayClient {
  constructor(opts: { url: string; auth: TuiAuth })
  async connect(): Promise<void>
  async request<T>(method: string, params?: unknown): Promise<T>
  onEvent(handler: (event: string, payload: unknown) => void): void
  disconnect(): void
}
```

### 5.2 Auth Modes

Same as web UI — device identity, gateway token, or password.

---

## 6. Command Handlers

### 6.1 Slash Commands

**File:** `src/tui/tui-command-handlers.ts`

| Command | Description |
|---------|-------------|
| `/new` | Start new conversation |
| `/reset` | Reset conversation |
| `/model <model>` | Switch model |
| `/provider <provider>` | Switch provider |
| `/clear` | Clear chat log |
| `/exit` | Exit TUI |
| `/help` | Show help |

### 6.2 Command Parser

```typescript
function parseCommand(input: string): { cmd: string; args: string[] } {
  const parts = input.trim().split(/\s+/)
  return { cmd: parts[0], args: parts.slice(1) }
}
```

---

## 7. Event Handlers

### 7.1 Keyboard Events

**File:** `src/tui/tui-event-handlers.ts`

```typescript
interface TuiKeyEvent {
  key: Key          // Which key was pressed
  ctrl: boolean     // Ctrl modifier
  shift: boolean    // Shift modifier
  alt: boolean      // Alt modifier
}

function handleKeyEvent(event: TuiKeyEvent): Action {
  switch (event.key) {
    case Key.Enter: return submitMessage()
    case Key.Escape: return closeOverlay()
    case Key.CtrlC: return interruptAgent()
    case Key.Up: return navigateHistory(-1)
    case Key.Down: return navigateHistory(+1)
    // ...
  }
}
```

### 7.2 Mouse Events

Mouse click/drag support for terminal emulators that support mouse reporting.

---

## 8. Chat Log Rendering

### 8.1 Chat Log Component

**File:** `src/tui/components/chat-log.ts`

Renders the conversation history in the terminal:

```typescript
export class ChatLog extends Container {
  constructor(messages: ChatMessage[])
  appendMessage(message: ChatMessage)
  clear()
}
```

### 8.2 Markdown Rendering

**File:** `src/tui/components/markdown-message.ts`

Renders markdown to ANSI-formatted text:

```typescript
export function renderMarkdownToAnsi(md: string): Text[] {
  // Parse markdown
  // Convert to ANSI escape codes for colors/bold/etc.
  // Return array of Text nodes
}
```

Supports: bold, italic, code blocks, links, lists.

---

## 9. Custom Editor

### 9.1 Multi-line Input

**File:** `src/tui/components/custom-editor.ts`

A terminal input editor that supports:
- Multi-line input (Enter for newline, Ctrl+Enter to send)
- History navigation (Up/Down arrows)
- Auto-complete (Tab)
- Emacs-style key bindings (Ctrl+A, Ctrl+E, etc.)

---

## 10. Overlays

### 10.1 Modal Overlays

**File:** `src/tui/tui-overlays.ts`

Overlays rendered on top of the main TUI:
- Confirmation dialogs
- Model/provider selection
- Settings panels
- Command palette

---

## 11. Theme

### 11.1 ANSI Color Theme

**File:** `src/tui/theme/theme.ts`

```typescript
export const THEME = {
  // User message (right-aligned, blue)
  userMessage: { fg: "blue", bold: true },
  // Agent message (left-aligned, white)
  agentMessage: { fg: "white" },
  // System message (centered, yellow)
  systemMessage: { fg: "yellow", italic: true },
  // Error (red)
  error: { fg: "red", bold: true },
  // Metadata (dim gray)
  metadata: { fg: "gray" },
  // ...
}
```

---

## 12. CLI Entry

**File:** `src/cli/tui-cli.ts`

```typescript
export async function runTuiCli(opts: TuiCliOptions) {
  // Parse CLI args
  // Set up terminal
  // Launch TUI
}
```

Called via: `openclaw tui`

---

## 13. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/tui/tui.ts` | ~200 | Main TUI setup |
| `src/tui/gateway-chat.ts` | ~300 | Gateway client for TUI |
| `src/tui/tui-command-handlers.ts` | ~200 | Slash command handlers |
| `src/tui/tui-event-handlers.ts` | ~150 | Keyboard/mouse handling |
| `src/tui/tui-overlays.ts` | ~100 | Modal overlays |
| `src/tui/components/chat-log.ts` | ~150 | Chat log rendering |
| `src/tui/components/markdown-message.ts` | ~100 | Markdown to ANSI |
| `src/tui/components/custom-editor.ts` | ~100 | Input editor |
| `src/tui/theme/theme.ts` | ~100 | ANSI color theme |
| `src/cli/tui-cli.ts` | ~50 | CLI entry |