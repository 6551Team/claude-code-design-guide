# 原生模块与核心工具

Claude Code 的原生模块层提供了文件索引、颜色差异计算、布局引擎等核心能力，以及输出样式、迁移和模型选择等基础设施。

### 设计理念：为什么需要 FFI 桥接而不是纯 JS？

原生模块层（`native-ts/`）的存在是性能驱动的设计决策：

1. **色差计算** -- Color Diff 涉及大量像素级数学运算（色彩空间转换、矩阵乘法），纯 JS 的浮点运算性能远不及编译型语言
2. **文件索引** -- FileIndex 需要对数万文件路径进行模糊匹配和排序，源码注释明确提到这是"Pure-TypeScript port of vendor/file-index-src (Rust NAPI module)"，原始版本基于 Rust nucleo 库（`native-ts/file-index/index.ts` 第 2-4 行）
3. **Yoga 布局** -- Ink 终端渲染需要 flex 布局计算，Yoga 是 Facebook 的 C++ 布局引擎，通过 FFI 绑定可以获得接近原生的布局性能

### 设计理念：为什么原生模块是可选的？

![原生模块可选降级策略](../diagrams/native-module-fallback.svg)

不是所有平台都能编译原生代码（如某些 CI 环境、受限容器、Windows ARM 等），优雅降级到纯 JS 实现保证可用性：
- FileIndex 已经实现了完整的纯 TypeScript 版本（371 行），评分算法精确模拟 nucleo 的行为
- 这种"可选原生"模式是 Node.js 生态的最佳实践——npm 包 `sharp`、`better-sqlite3` 等都采用类似策略

### 工程实践

**原生模块编译失败时的调试**：
- 检查 `node-gyp` 环境：需要 Python 3.x、C++ 编译器（Windows 需 Visual Studio Build Tools）
- 确认 Node.js 版本兼容性：原生模块通常针对特定 Node ABI 版本编译
- 如果是 Bun 运行时，检查 `bun:ffi` 支持状态

**添加新原生模块的清单**：
1. 在 `native-ts/` 下创建新模块目录
2. 同时提供纯 JS fallback 实现（这是硬性要求，不是可选的）
3. 在加载层实现自动检测：优先加载原生版本，失败时回退到 JS 版本
4. 确保纯 JS 版本的 API 签名与原生版本完全一致

---

## File Index (file-index/index.ts, 371 lines)

### 概述

纯 TypeScript 实现的模糊文件搜索引擎，替代了原先的 Rust nucleo 绑定，消除了原生依赖。

### FileIndex 类

#### loadFromFileList

```typescript
loadFromFileList(files: string[]) → void
```

同步加载文件列表：
- 对文件路径进行去重
- 为每个文件构建搜索索引（bitmap + 规范化路径）

#### loadFromFileListAsync

```typescript
loadFromFileListAsync(files: string[]) → Promise<void>
```

异步加载文件列表：
- 分块处理，每块之间 yield 回事件循环
- 避免大文件列表阻塞 UI 渲染
- 适用于包含数万文件的大型项目

#### search

```typescript
search(query: string, limit: number) → SearchResult[]
```

执行模糊搜索，返回 top-K 结果：
- 对 query 进行规范化处理
- 对所有已索引文件计算相似度分数
- 使用 top-K 选择算法（避免完整排序）
- 返回按分数降序排列的结果

### 评分算法

#### 基础分数

```typescript
SCORE_MATCH  // 基础匹配分
```

每个匹配字符获得基础分。

#### 奖励分

- **Boundary bonus**: 匹配位于单词边界（路径分隔符、下划线、连字符后）
- **CamelCase bonus**: 匹配位于驼峰命名的大写字母位置
- **Consecutive bonus**: 连续匹配字符获得递增奖励

#### Bitmap 优化

```
26-bit mask → O(1) 字母存在性检测
```

每个文件维护一个 26-bit 位掩码，记录文件路径中出现的字母。查询时先检查 bitmap，如果查询中的某个字母不在文件的 bitmap 中，可以立即跳过该文件，实现 O(1) 的快速拒绝。

#### Test 文件惩罚

```typescript
// 测试文件分数乘以 1.05x（除数，降低排名）
```

包含 test/spec/mock 等模式的文件路径会受到轻微的排名惩罚（分数除以 1.05），使非测试文件在同等匹配度下排名更高。

#### Top-K 选择

使用堆或部分排序算法选取前 K 个结果，时间复杂度 O(n log k) 而非完整排序的 O(n log n)。

---

## Color Diff (color-diff/index.ts, ~10KB)

### 概述

纯 TypeScript 实现的差异计算引擎。

### 核心功能

- Color matrix 计算，用于量化视觉差异
- 支持多种颜色空间的差异度量
- 为终端 diff 显示提供底层计算能力

---

## Yoga Layout (yoga-layout/index.ts, 27KB)

### 概述

Yoga 布局引擎的绑定，为 Ink 终端渲染提供 flex 布局支持。

### 枚举定义

