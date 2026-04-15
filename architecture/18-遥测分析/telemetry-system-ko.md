# 텔레메트리(Telemetry) & 분석 시스템

Claude Code의 텔레메트리(Telemetry) 및 분석 시스템은 여러 서브시스템을 포함합니다: 이벤트 분석(Analytics), OpenTelemetry 통합, 세션 추적(Session Tracing), 플러그인(Plugin) 텔레메트리(Telemetry), 비용 추적(Cost Tracking), 게이트웨이 감지(Gateway Detection).

---

## 분석 아키텍처

### 무의존성 설계 (events.ts)

분석 시스템은 **무의존성 설계**를 사용합니다: 이벤트는 `events.ts`에서 생성되어 큐에 저장되며, 싱크가 연결될 때까지(`attachAnalyticsSink()` 호출) 실제로 전송되지 않습니다. 이를 통해 초기화 중이거나 싱크가 아직 준비되지 않은 경우에도 이벤트가 손실되지 않습니다.

### AnalyticsSink 인터페이스

```typescript
interface AnalyticsSink {
  logEvent: (event: AnalyticsEvent) => void;        // 동기 이벤트 로깅
  logEventAsync: (event: AnalyticsEvent) => Promise<void>;  // 비동기 이벤트 로깅
}
```

### Proto 필드 제거

`stripProtoFields()` 함수는 이벤트 객체에서 `_PROTO_*` 접두사가 붙은 모든 키를 제거합니다. 이 필드들은 내부 전용이며 일반 접근 백엔드로 전송하기 전에 제거되어야 합니다.

### 설계 근거

#### 왜 Datadog + 1P 이중 라우팅인가?

`analytics/index.ts`의 소스 주석에는 다음과 같이 명시되어 있습니다: *"DESIGN: This module has NO dependencies to avoid import cycles. Events are queued until attachAnalyticsSink() is called."* `sinkKillswitch.ts`는 `SinkName = 'datadog' | 'firstParty'`를 두 개의 독립적인 채널로 정의합니다. Datadog는 실시간 모니터링과 알림을 제공하고(운영 관점 — P0 이벤트는 즉시 알림 트리거), 1P(퍼스트 파티 Anthropic 이벤트 로깅)는 장기 분석과 제품 의사결정 지원을 제공합니다(제품 관점 — 사용자 행동 트렌드, 기능 채택률). 두 채널은 데이터 요구사항, 접근 권한, 보존 정책이 완전히 다릅니다. `_PROTO_*` 필드는 1P에만 전송되며, `stripProtoFields()`를 통해 Datadog와 같은 일반 접근 백엔드에서는 제거됩니다 — 이것은 데이터 보안 레이어링입니다.

#### 왜 텔레메트리(Telemetry) 시스템이 "무의존성 큐" 설계인가?

소스 파일 `analytics/index.ts`는 상단에서 명시적으로 선언합니다: *"DESIGN: This module has NO dependencies to avoid import cycles. Events are queued until attachAnalyticsSink() is called during app initialization."* 텔레메트리(Telemetry)는 핵심 기능에 영향을 미쳐서는 안 됩니다 — 텔레메트리(Telemetry) 서비스가 중단되어도 사용자의 코드 편집은 영향받지 않아야 합니다. 큐 설계는 다음을 보장합니다: (1) 싱크가 아직 준비되지 않은 시작 중에도 이벤트가 손실되지 않음; (2) 전송 실패는 조용히 처리됨(fail-open 원칙: `sinkKillswitch`가 누락되거나 오류가 발생해도 싱크는 기본적으로 열려 있음); (3) 임포트 의존성 없음은 순환 참조를 방지합니다 — 텔레메트리(Telemetry) 모듈은 거의 모든 다른 모듈에서 사용되므로, 다른 모듈에 의존한다면 순환 임포트가 빠르게 형성될 것입니다.

#### 왜 커스텀 기능 플래그 시스템 대신 GrowthBook인가?

