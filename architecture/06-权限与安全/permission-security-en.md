# Permission & Security

> Source files: `src/utils/permissions/` (24 files), `src/hooks/useCanUseTool.tsx`,
> `src/hooks/toolPermission/`, `src/types/permissions.ts`,
> `src/utils/sandbox/`, `src/utils/permissions/yoloClassifier.ts`

---

## 1. Architecture Overview

The permission system is the security core of Claude Code, controlling model access to tools. It ensures a balance between security and usability through a multi-layered decision pipeline.

```
toolExecution.ts
  └─→ canUseTool() (permission decision entry point)
        ├── Step 1: Rule matching (allow/deny/ask rules)
        ├── Step 2: Tool-specific logic (tool-specific permissions)
        ├── Step 3: Classifier (auto mode classifier)
        └── Step 4: User interaction prompt (user prompt)
```

---

## 2. Six Permission Modes

### 2.1 Mode Definitions

```typescript
// src/types/permissions.ts
export type ExternalPermissionMode = 'acceptEdits' | 'bypassPermissions' | 'default' | 'dontAsk' | 'plan'
export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
export type PermissionMode = InternalPermissionMode
```

### 2.2 Mode Details

| Mode | Title | Symbol | Description |
|------|------|------|------|
| `default` | Default | (none) | Standard mode — requires user confirmation for destructive operations |
| `plan` | Plan Mode | (pause icon) | Plan mode — model can only use read-only tools, no modifications |
| `acceptEdits` | Accept Edits | - | Automatically accept file edits, but shell commands still require confirmation |
| `bypassPermissions` | Bypass Permissions | - | Bypass all permission checks (dangerous mode) |
| `dontAsk` | Don't Ask | - | Don't prompt user, reject operations requiring permissions |
| `auto` | Auto Mode | - | AI classifier automatically determines (requires `TRANSCRIPT_CLASSIFIER` feature flag) |
| `bubble` | (Internal) | - | Internal mode — permission decisions bubble to parent (used by sub-agents) |

#### Design Philosophy: Why 6 modes instead of simple allow/deny?

