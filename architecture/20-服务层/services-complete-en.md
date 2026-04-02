# Services Layer - Complete Reference

The Services Layer of Claude Code contains 13 independent services, each responsible for specific background functionality. This document provides a comprehensive description of each service, including its purpose, key function signatures, thresholds/constants, gating conditions, error handling, and state management.

### Design Philosophy

Why 13 independent services rather than merging them into fewer modules? Because each service has its own independent lifecycle and resource requirements — LSP needs to maintain long-lived connections and manage multiple subprocesses, OAuth needs token refresh and keychain interaction, Analytics needs an async event queue, and autoDream needs a PID lock and cross-session state. Merging them would create unnecessary coupling: a crash or restart of one service should not affect others. Additionally, these services have different startup timing — some (like LSP) are started on demand when a code file is first opened, some (like policyLimits) start immediately when the application launches, and some (like autoDream) only run when the session is idle. Independent services allow each module to autonomously manage its own lifecycle.

---

## 1. AgentSummary

### Purpose

Periodically summarizes the current session in the background, providing high-quality conversation summaries for subagents and context compression.

### Core Mechanism

- **30-second cycle**: Executes summary generation via a forked subprocess every 30 seconds
- **Cache sharing**: Summary results are shared through a caching system to avoid redundant computation. Subagents can directly read the main session's summary cache
- **Tool denial for cache key matching**: When a subagent's tool call pattern matches an existing cache key, the system rejects duplicate execution and directly returns the cached summary result

### State Management

A background timer manages the lifecycle of summary tasks, cleaning up the timer and cache when the session ends.

---

## 2. MagicDocs

### Purpose

Automatically maintains and updates Markdown documents in a specific format, supporting automatic generation and refresh of document content via subagents.

### Core Mechanism

- **File identification**: Identifies target documents through a `# MAGIC DOC` header at the top of the file
- **Pattern matching**: Uses pattern regex to match file paths or content patterns that need processing
- **Subagent generation**: Executes document generation/update tasks through a forked subagent
- **Custom prompts**: Supports loading custom prompt templates from `~/.claude/magic-docs/prompt.md`
- **Variable substitution**: Document templates support variable substitution, injecting runtime information into generated documents

### Key Functions

```typescript
// Check if a file is a MagicDoc
isMagicDoc(content: string): boolean
// Trigger a MagicDoc update
updateMagicDoc(filePath: string, context: Context): Promise<void>
```

### Error Handling

Subagent execution failures do not affect the main session; errors are logged but not propagated.

---

## 3. PromptSuggestion

### Purpose

Predicts the user's next input and prepares responses in advance through speculative execution, reducing perceived latency.

### Gating Conditions

- **Minimum conversation turns**: At least **2 assistant turns** are required before prediction begins (`MIN 2 assistant turns`)
- **Maximum uncached tokens**: The number of uncached tokens in the parent context must not exceed `MAX_PARENT_UNCACHED_TOKENS = 10000`

### Rejection Filters

Prediction results are filtered through the following filters; filtered-out predictions are not used:

| Filter Category | Description |
|-----------------|-------------|
| **done** | Predicted content implies the conversation has ended (e.g., "Thank you", "Got it") |
| **meta-text** | Predicted content is meta-text about the conversation itself |
| **evaluative** | Predicted content is evaluative (e.g., "Well done", "That's wrong") |
| **Claude-voice** | Predicted content uses Claude's voice rather than the user's voice |

### Speculation Sandbox

Speculative execution runs within a restricted **speculation sandbox**:

- **Copy-on-Write**: Write operations do not affect actual state
- **Maximum turns**: Speculative execution runs at most **20 turns**
- **Read-only Bash**: The Bash tool runs in read-only mode within the sandbox, not executing any commands with side effects

#### Design Rationale

The core concept of PromptSuggestion is "speculative execution" — predicting the next operation and running it in advance before the user sees the result, making interactions feel more fluid. But speculation is inherently a gamble: predictions may be right or wrong, so it cannot be done in the main loop (which would increase latency) and must be placed in the background and isolated in a Copy-on-Write sandbox. The source file `speculation.ts` stores speculative state in a temporary directory (`~/.claude/tmp/speculation/<pid>/<id>`), completely isolated from the main session. When the user's actual input arrives, the system compares it against the prediction — if it matches, the result is reused directly (saving wait time); if not, it is silently discarded. The introduction of `checkReadOnlyConstraints` ensures that the Bash tool does not produce real side effects during speculation.

