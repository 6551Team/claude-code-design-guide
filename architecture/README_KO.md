# Claude Code v2.1.88 완전한 엔지니어링 아키텍처 문서

**[English](./README_EN.md) | [中文](./README.md) | 한국어**

> 1884개의 TypeScript 소스 파일에 대한 역공학 분석을 기반으로 합니다.
> 분석 일자: 2026-03-31 | 빌드: 2026-03-30T21:59:52Z

## 문서 구성

이 문서는 46개의 전문 디렉터리로 구성되어 있으며, 각 디렉터리에는 하위 시스템에 대한 완전한 분석이 포함되어 있습니다:

| 챕터 | 내용 |
|---------|----------|
| [01 시스템 총람](./01-系统总览/system-overview-ko.md) | 소스 구조, 계층 아키텍처, 모듈 관계 |
| [02 시작 및 초기화](./02-启动与初始化/initialization-ko.md) | main.tsx → init → REPL 완전한 체인 |
| [03 쿼리 엔진](./03-查询引擎/query-engine-ko.md) | query.ts + QueryEngine.ts 핵심 루프 |
| [04 API 클라이언트](./04-API客户端/api-client-ko.md) | 멀티 백엔드, 스트리밍(Streaming), 재시도, 에러 분류 |
| [05 도구 시스템](./05-工具系统/tool-system-ko.md) | 40개 이상의 도구 등록, 오케스트레이션, 스트리밍 실행 |
| [06 권한 및 보안](./06-权限与安全/permission-security-ko.md) | 6가지 모드, 분류기, 샌드박스(Sandbox), 경로 유효성 검사 |
| [07 컨텍스트 관리](./07-上下文管理/context-management-ko.md) | 3계층 압축, 토큰 예산, 캐시 제어 |
| [08 MCP 통합](./08-MCP集成/mcp-integration-ko.md) | 설정, 전송, 인증, 지연 로딩 |
| [09 훅 시스템](./09-Hooks系统/hooks-system-ko.md) | 13개 이벤트 훅(Hooks) + 70개 이상의 React 훅(Hooks) |
| [10 스킬 시스템](./10-Skills系统/skills-system-ko.md) | 17개 내장 스킬(Skills), 스킬 검색, 포크 실행 |
| [11 멀티 에이전트](./11-多智能体/multi-agent-ko.md) | Agent/Teammate/Remote/Dream |
| [12 UI 렌더링](./12-UI渲染/ui-rendering-ko.md) | Ink 엔진, 컴포넌트 트리, 디자인 시스템 |
| [13 설정 시스템](./13-配置体系/config-system-ko.md) | 5단계 우선순위, 핫 리로드, MDM |
| [14 상태 관리](./14-状态管理/state-management-ko.md) | Bootstrap 싱글톤 + Zustand Store |
| [15 커맨드 시스템](./15-命令体系/command-system-ko.md) | 87개 이상의 슬래시 커맨드 전체 목록 |
| [16 메모리 시스템](./16-记忆系统/memory-system-ko.md) | memdir, 자동 추출, 팀 메모리 |
| [17 오류 복구](./17-错误恢复/error-recovery-ko.md) | 5계층 복구, 보존 전략, 성능 저하 |
| [18 텔레메트리 및 분석](./18-遥测分析/telemetry-system-ko.md) | OTel, Datadog, GrowthBook, Perfetto |
| [19 피드백 및 설문조사](./19-反馈与调查/feedback-system-ko.md) | 설문 상태 머신, 트랜스크립트 공유, 확률 게이팅 |
| [20 서비스 계층](./20-服务层/services-complete-ko.md) | 13개 백그라운드 서비스 완전 분석 |
| [21 플러그인 시스템](./21-插件系统/plugin-system-ko.md) | 검색, 설치, 마켓플레이스, 정책 |
| [22 OAuth 및 인증](./22-OAuth与认证/oauth-auth-ko.md) | PKCE, 키체인, 토큰 갱신 |
| [23 LSP 통합](./23-LSP集成/lsp-integration-ko.md) | JSON-RPC, 멀티 인스턴스, 익스텐션 라우팅 |
| [24 샌드박스 시스템](./24-沙箱系统/sandbox-system-ko.md) | 설정, 실행, 위반 감지 |
| [25 Git 및 GitHub](./25-Git与GitHub/git-github-ko.md) | 파일시스템 파싱, gitignore, gh CLI |
| [26 세션 관리](./26-会话管理/session-management-ko.md) | 히스토리, 복구, 내보내기, 공유 |
| [27 키바인딩 및 입력](./27-键绑定与输入/keybinding-system-ko.md) | 50개 이상의 액션, 코드, 컨텍스트 매칭 |
| [28 Vim 모드](./28-Vim模式/vim-mode-ko.md) | 완전한 상태 머신, 모션/오퍼레이터/텍스트 객체 |
| [29 음성 시스템](./29-语音系统/voice-system-ko.md) | 게이팅, 인증, 통합 |
| [30 원격 세션](./30-远程会话/remote-session-ko.md) | CCR WebSocket, 권한 브리징 |
| [31 브리지 프로토콜](./31-Bridge协议/bridge-protocol-ko.md) | 33개 파일, REST+WS, 신뢰성 있는 전송 |
| [32 버디 시스템](./32-Buddy系统/buddy-system-ko.md) | 반려 펫, PRNG, 스프라이트 렌더링 |
| [33 코디네이터 패턴](./33-协调器模式/coordinator-mode-ko.md) | 멀티 워커 오케스트레이션, 작업 알림 |
| [34 Swarm 시스템](./34-Swarm系统/swarm-architecture-ko.md) | tmux/iTerm2/인프로세스 백엔드, 권한 동기화 |
| [35 컴퓨터 사용](./35-Computer-Use/computer-use-ko.md) | macOS Enigo/Swift, 잠금, ESC 핫키 |
| [36 딥링크](./36-DeepLink/deeplink-system-ko.md) | 프로토콜 등록, 터미널 실행, URL 파싱 |
| [37 텔레포트](./37-Teleport/teleport-system-ko.md) | CCR 세션 API, Git Bundle, 환경 |
| [38 출력 스타일](./38-输出样式/output-styles-ko.md) | 마크다운 프론트매터, 스타일 로딩 |
| [39 네이티브 모듈](./39-原生模块/native-modules-ko.md) | 색상 차이, 파일 인덱싱, Yoga 레이아웃 |
| [40 마이그레이션 시스템](./40-迁移系统/migration-system-ko.md) | 11개 설정 마이그레이션 |
| [41 파일 지속성](./41-文件持久化/file-persistence-ko.md) | BYOC 파일 업로드, mtime 스캔 |
| [42 비용 추적](./42-代价追踪/cost-tracking-ko.md) | 모델 사용량, 세션 비용, 포맷팅 |
| [43 Shell 툴체인](./43-Shell工具链/shell-toolchain-ko.md) | Bash AST, PowerShell 파싱, 스펙 |
| [44 화면 컴포넌트](./44-Screens组件/screens-components-ko.md) | REPL, Doctor, Resume |
| [45 타입 시스템](./45-类型系统/type-system-ko.md) | 메시지, 권한, 커맨드, 훅(Hooks) 타입 |
| [46 완전한 데이터 흐름](./46-完整数据流图/complete-data-flow-ko.md) | 엔드투엔드 흐름, 호출 그래프, 시퀀스 다이어그램 |

