# OAuth 및 인증 시스템

Claude Code의 인증 시스템은 완전한 OAuth 2.0 PKCE 흐름, 안전한 토큰 저장소, 다중 소스 인증 해석 체인을 구현합니다.

---

## OAuth 서비스 (src/services/oauth/)

### 설계 철학

#### 왜 단순 OAuth 대신 PKCE인가?

Claude Code는 데스크톱 CLI 애플리케이션이므로 OAuth 용어로 "공개 클라이언트"입니다 — client_secret을 안전하게 저장할 수 없습니다(로컬 바이너리에 내장된 시크릿은 역공학을 통해 추출 가능). PKCE(Proof Key for Code Exchange)는 이 시나리오를 위해 특별히 설계되었습니다: client_secret을 일회용 `code_verifier`/`code_challenge` 쌍으로 대체하여, 인증 코드가 가로채이더라도 토큰으로 교환할 수 없습니다. `OAuthService` 생성자에서 `this.codeVerifier = crypto.generateCodeVerifier()`가 각 흐름에 대해 고유한 verifier를 생성하고, `client.ts`의 `code_challenge_method=S256`은 SHA-256 해싱을 사용하여 verifier가 평문으로 전송되지 않도록 보장합니다.

### OAuthService 클래스

완전한 인증 흐름을 관리하는 핵심 OAuth 서비스 클래스입니다.

#### 주요 메서드

**startOAuthFlow()**
- 완전한 OAuth 인증 흐름을 시작합니다
- 흐름: localhost 리스너 → PKCE + state → URL 빌드 → 코드 대기 → 교환 → 프로필 가져오기
- 자동으로 브라우저를 인증 페이지로 엽니다
- 콜백을 수신하기 위해 로컬 HTTP 서버를 시작합니다

**handleManualAuthCodeInput()**
- 자동 흐름이 실패할 때(예: 브라우저가 열리지 않는 경우) 인증 코드의 수동 붙여넣기를 지원합니다
- 사용자가 제공한 인증 코드를 파싱하고 토큰 교환 흐름을 계속합니다

**cleanup()**
- OAuth 흐름에서 임시 리소스를 정리합니다
- localhost HTTP 리스너를 닫습니다
- 임시 상태를 지웁니다

### OAuth 흐름 — 상세 단계

1. PKCE code_verifier 및 code_challenge 생성
2. CSRF를 방지하기 위한 랜덤 state 파라미터 생성
3. 콜백을 수신하기 위한 로컬 HTTP 서버 시작
4. 인증 URL 빌드 및 브라우저 열기
5. 사용자 인증 후 콜백 대기 (인증 코드 포함)
6. code + code_verifier를 사용하여 토큰 교환
7. 사용자 프로필 정보 가져오기

---

## 토큰 교환 (client.ts)

### buildAuthUrl

```typescript
buildAuthUrl(codeChallenge: string, state: string) → string
```

OAuth 인증 URL을 빌드합니다, 다음을 포함:
- client_id
- redirect_uri (localhost)
- response_type=code
- code_challenge + code_challenge_method=S256
- state
- scope

### exchangeCodeForTokens

```typescript
exchangeCodeForTokens() → { accessToken, refreshToken, expiresIn, scope }
```

인증 코드를 액세스 토큰 및 리프레시 토큰으로 교환합니다:
- 토큰 엔드포인트에 POST 요청을 전송합니다
- PKCE 검증을 위해 code_verifier를 포함합니다
- 일시적인 네트워크 오류를 처리하기 위한 내장 재시도 메커니즘을 가집니다

### refreshOAuthToken

```typescript
refreshOAuthToken(refreshToken: string, scopes?: string[]) → { accessToken, refreshToken, expiresIn }
```

리프레시 토큰을 사용하여 액세스 토큰을 갱신합니다:
- 권한을 좁히기 위한 범위 지정을 지원합니다
- 새 토큰 쌍(액세스 + 리프레시)을 반환합니다

#### 왜 자동 토큰 갱신인가?

토큰 만료로 인해 사용자의 워크플로우가 중단되어서는 안 됩니다 — 자동 갱신이 최선의 UX입니다. `isOAuthTokenExpired()`는 5분 버퍼를 내장하여(실제 만료 시간 전에 `true`를 반환), 갱신을 완료할 충분한 시간을 보장합니다. `jwtUtils.ts`의 `createTokenRefreshScheduler`는 심지어 사전 갱신을 구현합니다 — 요청이 실패하기를 기다리는 대신 토큰 만료에 기반하여 미리 갱신을 예약합니다. 갱신이 실패할 때만 시스템이 재로그인 흐름으로 폴백하여 사용자 작업에 대한 방해를 최소화합니다.

