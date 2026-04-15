# 스웜(Swarm) 멀티 에이전트(Multi-Agent) 시스템 아키텍처

## 백엔드 아키텍처

스웜(Swarm) 시스템은 `BackendType` 유니온 타입으로 정의되는 세 가지 백엔드 타입을 지원합니다:

```typescript
type BackendType = 'tmux' | 'iterm2' | 'in-process'
```

### 설계 철학

#### 왜 3개의 백엔드 (tmux/iTerm2/in-process)인가?

다양한 환경은 서로 다른 최적의 솔루션을 요구합니다:
- **tmux**: 헤드리스 환경(CI/SSH)에서 유일한 선택으로 가장 신뢰할 수 있습니다. 소스 코드는 완전한 창 분할, 테두리 색상 지정, 잠금 메커니즘을 구현합니다.
- **iTerm2**: 개발자가 각 에이전트의 터미널 출력을 직접 관찰할 수 있는 시각적 디버깅 시나리오에 이상적입니다.
- **in-process**: 서브프로세스 오버헤드가 없는 테스트 및 경량 시나리오. 소스 코드 주석: "프로세스 기반 팀원(tmux/iTerm2)과 달리, in-process 팀원은 동일한 Node.js 프로세스에서 실행됩니다" — 메모리를 공유하지만 논리적으로 격리되며, 각 팀원은 자체 독립적인 AbortController를 가집니다.

백엔드 감지는 계층적 우선순위 방식을 사용합니다: 이미 tmux 안에 있음 > iTerm2 사용 가능 > tmux로 폴백 > 오류로, 가장 적합한 백엔드가 자동으로 선택됩니다.

#### 왜 IPC 대신 파일 시스템을 권한 동기화에 사용하는가?

소스 파일 `permissionSync.ts`는 `writePermissionRequest()` / `readPendingPermissions()` / `resolvePermission()`을 통해 파일 수준 권한 요청-응답 흐름을 구현합니다. IPC 파이프/소켓 대신 파일 시스템을 선택한 세 가지 이유:
1. **크로스 프로세스/크로스 머신 범용성** — tmux 창, SSH 세션, in-process 모드 모두 파일에 접근할 수 있습니다.
2. **감사 로그** — 권한 요청과 결정이 파일로 지속되어 권한 결정 히스토리를 사후에 검사할 수 있습니다.
3. **크래시 복구** — 파일 시스템은 프로세스 크래시에서 살아남지만, IPC 파이프/소켓은 프로세스와 함께 죽어 복구 후 이전 권한 상태를 알 수 없게 됩니다.

#### 왜 17개의 환경 변수가 상속되는가?

소스 코드 주석에 설명되어 있습니다: "Tmux는 부모의 환경을 상속하지 않는 새 로그인 셸을 시작할 수 있으므로, 현재 프로세스에서 설정된 것들을 전달합니다." `TEAMMATE_ENV_VARS`에는 API 프로바이더 선택(`CLAUDE_CODE_USE_BEDROCK` 등), 프록시 설정, CA 인증서 경로 및 기타 중요한 변수가 포함됩니다. 이것들을 상속하지 않으면 팀원들이 기본적으로 firstParty 엔드포인트로 설정되어 잘못된 주소로 요청을 전송합니다 (소스는 "GitHub 이슈 #23561"을 참조합니다).

### PaneBackend 인터페이스

`PaneBackend`는 모든 백엔드 구현이 따라야 하는 핵심 인터페이스입니다. 창 생명주기 및 시각적 제어 메서드를 정의합니다:

- **createTeammatePaneInSwarmView(name, command)**: 새 팀원 창을 생성하고 스웜(Swarm) 뷰에 표시합니다.
- **sendCommandToPane(paneId, command)**: 지정된 창에 실행할 명령을 전송합니다.
- **setPaneBorderColor(paneId, color)**: 시각적 구별을 위해 창 테두리 색상을 설정합니다.
- **setPaneTitle(paneId, title)**: 창 제목을 설정합니다.
- **killPane(paneId)**: 창을 종료하고 소멸시킵니다.
- **hidePane(paneId)**: 프로세스를 유지하면서 창을 숨깁니다.
- **showPane(paneId)**: 이전에 숨겨진 창을 표시합니다.
- **rebalancePanes()**: 모든 창의 레이아웃 할당을 재조정합니다.

