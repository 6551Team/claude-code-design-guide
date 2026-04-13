# 피드백 & 설문 시스템

Claude Code에는 사용자 피드백 수집, 설문, 사용 팁을 위한 완전한 내장 시스템이 있습니다. 피드백 평점, 기록 공유, 메모리 설문, 컴팩트(Compact) 후 설문, 긍정적 피드백 명령, 버그 피드백 명령, 팁 시스템 등 여러 서브모듈을 포함합니다.

---

## 피드백 설문 상태 기계(State Machine)

피드백 설문 컴포넌트는 유한 상태 기계(finite state machine)에 의해 구동되어 사용자 피드백 평점의 전체 생명주기를 관리합니다.

### 상태 전환

```
'closed' → 'open' → 'thanks' → 'transcript_prompt' → 'submitting' → 'submitted'
```

- **closed**: 초기 상태; 설문이 표시되지 않음
- **open**: 설문이 나타났으며 사용자 입력을 대기 중
- **thanks**: 사용자가 평점을 선택함; 감사 메시지가 표시됨
- **transcript_prompt**: 사용자에게 기록 공유 의향을 묻는 단계
- **submitting**: 기록 데이터가 제출 중
- **submitted**: 제출 완료

#### 왜 이 설계인가?

피드백 수집은 다단계 사용자 상호작용(평점 → 감사 → 기록 문의 → 제출 → 완료)을 포함합니다. 상태 기계는 흐름이 단계를 건너뛰거나 교착 상태에 빠지지 않도록 보장합니다. 6가지 상태 각각에는 명확하게 정의된 전임자와 후임자가 있으며, 임의적인 건너뛰기는 허용되지 않습니다 — 예를 들어 `closed`에서 `submitting`으로 직접 이동하거나 `submitted`에서 `open`으로 되돌아갈 수 없습니다. 이 엄격한 선형 진행은 데이터 수집 무결성을 보장합니다: 사용자는 시스템이 기록 공유를 물어보기 전에 반드시 평점을 제공해야 합니다. 상태 기계 패턴은 불리언 플래그 조합보다 유지 관리하고 디버깅하기 쉽습니다. 어느 시점에서든 시스템은 정확히 하나의 잘 정의된 상태에 있기 때문입니다.

### 응답 타입

```typescript
type FeedbackSurveyResponse = 'dismissed' | 'bad' | 'fine' | 'good';
```

### 숫자 키 입력

숫자 키 입력은 **400ms** 디바운스 지연으로 `useDebouncedDigitInput` 훅(hook)에 의해 처리됩니다:

| 키 | 의미 |
|-----|---------|
| `0` | 무시 (응답 없이 닫기) |
| `1` | 나쁨 (좋지 않은 경험) |
| `2` | 보통 (평균적인 경험) |
| `3` | 좋음 (훌륭한 경험) |

#### 왜 이 설계인가?

숫자 키는 설문 컨텍스트에서 이중 역할을 합니다: 평점 입력이자 일반 텍스트 입력(사용자가 "1. 첫 번째 단계..."와 같이 번호 목록을 입력할 수 있음). 400ms 디바운스 창은 시스템이 이 두 가지 의도를 구분할 수 있게 합니다 — 400ms 내에 사용자가 다른 문자를 계속 입력하면 숫자는 텍스트의 일부로 처리되어 제출이 취소됩니다; 400ms 내에 추가 입력이 없으면 숫자는 평점 선택으로 처리됩니다. 소스 코드 주석은 이를 명시적으로 설명합니다: *"Short enough to feel instant for intentional presses, long enough to cancel when the user types more characters"* (`useDebouncedDigitInput.ts`).

### 확률 게이팅

설문 표시 확률은 동적 설정(Config) `tengu_feedback_survey_config`에 의해 제어됩니다. 이 설정(Config)은 원격 소스에서 가져와 표시 조건을 충족하는 세션에서 설문이 나타날 가능성을 결정합니다.

### 쿨다운 세션 추적

