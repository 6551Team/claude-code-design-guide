# 제21장: 플러그인(Plugin) 시스템

> 플러그인(Plugin) 시스템은 Claude Code의 개방성을 구현합니다 — 서드파티 개발자가 핵심 기능을 확장할 수 있게 합니다.

---

## 21.1 플러그인(Plugin) 시스템 설계 목표

Claude Code의 플러그인(Plugin) 시스템 (`src/plugins/`, `src/services/plugins/`)은 여러 설계 목표를 가집니다:

1. **확장성**: 서드파티가 핵심 코드를 수정하지 않고 새로운 기능을 추가할 수 있도록 허용
2. **격리**: 플러그인(Plugin) 오류가 핵심 시스템에 영향을 미치지 않음
3. **보안**: 플러그인(Plugin) 권한이 제한됨
4. **탐색 가능성**: 사용자가 플러그인(Plugin)을 쉽게 찾고 설치할 수 있음

---

## 21.2 플러그인(Plugin) 유형

Claude Code는 두 가지 플러그인(Plugin) 유형을 지원합니다:

**내장 플러그인(Built-in plugins)** (`src/plugins/builtinPlugins.ts`):
Claude Code와 함께 배포되는 공식 플러그인(Plugin)으로, 전체 시스템 권한을 보유합니다.

**서드파티 플러그인(Third-party plugins)** (마켓플레이스 경유):
사용자가 설치하는 외부 플러그인(Plugin)으로, 권한이 제한되며 명시적인 사용자 승인이 필요합니다.

---

## 21.3 플러그인(Plugin) 훅(Hooks) 시스템

플러그인(Plugin)은 훅(Hooks) 시스템을 통해 Claude Code와 통합됩니다:

```typescript
// src/types/plugin.ts
type PluginHookMatcher = {
  event: HookEvent          // 트리거 시점
  matcher?: string          // 매칭 조건 (선택사항)
  handler: PluginHandler    // 핸들러 함수
}

type HookEvent =
  | 'PreToolUse'            // 도구 호출(Tool Call) 이전
  | 'PostToolUse'           // 도구 호출(Tool Call) 이후
  | 'UserPromptSubmit'      // 사용자가 메시지를 제출할 때
  | 'Stop'                  // 대화가 종료될 때
  | 'Notification'          // 알림을 전송할 때
```

플러그인(Plugin)은 이러한 핵심 시점에 커스텀 로직을 주입할 수 있습니다:

```typescript
// 예시: 모든 도구 호출(Tool Call)을 기록하는 플러그인(Plugin)
const auditPlugin: Plugin = {
  name: 'audit-logger',
  hooks: [
    {
      event: 'PostToolUse',
      handler: async ({ toolName, input, result }) => {
        await appendToAuditLog({
          timestamp: Date.now(),
          tool: toolName,
          input,
          success: !result.is_error,
        })
      }
    }
  ]
}
```

---

## 21.4 플러그인(Plugin) 설정

플러그인(Plugin)은 `settings.json`을 통해 설정합니다:

```json
{
  "plugins": {
    "audit-logger": {
      "enabled": true,
      "logFile": "~/.claude/audit.log"
    },
    "custom-formatter": {
      "enabled": true,
      "style": "compact"
    }
  }
}
```

---

## 21.5 플러그인(Plugin) 보안 모델

플러그인(Plugin) 보안 모델은 **최소 권한 원칙**을 기반으로 합니다:

```typescript
type PluginPermissions = {
  canReadFiles: boolean      // 파일 읽기 가능 여부
  canWriteFiles: boolean     // 파일 쓰기 가능 여부
  canExecuteCommands: boolean // 명령어 실행 가능 여부
  canAccessNetwork: boolean  // 네트워크 접근 가능 여부
  canModifySettings: boolean // 설정 수정 가능 여부
}
```

사용자는 플러그인(Plugin) 설치 시 각 권한을 명시적으로 승인해야 합니다.

---

## 21.6 내장 플러그인(Plugin) 예시

Claude Code에는 몇 가지 중요한 내장 플러그인(Plugin)이 있습니다:

**AutoUpdater** (`src/components/AutoUpdater.tsx`):
Claude Code 업데이트를 자동으로 확인하고 설치합니다.

**PromptSuggestion** (`src/services/PromptSuggestion/`):
현재 컨텍스트를 기반으로 프롬프트 제안을 제공합니다.

**SessionMemory** (`src/services/SessionMemory/`):
세션 메모리 영속성을 관리합니다.

**AgentSummary** (`src/services/AgentSummary/`):
에이전트 실행의 요약 보고서를 생성합니다.

---

## 21.7 플러그인(Plugin) vs MCP vs 스킬(Skills)

세 가지 확장 시스템(Extension System) 비교:

| 구분 | 플러그인(Plugin) | MCP | 스킬(Skills) |
|------|----------------|-----|--------------|
| 구현 언어 | TypeScript | 모든 언어 | Markdown |
| 통합 깊이 | 깊음 (훅(Hooks) 시스템) | 중간 (도구/리소스) | 얕음 (프롬프트 템플릿) |
| 개발 난이도 | 높음 | 중간 | 낮음 |
| 사용 사례 | 핵심 기능 확장 | 외부 서비스 통합 | 워크플로우 캡슐화 |
| 권한 요구사항 | 높음 | 중간 | 낮음 |

---

## 21.8 요약

Claude Code의 플러그인(Plugin) 시스템은 깊이 있는 확장 기능을 제공합니다:

- **훅(Hooks) 시스템**: 핵심 시점에 커스텀 로직 주입
- **권한 모델(Permission Model)**: 최소 권한 원칙, 명시적 사용자 승인
- **두 가지 유형**: 내장 플러그인(공식) + 서드파티 플러그인(사용자 설치)

세 가지 확장 시스템(Extension System) (플러그인(Plugin), MCP, 스킬(Skills))은 깊은 통합부터 경량 캡슐화까지 전체 스펙트럼을 커버합니다.

---

*다음 장: [계층적 권한 모델(Permission Model) 설계](../part8/22-permission-model_ko.md)*
