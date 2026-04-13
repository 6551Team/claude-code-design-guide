# 샌드박스(Sandbox) 시스템 아키텍처 문서

> Claude Code v2.1.88 샌드박스(Sandbox) 보안 격리 시스템 — 완전 기술 참조서

---

### 설계 철학

#### 왜 샌드박스(Sandbox)는 선택적인가?

모든 환경이 샌드박싱을 지원하지는 않습니다 — Linux 네임스페이스는 Docker 컨테이너 내에서 중첩될 수 없고, 일부 CI/CD 환경은 `sandbox-exec` 권한이 없으며, 원격 개발 환경은 자체 격리 레이어가 있을 수 있습니다. `sandboxTypes.ts`의 `failIfUnavailable` 설정(Config) 옵션은 사용자가 동작을 선택할 수 있게 합니다: `true`로 설정하면 사용 불가능한 샌드박스(Sandbox)가 오류를 발생시키고 실행을 중단합니다(고보안 시나리오에 적합); `false`로 설정하면 시스템이 샌드박스(Sandbox) 없이 실행하는 것으로 저하됩니다(기능 우선 시나리오에 적합). `enableWeakerNestedSandbox`와 `enableWeakerNetworkIsolation`은 호환성 타협으로 약화된 격리 옵션을 제공합니다.

#### 왜 세 가지 샌드박스(Sandbox) 타입인가?

각 OS는 다른 네이티브 격리 메커니즘을 가집니다 — macOS는 `sandbox-exec`(Seatbelt)를 사용하고, Linux는 사용자 네임스페이스 + cgroups를 사용하며, Docker 환경은 컨테이너 격리를 사용합니다. 기본 시스템 호출이 완전히 다르기 때문에 단일 접근법으로 통합하는 것은 불가능합니다. 샌드박스(Sandbox) 어댑터 레이어(`sandbox-adapter.ts`)는 초기화 시 환경을 감지하고 적절한 구현을 선택하여 상위 레이어에 통합 인터페이스를 제공합니다.

#### 왜 세밀한 권한 설정(Config)을 허용하는가?

다른 도구는 다른 권한이 필요합니다 — Bash는 `npm install`을 위해 네트워크 접근이 필요할 수 있고, FileRead는 파일 시스템 읽기 권한만 필요하며, `excludedCommands` 목록은 특정 명령이 샌드박스(Sandbox)를 우회할 수 있게 합니다. `autoAllowBashIfSandboxed`의 존재는 실용적인 절충안을 보여줍니다: 샌드박스(Sandbox)가 이미 격리 보호를 제공할 때 사용자 확인 프롬프트를 줄여 상호작용 유창성을 향상시킬 수 있습니다.

## 샌드박스(Sandbox) 설정(Config) (settings.json의 sandbox 필드)

```typescript
sandbox: {
  enabled: boolean,                      // 샌드박스(Sandbox) 활성화
  failIfUnavailable: boolean,            // 사용 불가 시 실패 vs 저하
  allowUnsandboxedCommands: boolean,      // 샌드박스 없는 명령 실행 허용
  network: {...},                         // 네트워크 제한 설정(Config)
  filesystem: {...},                      // 파일 시스템 제한 설정(Config)
  ignoreViolations: boolean,             // 위반 무시 (실행 차단 안 함)
  excludedCommands: string[],            // 제외된 명령 (샌드박스(Sandbox) 우회)
  autoAllowBashIfSandboxed: boolean,     // 샌드박스 모드에서 bash 자동 허용
  enableWeakerNestedSandbox: boolean,    // 약화된 중첩 샌드박스(Sandbox) 활성화
  enableWeakerNetworkIsolation: boolean, // 약화된 네트워크 격리 활성화
  ripgrep: {...}                         // ripgrep 특화 설정(Config)
}
```

### 설정(Config) 필드 참조

| 필드 | 타입 | 설명 |
|-------|------|-------------|
| `enabled` | boolean | 샌드박스(Sandbox) 격리 활성화 여부 |
| `failIfUnavailable` | boolean | 샌드박스(Sandbox) 사용 불가 시 동작: `true`는 오류 발생 및 중단, `false`는 샌드박스(Sandbox) 없이 실행으로 저하 |
| `allowUnsandboxedCommands` | boolean | 샌드박스(Sandbox) 없는 명령 실행 허용 여부 |
| `ignoreViolations` | boolean | 샌드박스(Sandbox) 위반 보고를 무시할지 여부 |
| `excludedCommands` | string[] | 샌드박스(Sandbox)를 우회하는 명령 화이트리스트 |
| `autoAllowBashIfSandboxed` | boolean | 샌드박스(Sandbox) 모드에서 bash 명령을 자동으로 승인 (사용자 확인 불필요) |
| `enableWeakerNestedSandbox` | boolean | 약화된 중첩 샌드박스(Sandbox) 사용 허용 (호환성 옵션) |
| `enableWeakerNetworkIsolation` | boolean | 약화된 네트워크 격리 정책 사용 |

---

