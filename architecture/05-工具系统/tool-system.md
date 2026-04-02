# 工具系统 - Tool System

> 源文件: `src/Tool.ts` (792 行), `src/tools.ts`, `src/services/tools/toolExecution.ts` (1745 行),
> `src/services/tools/toolOrchestration.ts` (189 行), `src/services/tools/StreamingToolExecutor.ts`,
> `src/tools/` (40+ 工具目录)

---

## 1. 架构概览

```
tools.ts (注册总表)
  ├── getAllBaseTools() — 完整工具列表
  ├── getTools() — 权限过滤后工具列表
  └── assembleToolPool() — 含 MCP 工具的最终工具池

Tool.ts (类型定义)
  ├── Tool 接口 — 工具的类型契约
  └── ToolUseContext — 工具执行环境 (40+ 属性)

services/tools/ (编排层)
  ├── toolOrchestration.ts — 并发/串行调度
  ├── toolExecution.ts — 单工具执行管线
  ├── StreamingToolExecutor.ts — 流式工具并发器
  └── toolHooks.ts — 工具生命周期钩子
```

---

## 2. Tool.ts — Tool 类型接口 (792 行)

### 2.1 Tool 类型核心字段

```typescript
export type Tool = {
  // === 标识 ===
  name: string                      // 工具名（发送给 API，唯一标识）
  description: string               // 工具描述（发送给 API）
  inputSchema: z.ZodType            // Zod 输入 schema（运行时校验）

  // === 执行 ===
  execute: (
    input: unknown,
    context: ToolUseContext,
  ) => AsyncGenerator<ToolProgress | Message, Message | void>
  // execute 是 async generator，可以 yield 进度事件和中间消息

  // === 并发控制 ===
  isConcurrencySafe: (input: unknown) => boolean
  // 给定输入是否可以与其他工具并发执行
  // 例如: FileReadTool 总是 true, BashTool 看命令是否只读

  isReadOnly: boolean | ((input: unknown) => boolean)
  // 是否为只读工具（不修改文件系统）

  // === 结果控制 ===
  maxResultSizeChars?: number
  // 工具结果最大字符数，超过此值结果持久化到磁盘（>20KB）
  // Infinity 表示不限制（某些工具需要完整结果）

  // === Schema 与描述 ===
  backfillObservableInput?: (input: unknown) => unknown
  // 回填可观察输入（用于 UI 显示 streaming 输入参数）

  // === 启用状态 ===
  isEnabled: () => boolean
  // 当前环境/配置下工具是否可用

  // === MCP 信息 ===
  mcpInfo?: {
    serverName: string
    toolName: string
  }
  // MCP 工具的服务器来源信息

  // === 活动描述 ===
  getActivityDescription?: (input: unknown) => string
  // 返回人类可读的活动描述（显示在 UI 中）

  // === 权限 ===
  getPermissionDescription?: (input: unknown) => string
  // 权限提示中的操作描述

  // === 延迟加载 ===
  isDeferredTool?: boolean
  // 是否为延迟工具（通过 ToolSearch 按需加载）
}
```

### 2.2 ToolUseContext — 工具执行上下文 (40+ 属性)

