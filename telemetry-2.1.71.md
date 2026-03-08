# Claude Code v2.1.71 — Telemetry Reference

Source: `cli.js` (612,918 lines, built 2026-03-06T22:45:36Z)

---

## Overview

Claude Code v2.1.71 implements a multi-layered telemetry system with **6 pipelines**, **598 named event types**, and a full OpenTelemetry SDK. Telemetry is routed through a central dispatcher function, with sampling, gating, and kill-switch controls at multiple layers.

**Key facts:**
- All `tengu_*` events go to Anthropic's internal analytics (1P OTEL sink), with optional Segment and Datadog routing gated by feature flags
- 3P OTEL export (metrics, logs, traces) is separately opt-in via `CLAUDE_CODE_ENABLE_TELEMETRY=true`
- Prompt text is redacted by default; file contents are never collected
- Bedrock/Vertex/Foundry users have all internal analytics disabled automatically
- No Sentry SDK; crash reporting uses the normal `tengu_uncaught_exception` event pipeline

---

## Table of Contents

1. [Telemetry Architecture](#1-telemetry-architecture)
   - 1.1 Dispatcher Function
   - 1.2 Pipeline Overview
2. [Telemetry Pipelines (Detailed)](#2-telemetry-pipelines-detailed)
   - 2.1 First-Party OTEL (Anthropic)
   - 2.2 BigQuery Metrics
   - 2.3 DataDog Log Intake
   - 2.4 Segment Analytics
   - 2.5 User-Configured OTEL (3P)
   - 2.6 Beta Tracing
3. [Event Catalog](#3-event-catalog)
   - 3.1 Session Lifecycle Events
   - 3.2 API / LLM Events
   - 3.3 Tool Events
   - 3.4 File Operation Events
   - 3.5 Permission & Hook Events
   - 3.6 Authentication Events
   - 3.7 Error Events
   - 3.8 Feature Gate Events
   - 3.9 UI / UX Events
   - 3.10 Other Events
4. [OTEL Instrumentation](#4-otel-instrumentation)
   - 4.1 Metrics
   - 4.2 Spans
   - 4.3 Log Records
5. [Privacy & Data Controls](#5-privacy--data-controls)
   - 5.1 Default Behavior
   - 5.2 What's Never Sent
   - 5.3 Opt-In Data
   - 5.4 Data Redaction & Anonymization
   - 5.5 Disabling Telemetry
6. [Attribution & Billing](#6-attribution--billing)
7. [Configuration Reference](#7-configuration-reference)
8. [Summary Statistics](#8-summary-statistics)

---

## 1. Telemetry Architecture

### 1.1 Dispatcher Function

All internal analytics telemetry is routed through a single minified function `c()` at **line 4110**:

```js
function c(A, q) {      // A = eventName, q = metadata object
  if (BH6 === null) {
    R81.push({ eventName: A, metadata: q, async: false });
    return;
  }
  BH6.logEvent(A, q);  // BH6 is the analytics sink registered via tKA()
}
```

The sink is registered at startup via `f$6()` → `tKA({ logEvent: Ebz, logEventAsync: Lbz })` (**line 549079**).

The async dispatcher `Ebz()` (**line 549059**) fans out to up to three backends simultaneously:

```js
async function Ebz(A, q) {
  let K = Dx1(A);            // sample rate check
  if (K === 0) return;       // dropped by sampling
  let Y = K !== null ? { ...q, sample_rate: K } : q;
  if (ebq()) sb8(A, Y);     // Segment (if gate enabled)
  if (Axq()) Nb8(A, Y);     // DataDog (if gate enabled)
  Xx1(A, Y);                 // 1P OTEL (always, if !UV())
}
```

**Core metadata** (`mV6()`, **line 544633**) is attached to every event:

| Field | Source |
|-------|--------|
| `model` | Current model string |
| `sessionId` | `d1()` — persistent session UUID |
| `userType` | Always `"external"` |
| `betas` | Active beta flags (comma-separated) |
| `envContext` | Platform, arch, node version, terminal, CI flags, etc. |
| `entrypoint` | `CLAUDE_CODE_ENTRYPOINT` env var |
| `isInteractive` | `"true"` or `"false"` |
| `clientType` | `vH6()` |
| `subscriptionType` | `K3()` — subscription tier |
| `agentSdkVersion` | `CLAUDE_AGENT_SDK_VERSION` |
| `sweBenchRunId/InstanceId/TaskId` | SWE-Bench tracking env vars |

`envContext` includes: `platform`, `arch`, `nodeVersion`, `terminal`, `packageManagers`, `runtimes`, `isCi`, `isClaudeCodeRemote`, `remoteEnvironmentType`, `wslVersion`, `linuxDistroId/Version/Kernel`, `vcs`, `version`, `buildTime`, `isGithubAction`, `isClaudeCodeAction`, `isClaudeAiAuth`, and more.

### 1.2 Pipeline Overview

| # | Pipeline | Destination | What It Sends | Enabled By | Disable By |
|---|----------|-------------|---------------|------------|------------|
| 1 | **1P OTEL** | `api.anthropic.com/api/event_logging/batch` | All 598 `tengu_*` events + GrowthBook experiment data | Default ON | `DISABLE_TELEMETRY`, `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`, Bedrock/Vertex/Foundry |
| 2 | **BigQuery Metrics** | `api.anthropic.com/api/claude_code/metrics` | 8 OTEL counter metrics (sessions, tokens, cost, LOC, etc.) | Team/enterprise subscriptions OR first-party trust | Org opt-out setting |
| 3 | **DataDog** | `https://http-intake.logs.us5.datadoghq.com/api/v2/logs` | ~35 allowlisted events | Gate `tengu_log_datadog_events` + first-party provider | `DISABLE_TELEMETRY`, gate disabled, kill switch |
| 4 | **Segment** | Segment API (write key varies) | All events (same as 1P) | Gate `tengu_log_segment_events` | `DISABLE_TELEMETRY`, gate disabled, kill switch |
| 5 | **User-Configured OTEL (3P)** | User-specified OTLP endpoint | OTEL metrics, logs, and (optionally) traces | `CLAUDE_CODE_ENABLE_TELEMETRY=true` | Not setting the env var |
| 6 | **Beta Tracing** | `${BETA_TRACING_ENDPOINT}/v1/traces` and `/v1/logs` | Full OTEL traces + logs with rich context | `BETA_TRACING_ENDPOINT` + Statsig gate `tengu_trace_lantern` | Not configured |

---

## 2. Telemetry Pipelines (Detailed)

### 2.1 First-Party OTEL (Anthropic)

**Class**: `Re8` (Anthropic Event Exporter, **line 545562**)  
**Module**: `Pbq` — exports `initialize1PEventLogging`, `logEventTo1P`, `shutdown1PEventLogging`  
**Endpoint**: `https://api.anthropic.com/api/event_logging/batch`  
**Staging**: If `ANTHROPIC_BASE_URL=https://api-staging.anthropic.com`, uses staging endpoint  
**Logger name**: `"com.anthropic.claude_code.events"` (**line 546082**)

**Activation** — disabled (`UV()` returns true) when any of:
- `CLAUDE_CODE_USE_BEDROCK=true`
- `CLAUDE_CODE_USE_VERTEX=true`
- `CLAUDE_CODE_USE_FOUNDRY=true`
- `DISABLE_TELEMETRY` set
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` set

**Batching configuration** (configurable via Statsig `tengu_1p_event_batch_config`):

| Parameter | Default |
|-----------|--------|
| `maxBatchSize` | 200 |
| `batchDelayMs` | 100ms |
| `timeout` | 10,000ms |
| `baseBackoffDelayMs` | 500ms |
| `maxBackoffDelayMs` | 30,000ms |
| `maxAttempts` | 8 |
| `scheduledDelayMillis` | 10,000ms |
| `maxExportBatchSize` | 200 |
| `maxQueueSize` | 8,192 |

**Retry / Persistence**: Failed events are written to `~/.claude/telemetry/{prefix}{sessionId}.{uuid}.json` with exponential backoff (`baseBackoffDelay * attempts²`). On next startup, previously failed files are re-read and retried.

**Auth**: OAuth token (if available and not expired); falls back to unauthenticated on 401. Always sends `x-service-name: claude-code`.

**Kill switch**: Statsig gate `tengu_frond_boric["firstParty"]`.

**Event sampling**: All events pass through `shouldSampleEvent()` / `Dx1()` which checks Statsig dynamic config `tengu_event_sampling_config`. `sample_rate=0` drops event; `sample_rate` between 0–1 probabilistically samples.

**Event structure** (**line 545995**):
```json
{
  "events": [{
    "event_name": "<event_name>",
    "event_id": "<uuid>",
    "core_metadata": { /* mV6() output */ },
    "user_metadata": { /* ik(true) output */ },
    "event_metadata": { /* per-event properties */ },
    "user_id": "<ny() output>"
  }]
}
```

### 2.2 BigQuery Metrics

**Class**: `rg8` (**line ~387840**)  
**Endpoint**: `https://api.anthropic.com/api/claude_code/metrics` (hardcoded)  
**Export interval**: 300,000ms (5 minutes) via `PeriodicExportingMetricReader`  
**Timeout**: 5,000ms  
**Auth**: Anthropic auth headers via `gO()`

**Activation** (`coY()`):
- When `KF8()` returns true (first-party trust established), OR
- When authenticated with claude.ai AND subscription is `enterprise` or `team`

**Export gating** (`U1q()`):
- Checks organization's metrics opt-out status (cached 24h, refreshed 1h)
- If org has disabled metrics: skips export silently
- Requires trust established (`L$() || u7()`)

**Data format** (from `transformMetricsForInternal()`):
```json
{
  "resource_attributes": {
    "service.name": "claude-code",
    "service.version": "2.1.71",
    "aggregation.temporality": "delta",
    "user.customer_type": "claude_ai" | "api",
    "user.subscription_type": "..."
  },
  "metrics": [
    {
      "name": "...",
      "description": "...",
      "unit": "...",
      "data_points": [{"attributes": {...}, "value": ..., "timestamp": "ISO8601"}]
    }
  ]
}
```

### 2.3 DataDog Log Intake

**Module**: `cN1` (**line ~342560**)  
**Endpoint**: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`  
**API key**: `pubbbf48e6d78dae54bceaa4acf463299bf` (hardcoded, public write-only key)  
**Auth**: `DD-API-KEY` header  

**Activation requirements** (all must be true):
- Provider is `"firstParty"` (`D7() === "firstParty"`) — not Bedrock/Vertex
- Statsig gate `tengu_log_datadog_events` enabled
- Kill switch `J$6("datadog")` is false
- `Axq()` gate check passes

**Configuration**:

| Parameter | Value | Env Var |
|-----------|-------|---------|
| Flush interval | 15,000ms | `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS` |
| Max batch size | 100 events | hardcoded |
| Request timeout | 5,000ms | hardcoded |
| User bucket count | 30 | hardcoded |

**Allowlisted events** (only these 35 events are sent to DataDog):
```
chrome_bridge_connection_succeeded, chrome_bridge_connection_failed,
chrome_bridge_disconnected, chrome_bridge_tool_call_completed,
chrome_bridge_tool_call_error, chrome_bridge_tool_call_started,
chrome_bridge_tool_call_timeout,
tengu_api_error, tengu_api_success, tengu_cancel, tengu_compact_failed,
tengu_exit, tengu_flicker, tengu_init, tengu_model_fallback_triggered,
tengu_oauth_error, tengu_oauth_success,
tengu_oauth_token_refresh_failure, tengu_oauth_token_refresh_success,
tengu_oauth_token_refresh_lock_acquiring, tengu_oauth_token_refresh_lock_acquired,
tengu_oauth_token_refresh_starting, tengu_oauth_token_refresh_completed,
tengu_oauth_token_refresh_lock_releasing, tengu_oauth_token_refresh_lock_released,
tengu_query_error, tengu_repo_text_file_size, tengu_session_file_read,
tengu_started, tengu_tool_use_error,
tengu_tool_use_granted_in_prompt_permanent, tengu_tool_use_granted_in_prompt_temporary,
tengu_tool_use_rejected_in_prompt, tengu_tool_use_success,
tengu_uncaught_exception, tengu_unhandled_rejection,
tengu_voice_recording_started, tengu_voice_toggled,
tengu_team_mem_sync_pull, tengu_team_mem_sync_push, tengu_team_mem_sync_started
```

**DDtags fields** (`fEY`, **line ~342607**): `event:<name>` plus `arch`, `clientType`, `errorType`, `http_status_range`, `http_status`, `kairosActive`, `model`, `platform`, `provider`, `subscriptionType`, `toolName`, `userBucket`, `version`, `versionBase`.

**Privacy**: MCP tool names normalized to `"mcp"`. Model names mapped to canonical form or `"other"`. Dev build version metadata stripped.

**User bucket**: SHA-256 of user ID, first 8 hex chars as int mod 30.

**Log structure**:
```json
{
  "ddsource": "nodejs",
  "ddtags": "event:<name>,<field>:<value>,...",
  "message": "<event_name>",
  "service": "claude-code",
  "hostname": "claude-code",
  "env": "external"
}
```

### 2.4 Segment Analytics

**Package**: `@segment/analytics-node`  
**Write keys**:
- Production: `LKJN8LsLERHEOXkw487o7qCTFOrGPimI`
- Development: `b64sf1kxwDGe1PiSAlv5ixuH0f509RKK`

**Activation**:
- Not in UV() disablement state
- Statsig gate `tengu_log_segment_events` enabled
- Kill switch `J$6("segment")` is false

**Configuration**:
```js
new Analytics({
  writeKey: pyY(),       // production key
  flushAt: 50,           // batch size
  flushInterval: 10000,  // 10 seconds
})
```

**Identity**:
- `anonymousId` = `claudecode.v1.<uuid>` (persisted to local config, `eN1()`)
- `userId` = `deviceId` (from `ik(true)` user metadata)
- When authenticated: attaches `accountUuid` and `organizationUuid` as properties

**Properties per event**: Built from `aB4()` — includes environment context + event-specific fields, plus `surface: "claude-code"`. `envContext` and `processMetrics` are stripped from core metadata before sending.

**Calls made**:
- `K.track({ anonymousId, event, properties, userId? })` — per event
- `K.identify({ anonymousId, traits, userId? })` — when user traits available

**Shutdown**: `rB4()` → `ab8.closeAndFlush()`

### 2.5 User-Configured OTEL (3P)

Activated by `CLAUDE_CODE_ENABLE_TELEMETRY=true`. Supports user-supplied exporters for metrics, logs, and traces.

**Initialization** (`initializeTelemetry()`, **line ~388126**):
1. Sets OTEL DiagLogger at ERROR level
2. Creates metric readers based on `OTEL_METRICS_EXPORTER` (`QoY()`)
3. Creates log exporters based on `OTEL_LOGS_EXPORTER` (`UoY()`)
4. Creates trace exporters if enhanced telemetry beta active (`poY()`)
5. Sets global OTEL providers

**Metric exporters** (`OTEL_METRICS_EXPORTER`):
- `console` → `ConsoleMetricExporter`
- `otlp` → selects by `OTEL_EXPORTER_OTLP_METRICS_PROTOCOL`: `grpc`, `http/json`, or `http/protobuf`
- `prometheus` → `PrometheusExporter`

**Log exporters** (`OTEL_LOGS_EXPORTER`):
- `console` → `ConsoleLogRecordExporter`
- `otlp` → selects by `OTEL_EXPORTER_OTLP_LOGS_PROTOCOL`

**Trace exporters** (`OTEL_TRACES_EXPORTER`, requires enhanced telemetry beta):
- `console` → `ConsoleSpanExporter`
- `otlp` → selects by `OTEL_EXPORTER_OTLP_TRACES_PROTOCOL`

**OTEL resource attributes**:
```js
{
  "service.name": "claude-code",
  "service.version": "2.1.71",
  "wsl.version": "...",   // if WSL
  "os.type": ...,
  "os.version": ...,
  "host.arch": ...
}
```

**3P event logger** — `AX()` (**line 288145**) emits structured events:
- Logger name: `com.anthropic.claude_code.events`
- Body: `claude_code.{event_name}`
- Attributes include `mT6()` context + `event.name`, `event.timestamp`, `event.sequence`, `prompt.id`

**Flush settings**:
- Flush timeout: `CLAUDE_CODE_OTEL_FLUSH_TIMEOUT_MS` (default 5,000ms)
- Shutdown timeout: `CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS` (default 2,000ms)
- Called on `beforeExit` and `exit` events

### 2.6 Beta Tracing

**Setup function**: `loY()` (**line ~388141**)  
**Endpoint**: Internal Anthropic endpoint configured via `BETA_TRACING_ENDPOINT`  
**Tracer name**: `"com.anthropic.claude_code.tracing"` version `"1.0.0"`

**Activation**:
- `BETA_TRACING_ENDPOINT` must be set
- `ENABLE_BETA_TRACING_DETAILED=true`
- Statsig gate `tengu_trace_lantern` enabled OR non-interactive mode

**Creates**:
- `OTLPTraceExporter` → `${BETA_TRACING_ENDPOINT}/v1/traces`
- `BatchSpanProcessor` with `scheduledDelayMillis: e1q`
- `BasicTracerProvider` → set as global tracer
- `OTLPLogExporter` → `${BETA_TRACING_ENDPOINT}/v1/logs`
- `BatchLogRecordProcessor` → `LoggerProvider` set as global logger

**Additional data in beta tracing** (via `VG4()`): `system_prompt_hash`, `system_prompt_preview` (first 500 chars), full system prompt, tool names + hashes, new message content, `response.model_output` (all truncated to 60KB).

---

## 3. Event Catalog

All events use the `tengu_*` prefix and are dispatched via `c()`. Total: **598 unique event names**.

### 3.1 Session Lifecycle Events

| Event Name | Properties | Trigger |
|------------|-----------|--------|
| `tengu_started` | `{}` (empty) | Startup complete (after plugins loaded, background jobs launched) |
| `tengu_exit` | `last_session_cost`, `last_session_duration`, `last_session_api_duration`, `last_session_tool_duration`, `last_session_lines_added`, `last_session_lines_removed`, `last_session_total_input/output_tokens`, cache tokens, `fps_average`, `fps_low_1_pct`, `last_session_id` | Startup — reports **previous** session's stats from persistent storage |
| `tengu_init` | (varies) | Session initialization |
| `tengu_startup_telemetry` | `is_git`, `worktree_count`, `repo_text_file_size_bytes`, `sandbox_enabled`, `are_unsandboxed_commands_allowed`, `auto_updater_disabled`, `prefers_reduced_motion`, `has_node_extra_ca_certs`, `has_client_cert`, `has_use_system_ca`, `has_use_openssl_ca` | Startup environment report (**line 610215**) |
| `tengu_startup_perf` | `checkpoint_count` + all profiling timings | Startup performance checkpoint (**line 4202**) |
| `tengu_cancel` | — | User cancels current operation (Ctrl+C) |
| `tengu_session_resumed` | — | Session resumed from saved state |
| `tengu_session_file_read` | — | Session file loaded from disk (**line 249548**) |
| `tengu_began_setup` | `oauthEnabled` | Setup process initiated |
| `tengu_headless_latency` | latency fields | Headless mode first response latency (**line 288637**) |
| `tengu_stdin_interactive` | — | Stdin interactive mode detection |
| `tengu_concurrent_onquery_detected/enqueued` | — | Concurrent query handling |

**Session browser events** (session list UI): `tengu_session_renamed`, `tengu_session_tagged`, `tengu_session_linked_to_pr` (`prNumber`), `tengu_session_group_expanded`, `tengu_session_preview_opened`, `tengu_session_search_toggled`, `tengu_session_branch_filter_toggled`, `tengu_session_worktree_filter_toggled`, `tengu_session_all_projects_toggled`, `tengu_session_tag_filter_changed`.

### 3.2 API / LLM Events

| Event Name | Key Properties | Trigger |
|------------|---------------|--------|
| `tengu_api_query` | `model`, `messagesLength`, `temperature`, `provider`, `betas`, `permissionMode`, `querySource`, `queryChainId`, `queryDepth`, `thinkingType`, `effortValue`, `fastMode`, `previousRequestId`, `baseUrl?`, `envModel?`, `envSmallFastModel?` | Before each API call (**line 454194**) |
| `tengu_api_success` | `model`, `preNormalizedModel`, `inputTokens`, `outputTokens`, `cachedInputTokens`, `uncachedInputTokens`, `durationMs`, `durationMsIncludingRetries`, `attempt`, `ttftMs`, `requestId`, `stop_reason`, `costUSD`, `didFallBackToNonStreaming`, `textContentLength`, `thinkingContentLength`, `toolUseContentLengths`, `fastMode`, `gateway`, `globalCacheStrategy` | After successful API response (**line 454313**) |
| `tengu_api_error` | `model`, `error`, `status`, `errorType`, `messageCount`, `messageTokens`, `durationMs`, `attempt`, `provider`, `requestId`, `promptCategory`, `gateway`, `queryChainId`, `queryDepth` | API call failure (**line 454243**) |
| `tengu_api_retry` | retry count, error info | API call being retried (**line 244172**) |
| `tengu_api_opus_fallback_triggered` | — | Automatic model fallback to Opus (**line 244123**) |
| `tengu_model_fallback_triggered` | — | Model falls back to alternative (**line 453776**) |
| `tengu_max_tokens_context_overflow_adjustment` | — | Context window overflow, adjusts max_tokens (**line 244160**) |
| `tengu_context_size` | — | Context size measurement |
| `tengu_context_window_exceeded` | — | Context window limit hit |
| `tengu_query_error` | — | Query-level error wrapping API errors (**line 453796**) |
| `tengu_unknown_model_cost` | `model`, `shortName` | Cost calculation failed (unknown model, **line 145531**) |
| `tengu_headless_latency` | latency fields | Headless mode first response (**line 288637**) |
| `tengu_refusal_api_response` | — | API returned a refusal (**line 241612**) |
| `tengu_model_whitespace_response` | — | Model returned whitespace-only response |
| `tengu_api_before_normalize` / `tengu_api_after_normalize` | message count | Pre/post normalization stats |
| `tengu_api_cache_breakpoints` | cache control info | Cache breakpoint tracking |
| `tengu_query_before_attachments` / `tengu_query_after_attachments` | query stats | Attachment processing stats (**lines 454010, 454043**) |
| `tengu_structured_output_enabled/failure` | — | Structured output mode |

**Streaming events**: `tengu_streaming_stall`, `tengu_streaming_stall_summary`, `tengu_streaming_idle_timeout`, `tengu_streaming_error`, `tengu_streaming_fallback_to_non_streaming`, `tengu_streaming_tool_execution_used/not_used`, `tengu_stream_no_events`.

### 3.3 Tool Events

| Event Name | Key Properties | Trigger |
|------------|---------------|--------|
| `tengu_tool_use_success` | tool name, duration, output details | Tool completed successfully (**line 452605**) |
| `tengu_tool_use_error` | tool name, error type, error message | Tool failed (**lines 452110, 452246, 452288, 452743**) |
| `tengu_tool_use_cancelled` | — | Tool execution cancelled (**line 452148**) |
| `tengu_tool_use_progress` | — | Tool progress update (**line 452202**) |
| `tengu_tool_use_granted_in_config` | tool name, context | Permission granted via config (**line 288779**) |
| `tengu_tool_use_granted_by_classifier` | tool name, classifier result | Permission granted by AI classifier (**line 288783**) |
| `tengu_tool_use_granted_by_permission_hook` | — | Permission hook allowed tool (**line 288796**) |
| `tengu_tool_use_denied_in_config` | — | Tool denied by config rules (**line 288807**) |
| `tengu_tool_use_rejected_in_prompt` | — | Tool denied by user at prompt (**line 288810**) |
| `tengu_tool_use_can_use_tool_allowed` | — | `canUseTool` returned allowed (**line 452506**) |
| `tengu_tool_use_can_use_tool_rejected` | — | `canUseTool` returned rejected (**line 452463**) |
| `tengu_tool_use_show_permission_request` | — | Permission dialog shown to user |
| `tengu_tool_use_diff_computed` | — | Diff computed for file edit tool (**line 426815**) |
| `tengu_tool_result_persisted` | — | Tool result stored to disk (**line 246002**) |
| `tengu_tool_input_alias_applied` | — | Tool input alias substitution (**line 452064**) |
| `tengu_tool_use_tool_result_mismatch_error` | — | Tool use ID mismatch (**line 241317**) |
| `tengu_unexpected_tool_result` | — | Unexpected tool result structure (**line 241427**) |
| `tengu_tool_result_pairing_repaired` | — | Tool result pairing auto-repaired |
| `tengu_bash_tool_command_executed` | — | Bash tool ran a command |
| `tengu_bash_security_check_triggered` | `checkId` (e.g. `a5.MID_WORD_HASH`) | Bash security rule triggered (**lines 404474–405528**, ~38 instances) |
| `tengu_bash_tool_reset_to_original_dir` | — | Shell reset to original directory (**line 245744**) |
| `tengu_bash_tool_simple_echo` | — | Detected simple `echo` command |
| `tengu_bash_command_explicitly_backgrounded` | — | Command run with background operator |
| `tengu_code_indexing_tool_used` | — | Code indexing/search tool used (**line 524529**) |
| `tengu_tool_search_outcome` | — | Tool search completed (**line 250276**) |
| `tengu_tool_search_mode_decision` | — | Search mode selection (**line 522671**) |
| `tengu_deferred_tools_pool_change` | — | Deferred tools pool updated |
| `tengu_dir_search` | — | Directory search operation |
| `tengu_git_operation` | — | Git operation executed |

**Hook events**: `tengu_pre_tool_hook_error/hooks_cancelled`, `tengu_post_tool_hook_error/hooks_cancelled`, `tengu_post_tool_failure_hook_error/hooks_cancelled`, `tengu_stop_hook_error`, `tengu_pre_stop_hooks_cancelled`, `tengu_run_hook` (`hookName`, `numCommands`, `hookTypeCounts`), `tengu_repl_hook_finished`, `tengu_agent_stop_hook_error/max_turns/success`.

### 3.4 File Operation Events

| Event Name | Properties | Trigger |
|------------|-----------|--------|
| `tengu_file_operation` | `operation`, `tool`, `filePathHash` (SHA256, 16 chars), `contentHash?` (SHA256 of content), `repo_blob_sha?`, `type?` | File read/write/edit by a tool — **path is hashed, not sent in plaintext** (**line 89481**) |
| `tengu_file_changed` | `lines_added`, `lines_removed` | Watched file notification (**line 146432**) |
| `tengu_repo_text_file_size` | total text file size | Repository size measurement |
| `tengu_binary_content_persisted` | — | Binary content stored to disk |
| `tengu_file_suggestions_git_ls_files` | — | File suggestion from git ls-files |
| `tengu_file_suggestions_query` | — | File suggestion query |
| `tengu_file_suggestions_ripgrep` | — | File suggestion from ripgrep |
| `tengu_ripgrep_availability` | — | Ripgrep availability check |
| `tengu_ripgrep_eagain_retry` | — | Ripgrep EAGAIN retry |
| `tengu_write_claudemd` | — | CLAUDE.md file written |

**File history events**: `tengu_file_history_track_edit_success/failed`, `tengu_file_history_snapshot_success/failed`, `tengu_file_history_backup_file_created/deleted_file/failed`, `tengu_file_history_rewind_success/failed`, `tengu_file_history_rewind_restore_file_failed`, `tengu_file_history_resume_copy_failed`, `tengu_file_history_snapshots_setting_changed`.

### 3.5 Permission & Hook Events

Permission events track the full lifecycle of tool permission decisions:

- **Config-based**: `tengu_tool_use_granted_in_config`, `tengu_tool_use_denied_in_config`
- **Classifier-based**: `tengu_tool_use_granted_by_classifier`
- **Hook-based**: `tengu_tool_use_granted_by_permission_hook`
- **User prompt**: `tengu_tool_use_rejected_in_prompt`, `tengu_tool_use_granted_in_prompt_permanent/temporary`
- **Dialog**: `tengu_tool_use_show_permission_request`, `tengu_permission_request_escape`, `tengu_permission_request_option_selected`
- **Bypass mode**: `tengu_bypass_permissions_mode_dialog_accept/shown`

**Managed settings**: `tengu_managed_settings_loaded`, `tengu_managed_settings_security_dialog_accepted/rejected/shown`.

**System prompt events**: `tengu_sysprompt_block` (`snippet` — first 20 chars, `length`, `sha256` hash), `tengu_sysprompt_boundary_found/missing_boundary_marker`, `tengu_sysprompt_using_tool_based_cache`.

### 3.6 Authentication Events

| Event Name | Properties | Trigger |
|------------|-----------|--------|
| `tengu_oauth_flow_start` | `loginWithClaudeAi: boolean` | OAuth flow initiated (**lines 391284, 391494**) |
| `tengu_oauth_success` | `loginWithClaudeAi: boolean` | OAuth completed successfully |
| `tengu_oauth_error` | `error` | OAuth failure |
| `tengu_oauth_token_exchange_success/error` | `error` on failure | Token exchange result (**lines 328886, 391519**) |
| `tengu_oauth_token_refresh_success/failure` | — | Token refresh result (**lines 328905, 328956**) |
| `tengu_oauth_token_refresh_lock_*` | — | Mutex lifecycle: `acquiring`, `acquired`, `starting`, `releasing`, `released`, `retry`, `retry_limit_reached`, `error`, `race_recovered`, `race_resolved` |
| `tengu_oauth_api_key` | `status: "success"/"failure"`, `statusCode` | API key operation (**lines 328995, 329001**) |
| `tengu_oauth_profile_fetch_success` | — | Profile data fetched (**line 329045**) |
| `tengu_oauth_roles_stored` | `org_role: string` | Org roles saved (**line 328984**) |
| `tengu_oauth_auth_code_received` | `automatic: boolean` | Auth code received (**line 329309**) |
| `tengu_oauth_automatic_redirect` | `custom_handler?: boolean` | Automatic redirect (**lines 329188, 329195**) |
| `tengu_oauth_storage_warning` | `warning` | Token storage warning (**line 391238**) |
| `tengu_login_from_refresh_token` | — | Login via refresh token (**line 391263**) |
| `tengu_api_key_saved_to_config/keychain/keychain_error` | — | API key storage result |

**Provider selection**: `tengu_oauth_claudeai_forced/console_forced`, `tengu_oauth_claudeai_selected/console_selected/platform_selected`, `tengu_oauth_manual_entry`, `tengu_oauth_tokens_saved/save_failed/save_exception/inference_only/not_claude_ai`.

**MCP OAuth**: `tengu_mcp_oauth_flow_start/success/error`, `tengu_mcp_auth_config_authenticate/clear`, `tengu_mcp_server_needs_auth`, `tengu_mcp_tool_call_auth_error`, `tengu_mcp_session_expired`.

### 3.7 Error Events

| Event Name | Properties | Trigger |
|------------|-----------|--------|
| `tengu_uncaught_exception` | `error_name: A.name` | `process.on('uncaughtException')` (**line 345043**) |
| `tengu_unhandled_rejection` | `error_name: q` | `process.on('unhandledRejection')` (**line 345061**) |
| `tengu_atomic_write_error` | — | Atomic file write failure |
| `tengu_config_parse_error` | — | Config file parse failure |
| `tengu_preflight_check_failed` | — | Startup preflight check failed |
| `tengu_bridge_fatal_error` | — | Bridge fatal error |
| `tengu_bridge_repl_fatal_error` | — | REPL fatal error |
| `tengu_permission_explainer_error` | — | Permission explainer generation failed |
| `tengu_heap_dump` | — | Heap memory dump triggered |
| `tengu_node_warning` | — | Node.js runtime warning |
| `tengu_shell_unknown_error` | — | Unknown shell error |
| `tengu_shell_snapshot_error/failed` | — | Shell snapshot failure |
| `tengu_git_index_lock_error` | — | Git index lock conflict |
| `tengu_config_lock_contention/stale_write/cache_stats/auth_loss_prevented` | — | Config file management issues |

**No Sentry integration.** Zero matches for `Sentry.`, `captureException`, `captureMessage` in the codebase.

### 3.8 Feature Gate Events

**GrowthBook A/B experiment tracking** (`Ce8()`, **line 546015**) — sent to 1P OTEL only:
```json
{
  "event_type": "GrowthbookExperimentEvent",
  "event_id": "...",
  "experiment_id": "...",
  "variation_id": "...",
  "environment": "production",
  "device_id": "...",
  "session_id": "...",
  "user_attributes": "..."
}
```

**Sampling / kill switch events**: `tengu_off_switch_query` (query rejected by kill switch).

**Update / version events**: `tengu_auto_updater_success/fail/lock_contention/windows_npm_in_wsl`, `tengu_version_check_success/failure` (`latency_ms`), `tengu_binary_download_attempt/success/failure` (`latency_ms`), `tengu_native_auto_updater_start/success/fail/lock_contention/up_to_date`, `tengu_native_update_complete/lock_failed/skipped_max_version/skipped_minimum_version`, `tengu_native_install_package_success/failure/binary_success/binary_failure`, `tengu_native_staging_cleanup/temp_files_cleanup/stale_locks_cleanup/version_cleanup`, `tengu_autoupdate_enabled/channel_changed`.

### 3.9 UI / UX Events

| Event Name | Properties | Trigger |
|------------|-----------|--------|
| `tengu_input_prompt` | — | User submits a text prompt (**line 403616**) |
| `tengu_input_command` | command name, args | User runs a slash command (**lines 403658, 403701**) |
| `tengu_input_slash_missing/invalid` | `input` (for invalid) | Invalid slash command |
| `tengu_ultrathink` | — | "ultrathink" keyword detected (**line 254455**) |
| `tengu_single_word_prompt` | — | Prompt is a single word |
| `tengu_paste_image/paste_text` | — | User pastes content |
| `tengu_pasted_image_resize_attempt` | — | Pasted image being resized |
| `tengu_copy` | `block_count`, `always?`, `selected_block?` | User copies content (**lines 461140–461245**) |
| `tengu_fast_mode_toggled` | `enabled`, `source: "picker"/"shortcut"` | Fast mode toggled (**lines 503469, 503660**) |
| `tengu_fast_mode_picker_shown` | `unavailable_reason` | Fast mode picker shown (**line 503680**) |
| `tengu_voice_toggled` | — | Voice mode toggled |
| `tengu_voice_recording_started` | — | Voice recording started |
| `tengu_flicker` | — | UI flicker detected |
| `tengu_status_line_mount` | — | Status line component mounted |
| `tengu_help_toggled` | — | Help panel toggled |
| `tengu_toggle_todos/transcript` | — | UI panel toggles |
| `tengu_message_selector_cancelled/opened/selected/restore_option_selected` | — | Message selector UI |
| `tengu_editor_mode_changed` | `mode`, `source?` | Editor mode changed (**lines 464284, 501182**) |
| `tengu_tip_shown` | — | Tip/hint displayed |
| `tengu_onboarding_step` | — | Onboarding step tracked |
| `tengu_feedback_survey_event/post_compact_survey_event` | — | Survey interaction |
| `tengu_prompt_suggestion/prompt_suggestion_init` | — | Autocomplete suggestions |
| `tengu_plan_enter/exit/external_editor_used` | — | Plan mode lifecycle |
| `tengu_accept_feedback_mode_collapsed/entered`, `tengu_accept_submitted` | — | Accept mode events |
| `tengu_reject_feedback_mode_collapsed/entered`, `tengu_reject_submitted` | — | Reject mode events |

**@ mention events**: `tengu_at_mention_agent_not_found/agent_success/extracting_directory_success/extracting_filename_error`, `tengu_at_mention_mcp_resource_error/success`, `tengu_subagent_at_mention`.

**Settings change events**: `tengu_config_changed` (`key`, `value`), `tengu_config_model_changed` (`from_model`, `to_model`), `tengu_auto_compact_setting_changed` (`enabled`), `tengu_thinking_toggled` (`enabled`), `tengu_diff_tool_changed`, `tengu_reduce_motion_setting_changed`, `tengu_tips_setting_changed`, `tengu_terminal_progress_bar_setting_changed`, `tengu_respect_gitignore_setting_changed`, `tengu_pr_status_footer_setting_changed`, `tengu_auto_connect_ide_changed`, `tengu_auto_install_ide_extension_changed`, `tengu_claude_in_chrome_setting_changed`, `tengu_teammate_mode_changed`, `tengu_output_style_changed`, `tengu_language_changed`.

### 3.10 Other Events

**Compact / memory management**: `tengu_compact` (compaction stats), `tengu_compact_failed`, `tengu_compact_streaming_retry`, `tengu_compact_cache_sharing_success/fallback`, `tengu_partial_compact/partial_compact_failed`, `tengu_auto_compact_succeeded`, `tengu_post_autocompact_turn`, `tengu_orphaned_messages_tombstoned`, `tengu_sm_compact_no_session_memory/empty_template/summarized_id_not_found/resumed_session/threshold_exceeded/error`, `tengu_session_memory_loaded` (`content_length`), `tengu_session_memory_accessed/extraction/file_read`.

**MCP events**: `tengu_mcp_servers` (server count/config), `tengu_mcp_list_changed` (`type`, `newCount?`), `tengu_mcp_elicitation_shown/response` (`mode`, `action`), `tengu_mcp_instructions_pool_change`, `tengu_mcp_tools_commands_loaded`, `tengu_mcp_add/delete/get/list/start`, `tengu_mcp_dialog_choice`, `tengu_mcp_multidialog_choice`, `tengu_claudeai_mcp_eligibility` (`state: "disabled_gate"/"disabled_env_var"/"no_oauth_token"/"missing_scope"/"eligible"`), `tengu_claudeai_mcp_auth_started/completed/clear_auth_started/clear_auth_completed/reconnect/toggle`.

**Agent mode**: `tengu_auto_mode_outcome` (8 instances, **lines 411589–411878**), `tengu_auto_mode_decision`, `tengu_auto_mode_malformed_tool_input` (`toolName`), `tengu_auto_mode_denial_limit_exceeded`, `tengu_auto_mode_opt_in_dialog_shown/accept/accept_default/decline`, `tengu_agent_tool_selected`, `tengu_agent_tool_completed`, `tengu_agent_tool_terminated`, `tengu_agent_memory_loaded`, `tengu_agent_parse_error` (`error`, `location`), `tengu_fork_agent_query`, `tengu_slash_command_forked` (`command_name`), `tengu_conversation_forked`, `tengu_conversation_rewind`.

**Team memory sync**: `tengu_team_mem_sync_started`, `tengu_team_mem_sync_pull/push` (multiple outcomes), `tengu_team_mem_secret_skipped`, `tengu_team_mem_accessed/file_read/file_edit/file_write`, `tengu_team_memdir_disabled`.

**Teleport (remote session)**: `tengu_teleport_started/cancelled/resume_session/resume_error/interactive_mode`, `tengu_teleport_first_message_success/error` (`session_id`), `tengu_teleport_errors_detected/resolved`, `tengu_teleport_error_git_not_clean/branch_checkout_failed/repo_not_in_git_dir_sessions_api/repo_mismatch_sessions_api`.

**Chrome bridge events** (Datadog sink only, **lines ~18038–18357**): `chrome_bridge_connection_succeeded/failed/disconnected/tool_call_completed/tool_call_error/tool_call_started/tool_call_timeout`.

**Plugins/marketplace**: `tengu_plugin_install/uninstall/enable/disable/update/list_command`, `tengu_plugins_loaded`, `tengu_headless_plugin_install`, `tengu_marketplace_added/removed/updated/updated_all/background_install`, `tengu_official_marketplace_auto_install`.

**Skills**: `tengu_skill_loaded`, `tengu_skill_tool_invocation`, `tengu_skill_tool_slash_prefix`, `tengu_skill_file_changed`, `tengu_skill_improvement_survey`, `tengu_dynamic_skills_changed`.

**Grove policy** (AI usage policy): `tengu_grove_policy_dismissed/escaped/exited/submitted/toggled/viewed`, `tengu_grove_print_viewed`, `tengu_grove_privacy_settings_viewed`.

**Rate limits / billing**: `tengu_cost_threshold_reached/acknowledged`, `tengu_rate_limit_options_menu_cancel/select_extra_usage/select_upgrade`, `tengu_max_tokens_reached`, `tengu_claudeai_limits_status_changed`, `tengu_switch_to_subscription_notice_shown`, `tengu_guest_passes_link_copied/upsell_shown/visited`, `tengu_desktop_upsell_shown`.

**IDE/extension**: `tengu_ext_at_mentioned`, `tengu_ext_diff_accepted/rejected`, `tengu_ext_will_show_diff`, `tengu_ext_installed/install_error`, `tengu_ext_ide_command`, `tengu_external_editor_hint_shown`, `tengu_external_editor_used`, `tengu_bridge_*` (many bridge lifecycle events).

**GitHub integration**: `tengu_install_github_app_started/completed/error/step_completed`, `tengu_setup_github_actions_started/completed/failed`, `tengu_install_slack_app_clicked`.

**Worktree events**: `tengu_worktree_created/removed/kept`, `tengu_worktree_detection`.

**PDF/image**: `tengu_pdf_page_extraction`, `tengu_pdf_reference_attachment`, `tengu_image_api_validation_failed`, `tengu_image_compress_failed`, `tengu_image_resize_failed/fallback`.

**Misc notable**: `tengu_speculation` (speculative execution), `tengu_ultrathink` (extended thinking keyword), `tengu_timer`, `tengu_unary_event`, `tengu_tree_sitter_load`, `tengu_ripgrep_availability`, `tengu_notification_method_used`, `tengu_auto_memory_toggled`, `tengu_mode_cycle`.

---

## 4. OTEL Instrumentation

### 4.1 Metrics (Counters)

Defined in `gg1()` (**line ~2328**), registered with OTEL Meter `"com.anthropic.claude_code"` version `"2.1.71"`.

All metrics use DELTA temporality by default (`OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=delta`).

| Metric Name | Type | Unit | Description |
|-------------|------|------|-------------|
| `claude_code.session.count` | Counter | — | CLI sessions started |
| `claude_code.lines_of_code.count` | Counter | — | Lines of code modified; attr `type` = `added`/`removed` |
| `claude_code.pull_request.count` | Counter | — | Pull requests created |
| `claude_code.commit.count` | Counter | — | Git commits created |
| `claude_code.cost.usage` | Counter | USD | Session cost |
| `claude_code.token.usage` | Counter | tokens | Tokens used |
| `claude_code.code_edit_tool.decision` | Counter | — | Edit/Write/NotebookEdit accept/reject decisions |
| `claude_code.active_time.total` | Counter | s | Active time in seconds |

**Metric attributes** (from `mT6()`, **line ~288098**) — added to every metric record:

| Attribute | Condition |
|-----------|----------|
| `user.id` | Always (locally-generated random hex, persisted) |
| `session.id` | If `OTEL_METRICS_INCLUDE_SESSION_ID=true` (default: **true**) |
| `app.version` | If `OTEL_METRICS_INCLUDE_VERSION=true` (default: **false**) |
| `organization.id` | If logged in to claude.ai via OAuth |
| `user.email` | If logged in to claude.ai via OAuth |
| `user.account_uuid` | If logged in AND `OTEL_METRICS_INCLUDE_ACCOUNT_UUID=true` (default: **true**) |
| `terminal.type` | If terminal type is known |

Note: The full OTEL SDK supports all standard instrument types (`createGauge`, `createHistogram`, `createCounter`, `createUpDownCounter`, observable variants), but only the 8 counters above are actually registered.

### 4.2 Spans

Spans are created when `Fx()` returns true (enhanced telemetry beta OR beta tracing endpoint configured).

| Span Name | `span.type` | Key Attributes |
|-----------|------------|----------------|
| `claude_code.interaction` | `interaction` | `user_prompt` (redacted by default), `user_prompt_length`, `interaction.sequence`, `interaction.duration_ms` |
| `claude_code.llm_request` | `llm_request` | `model`, `llm_request.context`, `speed` (`fast`/`normal`), `query_source`, `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_creation_tokens`, `success`, `status_code`, `error`, `attempt`, `response.has_tool_call`, `ttft_ms`, `duration_ms` |
| `claude_code.tool` | `tool` | `tool_name`, extra attributes |
| `claude_code.tool.blocked_on_user` | `tool.blocked_on_user` | `duration_ms`, `decision`, `source` |
| `claude_code.tool.execution` | `tool.execution` | `duration_ms`, `success`, `error`, `result_tokens`, `new_context` (if `OTEL_LOG_TOOL_CONTENT`) |
| `claude_code.hook` | `hook` | `hook_event`, `hook_name`, `num_hooks`, `hook_definitions`, `num_success`, `num_blocking`, `num_non_blocking_error`, `num_cancelled`, `duration_ms` |

**Additional beta tracing data** (via `VG4()`, only when `qD()` is true): `system_prompt_hash`, `system_prompt_preview` (first 500 chars), `system_prompt_length`, `system_prompt` (full, up to 60KB), `tools` (JSON of names + hashes), `tools_count`, `new_context`, `system_reminders`, `response.model_output`.

**Content truncation limit**: `XjY = 61440` bytes (60KB). Truncation tracked with `_truncated: true` and `_original_length: N` attributes.

### 4.3 Log Records

The 3P telemetry event logger (`AX()`, **line 288145**) emits OTEL log records with body `claude_code.{event_name}`.

**Base attributes** (always present, from `mT6()`):
- `user.id`, `session.id`, `organization.id`, `user.email`, `user.account_uuid`, `app.version`, `terminal.type`
- `event.name`, `event.timestamp`, `event.sequence`, `prompt.id`

**Named log event schemas**:

| Log Body | Key Attributes |
|----------|----------------|
| `claude_code.user_prompt` | `prompt_length` (always), `prompt` (redacted unless `OTEL_LOG_USER_PROMPTS`), `prompt.id` |
| `claude_code.api_request` | `model`, `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_creation_tokens`, `cost_usd`, `duration_ms`, `speed` |
| `claude_code.api_error` | `model`, `error`, `status_code`, `duration_ms`, `attempt`, `speed` |
| `claude_code.tool_result` | `tool_name` (MCP anonymized to `"mcp_tool"`), `success`, `duration_ms`, `tool_parameters?` (requires `ANALYTICS_LOG_TOOL_DETAILS`), `tool_result_size_bytes`, `decision_source?`, `decision_type?`, `mcp_server_scope?`, `error?` |
| `claude_code.tool_decision` | `decision`, `source`, `tool_name` |
| `claude_code.system_prompt` | `system_prompt_hash`, `system_prompt`, `system_prompt_length`, `system_prompt_truncated` — **beta tracing only** |
| `claude_code.tool` (beta) | `tool_name`, `tool_hash`, `tool` (full JSON), `tool_truncated` — **beta tracing only** |
| `claude_code.hook_execution_start` | `hook_event`, `hook_name`, `num_hooks`, `managed_only`, `hook_definitions`, `hook_source` — **beta tracing only** |
| `claude_code.hook_execution_complete` | Completion metadata — **beta tracing only** |

---

## 5. Privacy & Data Controls

### 5.1 Default Behavior (What's Always Sent)

With no special configuration, the following is sent to Anthropic's 1P endpoint:

- **User ID**: Locally-generated random 32-byte hex string (`ny()`) — persisted to local config, NOT linked to OAuth account
- **Session ID**: UUID per session
- **OAuth data** (if authenticated): `user.email`, `organization.id`, `user.account_uuid`
- **Prompt length**: Character count (prompt text itself is redacted)
- **Token counts**: Input, output, cache reads, cache writes — per API call
- **Cost in USD**: Per API call
- **Latency metrics**: Duration, TTFT
- **Model name**: Exact model string used
- **Tool name**: Normalized (MCP tools shown as `"mcp_tool"` in OTEL logs; `"mcp"` in DataDog)
- **Tool success/failure**: Boolean + error type
- **HTTP status codes**: On API errors
- **Terminal/IDE type**: Detected from environment variables
- **Platform / arch / OS**: From `envContext`
- **Feature flag state**: Active betas, gates
- **Session metrics**: From previous session (cost, LOC, token totals) reported on next startup

### 5.2 What's Never Sent

- **File contents**: Not collected as telemetry fields under any configuration
- **User prompt text**: Redacted to `"<REDACTED>"` (only length is sent) in default configuration
- **Tool input/output content**: Not logged in default configuration
- **System prompt text**: Not sent (only hash + preview available in beta tracing)
- **Model output text**: Not sent in default configuration
- **Actual MCP server names**: Anonymized to `"mcp_tool"` in OTEL events

### 5.3 Opt-In Data (Env Var Controlled)

| Env Var | Default | What It Unlocks |
|---------|---------|----------------|
| `OTEL_LOG_USER_PROMPTS=true` | false | Actual prompt text in spans and OTEL log records |
| `OTEL_LOG_TOOL_CONTENT=true` | false | Tool input/output content as span events (truncated to 60KB) |
| `ANALYTICS_LOG_TOOL_DETAILS=true` | false | Tool parameters in `claude_code.tool_result` log records |
| `ENABLE_BETA_TRACING_DETAILED=true` + `BETA_TRACING_ENDPOINT` | false/unset | Full system prompt, model output, tool JSON (internal Anthropic endpoint) |
| `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=true` | false | Enhanced OTEL trace export to user-configured endpoint |
| `OTEL_METRICS_INCLUDE_VERSION=true` | false | `app.version` in metric attributes |

### 5.4 Data Redaction & Anonymization

| Data Type | Mechanism |
|-----------|-----------|
| File paths | SHA256 hashed to 16 chars — `tengu_file_operation` only sends `filePathHash` (**line 89481**) |
| File content | SHA256 hashed — `contentHash` field if file ≤ 100KB |
| User prompts | Replaced with `"<REDACTED>"` unless `OTEL_LOG_USER_PROMPTS=true` |
| MCP tool names | Normalized to `"mcp_tool"` in OTEL events (`wK()` function); `"mcp"` in DataDog |
| Model names | Mapped to canonical form or `"other"` |
| Dev build versions | Build metadata stripped (removes `t{timestamp}.sha{hash}` suffix) |
| Error messages | `TelemetrySafeError` class (**line 52977**) provides a sanitized `telemetryMessage` separate from the full internal error message |
| System prompt snippets | Only first 20 chars in `tengu_sysprompt_block`; SHA256 hash also sent |
| User bucket | SHA-256 of user ID mod 30 (used for DataDog sampling) |

### 5.5 Disabling Telemetry

**Complete disable** (disables 1P OTEL, DataDog, Segment):
```bash
export DISABLE_TELEMETRY=1
# or
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```

**Automatic disable** for non-Anthropic providers:
```bash
export CLAUDE_CODE_USE_BEDROCK=1    # disables 1P, DataDog, Segment
export CLAUDE_CODE_USE_VERTEX=1     # disables 1P, DataDog, Segment
export CLAUDE_CODE_USE_FOUNDRY=1    # disables 1P, DataDog, Segment
```

**Error reporting only** (separate from telemetry):
```bash
export DISABLE_ERROR_REPORTING=1
```

**BigQuery metrics** can be disabled at the organization level via an API setting (not via env var). The check is cached for 24 hours.

**GrowthBook gates** (`tengu_frond_boric`) can kill individual sinks server-side without client action:
- `tengu_frond_boric["firstParty"]` — kills 1P event logging
- `tengu_frond_boric["segment"]` — kills Segment sink
- `tengu_frond_boric["datadog"]` — kills DataDog sink

**3P OTEL** is opt-in and simply not active unless `CLAUDE_CODE_ENABLE_TELEMETRY=true` is set.

---

## 6. Attribution & Billing

### Billing Header (`x-anthropic-billing-header`)

Injected as a **system prompt prefix** (not an HTTP header) via `uY1()` (**line 89290**):

```
x-anthropic-billing-header: cc_version=<VERSION>.<modelSuffix>; cc_entrypoint=<ENTRYPOINT>; cch=90643; cc_workload=<workload>;
```

| Field | Value |
|-------|-------|
| `cc_version` | `"2.1.71.<3-char-hash>"` — hash derived from first user message content |
| `cc_entrypoint` | Value of `CLAUDE_CODE_ENTRYPOINT` env var (default: `"unknown"`) |
| `cch` | `90643` — hardcoded constant (customer/channel identifier) |
| `cc_workload` | `M31()` — workload identifier if set |

This header is parsed by the Anthropic API server to attribute usage to Claude Code. It is prepended to the system prompt and transmitted with every API request.

**Disabling the billing header**:
- Set `CLAUDE_CODE_ATTRIBUTION_HEADER` to a falsy value
- Or via feature flag `tengu_attribution_header = false`

### Usage Tracking in Telemetry

Per API request, tracked in both `tengu_api_success` (1P) and `claude_code.api_request` (OTEL):

| Field | OTEL Name | Internal Event Name |
|-------|-----------|---------------------|
| Input tokens | `input_tokens` | `inputTokens` |
| Output tokens | `output_tokens` | `outputTokens` |
| Cache read tokens | `cache_read_tokens` | `cachedInputTokens` |
| Cache creation tokens | `cache_creation_tokens` | `uncachedInputTokens` |
| Cost | `cost_usd` | `costUSD` |
| Duration | `duration_ms` | `durationMs` |
| Time to first token | `ttft_ms` | `ttftMs` |

**Environment model overrides** (`gl8()`, **line 454141**) — appended to API events when set:
- `baseUrl` — if `ANTHROPIC_BASE_URL` is set
- `envModel` — if `ANTHROPIC_MODEL` is set
- `envSmallFastModel` — if `ANTHROPIC_SMALL_FAST_MODEL` is set

---

## 7. Configuration Reference

All telemetry-related environment variables:

### Disable / Enable Controls

| Variable | Effect |
|----------|--------|
| `DISABLE_TELEMETRY` | Disables 1P event logging, DataDog, and Segment |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Same effect as `DISABLE_TELEMETRY` |
| `DISABLE_ERROR_REPORTING` | Disables error reporting (separate pipeline) |
| `CLAUDE_CODE_USE_BEDROCK=true` | Auto-disables 1P, DataDog, Segment |
| `CLAUDE_CODE_USE_VERTEX=true` | Auto-disables 1P, DataDog, Segment |
| `CLAUDE_CODE_USE_FOUNDRY=true` | Auto-disables 1P, DataDog, Segment |
| `CLAUDE_CODE_ENABLE_TELEMETRY=true` | **Opt-in**: enables 3P OTEL metric/log/trace export |
| `CLAUDE_CODE_ATTRIBUTION_HEADER` | Set falsy to disable billing attribution header |

### Privacy / Content Controls

| Variable | Default | Effect |
|----------|---------|--------|
| `OTEL_LOG_USER_PROMPTS` | `false` | If `true`: include actual prompt text in spans/logs |
| `OTEL_LOG_TOOL_CONTENT` | `false` | If `true`: include tool input/output in span events |
| `ANALYTICS_LOG_TOOL_DETAILS` | `false` | If `true`: include tool parameters in `tool_result` log records |
| `OTEL_METRICS_INCLUDE_SESSION_ID` | `true` | Whether to include `session.id` in metric attributes |
| `OTEL_METRICS_INCLUDE_VERSION` | `false` | Whether to include `app.version` in metric attributes |
| `OTEL_METRICS_INCLUDE_ACCOUNT_UUID` | `true` | Whether to include `user.account_uuid` in metric attributes |

### 3P OTEL Export Configuration

| Variable | Purpose |
|----------|---------|
| `OTEL_METRICS_EXPORTER` | Comma-separated: `console`, `otlp`, `prometheus` |
| `OTEL_METRIC_EXPORT_INTERVAL` | Export interval in ms |
| `OTEL_EXPORTER_OTLP_METRICS_PROTOCOL` | `grpc`, `http/json`, `http/protobuf` |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | Default: `delta` |
| `OTEL_EXPORTER_OTLP_METRICS_HEADERS` | Auth/custom headers |
| `OTEL_EXPORTER_OTLP_METRICS_CLIENT_CERTIFICATE` | mTLS certificate |
| `OTEL_EXPORTER_OTLP_METRICS_CLIENT_KEY` | mTLS key |
| `OTEL_LOGS_EXPORTER` | Comma-separated: `console`, `otlp` |
| `OTEL_LOGS_EXPORT_INTERVAL` | Batch delay in ms |
| `OTEL_EXPORTER_OTLP_LOGS_PROTOCOL` | `grpc`, `http/json`, `http/protobuf` |
| `OTEL_EXPORTER_OTLP_LOGS_HEADERS` | Auth/custom headers |
| `OTEL_TRACES_EXPORTER` | `console`, `otlp` — requires enhanced telemetry beta |
| `OTEL_EXPORTER_OTLP_TRACES_PROTOCOL` | `grpc`, `http/json`, `http/protobuf` |
| `OTEL_EXPORTER_OTLP_TRACES_HEADERS` | Auth/custom headers |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | Fallback protocol for all exporters |
| `OTEL_EXPORTER_OTLP_HEADERS` | Global auth headers |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP endpoint URL |
| `OTEL_RESOURCE_ATTRIBUTES` | Additional resource attributes |

### Enhanced / Beta Telemetry

| Variable | Default | Purpose |
|----------|---------|--------|
| `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA` | `false` | Enable enhanced OTEL trace export |
| `ENABLE_ENHANCED_TELEMETRY_BETA` | `false` | Alias for above |
| `BETA_TRACING_ENDPOINT` | unset | Internal Anthropic beta tracing endpoint URL |
| `ENABLE_BETA_TRACING_DETAILED` | `false` | Enable detailed beta tracing (requires endpoint) |

### Timing / Operational

| Variable | Default | Purpose |
|----------|---------|--------|
| `CLAUDE_CODE_OTEL_FLUSH_TIMEOUT_MS` | 5,000ms | OTEL flush timeout at shutdown |
| `CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS` | 2,000ms | OTEL shutdown timeout |
| `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS` | 15,000ms | DataDog flush interval |

### Model / Endpoint Overrides (Appear in Telemetry)

| Variable | Effect on Telemetry |
|----------|-----------------------|
| `ANTHROPIC_BASE_URL` | Reported as `baseUrl` in API events; routes 1P logging to staging if set to staging URL |
| `ANTHROPIC_MODEL` | Reported as `envModel` in API events |
| `ANTHROPIC_SMALL_FAST_MODEL` | Reported as `envSmallFastModel` in API events |
| `CLAUDE_CODE_ENTRYPOINT` | Sent in `entrypoint` core metadata field and billing header |

---

## 8. Summary Statistics

| Metric | Value |
|--------|-------|
| Total unique event names | **598** |
| Telemetry pipelines | **6** |
| OTEL metrics defined | **8** (all Counters) |
| OTEL span types | **6** |
| DataDog allowlisted events | **~40** (35 `tengu_*` + 7 `chrome_bridge_*`) |
| 1P batch retry max | **8 attempts** with exponential backoff |
| BigQuery export interval | **5 minutes** |
| Content truncation limit | **60,480 bytes** (60KB) |
| Failed event persistence | `~/.claude/telemetry/*.json` |
| OTEL SDK version | `@opentelemetry/api` v1.9.0 |
| Segment write key (prod) | `LKJN8LsLERHEOXkw487o7qCTFOrGPimI` |
| DataDog API key | `pubbbf48e6d78dae54bceaa4acf463299bf` |
| 1P OTEL endpoint | `https://api.anthropic.com/api/event_logging/batch` |
| BigQuery metrics endpoint | `https://api.anthropic.com/api/claude_code/metrics` |
| DataDog endpoint | `https://http-intake.logs.us5.datadoghq.com/api/v2/logs` |
| Billing constant `cch` | `90643` |

### Key Architectural Observations

1. **No Sentry**: Zero matches for `Sentry.`, `captureException`, or `captureMessage`. Crash reporting uses `tengu_uncaught_exception` / `tengu_unhandled_rejection` through the normal event pipeline.

2. **No Amplitude**: Zero matches.

3. **File path hashing**: `tengu_file_operation` hashes paths with SHA256 before sending. Only 16 chars of the hash are included.

4. **1P is the primary sink**: `Re8` exporter always fires if `UV()` is false (user has not triggered any of the disable conditions). Segment and DataDog are additionally gated behind GrowthBook flags that must be explicitly enabled.

5. **Event sampling**: All events pass through `Dx1(eventName)` which checks `tengu_event_sampling_config`. An event can be fully suppressed (`sample_rate=0`) or probabilistically sampled.

6. **Session metrics survive crashes**: `tengu_exit` properties are written at session end to persistent storage and reported on the next startup — meaning token/cost data is captured even from sessions that crash.

7. **Statsig is server-side only**: No Statsig SDK is bundled. Gates are fetched via Anthropic's remote settings API and cached in `T1().cachedStatsigGates`.

8. **Two separate event channels**: `c()` → `tengu_*` events go to Anthropic's internal analytics. `AX()` → `claude_code.*` log records go to user-configured 3P OTEL. These are distinct and independently configurable.

### Key Line Numbers (cli.js)

| Component | Approx. Lines |
|-----------|---------------|
| `c()` central dispatcher | 4110 |
| Metric instruments (`gg1`) | 2328–2355 |
| `TelemetrySafeError` | 52977–52983 |
| `UV()` disablement check | 57199–57208 |
| GrowthBook SDK | 56438–56990 |
| Segment Analytics client | 344815–344885 |
| DataDog log intake client | 342380–342680 |
| BigQuery metrics exporter (`rg8`) | 387840–387960 |
| Main OTEL telemetry init | 387989–388360 |
| OTEL 3P init (`rxz`) | 552480–552510 |
| `mT6()` metric attributes | 288098–288145 |
| Span types / trace setup | 289322–289530 |
| 1P Event Exporter (`Re8`) | 545562–545940 |
| `initialize1PEventLogging` (`LIz`) | 546010–546095 |
| `logEventTo1P` (`Xx1`) | 545965–546010 |
| Analytics dispatch (`Ebz`/`Lbz`) | 549051–549085 |
| Billing header injection | 89290 |
| File operation hashing | 89481 |
| `tengu_startup_telemetry` | 610215 |
| `tengu_started` / `tengu_exit` | 566819 / 566839 |
| `tengu_api_query` | 454194 |
| `tengu_api_success` | 454313 |
| `tengu_api_error` | 454243 |
| Chrome bridge events | ~18038–18357 |
