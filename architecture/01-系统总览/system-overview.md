# 系统总览 - Claude Code v2.1.88

> 基于 1884 个 TypeScript 源文件的逆向工程分析
> Build: 2026-03-30T21:59:52Z

---

## 1. 核心统计

| 维度 | 数量 |
|------|------|
| TypeScript 源文件 (.ts/.tsx) | 1884 |
| 内置工具 (Tools) | 40+ |
| React Hooks | 70+ |
| Slash 命令 | 87+ (101 个命令目录) |
| 后台服务 (Services) | 13 |
| 内置技能 (Skills) | 17 |
| Hook 事件类型 | 13 |
| 权限模式 (Permission Modes) | 6 (default/plan/acceptEdits/bypassPermissions/dontAsk/auto) + 1 内部 (bubble) |
| API 后端 (Backends) | 4 (Anthropic/Bedrock/Vertex/Foundry) |
| MCP 传输协议 | 4 (stdio/sse/streamable-http/local) |

---

## 2. 源码树结构

```
claudecode/sourcecode/src/
├── QueryEngine.ts          # SDK/print 模式的查询引擎入口（ask() 生成器）
├── Task.ts                 # 后台任务基类定义
├── Tool.ts                 # Tool 类型接口 + ToolUseContext (792 行)
├── commands.ts             # 命令注册表 + getSlashCommandToolSkills()
├── context.ts              # 全局 Context 工厂
├── cost-tracker.ts         # 费用追踪（getModelUsage/getTotalCost）
├── costHook.ts             # 费用变更钩子
├── dialogLaunchers.tsx     # 对话框启动器
├── history.ts              # 会话历史管理
├── ink.ts                  # Ink 渲染引擎入口
├── interactiveHelpers.tsx  # 交互式 UI 辅助组件
├── main.tsx                # 应用主入口（REPL 模式）
├── projectOnboardingState.ts # 项目引导状态
├── query.ts                # 核心查询循环 (1729 行) — async generator
├── replLauncher.tsx        # REPL 启动器
├── setup.ts                # 初始化设置
├── tasks.ts                # 任务系统入口
├── tools.ts                # 工具注册总表（getAllBaseTools/getTools/assembleToolPool）
│
├── assistant/              # 助手消息处理
├── bootstrap/              # 启动引导（state.ts 单例状态、growthbook 初始化）
├── bridge/                 # Bridge 协议（IDE 双向通信，33 文件）
├── buddy/                  # 伴侣宠物系统（PRNG + 精灵渲染）
├── cli/                    # CLI 入口与参数解析
├── commands/               # 87+ Slash 命令实现（101 个子目录）
│   ├── add-dir/
│   ├── clear/
│   ├── commit.ts
│   ├── compact/
│   ├── config/
│   ├── ... (101 directories/files total)
│
├── components/             # React/Ink UI 组件库
├── constants/              # 全局常量（betas, oauth, xml tags, querySource）
├── context/                # 上下文管理（notifications, providers）
├── coordinator/            # 协调器模式（多 Worker 编排）
├── entrypoints/            # 多入口点（SDK, print, headless, HFI）
├── hooks/                  # React Hooks（70+ 个）
│   ├── useCanUseTool.tsx   # 权限决策核心 Hook
│   ├── useTextInput.ts     # 文本输入
│   ├── useVimInput.ts      # Vim 模式输入
│   ├── useVoice.ts         # 语音输入
│   ├── toolPermission/     # 工具权限 UI 子系统
│   ├── notifs/             # 通知子系统
│   └── ... (85+ files)
│
├── ink/                    # Ink 渲染引擎扩展
├── keybindings/            # 键绑定系统（50+ 动作，和弦支持）
├── memdir/                 # 记忆目录系统（CLAUDE.md 读取与管理）
├── migrations/             # 数据迁移
├── moreright/              # 右侧面板扩展
├── native-ts/              # 原生 TypeScript 模块（FFI 桥接）
├── outputStyles/           # 输出样式系统（Markdown 前置 matter）
├── plugins/                # 插件系统入口
├── query/                  # 查询子模块
│   ├── config.ts           # QueryConfig 类型（sessionId + gates）
│   ├── deps.ts             # QueryDeps 依赖注入（callModel/microcompact/autocompact/uuid）
│   ├── stopHooks.ts        # 停止钩子处理（handleStopHooks）
│   └── tokenBudget.ts      # Token 预算追踪（BudgetTracker）
│
├── remote/                 # 远程会话（CCR WebSocket）
├── schemas/                # Zod 校验 Schema
├── screens/                # 全屏视图组件
├── server/                 # 内嵌服务器（LSP、Bridge）
├── services/               # 后台服务层（13 个子系统）
│   ├── analytics/          # 遥测分析（GrowthBook + Statsig + OTel）
│   ├── api/                # API 客户端（client.ts/claude.ts/withRetry.ts/errors.ts/logging.ts）
│   ├── autoDream/          # 自动梦境（会话间自主任务）
│   ├── compact/            # 上下文压缩（micro/auto/reactive/snip）
│   ├── extractMemories/    # 记忆提取服务
│   ├── lsp/                # LSP 集成（JSON-RPC）
│   ├── mcp/                # MCP 协议实现（配置/传输/认证/延迟加载）
│   ├── oauth/              # OAuth 认证（PKCE 流程）
│   ├── plugins/            # 插件服务
│   ├── policyLimits/       # 策略限制
│   ├── remoteManagedSettings/ # 远程管理设置
│   ├── settingsSync/       # 设置同步
│   ├── teamMemorySync/     # 团队记忆同步
│   ├── tips/               # 提示服务
│   ├── tokenEstimation.ts  # Token 估算
│   ├── toolUseSummary/     # 工具使用摘要生成
│   ├── tools/              # 工具编排层（StreamingToolExecutor/toolExecution/toolOrchestration）
│   ├── AgentSummary/       # 代理摘要
│   ├── MagicDocs/          # 魔法文档
│   ├── PromptSuggestion/   # 提示建议
│   ├── SessionMemory/      # 会话记忆
│   └── voice.ts            # 语音服务
│
├── skills/                 # 技能系统
│   ├── bundled/            # 17 个内置技能
│   │   ├── claudeApi.ts
│   │   ├── claudeApiContent.ts
│   │   ├── claudeInChrome.ts
│   │   ├── debug.ts
│   │   ├── keybindings.ts
│   │   ├── loop.ts
│   │   ├── loremIpsum.ts
│   │   ├── remember.ts
│   │   ├── scheduleRemoteAgents.ts
│   │   ├── simplify.ts
│   │   ├── skillify.ts
│   │   ├── stuck.ts
│   │   ├── updateConfig.ts
│   │   ├── verify.ts
│   │   ├── verifyContent.ts
│   │   ├── batch.ts
│   │   └── index.ts
│   ├── bundledSkills.ts
│   ├── loadSkillsDir.ts
│   └── mcpSkillBuilders.ts
│
├── state/                  # 状态管理（AppState + Zustand store）
├── tasks/                  # 任务系统实现
├── tools/                  # 工具实现（40+ 工具）
│   ├── AgentTool/          # 子代理工具
│   ├── AskUserQuestionTool/# 用户交互工具
│   ├── BashTool/           # Shell 执行
│   ├── BriefTool/          # 简报工具
│   ├── ConfigTool/         # 配置工具（ant-only）
│   ├── EnterPlanModeTool/  # 进入计划模式
│   ├── EnterWorktreeTool/  # 进入 Worktree
│   ├── ExitPlanModeTool/   # 退出计划模式
│   ├── ExitWorktreeTool/   # 退出 Worktree
│   ├── FileEditTool/       # 文件编辑（精确替换）
│   ├── FileReadTool/       # 文件读取
│   ├── FileWriteTool/      # 文件写入
│   ├── GlobTool/           # 文件模式搜索
│   ├── GrepTool/           # 内容搜索（ripgrep）
│   ├── LSPTool/            # LSP 工具
│   ├── ListMcpResourcesTool/ # MCP 资源列表
│   ├── MCPTool/            # MCP 工具桥接
│   ├── McpAuthTool/        # MCP 认证
│   ├── NotebookEditTool/   # Notebook 编辑
│   ├── PowerShellTool/     # PowerShell (Windows)
│   ├── REPLTool/           # REPL 工具（ant-only）
│   ├── ReadMcpResourceTool/# MCP 资源读取
│   ├── RemoteTriggerTool/  # 远程触发
│   ├── ScheduleCronTool/   # Cron 调度（Create/Delete/List）
│   ├── SendMessageTool/    # 消息发送
│   ├── SkillTool/          # 技能执行
│   ├── SleepTool/          # 休眠工具
│   ├── SyntheticOutputTool/# 合成输出
│   ├── TaskCreateTool/     # 任务创建
│   ├── TaskGetTool/        # 任务查询
│   ├── TaskListTool/       # 任务列表
│   ├── TaskOutputTool/     # 任务输出
│   ├── TaskStopTool/       # 任务停止
│   ├── TaskUpdateTool/     # 任务更新
│   ├── TeamCreateTool/     # 团队创建
│   ├── TeamDeleteTool/     # 团队删除
│   ├── TodoWriteTool/      # Todo 写入
│   ├── ToolSearchTool/     # 工具搜索（延迟加载支持）
│   ├── WebFetchTool/       # 网页抓取
│   ├── WebSearchTool/      # 网页搜索
│   ├── shared/             # 共享工具基础设施
│   ├── testing/            # 测试用工具
│   └── utils.ts            # 工具公用函数
│
├── types/                  # 类型定义
│   ├── message.ts          # 消息类型全集
│   ├── permissions.ts      # 权限类型（PermissionMode/Rule/Behavior）
│   ├── hooks.ts            # Hook 类型
│   ├── tools.ts            # 工具进度类型
│   ├── ids.ts              # ID 类型（AgentId/SessionId）
│   └── utils.ts            # 工具类型（DeepImmutable）
│
├── upstreamproxy/          # 上游代理
├── utils/                  # 工具函数库（最大子目录）
│   ├── permissions/        # 权限实现（24 文件）
│   ├── hooks/              # Hook 工具函数
│   ├── model/              # 模型选择与路由
│   ├── memory/             # 记忆管理
│   ├── settings/           # 设置加载
│   ├── shell/              # Shell 工具
│   ├── sandbox/            # 沙箱系统
│   ├── telemetry/          # 遥测工具
│   ├── messages.ts         # 消息构造与标准化
│   ├── tokens.ts           # Token 计数
│   ├── context.ts          # 上下文窗口计算
│   ├── config.ts           # 配置管理
│   └── ... (100+ files)
│
└── vim/                    # Vim 模式实现（完整状态机）
└── voice/                  # 语音系统
```

