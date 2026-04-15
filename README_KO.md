# Claude Code 디자인 가이드

<div align="center">

**[English](./README_EN.md) | [中文](./README.md) | 한국어**

</div>

> 초기 인터넷 디자인 패턴부터 AI 에이전트 실전까지 — 개발자를 위한 Claude Code 심층 분석
>
> 소스 코드를 유출한 프로그래머와 AI로 이 책을 완성한 제안자에게 감사드립니다
>
> https://x.com/cryptoxiao
>
> https://x.com/BoxMrChen
>
> https://x.com/0xfaskety

---

## 이 책은 무엇인가

Claude Code는 Anthropic이 공식 출시한 AI 프로그래밍 어시스턴트 CLI 도구입니다. 단순한 "코드를 작성하는 챗봇"이 아니라, 도구 호출(Tool Call), 컨텍스트 엔지니어링(Context Engineering), 멀티 에이전트(Multi-Agent) 협업, 권한 관리, 확장 시스템 등 현대 엔지니어링 방법론을 집대성한 완전한 **에이전트 런타임 시스템(Agent Runtime System)** 입니다.

이 책은 Claude Code의 소스 코드 설계를 심층 분석하여 다음을 이해하도록 돕습니다:

- AI 에이전트(Agent) 시스템을 처음부터 어떻게 구축하는가
- 현대 CLI 도구의 엔지니어링 철학
- 컨텍스트 엔지니어링(Context Engineering)의 핵심 사상
- 도구 시스템, 권한 모델, 확장 메커니즘의 디자인 패턴

---

## 대상 독자

| 독자 유형 | 이 책에서 얻을 수 있는 것 |
|----------|------------------------|
| **입문자** | Claude Code가 무엇인지, 무엇을 할 수 있는지, 어떻게 사용하는지 이해 |
| **고급 개발자** | 현대 CLI 도구의 엔지니어링 방법과 TypeScript 대규모 프로젝트 아키텍처 학습 |
| **에이전트 시스템 설계자** | 에이전트 런타임(Agent Runtime), 도구 시스템(Tooling), 컨텍스트 엔지니어링(Context Engineering), 확장 시스템의 디자인 패턴 심층 이해 |

---

## 목차

### 서문
- [서문: 왜 이 책을 읽어야 하는가](./00-preface_ko.md)

### 제1부: Claude Code 이해하기 (입문자 친화)
- [제1장: Claude Code란 무엇인가](./part1/01-introduction_ko.md)
- [제2장: 빠른 시작](./part1/02-quickstart_ko.md)

### 제2부: 초기 인터넷 설계에서 AI 에이전트까지
- [제3장: 유닉스 철학과 CLI의 전통](./part2/03-unix-philosophy_ko.md)
- [제4장: REPL의 진화사](./part2/04-repl-evolution_ko.md)
- [제5장: 챗봇에서 에이전트로](./part2/05-from-chatbot-to-agent_ko.md)

### 제3부: 아키텍처 설계
- [제6장: 쿼리 엔진 — 대화의 심장](./part3/06-query-engine_ko.md)
- [제7장: 상태 관리 설계](./part3/07-state-management_ko.md)
- [제8장: 메시지 루프와 스트리밍](./part3/08-message-loop_ko.md)

### 제4부: 도구 시스템 설계
- [제9장: 도구 시스템의 설계 철학](./part4/09-tool-design_ko.md)
- [제10장: 43개 내장 도구 전체 개요](./part4/10-builtin-tools_ko.md)
- [제11장: 도구 권한 모델](./part4/11-tool-permission_ko.md)

### 제5부: 컨텍스트 엔지니어링(Context Engineering)
- [제12장: 컨텍스트 엔지니어링이란 무엇인가](./part5/12-context-what_ko.md)
- [제13장: 시스템 프롬프트 구축의 기술](./part5/13-system-prompt_ko.md)
- [제14장: 메모리와 CLAUDE.md](./part5/14-memory-claudemd_ko.md)
- [제15장: 컨텍스트 압축 (Auto-Compact)](./part5/15-compact_ko.md)

### 제6부: 에이전트 런타임과 멀티 에이전트
- [제16장: 태스크 시스템 설계](./part6/16-task-system_ko.md)
- [제17장: 멀티 에이전트 아키텍처](./part6/17-multi-agent_ko.md)
- [제18장: 코디네이터 패턴](./part6/18-coordinator_ko.md)

### 제7부: 확장 시스템
- [제19장: MCP 프로토콜 — 도구의 인터넷](./part7/19-mcp_ko.md)
- [제20장: 스킬 시스템](./part7/20-skills_ko.md)
- [제21장: 플러그인 시스템](./part7/21-plugins_ko.md)

### 제8부: 보안, 권한, 성능
- [제22장: 계층형 권한 모델 설계](./part8/22-permission-model_ko.md)
- [제23장: 보안 설계](./part8/23-security_ko.md)
- [제24장: 성능 최적화](./part8/24-performance_ko.md)

### 제9부: 설계 철학
- [제25장: Claude Code의 설계 원칙](./part9/25-design-principles_ko.md)
- [제26장: 미래 전망](./part9/26-future_ko.md)

---

## 이 책을 읽는 방법

- **입문자라면**: 제1부부터 순서대로 읽으세요
- **개발자라면**: 제1부를 건너뛰고 제2부부터 시작할 수 있습니다
- **에이전트 시스템 설계자라면**: 제3, 4, 5, 6, 7부를 중점적으로 읽으세요

---

## 소스 코드에 대하여

이 책의 분석은 Claude Code의 유출된 완전한 TypeScript 소스 코드(2026년 3월)를 기반으로 합니다. 모든 코드 인용은 실제 소스 코드에서 가져온 것이며, 추측은 포함하지 않습니다.

---

## 심화 학습

Claude Code의 소스 코드 구현 세부사항을 더 깊이 알고 싶다면, 심화 문서를 참고하세요:

**[📚 Claude Code 소스 코드 아키텍처 분석](./architecture/README_KO.md)** | **[English](./architecture/README_EN.md)** | **[中文](./architecture/README.md)**

심화 문서에는 다음 내용이 포함되어 있습니다:
- 완전한 소스 트리 구조 (1,884개 TypeScript 파일)
- 6계층 아키텍처 설계 상세 해설
- 쿼리 엔진(Query Engine), 도구 시스템(Tool System), 권한 모델(Permission Model)의 구현 세부사항
- 40개 이상의 도구, 70개 이상의 훅(Hooks), 87개 이상의 명령어 소스 코드 분석
- 완전한 모듈 의존성 그래프 및 데이터 흐름 다이어그램

---

*이 책은 오픈 소스입니다. 기여와 오류 수정을 환영합니다.*
