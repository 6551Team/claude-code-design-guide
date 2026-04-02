# Bridge Protocol Architecture

The Bridge protocol consists of 33 source files and implements bidirectional communication bridging between Claude Code and remote environments.

### Design Philosophy

#### Why a 33-file complex protocol?

Bridge connects the IDE (VS Code/JetBrains) and the Claude Code core, and must handle message serialization, state synchronization, connection management, error recovery, and more. Each file carries a single responsibility: API client (`bridgeApi.ts`), authentication (`bridgeConfig.ts`), feature gating (`bridgeEnabled.ts`), message routing and deduplication (`bridgeMessaging.ts`), transport layer (WebSocket/SSE/Hybrid), and so on. This decomposition is not over-engineering — it is demanded by the intrinsic complexity of a communication protocol. Each sub-problem ("what if a message is lost?", "what if authentication expires?", "what if WebSocket is unavailable?") requires its own independent solution.

#### Why REST + WebSocket dual channels?

The two communication modes naturally match different interaction patterns: REST handles request/response semantics (creating sessions, sending messages, permission responses, and other one-shot operations), while WebSocket handles event streams (streaming output, state changes, permission-request pushes, and other continuous communications). The sheer size of `SSETransport.ts` (712 lines) and `WebSocketTransport.ts` (800 lines) in the source code reflects the independent complexity of each transport layer. `HybridTransport.ts` performs runtime switching and fallback between the two.

#### Why BoundedUUIDSet for deduplication?

The source code comments state: "FIFO-bounded set backed by a circular buffer. Evicts the oldest entry when capacity is reached, keeping memory usage constant at O(capacity)". Network unreliability can cause message retransmission, especially during reconnection scenarios where seq-num negotiation may produce duplicates. BoundedUUIDSet uses a fixed-size (default 2000) circular buffer to track already-processed message UUIDs. "Bounded" is the key — an unbounded, ever-growing Set would leak memory, and deduplicating stale messages is already meaningless because the external ordering mechanism (`lastWrittenIndexRef`) provides the primary deduplication guarantee. BoundedUUIDSet is only a "secondary safety net for echo filtering and race-condition dedup".

#### Why a reliable transport layer?

IDE integration cannot drop messages — a lost file-edit instruction means corrupted code. Therefore Bridge implements a complete reliable transport: the WebSocket layer has automatic reconnection and message buffering, the SSE layer has sequence-number-based resume-from-breakpoint, and both have heartbeat detection and sleep/wake recovery.

---

## Core API (bridgeApi.ts)

### Factory Function

**createBridgeApiClient(deps)** is the core factory function that creates an API client with authentication-retry capability. The `deps` parameter injects dependencies such as authentication and logging.

### Method Set

- **registerBridgeEnvironment()**: Register the current environment with the Bridge service
- **pollForWork()**: Long-poll to retrieve pending work tasks
- **acknowledgeWork()**: Acknowledge receipt of a work task
- **stopWork()**: Stop the currently executing work
- **deregisterEnvironment()**: Deregister the current environment
- **archiveSession()**: Archive a completed session
- **reconnectSession()**: Reconnect an interrupted session
- **heartbeatWork()**: Send a heartbeat to keep work alive
- **sendPermissionResponseEvent()**: Send a permission response event

### Error Handling

**BridgeFatalError** class encapsulates unrecoverable fatal errors:

- **401**: Unauthenticated
- **403**: Forbidden
- **404**: Resource not found
- **410**: Resource expired (Gone)

### Security Validation

**validateBridgeId()**: Validates the Bridge ID format to prevent path traversal attacks.

### Error Classification Utilities

- **isExpiredErrorType()**: Determines whether an error is of the expired type
- **isSuppressible403()**: Determines whether a 403 error can be silently suppressed (e.g., in known permission-restriction scenarios)

---

## Authentication (bridgeConfig.ts)

### Token Retrieval

- **getBridgeTokenOverride()**: Reads a token override value from ant-only environment variables (internal use only)
- **getBridgeAccessToken()**: Retrieves an access token, preferring the override; otherwise fetches from the system keychain

### URL Resolution

- **getBridgeBaseUrl()**: Resolves the base URL for the Bridge API, with support for environment variable overrides

