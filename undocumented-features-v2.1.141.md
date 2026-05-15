# Undocumented and Experimental Features in Claude Code 2.1.141

This document summarizes hidden, experimental, internal, or sparsely documented
2.1.141 surfaces observed in source. It is not a recommendation to enable them;
many are gated, internal-only, policy-controlled, or dead-code-eliminated by
build flags.

## Agent View

Agent View is a 2.1.141 feature. It adds teammate/subagent transcript viewing
state and telemetry:

- `tengu_transcript_view_enter`
- `tengu_transcript_view_exit`

Full writeup: `agent-view-2.1.141.md`.

## Agent Teams

Agent teams expose multi-agent swarms through:

- `TeamCreate`
- `TeamDelete`
- `SendMessage`
- `Agent` with `team_name` and `name`.

External enablement depends on `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` or hidden
CLI paths plus `tengu_amber_flint`. Full writeup:
`agent-teams-v2.1.141.md`.

## Cron and Scheduled Tasks

Cron tools are hidden behind build/runtime gates:

- build flag `AGENT_TRIGGERS`.
- runtime `tengu_kairos_cron`.
- durable gate `tengu_kairos_cron_durable`.

Tools:

- `CronCreate`
- `CronDelete`
- `CronList`

Durable tasks persist to `.claude/scheduled_tasks.json`.

## Kairos/Brief/Channels

Kairos-related hidden or gated surfaces include:

- `SendUserMessage` / legacy `Brief`.
- MCP channel notifications.
- channel permission relay.
- push notifications.
- `/dream` skill integration.

Important runtime gates:

- `tengu_kairos`
- `tengu_kairos_brief`
- `tengu_kairos_brief_config`
- `tengu_harbor`
- `tengu_harbor_ledger`
- `tengu_kairos_dream`

## Auto Mode

Auto mode is an internal permission mode that externalizes as `default` in many
public-facing paths. It is controlled by `tengu_auto_mode_config`, model support,
settings, and circuit-breaker state.

Hidden command area:

- `auto-mode defaults`
- `auto-mode config`
- `auto-mode critique`

## Prompt Suggestion and Speculation

Prompt suggestion is controlled by:

- `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION`
- `tengu_chomp_inflection`
- user setting `promptSuggestionEnabled`

Speculation is Anthropic-internal in 2.1.141 and runs forked agent work against
an overlay filesystem before a user accepts a predicted prompt.

## Auto Dream

Auto dream is controlled by:

- `tengu_onyx_plover`
- user setting `autoDreamEnabled`

It runs background memory consolidation after enough time and sessions have
changed, with a lock to prevent concurrent runs.

## Worktree Mode

Worktree tools are conditionally included:

- `EnterWorktree`
- `ExitWorktree`

Worktree events include:

- `tengu_worktree_created`
- `tengu_worktree_entered_existing`
- `tengu_worktree_kept`
- `tengu_worktree_removed`

## Tool Search

`ToolSearch` is included when optimistic tool-search enablement allows it. The
actual deferral decision happens later at request construction. Tool definitions
can provide `searchHint` to improve keyword retrieval.

Environment/gates:

- `ENABLE_TOOL_SEARCH`
- `tengu_tool_pear`

## LSP Tool

The `LSP` tool is present only when `ENABLE_LSP_TOOL` is truthy. It is not part
of the normal default external tool surface.

## PowerShell Tool

`PowerShell` is conditionally enabled. External users generally need
`CLAUDE_CODE_USE_POWERSHELL_TOOL=true`; platform and internal-user behavior can
change availability.

## Remote, Bridge, CCR, Teleport

2.1.141 contains substantial remote/harness infrastructure:

- bridge mode.
- CCR auto-connect/mirror.
- remote TUI/session creation.
- teleport resume.
- daemon/session ingress paths.

Representative gates/env:

- `BRIDGE_MODE`
- `CCR_AUTO_CONNECT`
- `CCR_MIRROR`
- `tengu_remote_backend`
- `tengu_ccr_bridge`
- `tengu_ccr_mirror`
- `CLAUDE_CODE_REMOTE`
- `CLAUDE_BRIDGE_BASE_URL`
- `SESSION_INGRESS_URL`

Harness detection is covered separately in `harness-detection-v2.1.141.md`.

## Memory Directory and Team Memory

Memory-directory behavior is gated by several runtime flags:

- `tengu_coral_fern`
- `tengu_passport_quail`
- `tengu_slate_thimble`
- `tengu_moth_copse`
- `tengu_session_memory`
- `tengu_herring_clock`

