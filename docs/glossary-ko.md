# 한국어 번역 용어 글로서리

> 이 문서는 Claude Code Design Guide 한국어 번역에서 사용하는 용어 통일 기준입니다.

## 핵심 용어

| English | Korean |
|---------|--------|
| Agent | 에이전트(Agent) |
| Agent Runtime System | 에이전트 런타임 시스템(Agent Runtime System) |
| Query Engine | 쿼리 엔진(Query Engine) |
| Context Window | 컨텍스트 윈도우(Context Window) |
| Context Engineering | 컨텍스트 엔지니어링(Context Engineering) |
| Context Management | 컨텍스트 관리(Context Management) |
| Context Compression | 컨텍스트 압축(Context Compression) |
| Message Loop | 메시지 루프(Message Loop) |
| Tool System | 도구 시스템(Tool System) |
| Tool Invocation / Tool Call | 도구 호출(Tool Call) |
| Permission Model | 권한 모델(Permission Model) |
| System Prompt | 시스템 프롬프트(System Prompt) |
| Multi-Agent | 멀티 에이전트(Multi-Agent) |
| MCP (Model Context Protocol) | MCP(Model Context Protocol) |
| Hooks | 훅(Hooks) |
| Skills | 스킬(Skills) |
| Plugin | 플러그인(Plugin) |
| Compact / Auto-Compact | 컴팩트(Compact) / 자동 컴팩트(Auto-Compact) |
| State Management | 상태 관리(State Management) |
| Sandbox | 샌드박스(Sandbox) |
| Streaming | 스트리밍(Streaming) |
| REPL | REPL |
| CLI | CLI |
| Slash Command | 슬래시 커맨드(Slash Command) |
| Coordinator | 코디네이터(Coordinator) |
| Task System | 태스크 시스템(Task System) |
| Memory System | 메모리 시스템(Memory System) |
| Extension System | 확장 시스템(Extension System) |
| Human-in-the-Loop | 휴먼 인 더 루프(Human-in-the-Loop) |
| Error Recovery | 오류 복구(Error Recovery) |
| Pipeline Mode | 파이프라인 모드(Pipeline Mode) |

## 아키텍처 관련 용어

| English | Korean |
|---------|--------|
| Architecture | 아키텍처(Architecture) |
| Design Pattern | 디자인 패턴(Design Pattern) |
| Module | 모듈(Module) |
| Dependency | 의존성(Dependency) |
| Data Flow | 데이터 흐름(Data Flow) |
| Source Tree | 소스 트리(Source Tree) |
| Layered Architecture | 계층 아키텍처(Layered Architecture) |
| Entry Point | 진입점(Entry Point) |

## 보안/권한 관련 용어

| English | Korean |
|---------|--------|
| Security | 보안(Security) |
| Permission Level | 권한 수준(Permission Level) |
| Auto-approve | 자동 승인(Auto-approve) |
| Dangerous Operation | 위험 작업(Dangerous Operation) |
| User Confirmation | 사용자 확인(User Confirmation) |

## 번역 규칙

1. **문체**: 경어체 (합니다/입니다)
2. **코드 블록**: 코드는 번역하지 않음, 주석만 한국어화
3. **기술 용어**: 한국어 번역 + 괄호 영문 병기 (예: "쿼리 엔진(Query Engine)")
4. **약어**: 원문 유지 (MCP, CLI, REPL, API 등)
5. **내부 링크**: `_ko.md` 파일끼리 상호 참조
6. **이미지 참조**: `-ko.svg`로 변경