GrowthBook은 카나리아 릴리스와 제어된 실험을 지원하는 업계 표준 A/B 테스팅 플랫폼입니다. 소스 코드는 `getFeatureValue_CACHED_MAY_BE_STALE()`와 GrowthBook 게이트(예: `tengu_log_datadog_events` 및 `enhanced_telemetry_beta`)를 많이 사용합니다. 커스텀 기능 플래그 시스템을 구축하면 사용자 세분화, 점진적 롤아웃, 실험 분석, 비상 롤백을 직접 구현해야 합니다. GrowthBook은 세션 안정적인 기능 평가도 제공합니다(`promptCache1hAllowlist`에 대한 소스 주석: *"Cached prompt cache 1h TTL allowlist from GrowthBook (session-stable)"*), 세션 중간에 기능 플래그가 바뀌지 않도록 보장합니다.

### 이중 채널 라우팅

이벤트는 이중 채널 라우팅 메커니즘을 통해 두 백엔드로 동시에 전송됩니다:

1. **Datadog** — 서드파티 모니터링 및 분석 플랫폼
2. **1P 이벤트 로깅** — Anthropic 퍼스트 파티 이벤트 로깅 시스템

### Datadog 게이팅

`shouldTrackDatadog()` 함수는 이벤트를 Datadog으로 전송할지 여부를 제어합니다:

- **킬스위치(Killswitch)**: Datadog 보고를 긴급 비활성화할 수 있는 전역 스위치
- **GrowthBook 기능 게이트**: `tengu_log_datadog_events` 기능 게이트를 통해 제어

### 샘플링 로직

`shouldSampleEvent()` 함수는 이벤트 샘플링을 구현하여, 고빈도 이벤트를 다운샘플링하여 데이터 볼륨과 비용을 제어합니다.

### 싱크 킬스위치(Sink Killswitch)

싱크별 독립 비활성화를 지원합니다:

```typescript
interface SinkKillswitch {
  datadog?: boolean;     // Datadog 싱크 비활성화
  firstParty?: boolean;  // 퍼스트 파티 싱크 비활성화
}
```

설계 원칙은 **fail-open**입니다: 킬스위치 설정(Config)이 누락되거나 로드에 실패하면 싱크는 기본적으로 열려 있어, 설정(Config) 문제로 텔레메트리(Telemetry) 데이터가 조용히 손실되지 않도록 보장합니다.

---

## OpenTelemetry 스택

### 초기화 흐름

#### bootstrapTelemetry()

OTEL 관련 환경 변수를 설정하는 부트스트랩 단계 함수입니다. OpenTelemetry SDK가 초기화되기 전에 호출되어 필요한 모든 환경 설정(Config)이 준비되도록 합니다.

#### initializeTelemetry()

다음 세 가지 프로바이더를 생성하는 핵심 초기화 함수입니다:

- **MeterProvider** — 메트릭 프로바이더
- **LoggerProvider** — 로깅 프로바이더
- **TracerProvider** — 추적 프로바이더

### 익스포터(Exporter) 타입

여러 익스포터 타입을 지원합니다:

| 익스포터 | 전송 | 설명 |
|----------|-----------|-------------|
| `console` | stdout | 개발/디버그 용도; 콘솔에 출력 |
| `otlp` (gRPC) | gRPC | 고성능 바이너리 전송 |
| `otlp` (HTTP) | HTTP/JSON | HTTP 전송 |
| `otlp` (Protobuf) | HTTP/Protobuf | HTTP + Protobuf 인코딩 |
| `prometheus` | HTTP pull | Prometheus 호환 풀 모드 |

### BigQuery 메트릭

API 고객 및 C4E/Team 사용자의 경우, 메트릭 데이터는 더 깊은 분석과 보고를 위해 BigQuery로 내보내집니다.

### 리소스 병합

OTEL 리소스는 여러 레이어의 감지기를 병합하여 빌드됩니다:

```
기본 리소스
  + OS 감지기 (운영 체제 정보)
  + 호스트 아치 감지기 (호스트 아키텍처 정보)
  + 환경 감지기 (환경 변수 감지)
  → 병합된 리소스
```

### 플러시(Flush) 메커니즘

