# 제8장: 메시지 루프(Message Loop)와 스트리밍(Streaming)

> 스트리밍(Streaming)은 단순한 사용자 경험 최적화가 아니라, 전체 시스템의 근본적인 설계 패턴입니다.

---

## 8.1 스트리밍(Streaming)이 중요한 이유

AI 시스템에서 스트리밍(Streaming)은 두 가지 층위의 의미를 가집니다:

**사용자 경험 층**: 사용자는 완전한 응답을 기다릴 필요 없이 Claude의 출력을 실시간으로 볼 수 있습니다. 긴 작업의 경우, 빈 화면을 바라보는 대신 Claude가 무엇을 하고 있는지 즉시 알 수 있습니다.

**시스템 아키텍처 층**: 스트리밍(Streaming)은 전체 시스템을 **데이터 흐름(Data Flow) 파이프라인**으로 전환합니다. 메시지는 API에서 흘러나와 파싱, 도구 실행, 결과 역채움을 거쳐 다시 API로 돌아가며 연속적인 데이터 스트림을 형성합니다.

Claude Code는 아래에서 위까지 스트리밍(Streaming)을 위해 설계되었습니다.

---

## 8.2 메시지 타입 시스템

스트리밍(Streaming)을 깊이 이해하기 전에, Claude Code의 메시지 타입 시스템을 먼저 파악해야 합니다:

```typescript
// src/types/message.ts (단순화)
type Message =
  | UserMessage          // 사용자 입력
  | AssistantMessage     // Claude의 응답
  | SystemMessage        // 시스템 메시지 (도구 결과, 오류 등)
  | AttachmentMessage    // 첨부파일 (이미지, 파일)
  | ToolUseSummaryMessage // 도구 사용 요약
  | TombstoneMessage     // 삭제된 메시지의 플레이스홀더

type AssistantMessage = {
  type: 'assistant'
  uuid: string
  message: {
    content: ContentBlock[]  // 여러 타입의 블록을 포함할 수 있음
  }
  apiError?: string
}

type ContentBlock =
  | TextBlock           // 일반 텍스트
  | ThinkingBlock       // 씽킹 블록 (확장 씽킹 모드)
  | RedactedThinkingBlock // 편집된 씽킹 블록
  | ToolUseBlock        // 도구 호출(Tool Call) 요청
```

이 타입 시스템 설계는 중요합니다: **단일 AssistantMessage는 여러 타입의 콘텐츠 블록을 포함할 수 있습니다**. Claude는 하나의 응답에서 텍스트, 씽킹, 그리고 여러 도구 호출(Tool Call)을 출력할 수 있습니다.

---

## 8.3 스트리밍(Streaming) 응답 파싱

Claude API는 서버-전송 이벤트(SSE, Server-Sent Events) 스트림을 반환합니다. 스트리밍(Streaming) 파싱 아키텍처:

`query.ts`의 스트리밍(Streaming) 파싱은 대략 다음과 같습니다:

```typescript
// 단순화된 스트리밍 파싱 로직
async function* parseStream(apiStream) {
  let currentTextBlock = ''
  let currentThinkingBlock = ''
  const toolUseBlocks = new Map()

  for await (const event of apiStream) {
    switch (event.type) {
      case 'content_block_start':
        if (event.content_block.type === 'text') {
          // 텍스트 블록 시작
        } else if (event.content_block.type === 'thinking') {
          // 씽킹 블록 시작
        } else if (event.content_block.type === 'tool_use') {
          // 도구 호출 블록 시작
          toolUseBlocks.set(event.index, {
            id: event.content_block.id,
            name: event.content_block.name,
            input: ''
          })
        }
        break

      case 'content_block_delta':
        if (event.delta.type === 'text_delta') {
          currentTextBlock += event.delta.text
          // 즉시 UI에 yield하여 표시
          yield { type: 'text_delta', text: event.delta.text }
        } else if (event.delta.type === 'thinking_delta') {
          currentThinkingBlock += event.delta.thinking
          // 씽킹 블록은 UI에 yield하지 않음
        } else if (event.delta.type === 'input_json_delta') {
          // 도구 호출 파라미터는 스트리밍 JSON
          toolUseBlocks.get(event.index).input += event.delta.partial_json
        }
        break

      case 'content_block_stop':
        // 블록 종료, 완전한 블록 처리
        break

      case 'message_stop':
        // 메시지 종료, 완전한 도구 호출 목록 yield
        yield { type: 'tool_calls', calls: [...toolUseBlocks.values()] }
        break
    }
  }
}
```

