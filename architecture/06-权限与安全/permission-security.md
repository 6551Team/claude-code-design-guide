# 权限与安全 - Permission & Security

> 源文件: `src/utils/permissions/` (24 文件), `src/hooks/useCanUseTool.tsx`,
> `src/hooks/toolPermission/`, `src/types/permissions.ts`,
> `src/utils/sandbox/`, `src/utils/permissions/yoloClassifier.ts`

---

## 1. 架构概览

权限系统是 Claude Code 的安全核心，控制模型对工具的访问权限。它通过多层决策管线确保安全性和可用性之间的平衡。

```
toolExecution.ts
  └─→ canUseTool() (权限判定入口)
        ├── Step 1: 规则匹配 (allow/deny/ask rules)
        ├── Step 2: 工具特定逻辑 (tool-specific permissions)
        ├── Step 3: 分类器 (auto mode classifier)
        └── Step 4: 用户交互提示 (user prompt)
```

---

## 2. 六种权限模式

### 2.1 模式定义

```typescript
// src/types/permissions.ts
export type ExternalPermissionMode = 'acceptEdits' | 'bypassPermissions' | 'default' | 'dontAsk' | 'plan'
export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
export type PermissionMode = InternalPermissionMode
```

### 2.2 模式详解

| 模式 | 标题 | 符号 | 描述 |
|------|------|------|------|
| `default` | Default | (无) | 标准模式 — 对破坏性操作需要用户确认 |
| `plan` | Plan Mode | (暂停图标) | 计划模式 — 模型只能使用只读工具，不执行修改 |
| `acceptEdits` | Accept Edits | - | 自动接受文件编辑，但 Shell 命令仍需确认 |
| `bypassPermissions` | Bypass Permissions | - | 绕过所有权限检查（危险模式） |
| `dontAsk` | Don't Ask | - | 不提示用户，拒绝需要权限的操作 |
| `auto` | Auto Mode | - | AI 分类器自动判定（需 `TRANSCRIPT_CLASSIFIER` feature flag） |
| `bubble` | (内部) | - | 内部模式 — 权限决策冒泡到父级（子代理使用） |

#### 设计理念：为什么需要6种模式而不是简单的allow/deny？

- **信任梯度**：这 6 种模式代表一个从最保守到最信任的梯度：`plan`（最安全，只读）-> `default`（用户确认）-> `acceptEdits`（信任文件操作）-> `auto`（AI 判断）-> `bypassPermissions`（完全信任）-> `bubble`（子代理委托）。简单的 allow/deny 无法表达"信任文件编辑但不信任 Shell 命令"这种细粒度需求。
- **场景匹配**：不同使用场景需要不同信任级别——安全审计用 `plan`（只看不动），日常开发用 `default`（人在环路），受信任的 CI 管道用 `bypassPermissions`（无人值守），AI 自主开发用 `auto`（分类器代替人判断）。
- **`bubble` 模式的必要性**：子代理（通过 `AgentTool` 创建）不应独立做权限决策。如果子代理自行弹出权限对话框，用户会看到来自不明上下文的权限请求，无法做出明智判断。`bubble` 模式让权限请求冒泡到父级，由能看到完整上下文的人/系统确认。
- **`dontAsk` 的存在意义**：后台代理（`shouldAvoidPermissionPrompts=true`）无法弹出 UI 提示用户，需要一种"不提示、直接拒绝"的模式，避免进程挂起等待永远不会到来的用户输入。

### 2.3 外部 vs 内部模式

- **外部模式** (`EXTERNAL_PERMISSION_MODES`): 用户可通过 UI/CLI/设置配置的 5 种模式
- **内部模式**: 包含 `auto`（需 feature flag）和 `bubble`（仅子代理内部使用）
- `PERMISSION_MODES` = `INTERNAL_PERMISSION_MODES`（运行时校验集）

---

## 3. 权限规则系统

### 3.1 规则来源 (PermissionRuleSource)

```typescript
export type PermissionRuleSource =
  | 'userSettings'      // ~/.claude/settings.json
  | 'projectSettings'   // .claude/settings.json (项目级)
  | 'localSettings'     // .claude/settings.local.json
  | 'flagSettings'      // Feature flag 远程设置
  | 'policySettings'    // 企业策略设置
  | 'cliArg'            // CLI 参数 (--allowedTools)
  | 'command'           // /allowed-tools 等命令
  | 'session'           // 会话内用户决策 ("always allow for this session")
```

#### 设计理念：为什么5个配置源形成层级而不是平等合并？

