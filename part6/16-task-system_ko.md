# 제16장: 태스크 시스템(Task System) 설계

> 백그라운드 태스크는 에이전트가 "동기식 어시스턴트"에서 "비동기식 협업자"로 진화하는 핵심 열쇠입니다.

---

## 16.1 태스크 시스템(Task System)이 필요한 이유

초기 AI 어시스턴트는 완전히 동기식이었습니다. 사용자가 질문하면 AI가 답하고, 사용자는 기다립니다.

그러나 실제 엔지니어링 작업은 이와 다릅니다.
- 대형 프로젝트 빌드는 10분이 걸릴 수 있습니다.
- 전체 테스트 스위트 실행은 30분이 걸릴 수 있습니다.
- 데이터 처리 작업은 몇 시간이 걸릴 수 있습니다.

Claude가 이러한 태스크를 실행하는 동안 사용자가 아무것도 할 수 없다면 경험이 저하됩니다.

태스크 시스템(Task System)은 이 문제를 해결합니다. **Claude가 장시간 실행되는 태스크를 백그라운드에서 실행하면서 다른 사용자 요청에 계속 응답할 수 있도록 합니다**.

---

## 16.2 태스크 유형

`src/Task.ts`는 7가지 태스크 유형을 정의합니다.

```typescript
type TaskType =
  | 'local_bash'          // 로컬 셸 명령어 (가장 일반적)
  | 'local_agent'         // 로컬 서브 에이전트(Sub-Agent) (독립적인 Claude 인스턴스)
  | 'remote_agent'        // 원격 에이전트 (CCR에서 실행)
  | 'in_process_teammate' // 인 프로세스 협업 에이전트 (공유 메모리)
  | 'local_workflow'      // 로컬 워크플로우 (다단계 태스크)
  | 'monitor_mcp'         // MCP(Model Context Protocol) 모니터링 태스크
  | 'dream'               // 자동 드림 모드 (실험적)
```

각 유형은 서로 다른 실행 환경과 기능을 가집니다.

| 유형 | 실행 위치 | 격리 수준 | 통신 방법 |
|------|---------|---------|---------|
| local_bash | 로컬 프로세스 | 낮음 | stdout/stderr |
| local_agent | 로컬 서브프로세스 | 중간 | 파일 + 메시지 |
| remote_agent | 원격 서버 | 높음 | HTTP API |
| in_process_teammate | 동일 프로세스 | 없음 | 공유 메모리 |

---

## 16.3 태스크 상태 머신

각 태스크는 명확한 상태 머신을 가집니다.

```
pending → running → completed
                 ↘ failed
                 ↘ killed
```

```typescript
type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'

// 종료 상태 확인
function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

종료 상태는 되돌릴 수 없습니다. 태스크가 완료(completed), 실패(failed), 또는 종료(killed)되면 다른 상태로 전환될 수 없습니다. 이 설계는 상태 머신의 혼란을 방지합니다.

---

## 16.4 태스크 ID 설계

태스크 ID 설계는 흥미롭습니다.

```typescript
// 태스크 ID 접두사
const TASK_ID_PREFIXES = {
  local_bash: 'b',
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  local_workflow: 'w',
  monitor_mcp: 'm',
  dream: 'd',
}

// 태스크 ID 생성: 접두사 + 8개의 랜덤 문자
// 예시: b3k9x2mf (로컬 bash 태스크)
//       a7p1n4qz (로컬 에이전트 태스크)
function generateTaskId(type: TaskType): string {
  const prefix = TASK_ID_PREFIXES[type]
  const bytes = randomBytes(8)
  // base-36 사용 (숫자 + 소문자)
  // 36^8 ≈ 2.8조 가지 조합으로 브루트 포스 공격을 방지하기에 충분
  return prefix + encode(bytes, TASK_ID_ALPHABET)
}
```

접두사를 통해 태스크 유형을 즉시 확인할 수 있으며, 랜덤 접미사는 고유성을 보장합니다. 주석에는 보안 고려 사항이 명시되어 있습니다. **브루트 포스 심볼릭 링크 공격 방지**.

---

## 16.5 태스크 출력 영속성

각 태스크의 출력은 디스크에 기록됩니다.

```typescript
type TaskStateBase = {
  id: string
  type: TaskType
  status: TaskStatus
  description: string
  outputFile: string    // 출력 파일 경로
  outputOffset: number  // 읽기 오프셋 (증분 읽기용)
  startTime: number
  endTime?: number
  notified: boolean     // 완료 알림 전송 여부
}
```

출력 파일의 장점:
- **영속성**: 프로세스가 재시작되어도 태스크 출력이 유실되지 않습니다.
- **증분 읽기**: `TaskOutputTool`이 `outputOffset`부터 읽어 중복 읽기를 방지합니다.
- **대용량 출력 지원**: 메모리 제한이 없으며 GB 수준의 출력을 처리할 수 있습니다.

---

## 16.6 태스크 라이프사이클 관리

```typescript
// 태스크 생성
const taskId = await TaskCreateTool.execute({
  command: 'npm run build',
  description: '프로덕션 버전 빌드'
}, context)
// 반환값: { taskId: 'b3k9x2mf' }

