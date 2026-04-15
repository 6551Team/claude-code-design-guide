# 제14장: 메모리(Memory)와 CLAUDE.md

> CLAUDE.md는 Claude를 위한 "프로젝트 매뉴얼"이고, 메모리 시스템(Memory System)은 Claude의 "장기 기억"입니다.

---

## 14.1 AI의 메모리(Memory) 문제

LLM은 본질적으로 무상태(stateless)입니다. 각 API 호출은 독립적이며, 모델은 이전 대화를 기억하지 못합니다.

이것은 AI 프로그래밍 어시스턴트에게 심각한 문제입니다.
- 새 세션마다 Claude는 프로젝트 배경을 모름
- 매번 코딩 표준을 다시 설명해야 함
- 매번 팀 컨벤션을 다시 언급해야 함

Claude Code는 **CLAUDE.md** (명시적 메모리(Memory))와 **메모리 시스템(Memory System)** (자동 메모리(Memory))이라는 두 가지 메커니즘으로 이 문제를 해결합니다.

---

## 14.2 CLAUDE.md: 명시적 프로젝트 메모리(Memory)

CLAUDE.md는 프로젝트 루트(또는 임의의 하위 디렉터리)에 배치하는 마크다운 파일로, Claude Code 시작 시 자동으로 읽힙니다.

**검색 메커니즘** (`src/utils/claudemd.ts`):

```typescript
// 현재 디렉터리에서 위로 올라가며 CLAUDE.md 검색
async function getMemoryFiles(): Promise<string[]> {
  const files = []
  let dir = cwd

  while (dir !== path.dirname(dir)) {  // 루트 디렉터리까지
    const claudeMd = path.join(dir, 'CLAUDE.md')
    if (await fileExists(claudeMd)) {
      files.push(claudeMd)
    }
    dir = path.dirname(dir)
  }

  // ~/.claude/CLAUDE.md (전역 설정)도 확인
  const globalClaudeMd = path.join(os.homedir(), '.claude', 'CLAUDE.md')
  if (await fileExists(globalClaudeMd)) {
    files.push(globalClaudeMd)
  }

  return files
}
```

**다중 레벨 CLAUDE.md**:
```
~/.claude/CLAUDE.md          # 전역 설정 (모든 프로젝트에서 공유)
/project/CLAUDE.md           # 프로젝트 레벨 설정
/project/src/CLAUDE.md       # 하위 디렉터리 레벨 설정 (더 구체적)
/project/src/auth/CLAUDE.md  # 모듈 레벨 설정 (가장 구체적)
```

발견된 모든 CLAUDE.md 파일은 전역에서 구체적인 순서로 읽혀 병합됩니다.

---

## 14.3 CLAUDE.md 모범 사례

좋은 CLAUDE.md에는 다음 내용이 포함되어야 합니다.

```markdown
# 프로젝트명

## 프로젝트 개요
프로젝트가 무엇이며 무엇을 하는지에 대한 간략한 설명.

## 기술 스택
- 언어: TypeScript 5.x
- 프레임워크: Next.js 14
- 데이터베이스: PostgreSQL + Prisma
- 테스트: Jest + Testing Library

## 디렉터리 구조
- src/app/        Next.js App Router 페이지
- src/components/ 재사용 가능한 컴포넌트
- src/lib/        유틸리티 함수 및 서비스
- src/types/      TypeScript 타입 정의

## 코딩 표준
- 클래스 컴포넌트가 아닌 함수형 컴포넌트 사용
- 모든 함수에 TypeScript 타입 필수
- any 타입 사용 금지
- 파일명: kebab-case
- 컴포넌트명: PascalCase

## 공통 명령어
- npm run dev      개발 서버 시작 (포트 3000)
- npm test         테스트 실행
- npm run build    프로덕션 버전 빌드
- npm run lint     ESLint 실행

## 중요 컨벤션
- src/generated/ 디렉터리를 직접 수정하지 말 것
- 데이터베이스 마이그레이션은 반드시 스테이징에서 먼저 검증
- API 라우트에는 반드시 인증 미들웨어 추가
- 모든 사용자 입력은 반드시 유효성 검사

## 현재 작업
- 인증 모듈 리팩토링 중 (src/auth/ 참고)
- TODO: 사용자 권한 시스템
```

---

## 14.4 CLAUDE.md @ 참조 문법

CLAUDE.md는 다른 파일을 참조하는 `@` 문법을 지원합니다.

```markdown
# 프로젝트 설정

## API 명세
@docs/api-spec.md

## 데이터베이스 스키마
@prisma/schema.prisma

## 환경 변수
@.env.example
```

이를 통해 CLAUDE.md가 기존 프로젝트 문서를 참조할 수 있어 중복 유지 관리를 피할 수 있습니다.

---

## 14.5 메모리 시스템(Memory System): 자동 메모리(Memory)

수동으로 유지 관리하는 CLAUDE.md 외에도, Claude Code에는 자동 메모리 시스템(Memory System) (`src/memdir/`)이 있습니다.

**메모리(Memory) 파일 저장 위치**:
```
~/.claude/projects/<project-hash>/memory/
├── user_role.md          # 사용자 역할 정보
├── feedback_testing.md   # 테스트 관련 피드백
├── project_context.md    # 프로젝트 컨텍스트
└── MEMORY.md             # 메모리(Memory) 인덱스
```

**메모리(Memory) 타입** (`src/memdir/memoryTypes.ts`):

