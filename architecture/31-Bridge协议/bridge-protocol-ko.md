# 브릿지 프로토콜(Bridge Protocol) 아키텍처

브릿지 프로토콜(Bridge Protocol)은 33개의 소스 파일로 구성되며 Claude Code와 원격 환경 간의 양방향 통신 브릿지를 구현합니다.

### 설계 철학

#### 왜 33개 파일의 복잡한 프로토콜인가?

브릿지 프로토콜(Bridge Protocol)은 IDE(VS Code/JetBrains)와 Claude Code 코어를 연결하며, 메시지 직렬화, 상태 동기화, 연결 관리, 오류 복구 등을 처리해야 합니다. 각 파일은 단일 책임을 담당합니다: API 클라이언트(`bridgeApi.ts`), 인증(`bridgeConfig.ts`), 기능 게이팅(`bridgeEnabled.ts`), 메시지 라우팅 및 중복 제거(`bridgeMessaging.ts`), 전송 레이어(WebSocket/SSE/Hybrid) 등. 이 분해는 과잉 엔지니어링이 아닙니다 — 통신 프로토콜의 내재적 복잡성이 요구하는 것입니다. 각 하위 문제("메시지가 손실되면 어떻게 되나?", "인증이 만료되면?", "WebSocket을 사용할 수 없다면?")는 독립적인 해결책이 필요합니다.

#### 왜 REST + WebSocket 이중 채널인가?

두 가지 통신 모드는 자연스럽게 서로 다른 상호작용 패턴에 맞습니다: REST는 요청/응답 의미론(세션 생성, 메시지 전송, 권한 응답 및 기타 일회성 작업)을 처리하고, WebSocket은 이벤트 스트림(스트리밍 출력, 상태 변경, 권한 요청 푸시 및 기타 지속적인 통신)을 처리합니다. 소스 코드에서 `SSETransport.ts`(712줄)와 `WebSocketTransport.ts`(800줄)의 크기는 각 전송 레이어의 독립적인 복잡성을 반영합니다. `HybridTransport.ts`는 두 가지 간의 런타임 전환 및 폴백을 수행합니다.

#### 왜 중복 제거에 BoundedUUIDSet을 사용하는가?

소스 코드 주석에는 이렇게 명시되어 있습니다: "순환 버퍼로 지원되는 FIFO 경계 집합. 용량에 도달하면 가장 오래된 항목을 제거하여 O(capacity)로 메모리 사용량을 일정하게 유지합니다". 네트워크 불신뢰성은 메시지 재전송을 유발할 수 있으며, 특히 재연결 시나리오에서 seq-num 협상이 중복을 생성할 수 있습니다. BoundedUUIDSet은 고정 크기(기본 2000)의 순환 버퍼를 사용하여 이미 처리된 메시지 UUID를 추적합니다. "경계"가 핵심입니다 — 무한정 증가하는 Set은 메모리를 누수시키고, 오래된 메시지 중복 제거는 이미 의미가 없습니다. 외부 순서 메커니즘(`lastWrittenIndexRef`)이 기본 중복 제거 보장을 제공하기 때문입니다. BoundedUUIDSet은 "에코 필터링 및 경쟁 조건 중복 제거를 위한 보조 안전망"일 뿐입니다.

#### 왜 신뢰할 수 있는 전송 레이어인가?

IDE 통합은 메시지를 삭제할 수 없습니다 — 파일 편집 지시가 손실되면 코드가 손상됩니다. 따라서 브릿지 프로토콜(Bridge Protocol)은 완전한 신뢰할 수 있는 전송을 구현합니다: WebSocket 레이어에는 자동 재연결과 메시지 버퍼링이, SSE 레이어에는 시퀀스 번호 기반 재개 기능이 있으며, 두 가지 모두 하트비트 감지와 수면/절전 복구 기능이 있습니다.

---

## 핵심 API (bridgeApi.ts)

### 팩토리 함수

**createBridgeApiClient(deps)**는 인증 재시도 기능을 갖춘 API 클라이언트를 생성하는 핵심 팩토리 함수입니다. `deps` 매개변수는 인증 및 로깅과 같은 의존성을 주입합니다.

### 메서드 세트