---

## 3. 分层架构

![6-Layer Architecture](layered-architecture.svg)

### 设计理念：为什么采用 6 层架构而非 MVC

传统 CLI 工具通常采用 MVC 或简单的 Controller→Service 两层模型。Claude Code 的 6 层（UI→Hooks→State→Query→Services→Tools）看似过度设计，但每一层的存在都源于具体的工程约束：

1. **UI 和 Hooks 分离** — UI 层是纯渲染（React/Ink 组件），Hooks 层封装副作用和状态逻辑。这允许 70+ 个 hooks 被不同的 UI 组件组合复用，而不是把逻辑嵌入组件树。证据：`hooks/useCanUseTool.tsx` 被权限请求 UI、工具执行流程和 auto 模式分类器三个完全不同的场景调用。

2. **State 和 Query 分离** — State 层（`bootstrap/state.ts` 全局单例 + Zustand store）管理进程级生命周期状态；Query 层（`query.ts` 的 async generator）管理单次对话轮次的瞬态。如果合并，进程级状态（如 `totalCostUSD`、`sessionId`）和轮次级状态（如 `messages`、`turnCount`）的生命周期混淆会导致状态泄漏。

3. **Services 和 Tools 分离** — Services 是无状态的能力提供者（API 客户端、压缩算法、MCP 协议），Tools 是有身份的执行单元（带名称、描述、权限要求）。分离使得同一个 Service（如 `services/api/claude.ts`）可以被 Query 引擎直接调用，也可以被 Tool 间接调用，而不需要 Tool 了解 API 细节。

