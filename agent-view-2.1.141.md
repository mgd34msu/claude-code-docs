# Agent View in Claude Code 2.1.141

This writeup is based on the local reconstructed 2.1.141 source in
`/home/buzzkill/Projects/lab/cc-linux-141/source`. It intentionally describes
what is present in the 141 tree, not behavior inferred from earlier versions.

## Executive Summary

Agent View in 2.1.141 is a background-session management feature. In the source
it is implemented mostly under the names `fleet`, `jobs`, `bg`, `daemon`, and
background session management. It provides a way to start, list, inspect,
message, attach to, stop, respawn, and remove long-running Claude Code sessions.

The main user-facing surface is `claude agents`, which opens an interactive TUI
when stdout is a terminal. In non-TTY mode, the same command prints a tabular
list of background agents. Sessions can be created with `claude --bg` or
`claude --background`, or from inside the REPL with `/background`.

There is also a related but separate in-REPL agent viewing system for local
`Agent` tool tasks. That part lets the user view or steer background subagents
from the main TUI. It shares concepts such as background tasks, progress
summaries, task retention, transcript viewing, and routing input to an agent,
but it is not the same as the external `claude agents` fleet view.

Important distinction:

- `claude agents` is Agent View for background Claude sessions.
- `/agents` is the custom-agent manager UI.
- `Agent` tool background tasks are local subagents shown inside the REPL task
  UI and coordinator panel.

## Source Map

Core files:

- `source/src/entrypoints/cli.tsx`
- `source/src/main.tsx`
- `source/src/cli/bg.ts`
- `source/src/cli/handlers/agents.ts`
- `source/src/fleet/FleetView.tsx`
- `source/src/fleet/jobModel.ts`
- `source/src/fleet/transcript.ts`
- `source/src/jobs/state.ts`
- `source/src/jobs/classifier.ts`
- `source/src/jobs/rendezvous.ts`
- `source/src/utils/concurrentSessions.ts`
- `source/src/utils/udsClient.ts`
- `source/src/daemon/main.ts`
- `source/src/daemon/status.ts`
- `source/src/commands/background/background.tsx`

Related local-agent view files:

- `source/src/tasks/LocalAgentTask/LocalAgentTask.tsx`
- `source/src/tools/AgentTool/AgentTool.tsx`
- `source/src/tools/AgentTool/resumeAgent.ts`
- `source/src/tools/SendMessageTool/SendMessageTool.ts`
- `source/src/components/CoordinatorAgentStatus.tsx`
- `source/src/components/tasks/BackgroundTasksDialog.tsx`
- `source/src/components/tasks/AsyncAgentDetailDialog.tsx`
- `source/src/state/teammateViewHelpers.ts`
- `source/src/screens/REPL.tsx`

Configuration and gating:

- `source/src/utils/config.ts`
- `source/src/utils/settings/types.ts`

## User-Facing Commands and Surfaces

### `claude agents`

Registered in `source/src/main.tsx`.

Behavior:

- If stdout is a TTY, imports `fleet/FleetView.js`, creates an Ink root, and
  mounts the interactive Agent View.
- If stdout is not a TTY, imports `cli/handlers/agents.js` and prints a TSV
  table.
- Supports `--cwd <path>` to filter background sessions by working directory.

Interactive `claude agents` TUI:

- Displays background sessions grouped by state or directory.
- Shows counts for awaiting input, working, and completed sessions.
- Supports filtering by typing.
- Supports `Enter` to open or attach.
- Supports `Space` for details.
- Supports `Ctrl+T` to pin/unpin.
- Supports `Ctrl+S` to switch grouping.
- Supports `Ctrl+X` to stop/delete.
- Supports `Esc` or `q` to quit.

Non-TTY output columns:

- `ID`
- `BAND`
- `ACTIVITY`
- `AGE`
- `AGENT`
- `NAME`
- `CWD`

The non-TTY handler is `source/src/cli/handlers/agents.ts`.

### `claude --bg` and `claude --background`

Fast-pathed from `source/src/entrypoints/cli.tsx` when `feature('BG_SESSIONS')`
is enabled. The implementation is in `source/src/cli/bg.ts`.

