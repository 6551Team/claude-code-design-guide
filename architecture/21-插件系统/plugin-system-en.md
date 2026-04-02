# Plugin System Architecture

### Design Philosophy

Why is an independent plugin system needed rather than just MCP? MCP (Model Context Protocol) is a tool protocol — it defines how AI calls external tools. But the plugin system is a broader extension mechanism: plugins can modify the UI (via `loadPluginOutputStyles.ts`), inject slash commands (via `loadPluginCommands.ts`), register lifecycle hooks (via `loadPluginHooks.ts`), and define custom Agents (via `loadPluginAgents.ts`). These capabilities go beyond the scope of the MCP tool protocol. The existence of `mcpPluginIntegration.ts` illustrates this relationship exactly — MCP servers are integrated as a subset of plugins, rather than replacing the entire plugin system.

## Plugin Discovery & Loading

The core loading flow of the plugin system is coordinated by the following modules:

- **pluginLoader.ts**: The main orchestrator, responsible for coordinating the entire plugin discovery and loading process. Scans all registered plugin directories, loads plugins by priority, handles and reports loading errors.
- **loadPluginAgents.ts**: Loads custom Agent definitions. Plugins can extend the system's agent capabilities by declaring agent configurations, defining custom agent behaviors and tool sets.
- **loadPluginCommands.ts**: Loads plugin slash commands. Each plugin can register its own commands, which appear in the user's command list.
- **loadPluginHooks.ts**: Loads plugin hooks. Plugins can register callbacks on specific lifecycle events, such as session start, before/after message send, etc.
- **loadPluginOutputStyles.ts**: Loads terminal output style definitions. Plugins can customize the terminal rendering of their output, including colors, formatting, and layout.
- **pluginDirectories.ts**: Path utility functions. Provides plugin directory resolution, lookup, and path joining functionality, supporting multi-level scoped directory structures.
- **installedPluginsManager.ts**: Plugin registry manager. Maintains a complete inventory of installed plugins, supporting CRUD operations and persisting plugin metadata.

## Scope Management

The plugin system uses a layered scope model to manage plugin installation and visibility:

```typescript
VALID_INSTALLABLE_SCOPES = ['user', 'project', 'local']  // excludes 'managed'
VALID_UPDATE_SCOPES  // includes 'managed', allows updates but not direct installation
```

**Scope priority**: `local > project > user`

- **findPluginInSettings()**: When looking up a plugin, the most specific scope takes precedence. The `local` scope overrides `project`, and `project` overrides `user`. This ensures that project-level and local-level configurations can override global settings.
- **V2 data fallback**: `resolveDelistedPluginId()` handles ID resolution for delisted plugins, ensuring backward compatibility with older data formats. After a plugin is removed from the marketplace, installed instances still need to be correctly identified.

## Marketplace Integration

Marketplace integration provides plugin discovery, installation, and management capabilities:

- **officialMarketplace.ts**: The official marketplace client, providing API interfaces for plugin search, detail retrieval, and version queries.
- **officialMarketplaceGcs.ts**: A GCS (Google Cloud Storage)-based marketplace backend implementation. Plugin packages and metadata are stored in GCS buckets, supporting high availability and global distribution.
- **officialMarketplaceStartupCheck.ts**: Startup marketplace check. Verifies marketplace reachability at application startup, checks for plugin updates, and handles degradation strategies for offline scenarios.
- **marketplaceManager.ts**: CRUD operations manager. Encapsulates the complete lifecycle operations for marketplace plugins: installation (Create), query (Read), update (Update), and uninstall (Delete).
- **parseMarketplaceInput.ts**: URL parser. Parses user-input marketplace URLs, plugin identifiers, and version constraints, supporting multiple input formats (full URL, short name, name@version, etc.).

## Plugin Lifecycle

Plugin lifecycle management covers the complete flow from validation to auto-update:

- **validatePlugin.ts**: Schema validator, performing strict structural and type validation against `plugin.json`. Ensures that plugin declaration files conform to the specification, including required fields, type constraints, and value range checks.
- **pluginVersioning.ts**: Version management module. Handles parsing, comparison, and compatibility checking of semantic versions (semver), supporting version range constraints and upgrade path calculation.

  #### Why This Design

  Plugin APIs may change across Claude Code versions — incompatible plugins can cause crashes or security vulnerabilities. Version compatibility checks (via semver comparison) intercept problems at the loading stage rather than discovering them at runtime. `pluginStartupCheck.ts` performs health checks before plugin loading to validate dependency integrity and runtime compatibility, and `pluginBlocklist.ts` maintains a blocklist of known malicious or incompatible plugins for pre-load interception. This "defense in depth" approach ensures that only verified plugins can enter the runtime.

