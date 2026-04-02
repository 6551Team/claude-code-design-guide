# Claude Code v2.1.88 Complete Engineering Architecture Documentation

**English | [中文](./README.md)**

> Based on reverse engineering analysis of 1884 TypeScript source files
> Analysis Date: 2026-03-31 | Build: 2026-03-30T21:59:52Z

## Document Structure

This documentation is organized into 46 specialized directories, each containing complete analysis of a subsystem:

| Chapter | Contents |
|---------|----------|
| [01 System Overview](./01-系统总览/system-overview-en.md) | Source structure, layered architecture, module relationships |
| [02 Startup & Initialization](./02-启动与初始化/initialization-en.md) | main.tsx → init → REPL complete chain |
| [03 Query Engine](./03-查询引擎/query-engine-en.md) | query.ts + QueryEngine.ts core loop |
| [04 API Client](./04-API客户端/api-client-en.md) | Multi-backend, streaming, retry, error classification |
| [05 Tool System](./05-工具系统/tool-system-en.md) | 40+ tool registration, orchestration, streaming execution |
| [06 Permission & Security](./06-权限与安全/permission-security-en.md) | 6 modes, classifier, sandbox, path validation |
| [07 Context Management](./07-上下文管理/context-management-en.md) | Three-layer compression, token budget, cache control |
| [08 MCP Integration](./08-MCP集成/mcp-integration-en.md) | Configuration, transport, authentication, lazy loading |
| [09 Hooks System](./09-Hooks系统/hooks-system-en.md) | 13 event hooks + 70+ React Hooks |
| [10 Skills System](./10-Skills系统/skills-system-en.md) | 17 built-in skills, skill discovery, fork execution |
| [11 Multi-Agent](./11-多智能体/multi-agent-en.md) | Agent/Teammate/Remote/Dream |
| [12 UI Rendering](./12-UI渲染/ui-rendering-en.md) | Ink engine, component tree, design system |
| [13 Config System](./13-配置体系/config-system-en.md) | 5-level priority, hot reload, MDM |
| [14 State Management](./14-状态管理/state-management-en.md) | Bootstrap singleton + Zustand Store |
| [15 Command System](./15-命令体系/command-system-en.md) | 87+ Slash commands complete list |
| [16 Memory System](./16-记忆系统/memory-system-en.md) | memdir, auto-extraction, team memory |
| [17 Error Recovery](./17-错误恢复/error-recovery-en.md) | 5-layer recovery, retention strategy, degradation |
| [18 Telemetry & Analytics](./18-遥测分析/telemetry-system-en.md) | OTel, Datadog, GrowthBook, Perfetto |
| [19 Feedback & Survey](./19-反馈与调查/feedback-system-en.md) | Survey state machine, transcript sharing, probability gating |
| [20 Service Layer](./20-服务层/services-complete-en.md) | 13 background services complete analysis |
| [21 Plugin System](./21-插件系统/plugin-system-en.md) | Discovery, installation, marketplace, policy |
| [22 OAuth & Auth](./22-OAuth与认证/oauth-auth-en.md) | PKCE, keychain, token refresh |
| [23 LSP Integration](./23-LSP集成/lsp-integration-en.md) | JSON-RPC, multi-instance, extension routing |
| [24 Sandbox System](./24-沙箱系统/sandbox-system-en.md) | Configuration, execution, violation detection |
| [25 Git & GitHub](./25-Git与GitHub/git-github-en.md) | Filesystem parsing, gitignore, gh CLI |
| [26 Session Management](./26-会话管理/session-management-en.md) | History, recovery, export, sharing |
| [27 Keybindings & Input](./27-键绑定与输入/keybinding-system-en.md) | 50+ actions, chords, context matching |
| [28 Vim Mode](./28-Vim模式/vim-mode-en.md) | Complete state machine, motion/operator/textobj |
| [29 Voice System](./29-语音系统/voice-system-en.md) | Gating, authentication, integration |
| [30 Remote Session](./30-远程会话/remote-session-en.md) | CCR WebSocket, permission bridging |
| [31 Bridge Protocol](./31-Bridge协议/bridge-protocol-en.md) | 33 files, REST+WS, reliable transport |
| [32 Buddy System](./32-Buddy系统/buddy-system-en.md) | Companion pet, PRNG, sprite rendering |
| [33 Coordinator Pattern](./33-协调器模式/coordinator-mode-en.md) | Multi-Worker orchestration, task notification |
| [34 Swarm System](./34-Swarm系统/swarm-architecture-en.md) | tmux/iTerm2/in-process backend, permission sync |
| [35 Computer Use](./35-Computer-Use/computer-use-en.md) | macOS Enigo/Swift, locks, ESC hotkey |
| [36 DeepLink](./36-DeepLink/deeplink-system-en.md) | Protocol registration, terminal launch, URL parsing |
| [37 Teleport](./37-Teleport/teleport-system-en.md) | CCR session API, Git Bundle, environment |
| [38 Output Styles](./38-输出样式/output-styles-en.md) | Markdown front matter, style loading |
| [39 Native Modules](./39-原生模块/native-modules-en.md) | Color difference, file indexing, Yoga layout |
| [40 Migration System](./40-迁移系统/migration-system-en.md) | 11 configuration migrations |
| [41 File Persistence](./41-文件持久化/file-persistence-en.md) | BYOC file upload, mtime scanning |
| [42 Cost Tracking](./42-代价追踪/cost-tracking-en.md) | Model usage, session cost, formatting |
| [43 Shell Toolchain](./43-Shell工具链/shell-toolchain-en.md) | Bash AST, PowerShell parsing, specs |
| [44 Screens Components](./44-Screens组件/screens-components-en.md) | REPL, Doctor, Resume |
| [45 Type System](./45-类型系统/type-system-en.md) | Message, permission, command, hook types |
| [46 Complete Data Flow](./46-完整数据流图/complete-data-flow-en.md) | End-to-end flow, call graph, sequence diagram |

## Recommended Reading Order

1. **Quick Overview**: 01-System Overview → 46-Complete Data Flow
2. **Core Loop**: 02-Startup & Initialization → 03-Query Engine → 04-API Client → 05-Tool System
3. **Security Model**: 06-Permission & Security → 24-Sandbox System → 34-Swarm System (permission sync)
4. **Context Strategy**: 07-Context Management → 16-Memory System → 20-Service Layer (5 background extraction services)
5. **Extensibility**: 08-MCP Integration → 10-Skills System → 21-Plugin System → 09-Hooks System
6. **User Experience**: 12-UI Rendering → 27-Keybindings & Input → 19-Feedback & Survey → 42-Cost Tracking
7. **Multi-Agent**: 11-Multi-Agent → 33-Coordinator Mode → 34-Swarm System → 37-Teleport
8. **Remote Capabilities**: 30-Remote Session → 31-Bridge Protocol → 36-DeepLink → 41-File Persistence

## Scale Statistics

| Dimension | Count |
|-----------|-------|
| TypeScript source files | 1,884 |
| Top-level directories | 35 |
| Service modules | 13 |
| Built-in tools | 40+ |
| React Hooks | 70+ |
| Slash commands | 87+ |
| Built-in skills | 17 |
| Event hook types | 13 |
| Permission modes | 6 |
| API backends | 4 (Anthropic/Bedrock/Vertex/Foundry) |
| MCP transport protocols | 4 (stdio/SSE/HTTP/WebSocket) |
| Analytics events | 50+ |
| Configuration migrations | 11 |
| Bundled total lines | 16,667 lines / 13MB |

---

[Start Reading →](./01-系统总览/system-overview-en.md)
