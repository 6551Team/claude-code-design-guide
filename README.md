# Claude Code 设计指南

<div align="center">

**[English](./README_EN.md) | 中文 | [한국어](./README_KO.md)**

</div>

> 从互联网早期设计模型到 AI Agent 实战 —— 一本写给开发者的 Claude Code 深度解析
>
> 鸣谢泄漏的程序员，鸣谢用AI完成这本书的提出者
>
> https://x.com/cryptoxiao
>
> https://x.com/BoxMrChen
>
> https://x.com/0xfaskety

---

## 这本书是什么

Claude Code 是 Anthropic 官方发布的 AI 编程助手 CLI 工具。它不只是一个"会写代码的聊天机器人"，而是一套完整的 **Agent Runtime 系统**，包含工具调用、上下文工程、多代理协作、权限管理、扩展系统等现代工程方法的集大成之作。

这本书通过深度解析 Claude Code 的源码设计，帮助你理解：

- AI Agent 系统是如何从零构建的
- 现代 CLI 工具的工程哲学
- Context Engineering 的核心思想
- 工具系统、权限模型、扩展机制的设计模式

---

## 目标读者

| 读者类型 | 你能从这本书得到什么 |
|---------|-------------------|
| **小白 / 初学者** | 搞清楚 Claude Code 是什么、能做什么、怎么用 |
| **高级开发者** | 学习现代 CLI 工具的工程方法、TypeScript 大型项目架构 |
| **Agent 系统设计者** | 深入理解 Agent Runtime、Tooling、Context Engineering、扩展系统的设计模式 |

---

## 目录

### 前言
- [前言：为什么要读这本书](./00-preface.md)

### 第一部分：认识 Claude Code（小白友好）
- [第 1 章：Claude Code 是什么](./part1/01-introduction.md)
- [第 2 章：快速上手](./part1/02-quickstart.md)

### 第二部分：从互联网早期设计到 AI Agent
- [第 3 章：Unix 哲学与 CLI 的传统](./part2/03-unix-philosophy.md)
- [第 4 章：REPL 的演化史](./part2/04-repl-evolution.md)
- [第 5 章：从聊天机器人到 Agent](./part2/05-from-chatbot-to-agent.md)

### 第三部分：架构设计
- [第 6 章：查询引擎 —— 对话的心脏](./part3/06-query-engine.md)
- [第 7 章：状态管理设计](./part3/07-state-management.md)
- [第 8 章：消息循环与流式处理](./part3/08-message-loop.md)

### 第四部分：工具系统设计
- [第 9 章：工具系统的设计哲学](./part4/09-tool-design.md)
- [第 10 章：43 个内置工具全览](./part4/10-builtin-tools.md)
- [第 11 章：工具权限模型](./part4/11-tool-permission.md)

### 第五部分：Context Engineering
- [第 12 章：什么是 Context Engineering](./part5/12-context-what.md)
- [第 13 章：系统提示的构建艺术](./part5/13-system-prompt.md)
- [第 14 章：Memory 与 CLAUDE.md](./part5/14-memory-claudemd.md)
- [第 15 章：上下文压缩（Auto-Compact）](./part5/15-compact.md)

### 第六部分：Agent Runtime 与多代理
- [第 16 章：任务系统设计](./part6/16-task-system.md)
- [第 17 章：多代理架构](./part6/17-multi-agent.md)
- [第 18 章：协调器模式](./part6/18-coordinator.md)

### 第七部分：扩展系统
- [第 19 章：MCP 协议 —— 工具的互联网](./part7/19-mcp.md)
- [第 20 章：Skills 系统](./part7/20-skills.md)
- [第 21 章：插件系统](./part7/21-plugins.md)

### 第八部分：安全、权限与性能
- [第 22 章：权限模型的分层设计](./part8/22-permission-model.md)
- [第 23 章：安全设计](./part8/23-security.md)
- [第 24 章：性能优化](./part8/24-performance.md)

### 第九部分：设计哲学
- [第 25 章：Claude Code 的设计原则](./part9/25-design-principles.md)
- [第 26 章：未来展望](./part9/26-future.md)

---

## 如何阅读这本书

- **如果你是小白**：从第一部分开始，按顺序读
- **如果你是开发者**：可以跳过第一部分，从第二部分开始
- **如果你是 Agent 系统设计者**：重点阅读第三、四、五、六、七部分

---

## 关于源码

本书分析基于 Claude Code 的公开源码（通过 `node_modules` 中的 TypeScript 源文件）。所有代码引用均来自真实源码，不做任何推测。

---

## 进阶阅读

如果你想深入了解 Claude Code 的源码实现细节，可以阅读我们的进阶文档：

**[📚 Claude Code 源码架构分析](./architecture/README.md)** | **[English Version](./architecture/README_EN.md)**

进阶文档包含：
- 完整的源码树结构（1884个TypeScript文件）
- 6层架构设计详解
- 查询引擎、工具系统、权限模型的实现细节
- 40+工具、70+ Hooks、87+命令的源码分析
- 完整的模块依赖图和数据流图

---

*本书开源，欢迎贡献和勘误。*
