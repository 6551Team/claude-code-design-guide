# Configuration System Architecture Document

> Claude Code v2.1.88 Configuration System Complete Technical Reference

---

## 5-Level Configuration Priority (Low → High)

Configurations are loaded by priority from low to high, with higher priority overriding lower priority:

| Priority | Source Name | Path | Description |
|----------|-------------|------|-------------|
| 1 (Lowest) | **policySettings** | `/etc/claude/managed-settings.json` + `/etc/claude/managed-settings.d/*.json` | Enterprise policy settings, files under `managed-settings.d/` loaded in alphabetical order |
| 2 | **userSettings** | `~/.claude/settings.json` or `~/.claude/cowork_settings.json` | User-level settings, uses cowork variant when `--cowork` flag is present |
| 3 | **projectSettings** | `./.claude/settings.json` | Project-level settings, committed to version control |
| 4 | **localSettings** | `./.claude/settings.local.json` | Local settings, gitignored, not committed |
| 5 (Highest) | **flagSettings** | `--settings` flag | Command-line override, supports path or inline JSON |

### Design Philosophy

#### Why 5 Priority Levels?

This encodes organizational governance into software: `policySettings` (enterprise security policies, CISO enforced) > `flagSettings` (CLI flags, ops override) > `localSettings` (personal local config, gitignored) > `projectSettings` (team conventions, committed to version control) > `userSettings` (personal preferences). Each level corresponds to a real-world decision-making role and scope. Enterprises can enforce security baselines for all developers through `/etc/claude/managed-settings.json`, while developers can still customize their preferences on top of this baseline.

#### Why Can policySettings Never Be Disabled?

Security is non-negotiable principle. In the source code, `allowedSettingSources` always includes `'policySettings'` during initialization (`bootstrap/state.ts`), and enterprise policy settings are loaded through MDM and `/etc/claude/managed-settings.json`, unaffected by user operations. If developers could bypass enterprise security policies (e.g., disable code review rules or allow unauthorized MCP servers), the entire security model would collapse. This embodies "defense in depth".

#### Why Support Hot Reload?

Users shouldn't need to restart after modifying settings during a session—this is a basic UX requirement for CLI tools. In the source code, `settingsChangeDetector` monitors changes to all settings files and triggers a series of updates through `applySettingsChange()`, including permission context reload, Hook configuration reload, environment variable reload, etc., using a debounce mechanism to avoid frequent reloads. Developers can edit `.claude/settings.json` in one terminal and have Claude Code take effect immediately in another terminal.

---

## Merge Strategy

- Uses `mergeWith()` + `settingsMergeCustomizer` for deep merging
- Higher priority overrides lower priority fields with the same name
- **Permission Rule Special Filtering**: Permission rules of type `allow` / `soft_deny` / `environment` have independent filtering and merging logic

---

## Core Functions (src/utils/settings/settings.ts)

### Main Loading Entry Point
```
getInitialSettings(): Settings
```
Main loading function, loads all configuration sources in priority order and merges them.

### Parsers
```
parseSettingsFile(path): ParsedSettings       // Core parser (with cache)
parseSettingsFileUncached(path): ParsedSettings // File read + JSON parse + Zod validation
```

### Policy Settings
```
loadManagedFileSettings(): ManagedSettings
```
Loads policy settings files under `/etc/claude/`.

### Paths and Sources
```
getSettingsFilePathForSource(source): string   // Returns file path for each source
getSettingsForSource(source): Settings         // Gets settings for a single source
```

### Error Handling
```
getSettingsWithErrors(): { settings, errors }
```
Returns settings object and validation error list.

### Cache Management
```
resetSettingsCache(): void
```
Invalidates cache, forces reload from files on next settings retrieval.

---

## Settings Schema (src/utils/settings/types.ts)

### EnvironmentVariablesSchema
Environment variable declaration schema, defines environment variables that can be declared in settings.

### PermissionsSchema
Permission rules and pattern definitions, including:
- Allow rules (allow)
- Soft deny rules (soft_deny)
- Environment rules (environment)

### ExtraKnownMarketplaceSchema
Additional known marketplace source definitions.

### AllowedMcpServerEntrySchema
Enterprise MCP whitelist entry schema, controls allowed MCP servers.

### HooksSchema
Hook configuration schema imported from `schemas/hooks.ts`.

---

## Hot Reload (changeDetector.ts)

### File Monitoring
- Monitors changes to all settings files
- Uses debounce mechanism to notify listeners, avoiding frequent reloads

### Applying Updates
```
applySettingsChange() → AppState update
```
Update process triggered when settings change:
1. Permission context reload
2. Hook configuration reload
3. Environment variable reload
4. Refresh other components dependent on settings

---

## MDM Integration (settings/mdm/)

### rawRead.ts
Raw MDM (Mobile Device Management) settings read module, supports configuration retrieval in enterprise mobile device management scenarios.

---

## Validation

### validation.ts
Schema validation module, uses Zod Schema to validate settings file structure and types.

### permissionValidation.ts
Permission rule-specific validation, ensures semantic correctness of permission configurations.

