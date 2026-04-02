# 错误恢复架构文档

> Claude Code v2.1.88 错误恢复系统完整技术参考

---

## 5层恢复层级 (由低成本到高成本)

错误恢复系统采用分层策略，从最低成本方案开始尝试，逐步升级到更昂贵的恢复方式。

![Error Recovery Chain](../diagrams/error-recovery-chain.svg)

---

### 设计理念

#### 为什么5层恢复而不是统一错误处理？

每层处理不同的故障域——上下文折叠排空处理上下文溢出（零成本）、响应式压缩处理提示过长（中等成本，需要摘要）、MaxOutput 恢复处理输出截断（继续生成）、模型降级处理服务过载（切换模型）、用户中断处理不可恢复场景（优雅退出）。统一的 try/catch 无法区分这些故障域的恢复策略和成本差异。分层设计让系统从最便宜的恢复方案开始尝试，逐步升级到更昂贵的方案——大部分错误在低层就被消化了。

#### 为什么tool retry在API retry之前？

工具错误最常见（文件不存在、进程超时、权限不足）且最便宜修复——只需要本地重试，不需要网络往返。API 错误（网络抖动、速率限制、服务过载）需要 HTTP 重试，成本更高且延迟更大。将便宜的恢复放在前面，可以避免不必要的 API 调用浪费。这也符合源码中"由低成本到高成本"的分层原则。

#### 为什么max_output_tokens恢复限3次？

源码 `query.ts:164` 中明确定义 `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`，且上方注释警告：*"rules, ye will be punished with an entire day of debugging and hair pulling"*。超过 3 次说明模型无法在给定空间内完成任务，继续恢复只会产生碎片化输出——每次恢复注入 `"Output token limit hit. Resume directly..."` 元消息，模型需要从断点继续，但上下文中充满了截断标记和恢复指令，输出质量急剧下降。3 次是经验阈值：足够处理正常的长输出（如大文件生成），又不会无限循环。每次新用户轮次会重置计数器。

### Layer 1: 上下文折叠排空

**触发条件**: `prompt-too-long` 保留错误

**动作**:
```
contextCollapse.recoverFromOverflow()
```
排空已暂存的折叠内容，释放上下文空间。

**成本**: 极低 -- 无 API 调用，保留粒度控制

**结果**: 成功 → 重试同一请求 (`continue`)

---

### Layer 2: 响应式压缩

**触发条件**: 折叠排空不足 或 直接 PTL (Prompt Too Long)

**动作**:
```
reactiveCompact.recoverFromPromptTooLong()
```

**处理过程**:
1. 剥离超大媒体内容
2. 摘要压缩废弃消息
3. yield 边界消息

**限制**: 每次循环迭代仅允许一次尝试（`hasAttemptedReactiveCompact` 标志防止重复）

**结果**: 成功 → 重试请求；失败 → 暴露错误给用户

---

### Layer 3: Max Output Tokens 恢复

**首次触发**:
- 使用 `ESCALATED_MAX_TOKENS` (64K)
- 无额外消息注入

**后续触发**:
- 注入元消息: `"Output token limit hit. Resume directly..."`

**计数管理**:
- 使用 `maxOutputTokensRecoveryCount` 计数器
- 最多允许 3 次恢复尝试
- 每次新轮次（用户消息后）重置计数器
- 第 3 次后：暴露保留的错误

---

### Layer 4: 模型降级

**触发条件**: `FallbackTriggeredError` (HTTP 529 过载)

**动作流程**:
1. yield 墓碑消息（标记孤立的 assistantMessages）
2. 清除工具状态（工具块/工具结果）
3. 切换 `currentModel` → `fallbackModel`
4. 清除 `StreamingToolExecutor`，创建新实例
5. 剥离思考签名 (`stripSignatureBlocks`)
6. 重试完整请求 (`continue`)

**日志**: 记录 `tengu_model_fallback_triggered` 事件

---

### Layer 5: 用户中断

**触发条件**: `AbortSignal`（用户按 Ctrl+C）

#### 流式传输期间:
1. 排空 `StreamingToolExecutor`
2. 为已排队但未完成的工具生成合成 `tool_result`（错误类型）
3. 返回 `{ reason: 'aborted_streaming' }`

#### 工具执行期间:
1. 等待当前工具完成或超时
2. 收集已完成的工具结果
3. 返回 `{ reason: 'aborted_tools' }`

---

## 保留策略 (Withholding)

可恢复错误先保留，不立即暴露给 SDK/REPL，给恢复层级机会尝试修复。

### 可保留的错误类型

| 错误类型 | 检测函数 |
|----------|---------|
| prompt-too-long | `reactiveCompact.isWithheldPromptTooLong()` |
| max-output-tokens | `isWithheldMaxOutputTokens()` |
| 媒体大小超限 | `reactiveCompact.isWithheldMediaSizeError()` |

### 保留流程
1. 错误发生时先检查是否可恢复
2. 可恢复 → 保留错误，执行恢复层级
3. 恢复成功 → 错误被消化，继续正常流程
4. 恢复失败 → 暴露原始错误给用户

