# Claude Code `-p` / `--print` Mode in 2.1.141

This document describes the non-interactive print path in the reconstructed
2.1.141 source. It focuses on what `claude -p` and `claude --print` do, how
input/output modes work, how the SDK-style stream is driven, and what telemetry
records print-mode usage.

Primary source files:

- `source/src/main.tsx`
- `source/src/cli/print.ts`
- `source/src/cli/structuredIO.ts`
- `source/src/cli/remoteIO.ts`
- `source/src/QueryEngine.ts`
- `source/src/utils/messages/systemInit.ts`
- `source/src/entrypoints/sdk/coreSchemas.ts`
- `source/src/entrypoints/sdk/controlSchemas.ts`
- `source/src/services/analytics/metadata.ts`
- `source/src/services/api/client.ts`
- `source/src/utils/http.ts`
- `source/src/constants/system.ts`
- `source/src/utils/headlessProfiler.ts`

## Summary

`-p` / `--print` is Claude Code's non-interactive execution mode. It accepts a
prompt from argv, stdin, or a streaming SDK input channel, runs the normal agent
loop without rendering the interactive Ink UI, emits the final answer or SDK
messages to stdout, and exits.

Internally the mode is treated as a headless SDK-like session:

- `main.tsx` marks the session non-interactive.
- `CLAUDE_CODE_ENTRYPOINT` defaults to `sdk-cli` unless a caller already set a
  more specific entrypoint.
- Workspace trust dialogs are skipped.
- Full config environment variables are applied before telemetry starts.
- MCP, hooks, plugins, commands, agents, and permissions are initialized into a
  headless app-state store.
- `source/src/cli/print.ts` owns the runtime queue, streaming protocol, control
  requests, permissions, background task draining, and output formatting.
- `source/src/QueryEngine.ts` runs query turns with `querySource: 'sdk'`.

The option help text says:

```text
-p, --print
Print response and exit (useful for pipes). Note: The workspace trust dialog is
skipped when Claude is run in non-interactive mode (via -p, or when stdout is
not a TTY, e.g. piped or redirected output). Only use this in directories you
trust.
```

## Non-Interactive Detection

`main.tsx` performs an early argv/stdout check before full initialization:

- `-p`
- `--print`
- `--init-only`
- `--sdk-url...`
- `!process.stdout.isTTY`

Any of these makes the session non-interactive. In that case Claude:

- stops early input capture;
- calls `setIsInteractive(false)`;
- initializes the entrypoint as non-interactive;
- skips interactive setup/trust UI;
- takes the print/headless path after setup.

The entrypoint initializer respects a pre-existing `CLAUDE_CODE_ENTRYPOINT`.
If nothing set it, the default is:

- `mcp` for `claude mcp serve`;
- `claude-code-github-action` when `CLAUDE_CODE_ACTION` is truthy;
- `sdk-cli` for non-interactive sessions;
- `cli` otherwise.

The client type is derived separately:

- `GITHUB_ACTIONS` -> `github-action`;
- `sdk-ts` -> `sdk-typescript`;
- `sdk-py` -> `sdk-python`;
- `sdk-cli` -> `sdk-cli`;
- `claude-vscode` -> `claude-vscode`;
- `local-agent` -> `local-agent`;
- `claude-desktop` -> `claude-desktop`;
- `CLAUDE_CODE_ENTRYPOINT=remote` or a session-ingress token -> `remote`;
- otherwise `cli`.

## CLI Options Related to Print Mode

Print mode is shaped by these top-level options.

### Core Print Options

- `-p, --print`: enable print/headless mode.
- `--output-format <format>`: only intended for print mode; choices are `text`,
  `json`, and `stream-json`.
- `--input-format <format>`: only intended for print mode; choices are `text`
  and `stream-json`.
- `--verbose`: in stream-json mode, required by `print.ts`; SDK URL mode
  auto-enables it unless the caller explicitly set verbose.

### Structured Output

- `--json-schema <schema>`: JSON Schema for structured output validation.
- When synthetic structured output is enabled, `main.tsx` parses the schema,
  creates the synthetic output tool, appends it to the tool list, and logs
  structured-output telemetry.

### Streaming Detail

- `--include-hook-events`: include all hook lifecycle events in stream-json.
- `--include-partial-messages`: include partial raw stream chunks as they arrive;
  requires print mode and `--output-format=stream-json`.
- `--replay-user-messages`: re-emit input user/control messages as stdout
  acknowledgments; requires stream-json input and stream-json output.

### Session Control

- `-c, --continue`: continue the most recent conversation in the current
  directory.
- `-r, --resume [value]`: resume by session ID, JSONL file, URL, or title.
- `--fork-session`: when resuming, create a new session ID instead of reusing
  the original.
- `--resume-session-at <message id>`: when resuming, truncate the loaded
  conversation through a specific message UUID.
- `--rewind-files <user-message-id>`: with `--resume`, restore files to the
  checkpoint for a user message and exit.
