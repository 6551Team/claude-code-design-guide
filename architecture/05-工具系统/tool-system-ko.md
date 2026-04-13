# 도구 시스템(Tool System)

> 소스 파일: `src/Tool.ts` (792줄), `src/tools.ts`, `src/services/tools/toolExecution.ts` (1745줄),
> `src/services/tools/toolOrchestration.ts` (189줄), `src/services/tools/StreamingToolExecutor.ts`,
> `src/tools/` (40개 이상의 도구 디렉터리)

---

## 1. 아키텍처 개요

```
tools.ts (등록 테이블)
  ├── getAllBaseTools() — 완전한 도구 목록
  ├── getTools() — 권한 필터링된 도구 목록
  └── assembleToolPool() — MCP 도구가 포함된 최종 도구 풀

Tool.ts (타입 정의)
  ├── Tool 인터페이스 — 도구 타입 계약
  └── ToolUseContext — 도구 실행 환경 (40개 이상의 속성)

services/tools/ (오케스트레이션 계층)
  ├── toolOrchestration.ts — 동시/직렬 스케줄링
  ├── toolExecution.ts — 단일 도구 실행 파이프라인
  ├── StreamingToolExecutor.ts — 스트리밍(Streaming) 도구 동시성
  └── toolHooks.ts — 도구 라이프사이클 훅(Hooks)
```

---

## 2. Tool.ts — 도구 타입 인터페이스 (792줄)

### 2.1 Tool 타입 핵심 필드

```typescript
export type Tool = {
  // === 신원 ===
  name: string                      // 도구 이름 (API에 전송, 고유 식별자)
  description: string               // 도구 설명 (API에 전송)
  inputSchema: z.ZodType            // Zod 입력 스키마 (런타임 유효성 검사)

  // === 실행 ===
  execute: (
    input: unknown,
    context: ToolUseContext,
  ) => AsyncGenerator<ToolProgress | Message, Message | void>
  // execute는 비동기 생성기로, 진행 이벤트와 중간 메시지를 yield할 수 있음

  // === 동시성 제어 ===
  isConcurrencySafe: (input: unknown) => boolean
  // 주어진 입력으로 다른 도구와 동시에 실행할 수 있는지 여부
  // 예: FileReadTool은 항상 true, BashTool은 커맨드가 읽기 전용인지 여부에 따라 다름

  isReadOnly: boolean | ((input: unknown) => boolean)
  // 도구가 읽기 전용인지 여부 (파일시스템을 수정하지 않음)

  // === 결과 제어 ===
  maxResultSizeChars?: number
  // 최대 도구 결과 문자 수, 이를 초과하는 결과는 디스크에 저장됨 (>20KB)
  // Infinity는 제한 없음을 의미 (일부 도구는 완전한 결과가 필요함)

  // === 스키마 및 설명 ===
  backfillObservableInput?: (input: unknown) => unknown
  // 관찰 가능한 입력 채우기 (스트리밍 입력 매개변수의 UI 표시용)

  // === 활성화 상태 ===
  isEnabled: () => boolean
  // 현재 환경/설정에서 도구가 사용 가능한지 여부

  // === MCP 정보 ===
  mcpInfo?: {
    serverName: string
    toolName: string
  }
  // MCP 도구의 서버 소스 정보

  // === 활동 설명 ===
  getActivityDescription?: (input: unknown) => string
  // 사람이 읽을 수 있는 활동 설명 반환 (UI에 표시)

  // === 권한 ===
  getPermissionDescription?: (input: unknown) => string
  // 권한 프롬프트의 작업 설명

  // === 지연 로딩 ===
  isDeferredTool?: boolean
  // 지연된 도구 여부 (ToolSearch를 통해 온디맨드 로딩)
}
```

### 2.2 ToolUseContext — 도구 실행 컨텍스트 (40개 이상의 속성)

