# Hooks系统架构文档

> Claude Code v2.1.88 Hooks系统完整技术参考

---

## 用户配置的事件钩子 (28种)

通过 `settings.json` 的 `hooks` 字段配置，支持在不同生命周期节点注入自定义逻辑。

### 设计理念

#### 为什么有 28 种事件钩子类型？

源码 `coreSchemas.ts:355-383` 定义了完整的 `HOOK_EVENTS` 数组，包含 28 种事件类型。每种事件对应工具/查询生命周期的一个关键决策点：

- **PreToolUse / PostToolUse / PostToolUseFailure**: 工具执行前/后/失败时拦截——安全审计、参数修改、日志记录、错误恢复
- **Stop / StopFailure / SubagentStart / SubagentStop**: 智能体生命周期控制——记忆提取、任务完成通知、子智能体协调
- **PreCompact / PostCompact**: 上下文压缩前后——允许外部系统保存/恢复关键信息
- **PermissionRequest / PermissionDenied**: 权限事件——自定义审批流程
- **ConfigChange / InstructionsLoaded / CwdChanged / FileChanged**: 环境变化事件——配置热重载、技能更新
- **WorktreeCreate / WorktreeRemove**: Git worktree 生命周期——多分支并行工作流

![Hooks Hot Reload Flow](../diagrams/hooks-hot-reload-flow.svg)

设计目标：让外部系统可以在不修改核心代码的情况下介入 **任何** 关键决策点。每新增一个系统能力（如 worktree、elicitation），就对应新增钩子事件，保持扩展性。

### 工具生命周期钩子

| 钩子名称 | 触发时机 | 返回值/行为 |
|----------|---------|------------|
| **PreToolUse** | 工具执行前 | 可阻止/修改输入，返回 `proceed` / `block` / `modify` |
| **PostToolUse** | 工具执行后 | 可附加反馈/触发后续操作 |

### 会话生命周期钩子

| 钩子名称 | 触发时机 | 用途 |
|----------|---------|------|
| **SessionStart** | 会话启动 | 初始化环境、加载上下文 |
| **SessionEnd** | 会话结束 | 清理资源、保存状态 |
| **UserPromptSubmit** | 用户提示提交前 | 预处理/验证用户输入 |

### 智能体控制钩子

| 钩子名称 | 触发时机 | 返回值/行为 |
|----------|---------|------------|
| **Stop** | 智能体停止信号 | `blockingErrors` → 注入消息重试；`preventContinuation` → 终止 |
| **SubagentStop** | 子智能体终止 | 子智能体执行结束后的处理 |

### 上下文管理钩子

| 钩子名称 | 触发时机 | 用途 |
|----------|---------|------|
| **PreCompact** | 上下文压缩前 | 压缩前的预处理 |
| **PostCompact** | 上下文压缩后 | 压缩后的后处理 |

### 系统事件钩子

| 钩子名称 | 触发时机 | 用途 |
|----------|---------|------|
| **Notification** | 系统通知 | 自定义通知处理 |
| **TeammateIdle** | 队友空闲检测 | 多智能体协作中检测空闲 |
| **TaskCreated** | 任务创建事件 | 任务创建后的自定义处理 |
| **TaskCompleted** | 任务完成事件 | 任务完成后的自定义处理 |

---

## 钩子命令类型 (schemas/hooks.ts)

钩子支持多种命令类型，通过 Schema 定义：

### BashCommandHook
执行 bash 命令，可访问环境变量和上下文信息。

### PromptHook
提示注入类型，将额外提示文本注入到对话流中。

### HttpHook
HTTP 请求类型，支持向外部服务发送 HTTP 请求。

### AgentHook
智能体调用类型，触发另一个智能体执行任务。

### HookMatcherSchema
条件匹配器，基于 `IfConditionSchema` 实现条件判断，决定钩子是否触发。

---

## 钩子执行管线

### PreToolUse 管线
```
runPreToolUseHooks(toolName, input, context) → HookResult
```
- 在工具执行前调用
- 返回 `HookResult` 决定是否继续执行、阻止或修改输入

