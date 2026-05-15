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

## Detailed Analytics Pipeline

The public event API is small:

- `logEvent(name, metadata)`
- `logEventAsync(name, metadata)`
- `attachAnalyticsSink(sink)`

The important implementation detail is that `logEvent` is callable before the
analytics sink is attached. Events emitted during startup are buffered, and
`attachAnalyticsSink` drains the buffer once initialization chooses the sink.
That is why startup events can appear even before the full application state is
available.

The sink layer in `source/src/services/analytics/sink.ts` controls fanout. In
2.1.141 the meaningful sinks are:

- Datadog log intake for production monitoring.
- first-party OpenTelemetry event logs.
- local debug/logging paths when enabled.

Runtime configuration and privacy checks are performed before event transport.
The code is intentionally defensive about strings: arbitrary code, file paths,
or user prompts should not be passed as event metadata unless a type marker says
the field has been reviewed.

## Disablement Matrix

Telemetry can be reduced at several layers:

- `NODE_ENV=test` disables analytics paths.
- Bedrock, Vertex, and Foundry provider modes disable normal first-party
  analytics.
- `DISABLE_TELEMETRY` maps to a no-telemetry privacy level.
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` maps to essential-only traffic.
- `DISABLE_ERROR_REPORTING` disables error reporting paths.
- `DISABLE_GROWTHBOOK` disables GrowthBook fetches and experiments.
- Sink-specific runtime gates can disable Datadog or first-party transport.

GrowthBook itself depends on first-party event logging being enabled. If the
privacy state disables 1P logging, GrowthBook is also disabled.

## Datadog Sink Details

Datadog transport is configured in `source/src/services/analytics/datadog.ts`.

Key properties:

- Endpoint: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`.
- Production-oriented.
- First-party provider only.
- Uses a public intake token in source.
- Flush interval can be overridden with
  `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS`.
- Event allowlist prevents arbitrary event families from being sent.

Representative Datadog-allowed event areas:

- API success/error.
- session start/exit/resume.
- tool use success/error/cancel/progress.
- OAuth and authentication events.
- brief and Kairos events.
- team and memory events.
- voice mode events.

## First-Party OpenTelemetry Event Logs

The first-party logger emits log records under
`com.anthropic.claude_code.events`.

Record fields include:

- `event_name`
- `event_id`
- `core_metadata`
- `user_metadata`
- `event_metadata`
- `user_id`

Config keys:

- `tengu_event_sampling_config`: event sampling and rate behavior.
- `tengu_1p_event_batch_config`: batch size/timing behavior.
- `tengu_frond_boric`: sink kill switch behavior.

## Metrics Counters

Counters are registered through the bootstrap analytics state and cost tracker.
Observed metric names:

- `claude_code.session.count`
- `claude_code.lines_of_code.count`
- `claude_code.pull_request.count`
- `claude_code.commit.count`
- `claude_code.cost.usage`
- `claude_code.token.usage`
- `claude_code.code_edit_tool.decision`
- `claude_code.active_time.total`

Token usage is tagged by token kind. The relevant token categories are:

- `input`
- `output`
- `cacheRead`
- `cacheCreation`

This separation is why prompt-cache reads and writes can be analyzed separately
from normal input/output.

## Event Catalog by Subsystem

### Startup, Session, and Query

- `tengu_started`
- `tengu_init`
- `tengu_exit`
- `tengu_startup_telemetry`
- `tengu_startup_perf`
- `tengu_session_resumed`
- `tengu_continue`
- `tengu_concurrent_sessions`
- `tengu_concurrent_onquery_detected`
- `tengu_concurrent_onquery_enqueued`
- `tengu_query_error`
- `tengu_query_before_attachments`
- `tengu_query_after_attachments`
- `tengu_orphaned_messages_tombstoned`
- `tengu_post_autocompact_turn`

### API, Model, and Token Budget

