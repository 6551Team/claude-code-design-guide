# Keybinding System

Claude Code 的键绑定系统提供完整的快捷键定义、解析、匹配和执行框架，支持单键、组合键和 chord 序列，并集成 Vim 模式。

### 设计理念

#### 为什么 50+ 个动作和 chord 支持？

Claude Code 的核心用户群是高级开发者——Vim/Emacs 用户、终端重度使用者。这些用户期望丰富的键绑定和 chord 序列（如 `ctrl+k ctrl+s`），缺少这种能力会流失 power user 群体。50+ 个动作标识符（`KEYBINDING_ACTIONS`）覆盖了从应用级（退出、中断、清屏）到组件级（自动补全、标签页、历史搜索）的所有交互场景。chord 支持让有限的键位组合空间得到扩展——单按键和修饰键组合数量有限，chord 序列提供了几乎无限的命名空间。

#### 为什么上下文匹配？

同一按键在不同 UI 状态下有不同行为：`Up` 键在 Chat 上下文中是历史上翻，在 Autocomplete 上下文中是选择上一个补全项，在 Transcript 上下文中是向上滚动。17 个上下文名称（`KEYBINDING_CONTEXTS`）定义了键绑定的作用域，`resolveKey` 在给定上下文中查找匹配项，避免跨上下文的按键冲突。`ChordResolveResult` 中的 `chord_started`/`chord_cancelled` 状态让系统能在 chord 序列进行中暂停正常按键处理，直到序列完成或超时取消。

---

## Schema (schema.ts)

### KEYBINDING_CONTEXTS

定义 17 个上下文名称，用于区分不同 UI 状态下的键绑定作用域：

- `Global` - 全局上下文，任何状态下均生效
- `Chat` - 聊天输入区域
- `Autocomplete` - 自动补全菜单
- `Task` - 任务执行中
- `Confirmation` - 确认对话框
- `Tabs` - 标签页导航
- `Transcript` - 会话记录浏览
- `HistorySearch` - 历史搜索模式
- 以及其他上下文（Vim Normal、Vim Insert、Diff、FileViewer 等）

### KEYBINDING_ACTIONS

定义 50+ 个动作标识符，采用 `context:action` 命名约定：

- **App 级别**: `app:exit`, `app:interrupt`, `app:clear`, `app:help`, `app:toggleVim`
- **Chat 级别**: `chat:submit`, `chat:newline`, `chat:paste`, `chat:historyUp`, `chat:historyDown`, `chat:cancel`
- **Confirm 级别**: `confirm:yes`, `confirm:no`, `confirm:always`, `confirm:explain`
- **Autocomplete 级别**: `autocomplete:accept`, `autocomplete:dismiss`, `autocomplete:next`, `autocomplete:prev`
- **Tabs 级别**: `tabs:next`, `tabs:prev`, `tabs:close`
- **Transcript 级别**: `transcript:scrollUp`, `transcript:scrollDown`, `transcript:pageUp`, `transcript:pageDown`, `transcript:top`, `transcript:bottom`
- **HistorySearch 级别**: `history:open`, `history:close`, `history:next`, `history:prev`, `history:select`

### KeybindingBlockSchema

Zod schema，定义单个上下文的绑定块结构：

```typescript
// 每个 block 包含 context + 绑定数组
{
  context: z.enum(KEYBINDING_CONTEXTS),
  bindings: z.array(z.object({
    action: z.enum(KEYBINDING_ACTIONS),
    key: z.string(),        // 如 "ctrl+k"
    when?: z.string(),      // 条件表达式
    unbound?: z.boolean(),  // 解绑该动作
  }))
}
```

### KeybindingsSchema

完整的 `keybindings.json` schema，为 block 数组：

```typescript
KeybindingsSchema = z.array(KeybindingBlockSchema)
```

用户可在 `~/.claude/keybindings.json` 中覆盖默认绑定。

---

## Default Bindings (defaultBindings.ts)

### 平台特定处理

- 检测 Windows VT 模式（Virtual Terminal processing），影响某些控制序列的可用性
- macOS/Linux 与 Windows 的修饰键映射差异（Meta vs Alt）

### 各上下文默认绑定

**Global 上下文**:
- `ctrl+c` → `app:interrupt` (中断当前操作)
- `ctrl+d` → `app:exit` (退出应用)
- `ctrl+l` → `app:clear` (清屏)

**Chat 上下文**:
- `Enter` → `chat:submit`
- `shift+Enter` → `chat:newline`
- `Up` → `chat:historyUp`
- `Down` → `chat:historyDown`

**Autocomplete 上下文**:
- `Tab` → `autocomplete:accept`
- `Escape` → `autocomplete:dismiss`
- `Up/Down` → `autocomplete:prev/next`

