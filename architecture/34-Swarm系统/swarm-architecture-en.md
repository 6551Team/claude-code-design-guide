# Swarm Multi-Agent System Architecture

## Backend Architecture

The Swarm system supports three backend types, defined via the `BackendType` union type:

```typescript
type BackendType = 'tmux' | 'iterm2' | 'in-process'
```

### Design Philosophy

#### Why 3 Backends (tmux/iTerm2/in-process)?

Different environments call for different optimal solutions:
- **tmux**: The only choice for headless environments (CI/SSH), most reliable. The source code implements full pane splitting, border coloring, and lock mechanisms.
- **iTerm2**: Visual debugging scenarios where developers can directly observe each agent's terminal output. Ideal for debugging and demos.
- **in-process**: Testing and lightweight scenarios with no subprocess overhead. Source code comment: "Unlike process-based teammates (tmux/iTerm2), in-process teammates run in the same Node.js process" — shared memory but logically isolated, each teammate has its own independent AbortController.

Backend detection uses a layered priority scheme: already inside tmux > iTerm2 available > fallback tmux > error, ensuring the most suitable backend is automatically selected.

#### Why Use the File System for Permission Sync Instead of IPC?

The source file `permissionSync.ts` implements a file-level permission request-response flow via `writePermissionRequest()` / `readPendingPermissions()` / `resolvePermission()`. There are three reasons to choose the file system over IPC pipes/sockets:
1. **Cross-process/cross-machine universality** — tmux panes, SSH sessions, and in-process mode can all access files.
2. **Audit log** — permission requests and decisions persist as files, enabling after-the-fact inspection of permission decision history.
3. **Crash recovery** — the file system survives process crashes, whereas IPC pipes/sockets die with the process, making it impossible to know prior permission state after recovery.

#### Why 17 Environment Variables Are Inherited?

The source code comment explains: "Tmux may start a new login shell that doesn't inherit the parent's env, so we forward any that are set in the current process." `TEAMMATE_ENV_VARS` contains API provider selection (`CLAUDE_CODE_USE_BEDROCK`, etc.), proxy configuration, CA certificate paths, and other critical variables. Without inheriting these, teammates would default to the firstParty endpoint and send requests to the wrong address (source references "GitHub issue #23561").

### PaneBackend Interface

`PaneBackend` is the core interface all backend implementations must follow. It defines pane lifecycle and visual control methods:

- **createTeammatePaneInSwarmView(name, command)**: Creates a new teammate pane and displays it in the swarm view.
- **sendCommandToPane(paneId, command)**: Sends a command to a specified pane for execution.
- **setPaneBorderColor(paneId, color)**: Sets the pane border color for visual differentiation.
- **setPaneTitle(paneId, title)**: Sets the pane title.
- **killPane(paneId)**: Terminates and destroys a pane.
- **hidePane(paneId)**: Hides a pane while preserving its process.
- **showPane(paneId)**: Shows a previously hidden pane.
- **rebalancePanes()**: Rebalances the layout allocation of all panes.

### TeammateExecutor Interface

`TeammateExecutor` manages the lifecycle of teammate processes:

- **spawn(config)**: Starts a new teammate process.
- **sendMessage(id, message)**: Sends a message to the specified teammate.
- **terminate(id)**: Gracefully terminates a teammate.
- **kill(id)**: Forcefully kills a teammate.
- **isActive(id)**: Checks whether a teammate is still actively running.

### Backend Detection Priority

The system determines the most suitable backend through layered detection:

1. **Inside tmux** (highest priority): Detected as already running inside a tmux session.
2. **In iTerm2 with it2**: Detected as running inside iTerm2 with the `it2` CLI available.
3. **Fallback tmux**: tmux is installed on the system but the current shell is not inside tmux.
4. **Error**: No backend available; an error is thrown.

### Caching Mechanism

To avoid repeated detection overhead, the system maintains the following caches:

- **cachedBackend**: Caches the already-initialized backend instance.
- **cachedDetectionResult**: Caches the backend type detection result.
- **inProcessFallbackActive**: Marks whether the system has fallen back to in-process mode.

---

## Permission Synchronization

Permission synchronization between the leader and teammates in Swarm uses multiple communication mechanisms.

### File-Level Permission Flow

A file-system-based permission request-response flow:

- **writePermissionRequest()**: The teammate writes a permission request to a shared directory.
- **readPendingPermissions()**: The leader polls and reads pending permission requests.
- **resolvePermission()**: The leader writes the permission resolution result.

### Mailbox-Level Permission Flow

Asynchronous permission communication based on the mailbox mechanism:

- **sendPermissionRequestViaMailbox()**: The teammate sends a permission request via mailbox.
- **sendPermissionResponseViaMailbox()**: The leader returns a permission response via mailbox.
- **sendSandboxPermissionRequestViaMailbox()**: A dedicated permission request for sandbox environments.

### SwarmPermissionRequest Schema

```typescript
interface SwarmPermissionRequest {
  id: string              // Unique request identifier
  workerId: string        // ID of the worker making the request
  workerName: string      // Human-readable name of the worker
  toolName: string        // Name of the tool requesting permission
  status: 'pending' | 'approved' | 'denied'  // Current status
}
```

### Leader Bridge

The leader side maintains a permission confirmation queue for UI interaction:

- **registerLeaderToolUseConfirmQueue()**: Registers the leader's tool-use confirmation queue.
- **getLeaderToolUseConfirmQueue()**: Retrieves the registered confirmation queue.

### Polling Interval

```typescript
const PERMISSION_POLL_INTERVAL_MS = 500
```

---

## In-Process Teammates

In-process mode runs teammates as independent agents within the same process, without requiring tmux/iTerm2.

### Permission Handling

**createInProcessCanUseTool()** creates the permission handling function:

- Prefers the bridge channel for permission confirmation.
- Falls back to the mailbox approach when the bridge is unavailable.
- Integrates the Classifier auto-approval mechanism to automatically approve specific bash commands.

### Process Management

**spawnInProcessTeammate()** starts an in-process teammate:

- Uses deterministic agentId generation to ensure traceability.
- Each teammate has its own independent AbortController, supporting individual termination.
- Shares memory space with the host process but is logically isolated.

**killInProcessTeammate()** terminates an in-process teammate:

- Triggers `AbortController.abort()` to stop execution.
- Removes the member record from the team file.
- Cleans up associated resources.

### InProcessBackend Class

`InProcessBackend` implements the full method set of the `TeammateExecutor` interface:

- **spawn**: Creates a new agent instance within the current process.
- **sendMessage**: Passes messages through in-memory channels.
- **terminate**: Gracefully stops an agent.
- **kill**: Forcefully stops an agent.
- **isActive**: Checks the agent's running state.

---

## Team Management

### TeamFile Structure

```typescript
interface TeamFile {
  members: TeamMember[]    // List of team members
  leaderId: string         // Agent ID of the leader
  allowedPaths: string[]   // Allowed paths shared by the team
  hiddenPanes: string[]    // List of currently hidden pane IDs
}
```

### Name Sanitization

- **sanitizeName(name)**: Sanitizes a general name string by removing illegal characters.
- **sanitizeAgentName(name)**: Specifically sanitizes an agent name to ensure it meets naming constraints.

### File Operations

- **readTeamFile()**: Synchronously reads the team file.
- **writeTeamFileAsync(data)**: Asynchronously writes the team file (ensures atomicity).
- **removeTeammateFromTeamFile(id)**: Removes a specified teammate from the team file.

---

## Tmux Backend

### Inside Tmux Mode

When detected as already running inside a tmux session:

- Splits the current window into two regions.
- **Leader region occupies 30%**, positioned on the left.
- **Teammates region occupies 70%**, positioned on the right.
- Teammate panes are further split within the right region.

### Outside Tmux Mode

When an external tmux session needs to be started:

- Connects using an external session socket.
- Creates a separate tmux session to manage teammates.

### Initialization Delay

```typescript
const PANE_SHELL_INIT_DELAY_MS = 200
```

After pane creation, waits 200 ms to ensure the shell has finished initializing.

### Lock Mechanism

A lock mechanism ensures panes are created sequentially:

- Prevents layout corruption from concurrent pane creation.
- Guarantees that each pane's shell initialization completes before the next pane is created.

---

## Environment Inheritance

### TEAMMATE_ENV_VARS

Defines 17 critical environment variables that must be forwarded to teammate processes, ensuring teammates inherit the leader's runtime environment configuration.

### buildInheritedCliFlags()

Builds the argument flags passed to the teammate CLI:

- **permission mode**: Permission mode configuration.
- **model**: The model identifier to use.
- **settings**: Configuration file path.
- **plugin-dir**: Plugin directory.
- **teammate-mode**: Marks the process as running in teammate mode.
- **chrome flags**: Chrome/browser-related flags.

### buildInheritedEnvVars()

Builds the set of environment variables passed to teammates:

- **CLAUDECODE=1**: Marks the process as running in the Claude Code environment.
- **API provider vars**: API provider related variables (keys, endpoints, etc.).
- **proxy config**: Proxy configuration.
- **CA certs**: CA certificate path configuration.

---

## Teammate Initialization

### initializeTeammateHooks()

Registers lifecycle hooks during teammate initialization:

- Registers a **Stop hook**: notifies the leader when a teammate stops.
- Upon receiving a stop notification, the leader can reassign tasks or clean up resources.

### Permission Rule Application

- Reads the `allowedPaths` configuration from the team file.
- Applies the team-scoped allowed paths as permission rules to the teammate.
- Ensures the teammate can only access authorized file paths.

### Idle Notification

- After completing a task, a teammate sends an idle notification.
- The notification includes a summary of task execution.
- It is delivered to the leader via the mailbox mechanism.
- The leader uses the idle notification to decide whether to assign a new task.

---

## Engineering Practice Guide

### Creating an Agent Swarm

1. **Create a team via TeamCreateTool**: Define the team name and member configuration.
2. **Choose a backend**:
   - **tmux**: The preferred choice for CI/SSH/headless environments — most reliable, cross-platform (requires tmux to be pre-installed).
   - **iTerm2**: Visual debugging scenarios — developers can directly observe each agent's terminal output; macOS only and requires the `it2` CLI.
   - **in-process**: Testing and lightweight scenarios — no subprocess overhead, shared memory but logically isolated, each teammate has its own independent AbortController.
3. **Assign tasks**: Assign specific task descriptions and tool permissions to each teammate.
4. **Automatic backend detection**: If not manually specified, the system selects automatically by priority: already inside tmux > iTerm2 available > fallback tmux > error.

### Debugging Permission Synchronization

1. **Check the file-level permission flow**:
   - View permission files under the `~/.claude/teams/{teamName}/permissions/` directory.
   - `pending` files: teammate's pending permission requests.
   - `resolved` files: leader's permission resolution results.
2. **Check the mailbox-level permission flow**:
   - If the file-level permission flow is not working, check whether the mailbox mechanism is functioning properly.
   - Did `sendPermissionRequestViaMailbox()` send successfully?
   - Did `sendPermissionResponseViaMailbox()` return successfully?
3. **Check the Leader Bridge**:
   - Does `getLeaderToolUseConfirmQueue()` return a valid queue?
   - Confirm that the leader's confirmation queue has been registered via `registerLeaderToolUseConfirmQueue()`.
4. **Polling interval**: `PERMISSION_POLL_INTERVAL_MS = 500ms` — if a permission response takes longer than 500 ms, the teammate may have already started the next polling cycle.

### Environment Variable Inheritance

17 env variables are passed from leader to worker — if a teammate behaves abnormally (e.g., connecting to the wrong API endpoint), check environment variable inheritance first:

1. Check the `TEAMMATE_ENV_VARS` list to confirm whether the required variable is included.
2. Check that the output of `buildInheritedEnvVars()` contains the expected variable values.
3. Critical variables: `CLAUDE_CODE_USE_BEDROCK`, proxy configuration, CA certificate paths — missing these will cause teammates to send requests to the wrong address.
4. CLI arguments are passed via `buildInheritedCliFlags()`: permission mode, model, settings, plugin-dir, teammate-mode.

### Decision Tree for Choosing a Backend

```
Need to run in CI/headless environment?
├─ Yes → tmux (confirm it is installed)
└─ No
   Need visual debugging?
   ├─ Yes → On macOS? → iTerm2 (confirm it2 CLI is available)
   │        Not on macOS → tmux
   └─ No
      Testing/lightweight scenario?
      ├─ Yes → in-process
      └─ No → tmux (most universal)
```

### Common Pitfalls

> **tmux must be pre-installed**: Swarm falls back to tmux by default, but if tmux is not installed on the system it will error immediately. In CI environments, ensure Docker images include tmux.

> **iTerm2 is macOS only**: The `it2` CLI is exclusive to iTerm2 — choosing the iTerm2 backend on a non-macOS system will fail. The backend detection priority handles this automatically, but be aware when specifying manually.

> **File locks may be unreliable on NFS**: Permission synchronization relies on file system operations. If the team directory is on NFS or another network file system, the semantics of file locks and atomic writes may not be guaranteed — this can lead to permission race conditions. Using a local file system is recommended.

> **Pane initialization delay**: After a tmux pane is created, `PANE_SHELL_INIT_DELAY_MS = 200ms` of wait time is needed for shell initialization. Concurrent pane creation uses a lock mechanism to ensure ordering — if "command not found" errors are observed, the initialization delay may be insufficient.

> **Memory sharing in in-process mode**: Unlike tmux/iTerm2, in-process teammates share the same Node.js process memory as the leader. Although logically isolated (each with its own AbortController), a memory leak in one teammate will affect all other teammates and the leader.


---

[← Coordinator Pattern](../33-协调器模式/coordinator-mode-en.md) | [Index](../README_EN.md) | [Computer Use →](../35-Computer-Use/computer-use-en.md)
