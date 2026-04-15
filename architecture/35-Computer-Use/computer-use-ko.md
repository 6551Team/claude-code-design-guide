# 컴퓨터 사용(Computer Use) 통합 아키텍처

컴퓨터 사용(Computer Use) 기능은 MCP(Model Context Protocol) 서버로 Claude Code에 통합되어 화면 상호작용, 마우스/키보드 제어 및 기타 컴퓨터 작업 기능을 제공합니다.

---

## 아키텍처 (utils/computerUse/의 15개 파일)

### 핵심 상수

```typescript
const COMPUTER_USE_MCP_SERVER_NAME = 'computer-use'
const CLI_HOST_BUNDLE_ID = 'com.anthropic.claude-code.cli-no-window'
```

### 터미널 Bundle ID 매핑

`TERMINAL_BUNDLE_ID_FALLBACK`은 일반적인 터미널 이름을 해당 macOS Bundle ID에 매핑합니다:

| 터미널 이름 | Bundle ID |
|-----------|-----------|
| iTerm | com.googlecode.iterm2 |
| Terminal | com.apple.Terminal |
| Ghostty | com.mitchellh.ghostty |
| Kitty | net.kovidgoyal.kitty |
| Warp | dev.warp.Warp |
| VSCode | com.microsoft.VSCode |

### 설계 철학

#### 왜 macOS에서 AppleScript 대신 Enigo/Swift인가?

소스 코드 주석은 기술 아키텍처를 공개합니다: "@ant/computer-use-input (Rust/enigo) -- 마우스, 키보드, 최전방 앱" + "@ant/computer-use-swift -- SCContentFilter 스크린샷, NSWorkspace 앱, TCC". AppleScript는 작업당 100ms 이상의 지연이 있고 저수준 이벤트 제어가 제한적입니다; Enigo는 시스템 API를 직접 호출(`DispatchQueue.main`을 통해)하며 지연이 10ms 수준입니다. 네이티브 Swift 모듈은 AppleScript로는 달성할 수 없는 화면 캡처 기능(SCContentFilter)과 TCC 권한 감지를 제공합니다.

#### 왜 O_EXCL 원자적 잠금인가?

소스 코드 주석에는 이렇게 명시되어 있습니다: "원자적 테스트-및-설정을 위해 O_EXCL (open 'wx')을 사용합니다 -- OS는 최대 하나의 프로세스만 성공하도록 보장합니다". 여러 에이전트(스웜(Swarm)의 팀원 등)가 동시에 마우스/키보드를 제어하려 할 수 있습니다 — 원자적 파일 잠금은 한 번에 하나의 세션만 작동할 수 있도록 보장하여 여러 에이전트가 동시에 마우스를 움직이는 혼란을 방지합니다. 잠금 파일에는 `sessionId`, `pid`, `acquiredAt`이 포함되어 PID 기반 오래된 잠금 감지와 60분 강제 회수 타임아웃을 지원합니다.

#### 왜 ESC 단축키에 CGEventTap인가?

소스 코드 주석은 보안 고려사항을 설명합니다: "전역 Escape -> 중단. Cowork의 escAbort.ts와 유사하지만 Electron 없이: CGEventTap을 @ant/computer-use-swift를 통해 사용. 등록된 동안 Escape는 시스템 전체에서 소비됩니다 (PI 방어 -- 프롬프트 주입 액션이 Escape로 대화 상자를 닫을 수 없음)". 사용자는 컴퓨터 사용(Computer Use) 작업을 긴급 중지해야 합니다; ESC를 전역 단축키로 사용하면 Claude Code 창에 포커스가 없어도 발생합니다. 더 중요하게는 보안 방어 역할을 합니다 — 시스템 수준의 Escape 가로채기는 프롬프트 주입 공격이 Escape 키를 사용하여 보안 대화 상자를 닫는 것을 방지합니다. `notifyExpectedEscape()`는 100ms 감소 창을 사용하여 모델 자체의 Escape 작업과 사용자 인터럽트 의도를 구분합니다.