---

## Feature Gates (bridgeEnabled.ts)

### Primary Toggle

- **isBridgeEnabled()**: Whether Bridge is enabled — requires an OAuth subscriber plus a GrowthBook feature flag
- **getBridgeDisabledReason()**: Returns a diagnostic reason message explaining why Bridge is not enabled

### Sub-Feature Toggles

- **isEnvLessBridgeEnabled()**: Whether env-less Bridge mode is enabled
- **isCseShimEnabled()**: Whether the CSE shim compatibility layer is enabled
- **getCcrAutoConnectDefault()**: Retrieves the default value for CCR auto-connect
- **isCcrMirrorEnabled()**: Whether the CCR mirror feature is enabled

---

## Messaging (bridgeMessaging.ts)

### Message Deduplication

**BoundedUUIDSet**: A FIFO bounded deduplication set backed by a circular buffer, preventing duplicate message processing. When the set is full, the oldest entry is automatically evicted.

### Message Routing

**handleIngressMessage()**: Core message routing function that processes inbound messages from WebSocket and dispatches them to the corresponding handler based on message type.

### Type Guards

- **isSDKMessage()**: Determines whether a value is an SDK message
- **isSDKControlResponse()**: Determines whether a value is an SDK control response
- **isSDKControlRequest()**: Determines whether a value is an SDK control request

### Control Request Handling

**handleServerControlRequest()**: Handles control requests initiated by the server:

- **Permission request**: Server requests client confirmation of a permission
- **Model switch**: Server requests a switch of the active model
- **Abort request**: Server requests interruption of the current operation

### Session Archival

**makeResultMessage()**: Constructs the envelope message for session archival, used to package and send completed session data.

---

## Transport Layer

Bridge provides three transport layer implementations to accommodate different network environments.

### WebSocketTransport.ts (800 lines)

Full-duplex WebSocket transport — the most feature-complete option:

- **Auto-reconnect**: Automatically reconnects after disconnection with an exponential back-off strategy
- **Ping/Pong**: Heartbeat detection to verify connection liveness
- **Message buffering**: Buffers outgoing messages during disconnection periods
- **Exponential back-off**: Reconnect intervals grow exponentially to avoid server overload
- **Sleep detection**: Detects system sleep/wake events and proactively reconnects
- **Keep-alive**: Periodic heartbeats to keep the connection active

### SSETransport.ts (712 lines)

Half-duplex transport based on Server-Sent Events:

- **SSE reading**: Uses SSE to receive server-pushed messages
- **HTTP POST writing**: Sends client messages via HTTP POST
- **Liveness timeout**: Detects whether the connection is still active
- **Sequence-number resume**: Resume-from-breakpoint based on sequence numbers to ensure no messages are lost

### HybridTransport.ts

Transport selection logic layer:

- Selects the most appropriate transport based on environment conditions and server capabilities
- Supports runtime switching between WebSocket and SSE
- Handles degradation scenarios (falls back to SSE when WebSocket is unavailable)

---

## Session Management

### Session CRUD

- **createBridgeSession()**: `POST /v1/sessions` — create a new session
- **getBridgeSession()**: `GET /v1/sessions/{id}` — retrieve session details
- **archiveBridgeSession()**: `POST /v1/sessions/{id}/archive` — archive a session
- **updateBridgeSessionTitle()**: `PATCH /v1/sessions/{id}` — update session title

### SpawnMode

```typescript
type SpawnMode = 'single-session' | 'worktree' | 'same-dir'
```

- **single-session**: Single-session mode — one environment, one session
- **worktree**: Uses git worktree to create an independent working directory for each session
- **same-dir**: Multiple sessions share the same directory

### BridgeWorkerType

```typescript
type BridgeWorkerType = 'claude_code' | 'claude_code_assistant'
```

- **claude_code**: Standard Claude Code worker process
- **claude_code_assistant**: Auxiliary worker process (e.g., a swarm teammate)

---

## Types

The Bridge protocol defines a rich type system to underpin the entire communication flow.

### Work-Related

- **WorkData**: Complete data payload for a work task
- **WorkResponse**: Response structure for a work execution result
- **WorkSecret**: Encryption keys and sensitive information related to work

