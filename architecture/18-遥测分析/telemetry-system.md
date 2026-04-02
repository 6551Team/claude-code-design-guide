# Telemetry & Analytics System

Claude Code 的遥测分析系统涵盖事件分析（Analytics）、OpenTelemetry 集成、会话追踪（Session Tracing）、插件遥测（Plugin Telemetry）、费用跟踪（Cost Tracking）以及网关检测（Gateway Detection）等多个子系统。

---

## Analytics Architecture

### 无依赖设计 (events.ts)

Analytics 系统采用 **无依赖设计**（no-dependency design）：事件在 `events.ts` 中产生后被放入队列，直到有 sink 被附加（attach）时才真正发送。这保证了在初始化阶段或 sink 未就绪时事件不会丢失。

### AnalyticsSink 接口

```typescript
interface AnalyticsSink {
  logEvent: (event: AnalyticsEvent) => void;        // 同步事件记录
  logEventAsync: (event: AnalyticsEvent) => Promise<void>;  // 异步事件记录
}
```

### Proto 字段剥离

`stripProtoFields()` 函数负责移除事件对象中所有以 `_PROTO_*` 为前缀的键。这些字段仅供内部使用，在发送到通用访问的后端（general-access backends）时需要被剥离。

### 设计理念

#### 为什么Datadog+1P双路由？

源码 `analytics/index.ts` 注释说明：*"DESIGN: This module has NO dependencies to avoid import cycles. Events are queued until attachAnalyticsSink() is called."* `sinkKillswitch.ts` 定义了 `SinkName = 'datadog' | 'firstParty'` 两个独立通道。Datadog 提供实时监控和告警（运维视角——P0 事件立即通知），1P（第一方 Anthropic 事件日志）提供长期分析和产品决策（产品视角——用户行为趋势、功能采用率）。两者的数据需求、访问权限、保留策略完全不同。`_PROTO_*` 字段仅发送给 1P，对 Datadog 等通用后端会被 `stripProtoFields()` 剥离——这是数据安全分层。

#### 为什么遥测系统是"无依赖队列"设计？

源码 `analytics/index.ts` 开头明确声明：*"DESIGN: This module has NO dependencies to avoid import cycles. Events are queued until attachAnalyticsSink() is called during app initialization."* 遥测不应影响主功能——如果遥测服务挂了，用户的代码编辑不应受影响。队列设计确保：(1) 启动阶段 sink 未就绪时事件不丢失；(2) 发送失败静默处理（fail-open 原则：`sinkKillswitch` 缺失/异常时默认保持 sink 开启）；(3) 无 import 依赖避免循环引用——遥测模块被几乎所有模块使用，如果它反过来依赖其他模块，很快就会形成环。

#### 为什么GrowthBook而不是自建feature flag？

GrowthBook 是行业标准的 A/B 测试平台，支持灰度发布和受控实验。源码中大量使用 `getFeatureValue_CACHED_MAY_BE_STALE()` 和 GrowthBook 门控（如 `tengu_log_datadog_events`、`enhanced_telemetry_beta`）。自建 feature flag 系统会重复造轮子：需要自己实现用户分群、渐进式发布、实验分析、紧急回滚。GrowthBook 还提供 session-stable 的特性评估（源码中 `promptCache1hAllowlist` 注释：*"Cached prompt cache 1h TTL allowlist from GrowthBook (session-stable)"*），确保同一会话中特性开关不会中途翻转。

### 双通道路由

事件通过双通道路由机制同时发送到两个后端：

1. **Datadog** — 第三方监控和分析平台
2. **1P Event Logging** — Anthropic 第一方事件日志系统

### Datadog 门控

`shouldTrackDatadog()` 函数控制是否向 Datadog 发送事件：

- **Killswitch**: 全局开关，可紧急关闭 Datadog 上报
- **GrowthBook 特性门控**: 通过 `tengu_log_datadog_events` feature gate 控制

### 采样逻辑

`shouldSampleEvent()` 函数实现事件采样，对高频事件进行降采样以控制数据量和成本。

### Sink Killswitch

支持按 sink 粒度的独立关闭：

```typescript
interface SinkKillswitch {
  datadog?: boolean;     // 关闭 Datadog sink
  firstParty?: boolean;  // 关闭第一方 sink
}
```

设计原则为 **fail-open**：当 killswitch 配置缺失或读取失败时，默认保持 sink 开启，确保遥测数据不会因配置问题而静默丢失。