- **Trust Gradient**: These 6 modes represent a gradient from most conservative to most trusting: `plan` (safest, read-only) -> `default` (user confirmation) -> `acceptEdits` (trust file operations) -> `auto` (AI judgment) -> `bypassPermissions` (full trust) -> `bubble` (sub-agent delegation). Simple allow/deny cannot express fine-grained needs like "trust file edits but not shell commands".
- **Scenario Matching**: Different use cases require different trust levels — security audits use `plan` (look but don't touch), daily development uses `default` (human in the loop), trusted CI pipelines use `bypassPermissions` (unattended), AI autonomous development uses `auto` (classifier replaces human judgment).
- **Necessity of `bubble` Mode**: Sub-agents (created via `AgentTool`) should not make independent permission decisions. If sub-agents pop up permission dialogs independently, users would see permission requests from unclear contexts and cannot make informed judgments. `bubble` mode lets permission requests bubble to the parent, where someone/something with full context can confirm.
- **Existence Rationale for `dontAsk`**: Background agents (`shouldAvoidPermissionPrompts=true`) cannot pop up UI prompts to users, requiring a "don't prompt, directly reject" mode to avoid process hanging waiting for user input that will never come.

### 2.3 External vs Internal Modes

- **External Modes** (`EXTERNAL_PERMISSION_MODES`): 5 modes users can configure via UI/CLI/settings
- **Internal Modes**: Include `auto` (requires feature flag) and `bubble` (only used internally by sub-agents)
- `PERMISSION_MODES` = `INTERNAL_PERMISSION_MODES` (runtime validation set)

---

## 3. Permission Rule System

### 3.1 Rule Sources (PermissionRuleSource)

```typescript
export type PermissionRuleSource =
  | 'userSettings'      // ~/.claude/settings.json
  | 'projectSettings'   // .claude/settings.json (project-level)
  | 'localSettings'     // .claude/settings.local.json
  | 'flagSettings'      // Feature flag remote settings
  | 'policySettings'    // Enterprise policy settings
  | 'cliArg'            // CLI arguments (--allowedTools)
  | 'command'           // /allowed-tools and similar commands
  | 'session'           // In-session user decisions ("always allow for this session")
```

#### Design Philosophy: Why do 5 config sources form a hierarchy instead of equal merging?

- **Organizational Governance Model Encoded as Software**: This is not a simple technical configuration system, but maps enterprise organizational structure to rule priority. `policySettings` (enterprise policy) > `flagSettings` (CLI arguments/remote settings) > `localSettings` (local) > `projectSettings` (project) > `userSettings` (personal).
- **Security Non-Negotiable Principle**: Enterprise CISO can force-prohibit `rm -rf` via `policySettings`, even if developers personally configured allow, enterprise policy overrides personal preference. Source code `settings/constants.ts:159-167` in `getEnabledSettingSources()` hardcodes `result.add('policySettings')` and `result.add('flagSettings')` — these two sources can never be disabled, even if users limit other config sources via environment variables.
- **Shadowed Rule Detection**: Source code `shadowedRuleDetection.ts` specifically detects whether low-priority source rules are shadowed by high-priority source rules (e.g., project deny "Bash" shadowed by user allow "Bash(git status)"), and warns when shared config files (projectSettings, policySettings) are shadowed, as these files affect the entire team.

### 3.2 Rule Behaviors (PermissionBehavior)

```typescript
export type PermissionBehavior = 'allow' | 'deny' | 'ask'
```

### 3.3 Rule Values (PermissionRuleValue)

```typescript
export type PermissionRuleValue = {
  toolName: string       // Tool name (exact match or prefix match)
  ruleContent?: string   // Optional content matching condition (e.g., Bash command pattern)
}
```

### 3.4 Complete Rule Type

```typescript
export type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: PermissionBehavior
  ruleValue: PermissionRuleValue
}
```

### 3.5 Rule Storage

Rules are stored grouped by source in `ToolPermissionContext`:

```typescript
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource     // allow rule set
  alwaysDenyRules: ToolPermissionRulesBySource       // deny rule set
  alwaysAskRules: ToolPermissionRulesBySource        // ask rule set
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource  // stripped dangerous rules
  shouldAvoidPermissionPrompts?: boolean               // background agent without UI
  awaitAutomatedChecksBeforeDialog?: boolean            // coordinator worker
  prePlanMode?: PermissionMode                          // mode before plan mode
}>
```

---

## 4. canUseTool() Decision Pipeline

### 4.1 Decision Flow

```
canUseTool(tool, input, assistantMessage)
    │
    ├── Step 1a: Blanket deny rules (deny without ruleContent)
    │     └─→ deny → return rejection
    │
    ├── Step 1b: Allow rules (with ruleContent matching)
    │     └─→ match → return allow
    │
    ├── Step 1c: Deny rules (with ruleContent matching)
    │     └─→ match → return rejection
    │
    ├── Step 1d: Ask rules
    │     └─→ match → enter user prompt
    │
    ├── Step 2: Tool-specific permission logic
    │     ├── Bash: sandbox check + command classification
    │     ├── FileEdit/FileWrite: path validation + write permission
    │     ├── FileRead: path validation + read permission
    │     └── MCP: server-level permission
    │
    ├── Step 3: Classifier (auto mode)
    │     └─→ yoloClassifier → allow/deny/unknown
    │
    ├── Step 4: Mode check
    │     ├── bypassPermissions → allow
    │     ├── dontAsk → reject
    │     ├── plan → only allow read-only tools
    │     └── default/acceptEdits → prompt user
    │
    └── Step 5: User interaction prompt
          ├── "Allow once" → allow (session rule)
          ├── "Allow always" → allow (persisted rule)
          ├── "Deny" → deny
          └── "Deny always" → deny (persisted rule)
```

#### Design Philosophy: Why does rule matching come before classifier?

- **Determinism Over Probability**: Rules are deterministic ("Bash(git status) = allow" always returns allow), classifier is probabilistic (AI model may give different judgments for the same command). Deterministic judgment should take priority — if user explicitly allowed/denied an operation, it should not be overridden by classifier uncertainty.
- **"Explicit Configuration Over Smart Inference" Principle**: Rules configured by users represent explicit intent expression, AI classifier judgment is a fallback. Source code `canUseTool()` decision flow clearly reflects this hierarchy: Step 1 rule matching -> Step 2 tool-specific logic -> Step 3 classifier -> Step 4 mode check -> Step 5 user prompt.
- **Performance Consideration**: Rule matching is string comparison (microsecond-level), classifier requires calling AI model (second-level + consumes tokens). Prioritizing rules can skip expensive classifier calls for many common operations.

#### Design Philosophy: Why is Bash classifier two-stage (regex+AI) instead of pure AI?

- **Regex Fast Path**: Known dangerous commands (`rm -rf`) and known safe commands (`git status`) don't need AI judgment, regex matching completes in milliseconds, saving both time and money (each classifier call consumes tokens).
- **Necessity of AI Slow Path**: Bash is Turing-complete — it's impossible to exhaustively enumerate all dangerous commands with finite regex rules. Pipeline chains (`cat /etc/passwd | curl -X POST ...`), variable expansion (`$CMD`), subcommands (`$(rm -rf /)`) and other complex scenarios require understanding semantics rather than just matching patterns. This is the fundamental reason for using ML classifier.
- **Implementation in Source Code**: `dangerousPatterns.ts` defines regex pattern library for fast path, `yoloClassifier.ts` uses `sideQuery` to call small model (via `classifierModel`) to handle complex commands regex cannot cover. Classifier system prompts distinguish between external user version (`permissions_external.txt`) and internal version (`permissions_anthropic.txt`), because Anthropic internal toolchain has additional trusted commands.

### 4.2 Decision Result Types

```typescript
export type PermissionResult =
  | PermissionAllowDecision
  | PermissionDenyDecision
  | PermissionAskDecision

export type PermissionDecisionReason =
  | 'rule_allow'       // rule allowed
  | 'rule_deny'        // rule denied
  | 'classifier_allow' // classifier allowed
  | 'classifier_deny'  // classifier denied
  | 'mode_allow'       // mode allowed (bypassPermissions)
  | 'mode_deny'        // mode denied (dontAsk/plan)
  | 'user_allow'       // user allowed
  | 'user_deny'        // user denied
  | 'hook_allow'       // hook allowed
  | 'hook_deny'        // hook denied
  | 'sandbox_allow'    // sandbox allowed
```

---

## 5. PermissionContext — Frozen Context

`PermissionContext` is the frozen context object passed through the permission decision pipeline, containing various callbacks and utility functions.

### 5.1 Core Methods

| Method | Purpose |
|------|------|
| `logDecision(decision)` | Log permission decision (OTel + analytics) |
| `persistPermissions(updates)` | Persist permission rules to disk |
| `tryClassifier(tool, input)` | Try classifier judgment |
| `runHooks(hookType, params)` | Execute permission-related hooks |
| `buildAllow(reason)` | Build allow decision |
| `buildDeny(reason, message)` | Build deny decision |

### 5.2 OTel Source Mapping

```typescript
// Rule source → OTel log tag
'session' + allow → 'user_temporary'
'session' + deny  → 'user_reject'
'localSettings'/'userSettings' + allow → 'user_permanent'
'localSettings'/'userSettings' + deny  → 'user_reject'
others → 'config'
```

---

## 6. Bash Classifier (yoloClassifier.ts)

### 6.1 Overview

`yoloClassifier.ts` implements the core classification logic for Auto mode, using AI model to judge whether Bash commands are safe.

### 6.2 Two-Stage Classification

**Stage 1: Regex Pattern Detection (Fast Path)**

Based on regex patterns in `dangerousPatterns.ts` to quickly detect obviously dangerous commands:
- Known safe command whitelist (git status, ls, cat, etc.)
- Known dangerous command blacklist (rm -rf, chmod 777, etc.)

**Stage 2: AI Classifier (Slow Path)**

```typescript
// Use sideQuery to call small model for classification
const result = await sideQuery({
  systemPrompt: BASE_PROMPT + PERMISSIONS_TEMPLATE,
  messages: transcriptContext,
  model: classifierModel,
  // ...
})
```

Classifier prompt templates:
- `auto_mode_system_prompt.txt` — Base system prompt
- `permissions_external.txt` — External user permission template
- `permissions_anthropic.txt` — Anthropic internal permission template (ant-only)

### 6.3 Classification Results

```typescript
export type YoloClassifierResult = {
  decision: 'allow' | 'deny' | 'unknown'
  reasoning?: string
  usage?: ClassifierUsage
}
```

### 6.4 Caching and Optimization

- `getLastClassifierRequests` / `setLastClassifierRequests` — Cache recent classifier requests
- Use `getCacheControl()` to cache classifier system prompts
- Classifier duration tracking: `addToTurnClassifierDuration`

---

## 7. Sandbox System

### 7.1 Sandbox Configuration Schema (11 Fields)

```typescript
// Inferred from sandbox configuration
type SandboxConfig = {
  enabled: boolean                    // Whether sandbox is enabled
  type: 'macos-sandbox' | 'linux-namespace' | 'docker'  // Sandbox type
  allowedDirectories: string[]        // Allowed access directories
  deniedDirectories: string[]         // Denied directories
  allowNetwork: boolean               // Whether network is allowed
  allowSubprocesses: boolean          // Whether subprocesses are allowed
  timeout: number                     // Timeout (ms)
  maxMemory: number                   // Maximum memory
  maxFileSize: number                 // Maximum file size
  readOnlyDirectories: string[]       // Read-only directories
  environmentVariables: Record<string, string>  // Environment variables
}
```

### 7.2 Sandbox Execution

```typescript
// utils/sandbox/sandbox-adapter.ts
export class SandboxManager {
  // shouldUseSandbox() — Determine whether sandbox should be used
  // execute() — Execute command within sandbox
  // validateViolation() — Check sandbox violations
}
```

### 7.3 Sandbox Decision Integration

When `shouldUseSandbox()` returns true:
1. Bash commands execute in sandbox environment
2. Sandbox provides filesystem isolation
3. Violations are detected and reported
4. Permission decision can be `sandbox_allow`

---

## 8. Path Validation

### 8.1 Path Safety Checks (pathValidation.ts)

```
pathValidation.ts
  ├── Absolute path vs relative path validation
  ├── Allowed directory check (CWD + additionalWorkingDirectories)
  ├── Symbolic link resolution and validation
  └── UNC path safety (Windows)
```

### 8.2 Check Rules

1. **Absolute paths**: Must be within allowed directory scope
2. **Relative paths**: Check after resolving to absolute path
3. **Symbolic links**: Validate target path after resolving to final target
4. **UNC paths** (Windows `\\server\share`): Special security handling
5. **Path traversal**: Detect `../` escapes

### 8.3 Allowed Directories

- Current working directory (CWD)
- `additionalWorkingDirectories` (added via /add-dir command)
- System temporary directory (certain operations)
- Configuration files under user home directory

---

## 9. High Severity Operation Detection

### 9.1 Dangerous Patterns (dangerousPatterns.ts)

System maintains a dangerous operation pattern library for fast path classification:

#### Bulk Deletion
- `rm -rf /`
- `find . -delete`
- `git clean -fdx`

#### Infrastructure Operations
- `terraform destroy`
- `kubectl delete`
- `docker rm -f`

#### Credential Operations
- `cat ~/.ssh/id_rsa`
- `echo $API_KEY`
- Reading `.env` files

#### Git Force Operations
- `git push --force`
- `git reset --hard`
- `git branch -D`

#### System Modifications
- `chmod 777`
- `chown root`
- `sudo` operations

### 9.2 Shell Rule Matching (shellRuleMatching.ts)

Parsing and pattern matching for Bash commands:
- Command parsing (shell-quote)
- Pipeline chain detection
- Redirection detection
- Environment variable expansion (limited)
- Subcommand detection (`$(...)`, backticks)

---

## 10. Filesystem Permissions

### 10.1 Read Permission Check

```typescript
checkReadPermissionForTool(filePath, toolUseContext)
```

- Whether file is within allowed directories
- Whether file is excluded by `.gitignore` (considered in certain modes)
- Whether file size is within limits (`fileReadingLimits`)

### 10.2 Write Permission Check

```typescript
checkWritePermissionForTool(filePath, toolUseContext)
```

- Whether file is within allowed write directories
- Path validation (absolute path, symbolic links, UNC)
- Whether file is a protected file (config files, credentials, etc.)

### 10.3 Team Memory Key Protection

Special protection for team memory files:
- Prevent writing team memory containing sensitive information
- Key pattern detection (API key patterns, token patterns)
- Reject writing content that may contain secrets

---

## 11. Denial Tracking (denialTracking.ts)

### 11.1 DenialTrackingState

```typescript
export type DenialTrackingState = {
  consecutiveDenials: number    // Consecutive denial count
  lastDenialTimestamp: number   // Last denial timestamp
  lastDeniedTool: string        // Last denied tool
}
```

### 11.2 Purpose

- When consecutive denials reach threshold, fall back to user prompt (even in auto mode)
- Prevent classifier from continuously making wrong decisions
- Sub-agents use `localDenialTracking` (because setAppState is no-op)

#### Design Philosophy: Why does denial tracking exist?

- **AI Classifier Error Correction Mechanism**: Classifier is probabilistic and may continuously make wrong judgments (e.g., repeatedly denying commands user truly needs to execute). Source code `permissions.ts:490-498`, when `consecutiveDenials > 0` and a tool is successfully allowed, calls `recordSuccess()` to reset denial count. When consecutive denials reach `DENIAL_LIMITS.maxConsecutive` (source code `denialTracking.ts:42`), system falls back to user prompt, letting humans make the decision.
- **Independent Tracking for Sub-agents**: Source code `permissions.ts:553-558` comment explains: "Use local denial tracking for async subagents (whose setAppState is a no-op), otherwise read from appState as before." Sub-agent's `setAppState` is empty operation (cannot modify parent state), so needs `localDenialTracking` as independent local state. `forkedAgent.ts:420-421` initializes `localDenialTracking` when creating sub-agent.
- **Allow Reset Mechanism**: Any successful tool use (whether rule allowed or classifier allowed) resets denial count (source code `permissions.ts:483-500`). This ensures denial tracking only triggers on "consecutive" denials, occasional denials don't accumulate.

---

## 12. Permission Rule Parsing and Persistence

### 12.1 Rule Format

Rule format in settings files:

```json
{
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(npm test)",
      "FileRead",
      "mcp__server"
    ],
    "deny": [
      "Bash(rm -rf)",
      "Bash(sudo *)"
    ]
  }
}
```

### 12.2 Rule Parsing

```typescript
// permissionRuleParser.ts
permissionRuleValueFromString("Bash(git status)")
// → { toolName: "Bash", ruleContent: "git status" }

permissionRuleValueFromString("FileRead")
// → { toolName: "FileRead", ruleContent: undefined }

permissionRuleValueFromString("mcp__server")
// → { toolName: "mcp__server", ruleContent: undefined }
```

### 12.3 Rule Persistence

```typescript
// PermissionUpdate.ts
applyPermissionUpdate(update, settingsPath)
applyPermissionUpdates(updates[], settingsPath)
persistPermissionUpdates(updates[], destination)

type PermissionUpdateDestination = 'user' | 'project' | 'local'
```

### 12.4 Shadowed Rule Detection

```typescript
// shadowedRuleDetection.ts
// Detect whether low-priority source rules are shadowed by high-priority source rules
// Example: project deny "Bash" shadowed by user allow "Bash(git status)"
```

---

## 13. Complete Permission Decision Data Flow

![Permission Decision Pipeline](../diagrams/permission-decision-pipeline-en.svg)

---

## Engineering Practice Guide

### Adding New Permission Rule Source

If you need to introduce a new configuration source (e.g., loading rules from a new remote service):

1. **Add new value in `PermissionRuleSource` type** — Modify `PermissionRuleSource` union type in `src/types/permissions.ts`
2. **Register in `SETTING_SOURCES` in `settings/constants.ts`** — Ensure `getEnabledSettingSources()` includes new source
3. **Implement loading logic** — Implement config loading function in `utils/settings/`, decide when to load (at startup? After Trust? On-demand?)
4. **Determine priority** — New source's position in rule hierarchy determines whether it can override/be overridden by other source rules
5. **Update shadowed rule detection** — `shadowedRuleDetection.ts` needs to know new source's priority relationships

**Key Constraints**:
- `policySettings` and `flagSettings` can never be disabled (hardcoded `result.add()` in `getEnabledSettingSources()`)
- If new source is shared (affects entire team), should warn when shadowed

### Adding New Permission Mode

1. **Add in `PermissionMode` type** — Modify `src/types/permissions.ts`
2. **Add corresponding branch in `canUseTool` pipeline** — Handle new mode's default behavior in Step 4 (mode check)
3. **Update UI mode selector** — If externally available mode, add to `EXTERNAL_PERMISSION_MODES`; if internal mode, only add to `INTERNAL_PERMISSION_MODES`
4. **Add tests** — Cover new mode's behavior under various rule combinations

### Debugging Permission Denials

When tool execution is denied by permission system:

1. **Enable `--debug`** — View complete decision chain of `canUseTool()`, including:
   - Which rule matched? (`rule_allow` / `rule_deny`)
   - From which source? (`userSettings` / `projectSettings` / `policySettings`)
   - Classifier's judgment? (`classifier_allow` / `classifier_deny`)
   - Final decision reason? (`PermissionDecisionReason`)
2. **Check rule priority** — In deny rules, blanket deny (without `ruleContent`) is checked before any allow rules. If configured `deny: ["Bash"]` (no content), all Bash commands will be denied, even if there are more specific allow rules
3. **Check shadowed rules** — Low-priority source allow rules may be shadowed by high-priority source deny rules
4. **OTel logs** — Permission decisions are logged to OTel via `logDecision()`, can track historical decision patterns in telemetry dashboard

### Bash Classifier Debugging

When auto mode's classification results for Bash commands don't meet expectations:

1. **Disable AI classifier** — Set `CLAUDE_CODE_DISABLE_BASH_CLASSIFIER=true`, only use regex fast path, confirm whether problem is in AI classifier
2. **Check regex patterns** — Regex patterns in `dangerousPatterns.ts` are fast path judgment basis, confirm whether command hit a pattern
3. **Check classifier cache** — `getLastClassifierRequests()` returns recent classifier requests and results
4. **Check classifier prompt template** — `auto_mode_system_prompt.txt` + `permissions_external.txt` (or `permissions_anthropic.txt`) constitute classifier's system prompt

### Adding Dangerous Pattern Detection

When adding new regex patterns in `dangerousPatterns.ts`:

1. **Write regex** — Match target dangerous command pattern
2. **Add test cases** — Positive matches (commands that should be detected) and negative matches (commands that shouldn't false positive)
3. **Consider variants** — Shell commands have many variant writings (short/long options, quoted/unquoted, pipeline chains, etc.), ensure regex covers common variants
4. **Note performance** — Regex matching runs on every Bash tool execution, avoid overly complex regex (e.g., excessive backtracking)

### Sandbox Debugging

1. **Check `shouldUseSandbox()` return value** — Confirm whether sandbox should be enabled
2. **Check sandbox type** — `macos-sandbox` / `linux-namespace` / `docker`, different platforms use different sandbox implementations
3. **Check violation detection** — `SandboxManager.validateViolation()` detects sandbox violations; if command fails in sandbox but succeeds externally, may be sandbox config's `allowedDirectories` or `allowNetwork` restrictions too strict
4. **bypassPermissions sandbox protection** — `bypassPermissions` mode in non-sandbox environment with network access will be rejected by `setup.ts` (security validation step 11), this is the last line of defense preventing unrestricted mode from running in unsafe environments

### Common Pitfalls

1. **`policySettings` can never be disabled** — Don't try to bypass enterprise policy settings during testing. `policySettings` and `flagSettings` in `getEnabledSettingSources()` are hardcoded additions, cannot be excluded via environment variables or config

2. **deny rules take priority over allow rules (when both exist at same level)** — If `projectSettings` has both `allow: ["Bash(git *)"]` and `deny: ["Bash"]`, blanket deny will take effect before allow matching

3. **MCP tool permissions use `mcp__server` prefix** — When matching MCP tools in permission rules, use `mcp__serverName__toolName` format, not tool's original name. `filterToolsByDenyRules()` does special prefix matching for MCP tools

4. **auto mode classifier results should not be viewed as final decision** — Classifier is probabilistic, denial tracking (`denialTracking.ts`) will fall back to user prompt when consecutive denials reach threshold. Don't assume classifier's `allow`/`deny` is irrevocable

5. **Sub-agent permissions use `bubble` mode** — Sub-agents (created via `AgentTool`) should not make independent permission decisions. `bubble` mode lets permission requests bubble to parent. If seeing unexpected permission behavior in sub-agent, check whether `bubble` mode is correctly set



---

[← Tool System](../05-工具系统/tool-system-en.md) | [Index](../README_EN.md) | [Context Management →](../07-上下文管理/context-management-en.md)
