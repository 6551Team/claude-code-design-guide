# LSP Integration Architecture Document

> Claude Code v2.1.88 LSP (Language Server Protocol) Integration — Complete Technical Reference

---

### Design Philosophy

#### Why is LSP a Client, Not a Server?

Claude Code consumes language services from the IDE (type information, go-to-definition, find-references) rather than providing language services itself. It plays the "client" role in the LSP protocol — spawning and managing LSP server subprocesses (e.g., TypeScript's tsserver, Python's pylsp), sending requests to them, and receiving responses. This architecture reuses the existing mature language server ecosystem rather than reimplementing language analysis capabilities from scratch.

#### Why Multi-Instance Management?

Different languages/projects require different LSP servers. `LSPServerManager` maintains a mapping from file extensions to server instances (`createLSPServerManager()`), running multiple independent LSP processes simultaneously. `Promise.allSettled` is used during shutdown to ensure that a failure to close one server does not affect the others — this is key to fault tolerance.

#### Why Extension-Based Routing?

`ensureServerStarted(filePath)` automatically selects the correct LSP server based on the file extension — users should not have to manually configure "which language server to use for this .ts file". Routing is transparent: the caller simply passes a file path, and the manager handles server selection and on-demand startup automatically.

## LSP Client (services/lsp/LSPClient.ts)

### Interface
- `start`: Start the LSP server subprocess
- `initialize`: Send the LSP initialize request
- `sendRequest<T>`: Send a request and wait for a response
- `sendNotification`: Send a notification (no response required)
- `onNotification`: Register a notification handler
- `onRequest`: Register a request handler

### Implementation
- Communication protocol: **JSON-RPC over stdio** (subprocess stdin/stdout)
- Startup safety: Wait for successful spawn before using streams to prevent unhandled rejections

### Error Handling
| Exit Code | Meaning | Handling |
|-----------|---------|---------|
| 0 | Intentional shutdown | Normal cleanup |
| Non-zero | Crash | Triggers `onCrash` callback |

### Connection Management
- Deferred queue mechanism: Buffers requests when the connection is not yet ready
- Buffered requests are sent automatically once the connection is ready

---

## LSP Server Manager (LSPServerManager.ts)

### Multi-Instance Management
Routes to the corresponding LSP server instance based on file extension.

### Core Methods

#### initialize()
Loads all configured LSP servers and builds the mapping from extensions to server instances.

#### shutdown()
Stops all running servers. Uses `Promise.allSettled` for fault tolerance — a failure to close one server does not affect the others.

#### ensureServerStarted(filePath)
On-demand startup: ensures the corresponding LSP server is started based on the file path's extension.

#### sendRequest\<T\>(filePath, method, params)
Routes the request: finds the corresponding LSP server based on the file path, forwards the request, and returns the result.

#### File Lifecycle Notifications
| Method | Corresponding LSP Notification | Purpose |
|--------|-------------------------------|---------|
| `openFile` | `textDocument/didOpen` | Open a file |
| `changeFile` | `textDocument/didChange` | File content changed |
| `saveFile` | `textDocument/didSave` | Save a file |
| `closeFile` | `textDocument/didClose` | Close a file |

### File Tracking
Maintains an `openedFiles` Map (`fileUri` → `serverName`) to prevent sending duplicate `didOpen` notifications for the same file.

---

## Singleton Management (manager.ts)

### Lifecycle Functions

#### initializeLspServerManager()
Creates a manager instance and initializes it asynchronously (non-blocking; does not wait for all servers to finish starting).

#### reinitializeLspServerManager()
Forces reinitialization on plugin refresh — shuts down the old instance and creates a new one.

#### shutdownLspServerManager()
Best-effort shutdown; errors are swallowed and not propagated.

#### waitForInitialization()
Waits for initialization to complete, with a **30-second timeout**.

#### isLspConnected()
Checks whether at least one LSP server is in a healthy state.

### generation Counter
Used to invalidate stale promises. When the manager is reinitialized, old initialization promises are detected and discarded via the generation counter.

### isBareMode()
Detects whether the process is running in script invocation mode (bare mode); LSP initialization is skipped in bare mode.

---

## LSPTool Operations

### Code Navigation
| Operation | Purpose |
|-----------|---------|
| `goToDefinition` | Jump to definition |
| `findReferences` | Find references |
| `goToImplementation` | Jump to implementation |

### Code Information
| Operation | Purpose |
|-----------|---------|
| `hover` | Hover information |
| `documentSymbol` | Document symbols |
| `workspaceSymbol` | Workspace symbol search |

