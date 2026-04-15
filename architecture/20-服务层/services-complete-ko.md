# 서비스 레이어 — 완전 참조서

Claude Code의 서비스 레이어에는 13개의 독립 서비스가 포함되어 있으며, 각각 특정 백그라운드 기능을 담당합니다. 이 문서는 각 서비스의 목적, 핵심 함수 서명, 임계값/상수, 게이팅 조건, 오류 처리, 상태 관리에 대한 포괄적인 설명을 제공합니다.

### 설계 철학

왜 13개의 독립 서비스이지 더 적은 모듈로 병합하지 않는가? 각 서비스는 독립적인 생명주기와 리소스 요구사항을 가지고 있기 때문입니다 — LSP(Language Server Protocol)는 오래 지속되는 연결을 유지하고 여러 서브프로세스를 관리해야 하고, OAuth는 토큰 갱신과 키체인 상호작용이 필요하며, Analytics는 비동기 이벤트 큐가 필요하고, autoDream은 PID 잠금과 세션 간 상태가 필요합니다. 병합하면 불필요한 결합이 생성됩니다: 하나의 서비스 충돌이나 재시작은 다른 서비스에 영향을 미쳐서는 안 됩니다. 또한 이러한 서비스들은 시작 타이밍이 다릅니다 — 일부(LSP(Language Server Protocol) 같은)는 코드 파일이 처음 열릴 때 요청 시 시작되고, 일부(policyLimits 같은)는 애플리케이션 시작 시 즉시 시작되며, 일부(autoDream 같은)는 세션이 유휴 상태일 때만 실행됩니다. 독립 서비스는 각 모듈이 자체 생명주기를 자율적으로 관리할 수 있게 합니다.

---

## 1. AgentSummary

### 목적

백그라운드에서 현재 세션을 주기적으로 요약하여, 서브에이전트와 컨텍스트 압축을 위한 고품질 대화 요약을 제공합니다.

### 핵심 메커니즘

- **30초 주기**: 포크된 서브프로세스를 통해 30초마다 요약 생성 실행
- **캐시 공유**: 요약 결과는 캐싱 시스템을 통해 공유되어 중복 계산을 방지합니다. 서브에이전트는 메인 세션의 요약 캐시를 직접 읽을 수 있습니다
- **캐시 키 매칭을 위한 도구 거부**: 서브에이전트의 도구 호출 패턴이 기존 캐시 키와 일치하면, 시스템은 중복 실행을 거부하고 캐시된 요약 결과를 직접 반환합니다

### 상태 관리

백그라운드 타이머가 요약 작업의 생명주기를 관리하여, 세션이 종료될 때 타이머와 캐시를 정리합니다.

---

## 2. MagicDocs

### 목적

특정 형식의 Markdown 문서를 자동으로 유지하고 업데이트하며, 서브에이전트를 통한 문서 콘텐츠의 자동 생성 및 갱신을 지원합니다.

### 핵심 메커니즘

- **파일 식별**: 파일 상단의 `# MAGIC DOC` 헤더를 통해 대상 문서를 식별
- **패턴 매칭**: 처리가 필요한 파일 경로나 콘텐츠 패턴을 매칭하기 위해 패턴 정규식 사용
- **서브에이전트 생성**: 포크된 서브에이전트를 통해 문서 생성/업데이트 작업 실행
- **커스텀 프롬프트**: `~/.claude/magic-docs/prompt.md`에서 커스텀 프롬프트 템플릿 로딩 지원
- **변수 치환**: 문서 템플릿은 변수 치환을 지원하여 런타임 정보를 생성된 문서에 주입

### 핵심 함수

```typescript
// 파일이 MagicDoc인지 확인
isMagicDoc(content: string): boolean
// MagicDoc 업데이트 트리거
updateMagicDoc(filePath: string, context: Context): Promise<void>
```

### 오류 처리

서브에이전트 실행 실패는 메인 세션에 영향을 미치지 않습니다; 오류는 기록되지만 전파되지 않습니다.

---

## 3. PromptSuggestion

### 목적

투기적 실행(speculative execution)을 통해 사용자의 다음 입력을 예측하고 미리 응답을 준비하여 지각된 지연을 줄입니다.

### 게이팅 조건

