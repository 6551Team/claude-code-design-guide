# Feedback & Survey System

Claude Code has a complete built-in system for user feedback collection, surveys, and usage tips. It covers multiple sub-modules including feedback rating, transcript sharing, memory surveys, post-compact surveys, the positive feedback command, the bug feedback command, and the tips system.

---

## Feedback Survey State Machine

The feedback survey component is driven by a finite state machine that manages the complete lifecycle of a user feedback rating.

### State Transitions

```
'closed' → 'open' → 'thanks' → 'transcript_prompt' → 'submitting' → 'submitted'
```

- **closed**: Initial state; the survey is not displayed
- **open**: The survey has appeared and is waiting for user input
- **thanks**: The user has selected a rating; a thank-you message is shown
- **transcript_prompt**: The user is asked whether they are willing to share the transcript
- **submitting**: The transcript data is being submitted
- **submitted**: Submission is complete

#### Why This Design

Feedback collection involves multi-step user interaction (rating → thank you → transcript inquiry → submit → done). The state machine ensures the flow never skips a step or deadlocks. Each of the 6 states has a clearly defined predecessor and successor, and arbitrary jumps are not permitted — for example, you cannot go directly from `closed` to `submitting`, nor can you go from `submitted` back to `open`. This strict linear progression guarantees data-collection integrity: the user must provide a rating before the system can ask about transcript sharing. The state-machine pattern is also easier to maintain and debug than a combination of boolean flags, because at any given moment the system is in exactly one well-defined state.

### Response Types

```typescript
type FeedbackSurveyResponse = 'dismissed' | 'bad' | 'fine' | 'good';
```

### Digit Key Input

Digit key input is handled by the `useDebouncedDigitInput` hook with a debounce delay of **400 ms**:

| Key | Meaning |
|-----|---------|
| `0` | Dismiss (close without answering) |
| `1` | Bad (poor experience) |
| `2` | Fine (average experience) |
| `3` | Good (great experience) |

#### Why This Design

Digit keys serve a dual role in survey contexts: they are both rating inputs and ordinary text input (the user might be typing a numbered list such as "1. Step one..."). The 400 ms debounce window lets the system distinguish these two intents — if the user continues typing other characters within 400 ms, the digit is treated as part of text and the submission is cancelled; if no further input arrives within 400 ms, the digit is treated as a rating selection. The source code comment states this explicitly: *"Short enough to feel instant for intentional presses, long enough to cancel when the user types more characters"* (`useDebouncedDigitInput.ts`).

### Probability Gating

The probability of displaying a survey is controlled by the dynamic configuration `tengu_feedback_survey_config`. This configuration is fetched from a remote source and determines how likely it is that the survey appears in any session that meets the display conditions.

### Cooldown Session Tracking

The system maintains a cooldown session count. After the user completes one survey, the survey will not appear again for a certain number of subsequent sessions, avoiding excessive interruption.

### Analytics Instrumentation

All survey events are reported through a unified analytics event:

```
Event name: 'tengu_feedback_survey_event'
Types:
  - appeared  — the survey was displayed to the user
  - responded — the user made a response
```

---

## Transcript Sharing

After the user completes a feedback rating, the system may further ask whether they are willing to share the session transcript to help improve the product.

### Response Types

```typescript
type TranscriptShareResponse = 'yes' | 'no' | 'dont_ask_again';
```

- **yes**: Agree to share the current transcript
- **no**: Do not share this time
- **dont_ask_again**: Never ask again (persisted preference)

### Submission Flow (submitTranscriptShare.ts)

1. **Collect messages**: Collect the normalized messages from the current session along with all subagent transcripts
2. **Read raw JSONL**: Read the raw JSONL-format transcript file from disk, protected by a `MAX_TRANSCRIPT_READ_BYTES` size limit to prevent memory issues from reading excessively large files
3. **Sensitive information redaction**: Run `redactSensitiveInfo()` on the transcript content to remove potentially sensitive data (API keys, tokens, passwords, etc.)

   #### Why This Design

   Conversation transcripts shared by users can easily contain API keys, passwords, tokens, and other sensitive information — developers frequently handle credentials in the terminal. `redactSensitiveInfo()` performs local redaction before upload, ensuring that sensitive data never leaves the user's machine. In the source code, `submitTranscriptShare.ts` explicitly calls `const content = redactSensitiveInfo(jsonStringify(data))` before the final upload — this is a mandatory security checkpoint that cannot be skipped. This "redact first, upload second" principle permeates the entire feedback system: in `Feedback.tsx`, all user-provided content including description, error, and stack traces is redacted without exception.

