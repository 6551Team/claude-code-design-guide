# 代价追踪架构文档

> Claude Code v2.1.88 费用与用量追踪系统完整技术参考

---

## 核心追踪 (cost-tracker.ts)

### 导出函数

#### 汇总查询
| 函数 | 返回值 |
|------|--------|
| `getTotalCost()` | 总费用 (USD) |
| `getTotalDuration()` | 总持续时间 |
| `getTotalAPIDuration()` | API 调用总耗时 |
| `getTotalAPIDurationWithoutRetries()` | API 调用总耗时（不含重试） |

#### Token 统计
| 函数 | 返回值 |
|------|--------|
| `getTotalInputTokens()` | 总输入 token 数 |
| `getTotalOutputTokens()` | 总输出 token 数 |
| `getTotalCacheReadInputTokens()` | 缓存读取输入 token 数 |
| `getTotalCacheCreationInputTokens()` | 缓存创建输入 token 数 |

#### 代码变更统计
| 函数 | 返回值 |
|------|--------|
| `addToTotalLinesChanged()` | 添加代码行变更记录 |
| `getTotalLinesAdded()` | 总新增行数 |
| `getTotalLinesRemoved()` | 总删除行数 |

#### 其他统计
| 函数 | 返回值 |
|------|--------|
| `getTotalWebSearchRequests()` | 网页搜索请求总数 |

#### 模型用量
| 函数 | 返回值 |
|------|--------|
| `getModelUsage()` | 所有模型的用量数据 |
| `getUsageForModel()` | 指定模型的用量数据 |

#### 格式化
| 函数 | 返回值 |
|------|--------|
| `formatCost(cost)` | 格式化美元金额显示 |
| `formatTotalCost()` | 模型用量分解显示（按模型列出） |
| `hasUnknownModelCost()` | 检测是否有未知模型的费用 |

#### 累积与持久化
| 函数 | 用途 |
|------|------|
| `addToTotalSessionCost()` | 按模型累积用量（input/output/cache tokens, cost, duration） |
| `getStoredSessionCosts()` | 从项目配置读取已保存的会话费用 |
| `restoreCostStateForSession()` | 恢复费用状态（仅当 sessionId 匹配时） |
| `saveCurrentSessionCosts()` | 持久化当前会话费用到项目配置 |

### 数据结构

```typescript
StoredCostState = {
  totalCostUSD: number,
  totalAPIDuration: number,
  totalAPIDurationWithoutRetries: number,
  totalToolDuration: number,
  totalLinesAdded: number,
  totalLinesRemoved: number,
  lastDuration: number,
  modelUsage: Record<string, {
    input_tokens: number,
    output_tokens: number,
    cache_read: number,
    cache_creation: number,
    cost: number,
    duration: number
  }>
}
```

### 设计理念：为什么 per-model 累积而不是全局单一计数？

源码 `cost-tracker.ts` 中 `modelUsage` 使用 `Record<string, ModelUsage>` 结构，按模型名称索引存储用量数据。这个设计决策基于：

1. **不同模型价格不同** -- Opus 和 Sonnet 的输入/输出 token 单价差异显著，单一计数无法准确计算费用
2. **缓存 token 的差异化计费** -- cache read 和 cache creation 的价格不同于普通 token，per-model 结构让每种 token 类型都能独立追踪
3. **模型切换场景** -- 一个会话中可能使用多个模型（主模型 + advisor 模型），`addToTotalSessionCost()` 函数（第 278-284 行）每次调用都指定 model 参数，确保费用归入正确模型

### 设计理念：为什么会话费用与总费用分开？

用户有两个不同的关注点：
- **"这次对话花了多少"** -- 会话费用帮助用户评估单次任务的成本效率
- **"总共花了多少"** -- 总费用帮助用户做预算管理和成本规划

源码通过 `saveCurrentSessionCosts()` 持久化会话费用到项目配置，`restoreCostStateForSession()` 在恢复会话时只恢复匹配 sessionId 的费用（避免混淆不同会话的成本数据）。

### 工程实践

**费用不准确时的排查**：
1. 检查模型价格表是否更新 -- `formatModelUsage()` 中通过 `getCanonicalName(model)` 将不同模型 ID 归一化后聚合显示，确认模型映射是否正确
2. 检查 cache token 是否被正确扣减 -- `cacheReadInputTokens` 和 `cacheCreationInputTokens` 分别追踪，缓存命中的 token 价格低于普通 token
3. 检查 `hasUnknownModelCost()` 返回值 -- 如果有未知模型的费用，说明价格表缺少对应条目
4. advisor 模型的费用通过独立的 `addToTotalSessionCost()` 调用累积（第 316 行），确认 advisor 费用是否被正确计入

**添加新模型的费用追踪**：
- 在价格表中添加模型条目（模型名 → 输入/输出/缓存 token 的美元单价）
- 确保 `getCanonicalName()` 能正确映射新模型 ID 到显示名称
- `contextWindow` 和 `maxOutputTokens` 信息通过 `getContextWindowForModel()` 和 `getModelMaxOutputTokens()` 获取，也需要添加对应配置

---

## React Hook (costHook.ts)

### useCostSummary(getFpsMetrics?)

注册进程退出处理器，在会话结束时：
1. 显示费用摘要（仅对有 billing access 的用户显示）
2. 保存会话费用
3. 记录 FPS 指标（如果提供了 `getFpsMetrics` 函数）

---

## /cost 命令

### 条件显示逻辑
- Claude.ai 订阅者：隐藏费用信息（因为包含在订阅中）
- ANT 用户例外：即使是订阅者也显示
- 检测 `currentLimits.isUsingOverage`：显示超额使用消息

### 输出
调用 `formatTotalCost()` 输出按模型分解的费用详情。

---

## /stats 命令

### 实现
渲染 `<Stats>` 组件（从 `components/Stats` 导入），展示使用统计和活动信息。

---

## /usage 命令

### 实现
渲染 `<Settings defaultTab="Usage">` 组件。

### 可用性
仅 claude-ai 订阅者可用。

---

## /extra-usage 命令

### 功能
管理额外用量配额，支持团队/企业管理员授予超额信用额度。

### 前置检查
- 验证团队/企业管理员资格
- 超额信用额度的获取和验证

### 执行流程
- **交互式**: 打开浏览器进行操作
- **非交互式**: 命令行直接处理

### 门控条件
- `isExtraUsageAllowed()`: 检查是否允许额外用量
- `isOverageProvisioningAllowed()`: 检查是否允许超额配置


---

[← 文件持久化](../41-文件持久化/file-persistence.md) | [目录](../README.md) | [Shell 工具链 →](../43-Shell工具链/shell-toolchain.md)
