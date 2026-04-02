# System Overview - Claude Code v2.1.88

> Based on reverse engineering analysis of 1884 TypeScript source files
> Build: 2026-03-30T21:59:52Z

---

## 1. Core Statistics

| Dimension | Count |
|-----------|-------|
| TypeScript source files (.ts/.tsx) | 1884 |
| Built-in tools (Tools) | 40+ |
| React Hooks | 70+ |
| Slash commands | 87+ (101 command directories) |
| Background services (Services) | 13 |
| Built-in skills (Skills) | 17 |
| Hook event types | 13 |
| Permission modes (Permission Modes) | 6 (default/plan/acceptEdits/bypassPermissions/dontAsk/auto) + 1 internal (bubble) |
| API backends (Backends) | 4 (Anthropic/Bedrock/Vertex/Foundry) |
| MCP transport protocols | 4 (stdio/sse/streamable-http/local) |

---

## 2. Source Tree Structure

```
claudecode/sourcecode/src/
├── QueryEngine.ts          # SDK/print mode query engine entry (ask() generator)
├── Task.ts                 # Background task base class definition
├── Tool.ts                 # Tool type interface + ToolUseContext (792 lines)
├── commands.ts             # Command registry + getSlashCommandToolSkills()
├── context.ts              # Global Context factory
├── cost-tracker.ts         # Cost tracking (getModelUsage/getTotalCost)
├── costHook.ts             # Cost change hooks
├── dialogLaunchers.tsx     # Dialog launchers
├── history.ts              # Session history management
├── ink.ts                  # Ink rendering engine entry
├── interactiveHelpers.tsx  # Interactive UI helper components
├── main.tsx                # Application main entry (REPL mode)
├── projectOnboardingState.ts # Project onboarding state
├── query.ts                # Core query loop (1729 lines) — async generator
├── replLauncher.tsx        # REPL launcher
├── setup.ts                # Initialization setup
├── tasks.ts                # Task system entry
├── tools.ts                # Tool registration table (getAllBaseTools/getTools/assembleToolPool)
│
├── assistant/              # Assistant message processing
├── bootstrap/              # Startup bootstrap (state.ts singleton state, growthbook init)
├── bridge/                 # Bridge protocol (IDE bidirectional communication, 33 files)
├── buddy/                  # Companion pet system (PRNG + sprite rendering)
├── cli/                    # CLI entry and argument parsing
├── commands/               # 87+ Slash command implementations (101 subdirectories)
│   ├── add-dir/
│   ├── clear/
│   ├── commit.ts
│   ├── compact/
│   ├── config/
│   ├── ... (101 directories/files total)
│
├── components/             # React/Ink UI component library
├── constants/              # Global constants (betas, oauth, xml tags, querySource)
├── context/                # Context management (notifications, providers)
├── coordinator/            # Coordinator mode (multi-Worker orchestration)
├── entrypoints/            # Multiple entry points (SDK, print, headless, HFI)
├── hooks/                  # React Hooks (70+)
│   ├── useCanUseTool.tsx   # Core permission decision Hook
│   ├── useTextInput.ts     # Text input
│   ├── useVimInput.ts      # Vim mode input
│   ├── useVoice.ts         # Voice input
│   ├── toolPermission/     # Tool permission UI subsystem
│   ├── notifs/             # Notification subsystem
│   └── ... (85+ files)
│
├── ink/                    # Ink rendering engine extensions
├── keybindings/            # Keybinding system (50+ actions, chord support)
├── memdir/                 # Memory directory system (CLAUDE.md reading and management)
├── migrations/             # Data migrations
├── moreright/              # Right panel extensions
├── native-ts/              # Native TypeScript modules (FFI bridging)
├── outputStyles/           # Output style system (Markdown front matter)
├── plugins/                # Plugin system entry
├── query/                  # Query submodules
│   ├── config.ts           # QueryConfig type (sessionId + gates)
│   ├── deps.ts             # QueryDeps dependency injection (callModel/microcompact/autocompact/uuid)
│   ├── stopHooks.ts        # Stop hook handling (handleStopHooks)
│   └── tokenBudget.ts      # Token budget tracking (BudgetTracker)
│
├── remote/                 # Remote sessions (CCR WebSocket)
├── schemas/                # Zod validation schemas
├── screens/                # Full-screen view components
├── server/                 # Embedded server (LSP, Bridge)
├── services/               # Background service layer (13 subsystems)
│   ├── analytics/          # Telemetry analytics (GrowthBook + Statsig + OTel)
│   ├── api/                # API client (client.ts/claude.ts/withRetry.ts/errors.ts/logging.ts)
│   ├── autoDream/          # Auto dream (inter-session autonomous tasks)
│   ├── compact/            # Context compression (micro/auto/reactive/snip)
│   ├── extractMemories/    # Memory extraction service
│   ├── lsp/                # LSP integration (JSON-RPC)
│   ├── mcp/                # MCP protocol implementation (config/transport/auth/lazy loading)
│   ├── oauth/              # OAuth authentication (PKCE flow)
│   ├── plugins/            # Plugin service
│   ├── policyLimits/       # Policy limits
│   ├── remoteManagedSettings/ # Remote managed settings
│   ├── settingsSync/       # Settings sync
│   ├── teamMemorySync/     # Team memory sync
│   ├── tips/               # Tips service
│   ├── tokenEstimation.ts  # Token estimation
│   ├── toolUseSummary/     # Tool use summary generation
│   ├── tools/              # Tool orchestration layer (StreamingToolExecutor/toolExecution/toolOrchestration)
│   ├── AgentSummary/       # Agent summary
│   ├── MagicDocs/          # Magic docs
│   ├── PromptSuggestion/   # Prompt suggestion
│   ├── SessionMemory/      # Session memory
│   └── voice.ts            # Voice service
│
├── skills/                 # Skill system
│   ├── bundled/            # 17 built-in skills
│   │   ├── claudeApi.ts
│   │   ├── claudeApiContent.ts
│   │   ├── claudeInChrome.ts
│   │   ├── debug.ts
│   │   ├── keybindings.ts
│   │   ├── loop.ts
│   │   ├── loremIpsum.ts
│   │   ├── remember.ts
│   │   ├── scheduleRemoteAgents.ts
│   │   ├── simplify.ts
│   │   ├── skillify.ts
│   │   ├── stuck.ts
│   │   ├── updateConfig.ts
│   │   ├── verify.ts
│   │   ├── verifyContent.ts
│   │   ├── batch.ts
│   │   └── index.ts
│   ├── bundledSkills.ts
│   ├── loadSkillsDir.ts
│   └── mcpSkillBuilders.ts
│
├── state/                  # State management (AppState + Zustand store)
├── tasks/                  # Task system implementation
├── tools/                  # Tool implementations (40+ tools)
│   ├── AgentTool/          # Sub-agent tool
│   ├── AskUserQuestionTool/# User interaction tool
│   ├── BashTool/           # Shell execution
│   ├── BriefTool/          # Brief tool
│   ├── ConfigTool/         # Config tool (ant-only)
│   ├── EnterPlanModeTool/  # Enter plan mode
│   ├── EnterWorktreeTool/  # Enter worktree
│   ├── ExitPlanModeTool/   # Exit plan mode
│   ├── ExitWorktreeTool/   # Exit worktree
│   ├── FileEditTool/       # File editing (precise replacement)
│   ├── FileReadTool/       # File reading
│   ├── FileWriteTool/      # File writing
│   ├── GlobTool/           # File pattern search
│   ├── GrepTool/           # Content search (ripgrep)
│   ├── LSPTool/            # LSP tool
│   ├── ListMcpResourcesTool/ # MCP resource list
│   ├── MCPTool/            # MCP tool bridging
│   ├── McpAuthTool/        # MCP authentication
│   ├── NotebookEditTool/   # Notebook editing
│   ├── PowerShellTool/     # PowerShell (Windows)
│   ├── REPLTool/           # REPL tool (ant-only)
│   ├── ReadMcpResourceTool/# MCP resource reading
│   ├── RemoteTriggerTool/  # Remote trigger
│   ├── ScheduleCronTool/   # Cron scheduling (Create/Delete/List)
│   ├── SendMessageTool/    # Message sending
│   ├── SkillTool/          # Skill execution
│   ├── SleepTool/          # Sleep tool
│   ├── SyntheticOutputTool/# Synthetic output
│   ├── TaskCreateTool/     # Task creation
│   ├── TaskGetTool/        # Task query
│   ├── TaskListTool/       # Task list
│   ├── TaskOutputTool/     # Task output
│   ├── TaskStopTool/       # Task stop
│   ├── TaskUpdateTool/     # Task update
│   ├── TeamCreateTool/     # Team creation
│   ├── TeamDeleteTool/     # Team deletion
│   ├── TodoWriteTool/      # Todo writing
│   ├── ToolSearchTool/     # Tool search (lazy loading support)
│   ├── WebFetchTool/       # Web fetching
│   ├── WebSearchTool/      # Web search
│   ├── shared/             # Shared tool infrastructure
│   ├── testing/            # Testing tools
│   └── utils.ts            # Tool utility functions
│
├── types/                  # Type definitions
│   ├── message.ts          # Complete message types
│   ├── permissions.ts      # Permission types (PermissionMode/Rule/Behavior)
│   ├── hooks.ts            # Hook types
│   ├── tools.ts            # Tool progress types
│   ├── ids.ts              # ID types (AgentId/SessionId)
│   └── utils.ts            # Utility types (DeepImmutable)
│
├── upstreamproxy/          # Upstream proxy
├── utils/                  # Utility function library (largest subdirectory)
│   ├── permissions/        # Permission implementation (24 files)
│   ├── hooks/              # Hook utility functions
│   ├── model/              # Model selection and routing
│   ├── memory/             # Memory management
│   ├── settings/           # Settings loading
│   ├── shell/              # Shell utilities
│   ├── sandbox/            # Sandbox system
│   ├── telemetry/          # Telemetry utilities
│   ├── messages.ts         # Message construction and normalization
│   ├── tokens.ts           # Token counting
│   ├── context.ts          # Context window calculation
│   ├── config.ts           # Configuration management
│   └── ... (100+ files)
│
└── vim/                    # Vim mode implementation (complete state machine)
└── voice/                  # Voice system
```

