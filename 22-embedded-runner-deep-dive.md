# pi-embedded-runner Deep Dive

**File:** `src/agents/pi-embedded-runner/run.ts` (main orchestrator, ~2641 lines)  
**Core attempt execution:** `src/agents/pi-embedded-runner/run/attempt.ts` (~2642 lines)  
**Backend proxy:** `src/agents/pi-embedded-runner/run/backend.ts` (9 lines)  
**Auth controller:** `src/agents/pi-embedded-runner/run/auth-controller.ts` (527 lines)  
**Transport stream resolution:** `src/agents/pi-embedded-runner/stream-resolution.ts` (128 lines)

---

## 1. Single-Attempt Execution Engine

### 1.1 Entry Points and Flow

The embedded runner has two distinct execution paths:

#### Primary Entry: `runEmbeddedPiAgent()` (`run.ts:204`)

This is the **outer retry loop orchestrator** that manages the complete agent lifecycle including:
- Auth profile rotation across attempts
- Compaction decision-making between attempts
- Retry policy enforcement
- Timeout tracking

```
runEmbeddedPiAgent()
├── Initialize auth profile candidates (line 444)
├── Acquire session write lock
├── while (true) {  // Main retry loop (line 630)
│   ├── runEmbeddedAttemptWithBackend()  // Single attempt
│   ├── Handle attempt results
│   │   ├── Check for context overflow → trigger compaction
│   │   ├── Check for prompt errors → classify failover
│   │   ├── Check for auth errors → rotate profiles
│   │   └── Check for timeout → timeout compaction
│   └── Loop termination or continue
└── Return final EmbeddedPiRunResult
```

#### Single-Attempt: `runEmbeddedAttemptWithBackend()` (`run/backend.ts:4-8`)

```typescript
export async function runEmbeddedAttemptWithBackend(
  params: EmbeddedRunAttemptParams,
): Promise<EmbeddedRunAttemptResult> {
  return runAgentHarnessAttemptWithFallback(params);
}
```

This delegates to `runAgentHarnessAttemptWithFallback()` which is the **actual per-attempt execution harness** - the single-shot execution unit. Each call to `runEmbeddedAttemptWithBackend()` represents one distinct attempt with one specific auth profile and model configuration.

#### Per-Attempt Execution: `runEmbeddedAttempt()` (`run/attempt.ts:417-2641`)

This is the core single-attempt execution engine. It:

1. **Initializes the session** (line 448-1127):
   - Creates/resumes a `SessionManager`
   - Builds system prompt
   - Resolves tools (built-in + custom + MCP + LSP)
   - Establishes transport stream function
   - Applies stream wrapper chain

2. **Applies stream wrappers** (lines 1198-1471):
   - Each wrapper transforms the stream function
   - Wrappers handle auth injection, payload normalization, error recovery

3. **Sanitizes session history** (lines 1473-1524):
   - Removes thinking blocks (transcript policy)
   - Sanitizes tool call IDs
   - Downgrades reasoning blocks for strict providers

4. **Runs the prompt** (lines 1824-2184):
   - Builds effective prompt with hooks
   - Calls `activeSession.prompt()` - the actual LLM call

5. **Collects results** (lines 2300-2578):
   - Extracts assistant texts
   - Collects tool metadata
   - Builds usage totals

### 1.2 Key Differences: `runEmbeddedPiAgent()` vs `runEmbeddedAttempt()`

| Aspect | `runEmbeddedPiAgent()` (outer) | `runEmbeddedAttempt()` (inner) |
|--------|-------------------------------|-------------------------------|
| **Scope** | Full agent lifecycle | Single API call attempt |
| **Auth handling** | Profile rotation across attempts | Uses provided auth profile |
| **Compaction** | Triggers between attempts | Self-contained (SDK auto-compacts) |
| **Retry loop** | Yes, manages iterations | No, one-shot |
| **Timeout handling** | Per-attempt timeout + grace period | Uses `runAbortController` |
| **Abort propagation** | Propagates to attempt via abort signal | Creates internal `runAbortController` |

### 1.3 Compaction Pipeline

Compaction is triggered when context overflow threatens to kill a prompt. The system has **three compaction triggers**:

#### Trigger 1: Timeout-Triggered Compaction (`run.ts:885-980`)

When LLM times out with high context usage (>65% of token budget consumed by prompt):

```
timedOut && tokenUsedRatio > 0.65
  → contextEngine.compact({ force: true, trigger: "timeout_recovery" })
  → MAX_TIMEOUT_COMPACTION_ATTEMPTS = 2
```

