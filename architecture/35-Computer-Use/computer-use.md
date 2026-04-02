# Computer Use 集成架构

Computer Use 功能通过 MCP Server 形式集成到 Claude Code 中，提供屏幕交互、键鼠控制等计算机操作能力。

---

## Architecture (15 files in utils/computerUse/)

### 核心常量

```typescript
const COMPUTER_USE_MCP_SERVER_NAME = 'computer-use'
const CLI_HOST_BUNDLE_ID = 'com.anthropic.claude-code.cli-no-window'
```

### 终端 Bundle ID 映射

`TERMINAL_BUNDLE_ID_FALLBACK` 将常见终端名称映射到对应的 macOS Bundle ID：

| 终端名称 | Bundle ID |
|---------|-----------|
| iTerm | com.googlecode.iterm2 |
| Terminal | com.apple.Terminal |
| Ghostty | com.mitchellh.ghostty |
| Kitty | net.kovidgoyal.kitty |
| Warp | dev.warp.Warp |
| VSCode | com.microsoft.VSCode |

### 设计理念

#### 为什么 macOS 用 Enigo/Swift 而不是 AppleScript?

源码注释揭示了技术架构:"@ant/computer-use-input (Rust/enigo) -- mouse, keyboard, frontmost app" + "@ant/computer-use-swift -- SCContentFilter screenshots, NSWorkspace apps, TCC"。AppleScript 每次操作延迟 >100ms,且对底层事件控制有限;Enigo 直接调用系统 API (通过 `DispatchQueue.main`),延迟在 10ms 级别。Swift 原生模块则提供 AppleScript 无法做到的屏幕截图 (SCContentFilter)、TCC 权限检测等能力。

#### 为什么 O_EXCL 原子锁?

源码注释写道:"Uses O_EXCL (open 'wx') for atomic test-and-set -- the OS guarantees at most one process succeeds"。多个 agent (如 Swarm 中的 teammates) 可能同时尝试控制鼠标/键盘——原子文件锁确保只有一个会话可以操作,防止多个 agent 同时移动鼠标造成混乱。锁文件包含 `sessionId`、`pid` 和 `acquiredAt`,支持基于 PID 的陈旧锁检测和 60 分钟超时强制回收。

#### 为什么 ESC 热键用 CGEventTap?

源码注释说明了安全考量:"Global Escape -> abort. Mirrors Cowork's escAbort.ts but without Electron: CGEventTap via @ant/computer-use-swift. While registered, Escape is consumed system-wide (PI defense -- a prompt-injected action can't dismiss a dialog with Escape)"。用户需要紧急停止 computer-use 操作,ESC 作为全局热键即使焦点不在 Claude Code 窗口也能触发。更关键的是安全防御——系统级 Escape 拦截防止 prompt injection 攻击利用 Escape 键关闭安全对话框。`notifyExpectedEscape()` 用 100ms 衰减窗口区分模型自身的 Escape 操作和用户的中断意图。

#### 为什么 CFRunLoop pump?

源码注释直接解释了根本原因:"Swift's @MainActor async methods and @ant/computer-use-input's key()/keys() all dispatch to DispatchQueue.main. Under libuv (Node/bun) that queue never drains -- the promises hang. Electron drains it via CFRunLoop so Cowork doesn't need this"。macOS 的主线程派发队列在 Node.js 环境下不会自动排空,必须通过 `_drainMainRunLoop` (每 1ms 调用一次) 手动泵送。引用计数机制 (`retainPump`/`releasePump`) 确保只在有活跃 Computer Use 操作时才运行 RunLoop,避免空转消耗 CPU。

---

## Feature Gates

### 主开关

**getChicagoEnabled()** 控制 Computer Use 功能是否可用：

- 要求 max 或 pro+ 订阅层级
- Anthropic 内部用户 (ants) 可通过 bypass 绕过限制

### 子功能开关

- **pixelValidation**: 像素级坐标验证
- **clipboardPasteMultiline**: 多行文本剪贴板粘贴
- **mouseAnimation**: 鼠标移动动画效果
- **hideBeforeAction**: 执行操作前隐藏 Claude Code 窗口
- **autoTargetDisplay**: 自动选择目标显示器
- **clipboardGuard**: 剪贴板内容保护（防止意外覆盖用户剪贴板）

### 坐标模式

**getChicagoCoordinateMode()** 返回坐标模式：

```typescript
type CoordinateMode = 'pixels' | 'normalized'
```

- **pixels**: 使用绝对像素坐标
- **normalized**: 使用 0-1 归一化坐标
- 首次读取后冻结，运行期间不可变更

---

## Executor (executor.ts, 658 lines)

### 工厂函数

**createCliExecutor()** 创建 CLI 环境下的执行器，底层封装：

- **@ant/computer-use-input**: Rust/enigo 实现的跨平台输入控制
- **@ant/computer-use-swift**: macOS 原生 Swift 实现的系统交互

