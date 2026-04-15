# 딥링크(DeepLink) 시스템 아키텍처

## 프로토콜

딥링크(DeepLink) 시스템은 외부 애플리케이션(브라우저, 편집기 등)이 커스텀 프로토콜을 통해 Claude CLI를 직접 실행하고 컨텍스트를 전달할 수 있도록 합니다.

### 설계 철학

#### 왜 claude-cli:// 프로토콜인가?

이를 통해 외부 애플리케이션(브라우저, IDE, 문서)이 Claude Code를 직접 실행하고 컨텍스트를 전달할 수 있습니다 — 원활한 통합이 가능합니다. 커맨드라인 인수와 달리 URL은 웹 페이지, 문서, Slack 메시지에 삽입하여 직접 클릭할 수 있으므로, 사용자가 터미널을 열고 직접 명령을 입력할 필요가 없습니다. 소스 파일 `parseDeepLink.ts`는 세 가지 파라미터를 지원합니다: `q`(쿼리), `cwd`(작업 디렉터리), `repo`(저장소). 이는 "이 저장소에서 이 질문을 묻겠다"는 완전한 컨텍스트 전달 요구를 충족합니다.

#### 왜 3개 플랫폼에 등록하는가?

macOS(`Info.plist` + `CFBundleURLTypes` + `lsregister`), Linux(`.desktop` 파일 + `xdg-mime`), Windows(레지스트리 `HKEY_CURRENT_USER\Software\Classes`)는 각기 다른 프로토콜 등록 메커니즘을 사용합니다. 소스 파일 `registerProtocol.ts`(349줄)는 각 플랫폼에 대한 전용 등록 로직을 구현합니다. 이는 추상화할 수 없는 차이점입니다 — 각 플랫폼의 URL 스킴 등록은 OS 수준 API이므로 개별적으로 적응해야 합니다. 자동 등록은 `backgroundHousekeeping` 작업에 의해 트리거되며, 실패 후 24시간 백오프를 적용하여 사용자 워크플로를 방해하지 않습니다.

#### 왜 커맨드라인 인수 대신 URL 파싱인가?

소스 코드의 보안 설계는 URL 방식에 대한 신중한 고려를 보여줍니다: 제어 문자 거부로 터미널 인젝션을 방지하고, `cwd`는 경로 탐색을 방지하기 위해 절대 경로를 요구하며, `repo` 슬러그 형식 검증(`owner/repo`)으로 인젝션 표면을 제한합니다. URL 형식은 다중 전략 셸 인용도 지원합니다 — `shellQuote()`(POSIX), `appleScriptQuote()`(macOS osascript), `psQuote()`(PowerShell), `cmdQuote()`(cmd.exe) — 모든 터미널 환경에서 인수를 안전하게 전달할 수 있습니다.

```
DEEP_LINK_PROTOCOL = 'claude-cli'
```

**URI 형식**:
```
claude-cli://open?q=...&cwd=...&repo=...
```

- `q`: 쿼리/프롬프트 내용, 초기 메시지로 Claude에 전달됨
- `cwd`: 작업 디렉터리, Claude 세션의 작업 경로를 지정함
- `repo`: 저장소 식별자(슬러그 형식), 컨텍스트 위치에 사용됨

**보안 조치**:
- **제어 문자 거부**: URI에 제어 문자가 포함된 요청을 거부하여 터미널 인젝션 공격을 방지
- **cwd 절대 경로 요구**: `cwd` 파라미터는 반드시 절대 경로여야 하며, 상대 경로는 경로 탐색 방지를 위해 거부됨
- **repo 슬러그 검증**: repo 파라미터가 슬러그 형식(`owner/repo`)을 준수하는지 검증하며, 불법 문자를 거부함

**길이 제한**:
```typescript
MAX_QUERY_LENGTH = 5000    // 쿼리 내용의 최대 길이
MAX_CWD_LENGTH = 4096      // 작업 디렉터리 경로의 최대 길이
```

## 등록

`registerProtocol.ts`(약 349줄)는 각 운영 체제에 `claude-cli://` 프로토콜 핸들러를 등록하는 역할을 합니다.

### macOS

`~/Applications` 디렉터리에 `.app` 번들을 생성합니다:
- `claude-cli` 스킴을 애플리케이션에 바인딩하는 `CFBundleURLTypes` 선언이 포함된 `Info.plist`를 생성
- CLI 실행 파일을 가리키는 심볼릭 링크를 생성
- Launch Services에 URL 스킴을 등록하기 위해 `lsregister`를 호출

### Linux

XDG 데스크톱 사양을 사용합니다:
- `$XDG_DATA_HOME/applications` 디렉터리(기본값: `~/.local/share/applications`)에 `.desktop` 파일을 생성
- `xdg-mime`을 통해 `x-scheme-handler/claude-cli`의 기본 핸들러를 설정