#### Trigger 2: Overflow-Triggered Compaction (`run.ts:1004-1152`)

When context overflow error detected in assistant response:

```
isLikelyContextOverflowError(errorText)
  → contextEngine.compact({ force: true, trigger: "overflow" })
  → MAX_OVERFLOW_COMPACTION_ATTEMPTS = 3
```

#### Trigger 3: Preemptive Compaction (`run.ts:2076-2164`)

Before prompt submission, checks if context would overflow:

```typescript
shouldPreemptivelyCompactBeforePrompt({
  messages, systemPrompt, prompt, contextTokenBudget, reserveTokens
})
  → Can truncate tool results only (fast path)
  → Or trigger full compaction
```

#### Compaction Execution (`compact.ts:146-300+`)

The `compactEmbeddedPiSession()` function:

1. Acquires session write lock
2. Creates a **separate compaction session** via `createAgentSession()`
3. Builds a summary prompt asking the model to summarize conversation
4. Runs the summary prompt against the model
5. Replaces old messages with the summary
6. Runs post-compaction hooks

Key code path in `compact.ts`:
```typescript
// Line ~400: Create compaction session
({ session } = await createAgentSession({...}));

// Line ~500: Run summary prompt
await session.prompt(summaryPrompt);

// Line ~600: Truncate session after compaction
truncateSessionAfterCompaction(sessionManager, firstKeptEntryId);

// Line ~700: Post-compaction hooks
await runPostCompactionHooks({...});
```

### 1.4 Auth Profile Rotation

The `createEmbeddedRunAuthController()` (`run/auth-controller.ts:43-527`) manages auth throughout the run loop.

#### Initialization (`initializeAuthProfile()`, line 436-497)

```typescript
async function initializeAuthProfile() {
  // Filter out locked profile from auto-candidates
  const autoProfileCandidates = profileCandidates.filter(
    c => c !== lockedProfileId
  );
  
  // Check if all auto-profiles are in cooldown
  const allAutoProfilesInCooldown = autoProfileCandidates.every(
    c => isProfileInCooldown(store, c, undefined, modelId)
  );
  
  // If all in cooldown and transient probe allowed, probe anyway
  const allowTransientCooldownProbe = allAutoProfilesInCooldown && shouldAllowCooldownProbe(...);
  
  // Iterate through candidates until one works
  while (profileIndex < profileCandidates.length) {
    const candidate = profileCandidates[profileIndex];
    if (inCooldown && !allowTransientCooldownProbe) {
      profileIndex++;
      continue;
    }
    await applyApiKeyInfo(candidate);
    break;
  }
}
```

#### Profile Advancement (`advanceAuthProfile()`, line 406-434)

```typescript
async function advanceAuthProfile(): Promise<boolean> {
  if (lockedProfileId) return false;  // Never rotate locked profiles
  
  let nextIndex = profileIndex + 1;
  while (nextIndex < profileCandidates.length) {
    const candidate = profileCandidates[nextIndex];
    
    // Skip profiles in cooldown
    if (isProfileInCooldown(store, candidate, undefined, modelId)) {
      nextIndex++;
      continue;
    }
    
    try {
      await applyApiKeyInfo(candidate);
      profileIndex = nextIndex;
      thinkLevel = initialThinkLevel;  // Reset thinking on profile change
      attemptedThinking.clear();
      return true;
    } catch (err) {
      nextIndex++;
    }
  }
  return false;
}
```

#### Runtime Auth Refresh (`refreshRuntimeAuth()`, line 160-235)

For providers with expiring credentials (OAuth tokens):

```typescript
async function refreshRuntimeAuth(reason: string) {
  const runtimeAuthState = getRuntimeAuthState();
  
  // Schedule refresh before expiry
  const refreshAt = runtimeAuthState.expiresAt - RUNTIME_AUTH_REFRESH_MARGIN_MS;
  scheduleRuntimeAuthRefresh(refreshAt);
  
  // On refresh, call provider's refresh endpoint
  const preparedAuth = await prepareRuntimeAuthForModel({
    runtimeModel, apiKey: sourceApiKey, authMode, profileId
  });
  
  // Update stored credentials
  authStorage.setRuntimeApiKey(provider, preparedAuth.apiKey);
}
```

### 1.5 Tool Execution and Result Streaming

Tools execute within `activeSession.prompt()` as part of the agent loop. The pi-agent-core SDK handles tool call/result cycling internally.

#### Tool Result Handling (`run/attempt.ts:2496-2501`)