- `tengu_api_success`
- `tengu_api_error`
- `tengu_model_fallback_triggered`
- `tengu_max_tokens_escalate`
- `tengu_token_budget_completed`
- `tengu_streaming_tool_execution_used`
- `tengu_streaming_tool_execution_not_used`
- `tengu_cost_threshold_reached`
- `tengu_cost_threshold_acknowledged`
- `tengu_advisor_tool_token_usage`
- `tengu_startup_manual_model_config`
- `tengu_model_command_menu_effort`

### Tool Execution

- `tengu_tool_use_success`
- `tengu_tool_use_error`
- `tengu_tool_use_cancelled`
- `tengu_tool_use_progress`
- `tengu_tool_use_can_use_tool_allowed`
- `tengu_tool_use_can_use_tool_rejected`
- `tengu_deferred_tool_schema_not_sent`
- `tengu_internal_tool_permission_decision`

### Bash, File, Web, and Skill Tools

- `tengu_bash_tool_command_executed`
- `tengu_bash_tool_reset_to_original_dir`
- `tengu_bash_command_timeout_backgrounded`
- `tengu_bash_command_assistant_auto_backgrounded`
- `tengu_bash_command_explicitly_backgrounded`
- `tengu_git_index_lock_error`
- `tengu_code_indexing_tool_used`
- `tengu_tree_sitter_load`
- `tengu_tree_sitter_parse_abort`
- `tengu_file_read_limits_override`
- `tengu_file_read_dedup`
- `tengu_pdf_page_extraction`
- `tengu_session_file_read`
- `tengu_write_claudemd`
- `tengu_edit_string_lengths`
- `tengu_tool_use_diff_computed`
- `tengu_web_fetch_http_error`
- `tengu_web_fetch_host`
- `tengu_skill_tool_invocation`
- `tengu_skill_tool_slash_prefix`
- `tengu_skill_descriptions_truncated`
- `tengu_tool_search_outcome`

### Hooks and Permissions

- `tengu_pre_tool_hooks_cancelled`
- `tengu_pre_tool_hook_error`
- `tengu_post_tool_hooks_cancelled`
- `tengu_post_tool_hook_error`
- `tengu_post_tool_failure_hooks_cancelled`
- `tengu_post_tool_failure_hook_error`
- `tengu_pre_stop_hooks_cancelled`
- `tengu_stop_hook_error`

### Auto Mode

- `tengu_auto_mode_opt_in_dialog_shown`
- `tengu_auto_mode_opt_in_dialog_accept`
- `tengu_auto_mode_opt_in_dialog_accept_default`
- `tengu_auto_mode_opt_in_dialog_decline`
- `tengu_auto_mode_subsequent_approval`
- `tengu_migrate_reset_auto_opt_in_for_default_offer`

### Brief, Channels, and Kairos

- `tengu_brief_mode_enabled`
- `tengu_brief_mode_toggled`
- `tengu_brief_send`
- `tengu_mcp_channel_flags`
- `tengu_mcp_channel_gate`
- `tengu_mcp_channel_enable`
- `tengu_mcp_channel_message`
- `tengu_notification_method_used`

### Teams, Agent View, and Background Tasks

- `tengu_team_created`
- `tengu_team_deleted`
- `tengu_agent_tool_terminated`
- `tengu_agent_flag`
- `tengu_agent_memory_loaded`
- `tengu_transcript_view_enter`
- `tengu_transcript_view_exit`
- `tengu_bg_classify`
- `tengu_bg_agent_terminal`
- `tengu_bg_agent_dispatch`
- `tengu_feature_ok`
- `tengu_feature_sad`

### Dream, Cron, Prompt Suggestion, and Speculation

- `tengu_dream_invoked`
- `tengu_auto_dream_fired`
- `tengu_auto_dream_completed`
- `tengu_auto_dream_failed`
- `tengu_scheduled_task_fire`
- `tengu_scheduled_task_missed`
- `tengu_scheduled_task_expired`
- `tengu_prompt_suggestion_init`
- `tengu_prompt_suggestion`
- `tengu_speculation`

### MCP, Plugins, Bridge, and Remote

