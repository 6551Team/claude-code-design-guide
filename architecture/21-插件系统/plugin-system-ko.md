# 플러그인(Plugin) 시스템 아키텍처

### 설계 철학

왜 MCP(Model Context Protocol) 대신 독립적인 플러그인(Plugin) 시스템이 필요한가? MCP(Model Context Protocol)는 도구 프로토콜입니다 — AI가 외부 도구를 호출하는 방법을 정의합니다. 하지만 플러그인(Plugin) 시스템은 더 광범위한 확장 메커니즘입니다: 플러그인(Plugin)은 UI를 수정하고(`loadPluginOutputStyles.ts`를 통해), 슬래시 명령을 주입하며(`loadPluginCommands.ts`를 통해), 생명주기 훅(Hooks)을 등록하고(`loadPluginHooks.ts`를 통해), 커스텀 에이전트를 정의할 수 있습니다(`loadPluginAgents.ts`를 통해). 이러한 기능들은 MCP(Model Context Protocol) 도구 프로토콜의 범위를 벗어납니다. `mcpPluginIntegration.ts`의 존재가 이 관계를 정확히 보여줍니다 — MCP(Model Context Protocol) 서버는 전체 플러그인(Plugin) 시스템을 대체하는 것이 아니라 플러그인(Plugin)의 하위 집합으로 통합됩니다.

## 플러그인(Plugin) 탐색 및 로딩

플러그인(Plugin) 시스템의 핵심 로딩 흐름은 다음 모듈에 의해 조율됩니다:

- **pluginLoader.ts**: 메인 오케스트레이터로, 전체 플러그인(Plugin) 탐색 및 로딩 프로세스를 조율합니다. 등록된 모든 플러그인(Plugin) 디렉터리를 스캔하고, 우선순위에 따라 플러그인(Plugin)을 로드하며, 로딩 오류를 처리하고 보고합니다.
- **loadPluginAgents.ts**: 커스텀 에이전트 정의를 로드합니다. 플러그인(Plugin)은 에이전트 설정(Config)을 선언하고 커스텀 에이전트 동작과 도구 집합을 정의하여 시스템의 에이전트 기능을 확장할 수 있습니다.
- **loadPluginCommands.ts**: 플러그인(Plugin) 슬래시 명령을 로드합니다. 각 플러그인(Plugin)은 자체 명령을 등록할 수 있으며, 이것들은 사용자의 명령 목록에 나타납니다.
- **loadPluginHooks.ts**: 플러그인(Plugin) 훅(Hooks)을 로드합니다. 플러그인(Plugin)은 세션 시작, 메시지 전송 전후 등 특정 생명주기 이벤트에 콜백을 등록할 수 있습니다.
- **loadPluginOutputStyles.ts**: 터미널 출력 스타일 정의를 로드합니다. 플러그인(Plugin)은 색상, 형식, 레이아웃을 포함하여 출력의 터미널 렌더링을 커스터마이징할 수 있습니다.
- **pluginDirectories.ts**: 경로 유틸리티 함수입니다. 다중 수준 범위 디렉터리 구조를 지원하는 플러그인(Plugin) 디렉터리 해석, 조회, 경로 결합 기능을 제공합니다.
- **installedPluginsManager.ts**: 플러그인(Plugin) 레지스트리 관리자입니다. 설치된 플러그인(Plugin)의 완전한 인벤토리를 유지하고, CRUD 작업을 지원하며 플러그인(Plugin) 메타데이터를 영속화합니다.

## 범위 관리

플러그인(Plugin) 시스템은 플러그인(Plugin) 설치와 가시성을 관리하기 위해 계층화된 범위 모델을 사용합니다:

```typescript
VALID_INSTALLABLE_SCOPES = ['user', 'project', 'local']  // 'managed' 제외
VALID_UPDATE_SCOPES  // 'managed' 포함, 업데이트는 허용하지만 직접 설치는 불가
```

**범위 우선순위**: `local > project > user`

