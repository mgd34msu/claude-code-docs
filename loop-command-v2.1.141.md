# Loop, Cron, and Scheduled Tasks in Claude Code 2.1.141

In 2.1.141 the old `/loop` surface is no longer the main implementation path.
The scheduling system is centered on Cron tools and the bundled loop/dream
skills. The slash command source under `source/src/commands/loops` is currently
minimal; scheduled work is implemented by `CronCreate`, `CronDelete`, and
`CronList`.

## Tool Names

Defined in `source/src/tools/ScheduleCronTool/prompt.ts`:

- `CronCreate`
- `CronDelete`
- `CronList`

The tools are imported into the main tool registry only when
`feature('AGENT_TRIGGERS')` is present.

## Runtime Gates

`isKairosCronEnabled()` requires:

- build-time `feature('AGENT_TRIGGERS')`.
- `CLAUDE_CODE_DISABLE_CRON` not set.
- runtime gate `tengu_kairos_cron` true.

`isDurableCronEnabled()` reads:

- `tengu_kairos_cron_durable`

Cron jitter and age behavior are controlled by:

- `tengu_kairos_cron_config`

## CronCreate

Source: `source/src/tools/ScheduleCronTool/CronCreateTool.ts`.

Input:

- `cron`: five-field cron expression.
- `prompt`: prompt to run.
- `recurring`: optional boolean.
- `durable`: optional boolean.

Behavior:

- validates a five-field cron expression.
- requires the next run to be within one year.
- enforces a maximum of 50 jobs.
- defaults to session-only unless durable is explicitly selected.
- durable jobs write to `.claude/scheduled_tasks.json`.
- session-only jobs live in memory and disappear when the session exits.
- durable teammate crons are rejected because teammates do not persist across
  sessions.
- recurring tasks auto-expire after the configured max age.

## CronDelete

Source: `source/src/tools/ScheduleCronTool/CronDeleteTool.ts`.

Input:

- `id`: job id returned by `CronCreate`.

Behavior:

- validates that the task exists.
- deletes durable tasks from `.claude/scheduled_tasks.json`.
- deletes session-only tasks from the in-memory store.
- teammates may only delete their own cron jobs.

## CronList

Source: `source/src/tools/ScheduleCronTool/CronListTool.ts`.

Behavior:

- read-only.
- concurrency safe.
- lists durable and session-only tasks.
- teammates see only their own tasks.
- lead sessions can see all team-visible tasks.

## Storage

The storage implementation is in `source/src/utils/cronTasks.ts`.

Durable path:

- `<project>/.claude/scheduled_tasks.json`

File shape:

- top-level object with `tasks`.
- each task includes id, cron, prompt, creation time, optional last-fire time,
  recurring/permanent fields, and runtime-only metadata.

Runtime-only fields such as durable/session owner state are stripped before
writing durable task files.

## Scheduler

Scheduler implementation:

- `source/src/utils/cronScheduler.ts`

Behavior:

- starts from persisted tasks or scheduler-enabled state.
- watches the durable scheduled tasks file.
- uses a lock-file ownership model.
- surfaces missed one-shot durable tasks when appropriate.
- handles fired, missed, expired, and killed task states.

Telemetry includes:

- `tengu_scheduled_task_fire`
- `tengu_scheduled_task_missed`
- `tengu_scheduled_task_expired`

## Bundled Skills

The bundled loop skill uses cron tooling for recurring work. The bundled dream
skill can schedule nightly consolidation through `CronCreate`.

Relevant files:

- `source/src/skills/bundled/loop.ts`
- `source/src/skills/bundled/dream.ts`

## 2.1.141 Source Index

- `source/src/commands/loops/loops.tsx`
- `source/src/tools/ScheduleCronTool/prompt.ts`
- `source/src/tools/ScheduleCronTool/CronCreateTool.ts`
- `source/src/tools/ScheduleCronTool/CronDeleteTool.ts`
- `source/src/tools/ScheduleCronTool/CronListTool.ts`
- `source/src/utils/cronTasks.ts`
- `source/src/utils/cronScheduler.ts`
- `source/src/skills/bundled/loop.ts`
- `source/src/skills/bundled/dream.ts`

## Detailed 2.1.141 Scheduling Model

The 2.1.141 scheduling model is not a `/loop` command implementation with a
little cron helper. It is a Cron tool family plus scheduler service. The slash
command file exists, but actual scheduling behavior is tool-driven.

Core components:

- prompt/feature helpers in `ScheduleCronTool/prompt.ts`.
- create/delete/list tools.
- durable/session task storage in `cronTasks.ts`.
- scheduler loop and file watcher in `cronScheduler.ts`.
- REPL/print integration that starts and reacts to scheduled tasks.
- bundled skills that call cron tools.

