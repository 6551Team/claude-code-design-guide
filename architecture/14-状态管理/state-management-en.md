# State Management

> Claude Code v2.1.88's dual-layer state architecture: Bootstrap State (global singleton) and AppState (Zustand-like Store), along with React Context providers.

---

## 1. Bootstrap State (src/bootstrap/state.ts)

Global singleton state that persists throughout the entire process lifecycle. The module header includes a warning: **"DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE"**.

### 1.1 State Type — Complete Field List

#### Directory and Session

| Field | Type | Description |
|---|---|---|
| `originalCwd` | `string` | Original working directory |
| `projectRoot` | `string` | Stable project root directory (set at startup, doesn't change with EnterWorktreeTool) |
| `cwd` | `string` | Current working directory |
| `sessionId` | `SessionId` | Current session ID (`randomUUID()`) |
| `parentSessionId` | `SessionId \| undefined` | Parent session ID (plan mode → implementation lineage tracking) |
| `sessionProjectDir` | `string \| null` | Directory containing session `.jsonl` file |

#### Cost and Usage

| Field | Type | Description |
|---|---|---|
| `totalCostUSD` | `number` | Cumulative cost (USD) |
| `totalAPIDuration` | `number` | Cumulative API duration |
| `totalAPIDurationWithoutRetries` | `number` | API duration excluding retries |
| `totalToolDuration` | `number` | Cumulative tool execution duration |
| `totalLinesAdded` | `number` | Cumulative lines added |
| `totalLinesRemoved` | `number` | Cumulative lines removed |
| `hasUnknownModelCost` | `boolean` | Whether there are costs from unknown models |
| `modelUsage` | `{ [modelName: string]: ModelUsage }` | Usage breakdown by model |

#### Performance Metrics

| Field | Type | Description |
|---|---|---|
| `startTime` | `number` | Process start time |
| `lastInteractionTime` | `number` | Last interaction time |
| `turnHookDurationMs` | `number` | Hook duration for current turn |
| `turnToolDurationMs` | `number` | Tool duration for current turn |
| `turnClassifierDurationMs` | `number` | Classifier duration for current turn |
| `turnToolCount` | `number` | Tool call count for current turn |
| `turnHookCount` | `number` | Hook call count for current turn |
| `turnClassifierCount` | `number` | Classifier call count for current turn |
| `slowOperations` | `Array<{ operation, durationMs, timestamp }>` | Slow operation tracking (dev bar) |

#### Authentication and Security

| Field | Type | Description |
|---|---|---|
| `sessionIngressToken` | `string \| null \| undefined` | Session ingress authentication token |
| `oauthTokenFromFd` | `string \| null \| undefined` | OAuth token read from FD |
| `apiKeyFromFd` | `string \| null \| undefined` | API key read from FD |
| `sessionBypassPermissionsMode` | `boolean` | Session-level bypass permissions mode flag |
| `sessionTrustAccepted` | `boolean` | Session-level trust flag (home directory scenario, not persisted) |

#### Telemetry

| Field | Type | Description |
|---|---|---|
| `meter` | `Meter \| null` | OpenTelemetry Meter |
| `sessionCounter` | `AttributedCounter \| null` | Session counter |
| `locCounter` | `AttributedCounter \| null` | Lines of code counter |
| `prCounter` | `AttributedCounter \| null` | PR counter |
| `commitCounter` | `AttributedCounter \| null` | Commit counter |
| `costCounter` | `AttributedCounter \| null` | Cost counter |
| `tokenCounter` | `AttributedCounter \| null` | Token counter |
| `codeEditToolDecisionCounter` | `AttributedCounter \| null` | Edit tool decision counter |
| `activeTimeCounter` | `AttributedCounter \| null` | Active time counter |
| `statsStore` | `{ observe(name, value): void } \| null` | Statistics store |
| `loggerProvider` | `LoggerProvider \| null` | Logger provider |
| `eventLogger` | `ReturnType<typeof logs.getLogger> \| null` | Event logger |
| `meterProvider` | `MeterProvider \| null` | Meter provider |
| `tracerProvider` | `BasicTracerProvider \| null` | Tracer provider |
| `promptId` | `string \| null` | UUID of current prompt |

#### Hooks

| Field | Type | Description |
|---|---|---|
| `registeredHooks` | `Partial<Record<HookEvent, RegisteredHookMatcher[]>> \| null` | Registered hooks |

#### Coordinator and Agent

| Field | Type | Description |
|---|---|---|
| `agentColorMap` | `Map<string, AgentColorName>` | Agent color assignment mapping |
| `agentColorIndex` | `number` | Next available color index |
| `mainThreadAgentType` | `string \| undefined` | Main thread agent type |

#### Settings and Runtime

| Field | Type | Description |
|---|---|---|
| `mainLoopModelOverride` | `ModelSetting \| undefined` | Main loop model override |
| `initialMainLoopModel` | `ModelSetting` | Initial main loop model |
| `modelStrings` | `ModelStrings \| null` | Model display strings |
| `isInteractive` | `boolean` | Whether in interactive mode |
| `kairosActive` | `boolean` | Whether Kairos assistant mode is active |
| `strictToolResultPairing` | `boolean` | Strict tool result pairing (HFI mode) |
| `userMsgOptIn` | `boolean` | User message opt-in |
| `clientType` | `string` | Client type |
| `sessionSource` | `string \| undefined` | Session source |
| `flagSettingsPath` | `string \| undefined` | --settings flag path |
| `flagSettingsInline` | `Record<string, unknown> \| null` | Inline settings |
| `allowedSettingSources` | `SettingSource[]` | List of allowed setting sources |

#### Beta Features and Cache

| Field | Type | Description |
|---|---|---|
| `sdkBetas` | `string[] \| undefined` | Betas provided by SDK |
| `promptCache1hAllowlist` | `string[] \| null` | 1h cache TTL allowlist |
| `promptCache1hEligible` | `boolean \| null` | 1h cache eligibility (session-stable, locked after first evaluation) |
| `afkModeHeaderLatched` | `boolean \| null` | AFK mode header latch flag |
| `fastModeHeaderLatched` | `boolean \| null` | Fast mode header latch flag |
| `cacheEditingHeaderLatched` | `boolean \| null` | Cache editing header latch flag |
| `thinkingClearLatched` | `boolean \| null` | Thinking clear latch flag |
| `pendingPostCompaction` | `boolean` | Post-compaction flag (marks first post-compaction API call) |
| `lastApiCompletionTimestamp` | `number \| null` | Last API completion timestamp |
| `lastMainRequestId` | `string \| undefined` | Last main request ID |

#### Skills and Memories

| Field | Type | Description |
|---|---|---|
| `invokedSkills` | `Map<string, { skillName, skillPath, content, invokedAt, agentId }>` | Invoked skills tracking |
| `teleportedSessionInfo` | `{ isTeleported, hasLoggedFirstMessage, sessionId } \| null` | Teleported session info |
| `planSlugCache` | `Map<string, string>` | Plan slug cache |

#### Permissions and Session

| Field | Type | Description |
|---|---|---|
| `hasExitedPlanMode` | `boolean` | Whether plan mode has been exited |
| `needsPlanModeExitAttachment` | `boolean` | Plan mode exit attachment flag |
| `needsAutoModeExitAttachment` | `boolean` | Auto mode exit attachment flag |
| `sessionPersistenceDisabled` | `boolean` | Session persistence disabled flag |
| `scheduledTasksEnabled` | `boolean` | Scheduled tasks enabled flag |
| `sessionCronTasks` | `SessionCronTask[]` | Session-level cron tasks (non-persistent) |
| `sessionCreatedTeams` | `Set<string>` | Teams created in session (cleaned up during gracefulShutdown) |

### Design Philosophy

#### Why Bootstrap Singleton + Zustand Dual-Track?

Bootstrap State stores process initialization state that is determined at startup and remains unchanged or changes infrequently throughout the lifecycle (`sessionId`, `apiKey`, `totalCostUSD`, feature flags, telemetry providers), while AppState (Zustand-like Store) stores runtime high-frequency UI state (message list, tool progress, permission mode, spinner text). The source code `state.ts` header explicitly warns **"DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE"** — because global mutable singletons can cause problems in concurrent scenarios. The benefit of separation is: initialization logic (bootstrap) and runtime rendering logic (React component tree) don't entangle with each other, and non-React code (tool execution, API calls) can directly read/write Bootstrap State without traversing the component tree.

#### Why Not Redux?

Zustand is more lightweight and doesn't require action/reducer boilerplate. The `createStore` implementation in the source is only about 20 lines — directly `setState(updater)` with reference equality skipping. For scenarios with 40+ tools concurrently updating state, Zustand's direct `set()` is more efficient than `dispatch(action) → reducer → newState`, and is also easier to use in non-React code (`store.getState() / store.setState()`). Redux's middleware and devtools have no value in a CLI scenario.

#### Why Does AppState Have So Many Fields?

Claude Code is a "rich client" — simultaneously managing conversations (messages), tool execution (tasks), file state, permissions (toolPermissionContext), UI modes (expandedView, footerSelection), performance metrics (speculation), and remote connections (replBridge*). These states are interconnected (e.g., permission mode affects tool availability, task state affects UI layout), and placing them in the same Store wrapped with `DeepImmutable<>` to ensure immutability makes it easier to guarantee consistency compared to splitting them into multiple independent stores.

### 1.2 getInitialState()

```typescript
function getInitialState(): State {
  let resolvedCwd = ''
  // Resolve symlinks to match the behavior of shell.ts setCwd
  const rawCwd = cwd()
  resolvedCwd = realpathSync(rawCwd).normalize('NFC')

  return {
    originalCwd: resolvedCwd,
    projectRoot: resolvedCwd,
    totalCostUSD: 0,
    sessionId: randomUUID() as SessionId,
    isInteractive: false,
    clientType: 'cli',
    allowedSettingSources: ['userSettings', 'projectSettings', 'localSettings', 'flagSettings', 'policySettings'],
    // ... all fields initialized to zero values/null/empty collections
  }
}
```

### 1.3 getSessionId() / regenerateSessionId()

```typescript
export function getSessionId(): SessionId {
  return state.sessionId
}

export function regenerateSessionId(): void {
  state.sessionId = randomUUID() as SessionId
}

export function switchSession(newSessionId: SessionId): void {
  state.sessionId = newSessionId
}
```

---

## 2. AppState (src/state/)

React-driven UI state managed through a custom Zustand-like Store implementation.

### 2.1 Store Implementation (src/state/store.ts)

Minimalist reactive Store:

```typescript
type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: (updater) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // Skip if reference equal
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    }
  }
}
```

### 2.2 AppStateStore Shape (src/state/AppStateStore.ts)

`AppState` is wrapped with `DeepImmutable<>` to ensure immutability. Core fields grouped:

#### UI State

```typescript
{
  verbose: boolean
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  isBriefOnly: boolean
  showTeammateMessagePreview?: boolean    // Gated by ENABLE_AGENT_SWARMS
  selectedIPAgentIndex: number
  coordinatorTaskIndex: number
  viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
  footerSelection: FooterItem | null      // 'tasks' | 'tmux' | 'bagel' | 'teams' | 'bridge' | 'companion'
  spinnerTip?: string
  agent: string | undefined
  kairosEnabled: boolean
}
```

#### Model and Permissions

```typescript
{
  settings: SettingsJson
  mainLoopModel: ModelSetting
  mainLoopModelForSession: ModelSetting
  toolPermissionContext: ToolPermissionContext
}
```

#### Tasks

```typescript
{
  tasks: Map<string, TaskState>
}
```

#### Messages and History

```typescript
{
  messages: Message[]
  // ...(message-related fields managed by the query engine)
}
```

#### Configuration

```typescript
{
  settings: SettingsJson
  replBridgeEnabled: boolean
  replBridgeExplicit: boolean
  replBridgeOutboundOnly: boolean
  replBridgeConnected: boolean
  replBridgeSessionActive: boolean
  replBridgeReconnecting: boolean
  replBridgeConnectUrl: string | undefined
  replBridgeSessionUrl: string | undefined
  replBridgeEnvironmentId: string | undefined
}
```

#### Agent / Teammate

```typescript
{
  remoteSessionUrl: string | undefined
  remoteConnectionStatus: 'connecting' | 'connected' | 'reconnecting' | 'disconnected'
  remoteBackgroundTaskCount: number
}
```

#### Speculative Execution

```typescript
type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      startTime: number
      messagesRef: { current: Message[] }
      writtenPathsRef: { current: Set<string> }
      boundary: CompletionBoundary | null
      suggestionLength: number
      toolUseCount: number
      isPipelined: boolean
      contextRef: { current: REPLHookContext }
      pipelinedSuggestion?: { text, promptId, generationRequestId } | null
    }
```

### 2.3 AppStateProvider (src/state/AppState.tsx)

React Context Provider that prevents nesting:

```typescript
export function AppStateProvider({ children, initialState, onChangeAppState }) {
  const hasAppStateContext = useContext(HasAppStateContext)
  if (hasAppStateContext) {
    throw new Error("AppStateProvider can not be nested within another AppStateProvider")
  }
  const [store] = useState(() => createStore(initialState ?? getDefaultAppState(), onChangeAppState))
  // ...
}

export const AppStoreContext = React.createContext<AppStateStore | null>(null)
```

### 2.4 onChangeAppState Callback

`src/state/onChangeAppState.ts` — Registers callbacks on AppState changes, used for:

- Syncing state changes to Bootstrap State
- Triggering side effects (e.g., settings change notifications)
- State persistence

### 2.5 Selectors

`src/state/selectors.ts` — Selector functions that extract derived data from AppState, avoiding unnecessary re-renders.

---

## 3. React Context Providers (src/context/)

### Core Context List

| Context | File | Description |
|---|---|---|
| **NotificationsContext** | notifications.tsx | Notification management (agent completion, errors, etc.) |
| **StatsContext** | stats.tsx | Statistics data (StatsStore: observe/name/value) |
| **ModalContext** | modalContext.tsx | Modal dialog management (open/close/stacking) |
| **OverlayContext** | overlayContext.tsx | Overlay management (fullscreen overlay) |
| **PromptOverlayContext** | promptOverlayContext.tsx | Input area overlay |
| **QueuedMessageContext** | QueuedMessageContext.tsx | Queued message management |
| **VoiceContext** | voice.tsx | Voice input/output (ant-only, gated by `feature('VOICE_MODE')`) |
| **MailboxContext** | mailbox.tsx | Teammate mailbox (message send/receive) |
| **FpsMetricsContext** | fpsMetrics.tsx | Frame rate metrics |

### VoiceProvider Conditional Loading

```typescript
const VoiceProvider = feature('VOICE_MODE')
  ? require('../context/voice.js').VoiceProvider
  : ({ children }) => children  // External builds use pass-through
```

### Notification Type

```typescript
type Notification = {
  // Agent completion notifications, error notifications, system notifications, etc.
  // Enqueued and displayed in the REPL message stream
}
```

### StatsStore

```typescript
type StatsStore = {
  observe(name: string, value: number): void
}
```

Created via `createStatsStore()`, injected into Bootstrap State via `setStatsStore()`. Initialized in `interactiveHelpers.tsx`.

---

## 4. State Flow Overview

![State Flow Overview](../diagrams/state-flow-overview-en.svg)

---

## Engineering Practice Guide

### Adding New AppState Fields

**Checklist:**

1. **Add field to AppState type**: Edit `src/state/AppStateStore.ts`, add the new field to the AppState type definition (wrap with `DeepImmutable<>` to ensure immutability)
2. **Initialize in getDefaultAppState**: Provide a reasonable default value (zero value/null/empty collection)
3. **Use in related hooks**:
   ```typescript
   // Read
   const myField = useAppState(s => s.myField)
   // Update (via setState functional update)
   store.setState(prev => ({ ...prev, myField: newValue }))
   ```
4. **Add selector** (optional): Add selector functions in `src/state/selectors.ts` to extract derived data and avoid unnecessary re-renders
5. **Sync to Bootstrap State** (if needed): Register callback in `onChangeAppState.ts`

### Debugging State Issues

1. **Check current value of Zustand store**: View current AppState via `store.getState()` in non-React code
2. **Note the difference between Bootstrap State and AppState**:
   | Dimension | Bootstrap State | AppState |
   |------|----------------|----------|
   | Implementation | Global mutable singleton | Zustand-like Store + DeepImmutable |
   | Lifecycle | Process-level | React component tree level |
   | Change frequency | Low-frequency / at initialization | High-frequency (messages, tool progress, permissions, etc.) |
   | Access method | `getXxx()` / `setXxx()` functions | `store.getState()` / `store.setState()` |
   | In non-React code | Direct access | Access via store instance |
3. **Check if listeners are triggered correctly**: `store.setState()` uses `Object.is()` reference equality check — if the same reference is returned, listeners won't trigger
4. **Check onChangeAppState side effects**: Callbacks registered in `onChangeAppState.ts` execute on every AppState change; errors in callbacks will affect state synchronization

### Best Practices for State Updates

1. **Use functional updates**:
   ```typescript
   // Correct: functional update, based on latest prev state
   store.setState(prev => ({ ...prev, counter: prev.counter + 1 }))

   // Wrong: passing value directly may overwrite concurrent updates
   const current = store.getState()
   store.setState(_ => ({ ...current, counter: current.counter + 1 }))
   ```

2. **Use Selectors to avoid unnecessary re-renders**:
   ```typescript
   // Correct: subscribe only to needed fields
   const messages = useAppState(s => s.messages)

   // Not recommended: subscribing to entire state triggers re-render on any field change
   const state = useAppState(s => s)
   ```

3. **State access in non-React code**:
   - Bootstrap State: Use exported functions directly (`getSessionId()`, `getTotalCostUSD()`, etc.)
   - AppState: Read via `store.getState()`, update via `store.setState()`

### Bootstrap State Field Classification

Bootstrap State fields are categorized by purpose as follows (confirm whether new state belongs in Bootstrap scope before adding):

| Category | Example Fields | Characteristics |
|------|---------|------|
| Directory and Session | `sessionId`, `cwd`, `projectRoot` | Determined at process startup |
| Cost and Usage | `totalCostUSD`, `modelUsage` | Accumulated over full lifecycle |
| Authentication and Security | `sessionIngressToken`, `oauthTokenFromFd` | Infrequently changed after initialization |
| Telemetry | `meter`, `statsStore`, `eventLogger` | Provider instances |
| Settings and Runtime | `mainLoopModelOverride`, `isInteractive` | Configuration level |

**Decision criteria**: If state rarely changes after process initialization and needs to be accessed in non-React code, put it in Bootstrap State; if state changes frequently and primarily drives UI rendering, put it in AppState.

### Common Pitfalls

> **Never directly modify the state object — must use set functions**
> AppState is wrapped with `DeepImmutable<>` to ensure immutability. Directly modifying the state object (e.g., `state.messages.push(msg)`) won't trigger listeners and violates the immutability contract. Always use `store.setState(prev => ...)` to create a new object.

> **Bootstrap State should be modified with caution after initialization**
> The source code `state.ts` header explicitly warns **"DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE"**. Global mutable singletons can cause problems in concurrent scenarios (multiple agents reading and writing simultaneously). Only use Bootstrap State when you truly need process-level global state.

> **AppStateProvider cannot be nested**
> In source code `AppState.tsx`, `AppStateProvider` detects nesting and throws an error: `"AppStateProvider can not be nested within another AppStateProvider"`. This is an intentional design — ensuring there is only one AppState store instance globally.

> **TODO: Alternative to DeepImmutable**
> The TODO comment at `AppStateStore.ts:172` mentions considering using `utility-types`'s `DeepReadonly` to replace the current `DeepImmutable` implementation — type definitions may change in the future.

> **store.subscribe returns an unsubscribe function**
> `store.subscribe(listener)` returns an unsubscribe function. When used in React components, be sure to call unsubscribe in the cleanup phase to avoid memory leaks and ghost updates.



---

[← Config System](../13-配置体系/config-system-en.md) | [Index](../README_EN.md) | [Command System →](../15-命令体系/command-system-en.md)