- **findPluginInSettings()**: 플러그인(Plugin)을 조회할 때 가장 구체적인 범위가 우선됩니다. `local` 범위가 `project`를 오버라이드하고, `project`가 `user`를 오버라이드합니다. 이를 통해 프로젝트 수준과 로컬 수준 설정(Config)이 전역 설정을 오버라이드할 수 있습니다.
- **V2 데이터 폴백**: `resolveDelistedPluginId()`는 목록에서 제거된 플러그인(Plugin)의 ID 해석을 처리하여 이전 데이터 형식과의 하위 호환성을 보장합니다. 플러그인(Plugin)이 마켓플레이스에서 제거된 후에도 설치된 인스턴스를 올바르게 식별해야 합니다.

## 마켓플레이스 통합

마켓플레이스 통합은 플러그인(Plugin) 탐색, 설치, 관리 기능을 제공합니다:

- **officialMarketplace.ts**: 공식 마켓플레이스 클라이언트로, 플러그인(Plugin) 검색, 세부 정보 검색, 버전 쿼리를 위한 API 인터페이스를 제공합니다.
- **officialMarketplaceGcs.ts**: GCS(Google Cloud Storage) 기반 마켓플레이스 백엔드 구현입니다. 플러그인(Plugin) 패키지와 메타데이터는 GCS 버킷에 저장되어 고가용성과 글로벌 배포를 지원합니다.
- **officialMarketplaceStartupCheck.ts**: 시작 마켓플레이스 검사입니다. 애플리케이션 시작 시 마켓플레이스 접근성을 확인하고, 플러그인(Plugin) 업데이트를 확인하며, 오프라인 시나리오에 대한 저하 전략을 처리합니다.
- **marketplaceManager.ts**: CRUD 작업 관리자입니다. 마켓플레이스 플러그인(Plugin)에 대한 완전한 생명주기 작업인 설치(Create), 쿼리(Read), 업데이트(Update), 제거(Delete)를 캡슐화합니다.
- **parseMarketplaceInput.ts**: URL 파서입니다. 사용자가 입력한 마켓플레이스 URL, 플러그인(Plugin) 식별자, 버전 제약을 파싱하여 여러 입력 형식(전체 URL, 짧은 이름, name@version 등)을 지원합니다.

## 플러그인(Plugin) 생명주기

플러그인(Plugin) 생명주기 관리는 유효성 검사부터 자동 업데이트까지의 완전한 흐름을 포함합니다:

- **validatePlugin.ts**: 스키마(Schema) 유효성 검사기로, `plugin.json`에 대해 엄격한 구조적 및 타입 유효성 검사를 수행합니다. 필수 필드, 타입 제약, 값 범위 검사를 포함하여 플러그인(Plugin) 선언 파일이 사양에 부합하는지 보장합니다.
- **pluginVersioning.ts**: 버전 관리 모듈입니다. 시맨틱 버전(semver)의 파싱, 비교, 호환성 검사를 처리하며, 버전 범위 제약과 업그레이드 경로 계산을 지원합니다.

  #### 왜 이 설계인가?

  플러그인(Plugin) API는 Claude Code 버전에 걸쳐 변경될 수 있습니다 — 호환되지 않는 플러그인(Plugin)은 충돌이나 보안 취약성을 일으킬 수 있습니다. 버전 호환성 검사(semver 비교를 통해)는 런타임에 문제를 발견하는 것이 아니라 로딩 단계에서 문제를 차단합니다. `pluginStartupCheck.ts`는 플러그인(Plugin) 로딩 전에 상태 검사를 수행하여 의존성 무결성과 런타임 호환성을 검증하고, `pluginBlocklist.ts`는 알려진 악성 또는 호환되지 않는 플러그인(Plugin)의 차단 목록을 유지하여 로드 전에 차단합니다. 이 "심층 방어" 접근법은 검증된 플러그인(Plugin)만 런타임에 진입할 수 있도록 보장합니다.

