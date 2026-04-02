# Keybinding System

Claude Code's keybinding system provides a complete framework for defining, parsing, matching, and executing keyboard shortcuts. It supports single keys, modifier combinations, and chord sequences, and integrates a Vim mode.

### Design Philosophy

#### Why 50+ Actions and Chord Support?

Claude Code's core user base is advanced developers — Vim/Emacs users and heavy terminal users. These users expect rich keybindings and chord sequences (such as `ctrl+k ctrl+s`). Without this capability, power users would be driven away. The 50+ action identifiers (`KEYBINDING_ACTIONS`) cover all interaction scenarios from the application level (exit, interrupt, clear screen) down to the component level (autocomplete, tabs, history search). Chord support expands the limited key combination space — the number of single keys and modifier combinations is finite, but chord sequences provide a nearly unlimited namespace.

#### Why Context-Based Matching?

The same key press behaves differently depending on the UI state: the `Up` key scrolls through history in the Chat context, selects the previous completion item in the Autocomplete context, and scrolls up in the Transcript context. The 17 context names (`KEYBINDING_CONTEXTS`) define the scope in which keybindings are active. `resolveKey` looks up matches within a given context, avoiding key conflicts across contexts. The `chord_started`/`chord_cancelled` states in `ChordResolveResult` allow the system to pause normal key processing while a chord sequence is in progress, until the sequence completes or times out and is cancelled.

---

## Schema (schema.ts)

### KEYBINDING_CONTEXTS

Defines 17 context names to distinguish keybinding scopes across different UI states:

- `Global` - Global context, active in any state
- `Chat` - Chat input area
- `Autocomplete` - Autocomplete menu
- `Task` - Task in progress
- `Confirmation` - Confirmation dialog
- `Tabs` - Tab navigation
- `Transcript` - Conversation history browsing
- `HistorySearch` - History search mode
- And other contexts (Vim Normal, Vim Insert, Diff, FileViewer, etc.)

### KEYBINDING_ACTIONS

Defines 50+ action identifiers, using the `context:action` naming convention:

- **App level**: `app:exit`, `app:interrupt`, `app:clear`, `app:help`, `app:toggleVim`
- **Chat level**: `chat:submit`, `chat:newline`, `chat:paste`, `chat:historyUp`, `chat:historyDown`, `chat:cancel`
- **Confirm level**: `confirm:yes`, `confirm:no`, `confirm:always`, `confirm:explain`
- **Autocomplete level**: `autocomplete:accept`, `autocomplete:dismiss`, `autocomplete:next`, `autocomplete:prev`
- **Tabs level**: `tabs:next`, `tabs:prev`, `tabs:close`
- **Transcript level**: `transcript:scrollUp`, `transcript:scrollDown`, `transcript:pageUp`, `transcript:pageDown`, `transcript:top`, `transcript:bottom`
- **HistorySearch level**: `history:open`, `history:close`, `history:next`, `history:prev`, `history:select`

### KeybindingBlockSchema

Zod schema defining the structure of a binding block for a single context:

```typescript
// Each block contains a context + an array of bindings
{
  context: z.enum(KEYBINDING_CONTEXTS),
  bindings: z.array(z.object({
    action: z.enum(KEYBINDING_ACTIONS),
    key: z.string(),        // e.g. "ctrl+k"
    when?: z.string(),      // conditional expression
    unbound?: z.boolean(),  // unbind this action
  }))
}
```

### KeybindingsSchema

The complete `keybindings.json` schema, which is an array of blocks:

```typescript
KeybindingsSchema = z.array(KeybindingBlockSchema)
```

Users can override default bindings in `~/.claude/keybindings.json`.

---

## Default Bindings (defaultBindings.ts)

### Platform-Specific Handling

- Detects Windows VT mode (Virtual Terminal processing), which affects the availability of certain control sequences
- Differences in modifier key mappings between macOS/Linux and Windows (Meta vs Alt)

### Default Bindings per Context

**Global context**:
- `ctrl+c` → `app:interrupt` (interrupt the current operation)
- `ctrl+d` → `app:exit` (exit the application)
- `ctrl+l` → `app:clear` (clear screen)

**Chat context**:
- `Enter` → `chat:submit`
- `shift+Enter` → `chat:newline`
- `Up` → `chat:historyUp`
- `Down` → `chat:historyDown`

**Autocomplete context**:
- `Tab` → `autocomplete:accept`
- `Escape` → `autocomplete:dismiss`
- `Up/Down` → `autocomplete:prev/next`