`flushTelemetry()`는 모든 프로바이더의 버퍼링된 데이터를 강제 플러시하며, 타임아웃은 **2초**입니다. 프로세스 종료 전에 호출되어 데이터 손실이 없도록 합니다.

---

## 세션 추적(Session Tracing) (sessionTracing.ts)

세션 추적 시스템은 상세한 요청 체인 추적 기능을 제공합니다.

### 활성화 조건

`isEnhancedTelemetryEnabled()` 함수는 향상된 텔레메트리(Telemetry)가 활성화되어 있는지 확인합니다:

- **기능 게이트**: 정적 기능 스위치
- **GrowthBook**: `enhanced_telemetry_beta`를 통한 동적 게이팅

### 스팬(Span) 타입

| 스팬 타입 | 설명 |
|-----------|-------------|
| `interaction` | 상호작용 스팬: 사용자 요청부터 Claude 응답까지의 전체 흐름을 래핑 |
| `llm_request` | LLM 요청 스팬: 단일 모델 호출 |
| `tool` | 도구 스팬: 도구 호출의 전체 생명주기 |
| `tool.blocked_on_user` | 도구 블로킹 스팬: 사용자 확인 대기 시간을 기록하는 자식 스팬 |
| `tool.execution` | 도구 실행 스팬: 실제 도구 실행 시간을 기록하는 자식 스팬 |
| `hook` | 훅(Hooks) 스팬: 훅(Hooks) 실행 |

#### 왜 세션 추적에 6가지 스팬 타입이 있는가?

단일 사용자 요청의 완전한 생명주기는 각 단계를 독립적으로 측정해야 합니다: `interaction`(전체 사용자 상호작용), `llm_request`(TTFT를 포함한 단일 모델 호출), `tool`(전체 도구 생명주기), `tool.blocked_on_user`(사용자 권한 확인 대기), `tool.execution`(실제 도구 실행), `hook`(훅(Hooks) 실행). 하나의 대략적인 스팬만으로는 "모델이 느렸는지", "도구가 느렸는지", "사용자 승인이 느렸는지"를 구분할 수 없습니다 — 이 세 가지는 최적화 전략이 완전히 다릅니다. 6가지 스팬 타입의 계층 관계(`interaction`이 `llm_request` + `tool`을 포함; `tool`이 `blocked_on_user` + `execution`을 포함)는 완전한 인과 체인을 형성합니다.

### 상호작용 스팬

상호작용 스팬은 루트 스팬으로, 사용자가 요청을 보낸 시점부터 Claude가 응답을 완료할 때까지의 전체 프로세스를 래핑합니다. 단일 상호작용 스팬은 여러 LLM 요청 스팬과 도구 스팬을 포함할 수 있습니다.

### LLM 스팬

LLM 요청 스팬은 단일 모델 호출의 상세 정보를 추적합니다:

- `input_tokens` — 입력 토큰 수
- `output_tokens` — 출력 토큰 수
- `cache_read_tokens` — 캐시에서 읽은 토큰 수
- `ttft_ms` — 첫 번째 토큰까지의 시간 (밀리초)

### 도구 스팬

도구 스팬은 도구 호출의 전체 생명주기를 추적하며, 내부적으로 다음을 포함할 수 있습니다:

- **blocked-on-user 자식 스팬**: 도구가 사용자 확인이 필요한 경우(예: 파일 쓰기 확인), 사용자 응답 대기 시간을 기록
- **execution 자식 스팬**: 실제 도구 실행에 소요된 시간

### 고아 스팬 정리

시스템은 백그라운드 주기적 작업을 통해 고아 스팬을 정리합니다:

- **TTL**: 30분
- 주기적으로 모든 활성 스팬을 스캔하고 TTL을 초과한 스팬을 강제 종료하고 제거
- 예외로 인해 닫히지 않는 스팬으로 인한 메모리 누수를 방지

### Perfetto 통합

**Perfetto** 추적 형식을 지원하며, 병렬 파일 쓰기를 통해 추적 파일을 생성합니다. 이 추적 파일은 Perfetto UI로 가져와 시각적 디버깅과 분석을 할 수 있습니다.

---

## 이벤트 로깅 (events.ts)

