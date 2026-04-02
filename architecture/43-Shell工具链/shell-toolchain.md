# Shell Toolchain Architecture

## 设计理念

### 设计理念：为什么解析 Bash AST 而不是正则匹配？

Bash 语法的复杂性远超正则表达式的处理能力：

1. **管道嵌套** -- `echo $(cat file | grep pattern | awk '{print $1}')` 包含命令替换、管道链、引号嵌套，正则无法正确识别边界
2. **Here Document** -- `<<EOF ... EOF` 语法中的内容不应被当作命令解析，需要 AST 级别的语法理解（由 `heredoc.ts` 专门处理）
3. **安全分类的准确性** -- 命令安全检查需要精确识别命令名和参数，正则误判可能导致危险命令被放行或安全命令被阻断。源码使用 `treeSitterAnalysis.ts` 提供基于 Tree-sitter 的深度语法分析
4. **多层解析架构** -- `bashParser.ts`（基础 AST）→ `bashPipeCommand.ts`（管道分析）→ `treeSitterAnalysis.ts`（语义理解），每层解决不同粒度的问题

### 设计理念：为什么同时支持 PowerShell？

Windows 用户的默认 shell 是 PowerShell，不支持等于排除 Windows 平台。源码中 `src/utils/powershell/` 提供了独立的 PowerShell 安全分析体系：

- `dangerousCmdlets.ts` 维护了 6 大类危险 cmdlet 列表（`FILEPATH_EXECUTION_CMDLETS`、`DANGEROUS_SCRIPT_BLOCK_CMDLETS`、`MODULE_LOADING_CMDLETS`、`SHELLS_AND_SPAWNERS`、`NETWORK_CMDLETS`、`ALIAS_HIJACK_CMDLETS`），覆盖了 PowerShell 特有的攻击面
- `SHELL_TYPES = ['bash', 'powershell']` 在 Shell Provider 抽象层中作为并列的一等公民
- `resolveDefaultShell.ts` 根据操作系统自动选择合适的默认 shell

---

## Bash Utilities

`src/utils/bash/` 目录包含 20+ 个文件，提供完整的 Bash 命令分析和处理能力。

### CommandSpec 类型

```typescript
type CommandSpec = {
  name: string           // 命令名称
  description: string    // 命令描述
  subcommands: ...       // 子命令定义
  args: ...              // 位置参数定义
  options: ...           // 选项/标志定义
}
```

### 命令规格获取

- **getCommandSpec()**: 获取命令的规格定义。使用 memoized LRU 缓存优化性能，首先检查本地 specs 目录，如未命中则从 `@withfig/autocomplete` 包获取。

### AST 解析

- **bashParser.ts**: Bash 脚本的 AST（抽象语法树）解析器，将命令字符串解析为结构化表示
- **bashPipeCommand.ts**: 管道命令（`|`）的解析和分析，处理管道链中每个命令的独立解析
- **heredoc.ts**: Here Document（`<<EOF`）语法的解析和处理
- **treeSitterAnalysis.ts**: 基于 Tree-sitter 的深度语法分析，提供更精确的 AST 节点识别和语义理解

### Shell 引号处理

- **shellQuote.ts**: Shell 引号/转义处理工具
- **shellQuoting.ts**: 扩展的引号处理功能，支持多种引号策略

### Shell 快照

- **ShellSnapshot.ts**: Shell 环境快照，捕获和恢复 shell 状态

### 本地命令规格

系统内置了以下常用命令的本地规格定义: `alias`, `nohup`, `pyright`, `sleep`, `srun`, `time`, `timeout`

## PowerShell Utilities

`src/utils/powershell/` 提供 PowerShell 命令的安全分析和权限控制。

### 危险 Cmdlet 分类

系统维护了多个危险 cmdlet 列表，用于命令执行前的安全检查:

- **FILEPATH_EXECUTION_CMDLETS**: 通过文件路径执行代码的 cmdlet
  - `invoke-command`, `start-job` 等