### TeammateExecutor 인터페이스

`TeammateExecutor`는 팀원 프로세스의 생명주기를 관리합니다:

- **spawn(config)**: 새 팀원 프로세스를 시작합니다.
- **sendMessage(id, message)**: 지정된 팀원에게 메시지를 전송합니다.
- **terminate(id)**: 팀원을 정상적으로 종료합니다.
- **kill(id)**: 팀원을 강제로 종료합니다.
- **isActive(id)**: 팀원이 여전히 활성 상태로 실행 중인지 확인합니다.

### 백엔드 감지 우선순위

시스템은 계층적 감지를 통해 가장 적합한 백엔드를 결정합니다:

1. **tmux 안에 있음** (최우선): 이미 tmux 세션 안에서 실행 중으로 감지됩니다.
2. **it2가 있는 iTerm2**: iTerm2 안에서 `it2` CLI를 사용하여 실행 중으로 감지됩니다.
3. **tmux로 폴백**: tmux가 시스템에 설치되어 있지만 현재 셸이 tmux 안에 없습니다.
4. **오류**: 사용 가능한 백엔드 없음; 오류가 발생합니다.

### 캐싱 메커니즘

반복적인 감지 오버헤드를 피하기 위해 시스템은 다음 캐시를 유지합니다:

- **cachedBackend**: 이미 초기화된 백엔드 인스턴스를 캐시합니다.
- **cachedDetectionResult**: 백엔드 타입 감지 결과를 캐시합니다.
- **inProcessFallbackActive**: 시스템이 in-process 모드로 폴백했는지 표시합니다.

---

## 권한 동기화

스웜(Swarm)에서 리더와 팀원 간의 권한 동기화는 여러 통신 메커니즘을 사용합니다.

### 파일 수준 권한 흐름

파일 시스템 기반 권한 요청-응답 흐름:

- **writePermissionRequest()**: 팀원이 공유 디렉터리에 권한 요청을 씁니다.
- **readPendingPermissions()**: 리더가 폴링하여 대기 중인 권한 요청을 읽습니다.
- **resolvePermission()**: 리더가 권한 해결 결과를 씁니다.

### 메일박스 수준 권한 흐름

메일박스 메커니즘 기반의 비동기 권한 통신:

- **sendPermissionRequestViaMailbox()**: 팀원이 메일박스를 통해 권한 요청을 전송합니다.
- **sendPermissionResponseViaMailbox()**: 리더가 메일박스를 통해 권한 응답을 반환합니다.
- **sendSandboxPermissionRequestViaMailbox()**: 샌드박스 환경을 위한 전용 권한 요청.

### SwarmPermissionRequest 스키마

```typescript
interface SwarmPermissionRequest {
  id: string              // 고유 요청 식별자
  workerId: string        // 요청하는 워커의 ID
  workerName: string      // 워커의 사람이 읽을 수 있는 이름
  toolName: string        // 권한을 요청하는 도구 이름
  status: 'pending' | 'approved' | 'denied'  // 현재 상태
}
```

### 리더 브릿지(Bridge)

리더 측은 UI 상호작용을 위한 권한 확인 큐를 유지합니다:

- **registerLeaderToolUseConfirmQueue()**: 리더의 도구 사용 확인 큐를 등록합니다.
- **getLeaderToolUseConfirmQueue()**: 등록된 확인 큐를 검색합니다.

### 폴링 간격

```typescript
const PERMISSION_POLL_INTERVAL_MS = 500
```

---

## In-Process 팀원

In-process 모드는 tmux/iTerm2 없이 동일한 프로세스 내에서 팀원을 독립적인 에이전트로 실행합니다.