- **pluginOptionsStorage.ts**: Persistent options storage. Provides persistent read/write for plugin runtime configuration, supports scope-isolated storage, and ensures plugin configuration persists across sessions.
- **pluginPolicy.ts**: Security policy engine. Defines and enforces the permission model for plugins, controlling plugin access to resources such as the filesystem, network, and tools.
- **pluginBlocklist.ts**: Blocklist management. Maintains a blocklist of known malicious or incompatible plugins, intercepts them before loading, and supports remote updates to blocklist rules.
- **pluginFlagging.ts**: Status flagging system. Marks various plugin states (e.g., deprecated, requires review, has security issues), affecting plugin display and loading behavior.
- **pluginStartupCheck.ts**: Startup validation. Performs health checks before plugin loading, verifying dependency integrity, runtime compatibility, and configuration validity.
- **pluginAutoupdate.ts**: Auto-update mechanism. Checks for plugin updates in the background, automatically or prompts for updates according to user policy, and handles update conflicts and rollbacks.
- **headlessPluginInstall.ts**: Programmatic installation interface. Supports non-interactive plugin installation, used for CI/CD environments, scripted deployments, and batch installation scenarios.

## Dependencies & Integration

Integration and dependency management between the plugin system and external systems:

- **dependencyResolver.ts**: Dependency resolver. Builds the dependency graph between plugins, detects circular dependencies, determines the correct loading order, and handles version conflicts.
- **reconciler.ts**: State reconciler. Compares the desired state (declared in configuration files) with the actual state (installed plugins), generates a plan of install/uninstall/update operations, and ensures system consistency.
- **mcpPluginIntegration.ts**: MCP server plugin integration. Integrates MCP (Model Context Protocol) servers as plugins, manages the lifecycle of MCP servers, and bridges MCP tools with the plugin tool system.

### Design Philosophy: DXT Extension Format

Why is the DXT format (`utils/dxt/`) needed? DXT is a standardized plugin packaging format that bundles the manifest, code, and assets into a single file (`.dxt` or `.mcpb`). Standardized packaging makes install/uninstall/update atomic operations — either fully succeeding or fully rolling back, with no corrupted partial-installation state. `helpers.ts` provides three parsing entry points — `parseDxtManifestFromJSON`, `parseDxtManifestFromText`, and `parseDxtManifestFromBinary` — supporting loading from different sources. `isMcpbOrDxt()` in `mcpbHandler.ts` provides a unified check for whether a file is in a packaged format. The DXT manifest defines the user configuration schema, enabling configuration completeness validation at installation time.

- **lspPluginIntegration.ts**: LSP integration. Integrates with Language Server Protocol servers to provide plugins with language intelligence capabilities (code completion, diagnostics, go-to-definition, etc.).
- **hintRecommendation.ts**: Hint recommendations. Recommends potentially useful plugins based on user behavior and context, supporting intelligent suggestions.
- **lspRecommendation.ts**: LSP recommendations. Recommends corresponding LSP plugins based on the programming languages and frameworks used in a project.

## Plugin Telemetry

The plugin telemetry system is used to collect usage data and error information:

- **hashPluginId()**: Applies privacy-preserving processing to plugin IDs by taking the first 16 characters of a SHA256 hash, ensuring that telemetry data does not expose plugin identities.

- **Scope classification**:
  - `official`: Official marketplace plugins
  - `org`: Organization-level plugins
  - `user-local`: User-locally developed plugins
  - `default-bundle`: Default bundled plugins

- **classifyPluginCommandError()**: Classifies plugin command execution errors into 5 categories, used for error attribution and monitoring alerts. The classification result affects retry strategies and error reporting paths.

- **logPluginsEnabledForSession()**: Records the list of plugins enabled in the current session, used for usage statistics and troubleshooting.
- **logPluginLoadErrors()**: Records detailed error information for plugin loading failures, including stack traces, plugin versions, and environment information.

## Bundled Plugins

The built-in plugin system provides out-of-the-box core functionality:

- **builtinPlugins.ts**: Built-in plugin registration and management module.
  - `registerBuiltinPlugin()`: Registers built-in plugins. Built-in plugins are automatically loaded at system startup without requiring user installation.
  - `isBuiltinPluginId()`: Determines whether a given ID is a built-in plugin, used to distinguish between built-in and user-installed plugins.

- **skillDefinitionToCommand()**: Converts a skill definition into a command format. This is the bridge layer between the built-in skill system and the plugin command system, enabling skills to be invoked and displayed as plugin commands.

## CLI Handlers

`cli/handlers/plugins.ts` (approximately 580 lines) provides complete CLI command handling:

**Validation & Query**:
- `pluginValidateHandler()`: Validates the legality of plugin structure and configuration
- `pluginListHandler()`: Lists installed plugins, with support for filtering by scope

**Marketplace Operations**:
- `marketplaceAddHandler()`: Adds a plugin from the marketplace
- `marketplaceListHandler()`: Lists plugins available on the marketplace
- `marketplaceRemoveHandler()`: Removes a plugin from the marketplace
- `marketplaceUpdateHandler()`: Updates a marketplace plugin

