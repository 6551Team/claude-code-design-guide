# Claude Code v2.1.88 完整工程架构文档

**[English](./README_EN.md) | 中文**

> 基于 1884 个 TypeScript 源文件的逐文件逆向工程分析
> 分析日期: 2026-03-31 | Build: 2026-03-30T21:59:52Z

## 文档结构

本文档按子系统拆分为 46 个专题目录，每个目录包含该子系统的完整分析：

| 章节 | 内容 |
|------|------|
| [01 系统总览](./01-系统总览/system-overview.md) | 源码结构、分层架构、模块关系图 |
| [02 启动与初始化](./02-启动与初始化/initialization.md) | main.tsx → init → REPL 完整链路 |
| [03 查询引擎](./03-查询引擎/query-engine.md) | query.ts + QueryEngine.ts 核心循环 |
| [04 API 客户端](./04-API客户端/api-client.md) | 多后端、流式传输、重试、错误分类 |
| [05 工具系统](./05-工具系统/tool-system.md) | 40+ 工具注册、编排、流式执行 |
| [06 权限与安全](./06-权限与安全/permission-security.md) | 6模式、分类器、沙箱、路径验证 |
| [07 上下文管理](./07-上下文管理/context-management.md) | 三层压缩、token预算、缓存控制 |
| [08 MCP 集成](./08-MCP集成/mcp-integration.md) | 配置、传输、认证、延迟加载 |
| [09 Hooks 系统](./09-Hooks系统/hooks-system.md) | 13事件钩子 + 70+ React Hooks |
| [10 Skills 系统](./10-Skills系统/skills-system.md) | 17内置技能、技能发现、fork执行 |
| [11 多智能体](./11-多智能体/multi-agent.md) | Agent/Teammate/Remote/Dream |
| [12 UI 渲染](./12-UI渲染/ui-rendering.md) | Ink引擎、组件树、设计系统 |
| [13 配置体系](./13-配置体系/config-system.md) | 5级优先级、热重载、MDM |
| [14 状态管理](./14-状态管理/state-management.md) | Bootstrap单例 + Zustand Store |
| [15 命令体系](./15-命令体系/command-system.md) | 87+ Slash命令完整清单 |
| [16 记忆系统](./16-记忆系统/memory-system.md) | memdir、自动提取、团队记忆 |
| [17 错误恢复](./17-错误恢复/error-recovery.md) | 5层恢复、保留策略、降级 |
| [18 遥测分析](./18-遥测分析/telemetry-system.md) | OTel、Datadog、GrowthBook、Perfetto |
| [19 反馈与调查](./19-反馈与调查/feedback-system.md) | 调查状态机、转录共享、概率门控 |
| [20 服务层](./20-服务层/services-complete.md) | 13个后台服务完整分析 |
| [21 插件系统](./21-插件系统/plugin-system.md) | 发现、安装、市场、策略 |
| [22 OAuth 与认证](./22-OAuth与认证/oauth-auth.md) | PKCE、keychain、令牌刷新 |
| [23 LSP 集成](./23-LSP集成/lsp-integration.md) | JSON-RPC、多实例、扩展名路由 |
| [24 沙箱系统](./24-沙箱系统/sandbox-system.md) | 配置、执行、违规检测 |
| [25 Git 与 GitHub](./25-Git与GitHub/git-github.md) | 文件系统解析、gitignore、gh CLI |
| [26 会话管理](./26-会话管理/session-management.md) | 历史、恢复、导出、分享 |
| [27 键绑定与输入](./27-键绑定与输入/keybinding-system.md) | 50+动作、和弦、上下文匹配 |
| [28 Vim 模式](./28-Vim模式/vim-mode.md) | 完整状态机、motion/operator/textobj |
| [29 语音系统](./29-语音系统/voice-system.md) | 门控、认证、集成 |
| [30 远程会话](./30-远程会话/remote-session.md) | CCR WebSocket、权限桥接 |
| [31 Bridge 协议](./31-Bridge协议/bridge-protocol.md) | 33文件、REST+WS、可靠传输 |
| [32 Buddy 系统](./32-Buddy系统/buddy-system.md) | 伴侣宠物、PRNG、精灵渲染 |
| [33 协调器模式](./33-协调器模式/coordinator-mode.md) | 多Worker编排、任务通知 |
| [34 Swarm 系统](./34-Swarm系统/swarm-architecture.md) | tmux/iTerm2/进程内后端、权限同步 |
| [35 Computer Use](./35-Computer-Use/computer-use.md) | macOS Enigo/Swift、锁、ESC热键 |
| [36 DeepLink](./36-DeepLink/deeplink-system.md) | 协议注册、终端启动、URL解析 |
| [37 Teleport](./37-Teleport/teleport-system.md) | CCR会话API、Git Bundle、环境 |
| [38 输出样式](./38-输出样式/output-styles.md) | Markdown前置matter、样式加载 |
| [39 原生模块](./39-原生模块/native-modules.md) | 色差、文件索引、Yoga布局 |
| [40 迁移系统](./40-迁移系统/migration-system.md) | 11个配置迁移 |
| [41 文件持久化](./41-文件持久化/file-persistence.md) | BYOC文件上传、mtime扫描 |
| [42 代价追踪](./42-代价追踪/cost-tracking.md) | 模型用量、会话费用、格式化 |
| [43 Shell 工具链](./43-Shell工具链/shell-toolchain.md) | Bash AST、PowerShell解析、规格 |
| [44 Screens 组件](./44-Screens组件/screens-components.md) | REPL、Doctor、Resume |
| [45 类型系统](./45-类型系统/type-system.md) | 消息、权限、命令、钩子类型 |
| [46 完整数据流图](./46-完整数据流图/complete-data-flow.md) | 端到端流转、调用图、时序图 |

## 阅读顺序建议

1. **快速了解**: 01-系统总览 → 46-完整数据流图
2. **核心循环**: 02-启动与初始化 → 03-查询引擎 → 04-API客户端 → 05-工具系统
3. **安全模型**: 06-权限与安全 → 24-沙箱系统 → 34-Swarm系统(权限同步)
4. **上下文策略**: 07-上下文管理 → 16-记忆系统 → 20-服务层(5个后台提取服务)
5. **扩展性**: 08-MCP集成 → 10-Skills系统 → 21-插件系统 → 09-Hooks系统
6. **用户体验**: 12-UI渲染 → 27-键绑定与输入 → 19-反馈与调查 → 42-代价追踪
7. **多智能体**: 11-多智能体 → 33-协调器模式 → 34-Swarm系统 → 37-Teleport
8. **远程能力**: 30-远程会话 → 31-Bridge协议 → 36-DeepLink → 41-文件持久化

## 规模统计

| 维度 | 数量 |
|------|------|
| TypeScript源文件 | 1,884 |
| 顶级目录 | 35 |
| 服务模块 | 13 |
| 内置工具 | 40+ |
| React Hooks | 70+ |
| Slash命令 | 87+ |
| 内置技能 | 17 |
| 事件钩子类型 | 13 |
| 权限模式 | 6 |
| API后端 | 4 (Anthropic/Bedrock/Vertex/Foundry) |
| MCP传输协议 | 4 (stdio/SSE/HTTP/WebSocket) |
| 分析事件 | 50+ |
| 配置迁移 | 11 |
| Bundled后总行数 | 16,667行 / 13MB |

---

[开始阅读 →](./01-系统总览/system-overview.md)