## 샌드박스(Sandbox) 실행 (sandbox-adapter.ts)

### 초기화
```
M7.initialize(SK8)  // 샌드박스(Sandbox) 엔진을 비동기적으로 초기화
```

### 사용 불가 처리
- `failIfUnavailable = true` → 오류 발생, 실행 차단
- `failIfUnavailable = false` → 샌드박스(Sandbox) 없이 실행으로 저하

### 명령 실행 결정
```
shouldUseSandbox()  // 현재 명령이 샌드박스(Sandbox)를 사용할지 결정
```
고려 요소: 샌드박스(Sandbox) 가용성, 명령 제외 목록, `dangerouslyDisableSandbox` 파라미터 등.

### BashTool 통합
`BashTool`의 `dangerouslyDisableSandbox` 파라미터는 샌드박스(Sandbox) 보호를 명시적으로 우회할 수 있습니다(권한 인증 필요).

---

## 위반 감지

### removeSandboxViolationTags(text)
오류 메시지에서 `<sandbox_violations>` 태그를 제거하여, 사용자에게 표시하기 전에 내부 마커를 정리합니다.

### 위반 처리 흐름
1. 샌드박스(Sandbox)가 위반을 감지합니다
2. 위반 메시지가 형식화됩니다
3. `ignoreViolations` 설정(Config)에 따라 차단 여부를 결정합니다
4. 사용자에게 표시합니다 (무시되지 않은 경우)

---

## 네트워크 제어

### MITM 프록시
- 중간자 프록시를 사용하여 네트워크 요청을 차단합니다
- 차단된 요청은 다음을 반환합니다: `X-Proxy-Error: blocked-by-allowlist`
- 도메인 허용 목록 및 차단 목록 메커니즘을 지원합니다

### 프록시 소켓
```
getMitmSocketPath()  // 프록시 소켓 경로 가져오기
```

### 업스트림 프록시 (src/upstreamproxy/)

#### relay.ts (456줄)
TCP에서 WebSocket으로, CCR 터널 릴레이.

**핵심 구현 세부 정보**:

| 기능 | 설명 |
|---------|-------------|
| Protobuf 인코딩 | 수동 작성된 varint 인코딩/디코딩 (외부 protobuf 라이브러리 의존성 없음) |
| 백프레셔 처리 | Bun 부분 쓰기 vs Node 버퍼링의 차이를 처리합니다 |
| Keepalive | 연결을 유지하기 위한 30초 간격 핑어 |

---

## 스웜(Swarm) 모드의 샌드박스(Sandbox) 권한

다중 에이전트 스웜(Swarm) 모드에서는 샌드박스(Sandbox) 권한이 메일박스 시스템을 통해 워커와 리더 간에 전달됩니다.

### 권한 요청
```
sendSandboxPermissionRequestViaMailbox()
```
워커 → 리더: 샌드박스(Sandbox) 권한 요청을 전송합니다.

### 권한 응답
```
sendSandboxPermissionResponseViaMailbox()
```
리더 → 워커: 샌드박스(Sandbox) 권한 결정 결과를 반환합니다.

### 흐름
1. 워커 에이전트가 제한된 작업을 수행해야 합니다
2. 워커가 메일박스를 통해 리더에게 권한 요청을 전송합니다
3. 리더가 요청을 평가하고 결정을 내립니다
4. 리더가 메일박스를 통해 허용/거부 응답을 반환합니다
5. 워커가 응답에 따라 작업을 계속하거나 중단합니다

---

## 엔지니어링 실천 가이드

### 샌드박스(Sandbox) 활성화/비활성화

**`settings.json`의 `sandbox` 필드 설정(Config):**

```json
{
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": false,
    "autoAllowBashIfSandboxed": true,
    "excludedCommands": ["specific-command"],
    "enableWeakerNestedSandbox": false,
    "enableWeakerNetworkIsolation": false
  }
}
```

**샌드박스(Sandbox) 타입 선택 (자동 감지):**

| 환경 | 샌드박스(Sandbox) 구현 | 설명 |
|-------------|----------------------|-------------|
| macOS | `sandbox-exec` (Seatbelt) | 네이티브 macOS 샌드박스(Sandbox) |
| Linux | 사용자 네임스페이스 + cgroup | 네이티브 Linux 격리 |
| Docker | 컨테이너 격리 | 호스트 컨테이너의 격리 레이어 사용 |
| Windows | 네이티브 샌드박스(Sandbox) 없음 | `sandbox-exec`/`bwrap` 사용 불가 |

**failIfUnavailable 동작:**
- `true` → 샌드박스(Sandbox) 사용 불가 시 오류 발생 및 중단 (고보안 시나리오)
- `false` → 샌드박스(Sandbox) 없이 실행으로 저하 (기능 우선 시나리오)

### 샌드박스(Sandbox) 위반 디버깅

**문제 해결 단계:**

