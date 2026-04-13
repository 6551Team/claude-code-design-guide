# 제20장: 스킬(Skills) 시스템

> 스킬(Skills)은 Claude Code의 "매크로"입니다 — 일반적인 워크플로우를 재사용 가능한 커맨드로 캡슐화합니다.

---

## 20.1 스킬(Skills)이란 무엇인가

스킬(Skills)은 `/스킬명` 커맨드를 통해 호출할 수 있는 미리 정의된 프롬프트 템플릿입니다.

스킬(Skills)의 본질: **세심하게 설계된 프롬프트에 선택적 파라미터를 더해, 일반적인 워크플로우를 캡슐화한 것**.

예를 들어, `/commit` 스킬(Skills)의 목적은 현재 git diff를 분석하고, 표준 커밋 메시지를 생성한 뒤, git commit을 실행하는 것입니다.

사용자는 매번 "git diff를 분석해서 Conventional Commits 표준을 따르는 커밋 메시지를 생성하고, git commit을 실행해줘"라고 말할 필요 없이 — `/commit`만 입력하면 됩니다.

---

## 20.2 스킬(Skills) 저장 구조

```
~/.claude/skills/           # 사용자 전역 스킬(Skills)
├── commit.md
├── review.md
└── deploy.md

.claude/skills/             # 프로젝트 수준 스킬(Skills)
├── run-tests.md
└── generate-api-docs.md
```

각 스킬(Skills)은 Markdown 파일입니다:

```markdown
---
name: commit
description: 표준 git 커밋 메시지를 생성하고 커밋 실행
---

현재 git diff를 분석하여 Conventional Commits 표준을 따르는 커밋 메시지를 생성합니다.

표준:
- feat: 새로운 기능
- fix: 버그 수정
- docs: 문서 업데이트
- refactor: 코드 리팩토링
- test: 테스트 관련
- chore: 빌드/도구 관련

단계:
1. git diff --staged를 실행하여 스테이징된 변경사항 확인
2. 변경사항을 분석하여 커밋 타입 결정
3. 간결한 커밋 메시지 생성 (최대 72자)
4. 변경사항이 복잡한 경우 상세 설명 추가
5. git commit 실행
```

---

## 20.3 스킬(Skills) 로딩 메커니즘

`src/skills/` 디렉토리가 스킬(Skills) 로딩을 구현합니다:

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

스킬(Skills)은 세션 시작 시 로딩되어 슬래시 커맨드(Slash Command)로 등록됩니다.

---

## 20.4 내장 스킬(Skills)

Claude Code에는 많은 내장 스킬(Skills)이 있습니다 (`src/skills/bundled/`):

| 스킬(Skills) | 기능 |
|--------------|------|
| `/commit` | git 커밋 생성 및 실행 |
| `/ship` | 전체 릴리즈 프로세스 (테스트→빌드→릴리즈) |
| `/plan` | 구현 계획 생성 |
| `/document-release` | 릴리즈 문서 업데이트 |
| `/investigate` | 체계적인 디버깅 |
| `/retro` | 엔지니어링 회고 |
| `/qa` | QA 테스트 |
| `/browse` | 브라우저 자동화 |

이러한 내장 스킬(Skills)은 일반적인 소프트웨어 개발 워크플로우를 다룹니다.

---

## 20.5 스킬(Skills) 파라미터 전달

스킬(Skills)은 파라미터를 지원합니다:

```bash
# 파라미터와 함께 스킬(Skills) 호출
/commit -m "feat: add user authentication"

# 스킬(Skills)이 내부적으로 파라미터를 사용
# commit.md 내부:
# 사용자가 -m 파라미터를 제공하면 해당 메시지를 직접 사용
# 그렇지 않으면 git diff를 분석하여 메시지 생성
```

파라미터는 `args`를 통해 스킬(Skills)의 프롬프트 템플릿에 전달됩니다.

---