- **최소 대화 턴**: 예측 시작 전 최소 **2번의 어시스턴트 턴**이 필요합니다 (`MIN 2 assistant turns`)
- **최대 미캐시 토큰**: 부모 컨텍스트의 미캐시 토큰 수는 `MAX_PARENT_UNCACHED_TOKENS = 10000`을 초과하지 않아야 합니다

### 거부 필터

예측 결과는 다음 필터를 통해 필터링됩니다; 필터링된 예측은 사용되지 않습니다:

| 필터 카테고리 | 설명 |
|-----------------|-------------|
| **done** | 예측된 콘텐츠가 대화가 끝났음을 내포함 (예: "감사합니다", "알겠습니다") |
| **meta-text** | 예측된 콘텐츠가 대화 자체에 대한 메타 텍스트 |
| **evaluative** | 예측된 콘텐츠가 평가적 (예: "잘 했습니다", "그것은 잘못되었습니다") |
| **Claude-voice** | 예측된 콘텐츠가 사용자의 목소리 대신 Claude의 목소리를 사용함 |

### 투기 샌드박스(Sandbox)

투기적 실행은 제한된 **투기 샌드박스(Sandbox)** 내에서 실행됩니다:

- **Copy-on-Write**: 쓰기 작업이 실제 상태에 영향을 미치지 않음
- **최대 턴**: 투기적 실행은 최대 **20턴** 실행
- **읽기 전용 Bash**: Bash 도구는 샌드박스(Sandbox) 내에서 읽기 전용 모드로 실행되며, 사이드 이펙트가 있는 명령을 실행하지 않음

#### 설계 근거

PromptSuggestion의 핵심 개념은 "투기적 실행"입니다 — 사용자가 결과를 보기 전에 다음 작업을 예측하고 미리 실행하여 상호작용을 더 유동적으로 만드는 것입니다. 하지만 투기는 본질적으로 도박입니다: 예측이 맞을 수도 있고 틀릴 수도 있으므로 메인 루프에서 할 수 없고(지연을 증가시킬 것임) 백그라운드에 배치하고 Copy-on-Write 샌드박스(Sandbox)에 격리해야 합니다. 소스 파일 `speculation.ts`는 투기 상태를 임시 디렉터리(`~/.claude/tmp/speculation/<pid>/<id>`)에 저장하여 메인 세션과 완전히 격리합니다. 사용자의 실제 입력이 도착하면, 시스템은 예측과 비교합니다 — 일치하면 결과를 직접 재사용(대기 시간 절약); 일치하지 않으면 조용히 폐기합니다. `checkReadOnlyConstraints`의 도입은 Bash 도구가 투기 중에 실제 사이드 이펙트를 생성하지 않도록 보장합니다.

### 상태 관리

현재 활성 투기 실행 상태를 유지합니다. 사용자의 실제 입력이 도착하면: 예측과 일치하면 결과를 재사용하고, 그렇지 않으면 투기 상태를 폐기합니다.

---

## 4. SessionMemory

### 목적

현재 세션에서 핵심 정보를 추출하고 Markdown 형식 세션 노트로 저장합니다.

### 핵심 메커니즘

- 대화 콘텐츠에서 구조화된 Markdown 세션 노트를 추출
- 출력 파일: `.session-memory.md`

### 임계값

| 파라미터 | 설명 |
|-----------|-------------|
| `minimumMessageTokensToInit` | 첫 번째 추출을 트리거하는 데 필요한 최소 메시지 토큰 수 |
| `minimumTokensBetweenUpdate` | 두 업데이트 사이에 필요한 최소 증분 토큰 수 |

### 핵심 함수

```typescript
// 세션 메모리 추출 초기화
initSessionMemory(messages: Message[]): Promise<void>
// 세션 메모리 업데이트
updateSessionMemory(messages: Message[]): Promise<void>
```

### 오류 처리

추출 실패는 메인 세션 흐름에 영향을 미치지 않고 조용히 저하됩니다.

---

## 5. autoDream

### 목적

주기적으로 단편화된 메모리 조각을 구조화된 장기 메모리로 통합하는 백그라운드 메모리 통합 서비스입니다.

### 게이트 순서

autoDream 실행은 다음 게이트 검사를 순서대로 통과해야 합니다:

1. **시간 게이트**: 마지막 통합 이후 충분한 시간이 경과했는지
2. **세션 스캔 게이트**: 현재 세션에 통합이 필요한 충분한 새 메모리 조각이 있는지
3. **잠금 게이트**: 한 번에 하나의 autoDream 인스턴스만 실행되도록 PID 기반 분산 잠금

### PID 잠금 메커니즘

- 동시 실행을 방지하기 위해 **PID 기반 잠금** 사용
- 잠금 오래됨 타임아웃: **60분** — 잠금을 보유한 프로세스가 60분 내에 잠금을 해제하지 않으면 오래된 잠금으로 간주되어 새 프로세스가 인수할 수 있습니다

### 4단계 통합 프롬프트

통합 프로세스는 **4단계 통합 프롬프트**를 사용하여 순서대로 실행합니다:

1. 기존 메모리 구조 검토
2. 새로 추가된 메모리 조각 식별
3. 병합 및 중복 제거
4. 통합된 메모리 문서 생성

#### 설계 근거

autoDream은 "세션 간" 작업입니다 — 사용자가 비활성 상태일 때 유휴 API 할당량을 사용하여 가치 있는 작업(메모리 정리, 병합 및 중복 제거)을 수행하며 활성 세션을 방해하지 않습니다. 소스 코드 주석은 게이트 순서를 명확하게 설명합니다: *"Gate order (cheapest first): 1. Time 2. Sessions 3. Lock"* (`autoDream.ts`) — 저렴한 시간 검사 먼저, 그 다음 세션 수 스캔, 그 다음에야 PID 잠금 획득. 이 계층화된 게이팅은 불필요한 리소스 소비를 방지합니다. PID 잠금의 60분 타임아웃은 충돌한 프로세스가 잠금을 영구적으로 보유하는 것을 방지합니다. `SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000` 스캔 스로틀은 시간 게이트는 통과하지만 세션 게이트는 통과하지 못할 때 유효하지 않은 반복 스캔을 추가로 방지합니다.

### 상태 관리

잠금 상태와 마지막 통합 타임스탬프는 파일 시스템을 통해 영속화됩니다.

---

## 6. extractMemories

### 목적

백그라운드에서 대화에서 메모리 조각을 추출하고 비동기적으로 메모리 저장소에 씁니다.

### 핵심 메커니즘

- **백그라운드 추출**: 메인 세션에 영향을 미치지 않고 비동기적으로 실행
- **병합 메커니즘**: 처리를 기다리는 메모리 조각을 임시로 저장하는 보류 스태시를 사용하고, 쓰기 전에 배치 병합
- **스로틀 제어**: `tengu_bramble_lintel` 동적 설정(Config)을 통해 추출 빈도 제어
- **상호 배제 쓰기**: 메인 에이전트의 메모리 쓰기 작업과 상호 배제되어 동시 쓰기로 인한 데이터 충돌 방지

### 4가지 메모리 분류 체계

| 타입 | 설명 |
|------|-------------|
| 타입 1 | 사용자 선호도 및 습관 |
| 타입 2 | 프로젝트 컨텍스트 및 기술 스택 |
| 타입 3 | 워크플로우 및 프로세스 |
| 타입 4 | 사실 및 지식 |

### 핵심 함수

```typescript
// 메모리 추출 트리거
extractMemories(messages: Message[], context: Context): Promise<void>
// 보류 스태시 플러시
flushPendingMemories(): Promise<void>
```

### 오류 처리

추출 실패 시 보류 중인 조각은 스태시에 보관되어 다음 시도 시 재시도됩니다.

---

## 7. LSP(Language Server Protocol)

### 목적

코드 인텔리전스 기능(자동 완성, 진단, 정의로 이동 등)을 지원하기 위해 Language Server Protocol 통합을 제공합니다.

### LSPClient

Claude Code와 LSP(Language Server Protocol) 서버 간의 메시지 직렬화/역직렬화 및 요청-응답 매칭을 담당하는 JSON-RPC 프로토콜 캡슐화 레이어입니다.

### LSPServerManager

파일 확장자를 기반으로 요청을 해당 LSP(Language Server Protocol) 서버 인스턴스로 라우팅하는 다중 인스턴스 라우팅 관리자입니다:

```typescript
// 주어진 확장자에 해당하는 LSP 서버 가져오기
getServerForExtension(ext: string): LSPServer
```

