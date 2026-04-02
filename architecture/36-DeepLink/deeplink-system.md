# DeepLink System Architecture

## Protocol

DeepLink 系统允许通过自定义协议从外部应用（浏览器、编辑器等）直接启动 Claude CLI 并传递上下文。

### 设计理念

#### 为什么 claude-cli:// 协议?

让外部应用 (浏览器、IDE、文档) 可以直接启动 Claude Code 并传递上下文——无缝集成。与命令行参数不同,URL 可以嵌入在网页、文档、Slack 消息中直接点击,用户无需打开终端手动输入命令。源码 `parseDeepLink.ts` 支持 `q` (查询)、`cwd` (工作目录)、`repo` (仓库) 三个参数,覆盖了"在这个仓库里问这个问题"的完整上下文传递需求。

#### 为什么 3 平台注册?

macOS (Info.plist + `CFBundleURLTypes` + `lsregister`)、Linux (`.desktop` 文件 + `xdg-mime`)、Windows (Registry `HKEY_CURRENT_USER\Software\Classes`) 各有不同的协议注册机制。源码 `registerProtocol.ts` (349 行) 为每个平台实现了专门的注册逻辑。这不是可以抽象掉的差异——每个平台的 URL scheme 注册是操作系统级的 API,必须各自适配。自动注册由 `backgroundHousekeeping` 任务触发,失败后 24 小时退避,不打断用户工作流。

#### 为什么 URL 解析而不是命令行参数?

源码安全设计证明了 URL 方式的深思熟虑:控制字符拒绝防止终端注入,`cwd` 要求绝对路径防止路径遍历,`repo` 的 slug 格式验证 (`owner/repo`) 限制注入面。URL 格式还支持 shell 引号的多策略处理——`shellQuote()` (POSIX)、`appleScriptQuote()` (macOS osascript)、`psQuote()` (PowerShell)、`cmdQuote()` (cmd.exe),确保参数在任何终端环境下都能安全传递。

```
DEEP_LINK_PROTOCOL = 'claude-cli'
```

**URI 格式**:
```
claude-cli://open?q=...&cwd=...&repo=...
```

- `q`: 查询/提示内容，将作为初始消息传递给 Claude
- `cwd`: 工作目录，指定 Claude 会话的工作路径
- `repo`: 仓库标识（slug 格式），用于上下文定位

**安全措施**:
- **控制字符拒绝**: 拒绝 URI 中包含控制字符的请求，防止终端注入攻击
- **cwd 绝对路径要求**: `cwd` 参数必须是绝对路径，拒绝相对路径以防止路径遍历
- **repo slug 验证**: 验证 repo 参数符合 slug 格式（`owner/repo`），拒绝非法字符

**长度限制**:
```typescript
MAX_QUERY_LENGTH = 5000    // 查询内容最大长度
MAX_CWD_LENGTH = 4096      // 工作目录路径最大长度
```

## Registration

`registerProtocol.ts`（约 349 行）负责在各操作系统上注册 `claude-cli://` 协议处理程序。

### macOS

在 `~/Applications` 目录下创建 `.app` bundle:
- 生成 `Info.plist`，包含 `CFBundleURLTypes` 声明，将 `claude-cli` scheme 绑定到该应用
- 创建指向 CLI 可执行文件的 symlink
- 调用 `lsregister` 向 Launch Services 注册 URL scheme

### Linux

使用 XDG 桌面规范:
- 在 `$XDG_DATA_HOME/applications`（默认 `~/.local/share/applications`）目录下创建 `.desktop` 文件
- 通过 `xdg-mime` 设置 `x-scheme-handler/claude-cli` 的默认处理程序

### Windows

写入注册表键:
- 路径: `HKEY_CURRENT_USER\Software\Classes\claude-cli`
- 设置 `URL Protocol` 值表明这是协议处理程序
- 在 `shell\open\command` 子键下设置 CLI 可执行文件路径

### 自动注册

协议注册由 `backgroundHousekeeping` 任务自动触发:
- **失败退避**: `FAILURE_BACKOFF_MS` 设置 24 小时退避时间，注册失败后不会频繁重试
- 在后台静默执行，不打断用户工作流

## Terminal Launch

`terminalLauncher.ts`（约 558 行）负责在用户首选的终端中启动新的 Claude CLI 会话。

### macOS 终端支持（按偏好排序）

1. iTerm2
2. Ghostty
3. Kitty
4. Alacritty
5. WezTerm
6. Terminal.app（系统默认）

### Linux 终端支持

ghostty, kitty, alacritty, wezterm, gnome-terminal, konsole, xfce4-terminal, mate-terminal, tilix, xterm

### Windows 终端支持

Windows Terminal, pwsh (PowerShell 7+), PowerShell (Windows PowerShell), cmd

### Shell 引号处理

不同终端和 shell 需要不同的引号转义策略:

- **shellQuote()**: 通用 POSIX shell 引号处理，使用单引号包裹并转义内部单引号
- **appleScriptQuote()**: AppleScript 字符串引号处理，用于 macOS 上通过 `osascript` 控制终端
- **psQuote()**: PowerShell 字符串引号处理，处理 PowerShell 特殊字符和转义序列
- **cmdQuote()**: Windows cmd.exe 引号处理，处理 `%`、`^`、`&` 等特殊字符

### 进程分离

