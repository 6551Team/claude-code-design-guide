# DeepLink System Architecture

## Protocol

The DeepLink system allows external applications (browsers, editors, etc.) to directly launch the Claude CLI and pass context through a custom protocol.

### Design Philosophy

#### Why the claude-cli:// protocol?

This enables external applications (browsers, IDEs, documents) to directly launch Claude Code and pass context — seamless integration. Unlike command-line arguments, URLs can be embedded in web pages, documents, and Slack messages for direct clicking, so users don't need to open a terminal and type commands manually. The source file `parseDeepLink.ts` supports three parameters: `q` (query), `cwd` (working directory), and `repo` (repository), covering the complete context-passing needs for "ask this question in this repository."

#### Why register on 3 platforms?

macOS (Info.plist + `CFBundleURLTypes` + `lsregister`), Linux (`.desktop` file + `xdg-mime`), and Windows (Registry `HKEY_CURRENT_USER\Software\Classes`) each have different protocol registration mechanisms. The source file `registerProtocol.ts` (349 lines) implements dedicated registration logic for each platform. This is a difference that cannot be abstracted away — URL scheme registration on each platform is an OS-level API that must be adapted individually. Automatic registration is triggered by the `backgroundHousekeeping` task, with a 24-hour backoff after failure so it does not disrupt the user's workflow.

#### Why URL parsing instead of command-line arguments?

The security design in the source code demonstrates the careful thought behind the URL approach: control character rejection prevents terminal injection; `cwd` requires an absolute path to prevent path traversal; `repo` slug format validation (`owner/repo`) limits the injection surface. The URL format also supports multi-strategy shell quoting — `shellQuote()` (POSIX), `appleScriptQuote()` (macOS osascript), `psQuote()` (PowerShell), `cmdQuote()` (cmd.exe) — ensuring arguments can be safely passed in any terminal environment.

```
DEEP_LINK_PROTOCOL = 'claude-cli'
```

**URI format**:
```
claude-cli://open?q=...&cwd=...&repo=...
```

- `q`: Query/prompt content, passed to Claude as the initial message
- `cwd`: Working directory, specifies the working path for the Claude session
- `repo`: Repository identifier (slug format), used for context location

**Security measures**:
- **Control character rejection**: Rejects requests containing control characters in the URI to prevent terminal injection attacks
- **cwd absolute path requirement**: The `cwd` parameter must be an absolute path; relative paths are rejected to prevent path traversal
- **repo slug validation**: Validates that the repo parameter conforms to slug format (`owner/repo`), rejecting illegal characters

**Length limits**:
```typescript
MAX_QUERY_LENGTH = 5000    // Maximum length for query content
MAX_CWD_LENGTH = 4096      // Maximum length for working directory path
```

## Registration

`registerProtocol.ts` (approximately 349 lines) is responsible for registering the `claude-cli://` protocol handler on each operating system.

### macOS

Creates a `.app` bundle in the `~/Applications` directory:
- Generates `Info.plist` containing a `CFBundleURLTypes` declaration that binds the `claude-cli` scheme to the application
- Creates a symlink pointing to the CLI executable
- Calls `lsregister` to register the URL scheme with Launch Services

### Linux

Uses the XDG desktop specification:
- Creates a `.desktop` file in the `$XDG_DATA_HOME/applications` directory (default: `~/.local/share/applications`)
- Sets the default handler for `x-scheme-handler/claude-cli` via `xdg-mime`

### Windows

Writes registry keys:
- Path: `HKEY_CURRENT_USER\Software\Classes\claude-cli`
- Sets the `URL Protocol` value to indicate this is a protocol handler
- Sets the CLI executable path under the `shell\open\command` subkey

### Automatic Registration

Protocol registration is automatically triggered by the `backgroundHousekeeping` task:
- **Failure backoff**: `FAILURE_BACKOFF_MS` sets a 24-hour backoff period; after a registration failure, retries will not occur frequently
- Executes silently in the background without disrupting the user's workflow

## Terminal Launch

`terminalLauncher.ts` (approximately 558 lines) is responsible for launching a new Claude CLI session in the user's preferred terminal.

### macOS Terminal Support (in preference order)

1. iTerm2
2. Ghostty
3. Kitty
4. Alacritty
5. WezTerm
6. Terminal.app (system default)

### Linux Terminal Support

ghostty, kitty, alacritty, wezterm, gnome-terminal, konsole, xfce4-terminal, mate-terminal, tilix, xterm

### Windows Terminal Support

Windows Terminal, pwsh (PowerShell 7+), PowerShell (Windows PowerShell), cmd

### Shell Quoting

Different terminals and shells require different quoting and escaping strategies:

- **shellQuote()**: Generic POSIX shell quoting, wraps in single quotes and escapes internal single quotes
- **appleScriptQuote()**: AppleScript string quoting, used for controlling terminals on macOS via `osascript`
- **psQuote()**: PowerShell string quoting, handles PowerShell special characters and escape sequences
- **cmdQuote()**: Windows cmd.exe quoting, handles special characters such as `%`, `^`, and `&`

### Process Detachment

- **spawnDetached()**: Launches the terminal as an independent process, ensuring the newly launched Claude CLI session is fully decoupled from the trigger source so that the parent process exiting does not affect the child process.