Behavior:

- Strips the background flag from the child args.
- Reads piped stdin up to 1 MiB and appends it to the prompt when provided.
- Creates a new session ID unless a session ID is supplied.
- Creates a job directory under `~/.claude/jobs/<short-id>`.
- Seeds `state.json`.
- Spawns a child Claude process.
- Sets environment variables so the child identifies as a background session.
- Prints a short hint block with follow-up commands.

The hint block includes:

- `claude agents`
- `claude attach <short-id>`
- `claude logs <short-id>`
- `claude stop <short-id>`

Important env passed to the background child:

- `CLAUDE_BG_BACKEND=daemon`
- `CLAUDE_BG_RENDEZVOUS_SOCK=<job>/rendezvous.sock`
- `CLAUDE_BG_SOURCE=<source>`
- `CLAUDE_CODE_ENTRYPOINT=bg`
- `CLAUDE_CODE_SESSION_KIND=bg`
- `CLAUDE_CODE_SESSION_LOG=<job>/output.log`
- `CLAUDE_CODE_SESSION_NAME=<seed name>`
- `CLAUDE_JOB_DIR=<job dir>`

### `/background`

Implemented in `source/src/commands/background/background.tsx`.

Behavior:

- Refuses to run when already inside a background session.
- Refuses to run when session persistence is disabled.
- Derives an intent/name from the explicit command argument or latest user
  message.
- Calls `spawnBgSession(['--resume', getSessionId()], getSessionId(), 'repl',
  getCwd(), seed)`.
- Prints the same background hints as `--bg`.
- Gracefully shuts down the foreground session.

This means `/background` is not a fresh task launcher. It backgrounds the
current conversation by resuming it in a detached process.

### `claude ps`

Fast-pathed through `source/src/entrypoints/cli.tsx` to `psHandler` in
`source/src/cli/bg.ts`.

Behavior:

- Reads live peer sessions.
- Reads job state from `~/.claude/jobs`.
- Marks stale jobs when no live peer exists.
- Prints a table with state/activity/age/name/origin.

`claude ps` and `claude agents` are related but not identical:

- `claude ps` is the older/plain status command.
- `claude agents` is the richer Agent View surface.

### `claude attach <id>`

Implemented in `source/src/cli/bg.ts`.

Behavior:

- Resolves a short ID/prefix.
- Reads saved job state.
- Refuses to attach to failed jobs.
- Starts a foreground Claude process with `--resume <session-id>` in the saved
  cwd.

The attach path is process-based. It does not embed Agent View inside the
existing process. It spawns a new foreground process and exits with that
process's exit code.

### `claude logs <id>`

Implemented in `source/src/cli/bg.ts`.

Behavior:

- Resolves job ID.
- Finds the live session log path when possible, otherwise uses the job log path.
- Prints the recent log tail.

### `claude stop <id>` and `claude kill <id>`

Implemented in `source/src/cli/bg.ts`.

Behavior:

- Resolves the job.
- Sends SIGTERM to live process IDs.
- Waits up to roughly 3 seconds for exit.
- Marks the job terminal as `stopped` when confirmed.
- Retains worktree information and tells the user how to remove it.

`killHandler` simply calls `stopHandler`; in 141 there is not a stronger
separate kill implementation in this path.

### `claude respawn <id>|--all`

Implemented in `source/src/cli/bg.ts`.

Behavior:

- Stops the selected job(s).
- Reads stored respawn flags and session ID.
- Relaunches with `spawnBgSession`.
- Preserves state fields such as intent, name, worktree, and backend.
- For failed/stopped jobs, may pass the saved intent as a new prompt.

### `claude rm <id>`

Implemented in `source/src/cli/bg.ts`.

Behavior:

- Resolves the job.
- Stops it first.
- Removes the job directory.
- Reports retained worktree path if present.

## Configuration and Gating

The user-facing setting is `disableAgentView`.

Setting schema text in `source/src/utils/settings/types.ts` says:

- Disable agent view.
- Covers `claude agents`, `--bg`, `/background`, and the on-demand daemon.
- Equivalent to `CLAUDE_CODE_DISABLE_AGENT_VIEW=1`.

