# Tool System

> Source files: `src/Tool.ts` (792 lines), `src/tools.ts`, `src/services/tools/toolExecution.ts` (1745 lines),
> `src/services/tools/toolOrchestration.ts` (189 lines), `src/services/tools/StreamingToolExecutor.ts`,
> `src/tools/` (40+ tool directories)

---

## 1. Architecture Overview

```
tools.ts (registration table)
  ├── getAllBaseTools() — Complete tool list
  ├── getTools() — Permission-filtered tool list
  └── assembleToolPool() — Final tool pool with MCP tools

Tool.ts (type definitions)
  ├── Tool interface — Tool type contract
  └── ToolUseContext — Tool execution environment (40+ properties)

services/tools/ (orchestration layer)
  ├── toolOrchestration.ts — Concurrent/serial scheduling
  ├── toolExecution.ts — Single tool execution pipeline
  ├── StreamingToolExecutor.ts — Streaming tool concurrency
  └── toolHooks.ts — Tool lifecycle hooks
```

---

## 2. Tool.ts — Tool Type Interface (792 lines)

### 2.1 Tool Type Core Fields

```typescript
export type Tool = {
  // === Identity ===
  name: string                      // Tool name (sent to API, unique identifier)
  description: string               // Tool description (sent to API)
  inputSchema: z.ZodType            // Zod input schema (runtime validation)

  // === Execution ===
  execute: (
    input: unknown,
    context: ToolUseContext,
  ) => AsyncGenerator<ToolProgress | Message, Message | void>
  // execute is async generator, can yield progress events and intermediate messages

  // === Concurrency Control ===
  isConcurrencySafe: (input: unknown) => boolean
  // Whether given input can execute concurrently with other tools
  // e.g.: FileReadTool always true, BashTool depends on whether command is read-only

  isReadOnly: boolean | ((input: unknown) => boolean)
  // Whether tool is read-only (doesn't modify filesystem)

  // === Result Control ===
  maxResultSizeChars?: number
  // Max tool result characters, results exceeding this are persisted to disk (>20KB)
  // Infinity means no limit (some tools need complete results)

  // === Schema & Description ===
  backfillObservableInput?: (input: unknown) => unknown
  // Backfill observable input (for UI display of streaming input parameters)

  // === Enabled Status ===
  isEnabled: () => boolean
  // Whether tool is available in current environment/config

  // === MCP Info ===
  mcpInfo?: {
    serverName: string
    toolName: string
  }
  // MCP tool's server source info

  // === Activity Description ===
  getActivityDescription?: (input: unknown) => string
  // Returns human-readable activity description (displayed in UI)

  // === Permission ===
  getPermissionDescription?: (input: unknown) => string
  // Operation description in permission prompt

  // === Lazy Loading ===
  isDeferredTool?: boolean
  // Whether deferred tool (loaded on-demand via ToolSearch)
}
```

### 2.2 ToolUseContext — Tool Execution Context (40+ properties)

```typescript
export type ToolUseContext = {
  // === Core Options ===
  options: {
    commands: Command[]                     // Available command list
    debug: boolean                          // Debug mode
    mainLoopModel: string                   // Main loop model
    tools: Tools                            // Available tools list
    verbose: boolean                        // Verbose mode
    thinkingConfig: ThinkingConfig          // Thinking config
    mcpClients: MCPServerConnection[]       // MCP client connections
    mcpResources: Record<string, ServerResource[]>  // MCP resources
    isNonInteractiveSession: boolean        // Non-interactive session
    agentDefinitions: AgentDefinitionsResult // Agent definitions
    maxBudgetUsd?: number                   // Max budget (USD)
    customSystemPrompt?: string             // Custom system prompt
    appendSystemPrompt?: string             // Append system prompt
    querySource?: QuerySource               // Query source override
    refreshTools?: () => Tools              // Tool refresh callback
  }

  // === Controller ===
  abortController: AbortController          // Abort controller

  // === File State ===
  readFileState: FileStateCache             // File read state LRU cache

  // === App State ===
  getAppState(): AppState                   // Get app state
  setAppState(f: (prev: AppState) => AppState): void  // Set app state
  setAppStateForTasks?: (f: (prev: AppState) => AppState) => void  // Task-scoped state

  // === UI Callbacks ===
  handleElicitation?: (...)  => Promise<ElicitResult>   // URL elicitation handling
  setToolJSX?: SetToolJSXFn                             // Set tool JSX
  addNotification?: (notif: Notification) => void       // Add notification
  appendSystemMessage?: (msg: SystemMessage) => void    // Append system message
  sendOSNotification?: (opts: {...}) => void            // OS notification

  // === Memory System ===
  nestedMemoryAttachmentTriggers?: Set<string>          // Nested memory triggers
  loadedNestedMemoryPaths?: Set<string>                 // Loaded nested memory paths
  dynamicSkillDirTriggers?: Set<string>                 // Dynamic skill dir triggers
  discoveredSkillNames?: Set<string>                    // Discovered skill names

  // === Progress Callbacks ===
  setInProgressToolUseIDs: (f: (prev: Set<string>) => Set<string>) => void
  setHasInterruptibleToolInProgress?: (v: boolean) => void
  setResponseLength: (f: (prev: number) => number) => void
  pushApiMetricsEntry?: (ttftMs: number) => void
  setStreamMode?: (mode: SpinnerMode) => void
  onCompactProgress?: (event: CompactProgressEvent) => void
  setSDKStatus?: (status: SDKStatus) => void
  openMessageSelector?: () => void

  // === History & Attribution ===
  updateFileHistoryState: (updater: ...) => void
  updateAttributionState: (updater: ...) => void
  setConversationId?: (id: UUID) => void

  // === Agent Identity ===
  agentId?: AgentId                                     // Sub-agent ID
  agentType?: string                                    // Sub-agent type name

  // === Messages & Limits ===
  requireCanUseTool?: boolean                           // Force permission check
  messages: Message[]                                   // Current message list
  fileReadingLimits?: { maxTokens?: number; maxSizeBytes?: number }
  globLimits?: { maxResults?: number }

  // === Permission Tracking ===
  toolDecisions?: Map<string, {
    source: string; decision: 'accept' | 'reject'; timestamp: number
  }>
  queryTracking?: QueryChainTracking                    // Query chain tracking

  // === Interactive Prompt ===
  requestPrompt?: (sourceName: string, toolInputSummary?: string | null) =>
    (request: PromptRequest) => Promise<PromptResponse>

  // === Tool Identity ===
  toolUseId?: string
  criticalSystemReminder_EXPERIMENTAL?: string

  // === Sub-agent ===
  preserveToolUseResults?: boolean                      // Preserve tool results (team members)
  localDenialTracking?: DenialTrackingState             // Local denial tracking
  contentReplacementState?: ContentReplacementState     // Content replacement state
  renderedSystemPrompt?: SystemPrompt                   // Parent-rendered system prompt
  userModified?: boolean                                // User modified
}
```

---

[← API Client](../04-API客户端/api-client-en.md) | [Index](../README_EN.md) | [Permission & Security →](../06-权限与安全/permission-security-en.md)
