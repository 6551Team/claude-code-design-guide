# 비용 추적(Cost Tracking) 아키텍처 문서

> Claude Code v2.1.88의 비용 및 사용량 추적 시스템에 대한 완전한 기술 참조

---

## 핵심 추적 (cost-tracker.ts)

### 내보내기 함수

#### 집계 쿼리
| 함수 | 반환값 |
|------|--------|
| `getTotalCost()` | 총 비용 (USD) |
| `getTotalDuration()` | 총 지속 시간 |
| `getTotalAPIDuration()` | 총 API 호출 시간 |
| `getTotalAPIDurationWithoutRetries()` | 총 API 호출 시간 (재시도 제외) |

#### 토큰 통계
| 함수 | 반환값 |
|------|--------|
| `getTotalInputTokens()` | 총 입력 토큰 수 |
| `getTotalOutputTokens()` | 총 출력 토큰 수 |
| `getTotalCacheReadInputTokens()` | 캐시 읽기 입력 토큰 수 |
| `getTotalCacheCreationInputTokens()` | 캐시 생성 입력 토큰 수 |

#### 코드 변경 통계
| 함수 | 반환값 |
|------|--------|
| `addToTotalLinesChanged()` | 코드 줄 변경 기록 추가 |
| `getTotalLinesAdded()` | 총 추가된 줄 수 |
| `getTotalLinesRemoved()` | 총 제거된 줄 수 |

#### 기타 통계
| 함수 | 반환값 |
|------|--------|
| `getTotalWebSearchRequests()` | 총 웹 검색 요청 수 |

#### 모델 사용량
| 함수 | 반환값 |
|------|--------|
| `getModelUsage()` | 모든 모델의 사용량 데이터 |
| `getUsageForModel()` | 지정된 모델의 사용량 데이터 |

#### 형식화
| 함수 | 반환값 |
|------|--------|
| `formatCost(cost)` | 표시를 위한 달러 금액 형식화 |
| `formatTotalCost()` | 모델 사용량 분석 표시 (모델별 나열) |
| `hasUnknownModelCost()` | 알 수 없는 모델의 비용이 있는지 감지 |

#### 누적 및 영속화
| 함수 | 목적 |
|------|------|
| `addToTotalSessionCost()` | 모델별 사용량 누적 (입력/출력/캐시 토큰, 비용, 지속 시간) |
| `getStoredSessionCosts()` | 프로젝트 구성에서 저장된 세션 비용 읽기 |
| `restoreCostStateForSession()` | 비용 상태 복원 (sessionId가 일치할 때만) |
| `saveCurrentSessionCosts()` | 현재 세션 비용을 프로젝트 구성에 영속화 |

### 데이터 구조

```typescript
StoredCostState = {
  totalCostUSD: number,
  totalAPIDuration: number,
  totalAPIDurationWithoutRetries: number,
  totalToolDuration: number,
  totalLinesAdded: number,
  totalLinesRemoved: number,
  lastDuration: number,
  modelUsage: Record<string, {
    input_tokens: number,
    output_tokens: number,
    cache_read: number,
    cache_creation: number,
    cost: number,
    duration: number
  }>
}
```

### 설계 철학: 왜 단일 전역 카운터 대신 모델별 누적인가?

`cost-tracker.ts`에서 `modelUsage`는 모델 이름으로 인덱싱된 사용량 데이터를 저장하는 `Record<string, ModelUsage>` 구조를 사용합니다. 이 설계 결정은 다음을 기반으로 합니다:

1. **모델마다 가격이 다름** -- Opus와 Sonnet의 입력/출력 토큰당 가격은 크게 다릅니다; 단일 카운터로는 비용을 정확하게 계산할 수 없습니다.
2. **캐시 토큰의 차별화된 청구** -- 캐시 읽기와 캐시 생성의 가격은 일반 토큰과 다릅니다; 모델별 구조를 통해 각 토큰 유형을 독립적으로 추적할 수 있습니다.
3. **모델 전환 시나리오** -- 세션은 여러 모델을 사용할 수 있습니다(기본 모델 + 어드바이저 모델). `addToTotalSessionCost()` 함수(278-284줄)는 모든 호출에서 `model` 파라미터를 지정하여 비용이 올바른 모델에 귀속되도록 보장합니다.

### 설계 철학: 왜 세션 비용과 총 비용이 분리되어 있는가?