Runtime helpers in `source/src/utils/config.ts`:

- `isAgentViewDisabled()`
- `isAgentsFleetEnabled()`
- `ensureFleetGateHydrated()`
- `isDaemonCliEnabled()`
- `fleetGateRejected()`

`isAgentViewDisabled()` checks:

- `CLAUDE_CODE_DISABLE_AGENT_VIEW`
- `getInitialSettings().disableAgentView === true`

Observed caveat in this reconstruction:

- The daemon path clearly uses `isDaemonCliEnabled()` and `fleetGateRejected()`.
- I do not see the same gate obviously enforced in every `claude agents`,
  `--bg`, or `/background` path in the reconstructed source.
- That may be an extraction gap, a feature flag/build difference, or a real
  2.1.141 wiring inconsistency.
- If validating behavior, this should be tested against the actual 141 binary
  with `CLAUDE_CODE_DISABLE_AGENT_VIEW=1`.

Build-time feature gates involved:

- `feature('BG_SESSIONS')`
- `feature('DAEMON')`
- `feature('TEMPLATES')`
- `feature('KAIROS')` for assistant-mode related behavior

Runtime/growthbook-ish flags in this area:

- `tengu_amber_anchor` controls daemon service installation.
- `tengu_copper_lantern` controls daemon service recall.
- `tengu_quiet_harbor` controls daemon cold-start default.

## Architecture

High-level flow:

```text
User
  |
  | claude --bg / --background
  v
cli/bg.ts
  |
  | create job dir and state.json
  | spawn child Claude process
  v
Background Claude process
  |
  | registers live session in ~/.claude/sessions
  | writes transcript/logs
  | updates job state through classifier
  v
~/.claude/jobs/<short-id>/state.json
~/.claude/sessions/<pid>.json
  |
  | read by claude agents
  v
FleetView TUI
```

Two persistent stores are important:

- `~/.claude/jobs`: durable background job state.
- `~/.claude/sessions`: live process registry.

The jobs store answers: "What is this background task, what is it doing, and
how can it be resumed?"

The sessions store answers: "Which Claude processes are alive and reachable
right now?"

## Job State Model

Defined in `source/src/fleet/jobModel.ts` as `FleetJobState`.

Important fields:

- `state`: user-visible lifecycle such as `working`, `blocked`, `done`,
  `failed`, `stopped`, `starting`, `resuming`.
- `detail`: concise status text.
- `tempo`: `active`, `idle`, or `blocked`.
- `needs`: exact user action needed when blocked.
- `block`: structured blocked state, when available.
- `suggestedReply`: optional reply suggestion.
- `output`: map of output labels to strings.
- `children`: child artifacts such as PRs or frames.
- `linkScanOffset`: transcript scan offset used to avoid rescanning.
- `template`: agent/template name, often `claude` or `bg`.
- `routine`: optional routine name.
- `respawnFlags`: flags to preserve for respawn.
- `intent`: original task/request.
- `initialPrompt`: template prompt when present.
- `name`: display name.
- `nameSource`: `auto` or `user`.
- `sessionId`: original session ID.
- `resumeSessionId`: session ID to resume.
- `daemonShort`: short ID.
- `bridgeSessionId` and `bridgeSessionSeq`: bridge integration state.
- `backend`: usually `daemon` in this feature path.
- `cwd`: working directory.
- `worktreePath`, `worktreeBranch`, `worktreeHookBased`: isolated worktree
  information.
- `originCwd`: original repository root when worktree is in use.
- `createdAt`, `updatedAt`, `firstTerminalAt`: timestamps.
- `pinned`: UI pin state, hydrated from `pins.json`.
- `pid`, `sock`: live process/socket data when merged from peer sessions.

The initial state is created by `createInitialJobState()` in
`source/src/jobs/state.ts`.

Default initial values:

- `state: 'working'`
- `detail: 'starting...'` unless a detail is supplied
- `tempo: 'active'` unless supplied
- `output: null`
- `children: null`
- `linkScanOffset: 0`
- `resumeSessionId: sessionId`
- `daemonShort: sessionId.slice(0, 8)`
- `backend: 'daemon'`