```typescript
export type ToolUseContext = {
  // === 核心选项 ===
  options: {
    commands: Command[]                     // 可用命令列表
    debug: boolean                          // 调试模式
    mainLoopModel: string                   // 主循环模型
    tools: Tools                            // 可用工具列表
    verbose: boolean                        // 详细模式
    thinkingConfig: ThinkingConfig          // 思维配置
    mcpClients: MCPServerConnection[]       // MCP 客户端连接
    mcpResources: Record<string, ServerResource[]>  // MCP 资源
    isNonInteractiveSession: boolean        // 非交互式会话
    agentDefinitions: AgentDefinitionsResult // 代理定义
    maxBudgetUsd?: number                   // 最大预算（美元）
    customSystemPrompt?: string             // 自定义系统提示
    appendSystemPrompt?: string             // 附加系统提示
    querySource?: QuerySource               // 查询来源覆盖
    refreshTools?: () => Tools              // 工具刷新回调
  }

  // === 控制器 ===
  abortController: AbortController          // 中止控制器

  // === 文件状态 ===
  readFileState: FileStateCache             // 文件读取状态 LRU 缓存

  // === 应用状态 ===
  getAppState(): AppState                   // 获取应用状态
  setAppState(f: (prev: AppState) => AppState): void  // 设置应用状态
  setAppStateForTasks?: (f: (prev: AppState) => AppState) => void  // 任务作用域状态

  // === UI 回调 ===
  handleElicitation?: (...)  => Promise<ElicitResult>   // URL elicitation 处理
  setToolJSX?: SetToolJSXFn                             // 设置工具 JSX
  addNotification?: (notif: Notification) => void       // 添加通知
  appendSystemMessage?: (msg: SystemMessage) => void    // 追加系统消息
  sendOSNotification?: (opts: {...}) => void            // 操作系统通知

  // === 记忆系统 ===
  nestedMemoryAttachmentTriggers?: Set<string>          // 嵌套记忆触发器
  loadedNestedMemoryPaths?: Set<string>                 // 已加载的嵌套记忆路径
  dynamicSkillDirTriggers?: Set<string>                 // 动态技能目录触发器
  discoveredSkillNames?: Set<string>                    // 已发现的技能名

  // === 进度回调 ===
  setInProgressToolUseIDs: (f: (prev: Set<string>) => Set<string>) => void
  setHasInterruptibleToolInProgress?: (v: boolean) => void
  setResponseLength: (f: (prev: number) => number) => void
  pushApiMetricsEntry?: (ttftMs: number) => void
  setStreamMode?: (mode: SpinnerMode) => void
  onCompactProgress?: (event: CompactProgressEvent) => void
  setSDKStatus?: (status: SDKStatus) => void
  openMessageSelector?: () => void

  // === 历史与归属 ===
  updateFileHistoryState: (updater: ...) => void
  updateAttributionState: (updater: ...) => void
  setConversationId?: (id: UUID) => void

  // === 代理标识 ===
  agentId?: AgentId                                     // 子代理 ID
  agentType?: string                                    // 子代理类型名

  // === 消息与限制 ===
  requireCanUseTool?: boolean                           // 强制权限检查
  messages: Message[]                                   // 当前消息列表
  fileReadingLimits?: { maxTokens?: number; maxSizeBytes?: number }
  globLimits?: { maxResults?: number }

  // === 权限追踪 ===
  toolDecisions?: Map<string, {
    source: string; decision: 'accept' | 'reject'; timestamp: number
  }>
  queryTracking?: QueryChainTracking                    // 查询链追踪

  // === 交互式提示 ===
  requestPrompt?: (sourceName: string, toolInputSummary?: string | null) =>
    (request: PromptRequest) => Promise<PromptResponse>

  // === 工具标识 ===
  toolUseId?: string
  criticalSystemReminder_EXPERIMENTAL?: string

  // === 子代理 ===
  preserveToolUseResults?: boolean                      // 保留工具结果（团队成员）
  localDenialTracking?: DenialTrackingState             // 本地拒绝追踪
  contentReplacementState?: ContentReplacementState     // 内容替换状态
  renderedSystemPrompt?: SystemPrompt                   // 父级渲染的系统提示
  userModified?: boolean                                // 用户是否修改过
}
```

#### 设计理念：为什么ToolUseContext有40+属性（"胖上下文"设计）？

- **叶子节点的信息饥渴**：工具是 query 循环的叶子节点，需要访问多种运行时状态——权限判定（`toolDecisions`）、文件系统缓存（`readFileState`）、UI 回调（`setToolJSX`、`addNotification`）、记忆系统（`nestedMemoryAttachmentTriggers`）、代理标识（`agentId`、`agentType`）等。
- **"宽接口" vs "窄接口"权衡**：替代方案是依赖注入——每个工具声明自己需要的接口子集。但 40+ 工具各需要不同子集，接口会爆炸式增长。更关键的是，添加新工具不应需要修改依赖注入管线。胖上下文的代价（内存浪费）微乎其微——`ToolUseContext` 中绝大多数字段是引用传递（回调函数、共享对象引用），实际只有一份数据。
- **演化友好性**：源码中 `ToolUseContext` 从最初的十几个字段逐步增长到 40+，每次新增字段（如 `contentReplacementState`、`criticalSystemReminder_EXPERIMENTAL`）都不需要修改现有工具代码——旧工具简单忽略不使用的字段。这是典型的"宽接口降低添加新工具的摩擦"模式。

#### 设计理念：为什么execute()返回AsyncGenerator而不是Promise？

- **实时进度反馈**：工具执行可能耗时很长（Bash 命令运行数分钟），需要 `yield` 进度事件（`ToolProgress`）让 UI 实时更新。如果是 Promise，用户只能看到"执行中..."然后突然看到完整结果，体验很差。
- **中途取消支持**：generator 允许中途取消——用户按 Ctrl+C 时 `generator.return()` 立即生效。Promise 模式下只能通过 `AbortController` 信号，但清理逻辑更复杂。
- **流式输出**：Bash 工具需要逐行 `yield` 输出，FileRead 需要 `yield` 读取进度。源码中 `execute` 的签名是 `AsyncGenerator<ToolProgress | Message, Message | void>`——yield 类型是进度/中间消息，return 类型是最终结果消息。