```typescript
const toolMetasNormalized = toolMetas
  .filter((entry): entry is { toolName: string; meta?: string } =>
    typeof entry.toolName === "string" && entry.toolName.trim().length > 0,
  )
  .map((entry) => ({ toolName: entry.toolName, meta: entry.meta }));
```

#### Tool Result Truncation (`tool-result-truncation.ts`)

When tool results are too large for context:
```typescript
truncateOversizedToolResultsInSession({
  sessionManager,
  contextWindowTokens,
  maxCharsOverride: toolResultMaxChars,
})
```

#### Client Tool Detection (`run/attempt.ts:1057-1107`)

Client tools (OpenResponses hosted tools) are detected via:
```typescript
clientToolCallDetected = { name: toolName, params: toolParams };
```

### 1.6 Error Classification and Handling Paths

#### Error Sources

1. **Prompt errors** - failures during `session.prompt()` call
2. **Assistant errors** - model returned `stopReason: "error"`
3. **Context overflow** - prompt too large for context window
4. **Auth errors** - invalid/expired credentials
5. **Rate limits** - provider rate limiting
6. **Timeouts** - LLM not responding

#### Error Classification (`pi-embedded-helpers.ts`)

```typescript
classifyFailoverReason(errorText, { provider })
  → "auth"        // 401, invalid credentials
  → "rate_limit"  // 429, too many requests
  → "overloaded"  // 503, server busy
  → "timeout"     // Request timeout
  → "context_overflow"  // Prompt too large
  → "billing"     // Payment required
```

#### Error Handling Decision Tree (`run.ts:1364-1391`)

```typescript
let promptFailoverDecision = resolveRunFailoverDecision({
  stage: "prompt",
  aborted,
  externalAbort,
  fallbackConfigured,
  failoverFailure: promptFailoverFailure,
  failoverReason: promptFailoverReason,
  profileRotated: false,
});

// Possible actions from failover policy:
// - "rotate_profile" → advanceAuthProfile(), continue
// - "fallback_model" → try next model in fallback chain
// - "give_up" → return error payload
```

#### Specific Error Paths

**Context Overflow** (`run.ts:1004-1234`):
```
detected → compact → retry → if still overflow → tool truncation → retry → give up
```

**Auth Error** (`run.ts:1253-1255`):
```
detected → maybeRefreshRuntimeAuth() → if refreshed, retry
           ↓ if not refreshable
           advanceAuthProfile() → retry with new profile
```

**Rate Limit** (`run.ts:1357-1362`):
```
detected → maybeEscalateRateLimitProfileFallback()
           → if profile rotation exhausted → fallback to model
```

---

## 2. Transport Stream System

### 2.1 Stream Function Architecture

The **stream function** (`StreamFn`) is the core abstraction for making LLM API calls. It takes:

```typescript
type StreamFn = (
  model: Model<Api>,           // Model descriptor
  context: StreamContext,       // Messages + system prompt
  options?: StreamOptions      // Auth headers, transport config
) => AsyncGenerator<StreamEvent>
```

### 2.2 Stream Resolution Chain (`stream-resolution.ts`)

The `resolveEmbeddedAgentStreamFn()` function (`stream-resolution.ts:64-128`) builds the appropriate stream function:

```typescript
export function resolveEmbeddedAgentStreamFn(params: {
  currentStreamFn: StreamFn | undefined;
  providerStreamFn?: StreamFn;
  shouldUseWebSocketTransport: boolean;
  wsApiKey?: string;
  sessionId: string;
  signal?: AbortSignal;
  model: EmbeddedRunAttemptParams["model"];
  resolvedApiKey?: string;
  authStorage?: { getApiKey(provider: string): Promise<string | undefined> };
}): StreamFn
```

**Resolution priority:**

1. **Provider stream** - If `providerStreamFn` exists (from `registerProviderStreamForModel`), use it
2. **WebSocket transport** - For OpenAI models with API key, use WebSocket
3. **Anthropic Vertex** - Special handling for Vertex AI
4. **Boundary-aware** - For models that need boundary markers
5. **Session default** - Fall back to session's stream function

### 2.3 The Stream Wrapper Chain (`run/attempt.ts:1198-1471`)

Each wrapper wraps the previous one, creating a **layered transformation pipeline**. Here's the complete chain:

#### Wrapper 1: Cache Trace (`attempt.ts:1304`)

```typescript
activeSession.agent.streamFn = cacheTrace.wrapStreamFn(activeSession.agent.streamFn);
```
- Records timing for each stage
- Logs prompt cache events