### 권한 처리

**createInProcessCanUseTool()**은 권한 처리 함수를 생성합니다:

- 권한 확인을 위해 브릿지(Bridge) 채널을 우선 사용합니다.
- 브릿지(Bridge)를 사용할 수 없을 때 메일박스 방식으로 폴백합니다.
- 특정 bash 명령을 자동 승인하기 위해 분류기(Classifier) 자동 승인 메커니즘을 통합합니다.

### 프로세스 관리

**spawnInProcessTeammate()**는 in-process 팀원을 시작합니다:

- 추적 가능성을 보장하기 위해 결정론적 agentId 생성을 사용합니다.
- 각 팀원은 자체 독립적인 AbortController를 가져 개별 종료를 지원합니다.
- 호스트 프로세스와 메모리 공간을 공유하지만 논리적으로 격리됩니다.

**killInProcessTeammate()**는 in-process 팀원을 종료합니다:

- `AbortController.abort()`를 트리거하여 실행을 중지합니다.
- 팀 파일에서 멤버 레코드를 제거합니다.
- 관련 리소스를 정리합니다.

### InProcessBackend 클래스

`InProcessBackend`는 `TeammateExecutor` 인터페이스의 완전한 메서드 세트를 구현합니다:

- **spawn**: 현재 프로세스 내에 새 에이전트 인스턴스를 생성합니다.
- **sendMessage**: 인메모리 채널을 통해 메시지를 전달합니다.
- **terminate**: 에이전트를 정상적으로 중지합니다.
- **kill**: 에이전트를 강제로 중지합니다.
- **isActive**: 에이전트의 실행 상태를 확인합니다.

---

## 팀 관리

### TeamFile 구조

```typescript
interface TeamFile {
  members: TeamMember[]    // 팀 멤버 목록
  leaderId: string         // 리더의 에이전트 ID
  allowedPaths: string[]   // 팀에서 공유하는 허용 경로
  hiddenPanes: string[]    // 현재 숨겨진 창 ID 목록
}
```

### 이름 정제

- **sanitizeName(name)**: 불법 문자를 제거하여 일반 이름 문자열을 정제합니다.
- **sanitizeAgentName(name)**: 명명 제약을 충족하도록 에이전트 이름을 특별히 정제합니다.

### 파일 작업

- **readTeamFile()**: 팀 파일을 동기적으로 읽습니다.
- **writeTeamFileAsync(data)**: 팀 파일을 비동기적으로 씁니다 (원자성 보장).
- **removeTeammateFromTeamFile(id)**: 팀 파일에서 지정된 팀원을 제거합니다.

---

## Tmux 백엔드

### Tmux 내부 모드

이미 tmux 세션 안에서 실행 중으로 감지된 경우:

- 현재 창을 두 영역으로 분할합니다.
- **리더 영역은 30%**를 차지하며 왼쪽에 위치합니다.
- **팀원 영역은 70%**를 차지하며 오른쪽에 위치합니다.
- 팀원 창은 오른쪽 영역 내에서 추가로 분할됩니다.

### Tmux 외부 모드

외부 tmux 세션을 시작해야 하는 경우:

- 외부 세션 소켓을 사용하여 연결합니다.
- 팀원을 관리하기 위한 별도의 tmux 세션을 생성합니다.

### 초기화 지연

```typescript
const PANE_SHELL_INIT_DELAY_MS = 200
```

창 생성 후 셸이 초기화를 완료할 때까지 200ms 대기합니다.

### 잠금 메커니즘

잠금 메커니즘은 창이 순차적으로 생성되도록 보장합니다:

- 동시 창 생성으로 인한 레이아웃 손상을 방지합니다.
- 각 창의 셸 초기화가 완료된 후 다음 창이 생성되도록 보장합니다.

---

## 환경 상속

### TEAMMATE_ENV_VARS

팀원 프로세스에 전달해야 하는 17개의 중요한 환경 변수를 정의하여, 팀원들이 리더의 런타임 환경 설정을 상속받도록 보장합니다.

