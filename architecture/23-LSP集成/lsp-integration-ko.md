# LSP(Language Server Protocol) 통합 아키텍처 문서

> Claude Code v2.1.88 LSP(Language Server Protocol) 통합 — 완전 기술 참조서

---

### 설계 철학

#### 왜 LSP(Language Server Protocol)는 서버가 아닌 클라이언트인가?

Claude Code는 언어 서비스를 제공하는 것이 아니라 IDE에서 언어 서비스(타입 정보, 정의로 이동, 참조 찾기)를 소비합니다. LSP(Language Server Protocol) 프로토콜에서 "클라이언트" 역할을 담당하여 — LSP(Language Server Protocol) 서버 서브프로세스(예: TypeScript의 tsserver, Python의 pylsp)를 생성하고 관리하며, 요청을 전송하고 응답을 수신합니다. 이 아키텍처는 언어 분석 기능을 처음부터 재구현하는 대신 기존의 성숙한 언어 서버 생태계를 재사용합니다.

#### 왜 다중 인스턴스 관리인가?

다른 언어/프로젝트는 다른 LSP(Language Server Protocol) 서버가 필요합니다. `LSPServerManager`는 파일 확장자에서 서버 인스턴스로의 매핑을 유지하며(`createLSPServerManager()`), 여러 독립 LSP(Language Server Protocol) 프로세스를 동시에 실행합니다. 종료 시 `Promise.allSettled`를 사용하여 하나의 서버를 닫는 데 실패해도 다른 서버에 영향을 미치지 않습니다 — 이것이 내결함성의 핵심입니다.

#### 왜 확장자 기반 라우팅인가?

`ensureServerStarted(filePath)`는 파일 확장자를 기반으로 올바른 LSP(Language Server Protocol) 서버를 자동으로 선택합니다 — 사용자가 "이 .ts 파일에 어떤 언어 서버를 사용할지"를 수동으로 설정(Config)할 필요가 없습니다. 라우팅은 투명합니다: 호출자는 파일 경로만 전달하면 되고, 관리자가 서버 선택과 요청 시 시작을 자동으로 처리합니다.

## LSP(Language Server Protocol) 클라이언트 (services/lsp/LSPClient.ts)

### 인터페이스
- `start`: LSP(Language Server Protocol) 서버 서브프로세스 시작
- `initialize`: LSP(Language Server Protocol) initialize 요청 전송
- `sendRequest<T>`: 요청을 전송하고 응답을 기다립니다
- `sendNotification`: 알림 전송 (응답 불필요)
- `onNotification`: 알림 핸들러 등록
- `onRequest`: 요청 핸들러 등록

### 구현
- 통신 프로토콜: **JSON-RPC over stdio** (서브프로세스 stdin/stdout)
- 시작 안전성: 처리되지 않은 거부를 방지하기 위해 스트림을 사용하기 전에 성공적인 스폰을 기다립니다

### 오류 처리
| 종료 코드 | 의미 | 처리 |
|-----------|---------|---------|
| 0 | 의도적인 종료 | 정상 정리 |
| 비제로 | 충돌 | `onCrash` 콜백 트리거 |

### 연결 관리
- 지연 큐 메커니즘: 연결이 아직 준비되지 않은 경우 요청을 버퍼링합니다
- 연결이 준비되면 버퍼링된 요청이 자동으로 전송됩니다

---

## LSP(Language Server Protocol) 서버 관리자 (LSPServerManager.ts)

### 다중 인스턴스 관리
파일 확장자를 기반으로 해당 LSP(Language Server Protocol) 서버 인스턴스로 라우팅합니다.

### 핵심 메서드

#### initialize()
설정(Config)된 모든 LSP(Language Server Protocol) 서버를 로드하고 확장자에서 서버 인스턴스로의 매핑을 빌드합니다.

#### shutdown()
실행 중인 모든 서버를 중지합니다. 내결함성을 위해 `Promise.allSettled`를 사용합니다 — 하나의 서버를 닫는 데 실패해도 다른 서버에 영향을 미치지 않습니다.

#### ensureServerStarted(filePath)
요청 시 시작: 파일 경로의 확장자를 기반으로 해당 LSP(Language Server Protocol) 서버가 시작되었는지 확인합니다.

#### sendRequest\<T\>(filePath, method, params)
요청 라우팅: 파일 경로를 기반으로 해당 LSP(Language Server Protocol) 서버를 찾고, 요청을 전달하고 결과를 반환합니다.

