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