핵심 포인트: **텍스트 블록은 실시간으로 yield되고, 도구 호출(Tool Call) 블록은 완료 후 처리됩니다**. 도구 호출(Tool Call)은 실행을 위해 완전한 JSON 파라미터가 필요하기 때문입니다.

---

## 8.4 StreamingToolExecutor: 스트리밍(Streaming) 도구 실행

`src/services/tools/StreamingToolExecutor.ts`는 도구 실행의 핵심입니다:

```typescript
class StreamingToolExecutor {
  async* execute(toolCalls: ToolUseBlock[], context: ToolUseContext) {
    // 모든 도구 호출을 병렬로 실행
    const executions = toolCalls.map(call =>
      this.executeSingle(call, context)
    )

    // 각 도구의 실행 과정을 스트리밍으로 yield
    for (const execution of executions) {
      for await (const event of execution) {
        yield event
      }
    }
  }

  async* executeSingle(call: ToolUseBlock, context: ToolUseContext) {
    // 도구 시작 이벤트 yield
    yield { type: 'tool_start', toolName: call.name, toolUseId: call.id }

    try {
      const tool = findToolByName(call.name, context.options.tools)
      const result = await tool.execute(call.input, context)

      // 도구 결과 yield
      yield { type: 'tool_result', toolUseId: call.id, result }
    } catch (error) {
      // 도구 오류 yield
      yield { type: 'tool_error', toolUseId: call.id, error }
    }
  }
}
```

---

## 8.5 메시지 정규화: normalizeMessagesForAPI

각 API 호출 전에 메시지 목록을 정규화해야 합니다. `normalizeMessagesForAPI()`는 다양한 경우를 처리합니다:

```typescript
function normalizeMessagesForAPI(messages: Message[]): APIMessage[] {
  return messages
    // API에 전송할 필요가 없는 메시지 타입 필터링
    .filter(msg => !isSyntheticMessage(msg))
    // 같은 타입의 인접 메시지 병합 (API는 user/assistant 교대를 요구함)
    .reduce(mergeAdjacentMessages, [])
    // 씽킹 블록의 특수 규칙 처리
    .map(handleThinkingBlocks)
    // 지나치게 큰 도구 결과 잘라내기
    .map(truncateLargeToolResults)
}
```

이 함수는 많은 엣지 케이스를 처리합니다:
- API는 메시지가 user/assistant로 교대해야 하며, 연속된 두 개의 user 메시지를 허용하지 않습니다
- 씽킹 블록에는 엄격한 위치 규칙이 있습니다
- 도구 결과는 매우 클 수 있으며 (예: 큰 파일 읽기), 잘라내기가 필요합니다

---

## 8.6 메시지 큐(Queue) 관리

Claude Code는 동시 입력을 처리하는 메시지 큐 시스템(`src/utils/messageQueueManager.ts`)을 갖추고 있습니다:

```typescript
// 사용자는 Claude가 실행 중에도 새 메시지를 입력할 수 있습니다
// 메시지 큐는 이 메시지들이 우선순위에 따라 처리되도록 보장합니다

const queue = {
  // 높은 우선순위: 슬래시 명령어 (/stop 등)
  // 보통 우선순위: 사용자 메시지
  // 낮은 우선순위: 백그라운드 태스크 메시지
}
```

이것은 실용적인 문제를 해결합니다: Claude가 긴 작업을 실행하는 동안 사용자가 중단하거나 지시를 수정하고 싶을 수 있으며, 메시지 큐는 이 작업들이 신속하게 응답할 수 있도록 보장합니다.

---

## 8.7 스트리밍(Streaming) 출력의 UI 렌더링

Claude Code는 CLI(Command Line Interface)용 React인 Ink를 사용하여 UI를 렌더링합니다. 스트리밍(Streaming) 텍스트 렌더링은 다음과 같이 동작합니다:

```tsx
// 단순화된 메시지 렌더링 컴포넌트
function StreamingMessage({ message }) {
  const [displayText, setDisplayText] = useState('')

  useEffect(() => {
    // 스트리밍 이벤트 구독
    const unsubscribe = subscribeToStream(message.id, (delta) => {
      setDisplayText(prev => prev + delta)
    })
    return unsubscribe
  }, [message.id])

  return <Text>{displayText}</Text>
}
```

Ink의 렌더링은 증분식입니다: 각 `setDisplayText`는 변경된 부분만 업데이트하며, 전체 인터페이스를 리렌더링하지 않습니다. 이를 통해 부드러운 스트리밍(Streaming) 출력을 보장합니다.

---

## 8.8 도구 호출(Tool Call)에 대한 UI 피드백

도구 실행 중 UI는 실시간 진행 상황을 표시합니다:

```
> 사용하지 않는 변수를 모두 찾아줘

Claude: 프로젝트에서 사용하지 않는 변수를 분석하겠습니다.

⠸ GlobTool: **/*.ts 검색 중...
✓ GlobTool: 47개 파일 발견

⠸ FileReadTool: src/main.ts 읽는 중...
✓ FileReadTool: 완료

⠸ FileReadTool: src/utils.ts 읽는 중...
✓ FileReadTool: 완료

분석 완료, 다음 사용하지 않는 변수들을 발견했습니다:
...
```

이 실시간 피드백을 통해 사용자는 Claude가 무엇을 하고 있는지 알고 언제든지 중단할지 결정할 수 있습니다.

---

## 8.9 백프레셔(Backpressure) 처리

도구 실행이 UI 렌더링보다 빠를 때, 메모리 오버플로우를 방지하기 위한 백프레셔(Backpressure) 메커니즘이 필요합니다:

```typescript
// Claude Code는 비동기 제너레이터의 자연스러운 백프레셔를 사용합니다
async function* query(params) {
  // 제너레이터는 소비자가 await할 때만 계속됩니다
  // 이것이 자연스럽게 백프레셔를 제공합니다
  for await (const event of apiStream) {
    yield event  // 소비자가 처리할 때까지 기다린 후 계속 진행
  }
}
```

비동기 제너레이터의 백프레셔(Backpressure)는 "무료"입니다: 소비자가 다음 값을 `await`하지 않으면 생산자는 계속 실행되지 않습니다.

---

## 8.10 스트리밍(Streaming)에서의 오류 경계

스트리밍(Streaming)에서의 오류 복구(Error Recovery)는 동기 코드보다 더 복잡합니다:

```typescript
async function* safeQuery(params) {
  try {
    yield* query(params)
  } catch (error) {
    if (error instanceof AbortError) {
      // 사용자가 능동적으로 중단, 정상 종료
      yield { type: 'interrupted' }
    } else if (isRetryableError(error)) {
      // 재시도 가능한 오류, 자동 재시도
      yield* safeQuery(params)
    } else {
      // 복구 불가능한 오류, 오류 메시지 yield
      yield { type: 'error', error: error.message }
    }
  }
}
```

---

## 8.11 요약

Claude Code의 메시지 루프(Message Loop)와 스트리밍(Streaming) 설계:

- **엔드투엔드 스트리밍(Streaming)**: API 응답부터 UI 렌더링까지 완전한 스트리밍(Streaming)
- **타입 안전 메시지 시스템**: 명확한 메시지 타입 계층 구조
- **병렬 도구 실행**: 여러 도구 호출(Tool Call)을 병렬로 처리
- **메시지 정규화**: 다양한 API 제약을 처리
- **메시지 큐**: 동시 입력을 처리
- **자연스러운 백프레셔(Backpressure)**: 비동기 제너레이터가 자연스럽게 백프레셔(Backpressure) 제공
- **계층적 오류 복구(Error Recovery)**: 중단, 재시도, 오류 각각에 대응하는 처리

이 스트리밍(Streaming) 아키텍처는 Claude Code가 긴 작업을 부드럽게 처리하는 데 필요한 기반입니다.

---

*다음 장: [도구 시스템(Tool System) 설계 철학](../part4/09-tool-design_ko.md)*
