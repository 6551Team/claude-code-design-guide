# Bridge 协议架构

Bridge 协议由 33 个源文件组成，实现了 Claude Code 与远程环境之间的双向通信桥接。

### 设计理念

#### 为什么 33 个文件的复杂协议?

Bridge 连接 IDE (VS Code/JetBrains) 和 Claude Code 核心,需要处理消息序列化、状态同步、连接管理、错误恢复等问题。每个文件承担单一职责:API 客户端 (`bridgeApi.ts`)、认证 (`bridgeConfig.ts`)、特性门控 (`bridgeEnabled.ts`)、消息路由与去重 (`bridgeMessaging.ts`)、传输层 (WebSocket/SSE/Hybrid) 等。这种拆分不是过度工程,而是通信协议的内在复杂性要求——任何一个子问题 (如"消息丢失怎么办"、"认证过期怎么办"、"WebSocket 不可用怎么办") 都需要独立的解决方案。

#### 为什么 REST + WebSocket 双通道?

两种通信模式天然适配不同的交互模式:REST 处理请求/响应语义 (创建会话、发送消息、权限响应等一次性操作),WebSocket 处理事件流 (流式输出、状态变更、权限请求推送等持续性通信)。源码中 `SSETransport.ts` (712 行) 和 `WebSocketTransport.ts` (800 行) 的体量说明了每种传输层独立的复杂度。`HybridTransport.ts` 则在两者之间做运行时切换和降级。

#### 为什么 BoundedUUIDSet 去重?

源码注释写道:"FIFO-bounded set backed by a circular buffer. Evicts the oldest entry when capacity is reached, keeping memory usage constant at O(capacity)"。网络不可靠可能导致消息重发,尤其是断线重连场景下 seq-num 协商可能产生重复。BoundedUUIDSet 用固定大小 (默认 2000) 的环形缓冲区跟踪已处理的消息 UUID。"Bounded"是关键——无限增长的 Set 会内存泄漏,而旧消息的去重已经没有意义,因为外部排序机制 (`lastWrittenIndexRef`) 提供了主要去重保证,BoundedUUIDSet 只是"secondary safety net for echo filtering and race-condition dedup"。

#### 为什么可靠传输层?

IDE 集成不能丢消息——文件编辑指令丢失意味着代码损坏。所以 Bridge 实现了完整的可靠传输:WebSocket 层有自动重连和消息缓冲,SSE 层有基于 sequence number 的断点续传,两者都有心跳检测和休眠唤醒恢复。

---

## Core API (bridgeApi.ts)

### 工厂函数

**createBridgeApiClient(deps)** 是核心工厂函数，创建具有认证重试能力的 API 客户端。`deps` 参数注入认证、日志等依赖。

### 方法集

- **registerBridgeEnvironment()**: 注册当前环境到 Bridge 服务
- **pollForWork()**: 长轮询获取待执行的工作任务
- **acknowledgeWork()**: 确认已接收工作任务
- **stopWork()**: 停止当前正在执行的工作
- **deregisterEnvironment()**: 注销当前环境
- **archiveSession()**: 归档已完成的会话
- **reconnectSession()**: 重连中断的会话
- **heartbeatWork()**: 发送心跳保持工作活跃状态
- **sendPermissionResponseEvent()**: 发送权限响应事件

### 错误处理

**BridgeFatalError** 类封装不可恢复的致命错误：

- **401**: 未认证
- **403**: 无权限
- **404**: 资源不存在
- **410**: 资源已过期 (Gone)

### 安全验证

**validateBridgeId()**: 验证 Bridge ID 格式，防止路径遍历攻击（path traversal）。

### 错误分类工具

- **isExpiredErrorType()**: 判断错误是否为过期类型
- **isSuppressible403()**: 判断 403 错误是否可以静默处理（例如已知的权限限制场景）

---

## Authentication (bridgeConfig.ts)

### Token 获取

- **getBridgeTokenOverride()**: 读取 ant-only 环境变量中的 token 覆盖值（仅限内部使用）
- **getBridgeAccessToken()**: 获取访问令牌，优先使用 override，否则从系统 keychain 获取

### URL 解析