```typescript
type MemoryType =
  | 'user'       // 사용자 정보 (역할, 선호도, 지식 배경)
  | 'feedback'   // 사용자 피드백 (해야 할 것/하지 말아야 할 것)
  | 'project'    // 프로젝트 정보 (목표, 제약, 결정)
  | 'reference'  // 외부 리소스 참조
```

---

## 14.6 자동 메모리(Memory) 추출

Claude Code는 대화에서 자동으로 메모리(Memory)를 추출할 수 있습니다 (`src/services/extractMemories/`):

```typescript
// 사용자가 "기억해"라고 말하면 자동으로 저장
User: 기억해: 우리 API는 JWT 인증을 사용하며, 토큰(Token) 유효 기간은 24시간입니다

// Claude가 메모리(Memory) 파일을 생성함
// ~/.claude/projects/.../memory/project_auth.md
---
name: API 인증 설정
type: project
---
API는 JWT 인증을 사용하며, 토큰(Token) 유효 기간은 24시간입니다.
```

**트리거 조건**:
- 사용자가 명시적으로 "기억해", "기록해"라고 말할 때
- 사용자가 Claude의 행동을 수정할 때 ("이렇게 하지 마세요")
- 사용자가 Claude의 비자명한 선택을 확인할 때

---

## 14.7 메모리(Memory) 관련성 검색

모든 메모리(Memory)가 모든 대화에서 로드되지는 않습니다. Claude Code는 관련성 검색을 사용합니다 (`src/memdir/findRelevantMemories.ts`):

```typescript
// 현재 작업을 기반으로 관련 메모리(Memory) 찾기
async function findRelevantMemories(
  currentTask: string,
  allMemories: Memory[]
): Promise<Memory[]> {
  // 단순 키워드 매칭
  // 또는 임베딩 벡터 유사도 사용 (활성화된 경우)
  return allMemories.filter(memory =>
    isRelevant(memory, currentTask)
  )
}
```

이를 통해 모든 메모리(Memory)를 컨텍스트에 넣는 것(토큰(Token) 낭비)을 피하고, 현재 작업과 관련된 메모리(Memory)만 로드합니다.

---

## 14.8 중첩 메모리(Memory): 동적 로드

Claude Code는 중첩 메모리(Memory)의 동적 로드를 지원합니다 (`loadedNestedMemoryPaths`):

```typescript
// 쿼리 엔진(Query Engine)이 로드된 메모리(Memory) 경로를 추적
private loadedNestedMemoryPaths = new Set<string>()

// Claude가 새 디렉터리에 접근할 때 관련 메모리(Memory)를 확인
// 발견되면 컨텍스트에 동적으로 주입
```

이를 통해 메모리 시스템(Memory System)이 Claude의 작업 위치에 따라 컨텍스트를 동적으로 조정할 수 있습니다.

---

## 14.9 팀 메모리(Memory) 동기화

Claude Code는 팀 메모리(Memory) 동기화를 지원합니다 (`src/services/teamMemorySync/`):

```
팀원 A의 메모리(Memory) → 팀 공유 메모리(Memory)로 동기화
팀원 B의 Claude Code → 팀 공유 메모리(Memory) 읽기
```

이를 통해 팀원 모두가 수동으로 CLAUDE.md를 유지 관리하지 않아도 프로젝트 컨벤션, 모범 사례, 일반적인 문제 해결책을 공유할 수 있습니다.

---

## 14.10 CLAUDE.md vs 메모리(Memory): 선택 방법

| 시나리오 | 권장 방법 |
|------|---------|
| 프로젝트 기술 스택 설명 | CLAUDE.md |
| 코딩 표준 | CLAUDE.md |
| 공통 명령어 | CLAUDE.md |
| 개인 작업 선호도 | 메모리(Memory) (user 타입) |
| 프로젝트 결정 기록 | 메모리(Memory) (project 타입) |
| Claude 행동 피드백 | 메모리(Memory) (feedback 타입) |
| 외부 문서 참조 | CLAUDE.md (@ 문법) |
| 임시 컨텍스트 | 대화에서 직접 언급 |

---

## 14.11 메모리 시스템(Memory System) 설계 원칙

Claude Code의 메모리 시스템(Memory System)은 몇 가지 설계 원칙을 따릅니다.

**도출 가능한 정보는 저장하지 않음**: 코드 구조, 파일 경로, git 히스토리 — 이것들은 코드베이스를 읽어서 얻을 수 있으므로 메모리(Memory)에 저장할 필요가 없습니다.

**비자명한 정보를 저장**: 팀 컨벤션, 과거 결정, 개인 선호도 — 이것들은 코드에서 도출할 수 없으므로 저장할 가치가 있습니다.

**메모리(Memory)는 구식이 될 수 있음**: 타임스탬프를 저장하고, 메모리(Memory)가 여전히 유효한지 주기적으로 확인합니다.

**메모리(Memory)는 로그가 아님**: "무엇을 했는지"가 아니라 "왜 그렇게 했는지"를 저장합니다.

---

## 14.12 요약

Claude Code의 메모리 시스템(Memory System)은 두 레이어로 구성됩니다.

- **CLAUDE.md**: 명시적이고 수동으로 유지 관리되는 프로젝트 메모리(Memory), 안정적인 프로젝트 정보에 적합
- **메모리 시스템(Memory System)**: 자동이고 동적인 개인/팀 메모리(Memory), 선호도와 결정 기록에 적합

이 두 가지를 결합하면 Claude가 매 세션에서 배경을 반복적으로 설명받지 않고 빠르게 업무에 투입될 수 있습니다.

---

*다음 챕터: [컨텍스트 압축(Context Compression) (자동 컴팩트(Auto-Compact))](./15-compact_ko.md)*
