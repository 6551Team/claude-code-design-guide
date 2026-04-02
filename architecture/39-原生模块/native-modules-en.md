# Native Modules and Core Tools

Claude Code's native module layer provides core capabilities such as file indexing, color diff calculation, and layout engine, as well as infrastructure for output styles, migrations, and model selection.

### Design Philosophy: Why FFI Bridging Instead of Pure JS?

The native module layer (`native-ts/`) exists as a performance-driven design decision:

1. **Color diff calculation** -- Color Diff involves heavy pixel-level math operations (color space conversion, matrix multiplication), and pure JS floating-point performance is far inferior to compiled languages
2. **File indexing** -- FileIndex needs to perform fuzzy matching and sorting over tens of thousands of file paths. The source comments explicitly state this is a "Pure-TypeScript port of vendor/file-index-src (Rust NAPI module)", with the original version based on Rust's nucleo library (`native-ts/file-index/index.ts` lines 2-4)
3. **Yoga layout** -- Ink terminal rendering requires flex layout computation. Yoga is Facebook's C++ layout engine, and FFI bindings provide near-native layout performance

### Design Philosophy: Why Are Native Modules Optional?

![Native Module Fallback Strategy](../diagrams/native-module-fallback-en.svg)

Not all platforms can compile native code (e.g., some CI environments, restricted containers, Windows ARM, etc.). Graceful degradation to a pure JS implementation guarantees availability:
- FileIndex already has a complete pure TypeScript implementation (371 lines), with a scoring algorithm that precisely emulates nucleo's behavior
- This "optional native" pattern is a best practice in the Node.js ecosystem — npm packages like `sharp` and `better-sqlite3` use a similar strategy

### Engineering Practices

**Debugging native module compilation failures**:
- Check the `node-gyp` environment: requires Python 3.x and a C++ compiler (Windows requires Visual Studio Build Tools)
- Confirm Node.js version compatibility: native modules are typically compiled for a specific Node ABI version
- If using the Bun runtime, check the `bun:ffi` support status

**Checklist for adding a new native module**:
1. Create a new module directory under `native-ts/`
2. Provide a pure JS fallback implementation simultaneously (this is a hard requirement, not optional)
3. Implement auto-detection in the loading layer: prefer the native version, fall back to the JS version on failure
4. Ensure the pure JS version's API signatures are exactly identical to the native version

---

## File Index (file-index/index.ts, 371 lines)

### Overview

A pure TypeScript implementation of a fuzzy file search engine, replacing the earlier Rust nucleo bindings and eliminating the native dependency.

### FileIndex Class

#### loadFromFileList

```typescript
loadFromFileList(files: string[]) → void
```

Synchronously loads a file list:
- Deduplicates file paths
- Builds a search index for each file (bitmap + normalized path)

#### loadFromFileListAsync

```typescript
loadFromFileListAsync(files: string[]) → Promise<void>
```

Asynchronously loads a file list:
- Processes in chunks, yielding to the event loop between chunks
- Avoids blocking UI rendering for large file lists
- Suitable for large projects with tens of thousands of files

#### search

```typescript
search(query: string, limit: number) → SearchResult[]
```

Performs a fuzzy search and returns top-K results:
- Normalizes the query
- Computes similarity scores for all indexed files
- Uses a top-K selection algorithm (avoids a full sort)
- Returns results sorted by score in descending order

### Scoring Algorithm

#### Base Score

```typescript
SCORE_MATCH  // base match score
```

Each matched character receives a base score.

#### Bonus Scores

- **Boundary bonus**: match occurs at a word boundary (after a path separator, underscore, or hyphen)
- **CamelCase bonus**: match occurs at an uppercase letter in camelCase naming
- **Consecutive bonus**: consecutively matched characters receive an increasing bonus

#### Bitmap Optimization

```
26-bit mask → O(1) letter presence detection
```

Each file maintains a 26-bit bitmask recording which letters appear in the file path. At query time, the bitmap is checked first; if a letter in the query is not in the file's bitmap, that file is immediately skipped, achieving O(1) fast rejection.

#### Test File Penalty

```typescript
// Test file scores are divided by 1.05x (denominator, lowering rank)
```

File paths containing patterns like test/spec/mock receive a slight ranking penalty (score divided by 1.05), causing non-test files to rank higher under equal match quality.

#### Top-K Selection

Uses a heap or partial sort algorithm to select the top K results, with time complexity O(n log k) rather than the O(n log n) of a full sort.

---

## Color Diff (color-diff/index.ts, ~10KB)

### Overview

A pure TypeScript implementation of a diff calculation engine.

### Core Features

- Color matrix computation for quantifying visual differences
- Supports diff metrics across multiple color spaces
- Provides the underlying computation for terminal diff display

---

## Yoga Layout (yoga-layout/index.ts, 27KB)

### Overview

Bindings for the Yoga layout engine, providing flex layout support for Ink terminal rendering.

### Enum Definitions