### logOTelEvent()

이벤트를 OTEL 로그 레코드로 발행합니다. 각 이벤트는 동시에 분석 이벤트이자 OTEL 로그 레코드입니다.

### 이벤트 순서 보장

전역 이벤트 순서는 **단조 증가** `eventSequence` 카운터로 보장됩니다. 각 이벤트는 생성 시 증가하는 시퀀스 번호가 할당되어, 비동기 환경에서도 인과 순서가 유지됩니다.

### 프롬프트 수정(Redaction)

`redactIfDisabled()` 함수는 `OTEL_LOG_USER_PROMPTS` 환경 변수가 설정되지 않은 경우 사용자 프롬프트를 자동으로 수정합니다. 기본 동작은 수정입니다(개인 정보 보호); 원시 프롬프트는 사용자가 해당 환경 변수를 명시적으로 활성화할 때만 보존됩니다.

---

## 플러그인(Plugin) 텔레메트리(Telemetry) (pluginTelemetry.ts)

플러그인(Plugin) 텔레메트리(Telemetry) 시스템은 플러그인(Plugin) 생태계를 위한 표준화된 텔레메트리(Telemetry) 기능을 제공합니다.

### 플러그인(Plugin) ID 해싱

`hashPluginId()` 함수는 플러그인(Plugin) ID에 **SHA256 해싱**을 적용하여 처음 **16자**를 익명화된 식별자로 사용합니다. 이를 통해 실제 플러그인(Plugin) 이름이나 경로가 텔레메트리(Telemetry) 데이터에 노출되지 않습니다.

### 플러그인(Plugin) 범위 분류

`getTelemetryPluginScope()` 함수는 플러그인(Plugin)을 다음 범위로 분류합니다:

| 범위 | 설명 |
|-------|-------------|
| `official` | Anthropic 공식 플러그인(Plugin) |
| `org` | 조직 수준 플러그인(Plugin) |
| `user-local` | 사용자 로컬 플러그인(Plugin) |
| `default-bundle` | 기본 번들 플러그인(Plugin) |

### 플러그인(Plugin) 명령 오류 분류

`classifyPluginCommandError()`는 플러그인(Plugin) 명령 실행 오류를 5가지 카테고리로 분류합니다:

1. **network** — 네트워크 관련 오류 (연결 실패, 타임아웃 등)
2. **not-found** — 명령 또는 리소스를 찾을 수 없음
3. **permission** — 권한 부족
4. **validation** — 입력 유효성 검사 실패
5. **unknown** — 기타 미분류 오류

### 세션 수준 플러그인(Plugin) 분석

`logPluginsEnabledForSession()`은 세션 시작 시 현재 활성화된 모든 플러그인(Plugin)을 기록하여, 플러그인(Plugin)별 세분성으로 분석 이벤트를 보고합니다.

### 로드 오류 분석

`logPluginLoadErrors()`는 플러그인(Plugin) 로딩 중 발생하는 오류를 기록하여, 오류별 세분성으로 보고함으로써 플러그인(Plugin) 호환성 및 설정(Config) 문제를 진단하는 데 도움을 줍니다.

---

## 비용 추적(Cost Tracking) (cost-tracker.ts)

비용 추적 시스템은 API 호출 토큰 소비와 비용을 실시간으로 추적합니다.

### 누적 추적

모델별로 다음 메트릭을 누적합니다:

| 메트릭 | 설명 |
|--------|-------------|
| `input_tokens` | 입력 토큰 수 |
| `output_tokens` | 출력 토큰 수 |
| `cache_read_tokens` | 캐시에서 읽은 토큰 수 |
| `cache_creation_tokens` | 캐시 생성 토큰 수 |
| `cost` | 누적 비용 (USD) |
| `duration` | 누적 지속 시간 |

### 비용 형식화

`formatTotalCost()` 함수는 모델 사용량의 분류 표시를 생성하여, 모델별로 각 메트릭과 비용을 나열합니다.

### 세션 영속화

- `saveCurrentSessionCosts()` — 현재 세션의 비용 상태를 디스크에 저장
- `restoreCostStateForSession()` — 세션이 복원될 때 디스크에서 이전 비용 상태를 로드

