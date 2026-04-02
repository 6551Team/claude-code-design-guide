# Shell Toolchain Architecture

## Design Philosophy

### Design Philosophy: Why Parse Bash AST Instead of Regex Matching?

The complexity of Bash syntax far exceeds what regular expressions can handle:

1. **Nested pipelines** -- `echo $(cat file | grep pattern | awk '{print $1}')` contains command substitution, pipeline chains, and nested quotes; regex cannot correctly identify boundaries
2. **Here Documents** -- content inside `<<EOF ... EOF` syntax must not be parsed as commands; this requires AST-level syntactic understanding (handled specifically by `heredoc.ts`)
3. **Accuracy of security classification** -- command security checks require precise identification of command names and arguments; regex misclassification can allow dangerous commands through or block safe ones. The source uses `treeSitterAnalysis.ts` to provide deep syntactic analysis based on Tree-sitter
4. **Multi-layer parsing architecture** -- `bashParser.ts` (basic AST) → `bashPipeCommand.ts` (pipeline analysis) → `treeSitterAnalysis.ts` (semantic understanding), each layer solving problems at a different granularity

### Design Philosophy: Why Support PowerShell as Well?

The default shell for Windows users is PowerShell; not supporting it means excluding the Windows platform. The source provides an independent PowerShell security analysis system under `src/utils/powershell/`:

- `dangerousCmdlets.ts` maintains a list of 6 categories of dangerous cmdlets (`FILEPATH_EXECUTION_CMDLETS`, `DANGEROUS_SCRIPT_BLOCK_CMDLETS`, `MODULE_LOADING_CMDLETS`, `SHELLS_AND_SPAWNERS`, `NETWORK_CMDLETS`, `ALIAS_HIJACK_CMDLETS`), covering the attack surface unique to PowerShell
- `SHELL_TYPES = ['bash', 'powershell']` are treated as parallel first-class citizens in the Shell Provider abstraction layer
- `resolveDefaultShell.ts` automatically selects the appropriate default shell based on the operating system

---

## Bash Utilities

The `src/utils/bash/` directory contains 20+ files providing complete Bash command analysis and processing capabilities.

### CommandSpec Type

```typescript
type CommandSpec = {
  name: string           // command name
  description: string    // command description
  subcommands: ...       // subcommand definitions
  args: ...              // positional argument definitions
  options: ...           // option/flag definitions
}
```

### Command Spec Retrieval

- **getCommandSpec()**: Retrieves the specification definition for a command. Uses a memoized LRU cache for performance optimization, first checking the local specs directory, then falling back to the `@withfig/autocomplete` package on a cache miss.

### AST Parsing

- **bashParser.ts**: AST (Abstract Syntax Tree) parser for Bash scripts; parses command strings into structured representations
- **bashPipeCommand.ts**: Parsing and analysis of pipeline commands (`|`); handles independent parsing of each command in a pipeline chain
- **heredoc.ts**: Parsing and handling of Here Document (`<<EOF`) syntax
- **treeSitterAnalysis.ts**: Deep syntactic analysis based on Tree-sitter, providing more precise AST node identification and semantic understanding

### Shell Quote Handling

- **shellQuote.ts**: Shell quoting/escaping utility
- **shellQuoting.ts**: Extended quoting functionality supporting multiple quoting strategies

### Shell Snapshot

- **ShellSnapshot.ts**: Shell environment snapshot; captures and restores shell state

### Local Command Specs

The system has built-in local spec definitions for the following common commands: `alias`, `nohup`, `pyright`, `sleep`, `srun`, `time`, `timeout`

## PowerShell Utilities

`src/utils/powershell/` provides security analysis and permission control for PowerShell commands.

### Dangerous Cmdlet Categories

The system maintains multiple dangerous cmdlet lists used for security checks before command execution:

- **FILEPATH_EXECUTION_CMDLETS**: Cmdlets that execute code via file paths
  - `invoke-command`, `start-job`, etc.

- **DANGEROUS_SCRIPT_BLOCK_CMDLETS**: 10 cmdlets that can execute arbitrary code blocks
  - These cmdlets accept a ScriptBlock parameter, allowing execution of arbitrary PowerShell code

- **MODULE_LOADING_CMDLETS**: Module loading related
  - `import-module`, `install-module`, etc.
  - May introduce untrusted code

- **SHELLS_AND_SPAWNERS**: Shell and process spawners
  - `pwsh`, `cmd`, `bash`, `wsl`, `start-process`, etc.
  - Can bypass PowerShell's security restrictions

- **NETWORK_CMDLETS**: Network request related
  - `invoke-webrequest`, `invoke-restmethod`
  - May cause data leakage

- **ALIAS_HIJACK_CMDLETS**: Alias hijacking related
  - `set-alias`, `set-variable`
  - May tamper with existing command behavior

### Parser

- **parsePowerShellCommand()**: Parses a PowerShell command string, extracting command names, arguments, and pipeline structure
- **getAllCommands()**: Extracts all commands from the parsed result (including each command in a pipeline chain)

### Permission Prefix Extraction

- **extractPrefixFromElement()**: Extracts the command prefix needed for permission checks from a PowerShell AST element; used to determine whether a command requires user confirmation.

## Shell Provider Abstraction

`src/utils/shell/` provides a cross-platform shell abstraction layer.

### ShellProvider Interface

```typescript
interface ShellProvider {
  type: string                    // shell type identifier
  shellPath: string               // path to the shell executable
  detached: boolean               // whether detached mode is used
  buildExecCommand(): ...         // build the execution command
  getSpawnArgs(): ...             // get spawn arguments
  getEnvironmentOverrides(): ...  // get environment variable overrides
}
```

### Shell Types

