# 키바인딩(Keybinding) 시스템

Claude Code의 키바인딩(Keybinding) 시스템은 키보드 단축키를 정의, 파싱, 매칭, 실행하는 완전한 프레임워크를 제공합니다. 단일 키, 수정자 조합, 코드 시퀀스를 지원하며 Vim 모드와 통합됩니다.

### 설계 철학

#### 왜 50개 이상의 액션과 코드 지원이 필요한가?

Claude Code의 핵심 사용자층은 고급 개발자로, Vim/Emacs 사용자와 터미널 헤비 유저들입니다. 이들은 풍부한 키바인딩과 코드 시퀀스(`ctrl+k ctrl+s` 같은)를 기대합니다. 이 기능이 없다면 파워 유저들은 이탈할 것입니다. 50개 이상의 액션 식별자(`KEYBINDING_ACTIONS`)는 애플리케이션 수준(종료, 인터럽트, 화면 지우기)부터 컴포넌트 수준(자동완성, 탭, 히스토리 검색)까지 모든 상호작용 시나리오를 포괄합니다. 코드 지원은 제한된 키 조합 공간을 확장합니다. 단일 키와 수정자 조합의 수는 유한하지만, 코드 시퀀스는 거의 무한한 네임스페이스를 제공합니다.

#### 왜 컨텍스트 기반 매칭인가?

동일한 키 입력이 UI 상태에 따라 다르게 동작합니다: `Up` 키는 Chat 컨텍스트에서 히스토리를 스크롤하고, Autocomplete 컨텍스트에서는 이전 완성 항목을 선택하며, Transcript 컨텍스트에서는 위로 스크롤합니다. 17개의 컨텍스트 이름(`KEYBINDING_CONTEXTS`)은 키바인딩이 활성화되는 범위를 정의합니다. `resolveKey`는 주어진 컨텍스트 내에서 매치를 찾아 컨텍스트 간 키 충돌을 방지합니다. `ChordResolveResult`의 `chord_started`/`chord_cancelled` 상태는 코드 시퀀스가 진행 중일 때 시스템이 일반 키 처리를 일시 중지하게 해주며, 시퀀스가 완료되거나 타임아웃으로 취소될 때까지 대기합니다.

---

## 스키마 (schema.ts)

### KEYBINDING_CONTEXTS

다양한 UI 상태에서 키바인딩 범위를 구분하는 17개의 컨텍스트 이름을 정의합니다:

- `Global` - 전역 컨텍스트, 모든 상태에서 활성
- `Chat` - 채팅 입력 영역
- `Autocomplete` - 자동완성 메뉴
- `Task` - 진행 중인 작업
- `Confirmation` - 확인 대화상자
- `Tabs` - 탭 내비게이션
- `Transcript` - 대화 히스토리 브라우징
- `HistorySearch` - 히스토리 검색 모드
- 기타 컨텍스트 (Vim Normal, Vim Insert, Diff, FileViewer 등)

### KEYBINDING_ACTIONS

`context:action` 명명 규칙을 사용하는 50개 이상의 액션 식별자를 정의합니다:

- **앱 수준**: `app:exit`, `app:interrupt`, `app:clear`, `app:help`, `app:toggleVim`
- **채팅 수준**: `chat:submit`, `chat:newline`, `chat:paste`, `chat:historyUp`, `chat:historyDown`, `chat:cancel`
- **확인 수준**: `confirm:yes`, `confirm:no`, `confirm:always`, `confirm:explain`
- **자동완성 수준**: `autocomplete:accept`, `autocomplete:dismiss`, `autocomplete:next`, `autocomplete:prev`
- **탭 수준**: `tabs:next`, `tabs:prev`, `tabs:close`
- **트랜스크립트 수준**: `transcript:scrollUp`, `transcript:scrollDown`, `transcript:pageUp`, `transcript:pageDown`, `transcript:top`, `transcript:bottom`
- **히스토리 검색 수준**: `history:open`, `history:close`, `history:next`, `history:prev`, `history:select`