시스템은 쿨다운 세션 수를 유지합니다. 사용자가 하나의 설문을 완료한 후, 설문은 일정 수의 후속 세션 동안 다시 나타나지 않아 과도한 방해를 방지합니다.

### 분석 계측

모든 설문 이벤트는 통합 분석 이벤트를 통해 보고됩니다:

```
이벤트 이름: 'tengu_feedback_survey_event'
타입:
  - appeared  — 설문이 사용자에게 표시됨
  - responded — 사용자가 응답함
```

---

## 기록(Transcript) 공유

사용자가 피드백 평점을 완료한 후, 시스템은 제품 개선을 위해 세션 기록을 공유할 의향이 있는지 추가로 물어볼 수 있습니다.

### 응답 타입

```typescript
type TranscriptShareResponse = 'yes' | 'no' | 'dont_ask_again';
```

- **yes**: 현재 기록 공유에 동의
- **no**: 이번에는 공유하지 않음
- **dont_ask_again**: 다시는 묻지 않음 (영속화된 선호도)

### 제출 흐름 (submitTranscriptShare.ts)

1. **메시지 수집**: 현재 세션의 정규화된 메시지와 모든 서브에이전트 기록을 수집
2. **원시 JSONL 읽기**: 디스크에서 원시 JSONL 형식 기록 파일을 읽고, 과도하게 큰 파일 읽기로 인한 메모리 문제를 방지하기 위해 `MAX_TRANSCRIPT_READ_BYTES` 크기 제한으로 보호됨
3. **민감한 정보 수정**: 기록 내용에 `redactSensitiveInfo()`를 실행하여 잠재적으로 민감한 데이터(API 키, 토큰, 비밀번호 등) 제거

   #### 왜 이 설계인가?

   사용자가 공유한 대화 기록에는 API 키, 비밀번호, 토큰 및 기타 민감한 정보가 쉽게 포함될 수 있습니다 — 개발자들은 터미널에서 자격 증명을 자주 다룹니다. `redactSensitiveInfo()`는 업로드 전에 로컬 수정을 수행하여 민감한 데이터가 사용자의 기기를 절대 벗어나지 않도록 보장합니다. 소스 코드에서 `submitTranscriptShare.ts`는 최종 업로드 전에 `const content = redactSensitiveInfo(jsonStringify(data))`를 명시적으로 호출합니다 — 이것은 건너뛸 수 없는 필수 보안 체크포인트입니다. 이 "먼저 수정, 그 다음 업로드" 원칙은 전체 피드백 시스템에 침투합니다: `Feedback.tsx`에서 설명, 오류, 스택 추적을 포함한 모든 사용자 제공 콘텐츠는 예외 없이 수정됩니다.

4. **업로드**: HTTP POST를 통해 다음으로 전송:
   ```
   https://api.anthropic.com/api/claude_code_shared_session_transcripts
   ```

### 트리거 타입

기록 공유는 다음 시나리오에 의해 트리거될 수 있습니다:

| 트리거 타입 | 설명 |
|--------------|-------------|
| `bad_feedback_survey` | 사용자가 피드백 설문에서 "나쁨"을 선택함 |
| `good_feedback_survey` | 사용자가 피드백 설문에서 "좋음"을 선택함 |
| `frustration` | 시스템이 사용자가 불만을 경험했을 수 있다고 감지함 |
| `memory_survey` | 메모리 설문 흐름 중에 트리거됨 |

---

## 메모리 설문 (useMemorySurvey)

메모리 설문은 자동 메모리 기능을 대상으로 하는 특화된 설문입니다.

### 트리거 조건

- 현재 세션 메시지에 **자동 메모리 파일 읽기**(즉, Claude가 자동으로 메모리 파일을 읽음)가 포함되어 있는지 확인
- 메모리 파일 읽기가 감지되면 **0.2** (20%) 확률로 설문 트리거

#### 왜 이 설계인가?