#### 파일 생명주기 알림
| 메서드 | 해당 LSP(Language Server Protocol) 알림 | 목적 |
|--------|-------------------------------|---------|
| `openFile` | `textDocument/didOpen` | 파일 열기 |
| `changeFile` | `textDocument/didChange` | 파일 내용 변경 |
| `saveFile` | `textDocument/didSave` | 파일 저장 |
| `closeFile` | `textDocument/didClose` | 파일 닫기 |

### 파일 추적
동일한 파일에 대해 중복 `didOpen` 알림을 전송하지 않도록 `openedFiles` Map(`fileUri` → `serverName`)을 유지합니다.

---

## 싱글턴 관리 (manager.ts)

### 생명주기 함수

#### initializeLspServerManager()
관리자 인스턴스를 생성하고 비동기적으로 초기화합니다(비차단; 모든 서버의 시작을 기다리지 않습니다).

#### reinitializeLspServerManager()
플러그인(Plugin) 새로고침 시 강제 재초기화 — 이전 인스턴스를 종료하고 새 인스턴스를 생성합니다.

#### shutdownLspServerManager()
최선 노력 종료; 오류는 삼켜지고 전파되지 않습니다.

#### waitForInitialization()
초기화 완료를 기다리며, **30초 타임아웃**이 있습니다.

#### isLspConnected()
적어도 하나의 LSP(Language Server Protocol) 서버가 정상 상태인지 확인합니다.

### generation 카운터
오래된 프로미스를 무효화하는 데 사용됩니다. 관리자가 재초기화될 때, generation 카운터를 통해 이전 초기화 프로미스가 감지되고 폐기됩니다.

### isBareMode()
프로세스가 스크립트 호출 모드(bare 모드)로 실행 중인지 감지합니다; bare 모드에서는 LSP(Language Server Protocol) 초기화가 건너뜁니다.

---

## LSPTool 작업

### 코드 탐색
| 작업 | 목적 |
|-----------|---------|
| `goToDefinition` | 정의로 이동 |
| `findReferences` | 참조 찾기 |
| `goToImplementation` | 구현으로 이동 |

### 코드 정보
| 작업 | 목적 |
|-----------|---------|
| `hover` | 호버 정보 |
| `documentSymbol` | 문서 심볼 |
| `workspaceSymbol` | 워크스페이스 심볼 검색 |

### 호출 계층
| 작업 | 목적 |
|-----------|---------|
| `prepareCallHierarchy` | 호출 계층 준비 |
| `incomingCalls` | 인커밍 호출 |
| `outgoingCalls` | 아웃고잉 호출 |

### 제약

| 제약 | 값 |
|------------|-------|
| 파일 크기 제한 | 10MB |
| 줄 번호 형식 | 1 기반 (줄 번호와 열 번호 모두 1부터 시작) |

---

## 엔지니어링 실천 가이드

### LSP(Language Server Protocol) 서버 설정(Config)

**체크리스트:**

1. **LSP(Language Server Protocol) 서버 설정(Config) 추가**: 설정(Config)에 LSP(Language Server Protocol) 서버의 명령, 인수, 시작 모드를 선언합니다
2. **언어/확장자 매핑 지정**: `LSPServerManager`는 파일 확장자를 기반으로 해당 LSP(Language Server Protocol) 서버로 요청을 라우팅합니다
3. **서버 가용성 확인**: LSP(Language Server Protocol) 서버 바이너리가 PATH에 있는지 또는 절대 경로를 지정합니다
4. **플러그인(Plugin)을 통해 통합**: `lspPluginIntegration.ts`와 `lspRecommendation.ts`가 플러그인(Plugin)의 LSP(Language Server Protocol) 서버 등록을 지원합니다

**라우팅 메커니즘**: `ensureServerStarted(filePath)`는 파일 확장자를 기반으로 LSP(Language Server Protocol) 서버를 자동으로 선택합니다 — 호출자는 파일 경로만 전달하면 되고, 관리자가 서버 선택과 요청 시 시작을 자동으로 처리합니다.

### LSP(Language Server Protocol) 연결 디버깅

**문제 해결 단계:**

1. **LSP(Language Server Protocol) 프로세스가 살아있는지 확인**:
   - LSP(Language Server Protocol) 서버는 stdio(JSON-RPC over stdin/stdout)를 통해 통신하는 서브프로세스로 실행됩니다
   - 종료 코드 0 = 의도적인 종료 (정상 정리); 비제로 = 충돌 (`onCrash` 콜백 트리거)
