# LSP集成架构文档

> Claude Code v2.1.88 LSP (Language Server Protocol) 集成完整技术参考

---

### 设计理念

#### 为什么 LSP 是客户端而不是服务端？

Claude Code 消费 IDE 的语言服务（类型信息、定义跳转、引用查找），而不是提供语言服务。它是 LSP 协议中的"客户端"角色——启动并管理 LSP 服务器子进程（如 TypeScript 的 tsserver、Python 的 pylsp），向它们发送请求并接收响应。这种架构复用了已有的成熟语言服务器生态，而非重新实现语言分析能力。

#### 为什么多实例管理？

不同语言/项目需要不同的 LSP 服务器，`LSPServerManager` 维护扩展名到服务器实例的映射（`createLSPServerManager()`），同时运行多个独立的 LSP 进程。`Promise.allSettled` 用于关闭操作，确保单个服务器关闭失败不影响其他服务器——这是容错性的关键。

#### 为什么扩展名路由？

`ensureServerStarted(filePath)` 根据文件扩展名自动选择正确的 LSP 服务器——用户不应手动配置"这个 .ts 文件用哪个语言服务器"。路由是透明的：调用者只需传入文件路径，管理器自动处理服务器选择和按需启动。

## LSP客户端 (services/lsp/LSPClient.ts)

### 接口
- `start`: 启动 LSP 服务器子进程
- `initialize`: 发送 LSP initialize 请求
- `sendRequest<T>`: 发送请求并等待响应
- `sendNotification`: 发送通知（无需响应）
- `onNotification`: 注册通知处理器
- `onRequest`: 注册请求处理器

### 实现
- 通信协议: **JSON-RPC over stdio**（子进程 stdin/stdout）
- 启动安全: 等待成功 spawn 后再使用流，防止 unhandled rejections

### 错误处理
| 退出码 | 含义 | 处理 |
|--------|------|------|
| 0 | 有意关闭 | 正常清理 |
| 非零 | 崩溃 | 触发 `onCrash` 回调 |

### 连接管理
- 延迟队列机制: 连接未就绪时缓冲请求
- 连接就绪后自动发送缓冲的请求

---

## LSP服务管理器 (LSPServerManager.ts)

### 多实例管理
按文件扩展名路由到对应的 LSP 服务器实例。

### 核心方法

#### initialize()
加载所有配置的 LSP 服务器，构建扩展名到服务器的映射关系。

#### shutdown()
停止所有运行中的服务器。使用 `Promise.allSettled` 实现容错，单个服务器关闭失败不影响其他服务器。

#### ensureServerStarted(filePath)
按需启动: 根据文件路径的扩展名，确保对应的 LSP 服务器已启动。

#### sendRequest\<T\>(filePath, method, params)
路由请求: 根据文件路径找到对应的 LSP 服务器，转发请求并返回结果。

#### 文件生命周期通知
| 方法 | 对应 LSP 通知 | 用途 |
|------|-------------|------|
| `openFile` | `textDocument/didOpen` | 打开文件 |
| `changeFile` | `textDocument/didChange` | 文件内容变更 |
| `saveFile` | `textDocument/didSave` | 保存文件 |
| `closeFile` | `textDocument/didClose` | 关闭文件 |

### 文件跟踪
维护 `openedFiles` Map (`fileUri` → `serverName`)，防止对同一文件重复发送 `didOpen` 通知。

---

## 单例管理 (manager.ts)

### 生命周期函数

#### initializeLspServerManager()
创建管理器实例并异步初始化（非阻塞，不等待所有服务器启动完成）。

#### reinitializeLspServerManager()
插件刷新时强制重新初始化，关闭旧实例并创建新实例。

#### shutdownLspServerManager()
尽力关闭（best-effort），错误被吞没不传播。

#### waitForInitialization()
等待初始化完成，设有 **30秒超时**。

#### isLspConnected()
检查是否至少有一个 LSP 服务器处于健康状态。

### generation 计数器
用于使陈旧的 promise 失效。当管理器被重新初始化时，旧的初始化 promise 通过 generation 计数器检测并丢弃。

### isBareMode()
检测是否为脚本调用模式（bare mode），脚本模式下跳过 LSP 初始化。

---

## LSPTool操作

### 代码导航
| 操作 | 用途 |
|------|------|
| `goToDefinition` | 跳转到定义 |
| `findReferences` | 查找引用 |
| `goToImplementation` | 跳转到实现 |

### 代码信息
| 操作 | 用途 |
|------|------|
| `hover` | 悬停信息 |
| `documentSymbol` | 文档符号 |
| `workspaceSymbol` | 工作区符号搜索 |

