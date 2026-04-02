# Sandbox System Architecture Document

> Claude Code v2.1.88 Sandbox Security Isolation System — Complete Technical Reference

---

### Design Philosophy

#### Why Is the Sandbox Optional?

Not all environments support sandboxing — Linux namespaces cannot be nested inside Docker containers, some CI/CD environments lack `sandbox-exec` permissions, and remote development environments may have their own isolation layers. The `failIfUnavailable` configuration option (in `sandboxTypes.ts`) lets users choose the behavior: when set to `true`, an unavailable sandbox causes an error and halts execution (suitable for high-security scenarios); when set to `false`, the system degrades to running without a sandbox (suitable for functionality-first scenarios). `enableWeakerNestedSandbox` and `enableWeakerNetworkIsolation` provide weakened isolation options as compatibility compromises.

#### Why Three Sandbox Types?

Each OS has a different native isolation mechanism — macOS uses `sandbox-exec` (Seatbelt), Linux uses user namespaces + cgroups, and Docker environments use container isolation. Unifying into a single approach is not feasible because the underlying system calls are completely different. The sandbox adapter layer (`sandbox-adapter.ts`) detects the environment at initialization and selects the appropriate implementation, presenting a unified interface to higher layers.

#### Why Allow Granular Permission Configuration?

Different tools require different permissions — Bash may need network access for `npm install`, FileRead only needs filesystem read permission, and the `excludedCommands` list allows specific commands to bypass the sandbox. The existence of `autoAllowBashIfSandboxed` illustrates a pragmatic trade-off: when the sandbox already provides isolation protection, user confirmation prompts can be reduced to improve interaction fluency.

## Sandbox Configuration (sandbox field in settings.json)

```typescript
sandbox: {
  enabled: boolean,                      // Enable sandbox
  failIfUnavailable: boolean,            // Fail vs degrade when unavailable
  allowUnsandboxedCommands: boolean,      // Allow unsandboxed command execution
  network: {...},                         // Network restriction configuration
  filesystem: {...},                      // Filesystem restriction configuration
  ignoreViolations: boolean,             // Ignore violations (do not block execution)
  excludedCommands: string[],            // Excluded commands (bypass sandbox)
  autoAllowBashIfSandboxed: boolean,     // Auto-allow bash in sandboxed mode
  enableWeakerNestedSandbox: boolean,    // Enable weaker nested sandbox
  enableWeakerNetworkIsolation: boolean, // Enable weaker network isolation
  ripgrep: {...}                         // ripgrep-specific configuration
}
```

### Configuration Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `enabled` | boolean | Whether to enable sandbox isolation |
| `failIfUnavailable` | boolean | Behavior when sandbox is unavailable: `true` errors and halts, `false` degrades to running without sandbox |
| `allowUnsandboxedCommands` | boolean | Whether to allow execution of unsandboxed commands |
| `ignoreViolations` | boolean | Whether to ignore sandbox violation reports |
| `excludedCommands` | string[] | Whitelist of commands that bypass the sandbox |
| `autoAllowBashIfSandboxed` | boolean | Automatically approve bash commands in sandboxed mode (no user confirmation required) |
| `enableWeakerNestedSandbox` | boolean | Allow use of a weakened nested sandbox (compatibility option) |
| `enableWeakerNetworkIsolation` | boolean | Use a weakened network isolation policy |

---

## Sandbox Execution (sandbox-adapter.ts)

### Initialization
```
M7.initialize(SK8)  // Asynchronously initialize the sandbox engine
```

### Unavailability Handling
- `failIfUnavailable = true` → Error, block execution
- `failIfUnavailable = false` → Degrade to running without sandbox

### Command Execution Decision
```
shouldUseSandbox()  // Decide whether the current command uses the sandbox
```
Factors considered: sandbox availability, command exclusion list, `dangerouslyDisableSandbox` parameter, etc.

### BashTool Integration
The `dangerouslyDisableSandbox` parameter of `BashTool` can explicitly bypass sandbox protection (requires permission authorization).

---

## Violation Detection

### removeSandboxViolationTags(text)
Removes `<sandbox_violations>` tags from error messages, cleaning internal markers before displaying them to the user.

### Violation Handling Flow
1. Sandbox detects a violation
2. Violation message is formatted
3. Based on the `ignoreViolations` setting, decide whether to block
4. Display to the user (if not ignored)

---

## Network Control

### MITM Proxy
- Uses a man-in-the-middle proxy to intercept network requests
- Blocked requests return: `X-Proxy-Error: blocked-by-allowlist`
- Supports domain allowlist and blocklist mechanisms

### Proxy Socket
```
getMitmSocketPath()  // Get the proxy socket path
```

### Upstream Proxy (src/upstreamproxy/)

#### relay.ts (456 lines)
TCP to WebSocket to CCR tunnel relay.

**Key Implementation Details**:

| Feature | Description |
|---------|-------------|
| Protobuf Encoding | Hand-written varint encode/decode (no external protobuf library dependency) |
| Backpressure Handling | Handles differences between Bun partial writes vs Node buffering |
| Keepalive | 30-second interval pinger to keep connections alive |

---