- `--no-session-persistence`: disables transcript persistence; only allowed in
  print mode.

### Permission / Tool Control

- `--allowedTools` / `--allowed-tools`: allow tool names or patterns.
- `--disallowedTools` / `--disallowed-tools`: deny tool names or patterns.
- `--tools`: restrict the built-in tool set.
- `--permission-mode <mode>`: sets permission behavior for the session.
- `--permission-prompt-tool <tool>`: use an MCP tool as a permission prompt
  handler; only intended for print mode.
- `--dangerously-skip-permissions`: bypass permission checks.
- `--allow-dangerously-skip-permissions`: make bypass mode available without
  starting in it.

### SDK / Remote Control

- `--sdk-url <url>`: hidden option for remote WebSocket/SSE SDK I/O streaming.
  It auto-sets:
  - `inputFormat = stream-json` when not explicitly set;
  - `outputFormat = stream-json` when not explicitly set;
  - `verbose = true` when not explicitly set;
  - `print = true` when not already set.

### Model and Prompt Control

- `--model <model>`
- `--fallback-model <model>`
- `--effort <level>`
- `--thinking <mode>`
- `--max-thinking-tokens <tokens>` hidden/deprecated
- `--max-turns <turns>`
- `--max-budget-usd <amount>`
- `--task-budget <tokens>`
- `--system-prompt <prompt>`
- `--system-prompt-file <file>`
- `--append-system-prompt <prompt>`
- `--append-system-prompt-file <file>`
- `--agent <agent>`
- `--agents <json>`
- `--workload <tag>` hidden; feeds billing-header workload attribution.

### Startup / Environment Control

- `--bare`: minimal mode. It skips hooks, LSP, plugin sync, attribution,
  auto-memory, background prefetches, keychain reads, and CLAUDE.md
  auto-discovery. Auth is limited to `ANTHROPIC_API_KEY` or an `apiKeyHelper`
  supplied through `--settings`.
- `--init`, `--init-only`, `--maintenance`: setup/session-start hook variants.
- `--mcp-config <configs...>`
- `--strict-mcp-config`
- `--settings <file-or-json>`
- `--setting-sources <sources>`
- `--add-dir <directories...>`
- `--plugin-dir <path>`
- `--plugin-url <url>`
- `--betas <betas...>`
- `--session-id <uuid>`
- `--name <name>`

## Input Handling

`main.tsx` calls `getInputPrompt(prompt, inputFormat)`.

If stdin is not a TTY and the command is not `mcp`:

- `input-format=stream-json`: returns `process.stdin` as an async iterable;
- text input: reads UTF-8 stdin, waits up to three seconds for data, warns on
  timeout, and joins argv prompt plus stdin data with a newline.

If stdin is a TTY, the input prompt is the positional prompt argument.

`print.ts` then wraps the input in `StructuredIO`:

- A string prompt becomes a one-line SDK user message:

```json
{
  "type": "user",
  "session_id": "",
  "message": {
    "role": "user",
    "content": "..."
  },
  "parent_tool_use_id": null
}
```

- An async iterable is consumed line-by-line as NDJSON.
- `--sdk-url` uses `RemoteIO`, which subclasses `StructuredIO` and bridges the
  same stream protocol over WebSocket/SSE transport.

`StructuredIO` accepts these stdin message classes:

- SDK user messages;
- SDK control requests;
- SDK control responses;
- keep-alives;
- runtime environment variable updates.

Empty lines are skipped. Invalid JSON lines are fatal.

## Validation Rules

The print path performs several validations before and inside `runHeadless`.

From `main.tsx`:

- `--input-format` must be `text` or `stream-json`.
- `--input-format=stream-json` requires `--output-format=stream-json`.
- `--sdk-url` requires both stream-json input and stream-json output.
- `--replay-user-messages` requires stream-json input and stream-json output.
- `--include-partial-messages` requires print mode and stream-json output.
- `--no-session-persistence` requires print mode.
- `--session-id` must be a UUID unless `--sdk-url` is used.
- `--session-id` can be combined with `--continue` or `--resume` only when
  `--fork-session` is present.

From `print.ts`:

- `--resume-session-at` requires `--resume`.
- `--rewind-files` requires `--resume`.
- `--rewind-files` cannot be combined with a prompt.
- If there is no prompt, no valid resume target, and no SDK URL, print mode
  errors with "Input must be provided either through stdin or as a prompt
  argument when using --print".
- `--output-format=stream-json` requires `--verbose`.
- `--rewind-files` requires the target UUID to be a user message.
- `--permission-prompt-tool` must name an available MCP tool and that tool must
  have an input JSON schema.

## Headless Startup Flow

In non-interactive mode `main.tsx` does the following before importing
`cli/print.ts`:

1. Marks formatted output if `output-format` is `json` or `stream-json`.
2. Applies full config environment variables because the trust dialog is
   bypassed.