메모리 설문은 매번 설문을 표시하는 대신 `SURVEY_PROBABILITY = 0.2` 확률 게이팅(소스: `useMemorySurvey.tsx`, 21줄)을 사용합니다. 이는 사용자 경험과 데이터 수집 간의 균형을 맞춥니다: 20% 확률은 메모리 기능 사용 평균 5번 중 1번 설문이 트리거되어 사용자가 반복적으로 괴롭힘을 느끼지 않습니다; 그러나 합리적인 시간 내에 통계적으로 의미 있는 샘플을 수집할 만큼 충분히 높습니다. 설문은 기능 플래그 `tengu_dunwich_bell`에 의해서도 게이팅되어 충분한 데이터가 수집된 후 사용자를 괴롭히는 것을 중단하기 위해 원격으로 비활성화할 수 있습니다.

### 분석 계측

```
이벤트 이름: 'tengu_memory_survey_event'
```

메모리 기능에 대한 사용자 피드백을 기록하여 자동 메모리 시스템의 실질적인 효과를 평가하는 데 도움을 줍니다.

---

## 컴팩트(Compact) 후 설문 (usePostCompactSurvey)

컴팩트(Compact) 후 설문은 세션이 **대화 컴팩트(Compact)**(대화 요약/압축)을 겪은 후 트리거됩니다.

대화 컨텍스트가 너무 길어져 자동 컴팩트(Compact)가 트리거되면, 시스템은 컴팩트(Compact) 완료 후 사용자에게 컴팩트(Compact) 결과를 평점하도록 요청하여 정보 손실, 컨텍스트 유지 품질 등에 대한 피드백을 수집합니다.

---

## Good Claude 명령 (/good-claude)

`/good-claude`는 긍정적 피드백 단축 명령입니다.

사용자가 Claude의 특정 응답에 만족하면, 전체 설문 흐름을 거치지 않고 이 명령을 통해 긍정적 피드백을 빠르게 전송할 수 있습니다. 이는 사용자가 "이 응답은 훌륭했습니다"라고 신호를 보내는 저마찰 방법을 제공합니다.

---

## 피드백 명령 (/feedback)

`/feedback` 명령은 완전한 피드백 제출 인터페이스를 제공합니다.

### 별칭

- `/bug` — `/feedback`의 별칭으로 사용 가능

### 게이팅 조건

다음 조건에서는 이 명령을 **사용할 수 없습니다**:

- **Bedrock** 백엔드 사용 시
- **Vertex** 백엔드 사용 시
- **Foundry** 백엔드 사용 시
- 사용자가 **ANT** (Anthropic 내부)에 속하는 경우
- 조직 정책이 `product_feedback`을 허용하지 않는 경우

### 렌더링

명령이 트리거되면 다음 파라미터와 함께 `Feedback` 컴포넌트를 렌더링합니다:

- **abort signal**: 피드백 제출 흐름을 취소하는 데 사용
- **messages**: 현재 세션 메시지 컨텍스트
- **initial description**: 초기 설명 텍스트 (예: 명령 인수에서 전달됨)

---

## 팁 시스템

팁 시스템은 사용자가 Claude 응답을 기다리는 동안(예: 스피너가 돌아가는 동안) 유용한 팁을 표시합니다.

### 팁 레지스트리 (tipRegistry.ts)

시스템은 **60개 이상의 팁**을 등록하며, 각각 다음 구조를 가집니다:

```typescript
interface Tip {
  id: string;                          // 고유 식별자
  content: () => Promise<string>;      // 비동기 콘텐츠 생성 함수
  cooldownSessions: number;            // 쿨다운 세션 수
  isRelevant: () => Promise<boolean>;  // 비동기 관련성 확인 함수
}
```

- **id**: 각 팁의 고유 식별자
- **content**: 팁의 표시 콘텐츠를 반환하는 비동기 함수 (동적 생성 지원)
- **cooldownSessions**: 표시된 후 팁이 쿨다운해야 하는 세션 수, 반복 표시 방지
- **isRelevant**: 현재 컨텍스트에서 팁이 관련성이 있는지 결정하는 비동기 함수 (예: 일부 팁은 특정 플랫폼이나 설정(Config)에서만 관련됨)

### 선택 알고리즘 (tipScheduler.ts)