```typescript
// 方向
enum Direction { Inherit, LTR, RTL }

// 主轴对齐
enum Justify { FlexStart, Center, FlexEnd, SpaceBetween, SpaceAround, SpaceEvenly }

// 交叉轴对齐
enum Align { Auto, FlexStart, Center, FlexEnd, Stretch, Baseline, SpaceBetween, SpaceAround }

// 显示模式
enum Display { Flex, None }

// 换行
enum Wrap { NoWrap, Wrap, WrapReverse }

// 溢出处理
enum Overflow { Visible, Hidden, Scroll }

// 定位方式
enum Position { Static, Relative, Absolute }
```

### 用途

- 被自定义 Ink renderer 使用
- 实现基于 flexbox 的终端 UI 布局
- 支持嵌套容器、弹性尺寸、对齐和换行

---

## Output Styles (src/outputStyles/)

### loadOutputStylesDir.ts

memoized 加载器，搜索并加载输出样式定义。

#### 搜索路径

```
项目级: .claude/output-styles/
用户级: ~/.claude/output-styles/
```

按优先级从项目级到用户级搜索样式文件。

#### 文件格式

Markdown 文件，带 frontmatter 元数据：

```markdown
---
name: "Custom Style"
description: "A custom output style"
keepCodingInstructions: true
---

Your prompt instructions here...
```

#### OutputStyleConfig 字段

```typescript
interface OutputStyleConfig {
  name: string                      // 样式名称
  description: string               // 样式描述
  prompt: string                    // 注入到系统提示的内容
  source: 'project' | 'user'       // 来源（项目级或用户级）
  keepCodingInstructions: boolean   // 是否保留默认的编码指令
}
```

#### clearOutputStyleCaches

```typescript
clearOutputStyleCaches() → void
```

清除 memoized 缓存，强制下次调用时重新加载样式文件。

---

## Migrations (src/migrations/, 11 files)

用于处理配置和设置的版本迁移，确保旧版配置能平滑升级到新版。

### 迁移列表

| 迁移函数 | 说明 |
|---------|------|
| `migrateAutoUpdatesToSettings` | 将自动更新配置迁移到统一设置系统 |
| `migrateBypassPermissionsAcceptedToSettings` | 将权限绕过标记迁移到设置 |
| `migrateEnableAllProjectMcpServersToSettings` | 将项目 MCP 服务器启用配置迁移到设置 |
| `migrateFennecToOpus` | 将 Fennec（内部代号）模型引用迁移为 Opus |
| `migrateLegacyOpusToCurrent` | 将旧版 Opus 模型 ID 迁移为当前版本 |
| `migrateOpusToOpus1m` | 将 Opus 迁移为 Opus 1M context 版本 |
| `migrateReplBridgeEnabledToRemoteControlAtStartup` | 将 REPL Bridge 配置迁移为远程控制启动配置 |
| `migrateSonnet1mToSonnet45` | 将 Sonnet 1M 迁移为 Sonnet 4.5 |
| `migrateSonnet45ToSonnet46` | 将 Sonnet 4.5 迁移为 Sonnet 4.6 |
| `resetAutoModeOptInForDefaultOffer` | 重置自动模式的 opt-in 状态 |
| `resetProToOpusDefault` | 重置 Pro 用户的默认模型为 Opus |

每个迁移函数：
- 检查是否需要迁移（幂等性）
- 执行迁移逻辑
- 记录迁移完成状态，避免重复执行

---

## Model Selection (src/utils/model/)

### getMainLoopModel

```typescript
getMainLoopModel() → string
```

按优先级确定主循环使用的模型：

```
1. override (代码级强制覆盖)
2. CLI flag (--model 参数)
3. env var (CLAUDE_MODEL 环境变量)
4. settings (用户设置中的 model 配置)
5. default (默认模型)
```

### MODEL_ALIASES

```typescript
const MODEL_ALIASES = [
  'sonnet',      // → claude-sonnet-4-6
  'opus',        // → claude-opus-4-6
  'haiku',       // → claude-haiku
  'best',        // → 当前最佳模型
  'sonnet[1m]',  // → claude-sonnet-4-6 with 1M context
  'opus[1m]',    // → claude-opus-4-6 with 1M context
  'opusplan',    // → opus with planning mode
]
```

用户可使用别名简化模型指定。

### APIProvider

```typescript
type APIProvider = 'firstParty' | 'bedrock' | 'vertex' | 'foundry'
```

- `firstParty`: Anthropic 直接 API
- `bedrock`: AWS Bedrock
- `vertex`: Google Cloud Vertex AI
- `foundry`: 自定义模型服务

### 已弃用模型追踪

维护已弃用模型列表及其退休日期：
- 当用户尝试使用已弃用模型时给出警告
- 自动建议迁移到替代模型
- 包含退休日期信息用于时间线展示

### 1M Context 资格检查

```typescript
checkOpus1mAccess() → Promise<boolean>
checkSonnet1mAccess() → Promise<boolean>
```

检查当前账户是否有权使用 1M context 版本的模型：
- 依据订阅类型和权限级别判断
- 无权限时降级到标准 context 版本

### Model Capabilities

```typescript
// API 查询端点
GET /v1/models

// 本地缓存
~/.claude/cache/model-capabilities.json
```

- 通过 API 查询模型的具体能力参数
- 查询结果缓存到本地文件，避免重复请求
- 包含 context window 大小、支持的功能特性等信息


---

[← 输出样式](../38-输出样式/output-styles.md) | [目录](../README.md) | [迁移系统 →](../40-迁移系统/migration-system.md)
