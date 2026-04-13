# 셸 도구 체인(Shell Toolchain) 아키텍처

## 설계 철학

### 설계 철학: 왜 정규식 매칭 대신 Bash AST를 파싱하는가?

Bash 구문의 복잡성은 정규식으로 처리할 수 있는 범위를 훨씬 초과합니다:

1. **중첩 파이프라인** -- `echo $(cat file | grep pattern | awk '{print $1}')`은 커맨드 치환, 파이프라인 체인, 중첩 따옴표를 포함하며; 정규식은 경계를 올바르게 식별할 수 없습니다
2. **Here Document** -- `<<EOF ... EOF` 구문 내부의 내용은 명령으로 파싱되어서는 안 됩니다; 이는 AST 수준의 구문적 이해가 필요합니다(`heredoc.ts`에서 특별히 처리됨)
3. **보안 분류의 정확성** -- 커맨드 보안 검사는 커맨드 이름과 인수를 정확하게 식별해야 합니다; 정규식 오분류는 위험한 명령을 통과시키거나 안전한 명령을 차단할 수 있습니다. 소스는 Tree-sitter 기반의 심층 구문 분석을 제공하기 위해 `treeSitterAnalysis.ts`를 사용합니다
4. **다층 파싱 아키텍처** -- `bashParser.ts`(기본 AST) → `bashPipeCommand.ts`(파이프라인 분석) → `treeSitterAnalysis.ts`(의미론적 이해). 각 레이어는 다른 세분화 수준에서 문제를 해결합니다

### 설계 철학: 왜 PowerShell도 지원하는가?

Windows 사용자의 기본 셸은 PowerShell입니다; 이를 지원하지 않으면 Windows 플랫폼이 제외됩니다. 소스는 `src/utils/powershell/` 아래에 독립적인 PowerShell 보안 분석 시스템을 제공합니다:

- `dangerousCmdlets.ts`는 6가지 범주의 위험한 cmdlet 목록(`FILEPATH_EXECUTION_CMDLETS`, `DANGEROUS_SCRIPT_BLOCK_CMDLETS`, `MODULE_LOADING_CMDLETS`, `SHELLS_AND_SPAWNERS`, `NETWORK_CMDLETS`, `ALIAS_HIJACK_CMDLETS`)을 유지하며, PowerShell에 고유한 공격 표면을 커버합니다
- `SHELL_TYPES = ['bash', 'powershell']`은 Shell Provider 추상화 레이어에서 병렬적인 일급 시민으로 취급됩니다
- `resolveDefaultShell.ts`는 운영 체제에 따라 적절한 기본 셸을 자동으로 선택합니다

---

## Bash 유틸리티

`src/utils/bash/` 디렉터리에는 완전한 Bash 커맨드 분석 및 처리 기능을 제공하는 20개 이상의 파일이 있습니다.

### CommandSpec 타입

```typescript
type CommandSpec = {
  name: string           // 커맨드 이름
  description: string    // 커맨드 설명
  subcommands: ...       // 서브커맨드 정의
  args: ...              // 위치 인수 정의
  options: ...           // 옵션/플래그 정의
}
```

### 커맨드 스펙 가져오기

- **getCommandSpec()**: 커맨드의 스펙 정의를 가져옵니다. 성능 최적화를 위해 메모이즈된 LRU 캐시를 사용하며, 먼저 로컬 스펙 디렉터리를 확인한 후 캐시 미스 시 `@withfig/autocomplete` 패키지로 폴백합니다.

### AST 파싱

- **bashParser.ts**: Bash 스크립트용 AST(추상 구문 트리) 파서; 커맨드 문자열을 구조화된 표현으로 파싱합니다
- **bashPipeCommand.ts**: 파이프라인 커맨드(`|`)의 파싱 및 분석; 파이프라인 체인의 각 커맨드를 독립적으로 파싱합니다
- **heredoc.ts**: Here Document(`<<EOF`) 구문의 파싱 및 처리
- **treeSitterAnalysis.ts**: Tree-sitter 기반의 심층 구문 분석으로 더 정확한 AST 노드 식별 및 의미론적 이해를 제공합니다

### 셸 인용 처리

- **shellQuote.ts**: 셸 인용/이스케이프 유틸리티
- **shellQuoting.ts**: 여러 인용 전략을 지원하는 확장 인용 기능

### 셸 스냅샷

- **ShellSnapshot.ts**: 셸 환경 스냅샷; 셸 상태를 캡처하고 복원합니다

### 로컬 커맨드 스펙

시스템에는 다음 일반 커맨드에 대한 내장 로컬 스펙 정의가 있습니다: `alias`, `nohup`, `pyright`, `sleep`, `srun`, `time`, `timeout`

## PowerShell 유틸리티

`src/utils/powershell/`은 PowerShell 커맨드에 대한 보안 분석 및 권한 제어를 제공합니다.

### 위험한 Cmdlet 범주

시스템은 커맨드 실행 전 보안 검사에 사용되는 여러 위험한 cmdlet 목록을 유지합니다:

- **FILEPATH_EXECUTION_CMDLETS**: 파일 경로를 통해 코드를 실행하는 cmdlet
  - `invoke-command`, `start-job` 등