**Confirmation 上下文**:
- `y` → `confirm:yes`
- `n` → `confirm:no`
- `a` → `confirm:always`
- `e` → `confirm:explain`

**Tabs 上下文**:
- `ctrl+tab` / `ctrl+shift+tab` → 标签页切换

**Transcript/HistorySearch 等上下文** 均有对应默认绑定。

---

## Parser (parser.ts)

### parseKeystroke

```typescript
parseKeystroke("ctrl+shift+k") → ParsedKeystroke
```

将字符串表示的按键组合解析为结构化对象：
- 提取修饰键：ctrl, shift, alt, meta
- 提取主键名
- 规范化大小写和别名

### parseChord

```typescript
parseChord("ctrl+k ctrl+s") → Chord
```

解析多按键序列（chord），以空格分隔的多个 keystroke 组成：
- 返回 `Chord` 类型，包含有序的 `ParsedKeystroke[]`
- 支持任意长度的 chord 序列

### keystrokeToString / keystrokeToDisplayString

```typescript
keystrokeToString(keystroke)              → "ctrl+shift+k"  // 规范化字符串
keystrokeToDisplayString(keystroke, platform) → "Ctrl+Shift+K"  // 平台感知的显示字符串
```

- `keystrokeToString`: 输出规范化的内部表示
- `keystrokeToDisplayString`: 根据平台输出用户友好的显示文本（macOS 使用符号如 ⌘⇧⌥）

### parseBindings

```typescript
parseBindings(blocks: KeybindingBlock[]) → ParsedBinding[]
```

将嵌套的绑定块结构展平为一维的 `ParsedBinding[]` 数组，每个条目包含已解析的 context、action 和 keystroke/chord。

---

## Matcher (match.ts)

### getKeyName

```typescript
getKeyName(input: string, key: Key) → string
```

将 Ink 的 input/key 事件规范化为统一的键名字符串，处理特殊键（箭头键、功能键、空格等）的映射。

### matchesKeystroke

```typescript
matchesKeystroke(input: string, key: Key, target: ParsedKeystroke) → boolean
```

判断当前按键事件是否匹配目标 keystroke：
- 比较修饰键状态（ctrl, shift, alt, meta）
- 比较主键名
- 处理大小写和平台差异

### matchesBinding

```typescript
matchesBinding(input: string, key: Key, binding: ParsedBinding) → boolean
```

判断当前按键事件是否匹配某个绑定的第一个 keystroke（对于 chord，仅匹配序列中的第一步）。

---

## Resolver (resolver.ts)

### ResolveResult

单步解析结果：

```typescript
type ResolveResult = 'match' | 'none' | 'unbound'
```

- `match`: 找到匹配的绑定
- `none`: 无匹配
- `unbound`: 匹配到但被用户显式解绑

### ChordResolveResult

Chord 感知的解析结果：

```typescript
type ChordResolveResult = 'match' | 'none' | 'unbound' | 'chord_started' | 'chord_cancelled'
```

- `chord_started`: 匹配到 chord 的前缀，进入等待后续按键状态
- `chord_cancelled`: chord 序列中途被取消（超时或无匹配的后续键）

### resolveKey

```typescript
resolveKey(input, key, context, bindings) → { result: ResolveResult, action?: string }
```

单次按键解析，在给定上下文的绑定集合中查找匹配项。

### resolveKeyWithChordState

```typescript
resolveKeyWithChordState(input, key, context, bindings, chordState) → {
  result: ChordResolveResult,
  action?: string,
  newChordState: ChordState
}
```

Chord 感知的按键解析：
- 维护 pending chord state
- 如果当前按键匹配某个 chord 的前缀，返回 `chord_started` 并更新状态
- 如果匹配完整 chord，返回 `match`
- 如果 pending chord 状态下按键不匹配，返回 `chord_cancelled` 并重置

### keystrokesEqual

```typescript
keystrokesEqual(a: ParsedKeystroke, b: ParsedKeystroke) → boolean
```

比较两个 keystroke 是否等价，将 alt 和 meta 视为等价（collapse alt/meta）。

---

## React Hooks

### useKeybinding

```typescript
useKeybinding(action: string, handler: () => void, options?: {
  context?: string,
  enabled?: boolean,
  priority?: number,
})
```

绑定单个 action 到 handler：
- 自动处理 chord 序列的状态管理
- 支持条件启用/禁用
- 在组件卸载时自动清理
- 支持优先级（高优先级覆盖低优先级）

### useKeybindings

```typescript
useKeybindings(handlers: Record<string, () => void>, options?: {
  context?: string,
  enabled?: boolean,
})
```