---

## 3. Layered Architecture

![6-Layer Architecture](../diagrams/layered-architecture-en.svg)

### Design Philosophy: Why 6-Layer Architecture Instead of MVC

Traditional CLI tools typically use MVC or a simple Controller→Service two-layer model. Claude Code's 6 layers (UI→Hooks→State→Query→Services→Tools) may seem over-engineered, but each layer exists due to specific engineering constraints:

1. **UI and Hooks Separation** — The UI layer is pure rendering (React/Ink components), while the Hooks layer encapsulates side effects and state logic. This allows 70+ hooks to be composed and reused by different UI components, rather than embedding logic in the component tree. Evidence: `hooks/useCanUseTool.tsx` is called by three completely different scenarios: permission request UI, tool execution flow, and auto mode classifier.

2. **State and Query Separation** — The State layer (`bootstrap/state.ts` global singleton + Zustand store) manages process-level lifecycle state; the Query layer (`query.ts` async generator) manages transient state for a single conversation turn. If merged, the lifecycles of process-level state (like `totalCostUSD`, `sessionId`) and turn-level state (like `messages`, `turnCount`) would be confused, leading to state leaks.

3. **Services and Tools Separation** — Services are stateless capability providers (API clients, compression algorithms, MCP protocol), while Tools are identity-bearing execution units (with names, descriptions, permission requirements). Separation allows the same Service (like `services/api/claude.ts`) to be called directly by the Query engine or indirectly by Tools, without Tools needing to understand API details.