---

## 3. tools.ts — 工具注册表

### 3.1 getAllBaseTools()

返回当前环境下所有可能的工具列表（编译时 + 运行时门控）：

```typescript
export function getAllBaseTools(): Tools {
  return [
    // === 核心工具（始终可用）===
    AgentTool,          // 子代理
    TaskOutputTool,     // 任务输出
    BashTool,           // Shell 执行

    // === 搜索工具（嵌入式搜索不可用时）===
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),

    // === 模式控制 ===
    ExitPlanModeV2Tool, // 退出计划模式
    EnterPlanModeTool,  // 进入计划模式

    // === 文件操作 ===
    FileReadTool,       // 读取文件
    FileEditTool,       // 编辑文件（精确替换）
    FileWriteTool,      // 写入文件
    NotebookEditTool,   // Notebook 编辑

    // === Web ===
    WebFetchTool,       // 网页抓取
    WebSearchTool,      // 网页搜索

    // === 任务管理 ===
    TodoWriteTool,      // Todo 写入
    TaskStopTool,       // 任务停止
    AskUserQuestionTool,// 用户交互

    // === 技能 ===
    SkillTool,          // 技能执行

    // === 条件工具（ant-only）===
    ...(USER_TYPE === 'ant' ? [ConfigTool, TungstenTool] : []),
    ...(SuggestBackgroundPRTool ? [SuggestBackgroundPRTool] : []),

    // === 条件工具（feature flags）===
    ...(WebBrowserTool ? [WebBrowserTool] : []),
    ...(isTodoV2Enabled() ? [TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool] : []),
    ...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
    ...(isAgentSwarmsEnabled() ? [TeamCreateTool, TeamDeleteTool] : []),

    // === 消息与协作 ===
    SendMessageTool,    // 消息发送

    // === 实验性工具 ===
    ...(SleepTool ? [SleepTool] : []),
    ...cronTools,       // CronCreate/CronDelete/CronList
    ...(RemoteTriggerTool ? [RemoteTriggerTool] : []),
    ...(MonitorTool ? [MonitorTool] : []),

    // === 其他 ===
    BriefTool,          // 简报
    ...(SendUserFileTool ? [SendUserFileTool] : []),
    ...(PushNotificationTool ? [PushNotificationTool] : []),
    ...(SubscribePRTool ? [SubscribePRTool] : []),
    ...(PowerShellTool ? [PowerShellTool] : []),  // Windows PowerShell
    ...(SnipTool ? [SnipTool] : []),

    // === MCP 资源 ===
    ListMcpResourcesTool,   // MCP 资源列表
    ReadMcpResourceTool,    // MCP 资源读取

    // === 工具搜索 ===
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),

    // === 测试 ===
    ...(NODE_ENV === 'test' ? [TestingPermissionTool] : []),
  ]
}
```

### 3.2 getTools() — 权限过滤

```typescript
export const getTools = (permissionContext: ToolPermissionContext): Tools => {
  // 简化模式: CLAUDE_CODE_SIMPLE → 仅 Bash/Read/Edit
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
    if (isReplModeEnabled() && REPLTool) {
      return filterToolsByDenyRules([REPLTool, ...coordinatorTools], permissionContext)
    }
    return filterToolsByDenyRules([BashTool, FileReadTool, FileEditTool, ...coordinatorTools], permissionContext)
  }

  // 完整模式: 获取所有基础工具 + REPL 过滤
  let tools = getAllBaseTools()
  // REPL 模式: 隐藏被 REPL 包装的原始工具
  if (isReplModeEnabled()) {
    tools = tools.filter(t => !REPL_ONLY_TOOLS.includes(t.name))
  }
  return filterToolsByDenyRules(tools, permissionContext)
}
```

### 3.3 filterToolsByDenyRules()

```typescript
export function filterToolsByDenyRules<T extends { name: string; mcpInfo?: {...} }>(
  tools: readonly T[],
  permissionContext: ToolPermissionContext,
): T[] {
  return tools.filter(tool => !getDenyRuleForTool(permissionContext, tool))
}
```

从工具列表中移除被 blanket deny（无 ruleContent 的 deny 规则）的工具。MCP 服务器前缀规则（如 `mcp__server`）在这里就能过滤掉整个服务器的工具。

### 3.4 assembleToolPool()

在 `getTools()` 基础上加入 MCP 工具，形成最终工具池。MCP 工具通过 `MCPTool` 桥接器包装为标准 Tool 接口。

---

## 4. toolExecution.ts — 单工具执行管线 (1745 行)

### 4.1 runToolUse() — 5 步管线