#### Wrapper 2: Thinking Block Dropping (`attempt.ts:1311-1329`)

```typescript
if (transcriptPolicy.dropThinkingBlocks) {
  activeSession.agent.streamFn = (model, context, options) => {
    const messages = ctx?.messages;
    if (!Array.isArray(messages)) return inner(model, context, options);
    const sanitized = dropThinkingBlocks(messages);
    return inner(model, { ...context, messages: sanitized }, options);
  };
}
```
- Strips `thinking` blocks from replayed messages
- Required for Anthropic endpoints that reject replayed thinking signatures

#### Wrapper 3: Tool Call ID Sanitization (`attempt.ts:1341-1374`)

```typescript
if (transcriptPolicy.sanitizeToolCallIds && transcriptPolicy.toolCallIdMode) {
  activeSession.agent.streamFn = (model, context, options) => {
    const nextMessages = sanitizeReplayToolCallIdsForStream({ messages, mode, allowedToolNames });
    return inner(model, { ...context, messages: nextMessages }, options);
  };
}
```
- Normalizes tool call IDs to provider-expected format
- e.g., `[a-zA-Z0-9]{9}` for strict providers

#### Wrapper 4: OpenAI Reasoning Block Downgrade (`attempt.ts:1376-1397`)

```typescript
if (isOpenAIResponsesApi) {
  activeSession.agent.streamFn = (model, context, options) => {
    const sanitized = downgradeOpenAIReasoningBlocks(messages);
    const paired = downgradeOpenAIFunctionCallReasoningPairs(reasoningSanitized);
    return inner(model, { ...context, messages: sanitized }, options);
  };
}
```
- For OpenAI Responses API, strips incompatible reasoning blocks

#### Wrapper 5: Sessions Yield Guard (`attempt.ts:1399-1408`)

```typescript
activeSession.agent.streamFn = (model, context, options) => {
  if (yieldDetected && signal.aborted && signal.reason === "sessions_yield") {
    return createYieldAbortedResponse(model);
  }
  return innerStreamFn(model, context, options);
};
```
- Intercepts abort signal to handle sessions_yield tool invocations

#### Wrapper 6: Malformed Tool Call Sanitization (`attempt.ts:1413-1424`)

```typescript
activeSession.agent.streamFn = wrapStreamFnSanitizeMalformedToolCalls(
  activeSession.agent.streamFn,
  allowedToolNames,
  transcriptPolicy,
);
```
- Fixes tool names with surrounding whitespace (e.g., " read " → "read")

#### Wrapper 7: Tool Name Trimming + Unknown Tool Guard (`attempt.ts:1418-1424`)

```typescript
activeSession.agent.streamFn = wrapStreamFnTrimToolCallNames(
  activeSession.agent.streamFn,
  allowedToolNames,
  { unknownToolThreshold: resolveUnknownToolGuardThreshold(clientToolLoopDetection) },
);
```
- Trims whitespace from tool names
- After N consecutive unknown tool calls, rewrites assistant message to stop

#### Wrapper 8: Anthropic Tool Argument Repair (`attempt.ts:1426-1433`)

```typescript
if (params.model.api === "anthropic-messages" && shouldRepairMalformedAnthropicToolCallArguments(params.provider)) {
  activeSession.agent.streamFn = wrapStreamFnRepairMalformedToolCallArguments(
    activeSession.agent.streamFn,
  );
}
```
- Repairs malformed JSON in tool call arguments

#### Wrapper 9: xAI HTML Entity Decoding (`attempt.ts:1435-1438`)

```typescript
if (resolveToolCallArgumentsEncoding(params.model) === "html-entities") {
  activeSession.agent.streamFn = wrapStreamFnDecodeXaiToolCallArguments(
    activeSession.agent.streamFn,
  );
}
```
- Decodes HTML entities in xAI tool call arguments

#### Wrapper 10: Anthropic Payload Logging (`attempt.ts:1441-1445`)

```typescript
if (anthropicPayloadLogger) {
  activeSession.agent.streamFn = anthropicPayloadLogger.wrapStreamFn(
    activeSession.agent.streamFn,
  );
}
```
- Logs full request/response payloads for debugging

#### Wrapper 11: Sensitive Stop Reason Recovery (`attempt.ts:1449-1451`)

```typescript
activeSession.agent.streamFn = wrapStreamFnHandleSensitiveStopReason(
  activeSession.agent.streamFn,
);
```
- Recovers from provider-specific stop reasons that would otherwise cause uncaught errors

#### Wrapper 12: LLM Idle Timeout (`attempt.ts:1464-1470`)