### StoredCostState 타입

```typescript
interface StoredCostState {
  input_tokens: number;
  output_tokens: number;
  cache_read_tokens: number;
  cache_creation_tokens: number;
  cost: number;
  duration: number;
  // ... 기타 추적된 필드
}
```

모든 추적된 필드의 완전한 스냅샷을 포함하여, 세션 간 비용 상태의 매끄러운 연속성을 가능하게 합니다.

### React 훅(Hook) (costHook.ts)

`costHook.ts`는 사용자가 종료할 때 비용 요약을 표시하는 React 훅(hook)을 제공하며, 현재 세션의 총 토큰 소비와 비용을 포함합니다.

---

## 게이트웨이 감지

시스템은 사용자의 요청 경로에 있을 수 있는 API 게이트웨이 또는 프록시를 자동으로 감지하고 식별할 수 있습니다.

### 알려진 게이트웨이 지문

응답 헤더, 요청 특성 및 기타 신호를 통해 다음 게이트웨이를 식별합니다:

| 게이트웨이 | 설명 |
|---------|-------------|
| **LiteLLM** | 오픈소스 LLM 프록시 게이트웨이 |
| **Helicone** | LLM 관찰성 플랫폼 |
| **Portkey** | AI 게이트웨이 및 관찰성 |
| **Cloudflare** | Cloudflare AI 게이트웨이 |
| **Kong** | Kong API 게이트웨이 |
| **Braintrust** | AI 평가 및 프록시 플랫폼 |
| **Databricks** | Databricks 모델 서빙 |

감지된 게이트웨이 정보는 텔레메트리(Telemetry) 데이터에 기록되어, Anthropic이 사용자의 API 접근 토폴로지와 게이트웨이가 도입할 수 있는 문제를 이해하는 데 도움을 줍니다.

---

## 엔지니어링 실천 가이드

### 새 분석 이벤트 추가

**단계 체크리스트:**

1. **`logEvent()`를 사용하여 이벤트 전송**: `src/services/analytics/index.ts`에서 `logEvent` 임포트
   ```typescript
   import { logEvent } from 'src/services/analytics/index.js'
   logEvent('tengu_your_event_name', {
     property1: 'value',
     property2: 123,
   })
   ```
2. **이벤트 이름 및 속성 정의**: 이벤트 이름은 `tengu_` 접두사를 사용합니다 (소스 코드의 모든 내부 이벤트가 이 규약을 따름)
3. **Proto 필드 주의**: `_PROTO_*` 접두사가 붙은 속성은 퍼스트 파티 백엔드에만 전송되며, `stripProtoFields()`에 의해 Datadog에서는 제거됨
4. **이중 채널 라우팅**: 이벤트는 Datadog(실시간 모니터링)와 1P(장기 분석)에 동시에 전송됨; 각 채널은 `SinkKillswitch`를 통해 독립적으로 비활성화 가능
5. **샘플링 제어**: 고빈도 이벤트는 `shouldSampleEvent()`를 사용하여 다운샘플링; 새 이벤트에 샘플링이 필요한지 평가

**참고**: `analytics/index.ts`의 헤더 주석에는 명시적으로 다음과 같이 나와 있습니다: "DESIGN: This module has NO dependencies to avoid import cycles" — 이 모듈에 다른 비즈니스 모듈에 대한 의존성을 도입하지 마십시오.

### 누락된 텔레메트리(Telemetry) 디버깅

**문제 해결 단계:**

1. **환경 변수 확인**: `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`을 설정하면 비필수 텔레메트리(Telemetry)가 비활성화됨
2. **싱크 킬스위치(Sink Killswitch) 확인**: `sinkKillswitch.ts`의 `datadog`와 `firstParty` 스위치는 채널을 독립적으로 비활성화할 수 있음
3. **GrowthBook 게이트 확인**: Datadog 이벤트는 `tengu_log_datadog_events` 기능 게이트에 의해 게이팅됨; `shouldTrackDatadog()`가 게이팅으로 인해 false를 반환할 수 있음
4. **싱크가 연결되었는지 확인**: `attachAnalyticsSink()` 호출 전에는 이벤트가 큐에 저장됨; 싱크가 연결되지 않으면 이벤트가 전송되지 않음
5. **fail-open 원칙**: 킬스위치 설정(Config)이 누락되거나 로드에 실패하면 싱크는 기본적으로 열려 있음 — 텔레메트리(Telemetry)가 여전히 누락된다면 문제는 네트워크 계층에 있을 수 있음

