# Claude Code Design Guide

<div align="center">

**English | [中文](./README.md) | [한국어](./README_KO.md)**

</div>

> From Early Internet Design Patterns to AI Agent Implementation — A Deep Dive into Claude Code for Developers
>
> Special thanks to the leaker and the proposer who completed this book with AI
>
> https://x.com/cryptoxiao
>
> https://x.com/BoxMrChen
>
> https://x.com/0xfaskety

---

## What is This Book

Claude Code is Anthropic's official AI programming assistant CLI tool. It's not just a "chatbot that writes code," but a complete **Agent Runtime System** that represents the culmination of modern engineering practices including tool invocation, context engineering, multi-agent collaboration, permission management, and extension systems.

This book helps you understand through deep analysis of Claude Code's source code design:

- How AI Agent systems are built from scratch
- Engineering philosophy of modern CLI tools
- Core concepts of Context Engineering
- Design patterns for tool systems, permission models, and extension mechanisms

---

## Target Audience

| Reader Type | What You'll Get |
|------------|----------------|
| **Beginners** | Understand what Claude Code is, what it can do, and how to use it |
| **Advanced Developers** | Learn modern CLI tool engineering methods and TypeScript large-scale project architecture |
| **Agent System Designers** | Deep dive into design patterns for Agent Runtime, Tooling, Context Engineering, and extension systems |

---

## Table of Contents

### Preface
- [Preface: Why Read This Book](./00-preface_en.md)

### Part 1: Understanding Claude Code (Beginner-Friendly)
- [Chapter 1: What is Claude Code](./part1/01-introduction_en.md)
- [Chapter 2: Quick Start](./part1/02-quickstart_en.md)

### Part 2: From Early Internet Design to AI Agents
- [Chapter 3: Unix Philosophy and CLI Traditions](./part2/03-unix-philosophy_en.md)
- [Chapter 4: Evolution of REPL](./part2/04-repl-evolution_en.md)
- [Chapter 5: From Chatbot to Agent](./part2/05-from-chatbot-to-agent_en.md)

### Part 3: Architecture Design
- [Chapter 6: Query Engine — The Heart of Conversation](./part3/06-query-engine_en.md)
- [Chapter 7: State Management Design](./part3/07-state-management_en.md)
- [Chapter 8: Message Loop and Streaming](./part3/08-message-loop_en.md)

### Part 4: Tool System Design
- [Chapter 9: Philosophy of Tool System Design](./part4/09-tool-design_en.md)
- [Chapter 10: Overview of 43 Built-in Tools](./part4/10-builtin-tools_en.md)
- [Chapter 11: Tool Permission Model](./part4/11-tool-permission_en.md)

### Part 5: Context Engineering
- [Chapter 12: What is Context Engineering](./part5/12-context-what_en.md)
- [Chapter 13: The Art of System Prompt Construction](./part5/13-system-prompt_en.md)
- [Chapter 14: Memory and CLAUDE.md](./part5/14-memory-claudemd_en.md)
- [Chapter 15: Context Compression (Auto-Compact)](./part5/15-compact_en.md)

### Part 6: Agent Runtime and Multi-Agent Systems
- [Chapter 16: Task System Design](./part6/16-task-system_en.md)
- [Chapter 17: Multi-Agent Architecture](./part6/17-multi-agent_en.md)
- [Chapter 18: Coordinator Pattern](./part6/18-coordinator_en.md)

### Part 7: Extension Systems
- [Chapter 19: MCP Protocol — The Internet of Tools](./part7/19-mcp_en.md)
- [Chapter 20: Skills System](./part7/20-skills_en.md)
- [Chapter 21: Plugin System](./part7/21-plugins_en.md)

### Part 8: Security, Permissions, and Performance
- [Chapter 22: Layered Permission Model Design](./part8/22-permission-model_en.md)
- [Chapter 23: Security Design](./part8/23-security_en.md)
- [Chapter 24: Performance Optimization](./part8/24-performance_en.md)

### Part 9: Design Philosophy
- [Chapter 25: Claude Code Design Principles](./part9/25-design-principles_en.md)
- [Chapter 26: Future Outlook](./part9/26-future_en.md)

---

## How to Read This Book

- **If you're a beginner**: Start from Part 1 and read sequentially
- **If you're a developer**: You can skip Part 1 and start from Part 2
- **If you're an Agent system designer**: Focus on Parts 3, 4, 5, 6, and 7

---

## About the Source Code

This book's analysis is based on the leaked complete TypeScript source code of Claude Code (March 2026). All code references come from the actual source code without speculation.

---

## Advanced Reading

For those who want to dive deeper into Claude Code's source code implementation details, check out our advanced documentation:

**[📚 Claude Code Source Code Architecture Analysis](./architecture/README_EN.md)** | **[中文版本](./architecture/README.md)**

The advanced documentation includes:
- Complete source tree structure (1,884 TypeScript files)
- Detailed explanation of the 6-layer architecture
- Implementation details of Query Engine, Tool System, and Permission Model
- Source code analysis of 40+ tools, 70+ hooks, and 87+ commands
- Complete module dependency graphs and data flow diagrams

---

*This book is open source. Contributions and corrections are welcome.*
