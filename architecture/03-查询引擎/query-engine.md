# 查询引擎 - Query Engine

> 源文件: `src/query.ts` (1729 行), `src/QueryEngine.ts`, `src/query/*.ts`

---

## 1. 架构概览

查询引擎是 Claude Code 的核心运行时，负责管理「用户输入 → 模型调用 → 工具执行 → 结果返回」的完整循环。它是一个 **async generator** 架构，通过 `yield` 向调用方推送流式事件，通过 `return` 返回终止原因。

```
QueryEngine.ts (SDK/print 入口)
  └─→ query.ts::query() (async generator 包装器)
        └─→ queryLoop() (while(true) 主循环)
              ├── Phase 1: 上下文准备 (压缩管线)
              ├── Phase 2: API 调用 (流式接收)
              ├── Phase 3: 工具执行 (并发/串行编排)
              ├── Phase 4: 停止钩子 (post-turn 处理)
              └── Phase 5: 继续/终止判定
```

---

## 2. query() — 核心 async generator

### 2.1 文件位置与签名

```typescript
// src/query.ts, line 219
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal  // return type — 终止原因
>
```

### 2.2 QueryParams 类型 — 完整字段

```typescript
// src/query.ts, line 181
export type QueryParams = {
  messages: Message[]                // 历史消息数组
  systemPrompt: SystemPrompt         // 系统提示词
  userContext: { [k: string]: string } // 用户上下文 (键值对注入 prompt)
  systemContext: { [k: string]: string } // 系统上下文
  canUseTool: CanUseToolFn           // 权限判定函数
  toolUseContext: ToolUseContext      // 工具执行上下文 (40+ 属性)
  fallbackModel?: string             // 降级模型
  querySource: QuerySource           // 查询来源标识
  maxOutputTokensOverride?: number   // 最大输出 token 覆盖
  maxTurns?: number                  // 最大轮数限制
  skipCacheWrite?: boolean           // 跳过缓存写入
  taskBudget?: { total: number }     // API task_budget (output_config.task_budget)
  deps?: QueryDeps                   // 依赖注入（测试用）
}
```

### 2.3 State 类型 — 可变循环状态

每次循环迭代时，state 被解构以提供裸名访问；continue 站点通过 `state = { ... }` 批量赋值。

```typescript
// src/query.ts, line 204
type State = {
  messages: Message[]                           // 当前消息列表
  toolUseContext: ToolUseContext                 // 当前工具上下文
  autoCompactTracking: AutoCompactTrackingState | undefined  // 压缩追踪
  maxOutputTokensRecoveryCount: number          // max_output_tokens 恢复计数
  hasAttemptedReactiveCompact: boolean          // 是否已尝试响应式压缩
  maxOutputTokensOverride: number | undefined   // 输出 token 覆盖
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined  // 待处理摘要
  stopHookActive: boolean | undefined           // 停止钩子是否激活
  turnCount: number                             // 当前轮次
  transition: Continue | undefined              // 上一次迭代的继续原因
}
```

State 的初始化：

```typescript
let state: State = {
  messages: params.messages,
  toolUseContext: params.toolUseContext,
  maxOutputTokensOverride: params.maxOutputTokensOverride,
  autoCompactTracking: undefined,
  stopHookActive: undefined,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  turnCount: 1,
  pendingToolUseSummary: undefined,
  transition: undefined,
}
```

### 设计理念：为什么用 async generator 而不是 async 函数

`query()` 的返回类型是 `AsyncGenerator<StreamEvent | ... , Terminal>`，而非简单的 `Promise<Terminal>`。这个选择影响了整个系统的架构风格，原因如下：

**流式输出** — LLM 逐 token 生成内容，async 函数只能在全部完成后返回，用户将面对数秒到数十秒的空白等待。generator 的 `yield` 将每个 `StreamEvent` 实时推送给 UI，实现逐字符渲染。

**背压控制** — 调用方通过 `for await...of` 消费 generator，按自己的速度拉取事件。如果 UI 渲染瓶颈导致消费变慢，generator 自然在 `yield` 点暂停，API 响应缓冲区不会无限增长。这比 EventEmitter 模式（推送方不知道消费方是否跟得上）安全得多。