### KeybindingBlockSchema

단일 컨텍스트의 바인딩 블록 구조를 정의하는 Zod 스키마:

```typescript
// 각 블록은 컨텍스트 + 바인딩 배열을 포함합니다
{
  context: z.enum(KEYBINDING_CONTEXTS),
  bindings: z.array(z.object({
    action: z.enum(KEYBINDING_ACTIONS),
    key: z.string(),        // 예: "ctrl+k"
    when?: z.string(),      // 조건 표현식
    unbound?: z.boolean(),  // 이 액션 바인딩 해제
  }))
}
```

### KeybindingsSchema

블록 배열인 완전한 `keybindings.json` 스키마:

```typescript
KeybindingsSchema = z.array(KeybindingBlockSchema)
```

사용자는 `~/.claude/keybindings.json`에서 기본 바인딩을 재정의할 수 있습니다.

---

## 기본 바인딩 (defaultBindings.ts)

### 플랫폼별 처리

- Windows VT 모드(가상 터미널 처리) 감지, 특정 제어 시퀀스의 가용성에 영향을 미침
- macOS/Linux와 Windows 간의 수정자 키 매핑 차이 (Meta vs Alt)

### 컨텍스트별 기본 바인딩

**전역 컨텍스트**:
- `ctrl+c` → `app:interrupt` (현재 작업 인터럽트)
- `ctrl+d` → `app:exit` (애플리케이션 종료)
- `ctrl+l` → `app:clear` (화면 지우기)

**채팅 컨텍스트**:
- `Enter` → `chat:submit`
- `shift+Enter` → `chat:newline`
- `Up` → `chat:historyUp`
- `Down` → `chat:historyDown`

**자동완성 컨텍스트**:
- `Tab` → `autocomplete:accept`
- `Escape` → `autocomplete:dismiss`
- `Up/Down` → `autocomplete:prev/next`

**확인 컨텍스트**:
- `y` → `confirm:yes`
- `n` → `confirm:no`
- `a` → `confirm:always`
- `e` → `confirm:explain`

**탭 컨텍스트**:
- `ctrl+tab` / `ctrl+shift+tab` → 탭 전환

**트랜스크립트/히스토리 검색 및 기타 컨텍스트**도 모두 기본 바인딩이 있습니다.

---

## 파서 (parser.ts)

### parseKeystroke

```typescript
parseKeystroke("ctrl+shift+k") → ParsedKeystroke
```

키 조합의 문자열 표현을 구조화된 객체로 파싱합니다:
- 수정자 키 추출: ctrl, shift, alt, meta
- 기본 키 이름 추출
- 대소문자 및 별칭 정규화

### parseChord

```typescript
parseChord("ctrl+k ctrl+s") → Chord
```

공백으로 구분된 여러 키 입력으로 구성된 다중 키 시퀀스(코드)를 파싱합니다:
- 순서가 있는 `ParsedKeystroke[]`를 포함하는 `Chord` 타입을 반환합니다
- 임의 길이의 코드 시퀀스를 지원합니다

### keystrokeToString / keystrokeToDisplayString

```typescript
keystrokeToString(keystroke)              → "ctrl+shift+k"  // 정규화된 문자열
keystrokeToDisplayString(keystroke, platform) → "Ctrl+Shift+K"  // 플랫폼 인식 표시 문자열
```

- `keystrokeToString`: 정규화된 내부 표현을 출력합니다
- `keystrokeToDisplayString`: 플랫폼 기반의 사용자 친화적 표시 문자열을 출력합니다 (macOS는 ⌘⇧⌥ 같은 기호 사용)

### parseBindings

```typescript
parseBindings(blocks: KeybindingBlock[]) → ParsedBinding[]
```