- **DANGEROUS_SCRIPT_BLOCK_CMDLETS**: 可执行任意代码块的 10 个 cmdlet
  - 这些 cmdlet 接受 ScriptBlock 参数，可以执行任意 PowerShell 代码

- **MODULE_LOADING_CMDLETS**: 模块加载相关
  - `import-module`, `install-module` 等
  - 可能引入不受信任的代码

- **SHELLS_AND_SPAWNERS**: Shell 和进程启动器
  - `pwsh`, `cmd`, `bash`, `wsl`, `start-process` 等
  - 可以绕过 PowerShell 的安全限制

- **NETWORK_CMDLETS**: 网络请求相关
  - `invoke-webrequest`, `invoke-restmethod`
  - 可能导致数据泄露

- **ALIAS_HIJACK_CMDLETS**: 别名劫持相关
  - `set-alias`, `set-variable`
  - 可能篡改现有命令行为

### 解析器

- **parsePowerShellCommand()**: 解析 PowerShell 命令字符串，提取命令名、参数和管道结构
- **getAllCommands()**: 从解析结果中提取所有命令（包括管道链中的每个命令）

### 权限前缀提取

- **extractPrefixFromElement()**: 从 PowerShell AST 元素中提取权限检查所需的命令前缀，用于判断命令是否需要用户确认。

## Shell Provider Abstraction

`src/utils/shell/` 提供了跨平台的 shell 抽象层。

### ShellProvider 接口

```typescript
interface ShellProvider {
  type: string                    // shell 类型标识
  shellPath: string               // shell 可执行文件路径
  detached: boolean               // 是否分离模式
  buildExecCommand(): ...         // 构建执行命令
  getSpawnArgs(): ...             // 获取 spawn 参数
  getEnvironmentOverrides(): ...  // 获取环境变量覆盖
}
```

### Shell 类型

```typescript
SHELL_TYPES = ['bash', 'powershell']
```

### Provider 实现

- **bashProvider.ts**: Bash shell provider
  - 会话环境管理: 设置和维护 shell 会话的环境变量
  - eval-wrap: 将命令包裹在 `eval` 中执行，确保正确的 shell 展开
  - pwd 跟踪: 追踪工作目录变化，确保后续命令在正确目录执行
  - TMUX socket 隔离: 在 TMUX 环境中隔离 socket，避免会话冲突
  - Windows null redirect 重写: 将 `/dev/null` 重写为 Windows 兼容的 `NUL`

- **powershellProvider.ts**: PowerShell provider，处理 PowerShell 特有的命令构建和环境设置

### 辅助模块

- **resolveDefaultShell.ts**: Shell 检测。根据操作系统和环境自动选择合适的默认 shell
- **readOnlyCommandValidation.ts**: 安全检查。验证命令是否为只读操作，用于权限控制决策
- **outputLimits.ts**: 输出大小限制。防止命令输出过大导致内存问题，对超限输出进行截断

## 工程实践

### Bash 命令安全分类的扩展

- 在 `bashSecurity.ts` 中添加新的危险模式检测规则——该文件导入并使用 `treeSitterAnalysis.ts` 的 AST 分析结果进行安全判断
- Auto 模式下的两阶段分类器（Stage 1: 正则快速路径 → Stage 2: AI 慢路径）可以在 Stage 1 中添加新的正则模式来短路常见的危险命令
- 新增的危险模式需要同时更新对应的单元测试

### PowerShell 解析的局限

- 某些 PowerShell 特殊语法（如 DSC 资源、自定义 class 定义、高级函数属性）可能无法被 `parsePowerShellCommand()` 正确解析
- 解析失败时会 fallback 到保守策略——将整个命令视为需要用户确认
- `extractPrefixFromElement()` 用于从 AST 中提取权限检查所需的命令前缀，如果提取失败则需要完整的用户确认流程
- Windows null redirect 重写（`/dev/null` → `NUL`）由 `bashProvider.ts` 处理，在 PowerShell 环境下需要特别注意这类跨平台差异