다음 팁을 선택하기 위해 **가장 오래 표시되지 않은 팁 우선** 전략을 사용합니다:

- 관련성 조건을 충족하고 쿨다운 기간 중이 아닌 모든 팁 중에서 가장 최근에 표시된 팁을 선택
- 팁 표시가 가능한 한 균등하게 분배되도록 하여 사용자가 동일한 콘텐츠를 반복해서 보는 것을 방지

#### 왜 이 설계인가?

60개 이상의 팁이 등록된 경우, 무작위 선택은 일부 팁이 전혀 표시되지 않고 다른 팁은 반복 표시될 위험이 있습니다. `selectTipWithLongestTimeSinceShown()` 함수(`tipScheduler.ts`)는 "마지막으로 표시된 이후 세션 수"로 내림차순 정렬하여 가장 오랫동안 표시되지 않은 팁에 우선순위를 부여합니다. `tipHistory.ts`에서 `getSessionsSinceLastShown()`은 `numStartups - lastShown`을 반환하고 한 번도 표시되지 않은 팁에는 `Infinity`를 반환하여 새 팁이 항상 먼저 표시되도록 보장합니다. 이 결정론적 스케줄링은 무작위 선택보다 더 공정합니다: 모든 팁이 표시될 기회를 보장하여 사용자가 지속적인 사용을 통해 모든 제품 기능을 점진적으로 발견할 수 있게 합니다.

### 히스토리 영속화 (tipHistory.ts)

팁 표시 히스토리는 전역 설정(Config)을 통해 영속화됩니다:

```typescript
// 저장 구조: tipId → numStartups
// 각 팁이 몇 번의 시작에서 표시되었는지 기록
Record<string, number>
```

이 데이터는 전역 설정(Config)에 저장되어 세션 간에 지속됩니다.

### 분석 계측

```
이벤트 이름: 'tengu_tip_shown'
```

팁이 표시될 때마다 보고되어 각 팁의 표시 빈도와 커버리지를 분석하는 데 사용됩니다.

### 커스텀 팁

사용자는 `settings.spinnerTipsOverride`를 설정하여 커스텀 팁 콘텐츠를 제공하고 기본 팁 목록을 오버라이드하거나 보완할 수 있습니다.

### 플러그인(Plugin) 팁

마켓플레이스 플러그인(Plugin)은 자체 팁을 등록할 수 있습니다. 이러한 플러그인(Plugin) 팁은 통합 팁 스케줄링 시스템에 통합되어 내장 팁과 함께 선택 및 표시에 참여합니다.

---

## 엔지니어링 실천 가이드

### 피드백 수집 트리거

**피드백 상태 기계(State Machine) 흐름:**

1. 사용자가 엄지 아래 버튼을 누르거나 (`/feedback` 또는 `/bug` 명령 사용) 피드백 흐름을 트리거
2. 상태 기계(State Machine) 전환: `closed → open → thanks → transcript_prompt → submitting → submitted`
3. 각 상태에는 명확하게 정의된 전임자와 후임자가 있으며; 단계 건너뛰기는 허용되지 않음 (예: `closed`에서 `submitting`으로 직접 이동 불가)
4. 사용자는 어느 단계에서든 취소할 수 있음 (피드백은 선택 사항)

**핵심 진입점:**
- `FeedbackSurvey.tsx` — 메인 피드백 설문 컴포넌트
- `useFeedbackSurvey.tsx` — 피드백 설문 훅(hook), 상태 및 확률 게이팅 관리
- `submitTranscriptShare.ts` — 기록 제출 흐름

### 피드백 상태 기계(State Machine) 디버깅

**문제 해결 단계:**