**中途取消** — `generator.return()` 可以在任意 `yield` 点优雅终止循环。`query.ts` 使用 `using` 声明（如 `using pendingMemoryPrefetch = startRelevantMemoryPrefetch(...)`，line ~301）确保取消时资源被自动 dispose。这比 `AbortController` 粒度更细：`AbortController` 只能取消网络请求，generator 可以在工具执行、停止钩子、上下文压缩等任意阶段停止。

**多类型事件 + 类型安全终止** — generator 的 `yield` 类型是 `StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage`（中间事件），`return` 类型是 `Terminal`（终止原因）。TypeScript 编译器可以完整检查两种类型路径，比 EventEmitter 的字符串事件名 + `any` payload 安全得多。

**隐式状态机** — generator 的代码执行位置本身就编码了"当前状态"。循环中的 5 个 Phase + 多个 `continue` 站点对应不同的状态转换，但不需要显式的状态枚举和 switch-case 矩阵。详见下文"为什么 while(true) 而不是显式状态机"。

### 设计理念：为什么 State 是可变的

`State` 类型（`src/query.ts:204`）在 `while(true)` 循环体内被解构、修改、重新赋值——这是可变状态模式，不是 Redux 式的 immutable。原因：

1. **无并发** — `state` 仅在 `queryLoop` 函数内可见，Node.js 单线程保证了它不会被其他代码并发修改。不可变性解决的核心问题（防止竞态条件）在这里不存在。

2. **样板代码** — 1729 行的循环体中，有 7 个 `state = { ... }` 的 continue 站点（line ~289 注释："Loop-local (not on State) to avoid touching the 7 continue sites"）。如果使用不可变模式，每个站点都需要深拷贝整个 state 对象，增加大量样板代码而无实际收益。

3. **清晰的状态转换点** — 批量重新赋值 `state = { ...next }` 在 continue 站点提供了清晰的、可 grep 的状态转换标记。每个 `continue` 都附带了 `transition: { reason: '...' }`（如 `'reactive_compact_retry'`、`'max_output_tokens_escalate'`、`'token_budget_continuation'`），使得状态转换原因可以被测试断言（line ~215 注释："Lets tests assert recovery paths fired without inspecting message contents"）。

### 设计理念：为什么 QueryDeps 用依赖注入

`QueryDeps`（`src/query/deps.ts`）只有 4 个依赖项（`callModel`、`microcompact`、`autocompact`、`uuid`），看起来规模不大。但源码注释（`src/query/deps.ts:9-12`）直接说明了动机：

> "I/O dependencies for query(). Passing a `deps` override into QueryParams lets tests inject fakes directly instead of spyOn-per-module — the most common mocks (callModel, autocompact) are each spied in 6-8 test files today with module-import-and-spy boilerplate."

这不是为了"架构纯洁性"，而是解决具体的测试维护问题：此前 `callModel` 和 `autocompact` 各被 6-8 个测试文件通过 `spyOn` 模块级 mock，模块级 mock 导致测试间互相干扰（一个测试的 spy 如果没正确恢复，会影响后续测试）。`QueryDeps` 通过函数参数传递依赖，每个测试创建自己的 fake 实例，彻底消除了共享 mock 状态的问题。

注释还提到："`Scope is intentionally narrow (4 deps) to prove the pattern.`"——当前是最小可行方案，后续 PR 可以逐步添加 `runTools`、`handleStopHooks` 等依赖。

### 设计理念：为什么有 9 种终止原因

9 种 `Terminal` reason 不是过度设计——每种对应不同的 UI 展示和后续处理路径：

| 终止原因 | UI 行为差异 |
|----------|------------|
| `completed` | 显示结果，执行停止钩子（记忆提取、建议提示） |
| `aborted_streaming` | 清理 partial 消息，丢弃 `StreamingToolExecutor` 中的待处理结果 |
| `aborted_tools` | 为每个未完成的 `tool_use` block 生成中断消息（`yieldMissingToolResultBlocks`） |
| `prompt_too_long` | 触发压缩建议，可能执行 `reactiveCompact` |
| `model_error` | 显示错误面板，调用 `executeStopFailureHooks` |
| `image_error` | 特定的图像大小/格式错误提示 |
| `blocking_limit` | auto-compact OFF 时的硬性限制提醒 |
| `hook_stopped` / `stop_hook_prevented` | 停止钩子的不同阻止模式——前者是 `preventContinuation`，后者是 `blockingErrors` |
| `max_turns` | 附带 `turnCount` 用于 SDK 调用方的预算管理 |