```typescript
if (idleTimeoutMs > 0) {
  activeSession.agent.streamFn = streamWithIdleTimeout(
    activeSession.agent.streamFn,
    idleTimeoutMs,
    (error) => idleTimeoutTrigger?.(error),
  );
}
```
- Detects when LLM stops responding mid-stream

### 2.4 Provider-Specific Stream Wrappers

#### OpenAI Stream Wrappers (`openai-stream-wrappers.ts`)

| Wrapper | Function | Purpose |
|---------|----------|---------|
| `createOpenAIResponsesContextManagementWrapper` | Line 162-195 | Handle `store`, `prompt_cache` policy |
| `createOpenAIReasoningCompatibilityWrapper` | Line 197-212 | Apply reasoning effort to OpenAI payloads |
| `createOpenAIStringContentWrapper` | Line 214-227 | Flatten completion messages to string content |
| `createOpenAIThinkingLevelWrapper` | Line 229-264 | Apply thinking/reasoning configuration |
| `createOpenAIFastModeWrapper` | Line 266-291 | Apply `fastMode: priority` override |
| `createOpenAIServiceTierWrapper` | Line 293-308 | Apply `service_tier` parameter |
| `createOpenAITextVerbosityWrapper` | Line 310-338 | Apply `text.verbosity` parameter |
| `createCodexNativeWebSearchWrapper` | Line 339-390 | Inject Codex native web search |
| `createOpenAIDefaultTransportWrapper` | Line 400-413 | Set default transport + WS warmup |
| `createOpenAIAttributionHeadersWrapper` | Line 415-437 | Add required attribution headers |

#### Proxy Stream Wrappers (`proxy-stream-wrappers.ts`)

| Wrapper | Function | Purpose |
|---------|----------|---------|
| `createOpenRouterSystemCacheWrapper` | Line 49-78 | Apply Anthropic cache markers for OpenRouter |
| `createOpenRouterWrapper` | Line 80-108 | Normalize reasoning payload for OpenRouter |
| `createKilocodeWrapper` | Line 114-143 | Apply Kilocode-specific headers and reasoning |

### 2.5 How Tools Emit Through the Stream

Tools emit events through the pi-agent-core SDK's `subscribeEmbeddedPiSession()` (`pi-embedded-subscribe.ts`):

```typescript
const subscription = subscribeEmbeddedPiSession(
  buildEmbeddedSubscriptionParams({
    session: activeSession,
    toolResultFormat: params.toolResultFormat,
    shouldEmitToolResult: params.shouldEmitToolResult,
    onToolResult: params.onToolResult,
    // ... callbacks
  }),
);

// Extracted from subscription:
const { assistantTexts, toolMetas, unsubscribe, waitForCompactionRetry } = subscription;
```

The subscription handles:
- Tool call parsing from model response
- Tool execution (via the agent session)
- Tool result collection
- Compaction notifications

### 2.6 How Auth Profile Changes Flow Through

Auth changes are applied **before** the stream wrapper chain via `applyApiKeyInfo()`:

```typescript
// In auth-controller.ts:360-404
async function applyApiKeyInfo(candidate?: string): Promise<void> {
  const apiKeyInfo = await resolveApiKeyInfoForCandidate(candidate);
  params.setApiKeyInfo(apiKeyInfo);
  
  // Prepare runtime auth (OAuth refresh tokens, etc.)
  const preparedAuth = await prepareRuntimeAuthForModel({
    runtimeModel, apiKey: apiKeyInfo.apiKey, authMode, profileId
  });
  
  // Apply auth to runtime model (headers, baseUrl)
  applyPreparedRuntimeRequestOverrides({ runtimeModel, preparedAuth });
  
  // Update auth storage for provider
  authStorage.setRuntimeApiKey(runtimeModel.provider, preparedAuth.apiKey);
  
  // Set runtime auth state for refresh scheduling
  params.setRuntimeAuthState({
    generation: nextRuntimeAuthGeneration(),
    sourceApiKey: apiKeyInfo.apiKey,
    authMode: apiKeyInfo.mode,
    profileId: apiKeyInfo.profileId,
    expiresAt: preparedAuth.expiresAt,
  });
}
```

When auth changes, the runtime model is updated with new headers/baseUrl, so subsequent stream calls use the new credentials automatically.

---

## 3. Agent Host Interface

**Note:** The `embedded-agent-host.ts` file does not exist in this codebase. The agent host functionality is distributed across:

- **`run/attempt.ts`** - Main session management and prompt execution
- **`run/backend.ts`** - Backend harness delegation
- **`pi-embedded-subscribe.ts`** - Session subscription for event streaming

### 3.1 Session Lifecycle

#### Session Creation (`run/attempt.ts:1111-1127`)

```typescript
({ session } = await createAgentSession({
  cwd: resolvedWorkspace,
  agentDir,
  authStorage: params.authStorage,
  modelRegistry: params.modelRegistry,
  model: params.model,
  thinkingLevel: mapThinkingLevel(params.thinkLevel),
  tools: builtInTools,
  customTools: allCustomTools,
  sessionManager,
  settingsManager,
  resourceLoader,
}));
```

#### Session Disposal (`run/attempt.ts:2627-2636`)

```typescript
await cleanupEmbeddedAttemptResources({
  removeToolResultContextGuard,
  flushPendingToolResultsAfterIdle,
  session,
  sessionManager,
  releaseWsSession,
  sessionId: params.sessionId,
  bundleLspRuntime,
  sessionLock,
});
```

### 3.2 Message Handling

Messages flow through the system:

1. **User prompt** arrives at `runEmbeddedPiAgent()`
2. **Hook processing** adds context (`resolvePromptBuildHookResult`)
3. **Prompt submission** via `activeSession.prompt(effectivePrompt)`
4. **Stream events** emitted through wrapper chain
5. **Subscription** (`subscribeEmbeddedPiSession`) processes events
6. **Tool calls** handled internally by agent loop
7. **Final response** collected after model stops

### 3.3 Tool Invocation Flow

```
Model response (stopReason: "tool_calls")
  → subscribeEmbeddedPiSession detects tool calls
  → Tools resolved from registry
  → Tool execution via session.agent.executeTool()
  → Tool results appended to session messages
  → Model called again with tool results
  → Loop continues until stopReason != "tool_calls"
```

---

## 4. Tool System

### 4.1 Tool Registration Pipeline (`run/attempt.ts:487-717`)

```typescript
const toolsRaw = params.disableTools
  ? []
  : createOpenClawCodingTools({
      agentId: sessionAgentId,
      exec: { ...params.execOverrides, elevated: params.bashElevated },
      sandbox,
      messageProvider: params.messageChannel ?? params.messageProvider,
      // ... many context parameters
    });

// Apply toolsAllow filter
const toolsFiltered = applyEmbeddedAttemptToolsAllow(toolsRaw, params.toolsAllow);

// Split into built-in vs custom
const { builtInTools, customTools } = splitSdkTools({
  tools: toolsFiltered,
  sandboxEnabled: !!sandbox?.enabled,
});

// Add MCP tools
const bundleMcpRuntime = await materializeBundleMcpToolsForRun({
  runtime: bundleMcpSessionRuntime,
  reservedToolNames: [...tools.map(t => t.name), ...clientToolNames],
});

// Add LSP tools
const bundleLspRuntime = await createBundleLspToolRuntime({
  workspaceDir: effectiveWorkspace,
  reservedToolNames: [...],
});

// Apply effective tool policy (sandbox constraints, etc.)
const filteredBundledTools = applyFinalEffectiveToolPolicy({
  bundledTools: [...bundleMcpTools, ...bundleLspTools],
  config: params.config,
  sandboxToolPolicy: sandbox?.tools,
  // ...
});

// Final effective tools
const effectiveTools = [...tools, ...filteredBundledTools];
```

### 4.2 Tool Schema Generation (`tool-schema-runtime.ts`)

Provider tool schemas are normalized:

```typescript
const tools = normalizeProviderToolSchemas({
  tools: toolsEnabled ? toolsRaw : [],
  provider: params.provider,
  config: params.config,
  workspaceDir: effectiveWorkspace,
  env: process.env,
  modelId: params.modelId,
  modelApi: params.model.api,
  model: params.model,
});
```

### 4.3 Tool Call Routing

Tool calls route based on name:

1. **Built-in tools** - Core tools like `Bash`, `Read`, `Write`, `Edit`, `Grep`
2. **Custom tools** - Agent-defined tools
3. **MCP tools** - Model Context Protocol tools from plugins
4. **LSP tools** - Language Server Protocol tools
5. **Client tools** - OpenResponses hosted tools

Routing is determined by `splitSdkTools()` which separates built-in vs custom tools.

---

## 5. Model Resolution

### 5.1 Model Selection Flow (`model.ts:658-706`)