**Confirmation context**:
- `y` → `confirm:yes`
- `n` → `confirm:no`
- `a` → `confirm:always`
- `e` → `confirm:explain`

**Tabs context**:
- `ctrl+tab` / `ctrl+shift+tab` → tab switching

**Transcript/HistorySearch and other contexts** all have corresponding default bindings.

---

## Parser (parser.ts)

### parseKeystroke

```typescript
parseKeystroke("ctrl+shift+k") → ParsedKeystroke
```

Parses a string representation of a key combination into a structured object:
- Extracts modifier keys: ctrl, shift, alt, meta
- Extracts the primary key name
- Normalizes casing and aliases

### parseChord

```typescript
parseChord("ctrl+k ctrl+s") → Chord
```

Parses a multi-key sequence (chord), composed of multiple keystrokes separated by spaces:
- Returns a `Chord` type containing an ordered `ParsedKeystroke[]`
- Supports chord sequences of arbitrary length

### keystrokeToString / keystrokeToDisplayString

```typescript
keystrokeToString(keystroke)              → "ctrl+shift+k"  // normalized string
keystrokeToDisplayString(keystroke, platform) → "Ctrl+Shift+K"  // platform-aware display string
```

- `keystrokeToString`: outputs a normalized internal representation
- `keystrokeToDisplayString`: outputs a user-friendly display string based on platform (macOS uses symbols such as ⌘⇧⌥)

### parseBindings

```typescript
parseBindings(blocks: KeybindingBlock[]) → ParsedBinding[]
```

Flattens the nested binding block structure into a one-dimensional `ParsedBinding[]` array. Each entry contains the already-parsed context, action, and keystroke/chord.

---

## Matcher (match.ts)

### getKeyName

```typescript
getKeyName(input: string, key: Key) → string
```

Normalizes Ink's input/key events into a unified key name string, handling mappings for special keys (arrow keys, function keys, space, etc.).

### matchesKeystroke

```typescript
matchesKeystroke(input: string, key: Key, target: ParsedKeystroke) → boolean
```

Determines whether the current key event matches a target keystroke:
- Compares modifier key states (ctrl, shift, alt, meta)
- Compares the primary key name
- Handles casing and platform differences

### matchesBinding

```typescript
matchesBinding(input: string, key: Key, binding: ParsedBinding) → boolean
```

Determines whether the current key event matches the first keystroke of a binding (for chords, only matches the first step in the sequence).

---

## Resolver (resolver.ts)

### ResolveResult

Single-step resolution result:

```typescript
type ResolveResult = 'match' | 'none' | 'unbound'
```

- `match`: a matching binding was found
- `none`: no match found
- `unbound`: a match was found but was explicitly unbound by the user

### ChordResolveResult

Chord-aware resolution result:

```typescript
type ChordResolveResult = 'match' | 'none' | 'unbound' | 'chord_started' | 'chord_cancelled'
```

- `chord_started`: a chord prefix was matched; the system enters a waiting state for subsequent key presses
- `chord_cancelled`: the chord sequence was cancelled mid-way (timeout or no matching subsequent key)

### resolveKey

```typescript
resolveKey(input, key, context, bindings) → { result: ResolveResult, action?: string }
```

Single key press resolution — looks for a match in the binding set for the given context.

### resolveKeyWithChordState

```typescript
resolveKeyWithChordState(input, key, context, bindings, chordState) → {
  result: ChordResolveResult,
  action?: string,
  newChordState: ChordState
}
```

Chord-aware key press resolution:
- Maintains a pending chord state
- If the current key press matches a chord prefix, returns `chord_started` and updates the state
- If a complete chord is matched, returns `match`
- If a key press in a pending chord state does not match, returns `chord_cancelled` and resets

### keystrokesEqual

```typescript
keystrokesEqual(a: ParsedKeystroke, b: ParsedKeystroke) → boolean
```

Compares whether two keystrokes are equivalent, treating alt and meta as equivalent (collapse alt/meta).

---

## React Hooks

### useKeybinding

```typescript
useKeybinding(action: string, handler: () => void, options?: {
  context?: string,
  enabled?: boolean,
  priority?: number,
})
```

Binds a single action to a handler:
- Automatically manages chord sequence state
- Supports conditional enable/disable
- Automatically cleans up when the component unmounts
- Supports priority (higher priority overrides lower priority)

### useKeybindings

