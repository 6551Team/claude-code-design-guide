# 第 2 章：快速上手

> 最好的学习方式是动手。

---

## 2.1 安装

Claude Code 需要 Node.js 18+ 或 Bun 运行时。

```bash
# 通过 npm 安装
npm install -g @anthropic-ai/claude-code

# 验证安装
claude --version
```

首次运行需要配置 API Key：

```bash
claude
# 会提示你输入 Anthropic API Key
# 或者设置环境变量
export ANTHROPIC_API_KEY=sk-ant-...
```

---

## 2.2 三种使用模式

### 模式一：交互式 REPL

最常用的模式，像聊天一样与 Claude 对话：

```bash
claude
```

进入后，你可以：
- 直接输入问题或任务
- 用 `/` 开头输入斜杠命令
- 用 `Ctrl+C` 中断当前操作
- 用 `Ctrl+D` 或输入 `/exit` 退出

### 模式二：单次执行

适合脚本集成或快速任务：

```bash
# 直接执行任务
claude "解释一下这个项目的结构"

# 非交互模式（不等待用户输入）
claude --print "列出所有 TypeScript 文件"
```

### 模式三：管道模式

从标准输入读取内容：

```bash
# 分析 git diff
git diff | claude "解释这些改动"

# 分析日志
cat error.log | claude "找出错误原因"
```

---

## 2.3 常用斜杠命令

在交互模式下，斜杠命令是控制 Claude Code 行为的快捷方式：

| 命令 | 作用 |
|------|------|
| `/help` | 显示帮助 |
| `/clear` | 清空对话历史 |
| `/compact` | 压缩对话上下文 |
| `/cost` | 显示本次对话的 token 消耗和费用 |
| `/config` | 查看或修改配置 |
| `/model` | 切换模型 |
| `/commit` | 生成 git commit 信息并提交 |
| `/exit` | 退出 |

---

## 2.4 第一个实战：分析项目结构

假设你有一个新项目，想快速了解它：

```
> 帮我分析这个项目的整体结构，包括主要模块和它们的职责
```

Claude Code 会：
1. 用 `GlobTool` 扫描文件结构
2. 读取关键文件（`package.json`、`README.md`、入口文件）
3. 给你一个清晰的项目概览

---

## 2.5 第二个实战：修复 Bug

```
> 运行 npm test，找出失败的测试并修复
```

Claude Code 会：
1. 执行 `npm test`
2. 分析错误输出
3. 定位相关源文件
4. 修改代码
5. 再次运行测试验证

---

## 2.6 第三个实战：添加功能

```
> 给 UserService 添加一个 getUserByEmail 方法，
  要求：参数验证、错误处理、单元测试
```

Claude Code 会：
1. 找到 `UserService` 文件
2. 理解现有代码风格
3. 实现方法
4. 找到测试文件
5. 添加对应的单元测试

---

## 2.7 CLAUDE.md：给 Claude 的说明书

在项目根目录创建 `CLAUDE.md` 文件，Claude Code 每次启动都会读取它，作为项目上下文：

```markdown
# 项目说明

## 技术栈
- Node.js 20 + TypeScript
- PostgreSQL + Prisma ORM
- Jest 测试框架

## 代码规范
- 使用 ESLint + Prettier
- 函数命名用 camelCase
- 文件命名用 kebab-case

## 常用命令
- `npm run dev` 启动开发服务器
- `npm test` 运行测试
- `npm run build` 构建生产版本

## 注意事项
- 不要直接修改 generated/ 目录下的文件
- 数据库迁移需要先在 staging 环境验证
```

这是 Claude Code 最重要的配置之一。一个好的 `CLAUDE.md` 能让 Claude 更准确地理解你的项目，减少误操作。

---

## 2.8 权限配置

Claude Code 默认会在执行危险操作前询问你。你可以通过配置调整：

```bash
# 查看当前权限配置
claude /config

# 允许自动执行 bash 命令（谨慎使用）
claude --dangerously-skip-permissions
```

权限级别从低到高：
1. **默认**：危险操作前询问
2. **自动批准**：特定工具自动批准
3. **跳过权限**：所有操作自动执行（仅用于受信任的自动化场景）

---

## 2.9 多文件项目的最佳实践

**给 Claude 足够的上下文**：
```
> 我要修改用户认证流程。相关文件在 src/auth/ 目录，
  数据库 schema 在 prisma/schema.prisma，
  请先读取这些文件再开始修改
```

**分步骤执行复杂任务**：
```
> 第一步：先帮我分析现有的认证流程，不要修改任何文件
> （确认分析正确后）
> 第二步：现在按照我们讨论的方案修改
```

**利用 `/compact` 管理上下文**：
长对话后，用 `/compact` 压缩历史，保留关键信息，释放 token 空间。

---

## 2.10 小结

你现在知道了：
- 如何安装和启动 Claude Code
- 三种使用模式（交互、单次、管道）
- 常用斜杠命令
- 如何用 `CLAUDE.md` 配置项目上下文
- 基本的权限配置

接下来，我们要深入理解 Claude Code 的设计思想。但在此之前，让我们先回顾一下历史——理解 Claude Code 从哪里来，才能更好地理解它为什么这样设计。

---

*下一章：[Unix 哲学与 CLI 的传统](../part2/03-unix-philosophy.md)*
