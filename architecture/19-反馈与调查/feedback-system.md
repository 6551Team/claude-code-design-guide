# Feedback & Survey System

Claude Code 内置了一套完整的用户反馈收集、调查问卷和使用提示系统，涵盖从反馈评级到 transcript 共享、记忆调查、压缩后调查、正向反馈命令、Bug 反馈命令以及 Tips 提示系统等多个子模块。

---

## Feedback Survey State Machine

反馈调查组件以有限状态机驱动，管理用户反馈评级的完整生命周期。

### 状态流转

```
'closed' → 'open' → 'thanks' → 'transcript_prompt' → 'submitting' → 'submitted'
```

- **closed**: 初始状态，调查未展示
- **open**: 调查弹出，等待用户输入
- **thanks**: 用户已选择评级，展示感谢信息
- **transcript_prompt**: 询问用户是否愿意共享 transcript
- **submitting**: 正在提交 transcript 数据
- **submitted**: 提交完成

#### 为什么这样设计

反馈收集涉及多步用户交互（评分 → 感谢 → transcript 询问 → 提交 → 完成），状态机确保流程不会跳步或死锁。6 个状态中每一个都有明确的前驱和后继，不允许任意跳转——例如不能从 `closed` 直接到 `submitting`，也不能从 `submitted` 回到 `open`。这种严格的线性流转保证了数据收集的完整性：用户必须先做出评分，系统才能询问是否分享 transcript。状态机模式比 boolean flag 组合更易于维护和调试，因为每个时刻系统只处于一个明确定义的状态。

### 响应类型

```typescript
type FeedbackSurveyResponse = 'dismissed' | 'bad' | 'fine' | 'good';
```

### 数字键输入

通过 `useDebouncedDigitInput` hook 处理数字键输入，debounce 延迟为 **400ms**：

| 按键 | 含义 |
|------|------|
| `0` | Dismiss（关闭不回答） |
| `1` | Bad（体验差） |
| `2` | Fine（一般） |
| `3` | Good（体验好） |

#### 为什么这样设计

数字键在调查场景中承担双重角色：既是评分输入，又是普通文本输入（用户可能正在输入 "1. 第一步..." 这样的编号列表）。400ms 的防抖窗口让系统区分这两种意图——如果用户在 400ms 内继续输入了其他字符，说明数字是文本的一部分，取消提交；如果 400ms 内没有后续输入，才将数字视为评分选择。源码注释明确说明了这一点：*"Short enough to feel instant for intentional presses, long enough to cancel when the user types more characters"*（`useDebouncedDigitInput.ts`）。

### 概率门控

调查的展示概率由动态配置 `tengu_feedback_survey_config` 控制。该配置从远端获取，决定在满足条件的会话中以多大概率向用户弹出调查。

### 冷却会话追踪

系统维护冷却会话计数（cooldown sessions tracking），在用户完成一次调查后，后续若干会话内不再重复弹出，避免过度打扰用户。

### 数据分析埋点

所有调查事件通过统一的 analytics 事件上报：

```
事件名: 'tengu_feedback_survey_event'
类型:
  - appeared  — 调查展示给用户
  - responded — 用户作出了回应
```

---

## Transcript Sharing

当用户完成反馈评级后，系统可能进一步询问是否愿意共享会话 transcript 以帮助改进产品。

### 响应类型

```typescript
type TranscriptShareResponse = 'yes' | 'no' | 'dont_ask_again';
```

- **yes**: 同意共享当前 transcript
- **no**: 本次不共享
- **dont_ask_again**: 永不再询问（持久化偏好）

### 提交流程 (submitTranscriptShare.ts)

1. **收集消息**: 将当前会话中的 normalized messages 和所有 subagent transcripts 一并收集
2. **读取原始 JSONL**: 从磁盘读取原始 JSONL 格式的 transcript 文件，受 `MAX_TRANSCRIPT_READ_BYTES` 大小限制保护，防止读取过大文件导致内存问题
3. **敏感信息脱敏**: 通过 `redactSensitiveInfo()` 函数对 transcript 内容进行脱敏处理，移除可能包含的敏感数据（API keys、tokens、密码等）

   #### 为什么这样设计

   用户分享的对话 transcript 中极易包含 API key、密码、token 等敏感信息——开发者经常在终端中操作凭证。`redactSensitiveInfo()` 在上传前执行本地脱敏，确保敏感数据永远不会离开用户机器。源码中，`submitTranscriptShare.ts` 在最终上传前显式调用 `const content = redactSensitiveInfo(jsonStringify(data))`，这是一道不可跳过的安全关卡。这种"脱敏在先、上传在后"的设计原则贯穿整个反馈系统——`Feedback.tsx` 中对 description、error、stack trace 等所有用户内容都一视同仁地脱敏。