核心洞察：这不是分层为了分层，而是 **生命周期管理** 的需要。6 层对应 3 种不同的生命周期——进程级（State/Infrastructure）、会话级（UI/Hooks）、轮次级（Query/Services/Tools）。MVC 只区分"展示"和"逻辑"，无法表达这种多级生命周期。

### 设计理念：为什么用 React/Ink 做 CLI

Claude Code 的终端输出不是传统的线性文本流——它同时存在多个动态更新的区域：

- **消息流区域** — 模型回复逐 token 流式渲染（`components/Messages.tsx`）
- **工具进度区域** — 并发工具执行的实时状态（`components/Spinner.tsx`、工具进度事件）
- **输入框区域** — 权限确认对话框可能在工具执行期间弹出（`components/PromptInput/`）
- **状态栏区域** — Token 用量、费用、模型信息持续更新（`components/StatusLine.tsx`、`components/Stats.tsx`）
- **全屏覆盖** — 设置、上下文可视化、会话恢复等全屏视图（`screens/`）

如果用传统 `console.log` + ANSI 转义序列实现，开发者需要手动追踪每个区域的行位置、处理重叠刷新、管理光标状态——这本质上是在重新发明 UI 框架。React 的声明式模型将这些复杂度封装在了调和（reconciliation）算法中：每个组件只声明"我当前应该长什么样"，Ink 引擎自动计算最小终端更新。

