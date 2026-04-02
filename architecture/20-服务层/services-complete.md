# Services Layer - Complete Reference

Claude Code 的服务层（Services Layer）包含 13 个独立服务，各自负责特定的后台功能。本文档对每个服务进行全面描述，包括用途、关键函数签名、阈值/常量、门控条件、错误处理和状态管理。

### 设计理念

为什么有 13 个独立服务而不是合并为更少的模块？因为每个服务有独立的生命周期和资源需求——LSP 需要保持长连接并管理多个子进程，OAuth 需要 token 刷新和 keychain 交互，Analytics 需要异步事件队列，autoDream 需要 PID 锁和跨会话状态。合并会导致不必要的耦合：一个服务的崩溃或重启不应影响其他服务。此外，这些服务的启动时机也不同——某些（如 LSP）在首次打开代码文件时按需启动，某些（如 policyLimits）在应用启动时立即启动，某些（如 autoDream）在会话空闲时才运行。独立服务让每个模块自主管理自己的生命周期。

---

## 1. AgentSummary

### 用途

在后台周期性地对当前会话进行摘要，为 subagent 和上下文压缩提供高质量的对话概要。

### 核心机制

- **30 秒周期**: 每 30 秒通过 forked 子进程执行一次摘要生成
- **缓存共享**: 摘要结果通过缓存系统共享，避免重复计算。subagent 可以直接读取主会话的摘要缓存
- **工具拒绝与缓存键匹配**: 当 subagent 的工具调用模式与已有缓存键匹配时，系统会拒绝重复执行（tool denial for cache key matching），直接返回缓存的摘要结果

### 状态管理

后台定时器管理摘要任务的生命周期，在会话结束时清理定时器和缓存。

---

## 2. MagicDocs

### 用途

自动维护和更新特定格式的 Markdown 文档，支持通过 subagent 自动生成和刷新文档内容。

### 核心机制

- **文件识别**: 通过文件头部的 `# MAGIC DOC` header 标识目标文档
- **模式匹配**: 使用 pattern regex 匹配需要处理的文件路径或内容模式
- **Subagent 生成**: 通过 forked subagent 执行文档生成/更新任务
- **自定义提示词**: 支持从 `~/.claude/magic-docs/prompt.md` 加载自定义提示词模板
- **变量替换**: 文档模板支持变量替换（variable substitution），将运行时信息注入到生成的文档中

### 关键函数

```typescript
// 检测文件是否为 MagicDoc
isMagicDoc(content: string): boolean
// 触发 MagicDoc 更新
updateMagicDoc(filePath: string, context: Context): Promise<void>
```

### 错误处理

Subagent 执行失败时不影响主会话，错误被记录但不传播。

---

## 3. PromptSuggestion

### 用途

预测用户的下一条输入，通过推测执行（speculative execution）提前准备响应，降低感知延迟。

### 门控条件

- **最小对话轮次**: 至少需要 **2 个 assistant turns** 才开始预测（`MIN 2 assistant turns`）
- **未缓存 token 上限**: 父上下文中未缓存的 token 数不超过 `MAX_PARENT_UNCACHED_TOKENS = 10000`

### 拒绝过滤器

预测结果会经过以下过滤器筛选，被过滤掉的预测不会被使用：

| 过滤类别 | 说明 |
|----------|------|
| **done** | 预测内容暗示对话已结束（如 "谢谢"、"好的"） |
| **meta-text** | 预测内容是关于对话本身的元文本 |
| **evaluative** | 预测内容是评价性的（如 "做得好"、"这不对"） |
| **Claude-voice** | 预测内容使用了 Claude 的口吻而非用户口吻 |

### 推测沙箱

预测执行在受限的 **speculation sandbox** 中运行：

- **Copy-on-Write**: 写操作不影响实际状态
- **最大轮次**: 推测执行最多 **20 turns**
- **只读 Bash**: Bash 工具在沙箱中以只读模式运行，不执行任何有副作用的命令

#### 为什么这样设计