### 핵심 파라미터

| 파라미터 | 값 | 설명 |
|-----------|-------|-------------|
| 초기화 타임아웃 | **30초** | LSP(Language Server Protocol) 서버 시작을 위한 최대 대기 시간 |

### 파일 추적

`openedFiles` Map을 통해 열린 파일을 추적합니다:

```typescript
// 열린 파일에 대한 추적 테이블
openedFiles: Map<string, TextDocument>
```

파일이 열리거나 수정되거나 닫힐 때, 해당 LSP(Language Server Protocol) 서버에 동기적으로 알립니다.

### 오류 처리

LSP(Language Server Protocol) 서버가 시작 시 타임아웃되거나 충돌하면, 관련 기능은 기본 편집 기능에 영향을 미치지 않고 조용히 저하됩니다.

---

## 8. OAuth

### 목적

OAuth 인증 흐름을 처리하여 사용자가 Anthropic 계정을 통해 Claude Code에 로그인할 수 있도록 합니다.

### 핵심 메커니즘

- **PKCE 흐름**: 보안 강화를 위해 PKCE(Proof Key for Code Exchange) 사용
- **이중 인증 경로**:
  - **수동 경로**: 사용자가 인증 URL을 브라우저에 수동으로 복사하고 인증 코드를 다시 붙여넣음
  - **자동 경로**: 자동으로 브라우저를 열고 로컬 콜백 서버를 통해 인증 코드를 수신

### AuthCodeListener

OAuth 콜백을 수신하는 로컬 localhost HTTP 서버:

```typescript
// 로컬 콜백 서버 시작
startAuthCodeListener(port: number): Promise<AuthCode>
```

### 프로필 가져오기

성공적인 인증 후 사용자의 프로필 정보를 가져옵니다:

- **subscription**: 구독 상태 및 플랜
- **rateLimitTier**: 속도 제한 티어

### 오류 처리

- 콜백 서버 포트가 점유된 경우 다른 포트를 자동으로 시도
- 인증 타임아웃 후 임시 서버 리소스 정리
- 토큰 갱신 실패 시 재로그인 흐름으로 폴백

---

## 9. 플러그인(Plugins)

### 목적

플러그인(Plugin) 범위, 탐색, 설정(Config)을 관리합니다.

### 범위 관리

플러그인(Plugin)은 다음 범위 계층에 따라 관리됩니다:

| 범위 | 설명 |
|-------|-------------|
| `user` | 사용자 수준 플러그인(Plugin), 전역적으로 효과 |
| `project` | 프로젝트 수준 플러그인(Plugin), 특정 프로젝트에서만 효과 |
| `local` | 로컬 개발 플러그인(Plugin) |
| `managed` | 조직 관리 플러그인(Plugin) |

### 핵심 함수

```typescript
// 설정(Config)에서 플러그인(Plugin) 찾기
findPluginInSettings(pluginId: string, settings: Settings): Plugin | undefined
```

### V2 데이터 폴백

V2 형식 플러그인(Plugin) 데이터를 지원하며, 새 형식을 사용할 수 없는 경우 자동으로 V2 데이터 형식으로 폴백하여 하위 호환성을 보장합니다.

### 상태 관리

플러그인(Plugin) 상태는 사용자 및 프로젝트 수준 설정(Config)에 저장되며, 핫 리로드(hot reload)를 지원합니다.

---

## 10. policyLimits

### 목적

조직 수준 정책 제한을 적용하여 사용자가 수행할 수 있는 작업 범위를 제어합니다.

### 핵심 메커니즘

- **조직 수준 제한**: 조직 관리 엔드포인트에서 정책 설정(Config)을 검색
- **ETag HTTP 캐싱**: HTTP ETag 메커니즘을 사용하여 정책 데이터를 캐시하여 불필요한 네트워크 요청 감소
- **폴링 주기**: 백그라운드에서 **1시간**마다 정책 업데이트를 폴링

### 실패 정책

**fail-open** 정책을 사용합니다: 정책 서비스에 접근할 수 없는 경우 기본적으로 작업이 허용되어, 정책 서비스 실패로 인해 사용자의 작업이 차단되지 않도록 보장합니다.

