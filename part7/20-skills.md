# 第 20 章：Skills 系统

> Skills 是 Claude Code 的"宏"——把常用的工作流封装成可复用的命令。

---

## 20.1 什么是 Skills

Skills 是预定义的提示模板，可以通过 `/skill-name` 命令调用。

一个 Skill 本质上是：**一段精心设计的提示，加上可选的参数，封装了一个常用的工作流**。

例如，`/commit` Skill 的作用是：分析当前的 git diff，生成规范的 commit message，并执行 git commit。

用户不需要每次都说"帮我分析 git diff，生成符合 Conventional Commits 规范的 commit message，然后执行 git commit"——只需要输入 `/commit`。

---

## 20.2 Skills 的存储结构

```
~/.claude/skills/           # 用户全局 Skills
├── commit.md
├── review.md
└── deploy.md

.claude/skills/             # 项目级 Skills
├── run-tests.md
└── generate-api-docs.md
```

每个 Skill 是一个 Markdown 文件：

```markdown
---
name: commit
description: 生成规范的 git commit message 并提交
---

分析当前的 git diff，生成符合 Conventional Commits 规范的 commit message。

规范：
- feat: 新功能
- fix: bug 修复
- docs: 文档更新
- refactor: 代码重构
- test: 测试相关
- chore: 构建/工具相关

步骤：
1. 运行 git diff --staged 查看暂存的变更
2. 分析变更内容，确定 commit 类型
3. 生成简洁的 commit message（不超过 72 字符）
4. 如果变更复杂，添加详细描述
5. 执行 git commit
```

---

## 20.3 Skills 的加载机制

`src/skills/` 目录实现了 Skills 的加载：

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

Skills 在会话开始时加载，注册为斜杠命令。

---

## 20.4 内置 Skills

Claude Code 有大量内置 Skills（`src/skills/bundled/`）：

| Skill | 功能 |
|-------|------|
| `/commit` | 生成并执行 git commit |
| `/ship` | 完整的发布流程（测试→构建→发布） |
| `/plan` | 生成实现计划 |
| `/document-release` | 更新发布文档 |
| `/investigate` | 系统性调试 |
| `/retro` | 工程回顾 |
| `/qa` | QA 测试 |
| `/browse` | 浏览器自动化 |

这些内置 Skills 覆盖了软件开发的常见工作流。

---

## 20.5 Skills 的参数传递

Skills 支持参数：

```bash
# 带参数调用 Skill
/commit -m "feat: add user authentication"

# Skill 内部使用参数
# 在 commit.md 中：
# 如果用户提供了 -m 参数，直接使用该 message
# 否则，分析 git diff 生成 message
```

参数通过 `args` 传递给 Skill 的提示模板。

---

## 20.6 MCP Skills：通过 MCP 分发 Skills

`src/skills/mcpSkillBuilders.ts` 实现了通过 MCP 协议分发 Skills：

```typescript
// MCP 服务器可以提供 Skills（作为 Prompts）
// Claude Code 自动将这些 Prompts 注册为 Skills
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

这让团队可以通过 MCP 服务器共享 Skills，而不需要每个人手动复制文件。

---

## 20.7 Skills 的动态发现

Claude Code 支持动态发现 Skills（`discoveredSkillNames`）：

```typescript
// QueryEngine 追踪已发现的 Skills
private discoveredSkillNames = new Set<string>()

// 当 Claude 在对话中提到某个 Skill 时，自动加载
// 例如：用户说"用 /plan 帮我制定实现计划"
// Claude Code 会检查是否有 plan Skill，如果有则加载
```

这让 Skills 可以在对话中被动态引用，而不需要提前知道所有可用的 Skills。

---

## 20.8 创建自定义 Skills

创建自定义 Skill 非常简单：

```markdown
<!-- .claude/skills/generate-changelog.md -->
---
name: generate-changelog
description: 根据 git log 生成 CHANGELOG
---

分析从上一个 tag 到现在的所有 commit，生成格式化的 CHANGELOG。

步骤：
1. 运行 git tag --sort=-version:refname | head -1 获取最新 tag
2. 运行 git log <tag>..HEAD --oneline 获取新 commit
3. 按类型分组（feat、fix、docs 等）
4. 生成 Markdown 格式的 CHANGELOG
5. 追加到 CHANGELOG.md 文件开头
```

保存后，在 Claude Code 中输入 `/generate-changelog` 即可使用。

---

## 20.9 Skills vs 斜杠命令

Skills 和斜杠命令（`/help`、`/clear` 等）的区别：

| 维度 | 斜杠命令 | Skills |
|------|---------|--------|
| 实现方式 | TypeScript 代码 | Markdown 文件 |
| 执行方式 | 直接执行代码 | 作为提示发送给 Claude |
| 可扩展性 | 需要修改源码 | 用户可以自定义 |
| 能力范围 | 系统级操作 | AI 辅助工作流 |
| 示例 | `/clear`、`/cost` | `/commit`、`/plan` |

---

## 20.10 小结

Skills 系统是 Claude Code 的"宏"机制：

- **Markdown 文件**：Skills 用 Markdown 定义，简单易写
- **多级存储**：全局 Skills + 项目级 Skills
- **MCP 分发**：通过 MCP 协议在团队间共享 Skills
- **动态发现**：对话中动态加载相关 Skills
- **内置 Skills**：覆盖常见开发工作流

Skills 让用户可以把自己的工作流封装成可复用的命令，大幅提升效率。

---

*下一章：[插件系统](./21-plugins.md)*