Idle background sessions can be seeded as blocked with:

- detail: `(idle - send a prompt to start)`
- needs: `send a prompt to start`

## Job Persistence

Job data lives under:

```text
~/.claude/jobs/<short-id>/
  state.json
  output.log
  rendezvous.sock
  timeline.jsonl
  order
  stateOrder
```

Observed files from code:

- `state.json` stores `FleetJobState`.
- `output.log` is the background process output log.
- `rendezvous.sock` is the rendezvous socket.
- `timeline.jsonl` records classifier timeline entries.
- `order` and `stateOrder` store sort ordering.
- `pins.json` lives under `~/.claude/jobs` and tracks pinned job IDs.

The state reader normalizes old/missing fields, including:

- `tempo`
- `output`
- `children`
- `linkScanOffset`
- `respawnFlags`
- `firstTerminalAt`
- `backend`

`listJobs()` reads every job directory, hydrates pinned state, and optionally
marks stale jobs based on a set of live short IDs.

Stale handling:

- Terminal jobs are left alone.
- Live jobs are left alone.
- Very new jobs are given a grace window.
- Non-live active jobs become `failed` with `tempo: 'idle'`, unless already
  blocked.

## Live Session Registry

Live sessions are tracked in `~/.claude/sessions/<pid>.json`.

Implemented by `source/src/utils/concurrentSessions.ts` and read by
`source/src/utils/udsClient.ts`.

Registered session fields include:

- `pid`
- `sessionId`
- `cwd`
- `startedAt`
- `procStart`
- `version`
- `peerProtocol`
- `kind`
- `entrypoint`
- `name`
- `logPath`
- `agent`
- `jobId`

Session kinds:

- `interactive`
- `bg`
- `daemon`
- `daemon-worker`

Live status values:

- `busy`
- `shell`
- `idle`
- `waiting`

`registerSession()` skips subagents/teammates by checking `getAgentId()`. This
is deliberate so the live-session roster represents real top-level sessions,
not every internal subagent.

`updateSessionActivity()` writes live status updates for the UI. The REPL uses
this to report whether it is busy, idle, or waiting for input/approval.

`listAllLiveSessions()`:

- Reads session files.
- Checks whether process IDs are alive.
- Verifies process start tokens when possible to avoid PID reuse mistakes.
- Removes stale files on cleanup-safe platforms.

`listLivePeerSessions()`:

- Reads session files that have sockets.
- Tests socket connectivity.
- Removes stale files when the process is no longer alive.

## Unix Domain Socket Messaging

Implemented in `source/src/utils/udsClient.ts`.

Agent View can send text into a live background session when a job has a socket.
The flow is:

- `FleetView` checks `action.job.state.sock`.
- If a filter/query string is present when opening, it sends that string to the
  socket before attaching.
- `sendToUdsSocket()` wraps the text in a cross-session message XML tag.
- The envelope is sent as a JSON line to the peer socket.

Envelope shape for user messages:

- `type: 'user'`
- `message.role: 'user'`
- `message.content: <cross-session-message>...`
- `priority: 'next'`

There is also `sendControlToUdsSocket()` for control messages with an `action`
field.

This is separate from the rendezvous socket used by `bg.ts` for lifecycle state
patches. UDS messaging is a peer-session communication mechanism.

## Rendezvous Socket

Implemented in `source/src/jobs/rendezvous.ts`.

The spawner passes `CLAUDE_BG_RENDEZVOUS_SOCK` to the child process. During
setup, the child calls `startRendezvousServer()` when
`CLAUDE_BG_BACKEND === 'daemon'`.

Rendezvous responsibilities:

- Create a socket server at the job rendezvous path.
- Accept one active socket.
- Send heartbeat frames.
- Accept control frames.
- Patch job state on startup settlement.
- Mark startup-dialog blockage if startup gets wedged.
- Handle shutdown requests.
- Handle repaint requests.
- Accept attacher capability data.
- Route reply text into the foreground message queue.

Recognized frame types include:

- `shutdown`
- `repaint`
- `attacher-caps`
- `reply`

Startup handling:

- Waits for Ink to be ready.
- Converts `starting`, `resuming`, `adopted`, or `crashed` states to
  `running` with `tempo: 'idle'`.