**예외**: `ESSENTIAL_TRAFFIC_DENY_ON_MISS` 모드(**HIPAA** 준수 시나리오용) — 정책에 접근할 수 없는 경우 작업이 **거부**되어, 엄격한 준수 요구사항이 있는 환경에서 비인가 작업이 허용되지 않도록 보장합니다.

### 핵심 함수

```typescript
// 작업이 정책에 의해 허용되는지 확인
checkPolicyLimit(action: string, context: PolicyContext): PolicyResult
// 정책 캐시 갱신
refreshPolicyLimits(): Promise<void>
```

### 오류 처리

- 네트워크 오류: fail-open (HIPAA 모드 제외)
- 파싱 오류: 마지막으로 유효한 정책 캐시 사용
- ETag 일치: 304 응답이 반환되면 로컬 캐시를 직접 사용

---

## 11. remoteManagedSettings

### 목적

조직이 원격으로 전달하는 설정(Config)을 관리하여, 조직 정책이 클라이언트 측에서 적용되도록 합니다.

### 핵심 메커니즘

- **조직 수준 설정(Config)**: 원격 서비스에서 조직 관리자가 설정(Config)한 설정을 가져옴
- **보안 검사**: `checkManagedSettingsSecurity()` 함수는 위험한 설정(Config) 변경을 감지합니다. 잠재적으로 위험한 변경이 감지되면(보안 기능 비활성화, 중요 경로 수정 등), 사용자에게 확인 프롬프트(위험한 변경 프롬프트)를 표시합니다
- **백그라운드 폴링**: 백그라운드에서 **1시간**마다 설정(Config) 업데이트를 폴링

### 핵심 함수

```typescript
// 원격 설정(Config) 변경의 보안 확인
checkManagedSettingsSecurity(
  oldSettings: ManagedSettings,
  newSettings: ManagedSettings
): SecurityCheckResult

// 현재 원격 관리 설정(Config) 가져오기
getManagedSettings(): Promise<ManagedSettings>
```

### 오류 처리

원격 설정(Config) 검색이 실패하면, 마지막으로 유효한 로컬 캐시 설정(Config)을 사용하고 백그라운드에서 재시도를 계속합니다.

---

## 12. settingsSync

### 목적

여러 기기에서 사용자 설정(Config)과 메모리 데이터를 양방향으로 동기화합니다.

### 핵심 메커니즘

- **양방향 Push/Pull**: 업로드(push)와 다운로드(pull) 방향 모두에서 동기화 지원
- **동기화 범위**: `SYNC_KEYS`는 동기화할 데이터 카테고리를 정의합니다:
  - **Settings**: 사용자 설정(Config)
  - **Memory**: 메모리 데이터
  - **Project-keyed data**: 프로젝트로 키화된 데이터, 프로젝트 식별자 키로 **git 해시** 사용

### 제한

| 파라미터 | 값 | 설명 |
|-----------|-------|-------------|
| 최대 업로드 크기 | **500KB** | 동기화 업로드당 최대 데이터 양 |

### 증분 업로드

**증분 업로드**를 지원하여 마지막 동기화 이후 변경된 데이터만 업로드하여 네트워크 전송을 줄입니다.

### 핵심 함수

```typescript
// 로컬 설정(Config)을 원격으로 푸시
pushSettings(keys: SyncKey[]): Promise<void>
// 원격에서 설정(Config) 풀
pullSettings(keys: SyncKey[]): Promise<void>
// 양방향 동기화 수행
syncSettings(): Promise<SyncResult>
```

### 오류 처리

- 충돌 해결: 원격 데이터가 우선 (pull 승리)
- 크기 제한 초과 시 업로드를 분할하거나 잘라냄
- 네트워크 실패 시 변경사항이 스테이징되어 다음 동기화를 기다림

---

## 13. teamMemorySync / secretScanner

### 목적

공유 메모리에서 민감한 정보가 유출되지 않도록 내장 시크릿 스캐너가 있는 팀 메모리 동기화 서비스입니다.

### 시크릿 스캐너

#### 규칙 집합

