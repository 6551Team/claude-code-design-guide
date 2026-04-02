# Swarm 多智能体系统架构

## Backend Architecture

Swarm 系统支持三种后端类型，通过 `BackendType` 联合类型定义：

```typescript
type BackendType = 'tmux' | 'iterm2' | 'in-process'
```

### 设计理念

#### 为什么 3 种后端 (tmux/iTerm2/in-process)?

不同环境有不同的最优解:
- **tmux**: 无头环境 (CI/SSH) 的唯一选择,最可靠,源码实现了完整的面板拆分、边框着色和锁机制
- **iTerm2**: 可视化调试场景,开发者可以直观看到每个 agent 的终端输出,适合调试和演示
- **in-process**: 测试和轻量场景,无子进程开销。源码注释:"Unlike process-based teammates (tmux/iTerm2), in-process teammates run in the same Node.js process"——共享内存但逻辑隔离,每个 teammate 有独立的 AbortController

后端检测采用分层优先级:已在 tmux 中 > iTerm2 可用 > fallback tmux > error,确保自动选择最合适的后端。

#### 为什么文件系统做权限同步而不是 IPC?

源码 `permissionSync.ts` 实现了 `writePermissionRequest()` / `readPendingPermissions()` / `resolvePermission()` 的文件级权限流。选择文件系统而非 IPC 管道/socket 有三个原因:
1. **跨进程/跨机器的通用性**——tmux 窗格、SSH 会话、in-process 模式都能访问文件
2. **审计日志**——权限请求和决策以文件形式持久存在,可以事后检查权限决策历史
3. **崩溃恢复**——文件系统在进程崩溃后仍然存在,IPC 管道/socket 随进程死亡而丢失,恢复后无法知道之前的权限状态

#### 为什么 17 个环境变量继承?

源码注释说明:"Tmux may start a new login shell that doesn't inherit the parent's env, so we forward any that are set in the current process"。`TEAMMATE_ENV_VARS` 包含 API provider 选择 (`CLAUDE_CODE_USE_BEDROCK` 等)、代理配置、CA 证书路径等关键变量。不继承这些,teammate 会默认使用 firstParty 端点并将请求发送到错误的地址 (源码引用 "GitHub issue #23561")。

### PaneBackend 接口

`PaneBackend` 是所有后端实现必须遵循的核心接口，定义了面板生命周期和视觉控制方法：

- **createTeammatePaneInSwarmView(name, command)**: 创建新的 teammate 面板并在 swarm 视图中显示
- **sendCommandToPane(paneId, command)**: 向指定面板发送命令执行
- **setPaneBorderColor(paneId, color)**: 设置面板边框颜色用于视觉区分
- **setPaneTitle(paneId, title)**: 设置面板标题
- **killPane(paneId)**: 终止并销毁面板
- **hidePane(paneId)**: 隐藏面板但保留进程
- **showPane(paneId)**: 显示之前隐藏的面板
- **rebalancePanes()**: 重新均衡所有面板的布局分配

### TeammateExecutor 接口

`TeammateExecutor` 管理 teammate 进程的生命周期：

- **spawn(config)**: 启动新的 teammate 进程
- **sendMessage(id, message)**: 向指定 teammate 发送消息
- **terminate(id)**: 优雅终止 teammate
- **kill(id)**: 强制终止 teammate
- **isActive(id)**: 检查 teammate 是否仍在活跃运行

### 后端检测优先级

系统通过分层检测确定最合适的后端：

1. **Inside tmux** (最高优先级): 检测到已在 tmux 会话中运行
2. **In iTerm2 with it2**: 检测到在 iTerm2 中运行且 `it2` CLI 可用
3. **Fallback tmux**: 系统安装了 tmux 但当前不在 tmux 中
4. **Error**: 无可用后端，抛出错误

### 缓存机制

为避免重复检测开销，系统维护以下缓存：

- **cachedBackend**: 缓存已初始化的后端实例
- **cachedDetectionResult**: 缓存后端类型检测结果
- **inProcessFallbackActive**: 标记是否已降级到 in-process 模式

---

## Permission Synchronization

Swarm 中 leader 与 teammate 之间的权限同步采用多种通信机制。

### 文件级权限流

基于文件系统的权限请求-响应流程：

- **writePermissionRequest()**: teammate 将权限请求写入共享目录
- **readPendingPermissions()**: leader 轮询读取待处理的权限请求
- **resolvePermission()**: leader 写入权限决议结果

### 邮箱级权限流

基于 mailbox 机制的异步权限通信：