证据：`src/components/` 包含 50+ 个组件文件，`src/screens/` 包含全屏视图组件，`src/hooks/` 包含 70+ 个 React Hooks——这种规模的 UI 复杂度用命令式方法维护将是一场灾难。Ink 把这个问题降维成了前端工程师熟悉的 React 组件开发。

---

## 4. 模块依赖图

### 4.1 核心依赖链

![Core Dependency Chain](../diagrams/core-dependency-chain.svg)

### 4.2 工具系统依赖

```
tools.ts (注册总表)
  ├─→ Tool.ts (Tool 类型接口 + ToolUseContext)
  ├─→ tools/AgentTool/      ← 创建子代理, 递归调用 query
  ├─→ tools/BashTool/       ← 执行 shell 命令
  ├─→ tools/SkillTool/      ← 执行技能 (fork agent)
  ├─→ tools/FileEditTool/   ← 精确文件编辑
  ├─→ tools/FileReadTool/   ← 读取文件
  ├─→ tools/FileWriteTool/  ← 写入文件
  ├─→ tools/GlobTool/       ← 文件搜索
  ├─→ tools/GrepTool/       ← 内容搜索
  ├─→ tools/MCPTool/        ← MCP 工具桥接
  ├─→ tools/WebFetchTool/   ← 网页抓取
  ├─→ tools/WebSearchTool/  ← 网页搜索
  └─→ tools/ToolSearchTool/ ← 工具延迟发现
```

### 4.3 权限系统依赖

```
hooks/useCanUseTool.tsx (权限决策入口)
  └─→ utils/permissions/permissions.ts (canUseTool 管线)
        ├─→ utils/permissions/PermissionRule.ts (规则类型)
        ├─→ utils/permissions/PermissionMode.ts (模式定义)
        ├─→ utils/permissions/yoloClassifier.ts (auto 模式分类器)
        ├─→ utils/permissions/bashClassifier.ts (Bash 命令分类)
        ├─→ utils/permissions/pathValidation.ts (路径安全)
        ├─→ utils/permissions/dangerousPatterns.ts (危险模式检测)
        ├─→ utils/permissions/shellRuleMatching.ts (Shell 规则匹配)
        └─→ utils/sandbox/sandbox-adapter.ts (沙箱执行)
```

