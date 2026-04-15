# 제24장: 성능 최적화

> 성능은 최적화하는 것이 아니라 설계하는 것입니다.

---

## 24.1 Claude Code의 성능 과제

Claude Code는 고유한 성능 과제에 직면합니다.

**시작 지연**: 사용자는 CLI 도구가 빠르게 시작되길 기대합니다. 그러나 Claude Code는 많은 모듈을 로드하고, 상태를 초기화하며, MCP 서버에 연결해야 합니다.

**API 지연**: 각 API 호출에는 네트워크 지연이 있습니다. 다중 턴 도구 호출(Tool Call)의 경우 지연이 누적됩니다.

**대용량 파일 처리**: 코드베이스에는 많은 파일이 있을 수 있으며, 검색과 읽기가 효율적이어야 합니다.

**스트리밍 렌더링**: 터미널 UI는 눈에 띄는 끊김 없이 스트리밍 출력을 부드럽게 렌더링해야 합니다.

**메모리 사용량**: 긴 대화의 메시지 기록은 커질 수 있으므로 메모리 사용량을 제어해야 합니다.

---

## 24.2 시작 성능 최적화

Claude Code에는 전용 시작 성능 분석 도구가 있습니다(`src/utils/startupProfiler.ts`).

```typescript
// 주요 시작 단계에서 체크포인트 기록
profileCheckpoint('imports_loaded')
profileCheckpoint('config_read')
profileCheckpoint('mcp_connected')
profileCheckpoint('repl_ready')

// 시작 시간 분석 출력
// imports_loaded: 120ms
// config_read: 45ms
// mcp_connected: 230ms
// repl_ready: 395ms (전체)
```

**빠른 경로(Fast path) 최적화**: 단순 명령(`--version`, `--help`)의 경우 전체 초기화를 건너뜁니다.

```typescript
// main.tsx의 빠른 경로
if (args.includes('--version')) {
  console.log(VERSION)
  process.exit(0)  // 아무것도 초기화하지 않음
}
```

**프리페치(Prefetch) 최적화**: 시작 시 필요한 리소스를 병렬로 미리 가져옵니다.

```typescript
// 병렬 프리페치, 메인 흐름을 차단하지 않음
Promise.all([
  prefetchApiKeyFromApiKeyHelperIfSafe(),  // API 키 미리 가져오기
  preconnectToAPI(),                        // API 연결 사전 수립
  prefetchGitStatus(),                      // git 상태 미리 가져오기
])
```

---

## 24.3 API 호출 최적화

**프롬프트 캐싱(Prompt Caching)**:
시스템 프롬프트의 안정적인 부분은 API에 의해 캐시되어 이후 요청에서 재전송이 불필요합니다.

```typescript
// 안정적인 부분(캐시 가능)을 먼저 배치
const systemPrompt = [
  coreInstructions,    // 거의 변경되지 않음 → 높은 캐시 적중률
  toolDefinitions,     // 도구 세트가 안정적이면 변경되지 않음 → 높은 캐시 적중률
  claudeMdContent,     // 파일이 변경되지 않으면 변경되지 않음 → 중간 캐시 적중률
  gitStatus,           // 매번 달라질 수 있음 → 캐시되지 않음
]
```

프롬프트 캐싱은 입력 토큰 비용을 90% 이상 절감하면서 지연도 낮출 수 있습니다.

**스트리밍 처리**:
스트리밍 API를 사용하여 사용자가 완전한 응답을 기다릴 필요가 없습니다.

```typescript
// 스트리밍 API 호출
const stream = await anthropic.messages.stream({
  model: 'claude-sonnet-4-6',
  messages,
  stream: true,  // 스트리밍 활성화
})

// 응답을 즉시 처리 시작
for await (const chunk of stream) {
  yield chunk  // 실시간으로 사용자에게 출력
}
```

**병렬 도구 실행**:
여러 도구 호출(Tool Call)이 직렬이 아닌 병렬로 실행됩니다.

```typescript
// 모든 도구 호출을 병렬로 실행
const results = await Promise.all(
  toolCalls.map(call => executeTool(call, context))
)
```

---

## 24.4 파일 시스템 최적화

**파일 읽기 캐싱** (`src/utils/fileStateCache.ts`):
대화 내에서 동일한 파일은 한 번만 읽히고 이후에는 캐시를 사용합니다.

```typescript
type FileStateCache = Map<string, {
  content: string
  mtime: number      // 파일 수정 시간
  readTime: number   // 읽기 시간
}>

// 파일 읽기 시 캐시 확인
async function readFileWithCache(path: string, cache: FileStateCache) {
  const cached = cache.get(path)
  const mtime = await getFileMtime(path)

  if (cached && cached.mtime === mtime) {
    return cached.content  // 캐시 적중
  }

  const content = await readFile(path)
  cache.set(path, { content, mtime, readTime: Date.now() })
  return content
}
```

**ripgrep 통합**:
GrepTool은 파일 검색에 Node.js fs 모듈이 아닌 ripgrep을 사용하여 10~100배 더 빠릅니다.

**glob 최적화**:
GlobTool 결과가 수정 시간 순으로 정렬되어 가장 관련성 높은 파일이 먼저 표시되고 이후 읽기가 줄어듭니다.