4. **上传**: 通过 HTTP POST 请求发送到:
   ```
   https://api.anthropic.com/api/claude_code_shared_session_transcripts
   ```

### 触发类型

Transcript 共享可由以下场景触发：

| 触发类型 | 说明 |
|----------|------|
| `bad_feedback_survey` | 用户在反馈调查中选择了 "Bad" |
| `good_feedback_survey` | 用户在反馈调查中选择了 "Good" |
| `frustration` | 系统检测到用户可能遇到了挫折 |
| `memory_survey` | 记忆调查流程中触发 |

---

## Memory Survey (useMemorySurvey)

记忆调查是一种针对自动记忆功能的专项调查。

### 触发条件

- 检查当前会话消息中是否存在 **auto-memory file reads**（即 Claude 自动读取了记忆文件）
- 如果检测到记忆文件被读取，以概率 **0.2**（20%）触发调查

#### 为什么这样设计

Memory survey 使用 `SURVEY_PROBABILITY = 0.2` 概率门控（源码 `useMemorySurvey.tsx` 第 21 行），而非每次都弹出调查。这是用户体验与数据收集之间的平衡：20% 的概率意味着平均每 5 次使用记忆功能才触发一次调查，不会让用户感到被反复打扰；同时这个比例足够高，能在合理时间内收集到统计学上有意义的样本量。调查还受 feature gate `tengu_dunwich_bell` 控制，可以远程关闭，避免在已收集到足够数据后继续打扰用户。

### 数据分析埋点

```
事件名: 'tengu_memory_survey_event'
```

记录用户对记忆功能的反馈数据，帮助评估自动记忆系统的实际效用。

---

## Post-Compact Survey (usePostCompactSurvey)

压缩后调查在会话经历了 **conversation compaction**（对话压缩/摘要化）之后触发。

当对话上下文过长触发自动压缩时，系统会在压缩完成后询问用户对压缩效果的评价，收集关于信息丢失、上下文保持质量等方面的反馈。

---

## Good Claude Command (/good-claude)

`/good-claude` 是一个正向反馈快捷命令。

当用户对 Claude 的某次回答感到满意时，可以快速通过该命令发送正向反馈，无需经过完整的调查流程。这为用户提供了一种低摩擦的方式来表达"这次回答很好"的信号。

---

## Feedback Command (/feedback)

`/feedback` 命令提供了完整的反馈提交界面。

### 别名

- `/bug` — 可作为 `/feedback` 的别名使用

### 门控条件

该命令在以下条件下 **不可用**：

- 使用 **Bedrock** 后端时
- 使用 **Vertex** 后端时
- 使用 **Foundry** 后端时
- 用户属于 **ANT**（Anthropic 内部）时
- 组织策略（policy）不允许 `product_feedback` 时

### 渲染

命令触发时渲染 `Feedback` 组件，传入以下参数：

- **abort signal**: 用于取消反馈提交流程
- **messages**: 当前会话消息上下文
- **initial description**: 初始描述文本（如从命令参数传入）

---

## Tips System

Tips 系统在用户等待 Claude 响应时（如 spinner 转动期间）展示有用的提示信息。

### Tip Registry (tipRegistry.ts)

系统注册了 **60+ 条提示**，每条提示具有以下结构：

```typescript
interface Tip {
  id: string;                          // 唯一标识符
  content: () => Promise<string>;      // 异步内容生成函数
  cooldownSessions: number;            // 冷却会话数
  isRelevant: () => Promise<boolean>;  // 异步相关性检查函数
}
```

- **id**: 每条 tip 的唯一标识
- **content**: 异步函数，返回 tip 的展示内容（支持动态生成）
- **cooldownSessions**: 该 tip 展示后需要冷却的会话数，避免重复展示
- **isRelevant**: 异步函数，根据当前上下文判断该 tip 是否相关（例如某些 tip 仅在特定平台或配置下相关）

### 选择算法 (tipScheduler.ts)

使用 **最长未展示时间优先** 策略选择下一条 tip：

- 在所有满足相关性条件且不在冷却期的 tip 中，选择距离上次展示时间最久的那条
- 确保 tip 的展示分布尽可能均匀，避免用户反复看到相同内容

#### 为什么这样设计

系统注册了 60+ 条 tip，如果使用随机选择，某些 tip 可能长期不被展示，而其他 tip 反复出现。`selectTipWithLongestTimeSinceShown()` 函数（`tipScheduler.ts`）按"距上次展示的会话数"降序排序，优先选择最久未展示的 tip。`tipHistory.ts` 中 `getSessionsSinceLastShown()` 返回 `numStartups - lastShown`，对从未展示过的 tip 返回 `Infinity`，确保新 tip 一定会被优先展示。这种确定性调度比随机选择更公平，保证所有 tip 都有展示机会，使用户在持续使用过程中逐步了解产品的全部功能。

