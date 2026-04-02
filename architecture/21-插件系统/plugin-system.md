# Plugin System Architecture

### 设计理念

为什么需要独立插件系统而不只是 MCP？MCP（Model Context Protocol）是工具协议——它定义了 AI 如何调用外部工具。但插件系统是更广泛的扩展机制：插件可以修改 UI（通过 `loadPluginOutputStyles.ts`）、注入斜杠命令（通过 `loadPluginCommands.ts`）、注册生命周期钩子（通过 `loadPluginHooks.ts`）、定义自定义 Agent（通过 `loadPluginAgents.ts`），这些能力超出了 MCP 工具协议的范畴。`mcpPluginIntegration.ts` 的存在正好说明了这种关系——MCP 服务器作为插件的一个子集被集成，而非替代整个插件体系。

## Plugin Discovery & Loading

插件系统的核心加载流程由以下模块协同完成:

- **pluginLoader.ts**: 主编排器，负责协调整个插件发现与加载流程。扫描所有已注册的插件目录，按优先级加载插件，处理加载错误并汇报。
- **loadPluginAgents.ts**: 加载自定义 Agent 定义。插件可以通过声明 agent 配置来扩展系统的 agent 能力，定义自定义的 agent 行为和工具集。
- **loadPluginCommands.ts**: 加载插件斜杠命令（slash commands）。每个插件可以注册自己的命令，这些命令会出现在用户的命令列表中。
- **loadPluginHooks.ts**: 加载插件钩子（hooks）。插件可以在特定生命周期事件上注册回调，如会话开始、消息发送前后等。
- **loadPluginOutputStyles.ts**: 加载终端输出样式定义。插件可以自定义其输出的终端渲染方式，包括颜色、格式和布局。
- **pluginDirectories.ts**: 路径工具函数。提供插件目录的解析、查找和路径拼接功能，支持多级作用域的目录结构。
- **installedPluginsManager.ts**: 插件注册表管理器。维护已安装插件的完整清单，支持增删改查操作，持久化存储插件元数据。

## Scope Management

插件系统使用分层作用域模型来管理插件的安装和可见性:

```typescript
VALID_INSTALLABLE_SCOPES = ['user', 'project', 'local']  // 排除 'managed'
VALID_UPDATE_SCOPES  // 包含 'managed'，允许更新但不允许直接安装
```

**作用域优先级**: `local > project > user`

- **findPluginInSettings()**: 查找插件时，最具体的作用域优先生效。local 作用域覆盖 project，project 覆盖 user。这确保了项目级和本地级的配置可以覆盖全局设置。
- **V2 数据回退**: `resolveDelistedPluginId()` 处理已下架插件的 ID 解析，确保旧版数据格式的向后兼容性。当插件从 marketplace 下架后，已安装的实例仍需正确识别。

## Marketplace Integration

Marketplace 集成提供了插件的发现、安装和管理能力:

- **officialMarketplace.ts**: 官方 marketplace 客户端，提供插件搜索、详情获取、版本查询等 API 接口。
- **officialMarketplaceGcs.ts**: 基于 GCS（Google Cloud Storage）的 marketplace 后端实现。插件包和元数据存储在 GCS bucket 中，支持高可用和全球分发。
- **officialMarketplaceStartupCheck.ts**: 启动时 marketplace 检查。在应用启动时验证 marketplace 可达性，检查插件更新，处理离线场景的降级策略。
- **marketplaceManager.ts**: CRUD 操作管理器。封装了 marketplace 插件的安装（Create）、查询（Read）、更新（Update）、卸载（Delete）完整生命周期操作。
- **parseMarketplaceInput.ts**: URL 解析器。解析用户输入的 marketplace URL、插件标识符和版本约束，支持多种输入格式（完整 URL、简短名称、name@version 等）。

## Plugin Lifecycle

插件生命周期管理涵盖从验证到自动更新的完整流程:

- **validatePlugin.ts**: Schema 验证器，针对 `plugin.json` 进行严格的结构和类型验证。确保插件声明文件符合规范，包括必需字段、类型约束和值范围检查。
- **pluginVersioning.ts**: 版本管理模块。处理语义化版本（semver）的解析、比较和兼容性检查，支持版本范围约束和升级路径计算。

  #### 为什么这样设计

  插件 API 可能随 Claude Code 版本变化——不兼容的插件会导致崩溃或安全漏洞。版本兼容检查（通过 semver 比较）在加载阶段拦截问题，而非在运行时才发现。`pluginStartupCheck.ts` 在插件加载前执行健康检查验证依赖完整性和运行时兼容性，`pluginBlocklist.ts` 维护已知恶意或不兼容插件的黑名单进行预加载拦截。这种"多层防御"确保只有经过验证的插件才能进入运行时。