### PostToolUse 管线
```
runPostToolUseHooks(toolName, input, output, context)
```
- 在工具成功执行后调用
- 可附加反馈信息或触发后续操作

### PostToolUse 失败管线
```
runPostToolUseFailureHooks(toolName, input, error)
```
- 在工具执行失败后调用
- 用于错误处理和恢复

### 权限决策影响
```
resolveHookPermissionDecision()
```
- 钩子可影响权限系统的决策结果
- 实现自定义权限逻辑

### 后采样钩子
```
executePostSamplingHooks()
```
- 在 API 采样完成后执行
- 已注册的钩子包括：
  - **SessionMemory**: 会话记忆管理
  - **extractMemories**: 记忆提取
  - **PromptSuggestion**: 提示建议生成
  - **MagicDocs**: 文档自动处理
  - 其他已注册的后采样处理器

---

## 70+ React Hooks (src/hooks/)

### 设计理念

#### 为什么 70+ React Hooks 而不是传统状态管理？

Claude Code 的 `src/hooks/` 目录包含 80+ 个 hook 文件，每个封装一个独立关注点（输入处理、权限、IDE 集成、语音、多智能体等）。选择 React hooks 而非传统集中式状态管理（如 Redux）有以下原因：

1. **组合性**: React hooks 允许状态和副作用的局部封装——`useVoice` 不需要知道 `useIDEIntegration` 的存在。传统 Redux 需要在集中的 reducer 中定义所有 action，80+ 个 concerns 会导致 reducer 爆炸式增长。

2. **增量扩展**: 新功能只需添加一个新 hook 文件（如 `useSwarmInitialization.ts`、`useTeleportResume.tsx`），不需要修改全局 store 定义。这对快速迭代的 CLI 工具至关重要。

3. **生命周期绑定**: 许多 hook 管理外部资源（文件 watcher、WebSocket 连接、定时器），React hooks 的 cleanup 机制（`useEffect` return）天然适合这类场景。

4. **条件加载**: 通过 feature flag 和 `isEnabled` 检查，hook 可以在不满足条件时完全跳过，不占用运行时资源。

按功能分类的完整列表：

### 输入/导航
| Hook | 用途 |
|------|------|
| `useArrowKeyHistory` | 方向键历史导航 |
| `useHistorySearch` | 历史搜索 |
| `useTypeahead` | 自动补全 (212KB) |
| `useInputBuffer` | 输入缓冲管理 |
| `useTextInput` | 文本输入处理 |
| `usePasteHandler` | 粘贴处理 |
| `useCopyOnSelect` | 选中复制 |
| `useSearchInput` | 搜索输入 |

### 权限/工具
| Hook | 用途 |
|------|------|
| `useCanUseTool` | 工具可用性检查 |
| `useToolPermissionUpdate` | 工具权限更新 |
| `useToolPermissionFeedback` | 工具权限反馈 |

### IDE集成
| Hook | 用途 |
|------|------|
| `useIDEIntegration` | IDE集成主入口 |
| `useIdeSelection` | IDE选区同步 |
| `useIdeAtMentioned` | IDE @提及 |
| `useIdeConnectionStatus` | IDE连接状态 |
| `useDiffInIDE` | IDE差异查看 |

### 语音
| Hook | 用途 |
|------|------|
| `useVoice` | 语音核心 |
| `useVoiceEnabled` | 语音启用状态 |
| `useVoiceIntegration` | 语音集成 |

### 多智能体
| Hook | 用途 |
|------|------|
| `useSwarmInitialization` | Swarm初始化 |
| `useSwarmPermissionPoller` | Swarm权限轮询 |
| `useTeammateViewAutoExit` | 队友视图自动退出 |
| `useMailboxBridge` | 邮箱桥接 |

### 状态/配置
| Hook | 用途 |
|------|------|
| `useMainLoopModel` | 主循环模型管理 |
| `useSettings` | 设置读取 |
| `useSettingsChange` | 设置变更监听 |
| `useDynamicConfig` | 动态配置 |
| `useTerminalSize` | 终端尺寸 |