1. **위반 메시지 확인**: 샌드박스(Sandbox)가 위반을 감지하면 위반 메시지를 형식화하고 `ignoreViolations`에 따라 차단 여부를 결정합니다
2. **위반 태그 검사**: `removeSandboxViolationTags(text)`는 오류 메시지에서 `<sandbox_violations>` 태그를 제거합니다 — 내부 마커는 사용자에게 표시하기 전에 정리됩니다
3. **명령 제외 목록 확인**: `excludedCommands`의 명령은 샌드박스(Sandbox)를 우회합니다
4. **`dangerouslyDisableSandbox` 파라미터 확인**: BashTool은 이 파라미터를 지원하여 샌드박스(Sandbox)를 명시적으로 우회합니다(권한 인증 필요)
   - 소스 코드 `BashTool/prompt.ts`에 따르면: 기본적으로 샌드박스(Sandbox)에서 실행됩니다; 샌드박스(Sandbox) 제한으로 명령이 실패한 후 재시도할 때만 `dangerouslyDisableSandbox: true`를 사용합니다
   - `allowUnsandboxedCommands = false`인 경우 이 파라미터는 완전히 무시됩니다

**`shouldUseSandbox()` 결정 요소:**
- 샌드박스(Sandbox) 가용성
- 명령 제외 목록
- `dangerouslyDisableSandbox` 파라미터
- 샌드박스(Sandbox) 설정(Config)의 다양한 스위치

### 커스텀 샌드박스(Sandbox) 규칙

**파일 시스템 규칙:**
- `sandbox.filesystem`을 통해 허용/거부 디렉터리를 설정(Config)합니다
- 작업 디렉터리와 프로젝트 디렉터리는 일반적으로 허용 목록에 있습니다

**네트워크 규칙:**
- `sandbox.network`는 네트워크 접근 권한을 설정(Config)합니다
- MITM 프록시가 네트워크 요청을 차단합니다; 차단된 요청은 `X-Proxy-Error: blocked-by-allowlist`를 반환합니다
- `enableWeakerNetworkIsolation`은 약화된 네트워크 격리를 제공합니다(호환성 타협)
- 업스트림 프록시(`relay.ts`)는 수동 작성된 varint 인코딩과 30초 keepalive가 있는 TCP→WebSocket→CCR 터널 릴레이를 사용합니다

**약화 옵션 (호환성 타협):**
- `enableWeakerNestedSandbox` — 이미 격리 레이어가 있는 환경에서 약화된 중첩 샌드박스(Sandbox) 사용
- `enableWeakerNetworkIsolation` — 약화된 네트워크 격리 정책 사용

### 스웜(Swarm) 모드의 샌드박스(Sandbox) 권한

**다중 에이전트 모드:**
1. 워커가 `sendSandboxPermissionRequestViaMailbox()`를 통해 권한을 요청합니다
2. 리더가 `sendSandboxPermissionResponseViaMailbox()`를 통해 결정을 반환합니다
3. 권한은 메일박스 시스템을 통해 전달되어 에이전트 간에 일관된 보안 결정을 보장합니다

### 일반적인 함정

| 함정 | 세부 정보 | 해결 방법 |
|---------|---------|----------|
| Docker 내에서 네임스페이스 샌드박스(Sandbox)를 중첩할 수 없음 | Linux 네임스페이스는 Docker 컨테이너 내에서 사용 불가 | `enableWeakerNestedSandbox`를 사용하거나 `failIfUnavailable: false`로 저하 |
| 샌드박스(Sandbox)가 정상적인 도구 작업을 차단할 수 있음 | `npm install` 및 `pip install`과 같이 네트워크 접근이 필요한 명령이 차단될 수 있음 | `excludedCommands`를 설정(Config)하거나 네트워크 접근 허용 |
| `autoAllowBashIfSandboxed`의 보안 절충 | 샌드박스(Sandbox)가 이미 격리를 제공할 때 사용자 확인 프롬프트를 줄임 | 개발 환경에 적합하지만 프로덕션/보안 민감 환경에서는 주의하여 사용 |
| Windows에 네이티브 샌드박스(Sandbox) 없음 | `sandbox-exec`/`bwrap`은 Windows에서 기본적으로 사용 불가 | 소스 코드 주석은 PowerShell 도구가 네이티브 Windows에서 샌드박스(Sandbox)가 없음을 확인합니다 |
| 샌드박스(Sandbox) 위반이 조용히 무시될 수 있음 | `ignoreViolations: true`인 경우 위반이 실행을 차단하지 않음 | 이 옵션은 디버깅/개발 시나리오에만 사용하십시오 |
| 각 명령은 개별적으로 샌드박싱 여부를 평가함 | 최근에 `dangerouslyDisableSandbox`를 사용한 후에도 이후 명령은 기본적으로 샌드박스(Sandbox) 모드 | 소스 코드 프롬프트는 명시적으로 "각 명령을 개별적으로 처리"를 요구합니다 |


---

[← LSP(Language Server Protocol) 통합](../23-LSP集成/lsp-integration-ko.md) | [인덱스](../README_KO.md) | [Git & GitHub →](../25-Git与GitHub/git-github-ko.md)