## CronCreate Schema and Validation

Input:

- `cron`: five-field cron expression.
- `prompt`: prompt to run at firing time.
- `recurring`: optional boolean.
- `durable`: optional boolean.

Validation:

- expression must be five fields.
- next run must be within one year.
- max job count is 50.
- durable teammate jobs are rejected.
- durable behavior only works when durable cron is enabled.

Storage behavior:

- default is session-only.
- `durable: true` writes `.claude/scheduled_tasks.json`.
- recurring jobs auto-expire after configured age.
- one-shot durable jobs missed while closed can be surfaced for catch-up.

## CronDelete Schema

Input:

- `id`: job id returned by `CronCreate`.

Behavior:

- validates task existence.
- removes from durable file or session store.
- teammate callers can only delete their own jobs.
- returns deletion state and message through tool result UI.

## CronList Schema

Input:

- empty object.

Behavior:

- lists durable and session-only jobs.
- read-only.
- concurrency safe.
- lead sessions see all relevant jobs.
- teammates see only teammate-owned jobs.

## Durable File Format

Path:

```text
<project>/.claude/scheduled_tasks.json
```

Top-level shape:

```json
{
  "tasks": [
    {
      "id": "cron_...",
      "cron": "*/5 * * * *",
      "prompt": "check status",
      "createdAt": 1730000000000,
      "lastFiredAt": 1730000300000,
      "recurring": true,
      "permanent": false
    }
  ]
}
```

Runtime fields such as durable/session owner metadata are stripped before write.
Malformed or invalid entries are dropped on read.

## Session-Only Tasks

Session-only jobs are stored in memory:

- they do not write disk state.
- they disappear when the process exits.
- they are appropriate for reminders or short follow-ups.
- they are the default because most user requests do not imply persistence.

The prompt explicitly tells the model to use durable only when the user asks for
persistence.

## Jitter and Expiration

The cron prompt discourages scheduling at `:00` and `:30` marks. The scheduler
also applies deterministic jitter from `tengu_kairos_cron_config`.

Expiration:

- recurring jobs auto-expire after `DEFAULT_MAX_AGE_DAYS` unless marked
  permanent by config/path behavior.
- expired task firing logs telemetry and removes/ignores the task.

## Scheduler Behavior

The scheduler:

- starts when scheduled tasks are enabled or a durable file exists.
- watches `.claude/scheduled_tasks.json`.
- uses a lock-owner model to avoid multiple sessions firing the same durable
  job.
- surfaces missed one-shot durable tasks.
- invokes an `onFireTask` callback with prompt/task data.
- invokes `onMissed` for missed one-shot behavior.
- can be killed/stopped.

The scheduler is used in both interactive REPL and print/SDK paths.

## Feature Gates

Required layers:

- build-time `feature('AGENT_TRIGGERS')`.
- runtime `tengu_kairos_cron`.
- local kill switch absent: `CLAUDE_CODE_DISABLE_CRON`.

Durability:

- `tengu_kairos_cron_durable`.

Config:

- `tengu_kairos_cron_config`.

## Bundled Skill Relationship

The bundled loop skill uses Cron tools rather than owning scheduling. The dream
skill can schedule nightly dream consolidation through `CronCreate`.

This matters for reconstruction: if cron tool behavior changes, `/loop` and
`/dream nightly` behavior can change even if the slash command file does not.

## Telemetry Detail

Scheduler events:

- `tengu_scheduled_task_fire`
- `tengu_scheduled_task_missed`
- `tengu_scheduled_task_expired`

Skill/user events:

- `tengu_dream_invoked`

Normal tool telemetry also records Cron tool execution through the generic
`tengu_tool_use_*` family.

## Operational Guidance

- Use session-only for "remind me soon" or "check again in an hour".
- Use durable only when the user asks for persistence across Claude restarts.
- Use one-shot durable for important catch-up tasks that should surface if
  missed.
- Use recurring durable for permanent recurring requests.
- Use `CronList` before deleting if the id is unknown.
- Use `CronDelete` for cleanup instead of editing the JSON file manually.

## Cron Parser Details

2.1.141 implements a deliberately narrow cron parser in `utils/cron.ts`.
Supported syntax:

- five fields only: minute, hour, day-of-month, month, day-of-week.
- wildcard: `*`.
- step: `*/N`.
- range: `N-M`.
- range with step: `N-M/S`.
- comma lists.
- numeric values only.
- day-of-week accepts `7` as a Sunday alias.

Unsupported syntax:

- named months.
- named weekdays.
- `L`.
- `W`.
- `?`.
- seconds fields.
- year fields.
- timezone suffixes.