### 调用层次
| 操作 | 用途 |
|------|------|
| `prepareCallHierarchy` | 准备调用层次 |
| `incomingCalls` | 入站调用 |
| `outgoingCalls` | 出站调用 |

### 限制

| 约束 | 值 |
|------|-----|
| 文件大小限制 | 10MB |
| 行号格式 | 1-based（行号和列号均从 1 开始） |

---

## 工程实践指南

### 配置 LSP 服务器

**步骤清单：**

1. **添加 LSP 服务器配置**：在设置中声明 LSP 服务器的命令、参数和启动方式
2. **指定语言/扩展名映射**：`LSPServerManager` 根据文件扩展名路由请求到对应的 LSP 服务器
3. **确认服务器可用性**：确保 LSP 服务器二进制文件在 PATH 中或指定绝对路径
4. **通过插件集成**：`lspPluginIntegration.ts` 和 `lspRecommendation.ts` 支持插件注册 LSP 服务器

**路由机制**：`ensureServerStarted(filePath)` 根据文件扩展名自动选择 LSP 服务器——调用者只需传入文件路径，管理器自动处理服务器选择和按需启动。

### 调试 LSP 连接

**排查步骤：**

1. **检查 LSP 进程是否存活**：
   - LSP 服务器作为子进程通过 stdio 通信（JSON-RPC over stdin/stdout）
   - 退出码 0 = 有意关闭（正常清理），非零 = 崩溃（触发 `onCrash` 回调）
2. **检查初始化超时**：`waitForInitialization()` 设有 **30 秒超时**，超时说明 LSP 服务器启动异常
3. **检查连接状态**：`isLspConnected()` 检查是否至少有一个 LSP 服务器健康
4. **检查文件追踪**：`openedFiles` Map 追踪已打开文件，确认目标文件是否已发送 `didOpen` 通知
5. **检查 generation 计数器**：管理器重新初始化时旧的 promise 通过 generation 计数器失效，确认是否有陈旧 promise
6. **检查 bare 模式**：`isBareMode()` 为 true 时跳过 LSP 初始化

**源码关键位置**：
- `LSPClient.ts` — JSON-RPC 协议封装，延迟队列机制
- `LSPServerManager.ts` — 多实例路由，文件生命周期通知
- `manager.ts` — 单例管理，generation 计数器，30 秒超时

**LSP 诊断处理注意事项**（源码 `passiveFeedback.ts`）：
- 诊断 handler 连续失败时会输出警告："WARNING: LSP diagnostic handler for {serverName} has failed {count} times consecutively"
- LSP 功能集成到 compact 流程的 TODO 尚未完成（源码注释："TODO: Integrate with compact - call closeFile() when compact removes files from context"）

### 文件生命周期管理

| LSP 通知 | 触发时机 | 方法 |
|----------|---------|------|
| `textDocument/didOpen` | 文件首次打开 | `openFile()` |
| `textDocument/didChange` | 文件内容变更 | `changeFile()` |
| `textDocument/didSave` | 文件保存 | `saveFile()` |
| `textDocument/didClose` | 文件关闭 | `closeFile()` |

**防重复通知**：`openedFiles` Map 记录 `fileUri → serverName`，防止对同一文件重复发送 `didOpen`。

### 扩展 LSP 功能

**可用操作一览：**
- 代码导航：`goToDefinition`、`findReferences`、`goToImplementation`
- 代码信息：`hover`、`documentSymbol`、`workspaceSymbol`
- 调用层次：`prepareCallHierarchy`、`incomingCalls`、`outgoingCalls`

**文件大小限制**：10MB——超过此限制的文件不发送给 LSP 服务器。

### 常见陷阱

| 陷阱 | 详情 | 解决方案 |
|------|------|----------|
| LSP 服务器崩溃不应影响主功能 | 崩溃时触发 `onCrash` 回调，相关功能静默降级 | `shutdownLspServerManager()` 是 best-effort，错误不传播 |
| 多实例并发需要资源管理 | 不同语言/项目运行多个独立 LSP 进程 | `Promise.allSettled` 确保单个服务器关闭失败不影响其他 |
| 插件刷新时重新初始化 | `reinitializeLspServerManager()` 关闭旧实例创建新实例 | generation 计数器使旧 promise 失效 |
| bare 模式跳过 LSP | 脚本调用模式下不需要代码智能功能 | `isBareMode()` 检查在初始化前执行 |
| 延迟队列 | 连接未就绪时请求被缓冲 | 连接就绪后自动发送，但可能导致响应延迟 |


---

[← OAuth 与认证](../22-OAuth与认证/oauth-auth.md) | [目录](../README.md) | [沙箱系统 →](../24-沙箱系统/sandbox-system.md)