```typescript
export async function* runToolUse(
  toolUse: ToolUseBlock,
  assistantMessage: AssistantMessage,
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdateLazy, void>
```

**5 步执行流程**:

#### Step 1: 工具查找与输入校验
- `findToolByName()` 查找工具定义
- 如果找不到: 返回 `<tool_use_error>Error: No such tool available</tool_use_error>`
- `inputSchema.safeParse(toolUse.input)` — Zod 校验输入
- 如果校验失败: 返回格式化的 Zod 错误信息

#### Step 2: 权限检查
- `canUseTool(tool, input, assistantMessage)` — 调用权限判定链
- 可能触发:
  - 规则匹配（allow/deny/ask）
  - 分类器（auto 模式）
  - 用户交互提示
  - 钩子执行
- 如果拒绝: 返回拒绝消息 + 执行 `permissionDeniedHooks`

#### Step 3: Pre-tool hooks
- `runPreToolUseHooks()` — 执行 pre_tool_use 钩子
- 如果钩子返回阻塞错误: 中止执行

#### Step 4: 工具执行
- `tool.execute(input, toolUseContext)` — 调用工具的 async generator
- 处理进度事件 (ToolProgress)
- 处理中间消息
- 错误捕获与分类 (`classifyToolError`)
- 计时追踪 (`addToToolDuration`)

#### Step 5: Post-tool hooks
- `runPostToolUseHooks()` — 执行 post_tool_use 钩子
- 如果工具失败: `runPostToolUseFailureHooks()`
- 结果处理:
  - `processToolResultBlock()` — 大结果持久化到磁盘（>20KB）
  - `processPreMappedToolResultBlock()` — 预映射结果处理

### 4.2 工具执行遥测

每次工具执行记录:
- 工具名（`sanitizeToolNameForAnalytics`）
- 执行时间
- 成功/失败
- 权限决策
- 输入参数摘要（如果启用详细日志）
- OTel span（如果启用 tracing）

### 4.3 慢阶段警告

```typescript
const HOOK_TIMING_DISPLAY_THRESHOLD_MS = 500   // 显示 hook 计时摘要的最小总时长
const SLOW_PHASE_LOG_THRESHOLD_MS = 2000        // 调试日志警告阈值
```

---

## 5. toolOrchestration.ts — 编排器 (189 行)

### 5.1 runTools() — 主编排函数

```typescript
export async function* runTools(
  toolUseMessages: ToolUseBlock[],
  assistantMessages: AssistantMessage[],
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdate, void>
```

### 5.2 partitionToolCalls() — 批次分区

```typescript
function partitionToolCalls(
  toolUseMessages: ToolUseBlock[],
  toolUseContext: ToolUseContext,
): Batch[]

type Batch = {
  isConcurrencySafe: boolean
  blocks: ToolUseBlock[]
}
```

分区规则:
1. 连续的 `isConcurrencySafe=true` 工具合并为一个并发批次
2. `isConcurrencySafe=false` 工具单独作为一个串行批次
3. 如果 `isConcurrencySafe()` 抛出异常，保守地视为非并发安全

#### 设计理念：为什么并发工具最多10个？

- **太少的代价**：文件读取密集场景（一次读 5-10 个文件）会被串行化，延迟翻倍甚至更多。
- **太多的风险**：文件描述符耗尽风险（特别是大量文件操作），且 MCP 外部服务可能有自己的并发限制。
- **10是经验值**：源码 `toolOrchestration.ts:7-10` 中 `getMaxToolUseConcurrency()` 默认返回 10，但通过 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` 环境变量可调。这允许在特定环境下根据实际资源情况调优。

#### 设计理念：为什么并发批次的上下文修改要排队而不是立即应用？

- **确定性保证**：并发执行的工具看到的 context 应该是一致的。如果工具 A 的 `contextModifier` 在工具 B 执行时就生效，B 看到的是被 A 修改后的状态，而这取决于执行顺序——这是不确定性 bug。
- **源码实现**：`toolOrchestration.ts:30-61` 中，并发模式下所有 `contextModifier` 被推入 `queuedContextModifiers` 字典，批次完成后才按工具接收顺序统一应用。对比串行模式（第 65 行开始），每个工具执行完毕后 `currentContext = update.contextModifier.modifyContext(currentContext)` 立即生效，因为串行执行天然保证了顺序。
- **设计本质**：这是事务隔离级别的选择——并发批次使用 "snapshot isolation"（所有工具看到相同快照），串行批次使用 "read committed"（每个工具看到前一个工具提交的结果）。

### 5.3 并发 vs 串行执行

**串行 (runToolsSerially)**:
- 逐个执行非并发安全的工具
- 每个工具执行完毕后立即更新 `currentContext`
- 上下文修改立即可见于下一个工具

**并发 (runToolsConcurrently)**:
- 使用 `all()` helper 实现受限并发
- 最大并发数: `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` 环境变量，默认 10
- 上下文修改排队，批次完成后统一应用
- 按工具接收顺序 yield 结果

### 5.4 上下文修改传播

```typescript
// 并发模式: 修改排队
const queuedContextModifiers: Record<string, ((context: ToolUseContext) => ToolUseContext)[]> = {}
// 批次完成后统一应用
for (const block of blocks) {
  for (const modifier of queuedContextModifiers[block.id] ?? []) {
    currentContext = modifier(currentContext)
  }
}