- **sendPermissionRequestViaMailbox()**: teammate 通过 mailbox 发送权限请求
- **sendPermissionResponseViaMailbox()**: leader 通过 mailbox 返回权限响应
- **sendSandboxPermissionRequestViaMailbox()**: sandbox 环境下的专用权限请求

### SwarmPermissionRequest Schema

```typescript
interface SwarmPermissionRequest {
  id: string              // 唯一请求标识符
  workerId: string        // 发起请求的 worker ID
  workerName: string      // worker 的人类可读名称
  toolName: string        // 请求权限的工具名称
  status: 'pending' | 'approved' | 'denied'  // 当前状态
}
```

### Leader Bridge

Leader 端维护权限确认队列用于与 UI 交互：

- **registerLeaderToolUseConfirmQueue()**: 注册 leader 的工具使用确认队列
- **getLeaderToolUseConfirmQueue()**: 获取已注册的确认队列

### 轮询间隔

```typescript
const PERMISSION_POLL_INTERVAL_MS = 500
```

---

## In-Process Teammates

In-process 模式将 teammate 作为同进程内的独立 agent 运行，无需 tmux/iTerm2。

### 权限处理

**createInProcessCanUseTool()** 创建权限处理函数：

- 优先使用 bridge 通道进行权限确认
- Bridge 不可用时降级到 mailbox 方式
- 集成 Classifier 自动批准机制，对特定 bash 命令进行自动审批

### 进程管理

**spawnInProcessTeammate()** 启动 in-process teammate：

- 使用确定性 agentId 生成，确保可追踪
- 每个 teammate 拥有独立的 AbortController，支持单独终止
- 与宿主进程共享内存空间但逻辑隔离

**killInProcessTeammate()** 终止 in-process teammate：

- 触发 AbortController.abort() 终止运行
- 从 team file 中移除成员记录
- 清理关联资源

### InProcessBackend 类

`InProcessBackend` 实现了 `TeammateExecutor` 接口的完整方法集：

- **spawn**: 在当前进程内创建新 agent 实例
- **sendMessage**: 通过内存通道传递消息
- **terminate**: 优雅停止 agent
- **kill**: 强制停止 agent
- **isActive**: 检查 agent 运行状态

---

## Team Management

### TeamFile 结构

```typescript
interface TeamFile {
  members: TeamMember[]    // 团队成员列表
  leaderId: string         // Leader 的 agent ID
  allowedPaths: string[]   // 团队共享的允许路径
  hiddenPanes: string[]    // 当前隐藏的面板 ID 列表
}
```

### 名称清理

- **sanitizeName(name)**: 清理通用名称字符串，移除非法字符
- **sanitizeAgentName(name)**: 专门清理 agent 名称，确保符合命名约束

### 文件操作

- **readTeamFile()**: 同步读取 team file
- **writeTeamFileAsync(data)**: 异步写入 team file（确保原子性）
- **removeTeammateFromTeamFile(id)**: 从 team file 中移除指定 teammate

---

## Tmux Backend

### Inside Tmux 模式

当检测到已在 tmux 会话中运行：

- 拆分当前窗口为两个区域
- **Leader 区域占 30%**，位于左侧
- **Teammates 区域占 70%**，位于右侧
- Teammate 面板在右侧区域内进一步拆分

### Outside Tmux 模式

当需要启动外部 tmux 会话：

- 使用外部 session socket 连接
- 创建独立的 tmux 会话管理 teammate

### 初始化延迟

```typescript
const PANE_SHELL_INIT_DELAY_MS = 200
```

面板创建后等待 200ms 确保 shell 初始化完成。

### 锁机制

采用锁机制确保面板顺序创建：

- 防止并发创建面板导致布局混乱
- 保证每个面板的 shell 初始化完成后再创建下一个

---

## Environment Inheritance

### TEAMMATE_ENV_VARS

定义了 17 个必须转发给 teammate 进程的关键环境变量，确保 teammate 继承 leader 的运行环境配置。

### buildInheritedCliFlags()

构建传递给 teammate CLI 的参数标志：

- **permission mode**: 权限模式配置
- **model**: 使用的模型标识
- **settings**: 配置文件路径
- **plugin-dir**: 插件目录
- **teammate-mode**: 标记为 teammate 模式运行
- **chrome flags**: Chrome/浏览器相关标志

### buildInheritedEnvVars()

构建传递给 teammate 的环境变量集：

- **CLAUDECODE=1**: 标记运行在 Claude Code 环境中
- **API provider vars**: API 提供商相关变量（密钥、端点等）
- **proxy config**: 代理配置
- **CA certs**: CA 证书路径配置

---

## Teammate Initialization

### initializeTeammateHooks()

注册 teammate 初始化时的生命周期钩子：

- 注册 **Stop hook**: teammate 停止时通知 leader
- Leader 收到停止通知后可以重新分配任务或清理资源

