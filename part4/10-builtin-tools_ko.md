# 제10장: 43개 내장 도구(Built-in Tools) 개요

> 도구 세트의 설계는 "AI가 무엇을 할 수 있는가"에 대한 시스템의 판단을 반영합니다.

---

## 10.1 도구 분류 개요

Claude Code에는 43개의 내장 도구(Built-in Tools)가 있으며, 8개의 주요 카테고리로 구분됩니다.

| 카테고리 | 도구 수 | 핵심 책임 |
|----------|---------|-----------|
| 파일 작업 | 5 | 파일 읽기/쓰기, 내용 및 경로 검색 |
| 셸(Shell) 실행 | 3 | 시스템 명령어 실행 |
| 코드 인텔리전스(Code Intelligence) | 2 | LSP 통합, 노트북(Notebook) 편집 |
| 작업 관리 | 6 | 백그라운드 작업의 전체 생명주기 |
| 멀티 에이전트(Multi-Agent) 협업 | 4 | 서브 에이전트(Sub-Agent), 팀, 메시지 전달 |
| 플랜 모드(Plan Mode) | 4 | 플랜 및 워크트리(Worktree) 진입/종료 |
| MCP(Model Context Protocol) 통합 | 4 | MCP 프로토콜 도구 및 리소스 |
| 확장 및 기타 | 15 | 스킬(Skills), 예약 작업, 웹(Web), 설정 등 |

---

## 10.2 파일 작업 도구

### FileReadTool
**책임**: 파일 내용 읽기

```typescript
// 입력
{
  file_path: string      // 파일 경로
  offset?: number        // 시작 줄 (대용량 파일의 분할 읽기용)
  limit?: number         // 읽을 줄 수
}

// 출력: 파일 내용 (줄 번호 포함)
```

**설계 하이라이트**: 분할 읽기(`offset` + `limit`)를 지원하여 대용량 파일을 한 번에 읽을 때 발생하는 토큰 초과를 방지합니다. Claude는 처음 100줄을 읽은 후 필요에 따라 더 읽을 수 있습니다.

---

### FileEditTool
**책임**: 정밀한 문자열 교체

```typescript
// 입력
{
  file_path: string      // 파일 경로
  old_string: string     // 교체할 내용 (고유해야 함)
  new_string: string     // 교체될 내용
}
```

**설계 하이라이트**: `old_string`이 파일에서 고유하게 존재할 것을 요구합니다. 이 제약은 엄격해 보이지만 실제로는 안전 장치입니다. Claude가 잘못된 위치를 실수로 수정하는 것을 방지합니다. `old_string`이 고유하지 않으면 도구가 오류를 발생시키며, Claude는 정확한 위치를 지정하기 위해 더 많은 컨텍스트를 제공해야 합니다.

---

### FileWriteTool
**책임**: 파일 생성 또는 완전 덮어쓰기

```typescript
// 입력
{
  file_path: string      // 파일 경로
  content: string        // 파일 내용
}
```

**FileEditTool과의 차이**: `FileWriteTool`은 전체 파일을 덮어쓰고, `FileEditTool`은 특정 내용만 수정합니다. 새 파일에는 `FileWriteTool`을 사용하고, 기존 파일 수정에는 `FileEditTool`을 선호합니다(더 안전하고 정밀).

---

### GlobTool
**책임**: 패턴으로 파일 경로 검색

```typescript
// 입력
{
  pattern: string        // 글로브(glob) 패턴, 예: "**/*.ts"
  path?: string          // 검색 루트 디렉터리
}

// 출력: 일치하는 파일 경로 목록 (수정 시간순 정렬)
```

**설계 하이라이트**: 결과가 수정 시간순으로 정렬되어 가장 최근에 수정된 파일이 먼저 표시됩니다. Claude에게 유용한데, 보통 가장 최근에 수정된 파일이 가장 관련성이 높기 때문입니다.

---

### GrepTool
**책임**: 파일 내용 검색

```typescript
// 입력
{
  pattern: string        // 정규 표현식
  path?: string          // 검색 디렉터리
  glob?: string          // 파일 필터 패턴
  output_mode?: 'content' | 'files_with_matches' | 'count'
  context?: number       // 매치 전후로 표시할 줄 수
}
```

**설계 하이라이트**: ripgrep 기반으로 매우 빠릅니다. 세 가지 출력 모드를 지원하며, Claude는 필요에 따라 선택할 수 있습니다. 어떤 파일에 매치가 있는지만 알면 되는 경우(`files_with_matches`), 또는 특정 내용을 확인해야 하는 경우(`content`).

---

## 10.3 셸(Shell) 실행 도구

### BashTool
**책임**: 셸(Shell) 명령어 실행

```typescript
// 입력
{
  command: string        // 셸 명령어
  timeout?: number       // 타임아웃 (밀리초)
  description?: string   // 명령어 설명 (사용자에게 표시)
}

// 출력: stdout + stderr + 종료 코드
```