如果简化为"成功/失败/取消"三种，SDK 调用方将无法区分"模型认为任务完成"和"Token 用完被迫停止"——这两种情况在自动化流程中需要完全不同的处理策略。

### 设计理念：为什么 while(true) 而不是显式状态机

`queryLoop` 使用 `while(true)` + `continue` + `return`，而不是 `enum State { PREPARING, CALLING_API, EXECUTING_TOOLS, ... }` + `switch(state)`。对于 1729 行的循环体，这是更好的选择：

1. **generator 的暂停点就是隐式状态** — 代码执行到 Phase 2（API 调用）时，"当前处于 API 调用阶段"这个状态信息已经编码在程序计数器中。显式状态枚举是在用数据冗余地表达代码位置已经表达的信息。

2. **状态转换矩阵爆炸** — 5 个 Phase + 7 个 continue 站点意味着至少 35 种可能的状态转换组合。显式状态机需要定义每种转换的合法性，产生巨大的 switch-case 矩阵，可读性远不如线性的 `if-continue` 流程。

3. **恢复路径是线性的** — `reactive_compact_retry`、`max_output_tokens_escalate`、`fallback` 等恢复路径都只是"修改 state，跳回循环头部"。线性代码中，这是一个 `state = next; continue`——清晰且局部化。显式状态机中，这需要在"从 Phase 2 转换到 Phase 1"和"从 Phase 4 转换到 Phase 1"中分别编写转换逻辑。

---

## 3. queryLoop() — while(true) 主循环结构

### 3.1 循环入口

```typescript
// src/query.ts, line 241
async function* queryLoop(
  params: QueryParams,
  consumedCommandUuids: string[],
): AsyncGenerator<...>
```

### 3.2 不可变参数解构

循环开始时提取不可变参数，这些在整个循环生命周期内不会被重新赋值：

```typescript
const {
  systemPrompt,
  userContext,
  systemContext,
  canUseTool,
  fallbackModel,
  querySource,
  maxTurns,
  skipCacheWrite,
} = params
const deps = params.deps ?? productionDeps()
```

### 3.3 循环体每次迭代的状态解构

```typescript
while (true) {
  let { toolUseContext } = state  // toolUseContext 在迭代内可被重新赋值
  const {
    messages,
    autoCompactTracking,
    maxOutputTokensRecoveryCount,
    hasAttemptedReactiveCompact,
    maxOutputTokensOverride,
    pendingToolUseSummary,
    stopHookActive,
    turnCount,
  } = state
  // ... 迭代体
}
```

### 3.4 五阶段循环流程

![Query Engine Loop](../diagrams/query-engine-loop.svg)

#### Phase 1: 上下文准备

1. **applyToolResultBudget** — 对工具结果超过 20KB 的内容持久化到磁盘
2. **snipCompact** (feature `HISTORY_SNIP`) — 历史裁剪
3. **microcompact** — 微压缩（无 API 调用，纯本地操作）
4. **contextCollapse** (feature `CONTEXT_COLLAPSE`) — 上下文折叠
5. **autocompact** — 自动压缩（可能触发 API 调用生成摘要）
6. 组装 `fullSystemPrompt`
7. 创建 `StreamingToolExecutor`（如果 streamingToolExecution 门控开启）

#### Phase 2: API 调用

1. 调用 `deps.callModel()` (即 `queryModelWithStreaming`)
2. 流式接收事件，构建 `assistantMessages[]`
3. 检测 `tool_use` block → 设置 `needsFollowUp = true`
4. 处理 streaming fallback (FallbackTriggeredError)
5. 处理 max_output_tokens 恢复 (MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3)
6. 处理 prompt_too_long / reactiveCompact

#### Phase 3: 工具执行