---

## OpenTelemetry Stack

### 初始化流程

#### bootstrapTelemetry()

引导阶段函数，负责设置 OTEL 相关环境变量。在 OpenTelemetry SDK 初始化之前调用，确保所有必要的环境配置就绪。

#### initializeTelemetry()

核心初始化函数，创建以下三个 Provider：

- **MeterProvider** — 指标（Metrics）提供者
- **LoggerProvider** — 日志提供者
- **TracerProvider** — 追踪（Traces）提供者

### Exporter 类型

支持多种导出器类型：

| Exporter | 传输协议 | 说明 |
|----------|----------|------|
| `console` | 标准输出 | 开发调试用，输出到控制台 |
| `otlp` (gRPC) | gRPC | 高性能二进制传输 |
| `otlp` (HTTP) | HTTP/JSON | HTTP 传输 |
| `otlp` (Protobuf) | HTTP/Protobuf | HTTP + Protobuf 编码 |
| `prometheus` | HTTP pull | Prometheus 兼容的 pull 模式 |

### BigQuery Metrics

针对 API 客户和 C4E/Team 用户，指标数据会导出到 BigQuery 用于更深入的分析和报表。

### Resource 合并

OTEL Resource 通过多层检测器合并构建：

```
base resource
  + OS detector（操作系统信息）
  + Host arch detector（主机架构信息）
  + Env detectors（环境变量检测）
  → merged resource
```

### 刷新机制

`flushTelemetry()` 强制刷新所有 Provider 中的缓冲数据，超时时间为 **2 秒**。在进程退出前调用以确保数据不丢失。

---

## Session Tracing (sessionTracing.ts)

会话追踪系统提供详细的请求链路追踪能力。

### 启用条件

`isEnhancedTelemetryEnabled()` 函数检查是否启用增强遥测：

- **Feature gate**: 静态特性开关
- **GrowthBook**: 通过 `enhanced_telemetry_beta` 动态门控

### Span 类型

| Span 类型 | 说明 |
|-----------|------|
| `interaction` | 交互 span：包裹从用户请求到 Claude 响应的完整流程 |
| `llm_request` | LLM 请求 span：单次模型调用 |
| `tool` | 工具 span：工具调用的完整生命周期 |
| `tool.blocked_on_user` | 工具阻塞 span：工具等待用户确认的子 span |
| `tool.execution` | 工具执行 span：工具实际执行的子 span |
| `hook` | Hook span：钩子执行 |

#### 为什么session tracing有6种span？

一个用户请求的完整生命周期需要独立度量每个阶段：`interaction`（整体用户交互）、`llm_request`（单次模型调用，含 TTFT）、`tool`（工具完整生命周期）、`tool.blocked_on_user`（等待用户确认权限）、`tool.execution`（工具实际执行）、`hook`（钩子执行）。如果只有一个粗粒度 span，无法区分"模型思考慢"和"工具执行慢"和"用户审批慢"——而这三者的优化方向完全不同。6 种 span 的层次关系（interaction 包含 llm_request + tool，tool 包含 blocked_on_user + execution）形成了完整的因果链路。

### Interaction Spans

交互 span 是根 span（root span），包裹了从用户发送请求到 Claude 完成响应的整个过程。一个 interaction span 内可能包含多个 LLM request span 和 tool span。

### LLM Spans

LLM 请求 span 追踪单次模型调用的详细信息：

- `input_tokens` — 输入 token 数
- `output_tokens` — 输出 token 数
- `cache_read_tokens` — 缓存读取 token 数
- `ttft_ms` — Time To First Token（首 token 延迟，毫秒）

### Tool Spans

工具 span 追踪工具调用的完整生命周期，内部可能包含：

- **blocked-on-user 子 span**: 当工具需要用户确认（如文件写入确认）时，记录等待用户响应的时间
- **execution 子 span**: 工具实际执行的时间

### 孤立 Span 清理

系统通过后台定时任务清理孤立 span（orphaned spans）：

- **TTL**: 30 分钟
- 定期扫描所有活跃 span，将超过 TTL 的 span 强制结束并驱逐
- 防止因异常导致 span 永不关闭而造成内存泄漏

### Perfetto 集成

支持 **Perfetto** 追踪格式，通过并行的文件写入方式生成 trace 文件，可导入 Perfetto UI 进行可视化调试分析。