**보안 설계**: BashTool은 임의의 명령어를 실행할 수 있기 때문에 가장 위험한 도구입니다. Claude Code는 BashTool에 특별한 보안 처리를 적용합니다.
- 기본적으로 사용자 확인 필요
- 명령어 안전성 분석 (예: `rm -rf` 같은 위험한 명령어 감지)
- 명령어 중단을 방지하기 위한 타임아웃 지원
- 샌드박스(Sandbox) 모드에서 제한

---

### PowerShellTool
**책임**: PowerShell 명령어 실행 (Windows)

BashTool과 유사하지만 Windows 환경을 위한 도구입니다.

---

### REPLTool
**책임**: 대화형 REPL 실행

```typescript
// 입력
{
  code: string           // 실행할 코드
  language?: string      // 언어 (python, node 등)
}
```

**BashTool과의 차이**: REPLTool은 지속적인 REPL 세션을 유지하여 여러 번 호출해도 변수가 유지됩니다. 다단계 계산이 필요한 시나리오에 적합합니다.

---

## 10.4 코드 인텔리전스(Code Intelligence) 도구

### LSPTool
**책임**: 언어 서버 프로토콜(Language Server Protocol) 통합

```typescript
// 지원되는 작업
type LSPOperation =
  | 'goToDefinition'      // 정의로 이동
  | 'findReferences'      // 참조 찾기
  | 'hover'               // 호버 문서
  | 'documentSymbol'      // 문서 심볼 목록
  | 'workspaceSymbol'     // 워크스페이스 심볼 검색
  | 'goToImplementation'  // 구현으로 이동
  | 'prepareCallHierarchy'// 호출 계층 준비
  | 'incomingCalls'       // 수신 호출
  | 'outgoingCalls'       // 발신 호출
```

**설계 하이라이트**: LSP 통합으로 Claude는 단순한 텍스트 검색이 아닌 실제 코드 이해를 수행할 수 있습니다. "`getUserById`를 호출하는 모든 위치 찾기"에 `findReferences`를 사용하면 `GrepTool`보다 정확합니다(이름 변경, 별칭 등 처리).

---

### NotebookEditTool
**책임**: Jupyter 노트북(Notebook) 편집

```typescript
// 입력
{
  notebook_path: string  // 노트북 경로
  cell_number?: number   // 대상 셀 (0 인덱스)
  new_source: string     // 새 셀 내용
  edit_mode?: 'replace' | 'insert' | 'delete'
  cell_type?: 'code' | 'markdown'
}
```

---

## 10.5 작업 관리 도구

작업 관리 도구는 Claude Code의 백그라운드 작업 시스템에 대한 인터페이스입니다.

| 도구 | 책임 |
|------|------|
| `TaskCreateTool` | 백그라운드 작업 생성 (bash 명령어 또는 서브 에이전트) |
| `TaskGetTool` | 단일 작업 상태 조회 |
| `TaskListTool` | 모든 작업 목록 조회 |
| `TaskOutputTool` | 작업 출력 읽기 |
| `TaskStopTool` | 작업 중지 |
| `TaskUpdateTool` | 작업 설명 업데이트 |

**사용 사례**: 장시간 실행되는 작업(빌드, 테스트, 데이터 처리 등)을 백그라운드 작업으로 실행하면 Claude가 다른 작업을 계속 처리하면서 주기적으로 작업 상태를 확인할 수 있습니다.

---

## 10.6 멀티 에이전트(Multi-Agent) 협업 도구

### AgentTool
**책임**: 서브 에이전트(Sub-Agent) 실행

```typescript
// 입력
{
  description: string    // 서브 에이전트의 작업 설명
  prompt: string         // 서브 에이전트의 초기 프롬프트
  subagent_type?: string // 에이전트 타입 (범용, Explore 등)
  isolation?: 'worktree' // 격리된 워크트리(Worktree)에서 실행할지 여부
  model?: string         // 서브 에이전트가 사용할 모델
  run_in_background?: boolean // 백그라운드에서 실행할지 여부
}
```

**설계 하이라이트**: 서브 에이전트는 독립적인 도구 세트, 컨텍스트(Context), 실행 환경을 가집니다. 부모 에이전트는 여러 서브 에이전트를 병렬로 실행하여 진정한 병렬 처리를 달성할 수 있습니다.

---

### TeamCreateTool / TeamDeleteTool
**책임**: 협업 에이전트 팀 생성/삭제

팀은 여러 에이전트의 집합으로, `SendMessageTool`을 통해 서로 통신할 수 있습니다.

---

### SendMessageTool
**책임**: 다른 에이전트에게 메시지 전송

에이전트 간 비동기 통신을 구현하며, 멀티 에이전트(Multi-Agent) 협업의 기반입니다.

---

## 10.7 플랜 모드(Plan Mode) 도구

### EnterPlanModeTool / ExitPlanModeTool
**책임**: 플랜 모드(Plan Mode) 진입/종료

플랜 모드에서 Claude는 계획만 생성할 수 있고 도구를 실행할 수 없습니다. 실행 전 사용자가 계획을 검토해야 하는 시나리오에 사용합니다.