- **组织治理模型编码为软件**：这不是简单的技术配置系统，而是将企业组织结构映射为规则优先级。`policySettings`（企业策略）> `flagSettings`（CLI 参数/远程设置）> `localSettings`（本地）> `projectSettings`（项目）> `userSettings`（个人）。
- **安全不可妥协原则**：企业 CISO 可以通过 `policySettings` 强制禁止 `rm -rf`，即使开发者个人配置了 allow，企业策略也会覆盖个人偏好。源码 `settings/constants.ts:159-167` 中 `getEnabledSettingSources()` 硬编码了 `result.add('policySettings')` 和 `result.add('flagSettings')`——这两个源永远不能被禁用，即使用户通过环境变量限制了其他配置源。
- **被遮蔽规则检测**：源码中 `shadowedRuleDetection.ts` 专门检测低优先级来源的规则是否被高优先级来源遮蔽（如 project deny "Bash" 被 user allow "Bash(git status)" 遮蔽），并在共享配置文件（projectSettings, policySettings）被遮蔽时发出警告，因为这些文件影响整个团队。

### 3.2 规则行为 (PermissionBehavior)

```typescript
export type PermissionBehavior = 'allow' | 'deny' | 'ask'
```

### 3.3 规则值 (PermissionRuleValue)

```typescript
export type PermissionRuleValue = {
  toolName: string       // 工具名（精确匹配或前缀匹配）
  ruleContent?: string   // 可选的内容匹配条件（如 Bash 命令模式）
}
```

### 3.4 完整规则类型

```typescript
export type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: PermissionBehavior
  ruleValue: PermissionRuleValue
}
```

### 3.5 规则存储

规则按 source 分组存储在 `ToolPermissionContext` 中:

```typescript
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource     // allow 规则集
  alwaysDenyRules: ToolPermissionRulesBySource       // deny 规则集
  alwaysAskRules: ToolPermissionRulesBySource        // ask 规则集
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource  // 被剥离的危险规则
  shouldAvoidPermissionPrompts?: boolean               // 后台代理无 UI
  awaitAutomatedChecksBeforeDialog?: boolean            // 协调器 worker
  prePlanMode?: PermissionMode                          // 计划模式前的模式
}>
```

---

## 4. canUseTool() 决策管线

### 4.1 决策流程

```
canUseTool(tool, input, assistantMessage)
    │
    ├── Step 1a: Blanket deny rules (无 ruleContent 的 deny)
    │     └─→ deny → 返回拒绝
    │
    ├── Step 1b: Allow rules (带 ruleContent 匹配)
    │     └─→ match → 返回允许
    │
    ├── Step 1c: Deny rules (带 ruleContent 匹配)
    │     └─→ match → 返回拒绝
    │
    ├── Step 1d: Ask rules
    │     └─→ match → 进入用户提示
    │
    ├── Step 2: 工具特定权限逻辑
    │     ├── Bash: 沙箱检查 + 命令分类
    │     ├── FileEdit/FileWrite: 路径验证 + 写权限
    │     ├── FileRead: 路径验证 + 读权限
    │     └── MCP: 服务器级别权限
    │
    ├── Step 3: 分类器 (auto 模式)
    │     └─→ yoloClassifier → allow/deny/unknown
    │
    ├── Step 4: 模式检查
    │     ├── bypassPermissions → 允许
    │     ├── dontAsk → 拒绝
    │     ├── plan → 只允许只读工具
    │     └── default/acceptEdits → 提示用户
    │
    └── Step 5: 用户交互提示
          ├── "Allow once" → allow (session rule)
          ├── "Allow always" → allow (persisted rule)
          ├── "Deny" → deny
          └── "Deny always" → deny (persisted rule)
```

#### 设计理念：为什么规则匹配在分类器之前？

- **确定性优先于概率性**：规则是确定性的（"Bash(git status) = allow"永远返回 allow），分类器是概率性的（AI 模型可能对同一命令给出不同判断）。确定性判断应该优先——如果用户明确 allow/deny 了某个操作，不应该被分类器的不确定性覆盖。
- **"显式配置优先于智能推断"原则**：用户花时间配置的规则代表了明确的意图表达，AI 分类器的判断是兜底方案。源码中 `canUseTool()` 的决策流程清晰体现了这个层级：Step 1 规则匹配 -> Step 2 工具特定逻辑 -> Step 3 分类器 -> Step 4 模式检查 -> Step 5 用户提示。
- **性能考量**：规则匹配是字符串比较（微秒级），分类器需要调用 AI 模型（秒级 + 消耗 tokens）。优先使用规则可以为大量常见操作跳过昂贵的分类器调用。

