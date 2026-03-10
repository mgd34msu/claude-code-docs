# Claude Code v2.1.71 — /loop Command Deep-Dive

> Source: `cli.js` (612,918 lines, prettified bundle)
> All line numbers reference `cli.js` directly.

---

## Overview

The `/loop` command lets users schedule a prompt or slash command to run on a recurring interval — e.g. `/loop 5m check the deploy` or `/loop 30s /babysit-prs`. It is designed for polling, status checks, and any task the user wants repeated automatically without manual re-invocation.

`/loop` is **not a JavaScript loop**. It is a "prompt skill" — a slash command that generates a scheduling prompt which Claude receives and acts on by calling the `CronCreate` tool. The actual scheduling and firing is handled by a cron-style background scheduler running on a 1-second tick.

**Current status: Statsig-gated, not publicly released.** The command and all three backing tools (`CronCreate`, `CronDelete`, `CronList`) are disabled by default behind the `tengu_kairos_cron` Statsig feature gate. The gate defaults to `false`; the feature is not available to general users.

---

## Table of Contents

1. [Usage](#1-usage)
   - 1.1 [Syntax](#11-syntax)
   - 1.2 [Examples](#12-examples)
   - 1.3 [Limitations](#13-limitations)
2. [Architecture](#2-architecture)
   - 2.1 [Command Registration](#21-command-registration)
   - 2.2 [Execution Flow](#22-execution-flow)
   - 2.3 [Architecture Diagram](#23-architecture-diagram-text)
3. [Cron Scheduler Engine](#3-cron-scheduler-engine)
   - 3.1 [Scheduler Implementation](#31-scheduler-implementation)
   - 3.2 [Cron Expression Parsing](#32-cron-expression-parsing)
   - 3.3 [Task Storage](#33-task-storage)
   - 3.4 [Mutex / Lock File](#34-mutex--lock-file)
4. [Task Lifecycle](#4-task-lifecycle)
   - 4.1 [Task Creation](#41-task-creation)
   - 4.2 [Task Firing](#42-task-firing)
   - 4.3 [Recurring vs One-Shot](#43-recurring-vs-one-shot)
   - 4.4 [Jitter](#44-jitter)
   - 4.5 [Task Expiry & Cancellation](#45-task-expiry--cancellation)
5. [CronCreate / CronDelete / CronList Tools](#5-croncreate--crondelete--cronlist-tools)
   - 5.1 [CronCreate Schema](#51-croncreate-schema)
   - 5.2 [CronDelete Schema](#52-crondelete-schema)
   - 5.3 [CronList Schema & Output](#53-cronlist-schema--output)
6. [Feature Gating](#6-feature-gating)
   - 6.1 [Statsig Gates](#61-statsig-gates)
   - 6.2 [Current Status](#62-current-status)
7. [Telemetry](#7-telemetry)
8. [Integration Points](#8-integration-points)
   - 8.1 [Slash Commands in Loops](#81-slash-commands-in-loops)
   - 8.2 [Background Agents](#82-background-agents)
   - 8.3 [Context Compaction](#83-context-compaction)
9. [Key Code Locations](#9-key-code-locations)

---

## 1. Usage

### 1.1 Syntax

```
/loop [interval] <prompt>
```

- `interval` is optional. If omitted, defaults to **10 minutes** (`Ze6 = "10m"`).
- `interval` must match `^\d+[smhd]$` — a positive integer followed by `s`, `m`, `h`, or `d`.
- `prompt` is the text or slash command to run at each interval.
- The interval may appear as a **leading token** or as a **trailing "every" clause** (see Section 2.2).
- Minimum granularity: **1 minute** (cron limitation). Second-level intervals are rounded up to the nearest minute.

### 1.2 Examples

```
/loop 5m /babysit-prs
    → runs /babysit-prs every 5 minutes

/loop 30m check the deploy
    → runs "check the deploy" every 30 minutes

/loop 1h /standup 1
    → runs /standup 1 every hour

/loop check the deploy
    → defaults to 10m — runs "check the deploy" every 10 minutes

/loop check the deploy every 20m
    → trailing "every" clause — runs every 20 minutes

/loop check the deploy every 5 minutes
    → natural language unit word also supported

/loop 2h summarize new issues
    → runs "summarize new issues" every 2 hours

/loop 1d generate daily report
    → runs at midnight local time every day
```

### 1.3 Limitations

| Constraint | Value | Notes |
|---|---|---|
| Max concurrent jobs | 50 | `tOq = 50`; error code 3 if exceeded |
| Recurring auto-expiry | 3 days | `DQq = 259200000 ms`; fires one final time then deletes |
| Minimum interval | 1 minute | Cron minimum; seconds are rounded up |
| Default interval | 10 minutes | `Ze6 = "10m"` |
| Session vs durable | Session-only by default | `/loop` always uses `durable: false`; jobs die when session ends |
| No `/stop` command | N/A | Cancellation via `CronDelete` tool with job ID |
| Intervals must divide cleanly | e.g. 7m is invalid | LLM rounds to nearest clean interval and notifies user |

---

## 2. Architecture

### 2.1 Command Registration

The `/loop` command is registered as a **"prompt skill"** — a slash command whose handler generates a prompt for Claude rather than executing imperative code directly.

**Registration function `WH` (line 513372–513397):**
```js
function WH(A) {
  let q = {
    type: "prompt",           // generates a prompt, not a direct action
    name: A.name,
    description: A.description,
    hasUserSpecifiedDescription: true,
    allowedTools: A.allowedTools ?? [],
    argumentHint: A.argumentHint,
    whenToUse: A.whenToUse,
    isEnabled: A.isEnabled ?? (() => true),
    isHidden: !(A.userInvocable ?? true),
    progressMessage: "running",
    userFacingName: () => A.name,
    getPromptForCommand: A.getPromptForCommand,
  };
  PLq.push(q);  // PLq is the prompt-skills registry array
}
```

**Loop skill registration `qgz` / `registerLoopSkill` (lines 560998–561033):**
```js
function qgz() {
  WH({
    name: "loop",
    description:
      "Run a prompt or slash command on a recurring interval (e.g. /loop 5m /foo, defaults to 10m)",
    whenToUse:
      'When the user wants to set up a recurring task, poll for status, or run something ' +
      'repeatedly on an interval (e.g. "check the deploy every 5 minutes", ' +
      '"keep running /babysit-prs"). Do NOT invoke for one-off tasks.',
    argumentHint: "[interval] <prompt>",
    userInvocable: true,
    isEnabled: lC,            // gated by tengu_kairos_cron Statsig feature gate
    async getPromptForCommand(A) {
      let q = A.trim();
      if (!q) return [{ type: "text", text: emz }];  // show usage string
      return [{ type: "text", text: Agz(q) }];         // generate scheduling prompt
    },
  });
}
```

**Call site (lines 566608–566612):**
```js
function lgq() {
  let { registerLoopSkill: A } = (Fmq(), W3(gmq));
  A();  // registers /loop into PLq
}
```

**Key properties:**

| Property | Value |
|---|---|
| `name` | `"loop"` |
| `type` | `"prompt"` |
| `argumentHint` | `"[interval] <prompt>"` |
| `userInvocable` | `true` |
| `isEnabled` | `lC` (Statsig gate `tengu_kairos_cron`) |
| `aliases` | none |
| `allowedTools` | `[]` (inherits session tools) |

### 2.2 Execution Flow

The flow has five distinct phases:

**Phase 1 — User invokes `/loop`**

The user types `/loop [interval] <prompt>`. The slash command dispatcher looks up `"loop"` in `PLq`, calls `getPromptForCommand("[interval] <prompt>")`. The handler generates a rich scheduling system prompt via `Agz(input)` and returns it as the message content.

**Phase 2 — Prompt injection and LLM parsing**

Claude receives the `Agz`-generated prompt containing:
1. Interval parsing rules (leading token or trailing "every" clause, regex `^\d+[smhd]$`)
2. Interval-to-cron conversion table
3. Instruction to call `CronCreate` with the derived parameters
4. Instruction to confirm back to the user

Claude performs the parsing and conversion. This is **LLM-driven**, not code-driven — there is no JavaScript `parseInterval()` function.

**Interval → cron conversion table** (line 560975–560981):

| Interval | Cron Expression | Notes |
|---|---|---|
| `Nm` where N ≤ 59 | `*/N * * * *` | Every N minutes |
| `Nm` where N ≥ 60 | `0 */H * * *` | Rounded to hours (H = N/60, must divide 24) |
| `Nh` where N ≤ 23 | `0 */N * * *` | Every N hours |
| `Nd` | `0 0 */N * *` | Every N days at midnight local |
| `Ns` | `ceil(N/60)m` then apply above | Seconds rounded up to minutes |

If an interval does not cleanly divide its unit (e.g. `7m`, `90m`), Claude rounds to the nearest clean interval and informs the user. If the resulting prompt is empty after extracting the interval (e.g. input was just `5m`), Claude shows usage and does not call `CronCreate`.

**Phase 3 — CronCreate tool call**

Claude calls `CronCreate({ cron: "*/5 * * * *", prompt: "check the deploy", recurring: true })`. The tool creates the task, stores it, enables the scheduler flag, and returns the job ID and human-readable schedule.

**Phase 4 — Scheduler runs in background**

The scheduler's `v()` tick function runs every 1000ms. When `Date.now() >= nextFireTimeMs` for a job and the REPL is idle (`isLoading()` returns `false`), the scheduler fires the prompt.

**Phase 5 — Prompt enqueued**

The fired prompt is enqueued into the REPL's priority queue with `priority: "later"` and `isMeta: true`. The REPL processes it as a normal user turn when idle.

### 2.3 Architecture Diagram (text)

```
┌─────────────────────────────────────────────────────────────────────┐
│  User: /loop 5m check the deploy                                    │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Slash Command Dispatcher                                           │
│  PLq registry → find "loop" → getPromptForCommand("5m check ...")  │
│  → Agz(input) generates scheduling system prompt                   │
└────────────────────────────┬────────────────────────────────────────┘
                             │  prompt injected into conversation
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  LLM (Claude)                                                       │
│  Parses: leading token "5m" → interval=5m, prompt="check the deploy"│
│  Converts: "5m" → "*/5 * * * *"                                     │
│  Calls: CronCreate(cron, prompt, recurring=true)                    │
└────────────────────────────┬────────────────────────────────────────┘
                             │  tool call
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  CronCreate Tool (lYz, line 450055)                                 │
│  → UOq() creates task { id: "a3f8b2c1", cron, prompt, createdAt }  │
│  → _F1(task): push to u1.sessionCronTasks[]  (in-memory)           │
│  → XR6(true): set scheduledTasksEnabled = true                     │
│  → returns { id, humanSchedule, recurring, durable }               │
└────────────────────────────┬────────────────────────────────────────┘
                             │
          ┌──────────────────┴──────────────────────┐
          │                                         │
          ▼                                         ▼
┌─────────────────────┐             ┌────────────────────────────────┐
│  LLM confirms:      │             │  Scheduler (g1A, line 572578)  │
│  "Scheduled job     │             │  setInterval(v, 1000ms)        │
│  a3f8b2c1 (Every    │             │  v() on each tick:             │
│  5 minutes).        │             │    if isLoading() → skip        │
│  Auto-expires 3d.   │             │    compute nextFireTime (_l8)  │
│  Use CronDelete     │             │    if now >= nextFireTime:     │
│  to cancel."        │             │      emit telemetry            │
└─────────────────────┘             │      onFire(prompt)            │
                                    │      recompute next fire time  │
                                    └───────────────┬────────────────┘
                                                    │
                                                    ▼
                                    ┌────────────────────────────────┐
                                    │  Priority Queue (Hz)           │
                                    │  HW/jW({ mode:"prompt",        │
                                    │    value: "check the deploy",  │
                                    │    priority: "later",          │
                                    │    isMeta: true,               │
                                    │    workload: "cron" })         │
                                    └───────────────┬────────────────┘
                                                    │
                                                    ▼
                                    ┌────────────────────────────────┐
                                    │  REPL processes as normal turn │
                                    │  (when idle, next in queue)    │
                                    └────────────────────────────────┘
```

---

## 3. Cron Scheduler Engine

### 3.1 Scheduler Implementation

The scheduler factory function is `g1A` (aliased `createCronScheduler`), defined at lines 572578–572758.

**Constructor signature:**
```js
function g1A({
  onFire,         // (prompt: string) => void  — fires the prompt string
  isLoading,      // () => boolean             — true when Claude is mid-query
  assistantMode,  // boolean                   — default false; if true, fires even when loading
  onFireTask,     // optional: (task: CronJob) => void  — fires full task object instead
  onMissed,       // optional: (tasks: CronJob[]) => void  — handles missed one-shots on startup
  dir,            // optional: string  — override project dir for durable tasks
  lockIdentity,   // optional: string  — multi-instance lock identity
  getJitterConfig,// () => JitterConfig  — live config from Statsig
  isKilled,       // () => boolean      — kill switch; if true, scheduler shuts down
}) { ... }
```

**Internal state:**

| Variable | Type | Purpose |
|---|---|---|
| `J` | `CronJob[]` | In-memory cache of durable tasks (loaded from disk) |
| `M` | `Map<id, nextFireTimeMs>` | Cached next fire time per job ID |
| `D` | `Set<id>` | IDs of missed one-shot tasks already surfaced |
| `X` | `Set<id>` | IDs currently being deleted asynchronously |
| `P, W, Z` | interval handles | Polling, tick (1000ms), lock-check intervals |
| `G` | chokidar watcher | File watcher on `scheduled_tasks.json` |
| `V` | `boolean` | Whether this process holds the scheduler lock |
| `f` | `boolean` | Whether scheduler has been stopped |

**Tick function `v()` (line 572625)** — runs every `MQq = 1000 ms`:

```js
function v() {
  if (isKilled?.()) return;                    // feature gate turned off → bail
  if (isLoading() && !assistantMode) return;   // REPL busy → skip this tick

  let now = Date.now();
  // For each task in (durable J[] + in-memory PR6()):
  //   Compute nextFireTime if not in M:
  //     recurring: _l8(cron, createdAt, id, jitterConfig)
  //     one-shot:  dOq(cron, createdAt, id, jitterConfig)
  //   If now >= nextFireTime AND id not in X (deleting):
  //     emit tengu_scheduled_task_fire
  //     call onFireTask(task) || onFire(task.prompt)
  //     if recurring && !aged-out: recompute nextFireTime
  //     if aged-out (XQq): emit tengu_scheduled_task_expired, delete task
  //     if one-shot: delete task (WR6 in-memory / Ha6 durable)
  // Clean up M for IDs no longer present
}
```

**Critical behavior:** The scheduler skips all ticks when `isLoading()` is `true`. It does not buffer missed ticks — it simply checks whether `now >= nextFireTime` on the next idle tick. A prompt that should have fired while Claude was busy fires on the first idle tick after Claude finishes.

**Tick interval constants:**

| Constant | Value | Meaning |
|---|---|---|
| `MQq` | 1000 ms | Scheduler tick interval |
| `bQz` | 5000 ms | Lock acquisition poll interval |
| `IQz` | 300 ms | chokidar `awaitWriteFinish` stabilityThreshold |
| `DQq` | 259200000 ms | Recurring task max age (3 days) |

**Start / lock acquisition `y()` (line 572684):**
1. Attempts to acquire `.claude/scheduled_tasks.lock` via `m1A()`.
2. If lock not available, polls every `bQz = 5000 ms` until acquired.
3. Once locked: attaches chokidar watcher on `scheduled_tasks.json` (re-reads on add/change), starts the 1000ms tick interval.
4. Calls `N(true)` on startup to surface any missed one-shot durable tasks.

**Start without gate (lines 572731–572745):**
If the Statsig gate is not yet enabled, the scheduler polls every `MQq` ms until `EH6()` (`scheduledTasksEnabled`) becomes `true`.

**Stop `stop()` (line 572747):**
Clears all intervals (`P`, `W`, `Z`), closes file watcher (`G`), releases the scheduler lock. Called when the REPL unmounts or receives `SIGTERM`.

### 3.2 Cron Expression Parsing

All cron parsing is handled by a small in-bundle implementation (no external library).

**`bYz(field, {min, max})` (line 449559)** — parses a single cron field:
- `*` → all values in `[min, max]`
- `*/N` → step (every N values)
- `A-B` or `A-B/N` → range with optional step
- Single integer
- Comma-separated list of any of the above

**Field ranges `IYz` (line 449706):**

| Field | Min | Max | Notes |
|---|---|---|---|
| minute | 0 | 59 | |
| hour | 0 | 23 | |
| dayOfMonth | 1 | 31 | |
| month | 1 | 12 | |
| dayOfWeek | 0 | 6 | Sunday=0; 7 is accepted and mapped to 0 |

**`_a6(cronStr)` (line 449593)** — validates and parses a full 5-field cron string. Returns `{ minute[], hour[], dayOfMonth[], month[], dayOfWeek[] }` or `null` if invalid.

**`FOq(parsed, date)` (line 449610)** — computes the next fire `Date` after `date`. Uses brute-force minute iteration with a max of `527040` iterations (≈ 1 year in minutes). Returns `null` if no match found within 1 year.

**`wk6(cronStr, nowMs)` (line 449823)** — convenience wrapper; returns next fire time as milliseconds or `null`.

**`zk6(cronStr, opts?)` (line 449662)** — converts a cron expression to a human-readable string:
- `*/N * * * *` → "Every N minutes" (or "Every minute" if N=1)
- `0 * * * *` → "Every hour"
- `M * * * *` → "Every hour at :MM"
- `M */H * * *` → "Every H hours" (or "Every H hours at :MM")
- `M H * * *` → "Every day at H:MM AM/PM"
- `M H * * N` → "Every Monday at H:MM AM/PM" (weekday name)
- Falls back to raw cron string for complex/unrecognized patterns

### 3.3 Task Storage

Two storage backends are used, selected by the `durable` flag on each task:

**Session-only (in-memory) — default for `/loop`:**

```js
// Global session state object u1
u1.sessionCronTasks  // array, initialized [] (line 2071)

_F1(task)   // push task into sessionCronTasks (line 2527)
PR6()       // get sessionCronTasks array (line 2524)
WR6(ids[])  // remove task IDs from sessionCronTasks, return count removed (line 2530)
XR6(bool)   // set u1.scheduledTasksEnabled (line 2518)
EH6()       // get u1.scheduledTasksEnabled (line 2521)
```

Session tasks do not survive session end. They are not written to disk.

**Durable (persisted to disk):**

File: `.claude/scheduled_tasks.json` relative to the project directory.
Path constant: `QYz = join(".claude", "scheduled_tasks.json")` (line 449863).

File format:
```json
{ "tasks": [ { "id": "a3f8b2c1", "cron": "*/5 * * * *", "prompt": "...", "createdAt": 1712000000000, "recurring": true } ] }
```

`Oa6(dir?)` (line 449730) — reads, parses, and validates the JSON. Required fields: `id` (string), `cron` (string), `prompt` (string), `createdAt` (number). Malformed entries are skipped. Returns `[]` on `ENOENT`, `EACCES`, or `EPERM`.

`QOq(tasks, dir?)` (line 449783) — writes `{ tasks: [...] }` to disk. The `durable` property is stripped from each task before writing.

**Merge function `_k6(dir?)` (line 449817)** — used by `CronList` and the scheduler to see all tasks:
```js
async function _k6(dir) {
  let durable = await Oa6(dir);
  if (dir !== undefined) return durable;   // dir-scoped: durable only
  let session = PR6().map(t => ({ ...t, durable: false }));
  return [...durable, ...session];
}
```

### 3.4 Mutex / Lock File

To prevent multiple Claude processes (same project) from double-firing durable tasks, the scheduler uses a lock file:

- **File**: `.claude/scheduled_tasks.lock`
- **Contents**: `{ sessionId, pid, acquiredAt }`
- **Mechanism** (`m1A`, line 572559): only the process holding the lock processes durable tasks (`V = true`). Other instances poll every `bQz = 5000 ms` trying to acquire it.
- **Stale lock recovery**: if the lock-holder PID is dead, the next process recovers the stale lock.
- **Session-only tasks**: always processed by the instance that created them, regardless of lock state.

---

## 4. Task Lifecycle

### 4.1 Task Creation

**`UOq(cron, prompt, recurring, durable)` (line 449795):**

```js
async function UOq(cron, prompt, recurring, durable) {
  let id = randomUUID().slice(0, 8);  // 8-char hex ID
  let task = {
    id,
    cron,
    prompt,
    createdAt: Date.now(),
    ...(recurring ? { recurring: true } : {}),
  };
  if (!durable) {
    _F1(task);   // push to u1.sessionCronTasks
    return id;
  }
  let tasks = await Oa6();   // read existing durable tasks
  tasks.push(task);
  await QOq(tasks);          // write back to .claude/scheduled_tasks.json
  return id;
}
```

After creation, `XR6(true)` sets `scheduledTasksEnabled = true`, which unblocks the scheduler's startup polling loop.

The `CronCreate` tool returns a confirmation message to the LLM:
- **Recurring**: `"Scheduled recurring job {id} ({humanSchedule}). {persistence}. Auto-expires after 3 days. Use CronDelete to cancel sooner."`
- **One-shot**: `"Scheduled one-shot task {id} ({humanSchedule}). {persistence}. It will fire once then auto-delete."`

(`{persistence}` is either `"Session-only"` or `"Durable".`)

### 4.2 Task Firing

When the scheduler tick determines `Date.now() >= nextFireTimeMs`:

1. Emits `tengu_scheduled_task_fire` telemetry event with `{ recurring, taskId }`.
2. Calls `onFire(task.prompt)` or `onFireTask(task)` depending on which callback was provided.

**REPL mode `onFire` (line 575162–575180):**
```js
onFire: (M6) => HW({
  mode: "prompt",
  value: M6,           // the prompt string
  uuid: NX(),
  priority: "later",   // lowest priority in queue
  isMeta: true,        // system-initiated, not user-typed
  workload: J31,       // J31 = "cron" (line 64107)
})
```

**React/hook mode `onFire` (line 604405):**
```js
onFire: (z) => jW({
  value: z,
  mode: "prompt",
  priority: "later",
  isMeta: true,
  workload: J31,       // "cron"
})
```

`HW` and `jW` both push to the `Hz` prompt queue (lines 250574–250583). `priority: "later"` corresponds to weight `2` in the priority queue (`tG6 = { now: 0, next: 1, later: 2 }` at line 250705).

Cron-fired prompts are the **lowest priority** — they always yield to in-progress and queued user interactions.

### 4.3 Recurring vs One-Shot

| Property | Recurring (`recurring: true`) | One-Shot (`recurring: false`) |
|---|---|---|
| Fires | Repeatedly on cron schedule | Once, then deleted |
| Auto-expiry | After 3 days (unless `permanent: true`) | After firing once |
| Jitter type | `_l8`: fires LATER than nominal | `dOq`: fires EARLIER than nominal |
| Missed on startup | Kept in store, fires next interval | Surfaced via `PQq` notification, then deleted |
| Default for `/loop` | `true` | N/A |

`/loop` always passes `recurring: true` to `CronCreate`. One-shot tasks can be created directly via the `CronCreate` tool (not via `/loop`).

The `permanent: true` field on a task bypasses the 3-day auto-expiry. This field cannot be set via `/loop` or `CronCreate` (it can only be set by manually editing `.claude/scheduled_tasks.json`).

### 4.4 Jitter

The scheduler applies deterministic per-task jitter to avoid thundering-herd problems at common cron boundaries (e.g., every process firing at exactly `:00`).

**Recurring jitter `_l8(cron, createdAt, taskId, config)` (line 449833):**
```js
function _l8(cron, createdAt, taskId, config = O_6) {
  let nextFire = wk6(cron, createdAt);          // nominal next fire time
  let followingFire = wk6(cron, nextFire);       // the fire after that
  let jitter = Math.min(
    pOq(taskId) * config.recurringFrac * (followingFire - nextFire),
    config.recurringCapMs
  );
  return nextFire + jitter;    // fires LATER than nominal
}
```

`pOq(taskId)` (line 449830): parses the first 8 hex chars of the task ID as a float in `[0, 1)`. This is deterministic — the same task ID always produces the same jitter offset.

**One-shot jitter `dOq(cron, createdAt, taskId, config)` (line 449841):**
- Only applied when the fire minute is divisible by `oneShotMinuteMod` (default 30 → fires landing on `:00` or `:30`).
- `jitter = config.oneShotFloorMs + pOq(taskId) * (config.oneShotMaxMs - config.oneShotFloorMs)`
- Result: `max(nextFire - jitter, createdAt)` — fires **earlier** than nominal.

**Default jitter config `O_6` (line 449864):**

```js
O_6 = {
  recurringFrac:     0.1,      // jitter = up to 10% of the period
  recurringCapMs:    900000,   // cap at 15 minutes
  oneShotMaxMs:      90000,    // one-shot fires up to 90 seconds early
  oneShotFloorMs:    0,        // minimum early-fire offset
  oneShotMinuteMod:  30,       // jitter one-shots landing on :00 or :30
}
```

Live config is read from Statsig dynamic config `tengu_kairos_cron_config` via `U1A()` (line 572791), with a 60-second stale window (`xQz = 60000`). Zod validates the live config:
- `recurringFrac`: 0–1
- `recurringCapMs`: 0–1800000 (max 30 min, `Q1A`)
- `oneShotMaxMs`: 0–1800000
- `oneShotFloorMs`: 0–1800000 (must be ≤ `oneShotMaxMs`)
- `oneShotMinuteMod`: 1–60

### 4.5 Task Expiry & Cancellation

**Auto-expiry (recurring tasks):**

`XQq(task, nowMs)` (line 572575):
```js
return Boolean(task.recurring && !task.permanent && nowMs - task.createdAt >= DQq);
// DQq = 259200000 ms = 3 days
```

When `XQq` returns true during a tick:
1. The task fires one final time.
2. `tengu_scheduled_task_expired` is emitted with `{ taskId, ageHours }`.
3. Task is deleted from in-memory store (`WR6`) or disk (`Ha6`).

**Manual cancellation:**

The user (or Claude) calls `CronDelete({ id: "a3f8b2c1" })`. This calls:
- `Ha6([id], dir)` — filters the ID from durable storage and writes back.
- `WR6([id])` — removes from `u1.sessionCronTasks` in-memory.

The scheduler's `M` map entry is cleaned up on the next tick.

**Other auto-stop conditions:**

| Condition | Mechanism |
|---|---|
| One-shot task fires | Deleted immediately after firing (`WR6` / `Ha6`) |
| Session ends | In-memory (`durable: false`) tasks die; durable tasks survive |
| Keyboard interrupt | Aborts current AI request, does NOT delete jobs |
| Feature gate disabled | `isKilled()` check at tick start → scheduler shuts down |
| Scheduler `stop()` | Clears all intervals, closes watcher, releases lock |

**No `/stop` slash command.** Cancellation is via `CronDelete` with the job ID. The job ID is returned by `CronCreate` in Claude's confirmation message and is retrievable via `CronList`.

---

## 5. CronCreate / CronDelete / CronList Tools

All three tools are gated on `lC()` (`tengu_kairos_cron`) and share the same variable-name prefixes.

```js
HU  = "CronCreate"
H_6 = "CronDelete"
RS1 = "CronList"
```

### 5.1 CronCreate Schema

**Lines 450028–450053, tool `lYz` (line 450055).**

**Tool description:** `"Schedule a prompt to run at a future time — either recurring on a cron schedule, or once at a specific time. Session-only: the job dies when this Claude session ends."`

**Input schema** (note: `durable` is intentionally omitted from the exposed schema — `dYz = pYz().omit({ durable: true })`):

```ts
{
  cron:      string   // 5-field standard cron in local timezone: "minute hour dayOfMonth month dayOfWeek"
  prompt:    string   // The prompt to enqueue at each fire time
  recurring: boolean? // default true — true = recurring, false = one-shot
}
```

**`call()` implementation (line 450114):**
```js
async call({ cron: A, prompt: q, recurring: K = true, durable: Y = false }) {
  let z = await UOq(A, q, K, Y);   // creates job, returns 8-char hex ID
  XR6(true);                        // sets scheduledTasksEnabled = true
  return {
    data: { id: z, humanSchedule: zk6(A), recurring: K, durable: Y }
  };
}
```

**Output schema:**
```ts
{ id: string, humanSchedule: string, recurring: boolean, durable?: boolean }
```

**Validation:** Returns error code 3 if 50-job limit (`tOq`) is exceeded.

### 5.2 CronDelete Schema

**Lines 450154+, tool `rYz`.**

**Tool description:** `"Cancel a scheduled cron job by ID"`

**Input schema:**
```ts
{ id: string }   // the 8-char hex ID returned by CronCreate
```

**`call()` implementation:**
```js
async call({ id }) {
  await Ha6([id]);   // removes from durable storage (filters + rewrites JSON)
  WR6([id]);         // removes from u1.sessionCronTasks
  return { data: { id } };
}
```

### 5.3 CronList Schema & Output

**Lines 450243+, tool `sYz`.**

**Tool description:** `"List scheduled cron jobs"`

**Input schema:** `{}` (no parameters)

**`call()` implementation:**
```js
async call() {
  let tasks = await _k6();  // merges durable + in-memory
  return { data: tasks };
}
```

**Output per task:**
```ts
{
  id:            string    // 8-char hex
  cron:          string    // e.g. "*/5 * * * *"
  humanSchedule: string    // e.g. "Every 5 minutes"
  prompt:        string    // the prompt that fires
  recurring?:    boolean
  durable?:      boolean
}
```

---

## 6. Feature Gating

### 6.1 Statsig Gates

**Gate: `tengu_kairos_cron`** (line 449885–449888)

```js
function lC() {
  return jU("tengu_kairos_cron", /* default */ false, /* staleIfOlderThanMs */ 300000);
}
var UYz = 300000;  // 5-minute stale window
```

This gate controls:
- Whether the `/loop` slash command appears in the command list (`isEnabled: lC`)
- Whether `CronCreate`, `CronDelete`, `CronList` tools are enabled (`isEnabled() { return lC(); }`)
- Whether the REPL instantiates the scheduler at all (line 575162)
- Whether the scheduler continues running (checked via `isKilled` on every tick)

**Dynamic config: `tengu_kairos_cron_config`** (read by `U1A()`, line 572791)

Stale window: 60 seconds (`xQz = 60000`). Provides live overrides for all jitter parameters (see Section 4.4). Falls back to `O_6` defaults if not set or invalid.

### 6.2 Current Status

- **Default**: disabled (`false`). The feature is not available to general users.
- **To enable**: requires Statsig gate `tengu_kairos_cron` to be set to `true` for the user/environment.
- **No user-facing toggle**: there is no settings option or environment variable to enable it locally.
- **Gate check interval**: 5 minutes (stale window `UYz = 300000 ms`). If the gate is toggled, it takes up to 5 minutes to take effect.

---

## 7. Telemetry

All telemetry events are emitted by the scheduler engine, not by the `/loop` command handler or the tools themselves.

| Event | Location | Payload | When Emitted |
|---|---|---|---|
| `tengu_scheduled_task_fire` | line 572647 | `{ recurring: boolean, taskId: string }` | Each time a task fires (including the final fire of an aged-out task) |
| `tengu_scheduled_task_missed` | line 572610 | `{ count: number, taskIds: string }` (comma-joined IDs) | On scheduler startup, when durable one-shot tasks were missed during downtime |
| `tengu_scheduled_task_expired` | line 572661 | `{ taskId: string, ageHours: number }` | When a recurring task is deleted due to 3-day age limit |

**Not instrumented:**
- `CronCreate` / `CronDelete` / `CronList` tool invocations
- `/loop` command invocations
- Lock acquisition/release
- Task creation or deletion (other than expiry)

The gate and config read events (`tengu_kairos_cron`, `tengu_kairos_cron_config`) are Statsig infrastructure events, not explicit application-level telemetry calls.

---

## 8. Integration Points

### 8.1 Slash Commands in Loops

Slash commands are passed through verbatim as the `prompt` field in `CronCreate`. The `Agz` system prompt explicitly states: "slash commands are passed through unchanged."

When the scheduler fires, `onFire(promptString)` enqueues the raw string (e.g. `/babysit-prs`) into the `Hz` prompt queue. The REPL processes it on the next idle cycle exactly as if the user had typed it — the slash command dispatcher resolves and executes the command normally.

This means any valid slash command (user-invocable, with or without arguments) can be looped.

### 8.2 Background Agents

The `workload: J31` field (`J31 = "cron"`, line 64107) is stored in `AsyncLocalStorage` when a cron-fired prompt is processed. This marks the entire agent turn (including any sub-calls) as workload type `"cron"`. This allows the runtime to distinguish cron-initiated work from user-initiated work for logging, billing, or timeout purposes.

The scheduler is instantiated per REPL instance. In headless/bridge mode (background agents), the `onFire` callback uses `HW` directly (line 575162) rather than the `jW` React hook variant. Both produce the same queue entry.

### 8.3 Context Compaction

**In-memory tasks (`durable: false`) survive context compaction.** Tasks live in `u1.sessionCronTasks`, which is session state — not conversation context. Context compaction truncates the conversation history but does not clear session state. A loop set up at the start of a session continues firing even after multiple compactions.

**Durable tasks (`durable: true`) survive process restarts.** On scheduler startup, `N(true)` loads tasks from `.claude/scheduled_tasks.json`, identifies any missed one-shot tasks via `cOq(tasks, nowMs)`, and surfaces them via the `PQq` notification prompt:

> "The following one-shot scheduled task(s) were missed while Claude was not running. They have already been removed from .claude/scheduled_tasks.json. Do NOT execute these prompts yet. First use the AskUserQuestion tool to ask whether to run each one now. Only execute if the user confirms."

Recurring durable tasks simply resume firing on their normal schedule — no missed-task notification is generated for them.

---

## 9. Key Code Locations

| Symbol | Lines | Description |
|---|---|---|
| `qgz` / `registerLoopSkill` | 560998–561033 | Registers the `/loop` slash command |
| `WH` | 513372–513397 | Generic prompt-skill registration function |
| `lgq` | 566608–566612 | Call site that invokes `registerLoopSkill()` |
| `Agz` | 560950–560996 | Generates the scheduling system prompt injected into conversation |
| `emz` | 561020–561032 | Usage string returned when `/loop` is invoked with no arguments |
| `lC` | 449885–449888 | `isKairosCronEnabled()` — Statsig gate check |
| `lYz` | 450055–450138 | `CronCreate` tool implementation |
| `rYz` | 450154+ | `CronDelete` tool implementation |
| `sYz` | 450243+ | `CronList` tool implementation |
| `UOq` | 449795–449807 | Creates a new task (in-memory or durable) |
| `_F1` | 2527 | Push task to `u1.sessionCronTasks` |
| `PR6` | 2524 | Get `u1.sessionCronTasks` |
| `WR6` | 2530 | Remove task IDs from in-memory store |
| `XR6` | 2518 | Set `u1.scheduledTasksEnabled` |
| `EH6` | 2521 | Get `u1.scheduledTasksEnabled` |
| `g1A` | 572578–572758 | `createCronScheduler()` factory function |
| `v()` (inner) | 572625–572683 | Scheduler tick function (runs every 1000ms) |
| `y()` (inner) | 572684 | Scheduler start / lock acquisition |
| `N()` (inner) | 572601–572623 | Startup missed-task detection |
| `bYz` | 449559 | Parse single cron field |
| `_a6` | 449593 | Validate and parse full 5-field cron string |
| `FOq` | 449610 | Compute next fire `Date` from parsed cron |
| `wk6` | 449823 | Convenience wrapper: next fire time as ms |
| `zk6` | 449662 | Convert cron to human-readable string |
| `_l8` | 449833 | Compute recurring task next fire time with jitter |
| `dOq` | 449841 | Compute one-shot task next fire time with jitter |
| `pOq` | 449830 | Derive jitter float `[0,1)` from task ID |
| `XQq` | 572575 | Check if recurring task has aged out (3-day check) |
| `O_6` | 449864 | Default jitter config |
| `U1A` | 572791 | `getCronJitterConfig()` — reads live config from Statsig |
| `cOq` | 449848 | Find missed one-shot tasks |
| `PQq` | 572760 | Build missed-task notification prompt |
| `_k6` | 449817 | Merge durable + in-memory tasks (used by CronList) |
| `Oa6` | 449730 | Read durable tasks from JSON file |
| `QOq` | 449783 | Write durable tasks to JSON file |
| `Ha6` | — | Delete task(s) from durable storage |
| `m1A` | 572559 | Scheduler lock acquisition |
| `QYz` | 449863 | Path constant: `.claude/scheduled_tasks.json` |
| `MQq` | 572777 | Tick interval constant: 1000 ms |
| `DQq` | 572780 | Max recurring task age: 259200000 ms (3 days) |
| `tOq` | 450016 | Max concurrent jobs: 50 |
| `Ze6` | 561015 | Default interval: `"10m"` |
| `J31` | 64107 | Workload label: `"cron"` |
| `tG6` | 250705 | Priority queue weights: `{ now:0, next:1, later:2 }` |
| `IYz` | 449706 | Cron field range definitions |
| `UYz` | 449888 | Statsig gate stale window: 300000 ms (5 min) |
| `xQz` | 572791 | Statsig config stale window: 60000 ms (1 min) |
| `Q1A` | — | Max allowed jitter cap in Statsig config: 1800000 ms (30 min) |