Team memory and coworker memory have additional environment variables such as
`CLAUDE_COWORK_MEMORY_PATH_OVERRIDE`, `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES`,
and `TEAM_MEMORY_SYNC_URL`.

## Hidden/Advanced Environment Switches

Examples:

- `CLAUDE_CODE_DISABLE_AGENT_VIEW`
- `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS`
- `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION`
- `CLAUDE_CODE_ENABLE_TASKS`
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`
- `CLAUDE_CODE_USE_POWERSHELL_TOOL`
- `CLAUDE_CODE_VERIFY_PLAN`
- `CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING`
- `CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES`
- `CLAUDE_CODE_EMIT_SESSION_STATE_EVENTS`
- `CLAUDE_CODE_SAVE_HOOK_ADDITIONAL_CONTEXT`
- `CLAUDE_AGENT_SDK_MCP_NO_PREFIX`

See `environment-variables-v2.1.141.md` for the broader list.

## 2.1.141 Source Index

- `source/src/main.tsx`
- `source/src/tools.ts`
- `source/src/services/PromptSuggestion`
- `source/src/services/autoDream`
- `source/src/services/mcp`
- `source/src/utils/swarm`
- `source/src/tools/ScheduleCronTool`
- `source/src/utils/agentSwarmsEnabled.ts`
- `source/src/services/analytics/growthbook.ts`

## Expanded 2.1.141 Feature Inventory

This section is organized by product surface rather than by gate name. It is
intended to capture the hidden behavior that matters when reconstructing or
diffing future releases.

## Hidden CLI Flags and Entrypoints

Observed hidden or specialized CLI paths include:

- `--brief`: activates brief/user-message mode when Kairos brief is available.
- `--channels`: opts specific MCP servers/plugins into channel notifications.
- `--dangerously-load-development-channels`: development-only channel bypass
  with confirmation.
- `--agent-teams`: external path into agent swarms when combined with gates.
- `--enable-auto-mode`: hidden/deprecated auto mode path.
- `--permission-mode auto`: internal permission mode path where enabled.
- `--workload`: billing/attribution workload tag.
- print/SDK structured input and output flags documented in
  `claude-print-v2.1.141.md`.
- background/agent-view subcommands documented in `agent-view-2.1.141.md`.

The existence of a hidden flag does not mean the feature is available. Most are
also gated by build flags, runtime gates, provider type, subscription state, or
managed policy.

## Background Sessions and Agent View

Agent View is one of the largest hidden 2.1.141 additions. Its behavior crosses:

- CLI process listing and control.
- daemon/session state.
- background job classification.
- transcript loading.
- concurrent session detection.
- task output retention.
- teammate transcript viewing.

The practical significance for future source-map work is that background
session code is not isolated to one folder. It appears in CLI handlers, fleet
components, job state, daemon paths, task code, and REPL UI state.

## Team/Swarm System

The team system adds:

- persistent team config files.
- team task-list directories.
- in-process teammate tasks.
- tmux/iTerm/backend abstractions.
- explicit teammate message routing.
- shutdown request/response messages.
- plan approval response messages.
- teammate transcript viewing.
- team memory hooks.

Hidden risk: some pieces are registered even when team creation is gated. For
example `SendMessage` can be in the base tool list, but it is only meaningful
when the session has an addressable team/agent context.

## Scheduled Work

Cron/scheduled tasks are more capable than an ordinary slash command:

- session-only tasks.
- durable `.claude/scheduled_tasks.json` tasks.
- recurring and one-shot cron expressions.
- missed one-shot catch-up behavior.
- file watcher and lock-owner behavior.
- teammate restrictions.
- deterministic jitter.
- max-age expiration.

The old `/loop` mental model is incomplete for 2.1.141. The active scheduling
surface is the Cron tool family plus bundled skills.

## Prompt Prediction and Speculative Execution

Prompt suggestion and speculation form a hidden predictive execution pipeline:

- prompt suggestion proposes a next prompt after assistant turns.
- speculation can fork execution against that prompt.
- speculative writes are redirected into a temp overlay.
- accepted speculation can copy overlay writes back and inject speculated
  messages into the visible transcript.
- incomplete speculation falls back to a normal query flow.

Important boundaries:

- disabled outside interactive main REPL.
- suppressed by plan mode and pending permissions.
- stops at unsafe Bash or denied tools.
- Anthropic-internal speculation enablement.

## Auto Dream and Memory Consolidation

Auto dream is a background memory consolidation feature:

- checks elapsed time and session-change thresholds.
- locks consolidation to avoid concurrent runs.
- runs a forked agent under `querySource: 'auto_dream'`.
- tracks touched files and task state through `DreamTask`.
- can be scheduled through the bundled dream skill and Cron tools.

Memory-related gates are spread across memdir, session memory, team memory, and
extract-memory services. Treat memory behavior as a cluster, not one feature.

## MCP Channels

Channels allow inbound external messages via MCP notifications:

- MCP server declares `experimental['claude/channel']`.
- session lists the server/plugin with `--channels`.
- org policy and plugin allowlists are checked.
- inbound content is wrapped in `<channel>` tags.
- optional structured permission relay avoids raw-text approval parsing.

Security implication: this is an intentional prompt-injection surface gated by
explicit opt-in and policy.

## Tool Search and Deferred Tools

Tool search exists to avoid sending every tool schema up front:

- tools may set `shouldDefer`.
- tools may provide `searchHint`.
- `ToolSearch` retrieves relevant deferred tools.
- the request path decides whether schema deferral is active.

This affects prompt cache behavior: stable sorting and deferred schemas are
partly about preserving cache compatibility while reducing prompt size.

## Memory Directory System

Observed memory-related controls:

- memdir loading.
- team memory directory.
- session memory compaction.
- auto memory disablement.
- memory extraction services.
- coworker memory path overrides.

Source areas:

- `source/src/memdir`
- `source/src/services/SessionMemory`
- `source/src/services/extractMemories`
- `source/src/services/teamMemorySync`
- `source/src/services/autoDream`

## Bridge, Remote, CCR, Teleport

2.1.141 contains several remote-control layers:

- bridge REPL/session runner.
- CCR bridge and mirror behavior.
- daemon session ingress.
- remote backend selection.
- teleport resume.
- background session fleet view.
- dangerous backend/server paths for controlled execution.

These are controlled by a mix of `feature(...)` flags, `tengu_*` runtime gates,
environment variables, and auth/provider constraints.

## Experimental Environment Controls Worth Tracking

Feature activation:

- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`
- `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION`
- `CLAUDE_CODE_ENABLE_TASKS`
- `ENABLE_TOOL_SEARCH`
- `ENABLE_LSP_TOOL`
- `CLAUDE_CODE_USE_POWERSHELL_TOOL`
- `CLAUDE_CODE_VERIFY_PLAN`
- `CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING`