사용자에게는 두 가지 구별된 관심사가 있습니다:
- **"이 대화에 얼마나 비용이 들었는가?"** -- 세션 비용은 사용자가 단일 작업의 비용 효율성을 평가하는 데 도움이 됩니다.
- **"총 얼마나 지출했는가?"** -- 총 비용은 사용자의 예산 관리와 비용 계획에 도움이 됩니다.

소스 코드는 `saveCurrentSessionCosts()`를 통해 세션 비용을 프로젝트 구성에 영속화하며, `restoreCostStateForSession()`은 세션을 재개할 때 주어진 `sessionId`와 일치하는 비용만 복원합니다(다른 세션의 비용 데이터 혼동을 방지).

### 엔지니어링 실천

**부정확한 비용 문제 해결**:
1. 모델 가격표가 최신 상태인지 확인하십시오 -- `formatModelUsage()`는 `getCanonicalName(model)`을 사용하여 다른 모델 ID를 정규화한 후 표시를 위해 집계합니다; 모델 매핑이 올바른지 확인하십시오.
2. 캐시 토큰이 올바르게 차감되고 있는지 확인하십시오 -- `cacheReadInputTokens`와 `cacheCreationInputTokens`는 별도로 추적됩니다; 캐시 히트당 토큰 가격은 일반 토큰보다 낮습니다.
3. `hasUnknownModelCost()`의 반환값을 확인하십시오 -- 알 수 없는 모델의 비용이 있으면 가격표에 해당 항목이 없는 것입니다.
4. 어드바이저 모델 비용은 별도의 `addToTotalSessionCost()` 호출(316줄)을 통해 누적됩니다; 어드바이저 비용이 올바르게 포함되고 있는지 확인하십시오.

**새 모델에 대한 비용 추적 추가**:
- 가격표에 모델 항목을 추가합니다 (모델 이름 → 입력/출력/캐시 토큰당 USD 단가).
- `getCanonicalName()`이 새 모델 ID를 표시 이름에 올바르게 매핑할 수 있는지 확인하십시오.
- `contextWindow`와 `maxOutputTokens` 정보는 `getContextWindowForModel()`과 `getModelMaxOutputTokens()`를 통해 가져옵니다; 해당 구성 항목도 추가해야 합니다.

---

## React 훅 (costHook.ts)

### useCostSummary(getFpsMetrics?)

프로세스 종료 핸들러를 등록하여 세션이 종료될 때:
1. 비용 요약을 표시합니다 (청구 접근 권한이 있는 사용자만).
2. 세션 비용을 저장합니다.
3. FPS 메트릭을 기록합니다 (`getFpsMetrics` 함수가 제공된 경우).

---

## /cost 명령

### 조건부 표시 로직
- claude.ai 구독자: 비용 정보가 숨겨집니다 (구독에 포함되어 있으므로).
- ANT 사용자는 예외: 구독자라도 비용이 표시됩니다.
- `currentLimits.isUsingOverage`를 감지합니다: 초과 사용 메시지를 표시합니다.

### 출력
`formatTotalCost()`를 호출하여 모델별로 분류된 비용 세부 정보를 출력합니다.

---

## /stats 명령

### 구현
`<Stats>` 컴포넌트(Component)를 렌더링합니다 (`components/Stats`에서 임포트). 사용량 통계와 활동 정보를 표시합니다.

---

## /usage 명령

### 구현
`<Settings defaultTab="Usage">` 컴포넌트(Component)를 렌더링합니다.

### 가용성
claude-ai 구독자만 사용 가능합니다.

---

## /extra-usage 명령

### 기능
추가 사용량 할당량을 관리하며, 팀/엔터프라이즈 관리자가 초과 크레딧을 부여하는 것을 지원합니다.

### 사전 확인
- 팀/엔터프라이즈 관리자 자격을 확인합니다.
- 초과 크레딧을 가져오고 검증합니다.

### 실행 흐름
- **인터랙티브**: 작업을 위해 브라우저를 엽니다.
- **비인터랙티브**: 커맨드라인에서 직접 처리합니다.

### 게이트 조건
- `isExtraUsageAllowed()`: 추가 사용이 허용되는지 확인합니다.
- `isOverageProvisioningAllowed()`: 초과 프로비저닝이 허용되는지 확인합니다.


---

[← 파일 영속화](../41-文件持久化/file-persistence-ko.md) | [인덱스](../README_KO.md) | [셸 도구 체인 →](../43-Shell工具链/shell-toolchain-ko.md)