---

## Git Integration

`src/utils/git/` 提供全面的 Git 仓库操作支持。

### 配置解析

- **gitConfigParser.ts**: `.git/config` 文件解析器，支持 Git 配置中的转义序列处理（如 `\t`, `\n`, `\\` 等）。

### 文件系统操作

**gitFilesystem.ts**（约 700 行）是 Git 文件系统操作的核心模块:

- **resolveGitDir()**: 解析 `.git` 目录的实际路径，正确处理 worktree（工作树文件中的 `gitdir:` 引用）和 submodule（子模块的嵌套 `.git` 目录）。

- **isSafeRefName()**: 验证 Git 引用名安全性:
  - 阻止路径遍历（`..`）
  - 阻止参数注入（`-` 开头）
  - 阻止 shell 元字符（`$`, `` ` ``, `|`, `;` 等）

- **isValidGitSha()**: 验证 Git SHA 格式:
  - SHA-1: 40 个十六进制字符
  - SHA-256: 64 个十六进制字符

- **readGitHead()**: 解析 HEAD 文件:
  - 分支引用: `ref: refs/heads/main`
  - 分离头指针: 直接 SHA 值

- **GitFileWatcher class**: 文件监视器
  - 监视 `.git/HEAD`, `config`, 分支引用文件的变化
  - 使用缓存 + dirty marking 策略: 文件变化时标记为 dirty，下次读取时刷新缓存

- **缓存查询函数**:
  - `getCachedBranch()`: 获取缓存的当前分支名
  - `getCachedHead()`: 获取缓存的 HEAD 引用
  - `getCachedRemoteUrl()`: 获取缓存的远程仓库 URL
  - `getCachedDefaultBranch()`: 获取缓存的默认分支名

- **仓库状态查询**:
  - `isShallowClone()`: 检测是否为浅克隆
  - `getWorktreeCountFromFs()`: 从文件系统获取 worktree 数量

### Gitignore 处理

- **gitignore.ts**:
  - `isPathGitignored()`: 检查指定路径是否被 gitignore 规则忽略
  - `addFileGlobRuleToGitignore()`: 向 `.gitignore` 文件添加 glob 规则

## GitHub Integration

`src/utils/github/` 提供 GitHub CLI 集成:

- **getGhAuthStatus()**: 检测 GitHub CLI 认证状态
  - 返回值: `'authenticated' | 'not_authenticated' | 'not_installed'`
  - 实现方式: 调用 `gh auth token` 命令（无网络请求），通过退出码和输出判断状态
  - 高效: 不发起网络调用，仅检查本地 token 状态

## DXT Extension System

`src/utils/dxt/` 实现了 DXT（Desktop Extension）扩展包的处理:

### Manifest 验证

- **validateManifest()**: 使用 Zod schema 验证扩展清单文件，schema 定义来自 `@anthropic-ai/mcpb` 包。验证包括必需字段、类型约束、值范围等。

### ID 生成

- **generateExtensionId()**: 基于作者名和扩展名生成清理后的唯一标识符，移除特殊字符并标准化格式。

### Zip 处理

对 DXT 扩展包（zip 格式）实施严格的安全限制:

| 限制项 | 值 |
|--------|------|
| 单文件最大体积 | 512 MB |
| 总解压体积 | 1024 MB |
| 最大文件数 | 100,000 |
| 最大压缩比 | 50:1 |

### 安全检查

- **isPathSafe()**: 路径安全验证，拒绝包含路径遍历（`..`）的条目，防止 zip slip 攻击
- **Zip bomb 检测**: 通过压缩比检查（50:1 上限）检测潜在的 zip 炸弹，防止解压时耗尽磁盘空间和内存


---

[← 代价追踪](../42-代价追踪/cost-tracking.md) | [目录](../README.md) | [Screens 组件 →](../44-Screens组件/screens-components.md)