4. **Upload**: Send via HTTP POST to:
   ```
   https://api.anthropic.com/api/claude_code_shared_session_transcripts
   ```

### Trigger Types

Transcript sharing can be triggered by the following scenarios:

| Trigger Type | Description |
|--------------|-------------|
| `bad_feedback_survey` | The user selected "Bad" in the feedback survey |
| `good_feedback_survey` | The user selected "Good" in the feedback survey |
| `frustration` | The system detected that the user may have encountered frustration |
| `memory_survey` | Triggered during the memory survey flow |

---

## Memory Survey (useMemorySurvey)

The memory survey is a specialized survey targeting the automatic memory feature.

### Trigger Conditions

- Check whether the current session messages contain **auto-memory file reads** (i.e., Claude automatically read a memory file)
- If a memory file read is detected, trigger the survey with probability **0.2** (20%)

#### Why This Design

The memory survey uses `SURVEY_PROBABILITY = 0.2` probability gating (source: `useMemorySurvey.tsx`, line 21) rather than displaying the survey every time. This balances user experience against data collection: a 20% probability means the survey is triggered on average once every 5 uses of the memory feature, so users do not feel repeatedly harassed; yet the rate is high enough to collect a statistically meaningful sample within a reasonable time. The survey is also gated by the feature flag `tengu_dunwich_bell`, which can be remotely disabled to stop bothering users once sufficient data has been collected.

### Analytics Instrumentation

```
Event name: 'tengu_memory_survey_event'
```

Records user feedback on the memory feature to help evaluate the practical effectiveness of the automatic memory system.

---

## Post-Compact Survey (usePostCompactSurvey)

The post-compact survey is triggered after a session has undergone **conversation compaction** (conversation summarization/compression).

When the conversation context becomes too long and automatic compaction is triggered, the system asks the user after compaction completes to rate the compaction result, collecting feedback on information loss, context retention quality, and related aspects.

---

## Good Claude Command (/good-claude)

`/good-claude` is a positive feedback shortcut command.

When the user is satisfied with a particular response from Claude, they can quickly send positive feedback via this command without going through the full survey flow. This provides a low-friction way for users to signal "this response was great."

---

## Feedback Command (/feedback)

The `/feedback` command provides a complete feedback submission interface.

### Aliases

- `/bug` — can be used as an alias for `/feedback`

### Gating Conditions

This command is **unavailable** under the following conditions:

- When using the **Bedrock** backend
- When using the **Vertex** backend
- When using the **Foundry** backend
- When the user belongs to **ANT** (Anthropic internal)
- When the organization policy does not allow `product_feedback`

### Rendering

When the command is triggered, it renders the `Feedback` component with the following parameters:

- **abort signal**: Used to cancel the feedback submission flow
- **messages**: The current session message context
- **initial description**: Initial description text (e.g., passed in from command arguments)

---

## Tips System

The tips system displays helpful tips while the user is waiting for a Claude response (e.g., while the spinner is spinning).

### Tip Registry (tipRegistry.ts)

The system registers **60+ tips**, each with the following structure:

```typescript
interface Tip {
  id: string;                          // Unique identifier
  content: () => Promise<string>;      // Async content generation function
  cooldownSessions: number;            // Number of cooldown sessions
  isRelevant: () => Promise<boolean>;  // Async relevance check function
}
```

- **id**: The unique identifier for each tip
- **content**: An async function that returns the display content of the tip (supports dynamic generation)
- **cooldownSessions**: The number of sessions the tip must cool down after being shown, to avoid repeated display
- **isRelevant**: An async function that determines whether the tip is relevant given the current context (e.g., some tips are only relevant on specific platforms or configurations)

### Selection Algorithm (tipScheduler.ts)

Uses a **longest-time-since-shown-first** strategy to select the next tip:

- Among all tips that meet the relevance conditions and are not in a cooldown period, select the one that was shown least recently
- Ensures tip display is as evenly distributed as possible, preventing the user from repeatedly seeing the same content

#### Why This Design

With 60+ tips registered, random selection risks some tips never being shown while others appear repeatedly. The `selectTipWithLongestTimeSinceShown()` function (`tipScheduler.ts`) sorts by "number of sessions since last shown" in descending order, giving priority to the tip that has gone unseen the longest. In `tipHistory.ts`, `getSessionsSinceLastShown()` returns `numStartups - lastShown` and returns `Infinity` for tips that have never been shown, guaranteeing that new tips are always shown first. This deterministic scheduling is fairer than random selection: it ensures every tip gets a chance to appear, allowing users to gradually discover all product features over continued use.

### History Persistence (tipHistory.ts)

Tip display history is persisted via global configuration:

```typescript
// Storage structure: tipId → numStartups
// Records how many startups each tip has been shown in
Record<string, number>
```

This data is stored in the global config and persists across sessions.

### Analytics Instrumentation

```
Event name: 'tengu_tip_shown'
```

Reported each time a tip is shown, used to analyze the display frequency and coverage of each tip.

### Custom Tips

Users can provide custom tip content by setting `settings.spinnerTipsOverride`, overriding or supplementing the default tip list.

### Plugin Tips

Marketplace plugins can register their own tips. These plugin tips are incorporated into the unified tip scheduling system and participate in selection and display alongside built-in tips.

---

## Engineering Practice Guide

### Triggering Feedback Collection

**Feedback state machine flow:**

1. The user presses thumbs down (or uses the `/feedback` or `/bug` command) to trigger the feedback flow
2. State machine transitions: `closed → open → thanks → transcript_prompt → submitting → submitted`
3. Each state has a clearly defined predecessor and successor; skipping steps is not allowed (e.g., you cannot go from `closed` directly to `submitting`)
4. The user can cancel at any stage (feedback is optional)

**Key entry points:**
- `FeedbackSurvey.tsx` — Main feedback survey component
- `useFeedbackSurvey.tsx` — Feedback survey hook, manages state and probability gating
- `submitTranscriptShare.ts` — Transcript submission flow

### Debugging the Feedback State Machine

**Troubleshooting steps:**

1. **Check the current state**: Which phase is the state machine in (closed/open/thanks/transcript_prompt/submitting/submitted)?
2. **Check transition conditions**: Confirm that the conditions for triggering the transition are satisfied
3. **400 ms debounce effect**: `useDebouncedDigitInput` sets a 400 ms debounce window; digit key input (0=dismiss, 1=bad, 2=fine, 3=good) within this window will be cancelled if followed by more input — this is to distinguish rating intent from ordinary text input
4. **Probability gating**: Survey display probability is controlled by the `tengu_feedback_survey_config` dynamic configuration; it does not trigger every time
5. **Cooldown period**: After completing one survey there is a cooldown sessions period; the survey will not appear in the next several sessions

**Debugging transcript submission failures:**
- Check the `MAX_TRANSCRIPT_READ_BYTES` limit (prevents memory issues from reading excessively large files)
- Confirm that `redactSensitiveInfo()` redaction is executing correctly (API keys, passwords, and tokens will be removed)
- Check network connectivity (upload goes to `https://api.anthropic.com/api/claude_code_shared_session_transcripts`)

### Customizing Survey Probability

**Memory Survey:**
- `SURVEY_PROBABILITY = 0.2` (source: `useMemorySurvey.tsx`, line 21) — 20% probability of triggering
- Gated by feature flag `tengu_dunwich_bell`, can be remotely disabled
- Triggers only when auto-memory file reads are detected

**Post-Compact Survey:**
- `SURVEY_PROBABILITY = 0.2` (source: `usePostCompactSurvey.tsx`, line 15) — 20% probability of triggering
- Triggers after a session undergoes conversation compaction

**Feedback Survey:**
- Probability controlled by the `tengu_feedback_survey_config` remote configuration
- Cooldown session count prevents excessive interruption

### Customizing Tips

- **Override Tips**: Set `settings.spinnerTipsOverride` to provide custom tip content
- **Plugin Tips**: Marketplace plugins can register their own tips, incorporated into unified scheduling
- **Scheduling algorithm**: Longest-time-since-shown-first (`selectTipWithLongestTimeSinceShown()`); new tips return `Infinity` to guarantee they are shown first

### Common Pitfalls

| Pitfall | Details | Solution |
|---------|---------|----------|
| Feedback is optional | Users can cancel the feedback flow at any stage | The UI must handle cancellation gracefully; do not assume the flow will always complete |
| Transcript sharing applies redaction | `redactSensitiveInfo()` performs local redaction before upload — this is a mandatory security checkpoint | If redaction is found to be incomplete, fix `redactSensitiveInfo()` rather than bypassing it |
| 400 ms debounce window | Digit key ratings require no further input within 400 ms to trigger | Rapid consecutive input may cause ratings to go unrecognized |
| `/feedback` command has gating conditions | Unavailable with Bedrock/Vertex/Foundry backends, ANT internal users, or when policy disallows it | The feedback entry point may differ across environments |
| Tip display history persists across sessions | Stored in the global config (`tipId → numStartups` mapping) | Clearing the config will reset the tip display history |


---

[← Telemetry & Analytics](../18-遥测分析/telemetry-system-en.md) | [Index](../README_EN.md) | [Service Layer →](../20-服务层/services-complete-en.md)