---

## Event Logging (events.ts)

### logOTelEvent()

将事件以 OTEL log record 的形式发射。每个事件既是分析事件，也同时作为 OTEL 日志记录。

### 事件排序

通过 **单调递增** 的 `eventSequence` 计数器保证事件的全局有序性。每个事件在创建时被分配一个递增的序号，确保在异步环境下事件的因果顺序不被打乱。

### 提示词脱敏

`redactIfDisabled()` 函数在 `OTEL_LOG_USER_PROMPTS` 环境变量未设置时，自动对用户提示词进行脱敏处理。默认行为是脱敏（保护隐私），仅在用户显式启用该环境变量时才保留原始提示词。

---

## Plugin Telemetry (pluginTelemetry.ts)

插件遥测系统为插件生态提供标准化的遥测能力。

### 插件 ID 哈希

`hashPluginId()` 函数对插件 ID 进行 **SHA256 哈希** 处理，截取前 **16 个字符** 作为匿名化标识。避免在遥测数据中暴露插件的具体名称或路径。

### 插件作用域分类

`getTelemetryPluginScope()` 函数将插件分类为以下作用域：

| 作用域 | 说明 |
|--------|------|
| `official` | Anthropic 官方插件 |
| `org` | 组织级插件 |
| `user-local` | 用户本地插件 |
| `default-bundle` | 默认捆绑插件 |

### 插件命令错误分类

`classifyPluginCommandError()` 将插件命令执行错误分为 5 个类别：

1. **network** — 网络相关错误（连接失败、超时等）
2. **not-found** — 命令或资源未找到
3. **permission** — 权限不足
4. **validation** — 输入验证失败
5. **unknown** — 未分类的其他错误

### 会话级插件分析

`logPluginsEnabledForSession()` 在会话开始时记录当前启用的所有插件，按插件粒度上报分析事件。

### 加载错误分析

`logPluginLoadErrors()` 记录插件加载过程中的错误，按错误粒度上报，帮助诊断插件兼容性和配置问题。

---

## Cost Tracking (cost-tracker.ts)

费用跟踪系统实时追踪 API 调用的 token 消耗和费用。

### 累计追踪

按模型（per-model）维度累计以下指标：

| 指标 | 说明 |
|------|------|
| `input_tokens` | 输入 token 数 |
| `output_tokens` | 输出 token 数 |
| `cache_read_tokens` | 缓存读取 token 数 |
| `cache_creation_tokens` | 缓存创建 token 数 |
| `cost` | 累计费用（美元） |
| `duration` | 累计耗时 |

### 费用格式化

`formatTotalCost()` 函数生成模型使用量的分解展示，按模型维度列出各项指标和费用。

### 会话持久化

- `saveCurrentSessionCosts()` — 将当前会话的费用状态保存到磁盘
- `restoreCostStateForSession()` — 在会话恢复时从磁盘加载之前的费用状态

### StoredCostState 类型

```typescript
interface StoredCostState {
  input_tokens: number;
  output_tokens: number;
  cache_read_tokens: number;
  cache_creation_tokens: number;
  cost: number;
  duration: number;
  // ... 其他追踪字段
}
```

包含所有追踪字段的完整快照，支持会话间费用状态的无缝衔接。

### React Hook (costHook.ts)

`costHook.ts` 提供 React hook，在用户退出时展示费用摘要信息，包括本次会话的总 token 消耗和费用。

---

## Gateway Detection

系统能够自动检测和识别用户请求链路中可能存在的 API 网关/代理。

### 已知网关指纹

通过响应头、请求特征等方式识别以下网关：

| 网关 | 说明 |
|------|------|
| **LiteLLM** | 开源 LLM 代理网关 |
| **Helicone** | LLM 可观测性平台 |
| **Portkey** | AI 网关和可观测性 |
| **Cloudflare** | Cloudflare AI Gateway |
| **Kong** | Kong API Gateway |
| **Braintrust** | AI 评估和代理平台 |
| **Databricks** | Databricks Model Serving |

检测到的网关信息会被记录到遥测数据中，帮助 Anthropic 了解用户的 API 访问拓扑和可能因网关引入的问题。

---

## 工程实践指南

### 添加新的分析事件

**步骤清单：**