### buildInheritedCliFlags()

팀원 CLI에 전달되는 인수 플래그를 구성합니다:

- **permission mode**: 권한 모드 설정.
- **model**: 사용할 모델 식별자.
- **settings**: 설정 파일 경로.
- **plugin-dir**: 플러그인(Plugin) 디렉터리.
- **teammate-mode**: 프로세스가 팀원 모드에서 실행 중임을 표시.
- **chrome flags**: Chrome/브라우저 관련 플래그.

### buildInheritedEnvVars()

팀원에게 전달되는 환경 변수 세트를 구성합니다:

- **CLAUDECODE=1**: 프로세스가 Claude Code 환경에서 실행 중임을 표시.
- **API provider vars**: API 프로바이더 관련 변수 (키, 엔드포인트 등).
- **proxy config**: 프록시 설정.
- **CA certs**: CA 인증서 경로 설정.

---

## 팀원 초기화

### initializeTeammateHooks()

팀원 초기화 중 생명주기 훅(Hooks)을 등록합니다:

- **정지 훅(Stop Hook)** 등록: 팀원이 정지할 때 리더에게 알립니다.
- 정지 알림을 받으면 리더는 태스크를 재할당하거나 리소스를 정리할 수 있습니다.

### 권한 규칙 적용

- 팀 파일에서 `allowedPaths` 설정을 읽습니다.
- 팀 범위의 허용 경로를 팀원에게 권한 규칙으로 적용합니다.
- 팀원이 승인된 파일 경로에만 접근할 수 있도록 보장합니다.

### 유휴 알림

- 태스크를 완료한 후 팀원은 유휴 알림을 전송합니다.
- 알림에는 태스크 실행 요약이 포함됩니다.
- 메일박스 메커니즘을 통해 리더에게 전달됩니다.
- 리더는 유휴 알림을 사용하여 새 태스크를 할당할지 결정합니다.

---

## 엔지니어링 실천 가이드

### 에이전트 스웜(Swarm) 생성

1. **TeamCreateTool을 통해 팀 생성**: 팀 이름 및 멤버 설정을 정의합니다.
2. **백엔드 선택**:
   - **tmux**: CI/SSH/헤드리스 환경에 선호되는 선택 — 가장 신뢰할 수 있으며 크로스 플랫폼 (tmux가 미리 설치되어 있어야 함).
   - **iTerm2**: 시각적 디버깅 시나리오 — 개발자가 각 에이전트의 터미널 출력을 직접 관찰할 수 있습니다; macOS 전용으로 `it2` CLI가 필요합니다.
   - **in-process**: 테스트 및 경량 시나리오 — 서브프로세스 오버헤드 없음, 메모리 공유하지만 논리적으로 격리됨, 각 팀원은 자체 독립적인 AbortController를 가집니다.
3. **태스크 할당**: 각 팀원에게 특정 태스크 설명과 도구 권한을 할당합니다.
4. **자동 백엔드 감지**: 수동으로 지정하지 않으면 시스템이 우선순위에 따라 자동으로 선택합니다: 이미 tmux 안에 있음 > iTerm2 사용 가능 > tmux로 폴백 > 오류.

### 권한 동기화 디버깅

1. **파일 수준 권한 흐름 확인**:
   - `~/.claude/teams/{teamName}/permissions/` 디렉터리 아래의 권한 파일을 확인합니다.
   - `pending` 파일: 팀원의 대기 중인 권한 요청.
   - `resolved` 파일: 리더의 권한 해결 결과.
2. **메일박스 수준 권한 흐름 확인**:
   - 파일 수준 권한 흐름이 작동하지 않는다면, 메일박스 메커니즘이 올바르게 작동하는지 확인합니다.
   - `sendPermissionRequestViaMailbox()`가 성공적으로 전송되었는가?
   - `sendPermissionResponseViaMailbox()`가 성공적으로 반환되었는가?