- **getBridgeBaseUrl()**: 解析 Bridge API 的基础 URL，支持环境变量覆盖

---

## Feature Gates (bridgeEnabled.ts)

### 主要开关

- **isBridgeEnabled()**: Bridge 是否启用，条件为 OAuth 订阅者 + GrowthBook 特性标志
- **getBridgeDisabledReason()**: 返回 Bridge 未启用的诊断原因消息

### 子功能开关

- **isEnvLessBridgeEnabled()**: 无环境 Bridge 模式是否启用
- **isCseShimEnabled()**: CSE shim 兼容层是否启用
- **getCcrAutoConnectDefault()**: 获取 CCR 自动连接的默认值
- **isCcrMirrorEnabled()**: CCR 镜像功能是否启用

---

## Messaging (bridgeMessaging.ts)

### 消息去重

**BoundedUUIDSet**: 基于循环缓冲区的 FIFO 有界去重集合，防止重复消息处理。当集合满时自动淘汰最早的条目。

### 消息路由

**handleIngressMessage()**: 核心消息路由函数，处理来自 WebSocket 的入站消息，根据消息类型分发到对应处理器。

### 类型守卫

- **isSDKMessage()**: 判断是否为 SDK 消息
- **isSDKControlResponse()**: 判断是否为 SDK 控制响应
- **isSDKControlRequest()**: 判断是否为 SDK 控制请求

### 控制请求处理

**handleServerControlRequest()**: 处理服务端发起的控制请求：

- **权限请求**: 服务端请求客户端确认权限
- **模型切换**: 服务端请求切换使用的模型
- **中断请求**: 服务端请求中断当前操作

### 会话归档

**makeResultMessage()**: 构造会话归档的信封消息，用于将完成的会话数据打包发送。

---

## Transport Layer

Bridge 提供三种传输层实现，适应不同的网络环境。

### WebSocketTransport.ts (800 lines)

全双工 WebSocket 传输，功能最完整：

- **自动重连**: 断线后自动重连，带指数退避策略
- **Ping/Pong**: 心跳检测连接存活
- **消息缓冲**: 断连期间缓冲待发送消息
- **指数退避**: 重连间隔指数增长，避免服务端过载
- **休眠检测**: 检测系统休眠唤醒后主动重连
- **Keep-alive**: 保持连接活跃的定期心跳

### SSETransport.ts (712 lines)

基于 Server-Sent Events 的半双工传输：

- **SSE 读取**: 使用 SSE 接收服务端推送消息
- **HTTP POST 写入**: 通过 HTTP POST 发送客户端消息
- **活性超时**: 检测连接是否仍然活跃
- **序列号恢复**: 基于 sequence number 的断点续传，确保消息不丢失

### HybridTransport.ts

传输选择逻辑层：

- 根据环境条件和服务端能力选择最合适的传输方式
- 支持运行时在 WebSocket 和 SSE 之间切换
- 处理降级场景（WebSocket 不可用时降级到 SSE）

---

## Session Management

### 会话 CRUD

- **createBridgeSession()**: `POST /v1/sessions` 创建新会话
- **getBridgeSession()**: `GET /v1/sessions/{id}` 获取会话详情
- **archiveBridgeSession()**: `POST /v1/sessions/{id}/archive` 归档会话
- **updateBridgeSessionTitle()**: `PATCH /v1/sessions/{id}` 更新会话标题

### SpawnMode

```typescript
type SpawnMode = 'single-session' | 'worktree' | 'same-dir'
```

- **single-session**: 单会话模式，一个环境一个会话
- **worktree**: 使用 git worktree 为每个会话创建独立工作目录
- **same-dir**: 多个会话共享同一目录

### BridgeWorkerType

```typescript
type BridgeWorkerType = 'claude_code' | 'claude_code_assistant'
```

- **claude_code**: 标准 Claude Code 工作进程
- **claude_code_assistant**: 辅助型工作进程（如 swarm teammate）

---

## Types

Bridge 协议定义了丰富的类型系统支撑整个通信流程。

### 工作相关

- **WorkData**: 工作任务的完整数据载荷
- **WorkResponse**: 工作执行结果的响应结构
- **WorkSecret**: 工作相关的加密密钥和敏感信息