### Call Hierarchy
| Operation | Purpose |
|-----------|---------|
| `prepareCallHierarchy` | Prepare call hierarchy |
| `incomingCalls` | Incoming calls |
| `outgoingCalls` | Outgoing calls |

### Constraints

| Constraint | Value |
|------------|-------|
| File size limit | 10MB |
| Line number format | 1-based (both line and column numbers start at 1) |

---

## Engineering Practice Guide

### Configuring an LSP Server

**Checklist:**

1. **Add the LSP server configuration**: Declare the LSP server's command, arguments, and startup mode in settings
2. **Specify the language/extension mapping**: `LSPServerManager` routes requests to the corresponding LSP server based on file extension
3. **Verify server availability**: Ensure the LSP server binary is on PATH or specify an absolute path
4. **Integrate via plugins**: `lspPluginIntegration.ts` and `lspRecommendation.ts` support plugin registration of LSP servers

**Routing mechanism**: `ensureServerStarted(filePath)` automatically selects the LSP server based on file extension — the caller only passes a file path, and the manager handles server selection and on-demand startup automatically.

### Debugging LSP Connections

**Troubleshooting steps:**

1. **Check whether the LSP process is alive**:
   - LSP servers run as subprocesses communicating via stdio (JSON-RPC over stdin/stdout)
   - Exit code 0 = intentional shutdown (normal cleanup); non-zero = crash (triggers `onCrash` callback)
2. **Check for initialization timeout**: `waitForInitialization()` has a **30-second timeout**; a timeout indicates abnormal LSP server startup
3. **Check connection status**: `isLspConnected()` checks whether at least one LSP server is healthy
4. **Check file tracking**: The `openedFiles` Map tracks opened files; verify whether a `didOpen` notification has been sent for the target file
5. **Check the generation counter**: When the manager is reinitialized, old promises are invalidated via the generation counter; check for stale promises
6. **Check bare mode**: LSP initialization is skipped when `isBareMode()` returns true

**Key source locations**:
- `LSPClient.ts` — JSON-RPC protocol encapsulation, deferred queue mechanism
- `LSPServerManager.ts` — multi-instance routing, file lifecycle notifications
- `manager.ts` — singleton management, generation counter, 30-second timeout

**LSP Diagnostic Handling Notes** (source: `passiveFeedback.ts`):
- When a diagnostic handler fails consecutively, a warning is emitted: "WARNING: LSP diagnostic handler for {serverName} has failed {count} times consecutively"
- The TODO for integrating LSP functionality into the compact flow is not yet complete (source comment: "TODO: Integrate with compact - call closeFile() when compact removes files from context")

### File Lifecycle Management

| LSP Notification | When Triggered | Method |
|-----------------|---------------|--------|
| `textDocument/didOpen` | File opened for the first time | `openFile()` |
| `textDocument/didChange` | File content changed | `changeFile()` |
| `textDocument/didSave` | File saved | `saveFile()` |
| `textDocument/didClose` | File closed | `closeFile()` |

**Duplicate notification prevention**: The `openedFiles` Map records `fileUri → serverName` to prevent sending duplicate `didOpen` notifications for the same file.

### Extending LSP Functionality

**Available operations at a glance:**
- Code navigation: `goToDefinition`, `findReferences`, `goToImplementation`
- Code information: `hover`, `documentSymbol`, `workspaceSymbol`
- Call hierarchy: `prepareCallHierarchy`, `incomingCalls`, `outgoingCalls`

**File size limit**: 10MB — files exceeding this limit are not sent to the LSP server.

### Common Pitfalls

| Pitfall | Details | Solution |
|---------|---------|---------|
| LSP server crashes must not affect core functionality | A crash triggers the `onCrash` callback; related features degrade silently | `shutdownLspServerManager()` is best-effort; errors are not propagated |
| Concurrent multi-instance requires resource management | Multiple independent LSP processes run for different languages/projects | `Promise.allSettled` ensures a single server shutdown failure does not affect the others |
| Reinitialization on plugin refresh | `reinitializeLspServerManager()` shuts down the old instance and creates a new one | The generation counter invalidates old promises |
| Bare mode skips LSP | Code intelligence is not needed in script invocation mode | The `isBareMode()` check runs before initialization |
| Deferred queue | Requests are buffered when the connection is not ready | Sent automatically once the connection is ready, but may cause response delays |


---

[← OAuth & Auth](../22-OAuth与认证/oauth-auth-en.md) | [Index](../README_EN.md) | [Sandbox System →](../24-沙箱系统/sandbox-system-en.md)