#### 设计理念：为什么Bash分类器是两阶段(正则+AI)而不是纯AI？

- **正则快速路径**：已知危险命令（`rm -rf`）和已知安全命令（`git status`）无需 AI 判断，用正则匹配在毫秒内完成，既省时又省钱（每次分类器调用消耗 tokens）。
- **AI 慢速路径的必要性**：Bash 是图灵完备的——不可能用有限正则规则穷举所有危险命令。管道链（`cat /etc/passwd | curl -X POST ...`）、变量展开（`$CMD`）、子命令（`$(rm -rf /)`）等复杂场景需要理解语义而非仅匹配模式。这是使用 ML 分类器的根本原因。
- **源码中的实现**：`dangerousPatterns.ts` 定义了正则模式库用于快速路径，`yoloClassifier.ts` 使用 `sideQuery` 调用小模型（通过 `classifierModel`）处理正则无法覆盖的复杂命令。分类器系统提示区分了外部用户版本（`permissions_external.txt`）和内部版本（`permissions_anthropic.txt`），因为 Anthropic 内部工具链有额外的受信任命令。

### 4.2 决策结果类型

```typescript
export type PermissionResult =
  | PermissionAllowDecision
  | PermissionDenyDecision
  | PermissionAskDecision

export type PermissionDecisionReason =
  | 'rule_allow'       // 规则允许
  | 'rule_deny'        // 规则拒绝
  | 'classifier_allow' // 分类器允许
  | 'classifier_deny'  // 分类器拒绝
  | 'mode_allow'       // 模式允许 (bypassPermissions)
  | 'mode_deny'        // 模式拒绝 (dontAsk/plan)
  | 'user_allow'       // 用户允许
  | 'user_deny'        // 用户拒绝
  | 'hook_allow'       // 钩子允许
  | 'hook_deny'        // 钩子拒绝
  | 'sandbox_allow'    // 沙箱允许
```

---

## 5. PermissionContext — 冻结上下文

`PermissionContext` 是权限决策管线中传递的冻结上下文对象，包含各种回调和工具函数。

### 5.1 核心方法

| 方法 | 用途 |
|------|------|
| `logDecision(decision)` | 记录权限决策（OTel + analytics） |
| `persistPermissions(updates)` | 持久化权限规则到磁盘 |
| `tryClassifier(tool, input)` | 尝试分类器判定 |
| `runHooks(hookType, params)` | 执行权限相关钩子 |
| `buildAllow(reason)` | 构建允许决策 |
| `buildDeny(reason, message)` | 构建拒绝决策 |

### 5.2 OTel Source 映射

```typescript
// 规则来源 → OTel 日志标签
'session' + allow → 'user_temporary'
'session' + deny  → 'user_reject'
'localSettings'/'userSettings' + allow → 'user_permanent'
'localSettings'/'userSettings' + deny  → 'user_reject'
其他 → 'config'
```

---

## 6. Bash 分类器 (yoloClassifier.ts)

### 6.1 概述

`yoloClassifier.ts` 实现了 Auto 模式的核心分类逻辑，使用 AI 模型判断 Bash 命令是否安全。

### 6.2 两阶段分类

**阶段 1: 正则模式检测 (快速路径)**

基于 `dangerousPatterns.ts` 中的正则模式快速检测明显危险的命令：
- 已知安全命令白名单（git status, ls, cat 等）
- 已知危险命令黑名单（rm -rf, chmod 777 等）

**阶段 2: AI 分类器 (慢速路径)**

```typescript
// 使用 sideQuery 调用小模型进行分类
const result = await sideQuery({
  systemPrompt: BASE_PROMPT + PERMISSIONS_TEMPLATE,
  messages: transcriptContext,
  model: classifierModel,
  // ...
})
```

分类器提示模板:
- `auto_mode_system_prompt.txt` — 基础系统提示
- `permissions_external.txt` — 外部用户权限模板
- `permissions_anthropic.txt` — Anthropic 内部权限模板 (ant-only)

### 6.3 分类结果

```typescript
export type YoloClassifierResult = {
  decision: 'allow' | 'deny' | 'unknown'
  reasoning?: string
  usage?: ClassifierUsage
}
```

### 6.4 缓存与优化