3. Initializes telemetry after config environment variables are applied.
4. Starts `SessionStart` hooks early unless continue/resume/teleport/setup
   paths require different ordering.
5. Validates org restrictions.
6. Filters available slash commands to prompt commands that are allowed in
   non-interactive mode plus local commands with `supportsNonInteractive`.
7. Builds a headless app-state store from default app state, MCP state,
   permissions, effort, fast mode, advisor model, and Kairos/proactive state.
8. Starts async kill-switch checks for bypass permissions and auto mode where
   applicable.
9. Applies `--no-session-persistence` to global state.
10. Stores SDK beta headers in global state.
11. Connects configured MCP servers into the headless store.
12. Fetches claude.ai MCP connectors when eligible, dedups them against manual
    and plugin MCP configs, and waits up to five seconds for those connector
    MCPs before proceeding.
13. Starts deferred prefetch/background housekeeping unless bare mode is active.
14. Logs session telemetry.
15. Imports `runHeadless` from `source/src/cli/print.ts` and calls it.

The print-specific MCP connection design matters: single-turn print sessions
need configured MCP tools available on turn one, so regular MCP configs are
awaited. Claude.ai connector MCPs are bounded by a five-second wait and may
finish in the background.

## `runHeadless`

`runHeadless` is the top-level print runtime. It:

- downloads user settings in remote/headless contexts when enabled;
- subscribes directly to settings changes because there is no React tree;
- activates proactive mode fallback when necessary;
- starts periodic Bun GC;
- initializes GrowthBook for feature flags;
- validates resume/rewind options;
- creates `StructuredIO` or `RemoteIO`;
- installs the stream-json stdout guard;
- initializes sandboxing and sandbox network permission callbacks;
- registers stream-json hook lifecycle emission when verbose stream-json is on;
- runs setup hooks when requested;
- loads initial messages for continue/resume/teleport/startup;
- prepends hook-provided `initialUserMessage` when present;
- restores agent settings from resumed sessions unless overridden;
- handles file rewind as a standalone operation;
- builds `canUseTool`;
- filters denied MCP tools;
- initializes model strings;
- iterates `runHeadlessStreaming`;
- writes final output in text/json/stream-json format;
- logs headless latency metrics;
- drains memory extraction when enabled;
- exits with status `1` if the last result is an error, otherwise `0`.

## Session Load Modes

`loadInitialMessages` in `print.ts` handles print-mode resume and startup
state.

### Continue

`--continue`:

- logs `tengu_continue_print`;
- loads the most recent local conversation;
- matches coordinator mode to the resumed session when the feature exists;
- reuses the old session ID unless `--fork-session` is set;
- restores session state, metadata, mode, turn interruption state, and agent
  setting.

### Teleport

`--teleport`:

- requires policy `allow_remote_sessions`;
- logs `tengu_teleport_print`;
- validates git state;
- resumes a teleported code session;
- checks out the teleported branch;
- converts teleport messages for resume.

### Resume

`--resume`:

- logs `tengu_resume_print`;
- requires a valid session ID, JSONL file, URL, or title in print mode;
- optionally hydrates remote session data;
- loads local conversation data;
- starts with empty `SessionStart` messages for empty remote/CCR v2 sessions;
- applies `--resume-session-at` truncation;
- matches coordinator mode;
- reuses old session ID unless `--fork-session` is set;
- restores state, metadata, mode, turn interruption state, and agent setting.

### New Session

With no continue/resume/teleport path, startup messages are the result of
`processSessionStartHooks('startup')`. `main.tsx` often kicks this promise early
and `print.ts` joins it later.

## Output Formats

### Text

This is the default output format. Only the final `result` message is rendered
to stdout:

- success: writes `result`, ensuring a trailing newline;
- `error_during_execution`: writes `Execution error`;
- `error_max_turns`: writes `Error: Reached max turns (...)`;
- `error_max_budget_usd`: writes `Error: Exceeded USD budget (...)`;
- `error_max_structured_output_retries`: writes a structured-output retry
  failure message.

### JSON

`--output-format=json` writes JSON to stdout after the run completes:

- without `--verbose`: writes only the final `result` message;
- with `--verbose`: writes the accumulated SDK messages array.

`print.ts` only retains the full message array in memory for `json + verbose`.
Text and stream-json retain only the last relevant message.

### Stream JSON

`--output-format=stream-json --verbose` writes NDJSON SDK messages as they
arrive. `print.ts` installs a stdout guard so stray non-JSON writes are diverted
to stderr rather than corrupting the stream.

In stream-json mode:

- SDK messages are written immediately.
- Hook lifecycle events are emitted when hook events are enabled and verbose
  stream-json is active.
- Control requests/responses and stream events use the same stdout channel.
- `RemoteIO` can mirror the same protocol over `--sdk-url`.
- A stream transformer may emit streamlined message types when the
  `STREAMLINED_OUTPUT` feature and `CLAUDE_CODE_STREAMLINED_OUTPUT` env var are
  both active.

