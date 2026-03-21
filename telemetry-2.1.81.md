# Claude Code v2.1.81 — Exhaustive Telemetry Reference

> Source: `cli.js` (619,056 lines, 18.4 MB), extracted TypeScript modules in `src/`  
> Build: `2026-03-20T21:25:42Z`  
> Total unique `tengu_` event types: **781**

---

## Table of Contents

1. [Overview](#1-overview)
2. [Telemetry Architecture](#2-telemetry-architecture)
3. [Transmission Backends](#3-transmission-backends)
   - [3.1 Segment (Customer Analytics)](#31-segment-customer-analytics)
   - [3.2 Datadog (Production Monitoring)](#32-datadog-production-monitoring)
   - [3.3 First-Party OpenTelemetry (Anthropic)](#33-first-party-opentelemetry-anthropic)
4. [Event Sampling & Gating](#4-event-sampling--gating)
5. [Disabling Telemetry](#5-disabling-telemetry)
6. [OpenTelemetry Metrics](#6-opentelemetry-metrics)
7. [OpenTelemetry Configuration](#7-opentelemetry-configuration)
8. [Error Tracking](#8-error-tracking)
9. [Privacy & Data Handling](#9-privacy--data-handling)
10. [Complete Event Reference](#10-complete-event-reference)
11. [Datadog Critical Events (46 events)](#11-datadog-critical-events-46-events)
12. [Quick Reference](#12-quick-reference)

---

## 1. Overview

Claude Code v2.1.81 implements a **three-tier telemetry system** that collects behavioral, performance, and error data across every user interaction.

| Tier | Backend | Purpose | Event Coverage |
|------|---------|---------|---------------|
| Customer Analytics | Segment | Product analytics, funnel tracking | All 781 events |
| Production Monitoring | Datadog | Real-time ops monitoring | 46 critical events only |
| First-Party OTEL | Anthropic internal | Internal behavioral logging | All events (gated by `tengu_frond_boric`) |

**Key facts:**
- 781 distinct `tengu_*` event types identified in v2.1.81 source
- All events route through a shared sink (`jKA` / `Q` / `JKA`) before being fanned out
- Telemetry is **auto-disabled** for third-party provider users (Bedrock, Vertex, Foundry) unless explicitly re-enabled
- No Sentry; uses custom error queue with 100-error ring buffer
- GrowthBook feature flags gate per-event sample rates and backend routing

---

## 2. Telemetry Architecture

### 2.1 Event Emission Functions

**Source:** `cli.js:4294-4318`

```javascript
// Synchronous event emission
function Q(A, q) {
  if (rr === null) {
    ty6.push({ eventName: A, metadata: q, async: false });
    return;
  }
  rr.logEvent(A, q);
}

// Asynchronous event emission
async function JKA(A, q) {
  if (rr === null) {
    ty6.push({ eventName: A, metadata: q, async: true });
    return;
  }
  await rr.logEventAsync(A, q);
}
```

| Symbol | Role | Calling convention |
|--------|------|--------------------|
| `Q(name, metadata)` | Synchronous fire-and-forget | Used for 99%+ of all events |
| `JKA(name, metadata)` | Async version — awaits delivery | Used where async context is available |
| `ty6` | Pre-sink queue (array) | Buffers events before sink initialized |
| `rr` | Active sink reference | `null` until `jKA()` is called |

### 2.2 Sink Initialization & Queue Flush

**Source:** `cli.js:4294-4307`

```javascript
function jKA(A) {
  if (rr !== null) return;           // idempotent
  rr = A;
  if (ty6.length > 0) {
    let q = [...ty6];
    ty6.length = 0;
    queueMicrotask(() => {
      for (let K of q)
        if (K.async) rr.logEventAsync(K.eventName, K.metadata);
        else         rr.logEvent(K.eventName, K.metadata);
    });
  }
}
```

The sink is initialized by `N26()` → `initializeAnalyticsSink()` (`src/telemetry/initializeanalyticssink-2.ts:228`), which calls:

```javascript
function N26() {
  jKA({ logEvent: JJY, logEventAsync: MJY });
}
```

`JJY` and `MJY` are the fanout functions that route to all three backends.

### 2.3 Event Fanout (Three Parallel Paths)

**Source:** `src/telemetry/initializeanalyticssink-2.ts:207-224`

```javascript
function JJY(A, q) {               // sync fanout
  let K = $H8(A);                  // check sample rate
  if (K === 0) return;             // sampled out → drop
  let _ = K !== null ? { ...q, sample_rate: K } : q;
  let Y = ey6(_);                  // enrich metadata
  if (tyq()) xb1(A, Y);           // → Segment
  if (eyq()) zb1(A, Y);           // → Datadog (only if in P9_ set)
  GP6(A, _);                       // → First-party OTEL
}

async function MJY(A, q) {         // async fanout
  let K = $H8(A);
  if (K === 0) return;
  let _ = K !== null ? { ...q, sample_rate: K } : q;
  let Y = ey6(_);
  if (tyq()) await xb1(A, Y);     // → Segment (awaited)
  if (eyq()) zb1(A, Y);           // → Datadog
  GP6(A, _);                       // → First-party OTEL
}
```

**Backend gates:**
- `tyq()` — checks GrowthBook gate `tengu_log_segment_events` (var: `oyq`)
- `eyq()` — checks GrowthBook gate `tengu_log_datadog_events` (var: `syq`)
- First-party OTEL always called when `o56()` returns true (i.e., telemetry not disabled)

---

## 3. Transmission Backends

### 3.1 Segment (Customer Analytics)

**Source:** `cli.js:354454-354515`

#### Credentials

| Key | Value |
|-----|-------|
| Production write key | `LKJN8LsLERHEOXkw487o7qCTFOrGPimI` |
| Development write key | `b64sf1kxwDGe1PiSAlv5ixuH0f509RKK` |

#### Client Configuration

```javascript
new uS4.Analytics({
  writeKey: pY_(),        // selects prod or dev key based on env
  flushAt: 50,            // batch up to 50 events
  flushInterval: 1e4,     // flush every 10 seconds (10,000 ms)
})
```

#### Payload Structure

```javascript
// Per-event track payload (xb1 function, cli.js:354454)
async function xb1(A, q) {
  let _ = ZH8(),            // anonymousId (device fingerprint)
      Y = x3(),             // current auth session
      z = await PP6(...),   // enriched metadata (model, betas, etc.)
      w = gC7(z, q),        // merge event properties
      O = { anonymousId: _, event: A, properties: w };
  if (Y) {
    let $ = YQ(true);       // user identity
    O.userId = $.deviceId;
    O.properties.accountUuid = Y.accountUuid;
    O.properties.organizationUuid = Y.organizationUuid;
  }
  K.track(O);
}
```

| Field | Source | Description |
|-------|--------|-------------|
| `anonymousId` | `ZH8()` | Device-derived anonymous identifier |
| `event` | Event name string | e.g. `tengu_api_success` |
| `userId` | `YQ(true).deviceId` | Device ID (when authenticated) |
| `properties.accountUuid` | Auth session | Anthropic account UUID |
| `properties.organizationUuid` | Auth session | Organization UUID |
| `properties.*` | Event metadata | All event-specific fields |

Segment also supports `identify()` calls via `BS4(traits)` which sends the same `anonymousId`/`userId` with user trait updates.

#### Gating

Segment routing is gated by GrowthBook feature `tengu_log_segment_events`. When this gate is `false`, Segment receives no events despite the sink being initialized.

---

### 3.2 Datadog (Production Monitoring)

**Source:** `cli.js:352107-352310`

#### Credentials & Endpoint

| Key | Value |
|-----|-------|
| Endpoint URL | `https://http-intake.logs.us5.datadoghq.com/api/v2/logs` |
| API key | `pubbbf48e6d78dae54bceaa4acf463299bf` |
| Variable names | `j9_` (URL), `J9_` (key) |

#### Batching Configuration

```javascript
var M9_ = 15000;     // default flush interval: 15 seconds
var X9_ = 100;       // max batch size: 100 events
var D9_ = 5000;      // retry delay: 5 seconds
```

Flush interval is runtime-configurable:

```javascript
function T9_() {
  return parseInt(process.env.CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS || "", 10)
    || M9_;          // default 15000ms
}
```

#### Payload Structure

```javascript
// zb1 function — Datadog log record
{
  ddsource: "nodejs",
  ddtags: "event:tengu_api_error,arch:arm64,model:claude-sonnet-4-6,...",
  message: "tengu_api_error",
  service: "claude-code",
  hostname: "claude-code",
  env: "external",
  // All non-null event metadata fields flattened as camelCase
  arch: "arm64",
  clientType: "...",
  model: "claude-sonnet-4-6",
  ...
}
```

#### Tagged Dimensions (W9_ array)

The following fields are promoted to Datadog `ddtags` for indexed filtering:

```
arch, clientType, errorType, http_status_range, http_status,
kairosActive, model, platform, provider, subscriptionType,
toolName, userBucket, userType, version, versionBase
```

**Note on toolName:** MCP tool names are normalized: `mcp__*` → `"mcp"` before sending to Datadog.

**Note on version:** Dev version strings are stripped of build metadata: `2.1.81-dev.20260320.t12345.sha1abc` → `2.1.81-dev.20260320`.

#### User Bucket

`v9_()` computes a deterministic 0–29 integer bucket from the SHA-256 hash of the device ID:

```javascript
v9_ = z1(() => {
  let A = XL(),                              // device ID
      q = H9_("sha256").update(A).digest("hex");
  return parseInt(q.slice(0, 8), 16) % G9_;  // G9_ = 30
});
```

#### Gating

- Gated by `eyq()` → GrowthBook `tengu_log_datadog_events`
- Gated by `QA() === "firstParty"` — only runs for first-party (Anthropic-hosted) users
- Only emits events in the `P9_` set (46 events — see Section 11)

---

### 3.3 First-Party OpenTelemetry (Anthropic)

**Source:** `cli.js:174151-174298`, `src/conversation/shutdown1peventlogging-2.ts`

#### SDK

Uses a vendored copy of the OpenTelemetry Logs SDK (`src/vendor/opentelemetry.ts`, 743 lines, 22 modules). Key components:

- `wH8.LoggerProvider` — OTEL log provider
- `wH8.BatchLogRecordProcessor` — batching processor
- `CX1` — custom Anthropic OTLP exporter
- Logger name: `"com.anthropic.claude_code.events"`

#### Initialization (`AI7` function)

```javascript
function AI7() {
  if (!o56()) return;   // bail if telemetry disabled

  let q = LG("tengu_1p_event_batch_config", {}),
      K = q.scheduledDelayMillis
            || parseInt(process.env.OTEL_LOGS_EXPORT_INTERVAL || "10000"),
      _ = q.maxExportBatchSize || 200,     // yo3 = 200
      Y = q.maxQueueSize      || 8192,     // Lo3 = 8192
      w = {
        [ATTR_SERVICE_NAME]:    "claude-code",
        [ATTR_SERVICE_VERSION]: "2.1.81",
      };

  // WSL extension: add wsl.version attribute
  if (E1() === "wsl") w["wsl.version"] = G46();

  let O = sC7.resourceFromAttributes(w),
      $ = new CX1({ maxBatchSize: _, ...q });

  vt = new wH8.LoggerProvider({
    resource: O,
    processors: [
      new wH8.BatchLogRecordProcessor($, {
        scheduledDelayMillis: K,
        maxExportBatchSize:   _,
        maxQueueSize:         Y,
      })
    ]
  });
  Tt = vt.getLogger("com.anthropic.claude_code.events", "2.1.81");
}
```

#### Constants

| Variable | Value | Meaning |
|----------|-------|---------|
| `Eo3` | `10000` | Default export interval (ms) |
| `yo3` | `200` | Default max batch size |
| `Lo3` | `8192` | Default max queue size |
| `ko3` | `"tengu_event_sampling_config"` | Sampling config feature key |

#### Per-Event Payload (`No3` function)

```javascript
async function No3(A, q, K = {}) {
  let _ = await PP6({ model: K.model, betas: K.betas }),
      Y = {
        event_name:      q,
        event_id:        randomUUID(),   // crypto.randomUUID()
        core_metadata:   _,             // enriched platform/model/etc.
        user_metadata:   YQ(true),      // deviceId, accountUuid, org, etc.
        event_metadata:  K,             // raw event-specific fields
      },
      z = XL();                         // user/device ID
  if (z) Y.user_id = z;
  A.emit({ body: q, attributes: Y });
}
```

| Field | Description |
|-------|-------------|
| `event_name` | Event string (e.g. `tengu_api_success`) |
| `event_id` | `crypto.randomUUID()` — unique per emission |
| `core_metadata` | Platform, model, beta flags, env context |
| `user_metadata` | deviceId, accountUuid, organizationUuid, sessionId |
| `event_metadata` | All event-specific data fields |
| `user_id` | Device/user ID when authenticated |

#### Gating

First-party OTEL is gated by `r56("firstParty")` (via `tengu_frond_boric` config). If the `firstParty` key in `tengu_frond_boric` is `true`, the first-party path is **killed**. This is the kill-switch for internal logging.

```javascript
var To3 = "tengu_frond_boric";   // cli.js:174134
// r56("firstParty") returns true  → OTEL disabled
// r56("firstParty") returns false → OTEL enabled (default)
```

#### Shutdown

`a56()` (`shutdown1PEventLogging`) calls `vt.shutdown()` which flushes the batch queue before process exit.

#### GrowthBook-Driven Reconfiguration

`Ro3()` (`reinitialize1PEventLoggingIfConfigChanged`) compares the current batch config against the cached value (`eC7`). If changed, it force-flushes the existing provider, shuts it down, and calls `AI7()` to reinitialize with the new config.

#### Third-Party OTEL (3P telemetry for operators)

**Source:** `src/conversation/session-2.ts:96-118`

A separate OTEL path exists for third-party operators (Bedrock/Vertex/Foundry). It uses `s2()` which emits to a separately configured logger.

```javascript
async function s2(A, q = {}) {
  let _ = {
    ...Xf6(),                         // session attributes
    "event.name":      A,
    "event.timestamp": new Date().toISOString(),
    "event.sequence":  Ny9++,         // monotonic sequence counter
  };
  K.emit({ body: `claude_code.${A}`, attributes: _ });
}
```

This path is enabled via `CLAUDE_CODE_ENABLE_TELEMETRY=1` for 3P providers.

---

## 4. Event Sampling & Gating

**Source:** `src/conversation/shutdown1peventlogging-2.ts:26-36`, `cli.js:174151-174175`

### Sampling Function

```javascript
function $H8(A) {
  let K = tC7()[A];                          // lookup per-event config
  if (!K) return null;                       // no config → pass through
  let _ = K.sample_rate;
  if (typeof _ !== "number" || _ < 0 || _ > 1) return null;  // invalid
  if (_ >= 1) return null;                   // 100% → pass through
  if (_ <= 0) return 0;                      // 0% → always drop
  return Math.random() < _ ? _ : 0;         // probabilistic sample
}
```

**Return values:**
- `null` → event passes through (no sampling applied)
- `0` → event is **dropped** (sampled out)
- `0 < n ≤ 1` → event **passes** with sample rate `n` attached as `sample_rate` property

### Config Source

```javascript
function tC7() {
  return LG("tengu_event_sampling_config", {});  // GrowthBook dynamic config
}
```

The `tengu_event_sampling_config` GrowthBook feature is a JSON object mapping event names to `{ sample_rate: number }` objects. Example:

```json
{
  "tengu_api_success": { "sample_rate": 0.1 },
  "tengu_bash_tool_command_executed": { "sample_rate": 0.05 }
}
```

### Backend Kill Switches

```javascript
var oyq = "tengu_log_segment_events";   // GrowthBook gate for Segment
var syq = "tengu_log_datadog_events";   // GrowthBook gate for Datadog
```

Both are checked via `d_()` (Statsig/GrowthBook gate check) at sink initialization and cached in `gt1` / `pt1`:

```javascript
async function Ft1() {
  gt1 = d_(oyq);   // cache Segment gate
  pt1 = d_(syq);   // cache Datadog gate
}
```

---

## 5. Disabling Telemetry

**Source:** `cli.js:52093-52105`

### Master Disable Function: `DL()`

```javascript
function DL() {
  return (
    a6(process.env.CLAUDE_CODE_USE_BEDROCK) ||
    a6(process.env.CLAUDE_CODE_USE_VERTEX)  ||
    a6(process.env.CLAUDE_CODE_USE_FOUNDRY) ||
    !!process.env.DISABLE_TELEMETRY          ||
    !!process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC
  );
}
```

`a6()` is a truthiness check for environment variable strings (handles `"1"`, `"true"`, `"yes"`, etc.).

### Error Reporting Disable: `yq8()`

```javascript
function yq8() {
  return (
    !!process.env.DISABLE_TELEMETRY ||
    !!process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC
    // Note: does NOT check 3P provider flags
  );
}
```

### 1P Event Logging Enable Check: `o56()`

```javascript
function o56() {
  return !DL();   // src/conversation/shutdown1peventlogging-2.ts:44-46
}
```

### Environment Variable Summary

| Variable | Effect | Scope |
|----------|--------|-------|
| `DISABLE_TELEMETRY` | Disables ALL telemetry (Segment, Datadog, OTEL, error reporting) | All |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Disables all telemetry + error reporting | All |
| `CLAUDE_CODE_USE_BEDROCK` | Auto-disables Segment + Datadog + 1P OTEL | 3P Bedrock |
| `CLAUDE_CODE_USE_VERTEX` | Auto-disables Segment + Datadog + 1P OTEL | 3P Vertex |
| `CLAUDE_CODE_USE_FOUNDRY` | Auto-disables Segment + Datadog + error reporting | 3P Foundry |
| `DISABLE_ERROR_REPORTING` | Disables error reporting only | Error queue |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | Explicitly enables 3P OTEL for Bedrock/Vertex/Foundry | 3P providers |

### Subprocess Env Scrubbing

**Source:** `src/core/ux8-1.ts` (cli.js:218153-218166)

```javascript
function nB() {
  if (!a6(process.env.CLAUDE_CODE_SUBPROCESS_ENV_SCRUB)) return process.env;
  let A = { ...process.env };
  for (let q of nD9) {
    delete A[q];
    delete A[`INPUT_${q}`];
  }
  return A;
}
```

When `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1`, all variables listed in `nD9` (includes OTEL headers, OTLP credentials) are stripped from subprocess environments. The full list includes:
`OTEL_EXPORTER_OTLP_HEADERS`, `OTEL_EXPORTER_OTLP_LOGS_HEADERS`, `OTEL_EXPORTER_OTLP_METRICS_HEADERS`, `OTEL_EXPORTER_OTLP_TRACES_HEADERS`, and ~60 other sensitive env vars.

---

## 6. OpenTelemetry Metrics

**Source:** `cli.js:397629-397660` (3P metrics setup), `src/conversation/session-2.ts:45-71`

### Metric Names

| Metric Name | Unit | Description |
|-------------|------|-------------|
| `claude_code.session.count` | sessions | Incremented on session start/end |
| `claude_code.lines_of_code.count` | lines | Lines of code added/removed per edit |
| `claude_code.pull_request.count` | PRs | Pull requests created |
| `claude_code.commit.count` | commits | Git commits made |
| `claude_code.cost.usage` | USD | Estimated API cost |
| `claude_code.token.usage` | tokens | Token consumption (input/output/cacheRead/cacheCreation) |
| `claude_code.code_edit_tool.decision` | — | Tool accept/reject decisions |
| `claude_code.active_time.total` | seconds | Active interaction time |
| `claude_code.interaction` | — | User interaction events |
| `claude_code.llm_request` | — | LLM API request metrics |
| `claude_code.tool` | — | Tool invocation metrics |
| `claude_code.tool.blocked_on_user` | — | Tool calls pending user approval |
| `claude_code.tool.execution` | — | Tool execution completion |
| `claude_code.hook` | — | Hook execution metrics |

### Metric Attributes (from `Xf6()` function)

```javascript
function Xf6() {
  let A = XL(),                    // user/device ID
      K = { "user.id": A };

  if (aN1("OTEL_METRICS_INCLUDE_SESSION_ID"))  K["session.id"] = E8();
  if (aN1("OTEL_METRICS_INCLUDE_VERSION"))     K["app.version"] = "2.1.81";

  let _ = x3();   // auth session
  if (_) {
    if (_.organizationUuid) K["organization.id"] = _.organizationUuid;
    if (_.emailAddress)     K["user.email"] = _.emailAddress;
    if (_.accountUuid && aN1("OTEL_METRICS_INCLUDE_ACCOUNT_UUID")) {
      K["user.account_uuid"] = _.accountUuid;
      K["user.account_id"]   = process.env.CLAUDE_CODE_ACCOUNT_TAGGED_ID
                                || O44("user", _.accountUuid);
    }
  }
  if (KT.terminal) K["terminal.type"] = KT.terminal;
  return K;
}
```

**Default values for OTEL metric config toggles:**

```javascript
ky9 = {
  OTEL_METRICS_INCLUDE_SESSION_ID:   true,   // session.id included by default
  OTEL_METRICS_INCLUDE_VERSION:      false,  // app.version excluded by default
  OTEL_METRICS_INCLUDE_ACCOUNT_UUID: true,   // account UUID included by default
};
```

---

## 7. OpenTelemetry Configuration

**Source:** `cli.js:355500-355560`, `src/conversation/session-2.ts:85-94`, `cli.js:397629-397660`

### Environment Variables

#### Metric Attribute Inclusion

| Variable | Default | Effect |
|----------|---------|--------|
| `OTEL_METRICS_INCLUDE_SESSION_ID` | `true` | Include `session.id` in metric attributes |
| `OTEL_METRICS_INCLUDE_VERSION` | `false` | Include `app.version` in metric attributes |
| `OTEL_METRICS_INCLUDE_ACCOUNT_UUID` | `true` | Include `user.account_uuid` in metric attributes |

#### Privacy

| Variable | Default | Effect |
|----------|---------|--------|
| `OTEL_LOG_USER_PROMPTS` | `false` | When `false`, prompt text is replaced with `"<REDACTED>"` |

#### Export Configuration

| Variable | Default | Effect |
|----------|---------|--------|
| `OTEL_LOGS_EXPORTER` | — | Override logs exporter (e.g. `otlp`, `console`) |
| `OTEL_METRICS_EXPORTER` | — | Override metrics exporter |
| `OTEL_TRACES_EXPORTER` | — | Override traces exporter |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | — | OTLP endpoint base URL |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | — | Protocol (`grpc`, `http/protobuf`, `http/json`) |
| `OTEL_EXPORTER_OTLP_HEADERS` | — | Key=value headers for OTLP export |
| `OTEL_EXPORTER_OTLP_LOGS_HEADERS` | — | Headers for logs exporter only |
| `OTEL_EXPORTER_OTLP_METRICS_HEADERS` | — | Headers for metrics exporter only |
| `OTEL_EXPORTER_OTLP_TRACES_HEADERS` | — | Headers for traces exporter only |
| `OTEL_EXPORTER_OTLP_LOGS_PROTOCOL` | — | Protocol for logs exporter |
| `OTEL_EXPORTER_OTLP_METRICS_PROTOCOL` | — | Protocol for metrics exporter |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | `delta` | Temporality (`cumulative` or `delta`) |
| `OTEL_EXPORTER_OTLP_TIMEOUT` | — | Export timeout (ms) |
| `OTEL_EXPORTER_OTLP_COMPRESSION` | — | Compression (`gzip` or `none`) |

#### TLS / Certificate

| Variable | Effect |
|----------|--------|
| `OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE` | Client cert path |
| `OTEL_EXPORTER_OTLP_CLIENT_KEY` | Client key path |
| `OTEL_EXPORTER_OTLP_CERTIFICATE` | CA cert path |
| `OTEL_EXPORTER_OTLP_METRICS_CLIENT_CERTIFICATE` | Metrics-specific client cert |
| `OTEL_EXPORTER_OTLP_METRICS_CLIENT_KEY` | Metrics-specific client key |

#### Timing

| Variable | Default | Effect |
|----------|---------|--------|
| `OTEL_LOGS_EXPORT_INTERVAL` | `10000` | Log export batch interval (ms) |
| `OTEL_METRIC_EXPORT_INTERVAL` | — | Metric export interval (ms) |
| `CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS` | — | Max time to wait for OTEL shutdown |

#### Resource

| Variable | Effect |
|----------|--------|
| `OTEL_RESOURCE_ATTRIBUTES` | Additional resource attributes (key=value pairs) |

#### Prompt Redaction

```javascript
function Vy9() {
  return a6(process.env.OTEL_LOG_USER_PROMPTS);
}
function sP8(A) {
  return Vy9() ? A : "<REDACTED>";
}
```

All user prompt text passes through `sP8()` before being emitted to OTEL.

---

## 8. Error Tracking

**Source:** `cli.js:39492-39576`

Claude Code uses a **custom error tracking system** — no Sentry or third-party crash reporter.

### Architecture

```javascript
var jJK = 100;      // max error buffer size
var i78;            // error ring buffer (array)
var $j6;            // pending error queue (array)
var EC;             // last error seen

// In initialization:
i78 = [];
$j6 = [];
```

### Error Buffer Mechanics

```javascript
// When error arrives:
if (i78.length >= jJK) i78.shift();   // evict oldest (ring buffer)
i78.push(A);                           // add new error

if (EC = A, $j6.length > 0) {
  let q = [...$j6];
  $j6.length = 0;
  // flush pending queue
}
```

### Opt-Out Check

```javascript
// Error reporting is skipped when:
process.env.DISABLE_ERROR_REPORTING         // explicit opt-out
|| process.env.CLAUDE_CODE_USE_FOUNDRY      // Foundry users
|| process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC  // broad disable

// When opted out:
$j6.push({ type: "error", error: q });     // buffer for later (never sent)
```

### MCP Error Logging

```javascript
function B_(A, q) {                          // cli.js:39539
  // A = serverName, q = error message
  $j6.push({ type: "mcpError", serverName: A, error: q });
}

function T3(A, q) {                          // cli.js:39551 (approximate)
  // A = serverName, q = debug message
  $j6.push({ type: "mcpDebug", serverName: A, message: q });
}
```

`B_()` is called throughout MCP code for errors including:
- Elicitation errors and hook errors (`cli.js:323611, 323662, 323694`)
- MCP header helper errors (`cli.js:330584`)
- Reconnection errors (`cli.js:331050`)
- Tool/resource fetch errors (`cli.js:331140`)
- Resource prefetch failures (`cli.js:331184`)

### TelemetrySafeError

```javascript
// cli.js:42569
{
  name: "TelemetrySafeError",
  telemetryMessage: q ?? A   // sanitized version for telemetry
}
```

`TelemetrySafeError` holds two messages:
- `message` — full error message (shown to user, may contain paths/details)
- `telemetryMessage` — sanitized version safe for telemetry transmission (no PII, no local paths)

---

## 9. Privacy & Data Handling

### Credential Storage

- Sensitive plugin options (API keys, tokens) stored in macOS Keychain when available, falling back to `~/.claude/.credentials.json`
- Keychain errors logged via `tengu_api_key_keychain_error`

### Path Sanitization

`iK(A)` sanitizes file paths before inclusion in telemetry:
1. Converts absolute paths to relative paths (from project root)
2. Replaces home directory prefix with `~`
3. Returns original path as fallback

### HTTP Request Sanitization

`ev.prototype._sanitizeOptions` strips sensitive headers (Authorization, API keys) before requests are logged.

### Home Directory Expansion

Path fields that expand to the home directory are replaced with `~` to prevent leaking system usernames.

### Prompt Redaction

User prompt text is replaced with `"<REDACTED>"` in OTEL metrics unless `OTEL_LOG_USER_PROMPTS=1` is explicitly set.

### WSL Detection

On WSL environments, the WSL version is added as `wsl.version` resource attribute.

### Subprocess Isolation

With `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1`, all OTEL and credential env vars are stripped from subprocess environments to prevent leakage via child processes.

---

## 10. Complete Event Reference

All 781 events are organized by category. For each category: event pattern, trigger context, and key data fields.

---

### 10.1 Core API Events (`tengu_api_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_api_success` | Successful API response received | `requestId`, `querySource`, `model`, `inputTokens`, `outputTokens`, `cachedInputTokens`, `uncachedInputTokens`, `durationMsIncludingRetries`, `timeSinceLastApiCallMs` |
| `tengu_api_error` | API call fails | `requestId`, `model`, `error`, `status`, `retryCount` |
| `tengu_api_query` | API query initiated | `model`, `querySource`, `inputTokens` |
| `tengu_api_retry` | Transient error triggers retry | `retryCount`, `error`, `model` |
| `tengu_api_persistent_retry_wait` | Persistent retry backoff entered | `waitMs`, `retryCount` |
| `tengu_api_custom_529_overloaded_error` | 529 overload response received | `model`, `status` |
| `tengu_api_opus_fallback_triggered` | Opus model fallback activated | `fromModel`, `toModel` |
| `tengu_api_cache_breakpoints` | Cache breakpoint logic executed | `count`, `model` |
| `tengu_api_before_normalize` | Pre-normalization hook | `model` |
| `tengu_api_after_normalize` | Post-normalization hook | `model` |
| `tengu_api_key_keychain_error` | Keychain access for API key failed | `error` |
| `tengu_api_key_saved_to_config` | API key persisted to config file | — |
| `tengu_api_key_saved_to_keychain` | API key persisted to keychain | — |
| `tengu_unknown_model_cost` | Cost calculation failed (unknown model) | `model`, `shortName` |
| `tengu_fast_mode_fallback_triggered` | Fast mode fell back to standard | `reason` |
| `tengu_fast_mode_overage_rejected` | Fast mode budget exceeded | `overage` |
| `tengu_org_penguin_mode_fetch_failed` | Org penguin mode fetch error | — |
| `tengu_model_fallback_triggered` | Generic model fallback (Datadog) | `fromModel`, `toModel`, `reason` |
| `tengu_refusal_api_response` | API returned refusal content | `model` |
| `tengu_nonstreaming_fallback_error` | Non-streaming fallback error | `error` |
| `tengu_nonstreaming_fallback_started` | Non-streaming fallback initiated | `reason` |
| `tengu_structured_output_enabled` | Structured output format activated | `model` |
| `tengu_structured_output_failure` | Structured output parsing failed | `error` |

---

### 10.2 Authentication Events (`tengu_oauth_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_oauth_flow_start` | OAuth flow begins | `platform` |
| `tengu_oauth_auth_code_received` | Authorization code obtained | `platform` |
| `tengu_oauth_token_exchange_success` | Token exchange succeeded | `platform` |
| `tengu_oauth_token_exchange_error` | Token exchange failed | `error`, `platform` |
| `tengu_oauth_success` | Full OAuth flow completed | `platform`, `userType` |
| `tengu_oauth_error` | OAuth flow failed | `error`, `platform` |
| `tengu_oauth_manual_entry` | User manually entered API key | — |
| `tengu_oauth_api_key` | API key auth used | — |
| `tengu_oauth_automatic_redirect` | Browser redirect auto-triggered | `platform` |
| `tengu_oauth_automatic_redirect_error` | Auto-redirect failed | `error` |
| `tengu_oauth_platform_selected` | Provider platform chosen | `platform` |
| `tengu_oauth_claudeai_selected` | Claude.ai selected as provider | — |
| `tengu_oauth_claudeai_forced` | Claude.ai provider forced | `reason` |
| `tengu_oauth_console_selected` | Console selected as provider | — |
| `tengu_oauth_console_forced` | Console provider forced | `reason` |
| `tengu_oauth_profile_fetch_success` | Profile data fetched post-auth | `hasEmail` |
| `tengu_oauth_roles_stored` | User roles persisted | — |
| `tengu_oauth_storage_warning` | Token storage warning | `warning` |
| `tengu_oauth_tokens_saved` | Tokens successfully persisted | `storage` |
| `tengu_oauth_tokens_save_failed` | Token persistence failed | `error` |
| `tengu_oauth_tokens_save_exception` | Exception during token save | `error` |
| `tengu_oauth_tokens_inference_only` | Tokens are inference-only scope | — |
| `tengu_oauth_tokens_not_claude_ai` | Tokens not from Claude.ai | — |
| `tengu_oauth_token_refresh_starting` | Token refresh initiated | `platform` |
| `tengu_oauth_token_refresh_success` | Token refresh succeeded | `platform` |
| `tengu_oauth_token_refresh_failure` | Token refresh failed | `error`, `platform` |
| `tengu_oauth_token_refresh_completed` | Token refresh cycle complete | `platform` |
| `tengu_oauth_token_refresh_lock_acquiring` | Acquiring distributed refresh lock | — |
| `tengu_oauth_token_refresh_lock_acquired` | Lock acquired | `waitMs` |
| `tengu_oauth_token_refresh_lock_releasing` | Releasing refresh lock | — |
| `tengu_oauth_token_refresh_lock_released` | Lock released | — |
| `tengu_oauth_token_refresh_lock_error` | Lock operation error | `error` |
| `tengu_oauth_token_refresh_lock_retry` | Lock acquire retry | `attempt` |
| `tengu_oauth_token_refresh_lock_retry_limit_reached` | Lock retry exhausted | `attempts` |
| `tengu_oauth_token_refresh_race_recovered` | Race condition recovered | — |
| `tengu_oauth_token_refresh_race_resolved` | Race condition resolved | — |
| `tengu_oauth_401_recovered_from_keychain` | 401 handled via keychain fallback | — |
| `tengu_grove_oauth_401_received` | 401 in Grove OAuth flow | — |
| `tengu_login_from_refresh_token` | Login via stored refresh token | — |
| `tengu_mcp_oauth_flow_start` | MCP server OAuth flow started | `serverName` |
| `tengu_mcp_oauth_flow_success` | MCP OAuth completed | `serverName` |
| `tengu_mcp_oauth_flow_error` | MCP OAuth failed | `serverName`, `error` |

---

### 10.3 Session Events (`tengu_session_*`, `tengu_started`, `tengu_exit`, `tengu_init`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_started` | Process startup complete | `version`, `platform`, `sessionId` |
| `tengu_init` | Initialization sequence complete | `version`, `hasApiKey`, `model` |
| `tengu_exit` | Process exit | `exitCode`, `reason` |
| `tengu_session_resumed` | Previous session loaded | `sessionId`, `messageCount` |
| `tengu_session_renamed` | Session given a name | `sessionId` |
| `tengu_session_rename_started` | Rename flow begun | — |
| `tengu_session_title_generated` | Auto-generated title created | `sessionId` |
| `tengu_session_file_read` | Session file loaded from disk | `sessionId`, `size` |
| `tengu_session_memory_loaded` | Session memory file loaded | `count` |
| `tengu_session_memory_accessed` | Session memory read during query | — |
| `tengu_session_memory_extraction` | Memory extracted from conversation | `count` |
| `tengu_session_memory_file_read` | Memory file read from disk | — |
| `tengu_session_persistence_failed` | Session save failed | `error` |
| `tengu_session_forked_branches_fetched` | Forked session branches loaded | `count` |
| `tengu_session_all_projects_toggled` | All-projects view toggled | `enabled` |
| `tengu_session_branch_filter_toggled` | Branch filter toggled | `enabled` |
| `tengu_session_search_toggled` | Session search activated | — |
| `tengu_session_group_expanded` | Session group expanded in UI | — |
| `tengu_session_linked_to_pr` | Session linked to pull request | — |
| `tengu_session_preview_opened` | Session preview opened | — |
| `tengu_session_tagged` | Session tagged with label | `tag` |
| `tengu_session_tag_filter_changed` | Tag filter changed | `tag` |
| `tengu_session_worktree_filter_toggled` | Worktree filter toggled | `enabled` |
| `tengu_concurrent_sessions` | Multiple concurrent sessions detected | `count` |
| `tengu_began_setup` | First-run setup started | — |
| `tengu_onboarding_step` | Onboarding step completed | `step` |
| `tengu_startup_telemetry` | Telemetry initialization checkpoint | — |
| `tengu_startup_manual_model_config` | Manual model configured at startup | `model` |
| `tengu_headless_latency` | Headless mode startup time | `latencyMs` |

---

### 10.4 Tool Use Events (`tengu_tool_use_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_tool_use_success` | Tool execution completed successfully | `toolName`, `durationMs` |
| `tengu_tool_use_error` | Tool execution failed | `toolName`, `error` |
| `tengu_tool_use_granted_in_prompt_permanent` | User granted permanent tool permission | `toolName` |
| `tengu_tool_use_granted_in_prompt_temporary` | User granted one-time permission | `toolName` |
| `tengu_tool_use_rejected_in_prompt` | User rejected tool at runtime | `toolName` |
| `tengu_tool_use_denied_in_config` | Tool blocked by configuration | `toolName` |
| `tengu_tool_use_granted_in_config` | Tool allowed by configuration | `toolName` |
| `tengu_tool_use_granted_by_classifier` | Tool allowed by AI classifier | `toolName`, `confidence` |
| `tengu_tool_use_granted_by_permission_hook` | Permission hook granted access | `toolName` |
| `tengu_tool_use_show_permission_request` | Permission dialog shown to user | `toolName` |
| `tengu_tool_use_cancelled` | Tool call cancelled by user | `toolName` |
| `tengu_tool_use_can_use_tool_allowed` | Tool use pre-check passed | `toolName` |
| `tengu_tool_use_can_use_tool_rejected` | Tool use pre-check failed | `toolName` |
| `tengu_tool_use_progress` | Tool execution in progress update | `toolName`, `progressPct` |
| `tengu_tool_use_diff_computed` | Diff computed for edit tool | `toolName`, `additions`, `deletions` |
| `tengu_tool_use_tool_result_mismatch_error` | Tool result ID mismatch | `toolName` |
| `tengu_tool_empty_result` | Tool returned empty result | `toolName` |
| `tengu_tool_result_persisted` | Tool result saved to disk (large) | `toolName`, `size` |
| `tengu_tool_result_persisted_message_budget` | Tool result persisted due to budget | `toolName` |
| `tengu_unexpected_tool_result` | Unexpected tool result received | `toolName` |
| `tengu_tool_search_outcome` | Tool search completed | `query`, `resultCount` |
| `tengu_tool_search_mode_decision` | Tool search mode selected | `mode` |
| `tengu_tool_input_alias_applied` | Input alias mapping applied | `toolName`, `alias` |
| `tengu_tool_result_pairing_repaired` | Tool result pair mismatch repaired | `toolName` |
| `tengu_deferred_tool_schema_not_sent` | Deferred tool schema not transmitted | `toolName` |
| `tengu_deferred_tools_pool_change` | Deferred tools pool updated | `delta` |
| `tengu_message_level_tool_result_budget_enforced` | Budget enforced on tool result | — |

---

### 10.5 Bash Tool Events (`tengu_bash_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_bash_tool_command_executed` | Bash command ran | `exitCode`, `durationMs`, `isBackground` |
| `tengu_bash_tool_simple_echo` | Simple echo command detected | — |
| `tengu_bash_tool_reset_to_original_dir` | CWD restored after command | `dir` |
| `tengu_bash_security_check_triggered` | Security pattern matched | `pattern`, `command` |
| `tengu_bash_ast_too_complex` | AST analysis skipped (too complex) | `commandLength` |
| `tengu_bash_command_explicitly_backgrounded` | Command run in background | `command` |
| `tengu_input_bash` | Bash input processed | `length` |
| `tengu_shell_set_cwd` | Shell working directory changed | `cwd` |
| `tengu_shell_snapshot_error` | Shell state snapshot error | `error` |
| `tengu_shell_snapshot_failed` | Shell snapshot creation failed | `error` |
| `tengu_shell_unknown_error` | Unknown shell error | `error` |
| `tengu_shell_completion_failed` | Tab completion failed | `error` |
| `tengu_tree_sitter_security_divergence` | Tree-sitter and regex security diverge | `command` |
| `tengu_git_operation` | Git operation executed | `operation`, `exitCode` |
| `tengu_git_index_lock_error` | Git index.lock contention | `path` |

---

### 10.6 File Operation Events (`tengu_file_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_file_operation` | File read/write/edit completed | `operation`, `toolName`, `size` |
| `tengu_file_changed` | File changed on disk (watch) | `path` |
| `tengu_file_read_limits_override` | File read limits bypassed | `reason` |
| `tengu_file_upload_failed` | File upload to API failed | `error` |
| `tengu_file_suggestions_query` | File autocomplete triggered | `query` |
| `tengu_file_suggestions_git_ls_files` | File suggestions via git ls-files | `count` |
| `tengu_file_suggestions_ripgrep` | File suggestions via ripgrep | `count` |
| `tengu_atomic_write_error` | Atomic file write failed (fallback) | `path` |
| `tengu_binary_content_persisted` | Binary content saved to disk | `size`, `mimeType` |
| `tengu_file_history_snapshot_success` | File history snapshot created | `path` |
| `tengu_file_history_snapshot_failed` | Snapshot creation failed | `error` |
| `tengu_file_history_backup_file_created` | Backup created before edit | `path` |
| `tengu_file_history_backup_deleted_file` | Backup created for deleted file | `path` |
| `tengu_file_history_backup_file_failed` | Backup creation failed | `error` |
| `tengu_file_history_track_edit_success` | Edit tracked in file history | `path` |
| `tengu_file_history_track_edit_failed` | Edit tracking failed | `error` |
| `tengu_file_history_rewind_success` | Rewind operation succeeded | `path`, `steps` |
| `tengu_file_history_rewind_failed` | Rewind failed | `error` |
| `tengu_file_history_rewind_restore_file_failed` | File restoration during rewind failed | `error` |
| `tengu_file_history_resume_copy_failed` | Resume copy failed | `error` |
| `tengu_file_history_snapshots_setting_changed` | Snapshots setting toggled | `enabled` |
| `tengu_dir_search` | Directory search performed | `query`, `resultCount` |
| `tengu_ripgrep_availability` | ripgrep availability checked | `available` |
| `tengu_ripgrep_eagain_retry` | ripgrep EAGAIN retry | `attempt` |

---

### 10.7 Streaming Events (`tengu_streaming_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_streaming_error` | Streaming response error | `error`, `model` |
| `tengu_streaming_stall` | Stream stall detected | `stallMs`, `model` |
| `tengu_streaming_stall_summary` | Stall summary at end of stream | `totalStalls`, `maxStallMs` |
| `tengu_streaming_idle_timeout` | Stream timed out (idle) | `timeoutMs` |
| `tengu_streaming_fallback_to_non_streaming` | Fell back to non-streaming | `reason` |
| `tengu_streaming_tool_execution_used` | Streaming tool execution path used | `toolName` |
| `tengu_streaming_tool_execution_not_used` | Non-streaming tool execution used | `toolName` |
| `tengu_stream_no_events` | Stream returned no events | `model` |
| `tengu_stream_loop_exited_after_watchdog` | Stream watchdog fired | `durationMs` |
| `tengu_query_error` | Query-level error | `error`, `model` |
| `tengu_query_before_attachments` | Query initiated before attachments | — |
| `tengu_query_after_attachments` | Query continued after attachments | — |

---

### 10.8 MCP Events (`tengu_mcp_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_mcp_server_connection_succeeded` | MCP server connected | `serverName`, `transport` |
| `tengu_mcp_server_connection_failed` | MCP server connection failed | `serverName`, `error` |
| `tengu_mcp_server_needs_auth` | MCP server requires auth | `serverName` |
| `tengu_mcp_session_expired` | MCP session expired | `serverName` |
| `tengu_mcp_ide_server_connection_succeeded` | IDE MCP server connected | `serverName` |
| `tengu_mcp_ide_server_connection_failed` | IDE MCP server failed | `serverName`, `error` |
| `tengu_mcp_tools_commands_loaded` | MCP tools and commands loaded | `serverName`, `toolCount` |
| `tengu_mcp_headers` | MCP headers processed | `serverName` |
| `tengu_mcp_tool_call_auth_error` | Auth error during MCP tool call | `serverName`, `toolName` |
| `tengu_mcp_claudeai_proxy_401` | Claude.ai MCP proxy returned 401 | `serverName` |
| `tengu_mcp_elicitation_shown` | MCP elicitation UI shown | `serverName` |
| `tengu_mcp_elicitation_response` | User responded to elicitation | `serverName`, `action` |
| `tengu_mcp_add` | MCP server added | `serverName`, `transport` |
| `tengu_mcp_delete` | MCP server removed | `serverName` |
| `tengu_mcp_get` | MCP server config retrieved | `serverName` |
| `tengu_mcp_dialog_choice` | Trust dialog choice made | `choice` |
| `tengu_mcp_auth_config_authenticate` | MCP auth config authenticate | `serverName` |
| `tengu_mcp_auth_config_clear` | MCP auth config cleared | `serverName` |
| `tengu_mcp_channel_flags` | MCP channel flags updated | `serverName` |
| `tengu_mcp_channel_gate` | MCP channel gate checked | `serverName` |
| `tengu_mcp_channel_message` | MCP channel message sent | `serverName` |
| `tengu_mcp_instructions_pool_change` | MCP instructions pool changed | `delta` |
| `tengu_claudeai_mcp_eligibility` | Claude.ai MCP eligibility checked | `eligible` |
| `tengu_claudeai_mcp_auth_started` | Claude.ai MCP auth flow started | — |
| `tengu_claudeai_mcp_auth_completed` | Claude.ai MCP auth completed | — |
| `tengu_claudeai_mcp_clear_auth_started` | Claude.ai MCP auth clear started | — |
| `tengu_claudeai_mcp_clear_auth_completed` | Claude.ai MCP auth clear completed | — |
| `tengu_claudeai_mcp_reconnect` | Claude.ai MCP reconnected | — |
| `tengu_claudeai_mcp_toggle` | Claude.ai MCP enabled/disabled | `enabled` |
| `tengu_trust_dialog_accept` | MCP trust dialog accepted | `serverName` |
| `tengu_trust_dialog_shown` | MCP trust dialog shown | `serverName` |

---

### 10.9 Agent Events (`tengu_agent_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_agent_created` | Sub-agent instantiated | `agentId`, `model` |
| `tengu_agent_tool_completed` | Sub-agent tool finished | `agentId`, `toolName`, `durationMs` |
| `tengu_agent_tool_selected` | Sub-agent selected a tool | `agentId`, `toolName` |
| `tengu_agent_tool_terminated` | Sub-agent tool terminated early | `agentId`, `toolName`, `reason` |
| `tengu_agent_memory_loaded` | Agent memory loaded | `agentId`, `count` |
| `tengu_agent_parse_error` | Agent output parse error | `agentId`, `error` |
| `tengu_agent_stop_hook_success` | Stop hook executed successfully | `agentId` |
| `tengu_agent_stop_hook_error` | Stop hook failed | `agentId`, `error` |
| `tengu_agent_stop_hook_max_turns` | Stop hook max turns reached | `agentId`, `turns` |
| `tengu_agent_definition_generated` | Agent definition auto-generated | `agentId` |
| `tengu_agent_name_set` | Agent given a name | `agentId`, `name` |
| `tengu_agent_color_set` | Agent color assigned | `agentId`, `color` |
| `tengu_agent_flag` | Agent flag set | `agentId`, `flag` |
| `tengu_at_mention_agent_success` | @mention resolved to agent | `agentId` |
| `tengu_at_mention_agent_not_found` | @mention agent not found | `mention` |
| `tengu_at_mention_extracting_filename_error` | Filename extraction error | `error` |
| `tengu_at_mention_extracting_directory_success` | Directory extraction succeeded | — |
| `tengu_at_mention_mcp_resource_success` | MCP resource @mention succeeded | `resource` |
| `tengu_at_mention_mcp_resource_error` | MCP resource @mention failed | `error` |
| `tengu_subagent_at_mention` | Sub-agent @mention used | `agentId` |
| `tengu_fork_agent_query` | Agent query forked | `agentId` |
| `tengu_agentic_search_cancelled` | Agentic search cancelled | — |
| `tengu_concurrent_onquery_detected` | Concurrent query conflict detected | — |
| `tengu_concurrent_onquery_enqueued` | Concurrent query queued | — |

---

### 10.10 Teams Events (`tengu_team_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_team_created` | Team created | `teamId` |
| `tengu_team_deleted` | Team deleted | `teamId` |
| `tengu_team_mem_sync_started` | Team memory sync started | `teamId` |
| `tengu_team_mem_sync_pull` | Team memory pull completed | `teamId`, `count` |
| `tengu_team_mem_sync_push` | Team memory push completed | `teamId`, `count` |
| `tengu_team_mem_entries_capped` | Team memory entry limit reached | `teamId`, `cap` |
| `tengu_team_mem_accessed` | Team memory read | `teamId` |
| `tengu_team_mem_file_read` | Team memory file read | `teamId` |
| `tengu_team_mem_file_write` | Team memory file written | `teamId` |
| `tengu_team_mem_file_edit` | Team memory file edited | `teamId` |
| `tengu_team_mem_push_suppressed` | Team memory push suppressed | `reason` |
| `tengu_team_mem_secret_skipped` | Secret field skipped in mem sync | `teamId` |
| `tengu_team_memdir_disabled` | Team memory directory disabled | `reason` |
| `tengu_teammate_mode_changed` | Teammate mode setting changed | `mode` |

---

### 10.11 Bridge Events (`tengu_bridge_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_bridge_started` | Bridge process started | `mode` |
| `tengu_bridge_shutdown` | Bridge process shutting down | `reason` |
| `tengu_bridge_message_received` | Message received from bridge | `type` |
| `tengu_bridge_command` | Bridge command executed | `command` |
| `tengu_bridge_heartbeat_error` | Bridge heartbeat failed | `error` |
| `tengu_bridge_heartbeat_mode_entered` | Bridge heartbeat mode entered | — |
| `tengu_bridge_heartbeat_mode_exited` | Bridge heartbeat mode exited | — |
| `tengu_bridge_reconnected` | Bridge reconnected after drop | `attempts` |
| `tengu_bridge_registration_failed` | Bridge registration failed | `error` |
| `tengu_bridge_poll_give_up` | Bridge polling gave up | `attempts` |
| `tengu_bridge_session_started` | Bridge session started | `sessionId` |
| `tengu_bridge_session_done` | Bridge session completed | `sessionId` |
| `tengu_bridge_session_timeout` | Bridge session timed out | `sessionId` |
| `tengu_bridge_token_refreshed` | Bridge auth token refreshed | — |
| `tengu_bridge_work_secret_failed` | Bridge work secret failed | `error` |
| `tengu_bridge_fatal_error` | Bridge fatal error | `error` |
| `tengu_bridge_spawn_mode_chosen` | Bridge spawn mode selected | `mode` |
| `tengu_bridge_spawn_mode_toggled` | Bridge spawn mode toggled | `mode` |
| `tengu_bridge_repl_started` | REPL bridge started | `sessionId` |
| `tengu_bridge_repl_teardown` | REPL bridge teardown | `sessionId` |
| `tengu_bridge_repl_started` | REPL bridge started | — |
| `tengu_bridge_repl_connect_timeout` | REPL bridge connection timed out | `timeoutMs` |
| `tengu_bridge_repl_skipped` | REPL bridge skipped | `reason` |
| `tengu_bridge_repl_env_registered` | REPL env registered | — |
| `tengu_bridge_repl_env_lost` | REPL env lost | `reason` |
| `tengu_bridge_repl_env_expired_fresh_session` | REPL env expired (fresh session) | — |
| `tengu_bridge_repl_reconnected_in_place` | REPL reconnected in-place | — |
| `tengu_bridge_repl_reconnect_failed` | REPL reconnect failed | `error` |
| `tengu_bridge_repl_fatal_error` | REPL fatal error | `error` |
| `tengu_bridge_repl_session_failed` | REPL session failed | `error` |
| `tengu_bridge_repl_poll_error` | REPL polling error | `error` |
| `tengu_bridge_repl_poll_give_up` | REPL polling gave up | `attempts` |
| `tengu_bridge_repl_work_received` | REPL work unit received | — |
| `tengu_bridge_repl_work_secret_failed` | REPL work secret failed | `error` |
| `tengu_bridge_repl_ws_connected` | REPL WebSocket connected | — |
| `tengu_bridge_repl_ws_closed` | REPL WebSocket closed | `code`, `reason` |
| `tengu_bridge_repl_suspension_detected` | REPL suspension detected | — |
| `tengu_bridge_repl_history_capped` | REPL history capped | `cap` |
| `tengu_bridge_repl_ccr_v2_init_failed` | REPL CCR v2 init failed | `error` |
| `tengu_ws_transport_closed` | WebSocket transport closed | `code` |
| `tengu_ws_transport_reconnecting` | WebSocket transport reconnecting | `attempt` |
| `tengu_ws_transport_reconnected` | WebSocket transport reconnected | `attempts` |

---

### 10.12 Teleport Events (`tengu_teleport_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_teleport_started` | Teleport operation started | `sessionId`, `mode` |
| `tengu_teleport_print` | Teleport print output | `sessionId` |
| `tengu_teleport_source_decision` | Teleport source determined | `source` |
| `tengu_teleport_interactive_mode` | Teleport interactive mode entered | — |
| `tengu_teleport_bundle_mode` | Teleport bundle mode entered | — |
| `tengu_teleport_cancelled` | Teleport cancelled by user | — |
| `tengu_teleport_resume_session` | Teleport session resumed | `sessionId` |
| `tengu_teleport_resume_error` | Teleport resume error | `error` |
| `tengu_teleport_first_message_success` | First teleport message succeeded | — |
| `tengu_teleport_first_message_error` | First teleport message failed | `error` |
| `tengu_teleport_errors_detected` | Errors detected in teleport | `count` |
| `tengu_teleport_errors_resolved` | Errors resolved | `count` |
| `tengu_teleport_error_branch_checkout_failed` | Branch checkout failed | `branch` |
| `tengu_teleport_error_git_not_clean` | Git working tree not clean | — |
| `tengu_teleport_error_repo_mismatch_sessions_api` | Repo mismatch with sessions API | — |
| `tengu_teleport_error_repo_not_in_git_dir_sessions_api` | Repo not in git dir (API) | — |
| `tengu_teleport_error_session_not_found_404` | Session 404 from API | — |
| `tengu_remote_create_session` | Remote session create initiated | — |
| `tengu_remote_create_session_success` | Remote session created | `sessionId` |
| `tengu_remote_create_session_error` | Remote session creation failed | `error` |
| `tengu_remote_setup_started` | Remote setup started | — |
| `tengu_remote_setup_result` | Remote setup result | `success` |
| `tengu_review_remote_launched` | Remote review launched | — |
| `tengu_review_remote_precondition_failed` | Remote review precondition failed | `reason` |
| `tengu_review_remote_teleport_failed` | Remote review teleport failed | `error` |

---

### 10.13 Voice Events (`tengu_voice_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_voice_toggled` | Voice input toggled on/off | `enabled` |
| `tengu_voice_recording_started` | Voice recording began | — |
| `tengu_voice_recording_completed` | Voice recording finished | `durationMs`, `sizeBytes` |
| `tengu_voice_silent_drop_replay` | Silent audio dropped and replayed | — |
| `tengu_voice_stream_early_retry` | Voice stream early retry | `attempt` |

---

### 10.14 Plugin Events (`tengu_plugin_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_plugins_loaded` | All plugins loaded at startup | `count` |
| `tengu_plugin_installed` | Plugin installed | `pluginId`, `version` |
| `tengu_plugin_install_command` | Plugin install command run | `pluginId` |
| `tengu_plugin_installed_cli` | Plugin installed via CLI | `pluginId` |
| `tengu_plugin_uninstall_command` | Plugin uninstall command run | `pluginId` |
| `tengu_plugin_uninstalled_cli` | Plugin uninstalled via CLI | `pluginId` |
| `tengu_plugin_enable_command` | Plugin enable command run | `pluginId` |
| `tengu_plugin_enabled_cli` | Plugin enabled via CLI | `pluginId` |
| `tengu_plugin_disable_command` | Plugin disable command run | `pluginId` |
| `tengu_plugin_disabled_cli` | Plugin disabled via CLI | `pluginId` |
| `tengu_plugin_disabled_all_cli` | All plugins disabled via CLI | — |
| `tengu_plugin_list_command` | Plugin list command run | — |
| `tengu_plugin_update_command` | Plugin update command run | `pluginId` |
| `tengu_plugin_updated_cli` | Plugin updated via CLI | `pluginId`, `fromVersion`, `toVersion` |
| `tengu_ext_installed` | Extension installed | `extensionId` |
| `tengu_ext_install_error` | Extension install failed | `error` |
| `tengu_ext_diff_accepted` | Extension diff accepted | `extensionId` |
| `tengu_ext_diff_rejected` | Extension diff rejected | `extensionId` |
| `tengu_ext_will_show_diff` | Extension about to show diff | `extensionId` |
| `tengu_ext_ide_command` | IDE extension command run | `command` |
| `tengu_ext_at_mentioned` | Extension @mentioned | `extensionId` |
| `tengu_headless_plugin_install` | Plugin installed in headless mode | `pluginId` |
| `tengu_sync_plugin_install_timeout` | Plugin sync install timed out | `pluginId` |
| `tengu_marketplace_added` | Marketplace item added | `itemId` |
| `tengu_marketplace_removed` | Marketplace item removed | `itemId` |
| `tengu_marketplace_updated` | Marketplace item updated | `itemId` |
| `tengu_marketplace_updated_all` | All marketplace items updated | `count` |
| `tengu_marketplace_background_install` | Background marketplace install | `itemId` |
| `tengu_official_marketplace_auto_install` | Official marketplace auto-install | `itemId` |
| `tengu_skill_loaded` | Skill loaded | `skillName` |
| `tengu_skill_tool_invocation` | Skill tool invoked | `skillName`, `toolName` |
| `tengu_skill_tool_slash_prefix` | Skill slash prefix used | `skillName` |
| `tengu_skill_file_changed` | Skill file changed on disk | `skillName` |
| `tengu_skill_improvement_survey` | Skill improvement survey shown | `skillName` |
| `tengu_dynamic_skills_changed` | Dynamic skills pool changed | `delta` |

---

### 10.15 Update Events (`tengu_update_*`, `tengu_native_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_update_check` | Update check initiated | `currentVersion` |
| `tengu_version_check_success` | Version check succeeded | `currentVersion`, `latestVersion` |
| `tengu_version_check_failure` | Version check failed | `error` |
| `tengu_version_lock_acquired` | Version lock acquired | — |
| `tengu_version_lock_failed` | Version lock failed | `error` |
| `tengu_autoupdate_enabled` | Auto-update enabled/disabled | `enabled` |
| `tengu_autoupdate_channel_changed` | Update channel changed | `channel` |
| `tengu_auto_updater_success` | Auto-update succeeded | `fromVersion`, `toVersion` |
| `tengu_auto_updater_fail` | Auto-update failed | `error` |
| `tengu_auto_updater_lock_contention` | Update lock contention | — |
| `tengu_auto_updater_windows_npm_in_wsl` | WSL Windows npm update attempted | — |
| `tengu_native_install_binary_success` | Native binary installed | `platform`, `version` |
| `tengu_native_install_binary_failure` | Native binary install failed | `error`, `platform` |
| `tengu_native_install_package_success` | Native package installed | `package`, `version` |
| `tengu_native_install_package_failure` | Native package install failed | `error`, `package` |
| `tengu_native_update_complete` | Native update complete | `fromVersion`, `toVersion` |
| `tengu_native_update_lock_failed` | Native update lock failed | `error` |
| `tengu_native_update_skipped_max_version` | Update skipped (max version) | `version` |
| `tengu_native_update_skipped_minimum_version` | Update skipped (min version) | `version` |
| `tengu_native_version_cleanup` | Old versions cleaned up | `count` |
| `tengu_native_staging_cleanup` | Staging dir cleaned | — |
| `tengu_native_stale_locks_cleanup` | Stale locks cleaned | `count` |
| `tengu_native_temp_files_cleanup` | Temp files cleaned | `count` |
| `tengu_binary_download_attempt` | Binary download started | `url`, `platform` |
| `tengu_binary_download_success` | Binary download succeeded | `url`, `size` |
| `tengu_binary_download_failure` | Binary download failed | `error`, `url` |
| `tengu_binary_manifest_fetch_failure` | Manifest fetch failed | `error` |
| `tengu_binary_platform_not_found` | Platform binary not in manifest | `platform` |
| `tengu_auto_migrate_to_native_attempt` | Native migration attempted | — |
| `tengu_auto_migrate_to_native_success` | Native migration succeeded | — |
| `tengu_auto_migrate_to_native_failure` | Native migration failed | `error` |
| `tengu_auto_migrate_to_native_partial` | Partial native migration | `step` |
| `tengu_auto_migrate_to_native_ui_shown` | Migration UI shown | — |
| `tengu_auto_migrate_to_native_ui_success` | Migration UI success | — |
| `tengu_auto_migrate_to_native_ui_error` | Migration UI error | `error` |

---

### 10.16 UI Interaction Events

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_cancel` | User cancelled (Ctrl+C or Esc) | `phase`, `turnCount` |
| `tengu_help_toggled` | Help panel opened/closed | `visible` |
| `tengu_toggle_todos` | TODO panel toggled | `visible` |
| `tengu_toggle_transcript` | Transcript view toggled | `visible` |
| `tengu_input_prompt` | User prompt submitted | `length`, `hasAttachments` |
| `tengu_input_command` | Slash command entered | `command` |
| `tengu_input_slash_invalid` | Invalid slash command | `command` |
| `tengu_input_slash_missing` | Unknown slash command | `command` |
| `tengu_immediate_command_executed` | Immediate command run | `command` |
| `tengu_single_word_prompt` | Single-word prompt detected | `word` |
| `tengu_copy` | Content copied to clipboard | `type` |
| `tengu_paste_text` | Text pasted | `length` |
| `tengu_paste_image` | Image pasted | `size` |
| `tengu_pasted_image_resize_attempt` | Pasted image resize attempted | — |
| `tengu_message_actions_enter` | Message actions bar entered | — |
| `tengu_accept_submitted` | Accept submitted | `toolName` |
| `tengu_reject_submitted` | Reject submitted | `toolName` |
| `tengu_accept_feedback_mode_entered` | Accept feedback mode entered | — |
| `tengu_accept_feedback_mode_collapsed` | Accept feedback mode collapsed | — |
| `tengu_reject_feedback_mode_entered` | Reject feedback mode entered | — |
| `tengu_reject_feedback_mode_collapsed` | Reject feedback mode collapsed | — |
| `tengu_diff_tool_changed` | Diff tool selection changed | `tool` |
| `tengu_external_editor_hint_shown` | External editor hint shown | `editor` |
| `tengu_external_editor_used` | External editor opened | `editor` |
| `tengu_plan_external_editor_used` | External editor used in plan mode | `editor` |
| `tengu_transcript_view_enter` | Transcript view opened | — |
| `tengu_transcript_view_exit` | Transcript view closed | — |
| `tengu_transcript_toggle_show_all` | Show all messages toggled | `showAll` |
| `tengu_transcript_exit` | Transcript exited | — |
| `tengu_transcript_accessed` | Transcript read | — |
| `tengu_transcript_input_to_teammate` | Input sent to teammate in transcript | — |
| `tengu_transcript_parent_cycle` | Transcript parent cycle detected | — |
| `tengu_permission_request_escape` | Permission request escaped | `toolName` |
| `tengu_permission_request_option_selected` | Permission option chosen | `toolName`, `option` |
| `tengu_bypass_permissions_mode_dialog_shown` | Bypass permissions dialog shown | — |
| `tengu_bypass_permissions_mode_dialog_accept` | Bypass permissions accepted | — |
| `tengu_auto_mode_opt_in_dialog_shown` | Auto-mode opt-in dialog shown | — |
| `tengu_auto_mode_opt_in_dialog_accept` | Auto-mode opt-in accepted | — |
| `tengu_auto_mode_opt_in_dialog_accept_default` | Auto-mode default accepted | — |
| `tengu_auto_mode_opt_in_dialog_decline` | Auto-mode opt-in declined | — |
| `tengu_thinking_toggled` | Thinking mode toggled | `enabled` |
| `tengu_thinking_toggled_hotkey` | Thinking toggled via hotkey | `enabled` |
| `tengu_ultrathink` | Ultrathink triggered | — |
| `tengu_effort_command` | Effort level command run | `level` |
| `tengu_tip_shown` | Tip shown to user | `tipId` |
| `tengu_feedback_survey_event` | Survey interaction | `surveyId`, `action` |
| `tengu_skill_improvement_survey` | Skill improvement survey | `skillName` |
| `tengu_post_compact_survey_event` | Post-compact survey | `action` |
| `tengu_bug_report_submitted` | Bug report submitted | — |
| `tengu_status_line_mount` | Status line mounted | — |
| `tengu_fast_mode_picker_shown` | Fast mode picker shown | — |
| `tengu_fast_mode_toggled` | Fast mode toggled | `enabled` |
| `tengu_desktop_upsell_shown` | Desktop upsell shown | — |
| `tengu_guest_passes_upsell_shown` | Guest passes upsell shown | — |
| `tengu_guest_passes_link_copied` | Guest pass link copied | — |
| `tengu_guest_passes_visited` | Guest passes visited | — |
| `tengu_switch_to_subscription_notice_shown` | Subscription switch notice shown | — |
| `tengu_rate_limit_options_menu_cancel` | Rate limit menu cancelled | — |
| `tengu_rate_limit_options_menu_select_extra_usage` | Extra usage selected | — |
| `tengu_rate_limit_options_menu_select_upgrade` | Upgrade selected | — |
| `tengu_cost_threshold_reached` | Cost threshold reached | `threshold`, `actual` |
| `tengu_cost_threshold_acknowledged` | User acknowledged cost threshold | — |
| `tengu_max_tokens_reached` | Max token context reached | `model` |
| `tengu_context_window_exceeded` | Context window exceeded | `model`, `tokens` |
| `tengu_max_tokens_context_overflow_adjustment` | Context adjusted for overflow | `removed` |
| `tengu_context_size` | Context size tracked | `tokens`, `model` |

---

### 10.17 Performance Events (`tengu_startup_perf`, `tengu_compact_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_startup_perf` | Startup performance marks collected | `checkpoint_count`, `import_time_ms`, `init_time_ms`, `settings_time_ms`, `total_time_ms` |
| `tengu_compact` | Compact operation completed | `durationMs`, `inputTokens`, `outputTokens` |
| `tengu_compact_failed` | Compact operation failed | `error` |
| `tengu_compact_streaming_retry` | Compact retry during streaming | `attempt` |
| `tengu_compact_cache_sharing_success` | Cache sharing in compact succeeded | — |
| `tengu_compact_cache_sharing_fallback` | Cache sharing fell back | `reason` |
| `tengu_partial_compact` | Partial compact performed | `removed` |
| `tengu_partial_compact_failed` | Partial compact failed | `error` |
| `tengu_auto_compact_succeeded` | Auto-compact triggered and succeeded | — |
| `tengu_auto_compact_setting_changed` | Auto-compact setting changed | `enabled` |
| `tengu_post_autocompact_turn` | Turn after auto-compact | — |
| `tengu_time_based_microcompact` | Time-based micro-compact triggered | — |
| `tengu_sm_compact_threshold_exceeded` | Session memory compact threshold | `size` |
| `tengu_sm_compact_error` | Session memory compact error | `error` |
| `tengu_sm_compact_empty_template` | Empty template in sm compact | — |
| `tengu_sm_compact_no_session_memory` | No session memory to compact | — |
| `tengu_sm_compact_resumed_session` | Resumed session sm compact | — |
| `tengu_sm_compact_summarized_id_not_found` | Summarized ID not found | `id` |
| `tengu_auto_dream_fired` | Auto-dream scheduled task fired | — |
| `tengu_auto_dream_completed` | Auto-dream completed | `durationMs` |
| `tengu_auto_dream_failed` | Auto-dream failed | `error` |
| `tengu_auto_dream_toggled` | Auto-dream toggled | `enabled` |
| `tengu_scheduled_task_fire` | Scheduled task fired | `taskId` |
| `tengu_scheduled_task_expired` | Scheduled task expired | `taskId` |
| `tengu_scheduled_task_missed` | Scheduled task missed | `taskId` |
| `tengu_heap_dump` | Heap dump taken | `size` |
| `tengu_timer` | Generic timer event | `label`, `durationMs` |

---

### 10.18 Error Events (`tengu_uncaught_exception`, `tengu_unhandled_rejection`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_uncaught_exception` | Unhandled exception in process | `error`, `stack`, `version` |
| `tengu_unhandled_rejection` | Unhandled Promise rejection | `error`, `stack`, `version` |
| `tengu_node_warning` | Node.js process warning | `name`, `message` |

---

### 10.19 Config Events (`tengu_config_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_config_changed` | Config file modified | `key`, `source` |
| `tengu_config_model_changed` | Model setting changed | `model`, `source` |
| `tengu_config_parse_error` | Config file parse error | `error`, `path` |
| `tengu_config_lock_contention` | Config file lock contention | `lockFile` |
| `tengu_config_stale_write` | Stale config write detected | `key` |
| `tengu_config_cache_stats` | Config cache statistics | `hits`, `misses` |
| `tengu_config_auth_loss_prevented` | Auth loss during config write prevented | — |
| `tengu_claudemd__initial_load` | CLAUDE.md file initially loaded | `path`, `size` |
| `tengu_claude_md_permission_error` | CLAUDE.md permission denied | `path` |
| `tengu_claude_rules_md_permission_error` | CLAUDE_RULES.md permission denied | `path` |
| `tengu_claude_md_external_includes_dialog_accepted` | External includes accepted | — |
| `tengu_claude_md_external_includes_dialog_declined` | External includes declined | — |
| `tengu_claude_md_includes_dialog_shown` | External includes dialog shown | — |
| `tengu_managed_settings_loaded` | Managed settings loaded | `count` |
| `tengu_managed_settings_security_dialog_shown` | Security dialog shown | — |
| `tengu_managed_settings_security_dialog_accepted` | Security dialog accepted | — |
| `tengu_managed_settings_security_dialog_rejected` | Security dialog rejected | — |
| `tengu_write_claudemd` | CLAUDE.md write command run | — |

---

### 10.20 Model Events (`tengu_model_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_sonnet45_to_46_migration` | Sonnet 4.5→4.6 model migration | `from_model`, `has_1m` |
| `tengu_opus_to_opus1m_migration` | Opus→Opus 1M migration | — |
| `tengu_reset_pro_to_opus_default` | Pro model reset to Opus default | `skipped`, `had_custom_model` |
| `tengu_legacy_opus_migration` | Legacy Opus model migration | `fromModel` |
| `tengu_startup_manual_model_config` | Manual model at startup | `model` |
| `tengu_off_switch_query` | Off-switch query issued | `model` |
| `tengu_speculation` | Speculative query issued | `model` |
| `tengu_sepia_heron_applied` | Sepia heron prompt applied | `model` |

---

### 10.21 Plan Mode Events (`tengu_plan_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_plan_enter` | Plan mode entered | — |
| `tengu_plan_exit` | Plan mode exited | `completedSteps` |
| `tengu_exit_plan_mode_called_outside_plan` | Plan exit called outside plan | — |
| `tengu_ask_user_question_accepted` | User accepted plan question | `questionId` |
| `tengu_ask_user_question_rejected` | User rejected plan question | `questionId` |
| `tengu_ask_user_question_respond_to_claude` | User responded to Claude | — |
| `tengu_ask_user_question_finish_plan_interview` | Plan interview completed | — |

---

### 10.22 Auto Mode Events (`tengu_auto_mode_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_auto_mode_decision` | Auto-mode permission decision | `decision`, `toolName` |
| `tengu_auto_mode_outcome` | Auto-mode session outcome | `toolsUsed`, `toolsDenied` |
| `tengu_auto_mode_malformed_tool_input` | Malformed tool input in auto-mode | `toolName` |
| `tengu_auto_mode_denial_limit_exceeded` | Auto-mode denial limit exceeded | `limit` |

---

### 10.23 Hooks Events (`tengu_*hook*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_run_hook` | Hook executed | `hookType`, `exitCode` |
| `tengu_pre_tool_hook_error` | Pre-tool hook error | `hookType`, `error` |
| `tengu_pre_tool_hooks_cancelled` | Pre-tool hooks cancelled | `toolName` |
| `tengu_post_tool_hook_error` | Post-tool hook error | `hookType`, `error` |
| `tengu_post_tool_hooks_cancelled` | Post-tool hooks cancelled | `toolName` |
| `tengu_post_tool_failure_hook_error` | Post-tool failure hook error | `error` |
| `tengu_post_tool_failure_hooks_cancelled` | Post-tool failure hooks cancelled | — |
| `tengu_pre_stop_hooks_cancelled` | Pre-stop hooks cancelled | — |
| `tengu_stop_hook_error` | Stop hook error | `error` |
| `tengu_repl_hook_finished` | REPL hook finished | `exitCode` |
| `tengu_hooks_command` | Hooks command run | `action` |

---

### 10.24 Image / Attachment Events

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_attachments` | Attachment(s) added to query | `count`, `types` |
| `tengu_attachment_compute_duration` | Attachment processing time | `durationMs`, `type` |
| `tengu_attachment_file_too_large` | Attachment too large | `size`, `limit` |
| `tengu_image_api_validation_failed` | Image API validation failed | `error` |
| `tengu_image_compress_failed` | Image compression failed | `error` |
| `tengu_image_resize_failed` | Image resize failed | `error` |
| `tengu_image_resize_fallback` | Image resize fell back to original | — |
| `tengu_pdf_page_extraction` | PDF page extracted | `pages` |
| `tengu_pdf_reference_attachment` | PDF referenced as attachment | `pages` |

---

### 10.25 Memory Events (`tengu_memdir_*`, `tengu_extract_memories_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_memdir_loaded` | Memory directory loaded | `count` |
| `tengu_memdir_disabled` | Memory directory disabled | `reason` |
| `tengu_auto_memory_toggled` | Auto memory toggled | `enabled` |
| `tengu_extract_memories_extraction` | Memories extracted from conversation | `count` |
| `tengu_extract_memories_coalesced` | Duplicate memories coalesced | `before`, `after` |
| `tengu_extract_memories_skipped_direct_write` | Memory extraction skipped (direct write) | — |

---

### 10.26 Worktree Events (`tengu_worktree_*`)

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_worktree_detection` | Worktree detected | `type` |
| `tengu_worktree_created` | Worktree created | `path` |
| `tengu_worktree_removed` | Worktree removed | `path` |
| `tengu_worktree_kept` | Worktree retained | `path`, `reason` |
| `tengu_worktree_cleanup` | Worktree cleanup performed | `count` |

---

### 10.27 Conversation Management

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_conversation_forked` | Conversation forked | `sessionId`, `forkPoint` |
| `tengu_conversation_rewind` | Conversation rewound | `sessionId`, `steps` |
| `tengu_continue` | `/continue` command | — |
| `tengu_continue_print` | Continue print output | — |
| `tengu_resume_print` | Resume print output | — |
| `tengu_slash_command_forked` | Slash command forked conversation | `command` |
| `tengu_filtered_orphaned_thinking_message` | Orphaned thinking block filtered | — |
| `tengu_filtered_trailing_thinking_block` | Trailing thinking block filtered | — |
| `tengu_filtered_whitespace_only_assistant` | Whitespace-only assistant message filtered | — |
| `tengu_fixed_empty_assistant_content` | Empty assistant content fixed | — |
| `tengu_orphaned_messages_tombstoned` | Orphaned messages tombstoned | `count` |
| `tengu_chain_parallel_tr_recovered` | Parallel tool result chain recovered | — |
| `tengu_chain_parent_cycle` | Parent cycle detected in chain | — |
| `tengu_relink_walk_broken` | Relink walk broken | — |
| `tengu_sysprompt_block` | System prompt block added | `type` |
| `tengu_sysprompt_boundary_found` | System prompt boundary found | — |
| `tengu_sysprompt_missing_boundary_marker` | Boundary marker missing | — |
| `tengu_sysprompt_using_tool_based_cache` | Tool-based cache used for sysprompt | — |
| `tengu_cache_eviction_hint` | Cache eviction hint emitted | — |

---

### 10.28 Grove / Privacy Events

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_grove_print_viewed` | Grove print output viewed | — |
| `tengu_grove_policy_viewed` | Grove privacy policy viewed | — |
| `tengu_grove_policy_submitted` | Grove policy submitted | — |
| `tengu_grove_policy_dismissed` | Grove policy dismissed | — |
| `tengu_grove_policy_escaped` | Grove policy escaped | — |
| `tengu_grove_policy_exited` | Grove policy exited | — |
| `tengu_grove_policy_toggled` | Grove policy toggled | `enabled` |
| `tengu_grove_privacy_settings_viewed` | Grove privacy settings viewed | — |
| `tengu_claudeai_limits_status_changed` | Claude.ai limits changed | `status` |

---

### 10.29 CCR / Remote Review

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_ccr_bundle_upload` | CCR bundle uploaded | `size` |
| `tengu_ccr_unsupported_default_mode_ignored` | CCR unsupported mode ignored | `mode` |

---

### 10.30 GitHub / CI Events

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_install_github_app_started` | GitHub App install started | `repo` |
| `tengu_install_github_app_completed` | GitHub App install completed | `repo` |
| `tengu_install_github_app_error` | GitHub App install error | `error` |
| `tengu_install_github_app_step_completed` | GitHub App install step done | `step` |
| `tengu_install_slack_app_clicked` | Slack App install clicked | — |
| `tengu_setup_github_actions_started` | GitHub Actions setup started | — |
| `tengu_setup_github_actions_completed` | GitHub Actions setup completed | — |
| `tengu_setup_github_actions_failed` | GitHub Actions setup failed | `error` |

---

### 10.31 Settings / Preferences

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_default_view_setting_changed` | Default view changed | `view` |
| `tengu_output_style_changed` | Output style changed | `style` |
| `tengu_language_changed` | Language setting changed | `language` |
| `tengu_editor_mode_changed` | Editor mode changed | `mode`, `source` |
| `tengu_reduce_motion_setting_changed` | Reduce motion toggled | `enabled` |
| `tengu_show_turn_duration_setting_changed` | Turn duration display toggled | `enabled` |
| `tengu_tips_setting_changed` | Tips display toggled | `enabled` |
| `tengu_respect_gitignore_setting_changed` | Gitignore respect toggled | `enabled` |
| `tengu_pr_status_footer_setting_changed` | PR status footer toggled | `enabled` |
| `tengu_terminal_progress_bar_setting_changed` | Progress bar toggled | `enabled` |
| `tengu_auto_connect_ide_changed` | Auto-connect IDE changed | `enabled` |
| `tengu_auto_install_ide_extension_changed` | Auto-install IDE extension changed | `enabled` |

---

### 10.32 Remaining / Miscellaneous Events

| Event | Trigger | Key Fields |
|-------|---------|------------|
| `tengu_setup_token_command` | Setup token command run | — |
| `tengu_doctor_command` | Doctor command run | — |
| `tengu_claude_install_command` | Claude install command run | — |
| `tengu_unary_event` | Unary (no-arg) event | `name` |
| `tengu_notification_method_used` | Notification method used | `method` |
| `tengu_preflight_check_failed` | Preflight check failed | `check`, `error` |
| `tengu_repo_text_file_size` | Repo text file size tracked | `size` |
| `tengu_code_prompt_ignored` | Code prompt ignored | `reason` |
| `tengu_code_indexing_tool_used` | Code indexing tool invoked | `tool` |
| `tengu_brief_mode_enabled` | Brief mode enabled | — |
| `tengu_brief_mode_toggled` | Brief mode toggled | `enabled`, `gated`, `source` |
| `tengu_brief_send` | Brief output sent | `size` |
| `tengu_keybinding_fallback_used` | Keybinding fell back to default | `action`, `context`, `fallback`, `reason` |
| `tengu_custom_keybindings_loaded` | Custom keybindings loaded | `user_binding_count` |
| `tengu_flicker` | UI flicker detected | — |
| `tengu_stdin_interactive` | stdin is interactive | — |
| `tengu_prompt_suggestion` | Prompt suggestion shown | `suggestionId` |
| `tengu_prompt_suggestion_init` | Prompt suggestion initialized | — |
| `tengu_permission_explainer_shortcut_used` | Permission explainer shortcut used | — |
| `tengu_permission_explainer_generated` | Permission explainer generated | `toolName` |
| `tengu_permission_explainer_error` | Permission explainer error | `error` |
| `tengu_cache_eviction_hint` | Cache eviction hinted | — |
| `tengu_claude_in_chrome_setup` | Claude in Chrome setup | `success` |
| `tengu_claude_in_chrome_setup_failed` | Chrome setup failed | `error` |
| `tengu_claude_in_chrome_onboarding_shown` | Chrome onboarding shown | — |
| `tengu_claude_in_chrome_setting_changed` | Chrome setting changed | `setting` |

---

## 11. Datadog Critical Events (46 events)

**Source:** `cli.js:352181-352225` (the `P9_` Set)

Only these 46 events are transmitted to Datadog. All others are silently dropped at the `zb1()` fanout check:

```javascript
P9_ = new Set([
  // Chrome bridge
  "chrome_bridge_connection_succeeded",
  "chrome_bridge_connection_failed",
  "chrome_bridge_disconnected",
  "chrome_bridge_tool_call_completed",
  "chrome_bridge_tool_call_error",
  "chrome_bridge_tool_call_started",
  "chrome_bridge_tool_call_timeout",

  // Core API
  "tengu_api_error",
  "tengu_api_success",

  // Brief mode
  "tengu_brief_mode_enabled",
  "tengu_brief_mode_toggled",
  "tengu_brief_send",

  // User interaction
  "tengu_cancel",

  // Compact
  "tengu_compact_failed",

  // Session lifecycle
  "tengu_exit",
  "tengu_init",
  "tengu_started",

  // Flicker
  "tengu_flicker",

  // Model
  "tengu_model_fallback_triggered",

  // Auth
  "tengu_oauth_error",
  "tengu_oauth_success",
  "tengu_oauth_token_refresh_failure",
  "tengu_oauth_token_refresh_success",
  "tengu_oauth_token_refresh_lock_acquiring",
  "tengu_oauth_token_refresh_lock_acquired",
  "tengu_oauth_token_refresh_starting",
  "tengu_oauth_token_refresh_completed",
  "tengu_oauth_token_refresh_lock_releasing",
  "tengu_oauth_token_refresh_lock_released",

  // Query
  "tengu_query_error",

  // Performance
  "tengu_repo_text_file_size",
  "tengu_session_file_read",

  // Tool use
  "tengu_tool_use_error",
  "tengu_tool_use_granted_in_prompt_permanent",
  "tengu_tool_use_granted_in_prompt_temporary",
  "tengu_tool_use_rejected_in_prompt",
  "tengu_tool_use_success",

  // Errors
  "tengu_uncaught_exception",
  "tengu_unhandled_rejection",

  // Voice
  "tengu_voice_recording_started",
  "tengu_voice_toggled",

  // Team memory
  "tengu_team_mem_sync_pull",
  "tengu_team_mem_sync_push",
  "tengu_team_mem_sync_started",
  "tengu_team_mem_entries_capped",
]);
```

| # | Event | Purpose |
|---|-------|---------|
| 1 | `chrome_bridge_connection_succeeded` | Chrome bridge health monitoring |
| 2 | `chrome_bridge_connection_failed` | Chrome bridge failure detection |
| 3 | `chrome_bridge_disconnected` | Chrome bridge disconnect tracking |
| 4 | `chrome_bridge_tool_call_completed` | Chrome tool execution success |
| 5 | `chrome_bridge_tool_call_error` | Chrome tool execution failure |
| 6 | `chrome_bridge_tool_call_started` | Chrome tool call initiated |
| 7 | `chrome_bridge_tool_call_timeout` | Chrome tool call timeout |
| 8 | `tengu_api_error` | API failure rate monitoring |
| 9 | `tengu_api_success` | API success rate + latency monitoring |
| 10 | `tengu_brief_mode_enabled` | Brief mode adoption |
| 11 | `tengu_brief_mode_toggled` | Brief mode toggle tracking |
| 12 | `tengu_brief_send` | Brief output volume |
| 13 | `tengu_cancel` | User cancellation rate |
| 14 | `tengu_compact_failed` | Compact failure rate |
| 15 | `tengu_exit` | Session end tracking |
| 16 | `tengu_flicker` | UI quality monitoring |
| 17 | `tengu_init` | Initialization success tracking |
| 18 | `tengu_model_fallback_triggered` | Model fallback rate |
| 19 | `tengu_oauth_error` | Auth failure rate |
| 20 | `tengu_oauth_success` | Auth success rate |
| 21 | `tengu_oauth_token_refresh_failure` | Token refresh failure rate |
| 22 | `tengu_oauth_token_refresh_success` | Token refresh success rate |
| 23 | `tengu_oauth_token_refresh_lock_acquiring` | Lock contention monitoring |
| 24 | `tengu_oauth_token_refresh_lock_acquired` | Lock acquisition success |
| 25 | `tengu_oauth_token_refresh_starting` | Token refresh initiation |
| 26 | `tengu_oauth_token_refresh_completed` | Token refresh completion |
| 27 | `tengu_oauth_token_refresh_lock_releasing` | Lock release start |
| 28 | `tengu_oauth_token_refresh_lock_released` | Lock release completion |
| 29 | `tengu_query_error` | Query failure rate |
| 30 | `tengu_repo_text_file_size` | Repository size monitoring |
| 31 | `tengu_session_file_read` | Session persistence monitoring |
| 32 | `tengu_started` | Session start tracking |
| 33 | `tengu_team_mem_entries_capped` | Team memory cap hits |
| 34 | `tengu_team_mem_sync_pull` | Team sync pull activity |
| 35 | `tengu_team_mem_sync_push` | Team sync push activity |
| 36 | `tengu_team_mem_sync_started` | Team sync initiation |
| 37 | `tengu_tool_use_error` | Tool failure rate |
| 38 | `tengu_tool_use_granted_in_prompt_permanent` | Permanent permission grants |
| 39 | `tengu_tool_use_granted_in_prompt_temporary` | Temporary permission grants |
| 40 | `tengu_tool_use_rejected_in_prompt` | Permission rejection rate |
| 41 | `tengu_tool_use_success` | Tool success rate |
| 42 | `tengu_uncaught_exception` | Critical error rate |
| 43 | `tengu_unhandled_rejection` | Promise rejection rate |
| 44 | `tengu_voice_recording_started` | Voice feature adoption |
| 45 | `tengu_voice_toggled` | Voice toggle rate |
| 46 | `tengu_flicker` | (already listed #16) |

> **Note:** The set literal in source contains exactly 46 entries including `tengu_flicker` appearing once.

---

## 12. Quick Reference

### 12.1 Telemetry Environment Variables

#### Disabling

| Variable | Effect |
|----------|--------|
| `DISABLE_TELEMETRY` | Disables all telemetry |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Disables all telemetry + error reporting |
| `CLAUDE_CODE_USE_BEDROCK` | Auto-disables (Bedrock users) |
| `CLAUDE_CODE_USE_VERTEX` | Auto-disables (Vertex users) |
| `CLAUDE_CODE_USE_FOUNDRY` | Auto-disables (Foundry users) |
| `DISABLE_ERROR_REPORTING` | Disables error queue only |

#### Enabling

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_ENABLE_TELEMETRY` | Re-enables OTEL for 3P provider users |
| `OTEL_LOG_USER_PROMPTS` | Enables prompt content in OTEL (default: redacted) |

#### OTEL Metrics Attributes

| Variable | Default | Effect |
|----------|---------|--------|
| `OTEL_METRICS_INCLUDE_SESSION_ID` | `true` | Include session ID in metrics |
| `OTEL_METRICS_INCLUDE_VERSION` | `false` | Include version in metrics |
| `OTEL_METRICS_INCLUDE_ACCOUNT_UUID` | `true` | Include account UUID in metrics |

#### OTEL Export

| Variable | Default | Effect |
|----------|---------|--------|
| `OTEL_LOGS_EXPORTER` | — | Logs exporter type |
| `OTEL_METRICS_EXPORTER` | — | Metrics exporter type |
| `OTEL_TRACES_EXPORTER` | — | Traces exporter type |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | — | OTLP endpoint |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | — | OTLP protocol |
| `OTEL_EXPORTER_OTLP_HEADERS` | — | OTLP headers |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | `delta` | Metric temporality |
| `OTEL_LOGS_EXPORT_INTERVAL` | `10000` | Log batch interval (ms) |
| `OTEL_METRIC_EXPORT_INTERVAL` | — | Metric export interval (ms) |
| `CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS` | — | OTEL shutdown timeout |
| `OTEL_RESOURCE_ATTRIBUTES` | — | Extra resource attributes |

#### Datadog

| Variable | Default | Effect |
|----------|---------|--------|
| `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS` | `15000` | Datadog flush interval |

#### Subprocess

| Variable | Default | Effect |
|----------|---------|--------|
| `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` | — | Strip OTEL vars from subprocesses |

---

### 12.2 OTEL Metric Names

```
claude_code.session.count
claude_code.lines_of_code.count
claude_code.pull_request.count
claude_code.commit.count
claude_code.cost.usage
claude_code.token.usage
claude_code.code_edit_tool.decision
claude_code.active_time.total
claude_code.interaction
claude_code.llm_request
claude_code.tool
claude_code.tool.blocked_on_user
claude_code.tool.execution
claude_code.hook
```

---

### 12.3 Transmission Endpoints

| Backend | Endpoint | Auth |
|---------|----------|------|
| Segment (production) | `https://api.segment.io` (SDK default) | Write key: `LKJN8LsLERHEOXkw487o7qCTFOrGPimI` |
| Segment (development) | `https://api.segment.io` (SDK default) | Write key: `b64sf1kxwDGe1PiSAlv5ixuH0f509RKK` |
| Datadog | `https://http-intake.logs.us5.datadoghq.com/api/v2/logs` | API key: `pubbbf48e6d78dae54bceaa4acf463299bf` |
| First-party OTEL | Configured via `CX1` exporter (internal Anthropic endpoint) | Bearer token from auth session |
| 3P OTEL | `OTEL_EXPORTER_OTLP_ENDPOINT` | `OTEL_EXPORTER_OTLP_HEADERS` |

---

### 12.4 Sampling Configuration Keys

| Key | Type | Description |
|-----|------|-------------|
| `tengu_event_sampling_config` | GrowthBook dynamic config | Per-event `{ sample_rate: 0-1 }` map |
| `tengu_log_segment_events` | GrowthBook gate | Enable/disable Segment routing |
| `tengu_log_datadog_events` | GrowthBook gate | Enable/disable Datadog routing |
| `tengu_frond_boric` | GrowthBook dynamic config | Per-backend kill switch (`firstParty`, `segment`, `datadog` keys) |
| `tengu_1p_event_batch_config` | GrowthBook dynamic config | Overrides OTEL batch processor config |

---

### 12.5 Source File Index

| File | Contents |
|------|----------|
| `cli.js:4294-4318` | `Q()`, `JKA()`, `jKA()` — core emission functions, pre-sink queue |
| `cli.js:52093-52105` | `DL()` — master telemetry disable check |
| `cli.js:174134` | `To3 = "tengu_frond_boric"` — 1P OTEL kill switch key |
| `cli.js:174151-174298` | `AI7()` — OTEL initialization, `$H8()` — sampling, `GP6()` — 1P emit |
| `cli.js:352107-352310` | `zb1()` — Datadog fanout, `P9_` — 46 event allowlist, `W9_` — tag dimensions |
| `cli.js:354454-354515` | `xb1()` — Segment fanout, write keys, `BS4()` — identify |
| `cli.js:355500-355560` | Telemetry-related env var list |
| `cli.js:39492-39576` | Error queue: `jJK`, `i78`, `$j6`, `B_()`, `T3()`, `TelemetrySafeError` |
| `src/telemetry/initializeanalyticssink-2.ts` | `N26()` sink init, `JJY()`/`MJY()` fanout, backend gates |
| `src/telemetry/events-1.ts` | Module batch: 30 extracted event-emitting modules |
| `src/telemetry/events-2.ts` | Module batch: 4 extracted modules (keybindings, brief, bridge) |
| `src/telemetry/events-3.ts` | Module batch: 3 model migration events |
| `src/telemetry/events-4.ts` | Module batch: 4 modules (WebFetch, WebSearch, effort levels) |
| `src/conversation/shutdown1peventlogging-2.ts` | Full 1P OTEL module: init, shutdown, reinit, sampling |
| `src/conversation/session-2.ts` | `Xf6()` metric attributes, `s2()` 3P OTEL emit, OTEL defaults |
| `src/vendor/opentelemetry.ts` | Vendored OTEL SDK, 743 lines, 22 modules, `LogsAPI`, `MetricsAPI` |
| `src/core/ux8-1.ts` | `nB()` subprocess env scrubbing |
