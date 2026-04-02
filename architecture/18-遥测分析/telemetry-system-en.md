# Telemetry & Analytics System

Claude Code's telemetry and analytics system encompasses several subsystems: event analytics (Analytics), OpenTelemetry integration, Session Tracing, Plugin Telemetry, Cost Tracking, and Gateway Detection.

---

## Analytics Architecture

### No-Dependency Design (events.ts)

The Analytics system uses a **no-dependency design**: events are generated in `events.ts` and placed into a queue, and are not actually sent until a sink is attached (via `attachAnalyticsSink()`). This ensures that events are not lost during initialization or when a sink is not yet ready.

### AnalyticsSink Interface

```typescript
interface AnalyticsSink {
  logEvent: (event: AnalyticsEvent) => void;        // Synchronous event logging
  logEventAsync: (event: AnalyticsEvent) => Promise<void>;  // Asynchronous event logging
}
```

### Proto Field Stripping

The `stripProtoFields()` function removes all keys prefixed with `_PROTO_*` from event objects. These fields are for internal use only and must be stripped before sending to general-access backends.

### Design Rationale

#### Why Datadog + 1P dual routing?

The source comment in `analytics/index.ts` states: *"DESIGN: This module has NO dependencies to avoid import cycles. Events are queued until attachAnalyticsSink() is called."* `sinkKillswitch.ts` defines `SinkName = 'datadog' | 'firstParty'` as two independent channels. Datadog provides real-time monitoring and alerting (operations perspective — P0 events trigger immediate notifications), while 1P (first-party Anthropic event logging) provides long-term analysis and product decision support (product perspective — user behavior trends, feature adoption rates). The two channels have completely different data requirements, access permissions, and retention policies. `_PROTO_*` fields are sent only to 1P; they are stripped from general-access backends like Datadog via `stripProtoFields()` — this is data security layering.

#### Why is the telemetry system a "no-dependency queue" design?

The source file `analytics/index.ts` explicitly declares at the top: *"DESIGN: This module has NO dependencies to avoid import cycles. Events are queued until attachAnalyticsSink() is called during app initialization."* Telemetry must not affect core functionality — if the telemetry service goes down, users' code editing should be unaffected. The queue design ensures: (1) events are not lost during startup when the sink is not yet ready; (2) send failures are handled silently (fail-open principle: when the `sinkKillswitch` is missing or errors, the sink remains open by default); (3) no import dependencies avoids circular references — the telemetry module is used by almost every other module, so if it in turn depended on other modules, circular imports would quickly form.

#### Why GrowthBook instead of a custom feature flag system?

GrowthBook is an industry-standard A/B testing platform that supports canary releases and controlled experiments. The source code makes heavy use of `getFeatureValue_CACHED_MAY_BE_STALE()` and GrowthBook gates (such as `tengu_log_datadog_events` and `enhanced_telemetry_beta`). Building a custom feature flag system would reinvent the wheel: you would need to implement user segmentation, progressive rollouts, experiment analysis, and emergency rollbacks yourself. GrowthBook also provides session-stable feature evaluation (the source comment on `promptCache1hAllowlist` reads: *"Cached prompt cache 1h TTL allowlist from GrowthBook (session-stable)"*), ensuring that feature flags do not flip mid-session.

### Dual-Channel Routing

Events are simultaneously sent to two backends via a dual-channel routing mechanism:

1. **Datadog** — Third-party monitoring and analytics platform
2. **1P Event Logging** — Anthropic first-party event logging system

### Datadog Gating

The `shouldTrackDatadog()` function controls whether events are sent to Datadog:

- **Killswitch**: A global switch that can emergency-disable Datadog reporting
- **GrowthBook feature gate**: Controlled via the `tengu_log_datadog_events` feature gate

### Sampling Logic

The `shouldSampleEvent()` function implements event sampling, downsampling high-frequency events to control data volume and cost.

### Sink Killswitch

Supports independent per-sink disabling:

```typescript
interface SinkKillswitch {
  datadog?: boolean;     // Disable the Datadog sink
  firstParty?: boolean;  // Disable the first-party sink
}
```

The design principle is **fail-open**: when killswitch configuration is missing or fails to load, the sink remains open by default, ensuring that telemetry data is not silently lost due to configuration issues.