### 方法集

#### 屏幕操作
- **screenshot()**: 截取屏幕快照
- **zoom(factor)**: 缩放显示

#### 键盘操作
- **key(keys)**: 按键组合（如 Ctrl+C）
- **holdKey(key, duration)**: 持续按住按键
- **type(text)**: 输入文本字符串

#### 剪贴板
- **readClipboard()**: 读取剪贴板内容
- **writeClipboard(text)**: 写入剪贴板内容

#### 鼠标操作
- **moveMouse(x, y)**: 移动鼠标到指定位置
- **click(x, y, button)**: 点击
- **mouseDown(x, y, button)**: 按下鼠标按钮
- **mouseUp(x, y, button)**: 释放鼠标按钮
- **getCursorPosition()**: 获取当前光标位置
- **drag(fromX, fromY, toX, toY)**: 拖拽操作
- **scroll(x, y, deltaX, deltaY)**: 滚动操作

#### 应用管理
- **getFrontmostApp()**: 获取当前前台应用信息
- **listInstalledApps()**: 列出已安装应用
- **getAppIcon(bundleId)**: 获取应用图标
- **listRunningApps()**: 列出正在运行的应用
- **openApp(bundleId)**: 打开指定应用

#### 准备操作
- **prepareForAction()**: 执行操作前的准备工作（如隐藏窗口）

### 动画移动

**animatedMove()** 实现平滑鼠标移动：

- 使用 **ease-out-cubic** 缓动曲线
- 移动速度：**2000 px/sec**
- 提供自然的鼠标移动视觉效果

### CLI 特殊处理

- **无 click-through**: CLI 模式不支持点击穿透
- **终端代理宿主**: 使用终端作为 surrogate host 应用
- **剪贴板**: 通过 `pbcopy`/`pbpaste` 命令操作剪贴板

---

## Lock System (computerUseLock.ts)

### 原子锁实现

使用 `O_EXCL` 标志实现原子文件创建锁，确保同一时间只有一个 Computer Use 会话：

```typescript
const HOLDER_STALE_MS = 60 * 60 * 1000  // 60 分钟
```

### 锁文件

路径：`~/.claude/computer-use.lock`

```json
{
  "sessionId": "session-uuid",
  "pid": 12345,
  "acquiredAt": "2025-01-01T00:00:00.000Z"
}
```

### 过期恢复

- 基于 PID 的陈旧锁检测
- 如果持锁进程已终止，自动回收锁
- `HOLDER_STALE_MS = 60min` 超时后强制回收

### 零系统调用检查

**isLockHeldLocally()**: 通过内存状态检查锁持有状态，无需任何系统调用，性能极高。

---

## ESC Hotkey (escHotkey.ts)

### CGEventTap 注册

注册系统级 Escape 键事件监听，用于用户中断 Computer Use 操作：

- 使用 macOS CGEventTap API
- 捕获全局 Escape 按键事件

### 预期 Escape 处理

**notifyExpectedEscape()**: 当模型自身需要合成 Escape 按键时调用：

- 创建 **100ms** 的衰减窗口
- 在此窗口内的 Escape 事件被视为模型操作而非用户中断
- 窗口过后恢复正常的中断检测

---

## CFRunLoop (drainRunLoop.ts)

### 引用计数泵

使用引用计数管理的 setInterval 泵维持 CFRunLoop 运转：

- 泵间隔：**1ms**
- 超时保护：**30s** 最大运行时间

### 生命周期管理

- **retainPump()**: 增加引用计数，首次调用时启动泵
- **releasePump()**: 减少引用计数，归零时停止泵
- 确保只在有活跃 Computer Use 操作时才运行 RunLoop

---

## App Filtering (appNames.ts)

### 过滤逻辑

**filterAppsForDescription()** 过滤应用列表，移除噪音应用：

- 屏蔽包含 Helper/Agent/Service/Updater 等关键词的后台应用
- 只保留用户可见的前台应用

### 白名单

**ALWAYS_KEEP_BUNDLE_IDS**: 约 30 个始终保留的核心应用：

- 浏览器：Chrome, Safari, Firefox, Arc, Edge
- 通信：Slack, Discord, Zoom, Teams
- 开发：VSCode, Xcode, Terminal, iTerm2
- 办公：Finder, Notes, Calendar, Mail
- 其他常用应用

### 名称验证

**APP_NAME_ALLOWED**: 应用名称验证规则：

- Unicode 感知的正则表达式
- 最大 **40 字符** 长度限制
- 单次返回最多 **50 个** 应用

---

## Cleanup

### cleanupComputerUseAfterTurn()

每个对话轮次结束后的清理流程：

1. **自动取消隐藏**: 恢复之前隐藏的窗口，设置 **5s 超时** 防止卡死
2. **注销 Esc 监听**: 移除 CGEventTap Escape 键监听
3. **释放锁**: 释放 computer-use.lock 文件锁
4. 释放 CFRunLoop 泵引用