## SDK Message Shapes

The public output schema is built in `source/src/entrypoints/sdk/coreSchemas.ts`.
Important stdout message families include:

- `assistant`: assistant API messages.
- `user`: user messages and replay acknowledgments.
- `result`: terminal success or error result.
- `system/init`: session metadata.
- `system/status`: requesting/compacting/permission-mode status.
- `stream_event`: raw partial assistant stream event, enabled by
  `--include-partial-messages`.
- `system/api_retry`: retry notices.
- `system/local_command_output`: local slash-command output.
- `system/hook_started`, `system/hook_progress`, `system/hook_response`: hook
  lifecycle events.
- `system/plugin_install`: headless plugin installation progress.
- `tool_progress`: tool progress.
- `auth_status`: cloud auth progress when enabled.
- `system/task_started`, `system/task_updated`, `system/task_progress`,
  `system/task_notification`: background task and agent events.
- `system/session_state_changed`: `idle`, `running`, or `requires_action`.
- `system/notification`: loop-side notifications.
- `system/files_persisted`: file persistence output.
- `tool_use_summary`: summarized tool activity.
- `system/memory_recall`: memory recall event.
- `rate_limit_event`: rate-limit state change.
- `system/elicitation_complete`: URL-mode MCP elicitation completion.
- `system/permission_denied`: auto-denied tool call event.
- `prompt_suggestion`: predicted next prompt when enabled.
- `system/mirror_error`: transcript mirror failure.

The final success result includes:

- `duration_ms`
- `duration_api_ms`
- `is_error`
- `num_turns`
- `result`
- `stop_reason`
- `total_cost_usd`
- `usage`
- `modelUsage`
- `permission_denials`
- `structured_output` when structured output is active
- `fast_mode_state`
- `api_error_status`
- `deferred_tool_use`
- `terminal_reason`
- `origin`
- `uuid`
- `session_id`

Error results include similar accounting fields plus `errors`.

## `system/init`

`QueryEngine` emits a `system/init` SDK message through
`buildSystemInitMessage`. This is the first SDK stream metadata message for the
SDK/print query path. It carries:

- `cwd`
- `session_id`
- `tools`
- `mcp_servers`
- `model`
- `permissionMode`
- `slash_commands`
- `apiKeySource`
- `betas`
- `claude_code_version`
- `output_style`
- `agents`
- `skills`
- `plugins`
- `fast_mode_state`
- `uuid`

Tool names are compatibility-mapped so the wire name for the Agent tool remains
the legacy `Task` name in init/result events.

## Streaming Runtime

`runHeadlessStreaming` is the long-lived async generator behind print mode.

It creates two cooperating loops:

- an stdin/control loop reading `StructuredIO.structuredInput`;
- a queued-command execution loop that drains prompts and background events.

The generator returns when stdin closes and queued work, background agents, async
hooks, suggestions, and teardown work have drained.

### Command Queue

Input user messages are enqueued as prompt commands. Consecutive prompt commands
can be batched into one ask call when:

- both are prompt-mode commands;
- workload tags match;
- the hidden/meta flag matches.

When `--replay-user-messages` is enabled, batched messages that do not survive
as the final prompt UUID are explicitly replayed so SDK clients can acknowledge
each UUID.

The queue also handles:

- background task notifications;
- orphaned permission responses;
- channel messages;
- proactive ticks;
- cron-triggered prompts;
- teammate/swarm messages.

### Query Execution

For each queued prompt, `print.ts`:

1. Refreshes SDK/dynamic MCP state.
2. Registers MCP elicitation handlers.
3. Registers channel notification handlers.
4. Builds the full tool list.
5. Starts query profiling.
6. Calls `ask(...)`.
7. Forwards messages to bridge remote-control if active.
8. Holds the final result while background agents are running.
9. Emits task/progress SDK events.
10. Runs file persistence and prompt suggestion side work when enabled.
11. Logs per-turn headless/query profile data.

`ask(...)` ultimately uses `source/src/query.ts` with `querySource: 'sdk'`.

### Background Agent Draining

The loop does not finish immediately when the main turn returns if background
agents or workflows are still running. Instead, it waits, drains task
notifications into the model, and only emits the held-back result after
background work settles. Long-lived in-process teammates are excluded from the
wait because they are designed to remain running.

### Ctrl+C

`print.ts` installs its own SIGINT handler:

- aborts the active query controller if present;
- calls graceful shutdown;
- lets shutdown persist state and flush analytics.

## Control Protocol

`StructuredIO` and `print.ts` implement an SDK control protocol over the same
NDJSON stream.

Incoming control requests handled by `print.ts` include:

- `interrupt`
- `end_session`
- `initialize`
- `set_permission_mode`
- `set_model`
- `set_max_thinking_tokens`
- `mcp_status`
- `get_context_usage`
- `mcp_message`
- `rewind_files`
- `cancel_async_message`
- `seed_read_state`
- `mcp_set_servers`
- `reload_plugins`
- `mcp_reconnect`
- `mcp_toggle`
- `channel_enable`
- `mcp_authenticate`
- `mcp_oauth_callback_url`
- `claude_authenticate`
- `claude_oauth_callback`
- `claude_oauth_wait_for_completion`
- `mcp_clear_auth`
- `stop_task`
- `apply_flag_settings`
- `get_settings`
- `generate_session_title`
- `side_question`
- `set_proactive` behind feature gates
- `remote_control`

Outgoing control requests from `StructuredIO` include:

- `can_use_tool`
- `hook_callback`
- `elicitation`
- `mcp_message`
- sandbox network permission requests represented as `can_use_tool` using the
  synthetic tool name `SandboxNetworkAccess`

`control_cancel_request` is emitted when a pending control request is aborted or
resolved elsewhere.

## Permissions in Print Mode

Print mode uses the same permission engine as interactive mode, but without a
terminal dialog.

Possible permission paths:

- Automatic allow/deny from existing permission rules, mode, classifier, hooks,
  or safety checks.
- `--permission-prompt-tool <tool>` delegates ask decisions to a named MCP tool.
- `--permission-prompt-tool stdio` uses the SDK control protocol.
- `--sdk-url` forces the effective permission prompt tool name to `stdio`, so
  permission prompts are delegated to the SDK/remote host.
- `PermissionRequest` hooks race against SDK permission prompts in
  `StructuredIO.createCanUseTool`. The first decision wins.

When a tool is auto-denied without an interactive permission prompt, SDK clients
can receive `system/permission_denied`.

`can_use_tool` requests include:

- `tool_name`
- `input`
- `permission_suggestions`
- `blocked_path`
- `decision_reason`
- `tool_use_id`
- `agent_id`
- optional description fields

## Hooks in Print Mode

Hooks are supported in print mode unless disabled by bare mode or configuration.

Important behaviors:

- `main.tsx` can start `SessionStart` hooks before importing `print.ts`.
- `runHeadless` runs setup hooks for `--init`, `--init-only`, and
  `--maintenance` flows.
- Stream-json hook lifecycle messages are emitted when verbose stream-json is
  active and hook event emission is enabled.
- SDK hosts can register callback hooks through the `initialize` control
  request.
- `PermissionRequest` hooks race SDK/stdio permission prompts.
- `finalizePendingAsyncHooks()` is awaited before output closes.

`--include-hook-events` enables all hook event types for the stream. Remote CCR
mode also enables all hook events because CCR needs them.

## MCP and Elicitation

Print mode supports configured MCP servers, SDK-injected MCP servers, dynamic MCP
servers, plugin MCP servers, and claude.ai connector MCPs.

MCP details:

- SDK MCP servers are tracked in `sdkMcpConfigs`, `sdkClients`, and `sdkTools`.
- Dynamic servers can be replaced through `mcp_set_servers`.
- `mcp_status` returns status, config, tool names, annotations, scope, and
  experimental capabilities.
- `mcp_reconnect` and `mcp_toggle` update both app state and dynamic MCP state.
- SDK MCP servers route messages through `StructuredIO.sendMcpMessage`.
- Non-SDK MCP elicitation requests run elicitation hooks first, then delegate to
  the SDK host if no hook responds.
- URL-mode MCP elicitation completion emits `system/elicitation_complete`.

Elicitation telemetry is emitted for shown/response states.

## RemoteIO / `--sdk-url`

When `--sdk-url` is used:

- `RemoteIO` is used instead of plain `StructuredIO`.
- It creates a transport based on the URL.
- Session-ingress auth is taken from `CLAUDE_CODE_SESSION_ACCESS_TOKEN` or the
  websocket auth file descriptor helper.
- `x-environment-runner-version` is sent when
  `CLAUDE_CODE_ENVIRONMENT_RUNNER_VERSION` exists.
- CCR v2 uses SSE transport plus `CCRClient` when `CLAUDE_CODE_USE_CCR_V2` is
  truthy.
- Bridge mode is detected by `CLAUDE_CODE_ENVIRONMENT_KIND=bridge`.
- In bridge debug mode, inbound data can be mirrored to stdout.

The bridge session runner spawns child Claude processes with:

- `--print`
- `--sdk-url`
- `--session-id`
- `--input-format stream-json`
- `--output-format stream-json`
- `--replay-user-messages`
- `--verbose` when enabled
- `CLAUDE_CODE_ENVIRONMENT_KIND=bridge`
- `CLAUDE_CODE_SESSION_ACCESS_TOKEN`

## Bare Mode Interaction

`--bare` is a performance and hermeticity mode often relevant to scripted
`-p` usage. It sets simple mode and skips:

- hooks;
- LSP;
- plugin sync;
- attribution;
- auto-memory;
- background prefetches;
- keychain reads;
- CLAUDE.md auto-discovery.