### 4.4 服务层内部依赖

```
services/
  ├─→ api/ ←── query.ts, QueryEngine.ts, services/compact/
  │     client.ts ← claude.ts ← withRetry.ts
  │     errors.ts ← claude.ts, withRetry.ts, query.ts
  │     logging.ts ← claude.ts
  │
  ├─→ compact/ ←── query.ts
  │     microCompact.ts ← autoCompact.ts ← compact.ts
  │     autoCompact.ts → api/claude.ts (getMaxOutputTokensForModel)
  │     compact.ts → api/claude.ts (queryModelWithStreaming)
  │
  ├─→ tools/ ←── query.ts
  │     toolOrchestration.ts → toolExecution.ts → Tool.ts
  │     StreamingToolExecutor.ts → toolExecution.ts
  │     toolHooks.ts → utils/hooks.ts
  │
  ├─→ mcp/ ←── tools/MCPTool, tools.ts (assembleToolPool)
  ├─→ analytics/ ←── 几乎所有模块 (logEvent 全局调用)
  └─→ oauth/ ←── services/api/client.ts, utils/auth.ts
```

### 4.5 跨层关键路径

| 路径 | 流向 |
|------|------|
| 用户输入 → 模型响应 | `useTextInput` → `processUserInput` → `query()` → `claude.ts` → API |
| 工具执行 | `query()` → `toolOrchestration` → `toolExecution` → `canUseTool` → tool.execute() |
| 上下文压缩 | `query()` → `microcompact` → `autocompact` → API (or `reactiveCompact` on 413) |
| 权限判定 | `toolExecution` → `canUseTool` → rules → classifier → user prompt |
| MCP 桥接 | `tools.ts` → `assembleToolPool` → MCP clients → MCPTool.execute() |
| 技能执行 | `SkillTool` → `runForkedAgent` → 新 `query()` 实例 |
| 子代理 | `AgentTool` → `createSubagentContext` → 新 `query()` 实例 |

---

## 5. 入口点矩阵

| 入口点 | 文件 | 用途 |
|--------|------|------|
| REPL (交互式) | `main.tsx` → `replLauncher.tsx` | 终端交互式会话 |
| Print (非交互式) | `entrypoints/print/` | 单次查询后退出 |
| SDK | `entrypoints/sdk/` → `QueryEngine.ts` | 编程式 API |
| Headless | `entrypoints/headless/` | 无 UI 后台运行 |
| HFI (Human-Friendly Interface) | `entrypoints/hfi/` | Web 友好接口 |
| Bridge | `bridge/` | IDE 双向通信 |
| CLI | `cli/` | 命令行参数解析 |

---

## 6. 构建与运行时特性

### 6.1 Feature Flags (编译时)

使用 `feature('FLAG_NAME')` 进行编译时特性门控（Bun bundler tree-shaking），未启用的代码路径在构建时完全移除：

- `REACTIVE_COMPACT` — 响应式压缩（413 触发）
- `CONTEXT_COLLAPSE` — 上下文折叠
- `HISTORY_SNIP` — 历史裁剪
- `TOKEN_BUDGET` — Token 预算
- `EXTRACT_MEMORIES` — 记忆提取
- `TEMPLATES` — 模板/工作分类
- `EXPERIMENTAL_SKILL_SEARCH` — 技能搜索
- `TRANSCRIPT_CLASSIFIER` — 转录分类器（auto 模式）
- `COORDINATOR_MODE` — 协调器模式
- `BASH_CLASSIFIER` — Bash 命令分类器（ant-only）
- `CACHED_MICROCOMPACT` — 缓存微压缩
- `BG_SESSIONS` — 后台会话
- `PROACTIVE` / `KAIROS` — 主动式代理
- `AGENT_TRIGGERS` / `AGENT_TRIGGERS_REMOTE` — 代理触发器
- `MONITOR_TOOL` — 监控工具
- `OVERFLOW_TEST_TOOL` — 溢出测试
- `TERMINAL_PANEL` — 终端面板
- `WEB_BROWSER_TOOL` — Web 浏览器工具
- `UDS_INBOX` — Unix Domain Socket 收件箱
- `WORKFLOW_SCRIPTS` — 工作流脚本