### State Management

Maintains the currently active speculative execution state. When the user's actual input arrives: if it matches the prediction, the result is reused; otherwise, the speculative state is discarded.

---

## 4. SessionMemory

### Purpose

Extracts key information from the current session and saves it as Markdown-formatted session notes.

### Core Mechanism

- Extracts structured Markdown session notes from conversation content
- Output file: `.session-memory.md`

### Thresholds

| Parameter | Description |
|-----------|-------------|
| `minimumMessageTokensToInit` | Minimum message token count required to trigger the first extraction |
| `minimumTokensBetweenUpdate` | Minimum incremental token count required between two updates |

### Key Functions

```typescript
// Initialize session memory extraction
initSessionMemory(messages: Message[]): Promise<void>
// Update session memory
updateSessionMemory(messages: Message[]): Promise<void>
```

### Error Handling

Extraction failures degrade silently without affecting the main session flow.

---

## 5. autoDream

### Purpose

A background memory consolidation service that periodically consolidates fragmented memory pieces into structured long-term memory.

### Gate Order

autoDream execution must pass through the following gate checks in sequence:

1. **Time gate**: Whether sufficient time has passed since the last consolidation
2. **Session scan gate**: Whether there are enough new memory fragments in the current session that need consolidation
3. **Lock gate**: A PID-based distributed lock ensuring only one autoDream instance runs at a time

### PID Lock Mechanism

- Uses a **PID-based lock** to prevent concurrent execution
- Lock staleness timeout: **60 minutes** — if the lock-holding process has not released the lock within 60 minutes, it is considered a stale lock and can be taken over by a new process

### 4-Phase Consolidation Prompt

The consolidation process uses a **4-phase consolidation prompt**, executing in sequence:

1. Review existing memory structure
2. Identify newly added memory fragments
3. Merge and deduplicate
4. Generate the consolidated memory document

#### Design Rationale

autoDream is a "between-session" task — using idle API quota when the user is inactive to do valuable work (organizing memories, merging and deduplicating), without interfering with active sessions. The source code comments clearly describe its gate order: *"Gate order (cheapest first): 1. Time 2. Sessions 3. Lock"* (`autoDream.ts`) — doing the cheap time check first, then scanning session count, and only then acquiring the PID lock. This layered gating avoids unnecessary resource consumption. The 60-minute timeout on the PID lock prevents crashed processes from permanently holding the lock. The `SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000` scan throttle further avoids invalid repeated scans when the time gate passes but the session gate does not.

### State Management

Lock state and last consolidation timestamp are persisted through the file system.

---

## 6. extractMemories

### Purpose

Extracts memory fragments from conversations in the background and asynchronously writes them to memory storage.

### Core Mechanism

- **Background extraction**: Runs asynchronously without affecting the main session
- **Coalescing mechanism**: Uses a pending stash to temporarily store memory fragments awaiting processing, then batch-merges them before writing
- **Throttle control**: Controls extraction frequency through the `tengu_bramble_lintel` dynamic configuration
- **Mutual exclusion writes**: Mutually exclusive with the main agent's memory write operations, preventing data conflicts from concurrent writes

### 4-Category Memory Taxonomy

| Type | Description |
|------|-------------|
| Type 1 | User preferences and habits |
| Type 2 | Project context and tech stack |
| Type 3 | Workflows and processes |
| Type 4 | Facts and knowledge |

### Key Functions

```typescript
// Trigger memory extraction
extractMemories(messages: Message[], context: Context): Promise<void>
// Flush the pending stash
flushPendingMemories(): Promise<void>
```

### Error Handling

On extraction failure, pending fragments are kept in the stash and retried on the next attempt.

---

## 7. LSP (Language Server Protocol)

### Purpose

Provides Language Server Protocol integration to support code intelligence features (auto-completion, diagnostics, go-to-definition, etc.).

