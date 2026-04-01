# Chapter 2: Quick Start

> The best way to learn is by doing.

---

## 2.1 Installation

Claude Code requires Node.js 18+ or Bun runtime.

```bash
# Install via npm
npm install -g @anthropic-ai/claude-code

# Verify installation
claude --version
```

First run requires API Key configuration:

```bash
claude
# You'll be prompted to enter your Anthropic API Key
# Or set environment variable
export ANTHROPIC_API_KEY=sk-ant-...
```

---

## 2.2 Three Usage Modes

### Mode 1: Interactive REPL

The most common mode, chat with Claude like a conversation:

```bash
claude
```

Once inside, you can:
- Directly input questions or tasks
- Use `/` prefix for slash commands
- Use `Ctrl+C` to interrupt current operation
- Use `Ctrl+D` or type `/exit` to quit

### Mode 2: Single Execution

Suitable for script integration or quick tasks:

```bash
# Execute task directly
claude "Explain the structure of this project"

# Non-interactive mode (doesn't wait for user input)
claude --print "List all TypeScript files"
```

### Mode 3: Pipeline Mode

Read content from standard input:

```bash
# Analyze git diff
git diff | claude "Explain these changes"

# Analyze logs
cat error.log | claude "Find the cause of errors"
```

---

## 2.3 Common Slash Commands

In interactive mode, slash commands are shortcuts to control Claude Code behavior:

| Command | Purpose |
|---------|---------|
| `/help` | Show help |
| `/clear` | Clear conversation history |
| `/compact` | Compress conversation context |
| `/cost` | Show token usage and cost for this conversation |
| `/config` | View or modify configuration |
| `/model` | Switch model |
| `/commit` | Generate git commit message and commit |
| `/exit` | Exit |

---

## 2.4 First Practice: Analyze Project Structure

Suppose you have a new project and want to quickly understand it:

```
> Help me analyze the overall structure of this project, including main modules and their responsibilities
```

Claude Code will:
1. Use `GlobTool` to scan file structure
2. Read key files (`package.json`, `README.md`, entry files)
3. Give you a clear project overview

---

## 2.5 Second Practice: Fix a Bug

```
> Run npm test, find the failing tests and fix them
```

Claude Code will:
1. Execute `npm test`
2. Analyze error output
3. Locate relevant source files
4. Modify code
5. Run tests again to verify

---

## 2.6 Third Practice: Add a Feature

```
> Add a getUserByEmail method to UserService,
  Requirements: parameter validation, error handling, unit tests
```

Claude Code will:
1. Find the `UserService` file
2. Understand existing code style
3. Implement the method
4. Find the test file
5. Add corresponding unit tests

---

## 2.7 CLAUDE.md: Instructions for Claude

Create a `CLAUDE.md` file in the project root directory. Claude Code will read it every time it starts, using it as project context:

```markdown
# Project Description

## Tech Stack
- Node.js 20 + TypeScript
- PostgreSQL + Prisma ORM
- Jest testing framework

## Code Standards
- Use ESLint + Prettier
- Function naming uses camelCase
- File naming uses kebab-case

## Common Commands
- `npm run dev` Start development server
- `npm test` Run tests
- `npm run build` Build production version

## Notes
- Don't directly modify files in generated/ directory
- Database migrations need to be verified in staging environment first
```

This is one of Claude Code's most important configurations. A good `CLAUDE.md` helps Claude understand your project more accurately and reduces mistakes.

---

## 2.8 Permission Configuration

Claude Code will ask you before executing dangerous operations by default. You can adjust through configuration:

```bash
# View current permission configuration
claude /config

# Allow automatic bash command execution (use with caution)
claude --dangerously-skip-permissions
```

Permission levels from low to high:
1. **Default**: Ask before dangerous operations
2. **Auto-approve**: Specific tools auto-approved
3. **Skip permissions**: All operations auto-executed (only for trusted automation scenarios)

---

## 2.9 Best Practices for Multi-File Projects

**Give Claude enough context**:
```
> I want to modify the user authentication flow. Related files are in src/auth/ directory,
  database schema is in prisma/schema.prisma,
  please read these files before starting modifications
```

**Execute complex tasks step by step**:
```
> Step 1: First help me analyze the existing authentication flow, don't modify any files
> (After confirming analysis is correct)
> Step 2: Now modify according to the plan we discussed
```

**Use `/compact` to manage context**:
After long conversations, use `/compact` to compress history, retain key information, and free up token space.

---

## 2.10 Summary

You now know:
- How to install and start Claude Code
- Three usage modes (interactive, single, pipeline)
- Common slash commands
- How to configure project context with `CLAUDE.md`
- Basic permission configuration

Next, we'll dive deep into Claude Code's design philosophy. But before that, let's review history—understanding where Claude Code comes from helps us better understand why it's designed this way.

---

*Next Chapter: [Unix Philosophy and CLI Tradition](../part2/03-unix-philosophy_en.md)*