중첩된 바인딩 블록 구조를 1차원 `ParsedBinding[]` 배열로 평탄화합니다. 각 항목에는 이미 파싱된 컨텍스트, 액션, 키 입력/코드가 포함됩니다.

---

## 매처 (match.ts)

### getKeyName

```typescript
getKeyName(input: string, key: Key) → string
```

Ink의 input/key 이벤트를 통합된 키 이름 문자열로 정규화하며, 특수 키(방향키, 함수키, 스페이스 등)의 매핑을 처리합니다.

### matchesKeystroke

```typescript
matchesKeystroke(input: string, key: Key, target: ParsedKeystroke) → boolean
```

현재 키 이벤트가 대상 키 입력과 일치하는지 결정합니다:
- 수정자 키 상태 비교 (ctrl, shift, alt, meta)
- 기본 키 이름 비교
- 대소문자 및 플랫폼 차이 처리

### matchesBinding

```typescript
matchesBinding(input: string, key: Key, binding: ParsedBinding) → boolean
```

현재 키 이벤트가 바인딩의 첫 번째 키 입력과 일치하는지 결정합니다 (코드의 경우 시퀀스의 첫 단계만 매칭).

---

## 리졸버 (resolver.ts)

### ResolveResult

단일 단계 해석 결과:

```typescript
type ResolveResult = 'match' | 'none' | 'unbound'
```

- `match`: 매칭 바인딩 발견됨
- `none`: 매치 없음
- `unbound`: 매치를 찾았지만 사용자에 의해 명시적으로 언바인딩됨

### ChordResolveResult

코드 인식 해석 결과:

```typescript
type ChordResolveResult = 'match' | 'none' | 'unbound' | 'chord_started' | 'chord_cancelled'
```

- `chord_started`: 코드 접두사가 매칭됨; 시스템이 이후 키 입력 대기 상태에 진입
- `chord_cancelled`: 코드 시퀀스가 중간에 취소됨 (타임아웃 또는 매칭되는 이후 키 없음)

### resolveKey

```typescript
resolveKey(input, key, context, bindings) → { result: ResolveResult, action?: string }
```

단일 키 입력 해석 — 주어진 컨텍스트의 바인딩 세트에서 매치를 찾습니다.

### resolveKeyWithChordState

```typescript
resolveKeyWithChordState(input, key, context, bindings, chordState) → {
  result: ChordResolveResult,
  action?: string,
  newChordState: ChordState
}
```

코드 인식 키 입력 해석:
- 대기 중인 코드 상태를 유지합니다
- 현재 키 입력이 코드 접두사와 매칭되면 `chord_started`를 반환하고 상태를 업데이트합니다
- 완전한 코드가 매칭되면 `match`를 반환합니다
- 대기 중인 코드 상태에서 키 입력이 매칭되지 않으면 `chord_cancelled`를 반환하고 초기화합니다

### keystrokesEqual

```typescript
keystrokesEqual(a: ParsedKeystroke, b: ParsedKeystroke) → boolean
```

두 키 입력이 동등한지 비교하며, alt와 meta를 동등하게 처리합니다(alt/meta 축소).

---

## React 훅(Hooks)

### useKeybinding

```typescript
useKeybinding(action: string, handler: () => void, options?: {
  context?: string,
  enabled?: boolean,
  priority?: number,
})
```

단일 액션을 핸들러에 바인딩합니다:
- 코드 시퀀스 상태를 자동으로 관리합니다
- 조건부 활성화/비활성화를 지원합니다
- 컴포넌트 언마운트 시 자동으로 정리합니다
- 우선순위를 지원합니다 (높은 우선순위가 낮은 우선순위를 재정의)

### useKeybindings

```typescript
useKeybindings(handlers: Record<string, () => void>, options?: {
  context?: string,
  enabled?: boolean,
})
```

여러 액션을 일괄 바인딩합니다:
- 액션 → 핸들러 매핑 객체를 받습니다
- 내부적으로 `useKeybinding`의 로직을 재사용합니다
- 여러 키보드 단축키에 응답해야 하는 컴포넌트에 적합합니다