- Arms a startup wedge watchdog if the job remains at the starting detail.
- If startup stays wedged, marks the job blocked with:
  - detail: `stuck on a startup dialog`
  - needs: `open this session to continue setup`

The classifier uses `sendRv({ type: 'state', patch })` to push live state
patches.

## State Classification

Implemented in `source/src/jobs/classifier.ts`.

This is a major part of Agent View. It turns transcript tail text into compact
job state that the UI can display.

Classifier state object tracks:

- previous state
- time in previous state
- accumulated outputs
- last classify time
- captured intent
- in-flight classifier promise
- name-generation state
- latest ask
- whether the task was kicked/active
- last assistant message count
- permission bridge state
- last emitted detail
- last result
- optional `onClassified` callback

Classifier output:

- `state`: `working`, `blocked`, `done`, or `failed`
- `detail`: one-line status for display/notification
- `tempo`: `active`, `idle`, or `blocked`
- `needs`: exact user action when blocked
- `output`: result fields for completed work
- `source`: `heuristic`, `llm`, `midturn`, or `preclassify`

Important behavior:

- Mid-turn classification can force `working` and derive latest activity.
- End-of-turn classification reads assistant messages since the last classify.
- API/auth/infra failures are treated as `blocked` when user-fixable.
- Terminal transitions log analytics and set `firstTerminalAt`.
- Link scanning discovers child artifacts such as PR links.
- Worktree ownership can be updated from transcript records.
- State writes also append `timeline.jsonl`.
- State writes can patch live rendezvous state.

The classifier prompt is long and explicit. It tries to distinguish:

- done vs working
- done vs blocked
- blocked vs failed
- passive wait vs agent-owned wait
- optional follow-up offers vs true blockers

Agent View depends on this. Without it, the TUI would only know process liveness
and raw transcript/log data.

## Fleet View UI

Implemented in `source/src/fleet/FleetView.tsx`.

Data inputs:

- `listLivePeerSessions()`
- `listJobs(liveShortIds)`
- `mergeRosterOrphans()`
- cached peer statuses
- cached PR statuses

Rows:

- headers
- job rows
- folded "done" rows when there are too many completed items

Group modes:

- `state`
- `directory`

State groups:

- `input required`
- `Review`
- `Working`
- `Completed`

Directory grouping uses `spawnOrigin()` and `repoGroupLabel()`.

Filtering searches across:

- job ID
- template
- intent
- name
- detail
- needs
- cwd
- worktree path
- output values

Job row display:

- focus pointer
- icon/glyph
- truncated name
- state text
- age
- cwd/repo group

Detail pane:

- label/name
- detail/needs
- state and tempo
- backend
- session ID
- cwd or worktree path
- peer status
- restartable warning

Actions:

- `Enter`: open selected job or toggle header/fold.
- `Space`: show detail pane.
- `Ctrl+S`: switch grouping mode.
- `Ctrl+T`: pin/unpin.
- `Ctrl+X`: stop/delete.
- `Backspace`: edit filter.
- `Esc` or `q`: quit.

When opening a job:

- If the job has a socket and there is a query string, `FleetView` sends the
  query to that socket.
- Then it unmounts and calls `attachHandler(job.id)`.

## Job Model Display Logic

Implemented in `source/src/fleet/jobModel.ts`.

Core derivations:

- `deriveActivity()`: maps job state/time into `flowing`, `slowing`, `stuck`,
  `success`, `failure`, or `stopped`.
- `deriveBand()`: maps state/peer status into `active`, `blocked`, or
  `completed`.
- `stateBucket()`: maps to `blocked`, `review`, `working`, or `done`.
- `needsRespawn()`: true for stopped/failed terminal jobs that can be restarted.
- `jobLabel()`: creates a short display label from name or intent.
- `spawnOrigin()`: resolves original cwd when a worktree is involved.
- `repoGroup()`: derives directory group label.
- `pickIcon()` and `glyphColor()`: visual glyph/color state.

Special cases:

- Self-driving jobs, loops, routines, and cron-like sessions can be treated as
  still active even after success-like states.
