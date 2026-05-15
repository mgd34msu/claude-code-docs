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