---

## Vim 모드 (src/vim/, 5개 파일)

#### 왜 Vim 모드를 통합하는가?

Vim 사용자들은 근육 기억을 형성해 두었습니다: 모든 텍스트 편집 컨텍스트에서 Vim 키바인딩을 기대합니다. Vim 모드를 제공하지 않으면 이 사용자들이 Claude Code의 입력 필드에서 익숙한 편집 방식(예: `ciw`로 현재 단어 변경, `dd`로 줄 삭제)을 사용할 수 없어 경험이 크게 저하됩니다. 5개 파일의 모듈 구조(motions / operators / textObjects / transitions / types)는 Vim 자체의 개념 모델을 반영합니다: 오퍼레이터 + 모션 = 명령. `RecordedChange`의 판별 유니온 설계는 도트 반복(`.` 명령)을 지원하며, `MAX_VIM_COUNT = 10000`은 실수로 입력한 큰 반복 횟수로 인터페이스가 멈추는 것을 방지합니다.

### VimState

```typescript
type VimState = 'INSERT' | 'NORMAL'
```

Vim의 두 가지 주요 모드입니다. INSERT 모드에서 키 입력은 직접 텍스트를 입력합니다. NORMAL 모드에서 키 입력은 Vim 명령을 트리거합니다.

### CommandState

10개의 명령 파싱 상태를 정의합니다:

- `idle` - 명령 입력 대기
- `count` - 숫자 접두사 입력 중
- `operator` - 모션 또는 텍스트 객체 대기 중 (예: `d` 이후 `w` 대기)
- `find` - f/F/t/T의 대상 문자 대기
- `replace` - r의 교체 문자 대기
- `register` - `"` 이후 레지스터 이름 대기
- `mark` - `m` 이후 마크 이름 대기
- `goto_mark` - `'` 또는 `` ` `` 이후 마크 이름 대기
- `z_command` - `z` 이후 하위 명령 대기
- `g_command` - `g` 이후 하위 명령 대기

### PersistentState

명령 간에 지속되는 상태:

```typescript
interface PersistentState {
  lastChange: RecordedChange | null   // 도트 반복(.)에 사용
  lastFind: { char: string, direction: 'forward' | 'backward', inclusive: boolean } | null
  register: Record<string, string>     // 레지스터 내용
}
```

### RecordedChange

도트 반복에 사용되는 변경 기록으로, 판별 유니온으로 설계됩니다:

```typescript
type RecordedChange =
  | { type: 'insert', text: string }
  | { type: 'delete', range: Range, register?: string }
  | { type: 'replace', range: Range, text: string }
  | { type: 'operator', operator: string, motion: string, count?: number }
  // ... 기타 변형