All local scheduled tasks use the process local timezone. `0 9 * * *` means 9am
where the CLI process is running. CCR/remote display helpers can render UTC cron
strings as local time for display, but local durable/session cron is local-time
based.

## Next-Run Semantics

The next-run calculator walks forward minute by minute from the next whole
minute after `from`. It is bounded at 366 days. For valid cron expressions this
should find a match; the bound exists to keep the function total and safe.

Day matching follows standard cron semantics:

- if day-of-month and day-of-week are both wildcarded, any day matches.
- if only one is constrained, that constrained field must match.
- if both are constrained, either one may match.

DST behavior is explicitly documented in source:

- a spring-forward gap skips nonexistent local times.
- wildcard-hour jobs fire at the first valid minute after the gap.
- fall-back repeated local times fire once because the stepping logic moves
  forward past the duplicate occurrence.

This makes scheduling predictable without pulling in a full cron dependency.

## Jitter Policy

The tool prompt tells the model to avoid the `:00` and `:30` minute marks when
the user does not require exact timing. The runtime also applies deterministic
jitter:

- recurring tasks can fire up to 10 percent of their period late, capped at 15
  minutes.
- one-shot tasks landing on `:00` or `:30` can fire up to 90 seconds early.
- off-minute scheduling is still preferred because it spreads fleet load before
  runtime jitter even matters.

The operational intent is load distribution. User prompts like "around 9" should
not be rounded to exactly `0 9 * * *`.

## Durable Versus Session-Only

`CronCreate` has a `durable` boolean, but durable behavior is gated separately
from the overall cron feature:

- `isKairosCronEnabled()` gates the scheduler and cron tools.
- `isDurableCronEnabled()` gates disk persistence.
- `CLAUDE_CODE_DISABLE_CRON` disables the whole scheduler.
- the durable GrowthBook gate can force `durable: false` while keeping
  session-only jobs available.

When durable cron is available, jobs persist to `.claude/scheduled_tasks.json`.
When durable cron is unavailable, the tool description and prompt are rewritten
so the model is told jobs are session-only.

## Runtime Firing Model

Cron jobs do not interrupt an active model turn. They enqueue prompts when the
REPL is idle. The lifecycle is:

1. scheduler computes next run.
2. idle loop observes due job.
3. runtime applies missed/expired/recurring logic.
4. prompt is enqueued into the correct context.
5. one-shot jobs are deleted after firing.
6. recurring jobs advance their next run.
7. expired recurring jobs fire their final time and are removed.

For teammate-created cron jobs, routing is different: the fired prompt is sent
to the creating teammate's pending message queue, not to the leader as a generic
prompt.

## Durable File Hygiene

The durable file is an implementation file, not a recommended user-editable
interface. Manual edits can break:

- schema shape.
- task IDs.
- recurring flags.
- next-run timestamps.
- teammate/agent routing metadata.
- max-age/expiration behavior.
- jitter assumptions.

The correct user-level modification tools are `CronList` and `CronDelete`.
Future source-map work should still document the durable file schema because it
is part of the product state.

## Loop Skill Relationship

The `/loop` surface delegates to the cron tools. It should be documented as a
user-facing workflow for creating scheduled prompts, not as a separate scheduler.
If `CronCreate` semantics change, `/loop` changes even if the skill file does
not.

`/dream nightly` also delegates to the same cron tools and intentionally uses a
prompt form that resolves to `/dream consolidate` without colliding with loop
scheduling keywords.

## Failure Modes

Important failure and edge cases:

- invalid cron syntax fails validation.
- unsupported cron syntax is rejected rather than partially interpreted.
- durable disabled means requested durable jobs become session-only or are
  described as unavailable, depending on call path.
- session-only jobs are lost when the process exits.
- durable missed one-shot jobs surface for catch-up.
- recurring jobs auto-expire after the configured max age.
- scheduler kill switch stops running schedulers on their next poll/isKilled
  check.
- remote/CCR paths may display UTC-derived schedules differently from local
  session schedules.

## Future Diff Checklist

For later releases, inspect:

1. `utils/cron.ts` parser support.
2. `utils/cronTasks.ts` storage and jitter constants.
3. `cronTasksLock` for file locking behavior.
4. `CronCreateTool`, `CronDeleteTool`, and `CronListTool` schemas.
5. `ScheduleCronTool/prompt.ts` for model-facing instructions.
6. `/loop` bundled skill text.
7. `/dream` bundled skill scheduling path.
8. teammate cron routing.
9. GrowthBook keys `tengu_kairos_cron` and `tengu_kairos_cron_durable`.
10. telemetry events for fire, missed, expired, create, delete, and list.