- **DANGEROUS_SCRIPT_BLOCK_CMDLETS**: 임의의 코드 블록을 실행할 수 있는 10개의 cmdlet
  - 이러한 cmdlet은 ScriptBlock 파라미터를 받아 임의의 PowerShell 코드 실행이 가능합니다

- **MODULE_LOADING_CMDLETS**: 모듈 로딩 관련
  - `import-module`, `install-module` 등
  - 신뢰할 수 없는 코드를 도입할 수 있습니다

- **SHELLS_AND_SPAWNERS**: 셸 및 프로세스 스포너
  - `pwsh`, `cmd`, `bash`, `wsl`, `start-process` 등
  - PowerShell의 보안 제한을 우회할 수 있습니다

- **NETWORK_CMDLETS**: 네트워크 요청 관련
  - `invoke-webrequest`, `invoke-restmethod`
  - 데이터 유출을 유발할 수 있습니다

- **ALIAS_HIJACK_CMDLETS**: 별칭 하이재킹 관련
  - `set-alias`, `set-variable`
  - 기존 커맨드 동작을 변조할 수 있습니다

### 파서

- **parsePowerShellCommand()**: PowerShell 커맨드 문자열을 파싱하여 커맨드 이름, 인수, 파이프라인 구조를 추출합니다
- **getAllCommands()**: 파싱된 결과에서 모든 커맨드를 추출합니다 (파이프라인 체인의 각 커맨드 포함)

### 권한 접두사 추출

- **extractPrefixFromElement()**: PowerShell AST 요소에서 권한 검사에 필요한 커맨드 접두사를 추출합니다; 커맨드에 사용자 확인이 필요한지 결정하는 데 사용됩니다.

## 셸 제공자 추상화

`src/utils/shell/`은 크로스 플랫폼 셸 추상화 레이어를 제공합니다.

### ShellProvider 인터페이스

```typescript
interface ShellProvider {
  type: string                    // 셸 유형 식별자
  shellPath: string               // 셸 실행 파일 경로
  detached: boolean               // 분리 모드 사용 여부
  buildExecCommand(): ...         // 실행 커맨드 빌드
  getSpawnArgs(): ...             // 스폰 인수 가져오기
  getEnvironmentOverrides(): ...  // 환경 변수 재정의 가져오기
}
```

### 셸 유형

```typescript
SHELL_TYPES = ['bash', 'powershell']
```

### 제공자 구현

- **bashProvider.ts**: Bash 셸 제공자
  - 세션 환경 관리: 셸 세션의 환경 변수를 설정하고 유지합니다
  - eval-wrap: 올바른 셸 확장을 보장하기 위해 커맨드를 `eval` 내부에 감싸서 실행합니다
  - pwd 추적: 이후 커맨드가 올바른 디렉터리에서 실행되도록 작업 디렉터리 변경을 추적합니다
  - TMUX 소켓 격리: 세션 충돌을 방지하기 위해 TMUX 환경에서 소켓을 격리합니다
  - Windows null 리다이렉트 재작성: `/dev/null`을 Windows 호환 `NUL`로 재작성합니다

- **powershellProvider.ts**: PowerShell 제공자; PowerShell 특정 커맨드 구성 및 환경 설정을 처리합니다

### 헬퍼 모듈

- **resolveDefaultShell.ts**: 셸 감지. 운영 체제와 환경에 따라 적절한 기본 셸을 자동으로 선택합니다
- **readOnlyCommandValidation.ts**: 보안 검사. 커맨드가 읽기 전용 작업인지 검증합니다; 권한 제어 결정에 사용됩니다
- **outputLimits.ts**: 출력 크기 제한. 제한을 초과하는 출력을 잘라내어 지나치게 큰 커맨드 출력으로 인한 메모리 문제를 방지합니다

## 엔지니어링 실천

### Bash 커맨드 보안 분류 확장

- `bashSecurity.ts`에 새로운 위험 패턴 감지 규칙을 추가합니다 -- 이 파일은 보안 결정을 위해 `treeSitterAnalysis.ts`의 AST 분석 결과를 임포트하고 사용합니다
- Auto 모드에서 사용되는 2단계 분류기(Stage 1: 정규식 빠른 경로 → Stage 2: AI 느린 경로)는 일반적인 위험 커맨드를 단락시키기 위해 Stage 1에 새 정규식 패턴을 추가할 수 있습니다
- 새로 추가된 위험 패턴은 해당 단위 테스트도 업데이트해야 합니다

### PowerShell 파싱의 한계

- 특정 PowerShell 특수 구문(DSC 리소스, 커스텀 클래스 정의, 고급 함수 속성 등)은 `parsePowerShellCommand()`로 올바르게 파싱되지 않을 수 있습니다
- 파싱 실패 시 시스템은 보수적 전략으로 폴백합니다 -- 전체 커맨드를 사용자 확인이 필요한 것으로 처리합니다
- `extractPrefixFromElement()`는 AST에서 권한 검사에 필요한 커맨드 접두사를 추출하는 데 사용됩니다; 추출이 실패하면 전체 사용자 확인 흐름이 필요합니다
- Windows null 리다이렉트 재작성(`/dev/null` → `NUL`)은 `bashProvider.ts`에서 처리됩니다; PowerShell 환경에서 이러한 크로스 플랫폼 차이에 특별한 주의가 필요합니다

