# Query Engine

> Source files: `src/query.ts` (1729 lines), `src/QueryEngine.ts`, `src/query/*.ts`

---

## 1. Architecture Overview

The query engine is the core runtime of Claude Code, responsible for managing the complete cycle of "user input → model invocation → tool execution → result return". It is an **async generator** architecture that pushes streaming events to the caller via `yield` and returns the termination reason via `return`.

```
QueryEngine.ts (SDK/print entry)
  └─→ query.ts::query() (async generator wrapper)
        └─→ queryLoop() (while(true) main loop)
              ├── Phase 1: Context preparation (compression pipeline)
              ├── Phase 2: API call (streaming reception)
              ├── Phase 3: Tool execution (concurrent/serial orchestration)
              ├── Phase 4: Stop hooks (post-turn processing)
              └── Phase 5: Continue/terminate decision
```

---

## 2. query() — Core async generator

### 2.1 File Location and Signature

```typescript
// src/query.ts, line 219
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal  // return type — termination reason
>
```

### 2.2 QueryParams Type — Complete Fields

```typescript
// src/query.ts, line 181
export type QueryParams = {
  messages: Message[]                // Message history array
  systemPrompt: SystemPrompt         // System prompt
  userContext: { [k: string]: string } // User context (key-value injection into prompt)
  systemContext: { [k: string]: string } // System context
  canUseTool: CanUseToolFn           // Permission check function
  toolUseContext: ToolUseContext      // Tool execution context (40+ properties)
  fallbackModel?: string             // Fallback model
  querySource: QuerySource           // Query source identifier
  maxOutputTokensOverride?: number   // Max output tokens override
  maxTurns?: number                  // Max turns limit
  skipCacheWrite?: boolean           // Skip cache write
  taskBudget?: { total: number }     // API task_budget (output_config.task_budget)
  deps?: QueryDeps                   // Dependency injection (for testing)
}
```

### 2.3 State Type — Mutable Loop State

Each loop iteration destructures state to provide bare-name access; continue sites use `state = { ... }` for batch assignment.

```typescript
// src/query.ts, line 204
type State = {
  messages: Message[]                           // Current message list
  toolUseContext: ToolUseContext                 // Current tool context
  autoCompactTracking: AutoCompactTrackingState | undefined  // Compaction tracking
  maxOutputTokensRecoveryCount: number          // max_output_tokens recovery count
  hasAttemptedReactiveCompact: boolean          // Whether reactive compact was attempted
  maxOutputTokensOverride: number | undefined   // Output token override
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined  // Pending summary
  stopHookActive: boolean | undefined           // Whether stop hook is active
  turnCount: number                             // Current turn count
  transition: Continue | undefined              // Continue reason from last iteration
}
```

State initialization:

```typescript
let state: State = {
  messages: params.messages,
  toolUseContext: params.toolUseContext,
  maxOutputTokensOverride: params.maxOutputTokensOverride,
  autoCompactTracking: undefined,
  stopHookActive: undefined,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  turnCount: 1,
  pendingToolUseSummary: undefined,
  transition: undefined,
}
```

### Design Philosophy: Why async generator instead of async function

The return type of `query()` is `AsyncGenerator<StreamEvent | ... , Terminal>`, not a simple `Promise<Terminal>`. This choice affects the entire system architecture for the following reasons:

**Streaming output** — LLMs generate content token by token. An async function can only return after completion, leaving users facing seconds to tens of seconds of blank waiting. The generator's `yield` pushes each `StreamEvent` to the UI in real-time, enabling character-by-character rendering.