---

## 错误分类 (errors.ts, 1181行)

### 核心分类函数

#### classifyAPIError()
将 API 错误分类到预定义的错误类别中。

#### parsePromptTooLongTokenCounts()
从错误消息中提取实际 token 数和限制 token 数。

#### getPromptTooLongTokenGap()
计算超出的 token 数量。

#### isMediaSizeError()
检测是否为媒体大小超限错误。

#### getErrorMessageIfRefusal()
检测并提取拒绝消息内容。

### 错误类别

| 类别 | 触发条件 | 说明 |
|------|---------|------|
| **API 错误** | 4xx/5xx | 通用 API 错误 |
| **速率限制** | 429 | 请求频率过高 |
| **容量不足** | 529 | 服务过载 |
| **连接错误** | timeout / ECONNRESET | 网络连接问题 |
| **认证错误** | 401 | 认证失败/过期 |
| **验证错误** | - | 请求参数验证失败 |
| **内容策略** | - | 内容审核拒绝 |
| **媒体大小** | - | 媒体文件超限 |
| **提示过长** | - | 上下文超出模型限制 |

---

## 工程实践指南

### 调试错误恢复链

**确定错误发生在哪一层，然后检查对应层的恢复逻辑：**

1. **定位错误层级**：
   - **Tool 层**：工具执行失败（文件不存在、进程超时、权限不足）→ 检查工具的本地重试逻辑
   - **API 层**：HTTP 4xx/5xx 错误 → 检查 `withRetry.ts` 中的重试策略和 backoff 配置
   - **Context 层**：prompt-too-long → 检查 Layer 1（上下文折叠）和 Layer 2（响应式压缩）
   - **Session 层**：用户中断/不可恢复 → 检查 Layer 5 的 abort 处理

2. **检查关键状态标志**（`query.ts`）：
   ```
   hasAttemptedReactiveCompact  — 是否已尝试响应式压缩（每次迭代仅允许一次）
   maxOutputTokensRecoveryCount — 当前 max_output_tokens 恢复次数（上限 3）
   ```

3. **查看恢复决策路径**：
   ```
   错误发生
     → classifyAPIError() 分类
     → isWithheldPromptTooLong() / isWithheldMaxOutputTokens() / isWithheldMediaSizeError() 判断是否可保留
     → 可保留 → 进入恢复层级
     → 不可保留 → 直接暴露给用户
   ```

### 自定义错误处理

- **工具执行失败注入**：通过插件钩子系统在 tool 执行失败时注入自定义逻辑
- **API 错误拦截**：`withRetry.ts` 中的重试逻辑支持自定义 backoff 策略
- **模型降级配置**：可配置 `fallbackModel` 作为 529 过载时的降级目标

### 测试恢复机制

**模拟 413 错误测试 reactive compact：**
1. 构造一个超长上下文（大量大文件读取）
2. 观察系统是否触发 `reactiveCompact.recoverFromPromptTooLong()`
3. 检查是否正确执行：剥离超大媒体 → 摘要压缩废弃消息 → yield 边界消息

**模拟 429 测试 retry backoff：**
1. 短时间内发送大量请求触发速率限制
2. 观察 `withRetry.ts` 中的退避策略是否正确执行
3. 注意源码中 `withRetry.ts:94` 的 TODO 注释：keep-alive 通过 `SystemAPIErrorMessage` yield 是一种 stopgap（临时方案）

**模拟 529 测试模型降级：**
1. 当主模型返回 `FallbackTriggeredError` 时
2. 检查是否正确执行：yield 墓碑消息 → 清除工具状态 → 切换模型 → 剥离思考签名 → 重试

### 常见陷阱

| 陷阱 | 详情 | 注意事项 |
|------|------|----------|
| max_output_tokens 恢复最多 3 次 | `query.ts:164` 定义 `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`，源码上方注释警告违反此规则的后果 | 每次新用户轮次重置计数器；超过 3 次说明模型无法在给定空间内完成任务 |
| reactive compact 每次循环迭代仅 1 次 | `hasAttemptedReactiveCompact` 标志防止重复，避免无限压缩循环 | 如果一次压缩不够，错误会被暴露给用户而非继续重试 |
| 保留错误可能被吞没 | 可恢复错误先保留不暴露，但恢复失败时必须暴露原始错误 | 调试时注意区分"错误被成功恢复"和"错误被静默吞没" |
| 模型降级后的状态清理 | 降级时需要清除 `StreamingToolExecutor`、剥离思考签名 | 如果降级后输出异常，检查 `stripSignatureBlocks` 是否正确执行 |
| abort 信号处理 | 流式传输期间中断需要为未完成工具生成合成 `tool_result` | 中断后检查是否有遗留的工具状态未被清理 |


---

[← 记忆系统](../16-记忆系统/memory-system.md) | [目录](../README.md) | [遥测分析 →](../18-遥测分析/telemetry-system.md)
