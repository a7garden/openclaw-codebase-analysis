# Testing Strategy

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

OpenClaw uses **Vitest** as its test framework. Tests are colocated with source (`*.test.ts`) and follow specific patterns to avoid common pitfalls with hot tests and module mocking.

---

## 2. Test Layout

### 2.1 Test File Placement

```
src/
├── foo.ts
└── foo.test.ts          ← Colocated with source

extensions/telegram/
├── src/
│   └── channel.ts
└── src/
    └── channel.test.ts  ← In src/ subdirectory
```

### 2.2 E2E Tests

```
src/
└── foo.e2e.test.ts      ← E2E tests with .e2e suffix
```

### 2.3 Test Helpers

```
test/
├── helpers/
│   ├── AGENTS.md         ← Test helper rules
│   ├── channels/
│   │   └── AGENTS.md    ← Channel test helper rules
│   ├── mock-plugin.ts    ← Mock plugin builder
│   ├── mock-runtime.ts   ← Mock runtime
│   └── ...
```

---

## 3. Test Commands

### 3.1 Basic Commands

| Command | Purpose |
|---------|---------|
| `pnpm test` | Full test suite |
| `pnpm test:changed` | Only tests for changed files |
| `pnpm test:coverage` | Coverage report |
| `pnpm test:live` | Live testing with `OPENCLAW_LIVE_TEST=1` |
| `pnpm test:live -- --filter="chat"` | Filtered live tests |

### 3.2 Extension Tests

| Command | Purpose |
|---------|---------|
| `pnpm test:extensions` | All extension shards |
| `pnpm test:extensions/<id>` | One extension lane |

### 3.3 Performance Profiling

| Command | Purpose |
|---------|---------|
| `pnpm test:perf:imports <file>` | Import time profiling |
| `pnpm test:perf:hotspots --limit N` | Find slowest test suites |

### 3.4 Shard Timing

Shard timing artifact: `.artifacts/vitest-shard-timings.json`

Used automatically for balanced shard ordering. Disable with `OPENCLAW_TEST_PROJECTS_TIMINGS=0`.

---

## 4. Hot Test Patterns

### 4.1 The Problem

Per-test `vi.resetModules()` + fresh heavy imports cause slow tests because:
- Module graph is re-evaluated for each test
- Heavy imports (providers, plugins) are re-parsed and re-executed

### 4.2 Recommended Pattern

```typescript
// GOOD: Static or beforeAll imports, reset state directly
import { createAgentRuntime } from "../src/agents/runtime"

let runtime: AgentRuntime

beforeAll(() => {
  runtime = createAgentRuntime({ /* config */ })
})

afterEach(() => {
  runtime.resetState()  // Reset without re-importing
})

test("agent can send message", async () => {
  const response = await runtime.sendMessage({ text: "hello" })
  expect(response.text).toBeDefined()
})
```

### 4.3 Avoid

```typescript
// BAD: vi.resetModules() + fresh imports per test
beforeEach(async () => {
  vi.resetModules()
  runtime = await import("../src/agents/runtime")
})

test("agent can send message", async () => {
  // Slow — re-imports everything each test
})
```

---

## 5. Mocking Strategy

### 5.1 Mock Expensive Runtime Seams

Mock at runtime boundaries, not at module level:

| Mock | Use Instead Of |
|------|----------------|
| Mock scanner | Scanning filesystem |
| Mock manifest | Plugin manifest parsing |
| Mock package registry | Package version lookups |
| Mock provider SDK | Provider API calls |
| Mock network | HTTP/WebSocket calls |

### 5.2 Prefer Injected Deps Over Module Mocks

```typescript
// GOOD: Inject mock at boundary
function createAgentRuntime(deps: { provider: Provider }) {
  return new AgentRuntime(deps)
}

const mockProvider = { runInference: vi.fn() }
const runtime = createAgentRuntime({ provider: mockProvider })
```

### 5.3 Narrow Local Mocks

If mocking modules, mock narrow local `*.runtime.ts` seams:

```typescript
// GOOD: Mock the local runtime seam
vi.mock("../src/agents/anthropic-transport-stream.runtime", () => ({
  createTransport: vi.fn().mockResolvedValue(mockTransport),
}))
```

### 5.4 Avoid

```typescript
// BAD: Broad module mocks
vi.mock("openclaw/plugin-sdk", () => ({ /* ... */ }))
vi.mock("../src/channels/plugins/types", () => ({ /* ... */ }))
```

---

## 6. Test Isolation

### 6.1 Worker Isolation

Tests run in Vitest workers. Each worker has its own module cache.

```bash
# Limit workers for memory pressure
OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test
```

### 6.2 Shared Fixtures

```typescript
// test/helpers/fixtures/agent-factory.ts
export function createTestAgent(config?: Partial<AgentConfig>): Agent {
  return new Agent({
    model: "sonnet-4.6",
    provider: "anthropic",
    systemPrompt: "You are a test agent",
    ...config,
  })
}
```

### 6.3 Cleanup

Clean up timers, env vars, globals, mocks, sockets, temp dirs:

```typescript
afterEach(() => {
  vi.clearAllMocks()
  vi.useRealTimers()
  delete process.env.TEST_FLAG
})
```

---

## 7. Live Testing

### 7.1 Live Test Mode

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live
```

### 7.2 Quiet Logs

```bash
OPENCLAW_LIVE_TEST_QUIET=0 pnpm test:live
# Shows full logs
```

### 7.3 When to Use

Live tests for:
- Actual provider API calls
- Real database/storage operations
- End-to-end flows that need real infrastructure

---

## 8. Performance Testing

### 8.1 Import Drag

```bash
pnpm test:perf:imports src/agents/pi-embedded-runner/index.ts
```

Reports:
- Total parse time
- Total import time
- Breakdown by imported module

### 8.2 Hotspot Discovery

```bash
pnpm test:perf:hotspots --limit 20
```

Finds the 20 slowest test files.

---

## 9. Architecture for Testability

### 9.1 Named Composition

Extract production logic into named helpers that can be tested independently:

```typescript
// GOOD: Named, testable helper
export function buildInferenceParams(options: InferenceOptions): InferenceParams {
  return {
    model: options.model,
    messages: options.messages,
    maxTokens: options.maxTokens ?? 4096,
    temperature: options.temperature ?? 1.0,
  }
}

// Test directly
test("buildInferenceParams applies defaults", () => {
  const params = buildInferenceParams({ model: "claude-3", messages: [] })
  expect(params.maxTokens).toBe(4096)
  expect(params.temperature).toBe(1.0)
})
```

### 9.2 NOT:

```typescript
// BAD: Inline anonymous logic in the inference method
async runInference(options: InferenceOptions) {
  const params = {
    model: options.model,
    messages: options.messages,
    maxTokens: options.maxTokens ?? 4096,  // Can't test this in isolation
  }
  // ...
}
```

---

## 10. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `vitest.config.ts` | ~100 | Vitest configuration |
| `test/helpers/AGENTS.md` | ~50 | Test helper rules |
| `test/helpers/channels/AGENTS.md` | ~50 | Channel test rules |
| `.artifacts/vitest-shard-timings.json` | — | Shard timing data |
| `docs/help/testing.md` | ~200 | Full testing guide |