```
사용자: 전체 인증 모듈을 리팩터링해 줘

Claude (플랜 모드):
  제 계획은 다음과 같습니다:
  1. 기존 인증 흐름 분석
  2. 새 인터페이스 설계
  3. 점진적 마이그레이션(Migration)

  실행을 승인하시겠습니까?

사용자: 승인

Claude (실행 모드): 실행을 시작합니다...
```

---

### EnterWorktreeTool / ExitWorktreeTool
**책임**: Git 워크트리(Worktree) 진입/종료

메인 브랜치에 영향을 주지 않고 격리된 워크트리에서 작업합니다. 실험적 수정이나 병렬 개발에 적합합니다.

---

## 10.8 MCP(Model Context Protocol) 통합 도구

| 도구 | 책임 |
|------|------|
| `MCPTool` | MCP 서버가 제공하는 도구 호출 |
| `McpAuthTool` | MCP 서버 인증 |
| `ListMcpResourcesTool` | MCP 리소스 목록 조회 |
| `ReadMcpResourceTool` | MCP 리소스 읽기 |

MCP(Model Context Protocol)는 Anthropic이 제안한 오픈 프로토콜로, 외부 서버가 Claude에게 도구와 리소스를 제공할 수 있게 합니다. 자세한 내용은 제19장을 참조하세요.

---

## 10.9 기타 주요 도구

### WebFetchTool
**책임**: 웹 페이지 내용 가져오기

```typescript
// 입력
{
  url: string            // URL
  prompt: string         // 페이지에서 추출할 정보
}
```

**설계 하이라이트**: 단순히 HTML을 반환하지 않고 AI를 사용하여 페이지 내용을 처리하고 사용자가 필요한 정보를 추출합니다.

---

### WebSearchTool
**책임**: 인터넷 검색

```typescript
// 입력
{
  query: string          // 검색 쿼리(Query)
  allowed_domains?: string[]  // 이 도메인만 검색
  blocked_domains?: string[]  // 이 도메인 제외
}
```

---

### TodoWriteTool
**책임**: 작업 목록 관리

```typescript
// 입력
{
  todos: Array<{
    content: string
    status: 'pending' | 'in_progress' | 'completed'
    activeForm: string
  }>
}
```

**설계 하이라이트**: TodoWriteTool은 Claude의 "작업 메모리"입니다. 복잡한 다단계 작업의 경우 Claude는 TodoWriteTool을 사용하여 진행 상황을 기록하고, 어떤 단계도 누락되지 않도록 합니다.

---

### AskUserQuestionTool
**책임**: 사용자에게 질문하기

```typescript
// 입력
{
  questions: Array<{
    question: string
    header: string
    options: Array<{ label: string; description: string }>
    multiSelect?: boolean
  }>
}
```

**설계 하이라이트**: 구조화된 질문 형식으로 사용자가 자유 텍스트가 아닌 옵션을 통해 답변할 수 있습니다. 이는 모호성을 줄이고 상호작용 효율성을 높입니다.

---

### SkillTool
**책임**: 미리 정의된 스킬(Skills) 실행

스킬(Skills)은 재사용 가능한 프롬프트 템플릿입니다. 자세한 내용은 제20장을 참조하세요.

---

### ScheduleCronTool / RemoteTriggerTool
**책임**: 예약 작업 및 원격 트리거(Remote Trigger)

Claude가 예약 작업을 생성하거나 원격 에이전트(Agent) 실행을 트리거할 수 있게 합니다.

---

## 10.10 도구의 진화

Claude Code의 도구 세트는 정적이지 않습니다. 소스 코드의 마이그레이션(Migration) 파일(`src/migrations/`)을 통해 도구의 진화 역사를 확인할 수 있습니다.

- Opus에서 Sonnet 4.5, 이후 Sonnet 4.6으로의 모델 마이그레이션(Migration)
- 권한 시스템(Permission System)의 여러 차례 리팩터링
- 지속적인 새 도구 추가

이 진화 능력은 도구 시스템(Tool System)의 좋은 추상화에서 비롯됩니다. 새 도구를 추가하려면 `Tool` 인터페이스를 구현하기만 하면 되고, 핵심 시스템을 수정할 필요가 없습니다.

---

## 10.11 요약

43개의 내장 도구(Built-in Tools)는 소프트웨어 개발의 전체 워크플로우를 포괄합니다.

- **탐색**: GlobTool, GrepTool, LSPTool
- **이해**: FileReadTool, WebFetchTool
- **수정**: FileEditTool, FileWriteTool
- **실행**: BashTool, REPLTool
- **협업**: AgentTool, TeamCreateTool, SendMessageTool
- **관리**: TaskCreateTool, TodoWriteTool, ScheduleCronTool
- **확장**: MCPTool, SkillTool

이 도구 세트의 설계 원칙: **각 도구는 하나의 일을 하고, Claude가 조율을 담당합니다.**

---

*다음 장: [도구 권한 모델(Tool Permission Model)](./11-tool-permission_ko.md)*