## Banner

`banner.ts` provides banner information display when launched via DeepLink:

```typescript
STALE_FETCH_WARN_MS = 7 days  // Show warning when fetch hasn't occurred in 7 days
LONG_PREFILL_THRESHOLD = 1000  // Threshold when prefill content exceeds 1000 characters
```

- **Git status detection**: Reads the modification time (mtime) of `.git/FETCH_HEAD`, supporting worktree scenarios. When the last fetch was more than 7 days ago, a warning is displayed in the banner to remind the user that the repository may not be up to date.
- **Long content hint**: When the query content passed via DeepLink exceeds `LONG_PREFILL_THRESHOLD` (1000 characters), a content length hint is displayed.

## Terminal Preference

The terminal preference system (macOS only):

- **Capture**: Reads the `TERM_PROGRAM` environment variable and maps it to the corresponding terminal application identifier
- **Persistence**: Persists the user's terminal preference so that subsequent DeepLink launches will prefer that terminal
- **Mapping**: Maps `TERM_PROGRAM` values (e.g., `iTerm.app`, `ghostty`, `Apple_Terminal`) to internal terminal identifiers

---

## Engineering Practice Guide

### Registering the Protocol Handler

Different platforms require different registration approaches:

1. **macOS** (automatic):
   - The system creates a `.app` bundle under `~/Applications/` containing the `CFBundleURLTypes` declaration in `Info.plist`
   - Calls `lsregister` to register with Launch Services
   - Typically completed automatically by `backgroundHousekeeping`; no manual action required
2. **Windows** (requires Registry):
   - Writes the `HKEY_CURRENT_USER\Software\Classes\claude-cli` registry key
   - Sets the `URL Protocol` value to mark it as a protocol handler
   - Sets the CLI executable path under the `shell\open\command` subkey
3. **Linux** (requires desktop file):
   - Creates a `.desktop` file in `$XDG_DATA_HOME/applications` (default: `~/.local/share/applications`)
   - Sets the default handler for `x-scheme-handler/claude-cli` via `xdg-mime`

### Debugging Links That Don't Open

1. **Check whether protocol registration is correct**:
   - macOS: Run `open claude-cli://open?q=test` in the terminal to confirm whether Claude CLI launches
   - Windows: Enter `claude-cli://open?q=test` in the browser address bar to check whether the protocol handler is triggered
   - Linux: Run `xdg-open claude-cli://open?q=test` to check the desktop file configuration
2. **Check URL format**: The full format is `claude-cli://open?q=...&cwd=...&repo=...`
   - `q` parameter: Query content, maximum 5000 characters (`MAX_QUERY_LENGTH`)
   - `cwd` parameter: Must be an absolute path, maximum 4096 characters (`MAX_CWD_LENGTH`)
   - `repo` parameter: Must conform to the `owner/repo` slug format
3. **Check security restrictions**: URLs containing control characters will be rejected; a relative `cwd` path will be rejected; `repo` containing illegal characters will be rejected
4. **Check registration failure backoff**: If registration previously failed, the system enters a 24-hour backoff period (`FAILURE_BACKOFF_MS`) — registration will not be retried during this period

### Terminal Launch Troubleshooting

If a DeepLink was triggered but the terminal did not launch correctly:

1. Confirm that one of the terminals supported by `terminalLauncher.ts` is installed on the current system
2. macOS preference order: iTerm2 > Ghostty > Kitty > Alacritty > WezTerm > Terminal.app
3. Linux preference order: ghostty > kitty > alacritty > wezterm > gnome-terminal > konsole > ...
4. Windows preference order: Windows Terminal > pwsh > PowerShell > cmd
5. Check shell quoting: different terminals use different quoting strategies (`shellQuote`/`appleScriptQuote`/`psQuote`/`cmdQuote`)

### Common Pitfalls

> **Registration mechanisms differ completely across platforms**: macOS uses Info.plist + lsregister, Windows uses the Registry, and Linux uses .desktop + xdg-mime. These three mechanisms cannot substitute for one another and cannot be unified into a single abstraction — in the 349 lines of `registerProtocol.ts`, each platform has its own independent implementation path. **When releasing cross-platform, you must test protocol registration on each platform separately.**

> **URL parameters must be properly encoded**: Special characters in the query content (`q` parameter) must be URL-encoded. Control characters will be rejected outright by the security check (to prevent terminal injection). When constructing a DeepLink URL, always use `encodeURIComponent()` to process parameter values.

> **Background registration may fail silently**: The `backgroundHousekeeping` task performs protocol registration silently in the background; failures are not reported to the user. If DeepLink is not working, first check whether registration succeeded — especially on Linux, if the `xdg-mime` command is unavailable, registration will fail silently.

> **Git repository stale warning**: The banner system checks the modification time of `.git/FETCH_HEAD`; if more than 7 days (`STALE_FETCH_WARN_MS`) have passed since the last fetch, a warning is displayed. This does not affect functionality, but reminds the user that the repository may not be up to date.


---

[← Computer Use](../35-Computer-Use/computer-use-en.md) | [Index](../README_EN.md) | [Teleport →](../37-Teleport/teleport-system-en.md)