- PR child status can move a job into review.
- Peer status `busy` overrides some persisted state and shows active.
- Peer status `waiting` can make a job blocked.

## Transcript Summaries

Implemented in `source/src/fleet/transcript.ts`.

Responsibilities:

- Resolve transcript path for a job.
- Read the tail of a transcript file.
- Estimate next scheduled task fire from transcript timestamps.
- Summarize recent transcript events.
- Summarize assistant text/tool-use/user/error events.
- Parse dispatch URLs.

This module is used to turn raw transcript tail into short activity text when
the richer job state is not enough.

## Daemon Integration

Implemented in `source/src/daemon/main.ts` and `source/src/daemon/status.ts`.

The daemon path is fast-pathed from `entrypoints/cli.tsx`.

Daemon commands include:

- `run`
- `install`
- `uninstall`
- `start`
- `stop`
- `restart`
- `status`
- `logs`
- `list`
- `scheduled`
- `assistant`
- `remote-control`
- `hub`

In 2.1.141, service installation is controlled by `isDaemonServiceInstallEnabled()`.
When service install is disabled, help text explains that the daemon runs on
demand and exits when the last client disconnects.

`daemon status` reports:

- supervisor pid/version/uptime
- socket/control status
- worker counts
- roster age
- log path and size
- configured workers
- lease clients

Observed caveat:

- `isDaemonWorkerRegistryEnabled()` returns `false` in this reconstructed
  source, so worker-registry subcommands are gated off.

## Local Agent View and Background Tasks

This is adjacent to Agent View and was also expanded in 141. It is important
because users may experience it as "agent view" inside the REPL.

### Agent tool schema

Implemented in `source/src/tools/AgentTool/AgentTool.tsx`.

Important input fields:

- `description`
- `prompt`
- `subagent_type`
- `model`
- `run_in_background`
- `name`
- `team_name`
- `mode`
- `isolation: 'worktree'`
- `cwd`

The `name` field makes a spawned agent addressable with `SendMessage({to:
name})` while running.

The output schema can return:

- `status: 'completed'`
- `status: 'async_launched'`

Async output includes:

- `agentId`
- `description`
- `prompt`
- `outputFile`
- `canReadOutputFile`

### When Agent tool tasks go async

The Agent tool runs async when any of these are true, unless background tasks
are disabled:

- `run_in_background === true`
- selected agent definition has `background === true`
- coordinator mode is active
- fork-subagent mode is active
- assistant/KAIROS mode is active
- proactive mode is active

When async:

- Registers a `LocalAgentTask`.
- Optionally registers name to agent ID in `agentNameRegistry`.
- Runs detached under `runWithAgentContext`.
- Tracks progress and summaries.
- Emits task notification on completion/failure/kill.

When sync:

- Registers a foreground local-agent task anyway, so it can later be
  backgrounded.
- Shows a background hint after the progress threshold.
- Races agent iteration against a background signal.
- If backgrounded, tears down the foreground iterator and resumes work in a
  detached async loop.

### LocalAgentTask state

Implemented in `source/src/tasks/LocalAgentTask/LocalAgentTask.tsx`.

Important fields:

- `agentId`
- `prompt`
- `selectedAgent`
- `agentType`
- `abortController`
- `error`
- `result`
- `progress`
- `retrieved`
- `messages`
- `lastReportedToolCount`
- `lastReportedTokenCount`
- `isBackgrounded`
- `pendingMessages`
- `retain`
- `diskLoaded`
- `evictAfter`

Progress tracks:

- tool-use count
- token count
- last activity
- recent activities
- summary

Recent activities are capped at 5.

`retain` is especially important. It means the UI is holding a local agent for
viewing, blocks eviction, enables stream append, and triggers transcript
bootstrap from disk.

### CoordinatorTaskPanel

Implemented in `source/src/components/CoordinatorAgentStatus.tsx`.

Purpose:

- Renders a steerable list of local background agents below the prompt.
- Shows `main` plus local agent rows.
- Lets the user enter a local agent view.
- Shows elapsed time, tokens, queued messages, summaries, and stop/clear hints.
- Evicts terminal rows after their deadline.

This is the closest in-REPL equivalent to an "agent view" panel.