---

## OpenTelemetry Stack

### Initialization Flow

#### bootstrapTelemetry()

A bootstrap-phase function responsible for setting OTEL-related environment variables. Called before the OpenTelemetry SDK initializes to ensure all required environment configuration is in place.

#### initializeTelemetry()

The core initialization function, which creates the following three providers:

- **MeterProvider** — Metrics provider
- **LoggerProvider** — Logging provider
- **TracerProvider** — Tracing provider

### Exporter Types

Multiple exporter types are supported:

| Exporter | Transport | Description |
|----------|-----------|-------------|
| `console` | stdout | Development/debug use; outputs to console |
| `otlp` (gRPC) | gRPC | High-performance binary transport |
| `otlp` (HTTP) | HTTP/JSON | HTTP transport |
| `otlp` (Protobuf) | HTTP/Protobuf | HTTP + Protobuf encoding |
| `prometheus` | HTTP pull | Prometheus-compatible pull mode |

### BigQuery Metrics

For API customers and C4E/Team users, metric data is exported to BigQuery for deeper analysis and reporting.

### Resource Merging

The OTEL Resource is built by merging multiple layers of detectors:

```
base resource
  + OS detector (operating system info)
  + Host arch detector (host architecture info)
  + Env detectors (environment variable detection)
  → merged resource
```

### Flush Mechanism

`flushTelemetry()` force-flushes all buffered data across all providers, with a timeout of **2 seconds**. It is called before process exit to ensure no data is lost.

---

## Session Tracing (sessionTracing.ts)

The session tracing system provides detailed request-chain tracing capabilities.

### Enablement Conditions

The `isEnhancedTelemetryEnabled()` function checks whether enhanced telemetry is enabled:

- **Feature gate**: Static feature switch
- **GrowthBook**: Dynamic gating via `enhanced_telemetry_beta`

### Span Types

| Span Type | Description |
|-----------|-------------|
| `interaction` | Interaction span: wraps the full flow from user request to Claude response |
| `llm_request` | LLM request span: a single model invocation |
| `tool` | Tool span: full lifecycle of a tool call |
| `tool.blocked_on_user` | Tool-blocked span: child span recording time spent waiting for user confirmation |
| `tool.execution` | Tool execution span: child span recording actual tool execution time |
| `hook` | Hook span: hook execution |

#### Why does session tracing have 6 span types?

The complete lifecycle of a single user request requires independent measurement of each phase: `interaction` (overall user interaction), `llm_request` (single model call, including TTFT), `tool` (full tool lifecycle), `tool.blocked_on_user` (waiting for user permission confirmation), `tool.execution` (actual tool execution), and `hook` (hook execution). With only one coarse-grained span, it is impossible to distinguish "the model was slow" from "the tool was slow" from "the user approval was slow" — and these three have completely different optimization strategies. The hierarchical relationship of the 6 span types (`interaction` contains `llm_request` + `tool`; `tool` contains `blocked_on_user` + `execution`) forms a complete causal chain.

### Interaction Spans

The interaction span is the root span, wrapping the entire process from when a user sends a request to when Claude completes its response. A single interaction span may contain multiple LLM request spans and tool spans.

### LLM Spans

LLM request spans track detailed information about a single model call:

- `input_tokens` — Number of input tokens
- `output_tokens` — Number of output tokens
- `cache_read_tokens` — Number of cache-read tokens
- `ttft_ms` — Time To First Token (milliseconds)

### Tool Spans

Tool spans track the full lifecycle of a tool call, and may internally contain:

- **blocked-on-user child span**: When a tool requires user confirmation (e.g., confirming a file write), this records the time spent waiting for the user's response
- **execution child span**: The time spent on actual tool execution

### Orphaned Span Cleanup

The system cleans up orphaned spans via a background periodic task:

- **TTL**: 30 minutes
- Periodically scans all active spans, force-ends and evicts any span that has exceeded the TTL
- Prevents memory leaks caused by spans that never close due to exceptions

### Perfetto Integration

Supports the **Perfetto** trace format, generating trace files via parallel file writes. These trace files can be imported into the Perfetto UI for visual debugging and analysis.

---

## Event Logging (events.ts)

### logOTelEvent()

