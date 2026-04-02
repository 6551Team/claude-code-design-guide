# Cost Tracking Architecture Document

> Complete technical reference for the cost and usage tracking system in Claude Code v2.1.88

---

## Core Tracking (cost-tracker.ts)

### Exported Functions

#### Aggregate Queries
| Function | Return Value |
|------|--------|
| `getTotalCost()` | Total cost (USD) |
| `getTotalDuration()` | Total duration |
| `getTotalAPIDuration()` | Total API call time |
| `getTotalAPIDurationWithoutRetries()` | Total API call time (excluding retries) |

#### Token Statistics
| Function | Return Value |
|------|--------|
| `getTotalInputTokens()` | Total input token count |
| `getTotalOutputTokens()` | Total output token count |
| `getTotalCacheReadInputTokens()` | Cache read input token count |
| `getTotalCacheCreationInputTokens()` | Cache creation input token count |

#### Code Change Statistics
| Function | Return Value |
|------|--------|
| `addToTotalLinesChanged()` | Add a code line change record |
| `getTotalLinesAdded()` | Total lines added |
| `getTotalLinesRemoved()` | Total lines removed |

#### Other Statistics
| Function | Return Value |
|------|--------|
| `getTotalWebSearchRequests()` | Total number of web search requests |

#### Model Usage
| Function | Return Value |
|------|--------|
| `getModelUsage()` | Usage data for all models |
| `getUsageForModel()` | Usage data for a specified model |

#### Formatting
| Function | Return Value |
|------|--------|
| `formatCost(cost)` | Format a dollar amount for display |
| `formatTotalCost()` | Model usage breakdown display (listed by model) |
| `hasUnknownModelCost()` | Detect whether there are costs from unknown models |

#### Accumulation and Persistence
| Function | Purpose |
|------|------|
| `addToTotalSessionCost()` | Accumulate usage by model (input/output/cache tokens, cost, duration) |
| `getStoredSessionCosts()` | Read saved session costs from project configuration |
| `restoreCostStateForSession()` | Restore cost state (only when sessionId matches) |
| `saveCurrentSessionCosts()` | Persist current session costs to project configuration |

### Data Structures

```typescript
StoredCostState = {
  totalCostUSD: number,
  totalAPIDuration: number,
  totalAPIDurationWithoutRetries: number,
  totalToolDuration: number,
  totalLinesAdded: number,
  totalLinesRemoved: number,
  lastDuration: number,
  modelUsage: Record<string, {
    input_tokens: number,
    output_tokens: number,
    cache_read: number,
    cache_creation: number,
    cost: number,
    duration: number
  }>
}
```

### Design Philosophy: Why Per-Model Accumulation Instead of a Single Global Counter?

In `cost-tracker.ts`, `modelUsage` uses a `Record<string, ModelUsage>` structure that stores usage data indexed by model name. This design decision is based on:

1. **Different models have different prices** -- The per-input/output token prices for Opus and Sonnet differ significantly; a single counter cannot accurately calculate costs.
2. **Differentiated billing for cache tokens** -- The prices for cache read and cache creation differ from regular tokens; the per-model structure allows each token type to be tracked independently.
3. **Model-switching scenarios** -- A session may use multiple models (primary model + advisor model). The `addToTotalSessionCost()` function (lines 278-284) specifies the `model` parameter on every call, ensuring costs are attributed to the correct model.

### Design Philosophy: Why Are Session Costs Separate from Total Costs?

Users have two distinct concerns:
- **"How much did this conversation cost?"** -- Session costs help users evaluate the cost efficiency of a single task.
- **"How much have I spent in total?"** -- Total costs help users with budget management and cost planning.

The source code persists session costs to the project configuration via `saveCurrentSessionCosts()`, and `restoreCostStateForSession()` only restores costs matching the given `sessionId` when resuming a session (avoiding confusion between cost data from different sessions).

### Engineering Practices

**Troubleshooting inaccurate costs:**
1. Check whether the model price table is up to date -- `formatModelUsage()` uses `getCanonicalName(model)` to normalize different model IDs before aggregating them for display; confirm that model mappings are correct.
2. Check whether cache tokens are being correctly deducted -- `cacheReadInputTokens` and `cacheCreationInputTokens` are tracked separately; the price per token for a cache hit is lower than for a regular token.
3. Check the return value of `hasUnknownModelCost()` -- if there are costs from unknown models, the price table is missing the corresponding entry.
4. Advisor model costs are accumulated via a separate `addToTotalSessionCost()` call (line 316); confirm that advisor costs are being correctly included.

**Adding cost tracking for a new model:**
- Add the model entry to the price table (model name → USD unit price per input/output/cache token).
- Ensure `getCanonicalName()` can correctly map the new model ID to a display name.
- The `contextWindow` and `maxOutputTokens` information is retrieved via `getContextWindowForModel()` and `getModelMaxOutputTokens()`; corresponding configuration entries must also be added.

---

## React Hook (costHook.ts)

### useCostSummary(getFpsMetrics?)

Registers a process-exit handler that, when the session ends:
1. Displays the cost summary (only for users with billing access).
2. Saves the session costs.
3. Records FPS metrics (if a `getFpsMetrics` function is provided).

---

## /cost Command

### Conditional Display Logic
- Claude.ai subscribers: cost information is hidden (because it is included in the subscription).
- ANT users are an exception: costs are displayed even for subscribers.
- Detects `currentLimits.isUsingOverage`: displays an overage usage message.

### Output
Calls `formatTotalCost()` to output cost details broken down by model.

---

## /stats Command

### Implementation
Renders the `<Stats>` component (imported from `components/Stats`), displaying usage statistics and activity information.

---

## /usage Command

### Implementation
Renders the `<Settings defaultTab="Usage">` component.

### Availability
Available to claude-ai subscribers only.

---

## /extra-usage Command

### Functionality
Manages extra usage quotas, supporting team/enterprise administrators granting overage credits.

### Pre-checks
- Verifies team/enterprise administrator eligibility.
- Retrieves and validates overage credits.

### Execution Flow
- **Interactive**: Opens a browser for the operation.
- **Non-interactive**: Handled directly from the command line.

### Gate Conditions
- `isExtraUsageAllowed()`: Checks whether extra usage is permitted.
- `isOverageProvisioningAllowed()`: Checks whether overage provisioning is permitted.


---

[← File Persistence](../41-文件持久化/file-persistence-en.md) | [Index](../README_EN.md) | [Shell Toolchain →](../43-Shell工具链/shell-toolchain-en.md)