### BackgroundTasksDialog

Implemented in `source/src/components/tasks/BackgroundTasksDialog.tsx`.

It is a unified task browser. It groups:

- teammates
- shells
- monitors
- remote agents
- local agents
- workflows
- dream tasks

It supports:

- selecting tasks
- viewing details
- stopping running tasks
- foregrounding teammates
- opening agent details

Local agent details use `AsyncAgentDetailDialog`.

### AsyncAgentDetailDialog

Implemented in `source/src/components/tasks/AsyncAgentDetailDialog.tsx`.

Shows:

- agent type and description
- status
- elapsed time
- tokens
- tool count
- recent progress/activity
- prompt or plan
- error

Controls:

- Esc/confirm to close.
- Left arrow to go back.
- `x` to stop a running agent.

### Viewing and Steering Agents

Implemented in `source/src/state/teammateViewHelpers.ts`,
`source/src/state/selectors.ts`, and `source/src/screens/REPL.tsx`.

`enterTeammateView(taskId)`:

- logs `tengu_transcript_view_enter`
- sets `viewingAgentTaskId`
- sets `viewSelectionMode: 'viewing-agent'`
- for local agents, sets `retain: true` and clears `evictAfter`
- releases a previously retained local agent if switching

`exitTeammateView()`:

- logs `tengu_transcript_view_exit`
- clears viewing state
- for retained local agents, clears messages/disk state and restores eviction

`stopOrDismissAgent(taskId)`:

- aborts running local agents
- immediately dismisses terminal local agents
- exits view if the dismissed agent was being viewed

When a user submits input while viewing a local agent:

- A user message is appended to the local agent display.
- If the task is running, the input is queued in `pendingMessages`.
- If the task has stopped, `resumeAgentBackground()` resumes it from transcript.

`SendMessage` can also target local agents:

- It resolves by registered name or raw agent ID.
- If running, it queues a pending message.
- If stopped, it auto-resumes from transcript.
- If evicted from AppState, it tries to resume from disk transcript.

## Backgrounding Foreground Tasks

The shared background shortcut is `task:background`, normally `Ctrl+B`.

Relevant files:

- `source/src/components/SessionBackgroundHint.tsx`
- `source/src/tools/BashTool/UI.tsx`
- `source/src/tasks/LocalShellTask/LocalShellTask.tsx`
- `source/src/tasks/LocalAgentTask/LocalAgentTask.tsx`

Foreground shell commands and foreground local agents can be backgrounded.
For local agents, `backgroundAgentTask()` marks `isBackgrounded: true` and
resolves a background signal so the Agent tool loop can detach and continue in
the background.

The session-level background hint exists, but in this reconstructed source the
session-background env check appears hardcoded as `isEnvTruthy('false')`,
meaning that particular UI path looks disabled. `/background` still exists as
the explicit slash command.

## Storage and Runtime Directories

Agent View uses at least these local stores:

```text
~/.claude/jobs/
~/.claude/jobs/<short-id>/state.json
~/.claude/jobs/<short-id>/output.log
~/.claude/jobs/<short-id>/rendezvous.sock
~/.claude/jobs/<short-id>/timeline.jsonl
~/.claude/jobs/pins.json
~/.claude/sessions/<pid>.json
```

Local-agent transcript storage is separate and uses session storage paths under
project-specific Claude directories. Subagent transcripts live under a
`subagents` directory for the parent session.

## Analytics Names Observed

Feature area analytics/event names seen in the source include:

- `tengu_transcript_view_enter`
- `tengu_transcript_view_exit`
- `tengu_background_already_bg`
- `tengu_bg_agent_terminal`
- `tengu_agent_tool_terminated`
- `tengu_agent_memory_loaded`

These names indicate that Agent View behavior is instrumented around:

- entering/exiting agent transcript views
- backgrounding attempts
- terminal background-agent state
- agent termination
- agent memory loading

## Important Edge Cases

### Job ID matching

Background job IDs are short 8-character hex prefixes derived from session IDs.
The code uses a strict regex for job IDs.

### Stale sessions

The sessions registry guards against PID reuse with process start tokens where
possible. On WSL, cleanup is conservative to avoid deleting live Windows-native
session files that WSL cannot probe.