1. **使用 `logEvent()` 发送事件**：从 `src/services/analytics/index.ts` 导入 `logEvent`
   ```typescript
   import { logEvent } from 'src/services/analytics/index.js'
   logEvent('tengu_your_event_name', {
     property1: 'value',
     property2: 123,
   })
   ```
2. **定义事件名和属性**：事件名使用 `tengu_` 前缀（源码中所有内部事件均遵循此约定）
3. **注意 Proto 字段**：以 `_PROTO_*` 为前缀的属性仅发送给第一方后端，会被 `stripProtoFields()` 从 Datadog 中剥离
4. **双通道路由**：事件同时发送到 Datadog（实时监控）和 1P（长期分析），通过 `SinkKillswitch` 可独立关闭
5. **采样控制**：高频事件使用 `shouldSampleEvent()` 进行降采样，新事件需评估是否需要采样

**注意**：`analytics/index.ts` 头部注释明确说明："DESIGN: This module has NO dependencies to avoid import cycles"——不要在此模块中引入其他业务模块的依赖。

### 调试遥测丢失

**排查步骤：**

1. **检查环境变量**：`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 设置时会禁用非必要遥测
2. **检查 Sink Killswitch**：`sinkKillswitch.ts` 中的 `datadog` 和 `firstParty` 开关可独立关闭某个通道
3. **检查 GrowthBook 门控**：Datadog 事件受 `tengu_log_datadog_events` feature gate 控制，`shouldTrackDatadog()` 可能因门控返回 false
4. **确认 sink 已附加**：事件在 `attachAnalyticsSink()` 被调用前会排入队列，如果 sink 从未附加则事件永不发送
5. **fail-open 原则**：killswitch 配置缺失或读取失败时默认保持 sink 开启——如果遥测仍然丢失，问题可能在网络层

**源码关键位置**：
- `analytics/index.ts` — 事件队列和 sink 附加
- `analytics/sinkKillswitch.ts` — kill switch 逻辑（注意注释："Must NOT be called from inside is1PEventLoggingEnabled()"）
- `analytics/datadog.ts` — Datadog sink 实现（注释："use via src/services/analytics/index.ts > logEvent"）

### 性能 Trace

**使用 Perfetto trace 文件分析请求链性能：**

1. Session tracing 生成 Perfetto 格式的 trace 文件
2. 导入到 [Perfetto UI](https://ui.perfetto.dev/) 进行可视化分析
3. 6 种 span 类型提供完整因果链路：`interaction` → `llm_request` + `tool`（含 `blocked_on_user` + `execution`）+ `hook`
4. 关键指标：`ttft_ms`（首 token 延迟）、`input_tokens`/`output_tokens`（token 消耗）、`cache_read_tokens`（缓存命中）

**启用增强遥测**：通过 `isEnhancedTelemetryEnabled()` 检查，受 `enhanced_telemetry_beta` GrowthBook 门控

### OTEL 导出配置

| Exporter 类型 | 适用场景 |
|--------------|----------|
| `console` | 本地开发调试 |
| `otlp` (gRPC) | 生产环境高性能传输 |
| `otlp` (HTTP/JSON) | 防火墙限制 gRPC 时的替代 |
| `prometheus` | 已有 Prometheus 基础设施时 |

**刷新机制**：`flushTelemetry()` 超时 2 秒，在进程退出前调用确保数据不丢失。

### 常见陷阱

| 陷阱 | 详情 | 解决方案 |
|------|------|----------|
| 遥测是异步 fire-and-forget | 不保证送达，发送失败静默处理 | 关键业务逻辑不要依赖遥测成功 |
| 不要在遥测中记录敏感数据 | `redactIfDisabled()` 在 `OTEL_LOG_USER_PROMPTS` 未设置时自动脱敏 | 默认行为是脱敏，仅显式启用时才保留原始提示词 |
| 孤立 span 内存泄漏 | 异常导致 span 永不关闭 | 系统自动清理 TTL 超过 30 分钟的孤立 span |
| 事件序号保证因果顺序 | `eventSequence` 单调递增计数器 | 异步环境下依赖序号而非时间戳判断事件顺序 |
| 插件遥测隐私化 | `hashPluginId()` 使用 SHA256 截取前 16 字符 | 遥测数据中看不到插件原始名称/路径 |


---

[← 错误恢复](../17-错误恢复/error-recovery.md) | [目录](../README.md) | [反馈与调查 →](../19-反馈与调查/feedback-system.md)