### validationTips.ts
User-friendly prompt messages, provides readable error descriptions and fix suggestions when validation fails.

---

## Global Configuration (utils/config.ts)

### Global Configuration File
```
~/.claude/config.json
```
Stores global configuration information.

### Project-Level Configuration File
```
.claude.json
```
Configuration file in the project root directory, contains project-level configurations such as MCP server definitions.

### Core Functions
```
saveGlobalConfig(config): void    // Save global configuration
readProjectConfig(): ProjectConfig // Read project-level configuration
```

---

## Engineering Practice Guide

### Adding New Configuration Items

**Checklist:**

1. **Define in Schema**: Add field definition (using Zod) in the corresponding Schema in `src/utils/settings/types.ts`
2. **Register to Merge Logic**: If the new field has special merge behavior (e.g., array append instead of override), add handling in `settingsMergeCustomizer`
3. **Read in Related Code**: Get configuration values through `getInitialSettings()` or `getSettingsForSource()`
4. **Add Validation Tips**: Add user-friendly error tips in `validationTips.ts` to help users quickly fix configuration format errors
5. **Test Hot Reload**: Confirm configuration takes effect immediately after modifying settings file — `settingsChangeDetector` monitors file changes and triggers updates via `applySettingsChange()`

**Configuration File Path Overview:**
| Level | Path | Purpose |
|-------|------|---------|
| Enterprise Policy | `/etc/claude/managed-settings.json` + `managed-settings.d/*.json` | Security baseline, cannot be bypassed |
| User Settings | `~/.claude/settings.json` | Personal preferences |
| Project Settings | `.claude/settings.json` | Team conventions, committed to VCS |
| Local Settings | `.claude/settings.local.json` | Personal local config, gitignored |
| CLI Override | `--settings <path-or-json>` | Runtime override |

### Debugging Configuration Priority

1. **Use `claude config list`**: View all effective configurations and their sources, quickly locate which level's configuration is taking effect
2. **Check merge result**: `getSettingsWithErrors()` returns merged configuration and validation error list
3. **Check single-layer configuration**: `getSettingsForSource('projectSettings')` gets configuration values for a specific source, locates conflicts
4. **Validation failure diagnosis**: `parseSettingsFileUncached()` performs JSON parsing + Zod Schema validation. If settings format is incorrect, `validationTips.ts` provides readable fix suggestions
5. **Check cache**: `resetSettingsCache()` forces cache invalidation, eliminates cache staleness issues

**Priority Mnemonic (Low→High)**: `Enterprise Policy < User Settings < Project Settings < Local Settings < CLI Flags`

> Note: Although enterprise policy has the lowest priority number, `policySettings` is always included in `allowedSettingSources` and cannot be disabled — it ensures security policies take effect through a different mechanism (forced override rather than priority).

### Enterprise Policy Configuration

Enterprise administrators can distribute configurations through the following methods:

1. **File method**: Write policies to `/etc/claude/managed-settings.json` or `/etc/claude/managed-settings.d/*.json` (loaded in alphabetical order)
2. **MDM method**: Distribute configuration through MDM API (`settings/mdm/rawRead.ts`)
3. **Remote hosting**: Distribute through `remoteManagedSettings` (`scope: 'managed'`)

Policy constraint capabilities:
- `areMcpConfigsAllowedWithEnterpriseMcpConfig()` — restricts MCP servers added by users
- `filterMcpServersByPolicy()` — filters MCP servers by policy
- Permission rules of type `allow`/`soft_deny`/`environment` have independent filtering and merging logic

### Hot Reload Implementation

Update process after configuration file changes:

![Config Hot Reload Flow](../diagrams/config-hot-reload-flow-en.svg)

Edit `.claude/settings.json` in one terminal, and Claude Code in another terminal takes effect immediately.

### Common Pitfalls

> **project settings are tracked by git — do not put sensitive information there**
> `.claude/settings.json` is a project-level configuration, typically committed to version control. Do not put API keys, authentication tokens, personal paths, or other sensitive information in it. These should go in `.claude/settings.local.json` (gitignored) or be passed through environment variables.

> **local settings (.local.json) are gitignored**
> `.claude/settings.local.json` file will not be committed to version control. Used to store personal local configurations (e.g., local proxy settings, personal API key paths, etc.).

> **Validation after editing settings.json**
> Source `validateEditTool.ts:39` performs Zod Schema validation after each tool-based edit of settings.json. If validation fails, an error message containing the complete schema is generated. It is also recommended to verify JSON format when editing manually.

> **--cowork flag switches user settings file**
> When using the `--cowork` flag, user settings switch from `~/.claude/settings.json` to `~/.claude/cowork_settings.json` — this allows using different personal configurations in team collaboration mode.

> **policySettings can never be disabled**
> `allowedSettingSources` always includes `'policySettings'` during initialization (`bootstrap/state.ts`), and is not affected by user operations. This is a security design — preventing developers from bypassing enterprise security policies.



---

[← UI Rendering](../12-UI渲染/ui-rendering-en.md) | [Index](../README_EN.md) | [State Management →](../14-状态管理/state-management-en.md)