Core insight: This isn't layering for layering's sake, but a need for **lifecycle management**. The 6 layers correspond to 3 different lifecycles—process-level (State/Infrastructure), session-level (UI/Hooks), turn-level (Query/Services/Tools). MVC only distinguishes "presentation" and "logic", unable to express this multi-level lifecycle.

### Design Philosophy: Why Use React/Ink for CLI

Claude Code's terminal output isn't a traditional linear text stream—it has multiple dynamically updating regions simultaneously:

- **Message stream area** — Model responses stream token-by-token (`components/Messages.tsx`)
- **Tool progress area** — Real-time status of concurrent tool execution (`components/Spinner.tsx`, tool progress events)
- **Input box area** — Permission confirmation dialogs may pop up during tool execution (`components/PromptInput/`)
- **Status bar area** — Token usage, cost, model info continuously update (`components/StatusLine.tsx`, `components/Stats.tsx`)
- **Full-screen overlays** — Settings, context visualization, session recovery full-screen views (`screens/`)

If implemented with traditional `console.log` + ANSI escape sequences, developers would need to manually track each region's line position, handle overlapping refreshes, manage cursor state—essentially reinventing a UI framework. React's declarative model encapsulates this complexity in the reconciliation algorithm: each component only declares "what I should look like now", and the Ink engine automatically calculates minimal terminal updates.