- `getLastClassifierRequests` / `setLastClassifierRequests` — 缓存最近的分类器请求
- 使用 `getCacheControl()` 缓存分类器系统提示
- 分类器时长追踪: `addToTurnClassifierDuration`

---

## 7. 沙箱系统

### 7.1 沙箱配置 Schema (11 字段)

```typescript
// 从沙箱配置推断
type SandboxConfig = {
  enabled: boolean                    // 是否启用沙箱
  type: 'macos-sandbox' | 'linux-namespace' | 'docker'  // 沙箱类型
  allowedDirectories: string[]        // 允许访问的目录
  deniedDirectories: string[]         // 禁止访问的目录
  allowNetwork: boolean               // 是否允许网络
  allowSubprocesses: boolean          // 是否允许子进程
  timeout: number                     // 超时时间 (ms)
  maxMemory: number                   // 最大内存
  maxFileSize: number                 // 最大文件大小
  readOnlyDirectories: string[]       // 只读目录
  environmentVariables: Record<string, string>  // 环境变量
}
```

### 7.2 沙箱执行

```typescript
// utils/sandbox/sandbox-adapter.ts
export class SandboxManager {
  // shouldUseSandbox() — 判断是否应使用沙箱
  // execute() — 在沙箱内执行命令
  // validateViolation() — 检查沙箱违规
}
```

### 7.3 沙箱决策集成

当 `shouldUseSandbox()` 返回 true 时：
1. Bash 命令在沙箱环境中执行
2. 沙箱提供文件系统隔离
3. 违规被检测并报告
4. 权限决策可以是 `sandbox_allow`

---

## 8. 路径验证

### 8.1 路径安全检查 (pathValidation.ts)

```
pathValidation.ts
  ├── 绝对路径 vs 相对路径验证
  ├── 允许目录检查 (CWD + additionalWorkingDirectories)
  ├── 符号链接解析与验证
  └── UNC 路径安全 (Windows)
```

### 8.2 检查规则

1. **绝对路径**: 必须在允许的目录范围内
2. **相对路径**: 解析为绝对路径后检查
3. **符号链接**: 解析到最终目标后验证目标路径
4. **UNC 路径** (Windows `\\server\share`): 特殊安全处理
5. **路径遍历**: 检测 `../` 逃逸

### 8.3 允许的目录

- 当前工作目录 (CWD)
- `additionalWorkingDirectories` (通过 /add-dir 命令添加)
- 系统临时目录（某些操作）
- 用户主目录下的配置文件

---

## 9. 高严重性操作检测

### 9.1 危险模式 (dangerousPatterns.ts)

系统维护一个危险操作模式库，用于快速路径分类：

#### 批量删除
- `rm -rf /`
- `find . -delete`
- `git clean -fdx`

#### 基础设施操作
- `terraform destroy`
- `kubectl delete`
- `docker rm -f`

#### 凭证操作
- `cat ~/.ssh/id_rsa`
- `echo $API_KEY`
- 读取 `.env` 文件

#### Git 强制操作
- `git push --force`
- `git reset --hard`
- `git branch -D`

#### 系统修改
- `chmod 777`
- `chown root`
- `sudo` 操作

### 9.2 Shell 规则匹配 (shellRuleMatching.ts)

对 Bash 命令进行解析和模式匹配：
- 命令解析（shell-quote）
- 管道链检测
- 重定向检测
- 环境变量展开（有限）
- 子命令检测 (`$(...)`, backticks)

---

## 10. 文件系统权限

### 10.1 读权限检查

```typescript
checkReadPermissionForTool(filePath, toolUseContext)
```

- 文件是否在允许的目录内
- 文件是否被 `.gitignore` 排除（某些模式下考虑）
- 文件大小是否在限制内 (`fileReadingLimits`)

### 10.2 写权限检查

```typescript
checkWritePermissionForTool(filePath, toolUseContext)
```

- 文件是否在允许的写入目录内
- 路径验证（绝对路径、符号链接、UNC）
- 文件是否为受保护文件（配置文件、凭证等）

### 10.3 团队记忆密钥保护

对团队记忆文件的特殊保护：
- 防止写入包含敏感信息的团队记忆
- 密钥模式检测（API key patterns, token patterns）
- 拒绝写入可能包含 secret 的内容

---

## 11. 拒绝追踪 (denialTracking.ts)

### 11.1 DenialTrackingState

```typescript
export type DenialTrackingState = {
  consecutiveDenials: number    // 连续拒绝次数
  lastDenialTimestamp: number   // 最后拒绝时间戳
  lastDeniedTool: string        // 最后被拒绝的工具
}
```