- **spawnDetached()**: 以独立进程方式启动终端，确保新启动的 Claude CLI 会话与触发源完全解耦，父进程退出不影响子进程。

## Banner

`banner.ts` 提供 DeepLink 启动时的横幅信息显示:

```typescript
STALE_FETCH_WARN_MS = 7 days  // 7天未 fetch 时显示警告
LONG_PREFILL_THRESHOLD = 1000  // 预填充内容超过1000字符时的阈值
```

- **Git 状态检测**: 读取 `.git/FETCH_HEAD` 的修改时间（mtime），支持 worktree 场景。当上次 fetch 超过 7 天时，在横幅中显示警告，提醒用户仓库可能不是最新状态。
- **长内容提示**: 当 DeepLink 传递的查询内容超过 `LONG_PREFILL_THRESHOLD`（1000 字符）时，显示内容长度提示。

## Terminal Preference

终端偏好系统（仅 macOS）:

- **捕获**: 读取 `TERM_PROGRAM` 环境变量，映射到对应的终端应用标识符
- **持久化**: 将用户的终端偏好持久化存储，后续 DeepLink 启动时优先使用该终端
- **映射关系**: `TERM_PROGRAM` 值（如 `iTerm.app`、`ghostty`、`Apple_Terminal`）到内部终端标识符的映射

---

## 工程实践指南

### 注册协议处理器

不同平台需要不同的注册方式:

1. **macOS** (自动):
   - 系统创建 `~/Applications/` 下的 `.app` bundle,包含 `Info.plist` 的 `CFBundleURLTypes` 声明
   - 调用 `lsregister` 向 Launch Services 注册
   - 通常由 `backgroundHousekeeping` 自动完成,无需手动操作
2. **Windows** (需要 Registry):
   - 写入 `HKEY_CURRENT_USER\Software\Classes\claude-cli` 注册表键
   - 设置 `URL Protocol` 值标记为协议处理程序
   - 在 `shell\open\command` 子键设置 CLI 可执行文件路径
3. **Linux** (需要 desktop file):
   - 在 `$XDG_DATA_HOME/applications` (默认 `~/.local/share/applications`) 创建 `.desktop` 文件
   - 通过 `xdg-mime` 设置 `x-scheme-handler/claude-cli` 的默认处理程序

### 调试链接不打开

1. **检查协议注册是否正确**:
   - macOS: 在终端执行 `open claude-cli://open?q=test`,确认是否启动 Claude CLI
   - Windows: 在浏览器地址栏输入 `claude-cli://open?q=test`,检查是否触发协议处理
   - Linux: 执行 `xdg-open claude-cli://open?q=test`,检查 desktop file 配置
2. **检查 URL 格式**: 完整格式为 `claude-cli://open?q=...&cwd=...&repo=...`
   - `q` 参数: 查询内容,最大 5000 字符 (`MAX_QUERY_LENGTH`)
   - `cwd` 参数: 必须是绝对路径,最大 4096 字符 (`MAX_CWD_LENGTH`)
   - `repo` 参数: 必须符合 `owner/repo` slug 格式
3. **检查安全限制**: URL 中包含控制字符会被拒绝;`cwd` 为相对路径会被拒绝;`repo` 含非法字符会被拒绝
4. **检查注册失败退避**: 如果之前注册失败,系统会进入 24 小时退避期 (`FAILURE_BACKOFF_MS`)——在此期间不会重试注册

### 终端启动排查

如果 DeepLink 触发了但终端没有正确启动:

1. 确认当前系统安装了 `terminalLauncher.ts` 支持的终端
2. macOS 偏好排序: iTerm2 > Ghostty > Kitty > Alacritty > WezTerm > Terminal.app
3. Linux 偏好排序: ghostty > kitty > alacritty > wezterm > gnome-terminal > konsole > ...
4. Windows 偏好排序: Windows Terminal > pwsh > PowerShell > cmd
5. 检查 shell 引号处理: 不同终端使用不同的引号策略 (`shellQuote`/`appleScriptQuote`/`psQuote`/`cmdQuote`)

### 常见陷阱

> **不同平台的注册机制完全不同**: macOS 用 Info.plist + lsregister,Windows 用 Registry,Linux 用 .desktop + xdg-mime。这三套机制不能互相替代,不能用统一的抽象——`registerProtocol.ts` 的 349 行代码中,每个平台都有独立的实现路径。**跨平台发布时必须分别测试每个平台的协议注册**。

> **URL 参数需要正确编码**: 查询内容 (`q` 参数) 中的特殊字符必须经过 URL 编码。控制字符会被安全检查直接拒绝 (防止终端注入)。在构造 DeepLink URL 时,始终使用 `encodeURIComponent()` 处理参数值。

> **背景注册可能静默失败**: `backgroundHousekeeping` 任务在后台静默执行协议注册,失败不会通知用户。如果 DeepLink 不工作,首先检查注册是否成功——特别是在 Linux 上,如果 `xdg-mime` 命令不可用,注册会静默失败。

> **Git 仓库陈旧警告**: Banner 系统会检查 `.git/FETCH_HEAD` 的修改时间,超过 7 天 (`STALE_FETCH_WARN_MS`) 未 fetch 会显示警告。这不影响功能,但提醒用户仓库可能不是最新状态。


---

[← Computer Use](../35-Computer-Use/computer-use.md) | [目录](../README.md) | [Teleport →](../37-Teleport/teleport-system.md)