PromptSuggestion 的核心理念是"推测执行"——在用户看到结果前就预测下一步操作并提前运行，让交互感觉更流畅。但推测本质上是赌博：预测可能正确也可能错误，因此不能在主循环中做（会增加延迟），必须放到后台并隔离在 Copy-on-Write 沙箱中。源码 `speculation.ts` 将推测状态存储在临时目录（`~/.claude/tmp/speculation/<pid>/<id>`），与主会话完全隔离。当用户实际输入到来时，系统对比预测——匹配则直接复用结果（节省等待时间），不匹配则静默丢弃。`checkReadOnlyConstraints` 的引入确保推测过程中 Bash 工具不会产生真实副作用。

### 状态管理

维护当前活跃的推测执行状态，当用户实际输入到来时：若匹配预测则复用结果，否则丢弃推测状态。

---

## 4. SessionMemory

### 用途

从当前会话中提取关键信息并保存为 Markdown 格式的会话笔记。

### 核心机制

- 从对话内容中提取结构化的 Markdown 会话笔记
- 输出文件: `.session-memory.md`

### 阈值

| 参数 | 说明 |
|------|------|
| `minimumMessageTokensToInit` | 触发首次提取所需的最小消息 token 数 |
| `minimumTokensBetweenUpdate` | 两次更新之间所需的最小增量 token 数 |

### 关键函数

```typescript
// 初始化会话记忆提取
initSessionMemory(messages: Message[]): Promise<void>
// 更新会话记忆
updateSessionMemory(messages: Message[]): Promise<void>
```

### 错误处理

提取失败时静默降级，不影响主会话流程。

---

## 5. autoDream

### 用途

后台记忆整合服务，定期将零散的记忆片段整合为结构化的长期记忆。

### 门控顺序（Gate Order）

autoDream 的执行需依次通过以下门控检查：

1. **时间门控（Time）**: 距离上次整合是否已过足够时间
2. **会话扫描门控（Session Scan）**: 当前会话中是否有足够新的记忆片段需要整合
3. **锁门控（Lock）**: 基于 PID 的分布式锁，确保同一时刻只有一个 autoDream 实例运行

### PID 锁机制

- 使用 **PID-based lock** 防止并发执行
- 锁超时（staleness）: **60 分钟** — 如果锁持有进程在 60 分钟内未释放锁，视为过期锁，可被新进程接管

### 4 阶段整合提示词

整合过程使用 **4-phase consolidation prompt**，依次执行：

1. 回顾现有记忆结构
2. 识别新增记忆片段
3. 合并与去重
4. 生成整合后的记忆文档

#### 为什么这样设计

autoDream 是"会话间"任务——在用户不活跃时利用空闲 API 配额做有价值的工作（整理记忆、合并去重），但不干扰活跃会话。源码注释清晰描述了其门控顺序：*"Gate order (cheapest first): 1. Time 2. Sessions 3. Lock"*（`autoDream.ts`）——先做廉价的时间检查，再扫描会话数量，最后才获取 PID 锁。这种分层门控避免了不必要的资源消耗。PID 锁的 60 分钟超时防止崩溃进程永久持有锁。`SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000` 的扫描节流进一步避免时间门控通过但会话门控未通过时的无效重复扫描。

### 状态管理

通过文件系统持久化锁状态和最后整合时间戳。

---

## 6. extractMemories

### 用途

后台从对话中提取记忆片段，异步写入记忆存储。

### 核心机制

- **后台提取**: 在不影响主会话的情况下异步运行
- **合并机制（Coalescing）**: 使用 pending stash 暂存待处理的记忆片段，批量合并后写入
- **节流控制**: 通过 `tengu_bramble_lintel` 动态配置控制提取频率
- **互斥写入**: 与主 agent 的记忆写入操作互斥（mutual exclusion），防止并发写入导致数据冲突

### 4 类记忆分类法

| 类型 | 说明 |
|------|------|
| 类型 1 | 用户偏好和习惯 |
| 类型 2 | 项目上下文和技术栈 |
| 类型 3 | 工作流和流程 |
| 类型 4 | 事实和知识 |

### 关键函数

```typescript
// 触发记忆提取
extractMemories(messages: Message[], context: Context): Promise<void>
// 刷新 pending stash
flushPendingMemories(): Promise<void>
```

### 错误处理

提取失败时将待处理片段保留在 stash 中，等待下次重试。

---

## 7. LSP (Language Server Protocol)

### 用途