### 会话活动

- **SessionDoneStatus**: 会话完成状态枚举
- **SessionActivityType**: 会话活动类型（消息、工具调用、错误等）
- **SessionActivity**: 会话活动记录，包含时间戳和详情

### 配置与句柄

- **BridgeConfig**: Bridge 完整配置结构
- **BridgeApiClient**: API 客户端实例类型
- **SessionHandle**: 会话句柄，用于引用活跃会话
- **SessionSpawner**: 会话生成器，封装会话创建逻辑

---

## 工程实践指南

### IDE 集成开发

1. **实现 Bridge 客户端**: 调用 `createBridgeApiClient(deps)` 创建 API 客户端,注入认证和日志依赖
2. **连接双通道**:
   - REST 通道: 用于请求/响应语义操作 (创建会话、发送消息、权限响应等)
   - WebSocket 通道: 用于事件流 (流式输出、状态变更、权限请求推送等)
3. **处理消息序列化**: 使用 `isSDKMessage()`/`isSDKControlResponse()`/`isSDKControlRequest()` 类型守卫确保消息类型正确
4. **处理认证过期**: 监听 `BridgeFatalError` 的 401 错误码,触发重新认证流程
5. **选择传输层**: WebSocket 优先;不可用时通过 `HybridTransport` 自动降级到 SSE

### 调试消息丢失

1. **检查 BoundedUUIDSet 是否误判重复**:
   - BoundedUUIDSet 容量默认 2000,使用环形缓冲区——当消息量大时,旧 UUID 被淘汰
   - 如果淘汰后同一 UUID 的消息再次到达 (极端重发场景),会被当作新消息处理——这通常不是问题
   - 真正的风险是:两条不同消息被误判为重复——检查 UUID 生成是否有碰撞
2. **检查 WebSocket 重连状态**:
   - 断连期间消息会被缓冲,重连后批量发送——检查缓冲区是否溢出
   - 重连后 sequence number 协商可能产生重复——BoundedUUIDSet 正是为此设计的二级去重保障
3. **检查 SSE 断点续传**:
   - SSE 传输层基于 sequence number 恢复——检查 `lastWrittenIndexRef` 是否正确更新
   - 如果 sequence number 不连续,说明有消息在传输层丢失

### 性能优化

1. **批量消息发送**: 多条消息尽量合并为一次网络请求,减少 HTTP 往返开销
2. **合理设置 BoundedUUIDSet 大小**: 默认 2000 适合大多数场景;高频消息场景可适当增大,但注意内存开销为 O(capacity)
3. **心跳间隔调优**: 默认心跳检测连接存活,过于频繁会增加网络开销,过于稀疏会延迟断线检测
4. **休眠唤醒处理**: WebSocket 和 SSE 传输层都有休眠检测,系统唤醒后主动重连——确保这个逻辑正常工作,否则笔记本合盖再开后会显示"已断开"

### 常见陷阱

> **消息顺序很重要**: 不要假设乱序到达是正常的。Bridge 协议保证消息按序传输,如果观察到乱序,说明传输层有 bug。`lastWrittenIndexRef` 提供主要的顺序保证,BoundedUUIDSet 只是辅助去重。

> **Bridge 消息有最大大小限制**: 单条消息不能无限大——特别是包含大文件内容的工具调用结果。如果消息超过限制,序列化会失败。拆分大消息或压缩内容。

> **认证过期的连锁反应**: Bridge 认证令牌过期 (401) 后,所有正在进行的请求都会失败。`createBridgeApiClient` 内置了认证重试能力,但如果令牌彻底无效 (而非临时过期),会抛出 `BridgeFatalError`。不要在循环中重试 fatal error。

> **validateBridgeId() 必须调用**: 这不是可选的验证——Bridge ID 直接用于构建 API URL 路径,不验证可能导致路径遍历攻击。任何接收外部 Bridge ID 的入口点都必须先调用此函数。


---

[← 远程会话](../30-远程会话/remote-session.md) | [目录](../README.md) | [Buddy 系统 →](../32-Buddy系统/buddy-system.md)