Evidence: `src/components/` contains 50+ component files, `src/screens/` contains full-screen view components, `src/hooks/` contains 70+ React Hooks—this scale of UI complexity would be a disaster to maintain with imperative methods. Ink reduces this problem to familiar React component development for frontend engineers.

---

## 4. Module Dependency Graph

### 4.1 Core Dependency Chain

![Core Dependency Chain](../diagrams/core-dependency-chain-en.svg)

### 4.2 Tool System Dependencies

```
tools.ts (registration table)
  ├─→ Tool.ts (Tool type interface + ToolUseContext)
  ├─→ tools/AgentTool/      ← Create sub-agent, recursively call query
  ├─→ tools/BashTool/       ← Execute shell commands
  ├─→ tools/SkillTool/      ← Execute skills (fork agent)
  ├─→ tools/FileEditTool/   ← Precise file editing
  ├─→ tools/FileReadTool/   ← Read files
  ├─→ tools/FileWriteTool/  ← Write files
  ├─→ tools/GlobTool/       ← File search
  ├─→ tools/GrepTool/       ← Content search
  ├─→ tools/MCPTool/        ← MCP tool bridging
  ├─→ tools/WebFetchTool/   ← Web fetching
  ├─→ tools/WebSearchTool/  ← Web search
  └─→ tools/ToolSearchTool/ ← Tool lazy discovery
```

### 4.3 Permission System Dependencies

```
hooks/useCanUseTool.tsx (permission decision entry)
  └─→ utils/permissions/permissions.ts (canUseTool pipeline)
        ├─→ utils/permissions/PermissionRule.ts (rule types)
        ├─→ utils/permissions/PermissionMode.ts (mode definitions)
        ├─→ utils/permissions/yoloClassifier.ts (auto mode classifier)
        ├─→ utils/permissions/bashClassifier.ts (Bash command classification)
        ├─→ utils/permissions/pathValidation.ts (path safety)
        ├─→ utils/permissions/dangerousPatterns.ts (dangerous pattern detection)
        ├─→ utils/permissions/shellRuleMatching.ts (shell rule matching)
        └─→ utils/sandbox/sandbox-adapter.ts (sandbox execution)
```

### 4.4 Service Layer Internal Dependencies

```
services/
  ├─→ api/ ←── query.ts, QueryEngine.ts, services/compact/
  │     client.ts ← claude.ts ← withRetry.ts
  │     errors.ts ← claude.ts, withRetry.ts, query.ts
  │     logging.ts ← claude.ts
  │
  ├─→ compact/ ←── query.ts
  │     microCompact.ts ← autoCompact.ts ← compact.ts
  │     autoCompact.ts → api/claude.ts (getMaxOutputTokensForModel)
  │     compact.ts → api/claude.ts (queryModelWithStreaming)
  │
  ├─→ tools/ ←── query.ts
  │     toolOrchestration.ts → toolExecution.ts → Tool.ts
  │     StreamingToolExecutor.ts → toolExecution.ts
  │     toolHooks.ts → utils/hooks.ts
  │
  ├─→ mcp/ ←── tools/MCPTool, tools.ts (assembleToolPool)
  ├─→ analytics/ ←── Almost all modules (logEvent global calls)
  └─→ oauth/ ←── services/api/client.ts, utils/auth.ts
```

### 4.5 Cross-Layer Critical Paths

| Path | Flow |
|------|------|
| User input → Model response | `useTextInput` → `processUserInput` → `query()` → `claude.ts` → API |
| Tool execution | `query()` → `toolOrchestration` → `toolExecution` → `canUseTool` → tool.execute() |
| Context compression | `query()` → `microcompact` → `autocompact` → API (or `reactiveCompact` on 413) |
| Permission decision | `toolExecution` → `canUseTool` → rules → classifier → user prompt |
| MCP bridging | `tools.ts` → `assembleToolPool` → MCP clients → MCPTool.execute() |
| Skill execution | `SkillTool` → `runForkedAgent` → new `query()` instance |
| Sub-agent | `AgentTool` → `createSubagentContext` → new `query()` instance |

---

## 5. Entry Point Matrix