// 串行模式: 立即应用
if (update.contextModifier) {
  currentContext = update.contextModifier.modifyContext(currentContext)
}
```

---

## 6. StreamingToolExecutor — 流式工具执行器

### 6.1 概述

`StreamingToolExecutor` 在模型流式输出的同时开始执行工具，减少等待时间。

### 6.2 状态机

```typescript
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'
```

状态转换:
```
queued → executing → completed → yielded
                  ↗
queued → completed (unknown tool → immediate error)
```

### 6.3 关键方法

```typescript
class StreamingToolExecutor {
  // 添加工具到执行队列（立即开始如果条件允许）
  addTool(block: ToolUseBlock, assistantMessage: AssistantMessage): void

  // 获取已完成但尚未 yield 的结果（按接收顺序）
  getCompletedResults(): AsyncGenerator<MessageUpdate, void>

  // 获取所有剩余结果（等待执行中的工具完成）
  getRemainingResults(): AsyncGenerator<MessageUpdate, void>

  // 丢弃所有待处理结果（streaming fallback 时使用）
  discard(): void
}
```

### 6.4 并发控制

- 并发安全工具: 可以与其他并发安全工具同时执行
- 非并发安全工具: 必须独占执行
- 子 AbortController: Bash 工具错误时立即中止同级子进程

### 6.5 进度推送

- 工具执行的进度消息 (`pendingProgress`) 立即 yield，不排队等待
- 通过 `progressAvailableResolve` 信号唤醒 `getRemainingResults` 消费者

---

## 7. 完整工具清单

### 7.1 按类别分组

#### 文件 I/O
| 工具名 | 文件 | 描述 |
|--------|------|------|
| `FileRead` | FileReadTool/ | 读取文件内容 |
| `FileEdit` | FileEditTool/ | 精确文本替换 |
| `FileWrite` | FileWriteTool/ | 写入文件 |
| `NotebookEdit` | NotebookEditTool/ | Jupyter Notebook 编辑 |

#### 搜索
| 工具名 | 文件 | 描述 |
|--------|------|------|
| `Glob` | GlobTool/ | 文件模式搜索 (glob patterns) |
| `Grep` | GrepTool/ | 内容搜索 (ripgrep) |
| `ToolSearch` | ToolSearchTool/ | 延迟工具发现 |

#### Shell 执行
| 工具名 | 文件 | 描述 |
|--------|------|------|
| `Bash` | BashTool/ | Bash 命令执行 |
| `PowerShell` | PowerShellTool/ | PowerShell 执行 (Windows) |
| `REPL` | REPLTool/ | REPL 虚拟机 (ant-only) |

#### Web
| 工具名 | 文件 | 描述 |
|--------|------|------|
| `WebFetch` | WebFetchTool/ | 网页抓取 |
| `WebSearch` | WebSearchTool/ | 网页搜索 |
| `WebBrowser` | WebBrowserTool/ | Web 浏览器 (feature flag) |

#### 代理与任务
| 工具名 | 文件 | 描述 |
|--------|------|------|
| `Agent` | AgentTool/ | 创建子代理 |
| `TaskCreate` | TaskCreateTool/ | 创建任务 |
| `TaskGet` | TaskGetTool/ | 获取任务 |
| `TaskUpdate` | TaskUpdateTool/ | 更新任务 |
| `TaskList` | TaskListTool/ | 列出任务 |
| `TaskOutput` | TaskOutputTool/ | 任务输出 |
| `TaskStop` | TaskStopTool/ | 停止任务 |

#### 模式控制
| 工具名 | 文件 | 描述 |
|--------|------|------|
| `EnterPlanMode` | EnterPlanModeTool/ | 进入计划模式 |
| `ExitPlanModeV2` | ExitPlanModeTool/ | 退出计划模式 |
| `Brief` | BriefTool/ | 简报模式 |

#### 配置
| 工具名 | 文件 | 描述 |
|--------|------|------|
| `Config` | ConfigTool/ | 配置管理 (ant-only) |
| `Tungsten` | TungstenTool/ | 调试工具 (ant-only) |

#### MCP
| 工具名 | 文件 | 描述 |
|--------|------|------|
| `ListMcpResources` | ListMcpResourcesTool/ | 列出 MCP 资源 |
| `ReadMcpResource` | ReadMcpResourceTool/ | 读取 MCP 资源 |
| (动态) | MCPTool/ | MCP 工具桥接 |
| `McpAuth` | McpAuthTool/ | MCP 认证 |

#### 团队协作
| 工具名 | 文件 | 描述 |
|--------|------|------|
| `TeamCreate` | TeamCreateTool/ | 创建团队 |
| `TeamDelete` | TeamDeleteTool/ | 删除团队 |
| `SendMessage` | SendMessageTool/ | 发送消息 |
| `ListPeers` | ListPeersTool/ | 列出对等节点 (UDS) |

#### 技能与工作流
| 工具名 | 文件 | 描述 |
|--------|------|------|
| `Skill` | SkillTool/ | 技能执行 |
| `Workflow` | WorkflowTool/ | 工作流脚本 |

#### 调度
| 工具名 | 文件 | 描述 |
|--------|------|------|
| `CronCreate` | ScheduleCronTool/ | 创建 Cron 任务 |
| `CronDelete` | ScheduleCronTool/ | 删除 Cron 任务 |
| `CronList` | ScheduleCronTool/ | 列出 Cron 任务 |
| `RemoteTrigger` | RemoteTriggerTool/ | 远程触发 |

#### 用户交互
| 工具名 | 文件 | 描述 |
|--------|------|------|
| `AskUserQuestion` | AskUserQuestionTool/ | 向用户提问 |
| `TodoWrite` | TodoWriteTool/ | 写入 Todo |

#### Worktree
| 工具名 | 文件 | 描述 |
|--------|------|------|
| `EnterWorktree` | EnterWorktreeTool/ | 进入 Git Worktree |
| `ExitWorktree` | ExitWorktreeTool/ | 退出 Git Worktree |

#### 其他
| 工具名 | 文件 | 描述 |
|--------|------|------|
| `LSP` | LSPTool/ | LSP 操作 |
| `Sleep` | SleepTool/ | 主动休眠 |
| `Monitor` | MonitorTool/ | 进程监控 |
| `SendUserFile` | SendUserFileTool/ | 发送文件给用户 |
| `PushNotification` | PushNotificationTool/ | 推送通知 |
| `SubscribePR` | SubscribePRTool/ | PR 订阅 |
| `SuggestBackgroundPR` | SuggestBackgroundPRTool/ | 建议后台 PR (ant-only) |
| `SyntheticOutput` | SyntheticOutputTool/ | 合成输出 |
| `Snip` | SnipTool/ | 历史裁剪 |
| `CtxInspect` | CtxInspectTool/ | 上下文检查 |
| `TerminalCapture` | TerminalCaptureTool/ | 终端捕获 |
| `OverflowTest` | OverflowTestTool/ | 溢出测试 |
| `VerifyPlanExecution` | VerifyPlanExecutionTool/ | 计划验证 |

---

## 8. 共享基础设施

### 8.1 buildTool 模式

大多数工具使用 `buildTool` 辅助函数构建，确保一致的接口：

```typescript
// 典型的工具结构
export const SomeTool: Tool = {
  name: TOOL_NAME,
  description: TOOL_DESCRIPTION,
  inputSchema: lazySchema(() => z.object({ ... })),  // lazy schema
  isEnabled: () => true,
  isReadOnly: true,
  isConcurrencySafe: (input) => true,
  async *execute(input, context) {
    // ... 执行逻辑
    return createUserMessage({ ... })  // 返回工具结果消息
  },
}
```

#### 设计理念：为什么工具结果超过阈值要持久化到磁盘？

- **上下文窗口是稀缺资源**：一个大文件读取结果可能消耗 10K+ tokens，而上下文窗口总量有限（128K-200K tokens）。源码 `constants/toolLimits.ts` 定义了 `DEFAULT_MAX_RESULT_SIZE_CHARS = 50_000` 和 `MAX_TOOL_RESULT_TOKENS = 100_000`。
- **虚拟内存思想**：持久化到磁盘后，消息中只保留引用指针（preview + 文件路径），实际内容在需要时从磁盘加载。这本质上是 context window = 物理内存，磁盘 = swap 的类比。源码中 `processToolResultBlock()` 检查大小后将超限内容序列化到临时文件，`recordContentReplacement()` 记录替换状态以支持会话恢复。
- **单消息聚合预算**：源码还定义了 `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS = 200_000`——防止 N 个并发工具各自不超限但总和爆炸的情况（如 10 个工具各返回 40K = 400K）。注释明确说明："This prevents N parallel tools from each hitting the per-tool max and collectively producing e.g. 10 x 40K = 400K in one turn's user message."

### 8.2 Lazy Schema

```typescript
import { lazySchema } from '../utils/lazySchema.js'

