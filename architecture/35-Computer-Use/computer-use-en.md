# Computer Use Integration Architecture

The Computer Use feature is integrated into Claude Code as an MCP Server, providing screen interaction, mouse/keyboard control, and other computer operation capabilities.

---

## Architecture (15 files in utils/computerUse/)

### Core Constants

```typescript
const COMPUTER_USE_MCP_SERVER_NAME = 'computer-use'
const CLI_HOST_BUNDLE_ID = 'com.anthropic.claude-code.cli-no-window'
```

### Terminal Bundle ID Mapping

`TERMINAL_BUNDLE_ID_FALLBACK` maps common terminal names to their corresponding macOS Bundle IDs:

| Terminal Name | Bundle ID |
|--------------|-----------|
| iTerm | com.googlecode.iterm2 |
| Terminal | com.apple.Terminal |
| Ghostty | com.mitchellh.ghostty |
| Kitty | net.kovidgoyal.kitty |
| Warp | dev.warp.Warp |
| VSCode | com.microsoft.VSCode |

### Design Philosophy

#### Why Enigo/Swift instead of AppleScript on macOS?

Source code comments reveal the technical architecture: "@ant/computer-use-input (Rust/enigo) -- mouse, keyboard, frontmost app" + "@ant/computer-use-swift -- SCContentFilter screenshots, NSWorkspace apps, TCC". AppleScript has latency >100ms per operation and limited control over low-level events; Enigo calls system APIs directly (via `DispatchQueue.main`), with latency in the 10ms range. The native Swift module provides screen capture capabilities (SCContentFilter) and TCC permission detection that AppleScript cannot achieve.

#### Why O_EXCL atomic locks?

Source code comments state: "Uses O_EXCL (open 'wx') for atomic test-and-set -- the OS guarantees at most one process succeeds". Multiple agents (such as teammates in a Swarm) may simultaneously attempt to control the mouse/keyboard — atomic file locks ensure only one session can operate at a time, preventing chaos from multiple agents moving the mouse simultaneously. The lock file contains `sessionId`, `pid`, and `acquiredAt`, supporting PID-based stale lock detection and a 60-minute forced reclamation timeout.

#### Why CGEventTap for the ESC hotkey?

Source code comments explain the security consideration: "Global Escape -> abort. Mirrors Cowork's escAbort.ts but without Electron: CGEventTap via @ant/computer-use-swift. While registered, Escape is consumed system-wide (PI defense -- a prompt-injected action can't dismiss a dialog with Escape)". Users need to emergency-stop computer-use operations; ESC as a global hotkey fires even when focus is not on the Claude Code window. More critically, it serves as a security defense — system-level Escape interception prevents prompt injection attacks from using the Escape key to dismiss security dialogs. `notifyExpectedEscape()` uses a 100ms decay window to distinguish between the model's own Escape operations and user interrupt intent.

#### Why CFRunLoop pump?

Source code comments directly explain the root cause: "Swift's @MainActor async methods and @ant/computer-use-input's key()/keys() all dispatch to DispatchQueue.main. Under libuv (Node/bun) that queue never drains -- the promises hang. Electron drains it via CFRunLoop so Cowork doesn't need this". macOS's main thread dispatch queue does not drain automatically in a Node.js environment; it must be manually pumped via `_drainMainRunLoop` (called every 1ms). A reference counting mechanism (`retainPump`/`releasePump`) ensures the RunLoop only runs when there are active Computer Use operations, avoiding idle CPU waste.

---

## Feature Gates

### Main Switch

**getChicagoEnabled()** controls whether the Computer Use feature is available:

- Requires max or pro+ subscription tier
- Anthropic internal users (ants) can bypass the restriction

### Sub-feature Switches