```typescript
SHELL_TYPES = ['bash', 'powershell']
```

### Provider Implementations

- **bashProvider.ts**: Bash shell provider
  - Session environment management: sets and maintains environment variables for the shell session
  - eval-wrap: wraps commands inside `eval` for execution to ensure correct shell expansion
  - pwd tracking: tracks working directory changes to ensure subsequent commands execute in the correct directory
  - TMUX socket isolation: isolates sockets in TMUX environments to avoid session conflicts
  - Windows null redirect rewriting: rewrites `/dev/null` to the Windows-compatible `NUL`

- **powershellProvider.ts**: PowerShell provider; handles PowerShell-specific command construction and environment setup

### Helper Modules

- **resolveDefaultShell.ts**: Shell detection. Automatically selects the appropriate default shell based on the operating system and environment
- **readOnlyCommandValidation.ts**: Security check. Validates whether a command is a read-only operation; used in permission control decisions
- **outputLimits.ts**: Output size limiting. Prevents excessively large command output from causing memory issues by truncating output that exceeds the limit

## Engineering Practices

### Extending Bash Command Security Classification

- Add new dangerous pattern detection rules in `bashSecurity.ts` -- this file imports and uses AST analysis results from `treeSitterAnalysis.ts` for security decisions
- The two-stage classifier used in Auto mode (Stage 1: regex fast path → Stage 2: AI slow path) allows new regex patterns to be added in Stage 1 to short-circuit common dangerous commands
- Newly added dangerous patterns must also update the corresponding unit tests

### Limitations of PowerShell Parsing

- Certain PowerShell special syntax (such as DSC resources, custom class definitions, and advanced function attributes) may not be correctly parsed by `parsePowerShellCommand()`
- On parse failure, the system falls back to a conservative strategy -- treating the entire command as requiring user confirmation
- `extractPrefixFromElement()` is used to extract the command prefix needed for permission checks from the AST; if extraction fails, a full user confirmation flow is required
- Windows null redirect rewriting (`/dev/null` → `NUL`) is handled by `bashProvider.ts`; special attention is needed for this type of cross-platform difference in PowerShell environments

---

## Git Integration

`src/utils/git/` provides comprehensive Git repository operation support.

### Configuration Parsing

- **gitConfigParser.ts**: `.git/config` file parser; supports handling of escape sequences in Git configuration (such as `\t`, `\n`, `\\`, etc.).

### File System Operations

**gitFilesystem.ts** (approximately 700 lines) is the core module for Git file system operations:

- **resolveGitDir()**: Resolves the actual path of the `.git` directory, correctly handling worktrees (`gitdir:` references in worktree files) and submodules (nested `.git` directories in submodules).

- **isSafeRefName()**: Validates Git reference name safety:
  - Blocks path traversal (`..`)
  - Blocks argument injection (leading `-`)
  - Blocks shell metacharacters (`$`, `` ` ``, `|`, `;`, etc.)

- **isValidGitSha()**: Validates Git SHA format:
  - SHA-1: 40 hexadecimal characters
  - SHA-256: 64 hexadecimal characters

- **readGitHead()**: Parses the HEAD file:
  - Branch reference: `ref: refs/heads/main`
  - Detached HEAD: direct SHA value

- **GitFileWatcher class**: File watcher
  - Monitors changes to `.git/HEAD`, `config`, and branch reference files
  - Uses a cache + dirty marking strategy: marks the cache as dirty when a file changes, refreshing it on the next read

- **Cached query functions**:
  - `getCachedBranch()`: Gets the cached current branch name
  - `getCachedHead()`: Gets the cached HEAD reference
  - `getCachedRemoteUrl()`: Gets the cached remote repository URL
  - `getCachedDefaultBranch()`: Gets the cached default branch name

- **Repository state queries**:
  - `isShallowClone()`: Detects whether the repository is a shallow clone
  - `getWorktreeCountFromFs()`: Gets the worktree count from the file system

### Gitignore Handling

- **gitignore.ts**:
  - `isPathGitignored()`: Checks whether a given path is ignored by gitignore rules
  - `addFileGlobRuleToGitignore()`: Adds a glob rule to the `.gitignore` file

## GitHub Integration

`src/utils/github/` provides GitHub CLI integration:

- **getGhAuthStatus()**: Detects GitHub CLI authentication status
  - Return values: `'authenticated' | 'not_authenticated' | 'not_installed'`
  - Implementation: calls the `gh auth token` command (no network request), determining status from the exit code and output
  - Efficient: does not make network calls; only checks local token status

## DXT Extension System

`src/utils/dxt/` implements handling of DXT (Desktop Extension) extension packages:

### Manifest Validation

- **validateManifest()**: Validates the extension manifest file using a Zod schema; the schema definition comes from the `@anthropic-ai/mcpb` package. Validation covers required fields, type constraints, value ranges, and more.

### ID Generation

- **generateExtensionId()**: Generates a sanitized unique identifier based on the author name and extension name; removes special characters and normalizes the format.

### Zip Handling

Strict security limits are enforced on DXT extension packages (zip format):

| Limit | Value |
|-------|-------|
| Maximum size per file | 512 MB |
| Total decompressed size | 1024 MB |
| Maximum file count | 100,000 |
| Maximum compression ratio | 50:1 |

### Security Checks

- **isPathSafe()**: Path safety validation; rejects entries containing path traversal (`..`) to prevent zip slip attacks
- **Zip bomb detection**: Detects potential zip bombs via compression ratio checking (50:1 upper limit), preventing disk space and memory exhaustion during extraction


---

[← Cost Tracking](../42-代价追踪/cost-tracking-en.md) | [Index](../README_EN.md) | [Screens Components →](../44-Screens组件/screens-components-en.md)