```typescript
useKeybindings(handlers: Record<string, () => void>, options?: {
  context?: string,
  enabled?: boolean,
})
```

Binds multiple actions in bulk:
- Accepts an action → handler mapping object
- Internally reuses the logic of `useKeybinding`
- Suited for components that need to respond to multiple keyboard shortcuts

---

## Vim Mode (src/vim/, 5 files)

#### Why Integrate Vim Mode?

Vim users have built up muscle memory: they expect Vim keybindings in every text editing context. Not providing a Vim mode means these users cannot use their accustomed editing style in Claude Code's input field (such as `ciw` to change the current word, or `dd` to delete a line) — a significant experience disruption. The module structure of 5 files (motions / operators / textObjects / transitions / types) mirrors Vim's own conceptual model: operator + motion = command. The discriminated union design of `RecordedChange` supports dot-repeat (the `.` command), and `MAX_VIM_COUNT = 10000` prevents an accidentally large repeat count from freezing the interface.

### VimState

```typescript
type VimState = 'INSERT' | 'NORMAL'
```

Vim's two primary modes. In INSERT mode, key presses directly input text. In NORMAL mode, key presses trigger Vim commands.

### CommandState

Defines 10 command parsing states:

- `idle` - waiting for command input
- `count` - entering a numeric prefix
- `operator` - waiting for a motion or text object (e.g. after `d`, waiting for `w`)
- `find` - waiting for the target character of f/F/t/T
- `replace` - waiting for the replacement character of r
- `register` - waiting for the register name after `"`
- `mark` - waiting for the mark name after `m`
- `goto_mark` - waiting for the mark name after `'` or `` ` ``
- `z_command` - waiting for the sub-command after `z`
- `g_command` - waiting for the sub-command after `g`

### PersistentState

State that persists across commands:

```typescript
interface PersistentState {
  lastChange: RecordedChange | null   // used for dot-repeat (.)
  lastFind: { char: string, direction: 'forward' | 'backward', inclusive: boolean } | null
  register: Record<string, string>     // register contents
}
```

### RecordedChange

Change record used for dot-repeat, designed as a discriminated union:

```typescript
type RecordedChange =
  | { type: 'insert', text: string }
  | { type: 'delete', range: Range, register?: string }
  | { type: 'replace', range: Range, text: string }
  | { type: 'operator', operator: string, motion: string, count?: number }
  // ... other variants