### Session Activity

- **SessionDoneStatus**: Enum of session completion statuses
- **SessionActivityType**: Session activity types (messages, tool calls, errors, etc.)
- **SessionActivity**: Session activity record, including timestamp and details

### Configuration and Handles

- **BridgeConfig**: Complete Bridge configuration structure
- **BridgeApiClient**: API client instance type
- **SessionHandle**: Session handle used to reference an active session
- **SessionSpawner**: Session spawner that encapsulates session creation logic

---

## Engineering Practice Guide

### IDE Integration Development

1. **Implement the Bridge client**: Call `createBridgeApiClient(deps)` to create the API client, injecting authentication and logging dependencies
2. **Connect the dual channels**:
   - REST channel: for request/response semantic operations (creating sessions, sending messages, permission responses, etc.)
   - WebSocket channel: for event streams (streaming output, state changes, permission-request pushes, etc.)
3. **Handle message serialization**: Use the `isSDKMessage()` / `isSDKControlResponse()` / `isSDKControlRequest()` type guards to ensure correct message types
4. **Handle authentication expiry**: Listen for the 401 error code on `BridgeFatalError` and trigger a re-authentication flow
5. **Choose the transport layer**: Prefer WebSocket; when unavailable, let `HybridTransport` automatically fall back to SSE

### Debugging Message Loss

1. **Check whether BoundedUUIDSet is incorrectly flagging duplicates**:
   - BoundedUUIDSet capacity defaults to 2000 and uses a circular buffer — when message volume is high, old UUIDs are evicted
   - If a message with the same UUID arrives again after eviction (extreme retransmission scenario), it will be treated as a new message — this is generally not a problem
   - The real risk is: two different messages being falsely judged as duplicates — check whether UUID generation has collisions
2. **Check WebSocket reconnection state**:
   - Messages are buffered during disconnection and sent in bulk after reconnection — check whether the buffer has overflowed
   - After reconnection, sequence-number negotiation may produce duplicates — BoundedUUIDSet exists precisely as a secondary deduplication safeguard for this
3. **Check SSE resume-from-breakpoint**:
   - The SSE transport layer resumes based on sequence numbers — check whether `lastWrittenIndexRef` is being updated correctly
   - If sequence numbers are non-contiguous, messages were lost at the transport layer

### Performance Optimization

1. **Batch message sending**: Consolidate multiple messages into a single network request where possible to reduce HTTP round-trip overhead
2. **Set BoundedUUIDSet size appropriately**: The default of 2000 suits most scenarios; increase it for high-frequency message scenarios, but note that memory usage is O(capacity)
3. **Tune heartbeat intervals**: The default heartbeat detects connection liveness; too frequent increases network overhead, too sparse delays disconnection detection
4. **Handle sleep/wake events**: Both WebSocket and SSE transport layers have sleep detection and proactively reconnect after system wake — make sure this logic works correctly, otherwise the UI will show "Disconnected" after closing and reopening a laptop lid

### Common Pitfalls

> **Message order matters**: Do not assume out-of-order arrival is normal. The Bridge protocol guarantees in-order message delivery; if you observe out-of-order messages, there is a bug in the transport layer. `lastWrittenIndexRef` provides the primary ordering guarantee; BoundedUUIDSet is only an auxiliary deduplication mechanism.

> **Bridge messages have a maximum size limit**: A single message cannot be arbitrarily large — especially tool-call results that include large file contents. If a message exceeds the limit, serialization will fail. Split large messages or compress the content.

> **Cascading effects of authentication expiry**: After a Bridge authentication token expires (401), all in-flight requests will fail. `createBridgeApiClient` has built-in authentication retry capability, but if the token is completely invalid (rather than temporarily expired), it will throw a `BridgeFatalError`. Do not retry fatal errors in a loop.

> **validateBridgeId() must be called**: This is not optional validation — the Bridge ID is used directly to construct API URL paths, and skipping validation can lead to path traversal attacks. Any entry point that receives an external Bridge ID must call this function first.


---

[← Remote Session](../30-远程会话/remote-session-en.md) | [Index](../README_EN.md) | [Buddy System →](../32-Buddy系统/buddy-system-en.md)