```

### MAX_VIM_COUNT

```typescript
const MAX_VIM_COUNT = 10000
```

숫자 접두사의 허용 최대 값으로, 실수로 큰 반복 횟수가 입력되는 것을 방지합니다.

### 모듈 분류 (5개 파일)

1. **motions.ts** - 커서 이동 명령
   - 문자 이동: h, l
   - 단어 이동: w, W, b, B, e, E
   - 인라인 이동: 0, ^, $, f, F, t, T
   - 줄 이동: j, k, gg, G
   - 검색 이동: /, ?, n, N

2. **operators.ts** - 오퍼레이터 명령
   - d (삭제), c (변경), y (복사)
   - 오퍼레이터 + 모션 조합
   - 줄 작업: dd, cc, yy
   - 대문자 변형: D, C, Y

3. **textObjects.ts** - 텍스트 객체
   - 내부/외부: iw, aw, iW, aW
   - 괄호 객체: i(, a(, i[, a[, i{, a{
   - 따옴표 객체: i", a", i', a'
   - 태그 객체: it, at

4. **transitions.ts** - 모드 전환
   - NORMAL → INSERT: i, I, a, A, o, O
   - INSERT → NORMAL: Escape
   - 명령 상태 전환 로직
   - 카운트 접두사 처리

5. **index.ts / types.ts** - 진입점 및 타입 정의
   - VimState, CommandState, PersistentState 타입 내보내기
   - Vim 엔진 메인 루프
   - 키 이벤트를 Vim 명령으로 라우팅

---

## 엔지니어링 실천 가이드

### 커스텀 키바인딩

**`~/.claude/keybindings.json`에서 기본 바인딩 재정의:**

```json
[
  {
    "context": "Chat",
    "bindings": [
      { "action": "chat:submit", "key": "ctrl+enter" },
      { "action": "chat:newline", "key": "enter" }
    ]
  },
  {
    "context": "Global",
    "bindings": [
      { "action": "app:help", "key": "ctrl+k ctrl+h" }
    ]
  }
]
```

**지원되는 기능:**
- **단일 키 바인딩**: `ctrl+c`, `shift+enter`, `tab` 등
- **코드 조합**: 공백으로 구분된 다중 키 시퀀스 (예: `ctrl+k ctrl+s`)
- **언바인딩 액션**: `"unbound": true`를 설정하여 기본 바인딩 제거
- **조건부 바인딩**: `"when"` 조건 표현식으로 바인딩이 적용되는 시점 제어

**병합 전략**: 사용자 바인딩은 기본 바인딩 이후에 추가됩니다. 동일한 컨텍스트 + 액션을 가진 사용자 바인딩이 기본 바인딩을 재정의합니다. 소스: `loadUserBindings.ts:197`: `mergedBindings = [...defaultBindings, ...userParsed]`

### 반응 없는 키 디버깅

**문제 해결 단계:**

1. **현재 컨텍스트 모드 확인**: 17개의 컨텍스트(Global, Chat, Autocomplete, Task, Confirmation, Tabs, Transcript, HistorySearch, Vim Normal, Vim Insert 등)가 활성 바인딩을 결정합니다
2. **더 높은 우선순위 바인딩에 의해 재정의되었는지 확인**: `resolveKey()`는 주어진 컨텍스트의 바인딩 세트에서 매치를 찾습니다; 나중에 등록된 바인딩이 우선순위를 가집니다
3. **코드 상태 확인**: 코드 시퀀스가 진행 중(`chord_started` 상태)이라면, 시퀀스가 완료되거나 타임아웃으로 취소될 때까지 일반 키 처리가 일시 중지됩니다
4. **Vim 모드 확인**: Vim 모드는 독립적인 키 처리 로직을 가집니다. INSERT 모드에서는 키 입력이 직접 텍스트를 입력하고; NORMAL 모드에서는 Vim 명령을 트리거합니다
5. **keybindings.json 형식 확인**:
   - 배열 형식이어야 합니다
   - 각 블록에는 `context`와 `bindings` 필드가 필요합니다
   - 유효하지 않은 형식은 경고를 트리거하고 기본 바인딩으로 대체됩니다
6. **플랫폼 차이 확인**:
   - Windows VT 모드는 특정 제어 시퀀스에 영향을 미칩니다
   - macOS/Linux와 Windows 간의 수정자 키 매핑 차이 (Meta vs Alt)
   - `keystrokesEqual()`은 alt와 meta를 동등하게 처리합니다

**소스 로깅**: `loadUserBindings.ts`는 `logForDebugging('[keybindings] ...')`를 통해 로드 및 검증 로그를 출력합니다.

**하드코딩된 키**: `useExitOnCtrlCD.ts`의 주석은 Ctrl+C와 Ctrl+D가 하드코딩된 종료 키로, `keybindings.json`을 통해 리바인딩할 수 없다고 설명합니다.

### 새 키바인딩 액션 추가

**단계별 체크리스트:**

1. **액션 정의**: `schema.ts`의 `KEYBINDING_ACTIONS`에 새 액션 식별자 추가 (`context:action` 명명 규칙 따름)
2. **기본 바인딩 등록**: `defaultBindings.ts`에서 새 액션에 기본 키 추가
3. **컨텍스트 조건 추가**: 특정 컨텍스트에서만 액션이 적용된다면, 올바른 `KEYBINDING_CONTEXTS` 아래에 등록되었는지 확인
4. **핸들러 구현**: `useKeybinding(action, handler)` 또는 `useKeybindings(handlers)` React 훅(Hook)을 사용하여 처리 로직 바인딩
5. **코드 상태 처리**: 액션이 코드 시퀀스를 사용한다면, `resolveKeyWithChordState()`가 자동으로 대기 상태를 관리합니다

**React 훅(Hook) 사용 예시:**
```typescript
// 단일 액션
useKeybinding('my-context:my-action', () => { /* 핸들러 */ }, {
  context: 'MyContext',
  enabled: true,
})