공개 [gitleaks 설정(Config)](https://github.com/gitleaks/gitleaks/blob/master/config/gitleaks.toml)(MIT 라이선스)에서 가져온 다음 시크릿 유형을 포함하는 내장 **30개의 gitleaks 규칙**:

| 카테고리 | 포함된 시크릿 유형 |
|----------|-----------------------|
| **AWS** | Access Key ID, Secret Access Key, Session Token 등 |
| **GCP** | Service Account Key, API Key 등 |
| **Azure** | Storage Account Key, Client Secret 등 |
| **GitHub** | Personal Access Token, OAuth Token, App Private Key 등 |
| **Slack** | Bot Token, Webhook URL, App Token 등 |
| **Stripe** | Secret Key, Publishable Key, Webhook Secret 등 |
| **Private Keys** | RSA, DSA, EC, PGP 개인 키 |

#### 스캔 함수

```typescript
// 시크릿에 대한 텍스트 스캔, 일치하는 규칙 ID 반환 (실제 시크릿 값이 아님)
scanForSecrets(text: string): SecretScanResult[]

interface SecretScanResult {
  ruleId: string;    // 일치하는 규칙 ID
  // 참고: 실제 시크릿 값을 포함하지 않아 2차 노출 방지
}
```

**보안 설계**: `scanForSecrets()`는 실제 시크릿 값이 아닌 일치하는 **규칙 ID**를 반환하여, 스캔 결과에서 시크릿이 2차 노출되는 것을 방지합니다.

#### 수정 함수

```typescript
// 텍스트에서 발견된 시크릿 수정
redactSecrets(text: string): string
```

감지된 시크릿을 플레이스홀더 값으로 대체합니다.

### teamMemSecretGuard

```typescript
// 시크릿을 포함한 콘텐츠가 동기화된 메모리에 쓰이는 것을 방지
teamMemSecretGuard(content: string): GuardResult
```

쓰기 가드 역할을 하여, 메모리 콘텐츠가 팀 동기화 저장소에 쓰이기 전에 시크릿 스캔을 수행합니다. 시크릿이 감지되면 **쓰기가 차단**되고 관련 규칙 ID 정보가 반환됩니다.

#### 설계 근거

팀 메모리는 팀원들 간에 동기화되므로, 한 사람의 시크릿이 전체 팀에 유출되지 않도록 방지하는 것이 필수적입니다. 30개의 gitleaks 규칙은 AWS, GCP, Azure, GitHub, Slack, Stripe를 포함한 주류 클라우드 서비스의 주요 형식을 포함합니다 (소스 코드 `secretScanner.ts` 주석: "Rule IDs and regexes sourced directly from the public gitleaks config"). 자체 제작 솔루션 대신 gitleaks 규칙 집합을 선택한 것은 오픈소스 커뮤니티에 의해 대규모로 검증되어 폭넓은 커버리지와 낮은 오탐률을 가지기 때문입니다. 보안 설계는 fail-closed 정책을 사용합니다 — 스캔이 실패하면 허용하는 대신 쓰기가 차단됩니다; 기능이 희생되더라도 시크릿이 유출되지 않습니다. `scanForSecrets()`는 로그나 텔레메트리(Telemetry)에서 시크릿이 2차 노출되는 것을 방지하기 위해 실제 시크릿 값 대신 규칙 ID를 반환합니다.

### 오류 처리

- 스캔 실패 시 기본적으로 쓰기가 차단됨 (fail-closed), 보안 보장
- 규칙 매칭은 결정론적 알고리즘 사용으로 확률론적 오탐 없음
- 모든 차단 이벤트는 텔레메트리(Telemetry) 시스템에 기록됨

---

## 엔지니어링 실천 가이드

### 새 서비스 추가

**단계 체크리스트:**

1. **서비스 모듈 생성**: `src/services/` 하위에 새 디렉터리와 진입점 파일 생성
2. **생명주기 정의**: 시작(init/start)과 종료(shutdown/cleanup) 메서드 구현
3. **서비스 등록**: `services/` 진입점 또는 `setup.ts`에 초기화 호출 등록
4. **시작 타이밍 결정**:
   - 애플리케이션 시작 시 (예: policyLimits) → `setup.ts`에서 직접 호출
   - 요청 시 시작 (예: LSP(Language Server Protocol)) → 첫 사용 시 지연 초기화
   - 유휴 시작 (예: autoDream) → 게이팅 조건에 의해 트리거됨
5. **오류 처리**: 백그라운드 서비스 실패는 메인 기능에 영향을 미쳐서는 안 됩니다 (조용한 저하 원칙)

**핵심 설계 제약**:
- 각 서비스는 독립적인 생명주기와 리소스 요구사항을 가집니다
- 하나의 서비스 충돌이나 재시작은 다른 서비스에 영향을 미쳐서는 안 됩니다
- 서비스는 헤드리스/bare 모드에서 다르게 동작할 수 있습니다 (예: `initSessionMemory()`는 bare 모드에서 실행되지 않음)

### 서비스 간 통신

**원칙: 서비스는 이벤트 버스나 공유 상태를 통해 통신합니다 — 서로 직접 호출하지 마십시오.**

- **이벤트 버스**: `logEvent`를 통해 분석 이벤트를 전송하고, 다른 서비스가 수신할 수 있음
- **공유 상태**: 전역 설정(Config)이나 앱 상태를 통해 데이터 공유
- **훅(Hook) 시스템**: `postSamplingHook` 및 `handleStopHooks`와 같은 훅(hook) 포인트를 통해 여러 서비스가 동일한 타이밍에 실행 가능
- **순환 의존성 방지**: 소스 코드의 분석 모듈이 의도적으로 "의존성 없음"으로 설계된 것은 바로 여러 서비스가 의존할 때 사이클 형성을 방지하기 위한 것입니다

### 서비스 시작 디버깅

**문제 해결 단계:**

1. **초기화 순서 확인**: 일부 서비스는 의존성이 있습니다 (예: API 클라이언트 전에 OAuth)
2. **게이팅 조건 확인**:
   - `feature('TEAMMEM')` — 팀 메모리 동기화
   - `feature('EXTRACT_MEMORIES')` — 메모리 추출
   - `isBareMode()` — bare 모드에서 LSP(Language Server Protocol) 같은 서비스 건너뜀
3. **PID 잠금 확인**: autoDream은 PID 잠금(60분 타임아웃)을 사용; 충돌 후 정리가 필요할 수 있음
4. **백그라운드 타이머 확인**: AgentSummary는 30초마다 실행; policyLimits/remoteManagedSettings는 1시간마다 폴링
5. **Promise.allSettled 확인**: LSP(Language Server Protocol) 관리자의 종료는 `Promise.allSettled`를 사용하므로 단일 서비스 종료 실패가 다른 서비스에 영향을 미치지 않음

**서비스별 핵심 임계값/타임아웃:**

| 서비스 | 핵심 임계값 |
|---------|---------------|
| AgentSummary | 30초 주기 |
| LSP(Language Server Protocol) | 30초 초기화 타임아웃 |
| autoDream | PID 잠금 60분 타임아웃, `SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000` |
| policyLimits | 1시간 폴링, fail-open (HIPAA 제외) |
| remoteManagedSettings | 1시간 폴링 |
| settingsSync | 최대 업로드 500KB |

### 일반적인 함정

| 함정 | 세부 정보 | 해결 방법 |
|---------|---------|---------|
| 헤드리스 모드에서 서비스 동작 차이 | SessionMemory 및 MagicDocs와 같이 UI에 의존하는 서비스는 bare/헤드리스 모드에서 시작되지 않을 수 있음 | `isBareMode()` 및 관련 게이팅 조건 확인 |
| 서비스 API 호출이 비용에 반영됨 | AgentSummary, extractMemories, autoDream은 모두 API 호출을 수행 | 백그라운드 서비스 토큰 소비 및 비용이 비용 추적기에 반영됨 |
| policyLimits fail-open 예외 | HIPAA 준수 시나리오는 `ESSENTIAL_TRAFFIC_DENY_ON_MISS` 모드 사용 | 이 모드에서는 정책에 접근할 수 없을 때 허용 대신 거부 |
| secretScanner fail-closed | 시크릿 스캐너 스캔 실패가 팀 메모리 쓰기를 차단 | 기능이 희생되더라도 시크릿이 유출되지 않음 |
| settingsSync 충돌 해결 | pull 승리 (원격 데이터가 우선) | 로컬 수정이 원격 데이터에 의해 덮어쓰일 수 있음 |


---

[← 피드백 & 설문](../19-反馈与调查/feedback-system-ko.md) | [인덱스](../README_KO.md) | [플러그인(Plugin) 시스템 →](../21-插件系统/plugin-system-ko.md)