- **pluginOptionsStorage.ts**: 영속적인 옵션 저장소입니다. 플러그인(Plugin) 런타임 설정(Config)에 대한 영속적인 읽기/쓰기를 제공하고, 범위 격리된 저장소를 지원하며, 플러그인(Plugin) 설정(Config)이 세션 간에 지속되도록 보장합니다.
- **pluginPolicy.ts**: 보안 정책 엔진입니다. 플러그인(Plugin)의 권한 모델을 정의하고 적용하여, 파일 시스템, 네트워크, 도구와 같은 리소스에 대한 플러그인(Plugin) 접근을 제어합니다.
- **pluginBlocklist.ts**: 차단 목록 관리입니다. 알려진 악성 또는 호환되지 않는 플러그인(Plugin)의 차단 목록을 유지하고, 로딩 전에 차단하며, 원격 차단 목록 규칙 업데이트를 지원합니다.
- **pluginFlagging.ts**: 상태 플래깅 시스템입니다. 다양한 플러그인(Plugin) 상태(예: 더 이상 사용되지 않음, 검토 필요, 보안 문제 있음)를 표시하여 플러그인(Plugin) 표시 및 로딩 동작에 영향을 미칩니다.
- **pluginStartupCheck.ts**: 시작 유효성 검사입니다. 플러그인(Plugin) 로딩 전에 상태 검사를 수행하여 의존성 무결성, 런타임 호환성, 설정(Config) 유효성을 확인합니다.
- **pluginAutoupdate.ts**: 자동 업데이트 메커니즘입니다. 백그라운드에서 플러그인(Plugin) 업데이트를 확인하고, 사용자 정책에 따라 자동으로 또는 업데이트를 요청하며, 업데이트 충돌과 롤백을 처리합니다.
- **headlessPluginInstall.ts**: 프로그래매틱 설치 인터페이스입니다. 비대화식 플러그인(Plugin) 설치를 지원하며, CI/CD 환경, 스크립트 배포, 일괄 설치 시나리오에 사용됩니다.

## 의존성 및 통합

플러그인(Plugin) 시스템과 외부 시스템 간의 통합 및 의존성 관리:

- **dependencyResolver.ts**: 의존성 해석기입니다. 플러그인(Plugin) 간의 의존성 그래프를 빌드하고, 순환 의존성을 감지하며, 올바른 로딩 순서를 결정하고, 버전 충돌을 처리합니다.
- **reconciler.ts**: 상태 조정기입니다. 원하는 상태(설정(Config) 파일에 선언됨)와 실제 상태(설치된 플러그인(Plugin))를 비교하고, 설치/제거/업데이트 작업 계획을 생성하며, 시스템 일관성을 보장합니다.
- **mcpPluginIntegration.ts**: MCP(Model Context Protocol) 서버 플러그인(Plugin) 통합입니다. MCP(Model Context Protocol)(Model Context Protocol) 서버를 플러그인(Plugin)으로 통합하고, MCP(Model Context Protocol) 서버의 생명주기를 관리하며, MCP(Model Context Protocol) 도구를 플러그인(Plugin) 도구 시스템과 연결합니다.

### 설계 철학: DXT 확장 형식

왜 DXT 형식(`utils/dxt/`)이 필요한가? DXT는 매니페스트, 코드, 에셋을 단일 파일(`.dxt` 또는 `.mcpb`)로 번들링하는 표준화된 플러그인(Plugin) 패키징 형식입니다. 표준화된 패키징은 설치/제거/업데이트를 원자적 작업으로 만듭니다 — 완전히 성공하거나 완전히 롤백되며, 손상된 부분 설치 상태가 없습니다. `helpers.ts`는 세 가지 파싱 진입점 — `parseDxtManifestFromJSON`, `parseDxtManifestFromText`, `parseDxtManifestFromBinary` — 을 제공하여 다른 소스에서의 로딩을 지원합니다. `mcpbHandler.ts`의 `isMcpbOrDxt()`는 파일이 패키지 형식인지 여부를 통합적으로 확인합니다. DXT 매니페스트는 사용자 설정(Config) 스키마(Schema)를 정의하여 설치 시 설정(Config) 완전성 유효성 검사를 가능하게 합니다.