- `tengu_vscode_*`
- `tengu_plugin_*`
- `tengu_dynamic_skills_changed`
- `tengu_bridge_*`
- `tengu_remote_create_session`
- `tengu_remote_create_session_error`
- `tengu_remote_create_session_success`
- `tengu_teleport_started`
- `tengu_teleport_cancelled`
- `tengu_teleport_resume_session`
- `tengu_teleport_interactive_mode`

### Memory, Worktree, and UI

- `tengu_memdir_loaded`
- `tengu_memdir_disabled`
- `tengu_team_memdir_disabled`
- `tengu_worktree_created`
- `tengu_worktree_entered_existing`
- `tengu_worktree_kept`
- `tengu_worktree_removed`
- `tengu_custom_keybindings_loaded`
- `tengu_keybinding_fallback_used`
- `tengu_paste_text`
- `tengu_immediate_command_executed`
- `tengu_idle_return_action`
- `tengu_tip_shown`
- `away_summary_generate`
- `tengu_return_to_session`

## Metadata Discipline

The analytics metadata types matter. A field typed as
`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` is a local assertion
that the value is safe for analytics. This is used for event metadata such as
tool names, feature names, model identifiers, mode names, and summarized states.
It should not be used for prompts, code, filesystem paths, raw command strings,
or arbitrary user-controlled text unless the source explicitly sanitizes or
truncates the value.

## Print Mode Relationship

Print mode uses `querySource: 'sdk'` and still runs much of the telemetry
pipeline. It can emit startup, init, query, API, tool, cost, permission, channel,
cron, and error events depending on selected flags. It does not run the normal
interactive REPL UI event paths. Details are in `claude-print-v2.1.141.md`.

## Source-Level Event Matrix

2.1.141 has telemetry calls throughout the product rather than a single
centralized event registry. The practical way to audit events is to search for
`logEvent(` and then group by subsystem. The following matrix captures the
source-visible families that matter for reconstruction.

Startup and session lifecycle:

- `tengu_startup_telemetry`: startup metadata, environment classification, and
  entrypoint context.
- `tengu_started`: session start marker.
- `tengu_exit`: shutdown marker with safe aggregate state.
- `tengu_continue`: resume/continue flow outcomes.
- `tengu_session_resumed`: session resume variants.
- `tengu_concurrent_sessions`: concurrent session detection.
- `tengu_managed_settings_loaded`: managed settings load result.

CLI and mode selection:

- `tengu_code_prompt_ignored`: prompt text ignored by startup rules.
- `tengu_single_word_prompt`: single-token/single-word prompt detection.
- `tengu_agent_flag`: `--agent`/agent-selection startup telemetry.
- `tengu_startup_manual_model_config`: explicit model/env model startup
  selection.
- `tengu_structured_output_enabled`: structured output activation.
- `tengu_structured_output_failure`: structured output parse/validation
  failure.
- `tengu_mcp_channel_flags`: MCP channel flag parsing and enablement.
- `tengu_teleport_interactive_mode`: teleport interactive path.

Model/query loop:

- `tengu_query_before_attachments`: query state before attachment/context
  assembly.
- `tengu_query_after_attachments`: query state after attachment/context
  assembly.
- `tengu_query_error`: query failure.
- `tengu_model_fallback_triggered`: fallback model use.
- `tengu_max_tokens_escalate`: output-token escalation.
- `tengu_token_budget_completed`: token budget completion.
- `tengu_post_autocompact_turn`: post-autocompact follow-up.
- `tengu_auto_compact_succeeded`: compaction success.
- `tengu_streaming_tool_execution_used`: streaming tool path selected.
- `tengu_streaming_tool_execution_not_used`: streaming tool path skipped.

Tool and permission subsystem:

- permission prompts and decisions are emitted around permission evaluation,
  classifier calls, and updates.
- classifier token/cost metrics are tracked separately from ordinary model
  usage.
- shell-readonly and yolo-classifier paths record safe categories and reasons,
  not raw command payloads as event names.
- `tengu_advisor_tool_token_usage` records advisor-side token accounting.

Hooks:

- pre-stop hook cancellation and stop-hook errors are logged as distinct events.
- hook lifecycle events can also be included in `stream-json` output when the
  print-mode flag asks for them.
- hook stdout/stderr is not a telemetry event by default; explicit hook
  execution metadata is the telemetry surface.

Memory and context:

- `tengu_memdir_loaded`: memory directory load result.
- `tengu_memdir_disabled`: auto-memory disabled reason.
- `tengu_team_memdir_disabled`: team memory disabled.
- dynamic skills changed events are emitted when skill directories change.
- agent memory load telemetry records agent-level memory context.

Agent, task, and view system:

- transcript view enter/exit events track Agent View use.
- local/remote/background task lifecycle events feed the same analytics pipeline
  as other UI events.
- team and teammate tools emit creation, deletion, spawn, messaging, and
  failure categories.

Dream, prompt suggestion, and speculation:

- `tengu_auto_dream_fired`: auto dream trigger passed its gates.
- `tengu_auto_dream_completed`: auto dream finished with cache counts.
- `tengu_auto_dream_failed`: auto dream fork/run failure.
- `tengu_dream_invoked`: manual `/dream` mode.
- prompt suggestion events cover initialization, suppression, generation, and
  display.
- speculation events cover activation, boundary, acceptance, time saved, and
  pipelining.

Remote, bridge, and channels:

- remote session creation success/error events.
- deep-link opened telemetry.
- CCR/remote delivery and bridge transport telemetry in lower-level transport
  modules.
- MCP channel flag and permission-relay telemetry.

Plugin and MCP system:

- marketplace/plugin install and autoupdate telemetry.
- MCP server connection and resource/tool discovery telemetry.
- headless plugin sync events are visible in SDK schemas.

Migrations:

- model migration events for Sonnet and Opus defaults.
- auto-update settings migration events.
- bypass-permissions setting migration.
- MCP approval-field migration.
- default offer/auto-mode migration reset events.

The naming pattern is consistent: user-facing product events usually use the
`tengu_` prefix, while metrics and lower-level diagnostic logs use meter,
debug, or OpenTelemetry APIs.

## Entrypoint Attribution

The telemetry code distinguishes entrypoints because identical behavior can mean
different things in different hosts. 2.1.141 recognizes:

- interactive CLI.
- SDK CLI print mode.
- TypeScript SDK.
- Python SDK.
- GitHub Action.
- VS Code.
- local agent.
- Claude Desktop.
- remote/CCR sessions.
- daemon and daemon-worker modes.
- MCP entrypoint.

`CLAUDE_CODE_ENTRYPOINT` is set early if absent, but several hosts set it before
the main CLI action. The source also derives higher-level environment kind from
GitHub Actions variables, remote/session ingress tokens, bridge environment
kind, and explicit entrypoint values.

## Disablement Precedence

Telemetry disablement is layered:

- environment variables can disable analytics/telemetry before runtime state is
  fully initialized.
- settings and managed policy can disable or constrain telemetry after config
  load.
- tests and fixture/VCR paths reduce or redirect telemetry.
- debug logs are separate from analytics events.
- OpenTelemetry metrics can exist even when a product analytics sink is absent,
  depending on configuration.

When auditing privacy behavior, do not assume one flag controls every sink.
Check analytics events, diagnostics logs, OpenTelemetry counters, Datadog sink
configuration, and debug-file behavior separately.

## Metric Instruments

The stable metrics visible in bootstrap state include:

- `claude_code.cost.usage`: cost counter.
- `claude_code.token.usage`: token counter.

Token metrics use type tags, including ordinary input/output, cache read, and
cache creation. Cost metrics are model-aware and depend on local cost tables.
Classifier/advisor/forked-agent paths can add usage to their own aggregates
before contributing to total session cost.

The source also has tracing helpers for session spans and Perfetto traces. Those
helpers accept attributes such as cache read tokens and cache creation tokens,
but they are not the same as user-facing analytics events.

## Safe Metadata Rules

2.1.141 uses typed wrappers and naming discipline to separate safe categorical
metadata from code/file/user content. In practice:

- event names are fixed strings, never raw user content.
- model names, mode names, booleans, counts, and enum-like reasons are normal
  metadata.
- file paths require explicit safe wrappers or redaction.
- commands and prompts are not safe metadata just because they are short.
- token and cost counts are safe aggregate metrics.
- URLs should be redacted or represented by host/category unless code marks
  them safe.

This matters for documentation because a list of event names is not enough. A
good reconstruction must also state what kind of metadata is intentionally
allowed and what remains out of scope.

## Audit Procedure

For future releases, use this telemetry audit sequence:

1. Search `source/src` for `logEvent(` and group by file.
2. Search for `createCounter`, `createHistogram`, and trace helper calls.
3. Search for `CLAUDE_CODE_DISABLE`, `DISABLE_TELEMETRY`, `OTEL`, `DATADOG`,
   and debug-file variables.
4. Read `entrypoints/init.ts`, `setup.ts`, and `main.tsx` for early
   entrypoint/session events.
5. Read query loop files for model, attachment, compaction, and streaming tool
   events.
6. Read tool and permission modules for tool-call and classifier telemetry.
7. Read memory, skills, plugin, MCP, remote, and task modules for background
   subsystem events.
8. Verify print/SDK output schemas separately because stream-json lifecycle
   records are not always analytics events.
9. Run a stale-event-name diff against the previous reconstructed source map and
   manually inspect new `tengu_` strings.
10. Confirm that new event metadata uses safe categorical fields rather than raw
    prompt/code/path data.

## Deep 2.1.141 Telemetry Reconstruction

2.1.141 telemetry is a fanout system: local call sites emit typed analytics,
metrics, trace/log records, debug logs, and stream protocol events. Those are
related but not interchangeable.

### Source Map

| Concern | Source |
| --- | --- |
| Analytics API | `source/src/services/analytics/index.ts` |
| Metadata typing/sanitization | `source/src/services/analytics/metadata.ts` |
| Sink fanout | `source/src/services/analytics/sink.ts` |
| Analytics config | `source/src/services/analytics/config.ts` |
| Datadog exporter | `source/src/services/analytics/datadog.ts` |
| First-party event logging | `source/src/services/analytics/firstPartyEventLogger.ts`, `firstPartyEventLoggingExporter.ts` |
| GrowthBook | `source/src/services/analytics/growthbook.ts` |
| OTel instrumentation | `source/src/utils/telemetry/instrumentation.ts` |
| Perfetto tracing | `source/src/utils/telemetry/perfettoTracing.ts` |
| Session tracing | `source/src/utils/telemetry/sessionTracing.ts` |
| Cost/token counters | `source/src/cost-tracker.ts`, `source/src/bootstrap/state.ts` |
| Print/headless profiling | `source/src/utils/headlessProfiler.ts`, `source/src/cli/print.ts` |

### Event Family Map

| Family | Representative events | Meaning |
| --- | --- | --- |
| Startup/session | `tengu_init`, `tengu_continue`, `tengu_exit`, `tengu_concurrent_sessions` | Session lifecycle and CLI startup properties. |
| Print/headless | `tengu_continue_print`, headless latency events | Non-interactive usage, resume, and latency. |
| Tools | `tengu_bash_tool_command_executed`, `tengu_file_operation`, `tengu_monitor_tool_started` | Tool execution and high-level decisions. |
| Permissions | `tengu_internal_tool_permission_decision`, permission request events | Permission prompts, denials, auto/bypass decisions. |
| Auto mode | `tengu_auto_mode_outcome`, `tengu_auto_mode_decision` | Classifier outcome and safety decisions. |
| Hooks | `tengu_pre_tool_hook_error`, `tengu_post_tool_hook_error`, hook cancellation events | Hook failures/cancellations, not stream hook events. |
| MCP | `tengu_mcp_*` events including auth, channels, list changes, tools loaded | MCP lifecycle and channel behavior. |
| Brief/Kairos | `tengu_brief_mode_enabled`, `tengu_brief_mode_toggled`, `tengu_brief_send` | Brief activation and sends. |
| Agent teams/background | `tengu_team_created`, `tengu_team_deleted`, background/agent events | Swarm/team/background task behavior. |
| Agent View | background session events, fleet/session status events | Background session discovery and attach/ps behavior. |
| Prompt/cache/compact | `tengu_compact`, cache sharing/fallback, prompt cache break events | Context compaction and cache behavior. |
| Settings/config | `tengu_config_changed`, managed settings events, migration events | Config changes and policy state. |
| Plugins/marketplace | marketplace add/remove/update, plugin install events | Plugin lifecycle. |
| IDE/bridge/remote | `tengu_bridge_*`, `tengu_ext_*`, CCR events | Remote control, IDE, and bridge connection behavior. |
| Updates/install | native/package updater/install events | Binary/package update path. |