It also limits auth for Anthropic API use to `ANTHROPIC_API_KEY` or an
`apiKeyHelper` explicitly supplied through `--settings`.

## Authentication Notes

`utils/auth.ts` has a print/CI-oriented path for direct environment API keys.
When third-party authentication is preferred and `ANTHROPIC_API_KEY` is present,
the direct key is used. Bare mode is stricter: it never reads keychain or config
auth sources.

Print mode can emit `auth_status` SDK messages when `--enable-auth-status` is
set. Despite the manager name in source, the auth status manager is reused for
AWS/GCP-style authentication output.

## Telemetry and Tracking

Telemetry for print usage is spread across startup, print runtime, query runtime,
HTTP headers, and global event metadata.

### `tengu_init`

`main.tsx` logs `tengu_init` for both interactive and print sessions. For print
mode, the key usage fields include:

- `entrypoint: 'claude'`
- `hasInitialPrompt`
- `hasStdin`
- `verbose`
- `debug`
- `debugToStderr`
- `print`
- `outputFormat`
- `inputFormat`
- `numAllowedTools`
- `numDisallowedTools`
- `mcpClientCount`
- `worktree`
- `skipWebFetchPreflight`
- `githubActionInputs`
- `dangerouslySkipPermissionsPassed`
- `permissionMode`
- `modeIsBypass`
- `inProtectedNamespace`
- `allowDangerouslySkipPermissionsPassed`
- `thinkingType`
- `systemPromptFlag`
- `appendSystemPromptFlag`
- `is_simple`
- `is_coordinator`
- `assistantActivationPath`
- `autoUpdatesChannel`

The `print` boolean is the direct indicator that `--print`/SDK URL/other
non-interactive parsing led to print mode.

### Structured Output Events

When `--json-schema` is supplied and synthetic structured output is enabled:

- `tengu_structured_output_enabled`
  - `schema_property_count`
  - `has_required_fields`
- `tengu_structured_output_failure`
  - `error: 'Invalid JSON schema'`

### Resume/Continue/Teleport Events

`print.ts` logs:

- `tengu_continue_print`
- `tengu_resume_print`
- `tengu_teleport_print`

These distinguish print-mode session loading from interactive resume telemetry.

### Runtime Print Events

`print.ts` logs:

- `tengu_mcp_elicitation_shown`
  - `mode`
- `tengu_mcp_elicitation_response`
  - `mode`
  - `action`
- `tengu_sync_plugin_install_timeout`
  - `timeout_ms`
- `tengu_bridge_message_received`
  - `is_repl: false`
- `tengu_oauth_flow_start`
  - `loginWithClaudeAi`
- `tengu_oauth_success`
  - `loginWithClaudeAi`
- `tengu_mcp_channel_enable`
  - `plugin`
- `tengu_mcp_channel_message`
  - `content_length`
  - `meta_key_count`
  - `entry_kind`
  - `is_dev`
  - `plugin`

### Query Events

Print-mode turns run through `QueryEngine` and `query.ts` with
`querySource: 'sdk'`. Query-level telemetry from `query.ts` therefore applies to
print/SDK runs, including:

- `tengu_auto_compact_succeeded`
- `tengu_orphaned_messages_tombstoned`
- `tengu_model_fallback_triggered`
- `tengu_query_error`
- `tengu_max_tokens_escalate`
- `tengu_token_budget_completed`
- `tengu_streaming_tool_execution_used`
- `tengu_streaming_tool_execution_not_used`
- `tengu_post_autocompact_turn`
- `tengu_query_before_attachments`
- `tengu_query_after_attachments`

The query source value is important because it distinguishes SDK/print from
interactive `repl_main_thread` and from subagent/query helper sources.

### Headless Latency

`utils/headlessProfiler.ts` can log `tengu_headless_latency` when sampled. It
adds:

- phase/checkpoint timing;
- query overhead;
- checkpoint count;
- `entrypoint` from `CLAUDE_CODE_ENTRYPOINT`.

This is specific to headless/print performance analysis.

### Global Event Metadata

Every analytics event is enriched by `services/analytics/metadata.ts`. Relevant
print/harness fields include:

- `entrypoint` from `CLAUDE_CODE_ENTRYPOINT`;
- `agentSdkVersion` from `CLAUDE_AGENT_SDK_VERSION`;
- `isInteractive`;
- `clientType`;
- `envContext.terminal`;
- `envContext.deploymentEnvironment`;
- `envContext.isCi`;
- `envContext.isClaudeCodeRemote`;
- `envContext.isLocalAgentMode`;
- `envContext.remoteEnvironmentType`;
- `envContext.claudeCodeContainerId`;
- `envContext.claudeCodeRemoteSessionId`;
- `envContext.tags`;
- GitHub Actions fields when present.

First-party event logging converts some fields to snake_case:

- `client_type`
- `entrypoint`
- `agent_sdk_version`
- `is_interactive`