### 11.2 用途

- 当连续拒绝次数达到阈值时，回退到用户提示（即使是 auto 模式）
- 防止分类器持续做出错误决策
- 子代理使用 `localDenialTracking`（因为 setAppState 是 no-op）

#### 设计理念：为什么拒绝追踪(denialTracking)存在？

- **AI 分类器的纠错机制**：分类器是概率性的，可能连续做出错误判断（如反复拒绝用户真正需要执行的命令）。源码 `permissions.ts:490-498` 中，当 `consecutiveDenials > 0` 且有工具被成功允许时，调用 `recordSuccess()` 重置拒绝计数。当连续拒绝达到 `DENIAL_LIMITS.maxConsecutive`（源码 `denialTracking.ts:42`）时，系统回退到用户提示，让人来裁决。
- **子代理的独立追踪**：源码 `permissions.ts:553-558` 中注释说明："Use local denial tracking for async subagents (whose setAppState is a no-op), otherwise read from appState as before."子代理的 `setAppState` 是空操作（不能修改父级状态），所以需要 `localDenialTracking` 作为独立的本地状态。`forkedAgent.ts:420-421` 中创建子代理时会初始化 `localDenialTracking`。
- **allow 重置机制**：任何成功的工具使用（无论是规则允许还是分类器允许）都会重置拒绝计数（源码 `permissions.ts:483-500`）。这确保了拒绝追踪只在"连续"拒绝时触发，偶尔的拒绝不会累积。

---

## 12. 权限规则解析与持久化

### 12.1 规则格式

设置文件中的规则格式：

```json
{
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(npm test)",
      "FileRead",
      "mcp__server"
    ],
    "deny": [
      "Bash(rm -rf)",
      "Bash(sudo *)"
    ]
  }
}
```

### 12.2 规则解析

```typescript
// permissionRuleParser.ts
permissionRuleValueFromString("Bash(git status)")
// → { toolName: "Bash", ruleContent: "git status" }

permissionRuleValueFromString("FileRead")
// → { toolName: "FileRead", ruleContent: undefined }

permissionRuleValueFromString("mcp__server")
// → { toolName: "mcp__server", ruleContent: undefined }
```

### 12.3 规则持久化

```typescript
// PermissionUpdate.ts
applyPermissionUpdate(update, settingsPath)
applyPermissionUpdates(updates[], settingsPath)
persistPermissionUpdates(updates[], destination)

type PermissionUpdateDestination = 'user' | 'project' | 'local'
```

### 12.4 被遮蔽规则检测

```typescript
// shadowedRuleDetection.ts
// 检测低优先级来源的规则是否被高优先级来源的规则遮蔽
// 例如: project deny "Bash" 被 user allow "Bash(git status)" 遮蔽
```

---

## 13. 权限决策完整数据流

![Permission Decision Pipeline](../diagrams/permission-decision-pipeline.svg)

---

## 工程实践指南

### 添加新权限规则来源

如果需要引入新的配置来源（例如从新的远程服务加载规则）：

1. **在 `PermissionRuleSource` 类型中添加新值** — 修改 `src/types/permissions.ts` 中的 `PermissionRuleSource` 联合类型
2. **在 `settings/constants.ts` 的 `SETTING_SOURCES` 中注册** — 确保 `getEnabledSettingSources()` 包含新来源
3. **实现加载逻辑** — 在 `utils/settings/` 中实现配置加载函数，决定何时加载（启动时？Trust 之后？按需？）
4. **确定优先级** — 新来源在规则层级中的位置决定了它能否覆盖/被覆盖其他来源的规则
5. **更新被遮蔽规则检测** — `shadowedRuleDetection.ts` 需要知道新来源的优先级关系

**关键约束**：
- `policySettings` 和 `flagSettings` 永远不能被禁用（`getEnabledSettingSources()` 中硬编码了 `result.add()`）
- 新来源如果是共享的（影响整个团队），被遮蔽时应发出警告

### 添加新的权限模式

1. **在 `PermissionMode` 类型中添加** — 修改 `src/types/permissions.ts`
2. **在 `canUseTool` 管线中添加对应分支** — 在 Step 4（模式检查）中处理新模式的默认行为
3. **更新 UI 模式选择器** — 如果是外部可用模式，添加到 `EXTERNAL_PERMISSION_MODES`；如果是内部模式，只添加到 `INTERNAL_PERMISSION_MODES`
4. **添加测试** — 覆盖新模式在各种规则组合下的行为