1. **非流式路径**: `runTools()` → `partitionToolCalls()` → concurrent/serial batches
2. **流式路径**: `StreamingToolExecutor` → `getCompletedResults()` + `getRemainingResults()`
3. 每个工具: `runToolUse()` → 权限检查 → execute → 结果处理
4. 执行 `postSamplingHooks`

#### Phase 4: 停止钩子

1. 如果没有 `needsFollowUp`（模型未请求工具调用），进入停止判定
2. 调用 `handleStopHooks()` → 执行各停止钩子
3. Token Budget 检查（如果启用）
4. 如果停止钩子返回 `blockingErrors` 或 `preventContinuation`，决定是否继续

#### Phase 5: 继续/终止

1. 工具结果附加到消息列表
2. 获取 attachment messages（记忆、命令队列、技能发现）
3. 检查 `maxTurns` 限制
4. 组装下一个 `State` 对象
5. `state = next` → 回到 `while(true)` 头部

### 3.5 State 重新赋值站点

循环内有多个 `state = { ... }` 的 continue 站点，每个代表一个不同的继续原因：

- **next_turn** — 正常的工具结果跟进循环 (line ~1715)
- **reactive_compact** — 413 触发的响应式压缩后重试
- **max_output_tokens_recovery** — 输出 token 耗尽后的恢复重试
- **fallback** — 流式降级后使用备用模型重试
- **prompt_too_long_retry** — prompt too long 错误后的重试

---

## 4. 9 种终止原因 (Terminal Reasons)

| 原因 | 描述 | 触发条件 |
|------|------|----------|
| `completed` | 正常完成 | 模型未请求工具调用，停止钩子无阻塞错误 |
| `aborted_streaming` | 流式传输中中止 | 用户中断 (Ctrl+C) 在 streaming 阶段 |
| `aborted_tools` | 工具执行中中止 | 用户中断 (Ctrl+C) 在工具执行阶段 |
| `model_error` | 模型错误 | API 返回不可恢复错误 |
| `image_error` | 图像错误 | 图像大小/格式错误 |
| `prompt_too_long` | 提示过长 | 413 错误且无法通过压缩恢复 |
| `blocking_limit` | 阻塞限制 | 达到硬性 token 限制（auto-compact OFF 时） |
| `hook_stopped` | 钩子阻止 | 停止钩子明确阻止继续 |
| `stop_hook_prevented` | 停止钩子阻止 | 停止钩子的 blockingErrors |
| `max_turns` | 最大轮数 | 达到 maxTurns 限制 |

---

## 5. QueryEngine.ts — SDK/Print 入口

### 5.1 位置与角色

`QueryEngine.ts` 是 `query()` 的上层包装，为 SDK 和 print 模式提供更高级的 API。

### 5.2 QueryEngineConfig

```typescript
// 从 QueryEngine.ts 的构造参数推断
{
  sessionId: SessionId
  model: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  mcpResources: Record<string, ServerResource[]>
  agentDefinitions: AgentDefinitionsResult
  thinkingConfig: ThinkingConfig
  permissionMode: PermissionMode
  // ... 更多配置
}
```

### 5.3 ask() generator

`ask()` 方法封装了完整的用户输入到模型响应周期：

1. **processUserInput** — 预处理用户输入（命令检测、附件处理）
2. **fetchSystemPromptParts** — 组装系统提示（CLAUDE.md、MCP 指令、代理定义等）
3. **query()** 调用 — 启动核心循环
4. **事件分发** — 将 generator 事件转换为 SDK 兼容的消息格式
5. **使用量追踪** — 累积 API 使用量（accumulateUsage/updateUsage）
6. **会话记录** — recordTranscript, flushSessionStorage

### 5.4 关键行为

- 自动创建 `AbortController` 用于取消
- 会话持久化检查 (`isSessionPersistenceDisabled`)
- 文件历史快照 (`fileHistoryMakeSnapshot`)
- 错误恢复（`categorizeRetryableAPIError` 分类后决定重试或终止）

---

## 6. QueryConfig — 不可变查询配置

每次 `query()` 调用入口处冻结一次，整个循环内不再变化。