- **pixelValidation**: Pixel-level coordinate validation
- **clipboardPasteMultiline**: Multi-line text clipboard paste
- **mouseAnimation**: Mouse movement animation effects
- **hideBeforeAction**: Hide the Claude Code window before executing an action
- **autoTargetDisplay**: Automatically select the target display
- **clipboardGuard**: Clipboard content protection (prevents accidentally overwriting the user's clipboard)

### Coordinate Mode

**getChicagoCoordinateMode()** returns the coordinate mode:

```typescript
type CoordinateMode = 'pixels' | 'normalized'
```

- **pixels**: Use absolute pixel coordinates
- **normalized**: Use 0-1 normalized coordinates
- Frozen after the first read; cannot be changed during runtime

---

## Executor (executor.ts, 658 lines)

### Factory Function

**createCliExecutor()** creates an executor for the CLI environment, wrapping:

- **@ant/computer-use-input**: Cross-platform input control implemented in Rust/enigo
- **@ant/computer-use-swift**: macOS native Swift implementation for system interaction

### Method Set

#### Screen Operations
- **screenshot()**: Capture a screen snapshot
- **zoom(factor)**: Zoom the display

#### Keyboard Operations
- **key(keys)**: Key combinations (e.g., Ctrl+C)
- **holdKey(key, duration)**: Hold a key down for a duration
- **type(text)**: Type a text string

#### Clipboard
- **readClipboard()**: Read clipboard content
- **writeClipboard(text)**: Write content to clipboard

#### Mouse Operations
- **moveMouse(x, y)**: Move the mouse to a specified position
- **click(x, y, button)**: Click
- **mouseDown(x, y, button)**: Press a mouse button
- **mouseUp(x, y, button)**: Release a mouse button
- **getCursorPosition()**: Get the current cursor position
- **drag(fromX, fromY, toX, toY)**: Drag operation
- **scroll(x, y, deltaX, deltaY)**: Scroll operation

#### Application Management
- **getFrontmostApp()**: Get information about the current foreground application
- **listInstalledApps()**: List installed applications
- **getAppIcon(bundleId)**: Get the application icon
- **listRunningApps()**: List running applications
- **openApp(bundleId)**: Open a specified application

#### Preparation Operations
- **prepareForAction()**: Preparation work before executing an action (e.g., hiding the window)

### Animated Movement

**animatedMove()** implements smooth mouse movement:

- Uses **ease-out-cubic** easing curve
- Movement speed: **2000 px/sec**
- Provides natural mouse movement visual effects

### CLI-Specific Handling

- **No click-through**: CLI mode does not support click-through
- **Terminal surrogate host**: Uses the terminal as a surrogate host application
- **Clipboard**: Operates the clipboard via `pbcopy`/`pbpaste` commands

---

## Lock System (computerUseLock.ts)

### Atomic Lock Implementation

Uses the `O_EXCL` flag to implement an atomic file creation lock, ensuring only one Computer Use session is active at a time:

```typescript
const HOLDER_STALE_MS = 60 * 60 * 1000  // 60 minutes
```

### Lock File

Path: `~/.claude/computer-use.lock`

```json
{
  "sessionId": "session-uuid",
  "pid": 12345,
  "acquiredAt": "2025-01-01T00:00:00.000Z"
}
```

### Stale Lock Recovery

- PID-based stale lock detection
- If the lock-holding process has terminated, the lock is automatically reclaimed
- Forced reclamation after `HOLDER_STALE_MS = 60min` timeout

### Zero-Syscall Check

**isLockHeldLocally()**: Checks lock hold status via in-memory state, requiring no system calls, making it extremely performant.

---

## ESC Hotkey (escHotkey.ts)

### CGEventTap Registration

Registers a system-level Escape key event listener for user interruption of Computer Use operations:

- Uses the macOS CGEventTap API
- Captures global Escape key events

### Expected Escape Handling

**notifyExpectedEscape()**: Called when the model itself needs to synthesize an Escape keypress:

- Creates a **100ms** decay window
- Escape events within this window are treated as model actions rather than user interrupts
- Normal interrupt detection resumes after the window expires

---

## CFRunLoop (drainRunLoop.ts)

### Reference-Counted Pump

Uses a reference-counted setInterval pump to keep the CFRunLoop running:

- Pump interval: **1ms**
- Timeout protection: **30s** maximum runtime

### Lifecycle Management

- **retainPump()**: Increments the reference count; starts the pump on first call
- **releasePump()**: Decrements the reference count; stops the pump when it reaches zero
- Ensures the RunLoop only runs when there are active Computer Use operations

---

## App Filtering (appNames.ts)

### Filtering Logic

**filterAppsForDescription()** filters the application list, removing noise applications:

- Blocks background applications containing keywords such as Helper/Agent/Service/Updater
- Retains only user-visible foreground applications

### Allowlist

**ALWAYS_KEEP_BUNDLE_IDS**: Approximately 30 core applications that are always retained:

- Browsers: Chrome, Safari, Firefox, Arc, Edge
- Communication: Slack, Discord, Zoom, Teams
- Development: VSCode, Xcode, Terminal, iTerm2
- Productivity: Finder, Notes, Calendar, Mail
- Other common applications

### Name Validation

**APP_NAME_ALLOWED**: Application name validation rules:

- Unicode-aware regular expression
- Maximum **40 character** length limit
- At most **50 applications** returned per call

---

## Cleanup

### cleanupComputerUseAfterTurn()

Cleanup process run at the end of each conversation turn:

1. **Auto-unhide**: Restores previously hidden windows, with a **5s timeout** to prevent hangs
2. **Deregister ESC listener**: Removes the CGEventTap Escape key listener
3. **Release lock**: Releases the computer-use.lock file lock
4. Releases the CFRunLoop pump reference

---

## MCP Server

### Server Creation

**createComputerUseMcpServerForCli()** builds an MCP Server for the CLI environment:

- Initializes all tool definitions
- **Replaces ListTools**: Replaces the standard ListTools with an enhanced version containing application description information
- Injects application context into tool descriptions to help the model understand the current desktop environment

### Subprocess Entry Point

**runComputerUseMcpServer()** is the entry point run as a standalone subprocess:

- Uses **stdio transport** to communicate with the host process
- Standard MCP Server lifecycle management
- Receives and executes tool call requests from the host process

---

## Engineering Practice Guide

### Enabling Computer Use

1. **Platform confirmation**: Computer Use currently **only supports macOS** — requires Enigo (Rust) and native Swift modules
2. **Granting permissions**:
   - Accessibility access: System Preferences → Privacy & Security → Accessibility → Add Claude Code
   - Screen Recording permission: SCContentFilter screenshots require Screen Recording authorization
   - TCC permissions: The native Swift module detects permission status via the TCC framework
3. **Subscription tier**: Requires max or pro+ subscription — Anthropic internal users can bypass this
4. **Confirm MCP Server**: `createComputerUseMcpServerForCli()` initializes the Computer Use MCP Server, using stdio transport to communicate with the host process

### Debugging Lock Conflicts

1. **Inspect the lock file**: View the contents of `~/.claude/computer-use.lock` — contains `sessionId`, `pid`, and `acquiredAt`
2. **Confirm whether the lock-holding process is alive**: Use the `pid` from the lock file to check process status — if the process has terminated, it is a stale lock
3. **Manually clean up stale locks**: After a process crash, the lock file may remain — safely deleting `~/.claude/computer-use.lock` releases the lock
4. **Forced reclamation on timeout**: Locks held longer than `HOLDER_STALE_MS = 60min` are automatically reclaimed — but stale locks within one hour require manual cleanup
5. **Zero-syscall check**: `isLockHeldLocally()` checks the lock via in-memory state, producing no system calls — suitable for high-frequency check scenarios

### ESC Hotkey Not Working

1. **Check CGEventTap permissions**: The ESC hotkey registers a system-level event listener via the macOS CGEventTap API — requires Accessibility permissions
2. **Confirm CFRunLoop is running**: Swift's `@MainActor` and Enigo operations are both dispatched to `DispatchQueue.main`, requiring `_drainMainRunLoop` to pump every 1ms. Check whether `retainPump()` has been called and whether the reference count is > 0
3. **Check the expected Escape window**: `notifyExpectedEscape()` creates a 100ms decay window — if the model just sent an Escape key operation, a user Escape within 100ms will not trigger an interrupt. Wait 100ms and retry
4. **Check for conflicts with other applications**: Escape is a global hotkey; if other applications have also registered Escape listeners, they may interfere with each other

### Per-Turn Cleanup Checklist

After each conversation turn, `cleanupComputerUseAfterTurn()` performs the following cleanup:

- [ ] Auto-unhide windows (5s timeout protection)
- [ ] Deregister CGEventTap Escape key listener
- [ ] Release `computer-use.lock` file lock
- [ ] Release CFRunLoop pump reference count

If cleanup is incomplete (e.g., the process is killed with SIGKILL), manually check: Is the lock file still present? Is the Escape listener still active? Have hidden windows been restored?

### Common Pitfalls

> **Multiple agents operating simultaneously will cause lock conflicts**: The O_EXCL atomic lock ensures only one session can control the mouse/keyboard at a time. In Swarm scenarios, multiple teammates cannot use Computer Use simultaneously — operations must be serialized through a permission synchronization mechanism.

> **ESC is a global hotkey**: Once registered, Escape is intercepted at the system level and **consumed by all applications** — including Vim's Escape, dialog cancel buttons, etc. This is a deliberate design choice for prompt injection defense, but it may affect normal user operations. Ensure the listener is deregistered when Computer Use is not active.

> **macOS only**: Windows and Linux do not support Computer Use — the native modules (`@ant/computer-use-input` and `@ant/computer-use-swift`) depend on macOS-specific APIs (CGEvent, SCContentFilter, NSWorkspace, TCC).

> **CFRunLoop pump CPU overhead**: `_drainMainRunLoop` is called every 1ms; be sure to stop it via `releasePump()` when Computer Use is not active. The 30s timeout protection stops it automatically, but 30 seconds of idle spinning still wastes CPU.


---

[← Swarm System](../34-Swarm系统/swarm-architecture-en.md) | [Index](../README_EN.md) | [DeepLink →](../36-DeepLink/deeplink-system-en.md)