- **lspPluginIntegration.ts**: LSP(Language Server Protocol) 통합입니다. Language Server Protocol 서버와 통합하여 플러그인(Plugin)에 언어 인텔리전스 기능(코드 완성, 진단, 정의로 이동 등)을 제공합니다.
- **hintRecommendation.ts**: 힌트 추천입니다. 사용자 행동과 컨텍스트를 기반으로 유용할 수 있는 플러그인(Plugin)을 추천하고, 지능적인 제안을 지원합니다.
- **lspRecommendation.ts**: LSP(Language Server Protocol) 추천입니다. 프로젝트에서 사용되는 프로그래밍 언어와 프레임워크를 기반으로 해당 LSP(Language Server Protocol) 플러그인(Plugin)을 추천합니다.

## 플러그인(Plugin) 텔레메트리(Telemetry)

플러그인(Plugin) 텔레메트리(Telemetry) 시스템은 사용 데이터와 오류 정보를 수집하는 데 사용됩니다:

- **hashPluginId()**: SHA256 해시의 처음 16자를 취하여 플러그인(Plugin) ID에 개인 정보 보호 처리를 적용하여, 텔레메트리(Telemetry) 데이터가 플러그인(Plugin) ID를 노출하지 않도록 합니다.

- **범위 분류**:
  - `official`: 공식 마켓플레이스 플러그인(Plugin)
  - `org`: 조직 수준 플러그인(Plugin)
  - `user-local`: 사용자 로컬 개발 플러그인(Plugin)
  - `default-bundle`: 기본 번들 플러그인(Plugin)

- **classifyPluginCommandError()**: 플러그인(Plugin) 명령 실행 오류를 5가지 카테고리로 분류하여 오류 귀속 및 모니터링 알림에 사용됩니다. 분류 결과는 재시도 전략 및 오류 보고 경로에 영향을 미칩니다.

- **logPluginsEnabledForSession()**: 현재 세션에서 활성화된 플러그인(Plugin) 목록을 기록하여 사용 통계와 문제 해결에 사용됩니다.
- **logPluginLoadErrors()**: 플러그인(Plugin) 로딩 실패에 대한 상세한 오류 정보를 스택 추적, 플러그인(Plugin) 버전, 환경 정보를 포함하여 기록합니다.

## 번들 플러그인(Bundled Plugins)

내장 플러그인(Plugin) 시스템은 즉시 사용 가능한 핵심 기능을 제공합니다:

- **builtinPlugins.ts**: 내장 플러그인(Plugin) 등록 및 관리 모듈입니다.
  - `registerBuiltinPlugin()`: 내장 플러그인(Plugin)을 등록합니다. 내장 플러그인(Plugin)은 사용자 설치 없이 시스템 시작 시 자동으로 로드됩니다.
  - `isBuiltinPluginId()`: 주어진 ID가 내장 플러그인(Plugin)인지 확인하여 내장 플러그인(Plugin)과 사용자 설치 플러그인(Plugin)을 구분하는 데 사용됩니다.

- **skillDefinitionToCommand()**: 스킬(Skills) 정의를 명령 형식으로 변환합니다. 이것은 내장 스킬(Skills) 시스템과 플러그인(Plugin) 명령 시스템 사이의 브리지 레이어로, 스킬(Skills)이 플러그인(Plugin) 명령으로 호출되고 표시될 수 있게 합니다.

## CLI 핸들러

`cli/handlers/plugins.ts` (약 580줄)는 완전한 CLI 명령 처리를 제공합니다:

**유효성 검사 및 쿼리**:
- `pluginValidateHandler()`: 플러그인(Plugin) 구조 및 설정(Config)의 합법성을 검증합니다
- `pluginListHandler()`: 설치된 플러그인(Plugin)을 나열하며, 범위별 필터링을 지원합니다

**마켓플레이스 작업**:
- `marketplaceAddHandler()`: 마켓플레이스에서 플러그인(Plugin)을 추가합니다
- `marketplaceListHandler()`: 마켓플레이스에서 사용 가능한 플러그인(Plugin)을 나열합니다
- `marketplaceRemoveHandler()`: 마켓플레이스에서 플러그인(Plugin)을 제거합니다
- `marketplaceUpdateHandler()`: 마켓플레이스 플러그인(Plugin)을 업데이트합니다

**플러그인(Plugin) 관리**:
- `pluginInstall()`: 지정된 범위에 플러그인(Plugin)을 설치합니다
- `pluginUninstall()`: 플러그인(Plugin)을 제거하고 리소스를 정리합니다
- `pluginEnable()`: 비활성화된 플러그인(Plugin)을 활성화합니다
- `pluginDisable()`: 플러그인(Plugin)을 설치된 상태로 유지하면서 비활성화합니다
- `pluginUpdate()`: 플러그인(Plugin)을 지정된 버전 또는 최신 버전으로 업데이트합니다

---

## 엔지니어링 실천 가이드

### DXT 플러그인(Plugin) 개발

**단계별 체크리스트:**

1. **플러그인(Plugin) 패키지 구조 생성**:
   ```
   my-plugin/
   ├── plugin.json          # DXT 매니페스트 (필수)
   ├── src/                  # 플러그인(Plugin) 코드
   └── README.md            # 문서
   ```
2. **매니페스트 정의**: `KeybindingBlockSchema` 및 `plugin.json` 스키마(Schema) 사양에 따라 플러그인(Plugin) 메타데이터, 권한 선언, 훅(Hooks) 인터페이스를 정의합니다.
3. **훅(Hooks) 인터페이스 구현**:
   - 커스텀 에이전트 — `loadPluginAgents.ts`를 통해 로드
   - 슬래시 명령 — `loadPluginCommands.ts`를 통해 등록
   - 생명주기 훅(Hooks) — `loadPluginHooks.ts`를 통해 등록
   - 출력 스타일 — `loadPluginOutputStyles.ts`를 통해 정의
4. **호환성 테스트**: `pluginValidateHandler()` CLI 명령을 사용하여 플러그인(Plugin) 구조를 검증합니다.
5. **패키징 및 게시**: DXT 형식은 매니페스트, 코드, 에셋을 단일 `.dxt` 또는 `.mcpb` 파일로 번들링합니다(원자적 작업 — 설치/제거/업데이트는 완전히 성공하거나 완전히 롤백됨).

**DXT 파싱 진입점** (`utils/dxt/helpers.ts`):
- `parseDxtManifestFromJSON` — JSON 객체에서 파싱
- `parseDxtManifestFromText` — 원시 텍스트에서 파싱
- `parseDxtManifestFromBinary` — 바이너리 데이터에서 파싱

### MCP(Model Context Protocol) 통합

**MCP(Model Context Protocol) 서버는 플러그인(Plugin)의 하위 집합으로 통합됩니다 (`mcpPluginIntegration.ts`):**

1. DXT 매니페스트 다운로드 및 추출
2. DXT 매니페스트 이름을 MCP(Model Context Protocol) 서버 이름으로 사용
3. MCP(Model Context Protocol) 도구를 플러그인(Plugin) 도구 시스템과 연결
4. `.mcp.json` 설정(Config) 파일 및 `.mcpb` 패키지 파일 지원

**파일이 패키지 형식인지 확인**: `isMcpbOrDxt()`는 `.mcpb` 및 `.dxt` 확장자에 대한 통합 검사를 제공합니다.

### 플러그인(Plugin) 로딩 디버깅

**문제 해결 단계:**