1. **현재 상태 확인**: 상태 기계(State Machine)가 어떤 단계에 있는지 (closed/open/thanks/transcript_prompt/submitting/submitted)?
2. **전환 조건 확인**: 전환을 트리거하는 조건이 충족되었는지 확인
3. **400ms 디바운스 효과**: `useDebouncedDigitInput`은 400ms 디바운스 창을 설정함; 이 창 내의 숫자 키 입력 (0=무시, 1=나쁨, 2=보통, 3=좋음)은 더 많은 입력이 이어지면 취소됨 — 이는 평점 의도와 일반 텍스트 입력을 구분하기 위한 것
4. **확률 게이팅**: 설문 표시 확률은 `tengu_feedback_survey_config` 동적 설정(Config)에 의해 제어되며; 매번 트리거되지 않음
5. **쿨다운 기간**: 하나의 설문 완료 후 쿨다운 세션 기간이 있음; 설문은 다음 여러 세션에서 나타나지 않음

**기록 제출 실패 디버깅:**
- `MAX_TRANSCRIPT_READ_BYTES` 제한 확인 (과도하게 큰 파일 읽기로 인한 메모리 문제 방지)
- `redactSensitiveInfo()` 수정이 올바르게 실행되는지 확인 (API 키, 비밀번호, 토큰이 제거됨)
- 네트워크 연결 확인 (업로드는 `https://api.anthropic.com/api/claude_code_shared_session_transcripts`로 이동)

### 설문 확률 커스터마이징

**메모리 설문:**
- `SURVEY_PROBABILITY = 0.2` (소스: `useMemorySurvey.tsx`, 21줄) — 트리거 확률 20%
- 기능 플래그 `tengu_dunwich_bell`에 의해 게이팅됨, 원격으로 비활성화 가능
- 자동 메모리 파일 읽기가 감지된 경우에만 트리거됨

**컴팩트(Compact) 후 설문:**
- `SURVEY_PROBABILITY = 0.2` (소스: `usePostCompactSurvey.tsx`, 15줄) — 트리거 확률 20%
- 세션이 대화 컴팩트(Compact)를 겪은 후 트리거됨

**피드백 설문:**
- 확률은 `tengu_feedback_survey_config` 원격 설정(Config)에 의해 제어됨
- 쿨다운 세션 수가 과도한 방해를 방지함

### 팁 커스터마이징

- **팁 오버라이드**: `settings.spinnerTipsOverride`를 설정하여 커스텀 팁 콘텐츠 제공
- **플러그인(Plugin) 팁**: 마켓플레이스 플러그인(Plugin)은 자체 팁을 등록할 수 있으며 통합 스케줄링에 통합됨
- **스케줄링 알고리즘**: 가장 오래 표시되지 않은 팁 우선 (`selectTipWithLongestTimeSinceShown()`); 새 팁은 항상 먼저 표시되도록 `Infinity`를 반환

### 일반적인 함정

| 함정 | 세부 정보 | 해결 방법 |
|---------|---------|----------|
| 피드백은 선택 사항 | 사용자는 어느 단계에서든 피드백 흐름을 취소할 수 있음 | UI는 취소를 정상적으로 처리해야 함; 흐름이 항상 완료된다고 가정하지 마십시오 |
| 기록 공유는 수정을 적용함 | `redactSensitiveInfo()`는 업로드 전에 로컬 수정을 수행함 — 이것은 필수 보안 체크포인트 | 수정이 불완전하다고 발견되면 우회하는 대신 `redactSensitiveInfo()`를 수정하십시오 |
| 400ms 디바운스 창 | 숫자 키 평점은 400ms 내에 추가 입력이 없어야 트리거됨 | 빠른 연속 입력은 평점이 인식되지 않게 할 수 있음 |
| `/feedback` 명령에는 게이팅 조건이 있음 | Bedrock/Vertex/Foundry 백엔드, ANT 내부 사용자 또는 정책이 허용하지 않는 경우 사용 불가 | 피드백 진입점은 환경에 따라 다를 수 있음 |
| 팁 표시 히스토리는 세션 간에 지속됨 | 전역 설정(Config)에 저장됨 (`tipId → numStartups` 매핑) | 설정(Config) 지우기는 팁 표시 히스토리를 초기화함 |


---

[← 텔레메트리(Telemetry) & 분석](../18-遥测分析/telemetry-system-ko.md) | [인덱스](../README_KO.md) | [서비스 레이어 →](../20-服务层/services-complete-ko.md)