- **pluginOptionsStorage.ts**: 持久化选项存储。插件运行时配置的持久化读写，支持按作用域隔离存储，确保插件配置在会话间保持。
- **pluginPolicy.ts**: 安全策略引擎。定义和执行插件的权限模型，控制插件对文件系统、网络、工具等资源的访问权限。
- **pluginBlocklist.ts**: 封禁列表管理。维护已知恶意或不兼容插件的黑名单，在加载前进行拦截，支持远程更新封禁规则。
- **pluginFlagging.ts**: 状态标记系统。标记插件的各种状态（如已弃用、需要审核、存在安全问题），影响插件的展示和加载行为。
- **pluginStartupCheck.ts**: 启动验证。在插件加载前执行健康检查，验证依赖完整性、运行时兼容性和配置有效性。
- **pluginAutoupdate.ts**: 自动更新机制。后台检查插件更新，根据用户策略自动或提示更新，处理更新冲突和回滚。
- **headlessPluginInstall.ts**: 程序化安装接口。支持无交互式的插件安装，用于 CI/CD 环境、脚本化部署和批量安装场景。

## Dependencies & Integration

插件系统与外部系统的集成和依赖管理:

- **dependencyResolver.ts**: 依赖解析器。构建插件间的依赖图，检测循环依赖，确定正确的加载顺序，处理版本冲突。
- **reconciler.ts**: 状态协调器。对比期望状态（配置文件声明）和实际状态（已安装插件），生成安装/卸载/更新操作计划，确保系统一致性。
- **mcpPluginIntegration.ts**: MCP 服务器插件集成。将 MCP（Model Context Protocol）服务器作为插件集成，管理 MCP 服务器的生命周期，桥接 MCP 工具与插件工具体系。

### 设计理念：DXT 扩展格式

为什么需要 DXT 格式（`utils/dxt/`）？DXT 是标准化的插件打包格式，将 manifest、代码和资源封装为单一文件（`.dxt` 或 `.mcpb`）。标准化打包让安装/卸载/更新成为原子操作——要么完全成功，要么完全回滚，不会出现部分安装的损坏状态。`helpers.ts` 提供了 `parseDxtManifestFromJSON`、`parseDxtManifestFromText`、`parseDxtManifestFromBinary` 三种解析入口，支持从不同来源加载。`mcpbHandler.ts` 中 `isMcpbOrDxt()` 统一判断文件是否为打包格式，DXT manifest 定义了用户配置 schema，使得安装时可以验证配置完整性。
- **lspPluginIntegration.ts**: LSP 集成。与 Language Server Protocol 服务器集成，为插件提供语言智能能力（代码补全、诊断、跳转等）。
- **hintRecommendation.ts**: 提示推荐。基于用户行为和上下文，推荐可能有用的插件，支持智能提示。
- **lspRecommendation.ts**: LSP 推荐。根据项目中使用的编程语言和框架，推荐相应的 LSP 插件。

## Plugin Telemetry

插件遥测系统用于收集使用数据和错误信息:

- **hashPluginId()**: 对插件 ID 进行隐私化处理，使用 SHA256 哈希后截取前 16 个字符，确保遥测数据不泄露插件标识。

- **scope 分类**:
  - `official`: 官方 marketplace 插件
  - `org`: 组织级插件
  - `user-local`: 用户本地开发的插件
  - `default-bundle`: 默认捆绑插件

- **classifyPluginCommandError()**: 将插件命令执行错误分为 5 个类别，用于错误归因和监控告警。分类结果影响重试策略和错误上报路径。

- **logPluginsEnabledForSession()**: 记录当前会话中启用的插件列表，用于使用统计和问题排查。
- **logPluginLoadErrors()**: 记录插件加载失败的详细错误信息，包括堆栈跟踪、插件版本和环境信息。

## Bundled Plugins

内置插件系统提供开箱即用的核心功能:

- **builtinPlugins.ts**: 内置插件注册和管理模块。
  - `registerBuiltinPlugin()`: 注册内置插件。内置插件在系统启动时自动加载，无需用户安装。
  - `isBuiltinPluginId()`: 判断给定 ID 是否为内置插件，用于区分内置和用户安装的插件。

- **skillDefinitionToCommand()**: 将 skill 定义转换为命令格式。这是内置 skill 系统与插件命令系统的桥接层，使 skill 可以作为插件命令被调用和展示。

## CLI Handlers

`cli/handlers/plugins.ts`（约 580 行）提供了完整的 CLI 命令处理:

**验证与查询**:
- `pluginValidateHandler()`: 验证插件结构和配置的合法性
- `pluginListHandler()`: 列出已安装的插件，支持按作用域筛选

**Marketplace 操作**:
- `marketplaceAddHandler()`: 从 marketplace 添加插件
- `marketplaceListHandler()`: 列出 marketplace 可用插件
- `marketplaceRemoveHandler()`: 从 marketplace 移除插件
- `marketplaceUpdateHandler()`: 更新 marketplace 插件