### 权限规则应用

- 读取 team file 中的 `allowedPaths` 配置
- 将团队范围的允许路径作为权限规则应用到 teammate
- 确保 teammate 只能访问被授权的文件路径

### 空闲通知

- Teammate 完成任务后发送 idle 通知
- 通知包含任务执行摘要（summary）
- 通过 mailbox 机制传递给 leader
- Leader 根据 idle 通知决定是否分配新任务

---

## 工程实践指南

### 创建 Agent Swarm

1. **通过 TeamCreateTool 创建团队**: 定义团队名称和成员配置
2. **选择后端**:
   - **tmux**: CI/SSH/无头环境的首选——最可靠,跨平台 (需预安装 tmux)
   - **iTerm2**: 可视化调试场景——开发者可以直观看到每个 agent 的终端输出,仅 macOS + 需要 `it2` CLI
   - **in-process**: 测试和轻量场景——无子进程开销,共享内存但逻辑隔离,每个 teammate 有独立 AbortController
3. **分配任务**: 为每个 teammate 分配具体任务描述和工具权限
4. **后端自动检测**: 如果不手动指定,系统按优先级自动选择: 已在 tmux 中 > iTerm2 可用 > fallback tmux > error

### 调试权限同步

1. **检查文件级权限流**:
   - `~/.claude/teams/{teamName}/permissions/` 目录下查看权限文件
   - `pending` 文件: teammate 的待处理权限请求
   - `resolved` 文件: leader 的权限决议结果
2. **检查邮箱级权限流**:
   - 如果文件级权限流不工作,检查 mailbox 机制是否正常
   - `sendPermissionRequestViaMailbox()` 是否成功发送?
   - `sendPermissionResponseViaMailbox()` 是否成功回传?
3. **检查 Leader Bridge**:
   - `getLeaderToolUseConfirmQueue()` 是否返回有效队列?
   - 确认 leader 的确认队列已通过 `registerLeaderToolUseConfirmQueue()` 注册
4. **轮询间隔**: `PERMISSION_POLL_INTERVAL_MS = 500ms`——如果权限响应超过 500ms,teammate 可能已经开始下一轮轮询

### 环境变量继承

17 个 env 变量从 leader 传递到 worker——如果 teammate 行为异常 (如连接到错误的 API 端点),首先检查环境变量继承:

1. 查看 `TEAMMATE_ENV_VARS` 列表确认需要的变量是否在列表中
2. 检查 `buildInheritedEnvVars()` 输出是否包含预期的变量值
3. 关键变量: `CLAUDE_CODE_USE_BEDROCK`、代理配置、CA 证书路径——缺少这些会导致 teammate 请求发送到错误地址
4. CLI 参数通过 `buildInheritedCliFlags()` 传递: permission mode、model、settings、plugin-dir、teammate-mode

### 选择后端的决策树

```
需要在 CI/无头环境运行?
├─ Yes → tmux (确认已安装)
└─ No
   需要可视化调试?
   ├─ Yes → 在 macOS 上? → iTerm2 (确认 it2 CLI 可用)
   │        不在 macOS → tmux
   └─ No
      是测试/轻量场景?
      ├─ Yes → in-process
      └─ No → tmux (最通用)
```

### 常见陷阱

> **tmux 需要预安装**: Swarm 默认 fallback 到 tmux,但如果系统未安装 tmux 会直接报错。在 CI 环境中确保 Docker 镜像包含 tmux。

> **iTerm2 仅 macOS**: `it2` CLI 是 iTerm2 独有的——在非 macOS 系统上选择 iTerm2 后端会失败。后端检测优先级会自动处理这个问题,但手动指定时需注意。

> **文件锁在 NFS 上可能不可靠**: 权限同步依赖文件系统操作,如果团队目录在 NFS 或其他网络文件系统上,文件锁和原子写入的语义可能不保证——这会导致权限竞争条件。建议使用本地文件系统。

> **面板初始化延迟**: tmux 面板创建后需要 `PANE_SHELL_INIT_DELAY_MS = 200ms` 等待 shell 初始化,并发创建面板使用锁机制保证顺序——如果观察到"command not found"错误,可能是初始化延迟不够。

> **in-process 模式的内存共享**: 与 tmux/iTerm2 不同,in-process teammates 与 leader 共享同一 Node.js 进程内存。虽然逻辑隔离 (独立 AbortController),但一个 teammate 的内存泄漏会影响所有 teammates 和 leader。


---

[← 协调器模式](../33-协调器模式/coordinator-mode.md) | [目录](../README.md) | [Computer Use →](../35-Computer-Use/computer-use.md)