### HTTP and Attribution Headers

API requests include:

- `User-Agent`: `claude-cli/<version> (<USER_TYPE>, <CLAUDE_CODE_ENTRYPOINT>...)`
- optional `agent-sdk/<CLAUDE_AGENT_SDK_VERSION>` in the user agent;
- optional `client-app/<CLAUDE_AGENT_SDK_CLIENT_APP>` in the user agent;
- `X-Claude-Code-Session-Id`;
- optional `x-claude-remote-container-id`;
- optional `x-claude-remote-session-id`;
- optional `x-client-app` from `CLAUDE_AGENT_SDK_CLIENT_APP`.

The attribution header helper creates:

```text
x-anthropic-billing-header: cc_version=<version.fingerprint>; cc_entrypoint=<entrypoint>; ...
```

When workload is set, it adds `cc_workload=<workload>`.

## Privacy / Disablement Notes

Telemetry can be suppressed globally through analytics/privacy configuration.
The SDK `system/init` message can also include `analytics_disabled` according to
the SDK schema, allowing IDE clients to hide per-message feedback UI when
analytics would be a no-op.

## Practical Examples

Text prompt:

```bash
claude -p "explain this repository"
```

Prompt from stdin:

```bash
printf '%s\n' "summarize this file" | claude --print
```

JSON result:

```bash
claude -p --output-format json "list the public APIs"
```

Verbose stream-json:

```bash
claude -p --input-format stream-json --output-format stream-json --verbose
```

Stream-json with hook lifecycle events:

```bash
claude -p --output-format stream-json --verbose --include-hook-events "run checks"
```

Structured output:

```bash
claude -p \
  --output-format json \
  --json-schema '{"type":"object","properties":{"summary":{"type":"string"}},"required":["summary"]}' \
  "summarize the project"
```

Resume and fork:

```bash
claude -p --resume <session-id> --fork-session "continue from there"
```

Rewind files and exit:

```bash
claude -p --resume <session-id> --rewind-files <user-message-uuid>
```

## Key Takeaways

- `--print` is not a thin `console.log` path. It is a full headless SDK
  runtime.
- It skips workspace trust UI and applies config environment variables directly.
- The default output is plain final text; `json` and `stream-json` expose SDK
  message structures.
- Stream-json requires `--verbose`.
- `--sdk-url` implicitly converts the session into verbose stream-json print
  mode.
- Permission prompts are delegated through SDK control requests or MCP prompt
  tools rather than terminal UI.
- Telemetry tracks print usage through `tengu_init.print`, print-specific
  resume events, query source `sdk`, headless latency events, HTTP user-agent
  fields, attribution headers, and global metadata.

## Deep 2.1.141 Print/SDK Runtime Addendum

The 2.1.141 `-p` path is implemented mostly in `main.tsx`, `cli/print.ts`,
`cli/structuredIO.ts`, `cli/transports/*`, and `entrypoints/sdk/*`. It is the
same path used by the Agent SDK subprocess protocol, not a separate lightweight
request wrapper.

### CLI Option Contract

`main.tsx` defines:

- `-p, --print`: print response and exit.
- `--output-format text|json|stream-json`: only meaningful with `--print`.
- `--input-format text|stream-json`: only meaningful with `--print`.
- `--json-schema <schema>`: structured-output schema validation.
- `--include-hook-events`: include all hook lifecycle events in stream-json output.
- `--include-partial-messages`: include partial chunks with print/stream-json.
- `--max-turns`, `--max-budget-usd`, `--task-budget`: non-interactive execution caps.
- `--replay-user-messages`: re-emit stdin user messages for stream-json ack.
- `--permission-prompt-tool`: use an MCP tool for permission prompts.
- `--workload`: hidden billing/workload attribution tag.
- `--sdk-url`: hidden remote WebSocket endpoint for SDK I/O streaming.

`--sdk-url` forces the process into print mode if the caller did not explicitly
set `--print`, defaults both input and output formats to `stream-json`, and
defaults verbose mode on. It also relaxes local UUID validation for
`--session-id`, because the remote session id may be server-assigned rather than
a local UUID.

### Validation And Failure-Closed Cases

The source enforces several hard constraints before the run loop:

- `--input-format=stream-json` requires `--output-format=stream-json`.
- `--sdk-url` requires both input and output to be `stream-json`.
- `--replay-user-messages` requires stream-json input and output.
- `--include-partial-messages` requires `--print` and `--output-format=stream-json`.
- `--output-format=stream-json` requires `--verbose`.
- a normal print run needs either stdin/prompt input, a valid resume target, or `--sdk-url`.
- `--rewind-files` requires `--resume` and cannot be combined with a prompt.
- `--resume-session-at` requires `--resume`.
- `--no-session-persistence` is only valid with print mode.

Those checks are important for automation because malformed SDK invocations exit
early rather than falling back to interactive behavior.

### Stdout Discipline

