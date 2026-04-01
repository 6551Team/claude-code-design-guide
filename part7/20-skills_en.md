# Chapter 20: Skills System

> Skills are Claude Code's "macros" — encapsulating common workflows into reusable commands.

---

## 20.1 What are Skills

Skills are predefined prompt templates that can be invoked through `/skill-name` commands.

A Skill is essentially: **a carefully designed prompt, plus optional parameters, encapsulating a common workflow**.

For example, the `/commit` Skill's purpose is: analyze current git diff, generate standard commit message, and execute git commit.

Users don't need to say "help me analyze git diff, generate a commit message following Conventional Commits standard, then execute git commit" every time — just type `/commit`.

---

## 20.2 Skills Storage Structure

```
~/.claude/skills/           # User global Skills
├── commit.md
├── review.md
└── deploy.md

.claude/skills/             # Project-level Skills
├── run-tests.md
└── generate-api-docs.md
```

Each Skill is a Markdown file:

```markdown
---
name: commit
description: Generate standard git commit message and commit
---

Analyze current git diff, generate commit message following Conventional Commits standard.

Standards:
- feat: new feature
- fix: bug fix
- docs: documentation update
- refactor: code refactoring
- test: test-related
- chore: build/tool-related

Steps:
1. Run git diff --staged to view staged changes
2. Analyze changes, determine commit type
3. Generate concise commit message (max 72 characters)
4. If changes are complex, add detailed description
5. Execute git commit
```

---

## 20.3 Skills Loading Mechanism

`src/skills/` directory implements Skills loading:

```typescript
// src/skills/loadSkillsDir.ts
async function loadSkillsDir(dir: string): Promise<Skill[]> {
  const files = await glob('**/*.md', { cwd: dir })

  return Promise.all(files.map(async file => {
    const content = await readFile(path.join(dir, file))
    const { frontmatter, body } = parseFrontmatter(content)

    return {
      name: frontmatter.name || path.basename(file, '.md'),
      description: frontmatter.description,
      prompt: body,
      filePath: path.join(dir, file),
    }
  }))
}
```

Skills are loaded at session start and registered as slash commands.

---

## 20.4 Built-in Skills

Claude Code has many built-in Skills (`src/skills/bundled/`):

| Skill | Function |
|-------|------|
| `/commit` | Generate and execute git commit |
| `/ship` | Complete release process (test→build→release) |
| `/plan` | Generate implementation plan |
| `/document-release` | Update release documentation |
| `/investigate` | Systematic debugging |
| `/retro` | Engineering retrospective |
| `/qa` | QA testing |
| `/browse` | Browser automation |

These built-in Skills cover common software development workflows.

---

## 20.5 Skills Parameter Passing

Skills support parameters:

```bash
# Call Skill with parameters
/commit -m "feat: add user authentication"

# Skill uses parameters internally
# In commit.md:
# If user provides -m parameter, use that message directly
# Otherwise, analyze git diff to generate message
```

Parameters are passed to Skill's prompt template through `args`.

---

## 20.6 MCP Skills: Distributing Skills via MCP

`src/skills/mcpSkillBuilders.ts` implements Skills distribution through MCP protocol:

```typescript
// MCP servers can provide Skills (as Prompts)
// Claude Code automatically registers these Prompts as Skills
function buildMCPSkills(mcpClients: MCPServerConnection[]): Skill[] {
  return mcpClients.flatMap(client =>
    client.prompts.map(prompt => ({
      name: `${client.name}:${prompt.name}`,
      description: prompt.description,
      prompt: prompt.template,
      source: 'mcp',
    }))
  )
}
```

This allows teams to share Skills through MCP servers without everyone manually copying files.

---

## 20.7 Dynamic Skills Discovery

Claude Code supports dynamic Skills discovery (`discoveredSkillNames`):

```typescript
// QueryEngine tracks discovered Skills
private discoveredSkillNames = new Set<string>()

// When Claude mentions a Skill in conversation, automatically load it
// Example: user says "use /plan to help me create implementation plan"
// Claude Code checks if plan Skill exists, loads it if found
```

This allows Skills to be dynamically referenced in conversations without needing to know all available Skills in advance.

---

## 20.8 Creating Custom Skills

Creating custom Skills is very simple:

```markdown
<!-- .claude/skills/generate-changelog.md -->
---
name: generate-changelog
description: Generate CHANGELOG based on git log
---

Analyze all commits from last tag to now, generate formatted CHANGELOG.

Steps:
1. Run git tag --sort=-version:refname | head -1 to get latest tag
2. Run git log <tag>..HEAD --oneline to get new commits
3. Group by type (feat, fix, docs, etc.)
4. Generate Markdown format CHANGELOG
5. Prepend to CHANGELOG.md file
```

After saving, type `/generate-changelog` in Claude Code to use it.

---

## 20.9 Skills vs Slash Commands

Differences between Skills and slash commands (`/help`, `/clear`, etc.):

| Dimension | Slash Commands | Skills |
|------|---------|--------|
| Implementation | TypeScript code | Markdown files |
| Execution | Direct code execution | Sent as prompt to Claude |
| Extensibility | Requires source code modification | User-customizable |
| Capability scope | System-level operations | AI-assisted workflows |
| Examples | `/clear`, `/cost` | `/commit`, `/plan` |

---

## 20.10 Summary

The Skills system is Claude Code's "macro" mechanism:

- **Markdown files**: Skills defined in Markdown, simple to write
- **Multi-level storage**: Global Skills + project-level Skills
- **MCP distribution**: Share Skills across teams via MCP protocol
- **Dynamic discovery**: Dynamically load relevant Skills in conversation
- **Built-in Skills**: Cover common development workflows

Skills allow users to encapsulate their workflows into reusable commands, greatly improving efficiency.

---

*Next chapter: [Plugin System](./21-plugins_en.md)*