```typescript
export type ToolUseContext = {
  // === 핵심 옵션 ===
  options: {
    commands: Command[]                     // 사용 가능한 커맨드 목록
    debug: boolean                          // 디버그 모드
    mainLoopModel: string                   // 메인 루프 모델
    tools: Tools                            // 사용 가능한 도구 목록
    verbose: boolean                        // 상세 모드
    thinkingConfig: ThinkingConfig          // 사고 설정
    mcpClients: MCPServerConnection[]       // MCP 클라이언트 연결
    mcpResources: Record<string, ServerResource[]>  // MCP 리소스
    isNonInteractiveSession: boolean        // 비대화형 세션
    agentDefinitions: AgentDefinitionsResult // 에이전트 정의
    maxBudgetUsd?: number                   // 최대 예산 (USD)
    customSystemPrompt?: string             // 사용자 정의 시스템 프롬프트
    appendSystemPrompt?: string             // 추가 시스템 프롬프트
    querySource?: QuerySource               // 쿼리 소스 재정의
    refreshTools?: () => Tools              // 도구 새로 고침 콜백
  }

  // === 컨트롤러 ===
  abortController: AbortController          // 중단 컨트롤러

  // === 파일 상태 ===
  readFileState: FileStateCache             // 파일 읽기 상태 LRU 캐시

  // === 앱 상태 ===
  getAppState(): AppState                   // 앱 상태 가져오기
  setAppState(f: (prev: AppState) => AppState): void  // 앱 상태 설정
  setAppStateForTasks?: (f: (prev: AppState) => AppState) => void  // 작업 범위 상태

  // === UI 콜백 ===
  handleElicitation?: (...)  => Promise<ElicitResult>   // URL 유도 처리
  setToolJSX?: SetToolJSXFn                             // 도구 JSX 설정
  addNotification?: (notif: Notification) => void       // 알림 추가
  appendSystemMessage?: (msg: SystemMessage) => void    // 시스템 메시지 추가
  sendOSNotification?: (opts: {...}) => void            // OS 알림

  // === 메모리 시스템(Memory System) ===
  nestedMemoryAttachmentTriggers?: Set<string>          // 중첩 메모리 트리거
  loadedNestedMemoryPaths?: Set<string>                 // 로드된 중첩 메모리 경로
  dynamicSkillDirTriggers?: Set<string>                 // 동적 스킬 디렉터리 트리거
  discoveredSkillNames?: Set<string>                    // 검색된 스킬(Skills) 이름

  // === 진행 콜백 ===
  setInProgressToolUseIDs: (f: (prev: Set<string>) => Set<string>) => void
  setHasInterruptibleToolInProgress?: (v: boolean) => void
  setResponseLength: (f: (prev: number) => number) => void
  pushApiMetricsEntry?: (ttftMs: number) => void
  setStreamMode?: (mode: SpinnerMode) => void
  onCompactProgress?: (event: CompactProgressEvent) => void
  setSDKStatus?: (status: SDKStatus) => void
  openMessageSelector?: () => void

  // === 히스토리 및 어트리뷰션 ===
  updateFileHistoryState: (updater: ...) => void
  updateAttributionState: (updater: ...) => void
  setConversationId?: (id: UUID) => void

  // === 에이전트 신원 ===
  agentId?: AgentId                                     // 서브 에이전트 ID
  agentType?: string                                    // 서브 에이전트 타입 이름

  // === 메시지 및 제한 ===
  requireCanUseTool?: boolean                           // 강제 권한 체크
  messages: Message[]                                   // 현재 메시지 목록
  fileReadingLimits?: { maxTokens?: number; maxSizeBytes?: number }
  globLimits?: { maxResults?: number }

  // === 권한 추적 ===
  toolDecisions?: Map<string, {
    source: string; decision: 'accept' | 'reject'; timestamp: number
  }>
  queryTracking?: QueryChainTracking                    // 쿼리 체인 추적

  // === 대화형 프롬프트 ===
  requestPrompt?: (sourceName: string, toolInputSummary?: string | null) =>
    (request: PromptRequest) => Promise<PromptResponse>

  // === 도구 신원 ===
  toolUseId?: string
  criticalSystemReminder_EXPERIMENTAL?: string

  // === 서브 에이전트 ===
  preserveToolUseResults?: boolean                      // 도구 결과 보존 (팀원)
  localDenialTracking?: DenialTrackingState             // 로컬 거부 추적
  contentReplacementState?: ContentReplacementState     // 콘텐츠 교체 상태
  renderedSystemPrompt?: SystemPrompt                   // 부모가 렌더링한 시스템 프롬프트
  userModified?: boolean                                // 사용자 수정 여부
}
```

---

[← API 클라이언트](../04-API客户端/api-client-ko.md) | [색인](../README_KO.md) | [권한 및 보안 →](../06-权限与安全/permission-security-ko.md)