```typescript
// src/query/config.ts
export type QueryConfig = {
  sessionId: SessionId

  gates: {
    streamingToolExecution: boolean  // tengu_streaming_tool_execution2 门控
    emitToolUseSummaries: boolean    // CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES 环境变量
    isAnt: boolean                   // USER_TYPE === 'ant'
    fastModeEnabled: boolean         // !CLAUDE_CODE_DISABLE_FAST_MODE
  }
}
```

**设计决策**: 刻意排除 `feature()` 门控（那些是编译时 tree-shaking 边界），只包含运行时可变的 statsig/env 状态。这使得 QueryConfig 是纯数据（plain data），方便未来提取为纯函数 reducer: `(state, event, config) => state`。

---

## 7. QueryDeps — 依赖注入

```typescript
// src/query/deps.ts
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming    // 模型调用
  microcompact: typeof microcompactMessages   // 微压缩
  autocompact: typeof autoCompactIfNeeded     // 自动压缩
  uuid: () => string                          // UUID 生成
}

// 生产环境工厂
export function productionDeps(): QueryDeps {
  return {
    callModel: queryModelWithStreaming,
    microcompact: microcompactMessages,
    autocompact: autoCompactIfNeeded,
    uuid: randomUUID,
  }
}
```

**设计意图**: 通过 `params.deps` 传入覆盖项，测试可直接注入 fake 实现，无需 `spyOn` 模块级别 mock（此前 callModel 和 autocompact 各被 6-8 个测试文件 spy）。

---

## 8. Token Budget — Token 预算追踪

### 8.1 文件位置

`src/query/tokenBudget.ts` (93 行)

### 8.2 BudgetTracker 类型

```typescript
export type BudgetTracker = {
  continuationCount: number       // 已继续次数
  lastDeltaTokens: number         // 上次检查的 delta token
  lastGlobalTurnTokens: number    // 上次全局轮次 token 数
  startedAt: number               // 开始时间戳
}
```

### 8.3 checkTokenBudget 决策逻辑

```typescript
export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,  // 子代理跳过预算检查
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision
```

**决策规则**:

1. **跳过条件**: `agentId` 存在（子代理）或 `budget` 为 null/非正数 → 返回 `stop`
2. **计算比例**: `pct = turnTokens / budget * 100`
3. **计算 delta**: `deltaSinceLastCheck = globalTurnTokens - tracker.lastGlobalTurnTokens`
4. **递减检测**: 当 `continuationCount >= 3` 且连续两次 `delta < 500` (DIMINISHING_THRESHOLD) → `isDiminishing = true`
5. **继续条件**: 非递减 且 `turnTokens < budget * 0.9` (COMPLETION_THRESHOLD=90%) → `continue`
6. **停止条件**: 递减或已有继续计数 → `stop`（附带完成事件）

### 8.4 Token Budget Decision 类型

```typescript
type ContinueDecision = {
  action: 'continue'
  nudgeMessage: string        // 提示消息（百分比、已用/总量）
  continuationCount: number
  pct: number
  turnTokens: number
  budget: number
}

type StopDecision = {
  action: 'stop'
  completionEvent: {          // null 表示未参与预算系统
    continuationCount: number
    pct: number
    turnTokens: number
    budget: number
    diminishingReturns: boolean
    durationMs: number
  } | null
}
```

### 设计理念：为什么需要递减检测 (diminishing returns)

`checkTokenBudget()`（`src/query/tokenBudget.ts:45`）在判断是否继续循环时，不仅检查 Token 用量百分比，还检测"递减收益"——当 `continuationCount >= 3` 且连续两次迭代的 `delta < 500 tokens`（`DIMINISHING_THRESHOLD`）时，强制终止。

这个机制防止模型陷入无限循环：当模型在连续 3 次迭代中每次只产生不到 500 个新 token，说明它在重复自己（例如反复修改同一段代码然后撤销）而非实质性推进任务。没有这个检测，一个陷入重复的模型会持续消耗 API 额度直到 Token Budget 完全耗尽，而实际产出为零。

递减检测与百分比阈值（`COMPLETION_THRESHOLD = 0.9`，即 90%）协同工作：正常推进任务时，模型在达到 90% 预算后停止；异常重复时，模型在连续低产出后提前停止。`completionEvent` 中的 `diminishingReturns: boolean` 字段让调用方可以区分这两种停止原因。