### 历史持久化 (tipHistory.ts)

Tip 展示历史通过全局配置持久化存储：

```typescript
// 存储结构: tipId → numStartups
// 记录每条 tip 在多少次启动中被展示过
Record<string, number>
```

该数据存储在全局 config 中，跨会话保持。

### 数据分析埋点

```
事件名: 'tengu_tip_shown'
```

每次 tip 展示时上报，用于分析各 tip 的展示频率和覆盖情况。

### 自定义 Tips

用户可以通过设置 `settings.spinnerTipsOverride` 来提供自定义的 tip 内容，覆盖或补充默认的 tip 列表。

### 插件 Tips

Marketplace 插件可以注册自己的 tips，这些插件 tips 会被纳入统一的 tip 调度系统中，与内置 tips 一同参与选择和展示。

---

## 工程实践指南

### 触发反馈收集

**反馈状态机流程：**

1. 用户按 thumbs down（或使用 `/feedback`、`/bug` 命令）触发反馈流程
2. 状态机流转：`closed → open → thanks → transcript_prompt → submitting → submitted`
3. 每个状态有明确的前驱和后继，不允许跳步（如不能从 `closed` 直接到 `submitting`）
4. 用户可在任何阶段取消（feedback 是可选的）

**关键入口点：**
- `FeedbackSurvey.tsx` — 反馈调查主组件
- `useFeedbackSurvey.tsx` — 反馈调查 hook，管理状态和概率门控
- `submitTranscriptShare.ts` — Transcript 提交流程

### 调试反馈状态机

**排查步骤：**

1. **检查当前状态**：状态机处于哪个阶段（closed/open/thanks/transcript_prompt/submitting/submitted）
2. **检查转换条件**：确认触发转换的条件是否满足
3. **400ms 防抖影响**：`useDebouncedDigitInput` 设置了 400ms 防抖窗口，数字键输入（0=dismiss, 1=bad, 2=fine, 3=good）在此窗口内如果有后续输入会被取消——这是为了区分评分意图和普通文本输入
4. **概率门控**：调查展示由 `tengu_feedback_survey_config` 动态配置控制概率，非每次都触发
5. **冷却期**：完成一次调查后有 cooldown sessions，后续若干会话内不再弹出

**调试 transcript 提交失败：**
- 检查 `MAX_TRANSCRIPT_READ_BYTES` 限制（防止读取过大文件导致内存问题）
- 确认 `redactSensitiveInfo()` 脱敏是否正常执行（API key、密码、token 会被移除）
- 检查网络连接（上传到 `https://api.anthropic.com/api/claude_code_shared_session_transcripts`）

### 自定义调查概率

**Memory Survey：**
- `SURVEY_PROBABILITY = 0.2`（源码 `useMemorySurvey.tsx` 第 21 行）——20% 概率触发
- 受 feature gate `tengu_dunwich_bell` 控制，可远程关闭
- 仅在检测到 auto-memory file reads 时触发

**Post-Compact Survey：**
- `SURVEY_PROBABILITY = 0.2`（源码 `usePostCompactSurvey.tsx` 第 15 行）——20% 概率触发
- 在会话经历 conversation compaction 后触发

**Feedback Survey：**
- 概率由 `tengu_feedback_survey_config` 远端配置控制
- 冷却会话计数防止过度打扰

### 自定义 Tips

- **覆盖 Tips**：设置 `settings.spinnerTipsOverride` 提供自定义 tip 内容
- **插件 Tips**：Marketplace 插件可注册自己的 tips，纳入统一调度
- **调度算法**：最长未展示时间优先（`selectTipWithLongestTimeSinceShown()`），新 tip 返回 `Infinity` 确保优先展示

### 常见陷阱

| 陷阱 | 详情 | 解决方案 |
|------|------|----------|
| Feedback 是可选的 | 用户可以在任何阶段取消反馈流程 | UI 需要优雅处理取消操作，不要假设流程一定完成 |
| Transcript 分享会脱敏处理 | `redactSensitiveInfo()` 在上传前执行本地脱敏，这是不可跳过的安全关卡 | 如果发现脱敏不完整，修复 `redactSensitiveInfo()` 而非绕过它 |
| 400ms 防抖窗口 | 数字键评分需要 400ms 内无后续输入才触发 | 快速连续输入可能导致评分不被识别 |
| `/feedback` 命令有门控条件 | Bedrock/Vertex/Foundry 后端、ANT 内部用户、policy 不允许时不可用 | 不同环境下反馈入口可能不同 |
| Tip 展示历史跨会话保持 | 存储在全局 config 中（`tipId → numStartups` 映射） | 清除 config 会重置 tip 展示历史 |


---

[← 遥测分析](../18-遥测分析/telemetry-system.md) | [目录](../README.md) | [服务层 →](../20-服务层/services-complete.md)