## 20.6 MCP 스킬(Skills): MCP를 통한 스킬(Skills) 배포

`src/skills/mcpSkillBuilders.ts`는 MCP(Model Context Protocol)를 통한 스킬(Skills) 배포를 구현합니다:

```typescript
// MCP 서버가 스킬(Skills)을 제공할 수 있음 (프롬프트로서)
// Claude Code가 이 프롬프트들을 자동으로 스킬(Skills)로 등록
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

이를 통해 팀원들이 파일을 수동으로 복사하지 않고도 MCP 서버를 통해 스킬(Skills)을 공유할 수 있습니다.

---

## 20.7 동적 스킬(Skills) 탐색

Claude Code는 동적 스킬(Skills) 탐색을 지원합니다 (`discoveredSkillNames`):

```typescript
// 쿼리 엔진(Query Engine)이 탐색된 스킬(Skills)을 추적
private discoveredSkillNames = new Set<string>()

// Claude가 대화에서 스킬(Skills)을 언급하면 자동으로 로딩
// 예시: 사용자가 "/plan을 사용해서 구현 계획을 만들어줘"라고 말하면
// Claude Code가 plan 스킬(Skills)의 존재 여부를 확인하고, 있으면 로딩
```

이를 통해 사용 가능한 모든 스킬(Skills)을 미리 알 필요 없이 대화 중에 동적으로 참조할 수 있습니다.

---

## 20.8 커스텀 스킬(Skills) 만들기

커스텀 스킬(Skills) 생성은 매우 간단합니다:

```markdown
<!-- .claude/skills/generate-changelog.md -->
---
name: generate-changelog
description: git log를 기반으로 CHANGELOG 생성
---

마지막 태그부터 현재까지의 모든 커밋을 분석하여 형식에 맞는 CHANGELOG를 생성합니다.

단계:
1. git tag --sort=-version:refname | head -1을 실행하여 최신 태그 확인
2. git log <tag>..HEAD --oneline을 실행하여 새 커밋 목록 확인
3. 타입별로 그룹화 (feat, fix, docs 등)
4. Markdown 형식의 CHANGELOG 생성
5. CHANGELOG.md 파일 앞에 추가
```

저장 후 Claude Code에서 `/generate-changelog`를 입력하면 사용할 수 있습니다.

---

## 20.9 스킬(Skills)과 슬래시 커맨드(Slash Command)의 차이

스킬(Skills)과 슬래시 커맨드(Slash Command) (`/help`, `/clear` 등)의 차이점:

| 구분 | 슬래시 커맨드(Slash Command) | 스킬(Skills) |
|------|--------------------------|--------------|
| 구현 방식 | TypeScript 코드 | Markdown 파일 |
| 실행 방식 | 코드 직접 실행 | Claude에게 프롬프트로 전송 |
| 확장성 | 소스 코드 수정 필요 | 사용자 커스터마이징 가능 |
| 기능 범위 | 시스템 수준 작업 | AI 보조 워크플로우 |
| 예시 | `/clear`, `/cost` | `/commit`, `/plan` |

---

## 20.10 요약

스킬(Skills) 시스템은 Claude Code의 "매크로" 메커니즘입니다:

- **Markdown 파일**: 스킬(Skills)을 Markdown으로 정의하여 작성이 간단합니다
- **다단계 저장**: 전역 스킬(Skills) + 프로젝트 수준 스킬(Skills)
- **MCP 배포**: MCP(Model Context Protocol)를 통해 팀 간 스킬(Skills) 공유
- **동적 탐색**: 대화 중 관련 스킬(Skills)을 동적으로 로딩
- **내장 스킬(Skills)**: 일반적인 개발 워크플로우를 커버

스킬(Skills)을 통해 사용자는 자신의 워크플로우를 재사용 가능한 커맨드로 캡슐화하여 효율성을 크게 향상시킬 수 있습니다.

---

*다음 장: [플러그인(Plugin) 시스템](./21-plugins_ko.md)*