### 通知/显示
| Hook | 用途 |
|------|------|
| `useNotifyAfterTimeout` | 超时通知 |
| `useUpdateNotification` | 更新通知 |
| `useBlink` | 闪烁效果 |
| `useElapsedTime` | 已用时间 |
| `useMinDisplayTime` | 最小显示时间 |

### API/网络
| Hook | 用途 |
|------|------|
| `useApiKeyVerification` | API密钥验证 |
| `useDirectConnect` | 直连管理 |
| `useSessionToken` | 会话令牌 |

### 插件
| Hook | 用途 |
|------|------|
| `useManagePlugins` | 插件管理 |
| `useLspPluginRecommendation` | LSP插件推荐 |
| `useOfficialMarketplaceNotification` | 官方市场通知 |

### 建议
| Hook | 用途 |
|------|------|
| `usePromptSuggestion` | 提示建议 |
| `useClaudeCodeHintRecommendation` | Claude Code提示推荐 |
| `fileSuggestions` | 文件建议 |
| `unifiedSuggestions` | 统一建议 |

### 任务
| Hook | 用途 |
|------|------|
| `useTaskListWatcher` | 任务列表监控 |
| `useTasksV2` | 任务V2 |
| `useScheduledTasks` | 定时任务 |
| `useBackgroundTaskNavigation` | 后台任务导航 |

### 历史
| Hook | 用途 |
|------|------|
| `useHistorySearch` | 历史搜索 |
| `useAssistantHistory` | 助手历史 |

### 文件
| Hook | 用途 |
|------|------|
| `useFileHistorySnapshotInit` | 文件历史快照初始化 |
| `useClipboardImageHint` | 剪贴板图片提示 |

### 会话
| Hook | 用途 |
|------|------|
| `useSessionBackgrounding` | 会话后台化 |
| `useTeleportResume` | Teleport恢复 |

### 渲染
| Hook | 用途 |
|------|------|
| `useVirtualScroll` | 虚拟滚动 |
| `renderPlaceholder` | 占位渲染 |

### 日志
| Hook | 用途 |
|------|------|
| `useLogMessages` | 日志消息 |
| `useDeferredHookMessages` | 延迟钩子消息 |
| `useDiffData` | 差异数据 |

---

## 权限钩子子系统 (hooks/toolPermission/)

### PermissionContext.ts

`createPermissionContext()` 返回一个冻结的上下文对象，包含以下方法：

#### 决策与日志
- `logDecision`: 记录权限决策
- `persistPermissions`: 持久化权限设置

#### 生命周期控制
- `resolveIfAborted`: 中止时解析
- `cancelAndAbort`: 取消并中止

#### 分类与钩子
- `tryClassifier`: 尝试权限分类器
- `runHooks`: 执行权限相关钩子

#### 构建决策
- `buildAllow`: 构建允许决策
- `buildDeny`: 构建拒绝决策

#### 用户/钩子处理
- `handleUserAllow`: 处理用户允许操作
- `handleHookAllow`: 处理钩子允许操作

#### 队列管理
- `pushToQueue`: 推入权限请求队列
- `removeFromQueue`: 从队列移除
- `updateQueueItem`: 更新队列项

### createPermissionQueueOps()
创建 React state-backed 的权限请求队列操作，提供基于 React 状态的队列管理。

### createResolveOnce\<T\>()
原子 resolve 跟踪机制，防止竞态条件。确保每个权限请求只被解析一次。

### handlers/ 目录
包含各工具的权限请求处理器，每个工具类型有对应的处理逻辑。

---

## 工程实践指南

### 创建自定义事件钩子