| Entry Point | File | Purpose |
|-------------|------|---------|
| REPL (interactive) | `main.tsx` → `replLauncher.tsx` | Terminal interactive session |
| Print (non-interactive) | `entrypoints/print/` | Single query then exit |
| SDK | `entrypoints/sdk/` → `QueryEngine.ts` | Programmatic API |
| Headless | `entrypoints/headless/` | No UI background running |
| HFI (Human-Friendly Interface) | `entrypoints/hfi/` | Web-friendly interface |
| Bridge | `bridge/` | IDE bidirectional communication |
| CLI | `cli/` | Command-line argument parsing |

---

## 6. Build and Runtime Features

### 6.1 Feature Flags (Compile-time)

Uses `feature('FLAG_NAME')` for compile-time feature gating (Bun bundler tree-shaking), with disabled code paths completely removed at build time:

- `REACTIVE_COMPACT` — Reactive compression (413 triggered)
- `CONTEXT_COLLAPSE` — Context collapse
- `HISTORY_SNIP` — History snipping
- `TOKEN_BUDGET` — Token budget
- `EXTRACT_MEMORIES` — Memory extraction
- `TEMPLATES` — Templates/work classification
- `EXPERIMENTAL_SKILL_SEARCH` — Skill search
- `TRANSCRIPT_CLASSIFIER` — Transcript classifier (auto mode)
- `COORDINATOR_MODE` — Coordinator mode
- `BASH_CLASSIFIER` — Bash command classifier (ant-only)
- `CACHED_MICROCOMPACT` — Cached microcompact
- `BG_SESSIONS` — Background sessions
- `PROACTIVE` / `KAIROS` — Proactive agent
- `AGENT_TRIGGERS` / `AGENT_TRIGGERS_REMOTE` — Agent triggers
- `MONITOR_TOOL` — Monitor tool
- `OVERFLOW_TEST_TOOL` — Overflow test
- `TERMINAL_PANEL` — Terminal panel
- `WEB_BROWSER_TOOL` — Web browser tool
- `UDS_INBOX` — Unix Domain Socket inbox
- `WORKFLOW_SCRIPTS` — Workflow scripts

#### Why Feature Flags Use Compile-time Tree-shaking

Claude Code's `feature('FLAG_NAME')` isn't a runtime `if (config.featureEnabled('FLAG_NAME'))` check—it's a compile-time macro from Bun bundler (`from 'bun:bundle'`), with disabled code paths completely removed at build time. The entire codebase has 196 files using `feature()` calls.

This choice makes a clear trade-off between security and flexibility:

**Security (core advantage of compile-time removal):**
- Code doesn't exist = can't be exploited. For example, `BASH_CLASSIFIER` (ant-only) if only runtime-checked, reverse engineering could still find the classifier logic; after compile-time removal, this code physically doesn't exist in external builds.
- Reduced attack surface: Debug tools like `OVERFLOW_TEST_TOOL`, `MONITOR_TOOL` are completely removed in production builds, impossible to activate via environment variable injection.

**Cost of flexibility:**
- Changing feature flags requires rebuild and release—can't remotely toggle in real-time like LaunchDarkly.
- This is why `QueryConfig` (`src/query/config.ts`) deliberately excludes `feature()` gating, only including runtime-variable statsig/env state: compile-time and runtime gating are two independent systems.

**Implementation details:** `feature()` calls can only appear in `if` conditions or ternary expressions (`src/query.ts:796` comment: "feature() only works in if/ternary conditions (bun:bundle...)"), ensuring the bundler can correctly identify and remove dead code branches. Conditional `require()` patterns (like `const reactiveCompact = feature('REACTIVE_COMPACT') ? require(...) : null`, `src/query.ts:15-17`) exclude entire module dependency trees from the build.

### 6.2 Environment Variable Gating

- `USER_TYPE=ant` — Anthropic internal employee features
- `CLAUDE_CODE_SIMPLE=true` — Simplified mode (Bash/Read/Edit only)
- `CLAUDE_CODE_DISABLE_FAST_MODE` — Disable fast mode
- `NODE_ENV=test` — Test environment (enables TestingPermissionTool)
- `CLAUDE_CODE_VERIFY_PLAN=true` — Plan verification tool

---

## 7. Data Flow Overview