## 권장 읽기 순서

1. **빠른 개요**: 01-시스템 총람 → 46-완전한 데이터 흐름
2. **핵심 루프**: 02-시작 및 초기화 → 03-쿼리 엔진(Query Engine) → 04-API 클라이언트 → 05-도구 시스템(Tool System)
3. **보안 모델**: 06-권한 및 보안 → 24-샌드박스(Sandbox) 시스템 → 34-Swarm 시스템 (권한 동기화)
4. **컨텍스트 전략**: 07-컨텍스트 관리 → 16-메모리 시스템(Memory System) → 20-서비스 계층 (5개 백그라운드 추출 서비스)
5. **확장성**: 08-MCP(Model Context Protocol) 통합 → 10-스킬(Skills) 시스템 → 21-플러그인(Plugin) 시스템 → 09-훅(Hooks) 시스템
6. **사용자 경험**: 12-UI 렌더링 → 27-키바인딩 및 입력 → 19-피드백 및 설문조사 → 42-비용 추적
7. **멀티 에이전트(Multi-Agent)**: 11-멀티 에이전트 → 33-코디네이터(Coordinator) 모드 → 34-Swarm 시스템 → 37-텔레포트
8. **원격 기능**: 30-원격 세션 → 31-브리지 프로토콜 → 36-딥링크 → 41-파일 지속성

## 규모 통계

| 항목 | 수량 |
|-----------|-------|
| TypeScript 소스 파일 | 1,884 |
| 최상위 디렉터리 | 35 |
| 서비스 모듈 | 13 |
| 내장 도구 | 40+ |
| React 훅(Hooks) | 70+ |
| 슬래시 커맨드 | 87+ |
| 내장 스킬(Skills) | 17 |
| 이벤트 훅(Hooks) 타입 | 13 |
| 권한 모드(Permission Modes) | 6 |
| API 백엔드 | 4 (Anthropic/Bedrock/Vertex/Foundry) |
| MCP(Model Context Protocol) 전송 프로토콜 | 4 (stdio/SSE/HTTP/WebSocket) |
| 분석 이벤트 | 50+ |
| 설정 마이그레이션 | 11 |
| 번들 총 라인 | 16,667줄 / 13MB |

---

[읽기 시작 →](./01-系统总览/system-overview-ko.md)