```typescript
export function resolveModel(
  provider: string,
  modelId: string,
  agentDir?: string,
  cfg?: OpenClawConfig,
): {
  model?: Model<Api>;
  error?: string;
  authStorage: AuthStorage;
  modelRegistry: ModelRegistry;
} {
  // 1. Normalize provider and model ID
  const normalizedRef = {
    provider: normalizeProviderId(provider),
    model: normalizeStaticProviderModelId(normalizeProviderId(provider), modelId),
  };
  
  // 2. Resolve agent directory
  const resolvedAgentDir = agentDir ?? resolveOpenClawAgentDir();
  
  // 3. Discover auth storage and model registry
  const authStorage = discoverAuthStorage(resolvedAgentDir);
  const modelRegistry = discoverModels(authStorage, resolvedAgentDir);
  
  // 4. Resolve with registry
  const model = resolveModelWithRegistry({
    provider: normalizedRef.provider,
    modelId: normalizedRef.model,
    modelRegistry,
    cfg,
    agentDir: resolvedAgentDir,
  });
  
  // 5. If found, return
  if (model) return { model, authStorage, modelRegistry };
  
  // 6. Otherwise, build helpful error
  return { error: buildUnknownModelError({...}), authStorage, modelRegistry };
}
```

### 5.2 Provider Fallback Chain

Model resolution follows this priority:

1. **Inline provider models** - Models defined in `models.json` inline config
2. **Registry models** - Models discovered from provider plugins
3. **Dynamic models** - Plugin-provided dynamic model resolution
4. **Configured fallback** - Provider config fallback models

```typescript
// From resolveModelWithRegistry (model.ts:368-656)
function resolveModelWithRegistry(params): Model<Api> | undefined {
  // Priority 1: Explicit model with registry
  const explicitModel = resolveExplicitModelWithRegistry(params);
  if (explicitModel?.kind === "resolved") {
    // Compare with plugin dynamic model
    if (shouldCompareProviderRuntimeResolvedModel(...)) {
      const pluginDynamicModel = resolvePluginDynamicModelWithRegistry(params);
      return preferProviderRuntimeResolvedModel({
        explicitModel: explicitModel.model,
        runtimeResolvedModel: pluginDynamicModel,
      });
    }
    return explicitModel.model;
  }
  
  // Priority 2: Plugin dynamic model only
  const pluginDynamicModel = resolvePluginDynamicModelWithRegistry(params);
  if (pluginDynamicModel) return pluginDynamicModel;
  
  // Priority 3: Configured fallback
  return resolveConfiguredFallbackModel(params);
}
```

### 5.3 Auth Profile Selection

Auth profiles are selected via `resolveAuthProfileOrder()`:

```typescript
const profileOrder = shouldPreferExplicitConfigApiKeyAuth(params.config, provider)
  ? []  // Use config API key only
  : resolveAuthProfileOrder({
      cfg: params.config,
      store: authStore,
      provider,
      preferredProfile: preferredProfileId,
    });

// Locked profile takes precedence
const profileCandidates = lockedProfileId
  ? [lockedProfileId]
  : profileOrder.length > 0
    ? profileOrder
    : [undefined];  // Fall back to keychain/default
```

---

## 6. Context Engine Integration

The context engine manages conversation history and prompt cache. See `context-engine-maintenance.ts` for lifecycle.

### 6.1 Bootstrap Context Assembly (`attempt.context-engine-helpers.ts`)

```typescript
const assembled = await assembleAttemptContextEngine({
  contextEngine: params.contextEngine,
  sessionId: params.sessionId,
  sessionKey: params.sessionKey,
  messages: activeSession.messages,
  tokenBudget: params.contextTokenBudget,
  availableTools: new Set(effectiveTools.map((tool) => tool.name)),
  citationsMode: params.config?.memory?.citations,
  modelId: params.modelId,
  ...(params.prompt !== undefined ? { prompt: params.prompt } : {}),
});
```

### 6.2 Post-Turn Finalization (`run/attempt.ts:2375-2410`)

```typescript
await finalizeAttemptContextEngineTurn({
  contextEngine: params.contextEngine,
  promptError: Boolean(promptError),
  aborted,
  yieldAborted,
  sessionIdUsed,
  sessionKey: params.sessionKey,
  sessionFile: params.sessionFile,
  messagesSnapshot,
  prePromptMessageCount,
  tokenBudget: params.contextTokenBudget,
  runtimeContext: afterTurnRuntimeContext,
});
```

---

## 7. Key Files Summary