---

## Git 통합

`src/utils/git/`은 포괄적인 Git 저장소 작업 지원을 제공합니다.

### 구성 파싱

- **gitConfigParser.ts**: `.git/config` 파일 파서; Git 구성의 이스케이프 시퀀스(`\t`, `\n`, `\\` 등) 처리를 지원합니다.

### 파일 시스템 작업

**gitFilesystem.ts**(약 700줄)는 Git 파일 시스템 작업의 핵심 모듈입니다:

- **resolveGitDir()**: `.git` 디렉터리의 실제 경로를 해석합니다. 워크트리(워크트리 파일의 `gitdir:` 참조)와 서브모듈(서브모듈의 중첩된 `.git` 디렉터리)을 올바르게 처리합니다.

- **isSafeRefName()**: Git 참조 이름 안전성을 검증합니다:
  - 경로 탐색 차단(`..`)
  - 인수 인젝션 차단(선행 `-`)
  - 셸 메타문자 차단(`$`, `` ` ``, `|`, `;` 등)

- **isValidGitSha()**: Git SHA 형식을 검증합니다:
  - SHA-1: 40개의 16진수 문자
  - SHA-256: 64개의 16진수 문자

- **readGitHead()**: HEAD 파일을 파싱합니다:
  - 브랜치 참조: `ref: refs/heads/main`
  - 분리된 HEAD: 직접 SHA 값

- **GitFileWatcher 클래스**: 파일 감시기
  - `.git/HEAD`, `config`, 브랜치 참조 파일의 변경을 모니터링합니다
  - 캐시 + 더티 마킹 전략 사용: 파일이 변경될 때 캐시를 더티로 표시하고 다음 읽기 시 새로 고침합니다

- **캐시된 쿼리 함수**:
  - `getCachedBranch()`: 캐시된 현재 브랜치 이름을 가져옵니다
  - `getCachedHead()`: 캐시된 HEAD 참조를 가져옵니다
  - `getCachedRemoteUrl()`: 캐시된 원격 저장소 URL을 가져옵니다
  - `getCachedDefaultBranch()`: 캐시된 기본 브랜치 이름을 가져옵니다

- **저장소 상태 쿼리**:
  - `isShallowClone()`: 저장소가 얕은 클론인지 감지합니다
  - `getWorktreeCountFromFs()`: 파일 시스템에서 워크트리 수를 가져옵니다

### Gitignore 처리

- **gitignore.ts**:
  - `isPathGitignored()`: 주어진 경로가 gitignore 규칙에 의해 무시되는지 확인합니다
  - `addFileGlobRuleToGitignore()`: `.gitignore` 파일에 glob 규칙을 추가합니다

## GitHub 통합

`src/utils/github/`은 GitHub CLI 통합을 제공합니다:

- **getGhAuthStatus()**: GitHub CLI 인증 상태를 감지합니다
  - 반환값: `'authenticated' | 'not_authenticated' | 'not_installed'`
  - 구현: `gh auth token` 커맨드를 호출하여(네트워크 요청 없음) 종료 코드와 출력으로 상태를 결정합니다
  - 효율적: 네트워크 호출을 하지 않으며 로컬 토큰 상태만 확인합니다

## DXT 확장 시스템

`src/utils/dxt/`은 DXT(Desktop Extension) 확장 패키지의 처리를 구현합니다:

### 매니페스트 검증

- **validateManifest()**: Zod 스키마를 사용하여 확장 매니페스트 파일을 검증합니다; 스키마 정의는 `@anthropic-ai/mcpb` 패키지에서 가져옵니다. 검증은 필수 필드, 타입 제약, 값 범위 등을 커버합니다.

### ID 생성

- **generateExtensionId()**: 작성자 이름과 확장 이름을 기반으로 정제된 고유 식별자를 생성합니다; 특수 문자를 제거하고 형식을 정규화합니다.

### Zip 처리

DXT 확장 패키지(zip 형식)에는 엄격한 보안 제한이 적용됩니다:

| 제한 | 값 |
|-------|-------|
| 파일당 최대 크기 | 512 MB |
| 총 압축 해제 크기 | 1024 MB |
| 최대 파일 수 | 100,000 |
| 최대 압축 비율 | 50:1 |

### 보안 검사

- **isPathSafe()**: 경로 안전성 검증; 경로 탐색(`..`)이 포함된 항목을 거부하여 zip slip 공격을 방지합니다
- **Zip bomb 감지**: 압축 비율 확인(50:1 상한선)을 통해 잠재적인 zip bomb을 감지하여, 압축 해제 중 디스크 공간과 메모리 소진을 방지합니다


---

[← 비용 추적](../42-代价追踪/cost-tracking-ko.md) | [인덱스](../README_KO.md) | [스크린 컴포넌트 →](../44-Screens组件/screens-components-ko.md)