在 `settings.json` 的 `hooks` 字段定义钩子配置：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "type": "bash",
        "command": "echo 'Tool: $TOOL_NAME, Input: $TOOL_INPUT' >> /tmp/tool-audit.log",
        "if": { "tool": "Bash" }
      }
    ],
    "Stop": [
      {
        "type": "bash",
        "command": "echo '任务完成' | notify-send 'Claude Code'"
      }
    ],
    "PostToolUse": [
      {
        "type": "prompt",
        "prompt": "请检查上一步操作的结果是否符合预期"
      }
    ]
  }
}
```

**步骤清单：**
1. 选择合适的事件类型（28 种可选，定义在 `coreSchemas.ts:355-383` 的 `HOOK_EVENTS` 数组）
2. 选择命令类型：`bash`（Shell 命令）、`prompt`（提示注入）、`http`（HTTP 请求）、`agent`（智能体调用）
3. 使用 `if` 条件（`HookMatcherSchema` / `IfConditionSchema`）控制触发时机
4. 测试钩子执行：修改 settings 后立即生效（热重载），观察钩子是否正确触发

### 调试钩子执行

1. **检查钩子超时**：钩子执行超过 500ms 会在 UI 中显示计时器（`HookProgress` 组件）。如果钩子频繁超时，需要优化命令执行速度。
2. **检查返回值格式**：
   - `PreToolUse` 钩子返回 `HookResult`，可以是 `proceed`（继续）、`block`（阻止）、`modify`（修改输入）
   - `Stop` 钩子返回 `blockingErrors` 时注入消息触发重试，`preventContinuation` 时终止
3. **检查环境变量**：Bash 类型钩子可访问预设环境变量（`$TOOL_NAME`、`$TOOL_INPUT` 等）
4. **查看 Bootstrap State**：`state.registeredHooks` 包含所有已注册的钩子列表，可用于确认钩子是否正确加载

### 创建新的 React Hook

在 `src/hooks/` 目录下创建新的 React Hook：

**步骤清单：**
1. 创建 `useXxx.ts`（或 `.tsx`）文件
2. 从合适的 Provider 消费 context（如 `AppStoreContext`、`ModalContext`、`NotificationsContext`）
3. 使用 `useSyncExternalStore` 订阅外部 store（而非直接 `useContext`）以获得更好的性能
4. 注意 cleanup 逻辑——在 `useEffect` 的 return 中释放外部资源（文件 watcher、WebSocket、定时器）
5. 如需条件启用，通过 `feature()` 检查或 `isEnabled` 标志控制

**示例模板：**
```typescript
import { useEffect, useCallback } from 'react'
import { useAppState } from '../state/AppState.js'

export function useMyFeature() {
  const [state, setState] = useAppState(s => s.myField)

  useEffect(() => {
    // 初始化逻辑
    const cleanup = setupSomething()
    return () => {
      // 必须清理！
      cleanup()
    }
  }, []) // 依赖数组必须正确

  return { state, doSomething: useCallback(() => { /* ... */ }, []) }
}
```

### 常见陷阱

> **事件钩子是同步阻塞的**
> 钩子在查询循环中同步执行。长时间运行的钩子命令会阻塞整个查询循环——模型无法继续响应直到钩子完成。如果必须执行耗时操作，在钩子中使用后台进程（`command &`）并立即返回。

> **React Hook 的依赖数组必须正确**
> 错误的依赖数组会导致：
> - 缺失依赖 → 闭包捕获过期值，产生 stale data bug
> - 多余依赖 → 不必要的重执行，影响性能
> - 源码中多处 TODO 注释（如 `useBackgroundTaskNavigation.ts:245`）标注了正在进行的 `onKeyDown-migration`，修改相关 hook 时需关注迁移状态

> **权限钩子的 resolveOnce 机制**
> `createResolveOnce<T>()` 确保每个权限请求只被解析一次，防止竞态条件。如果在权限处理流程中引入异步逻辑，务必通过 resolveOnce 保护，避免多次 resolve 导致不可预测行为。

> **NOTE(keybindings) 标注的 escape handler**
> 源码中 `useTextInput.ts:122` 和 `useVimInput.ts:189` 的 escape handler 被明确标注为"intentionally NOT migrated to the keybindings system"——这些是刻意不迁移的，修改时不要将其移入 keybindings 系统。


---

[← MCP 集成](../08-MCP集成/mcp-integration.md) | [目录](../README.md) | [Skills 系统 →](../10-Skills系统/skills-system.md)
