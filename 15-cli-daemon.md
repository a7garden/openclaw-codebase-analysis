# CLI & Daemon Architecture

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

The CLI (`src/cli/`) handles all command-line interaction with OpenClaw, from launching the daemon to running the setup wizard. The daemon is the long-lived process that runs the gateway server and manages plugins.

---

## 2. CLI Entry Chain

### 2.1 Process Entry

```
Node.js process starts
  → src/index.ts:85         runLegacyCliEntry()
  → src/entry.ts:182        runMainOrRootHelp()
    → import("./cli/run-main.js")
    → src/cli/run-main.ts:150  runCli()
```

### 2.2 Main CLI File

**File:** `src/cli/run-main.ts` (292 lines)

```typescript
export async function runCli(args: string[] = process.argv): Promise<number> {
  const program = new Command()

  // Set program name and version
  program.name("openclaw").version(version)

  // Register global options
  // --profile, --config, --verbose, etc.

  // Register built-in commands
  // dev, gateway, tui, wizard, doctor, config, plugins

  // Register plugin commands via registerCli()
  for (const plugin of loadedPlugins) {
    for (const cmd of plugin.cliCommands) {
      program.addCommand(adaptCommand(cmd))
    }
  }

  // Parse and execute
  return program.parse(args)
}
```

---

## 3. CLI Commands

### 3.1 Built-in Commands

| Command | File | Description |
|---------|------|-------------|
| `openclaw dev` | `src/cli/dev.ts` | Development mode |
| `openclaw gateway` | `src/cli/gateway-cli.ts` | Start gateway daemon |
| `openclaw tui` | `src/cli/tui-cli.ts` | Launch terminal UI |
| `openclaw wizard` | `src/cli/wizard-cli.ts` | Launch setup wizard |
| `openclaw doctor` | `src/cli/doctor.ts` | Diagnostics and fixes |
| `openclaw config` | `src/cli/config-cli.ts` | Config inspection/mutation |
| `openclaw plugins` | `src/cli/plugins-cli.ts` | Plugin management |
| `openclaw help` | (built-in) | Show help |

### 3.2 Plugin Commands

Plugins can register CLI commands via `registerCli()`:

```typescript
// Plugin API
api.registerCli({
  name: "my-plugin command",
  description: "Does something",
  options: [
    { name: "--flag", description: "A flag" }
  ],
  handler: async (args) => {
    // Handle command
  }
})
```

---

## 4. Gateway Daemon

### 4.1 Gateway CLI

**File:** `src/cli/gateway-cli.ts` (100+ lines)

```typescript
export async function runGatewayCli(args: GatewayCliOptions): Promise<void> {
  // 1. Load config
  const config = await loadConfig(args.profile)

  // 2. Load plugins
  const plugins = await loadPlugins(config.plugins)

  // 3. Create gateway server
  const server = await createGatewayServer({
    config,
    plugins,
    tls: args.tls,
    port: args.port,
    host: args.host,
  })

  // 4. Start server
  await server.listen()

  // 5. Handle shutdown
  process.on("SIGTERM", () => server.shutdown())
}
```

### 4.2 Daemon Lifecycle

```
start
  → load config
  → load plugins (discovery → manifest → activation)
  → initialize gateway server
  → start listening
  → [daemon running]
  → SIGTERM/SIGINT
  → shutdown (close connections, flush state)
  → exit
```

---

## 5. Config Resolution

### 5.1 Config File Locations

```
~/.openclaw/config.json         # Main config
~/.openclaw/credentials/         # Secrets (not in config.json)
~/.openclaw/agents/             # Per-agent config
```

### 5.2 Profile Support

```
openclaw --profile dev gateway     # Use ~/.openclaw/profiles/dev/config.json
openclaw --profile prod gateway    # Use ~/.openclaw/profiles/prod/config.json
```

### 5.3 Config Override Precedence

```
CLI flags (highest)
  → Profile config
    → ~/.openclaw/config.json
      → Defaults (lowest)
```

---

## 6. Wizard System

### 6.1 Wizard CLI

**File:** `src/cli/wizard-cli.ts`

Launches interactive setup via `src/wizard/`:

```typescript
export async function runWizardCli(args: WizardCliOptions): Promise<void> {
  const wizard = createWizard({
    steps: ["plugin-config", "gateway-config", "secrets"],
    interactive: args.interactive,
    profile: args.profile,
  })
  await wizard.run()
}
```

### 6.2 Wizard Steps

| Step | File | Purpose |
|------|------|---------|
| Plugin config | `src/wizard/setup.plugin-config.ts` (416 lines) | Configure plugins |
| Gateway config | `src/wizard/setup.gateway-config.ts` (358 lines) | Gateway settings |
| Secrets | `src/wizard/setup.secret-input.ts` (41 lines) | Enter API keys |
| Security note | `src/wizard/setup.security-note.ts` (40 lines) | Security warnings |
| Finalize | `src/wizard/setup.finalize.ts` (662 lines) | Write config |

### 6.3 Wizard Session

**File:** `src/wizard/session.ts` (272 lines)

Manages wizard state across steps:
- Current step
- Completed steps
- Config values entered
- Validation state

---

## 7. Progress Display

### 7.1 CLI Progress

**File:** `src/cli/progress.ts`

```typescript
export function showProgress(message: string, spinner?: Spinner): void
export function stopProgress(): void
export function log(message: string, level?: "info" | "warn" | "error"): void
```

Used throughout CLI for user feedback during long operations.

### 7.2 Status Tables

**File:** `src/terminal/table.ts`

```typescript
export function renderStatusTable(rows: StatusRow[]): string
// Renders aligned columns for CLI output
```

---

## 8. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/cli/run-main.ts` | 292 | Main CLI orchestration |
| `src/cli/run-main.ts:150` | — | `runCli()` entry |
| `src/cli/gateway-cli.ts` | 100+ | Gateway daemon CLI |
| `src/cli/tui-cli.ts` | ~50 | TUI launch |
| `src/cli/wizard-cli.ts` | ~50 | Wizard launch |
| `src/cli/doctor.ts` | ~200 | Diagnostics |
| `src/cli/config-cli.ts` | ~100 | Config CLI |
| `src/cli/plugins-cli.ts` | ~100 | Plugin management CLI |
| `src/cli/progress.ts` | ~50 | Progress display |
| `src/terminal/table.ts` | ~50 | Status table rendering |
| `src/wizard/session.ts` | 272 | Wizard session state |
| `src/wizard/setup.finalize.ts` | 662 | Config finalization |