### Windows

레지스트리 키를 작성합니다:
- 경로: `HKEY_CURRENT_USER\Software\Classes\claude-cli`
- 이것이 프로토콜 핸들러임을 나타내기 위해 `URL Protocol` 값을 설정
- `shell\open\command` 하위 키 아래에 CLI 실행 파일 경로를 설정

### 자동 등록

프로토콜 등록은 `backgroundHousekeeping` 작업에 의해 자동으로 트리거됩니다:
- **실패 백오프**: `FAILURE_BACKOFF_MS`는 24시간 백오프 기간을 설정하며, 등록 실패 후 자주 재시도하지 않음
- 사용자 워크플로를 방해하지 않고 백그라운드에서 자동으로 실행됨

## 터미널 실행

`terminalLauncher.ts`(약 558줄)는 사용자가 선호하는 터미널에서 새 Claude CLI 세션을 실행하는 역할을 합니다.

### macOS 터미널 지원 (우선 순위 순)

1. iTerm2
2. Ghostty
3. Kitty
4. Alacritty
5. WezTerm
6. Terminal.app (시스템 기본값)

### Linux 터미널 지원

ghostty, kitty, alacritty, wezterm, gnome-terminal, konsole, xfce4-terminal, mate-terminal, tilix, xterm

### Windows 터미널 지원

Windows Terminal, pwsh (PowerShell 7+), PowerShell (Windows PowerShell), cmd

### 셸 인용

터미널과 셸마다 서로 다른 인용 및 이스케이프 전략이 필요합니다:

- **shellQuote()**: 일반 POSIX 셸 인용, 단일 따옴표로 감싸고 내부 단일 따옴표를 이스케이프
- **appleScriptQuote()**: AppleScript 문자열 인용, macOS에서 `osascript`를 통해 터미널을 제어하는 데 사용
- **psQuote()**: PowerShell 문자열 인용, PowerShell 특수 문자 및 이스케이프 시퀀스 처리
- **cmdQuote()**: Windows cmd.exe 인용, `%`, `^`, `&` 등의 특수 문자 처리

### 프로세스 분리

- **spawnDetached()**: 터미널을 독립적인 프로세스로 실행하여, 새로 실행된 Claude CLI 세션이 트리거 소스로부터 완전히 분리되도록 보장하며, 부모 프로세스 종료가 자식 프로세스에 영향을 미치지 않습니다.

## 배너

`banner.ts`는 딥링크(DeepLink)를 통해 실행될 때 배너 정보를 표시합니다:

```typescript
STALE_FETCH_WARN_MS = 7 days  // 페치가 7일 이상 발생하지 않았을 때 경고 표시
LONG_PREFILL_THRESHOLD = 1000  // 프리필 내용이 1000자를 초과할 때의 임계값
```

- **Git 상태 감지**: `.git/FETCH_HEAD`의 수정 시간(mtime)을 읽어 워크트리 시나리오를 지원합니다. 마지막 페치가 7일 이상 지났으면 저장소가 최신 상태가 아닐 수 있음을 사용자에게 알리는 경고가 배너에 표시됩니다.
- **긴 내용 힌트**: 딥링크(DeepLink)를 통해 전달된 쿼리 내용이 `LONG_PREFILL_THRESHOLD`(1000자)를 초과하면 내용 길이 힌트가 표시됩니다.

## 터미널 기본 설정

터미널 기본 설정 시스템(macOS 전용):

- **캡처**: `TERM_PROGRAM` 환경 변수를 읽고 해당 터미널 애플리케이션 식별자에 매핑
- **영속화**: 사용자의 터미널 기본 설정을 유지하여 이후 딥링크(DeepLink) 실행이 해당 터미널을 선호하도록 함
- **매핑**: `TERM_PROGRAM` 값(예: `iTerm.app`, `ghostty`, `Apple_Terminal`)을 내부 터미널 식별자에 매핑

---

## 엔지니어링 실천 가이드

### 프로토콜 핸들러 등록

플랫폼마다 다른 등록 방식이 필요합니다:

1. **macOS** (자동):
   - 시스템은 `~/Applications/` 아래에 `Info.plist`에 `CFBundleURLTypes` 선언이 포함된 `.app` 번들을 생성
   - Launch Services에 등록하기 위해 `lsregister`를 호출
   - 일반적으로 `backgroundHousekeeping`에 의해 자동으로 완료되며 수동 작업이 필요하지 않음
2. **Windows** (레지스트리 필요):
   - `HKEY_CURRENT_USER\Software\Classes\claude-cli` 레지스트리 키 작성
   - 프로토콜 핸들러로 표시하기 위해 `URL Protocol` 값을 설정
   - `shell\open\command` 하위 키 아래에 CLI 실행 파일 경로를 설정