1. **플러그인(Plugin) 디렉터리 경로 확인**: `pluginDirectories.ts`가 경로 해석을 제공합니다; 플러그인(Plugin) 파일이 올바른 위치에 있는지 확인합니다.
2. **버전 호환성 확인**: `pluginVersioning.ts`가 semver 비교를 수행합니다; 호환되지 않는 플러그인(Plugin)은 로딩 단계에서 차단됩니다.
3. **차단 목록 확인**: `pluginBlocklist.ts`는 악성/호환되지 않는 플러그인(Plugin)의 차단 목록을 유지하고 로딩 전에 차단합니다.
4. **시작 유효성 검사 확인**: `pluginStartupCheck.ts`는 로딩 전에 상태 검사를 수행합니다(의존성 무결성, 런타임 호환성, 설정(Config) 유효성).
5. **로딩 오류 검토**: `logPluginLoadErrors()`는 상세한 오류 정보를 기록합니다(스택 추적, 플러그인(Plugin) 버전, 환경 정보).
6. **범위 우선순위 확인**: `local > project > user`; 플러그인(Plugin)이 동일한 이름의 더 높은 우선순위 플러그인(Plugin)에 의해 오버라이드되고 있는지 확인합니다.

**CLI 디버깅 명령:**
```bash
claude plugin validate <path>       # 플러그인(Plugin) 구조 검증
claude plugin list                  # 설치된 플러그인(Plugin) 나열
claude marketplace list             # 마켓플레이스에서 사용 가능한 플러그인(Plugin) 나열
```

### 플러그인(Plugin) 범위 관리

| 범위 | 설치 | 업데이트 | 설명 |
|-------|---------|--------|-------------|
| `user` | 허용 | 허용 | 전역적으로 효과 |
| `project` | 허용 | 허용 | 특정 프로젝트만 |
| `local` | 허용 | 허용 | 로컬 개발 |
| `managed` | 직접 설치 불가 | 허용 | 조직 관리 |

**V2 데이터 폴백**: `resolveDelistedPluginId()`는 목록에서 제거된 플러그인(Plugin)의 ID 해석을 처리하여 하위 호환성을 보장합니다.

### 일반적인 함정

| 함정 | 세부 정보 | 해결 방법 |
|---------|---------|----------|
| 플러그인(Plugin)은 파일 시스템에 접근할 수 있음 | 플러그인(Plugin)은 파일을 읽고 쓸 수 있는 능력이 있음; 보안 검토가 중요 | `pluginPolicy.ts`는 파일 시스템/네트워크/도구 접근을 제어하는 권한 모델을 정의함 |
| 플러그인(Plugin) API 변경은 버전 적응이 필요함 | 플러그인(Plugin) API는 Claude Code 버전에 걸쳐 변경됨 | semver 제약 사용; `pluginVersioning.ts`가 호환성을 확인함 |
| 순환 의존성 | 플러그인(Plugin) 간에 순환 의존성이 형성될 수 있음 | `dependencyResolver.ts`가 순환 의존성을 감지하고 올바른 로딩 순서를 설정함 |
| 플러그인(Plugin) 상태 조정 | 원하는 상태와 실제 상태가 일치하지 않을 수 있음 | `reconciler.ts`가 선언된 상태와 설치된 상태를 비교하고 작업 계획을 생성함 |
| 헤드리스 설치 | CI/CD 환경은 비대화식 설치가 필요함 | `headlessPluginInstall.ts` 프로그래매틱 설치 인터페이스 사용 |
| 플러그인(Plugin) 텔레메트리(Telemetry) 개인 정보 | 텔레메트리(Telemetry)에 특정 플러그인(Plugin) 이름/경로를 노출하지 마십시오 | `hashPluginId()`는 SHA256 해시의 처음 16자를 취함 |


---

[← 서비스 레이어](../20-服务层/services-complete-ko.md) | [인덱스](../README_KO.md) | [OAuth & 인증 →](../22-OAuth与认证/oauth-auth-ko.md)