### fetchProfileInfo

```typescript
fetchProfileInfo(accessToken: string) → {
  subscription: string,
  rateLimitTier: string,
  displayName: string,
  billingType: string,
}
```

사용자 프로필 정보를 가져옵니다:
- subscription: 구독 유형 (무료, 프로, 맥스 등)
- rateLimitTier: 속도 제한 티어
- displayName: 사용자 표시 이름
- billingType: 청구 유형

### isOAuthTokenExpired

```typescript
isOAuthTokenExpired() → boolean
```

현재 OAuth 토큰이 만료되었는지 확인합니다:
- 5분 버퍼를 내장하여 실제 만료 전에 true를 반환합니다
- 토큰 갱신을 완료할 충분한 시간을 보장합니다

### populateOAuthAccountInfoIfNeeded

```typescript
populateOAuthAccountInfoIfNeeded() → void
```

OAuth 계정 정보를 지연 가져옵니다:
- 필요한 경우에만 요청을 수행합니다(예: 계정 정보에 처음 접근할 때)
- 가져온 후 결과를 캐시하여 반복 요청을 방지합니다

---

## 인증 코드 리스너 (auth-code-listener.ts)

### AuthCodeListener 클래스

OAuth 콜백을 수신하기 위한 로컬 HTTP 서버를 시작합니다.

#### start

```typescript
start(port?: number) → Promise<number>
```

- localhost HTTP 서버를 시작합니다
- 포트가 지정되지 않으면 OS가 할당하는 랜덤 포트(포트 0)를 사용합니다
- 실제로 리스닝 중인 포트를 반환합니다

#### waitForAuthorization

```typescript
waitForAuthorization(state: string, onReady: (url: string) => void) → Promise<string>
```

- OAuth 프로바이더의 콜백 요청을 기다립니다
- `state` 파라미터는 콜백의 합법성을 검증하는 데 사용됩니다
- `onReady` 콜백은 서버가 준비되면 리다이렉트 URL을 전달하며 실행됩니다

#### 보안 검증 및 오류 처리

- **State 불일치**: 콜백의 state가 예상 값과 일치하지 않음 → 거부, CSRF 공격 방지
- **코드 없음**: 인증 코드가 콜백에 없음 → HTTP 400 반환
- **성공 리다이렉트**: 성공적인 인증 후 성공 페이지로 리다이렉트
- **오류 리다이렉트**: 실패한 인증 후 오류 정보를 표시하는 오류 페이지로 리다이렉트

---

## 안전한 저장소 (src/utils/secureStorage/)

### 설계 철학

#### 왜 키체인 통합인가?

토큰을 평문으로 저장하는 것은 보안 안티패턴입니다 — 사용자 파일을 읽을 수 있는 모든 프로세스가 자격 증명을 도용할 수 있습니다. OS 키체인(macOS Keychain / Windows Credential Manager)은 OS 수준의 암호화 저장소를 제공하며, 자격 증명은 사용자의 로그인 비밀번호로 보호됩니다. macOS 구현(`macOsKeychainStorage.ts`)은 `security` CLI 도구를 통해 키체인과 상호작용하여, 자격 증명을 stdin을 통해 전달하는 것을 선호합니다(커맨드라인 인수보다 더 안전하고 프로세스 목록에 노출되지 않음), 자격 증명이 4032바이트(`SECURITY_STDIN_LINE_LIMIT`)를 초과할 때만 argv로 폴백합니다. 자격 증명 값은 커맨드라인에서 프로세스 모니터링 도구가 평문을 캡처하지 못하도록 16진수 인코딩으로 저장됩니다. 5분 `KEYCHAIN_CACHE_TTL_MS`는 시스템 키체인에 대한 빈번한 접근을 줄이고, stale-while-error 전략은 토큰 갱신이 실패해도 캐시된 오래된 값으로 시스템이 계속 작동할 수 있게 합니다 — 이 모든 것은 보안과 사용성 간의 균형을 맞추기 위해 설계되었습니다.

### 플랫폼 추상화

실행 플랫폼에 따라 다른 저장소 백엔드를 선택합니다:

- **macOS**: 키체인 저장소 + 평문 폴백
- **Windows/Linux**: 평문 저장소

### plainTextStorage

```
저장 위치: ~/.claude/.credentials.json
파일 권한: 0600 (소유자 읽기/쓰기 전용)
```

- 파일 시스템에 JSON 형식으로 자격 증명을 저장합니다
- 보안은 엄격한 파일 권한으로 보호됩니다
- 모든 플랫폼의 폴백 저장소 역할을 합니다

### macOsKeychainStorage

macOS 시스템 키체인을 안전한 저장소로 사용합니다:

```
명령: security find-generic-password / security add-generic-password
```

#### KEYCHAIN_CACHE_TTL_MS

```typescript
const KEYCHAIN_CACHE_TTL_MS = 5 * 60 * 1000  // 5분
```

키체인 읽기 결과에 대한 인메모리 캐시 TTL로, 시스템 키체인에 대한 빈번한 접근을 줄입니다.

#### Stale-while-error 전략

토큰 갱신이 실패할 때, 캐시에 오래된 값이 있으면(TTL을 지났더라도) 즉시 실패하는 대신 그 오래된 값을 사용하여 계속 작업합니다. 이것은 HTTP stale-while-revalidate 패턴과 유사합니다.

#### 16진수 인코딩

```
자격 증명 값은 키체인에 저장되기 전에 16진수로 인코딩됩니다
```

자격 증명은 16진수 인코딩으로 저장되어 프로세스 모니터링 도구가 커맨드라인 인수에서 평문 자격 증명을 캡처하지 못하도록 합니다.

#### stdin 제한 및 폴백

```
stdin 전송 제한: 4032바이트
제한 초과 시: argv로 폴백
```

- `security` 명령에 자격 증명을 stdin을 통해 전달하는 것을 선호합니다(더 안전하고 프로세스 인수에 노출되지 않음)
- 자격 증명이 4032바이트를 초과하면 커맨드라인 인수를 통해 전달하는 것으로 폴백합니다

#### isMacOsKeychainLocked

```typescript
isMacOsKeychainLocked() → boolean
```

macOS 키체인이 잠긴 상태인지 확인합니다:
- 결과가 캐시되어 빈번한 감지를 방지합니다
- 키체인이 잠겨 있으면 자동으로 평문 저장소로 폴백합니다

---

## 인증 해석 체인

### 인증 소스 우선순위

인증 시스템은 다음 우선순위 순서로 각 소스를 시도하며, 처음으로 성공한 것을 사용합니다:

```
1. 3P 컨텍스트        → 서드파티 통합 컨텍스트에서 제공하는 자격 증명
2. bare 모드          → bare 모드의 특수 인증
3. 관리형 OAuth       → 관리 환경을 위한 OAuth 자격 증명
4. 명시적 토큰        → 명시적으로 설정된 토큰
5. OAuth              → 표준 OAuth 흐름을 통해 얻은 토큰
6. API 키 폴백        → API 키 폴백
```

### 인증 소스 세부 정보

**3P 컨텍스트 (서드파티 컨텍스트)**
- Claude Code가 서드파티 도구로 통합될 때, 자격 증명은 호스트 환경이 제공합니다
- 가장 높은 우선순위, 직접 사용

**Bare 모드**
- 슬림 모드의 인증으로, 표준 흐름을 우회합니다

**관리형 OAuth**
- 기업 또는 관리 환경을 위해 미리 설정된 OAuth 자격 증명

**명시적 토큰**
- `ANTHROPIC_AUTH_TOKEN` 환경 변수
- `apiKeyHelper` 외부 프로그램을 통해 얻은 토큰
- 파일 디스크립터를 통한 토큰 전달을 위한 `FILE_DESCRIPTOR`

**OAuth (claude.ai)**
- 표준 OAuth 2.0 PKCE 흐름
- claude.ai 인증을 통해 얻은 토큰
- 자동 갱신 지원

**API 키 폴백**
- `ANTHROPIC_API_KEY` 환경 변수
- 가장 낮은 우선순위 폴백
- OAuth 토큰 대신 API 키를 직접 사용

---

## 엔지니어링 실천 가이드

### 인증 실패 디버깅

**문제 해결 체크리스트:**

1. **키체인의 토큰 확인**:
   - macOS: `security find-generic-password -s claude-code`로 키체인 항목 확인
   - Windows/Linux: `~/.claude/.credentials.json` 확인 (권한 0600)
2. **토큰 수동 갱신**:
   - `isOAuthTokenExpired()`는 5분 버퍼를 내장합니다 (실제 만료 전에 true 반환)
   - `refreshOAuthToken()`은 리프레시 토큰을 사용하여 새 액세스 토큰을 얻습니다
   - 갱신 실패 시 재로그인 흐름으로 폴백합니다
3. **PKCE 파라미터 확인**:
   - `code_verifier`는 각 OAuth 흐름에 대해 고유하게 생성됩니다(`crypto.generateCodeVerifier()`)
   - `code_challenge_method=S256`은 SHA-256 해싱을 사용합니다
   - PKCE 파라미터 불일치는 토큰 교환 실패를 초래합니다