批量绑定多个 action：
- 传入 action → handler 映射对象
- 内部复用 `useKeybinding` 的逻辑
- 适用于组件需要响应多个快捷键的场景

---

## Vim Mode (src/vim/, 5 files)

#### 为什么集成 Vim 模式？

Vim 用户形成了一种肌肉记忆：他们在所有文本编辑场景中都期望 Vim 键绑定。不提供 Vim 模式意味着这些用户在 Claude Code 的输入框中无法使用习惯的编辑方式（如 `ciw` 修改当前单词、`dd` 删除行），这是一个显著的体验断裂。5 个文件的模块划分（motions / operators / textObjects / transitions / types）镜像了 Vim 自身的概念模型：操作符 + 动作 = 命令。`RecordedChange` 的 discriminated union 设计支持 dot-repeat（`.` 命令），`MAX_VIM_COUNT = 10000` 防止意外的极大重复次数造成界面卡死。

### VimState

```typescript
type VimState = 'INSERT' | 'NORMAL'
```

Vim 的两个主要模式，INSERT 模式下按键直接输入文本，NORMAL 模式下按键触发 Vim 命令。

### CommandState

定义 10 种命令解析状态：

- `idle` - 等待命令输入
- `count` - 正在输入数字前缀
- `operator` - 等待 motion 或 text object（如 `d` 后等待 `w`）
- `find` - 等待 f/F/t/T 的目标字符
- `replace` - 等待 r 的替换字符
- `register` - 等待 " 后的寄存器名
- `mark` - 等待 m 后的标记名
- `goto_mark` - 等待 ' 或 ` 后的标记名
- `z_command` - 等待 z 后的子命令
- `g_command` - 等待 g 后的子命令

### PersistentState

跨命令持久化的状态：

```typescript
interface PersistentState {
  lastChange: RecordedChange | null   // dot-repeat (.) 用
  lastFind: { char: string, direction: 'forward' | 'backward', inclusive: boolean } | null
  register: Record<string, string>     // 寄存器内容
}
```

### RecordedChange

用于 dot-repeat 的变更记录，采用 discriminated union 设计：

```typescript
type RecordedChange =
  | { type: 'insert', text: string }
  | { type: 'delete', range: Range, register?: string }
  | { type: 'replace', range: Range, text: string }
  | { type: 'operator', operator: string, motion: string, count?: number }
  // ... 其他变体
```

### MAX_VIM_COUNT

```typescript
const MAX_VIM_COUNT = 10000
```

数字前缀的最大值上限，防止意外输入过大的重复次数。

### 模块划分（5 个文件）

1. **motions.ts** - 光标移动命令
   - 字符移动: h, l
   - 单词移动: w, W, b, B, e, E
   - 行内移动: 0, ^, $, f, F, t, T
   - 行间移动: j, k, gg, G
   - 搜索移动: /, ?, n, N

2. **operators.ts** - 操作符命令
   - d (删除), c (修改), y (复制)
   - 操作符 + motion 组合
   - 行操作: dd, cc, yy
   - 大写变体: D, C, Y