---

## 24.5 메모리 최적화

**메시지 잘라내기**:
도구 결과가 클 수 있으며(대용량 파일 읽기 등), 제한을 초과하면 자동으로 잘라냅니다.

```typescript
// 도구 결과 크기 제한
const MAX_TOOL_RESULT_TOKENS = 25000

function truncateToolResult(result: string, maxTokens: number): string {
  const tokens = estimateTokens(result)
  if (tokens <= maxTokens) return result

  // 잘라내고 메모 추가
  const truncated = result.substring(0, estimateChars(maxTokens))
  return truncated + '\n\n[내용이 잘렸습니다. 원본 크기가 제한을 초과합니다]'
}
```

**자동 압축(Auto-compact)**:
메시지 기록이 토큰 제한을 초과하면 자동으로 압축합니다(제15장 참조).

**순환 버퍼(Circular Buffer)** (`src/utils/CircularBuffer.ts`):
제한된 수의 기록을 저장하고 가장 오래된 항목을 자동으로 삭제하는 데 사용됩니다.

---

## 24.6 렌더링 성능

Claude Code는 UI 렌더링에 Ink(CLI용 React)를 사용합니다. React의 가상 DOM 메커니즘은 효율적인 증분 업데이트를 보장합니다.

```tsx
// 변경된 부분만 다시 렌더링
function MessageList({ messages }) {
  return messages.map(msg => (
    <Message key={msg.uuid} message={msg} />
  ))
}
```

**React 컴파일러 최적화**:
소스 코드에서 React 컴파일러 흔적을 확인할 수 있습니다.

```typescript
// src/state/AppState.tsx
import { c as _c } from "react/compiler-runtime";

export function AppStateProvider(t0) {
  const $ = _c(13)  // React 컴파일러가 생성한 캐시
  // ...
}
```

React 컴파일러는 메모이제이션을 자동으로 추가하여 불필요한 재렌더링을 줄입니다.

---

## 24.7 Bun 런타임 성능 이점

Claude Code는 Node.js가 아닌 Bun을 런타임으로 사용합니다.

| 항목 | Node.js | Bun |
|------|---------|-----|
| 시작 시간 | ~100ms | ~10ms |
| 모듈 로딩 | 느림 (CommonJS) | 빠름 (네이티브 ESM) |
| TypeScript | 컴파일 필요 | 네이티브 지원 |
| 패키지 관리 | npm (느림) | bun (10~25배 빠름) |
| 내장 도구 | 적음 | 많음 (테스트, 번들링 등) |

Bun의 시작 속도 이점은 CLI 도구에서 특히 중요합니다. 사용자는 CLI 도구가 거의 즉시 시작되길 기대하기 때문입니다.

---

## 24.8 기능 플래그와 데드 코드 제거

Claude Code는 컴파일 타임 데드 코드 제거를 위해 `bun:bundle`의 `feature()` 함수를 사용합니다.

```typescript
// VOICE_MODE가 활성화된 경우에만 음성 관련 코드 포함
const VoiceProvider = feature('VOICE_MODE')
  ? require('../context/voice.js').VoiceProvider
  : ({ children }) => children  // 빈 구현체

// 빌드 시 비활성화된 기능의 코드는 완전히 제거됩니다
// 번들 크기를 줄이고 로딩 속도를 향상시킵니다
```

이를 통해 Claude Code는 다양한 시나리오(표준, 엔터프라이즈, 경량)에 맞는 서로 다른 번들을 빌드할 수 있으며, 각 번들에는 필요한 코드만 포함됩니다.

---

## 24.9 성능 모니터링

Claude Code에는 내장 성능 모니터링 기능이 있습니다.

```typescript
// API 호출 지연 추적
pushApiMetricsEntry?.(ttftMs)  // TTFT: Time To First Token (첫 번째 토큰까지의 시간)

// 도구 실행 시간 추적
const toolStart = Date.now()
const result = await tool.execute(input, context)
const toolDuration = Date.now() - toolStart

// 턴별 통계
turnToolDurationMs += toolDuration
turnToolCount++
```

이 메트릭은 성능 병목 지점을 파악하는 데 도움이 됩니다. API 지연이 높은 것인지, 도구 실행이 느린 것인지 확인할 수 있습니다.

---

## 24.10 정리

Claude Code의 성능 최적화는 체계적입니다.

- **시작 최적화**: 빠른 경로 + 병렬 프리페치 + 시작 프로파일링
- **API 최적화**: 프롬프트 캐싱 + 스트리밍 처리 + 병렬 도구 실행
- **파일 시스템 최적화**: 읽기 캐싱 + ripgrep + 스마트 정렬
- **메모리 최적화**: 결과 잘라내기 + 자동 압축 + 순환 버퍼
- **렌더링 최적화**: React 컴파일러 + 증분 업데이트
- **런타임 최적화**: Bun의 네이티브 성능 이점
- **빌드 최적화**: 기능 플래그 + 데드 코드 제거

성능 최적화는 전체 시스템 설계에 스며들어 있으며, 사후에 추가된 패치가 아닙니다.

---

*다음 장: [Claude Code 설계 원칙](../part9/25-design-principles_ko.md)*