```
User Input
    │
    ▼
processUserInput() ─── Command detection ──→ Slash Command handling
    │
    ▼
query() async generator (while true loop)
    │
    ├─→ Phase 1: Context preparation
    │     ├── applyToolResultBudget (>20KB tool results persisted to disk)
    │     ├── snipCompact (history snipping)
    │     ├── microcompact (tool result compression, COMPACTABLE_TOOLS)
    │     ├── contextCollapse (context collapse)
    │     └── autoCompact (auto compression, 13K buffer)
    │
    ├─→ Phase 2: API call
    │     ├── System prompt assembly (systemPrompt + userContext + systemContext)
    │     ├── queryModelWithStreaming → getAnthropicClient
    │     └── Streaming reception (streaming events → messages)
    │
    ├─→ Phase 3: Tool execution
    │     ├── partitionToolCalls (concurrent vs serial batching)
    │     ├── runToolUse (permission → execute → result)
    │     └── StreamingToolExecutor (streaming tool concurrency)
    │
    ├─→ Phase 4: Stop hooks
    │     ├── executeStopHooks
    │     ├── executeExtractMemories
    │     ├── executePromptSuggestion
    │     ├── executeAutoDream
    │     └── cleanupComputerUseAfterTurn
    │
    └─→ Phase 5: Continue/terminate decision
          ├── needsFollowUp → Continue loop (tool_use blocks exist)
          ├── Token Budget → Continue or stop
          └── One of 9 termination reasons → return Terminal
```

### Design Philosophy: Why the Core is an Async Generator

The `query()` function (`src/query.ts:219`) is declared as `async function*`—this isn't an arbitrary syntax choice, but the cornerstone of the entire system's architectural style. Async generators simultaneously solve four core problems:

1. **Streaming push** — LLMs generate content token-by-token, generators push `StreamEvent`, `Message` and other events to callers via `yield`. Callers can render in real-time rather than waiting for complete responses. This allows REPL mode, SDK mode, and Headless mode to use the same generator but consume differently.

2. **Backpressure control** — Callers consume events at their own pace via `for await...of`. If UI rendering is slower than API reception speed, the generator naturally pauses at `yield` points, preventing memory overflow. This is much safer than EventEmitter patterns (`on('data')` callbacks).

3. **Type-safe polymorphic returns** — The generator's `yield` type (`StreamEvent | Message | ...`) and `return` type (`Terminal`) are separate. `yield` pushes intermediate events, `return` is only for termination reasons. This is more type-safe than EventEmitter's string event names, with TypeScript compiler fully checking all event handling paths.

4. **Graceful cancellation** — Callers can immediately terminate the loop via `generator.return()`, with `using` declarations inside the generator (like `pendingMemoryPrefetch`) automatically disposed. This is finer-grained than `AbortController`—`AbortController` can only cancel fetch requests, generators can stop at any `yield` point.

This choice has far-reaching ripple effects: because the core is a generator, stop hooks (`stopHooks.ts`) are also designed as generators (need to yield progress events to UI); `QueryEngine.ask()` is also a generator; even sub-agents (`AgentTool`) run through nested generators. The entire system forms a pipeline architecture of generator composition.

### Design Philosophy: Why 7 Entry Points Instead of One Unified Entry

The 7 entry points (REPL/Print/SDK/Headless/HFI/Bridge/CLI) seem like they could be unified into one general entry, but they solve fundamentally different deployment scenarios:

| Entry Point | Core Difference | Why Not Unified |
|-------------|-----------------|-----------------|
| **REPL** (`main.tsx`) | Complete Ink rendering + interactive input loop | Needs React component tree, keybinding system, Vim mode |
| **Print** (`entrypoints/print/`) | Single query then exit | No UI loop, output to stdout/file, needs `gracefulShutdown` |
| **SDK** (`entrypoints/sdk/`) | Programmatic API | No CLI argument parsing, returns structured data not terminal output |
| **Headless** (`entrypoints/headless/`) | No UI background running | No terminal dependencies, suitable for CI/CD |
| **HFI** (`entrypoints/hfi/`) | Web-friendly interface | HTTP protocol, JSON serialization not terminal rendering |
| **Bridge** (`bridge/`, 33 files) | IDE bidirectional communication | LSP protocol, needs to maintain long connection + bidirectional messages |
| **CLI** (`cli/`) | Command-line argument parsing | Commander.js configuration, is the routing layer for other entries |

The cost of a unified entry would be massive conditional branching—each entry has completely different needs for I/O models (interactive/batch/streaming/bidirectional), lifecycle management (resident/single/on-demand), output formats (terminal/JSON/LSP). Separate entries let each scenario load only the modules it needs, while sharing the lower Query engine and Services layers.