---

## 9. Stop Hooks — 停止钩子处理

### 9.1 文件位置

`src/query/stopHooks.ts`

### 9.2 handleStopHooks() 签名

```typescript
export async function* handleStopHooks(
  messagesForQuery: Message[],
  assistantMessages: AssistantMessage[],
  systemPrompt: SystemPrompt,
  userContext: { [k: string]: string },
  systemContext: { [k: string]: string },
  toolUseContext: ToolUseContext,
  querySource: QuerySource,
  stopHookActive?: boolean,
): AsyncGenerator<..., StopHookResult>
```

### 9.3 返回类型

```typescript
type StopHookResult = {
  blockingErrors: Message[]        // 阻塞性错误消息
  preventContinuation: boolean     // 是否阻止继续
}
```

### 9.4 执行的钩子

`handleStopHooks` 按顺序执行以下操作：

1. **saveCacheSafeParams** — 保存缓存安全参数（仅主线程查询）
2. **模板工作分类** (feature `TEMPLATES`) — 分类工作类型
3. **executeStopHooks** — 执行用户配置的 `stop` 事件钩子
4. **executeTaskCompletedHooks / executeTeammateIdleHooks** — 任务/团队钩子
5. **executeExtractMemories** (feature `EXTRACT_MEMORIES`) — 自动提取记忆到 CLAUDE.md
6. **executePromptSuggestion** — 生成下一步建议提示
7. **executeAutoDream** — 自动梦境（会话间自主任务）
8. **cleanupComputerUseAfterTurn** — 清理 Computer-Use 资源

### 9.5 stopHookActive 标志

当停止钩子返回 `blockingErrors` 时，`stopHookActive` 被设置为 `true`，防止在后续循环迭代中再次执行停止钩子（避免无限递归）。

### 设计理念：为什么停止钩子是 generator 而不是普通函数

`handleStopHooks()` 的签名是 `async function*`（`src/query/stopHooks.ts:65`），返回 `AsyncGenerator<StreamEvent | ... , StopHookResult>`——和 `query()` 本身一样是 generator。这不是为了风格一致，而是因为停止钩子内部执行的操作本身需要流式通信：

1. **`executeExtractMemories`** — 调用 API 从对话中提取记忆写入 CLAUDE.md，这是一个异步操作，需要向 UI 推送进度事件。
2. **`executePromptSuggestion`** — 调用 API 生成下一步建议提示。
3. **`executeStopHooks`** — 执行用户配置的 `stop` 事件钩子，每个钩子都可能是外部命令执行，需要通过 `yield` 推送 `HookProgress` 事件（包含 `toolUseID`、`command`、`promptText`，`src/query/stopHooks.ts:200-214`）。
4. **`executeAutoDream`** — 自动梦境的后台任务启动。

如果 `handleStopHooks` 是普通 `async` 函数，这些中间进度事件无法传递给调用方（`query()` 的 generator），UI 在停止钩子执行期间将完全无响应。generator 嵌套（`yield* handleStopHooks(...)`）使得停止钩子的进度事件可以透明地向上冒泡到最外层消费者。

---

## 10. Query Chain Tracking — 查询链追踪

### 10.1 类型定义

```typescript
// src/Tool.ts, line 90
export type QueryChainTracking = {
  chainId: string   // UUID，在整个用户轮次（包括所有工具调用回合）中保持不变
  depth: number     // 每次递归/子代理调用时递增
}
```

### 10.2 初始化与递增

```typescript
// query.ts, 每次循环迭代的头部
const queryTracking = toolUseContext.queryTracking
  ? {
      chainId: toolUseContext.queryTracking.chainId,
      depth: toolUseContext.queryTracking.depth + 1,
    }
  : {
      chainId: deps.uuid(),  // 首次调用生成新 UUID
      depth: 0,
    }
```

### 10.3 用途

- **遥测关联**: `queryChainId` 和 `queryDepth` 被传递到所有 `logEvent` 调用，用于在分析中关联同一轮次的多个 API 请求
- **子代理追踪**: AgentTool 创建子代理时传递 `queryTracking`，depth 递增
- **分析面板**: 按 chainId 聚合可以看到一个用户请求触发的完整 API 调用链