4. **리다이렉트 URI 확인**:
   - 로컬 HTTP 서버는 localhost 포트를 수신합니다
   - 포트가 점유된 경우 자동으로 다른 포트를 시도합니다(포트 0은 OS가 할당)
   - state 파라미터 불일치는 CSRF 보호를 트리거하고 콜백을 거부합니다

**인증 소스 우선순위 확인**:
```
1. 3P 컨텍스트       → 서드파티 통합 자격 증명 (가장 높은 우선순위)
2. bare 모드        → 슬림 모드 특수 인증
3. 관리형 OAuth    → 관리 환경 OAuth
4. 명시적 토큰      → ANTHROPIC_AUTH_TOKEN / apiKeyHelper / FILE_DESCRIPTOR
5. OAuth            → 표준 OAuth PKCE 흐름
6. API 키 폴백      → ANTHROPIC_API_KEY (가장 낮은 우선순위)
```

### 새 인증 방법 추가

**`getAnthropicClient()`에 인증 브랜치 추가:**

1. `src/services/api/client.ts`의 `getAnthropicClient()` 함수에 새 인증 브랜치를 추가합니다
2. 토큰 검색 로직(동기 또는 비동기) 구현
3. 우선순위 순서에 따라 인증 해석 체인에 삽입
4. 새 인증 방법이 자동 갱신도 지원하는지 확인(해당되는 경우)

### 토큰 생명주기 관리

**자동 갱신 메커니즘:**

- `isOAuthTokenExpired()`는 **5분 버퍼**를 내장하여 갱신을 완료할 충분한 시간을 보장합니다
- `jwtUtils.ts`의 `createTokenRefreshScheduler`는 사전 갱신을 구현합니다 — 토큰 만료에 기반하여 미리 예약됩니다
- 갱신 실패 시 전체 OAuth 로그인 흐름으로 폴백합니다

**키체인 캐시 (macOS):**

- `KEYCHAIN_CACHE_TTL_MS = 5 * 60 * 1000` (5분)은 시스템 키체인에 대한 빈번한 접근을 줄입니다
- **Stale-while-error 전략**: 토큰 갱신이 실패할 때, 캐시에 오래된 값이 있으면 그 값으로 계속 작업합니다
- 자격 증명은 **16진수 인코딩**으로 저장되어 프로세스 모니터링 도구가 평문을 캡처하지 못하도록 합니다
- stdin 전송 제한은 **4032바이트**(`SECURITY_STDIN_LINE_LIMIT`); 초과 시 argv로 폴백합니다

### 안전한 저장소 플랫폼 차이

| 플랫폼 | 저장소 백엔드 | 보안 수준 | 폴백 |
|----------|----------------|----------------|---------|
| macOS | 키체인(`security` CLI) | 높음 (OS 수준 암호화) | 평문 (`~/.claude/.credentials.json`) |
| Windows | 평문 | 중간 (파일 권한 0600) | 없음 |
| Linux | 평문 | 중간 (파일 권한 0600) | 없음 |

### 일반적인 함정

| 함정 | 세부 정보 | 해결 방법 |
|---------|---------|---------|
| 키체인의 OAuth 토큰 — 제거 시 삭제되지 않음 | macOS 키체인의 자격 증명은 Claude Code 제거 후에도 지속됩니다 | `security delete-generic-password`를 사용하여 수동으로 삭제 |
| 만료된 refresh_token은 재인증이 필요함 | 리프레시 토큰 자체도 유효 기간이 있으며 만료 후 자동 갱신이 불가능 | 시스템이 자동으로 전체 OAuth 로그인 흐름으로 폴백합니다 |
| 키체인이 잠겨 있을 때 폴백 | macOS 키체인이 잠겨 있는 동안 읽기/쓰기 불가 | `isMacOsKeychainLocked()`가 이를 감지하고 자동으로 평문으로 폴백 |
| 포트 충돌 | 로컬 OAuth 콜백 서버 포트가 점유됨 | 포트 0을 사용하여 OS가 사용 가능한 랜덤 포트를 할당하도록 합니다 |
| state 파라미터 CSRF 보호 | 콜백의 불일치 state는 거부됩니다 | 브라우저 콜백의 state 파라미터가 원래 요청에서 전송된 것과 일치하는지 확인 |
| `getAnthropicClient` 순환 참조 위험 | 이 함수는 여러 곳에서 호출됩니다 | 소스 주석: "Currently we create a new GoogleAuth instance for every getAnthropicClient() call" — 성능 영향에 주의 |


---

[← 플러그인(Plugin) 시스템](../21-插件系统/plugin-system-ko.md) | [인덱스](../README_KO.md) | [LSP(Language Server Protocol) 통합 →](../23-LSP集成/lsp-integration-ko.md)