提供 Language Server Protocol 集成，为代码智能功能（自动补全、诊断、跳转等）提供支持。

### LSPClient

JSON-RPC 协议封装层，负责与 LSP Server 之间的消息序列化/反序列化和请求-响应匹配。

### LSPServerManager

多实例路由管理器，根据文件扩展名将请求路由到对应的 LSP Server 实例：

```typescript
// 根据扩展名获取对应的 LSP Server
getServerForExtension(ext: string): LSPServer
```

### 关键参数

| 参数 | 值 | 说明 |
|------|-----|------|
| 初始化超时 | **30 秒** | LSP Server 启动的最大等待时间 |

### 文件追踪

通过 `openedFiles` Map 追踪已打开的文件：

```typescript
// 已打开文件的追踪表
openedFiles: Map<string, TextDocument>
```

当文件被打开、修改或关闭时，同步通知对应的 LSP Server。

### 错误处理

LSP Server 启动超时或崩溃时，相关功能静默降级，不影响基本编辑能力。

---

## 8. OAuth

### 用途

处理 OAuth 认证流程，支持用户通过 Anthropic 账户登录 Claude Code。

### 核心机制

- **PKCE 流程**: 使用 Proof Key for Code Exchange (PKCE) 增强安全性
- **双路径认证**:
  - **Manual path**: 用户手动复制授权 URL 到浏览器并粘贴回授权码
  - **Automatic path**: 自动打开浏览器并通过本地回调服务器接收授权码

### AuthCodeListener

本地 localhost HTTP 服务器，监听 OAuth 回调：

```typescript
// 启动本地回调服务器
startAuthCodeListener(port: number): Promise<AuthCode>
```

### Profile Fetch

认证成功后获取用户 profile 信息：

- **subscription**: 订阅状态和计划
- **rateLimitTier**: 速率限制层级

### 错误处理

- 回调服务器端口占用时自动尝试其他端口
- 认证超时后清理临时服务器资源
- Token 刷新失败时回退到重新登录流程

---

## 9. Plugins

### 用途

管理插件的作用域、发现和配置。

### 作用域管理

插件按以下作用域层级管理：

| 作用域 | 说明 |
|--------|------|
| `user` | 用户级插件，全局生效 |
| `project` | 项目级插件，仅在特定项目中生效 |
| `local` | 本地开发插件 |
| `managed` | 组织管理的插件 |

### 关键函数

```typescript
// 在设置中查找插件
findPluginInSettings(pluginId: string, settings: Settings): Plugin | undefined
```

### V2 数据回退

支持 V2 格式的插件数据，当新格式不可用时自动回退到 V2 数据格式（V2 data fallback），确保向后兼容。

### 状态管理

插件状态存储在用户和项目级配置中，支持热重载。

---

## 10. policyLimits

### 用途

实施组织级别的策略限制，控制用户可以执行的操作范围。

### 核心机制

- **组织级限制**: 从组织管理端获取策略配置
- **ETag HTTP 缓存**: 使用 HTTP ETag 机制缓存策略数据，减少不必要的网络请求
- **轮询周期**: 每 **1 小时** 在后台轮询一次策略更新

### 失败策略

采用 **fail-open** 策略：当策略服务不可达时默认允许操作，确保不会因策略服务故障而阻断用户工作。

**例外**: `ESSENTIAL_TRAFFIC_DENY_ON_MISS` 模式（用于 **HIPAA** 合规场景）— 当策略不可达时 **拒绝** 操作，确保在合规要求严格的环境中不会放行未经授权的操作。

### 关键函数

```typescript
// 检查操作是否被策略允许
checkPolicyLimit(action: string, context: PolicyContext): PolicyResult
// 刷新策略缓存
refreshPolicyLimits(): Promise<void>
```

### 错误处理

- 网络错误: fail-open（HIPAA 模式除外）
- 解析错误: 使用上次有效的策略缓存
- ETag 匹配: 返回 304 时直接使用本地缓存

---

## 11. remoteManagedSettings

### 用途

管理由组织远程下发的配置，确保组织策略在客户端得到执行。

### 核心机制

- **组织级设置**: 从远程服务拉取组织管理员配置的设置项
- **安全检查**: `checkManagedSettingsSecurity()` 函数检测危险的设置变更。当检测到潜在危险变更（如禁用安全功能、修改关键路径等）时，向用户展示确认提示（dangerous change prompt）
- **后台轮询**: 每 **1 小时** 在后台轮询一次设置更新