**핵심 소스 위치**:
- `analytics/index.ts` — 이벤트 큐 및 싱크 연결
- `analytics/sinkKillswitch.ts` — 킬스위치 로직 (주석 주의: "Must NOT be called from inside is1PEventLoggingEnabled()")
- `analytics/datadog.ts` — Datadog 싱크 구현 (주석: "use via src/services/analytics/index.ts > logEvent")

### 성능 추적

**Perfetto 추적 파일을 사용하여 요청 체인 성능 분석:**

1. 세션 추적이 Perfetto 형식으로 추적 파일을 생성
2. [Perfetto UI](https://ui.perfetto.dev/)로 가져와 시각적 분석
3. 6가지 스팬 타입이 완전한 인과 체인을 제공: `interaction` → `llm_request` + `tool` (`blocked_on_user` + `execution` 포함) + `hook`
4. 핵심 메트릭: `ttft_ms` (첫 번째 토큰까지의 시간), `input_tokens`/`output_tokens` (토큰 소비), `cache_read_tokens` (캐시 적중)

**향상된 텔레메트리(Telemetry) 활성화**: `isEnhancedTelemetryEnabled()`를 통해 확인되며, `enhanced_telemetry_beta` GrowthBook 기능 게이트에 의해 게이팅됨

### OTEL 내보내기 설정(Config)

| 익스포터 타입 | 적합한 시나리오 |
|--------------|-------------------|
| `console` | 로컬 개발 및 디버깅 |
| `otlp` (gRPC) | 프로덕션에서 고성능 전송 |
| `otlp` (HTTP/JSON) | 방화벽이 gRPC를 차단할 때의 대안 |
| `prometheus` | 기존 Prometheus 인프라가 있는 경우 |

**플러시(Flush) 메커니즘**: `flushTelemetry()`는 2초 타임아웃을 가짐; 프로세스 종료 전에 호출하여 데이터 손실 방지.

### 일반적인 함정

| 함정 | 세부 정보 | 해결 방법 |
|---------|---------|---------|
| 텔레메트리(Telemetry)는 비동기 fire-and-forget | 전송이 보장되지 않음; 전송 실패는 조용히 처리됨 | 중요한 비즈니스 로직을 텔레메트리(Telemetry) 성공에 의존시키지 마십시오 |
| 텔레메트리(Telemetry)에 민감한 데이터를 로깅하지 마십시오 | `redactIfDisabled()`는 `OTEL_LOG_USER_PROMPTS`가 설정되지 않으면 자동으로 수정함 | 기본 동작은 수정; 원시 프롬프트는 명시적으로 활성화된 경우에만 보존됨 |
| 고아 스팬 메모리 누수 | 예외로 인해 스팬이 닫히지 않을 수 있음 | 시스템은 TTL 30분을 초과하는 고아 스팬을 자동으로 정리함 |
| 이벤트 시퀀스 번호는 인과 순서를 보장함 | `eventSequence`는 단조 증가 카운터 | 비동기 환경에서는 이벤트 순서를 결정하기 위해 타임스탬프보다 시퀀스 번호에 의존하십시오 |
| 플러그인(Plugin) 텔레메트리(Telemetry) 개인 정보 | `hashPluginId()`는 SHA256을 사용하여 처음 16자를 사용 | 원시 플러그인(Plugin) 이름/경로는 텔레메트리(Telemetry) 데이터에 보이지 않음 |


---

[← 오류 복구(Error Recovery)](../17-错误恢复/error-recovery-ko.md) | [인덱스](../README_KO.md) | [피드백 & 설문 →](../19-反馈与调查/feedback-system-ko.md)