#### 为什么 Feature Flags 用编译时 tree-shaking

Claude Code 的 `feature('FLAG_NAME')` 不是运行时的 `if (config.featureEnabled('FLAG_NAME'))` 检查——它是 Bun bundler 的编译时宏（`from 'bun:bundle'`），未启用的代码路径在构建时被完全删除。全代码库有 196 个文件使用了 `feature()` 调用。

这个选择在安全性和灵活性之间做出了明确的权衡：

**安全性（编译时删除的核心优势）：**
- 代码不存在 = 不可能被利用。例如 `BASH_CLASSIFIER`（ant-only）如果只是运行时检查，逆向工程仍然可以找到分类器逻辑；编译时删除后，外部构建中这段代码物理上不存在。
- 减少攻击面：像 `OVERFLOW_TEST_TOOL`、`MONITOR_TOOL` 这类调试工具在生产构建中被完全移除，不可能通过注入环境变量激活。

**灵活性的代价：**
- 改变 feature flag 需要重新构建和发布——不能像 LaunchDarkly 那样远程实时切换。
- 这就是为什么 `QueryConfig`（`src/query/config.ts`）刻意排除 `feature()` 门控，只包含运行时可变的 statsig/env 状态：编译时和运行时的门控是两个独立的系统。

**实现细节：** `feature()` 调用只能出现在 `if` 条件或三元表达式中（`src/query.ts:796` 注释："feature() only works in if/ternary conditions (bun:bundle...)"），确保 bundler 能正确识别和删除死代码分支。条件 `require()` 模式（如 `const reactiveCompact = feature('REACTIVE_COMPACT') ? require(...) : null`，`src/query.ts:15-17`）将整个模块依赖树从构建中排除。

### 6.2 环境变量门控

- `USER_TYPE=ant` — Anthropic 内部员工特性
- `CLAUDE_CODE_SIMPLE=true` — 简化模式（仅 Bash/Read/Edit）
- `CLAUDE_CODE_DISABLE_FAST_MODE` — 禁用快速模式
- `NODE_ENV=test` — 测试环境（启用 TestingPermissionTool）
- `CLAUDE_CODE_VERIFY_PLAN=true` — 计划验证工具

---

## 7. 数据流概览

```
User Input
    │
    ▼
processUserInput() ─── 命令检测 ──→ Slash Command 处理
    │
    ▼
query() async generator (while true loop)
    │
    ├─→ Phase 1: 上下文准备
    │     ├── applyToolResultBudget (>20KB 工具结果持久化到磁盘)
    │     ├── snipCompact (历史裁剪)
    │     ├── microcompact (工具结果压缩, COMPACTABLE_TOOLS)
    │     ├── contextCollapse (上下文折叠)
    │     └── autoCompact (自动压缩, 13K buffer)
    │
    ├─→ Phase 2: API 调用
    │     ├── 系统提示组装 (systemPrompt + userContext + systemContext)
    │     ├── queryModelWithStreaming → getAnthropicClient
    │     └── 流式接收 (streaming events → messages)
    │
    ├─→ Phase 3: 工具执行
    │     ├── partitionToolCalls (concurrent vs serial batching)
    │     ├── runToolUse (permission → execute → result)
    │     └── StreamingToolExecutor (流式工具并发)
    │
    ├─→ Phase 4: 停止钩子
    │     ├── executeStopHooks
    │     ├── executeExtractMemories
    │     ├── executePromptSuggestion
    │     ├── executeAutoDream
    │     └── cleanupComputerUseAfterTurn
    │
    └─→ Phase 5: 继续/终止判定
          ├── needsFollowUp → 继续循环 (tool_use blocks 存在)
          ├── Token Budget → 继续或停止
          └── 9 种终止原因之一 → return Terminal
```

### 设计理念：为什么核心是 async generator