### 关键函数

```typescript
// 检查远程设置变更的安全性
checkManagedSettingsSecurity(
  oldSettings: ManagedSettings,
  newSettings: ManagedSettings
): SecurityCheckResult

// 获取当前远程管理设置
getManagedSettings(): Promise<ManagedSettings>
```

### 错误处理

远程设置获取失败时使用本地缓存的上次有效设置，并在后台持续重试。

---

## 12. settingsSync

### 用途

在多个设备间双向同步用户设置和记忆数据。

### 核心机制

- **双向 Push/Pull**: 支持上传（push）和下载（pull）两个方向的同步
- **同步范围**: `SYNC_KEYS` 定义了需要同步的数据类别：
  - **Settings**: 用户设置
  - **Memory**: 记忆数据
  - **Project-keyed data**: 按项目区分的数据，使用 **git hash** 作为项目标识键

### 限制

| 参数 | 值 | 说明 |
|------|-----|------|
| 最大上传大小 | **500KB** | 单次同步上传的数据量上限 |

### 增量上传

支持 **incremental upload**，仅上传自上次同步以来发生变更的数据，减少网络传输量。

### 关键函数

```typescript
// 推送本地设置到远端
pushSettings(keys: SyncKey[]): Promise<void>
// 从远端拉取设置
pullSettings(keys: SyncKey[]): Promise<void>
// 执行双向同步
syncSettings(): Promise<SyncResult>
```

### 错误处理

- 冲突解决: 以远端数据为准（pull wins）
- 超出大小限制时拆分上传或截断
- 网络失败时暂存变更，等待下次同步

---

## 13. teamMemorySync / secretScanner

### 用途

团队记忆同步服务，内置密钥扫描器防止敏感信息泄露到共享记忆中。

### Secret Scanner

#### 规则集