3. **textObjects.ts** - 文本对象
   - 内部/外部: iw, aw, iW, aW
   - 括号对象: i(, a(, i[, a[, i{, a{
   - 引号对象: i", a", i', a'
   - 标签对象: it, at

4. **transitions.ts** - 模式转换
   - NORMAL → INSERT: i, I, a, A, o, O
   - INSERT → NORMAL: Escape
   - 命令状态转换逻辑
   - 计数前缀处理

5. **index.ts / types.ts** - 入口和类型定义
   - VimState, CommandState, PersistentState 类型导出
   - Vim 引擎主循环
   - 按键事件到 Vim 命令的路由

---

## 工程实践指南

### 自定义键绑定

**在 `~/.claude/keybindings.json` 中覆盖默认绑定：**

```json
[
  {
    "context": "Chat",
    "bindings": [
      { "action": "chat:submit", "key": "ctrl+enter" },
      { "action": "chat:newline", "key": "enter" }
    ]
  },
  {
    "context": "Global",
    "bindings": [
      { "action": "app:help", "key": "ctrl+k ctrl+h" }
    ]
  }
]
```

**支持的功能：**
- **单键绑定**：`ctrl+c`、`shift+enter`、`tab` 等
- **Chord 组合**：空格分隔的多按键序列（如 `ctrl+k ctrl+s`）
- **解绑动作**：设置 `"unbound": true` 移除默认绑定
- **条件绑定**：`"when"` 条件表达式控制绑定生效时机

**合并策略**：用户绑定追加到默认绑定之后，相同 context + action 的用户绑定会覆盖默认绑定。源码 `loadUserBindings.ts:197`：`mergedBindings = [...defaultBindings, ...userParsed]`

### 调试按键不响应

**排查步骤：**

1. **检查当前上下文模式**：17 个上下文（Global, Chat, Autocomplete, Task, Confirmation, Tabs, Transcript, HistorySearch, Vim Normal, Vim Insert 等）决定了哪些绑定生效
2. **检查是否被更高优先级绑定覆盖**：`resolveKey()` 在给定上下文的绑定集合中查找匹配项，后注册的绑定优先
3. **检查 chord 状态**：如果处于 chord 序列中间（`chord_started` 状态），正常按键处理被暂停直到序列完成或超时取消
4. **检查 Vim 模式**：Vim 模式有独立的键处理逻辑，INSERT 模式下按键直接输入文本，NORMAL 模式下触发 Vim 命令
5. **检查 keybindings.json 格式**：
   - 必须是数组格式
   - 每个 block 需要 `context` 和 `bindings` 字段
   - 无效格式会触发警告并回退到默认绑定
6. **检查平台差异**：
   - Windows VT 模式影响某些控制序列
   - macOS/Linux 与 Windows 的修饰键映射差异（Meta vs Alt）
   - `keystrokesEqual()` 将 alt 和 meta 视为等价

**源码日志**：`loadUserBindings.ts` 中 `logForDebugging('[keybindings] ...')` 输出加载和验证日志。

**硬编码按键**：`useExitOnCtrlCD.ts` 注释说明 Ctrl+C 和 Ctrl+D 是硬编码的退出键，不能通过 `keybindings.json` 重绑定。

### 添加新的键绑定动作

**步骤清单：**

1. **定义 action**：在 `schema.ts` 的 `KEYBINDING_ACTIONS` 中添加新的动作标识符（遵循 `context:action` 命名约定）
2. **注册默认绑定**：在 `defaultBindings.ts` 中为新动作添加默认按键
3. **添加上下文条件**：如果动作仅在特定上下文生效，确保在正确的 `KEYBINDING_CONTEXTS` 中注册
4. **实现 handler**：使用 `useKeybinding(action, handler)` 或 `useKeybindings(handlers)` React hook 绑定处理逻辑
5. **处理 chord 状态**：如果动作使用 chord 序列，`resolveKeyWithChordState()` 自动管理 pending 状态

**React Hook 使用示例：**
```typescript
// 单个动作
useKeybinding('my-context:my-action', () => { /* handler */ }, {
  context: 'MyContext',
  enabled: true,
})

// 批量绑定
useKeybindings({
  'my-context:action1': () => { /* handler 1 */ },
  'my-context:action2': () => { /* handler 2 */ },
}, { context: 'MyContext' })
```

### Vim 模式扩展

**Vim 引擎模块划分：**
- `motions.ts` — 光标移动命令（h/l/w/b/e/0/$/f/F/t/T/gg/G 等）
- `operators.ts` — 操作符命令（d/c/y 及组合 dd/cc/yy/D/C/Y）
- `textObjects.ts` — 文本对象（iw/aw/i(/a(/i"/a" 等）
- `transitions.ts` — 模式转换（NORMAL↔INSERT）和命令状态转换
- `types.ts` — 类型定义和 Vim 引擎主循环

**Dot-repeat 支持**：`RecordedChange` 使用 discriminated union 记录变更，支持 `.` 命令重复。

### 常见陷阱

| 陷阱 | 详情 | 解决方案 |
|------|------|----------|
| 终端模拟器可能拦截某些组合键 | Ctrl+S（终端流控制）、Ctrl+Z（挂起）等被终端拦截，不会到达 Claude Code | 使用 chord 序列绕过，或在终端设置中禁用这些快捷键 |
| Vim 模式有独立的键处理逻辑 | Vim NORMAL 模式下按键触发 Vim 命令而非默认键绑定 | 10 种 Vim 命令状态（idle/count/operator/find/replace 等）独立管理 |
| `MAX_VIM_COUNT = 10000` | 防止意外输入过大的数字前缀重复次数 | 超过此值的数字前缀被截断 |
| Chord 序列中途取消 | chord 序列进行中无匹配的后续键会返回 `chord_cancelled` 并重置 | 用户需要重新开始整个 chord 序列 |
| keybindings.json 格式错误 | 无效的 JSON 或不符合 schema 的结构 | 自动回退到默认绑定，输出警告日志 |
| 平台修饰键差异 | macOS 使用 Meta（Command），Windows/Linux 使用 Alt | `keystrokesEqual()` 将 alt 和 meta 视为等价处理 |


---

[← 会话管理](../26-会话管理/session-management.md) | [目录](../README.md) | [Vim 模式 →](../28-Vim模式/vim-mode.md)
