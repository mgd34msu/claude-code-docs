# Agent Teams in Claude Code 2.1.141

Agent Teams are the 2.1.141 multi-agent swarm system. They let a lead session
create a named team, spawn named teammates, assign work through task lists, and
exchange explicit messages. This is not just the legacy `Agent` subagent tool:
team mode adds persistent team metadata, teammate identities, team-scoped task
storage, inboxes, shutdown semantics, and a teammate transcript view.

## Enablement

The primary gate is `isAgentSwarmsEnabled()` in
`source/src/utils/agentSwarmsEnabled.ts`.

- Anthropic internal builds enable swarms by default.
- External users need `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` or the hidden
  `--agent-teams` path, plus the runtime gate `tengu_amber_flint`.
- Tool registration lives in `source/src/tools.ts`; `SendMessage` is registered
  outside the swarm gate, while `TeamCreate` and `TeamDelete` are included only
  when swarms are enabled.

## Tool Surface

### TeamCreate

`TeamCreate` lives in `source/src/tools/TeamCreateTool/TeamCreateTool.ts`.
It creates the team file, initializes the team task list, registers cleanup,
sets lead context, and logs `tengu_team_created`.

Input:

- `team_name`: requested team name.
- `description`: optional purpose text.
- `agent_type`: optional default teammate agent type.

Important behavior:

- The lead can have only one active team.
- If a requested team name already exists, the implementation generates a unique
  slug instead of mutating the existing team.
- The lead agent id is derived as a team lead identity.
- App state gets a `teamContext` with team name, config path, lead id, and a
  teammate map.
- A team task list is created or reset for the team.

### TeamDelete

`TeamDelete` lives in `source/src/tools/TeamDeleteTool/TeamDeleteTool.ts`.
It removes team state and deletes the team/task directories, but only after
teammates have been shut down.

Important behavior:

- It refuses deletion while active non-lead members remain.
- It cleans team config, task directories, teammate colors, cleanup handlers,
  leader team name, and app-state team context.
- It logs `tengu_team_deleted`.

### SendMessage

`SendMessage` lives in `source/src/tools/SendMessageTool/SendMessageTool.ts`.
It is the explicit inter-agent communication path. Plain text written outside
the tool is not automatically delivered to teammates.

Input:

- `to`: teammate name or target identifier.
- `summary`: optional concise summary.
- `message`: string or structured protocol message.

Structured message types observed in 2.1.141:

- `shutdown_request`
- `shutdown_response`
- `plan_approval_response`

Plain messages are written to the teammate mailbox with sender, text, summary,
timestamp, and color metadata.

### Agent Tool Team Spawn Path

The normal `Agent` tool has team-specific input fields in
`source/src/tools/AgentTool/AgentTool.tsx`:

- `team_name`
- `name`
- `mode`

When both `team_name` and `name` are present, the tool spawns a named teammate
instead of a one-shot subagent. Teammates cannot spawn teammates recursively.
In-process teammates are restricted to synchronous subagent behavior when nested
agent calls are allowed.

## Storage Layout

Team paths are built in `source/src/utils/swarm/teamHelpers.ts`.
The base config directory is `CLAUDE_CONFIG_DIR` if set, otherwise `~/.claude`.

Main locations:

- `~/.claude/teams/{team}/config.json`
- `~/.claude/tasks/{team}/...`
- teammate mailbox/inbox files under the team support paths.

The team config schema includes:

- `name`
- `description`
- `createdAt`
- `leadAgentId`
- `leadSessionId`
- `hiddenPaneIds`
- `teamAllowedPaths`
- `members`

Member records include:

- `agentId`
- `name`
- `agentType`
- `model`
- `prompt`
- `color`
- `planModeRequired`
- `joinedAt`
- `tmuxPaneId`
- `cwd`
- `worktreePath`
- `sessionId`
- `subscriptions`
- `backendType`
- `isActive`
- `mode`

## Task Lists

Teams are tied to the task-list subsystem. A team is conceptually paired with a
team task list, and teammate workflow is built around that list:

- Create a team with `TeamCreate`.
- Create or update tasks with the task tools when todo-v2 tasks are enabled.
- Spawn teammates with `Agent` using `team_name` and `name`.
- Assign or update work using `TaskUpdate`.
- Use `SendMessage` for status, shutdown, or plan approval messages.
- Delete with `TeamDelete` only after teammates are inactive.

The task tools come from `TaskCreate`, `TaskGet`, `TaskUpdate`, and `TaskList`
when `isTodoV2Enabled()` returns true.

## Teammate Execution

The in-process teammate path is in `source/src/utils/swarm/spawnInProcess.ts`.
It constructs teammate identity with:

- `agentId`
- `agentName`
- `teamName`
- `color`
- `planModeRequired`
- `parentSessionId`

Permission mode is derived from parent context:

- If the teammate requires plan mode, it starts in `plan`.
- If the parent is in `plan` or `dontAsk`, teammate execution falls back to
  `default`.
- Otherwise it may inherit the parent permission mode.

## UI State

Agent teams add state for:

- `expandedView` values such as `tasks` and `teammates`.
- viewed teammate transcript selection.
- teammate colors.
- team inbox state.
- active teammate metadata.

The Agent View writeup in `agent-view-2.1.141.md` covers the user-visible
transcript viewing side of this.

## Telemetry

Observed team-related events include:

- `tengu_team_created`
- `tengu_team_deleted`
- `tengu_transcript_view_enter`
- `tengu_transcript_view_exit`
- task and teammate tool events through the normal `tengu_tool_use_*` pipeline.

## 2.1.141 Source Index

- `source/src/utils/agentSwarmsEnabled.ts`
- `source/src/tools.ts`
- `source/src/tools/TeamCreateTool/TeamCreateTool.ts`
- `source/src/tools/TeamCreateTool/prompt.ts`
- `source/src/tools/TeamDeleteTool/TeamDeleteTool.ts`
- `source/src/tools/TeamDeleteTool/prompt.ts`
- `source/src/tools/SendMessageTool/SendMessageTool.ts`
- `source/src/tools/SendMessageTool/prompt.ts`
- `source/src/tools/AgentTool/AgentTool.tsx`
- `source/src/utils/swarm/teamHelpers.ts`
- `source/src/utils/swarm/spawnInProcess.ts`
- `source/src/state/teammateViewHelpers.ts`