Evidence: Line 1 of `src/main.tsx` has import side effects (`profileCheckpoint`/`startMdmRawRead`/`startKeychainPrefetch`) that are REPL-specific startup optimizations, which SDK entry doesn't need and shouldn't execute. The `CLAUDE_CODE_ENTRYPOINT` environment variable (`src/interactiveHelpers.tsx`) has 10 values precisely to distinguish call sources in shared code.

---

## Engineering Practice Guide

### Checklist for Adding New Subsystems

If you need to add a completely new subsystem to Claude Code (e.g., new tool category, new service module), follow these steps:

1. **Create tool directory in `tools/`** — e.g., `tools/MyNewTool/`, implement `Tool` interface (see `src/Tool.ts`)
2. **Register in `tools.ts`** — Add tool reference in appropriate place in `getAllBaseTools()`; if feature flag gating needed, use conditional spread `...(feature('MY_FLAG') ? [MyNewTool] : [])`
3. **Add permission rules** — If tool involves filesystem or network operations, add corresponding permission check logic in `utils/permissions/`; update `dangerousPatterns.ts` (if needed)
4. **Add tests** — Tool tests, permission tests, integration tests
5. **Update documentation** — Document design decisions for new subsystem in architecture docs

**Key checkpoints**:
- New tools must implement `isEnabled()` — returns false when tool won't appear in tool pool
- New tools must implement `isConcurrencySafe()` — if unsure, conservatively return false
- If new tool needs UI interaction, use `ToolUseContext` callbacks (like `setToolJSX`, `addNotification`), don't directly import UI modules

### Cross-Layer Debugging Tips

| Debugging Scenario | Action |
|---------------------|--------|
| **View complete log chain** | Start with `--debug`, all layer logs output to stderr |
| **Force serial execution** | Set `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY=1`, eliminate concurrency-induced non-determinism, locate concurrency bugs |
| **Track complete API call chain** | Check `dumpPrompts` output (enable in `services/api/dumpPrompts.ts`), view complete messages sent to API |
| **Trace query loop state changes** | Set breakpoints at `state = { ... }` continue sites in `query.ts`, focus on `transition.reason` field |
| **Confirm feature flag status** | `feature()` is a compile-time macro—if functionality "disappeared", check if corresponding flag is included in build config, not runtime environment variables |

### Performance Analysis Entry Points

- **Perfetto trace**: If OTel tracing is enabled (after `initializeTelemetryAfterTrust()`), can view complete call chain via Perfetto, including API latency, tool execution time, compression duration
- **Startup performance checkpoints**: `profileCheckpoint()` instrumentation in `startupProfiler.ts` covers complete startup chain from `main_tsx_entry` to `REPL` rendering
- **FPS Metrics**: UI rendering performance monitored via `FpsMetrics`, focus on Ink rendering engine frame rate drops
- **Token estimation**: `services/tokenEstimation.ts` provides token counting for diagnosing context window usage efficiency

### Common Architecture Pitfalls

1. **Don't directly access UI layer in Tools layer** — Tools communicate with UI via `ToolUseContext` callbacks (`setToolJSX`, `addNotification`, `sendOSNotification`). Directly importing UI components breaks Headless/SDK mode compatibility (these modes have no React runtime).

2. **Don't assume interactive environment in Services layer** — Services may run in headless mode, SDK mode, or CI environments. Check `isNonInteractiveSession` when user input needed; check `shouldAvoidPermissionPrompts` when UI feedback needed.

3. **Don't confuse compile-time flags and runtime flags** — `feature('FLAG_NAME')` is Bun bundler's compile-time macro, immutable after build; `QueryConfig.gates` statsig/env gating is runtime-variable. Checking flags at wrong level causes "clearly set environment variable but feature doesn't work" confusion.

4. **Don't access state before global singleton initialization completes** — Calling `getBootstrapState()` in `bootstrap/state.ts` before `init()` completes gets undefined or initial values. Ensure your code executes after the `init()` chain.

5. **Don't bypass query loop to directly call API** — `queryModelWithStreaming()` in `services/api/claude.ts` needs to coordinate with retry/fallback/cooldown logic in `withRetry.ts`. Directly calling SDK skips all error recovery mechanisms.


---

[Index](../README_EN.md) | [Startup & Initialization →](../02-启动与初始化/initialization-en.md)