### LSPClient

A JSON-RPC protocol encapsulation layer responsible for message serialization/deserialization and request-response matching between Claude Code and LSP servers.

### LSPServerManager

A multi-instance routing manager that routes requests to the corresponding LSP server instance based on file extension:

```typescript
// Get the corresponding LSP server for a given extension
getServerForExtension(ext: string): LSPServer
```

### Key Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| Initialization timeout | **30 seconds** | Maximum wait time for LSP server startup |

### File Tracking

Tracks opened files through the `openedFiles` Map:

```typescript
// Tracking table for opened files
openedFiles: Map<string, TextDocument>
```

When a file is opened, modified, or closed, the corresponding LSP server is notified synchronously.

### Error Handling

When an LSP server times out on startup or crashes, related features degrade silently without affecting basic editing capabilities.

---

## 8. OAuth

### Purpose

Handles the OAuth authentication flow, allowing users to log in to Claude Code through their Anthropic account.

### Core Mechanism

- **PKCE flow**: Uses Proof Key for Code Exchange (PKCE) to enhance security
- **Dual authentication paths**:
  - **Manual path**: The user manually copies the authorization URL to a browser and pastes back the authorization code
  - **Automatic path**: Automatically opens the browser and receives the authorization code via a local callback server

### AuthCodeListener

A local localhost HTTP server that listens for OAuth callbacks:

```typescript
// Start the local callback server
startAuthCodeListener(port: number): Promise<AuthCode>
```

### Profile Fetch

After successful authentication, fetches the user's profile information:

- **subscription**: Subscription status and plan
- **rateLimitTier**: Rate limit tier

### Error Handling

- Automatically tries other ports when the callback server port is occupied
- Cleans up temporary server resources after authentication timeout
- Falls back to the re-login flow when token refresh fails

---

## 9. Plugins

### Purpose

Manages plugin scope, discovery, and configuration.

### Scope Management

Plugins are managed according to the following scope hierarchy:

| Scope | Description |
|-------|-------------|
| `user` | User-level plugins, effective globally |
| `project` | Project-level plugins, effective only in specific projects |
| `local` | Local development plugins |
| `managed` | Organization-managed plugins |

### Key Functions

```typescript
// Find a plugin in settings
findPluginInSettings(pluginId: string, settings: Settings): Plugin | undefined
```

### V2 Data Fallback

Supports V2 format plugin data, automatically falling back to the V2 data format when the new format is unavailable, ensuring backward compatibility.

### State Management

Plugin state is stored in user and project-level configuration, with support for hot reloading.

---

## 10. policyLimits

### Purpose

Enforces organization-level policy restrictions, controlling the range of operations users can perform.

### Core Mechanism

- **Organization-level restrictions**: Retrieves policy configuration from the organization management endpoint
- **ETag HTTP caching**: Uses the HTTP ETag mechanism to cache policy data, reducing unnecessary network requests
- **Polling cycle**: Polls for policy updates in the background every **1 hour**

### Failure Policy

Uses a **fail-open** policy: when the policy service is unreachable, operations are allowed by default, ensuring that users' work is not blocked due to policy service failures.

**Exception**: `ESSENTIAL_TRAFFIC_DENY_ON_MISS` mode (for **HIPAA** compliance scenarios) — when the policy is unreachable, operations are **denied**, ensuring that unauthorized operations are not permitted in environments with strict compliance requirements.

### Key Functions

```typescript
// Check whether an operation is allowed by policy
checkPolicyLimit(action: string, context: PolicyContext): PolicyResult
// Refresh the policy cache
refreshPolicyLimits(): Promise<void>
```

### Error Handling

- Network errors: fail-open (except in HIPAA mode)
- Parse errors: use the last valid policy cache
- ETag match: use local cache directly when a 304 response is returned

---

## 11. remoteManagedSettings

### Purpose

Manages configuration remotely delivered by the organization, ensuring that organization policies are enforced on the client side.

### Core Mechanism