Feature disablement:

- `CLAUDE_CODE_DISABLE_AGENT_VIEW`
- `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS`
- `CLAUDE_CODE_DISABLE_AUTO_MEMORY`
- `CLAUDE_CODE_DISABLE_CLAUDE_MDS`
- `CLAUDE_CODE_DISABLE_CRON`
- `CLAUDE_CODE_DISABLE_FAST_MODE`
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`
- `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS`

Remote/bridge:

- `CLAUDE_CODE_REMOTE`
- `CLAUDE_BRIDGE_BASE_URL`
- `CLAUDE_BRIDGE_OAUTH_TOKEN`
- `CLAUDE_CODE_CCR_MIRROR`
- `CLAUDE_CODE_USE_CCR_V2`
- `SESSION_INGRESS_URL`
- `CLAUDE_SESSION_INGRESS_TOKEN_FILE`

Debug/profiling:

- `CLAUDE_CODE_DEBUG_LOG_LEVEL`
- `CLAUDE_CODE_DEBUG_LOGS_DIR`
- `CLAUDE_CODE_PROFILE_STARTUP`
- `CLAUDE_CODE_PROFILE_QUERY`
- `CLAUDE_CODE_PERFETTO_TRACE`
- `CLAUDE_CODE_FRAME_TIMING_LOG`

## Feature Reconstruction Notes

When diffing later versions against 2.1.141, these are high-signal seams:

- New tools in `source/src/tools.ts`.
- New `feature('...')` strings.
- New `tengu_*` gate/dynamic config names.
- New environment variables in `process.env` or `Bun.env` references.
- New task types under `source/src/tasks`.
- New CLI options in `source/src/main.tsx` and CLI subcommand handlers.
- New hook schemas in `source/src/schemas/hooks.ts` and SDK schema files.
- New MCP notification names.
- New telemetry event families.
- New AppState fields that connect UI state to background execution.

## Hidden Command Surface

2.1.141 exposes many commands that are either hidden, experimental, internal, or
only meaningful in certain hosts. The command tree includes:

- background/task commands: `background`, `tasks`, `workflows`, `stop`.
- bridge/remote commands: `bridge`, `daemon`, `mobile`, `session`, `teleport`,
  `remote-env`, `remote-setup`.
- context/debug commands: `context`, `extra-usage`, `heapdump`, `doctor`,
  `status`.
- product/config commands: `config`, `permissions`, `privacy-settings`,
  `output-style`, `model`, `effort`, `scroll-speed`, `theme`, `color`.
- feature commands: `loops`, `memory`, `skills`, `agents`, `plugin`, `mcp`,
  `ide`, `voice`.
- internal/experimental commands: `btw`, `powerup`, `radio`, `passes`,
  `stickers`, `tui`, `chrome`, `desktop`.
- GitHub/app integrations: `install-github-app`, `install-slack-app`,
  `pr_comments`, `autofix-pr`.

The existence of a command file does not prove public availability. Each command
can have its own `isEnabled`, feature gate, host requirement, user type, or
hidden help behavior.

## Background Execution Surfaces

Undocumented background execution in 2.1.141 spans several task types:

- local shell tasks.
- local agent tasks.
- remote agent tasks.
- monitor MCP tasks.
- dream tasks.
- local workflow tasks.
- in-process teammate tasks.
- local main session tasks.

These task types share status concepts (`pending`, `running`, `completed`,
`failed`, `killed`) but differ in persistence, notification behavior, output
retrieval, and stop semantics.

The important doc point: background execution is not a single "Bash background"
feature. It is a general task subsystem with multiple producers and UI views.

## Agent View and Fleet View

Agent View, Fleet View, and transcript switching are source-visible in 2.1.141
but not fully documented in public CLI docs. They cover:

- selecting a running or completed agent/task transcript.
- grouping jobs.
- showing peer/session status.
- rendering PR/check/review state.
- tracking enter/exit telemetry.
- suppressing leader-context prompt suggestions while viewing an agent task.

This is a major 2.1.141 feature seam. Future extraction should treat it as an
independent subsystem, not merely a UI component.

## Remote and CCR Seams

Remote/CCR functionality includes:

- remote session creation.
- session ingress tokens.
- websocket auth file descriptors.
- remote memory directories.
- CCR mirror and v2 toggles.
- delivery acknowledgements.
- worker status updates.
- file download paths.
- hybrid/SSE transports.
- bridge/remote bridge core loops.

Some of these paths are host integration infrastructure rather than user-facing
CLI features, but they are part of the shipped 2.1.141 code and matter for a
complete reconstruction.

## Prompt Prediction Surfaces

Prompt prediction is split across:

- prompt suggestion generation.
- typeahead rendering.
- right-arrow acceptance.
- speculation overlay execution.
- speculation acceptance and transcript injection.
- pipelined speculation.
- SDK prompt-suggestion events.

The feature is easy to underdocument because the visible UI is small. Source
coverage shows it affects input handling, query execution, file overlays,
session storage, telemetry, and cache economics.

## Memory and Dream Surfaces

Memory-related undocumented behavior includes:

- auto-memory enable/disable logic.
- relevant-memory prefetch.
- memory directory path selection.
- team memory path detection.
- auto-dream background consolidation.
- manual `/dream` skill.
- consolidation locks.
- memory-saved messages.
- stale CLAUDE.md warning logic inside the dream prompt.

The dream system does not just summarize. It edits and organizes durable memory
files through a forked agent with constrained file tools.

## Hidden Tool Surfaces

Tools that are present but conditional or special:

- `Config`
- `CtxInspect`
- `LSP`
- `Monitor`
- `PowerShell`
- `PushNotification`
- `REPL`
- `RemoteTrigger`
- `Snip`
- `SyntheticOutput`
- `ToolSearch`
- `TeamCreate`
- `TeamDelete`
- `CronCreate`
- `CronDelete`
- `CronList`
- MCP resource tools

For each future release, verify whether the tool is compiled, registered,
enabled, visible to the model, callable by agents, callable by teammates, and
callable in print/SDK mode.

## Reconstruction Bias

For undocumented features, prefer false positives over false negatives when
mapping the source. A feature should be documented if it is:

- compiled into 2.1.141 source.
- reachable behind a gate.
- referenced by CLI flags.
- referenced by settings schema.
- referenced by AppState.
- referenced by SDK schemas.
- represented as a tool.
- represented as a task type.
- represented as a telemetry family.

Mark it as conditional or internal if necessary, but do not omit it just because
the gate is off by default.