#### 왜 CFRunLoop 펌프인가?

소스 코드 주석은 근본 원인을 직접 설명합니다: "Swift의 @MainActor 비동기 메서드와 @ant/computer-use-input의 key()/keys()는 모두 DispatchQueue.main으로 디스패치됩니다. libuv (Node/bun)에서 해당 큐는 절대 비워지지 않습니다 -- 프로미스가 중단됩니다. Electron은 CFRunLoop를 통해 비워지므로 Cowork는 이것이 필요하지 않습니다". macOS의 메인 스레드 디스패치 큐는 Node.js 환경에서 자동으로 비워지지 않습니다; `_drainMainRunLoop`을 통해 수동으로 펌프해야 합니다 (1ms마다 호출). 참조 카운팅 메커니즘(`retainPump`/`releasePump`)은 활성 컴퓨터 사용(Computer Use) 작업이 있을 때만 RunLoop가 실행되도록 보장하여 유휴 CPU 낭비를 방지합니다.

---

## 기능 게이트

### 메인 스위치

**getChicagoEnabled()**는 컴퓨터 사용(Computer Use) 기능의 사용 가능 여부를 제어합니다:

- max 또는 pro+ 구독 등급이 필요합니다
- Anthropic 내부 사용자(ants)는 제한을 우회할 수 있습니다

### 서브 기능 스위치

- **pixelValidation**: 픽셀 수준 좌표 검증
- **clipboardPasteMultiline**: 다중 줄 텍스트 클립보드 붙여넣기
- **mouseAnimation**: 마우스 이동 애니메이션 효과
- **hideBeforeAction**: 액션 실행 전 Claude Code 창 숨기기
- **autoTargetDisplay**: 대상 디스플레이 자동 선택
- **clipboardGuard**: 클립보드 내용 보호 (사용자의 클립보드를 실수로 덮어쓰는 것 방지)

### 좌표 모드

**getChicagoCoordinateMode()**는 좌표 모드를 반환합니다:

```typescript
type CoordinateMode = 'pixels' | 'normalized'
```

- **pixels**: 절대 픽셀 좌표 사용
- **normalized**: 0-1 정규화된 좌표 사용
- 첫 번째 읽기 후 고정됩니다; 런타임 중에 변경할 수 없습니다

---

## 실행자(Executor) (executor.ts, 658줄)

### 팩토리 함수

**createCliExecutor()**는 CLI 환경용 실행자를 생성하며 다음을 래핑합니다:

- **@ant/computer-use-input**: Rust/enigo로 구현된 크로스 플랫폼 입력 제어
- **@ant/computer-use-swift**: 시스템 상호작용을 위한 macOS 네이티브 Swift 구현

### 메서드 세트

#### 화면 작업
- **screenshot()**: 화면 스냅샷 캡처
- **zoom(factor)**: 디스플레이 확대/축소

#### 키보드 작업
- **key(keys)**: 키 조합 (예: Ctrl+C)
- **holdKey(key, duration)**: 키를 지정된 시간 동안 누름
- **type(text)**: 텍스트 문자열 입력

#### 클립보드
- **readClipboard()**: 클립보드 내용 읽기
- **writeClipboard(text)**: 클립보드에 내용 쓰기

#### 마우스 작업
- **moveMouse(x, y)**: 마우스를 지정된 위치로 이동
- **click(x, y, button)**: 클릭
- **mouseDown(x, y, button)**: 마우스 버튼 누르기
- **mouseUp(x, y, button)**: 마우스 버튼 놓기
- **getCursorPosition()**: 현재 커서 위치 가져오기
- **drag(fromX, fromY, toX, toY)**: 드래그 작업
- **scroll(x, y, deltaX, deltaY)**: 스크롤 작업

#### 애플리케이션 관리
- **getFrontmostApp()**: 현재 최전방 애플리케이션 정보 가져오기
- **listInstalledApps()**: 설치된 애플리케이션 목록
- **getAppIcon(bundleId)**: 애플리케이션 아이콘 가져오기
- **listRunningApps()**: 실행 중인 애플리케이션 목록
- **openApp(bundleId)**: 지정된 애플리케이션 열기