### Failed vs stopped jobs

Failed/stopped terminal jobs can be restartable through `respawn`. Agent View
shows restartable text for jobs that need respawn.

### Startup wedges

If a background session starts but gets stuck before the TUI is ready, the
rendezvous watchdog marks it blocked and tells the user to open the session to
continue setup.

### Worktrees

Background jobs can preserve worktree state. Stop/remove messages mention
retained worktrees. Agent-tool worktree isolation separately creates
`agent-<id>` worktrees and retains them when changes exist.

### Piped stdin

`claude --bg` reads stdin when not a TTY and truncates at 1 MiB.

### Dangerous permission modes

`--bg` refuses dangerous/auto modes unless the corresponding opt-in/disclaimer
has already been accepted.

### Completed local agents

Completed local agents can linger in the in-REPL panel briefly. Viewing them
sets `retain` to avoid eviction. Dismissing sets `evictAfter = 0`.

## Open Questions and Audit Notes

These are not claims about product behavior, only source-level observations
from the 2.1.141 reconstruction.

1. `disableAgentView` says it disables `claude agents`, `--bg`, `/background`,
   and the on-demand daemon. I see daemon gating clearly, but I do not see an
   equally obvious direct gate in every `claude agents`, `--bg`, or
   `/background` path. This needs binary behavior verification or a closer
   extraction audit.

2. `isDaemonWorkerRegistryEnabled()` returns `false`, which gates worker
   registry commands such as `daemon list`, `scheduled`, `assistant`,
   `remote-control`, and `hub`. The implementation may be present but disabled
   in external 141.

3. Session-level `Ctrl+B` backgrounding appears disabled in the reconstructed
   `SessionBackgroundHint` because the condition uses `isEnvTruthy('false')`.
   Explicit `/background` and `--bg` are separate and present.

4. The names `fleet`, `jobs`, and `daemon` make the code look larger than a
   single feature. Agent View is a cross-cutting feature over background session
   lifecycle, live session discovery, status classification, and terminal UI.

5. Some files include ant/internal feature gates and dead-code-elimination
   patterns. The external build may include disabled code paths as inert code or
   may DCE some branches depending on build defines.

## Practical Reconstruction Notes

If using 2.1.141 as the source-map baseline for future releases, Agent View is
one of the most important feature clusters to track because it touches many
subsystems:

- CLI fast paths
- command registration
- process spawning
- environment propagation
- persistent job state
- live session registry
- socket messaging
- transcript classification
- daemon lifecycle
- Ink TUI
- background task panel
- Agent tool async lifecycle
- SendMessage routing and resume

For future release diffs, changed files in this feature are likely to indicate:

- new background commands
- changes in job state schema
- changes in classifier prompt/heuristics
- changes in the Agent View UI
- changes in daemon/service behavior
- changes in subagent steering/resume behavior
- changes in remote control or bridge integration

The highest-value files to diff first are:

- `source/src/cli/bg.ts`
- `source/src/fleet/FleetView.tsx`
- `source/src/fleet/jobModel.ts`
- `source/src/jobs/state.ts`
- `source/src/jobs/classifier.ts`
- `source/src/jobs/rendezvous.ts`
- `source/src/utils/concurrentSessions.ts`
- `source/src/utils/udsClient.ts`
- `source/src/tools/AgentTool/AgentTool.tsx`
- `source/src/tasks/LocalAgentTask/LocalAgentTask.tsx`
- `source/src/components/CoordinatorAgentStatus.tsx`
- `source/src/components/tasks/BackgroundTasksDialog.tsx`

## Bottom Line

Agent View in 2.1.141 is a substantial background-session orchestration layer.
It is not just a list UI. It combines a process registry, persisted job state,
status classification, socket-based interaction, attach/respawn lifecycle
commands, daemon support, and in-REPL task viewing/steering for local agents.

The feature gives Claude Code a way to keep long-running sessions alive outside
the current terminal and later re-enter them through `claude agents` or
`claude attach <id>`, while keeping enough state to tell the user whether each
session is working, waiting, blocked, completed, failed, or restartable.