// 상태 확인
const status = await TaskGetTool.execute({ taskId }, context)
// 반환값: { status: 'running', outputOffset: 1024 }

// 출력 읽기 (증분)
const output = await TaskOutputTool.execute({
  taskId,
  block: false  // 논블로킹, 현재 출력 즉시 반환
}, context)

// 완료 대기 (블로킹)
const result = await TaskOutputTool.execute({
  taskId,
  block: true,  // 블로킹, 태스크 완료 대기
  timeout: 300000  // 5분 타임아웃
}, context)

// 태스크 중지
await TaskStopTool.execute({ taskId }, context)
```

---

## 16.7 메인 대화와 태스크 조율

태스크가 백그라운드에서 실행되는 동안 메인 대화는 계속될 수 있습니다. Claude는 다음을 수행할 수 있습니다.

1. 여러 백그라운드 태스크 시작
2. 다른 사용자 요청을 계속 처리
3. 주기적으로 태스크 상태 확인
4. 태스크 완료 시 결과 보고

```
사용자: 프론트엔드와 백엔드 테스트를 동시에 실행해줘

Claude: 네, 두 테스트 스위트를 병렬로 실행하겠습니다.

  → TaskCreateTool: npm run test:frontend (태스크 ID: b1a2b3c4)
  → TaskCreateTool: npm run test:backend (태스크 ID: b5d6e7f8)

  두 테스트가 백그라운드에서 실행 중입니다. 진행 상황을 확인해보겠습니다...

  → TaskGetTool: b1a2b3c4 → running (30초 경과)
  → TaskGetTool: b5d6e7f8 → running (30초 경과)

  [2분 후]

  → TaskGetTool: b1a2b3c4 → completed ✓
  → TaskGetTool: b5d6e7f8 → failed ✗

  프론트엔드 테스트 통과! 백엔드 테스트 실패, 에러를 확인해보겠습니다...
  → TaskOutputTool: b5d6e7f8 → [에러 출력]
```

---

## 16.8 태스크 정리 메커니즘

태스크는 완료 후 리소스를 정리해야 합니다.

```typescript
type TaskHandle = {
  taskId: string
  cleanup?: () => void  // 정리 함수 (프로세스 종료, 리소스 해제 등)
}
```

`cleanup` 함수는 태스크 종료 시 호출되어 다음을 보장합니다.
- 서브프로세스가 정상적으로 종료됨
- 임시 파일이 삭제됨
- 네트워크 연결이 닫힘

---

## 16.9 요약

태스크 시스템(Task System)은 Claude Code가 동기식 어시스턴트에서 비동기식 협업자로 진화할 수 있게 합니다.

- **7가지 태스크 유형**: 단순 셸 명령어부터 복잡한 멀티 에이전트(Multi-Agent) 협업까지 포괄
- **명확한 상태 머신**: pending → running → 종료
- **영속적 출력**: 태스크 출력이 디스크에 기록되며 증분 읽기 지원
- **병렬 실행**: 여러 태스크를 동시에 실행 가능
- **라이프사이클 관리**: 생성, 모니터링, 읽기, 중지를 위한 완전한 도구 세트

---

*다음 장: [멀티 에이전트(Multi-Agent) 아키텍처](./17-multi-agent_ko.md)*