3. **Linux** (데스크톱 파일 필요):
   - `$XDG_DATA_HOME/applications`(기본값: `~/.local/share/applications`)에 `.desktop` 파일을 생성
   - `xdg-mime`을 통해 `x-scheme-handler/claude-cli`의 기본 핸들러를 설정

### 열리지 않는 링크 디버깅

1. **프로토콜 등록이 올바른지 확인**:
   - macOS: 터미널에서 `open claude-cli://open?q=test`를 실행하여 Claude CLI가 실행되는지 확인
   - Windows: 브라우저 주소창에 `claude-cli://open?q=test`를 입력하여 프로토콜 핸들러가 트리거되는지 확인
   - Linux: `xdg-open claude-cli://open?q=test`를 실행하여 데스크톱 파일 구성을 확인
2. **URL 형식 확인**: 전체 형식은 `claude-cli://open?q=...&cwd=...&repo=...`
   - `q` 파라미터: 쿼리 내용, 최대 5000자(`MAX_QUERY_LENGTH`)
   - `cwd` 파라미터: 반드시 절대 경로, 최대 4096자(`MAX_CWD_LENGTH`)
   - `repo` 파라미터: 반드시 `owner/repo` 슬러그 형식을 준수해야 함
3. **보안 제한 확인**: 제어 문자가 포함된 URL은 거부됨; 상대 경로 `cwd`는 거부됨; 불법 문자가 포함된 `repo`는 거부됨
4. **등록 실패 백오프 확인**: 이전에 등록이 실패했다면 시스템은 24시간 백오프 기간(`FAILURE_BACKOFF_MS`)에 들어가며 — 이 기간 동안 등록이 재시도되지 않음

### 터미널 실행 문제 해결

딥링크(DeepLink)가 트리거되었지만 터미널이 올바르게 실행되지 않은 경우:

1. 현재 시스템에 `terminalLauncher.ts`가 지원하는 터미널 중 하나가 설치되어 있는지 확인
2. macOS 우선 순위: iTerm2 > Ghostty > Kitty > Alacritty > WezTerm > Terminal.app
3. Linux 우선 순위: ghostty > kitty > alacritty > wezterm > gnome-terminal > konsole > ...
4. Windows 우선 순위: Windows Terminal > pwsh > PowerShell > cmd
5. 셸 인용 확인: 터미널마다 다른 인용 전략을 사용함(`shellQuote`/`appleScriptQuote`/`psQuote`/`cmdQuote`)

### 흔한 함정

> **등록 메커니즘은 플랫폼마다 완전히 다릅니다**: macOS는 Info.plist + lsregister를 사용하고, Windows는 레지스트리를 사용하며, Linux는 .desktop + xdg-mime을 사용합니다. 이 세 가지 메커니즘은 서로 대체할 수 없으며 단일 추상화로 통합할 수 없습니다 — `registerProtocol.ts`의 349줄에서 각 플랫폼은 자체적인 독립 구현 경로를 가집니다. **크로스 플랫폼 릴리스 시 각 플랫폼에서 프로토콜 등록을 별도로 테스트해야 합니다.**

> **URL 파라미터는 올바르게 인코딩되어야 합니다**: 쿼리 내용(`q` 파라미터)의 특수 문자는 URL 인코딩되어야 합니다. 제어 문자는 보안 검사에 의해 즉시 거부됩니다(터미널 인젝션 방지). 딥링크(DeepLink) URL을 구성할 때 항상 `encodeURIComponent()`를 사용하여 파라미터 값을 처리하십시오.

> **백그라운드 등록은 자동으로 실패할 수 있습니다**: `backgroundHousekeeping` 작업은 백그라운드에서 자동으로 프로토콜 등록을 수행하며, 실패가 사용자에게 보고되지 않습니다. 딥링크(DeepLink)가 작동하지 않는 경우 먼저 등록이 성공했는지 확인하십시오 — 특히 Linux에서 `xdg-mime` 명령을 사용할 수 없으면 등록이 자동으로 실패합니다.

> **Git 저장소 오래됨 경고**: 배너 시스템은 `.git/FETCH_HEAD`의 수정 시간을 확인합니다; 마지막 페치가 7일 이상(`STALE_FETCH_WARN_MS`) 지났으면 경고가 표시됩니다. 이는 기능에 영향을 미치지 않지만 저장소가 최신 상태가 아닐 수 있음을 사용자에게 알립니다.


---

[← Computer Use](../35-Computer-Use/computer-use-ko.md) | [인덱스](../README_KO.md) | [텔레포트 →](../37-Teleport/teleport-system-ko.md)