**插件管理**:
- `pluginInstall()`: 安装插件到指定作用域
- `pluginUninstall()`: 卸载插件并清理资源
- `pluginEnable()`: 启用已禁用的插件
- `pluginDisable()`: 禁用插件但保留安装
- `pluginUpdate()`: 更新插件到指定或最新版本

---

## 工程实践指南

### 开发 DXT 插件

**步骤清单：**

1. **创建插件包结构**：
   ```
   my-plugin/
   ├── plugin.json          # DXT manifest（必需）
   ├── src/                  # 插件代码
   └── README.md            # 文档
   ```
2. **定义 manifest**：按 `KeybindingBlockSchema` 和 `plugin.json` schema 规范定义插件元数据、权限声明、钩子接口
3. **实现钩子接口**：
   - 自定义 Agent — 通过 `loadPluginAgents.ts` 加载
   - 斜杠命令 — 通过 `loadPluginCommands.ts` 注册
   - 生命周期钩子 — 通过 `loadPluginHooks.ts` 注册
   - 输出样式 — 通过 `loadPluginOutputStyles.ts` 定义
4. **测试兼容性**：使用 `pluginValidateHandler()` CLI 命令验证插件结构
5. **打包发布**：DXT 格式将 manifest、代码和资源封装为 `.dxt` 或 `.mcpb` 单一文件（原子操作——安装/卸载/更新要么完全成功要么完全回滚）

**DXT 解析入口**（`utils/dxt/helpers.ts`）：
- `parseDxtManifestFromJSON` — 从 JSON 对象解析
- `parseDxtManifestFromText` — 从原始文本解析
- `parseDxtManifestFromBinary` — 从二进制数据解析

### MCP 集成

**MCP 服务器作为插件子集被集成（`mcpPluginIntegration.ts`）：**

1. 下载和提取 DXT manifest
2. 将 DXT manifest name 作为 MCP server name
3. 桥接 MCP 工具与插件工具体系
4. 支持 `.mcp.json` 配置文件和 `.mcpb` 打包文件

**判断文件是否为打包格式**：`isMcpbOrDxt()` 统一判断 `.mcpb` 和 `.dxt` 后缀

### 调试插件加载

**排查步骤：**

1. **检查插件目录路径**：`pluginDirectories.ts` 提供路径解析，确认插件文件在正确位置
2. **检查版本兼容性**：`pluginVersioning.ts` 进行 semver 比较，不兼容的插件在加载阶段被拦截
3. **检查封禁列表**：`pluginBlocklist.ts` 维护恶意/不兼容插件黑名单，加载前拦截
4. **检查启动验证**：`pluginStartupCheck.ts` 在加载前执行健康检查（依赖完整性、运行时兼容性、配置有效性）
5. **查看加载错误**：`logPluginLoadErrors()` 记录详细错误信息（堆栈跟踪、插件版本、环境信息）
6. **检查作用域优先级**：`local > project > user`，确认是否被更高优先级的同名插件覆盖

**CLI 调试命令：**
```bash
claude plugin validate <path>       # 验证插件结构
claude plugin list                  # 列出已安装插件
claude marketplace list             # 列出 marketplace 可用插件
```

### 插件作用域管理

| 作用域 | 安装 | 更新 | 说明 |
|--------|------|------|------|
| `user` | 允许 | 允许 | 全局生效 |
| `project` | 允许 | 允许 | 仅特定项目 |
| `local` | 允许 | 允许 | 本地开发 |
| `managed` | 不允许直接安装 | 允许 | 组织管理 |

**V2 数据回退**：`resolveDelistedPluginId()` 处理已下架插件的 ID 解析，确保向后兼容。

### 常见陷阱

| 陷阱 | 详情 | 解决方案 |
|------|------|----------|
| 插件可以访问文件系统 | 插件有读写文件的能力，安全审查很重要 | `pluginPolicy.ts` 定义权限模型，控制文件系统/网络/工具访问 |
| 插件 API 变更需要版本适配 | 插件 API 随 Claude Code 版本变化 | 使用 semver 约束，`pluginVersioning.ts` 检查兼容性 |
| 循环依赖 | 插件间可能形成循环依赖 | `dependencyResolver.ts` 检测循环依赖，构建正确的加载顺序 |
| 插件状态协调 | 期望状态和实际状态可能不一致 | `reconciler.ts` 对比声明状态和已安装状态，生成操作计划 |
| headless 安装 | CI/CD 环境需要无交互安装 | 使用 `headlessPluginInstall.ts` 程序化安装接口 |
| 插件遥测隐私 | 遥测中不暴露插件具体名称/路径 | `hashPluginId()` SHA256 哈希后截取前 16 字符 |


---

[← 服务层](../20-服务层/services-complete.md) | [目录](../README.md) | [OAuth 与认证 →](../22-OAuth与认证/oauth-auth.md)