2. **초기화 타임아웃 확인**: `waitForInitialization()`은 **30초 타임아웃**을 가집니다; 타임아웃은 LSP(Language Server Protocol) 서버 시작 이상을 나타냅니다
3. **연결 상태 확인**: `isLspConnected()`는 적어도 하나의 LSP(Language Server Protocol) 서버가 정상 상태인지 확인합니다
4. **파일 추적 확인**: `openedFiles` Map은 열린 파일을 추적합니다; 대상 파일에 대해 `didOpen` 알림이 전송되었는지 확인합니다
5. **generation 카운터 확인**: 관리자가 재초기화될 때, generation 카운터를 통해 이전 프로미스가 무효화됩니다; 오래된 프로미스가 있는지 확인합니다
6. **bare 모드 확인**: `isBareMode()`가 true를 반환하면 LSP(Language Server Protocol) 초기화가 건너뜁니다

**핵심 소스 위치**:
- `LSPClient.ts` — JSON-RPC 프로토콜 캡슐화, 지연 큐 메커니즘
- `LSPServerManager.ts` — 다중 인스턴스 라우팅, 파일 생명주기 알림
- `manager.ts` — 싱글턴 관리, generation 카운터, 30초 타임아웃

**LSP(Language Server Protocol) 진단 처리 참고** (소스: `passiveFeedback.ts`):
- 진단 핸들러가 연속으로 실패할 때 경고가 발생합니다: "WARNING: LSP diagnostic handler for {serverName} has failed {count} times consecutively"
- LSP(Language Server Protocol) 기능을 컴팩트(Compact) 흐름에 통합하기 위한 TODO는 아직 완료되지 않았습니다 (소스 주석: "TODO: Integrate with compact - call closeFile() when compact removes files from context")

### 파일 생명주기 관리

| LSP(Language Server Protocol) 알림 | 트리거 시점 | 메서드 |
|-----------------|---------------|--------|
| `textDocument/didOpen` | 처음으로 파일이 열릴 때 | `openFile()` |
| `textDocument/didChange` | 파일 내용 변경 | `changeFile()` |
| `textDocument/didSave` | 파일 저장 | `saveFile()` |
| `textDocument/didClose` | 파일 닫기 | `closeFile()` |

**중복 알림 방지**: `openedFiles` Map은 `fileUri → serverName`을 기록하여 동일한 파일에 대해 중복 `didOpen` 알림을 전송하지 않도록 합니다.

### LSP(Language Server Protocol) 기능 확장

**한눈에 보는 사용 가능한 작업:**
- 코드 탐색: `goToDefinition`, `findReferences`, `goToImplementation`
- 코드 정보: `hover`, `documentSymbol`, `workspaceSymbol`
- 호출 계층: `prepareCallHierarchy`, `incomingCalls`, `outgoingCalls`

**파일 크기 제한**: 10MB — 이 제한을 초과하는 파일은 LSP(Language Server Protocol) 서버로 전송되지 않습니다.

### 일반적인 함정

| 함정 | 세부 정보 | 해결 방법 |
|---------|---------|---------|
| LSP(Language Server Protocol) 서버 충돌이 핵심 기능에 영향을 미쳐서는 안 됨 | 충돌은 `onCrash` 콜백을 트리거; 관련 기능은 조용히 저하됩니다 | `shutdownLspServerManager()`는 최선 노력으로; 오류가 전파되지 않습니다 |
| 동시 다중 인스턴스는 리소스 관리가 필요함 | 다른 언어/프로젝트를 위해 여러 독립 LSP(Language Server Protocol) 프로세스가 실행됩니다 | `Promise.allSettled`는 단일 서버 종료 실패가 다른 서버에 영향을 미치지 않도록 보장합니다 |
| 플러그인(Plugin) 새로고침 시 재초기화 | `reinitializeLspServerManager()`는 이전 인스턴스를 종료하고 새 인스턴스를 생성합니다 | generation 카운터가 이전 프로미스를 무효화합니다 |
| Bare 모드에서 LSP(Language Server Protocol) 건너뜀 | 스크립트 호출 모드에서는 코드 인텔리전스가 필요하지 않습니다 | 초기화 전에 `isBareMode()` 검사가 실행됩니다 |
| 지연 큐 | 연결이 준비되지 않은 경우 요청이 버퍼링됩니다 | 연결이 준비되면 자동으로 전송되지만 응답 지연이 발생할 수 있습니다 |


---

[← OAuth & 인증](../22-OAuth与认证/oauth-auth-ko.md) | [인덱스](../README_KO.md) | [샌드박스(Sandbox) 시스템 →](../24-沙箱系统/sandbox-system-ko.md)