#### 준비 작업
- **prepareForAction()**: 액션 실행 전 준비 작업 (예: 창 숨기기)

### 애니메이션 이동

**animatedMove()**는 부드러운 마우스 이동을 구현합니다:

- **ease-out-cubic** 이징 곡선 사용
- 이동 속도: **2000 px/초**
- 자연스러운 마우스 이동 시각 효과 제공

### CLI 전용 처리

- **클릭 통과 없음**: CLI 모드는 클릭 통과를 지원하지 않습니다
- **터미널 대리 호스트**: 터미널을 대리 호스트 애플리케이션으로 사용합니다
- **클립보드**: `pbcopy`/`pbpaste` 명령어를 통해 클립보드를 조작합니다

---

## 잠금 시스템 (computerUseLock.ts)

### 원자적 잠금 구현

`O_EXCL` 플래그를 사용하여 원자적 파일 생성 잠금을 구현하고, 한 번에 하나의 컴퓨터 사용(Computer Use) 세션만 활성화되도록 보장합니다:

```typescript
const HOLDER_STALE_MS = 60 * 60 * 1000  // 60분
```

### 잠금 파일

경로: `~/.claude/computer-use.lock`

```json
{
  "sessionId": "session-uuid",
  "pid": 12345,
  "acquiredAt": "2025-01-01T00:00:00.000Z"
}
```

### 오래된 잠금 복구

- PID 기반 오래된 잠금 감지
- 잠금을 보유한 프로세스가 종료되면 잠금이 자동으로 회수됩니다
- `HOLDER_STALE_MS = 60분` 타임아웃 후 강제 회수

### 제로 시스콜 확인

**isLockHeldLocally()**: 인메모리 상태를 통해 잠금 보유 상태를 확인하여 시스템 호출이 필요 없어 매우 성능이 좋습니다.

---

## ESC 단축키 (escHotkey.ts)

### CGEventTap 등록

사용자의 컴퓨터 사용(Computer Use) 작업 인터럽트를 위한 시스템 수준 Escape 키 이벤트 리스너를 등록합니다:

- macOS CGEventTap API 사용
- 전역 Escape 키 이벤트 캡처

### 예상되는 Escape 처리

**notifyExpectedEscape()**: 모델 자체가 Escape 키 입력을 합성해야 할 때 호출됩니다:

- **100ms** 감소 창 생성
- 이 창 내의 Escape 이벤트는 사용자 인터럽트가 아닌 모델 액션으로 처리됩니다
- 창이 만료된 후 일반적인 인터럽트 감지가 재개됩니다

---

## CFRunLoop (drainRunLoop.ts)

### 참조 카운팅 펌프

CFRunLoop를 실행 상태로 유지하는 참조 카운팅 setInterval 펌프를 사용합니다:

- 펌프 간격: **1ms**
- 타임아웃 보호: **30초** 최대 런타임

### 생명주기 관리

- **retainPump()**: 참조 카운트를 증가시킵니다; 첫 번째 호출 시 펌프를 시작합니다
- **releasePump()**: 참조 카운트를 감소시킵니다; 카운트가 0에 도달하면 펌프를 중지합니다
- 활성 컴퓨터 사용(Computer Use) 작업이 있을 때만 RunLoop가 실행되도록 보장합니다

---

## 앱 필터링 (appNames.ts)

### 필터링 로직

**filterAppsForDescription()**은 애플리케이션 목록을 필터링하여 노이즈 애플리케이션을 제거합니다:

- Helper/Agent/Service/Updater 같은 키워드를 포함하는 백그라운드 애플리케이션을 차단합니다
- 사용자에게 보이는 포그라운드 애플리케이션만 유지합니다

### 허용 목록

**ALWAYS_KEEP_BUNDLE_IDS**: 항상 유지되는 약 30개의 핵심 애플리케이션:

- 브라우저: Chrome, Safari, Firefox, Arc, Edge
- 커뮤니케이션: Slack, Discord, Zoom, Teams
- 개발: VSCode, Xcode, Terminal, iTerm2
- 생산성: Finder, Notes, Calendar, Mail
- 기타 일반적인 애플리케이션

### 이름 검증

**APP_NAME_ALLOWED**: 애플리케이션 이름 검증 규칙:

- 유니코드 인식 정규 표현식
- 최대 **40자** 길이 제한
- 호출당 최대 **50개** 애플리케이션 반환

---

## 정리

### cleanupComputerUseAfterTurn()

각 대화 턴 종료 시 실행되는 정리 프로세스:

1. **자동 숨김 해제**: 이전에 숨겨진 창을 복원하며, 중단을 방지하는 **5초 타임아웃**
2. **ESC 리스너 등록 해제**: CGEventTap Escape 키 리스너를 제거합니다
3. **잠금 해제**: computer-use.lock 파일 잠금을 해제합니다
4. CFRunLoop 펌프 참조를 해제합니다

---

## MCP(Model Context Protocol) 서버

### 서버 생성

**createComputerUseMcpServerForCli()**는 CLI 환경용 MCP 서버를 구축합니다:

- 모든 도구 정의를 초기화합니다
- **ListTools 대체**: 표준 ListTools를 애플리케이션 설명 정보가 포함된 향상된 버전으로 대체합니다
- 모델이 현재 데스크톱 환경을 이해하는 데 도움이 되도록 도구 설명에 애플리케이션 컨텍스트를 주입합니다

### 서브프로세스 진입점

**runComputerUseMcpServer()**는 독립적인 서브프로세스로 실행되는 진입점입니다:

- **stdio 전송**을 사용하여 호스트 프로세스와 통신합니다
- 표준 MCP 서버 생명주기 관리
- 호스트 프로세스로부터 도구 호출 요청을 수신하고 실행합니다

---

## 엔지니어링 실천 가이드

### 컴퓨터 사용(Computer Use) 활성화

1. **플랫폼 확인**: 컴퓨터 사용(Computer Use)은 현재 **macOS만 지원**합니다 — Enigo (Rust)와 네이티브 Swift 모듈이 필요합니다
2. **권한 부여**:
   - 접근성 접근: 시스템 환경설정 → 개인 정보 보호 및 보안 → 손쉬운 사용 → Claude Code 추가
   - 화면 녹화 권한: SCContentFilter 스크린샷에는 화면 녹화 인증이 필요합니다
   - TCC 권한: 네이티브 Swift 모듈은 TCC 프레임워크를 통해 권한 상태를 감지합니다
3. **구독 등급**: max 또는 pro+ 구독이 필요합니다 — Anthropic 내부 사용자는 이를 우회할 수 있습니다
4. **MCP 서버 확인**: `createComputerUseMcpServerForCli()`는 컴퓨터 사용(Computer Use) MCP 서버를 초기화하며, stdio 전송을 사용하여 호스트 프로세스와 통신합니다

### 잠금 충돌 디버깅

1. **잠금 파일 검사**: `~/.claude/computer-use.lock`의 내용을 확인합니다 — `sessionId`, `pid`, `acquiredAt`을 포함합니다
2. **잠금을 보유한 프로세스가 살아있는지 확인**: 잠금 파일의 `pid`를 사용하여 프로세스 상태를 확인합니다 — 프로세스가 종료된 경우 오래된 잠금입니다
3. **오래된 잠금 수동 정리**: 프로세스 크래시 후 잠금 파일이 남아있을 수 있습니다 — `~/.claude/computer-use.lock`을 안전하게 삭제하면 잠금이 해제됩니다
4. **타임아웃 시 강제 회수**: `HOLDER_STALE_MS = 60분`보다 오래 보유된 잠금은 자동으로 회수됩니다 — 하지만 한 시간 이내의 오래된 잠금은 수동 정리가 필요합니다
5. **제로 시스콜 확인**: `isLockHeldLocally()`는 인메모리 상태를 통해 잠금을 확인하며 시스템 호출이 없습니다 — 고빈도 확인 시나리오에 적합합니다