- **registerBridgeEnvironment()**: 현재 환경을 브릿지 서비스에 등록합니다
- **pollForWork()**: 대기 중인 작업 태스크를 가져오기 위해 롱 폴링합니다
- **acknowledgeWork()**: 작업 태스크 수신을 확인합니다
- **stopWork()**: 현재 실행 중인 작업을 중지합니다
- **deregisterEnvironment()**: 현재 환경을 등록 해제합니다
- **archiveSession()**: 완료된 세션을 아카이브합니다
- **reconnectSession()**: 중단된 세션을 재연결합니다
- **heartbeatWork()**: 작업을 유지하기 위해 하트비트를 전송합니다
- **sendPermissionResponseEvent()**: 권한 응답 이벤트를 전송합니다

### 오류 처리

**BridgeFatalError** 클래스는 복구 불가능한 치명적 오류를 캡슐화합니다:

- **401**: 인증되지 않음
- **403**: 금지됨
- **404**: 리소스를 찾을 수 없음
- **410**: 리소스 만료됨 (Gone)

### 보안 검증

**validateBridgeId()**: 경로 탐색 공격을 방지하기 위해 브릿지 ID 형식을 검증합니다.

### 오류 분류 유틸리티

- **isExpiredErrorType()**: 오류가 만료 타입인지 결정합니다
- **isSuppressible403()**: 403 오류를 자동으로 억제할 수 있는지 결정합니다 (예: 알려진 권한 제한 시나리오)

---

## 인증 (bridgeConfig.ts)

### 토큰 검색

- **getBridgeTokenOverride()**: ant-only 환경 변수에서 토큰 재정의 값을 읽습니다 (내부 전용)
- **getBridgeAccessToken()**: 액세스 토큰을 검색합니다; 재정의를 우선시하고, 그렇지 않으면 시스템 키체인에서 가져옵니다

### URL 해석

- **getBridgeBaseUrl()**: 환경 변수 재정의를 지원하면서 브릿지 API의 기본 URL을 해석합니다

---

## 기능 게이트 (bridgeEnabled.ts)

### 기본 토글

- **isBridgeEnabled()**: 브릿지가 활성화되었는지 여부 — OAuth 구독자와 GrowthBook 기능 플래그가 모두 필요합니다
- **getBridgeDisabledReason()**: 브릿지가 활성화되지 않은 이유를 설명하는 진단 이유 메시지를 반환합니다

### 서브 기능 토글

- **isEnvLessBridgeEnabled()**: env-less 브릿지 모드가 활성화되었는지 여부
- **isCseShimEnabled()**: CSE 심(shim) 호환성 레이어가 활성화되었는지 여부
- **getCcrAutoConnectDefault()**: CCR 자동 연결의 기본값을 검색합니다
- **isCcrMirrorEnabled()**: CCR 미러 기능이 활성화되었는지 여부

---

## 메시징 (bridgeMessaging.ts)

### 메시지 중복 제거

**BoundedUUIDSet**: 순환 버퍼로 지원되는 FIFO 경계 중복 제거 집합으로, 중복 메시지 처리를 방지합니다. 집합이 가득 찼을 때 가장 오래된 항목이 자동으로 제거됩니다.

### 메시지 라우팅

**handleIngressMessage()**: WebSocket에서 인바운드 메시지를 처리하고 메시지 타입에 따라 해당 핸들러로 디스패치하는 핵심 메시지 라우팅 함수입니다.

### 타입 가드

- **isSDKMessage()**: 값이 SDK 메시지인지 결정합니다
- **isSDKControlResponse()**: 값이 SDK 제어 응답인지 결정합니다
- **isSDKControlRequest()**: 값이 SDK 제어 요청인지 결정합니다

### 제어 요청 처리

**handleServerControlRequest()**: 서버가 시작한 제어 요청을 처리합니다:

- **권한 요청**: 서버가 클라이언트에 권한 확인을 요청합니다
- **모델 전환**: 서버가 활성 모델 전환을 요청합니다
- **중단 요청**: 서버가 현재 작업 중단을 요청합니다

### 세션 아카이빙

**makeResultMessage()**: 세션 아카이빙을 위한 봉투 메시지를 구성하며, 완료된 세션 데이터를 패키지화하고 전송하는 데 사용됩니다.

---

## 전송 레이어

브릿지 프로토콜(Bridge Protocol)은 다양한 네트워크 환경을 수용하기 위해 세 가지 전송 레이어 구현을 제공합니다.

### WebSocketTransport.ts (800줄)

전이중 WebSocket 전송 — 가장 완전한 기능 옵션:

- **자동 재연결**: 연결 해제 후 지수 백오프 전략으로 자동 재연결
- **핑/퐁(Ping/Pong)**: 연결 활성 상태 확인을 위한 하트비트 감지
- **메시지 버퍼링**: 연결 해제 기간 동안 발신 메시지 버퍼링
- **지수 백오프(Exponential Backoff)**: 서버 과부하를 피하기 위해 재연결 간격이 지수적으로 증가
- **수면 감지**: 시스템 수면/절전 해제 이벤트를 감지하고 사전적으로 재연결
- **Keep-alive**: 연결을 유지하기 위한 주기적 하트비트

### SSETransport.ts (712줄)

서버 전송 이벤트(Server-Sent Events) 기반 반이중(half-duplex) 전송:

- **SSE 읽기**: SSE를 사용하여 서버 푸시 메시지를 수신합니다
- **HTTP POST 쓰기**: HTTP POST를 통해 클라이언트 메시지를 전송합니다
- **활성 타임아웃**: 연결이 여전히 활성 상태인지 감지합니다
- **시퀀스 번호 재개**: 시퀀스 번호 기반 중단 지점 재개로 메시지 손실 없음 보장

### HybridTransport.ts

전송 선택 로직 레이어:

- 환경 조건 및 서버 기능에 따라 가장 적합한 전송을 선택합니다
- WebSocket과 SSE 간의 런타임 전환을 지원합니다
- 저하 시나리오를 처리합니다 (WebSocket을 사용할 수 없을 때 SSE로 폴백)

---

## 세션 관리(Session Management)

### 세션 CRUD

- **createBridgeSession()**: `POST /v1/sessions` — 새 세션 생성
- **getBridgeSession()**: `GET /v1/sessions/{id}` — 세션 상세 정보 검색
- **archiveBridgeSession()**: `POST /v1/sessions/{id}/archive` — 세션 아카이브
- **updateBridgeSessionTitle()**: `PATCH /v1/sessions/{id}` — 세션 제목 업데이트

### SpawnMode

```typescript
type SpawnMode = 'single-session' | 'worktree' | 'same-dir'
```

- **single-session**: 단일 세션 모드 — 하나의 환경, 하나의 세션
- **worktree**: git worktree를 사용하여 각 세션에 대한 독립적인 작업 디렉터리 생성
- **same-dir**: 여러 세션이 동일한 디렉터리를 공유

### BridgeWorkerType

```typescript
type BridgeWorkerType = 'claude_code' | 'claude_code_assistant'
```

- **claude_code**: 표준 Claude Code 워커 프로세스
- **claude_code_assistant**: 보조 워커 프로세스 (예: 스웜(Swarm) 팀원)

---

## 타입

브릿지 프로토콜(Bridge Protocol)은 전체 통신 흐름을 지원하는 풍부한 타입 시스템을 정의합니다.

### 작업 관련

- **WorkData**: 작업 태스크의 완전한 데이터 페이로드
- **WorkResponse**: 작업 실행 결과의 응답 구조
- **WorkSecret**: 작업과 관련된 암호화 키 및 민감한 정보

### 세션 활동

- **SessionDoneStatus**: 세션 완료 상태의 열거형
- **SessionActivityType**: 세션 활동 타입 (메시지, 도구 호출, 오류 등)
- **SessionActivity**: 타임스탬프 및 세부 사항을 포함한 세션 활동 기록

### 설정 및 핸들

- **BridgeConfig**: 완전한 브릿지 설정 구조
- **BridgeApiClient**: API 클라이언트 인스턴스 타입
- **SessionHandle**: 활성 세션을 참조하는 데 사용되는 세션 핸들
- **SessionSpawner**: 세션 생성 로직을 캡슐화하는 세션 스포너

---

## 엔지니어링 실천 가이드

### IDE 통합 개발

1. **브릿지 클라이언트 구현**: `createBridgeApiClient(deps)`를 호출하여 API 클라이언트를 생성하고, 인증 및 로깅 의존성을 주입합니다
2. **이중 채널 연결**:
   - REST 채널: 요청/응답 의미론 작업 (세션 생성, 메시지 전송, 권한 응답 등)
   - WebSocket 채널: 이벤트 스트림 (스트리밍 출력, 상태 변경, 권한 요청 푸시 등)
3. **메시지 직렬화 처리**: `isSDKMessage()` / `isSDKControlResponse()` / `isSDKControlRequest()` 타입 가드를 사용하여 올바른 메시지 타입 보장
4. **인증 만료 처리**: `BridgeFatalError`의 401 오류 코드를 수신하고 재인증 흐름 트리거
5. **전송 레이어 선택**: WebSocket을 우선 사용; 사용할 수 없을 때는 `HybridTransport`가 자동으로 SSE로 폴백