`cli/print.ts` installs a stream-json stdout guard before the first structured
write. The reason is explicit in the source: any stray `console.log`, dependency
banner, or debug line would corrupt the line-by-line JSON parser used by SDK
clients. Non-JSON stdout is diverted to stderr in stream-json mode.

That makes `stream-json` a protocol boundary. Docs and future maps should not
treat stdout as generic log output once this mode is active.

### Output Modes

The output modes differ materially:

- `text`: writes only the final successful result text, with a trailing newline.
- `json`: writes the final `result` message unless `--verbose` requests the full message array.
- `stream-json`: writes structured messages during execution and does not print a final extra wrapper after the stream.

During streaming, print mode suppresses some internal/system-only messages from
the `lastMessage` calculation so the final text/json result is not replaced by
late `session_state_changed`, `task_notification`, `task_started`,
`task_progress`, `post_turn_summary`, `stream_event`, `keep_alive`,
`prompt_suggestion`, or streamlined output messages.

### Hook Event Streaming

When output is `stream-json` and verbose, print mode registers a hook event
handler that emits:

- `system/hook_started`.
- `system/hook_progress`.
- `system/hook_response`.

Each event includes hook id/name/event plus stdout/stderr/output/exit/outcome
fields as appropriate. The `--include-hook-events` flag, or
`CLAUDE_CODE_REMOTE`, enables all hook event types rather than the limited
startup subset. This is why hook docs should treat stream-json as a first-class
observability surface.

### Permission Prompt Routing

Print mode does not show the interactive terminal permission dialog. It builds
`canUseTool` through `getCanUseToolFn()` with either:

- a configured MCP permission-prompt tool.
- `stdio` when `--sdk-url` is active.
- the normal non-interactive permission mode and rules otherwise.

When a permission prompt is shown, `notifySessionStateChanged('requires_action',
details)` is called. With SDK transport this becomes a control request for the
host. With bridge/remote transport it can be relayed to the remote UI. The
prompt path increments attribution counters when the commit-attribution feature
is active.

### MCP And Dynamic SDK Control

The headless loop maintains several mutable MCP sets:

- startup MCP clients from config.
- SDK MCP clients declared by the SDK initialize request.
- dynamically managed MCP servers from `mcp_set_servers` control messages.
- plugin-driven MCP changes after plugin state refresh.

The print loop caches SDK MCP clients, registers elicitation handlers for
non-SDK MCP servers, supports MCP status/control requests, and keeps
`appState.mcp.tools` synchronized so subagents can see SDK MCP tools. It also
routes SDK MCP elicitation through `SdkControlClientTransport` instead of the
normal in-terminal dialog.

### Stream Input Queue

With stream-json input, stdin messages are deduplicated by UUID against both
transcript history and runtime receipt. Historical duplicates can be acked when
`--replay-user-messages` is enabled and close the command lifecycle without
re-running the prompt. New messages are enqueued with:

- `mode: prompt`.
- caller-supplied priority.
- resolved/prepended file attachments when present.
- message UUID and timestamp.

This makes print mode suitable for a long-lived SDK worker, not just a single
prompt invocation.

### Background Agents In Print Mode

The headless loop deliberately holds back the final `result` while background
local agents or workflows are still running. It drains SDK events before command
queue processing so `task_started` and `task_progress` appear before
`task_notification`. Teammates are excluded from the "wait before exit" path
because they are long-lived by design.

This affects automation harnesses: a print run can remain alive after the main
assistant text is available if background work must report completion first.

### Prompt Suggestions In Print/SDK

Prompt suggestions are not TUI-only in 2.1.141. If `options.promptSuggestions`
is set and `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` is not explicitly false,
print mode can generate a `prompt_suggestion` message after a turn. It uses
`getLastCacheSafeParams()` so the suggestion request can reuse a stable prompt
prefix. If a final result is held for background agents, the suggestion is
deferred so it appears after the result.

### Rewind And Read-State Cache

`--rewind-files` restores file state to a target user message and exits. The
normal print loop also seeds `readFileState` from transcript content and merges
pending seeds into the cache before `ask()`. This is a correctness feature for
headless edit sessions: file-state knowledge survives across multiple queued
turns in one print process.

### Telemetry Surface

The print path can contribute telemetry from:

- root `tengu_init` fields such as `print`, input/output formats, and mode flags.
- `tengu_code_prompt_ignored` for the special `code` prompt case.
- `tengu_single_word_prompt` for single-token prompts.
- `tengu_structured_output_enabled` and `tengu_structured_output_failure`.
- headless profiler checkpoints and query profile reports.
- permission prompt attribution count.
- SDK/no-params prompt suggestion suppressions.
- background task counts in worker status.
- stream/protocol lifecycle diagnostics in CCR transport.

The docs should continue to separate telemetry metadata from streamed SDK
messages. Some telemetry is local analytics; some stream messages are protocol
events visible to the SDK host.