| File | Lines | Purpose |
|------|-------|---------|
| `run.ts` | ~2641 | Main orchestrator, retry loop, compaction decision |
| `run/attempt.ts` | ~2642 | Single attempt execution, stream wrapper chain |
| `run/backend.ts` | 9 | Backend harness delegation |
| `run/auth-controller.ts` | 527 | Auth profile lifecycle management |
| `run/payloads.ts` | ~500 | Response payload construction |
| `compact.ts` | ~1400 | Session compaction execution |
| `compaction-hooks.ts` | ~309 | Pre/post compaction hooks |
| `model.ts` | ~855 | Model resolution and discovery |
| `stream-resolution.ts` | 128 | Stream function resolution |
| `openai-stream-wrappers.ts` | 438 | OpenAI-specific stream wrappers |
| `proxy-stream-wrappers.ts` | 143 | Proxy provider stream wrappers |
| `tool-result-truncation.ts` | ~700 | Tool result size management |
| `replay-history.ts` | ~500 | Session history sanitization |

---

## 8. Architectural Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    runEmbeddedPiAgent() (run.ts)                    │
│                         ~2641 lines                                  │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Retry Loop (while true)                    │   │
│  │  ┌────────────────────────────────────────────────────────┐  │   │
│  │  │  1. advanceAuthProfile() if needed                     │  │   │
│  │  │  2. runEmbeddedAttemptWithBackend()                     │  │   │
│  │  │  3. Check results:                                     │  │   │
│  │  │     - Context overflow → compact → continue            │  │   │
│  │  │     - Auth error    → rotate profile → continue        │  │   │
│  │  │     - Rate limit    → escalate to fallback → continue   │  │   │
│  │  │     - Max retries   → return error payload              │  │   │
│  │  └────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│               runEmbeddedAttempt() (run/attempt.ts)                  │
│                         ~2642 lines                                  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Phase 1: Session Setup                                      │    │
│  │  - createAgentSession()                                     │    │
│  │  - createOpenClawCodingTools()                             │    │
│  │  - buildEmbeddedSystemPrompt()                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Phase 2: Stream Wrapper Chain (12 layers)                    │    │
│  │  1. cacheTrace.wrapStreamFn()                               │    │
│  │  2. dropThinkingBlocks()                                    │    │
│  │  3. sanitizeReplayToolCallIds()                             │    │
│  │  4. downgradeOpenAIReasoningBlocks()                        │    │
│  │  5. handleSessionsYieldAbort()                              │    │
│  │  6. sanitizeMalformedToolCalls()                            │    │
│  │  7. trimToolCallNames() + unknownToolGuard                  │    │
│  │  8. repairMalformedAnthropicToolArgs()                      │    │
│  │  9. decodeXaiHtmlEntities()                                │    │
│  │  10. anthropicPayloadLogger.wrap()                          │    │
│  │  11. handleSensitiveStopReason()                            │    │
│  │  12. streamWithIdleTimeout()                                │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Phase 3: Prompt Execution                                   │    │
│  │  - resolvePromptBuildHookResult()                           │    │
│  │  - prependBootstrapPromptWarning()                         │    │
│  │  - session.prompt(effectivePrompt)                         │    │
│  │  - subscribeEmbeddedPiSession()                            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Phase 4: Result Collection                                  │    │
│  │  - Extract assistantTexts                                  │    │
│  │  - Collect toolMetas                                       │    │
│  │  - Build usage totals                                      │    │
│  │  - finalizeAttemptContextEngineTurn()                      │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     compact.ts (~1400 lines)                        │
│              (Separate compaction session)                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  1. Create compaction agent session                          │    │
│  │  2. Build summary prompt                                    │    │
│  │  3. Run summary through compact session                      │    │
│  │  4. Truncate session after compaction                        │    │
│  │  5. Run post-compaction hooks                               │    │
│  │  6. Return compaction result                                │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. Key Insights

1. **Stream wrappers are the interceptor layer** - They transform both requests (sanitizing messages) and behavior (handling errors, timeouts). The chain order matters significantly.

2. **Compaction is a separate session** - It doesn't reuse the main session because compaction modifies the message history that the main session depends on.

3. **Auth rotation is profile-based, not key-based** - The system rotates through named auth profiles, each with potentially different credentials, not just API keys.

4. **The retry loop and attempt execution are cleanly separated** - `runEmbeddedPiAgent()` handles retry policy; `runEmbeddedAttempt()` handles single-shot execution.

5. **Tool execution is internal to the agent loop** - Tools execute through the SDK's internal loop; the runner only collects metadata about what tools were called.

6. **Context engine integration is optional** - If `params.contextEngine` is not provided, the system works without it, falling back to session file-based history management.