**Plugin Management**:
- `pluginInstall()`: Installs a plugin to a specified scope
- `pluginUninstall()`: Uninstalls a plugin and cleans up resources
- `pluginEnable()`: Enables a disabled plugin
- `pluginDisable()`: Disables a plugin while keeping it installed
- `pluginUpdate()`: Updates a plugin to a specified or the latest version

---

## Engineering Practice Guide

### Developing DXT Plugins

**Step-by-step checklist:**

1. **Create the plugin package structure**:
   ```
   my-plugin/
   ├── plugin.json          # DXT manifest (required)
   ├── src/                  # Plugin code
   └── README.md            # Documentation
   ```
2. **Define the manifest**: Define plugin metadata, permission declarations, and hook interfaces according to the `KeybindingBlockSchema` and `plugin.json` schema specifications.
3. **Implement hook interfaces**:
   - Custom Agent — loaded via `loadPluginAgents.ts`
   - Slash commands — registered via `loadPluginCommands.ts`
   - Lifecycle hooks — registered via `loadPluginHooks.ts`
   - Output styles — defined via `loadPluginOutputStyles.ts`
4. **Test compatibility**: Use the `pluginValidateHandler()` CLI command to validate the plugin structure.
5. **Package and publish**: The DXT format bundles the manifest, code, and assets into a single `.dxt` or `.mcpb` file (atomic operation — install/uninstall/update either fully succeeds or fully rolls back).

**DXT parsing entry points** (`utils/dxt/helpers.ts`):
- `parseDxtManifestFromJSON` — parses from a JSON object
- `parseDxtManifestFromText` — parses from raw text
- `parseDxtManifestFromBinary` — parses from binary data

### MCP Integration

**MCP servers are integrated as a subset of plugins (`mcpPluginIntegration.ts`):**

1. Download and extract the DXT manifest
2. Use the DXT manifest name as the MCP server name
3. Bridge MCP tools with the plugin tool system
4. Supports `.mcp.json` configuration files and `.mcpb` packaged files

**Check if a file is in a packaged format**: `isMcpbOrDxt()` provides a unified check for `.mcpb` and `.dxt` extensions.

### Debugging Plugin Loading

**Troubleshooting steps:**

1. **Check plugin directory paths**: `pluginDirectories.ts` provides path resolution; confirm that plugin files are in the correct location.
2. **Check version compatibility**: `pluginVersioning.ts` performs semver comparison; incompatible plugins are intercepted at the loading stage.
3. **Check the blocklist**: `pluginBlocklist.ts` maintains a blocklist of malicious/incompatible plugins and intercepts them before loading.
4. **Check startup validation**: `pluginStartupCheck.ts` performs health checks before loading (dependency integrity, runtime compatibility, configuration validity).
5. **Review loading errors**: `logPluginLoadErrors()` records detailed error information (stack traces, plugin versions, environment information).
6. **Check scope priority**: `local > project > user`; confirm whether the plugin is being overridden by a higher-priority plugin with the same name.

**CLI debugging commands:**
```bash
claude plugin validate <path>       # Validate plugin structure
claude plugin list                  # List installed plugins
claude marketplace list             # List plugins available on the marketplace
```

### Plugin Scope Management

| Scope | Install | Update | Description |
|-------|---------|--------|-------------|
| `user` | Allowed | Allowed | Takes effect globally |
| `project` | Allowed | Allowed | Specific project only |
| `local` | Allowed | Allowed | Local development |
| `managed` | Direct install not allowed | Allowed | Organization managed |

**V2 data fallback**: `resolveDelistedPluginId()` handles ID resolution for delisted plugins, ensuring backward compatibility.

### Common Pitfalls

| Pitfall | Details | Solution |
|---------|---------|----------|
| Plugins can access the filesystem | Plugins have the ability to read and write files; security review is important | `pluginPolicy.ts` defines the permission model, controlling filesystem/network/tool access |
| Plugin API changes require version adaptation | Plugin APIs change across Claude Code versions | Use semver constraints; `pluginVersioning.ts` checks compatibility |
| Circular dependencies | Circular dependencies can form between plugins | `dependencyResolver.ts` detects circular dependencies and establishes the correct loading order |
| Plugin state reconciliation | Desired state and actual state may be inconsistent | `reconciler.ts` compares declared state with installed state and generates an operations plan |
| Headless installation | CI/CD environments require non-interactive installation | Use the `headlessPluginInstall.ts` programmatic installation interface |
| Plugin telemetry privacy | Do not expose specific plugin names/paths in telemetry | `hashPluginId()` takes the first 16 characters of a SHA256 hash |


---

[← Service Layer](../20-服务层/services-complete-en.md) | [Index](../README_EN.md) | [OAuth & Auth →](../22-OAuth与认证/oauth-auth-en.md)