```

### MAX_VIM_COUNT

```typescript
const MAX_VIM_COUNT = 10000
```

The maximum allowed value for a numeric prefix, preventing an accidentally large repeat count.

### Module Breakdown (5 files)

1. **motions.ts** - Cursor movement commands
   - Character movement: h, l
   - Word movement: w, W, b, B, e, E
   - In-line movement: 0, ^, $, f, F, t, T
   - Line movement: j, k, gg, G
   - Search movement: /, ?, n, N

2. **operators.ts** - Operator commands
   - d (delete), c (change), y (yank)
   - Operator + motion combinations
   - Line operations: dd, cc, yy
   - Uppercase variants: D, C, Y

3. **textObjects.ts** - Text objects
   - Inner/outer: iw, aw, iW, aW
   - Bracket objects: i(, a(, i[, a[, i{, a{
   - Quote objects: i", a", i', a'
   - Tag objects: it, at

4. **transitions.ts** - Mode transitions
   - NORMAL → INSERT: i, I, a, A, o, O
   - INSERT → NORMAL: Escape
   - Command state transition logic
   - Count prefix handling

5. **index.ts / types.ts** - Entry point and type definitions
   - Exports of VimState, CommandState, PersistentState types
   - Vim engine main loop
   - Routing of key events to Vim commands

---

## Engineering Practice Guide

### Custom Keybindings

**Override default bindings in `~/.claude/keybindings.json`:**

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

**Supported features:**
- **Single key bindings**: `ctrl+c`, `shift+enter`, `tab`, etc.
- **Chord combinations**: multi-key sequences separated by spaces (e.g. `ctrl+k ctrl+s`)
- **Unbinding actions**: set `"unbound": true` to remove a default binding
- **Conditional bindings**: `"when"` conditional expression controls when the binding takes effect

**Merge strategy**: User bindings are appended after the default bindings. User bindings with the same context + action override the default binding. Source: `loadUserBindings.ts:197`: `mergedBindings = [...defaultBindings, ...userParsed]`

### Debugging Unresponsive Keys

**Troubleshooting steps:**

1. **Check the current context mode**: The 17 contexts (Global, Chat, Autocomplete, Task, Confirmation, Tabs, Transcript, HistorySearch, Vim Normal, Vim Insert, etc.) determine which bindings are active
2. **Check whether overridden by a higher-priority binding**: `resolveKey()` searches for a match in the binding set for a given context; bindings registered later take priority
3. **Check chord state**: If a chord sequence is in progress (`chord_started` state), normal key processing is paused until the sequence completes or times out and is cancelled
4. **Check Vim mode**: Vim mode has its own independent key handling logic. In INSERT mode, key presses directly input text; in NORMAL mode, they trigger Vim commands
5. **Check keybindings.json format**:
   - Must be an array format
   - Each block requires `context` and `bindings` fields
   - An invalid format triggers a warning and falls back to the default bindings
6. **Check platform differences**:
   - Windows VT mode affects certain control sequences
   - Differences in modifier key mappings between macOS/Linux and Windows (Meta vs Alt)
   - `keystrokesEqual()` treats alt and meta as equivalent

**Source logging**: `loadUserBindings.ts` outputs load and validation logs via `logForDebugging('[keybindings] ...')`.

**Hard-coded keys**: The comment in `useExitOnCtrlCD.ts` notes that Ctrl+C and Ctrl+D are hard-coded exit keys and cannot be rebound through `keybindings.json`.

### Adding New Keybinding Actions

**Step-by-step checklist:**

1. **Define the action**: Add the new action identifier to `KEYBINDING_ACTIONS` in `schema.ts` (following the `context:action` naming convention)
2. **Register a default binding**: Add a default key for the new action in `defaultBindings.ts`
3. **Add context conditions**: If the action only takes effect in a specific context, ensure it is registered under the correct `KEYBINDING_CONTEXTS`
4. **Implement the handler**: Use the `useKeybinding(action, handler)` or `useKeybindings(handlers)` React hook to bind the handling logic
5. **Handle chord state**: If the action uses a chord sequence, `resolveKeyWithChordState()` automatically manages the pending state

**React Hook usage examples:**
```typescript
// Single action
useKeybinding('my-context:my-action', () => { /* handler */ }, {
  context: 'MyContext',
  enabled: true,
})

// Bulk binding
useKeybindings({
  'my-context:action1': () => { /* handler 1 */ },
  'my-context:action2': () => { /* handler 2 */ },
}, { context: 'MyContext' })
```

### Vim Mode Extension

**Vim engine module breakdown:**
- `motions.ts` — cursor movement commands (h/l/w/b/e/0/$/f/F/t/T/gg/G, etc.)
- `operators.ts` — operator commands (d/c/y and combinations dd/cc/yy/D/C/Y)
- `textObjects.ts` — text objects (iw/aw/i(/a(/i"/a", etc.)
- `transitions.ts` — mode transitions (NORMAL↔INSERT) and command state transitions
- `types.ts` — type definitions and the Vim engine main loop

**Dot-repeat support**: `RecordedChange` uses a discriminated union to record changes, enabling the `.` command to repeat them.

### Common Pitfalls

| Pitfall | Details | Solution |
|---------|---------|----------|
| Terminal emulators may intercept certain key combinations | Ctrl+S (terminal flow control), Ctrl+Z (suspend), etc. are intercepted by the terminal and never reach Claude Code | Use chord sequences to work around this, or disable those shortcuts in terminal settings |
| Vim mode has independent key handling logic | In Vim NORMAL mode, key presses trigger Vim commands rather than default keybindings | 10 Vim command states (idle/count/operator/find/replace, etc.) are managed independently |
| `MAX_VIM_COUNT = 10000` | Prevents accidentally entering an excessively large numeric prefix repeat count | Numeric prefixes exceeding this value are truncated |
| Chord sequence cancelled mid-way | A key press with no matching continuation during a chord sequence returns `chord_cancelled` and resets | The user must restart the entire chord sequence from the beginning |
| keybindings.json format errors | Invalid JSON or a structure that does not conform to the schema | Automatically falls back to default bindings and outputs a warning log |
| Platform modifier key differences | macOS uses Meta (Command), Windows/Linux uses Alt | `keystrokesEqual()` treats alt and meta as equivalent |


---

[← Session Management](../26-会话管理/session-management-en.md) | [Index](../README_EN.md) | [Vim Mode →](../28-Vim模式/vim-mode-en.md)