```typescript
// Direction
enum Direction { Inherit, LTR, RTL }

// Main-axis alignment
enum Justify { FlexStart, Center, FlexEnd, SpaceBetween, SpaceAround, SpaceEvenly }

// Cross-axis alignment
enum Align { Auto, FlexStart, Center, FlexEnd, Stretch, Baseline, SpaceBetween, SpaceAround }

// Display mode
enum Display { Flex, None }

// Wrapping
enum Wrap { NoWrap, Wrap, WrapReverse }

// Overflow handling
enum Overflow { Visible, Hidden, Scroll }

// Positioning
enum Position { Static, Relative, Absolute }
```

### Usage

- Used by the custom Ink renderer
- Implements flexbox-based terminal UI layout
- Supports nested containers, flexible sizing, alignment, and wrapping

---

## Output Styles (src/outputStyles/)

### loadOutputStylesDir.ts

A memoized loader that searches for and loads output style definitions.

#### Search Paths

```
Project-level: .claude/output-styles/
User-level:    ~/.claude/output-styles/
```

Searches for style files from project-level to user-level in priority order.

#### File Format

Markdown files with frontmatter metadata:

```markdown
---
name: "Custom Style"
description: "A custom output style"
keepCodingInstructions: true
---

Your prompt instructions here...
```

#### OutputStyleConfig Fields

```typescript
interface OutputStyleConfig {
  name: string                      // style name
  description: string               // style description
  prompt: string                    // content injected into the system prompt
  source: 'project' | 'user'       // source (project-level or user-level)
  keepCodingInstructions: boolean   // whether to retain default coding instructions
}
```

#### clearOutputStyleCaches

```typescript
clearOutputStyleCaches() → void
```

Clears the memoized cache, forcing style files to be reloaded on the next call.

---

## Migrations (src/migrations/, 11 files)

Handles version migrations for configuration and settings, ensuring smooth upgrades from older configurations to newer ones.

### Migration List

| Migration Function | Description |
|-------------------|-------------|
| `migrateAutoUpdatesToSettings` | Migrates auto-update configuration to the unified settings system |
| `migrateBypassPermissionsAcceptedToSettings` | Migrates permission bypass flags to settings |
| `migrateEnableAllProjectMcpServersToSettings` | Migrates project MCP server enable configuration to settings |
| `migrateFennecToOpus` | Migrates Fennec (internal codename) model references to Opus |
| `migrateLegacyOpusToCurrent` | Migrates legacy Opus model IDs to the current version |
| `migrateOpusToOpus1m` | Migrates Opus to the Opus 1M context version |
| `migrateReplBridgeEnabledToRemoteControlAtStartup` | Migrates REPL Bridge configuration to the remote control at startup configuration |
| `migrateSonnet1mToSonnet45` | Migrates Sonnet 1M to Sonnet 4.5 |
| `migrateSonnet45ToSonnet46` | Migrates Sonnet 4.5 to Sonnet 4.6 |
| `resetAutoModeOptInForDefaultOffer` | Resets the opt-in status for auto mode |
| `resetProToOpusDefault` | Resets the default model for Pro users to Opus |

Each migration function:
- Checks whether migration is needed (idempotent)
- Executes the migration logic
- Records the migration completion status to avoid re-execution

---

## Model Selection (src/utils/model/)

### getMainLoopModel

```typescript
getMainLoopModel() → string
```

Determines the model used by the main loop in priority order:

```
1. override (code-level forced override)
2. CLI flag (--model argument)
3. env var (CLAUDE_MODEL environment variable)
4. settings (model setting in user settings)
5. default (default model)
```

### MODEL_ALIASES

```typescript
const MODEL_ALIASES = [
  'sonnet',      // → claude-sonnet-4-6
  'opus',        // → claude-opus-4-6
  'haiku',       // → claude-haiku
  'best',        // → current best model
  'sonnet[1m]',  // → claude-sonnet-4-6 with 1M context
  'opus[1m]',    // → claude-opus-4-6 with 1M context
  'opusplan',    // → opus with planning mode
]
```

Users can use aliases to simplify model specification.

### APIProvider

```typescript
type APIProvider = 'firstParty' | 'bedrock' | 'vertex' | 'foundry'
```

- `firstParty`: Anthropic direct API
- `bedrock`: AWS Bedrock
- `vertex`: Google Cloud Vertex AI
- `foundry`: Custom model service

### Deprecated Model Tracking

Maintains a list of deprecated models and their retirement dates:
- Warns users when they attempt to use a deprecated model
- Automatically suggests migration to a replacement model
- Includes retirement date information for timeline display

### 1M Context Access Check

```typescript
checkOpus1mAccess() → Promise<boolean>
checkSonnet1mAccess() → Promise<boolean>
```

Checks whether the current account has access to 1M context versions of models:
- Determined by subscription type and permission level
- Downgrades to the standard context version when access is unavailable

### Model Capabilities

```typescript
// API query endpoint
GET /v1/models

// Local cache
~/.claude/cache/model-capabilities.json
```

- Queries specific capability parameters of models via the API
- Query results are cached to a local file to avoid repeated requests
- Includes context window size, supported feature flags, and other information


---

[← Output Styles](../38-输出样式/output-styles-en.md) | [Index](../README_EN.md) | [Migration System →](../40-迁移系统/migration-system-en.md)
