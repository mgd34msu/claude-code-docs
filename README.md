# Claude Code 2.1.141 Source Notes

This directory contains source-derived notes for Claude Code 2.1.141. The
writeups are based on the reconstructed 2.1.141 source under `source/src` in
this checkout. Older docs from `~/Projects/claude-code-docs` were used as a
topic checklist only; the behavior below is tied to 2.1.141.

## New 2.1.141 Companion Docs

- `agent-teams-v2.1.141.md` - team/swarm tools, file layout, teammate spawning, task lists, and messaging.
- `auto-mode-v2.1.141.md` - auto permission mode, classifier gating, dangerous rule stripping, and bypass interactions.
- `brief-mode-v2.1.141.md` - `SendUserMessage` / legacy `Brief`, entitlement, activation, attachments, rendering, and telemetry.
- `cached-tokens-v2.1.141.md` - usage object fields, prompt-cache counters, server tool counters, cost tracking, and stats aggregation.
- `channels-v2.1.141.md` - MCP channel notifications, CLI flags, allowlist/policy gates, permissions relay, and telemetry.
- `claude-hooks-reference-v2.1.141.md` - full source-backed hooks reference.
- `claude-md-best-practices-v2.1.141.md` - actual 2.1.141 CLAUDE.md loading, memory hierarchy, imports, and subagent behavior.
- `context-injection-v2.1.141.md` - every observed context injection path: CLAUDE.md, agents, hooks, MCP, output styles, skills, channels, teams.
- `dream-and-speculation-v2.1.141.md` - auto dream, `/dream`, prompt suggestions, speculative execution, and deferred prefetch.
- `environment-variables-v2.1.141.md` - environment variable reference organized by subsystem.
- `inject-after-agent-finish-v2.1.141.md` - `PostToolUse` context injection after `Agent` / legacy `Task`.
- `loop-command-v2.1.141.md` - the 2.1.141 state of `/loop`, Cron tools, durable scheduled tasks, scheduler behavior, and gates.
- `native-tools-v2.1.141.md` - built-in tool registry, availability gates, aliases, special tools, MCP integration, and permission filtering.
- `statsig-gates-v2.1.141.md` - build-time feature flags and runtime GrowthBook/Statsig gates observed in 2.1.141.
- `telemetry-v2.1.141.md` - analytics architecture, privacy controls, Datadog/1P OTel sinks, metrics, and event families.
- `tool-aliasing-v2.1.141.md` - primary tool names, aliases, legacy permission normalization, SDK compatibility names, and MCP shadowing.
- `undocumented-features-v2.1.141.md` - hidden/experimental product surfaces observed in 2.1.141.

## Existing 2.1.141 Detailed Docs

- `agent-view-2.1.141.md` - Agent View feature writeup.
- `claude-hooks-reference-v2.1.141.md` - full hooks reference.
- `claude-print-v2.1.141.md` - `-p` / `--print` and print-mode telemetry.
- `harness-detection-v2.1.141.md` - harness detection behavior.

## Source Basis

Most topic-specific behavior is concentrated in these areas:

- Tool registry and dispatch: `source/src/tools.ts`, `source/src/Tool.ts`, `source/src/services/tools`.
- CLI flags and startup: `source/src/main.tsx`, `source/src/setup.ts`, `source/src/cli`.
- Permissions: `source/src/utils/permissions`, `source/src/types/permissions.ts`, `source/src/hooks/useCanUseTool.tsx`.
- Hooks: `source/src/types/hooks.ts`, `source/src/services/tools/toolHooks.ts`, `source/src/query/stopHooks.ts`, `source/src/commands/hooks`.
- MCP/channels/plugins: `source/src/services/mcp`, `source/src/plugins`, `source/src/utils/plugins`.
- Teams/tasks: `source/src/tools/TeamCreateTool`, `source/src/tools/TeamDeleteTool`, `source/src/tools/SendMessageTool`, `source/src/utils/swarm`, `source/src/tasks`.
- Prompt suggestion/speculation/dream: `source/src/services/PromptSuggestion`, `source/src/services/autoDream`, `source/src/skills/bundled/dream.ts`.
- Telemetry: `source/src/services/analytics`, `source/src/utils/privacyLevel.ts`, `source/src/cost-tracker.ts`.

## Quality Bar for 2.1.141 Docs

The 2.1.141 writeups should be read as source reconstruction notes, not public
product docs. Each topic is expected to include:

- exact 2.1.141 source areas.
- feature gates and environment overrides.
- data schemas or file formats where applicable.
- lifecycle/flow descriptions.
- telemetry events.
- edge cases and failure modes.
- notes for future release diffing.

The long-form canonical references are:

- `claude-hooks-reference-v2.1.141.md`
- `agent-view-2.1.141.md`
- `claude-print-v2.1.141.md`
- `harness-detection-v2.1.141.md`

The companion topic references now expand the same source-backed style across
tools, telemetry, feature gates, environment variables, teams, channels, brief
mode, auto mode, cron, context injection, CLAUDE.md, token accounting, aliases,
and experimental features.

## 2.1.141 Documentation Standard

The 2.1.141 documents are intended to be a source-map baseline for future
Claude Code releases. The archive documents provide structure and minimum
coverage expectations only. The factual content in the 2.1.141 versions is
derived from the reconstructed 2.1.141 source tree.

For future updates:

- keep archive docs as quality baselines, not copy sources.
- cite current-version source paths in each topic.
- preserve exact flag, schema, event, env var, gate, and file-format names.
- distinguish user-facing, hidden, gated, internal, and protocol-only behavior.
- prefer documenting a conditional compiled surface over omitting it.
- do not collapse separate systems just because they share UI labels.
- keep migration/legacy names separate from canonical current names.
- update future diff checklists when new source anchors appear.