### 调试权限拒绝

当工具执行被权限系统拒绝时：

1. **开启 `--debug`** — 查看 `canUseTool()` 的完整决策链，包括：
   - 哪个规则匹配了？（`rule_allow` / `rule_deny`）
   - 来自哪个来源？（`userSettings` / `projectSettings` / `policySettings`）
   - 分类器的判断？（`classifier_allow` / `classifier_deny`）
   - 最终决策原因？（`PermissionDecisionReason`）
2. **检查规则优先级** — deny 规则中，blanket deny（无 `ruleContent`）在任何 allow 规则之前检查。如果配置了 `deny: ["Bash"]`（无内容），所有 Bash 命令都会被拒绝，即使有更具体的 allow 规则
3. **检查被遮蔽规则** — 低优先级来源的 allow 规则可能被高优先级来源的 deny 规则遮蔽
4. **OTel 日志** — 权限决策通过 `logDecision()` 记录到 OTel，可以在遥测面板中追踪历史决策模式

### Bash 分类器调试

当 auto 模式对 Bash 命令的分类结果不符合预期时：

1. **禁用 AI 分类器** — 设 `CLAUDE_CODE_DISABLE_BASH_CLASSIFIER=true`，只使用正则快速路径，确认问题是否出在 AI 分类器
2. **检查正则模式** — `dangerousPatterns.ts` 中的正则模式是快速路径的判定依据，确认命令是否命中了某个模式
3. **检查分类器缓存** — `getLastClassifierRequests()` 返回最近的分类器请求和结果
4. **检查分类器提示模板** — `auto_mode_system_prompt.txt` + `permissions_external.txt`（或 `permissions_anthropic.txt`）构成分类器的系统提示

### 添加危险模式检测

在 `dangerousPatterns.ts` 中添加新的正则模式时：

1. **编写正则** — 匹配目标危险命令模式
2. **添加测试用例** — 正面匹配（应该检测到的命令）和负面匹配（不应误报的命令）
3. **考虑变体** — Shell 命令有很多变体写法（短选项/长选项、引号/无引号、管道链等），确保正则覆盖常见变体
4. **注意性能** — 正则匹配在每次 Bash 工具执行时运行，避免过于复杂的正则（如大量回溯）

### 沙箱调试

1. **检查 `shouldUseSandbox()` 返回值** — 确认沙箱是否应该启用
2. **检查沙箱类型** — `macos-sandbox` / `linux-namespace` / `docker`，不同平台使用不同沙箱实现
3. **检查违规检测** — `SandboxManager.validateViolation()` 检测沙箱违规；如果命令在沙箱中失败但在外部成功，可能是沙箱配置中的 `allowedDirectories` 或 `allowNetwork` 限制过严
4. **bypassPermissions 的沙箱保护** — `bypassPermissions` 模式在非沙箱环境且有网络访问时会被 `setup.ts` 拒绝（安全校验步骤 11），这是防止不受限模式在不安全环境中运行的最后防线

### 常见陷阱

1. **`policySettings` 永远不能被禁用** — 测试时不要尝试绕过企业策略设置。`getEnabledSettingSources()` 中 `policySettings` 和 `flagSettings` 是硬编码添加的，无法通过环境变量或配置排除

2. **deny 规则优先于 allow 规则（同一层级同时存在时）** — 如果 `projectSettings` 中同时有 `allow: ["Bash(git *)"]` 和 `deny: ["Bash"]`，blanket deny 会在 allow 匹配之前生效

3. **MCP 工具的权限使用 `mcp__server` 前缀** — 权限规则中匹配 MCP 工具时使用 `mcp__serverName__toolName` 格式，不是工具的原始名称。`filterToolsByDenyRules()` 中对 MCP 工具做了特殊的前缀匹配

4. **auto 模式的分类器结果不应被视为最终决策** — 分类器是概率性的，拒绝追踪（`denialTracking.ts`）会在连续拒绝达到阈值时回退到用户提示。不要假设分类器的 `allow`/`deny` 是不可推翻的

5. **子代理的权限使用 `bubble` 模式** — 子代理（通过 `AgentTool` 创建）不应独立做权限决策。`bubble` 模式让权限请求冒泡到父级。如果在子代理中看到意外的权限行为，检查是否正确设置了 `bubble` 模式


---

[← 工具系统](../05-工具系统/tool-system.md) | [目录](../README.md) | [上下文管理 →](../07-上下文管理/context-management.md)