- **Organization-level settings**: Pulls settings configured by organization administrators from a remote service
- **Security check**: The `checkManagedSettingsSecurity()` function detects dangerous settings changes. When potentially dangerous changes are detected (such as disabling security features, modifying critical paths, etc.), it presents a confirmation prompt (dangerous change prompt) to the user
- **Background polling**: Polls for settings updates in the background every **1 hour**

### Key Functions

```typescript
// Check the security of remote settings changes
checkManagedSettingsSecurity(
  oldSettings: ManagedSettings,
  newSettings: ManagedSettings
): SecurityCheckResult

// Get the current remotely managed settings
getManagedSettings(): Promise<ManagedSettings>
```

### Error Handling

When remote settings retrieval fails, the last valid locally cached settings are used, and retries continue in the background.

---

## 12. settingsSync

### Purpose

Bidirectionally synchronizes user settings and memory data across multiple devices.

### Core Mechanism

- **Bidirectional Push/Pull**: Supports synchronization in both upload (push) and download (pull) directions
- **Sync scope**: `SYNC_KEYS` defines the data categories to be synchronized:
  - **Settings**: User settings
  - **Memory**: Memory data
  - **Project-keyed data**: Data keyed by project, using a **git hash** as the project identifier key

### Limits

| Parameter | Value | Description |
|-----------|-------|-------------|
| Maximum upload size | **500KB** | Maximum amount of data per sync upload |

### Incremental Upload

Supports **incremental upload**, uploading only data that has changed since the last sync, reducing network transfer.

### Key Functions

```typescript
// Push local settings to the remote
pushSettings(keys: SyncKey[]): Promise<void>
// Pull settings from the remote
pullSettings(keys: SyncKey[]): Promise<void>
// Perform bidirectional synchronization
syncSettings(): Promise<SyncResult>
```

### Error Handling

- Conflict resolution: remote data takes precedence (pull wins)
- When the size limit is exceeded, uploads are split or truncated
- On network failure, changes are staged and await the next sync

---

## 13. teamMemorySync / secretScanner

### Purpose

A team memory synchronization service with a built-in secret scanner to prevent sensitive information from leaking into shared memory.

### Secret Scanner

#### Rule Set