内置 **30 条 gitleaks 规则**，覆盖以下密钥类型（源自公开的 [gitleaks 配置](https://github.com/gitleaks/gitleaks/blob/master/config/gitleaks.toml)，MIT 许可）：

| 类别 | 包含的密钥类型 |
|------|---------------|
| **AWS** | Access Key ID, Secret Access Key, Session Token 等 |
| **GCP** | Service Account Key, API Key 等 |
| **Azure** | Storage Account Key, Client Secret 等 |
| **GitHub** | Personal Access Token, OAuth Token, App Private Key 等 |
| **Slack** | Bot Token, Webhook URL, App Token 等 |
| **Stripe** | Secret Key, Publishable Key, Webhook Secret 等 |
| **Private Keys** | RSA, DSA, EC, PGP 私钥 |

#### 扫描函数

```typescript
// 扫描文本中的密钥，返回匹配的规则 ID（不返回实际密钥值）
scanForSecrets(text: string): SecretScanResult[]

interface SecretScanResult {
  ruleId: string;    // 匹配的规则 ID
  // 注意：不包含实际密钥值，防止二次泄露
}
```

**安全设计**: `scanForSecrets()` 返回匹配的 **规则 ID** 而非实际密钥值，避免密钥在扫描结果中被二次暴露。

#### 脱敏函数

```typescript
// 对文本中的密钥进行脱敏处理
redactSecrets(text: string): string
```

将检测到的密钥替换为占位符。

### teamMemSecretGuard

```typescript
// 防止将包含密钥的内容写入同步记忆
teamMemSecretGuard(content: string): GuardResult
```

作为写入守卫，在记忆内容被写入团队同步存储之前进行密钥扫描。如果检测到密钥，**阻止写入** 并返回相关的规则 ID 信息。

#### 为什么这样设计

团队记忆在团队成员间同步，必须防止一个人的 secret 泄露给整个团队。30 条 gitleaks 规则覆盖了 AWS、GCP、Azure、GitHub、Slack、Stripe 等主流云服务的密钥格式（源码 `secretScanner.ts` 注释："Rule IDs and regexes sourced directly from the public gitleaks config"）。选择 gitleaks 规则集而非自研，是因为它经过开源社区大规模验证，覆盖面广且误报率低。安全设计上采用 fail-closed 策略——扫描失败时阻止写入而非放行，宁可牺牲功能也不泄露密钥。`scanForSecrets()` 返回规则 ID 而非实际密钥值，避免密钥在日志或遥测中二次暴露。

### 错误处理

- 扫描失败时默认阻止写入（fail-closed），确保安全
- 规则匹配采用确定性算法，不存在概率性误报
- 所有拦截事件记录到遥测系统

---

## 工程实践指南

### 添加新服务

**步骤清单：**

1. **创建服务模块**：在 `src/services/` 下创建新目录和入口文件
2. **定义生命周期**：实现启动（init/start）和停止（shutdown/cleanup）方法
3. **注册服务**：在 `services/` 入口或 `setup.ts` 中注册初始化调用
4. **确定启动时机**：
   - 应用启动时（如 policyLimits）→ 在 `setup.ts` 中直接调用
   - 按需启动（如 LSP）→ 在首次需要时延迟初始化
   - 空闲时启动（如 autoDream）→ 通过门控条件触发
5. **错误处理**：后台服务失败不应影响主功能（静默降级原则）

**关键设计约束**：
- 每个服务有独立的生命周期和资源需求
- 一个服务的崩溃或重启不应影响其他服务
- 服务在 headless/bare 模式下可能行为不同（如 `initSessionMemory()` 在 bare 模式下不执行）

### 服务间通信

**原则：服务通过事件总线或共享状态通信——不要直接互相调用。**

- **事件总线**：通过 `logEvent` 发送分析事件，其他服务可监听
- **共享状态**：通过全局 config 或 app state 共享数据
- **钩子系统**：`postSamplingHook`、`handleStopHooks` 等钩子点让多个服务在同一时机执行
- **避免循环依赖**：源码中 analytics 模块特意设计为"无依赖"，正是为了避免被多个服务依赖时形成环

### 调试服务启动

**排查步骤：**

1. **检查初始化顺序**：某些服务有依赖关系（如 OAuth 在 API client 之前）
2. **检查门控条件**：
   - `feature('TEAMMEM')` — 团队记忆同步
   - `feature('EXTRACT_MEMORIES')` — 记忆提取
   - `isBareMode()` — bare 模式下跳过 LSP 等服务
3. **检查 PID 锁**：autoDream 使用 PID 锁（60 分钟超时），崩溃后可能需要清理
4. **检查后台定时器**：AgentSummary 每 30 秒执行一次，policyLimits/remoteManagedSettings 每 1 小时轮询
5. **检查 Promise.allSettled**：LSP manager 的 shutdown 使用 `Promise.allSettled`，单个服务关闭失败不影响其他

**各服务超时/阈值参考：**

| 服务 | 关键阈值 |
|------|---------|
| AgentSummary | 30 秒周期 |
| LSP | 30 秒初始化超时 |
| autoDream | PID 锁 60 分钟超时，`SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000` |
| policyLimits | 1 小时轮询，fail-open（HIPAA 除外） |
| remoteManagedSettings | 1 小时轮询 |
| settingsSync | 最大上传 500KB |

### 常见陷阱

| 陷阱 | 详情 | 解决方案 |
|------|------|----------|
| 服务在 headless 模式下行为不同 | SessionMemory、MagicDocs 等依赖 UI 的服务在 bare/headless 模式下可能不启动 | 检查 `isBareMode()` 和相关门控 |
| 服务的 API 调用计入费用 | AgentSummary、extractMemories、autoDream 都会发起 API 调用 | 后台服务的 token 消耗和费用会反映在 cost tracker 中 |
| policyLimits fail-open 例外 | HIPAA 合规场景使用 `ESSENTIAL_TRAFFIC_DENY_ON_MISS` 模式 | 此模式下策略不可达时会拒绝操作而非放行 |
| secretScanner fail-closed | 团队记忆写入时 secret scanner 扫描失败会阻止写入 | 宁可牺牲功能也不泄露密钥 |
| settingsSync 冲突解决 | pull wins（远端数据优先） | 本地修改可能被远端覆盖 |


---

[← 反馈与调查](../19-反馈与调查/feedback-system.md) | [目录](../README.md) | [插件系统 →](../21-插件系统/plugin-system.md)