### ESC 단축키가 작동하지 않는 경우

1. **CGEventTap 권한 확인**: ESC 단축키는 macOS CGEventTap API를 통해 시스템 수준 이벤트 리스너를 등록합니다 — 접근성 권한이 필요합니다
2. **CFRunLoop 실행 중인지 확인**: Swift의 `@MainActor`와 Enigo 작업 모두 `DispatchQueue.main`으로 디스패치되며, `_drainMainRunLoop`이 1ms마다 펌프해야 합니다. `retainPump()`가 호출되었고 참조 카운트가 > 0인지 확인합니다
3. **예상되는 Escape 창 확인**: `notifyExpectedEscape()`는 100ms 감소 창을 생성합니다 — 모델이 방금 Escape 키 작업을 전송했다면, 100ms 이내의 사용자 Escape는 인터럽트를 트리거하지 않습니다. 100ms 기다린 후 다시 시도합니다
4. **다른 애플리케이션과의 충돌 확인**: Escape는 전역 단축키입니다; 다른 애플리케이션도 Escape 리스너를 등록했다면 서로 간섭할 수 있습니다

### 턴별 정리 체크리스트

각 대화 턴 후 `cleanupComputerUseAfterTurn()`은 다음 정리를 수행합니다:

- [ ] 창 자동 숨김 해제 (5초 타임아웃 보호)
- [ ] CGEventTap Escape 키 리스너 등록 해제
- [ ] `computer-use.lock` 파일 잠금 해제
- [ ] CFRunLoop 펌프 참조 카운트 해제

정리가 완료되지 않은 경우 (예: SIGKILL로 프로세스가 종료된 경우) 수동으로 확인합니다: 잠금 파일이 여전히 있는가? Escape 리스너가 여전히 활성 상태인가? 숨겨진 창이 복원되었는가?

### 흔한 함정

> **여러 에이전트가 동시에 작동하면 잠금 충돌이 발생합니다**: O_EXCL 원자적 잠금은 한 번에 하나의 세션만 마우스/키보드를 제어할 수 있도록 보장합니다. 스웜(Swarm) 시나리오에서 여러 팀원이 동시에 컴퓨터 사용(Computer Use)을 사용할 수 없습니다 — 작업은 권한 동기화 메커니즘을 통해 직렬화되어야 합니다.

> **ESC는 전역 단축키입니다**: 등록되면 Escape가 시스템 수준에서 가로채여 **모든 애플리케이션에서 소비됩니다** — Vim의 Escape, 대화 상자 취소 버튼 등을 포함합니다. 이것은 프롬프트 주입 방어를 위한 의도적인 설계 선택이지만, 정상적인 사용자 작업에 영향을 미칠 수 있습니다. 컴퓨터 사용(Computer Use)이 활성 상태가 아닐 때는 리스너가 등록 해제되었는지 확인하세요.

> **macOS 전용**: Windows와 Linux는 컴퓨터 사용(Computer Use)을 지원하지 않습니다 — 네이티브 모듈(`@ant/computer-use-input`과 `@ant/computer-use-swift`)은 macOS 전용 API(CGEvent, SCContentFilter, NSWorkspace, TCC)에 의존합니다.

> **CFRunLoop 펌프 CPU 오버헤드**: `_drainMainRunLoop`은 1ms마다 호출됩니다; 컴퓨터 사용(Computer Use)이 활성 상태가 아닐 때는 `releasePump()`를 통해 중지하세요. 30초 타임아웃 보호가 자동으로 중지하지만, 30초 동안 유휴 상태에서 실행되는 것은 여전히 CPU를 낭비합니다.


---

[← 스웜 시스템](../34-Swarm系统/swarm-architecture-ko.md) | [목차](../README_KO.md) | [DeepLink →](../36-DeepLink/deeplink-system-ko.md)