### 메시지 손실 디버깅

1. **BoundedUUIDSet이 잘못 중복 플래그를 지정하는지 확인**:
   - BoundedUUIDSet 용량은 기본 2000이며 순환 버퍼를 사용합니다 — 메시지 볼륨이 높을 때 이전 UUID가 제거됩니다
   - 제거 후 동일한 UUID를 가진 메시지가 다시 도착하면 (극단적인 재전송 시나리오) 새 메시지로 처리됩니다 — 이는 일반적으로 문제가 아닙니다
   - 실제 위험은: 두 개의 다른 메시지가 중복으로 잘못 판단되는 것 — UUID 생성에 충돌이 있는지 확인
2. **WebSocket 재연결 상태 확인**:
   - 연결 해제 중에 메시지가 버퍼링되고 재연결 후 일괄 전송됩니다 — 버퍼가 오버플로되었는지 확인
   - 재연결 후 시퀀스 번호 협상이 중복을 생성할 수 있습니다 — BoundedUUIDSet이 바로 이를 위한 보조 중복 제거 보호막입니다
3. **SSE 중단 지점 재개 확인**:
   - SSE 전송 레이어는 시퀀스 번호를 기반으로 재개합니다 — `lastWrittenIndexRef`가 올바르게 업데이트되고 있는지 확인
   - 시퀀스 번호가 연속적이지 않으면 전송 레이어에서 메시지가 손실된 것입니다

### 성능 최적화

1. **메시지 배치 전송**: 가능한 경우 여러 메시지를 단일 네트워크 요청으로 통합하여 HTTP 라운드트립 오버헤드 감소
2. **BoundedUUIDSet 크기 적절하게 설정**: 기본 2000은 대부분의 시나리오에 적합합니다; 고빈도 메시지 시나리오에서는 증가시키되, 메모리 사용량이 O(capacity)임을 주의
3. **하트비트 간격 조정**: 기본 하트비트는 연결 활성 상태를 감지합니다; 너무 자주 보내면 네트워크 오버헤드가 증가하고, 너무 드물게 보내면 연결 해제 감지가 지연됩니다
4. **수면/절전 해제 이벤트 처리**: WebSocket과 SSE 전송 레이어 모두 수면 감지와 시스템 절전 해제 후 사전적 재연결을 제공합니다 — 이 로직이 올바르게 작동하는지 확인하세요. 그렇지 않으면 노트북 뚜껑을 닫았다 열면 UI에 "연결 해제"가 표시됩니다

### 흔한 함정

> **메시지 순서가 중요합니다**: 순서가 뒤바뀐 도착이 정상이라고 가정하지 마세요. 브릿지 프로토콜(Bridge Protocol)은 순서대로 메시지 전달을 보장합니다; 순서가 뒤바뀐 메시지를 관찰한다면 전송 레이어에 버그가 있는 것입니다. `lastWrittenIndexRef`가 기본 순서 보장을 제공합니다; BoundedUUIDSet은 보조 중복 제거 메커니즘일 뿐입니다.

> **브릿지 메시지에는 최대 크기 제한이 있습니다**: 단일 메시지는 임의로 클 수 없습니다 — 특히 대용량 파일 내용을 포함하는 도구 호출 결과의 경우. 메시지가 제한을 초과하면 직렬화가 실패합니다. 대용량 메시지를 분할하거나 내용을 압축하세요.

> **인증 만료의 연쇄 효과**: 브릿지 인증 토큰이 만료되면(401) 진행 중인 모든 요청이 실패합니다. `createBridgeApiClient`에는 인증 재시도 기능이 내장되어 있지만, 토큰이 완전히 유효하지 않은 경우(일시적으로 만료된 것이 아닌) `BridgeFatalError`를 던집니다. 치명적 오류를 루프에서 재시도하지 마세요.

> **validateBridgeId()는 반드시 호출해야 합니다**: 이것은 선택적 검증이 아닙니다 — 브릿지 ID는 API URL 경로를 구성하는 데 직접 사용되며, 검증을 건너뛰면 경로 탐색 공격으로 이어질 수 있습니다. 외부 브릿지 ID를 받는 모든 진입점은 이 함수를 먼저 호출해야 합니다.


---

[← 원격 세션](../30-远程会话/remote-session-ko.md) | [목차](../README_KO.md) | [버디 시스템 →](../32-Buddy系统/buddy-system-ko.md)