Emits events as OTEL log records. Each event is simultaneously an analytics event and an OTEL log record.

### Event Ordering

Global event ordering is guaranteed by a **monotonically increasing** `eventSequence` counter. Each event is assigned an incrementing sequence number at creation time, ensuring that causal order is preserved in asynchronous environments.

### Prompt Redaction

The `redactIfDisabled()` function automatically redacts user prompts when the `OTEL_LOG_USER_PROMPTS` environment variable is not set. The default behavior is redaction (privacy protection); raw prompts are retained only when the user explicitly enables that environment variable.

---

## Plugin Telemetry (pluginTelemetry.ts)

The plugin telemetry system provides standardized telemetry capabilities for the plugin ecosystem.

### Plugin ID Hashing

The `hashPluginId()` function applies **SHA256 hashing** to plugin IDs, taking the first **16 characters** as an anonymized identifier. This prevents the actual plugin names or paths from being exposed in telemetry data.

### Plugin Scope Classification

The `getTelemetryPluginScope()` function classifies plugins into the following scopes:

| Scope | Description |
|-------|-------------|
| `official` | Anthropic official plugins |
| `org` | Organization-level plugins |
| `user-local` | User local plugins |
| `default-bundle` | Default bundled plugins |

### Plugin Command Error Classification

`classifyPluginCommandError()` classifies plugin command execution errors into 5 categories:

1. **network** — Network-related errors (connection failures, timeouts, etc.)
2. **not-found** — Command or resource not found
3. **permission** — Insufficient permissions
4. **validation** — Input validation failure
5. **unknown** — Other unclassified errors

### Session-Level Plugin Analytics

`logPluginsEnabledForSession()` records all currently enabled plugins at session start, reporting analytics events at per-plugin granularity.

### Load Error Analytics

`logPluginLoadErrors()` records errors that occur during plugin loading, reporting them at per-error granularity to help diagnose plugin compatibility and configuration issues.

---

## Cost Tracking (cost-tracker.ts)

The cost tracking system tracks API call token consumption and costs in real time.

### Cumulative Tracking

Accumulates the following metrics on a per-model basis:

| Metric | Description |
|--------|-------------|
| `input_tokens` | Number of input tokens |
| `output_tokens` | Number of output tokens |
| `cache_read_tokens` | Number of cache-read tokens |
| `cache_creation_tokens` | Number of cache-creation tokens |
| `cost` | Cumulative cost (USD) |
| `duration` | Cumulative duration |

### Cost Formatting

The `formatTotalCost()` function generates a breakdown display of model usage, listing each metric and cost by model.

### Session Persistence

- `saveCurrentSessionCosts()` — Saves the current session's cost state to disk
- `restoreCostStateForSession()` — Loads the previous cost state from disk when a session is restored

### StoredCostState Type

```typescript
interface StoredCostState {
  input_tokens: number;
  output_tokens: number;
  cache_read_tokens: number;
  cache_creation_tokens: number;
  cost: number;
  duration: number;
  // ... other tracked fields
}
```

Contains a complete snapshot of all tracked fields, enabling seamless cost state continuity across sessions.

### React Hook (costHook.ts)

`costHook.ts` provides a React hook that displays a cost summary when the user exits, including the total token consumption and cost for the current session.

---

## Gateway Detection

The system can automatically detect and identify API gateways or proxies that may exist in the user's request path.

### Known Gateway Fingerprints

The following gateways are identified via response headers, request characteristics, and other signals:

| Gateway | Description |
|---------|-------------|
| **LiteLLM** | Open-source LLM proxy gateway |
| **Helicone** | LLM observability platform |
| **Portkey** | AI gateway and observability |
| **Cloudflare** | Cloudflare AI Gateway |
| **Kong** | Kong API Gateway |
| **Braintrust** | AI evaluation and proxy platform |
| **Databricks** | Databricks Model Serving |

Detected gateway information is recorded in telemetry data, helping Anthropic understand the user's API access topology and issues that may be introduced by the gateway.

---

## Engineering Practice Guide

### Adding a New Analytics Event

**Step checklist:**

1. **Send the event using `logEvent()`**: Import `logEvent` from `src/services/analytics/index.ts`
   ```typescript
   import { logEvent } from 'src/services/analytics/index.js'
   logEvent('tengu_your_event_name', {
     property1: 'value',
     property2: 123,
   })
   ```