---

## MCP Server

### 服务端创建

**createComputerUseMcpServerForCli()** 为 CLI 环境构建 MCP Server：

- 初始化所有工具定义
- **替换 ListTools**: 用包含应用描述信息的增强版本替换标准 ListTools
- 将应用上下文注入到工具描述中，帮助模型理解当前桌面环境

### 子进程入口

**runComputerUseMcpServer()** 作为独立子进程运行的入口点：

- 使用 **stdio transport** 与宿主进程通信
- 标准的 MCP Server 生命周期管理
- 接收来自宿主进程的工具调用请求并执行

---

## 工程实践指南

### 启用 Computer-Use

1. **平台确认**: Computer-Use 目前**仅支持 macOS**——需要 Enigo (Rust) 和 Swift 原生模块
2. **权限授予**:
   - 辅助功能访问 (Accessibility): 系统偏好设置 → 隐私与安全 → 辅助功能 → 添加 Claude Code
   - 屏幕录制权限: SCContentFilter 截图需要屏幕录制授权
   - TCC 权限: Swift 原生模块通过 TCC 框架检测权限状态
3. **订阅层级**: 需要 max 或 pro+ 订阅——Anthropic 内部用户可通过 bypass 绕过
4. **确认 MCP Server**: `createComputerUseMcpServerForCli()` 初始化 Computer-Use MCP Server,使用 stdio transport 与宿主进程通信

### 调试锁冲突

1. **检查锁文件**: 查看 `~/.claude/computer-use.lock` 内容——包含 `sessionId`、`pid`、`acquiredAt`
2. **确认持锁进程是否存活**: 用锁文件中的 `pid` 检查进程状态——如果进程已终止,说明是残留锁
3. **手动清理残留锁**: 进程崩溃后锁文件可能残留——安全删除 `~/.claude/computer-use.lock` 即可释放锁
4. **超时强制回收**: 锁持有超过 `HOLDER_STALE_MS = 60min` 后自动回收——但一小时内的残留锁需要手动清理
5. **零系统调用检查**: `isLockHeldLocally()` 通过内存状态检查锁,不产生系统调用——用于高频检查场景

### ESC 热键不工作

1. **检查 CGEventTap 权限**: ESC 热键通过 macOS CGEventTap API 注册系统级事件监听——需要辅助功能权限
2. **确认 CFRunLoop 在运行**: Swift 的 `@MainActor` 和 Enigo 的操作都派发到 `DispatchQueue.main`,需要 `_drainMainRunLoop` 每 1ms 泵送。检查 `retainPump()` 是否被调用,引用计数是否 > 0
3. **检查预期 Escape 窗口**: `notifyExpectedEscape()` 创建 100ms 衰减窗口——如果模型刚发送了 Escape 键操作,用户在 100ms 内按 Escape 不会触发中断。等 100ms 后重试
4. **检查与其他应用的冲突**: Escape 是全局热键,如果其他应用也注册了 Escape 监听,可能互相干扰

### 每轮清理检查清单

每个对话轮次结束后,`cleanupComputerUseAfterTurn()` 执行以下清理:

- [ ] 自动取消隐藏窗口 (5s 超时保护)
- [ ] 注销 CGEventTap Escape 键监听
- [ ] 释放 `computer-use.lock` 文件锁
- [ ] 释放 CFRunLoop 泵引用计数

如果清理不完整 (如进程被 SIGKILL),手动检查:锁文件是否残留? Escape 监听是否仍在? 隐藏的窗口是否恢复?

### 常见陷阱

> **多 agent 同时操作会锁冲突**: O_EXCL 原子锁确保同一时间只有一个会话可以控制鼠标/键盘。在 Swarm 场景中,多个 teammate 不能同时使用 Computer-Use——必须通过权限同步机制串行化操作。

> **ESC 是全局热键**: 注册后 Escape 被系统级拦截,**所有应用的 Escape 键都会被消费**——包括 Vim 的 Escape、对话框的取消按钮等。这是 prompt injection 防御的设计选择,但可能影响用户的正常操作。Computer-Use 不活跃时确保注销监听。

> **仅 macOS 支持**: Windows 和 Linux 不支持 Computer-Use——原生模块 (`@ant/computer-use-input` 和 `@ant/computer-use-swift`) 依赖 macOS 特有的 API (CGEvent、SCContentFilter、NSWorkspace、TCC)。

> **CFRunLoop 泵的 CPU 开销**: `_drainMainRunLoop` 每 1ms 调用一次,在 Computer-Use 不活跃时务必通过 `releasePump()` 停止。30s 超时保护会自动停止,但 30s 的空转仍然浪费 CPU。


---

[← Swarm 系统](../34-Swarm系统/swarm-architecture.md) | [目录](../README.md) | [DeepLink →](../36-DeepLink/deeplink-system.md)
