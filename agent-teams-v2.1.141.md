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

## Detailed Architecture

Agent Teams in 2.1.141 are implemented as a lead-centric swarm. The lead session
owns the team, creates the durable team file, owns the team task list, and
coordinates teammate lifecycle. Teammates are not independent project roots:
they are members of a flat team roster recorded in a single config file.

Core layers:

- Tool layer: `TeamCreate`, `TeamDelete`, `SendMessage`, and the team-aware
  path inside `Agent`.
- Storage layer: team config files under the Claude config directory and task
  lists under the task directory.
- Runtime layer: in-process teammate tasks or pane/process backends.
- Messaging layer: mailbox files plus in-process pending-message injection.
- UI layer: task panel, teammate transcript selection, colors, and Agent View.
- Hook layer: teammate idle, task completed, subagent, and tool hooks.

## Enablement Details

`isAgentSwarmsEnabled()` is the main availability predicate.

Observed rules:

- Anthropic internal user type enables the feature.
- External use requires local opt-in (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` or
  hidden CLI activation) plus runtime gate `tengu_amber_flint`.
- `TeamCreate` and `TeamDelete` are not registered when the feature is disabled.
- `Agent` rejects team spawning if `team_name` is supplied while swarms are not
  enabled.

This produces two separate failure modes:

- The model may not see `TeamCreate`/`TeamDelete` at all.
- A direct `Agent` team spawn attempt can still error with "Agent Teams is not
  yet available on your plan."

## Team Creation Flow

`TeamCreate.call()` performs these steps:

1. Reads current app state and rejects creation if the lead already has a team.
2. Checks whether the requested team name already exists.
3. Uses the requested name if available, otherwise generates a unique word slug.
4. Creates a deterministic lead agent id from `team-lead` and team name.
5. Resolves the lead model from session/main/default model state.
6. Writes the team file.
7. Registers the team for session cleanup.
8. Resets and creates the matching task-list directory.
9. Sets the leader team name so `getTaskListId()` resolves to the team.
10. Adds team context to AppState.
11. Logs `tengu_team_created`.

The lead deliberately does not set `CLAUDE_CODE_AGENT_ID`. The source comment
explains the reason: setting it would cause `isTeammate()` to treat the lead as
a teammate and break inbox polling.

## Team File Schema

`TeamFile` in `source/src/utils/swarm/teamHelpers.ts`:

- `name`: team name.
- `description`: optional description.
- `createdAt`: timestamp.
- `leadAgentId`: deterministic lead id.
- `leadSessionId`: actual leader session UUID for discovery.
- `hiddenPaneIds`: optional list of hidden teammate panes.
- `teamAllowedPaths`: optional team-wide edit permission paths.
- `members`: flat member array.

Member fields:

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

`isActive === false` means idle/inactive. Undefined or true is treated as
active by cleanup checks.

## Path Functions

Team paths are sanitized:

- non-alphanumeric characters become `-`.
- names are lowercased.
- `@` is replaced for agent-name use to avoid ambiguity in `name@team`.

Important functions:

- `sanitizeName(name)`
- `sanitizeAgentName(name)`
- `getTeamDir(teamName)`
- `getTeamFilePath(teamName)`
- `readTeamFile(teamName)`
- `readTeamFileAsync(teamName)`
- `writeTeamFileAsync(teamName, file)`
- `cleanupTeamDirectories(teamName)`

## Teammate Spawn Flow

The `Agent` tool switches into team-spawn mode when:

- a team name is supplied directly or resolved from app state.
- `name` is supplied.

Spawn flow:

1. Resolve team name from input/context.
2. Reject nested teammate spawn if caller is already a teammate.
3. Reject in-process teammate background spawn.
4. Pull agent definition color/model where available.
5. Call `spawnTeammate(...)`.
6. Return status `teammate_spawned` with teammate data.

The returned output is intentionally an internal extension of the normal Agent
tool output shape; the exported public Agent output schema does not fully expose
every teammate-spawn-only field.

## In-Process Teammate Runtime

`spawnInProcessTeammate()` creates:

- deterministic `agentId`.
- task id with `in_process_teammate` prefix.
- independent abort controller.
- parent session id.
- teammate identity.
- teammate AsyncLocalStorage context.
- Perfetto agent registration when tracing is enabled.
- `InProcessTeammateTaskState` in AppState.
- cleanup handler for graceful shutdown.

Permission mode resolution:

- plan-mode-required teammates start in `plan`.
- if parent mode is `plan` or `dontAsk`, teammate mode becomes `default`.
- otherwise teammate inherits the parent mode.

This prevents teammates from silently inheriting parent plan/dontAsk states that
would be inappropriate for independent work.

## Task System Integration

Teams use the todo-v2 task system. The team name becomes the task-list id for
the lead and teammates.

Task workflow:

1. `TeamCreate` resets the team task list.
2. The lead creates tasks.
3. The lead spawns named teammates.
4. Teammates claim or receive tasks using task tools.
5. Teammates update status as `in_progress`, `completed`, `deleted`, etc.
6. `TaskCompleted` hooks can run when completion is recorded.
7. The lead deletes the team after all teammates are inactive.

Important task update capabilities:

- change status.
- change subject/description.
- assign `owner`.
- merge metadata.
- delete metadata keys by setting null.
- add dependency edges with `addBlocks`.
- add prerequisite edges with `addBlockedBy`.

## Messaging Protocol

`SendMessage` accepts either plain text or structured messages.

Plain text:

- requires/benefits from a short `summary`.
- writes a mailbox entry containing sender, text, summary, timestamp, and color.
- returns routing metadata for UI display.

Structured messages:

- `shutdown_request`
- `shutdown_response`
- `plan_approval_response`

Shutdown request:

1. Lead or teammate sends `shutdown_request`.
2. A request id is generated with a `shutdown` prefix.
3. The request is written to target mailbox.
4. Target responds with `shutdown_response`.
5. Approval can abort an in-process teammate controller or signal the pane
   backend.
6. Lead removes or deactivates the teammate record.

Plan approval response:

- carries request id.
- carries approve/deny boolean.
- can include feedback.
- is used when a teammate requires approval before leaving plan mode.

## TeamDelete Cleanup Semantics

`TeamDelete` is conservative:

- It reads the team file.
- It ignores the team lead when checking active members.
- It considers non-lead members active unless `isActive === false`.
- It refuses cleanup while active members remain.
- It tells the model to use shutdown before deletion.

On success it:

- removes team directories.
- unregisters session cleanup.
- clears teammate colors.
- clears leader team name.
- logs `tengu_team_deleted`.
- clears `teamContext`.
- clears queued inbox messages.

## UI and Agent View Relationship

Agent Teams and Agent View overlap but are not identical.

Teams provide:

- roster.
- teammate task state.
- message routing.
- team task lists.
- teammate identity.

Agent View provides:

- transcript viewing.
- background session/task visibility.
- process/job controls.
- fleet/session UI.

The connector between them is task state and transcript selection. Team docs
should be read with `agent-view-2.1.141.md`.

## Telemetry

Core team events:

- `tengu_team_created`
- `tengu_team_deleted`
- `tengu_transcript_view_enter`
- `tengu_transcript_view_exit`

Related events:

- `tengu_tool_use_success`
- `tengu_tool_use_error`
- `tengu_tool_use_can_use_tool_allowed`
- `tengu_tool_use_can_use_tool_rejected`
- `tengu_agent_tool_terminated`
- `tengu_agent_memory_loaded`
- `tengu_task_*` style events where task tools emit them.

## Failure and Edge Cases

- Creating a second team from the same lead is rejected.
- Reusing an existing team name silently creates a new unique slug.
- TeamDelete can return success with "nothing to clean up" if no team context is
  active.
- Teammates cannot spawn teammates because the team roster is flat.
- In-process teammates cannot spawn background agents.
- Durable cron tasks are rejected for teammates because teammates do not persist
  across sessions.
- Team file read failures generally fail closed or log debug information.

## Source-Level Object Model

Agent teams in 2.1.141 are not a single feature flag around `Agent`. They are a
separate object model layered on top of the task system, agent definitions,
memory directories, and the in-process teammate runner.

The key source concepts are:

- team definition files: durable team metadata.
- team identities: names and agent identities used for display and routing.
- teammate task state: live in-process execution state.
- teammate messages: queued user/leader messages routed to a specific running
  teammate.
- task-list IDs: shared work queue state that teammates can create/update.
- memory path helpers: optional team/shared memory locations.
- Agent View helper state: transcript switching and agent-owned view state.

The user-visible "team" is therefore a combination of durable definition,
running task state, message routing, and optional memory context.

## Team File Lifecycle

The team file lifecycle reconstructed from the 2.1.141 source is:

1. The user invokes `TeamCreate`.
2. Input is validated against the team schema.
3. Agent/member definitions are normalized.
4. Paths are derived from the configured Claude home/project context.
5. Existing definitions are checked to avoid accidental overwrite or invalid
   duplication.
6. The team file is written.
7. Team list/discovery state becomes able to surface the new team.
8. Team execution can spawn in-process teammates based on that file.

Deletion is the reverse only for durable definition files. Running teammates are
task state and must be stopped or allowed to finish separately depending on the
calling path. A correct implementation cannot just unlink a file and assume the
runtime is gone.

## Team Definition Fields

The definition schema supports the fields needed to recreate teammate execution:

- team name or identifier.
- human-facing description.
- member list.
- per-member agent name.
- per-member prompt/instructions.
- per-member model selection.
- `inherit` model handling.
- tool allow/disallow configuration through agent/tool policy.
- background/execution hints.
- optional memory/team context.
- optional task-list integration.

Different source files use different narrowed views of this data. The writer
validates the whole definition, while the spawn/runtime path extracts only the
fields needed to launch each teammate.

## In-Process Teammate Runtime

The in-process teammate runner is intentionally different from ordinary
background `Agent` tasks:

- it runs inside the same process and app-state store.
- it has a teammate identity and a task ID.
- it can receive leader messages after launch.
- it can update shared task lists.
- it can send messages through `SendMessage`.
- it can be displayed in Agent View-like transcript views.
- it can be shut down through task stop semantics.

This makes teams closer to a coordinated multi-agent runtime than a convenience
wrapper around multiple one-shot subagents.

## Message Routing

Message routing has to preserve which teammate owns which message. The routing
model is:

- leader/user submits a team or teammate instruction.
- runtime resolves the target teammate identity.
- message is appended to that teammate's pending message queue.
- in-process runner consumes queued messages.
- output and task notifications are associated with the teammate task.
- UI helpers decide whether the user is viewing the leader or a teammate
  transcript.

Cron-triggered teammate work uses the same principle. Scheduled tasks created by
teammates are tagged so that fired prompts route back to the teammate's pending
user messages rather than becoming unrelated leader prompts.

## Task Tool Relationship

The task-list tools are team-critical:

- `TaskCreate` creates shared work items.
- `TaskGet` fetches one work item.
- `TaskList` enumerates work items.
- `TaskUpdate` mutates status/content.
- `SendMessage` communicates between leader and teammate.

These tools are not generally allowed to every async agent. They are explicitly
allowed for in-process teammates in the 2.1.141 tool policy, which prevents
ordinary agents from accidentally getting the full team coordination surface.

## Memory Relationship

Team memory is separate from ordinary project memory:

- memory paths can be derived for team-specific shared context.
- team memory can be disabled independently.
- auto-memory/dream code has explicit warnings around team memory promotion.
- teammates may receive memory context through agent/team prompt construction.

The important behavioral rule is that personal memory should not be promoted
into team memory automatically. The dream prompt in 2.1.141 explicitly treats
team memory as a deliberate user choice.

## Agent View Relationship

Agent View and teams meet at task/transcript state:

- each teammate is represented as a task.
- helper state tracks which task transcript is being viewed.
- entering/exiting transcript view emits telemetry.
- prompt input suppresses some leader-context suggestions while viewing an
  agent task.
- output messages and task notifications preserve the target task ID.

This means an Agent View audit must include team tasks. A team member's output
is not just an ordinary assistant message in the leader transcript.

## Permission and Tool Safety

Team execution still respects the same safety layers as other execution paths:

- mode-level tool filtering.
- agent disallowed-tool sets.
- teammate-specific allowlist.
- permission context and deny rules.
- shell classifier for shell tools.
- managed settings and policy.
- workspace trust and simple/bare reductions.

The team feature adds coordination, not an exemption from tool safety.

## Future Diff Checklist

When extracting a later release, inspect these areas for team changes:

1. `TeamCreateTool` and `TeamDeleteTool` schemas.
2. team path helpers and storage layout.
3. `spawnMultiAgent` and teammate model resolution.
4. `InProcessTeammateTask` lifecycle.
5. `SendMessageTool` input schema and routing.
6. task-list tool access in `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`.
7. cron routing for teammate-created jobs.
8. Agent View helpers and transcript switching.
9. memory directory team path behavior.
10. telemetry events around team create/delete/spawn/message/failure.