2. **Define the event name and properties**: Event names use the `tengu_` prefix (all internal events in the source code follow this convention)
3. **Be mindful of Proto fields**: Properties prefixed with `_PROTO_*` are sent only to the first-party backend; they are stripped from Datadog by `stripProtoFields()`
4. **Dual-channel routing**: Events are sent simultaneously to Datadog (real-time monitoring) and 1P (long-term analysis); each channel can be independently disabled via `SinkKillswitch`
5. **Sampling control**: High-frequency events use `shouldSampleEvent()` for downsampling; evaluate whether new events require sampling

**Note**: The header comment in `analytics/index.ts` explicitly states: "DESIGN: This module has NO dependencies to avoid import cycles" — do not introduce dependencies on other business modules in this module.

### Debugging Missing Telemetry

**Troubleshooting steps:**

1. **Check environment variables**: Setting `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` disables non-essential telemetry
2. **Check the Sink Killswitch**: The `datadog` and `firstParty` switches in `sinkKillswitch.ts` can independently disable a channel
3. **Check the GrowthBook gate**: Datadog events are gated by the `tengu_log_datadog_events` feature gate; `shouldTrackDatadog()` may return false due to gating
4. **Confirm the sink is attached**: Events are queued before `attachAnalyticsSink()` is called; if the sink is never attached, events will never be sent
5. **Fail-open principle**: When killswitch configuration is missing or fails to load, the sink stays open by default — if telemetry is still missing, the problem may be at the network layer

**Key source locations**:
- `analytics/index.ts` — Event queue and sink attachment
- `analytics/sinkKillswitch.ts` — Kill switch logic (note comment: "Must NOT be called from inside is1PEventLoggingEnabled()")
- `analytics/datadog.ts` — Datadog sink implementation (comment: "use via src/services/analytics/index.ts > logEvent")

### Performance Tracing

**Using Perfetto trace files to analyze request chain performance:**

1. Session tracing generates trace files in Perfetto format
2. Import into [Perfetto UI](https://ui.perfetto.dev/) for visual analysis
3. The 6 span types provide a complete causal chain: `interaction` → `llm_request` + `tool` (containing `blocked_on_user` + `execution`) + `hook`
4. Key metrics: `ttft_ms` (time to first token), `input_tokens`/`output_tokens` (token consumption), `cache_read_tokens` (cache hits)

**Enabling enhanced telemetry**: Checked via `isEnhancedTelemetryEnabled()`, gated by the `enhanced_telemetry_beta` GrowthBook feature gate

### OTEL Export Configuration

| Exporter Type | Suitable Scenario |
|--------------|-------------------|
| `console` | Local development and debugging |
| `otlp` (gRPC) | High-performance transport in production |
| `otlp` (HTTP/JSON) | Alternative when gRPC is blocked by firewalls |
| `prometheus` | When an existing Prometheus infrastructure is in place |

**Flush mechanism**: `flushTelemetry()` has a 2-second timeout; call it before process exit to ensure no data is lost.

### Common Pitfalls

| Pitfall | Details | Solution |
|---------|---------|---------|
| Telemetry is async fire-and-forget | Delivery is not guaranteed; send failures are handled silently | Do not make critical business logic dependent on telemetry success |
| Do not log sensitive data in telemetry | `redactIfDisabled()` auto-redacts when `OTEL_LOG_USER_PROMPTS` is not set | Default behavior is redaction; raw prompts are retained only when explicitly enabled |
| Orphaned span memory leaks | Exceptions may cause spans to never close | The system automatically cleans up orphaned spans with a TTL exceeding 30 minutes |
| Event sequence numbers guarantee causal order | `eventSequence` is a monotonically increasing counter | In async environments, rely on sequence numbers rather than timestamps to determine event order |
| Plugin telemetry privacy | `hashPluginId()` uses SHA256 and takes the first 16 characters | Raw plugin names/paths are not visible in telemetry data |


---

[← Error Recovery](../17-错误恢复/error-recovery-en.md) | [Index](../README_EN.md) | [Feedback & Survey →](../19-反馈与调查/feedback-system-en.md)