---

## 11. 跨轮次持久化状态

以下状态在 `while(true)` 循环的多次迭代间持久化：

| 状态 | 存储位置 | 用途 |
|------|----------|------|
| `messages` | `State.messages` | 完整消息历史（含工具结果），每次迭代追加 |
| `toolUseContext` | `State.toolUseContext` | 工具上下文（含 readFileState LRU 缓存） |
| `autoCompactTracking` | `State.autoCompactTracking` | 压缩状态（是否已压缩、轮次计数器、连续失败数） |
| `taskBudgetRemaining` | 循环局部变量 | task_budget 剩余量（跨压缩边界累积） |
| `pendingToolUseSummary` | `State.pendingToolUseSummary` | 异步工具摘要 Promise |
| `budgetTracker` | 循环局部变量 | Token 预算追踪器（continuationCount/lastDelta） |
| `taskBudgetRemaining` | 循环局部变量 | 任务预算剩余 token（跨压缩累积） |

### 11.1 跨压缩边界的 taskBudget 追踪

```typescript
// 压缩发生时捕获压缩前的最终上下文窗口
if (params.taskBudget) {
  const preCompactContext =
    finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

---

## 12. 错误恢复机制

### 12.1 max_output_tokens 恢复

- 限制: `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3` 次
- 触发: 助手消息的 `apiError === 'max_output_tokens'`
- 行为: 递增 `maxOutputTokensRecoveryCount`，将助手消息作为部分结果附加，继续循环

### 12.2 Reactive Compact (413 恢复)

- 触发: prompt_too_long API 错误 (413)
- 限制: 每次循环迭代仅尝试一次 (`hasAttemptedReactiveCompact`)
- 行为: 压缩当前消息，使用压缩后消息重试
- Gate: `feature('REACTIVE_COMPACT')`

### 12.3 Streaming Fallback

- 触发: `FallbackTriggeredError` 在流式传输中抛出
- 行为: 丢弃 StreamingToolExecutor 中的待处理结果，使用 fallbackModel 重试
- 注意: withheld max_output_tokens 消息不会在恢复循环期间泄漏给 SDK 调用方

### 12.4 Thinking 规则

代码注释中记录的三条规则（"The rules of thinking"）:

1. 包含 thinking/redacted_thinking block 的消息必须属于 `max_thinking_length > 0` 的查询
2. Thinking block 不得是消息中的最后一个 block
3. Thinking blocks 必须在整个助手轨迹期间保持不变（包括后续的 tool_result 和助手消息）

---

## 工程实践指南

### 调试查询循环

`query.ts` 的 `while(true)` 循环是整个系统的运行时核心。调试时关注以下要点：

1. **在 `state = { ... }` 站点设断点** — 每个 `continue` 站点都是一次状态转换，`transition.reason` 字段告诉你为什么循环要重新开始（如 `'reactive_compact_retry'`、`'max_output_tokens_escalate'`、`'fallback'`）
2. **跟踪 State 变化** — 在循环头部的 `const { messages, ... } = state` 解构处设断点，观察每次迭代时 `messages` 数组的长度变化、`turnCount` 递增等
3. **检查 `needsFollowUp`** — 这个标志决定循环是否继续（Phase 3 检测到 `tool_use` block 时设为 true）。如果循环意外停止或不停止，检查此标志
4. **使用 QueryDeps 注入调试** — 通过 `params.deps` 注入自定义的 `callModel`/`autocompact`/`microcompact`，可以在不修改源码的情况下拦截和检查中间状态

### 添加新的终止原因

如果需要添加新的 `Terminal` reason（例如新的错误类型或停止条件）：

1. **在 `Terminal` 类型中添加新的 reason** — 修改 `query.ts` 中的 `Terminal` 类型定义
2. **在 `queryLoop()` 中添加对应的 `return` 条件** — 在适当的 Phase 中添加 `return { reason: 'your_new_reason', ... }`
3. **在 UI 中添加对应的展示逻辑** — 不同终止原因在 UI 中有不同的展示（错误面板、建议提示等），需要在消费 generator 的代码中处理新 reason
4. **在 SDK/QueryEngine 中处理** — `QueryEngine.ts` 的 `ask()` 方法需要知道如何将新 reason 转换为 SDK 响应
5. **添加测试** — 覆盖触发新终止原因的条件路径

### 添加新的 Phase

在 `while(true)` 循环中插入新阶段时：

1. **确定位置** — 5 个 Phase 的顺序有依赖关系：上下文准备 → API 调用 → 工具执行 → 停止钩子 → 继续/终止。新阶段应该插入到逻辑上正确的位置
2. **注意 continue 站点** — 插入新阶段可能影响已有的 7 个 `state = { ... }; continue` 站点。确保新阶段不会在意外的位置被跳过
3. **保持 generator 语义** — 如果新阶段需要向调用方报告进度，使用 `yield` 推送事件；如果需要终止循环，使用 `return Terminal`
4. **考虑对 `using` 声明的影响** — `query.ts` 使用 `using` 声明管理资源（如 `pendingMemoryPrefetch`），确保新阶段不会干扰资源的自动 dispose

### 调试上下文压缩问题

当模型行为异常且怀疑是压缩导致时：

1. **禁用自动压缩** — 设 `CLAUDE_CODE_DISABLE_AUTO_COMPACT=true`，看问题是否消失
2. **检查压缩触发条件** — `autocompact` 在上下文接近窗口限制时触发（保留 13K buffer）。如果频繁触发，可能是消息历史增长过快
3. **检查压缩后消息** — 压缩后 `messagesForQuery` 是新数组（不是 `state.messages` 的引用），内容可能被显著缩减。如果模型"忘记"了之前的上下文，检查压缩是否过于激进
4. **检查 `autoCompactTracking`** — `State.autoCompactTracking` 记录了是否已压缩、连续失败次数等状态，可以用于诊断压缩行为

### Token 预算调试

当模型提前停止或延迟停止时：

1. **打印 `checkTokenBudget` 的决策数据** — 关注以下字段：
   - `pct` — 当前 token 使用百分比（相对于 budget）
   - `delta` (即 `deltaSinceLastCheck`) — 上次检查以来的新增 token 数
   - `continuationCount` — 已经继续的次数
   - `isDiminishing` — 是否检测到递减收益（`continuationCount >= 3` 且连续两次 `delta < 500`）
2. **检查 `COMPLETION_THRESHOLD`** — 默认 0.9（90%），达到此比例后停止。如果需要更多预算利用率，调整此值
3. **检查 `DIMINISHING_THRESHOLD`** — 默认 500 tokens。如果模型在做有意义但产出较少的工作（如反复微调代码），可能被误判为递减

### 常见陷阱

1. **不要在 generator 外部修改 State** — `State` 只在 `queryLoop` 函数内可见，且只在 `continue` 站点通过批量赋值更新。在回调函数或工具执行中修改 State 会导致不确定行为

2. **不要假设 `messagesForQuery` 和 `state.messages` 是同一个引用** — 压缩（autocompact/reactiveCompact）后会创建新数组。如果在压缩后仍持有旧 `messages` 引用，操作的是过期数据

3. **`StreamingToolExecutor` 的 `discard()` 必须在 fallback 前调用** — 流式降级（`FallbackTriggeredError`）时，必须先调用 `discard()` 丢弃待处理的结果，否则旧模型的部分结果会泄漏到新模型的上下文中

4. **不要在 Phase 2（API 调用）和 Phase 3（工具执行）之间插入长时间操作** — 这个间隙中 `assistantMessages` 已经生成但工具尚未执行，此时的消息状态是不完整的

5. **`stopHookActive` 防止停止钩子递归** — 当停止钩子返回 `blockingErrors` 后，`stopHookActive` 设为 true，防止下一轮循环再次执行停止钩子。如果你的代码依赖停止钩子的执行，注意这个标志可能导致钩子被跳过

6. **Token Budget 的 `completionEvent` 可能为 null** — `StopDecision.completionEvent` 为 null 表示查询未参与预算系统（如子代理查询），不要对 null 值调用属性访问


---

[← 启动与初始化](../02-启动与初始化/initialization.md) | [目录](../README.md) | [API 客户端 →](../04-API客户端/api-client.md)