`query()` 函数（`src/query.ts:219`）被声明为 `async function*`——这不是随意的语法选择，而是整个系统架构风格的基石。async generator 同时解决了四个核心问题：

1. **流式推送** — LLM 逐 token 生成内容，generator 通过 `yield` 逐步推送 `StreamEvent`、`Message` 等事件给调用方。调用方可以实时渲染，而不必等待完整响应。这使得 REPL 模式、SDK 模式和 Headless 模式可以用同一个 generator 但消费方式不同。

2. **背压控制** — 调用方通过 `for await...of` 按自己的节奏消费事件。如果 UI 渲染慢于 API 接收速度，generator 自然暂停在 `yield` 点，不会导致内存溢出。这比 EventEmitter 模式（`on('data')` 回调）安全得多。

3. **类型安全的多态返回** — generator 的 `yield` 类型（`StreamEvent | Message | ...`）和 `return` 类型（`Terminal`）是分离的。`yield` 推送中间事件，`return` 只用于终止原因。这比 EventEmitter 的字符串事件名更类型安全，TypeScript 编译器可以完整检查所有事件处理路径。

4. **优雅取消** — 调用方可以通过 `generator.return()` 立即终止循环，generator 内部的 `using` 声明（如 `pendingMemoryPrefetch`）会被自动 dispose。这比 `AbortController` 粒度更细——`AbortController` 只能取消 fetch 请求，generator 可以在任意 `yield` 点停止。

这个选择的连锁效应深远：因为核心是 generator，停止钩子（`stopHooks.ts`）也被设计为 generator（需要 yield 进度事件给 UI）；`QueryEngine.ask()` 也是 generator；甚至子代理（`AgentTool`）也通过嵌套 generator 运行。整个系统形成了一个 generator 组合的管道架构。

### 设计理念：为什么有 7 个入口点而不统一为一个

7 个入口点（REPL/Print/SDK/Headless/HFI/Bridge/CLI）看似可以统一为一个通用入口，但它们解决的部署场景有根本差异：

| 入口点 | 核心差异 | 不统一的原因 |
|--------|----------|-------------|
| **REPL** (`main.tsx`) | 完整 Ink 渲染 + 交互式输入循环 | 需要 React 组件树、键绑定系统、Vim 模式 |
| **Print** (`entrypoints/print/`) | 单次查询后退出 | 无 UI 循环，输出到 stdout/file，需要 `gracefulShutdown` |
| **SDK** (`entrypoints/sdk/`) | 编程式 API | 不需要 CLI 参数解析，返回结构化数据而非终端输出 |
| **Headless** (`entrypoints/headless/`) | 无 UI 后台运行 | 无终端依赖，适合 CI/CD |
| **HFI** (`entrypoints/hfi/`) | Web 友好接口 | HTTP 协议，JSON 序列化而非终端渲染 |
| **Bridge** (`bridge/`, 33 文件) | IDE 双向通信 | LSP 协议，需要维护长连接 + 双向消息 |
| **CLI** (`cli/`) | 命令行参数解析 | Commander.js 配置，是其他入口的前置路由 |

统一入口的代价是巨大的条件分支——每个入口对 I/O 模型（交互式/批处理/流式/双向）、生命周期管理（长驻/单次/按需）、输出格式（终端/JSON/LSP）的需求完全不同。分离入口让每个场景可以只加载自己需要的模块，同时共享下层的 Query 引擎和 Services 层。

证据：`src/main.tsx` 第 1 行的 import 副作用（`profileCheckpoint`/`startMdmRawRead`/`startKeychainPrefetch`）是 REPL 独有的启动优化，SDK 入口不需要也不应该执行这些操作。`CLAUDE_CODE_ENTRYPOINT` 环境变量（`src/interactiveHelpers.tsx`）的 10 种值正是为了在共享代码中区分调用来源。

---

## 工程实践指南

### 添加新子系统的清单

如果你需要为 Claude Code 添加一个全新的子系统（例如新的工具类别、新的服务模块），按以下步骤操作：