Built-in **30 gitleaks rules** covering the following secret types (sourced from the public [gitleaks configuration](https://github.com/gitleaks/gitleaks/blob/master/config/gitleaks.toml), MIT license):

| Category | Included Secret Types |
|----------|-----------------------|
| **AWS** | Access Key ID, Secret Access Key, Session Token, etc. |
| **GCP** | Service Account Key, API Key, etc. |
| **Azure** | Storage Account Key, Client Secret, etc. |
| **GitHub** | Personal Access Token, OAuth Token, App Private Key, etc. |
| **Slack** | Bot Token, Webhook URL, App Token, etc. |
| **Stripe** | Secret Key, Publishable Key, Webhook Secret, etc. |
| **Private Keys** | RSA, DSA, EC, PGP private keys |

#### Scan Functions

```typescript
// Scan text for secrets, returning matching rule IDs (not the actual secret values)
scanForSecrets(text: string): SecretScanResult[]

interface SecretScanResult {
  ruleId: string;    // The matching rule ID
  // Note: does not contain the actual secret value, preventing secondary exposure
}
```

**Security design**: `scanForSecrets()` returns matching **rule IDs** rather than actual secret values, preventing secrets from being secondarily exposed in scan results.

#### Redaction Function

```typescript
// Redact secrets found in text
redactSecrets(text: string): string
```

Replaces detected secrets with placeholder values.

### teamMemSecretGuard

```typescript
// Prevent writing content containing secrets to synchronized memory
teamMemSecretGuard(content: string): GuardResult
```

Acts as a write guard, performing secret scanning before memory content is written to team synchronized storage. If secrets are detected, **writing is blocked** and the relevant rule ID information is returned.

#### Design Rationale

Team memory is synchronized among team members, so it is essential to prevent one person's secrets from leaking to the entire team. The 30 gitleaks rules cover key formats for mainstream cloud services including AWS, GCP, Azure, GitHub, Slack, and Stripe (source code `secretScanner.ts` comment: "Rule IDs and regexes sourced directly from the public gitleaks config"). The gitleaks rule set was chosen over a homegrown solution because it has been validated at scale by the open-source community, with broad coverage and low false positive rates. The security design uses a fail-closed policy — when scanning fails, writing is blocked rather than allowed; functionality is sacrificed rather than secrets leaked. `scanForSecrets()` returns rule IDs rather than actual secret values, preventing secrets from being secondarily exposed in logs or telemetry.

### Error Handling

- On scan failure, writing is blocked by default (fail-closed), ensuring security
- Rule matching uses deterministic algorithms with no probabilistic false positives
- All interception events are recorded to the telemetry system

---

## Engineering Practice Guide

### Adding a New Service

**Step Checklist:**

1. **Create the service module**: Create a new directory and entry file under `src/services/`
2. **Define the lifecycle**: Implement startup (init/start) and shutdown (shutdown/cleanup) methods
3. **Register the service**: Register the initialization call in the `services/` entry point or `setup.ts`
4. **Determine startup timing**:
   - At application startup (e.g., policyLimits) → call directly in `setup.ts`
   - On-demand startup (e.g., LSP) → lazy initialization at first use
   - Idle startup (e.g., autoDream) → triggered by gating conditions
5. **Error handling**: Background service failures should not affect main functionality (silent degradation principle)

**Key Design Constraints**:
- Each service has an independent lifecycle and resource requirements
- A crash or restart of one service should not affect other services
- Services may behave differently in headless/bare mode (e.g., `initSessionMemory()` does not execute in bare mode)

### Inter-Service Communication

**Principle: Services communicate through an event bus or shared state — do not call each other directly.**

- **Event bus**: Send analytics events via `logEvent`, which other services can listen to
- **Shared state**: Share data through global config or app state
- **Hook system**: Hook points such as `postSamplingHook` and `handleStopHooks` allow multiple services to execute at the same timing
- **Avoid circular dependencies**: The analytics module in the source code is deliberately designed to be "dependency-free", precisely to avoid forming cycles when multiple services depend on it

### Debugging Service Startup

**Troubleshooting Steps:**

1. **Check initialization order**: Some services have dependencies (e.g., OAuth before API client)
2. **Check gating conditions**:
   - `feature('TEAMMEM')` — team memory sync
   - `feature('EXTRACT_MEMORIES')` — memory extraction
   - `isBareMode()` — skip services like LSP in bare mode
3. **Check PID lock**: autoDream uses a PID lock (60-minute timeout); may need cleanup after a crash
4. **Check background timers**: AgentSummary executes every 30 seconds; policyLimits/remoteManagedSettings poll every 1 hour
5. **Check Promise.allSettled**: The LSP manager's shutdown uses `Promise.allSettled`, so a single service shutdown failure does not affect others

**Key Thresholds/Timeouts by Service:**

| Service | Key Threshold |
|---------|---------------|
| AgentSummary | 30-second cycle |
| LSP | 30-second initialization timeout |
| autoDream | PID lock 60-minute timeout, `SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000` |
| policyLimits | 1-hour polling, fail-open (except HIPAA) |
| remoteManagedSettings | 1-hour polling |
| settingsSync | Maximum upload 500KB |

### Common Pitfalls

| Pitfall | Details | Solution |
|---------|---------|---------|
| Services behave differently in headless mode | Services that depend on UI, such as SessionMemory and MagicDocs, may not start in bare/headless mode | Check `isBareMode()` and related gating conditions |
| Service API calls count toward costs | AgentSummary, extractMemories, and autoDream all make API calls | Background service token consumption and costs are reflected in the cost tracker |
| policyLimits fail-open exception | HIPAA compliance scenarios use `ESSENTIAL_TRAFFIC_DENY_ON_MISS` mode | In this mode, operations are denied rather than allowed when the policy is unreachable |
| secretScanner fail-closed | Secret scanner scan failure blocks team memory writes | Functionality is sacrificed rather than secrets leaked |
| settingsSync conflict resolution | pull wins (remote data takes precedence) | Local modifications may be overwritten by remote data |


---

[← Feedback & Survey](../19-反馈与调查/feedback-system-en.md) | [Index](../README_EN.md) | [Plugin System →](../21-插件系统/plugin-system-en.md)