### Privacy Boundary

The telemetry code uses explicit metadata types such as
`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` to mark values that
have been reviewed as safe categorical data. This is a strong signal in source:
when a field is cast through that type, the author intended to avoid raw code,
paths, or prompt text.

Useful rules for reading 2.1.141 telemetry:

1. Event names are often `tengu_` prefixed, but not every `tengu_` string is
   equally user-facing.
2. Metadata often records counts, booleans, source names, skip kinds, and mode
   names instead of raw content.
3. Some protocol events in stream-json are not analytics events.
4. Debug logs and diagnostics logs are separate from analytics sink events.
5. OTel metrics/counters are separate from event logging even when they share
   source files.

### Controls And Disable Paths

| Control | Effect |
| --- | --- |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Disables or limits nonessential network paths, including some registry/telemetry-adjacent traffic. |
| Privacy/config disable settings | Control analytics behavior through config helpers. |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | Enables OTel instrumentation path. |
| `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS` | Datadog flush timing. |
| `CLAUDE_CODE_OTEL_FLUSH_TIMEOUT_MS` | OTel flush timeout. |
| `CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS` | OTel shutdown timeout. |
| Provider mode env vars | Provider choice affects auth and metadata, not just model routing. |

Do not describe telemetry as a single off/on switch. The runtime contains
separate controls for product analytics, OTel metrics/logs/traces, debug logs,
diagnostics, first-party event logging, and nonessential traffic.

### Print Mode Telemetry

Print mode is tracked as a different surface rather than as "interactive with
no TTY":

| Area | Print-specific behavior |
| --- | --- |
| CLI init | Startup metadata records `print` in `tengu_init` style metadata. |
| Resume | `tengu_continue_print` separates print resume from interactive continue. |
| Query source | API paths can use query source `sdk` or print/headless variants. |
| Hooks | Stream-json can include hook lifecycle subtypes when requested. |
| MCP channels | `channel_enable` control path logs `tengu_mcp_channel_enable`. |
| Headless profiling | Separate latency profiling utilities measure headless paths. |

### Metrics And Counters

Bootstrap state stores counters/meters for:

| Counter | Meaning |
| --- | --- |
| `sessionCounter` | Session count. |
| `locCounter` | Lines-of-code/edit counters. |
| `prCounter` | Pull request count. |
| `commitCounter` | Commit count. |
| `costCounter` | Cost. |
| `tokenCounter` | Token usage. |
| `codeEditToolDecisionCounter` | Edit tool decision metrics. |
| `activeTimeCounter` | Active time. |
| `statsStore` | Local stats observations. |

These are metric observations, not necessarily user-visible analytics events.

### Future Diff Process

For future releases, diff telemetry in this order:

1. Extract every `logEvent(` call and group by source file.
2. Extract every `tengu_` literal and classify whether it is analytics,
   stream protocol, debug, or test/config data.
3. Extract metadata type casts to identify intentionally safe categorical
   fields.
4. Compare GrowthBook keys because many telemetry and feature decisions are
   coupled.
5. Compare print/SDK paths separately from interactive REPL paths.
6. Compare bridge/remote/CCR paths separately because those include session and
   transport metadata not present locally.