1. **在 `tools/` 目录下创建工具目录** — 例如 `tools/MyNewTool/`，实现 `Tool` 接口（参见 `src/Tool.ts`）
2. **注册到 `tools.ts`** — 在 `getAllBaseTools()` 的适当位置添加工具引用；如果需要 feature flag 门控，使用条件展开 `...(feature('MY_FLAG') ? [MyNewTool] : [])`
3. **添加权限规则** — 如果工具涉及文件系统或网络操作，在 `utils/permissions/` 中添加对应的权限检查逻辑；更新 `dangerousPatterns.ts`（如需要）
4. **添加测试** — 工具测试、权限测试、集成测试
5. **更新文档** — 在架构文档中记录新子系统的设计决策

**关键检查点**：
- 新工具必须实现 `isEnabled()` — 返回 false 时工具不会出现在工具池中
- 新工具必须实现 `isConcurrencySafe()` — 如果不确定，保守地返回 false
- 如果新工具需要 UI 交互，通过 `ToolUseContext` 的回调（如 `setToolJSX`、`addNotification`），不要直接导入 UI 模块

### 跨层调试技巧

| 调试场景 | 操作 |
|----------|------|
| **查看完整日志链** | 使用 `--debug` 启动，所有层的日志输出到 stderr |
| **强制串行执行** | 设 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY=1`，消除并发导致的不确定性，定位并发 bug |
| **跟踪完整 API 调用链** | 检查 `dumpPrompts` 输出（在 `services/api/dumpPrompts.ts` 中启用），查看发送给 API 的完整消息 |
| **追踪 query 循环状态变化** | 在 `query.ts` 的 `state = { ... }` continue 站点设断点，关注 `transition.reason` 字段 |
| **确认 feature flag 状态** | `feature()` 是编译时宏——如果功能"消失了"，检查构建配置中是否包含对应 flag，而不是运行时环境变量 |

### 性能分析入口

- **Perfetto trace**: 如果启用了 OTel tracing（`initializeTelemetryAfterTrust()` 之后），可以通过 Perfetto 查看完整调用链，包括 API 延迟、工具执行时间、压缩耗时
- **启动性能检查点**: `startupProfiler.ts` 中的 `profileCheckpoint()` 埋点覆盖了从 `main_tsx_entry` 到 `REPL` 渲染的完整启动链路
- **FPS Metrics**: UI 渲染性能通过 `FpsMetrics` 监控，关注 Ink 渲染引擎的帧率下降
- **Token 估算**: `services/tokenEstimation.ts` 提供 token 计数，用于诊断上下文窗口使用效率

### 常见架构陷阱

1. **不要在 Tools 层直接访问 UI 层** — 工具通过 `ToolUseContext` 的回调（`setToolJSX`、`addNotification`、`sendOSNotification`）与 UI 通信。直接 import UI 组件会破坏 Headless/SDK 模式的兼容性（这些模式没有 React 运行时）。

2. **不要在 Services 层假设交互式环境** — Services 可能在 headless 模式、SDK 模式或 CI 环境中运行。需要用户输入时检查 `isNonInteractiveSession`；需要 UI 反馈时检查 `shouldAvoidPermissionPrompts`。

3. **不要混淆编译时 flag 和运行时 flag** — `feature('FLAG_NAME')` 是 Bun bundler 的编译时宏，构建后不可变；`QueryConfig.gates` 中的 statsig/env 门控是运行时可变的。在错误的层级检查 flag 会导致"明明设了环境变量但功能不生效"的困惑。

4. **不要在全局单例初始化完成前访问状态** — `bootstrap/state.ts` 的 `getBootstrapState()` 在 `init()` 完成前调用会得到 undefined 或初始值。确保你的代码在 `init()` 链之后执行。

5. **不要绕过 query 循环直接调用 API** — `services/api/claude.ts` 的 `queryModelWithStreaming()` 需要配合 `withRetry.ts` 的重试/降级/冷却逻辑。直接调用 SDK 会跳过所有错误恢复机制。


---

[目录](../README.md) | [启动与初始化 →](../02-启动与初始化/initialization.md)