inputSchema: lazySchema(() => z.object({
  file_path: z.string().describe('The path to read'),
  // ...
}))
```

Zod schema 在首次访问时惰性求值，减少启动时间。

#### 设计理念：为什么使用lazySchema（惰性求值Zod schema）？

- **启动延迟优化**：40+ 工具的 schema 在启动时全部编译会增加数百 ms 启动延迟。CLI 工具的首次响应速度是关键用户体验指标——用户输入命令后期望立即看到反馈。
- **按需编译**：惰性求值确保只有实际使用的工具才会编译 schema。许多工具在常见工作流中从不被调用（如 `CronCreate`、`TeamCreate`），为它们预编译 schema 是浪费。
- **广泛应用**：源码中 `lazySchema` 不仅用于工具 schema，还用于 SDK 控制协议 schema（`controlSchemas.ts` 中大量使用）和沙箱配置 schema（`sandboxTypes.ts`），这说明惰性求值是整个项目的通用优化模式。

### 8.3 Activity Descriptions

工具可提供 `getActivityDescription(input)` 返回人类可读描述，用于 UI 显示：

```
"Reading /path/to/file.ts"
"Running: git status"
"Searching for *.tsx files"
```

### 8.4 工具结果持久化 (>20KB)

```typescript
// utils/toolResultStorage.ts
// 当工具结果超过 maxResultSizeChars (默认 ~20KB) 时:
// 1. 结果被序列化到磁盘临时文件
// 2. 消息中替换为指向文件的引用
// 3. 后续读取时从磁盘恢复
```

- `processToolResultBlock()` — 检查大小并持久化
- `processPreMappedToolResultBlock()` — 预映射结果
- `recordContentReplacement()` — 记录替换状态（支持会话恢复）

### 8.5 工具名匹配

```typescript
export function toolMatchesName(tools: Tools, name: string): Tool | undefined
export function findToolByName(tools: Tools, name: string): Tool | undefined
```

支持精确匹配和 MCP 工具名的 `mcp__server__tool` 格式匹配。

---

## 9. 工具执行完整数据流

![Tool Execution Data Flow](../diagrams/tool-execution-flow.svg)

---

## 工程实践指南

### 添加新工具的完整清单

按以下步骤创建一个新工具：

1. **在 `tools/` 目录下创建工具目录** — 例如 `tools/MyNewTool/`，包含主文件 `MyNewTool.ts`
2. **实现 Tool 接口** — 必须包含以下字段：
   ```typescript
   export const MyNewTool: Tool = {
     name: 'MyNewTool',
     description: '工具描述（发送给 API）',
     inputSchema: lazySchema(() => z.object({ /* 输入字段 */ })),
     isEnabled: () => true,  // 当前环境下是否可用
     isReadOnly: true,       // 或 (input) => boolean
     isConcurrencySafe: (input) => true,  // 见下方并发安全检查清单
     async *execute(input, context) {
       // yield 进度事件（可选）
       // return 最终结果消息
     },
   }
   ```
3. **用 `lazySchema` 包装 inputSchema** — 惰性求值，减少启动时间
4. **实现 `isConcurrencySafe`** — 如果工具不修改文件系统且不依赖执行顺序，返回 true
5. **在 `getAllBaseTools()` 中注册** — 在 `tools.ts` 的 `getAllBaseTools()` 返回数组中添加工具引用。如果需要 feature flag 门控：`...(feature('MY_FLAG') ? [MyNewTool] : [])`
6. **添加权限规则**（如需要）— 如果工具涉及破坏性操作，在权限系统中添加对应检查
7. **添加 `getActivityDescription`** — 返回人类可读的活动描述，用于 UI 显示（如 `"Running: git status"`）
8. **添加测试** — 工具执行测试、输入校验测试、权限测试

### 使工具并发安全的检查清单

在决定 `isConcurrencySafe` 的返回值时，逐项检查：

| 检查项 | 如果是 | 如果否 |
|--------|--------|--------|
| 工具是否修改文件系统（写入/删除文件）？ | 返回 false | 继续检查 |
| 工具是否修改共享状态（如 `ToolUseContext` 上的可变字段）？ | 返回 false | 继续检查 |
| 工具是否依赖执行顺序（如需要上一个工具的输出）？ | 返回 false | 继续检查 |
| 工具是否有副作用可能影响同批次其他工具？ | 返回 false | 可以返回 true |

**特殊情况**：
- `BashTool` 的 `isConcurrencySafe` 根据命令内容判断（只读命令如 `git status` 返回 true，写命令如 `rm` 返回 false）
- `FileReadTool` 总是返回 true（只读操作）
- `FileEditTool`/`FileWriteTool` 总是返回 false（修改文件系统）

### 调试工具执行失败

`runToolUse()` 的 5 步管线中，定位失败发生在哪一步：

| 步骤 | 失败表现 | 排查方向 |
|------|----------|----------|
| **Step 1: 工具查找与输入校验** | `"Error: No such tool available"` 或 Zod 校验错误 | 检查工具名是否正确注册；检查 `inputSchema` 是否匹配 API 发送的输入 |
| **Step 2: 权限检查** | 拒绝消息 + `permissionDeniedHooks` | 检查权限规则（参见权限与安全文档）；开启 `--debug` 查看权限决策链 |
| **Step 3: Pre-tool hooks** | 钩子返回阻塞错误 | 检查 `pre_tool_use` 钩子配置；钩子可能是外部脚本，检查其退出码 |
| **Step 4: 工具执行** | 执行异常或超时 | 检查 `tool.execute()` 内部逻辑；检查 `AbortController` 是否被提前中止 |
| **Step 5: Post-tool hooks** | 钩子失败不影响结果，但会触发 `runPostToolUseFailureHooks` | 检查 `post_tool_use` 钩子配置 |

### 大工具结果的处理

当工具返回的结果超过阈值时：

- **默认限制**: `DEFAULT_MAX_RESULT_SIZE_CHARS = 50_000`（约 20KB）
- **超限行为**: 结果被 `processToolResultBlock()` 持久化到磁盘临时文件，消息中只保留引用（preview + 文件路径）
- **单消息聚合预算**: `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS = 200_000`，防止 N 个并发工具各自不超限但总和爆炸

**如果工具结果被截断**：
1. 检查工具是否设置了 `maxResultSizeChars`（设为 `Infinity` 可禁用截断，但会消耗大量上下文窗口）
2. 检查 `contentReplacementState` 中是否有对应的替换记录
3. 考虑在工具内部预先裁剪结果，只返回关键信息

### ToolUseContext 扩展

添加新属性到 `ToolUseContext` 时：

1. **更新 `ToolUseContext` 类型** — 在 `src/Tool.ts` 中添加新字段（建议标记为可选 `?`）
2. **更新 `createSubagentContext()`** — 子代理的 `ToolUseContext` 由 `AgentTool` 通过此函数创建，确保新属性正确传递或初始化
3. **更新测试 mock** — 测试中的 `ToolUseContext` mock 需要包含新字段的默认值
4. **考虑是否需要跨轮次持久化** — 如果新属性需要在 `while(true)` 循环的多次迭代间保持，需要添加到 `State` 类型中

**注意**：`ToolUseContext` 采用"胖上下文"设计，新增字段不会影响现有工具——旧工具简单忽略不使用的字段。

### 常见陷阱

1. **`tool.execute` 必须是 AsyncGenerator（不是 async function）** — 即使不需要 yield 进度事件，也必须使用 `async *execute()` 语法。async function 的返回值不兼容 generator 消费逻辑，会导致运行时类型错误

2. **`isConcurrencySafe(input)` 可能抛异常** — 例如 `BashTool` 内部使用 `shell-quote` 解析命令，格式异常会抛出。系统会捕获异常并保守地按 serial 处理（`partitionToolCalls` 中的 try-catch），但不要依赖这个行为

3. **并发批次中的 `contextModifier` 排队而非立即应用** — 并发执行的工具看到的 context 是同一个快照（snapshot isolation）。不要依赖同批次其他工具的上下文修改——如果工具 A 的 contextModifier 修改了文件缓存，工具 B 在同一批次中看不到这个修改

4. **不要在工具中直接 import UI 组件** — 通过 `ToolUseContext` 的回调（`setToolJSX`、`addNotification`）与 UI 通信。直接 import 会破坏 Headless/SDK 模式

5. **`lazySchema` 的首次求值可能在热路径上** — 如果工具在用户第一次请求时被调用，lazy schema 的编译延迟会计入首次响应时间。对于核心工具（如 `Bash`、`FileRead`），这个延迟通常可忽略；对于复杂 schema，注意首次调用可能较慢

6. **MCP 工具通过 `MCPTool` 桥接** — MCP 工具不直接实现 `Tool` 接口，而是通过 `MCPTool` 包装。如果需要修改 MCP 工具的行为，修改 `MCPTool` 桥接层，不要尝试修改 MCP 服务器返回的工具定义


---

[← API 客户端](../04-API客户端/api-client.md) | [目录](../README.md) | [权限与安全 →](../06-权限与安全/permission-security.md)