## Sandbox Permissions in Swarm Mode

In multi-agent Swarm mode, sandbox permissions are passed between workers and leaders via the mailbox system.

### Permission Request
```
sendSandboxPermissionRequestViaMailbox()
```
Worker → Leader: Send a sandbox permission request.

### Permission Response
```
sendSandboxPermissionResponseViaMailbox()
```
Leader → Worker: Return the sandbox permission decision result.

### Flow
1. Worker agent needs to perform a restricted operation
2. Worker sends a permission request to the leader via mailbox
3. Leader evaluates the request and makes a decision
4. Leader returns an allow/deny response via mailbox
5. Worker continues or aborts the operation based on the response

---

## Engineering Practice Guide

### Enabling/Disabling the Sandbox

**Configure the `sandbox` field in `settings.json`:**

```json
{
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": false,
    "autoAllowBashIfSandboxed": true,
    "excludedCommands": ["specific-command"],
    "enableWeakerNestedSandbox": false,
    "enableWeakerNetworkIsolation": false
  }
}
```

**Sandbox Type Selection (auto-detected):**

| Environment | Sandbox Implementation | Description |
|-------------|----------------------|-------------|
| macOS | `sandbox-exec` (Seatbelt) | Native macOS sandbox |
| Linux | user namespace + cgroup | Native Linux isolation |
| Docker | Container isolation | Uses the host container's isolation layer |
| Windows | No native sandbox | `sandbox-exec`/`bwrap` not available |

**failIfUnavailable Behavior:**
- `true` → Error and halt when sandbox is unavailable (high-security scenarios)
- `false` → Degrade to running without sandbox (functionality-first scenarios)

### Debugging Sandbox Violations

**Troubleshooting Steps:**

1. **Check violation messages**: After the sandbox detects a violation, it formats the violation message and decides whether to block based on `ignoreViolations`
2. **Inspect violation tags**: `removeSandboxViolationTags(text)` removes `<sandbox_violations>` tags from error messages — internal markers are cleaned before displaying to the user
3. **Check the command exclusion list**: Commands in `excludedCommands` bypass the sandbox
4. **Check the `dangerouslyDisableSandbox` parameter**: BashTool supports this parameter to explicitly bypass the sandbox (requires permission authorization)
   - Source code `BashTool/prompt.ts` indicates: runs in the sandbox by default; only use `dangerouslyDisableSandbox: true` to retry after a command fails due to sandbox restrictions
   - When `allowUnsandboxedCommands = false`, this parameter is completely ignored

**`shouldUseSandbox()` Decision Factors:**
- Sandbox availability
- Command exclusion list
- `dangerouslyDisableSandbox` parameter
- Various switches in the sandbox configuration

### Custom Sandbox Rules

**Filesystem Rules:**
- Configure allowed/denied directories via `sandbox.filesystem`
- Working directories and project directories are typically on the allowlist

**Network Rules:**
- `sandbox.network` configures network access permissions
- MITM proxy intercepts network requests; blocked requests return `X-Proxy-Error: blocked-by-allowlist`
- `enableWeakerNetworkIsolation` provides weakened network isolation (compatibility compromise)
- The upstream proxy (`relay.ts`) uses a TCP→WebSocket→CCR tunnel relay with hand-written varint encoding and a 30-second keepalive

**Weakening Options (Compatibility Compromises):**
- `enableWeakerNestedSandbox` — Use a weakened nested sandbox in environments that already have an isolation layer
- `enableWeakerNetworkIsolation` — Use a weakened network isolation policy

### Sandbox Permissions in Swarm Mode

**In multi-agent mode:**
1. Workers request permissions via `sendSandboxPermissionRequestViaMailbox()`
2. The leader returns decisions via `sendSandboxPermissionResponseViaMailbox()`
3. Permissions are passed through the mailbox system to ensure consistent security decisions across agents

### Common Pitfalls

| Pitfall | Details | Solution |
|---------|---------|----------|
| Cannot nest namespace sandbox inside Docker | Linux namespaces are unavailable inside Docker containers | Use `enableWeakerNestedSandbox` or downgrade with `failIfUnavailable: false` |
| Sandbox may block normal tool operations | Commands requiring network access such as `npm install` and `pip install` may be intercepted | Configure `excludedCommands` or allow network access |
| Security trade-off of `autoAllowBashIfSandboxed` | Reduces user confirmation prompts when sandbox already provides isolation | Suitable for development environments, but use with caution in production/security-sensitive environments |
| No native sandbox on Windows | `sandbox-exec`/`bwrap` are not available on Windows natively | Source code comments confirm that the PowerShell tool has no sandbox on native Windows |
| Sandbox violations may be silently ignored | When `ignoreViolations: true`, violations do not block execution | Use this option only for debugging/development scenarios |
| Each command is evaluated for sandboxing independently | Even after recently using `dangerouslyDisableSandbox`, subsequent commands still default to sandbox mode | Source code prompt explicitly requires "Treat each command individually" |


---

[← LSP Integration](../23-LSP集成/lsp-integration-en.md) | [Index](../README_EN.md) | [Git & GitHub →](../25-Git与GitHub/git-github-en.md)