**Backpressure control** — The caller consumes the generator via `for await...of`, pulling events at its own pace. If UI rendering bottlenecks slow consumption, the generator naturally pauses at `yield` points, preventing unbounded growth of API response buffers. This is much safer than the EventEmitter pattern (where the pusher doesn't know if the consumer can keep up).

**Mid-stream cancellation** — `generator.return()` can gracefully terminate the loop at any `yield` point. `query.ts` uses `using` declarations (like `using pendingMemoryPrefetch = startRelevantMemoryPrefetch(...)`，line ~301) to ensure resources are automatically disposed on cancellation. This is more fine-grained than `AbortController`: `AbortController` can only cancel network requests, while generators can stop at any stage—tool execution, stop hooks, context compression, etc.

**Multi-type events + type-safe termination** — The generator's `yield` type is `StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage` (intermediate events), and the `return` type is `Terminal` (termination reason). The TypeScript compiler can fully check both type paths, much safer than EventEmitter's string event names + `any` payload.

**Implicit state machine** — The code execution position itself encodes the "current state". The 5 Phases + multiple `continue` sites in the loop correspond to different state transitions, but no explicit state enum or switch-case matrix is needed. See "Why while(true) instead of explicit state machine" below.

### Design Philosophy: Why State is mutable

The `State` type (`src/query.ts:204`) is destructured, modified, and reassigned within the `while(true)` loop body—this is a mutable state pattern, not Redux-style immutability. Reasons:

1. **No concurrency** — `state` is only visible within the `queryLoop` function. Node.js single-threading guarantees it won't be concurrently modified by other code. The core problem immutability solves (preventing race conditions) doesn't exist here.

2. **Boilerplate code** — In the 1729-line loop body, there are 7 `state = { ... }` continue sites (line ~289 comment: "Loop-local (not on State) to avoid touching the 7 continue sites"). Using immutable patterns would require deep copying the entire state object at each site, adding massive boilerplate without actual benefit.

3. **Clear state transition points** — Batch reassignment `state = { ...next }` at continue sites provides clear, greppable state transition markers. Each `continue` is accompanied by `transition: { reason: '...' }` (like `'reactive_compact_retry'`, `'max_output_tokens_escalate'`, `'token_budget_continuation'`), making state transition reasons testable (line ~215 comment: "Lets tests assert recovery paths fired without inspecting message contents").

### Design Philosophy: Why QueryDeps uses dependency injection

`QueryDeps` (`src/query/deps.ts`) only has 4 dependencies (`callModel`, `microcompact`, `autocompact`, `uuid`), which seems modest. But the source code comment (`src/query/deps.ts:9-12`) directly explains the motivation:

> "I/O dependencies for query(). Passing a `deps` override into QueryParams lets tests inject fakes directly instead of spyOn-per-module — the most common mocks (callModel, autocompact) are each spied in 6-8 test files today with module-import-and-spy boilerplate."

This isn't for "architectural purity", but to solve a concrete test maintenance problem: previously `callModel` and `autocompact` were each mocked via `spyOn` in 6-8 test files. Module-level mocking causes test interference (if one test's spy isn't properly restored, it affects subsequent tests). `QueryDeps` passes dependencies via function parameters, allowing each test to create its own fake instance, completely eliminating shared mock state issues.

The comment also mentions: "`Scope is intentionally narrow (4 deps) to prove the pattern.`"—this is currently a minimal viable solution, and future PRs can gradually add dependencies like `runTools`, `handleStopHooks`, etc.

### Design Philosophy: Why 9 termination reasons

The 9 `Terminal` reasons aren't over-engineering—each corresponds to different UI display and subsequent processing paths:

| Termination Reason | UI Behavior Difference |
|----------|------------|
| `completed` | Display result, execute stop hooks (memory extraction, suggestion prompts) |
| `aborted_streaming` | Clean up partial messages, discard pending results in `StreamingToolExecutor` |
| `aborted_tools` | Generate interruption messages for each incomplete `tool_use` block (`yieldMissingToolResultBlocks`) |
| `prompt_too_long` | Trigger compression suggestion, possibly execute `reactiveCompact` |
| `model_error` | Display error panel, call `executeStopFailureHooks` |
| `image_error` | Specific image size/format error prompt |
| `blocking_limit` | Hard limit reminder when auto-compact is OFF |
| `hook_stopped` / `stop_hook_prevented` | Different stop hook blocking modes—former is `preventContinuation`, latter is `blockingErrors` |
| `max_turns` | Includes `turnCount` for SDK caller's budget management |

If simplified to "success/failure/cancel" three types, SDK callers couldn't distinguish "model thinks task is complete" from "forced stop due to token exhaustion"—these two cases require completely different handling strategies in automated workflows.

### Design Philosophy: Why while(true) instead of explicit state machine

`queryLoop` uses `while(true)` + `continue` + `return`, rather than `enum State { PREPARING, CALLING_API, EXECUTING_TOOLS, ... }` + `switch(state)`. For a 1729-line loop body, this is a better choice:

1. **Generator's pause points are implicit state** — When code execution reaches Phase 2 (API call), the state information "currently in API call phase" is already encoded in the program counter. An explicit state enum redundantly expresses information already conveyed by code position.

2. **State transition matrix explosion** — 5 Phases + 7 continue sites mean at least 35 possible state transition combinations. An explicit state machine would need to define the legality of each transition, producing a massive switch-case matrix with far worse readability than linear `if-continue` flow.

3. **Recovery paths are linear** — Recovery paths like `reactive_compact_retry`, `max_output_tokens_escalate`, `fallback` are all just "modify state, jump back to loop head". In linear code, this is a `state = next; continue`—clear and localized. In an explicit state machine, this requires writing transition logic separately for "Phase 2 to Phase 1" and "Phase 4 to Phase 1".

---

## 3. queryLoop() — while(true) Main Loop Structure

### 3.1 Loop Entry

```typescript
// src/query.ts, line 241
async function* queryLoop(
  params: QueryParams,
  consumedCommandUuids: string[],
): AsyncGenerator<...>
```

### 3.2 Immutable Parameter Destructuring

At loop start, immutable parameters are extracted that won't be reassigned throughout the loop lifecycle:

```typescript
const {
  systemPrompt,
  userContext,
  systemContext,
  canUseTool,
  fallbackModel,
  querySource,
  maxTurns,
  skipCacheWrite,
} = params
const deps = params.deps ?? productionDeps()
```

### 3.3 State Destructuring Per Loop Iteration

```typescript
while (true) {
  let { toolUseContext } = state  // toolUseContext can be reassigned within iteration
  const {
    messages,
    autoCompactTracking,
    maxOutputTokensRecoveryCount,
    hasAttemptedReactiveCompact,
    maxOutputTokensOverride,
    pendingToolUseSummary,
    stopHookActive,
    turnCount,
  } = state
  // ... iteration body
}
```

### 3.4 Five-Phase Loop Flow

![Query Engine Loop](../diagrams/query-engine-loop-en.svg)

#### Phase 1: Context Preparation

1. **applyToolResultBudget** — Persist tool results exceeding 20KB to disk
2. **snipCompact** (feature `HISTORY_SNIP`) — History trimming
3. **microcompact** — Micro-compaction (no API call, pure local operation)
4. **contextCollapse** (feature `CONTEXT_COLLAPSE`) — Context collapse
5. **autocompact** — Auto-compaction (may trigger API call to generate summary)
6. Assemble `fullSystemPrompt`
7. Create `StreamingToolExecutor` (if streamingToolExecution gate is enabled)

#### Phase 2: API Call

1. Call `deps.callModel()` (i.e., `queryModelWithStreaming`)
2. Stream receive events, build `assistantMessages[]`
3. Detect `tool_use` block → set `needsFollowUp = true`
4. Handle streaming fallback (FallbackTriggeredError)
5. Handle max_output_tokens recovery (MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3)
6. Handle prompt_too_long / reactiveCompact

#### Phase 3: Tool Execution

1. **Non-streaming path**: `runTools()` → `partitionToolCalls()` → concurrent/serial batches
2. **Streaming path**: `StreamingToolExecutor` → `getCompletedResults()` + `getRemainingResults()`
3. Each tool: `runToolUse()` → permission check → execute → result processing
4. Execute `postSamplingHooks`

#### Phase 4: Stop Hooks

1. If no `needsFollowUp` (model didn't request tool calls), enter stop decision
2. Call `handleStopHooks()` → execute various stop hooks
3. Token Budget check (if enabled)
4. If stop hooks return `blockingErrors` or `preventContinuation`, decide whether to continue

#### Phase 5: Continue/Terminate

1. Tool results appended to message list
2. Get attachment messages (memory, command queue, skill discovery)
3. Check `maxTurns` limit
4. Assemble next `State` object
5. `state = next` → back to `while(true)` head

### 3.5 State Reassignment Sites

There are multiple `state = { ... }` continue sites within the loop, each representing a different continuation reason:

- **next_turn** — Normal tool result follow-up loop (line ~1715)
- **reactive_compact** — Retry after reactive compaction triggered by 413
- **max_output_tokens_recovery** — Recovery retry after output token exhaustion
- **fallback** — Retry with fallback model after streaming degradation
- **prompt_too_long_retry** — Retry after prompt too long error

---

## 4. 9 Termination Reasons (Terminal Reasons)

| Reason | Description | Trigger Condition |
|------|------|----------|
| `completed` | Normal completion | Model didn't request tool calls, stop hooks have no blocking errors |
| `aborted_streaming` | Aborted during streaming | User interrupt (Ctrl+C) during streaming phase |
| `aborted_tools` | Aborted during tool execution | User interrupt (Ctrl+C) during tool execution phase |
| `model_error` | Model error | API returns unrecoverable error |
| `image_error` | Image error | Image size/format error |
| `prompt_too_long` | Prompt too long | 413 error and cannot recover via compression |
| `blocking_limit` | Blocking limit | Hard token limit reached (when auto-compact is OFF) |
| `hook_stopped` | Hook prevented | Stop hook explicitly prevents continuation |
| `stop_hook_prevented` | Stop hook prevented | Stop hook's blockingErrors |
| `max_turns` | Max turns | Reached maxTurns limit |

---

## 5. QueryEngine.ts — SDK/Print Entry

### 5.1 Location and Role

`QueryEngine.ts` is the upper-layer wrapper for `query()`, providing higher-level APIs for SDK and print modes.

### 5.2 QueryEngineConfig

```typescript
// Inferred from QueryEngine.ts constructor parameters
{
  sessionId: SessionId
  model: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  mcpResources: Record<string, ServerResource[]>
  agentDefinitions: AgentDefinitionsResult
  thinkingConfig: ThinkingConfig
  permissionMode: PermissionMode
  // ... more config
}
```

### 5.3 ask() generator

The `ask()` method encapsulates the complete user input to model response cycle:

1. **processUserInput** — Preprocess user input (command detection, attachment processing)
2. **fetchSystemPromptParts** — Assemble system prompt (CLAUDE.md, MCP instructions, agent definitions, etc.)
3. **query()** call — Start core loop
4. **Event dispatch** — Convert generator events to SDK-compatible message format
5. **Usage tracking** — Accumulate API usage (accumulateUsage/updateUsage)
6. **Session recording** — recordTranscript, flushSessionStorage

### 5.4 Key Behaviors

- Automatically creates `AbortController` for cancellation
- Session persistence check (`isSessionPersistenceDisabled`)
- File history snapshot (`fileHistoryMakeSnapshot`)
- Error recovery (`categorizeRetryableAPIError` categorizes then decides retry or terminate)

---

## 6. QueryConfig — Immutable Query Configuration

Frozen once at each `query()` call entry, doesn't change throughout the loop.

```typescript
// src/query/config.ts
export type QueryConfig = {
  sessionId: SessionId

  gates: {
    streamingToolExecution: boolean  // tengu_streaming_tool_execution2 gate
    emitToolUseSummaries: boolean    // CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES env var
    isAnt: boolean                   // USER_TYPE === 'ant'
    fastModeEnabled: boolean         // !CLAUDE_CODE_DISABLE_FAST_MODE
  }
}
```

**Design decision**: Deliberately excludes `feature()` gates (those are compile-time tree-shaking boundaries), only includes runtime-variable statsig/env state. This makes QueryConfig pure data, convenient for future extraction as pure function reducer: `(state, event, config) => state`.

---

## 7. QueryDeps — Dependency Injection

```typescript
// src/query/deps.ts
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming    // Model invocation
  microcompact: typeof microcompactMessages   // Micro-compaction
  autocompact: typeof autoCompactIfNeeded     // Auto-compaction
  uuid: () => string                          // UUID generation
}

// Production factory
export function productionDeps(): QueryDeps {
  return {
    callModel: queryModelWithStreaming,
    microcompact: microcompactMessages,
    autocompact: autoCompactIfNeeded,
    uuid: randomUUID,
  }
}
```

**Design intent**: By passing overrides via `params.deps`, tests can directly inject fake implementations without needing `spyOn` module-level mocks (previously callModel and autocompact were each spied in 6-8 test files).

---

## 8. Token Budget — Token Budget Tracking

### 8.1 File Location

`src/query/tokenBudget.ts` (93 lines)

### 8.2 BudgetTracker Type

```typescript
export type BudgetTracker = {
  continuationCount: number       // Number of continuations
  lastDeltaTokens: number         // Delta tokens from last check
  lastGlobalTurnTokens: number    // Global turn tokens from last check
  startedAt: number               // Start timestamp
}
```

### 8.3 checkTokenBudget Decision Logic

```typescript
export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,  // Sub-agents skip budget check
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision
```

**Decision rules**:

1. **Skip conditions**: `agentId` exists (sub-agent) or `budget` is null/non-positive → return `stop`
2. **Calculate ratio**: `pct = turnTokens / budget * 100`
3. **Calculate delta**: `deltaSinceLastCheck = globalTurnTokens - tracker.lastGlobalTurnTokens`
4. **Diminishing detection**: When `continuationCount >= 3` and consecutive two `delta < 500` (DIMINISHING_THRESHOLD) → `isDiminishing = true`
5. **Continue condition**: Not diminishing and `turnTokens < budget * 0.9` (COMPLETION_THRESHOLD=90%) → `continue`
6. **Stop condition**: Diminishing or already has continuation count → `stop` (with completion event)

### 8.4 Token Budget Decision Type

```typescript
type ContinueDecision = {
  action: 'continue'
  nudgeMessage: string        // Prompt message (percentage, used/total)
  continuationCount: number
  pct: number
  turnTokens: number
  budget: number
}

type StopDecision = {
  action: 'stop'
  completionEvent: {          // null means not participating in budget system
    continuationCount: number
    pct: number
    turnTokens: number
    budget: number
    diminishingReturns: boolean
    durationMs: number
  } | null
}
```

### Design Philosophy: Why diminishing returns detection is needed

`checkTokenBudget()` (`src/query/tokenBudget.ts:45`) when deciding whether to continue the loop, not only checks token usage percentage but also detects "diminishing returns"—when `continuationCount >= 3` and consecutive two iterations have `delta < 500 tokens` (`DIMINISHING_THRESHOLD`), it forces termination.

This mechanism prevents the model from falling into infinite loops: when the model produces less than 500 new tokens in 3 consecutive iterations, it indicates it's repeating itself (e.g., repeatedly modifying then undoing the same code) rather than substantially advancing the task. Without this detection, a model stuck in repetition would continue consuming API quota until Token Budget is completely exhausted, with zero actual output.

Diminishing detection works in tandem with percentage threshold (`COMPLETION_THRESHOLD = 0.9`, i.e., 90%): when normally advancing tasks, the model stops after reaching 90% budget; when abnormally repeating, the model stops early after consecutive low output. The `diminishingReturns: boolean` field in `completionEvent` lets callers distinguish these two stop reasons.

---

## 9. Stop Hooks — Stop Hook Handling

### 9.1 File Location

`src/query/stopHooks.ts`

### 9.2 handleStopHooks() Signature

```typescript
export async function* handleStopHooks(
  messagesForQuery: Message[],
  assistantMessages: AssistantMessage[],
  systemPrompt: SystemPrompt,
  userContext: { [k: string]: string },
  systemContext: { [k: string]: string },
  toolUseContext: ToolUseContext,
  querySource: QuerySource,
  stopHookActive?: boolean,
): AsyncGenerator<..., StopHookResult>
```

### 9.3 Return Type

```typescript
type StopHookResult = {
  blockingErrors: Message[]        // Blocking error messages
  preventContinuation: boolean     // Whether to prevent continuation
}
```

### 9.4 Executed Hooks

`handleStopHooks` executes the following operations in order:

1. **saveCacheSafeParams** — Save cache-safe parameters (main thread queries only)
2. **Template work classification** (feature `TEMPLATES`) — Classify work type
3. **executeStopHooks** — Execute user-configured `stop` event hooks
4. **executeTaskCompletedHooks / executeTeammateIdleHooks** — Task/team hooks
5. **executeExtractMemories** (feature `EXTRACT_MEMORIES`) — Auto-extract memories to CLAUDE.md
6. **executePromptSuggestion** — Generate next step suggestion prompt
7. **executeAutoDream** — Auto-dream (inter-session autonomous tasks)
8. **cleanupComputerUseAfterTurn** — Clean up Computer-Use resources

### 9.5 stopHookActive Flag

When stop hooks return `blockingErrors`, `stopHookActive` is set to `true`, preventing stop hooks from executing again in subsequent loop iterations (avoiding infinite recursion).

### Design Philosophy: Why stop hooks are generators instead of regular functions

The signature of `handleStopHooks()` is `async function*` (`src/query/stopHooks.ts:65`), returning `AsyncGenerator<StreamEvent | ... , StopHookResult>`—just like `query()` itself is a generator. This isn't for style consistency, but because operations executed within stop hooks themselves require streaming communication:

1. **`executeExtractMemories`** — Calls API to extract memories from conversation and write to CLAUDE.md, an async operation that needs to push progress events to UI.
2. **`executePromptSuggestion`** — Calls API to generate next step suggestion prompt.
3. **`executeStopHooks`** — Executes user-configured `stop` event hooks, each hook may be external command execution, needs to push `HookProgress` events via `yield` (containing `toolUseID`, `command`, `promptText`, `src/query/stopHooks.ts:200-214`).
4. **`executeAutoDream`** — Auto-dream background task startup.

If `handleStopHooks` were a regular `async` function, these intermediate progress events couldn't be passed to the caller (`query()`'s generator), and the UI would be completely unresponsive during stop hook execution. Generator nesting (`yield* handleStopHooks(...)`) allows stop hook progress events to transparently bubble up to the outermost consumer.

---

## 10. Query Chain Tracking — Query Chain Tracking

### 10.1 Type Definition

```typescript
// src/Tool.ts, line 90
export type QueryChainTracking = {
  chainId: string   // UUID, remains constant throughout entire user turn (including all tool call rounds)
  depth: number     // Increments with each recursive/sub-agent call
}
```

### 10.2 Initialization and Increment

```typescript
// query.ts, at the head of each loop iteration
const queryTracking = toolUseContext.queryTracking
  ? {
      chainId: toolUseContext.queryTracking.chainId,
      depth: toolUseContext.queryTracking.depth + 1,
    }
  : {
      chainId: deps.uuid(),  // First call generates new UUID
      depth: 0,
    }
```

### 10.3 Usage

- **Telemetry correlation**: `queryChainId` and `queryDepth` are passed to all `logEvent` calls, used to correlate multiple API requests in the same turn in analytics
- **Sub-agent tracking**: AgentTool increments depth when creating sub-agents and passes `queryTracking`
- **Analytics panel**: Aggregating by chainId shows the complete API call chain triggered by a user request

---

## 11. Cross-Turn Persistent State

The following state persists across multiple iterations of the `while(true)` loop:

| State | Storage Location | Purpose |
|------|----------|------|
| `messages` | `State.messages` | Complete message history (including tool results), appended each iteration |
| `toolUseContext` | `State.toolUseContext` | Tool context (including readFileState LRU cache) |
| `autoCompactTracking` | `State.autoCompactTracking` | Compaction state (whether compacted, turn counter, consecutive failures) |
| `taskBudgetRemaining` | Loop-local variable | task_budget remaining amount (accumulated across compaction boundaries) |
| `pendingToolUseSummary` | `State.pendingToolUseSummary` | Async tool summary Promise |
| `budgetTracker` | Loop-local variable | Token budget tracker (continuationCount/lastDelta) |
| `taskBudgetRemaining` | Loop-local variable | Task budget remaining tokens (accumulated across compaction) |

### 11.1 taskBudget Tracking Across Compaction Boundaries

```typescript
// Capture final context window before compaction when compaction occurs
if (params.taskBudget) {
  const preCompactContext =
    finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

---

## 12. Error Recovery Mechanisms

### 12.1 max_output_tokens Recovery

- Limit: `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3` times
- Trigger: Assistant message's `apiError === 'max_output_tokens'`
- Behavior: Increment `maxOutputTokensRecoveryCount`, append assistant message as partial result, continue loop

### 12.2 Reactive Compact (413 Recovery)

- Trigger: prompt_too_long API error (413)
- Limit: Only attempt once per loop iteration (`hasAttemptedReactiveCompact`)
- Behavior: Compact current messages, retry with compacted messages
- Gate: `feature('REACTIVE_COMPACT')`

### 12.3 Streaming Fallback

- Trigger: `FallbackTriggeredError` thrown during streaming
- Behavior: Discard pending results in StreamingToolExecutor, retry with fallbackModel
- Note: Withheld max_output_tokens messages won't leak to SDK callers during recovery loop

### 12.4 Thinking Rules

Three rules documented in code comments ("The rules of thinking"):

1. Messages containing thinking/redacted_thinking blocks must belong to queries with `max_thinking_length > 0`
2. Thinking blocks must not be the last block in a message
3. Thinking blocks must remain unchanged throughout the entire assistant trajectory (including subsequent tool_result and assistant messages)

---

## Engineering Practice Guide

### Debugging the Query Loop

The `while(true)` loop in `query.ts` is the runtime core of the entire system. When debugging, focus on these key points:

1. **Set breakpoints at `state = { ... }` sites** — Each `continue` site is a state transition, the `transition.reason` field tells you why the loop is restarting (like `'reactive_compact_retry'`, `'max_output_tokens_escalate'`, `'fallback'`)
2. **Track State changes** — Set breakpoint at the `const { messages, ... } = state` destructuring at loop head, observe changes in `messages` array length, `turnCount` increment, etc., each iteration
3. **Check `needsFollowUp`** — This flag determines whether the loop continues (set to true when `tool_use` block detected in Phase 3). If loop unexpectedly stops or doesn't stop, check this flag
4. **Use QueryDeps injection for debugging** — Inject custom `callModel`/`autocompact`/`microcompact` via `params.deps`, can intercept and inspect intermediate state without modifying source code

### Adding New Termination Reasons

If you need to add a new `Terminal` reason (e.g., new error type or stop condition):

1. **Add new reason to `Terminal` type** — Modify the `Terminal` type definition in `query.ts`
2. **Add corresponding `return` condition in `queryLoop()`** — Add `return { reason: 'your_new_reason', ... }` in appropriate Phase
3. **Add corresponding display logic in UI** — Different termination reasons have different UI displays (error panel, suggestion prompts, etc.), need to handle new reason in code consuming the generator
4. **Handle in SDK/QueryEngine** — The `ask()` method in `QueryEngine.ts` needs to know how to convert new reason to SDK response
5. **Add tests** — Cover condition paths that trigger the new termination reason

### Adding New Phases

When inserting a new phase in the `while(true)` loop:

1. **Determine position** — The order of the 5 Phases has dependencies: context preparation → API call → tool execution → stop hooks → continue/terminate. New phase should be inserted at logically correct position
2. **Note continue sites** — Inserting a new phase may affect existing 7 `state = { ... }; continue` sites. Ensure new phase won't be skipped at unexpected positions
3. **Maintain generator semantics** — If new phase needs to report progress to caller, use `yield` to push events; if needs to terminate loop, use `return Terminal`
4. **Consider impact on `using` declarations** — `query.ts` uses `using` declarations to manage resources (like `pendingMemoryPrefetch`), ensure new phase doesn't interfere with automatic resource disposal

### Debugging Context Compaction Issues

When model behavior is abnormal and compaction is suspected:

1. **Disable auto-compaction** — Set `CLAUDE_CODE_DISABLE_AUTO_COMPACT=true`, see if problem disappears
2. **Check compaction trigger conditions** — `autocompact` triggers when context approaches window limit (reserves 13K buffer). If frequently triggered, message history may be growing too fast
3. **Check post-compaction messages** — After compaction, `messagesForQuery` is a new array (not a reference to `state.messages`), content may be significantly reduced. If model "forgets" previous context, check if compaction is too aggressive
4. **Check `autoCompactTracking`** — `State.autoCompactTracking` records whether compacted, consecutive failure count, etc., can be used to diagnose compaction behavior

### Token Budget Debugging

When model stops prematurely or delays stopping:

1. **Print `checkTokenBudget` decision data** — Focus on these fields:
   - `pct` — Current token usage percentage (relative to budget)
   - `delta` (i.e., `deltaSinceLastCheck`) — New tokens since last check
   - `continuationCount` — Number of continuations already made
   - `isDiminishing` — Whether diminishing returns detected (`continuationCount >= 3` and consecutive two `delta < 500`)
2. **Check `COMPLETION_THRESHOLD`** — Default 0.9 (90%), stops after reaching this ratio. If need more budget utilization, adjust this value
3. **Check `DIMINISHING_THRESHOLD`** — Default 500 tokens. If model is doing meaningful but low-output work (like repeatedly fine-tuning code), may be misjudged as diminishing

### Common Pitfalls

1. **Don't modify State outside generator** — `State` is only visible within the `queryLoop` function, and only updated via batch assignment at `continue` sites. Modifying State in callbacks or tool execution causes undefined behavior

2. **Don't assume `messagesForQuery` and `state.messages` are the same reference** — After compaction (autocompact/reactiveCompact), a new array is created. If still holding old `messages` reference after compaction, operating on stale data

3. **`StreamingToolExecutor`'s `discard()` must be called before fallback** — During streaming degradation (`FallbackTriggeredError`), must first call `discard()` to discard pending results, otherwise partial results from old model will leak into new model's context

4. **Don't insert long operations between Phase 2 (API call) and Phase 3 (tool execution)** — In this gap, `assistantMessages` are already generated but tools not yet executed, message state at this point is incomplete

5. **`stopHookActive` prevents stop hook recursion** — After stop hooks return `blockingErrors`, `stopHookActive` is set to true, preventing stop hooks from executing again in next loop iteration. If your code depends on stop hook execution, note this flag may cause hooks to be skipped

6. **Token Budget's `completionEvent` may be null** — `StopDecision.completionEvent` being null means query didn't participate in budget system (like sub-agent queries), don't call property access on null value


---

[← Startup & Initialization](../02-启动与初始化/initialization-en.md) | [Index](../README_EN.md) | [API Client →](../04-API客户端/api-client-en.md)