// 일괄 바인딩
useKeybindings({
  'my-context:action1': () => { /* 핸들러 1 */ },
  'my-context:action2': () => { /* 핸들러 2 */ },
}, { context: 'MyContext' })
```

### Vim 모드 확장

**Vim 엔진 모듈 분류:**
- `motions.ts` — 커서 이동 명령 (h/l/w/b/e/0/$/f/F/t/T/gg/G 등)
- `operators.ts` — 오퍼레이터 명령 (d/c/y 및 조합 dd/cc/yy/D/C/Y)
- `textObjects.ts` — 텍스트 객체 (iw/aw/i(/a(/i"/a" 등)
- `transitions.ts` — 모드 전환 (NORMAL↔INSERT) 및 명령 상태 전환
- `types.ts` — 타입 정의 및 Vim 엔진 메인 루프

**도트 반복 지원**: `RecordedChange`는 판별 유니온을 사용하여 변경 사항을 기록함으로써 `.` 명령으로 반복할 수 있습니다.

### 흔한 함정

| 함정 | 세부사항 | 해결책 |
|------|---------|--------|
| 터미널 에뮬레이터가 특정 키 조합을 가로챌 수 있음 | Ctrl+S (터미널 흐름 제어), Ctrl+Z (일시 중단) 등은 터미널에 의해 가로채여 Claude Code에 도달하지 않음 | 코드 시퀀스로 우회하거나 터미널 설정에서 해당 단축키 비활성화 |
| Vim 모드는 독립적인 키 처리 로직을 가짐 | Vim NORMAL 모드에서 키 입력은 기본 키바인딩 대신 Vim 명령을 트리거함 | 10개의 Vim 명령 상태(idle/count/operator/find/replace 등)가 독립적으로 관리됨 |
| `MAX_VIM_COUNT = 10000` | 실수로 지나치게 큰 숫자 접두사 반복 횟수 입력을 방지 | 이 값을 초과하는 숫자 접두사는 잘립니다 |
| 코드 시퀀스 중간에 취소됨 | 코드 시퀀스 중 매칭되는 이후 키가 없는 키 입력은 `chord_cancelled`를 반환하고 초기화됨 | 사용자는 처음부터 전체 코드 시퀀스를 다시 시작해야 합니다 |
| keybindings.json 형식 오류 | 유효하지 않은 JSON 또는 스키마에 맞지 않는 구조 | 자동으로 기본 바인딩으로 대체되고 경고 로그 출력 |
| 플랫폼 수정자 키 차이 | macOS는 Meta(Command)를 사용하고, Windows/Linux는 Alt를 사용 | `keystrokesEqual()`은 alt와 meta를 동등하게 처리 |


---

[← 세션 관리](../26-会话管理/session-management-ko.md) | [목차](../README_KO.md) | [Vim 모드 →](../28-Vim模式/vim-mode-ko.md)
