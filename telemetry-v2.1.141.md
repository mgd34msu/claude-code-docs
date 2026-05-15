# Telemetry in Claude Code 2.1.141

2.1.141 telemetry is centered on a local analytics API that fans events out to
configured sinks. The primary model is Datadog production monitoring and
first-party OpenTelemetry event logging, with privacy controls gating both.

## Core API

Main files:

- `source/src/services/analytics/index.ts`
- `source/src/services/analytics/sink.ts`
- `source/src/services/analytics/config.ts`
- `source/src/services/analytics/datadog.ts`
- `source/src/services/analytics/firstPartyEventLogger.ts`

Primary functions:

- `logEvent`
- `logEventAsync`
- `attachAnalyticsSink`

Events emitted before a sink is attached are queued. When the sink attaches, the
queue is drained asynchronously.

## Privacy and Disable Controls

Telemetry is disabled or reduced by:

- `DISABLE_TELEMETRY`
- `DISABLE_ERROR_REPORTING`
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`
- privacy level no-telemetry / essential-traffic modes.
- provider paths such as Bedrock, Vertex, and Foundry.
- `NODE_ENV=test`.

`source/src/utils/privacyLevel.ts` maps:

- `DISABLE_TELEMETRY` to no telemetry.
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` to essential traffic only.

## Sinks

### Datadog

Datadog sink:

- endpoint: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- production-oriented.
- first-party provider only.
- controlled by `tengu_log_datadog_events` and sink kill switches.
- flush interval can be affected by `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS`.

Datadog allows a constrained event set rather than arbitrary event forwarding.

### First-Party OpenTelemetry

The first-party event logger emits OTel logs under:

- `com.anthropic.claude_code.events`

Payload includes:

- `event_name`
- `event_id`
- `core_metadata`
- `user_metadata`
- `event_metadata`
- `user_id`

Batching is controlled by:

- `tengu_1p_event_batch_config`

Sampling is controlled by:

- `tengu_event_sampling_config`

## Metrics

Observed metric counters include:

- `claude_code.session.count`
- `claude_code.lines_of_code.count`
- `claude_code.pull_request.count`
- `claude_code.commit.count`
- `claude_code.cost.usage`
- `claude_code.token.usage`
- `claude_code.code_edit_tool.decision`
- `claude_code.active_time.total`

Token metrics distinguish input, output, cache read, and cache creation.

## Event Families

2.1.141 emits events across these areas:

- startup and init: `tengu_started`, `tengu_init`, `tengu_startup_telemetry`,
  `tengu_startup_perf`.
- session lifecycle: resume, continue, exit, search, rename, preview.
- API/query: API success/error, query errors, streaming behavior, fallback.
- tools: `tengu_tool_use_*`, Bash, file read/write/edit, web fetch/search,
  skill tool, tool search.
- permissions: internal permission decisions and can-use-tool allow/reject.
- hooks: pre/post tool hook cancellation and errors, stop hook errors.
- auto mode: opt-in dialog and subsequent approvals.
- brief: brief enabled/toggled/send.
- channels: flags, gate, enable, message.
- teams: team created/deleted and teammate transcript view events.
- cron/dream/speculation: scheduled task fire/missed/expired, dream invoked,
  auto dream, prompt suggestion, speculation.
- MCP/plugins: MCP connection/auth/tool/resource/plugin events.
- remote/bridge/CCR/teleport: remote session creation, bridge, teleport resume.
- memory: memdir, team memory, extracted/session memory.
- UI: keybindings, paste, idle return, tips, model menu.
- updates/native: auto-update, native binary, installation checks.

## Sensitive Metadata Handling

Telemetry types mark event metadata that has been reviewed as not containing
code or file paths. Event logging helpers are designed to avoid arbitrary user
content in event names/metadata. Some internal debug or Anthropic-only paths are
explicitly gated.

Subprocess environment scrubbing and privacy controls are handled separately so
hook/tool child processes do not automatically inherit telemetry credentials or
tokens that should stay in the parent process.

## Print Mode

`-p` / `--print` has its own detailed telemetry writeup in
`claude-print-v2.1.141.md`. Print mode still logs startup, init, query, cost,
tool, and scheduler/channel activity where those paths run.

## 2.1.141 Source Index

- `source/src/services/analytics/index.ts`
- `source/src/services/analytics/sink.ts`
- `source/src/services/analytics/config.ts`
- `source/src/services/analytics/datadog.ts`
- `source/src/services/analytics/firstPartyEventLogger.ts`
- `source/src/services/analytics/growthbook.ts`
- `source/src/utils/privacyLevel.ts`
- `source/src/cost-tracker.ts`
- `source/src/bootstrap/state.ts`