3. **리더 브릿지(Bridge) 확인**:
   - `getLeaderToolUseConfirmQueue()`가 유효한 큐를 반환하는가?
   - `registerLeaderToolUseConfirmQueue()`를 통해 리더의 확인 큐가 등록되었는지 확인합니다.
4. **폴링 간격**: `PERMISSION_POLL_INTERVAL_MS = 500ms` — 권한 응답이 500ms 이상 걸리면 팀원이 이미 다음 폴링 사이클을 시작했을 수 있습니다.

### 환경 변수 상속

17개의 환경 변수가 리더에서 워커로 전달됩니다 — 팀원이 비정상적으로 동작하면 (예: 잘못된 API 엔드포인트에 연결) 환경 변수 상속을 먼저 확인하세요:

1. `TEAMMATE_ENV_VARS` 목록을 확인하여 필요한 변수가 포함되어 있는지 확인합니다.
2. `buildInheritedEnvVars()`의 출력에 예상 변수 값이 포함되어 있는지 확인합니다.
3. 중요 변수: `CLAUDE_CODE_USE_BEDROCK`, 프록시 설정, CA 인증서 경로 — 이것들이 없으면 팀원들이 잘못된 주소로 요청을 전송합니다.
4. CLI 인수는 `buildInheritedCliFlags()`를 통해 전달됩니다: 권한 모드, 모델, settings, plugin-dir, teammate-mode.

### 백엔드 선택 결정 트리

```
CI/헤드리스 환경에서 실행해야 하는가?
├─ 예 → tmux (설치되어 있는지 확인)
└─ 아니오
   시각적 디버깅이 필요한가?
   ├─ 예 → macOS인가? → iTerm2 (it2 CLI 사용 가능 여부 확인)
   │        macOS가 아님 → tmux
   └─ 아니오
      테스트/경량 시나리오인가?
      ├─ 예 → in-process
      └─ 아니오 → tmux (가장 범용적)
```

### 흔한 함정

> **tmux는 미리 설치되어 있어야 합니다**: 스웜(Swarm)은 기본적으로 tmux로 폴백하지만, tmux가 시스템에 설치되어 있지 않으면 즉시 오류가 발생합니다. CI 환경에서는 Docker 이미지에 tmux가 포함되어 있는지 확인하세요.

> **iTerm2는 macOS 전용입니다**: `it2` CLI는 iTerm2 전용입니다 — macOS가 아닌 시스템에서 iTerm2 백엔드를 선택하면 실패합니다. 백엔드 감지 우선순위가 이를 자동으로 처리하지만, 수동으로 지정할 때는 주의하세요.

> **NFS에서 파일 잠금이 신뢰할 수 없을 수 있습니다**: 권한 동기화는 파일 시스템 작업에 의존합니다. 팀 디렉터리가 NFS 또는 다른 네트워크 파일 시스템에 있으면 파일 잠금과 원자적 쓰기의 의미론이 보장되지 않을 수 있습니다 — 이는 권한 경쟁 조건으로 이어질 수 있습니다. 로컬 파일 시스템 사용이 권장됩니다.

> **창 초기화 지연**: tmux 창이 생성된 후 셸 초기화를 위해 `PANE_SHELL_INIT_DELAY_MS = 200ms`의 대기 시간이 필요합니다. 동시 창 생성은 잠금 메커니즘을 사용하여 순서를 보장합니다 — "명령을 찾을 수 없음" 오류가 관찰되면 초기화 지연이 충분하지 않을 수 있습니다.

> **in-process 모드에서의 메모리 공유**: tmux/iTerm2와 달리, in-process 팀원들은 리더와 동일한 Node.js 프로세스 메모리를 공유합니다. 논리적으로 격리되어 있지만 (각자 자체 AbortController를 가짐), 한 팀원의 메모리 누수는 다른 모든 팀원과 리더에게 영향을 미칩니다.


---

[← 코디네이터 패턴](../33-协调器模式/coordinator-mode-ko.md) | [목차](../README_KO.md) | [컴퓨터 사용 →](../35-Computer-Use/computer-use-ko.md)
