# TeammateIdle & TaskCompleted Hooks -- Complete Deep Dive

In-depth reverse-engineering analysis of the two agent-team hook events in Claude Code CLI.

---

## Table of Contents

1. [Hook I/O Schemas](#hook-io-schemas)
   - [stdin Input Schema (what your hook receives)](#stdin-input-schema)
   - [stdout Output Schema (what your hook returns)](#stdout-output-schema)
   - [Exit Code Semantics](#exit-code-semantics)
   - [I/O Flow Diagram](#io-flow-diagram)
2. [TeammateIdle Hook](#teammateidle-hook)
   - [Schema Definition](#ti-schema-definition)
   - [stdin Payload Construction](#ti-stdin-payload-construction)
   - [Complete Trigger Chain](#ti-complete-trigger-chain)
   - [Exit Code Behavior](#ti-exit-code-behavior)
   - [Idle Detection Flow](#ti-idle-detection-flow)
3. [TaskCompleted Hook](#taskcompleted-hook)
   - [Schema Definition](#tc-schema-definition)
   - [stdin Payload Construction](#tc-stdin-payload-construction)
   - [Trigger Points](#tc-trigger-points)
   - [Exit Code Behavior](#tc-exit-code-behavior)
   - [Unused Notification Infrastructure](#tc-unused-notification-infrastructure)
4. [Side-by-Side Comparison](#side-by-side-comparison)
5. [Common Hook Infrastructure](#common-hook-infrastructure)

---

## Hook I/O Schemas

This section describes the complete input/output contract for hooks -- what your hook command receives via stdin, what it should return via stdout, and how exit codes are interpreted.

### stdin Input Schema

**Serialization:** The hook input is `JSON.stringify(hookInput)` written to the child process's stdin (`W.stdin.write(Y, "utf8")`), then stdin is closed (`W.stdin.end()`). The JSON is constructed via `x = Q1(A)` where `Q1` = `JSON.stringify`.


#### Base Fields (all hook events)

Every hook event includes these base fields from `NZ` populated by `hX()`:

```json
{
  "session_id": "string (required) - current session UUID from U6()",
  "transcript_path": "string (required) - path to transcript file from p$(session_id)",
  "cwd": "string (required) - current working directory from y6()",
  "permission_mode": "string (optional) - e.g. 'default', 'plan', 'acceptEdits', etc.",
  "hook_event_name": "string (required) - the event type literal"
}
```

#### TeammateIdle stdin Payload

Constructed by `KLA()`:

```json
{
  "session_id": "abc-123-def",
  "transcript_path": "/home/user/.claude/projects/.../transcript.jsonl",
  "cwd": "/home/user/project",
  "permission_mode": "default",
  "hook_event_name": "TeammateIdle",
  "teammate_name": "string (required) - agent name e.g. 'researcher'",
  "team_name": "string (required) - team name e.g. 'my-team'"
}
```

#### TaskCompleted stdin Payload

Constructed by `OQ1()`:

```json
{
  "session_id": "abc-123-def",
  "transcript_path": "/home/user/.claude/projects/.../transcript.jsonl",
  "cwd": "/home/user/project",
  "permission_mode": "default",
  "hook_event_name": "TaskCompleted",
  "task_id": "string (required) - task identifier",
  "task_subject": "string (required) - task title/subject",
  "task_description": "string (optional) - task description body",
  "teammate_name": "string (optional) - which teammate completed it",
  "team_name": "string (optional) - which team"
}
```

### stdout Output Schema

**Parsing:** stdout is parsed by `jd4()`. Validated against `yO6 = b.union([Av9, qv9])`.

**Parsing rules (from `jd4()`):**
1. stdout is trimmed
2. If it does NOT start with `{` -> treated as plain text, no structured processing
3. If it starts with `{` -> parsed as JSON via `MA()` (JSON.parse), then validated against `yO6`
4. If JSON validation fails -> treated as non-blocking error with validation message

**The output schema is a union of two forms:**

#### Form 1: Async Response (`Av9`)

Tells Claude Code to background the hook process:

```json
{
  "async": true,
  "asyncTimeout": 30000
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `async` | `true` (literal) | Yes | Must be exactly `true` |
| `asyncTimeout` | `number` | No | Timeout in ms for the backgrounded process |

Detected by `gq1()`: `"async" in A && A.async === true`

#### Form 2: Synchronous Response (`qv9`)

The primary response form -- all fields optional:

```json
{
  "continue": true,
  "suppressOutput": false,
  "stopReason": "Stopping because...",
  "decision": "approve",
  "reason": "Explanation for the decision",
  "systemMessage": "Warning shown to user",
  "hookSpecificOutput": { ... }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `continue` | `boolean` | No | Whether Claude should continue after hook (default: `true`). Set to `false` to halt. |
| `suppressOutput` | `boolean` | No | Hide stdout from transcript (default: `false`) |
| `stopReason` | `string` | No | Message shown when `continue` is `false` |
| `decision` | `"approve"` \| `"block"` | No | Approve or block the operation |
| `reason` | `string` | No | Explanation for the decision |
| `systemMessage` | `string` | No | Warning message shown to the user |
| `hookSpecificOutput` | `object` | No | Event-specific structured output (see below) |

**Processing by `Wd4()`:**
- `continue: false` -> sets `preventContinuation: true`, optionally `stopReason`
- `decision: "block"` -> sets `blockingError` with `reason` as message
- `decision: "approve"` -> sets `permissionBehavior: "allow"`
- `systemMessage` -> forwarded as system-level warning

#### hookSpecificOutput Variants

The `hookSpecificOutput` field is a discriminated union on `hookEventName`. **Neither TeammateIdle nor TaskCompleted have their own variant** -- they use only the top-level fields. The available variants are:

| hookEventName | Key Fields | Description |
|---|---|---|
| `"PreToolUse"` | `permissionDecision` (`"allow"` \| `"deny"` \| `"ask"`), `permissionDecisionReason`, `updatedInput` (object), `additionalContext` | Can modify tool input, override permissions |
| `"UserPromptSubmit"` | `additionalContext` | Inject context into user prompt |
| `"SessionStart"` | `additionalContext` | Inject context at session start |
| `"Setup"` | `additionalContext` | Inject context during setup |
| `"SubagentStart"` | `additionalContext` | Inject context for subagent |
| `"PostToolUse"` | `additionalContext`, `updatedMCPToolOutput` | Modify MCP tool output |
| `"PostToolUseFailure"` | `additionalContext` | Add context after tool failure |
| `"Notification"` | `additionalContext` | Add context to notification |
| `"PermissionRequest"` | `decision: { behavior: "allow", updatedInput?, updatedPermissions? }` or `{ behavior: "deny", message?, interrupt? }` | Override permission decisions |

**For TeammateIdle and TaskCompleted hooks:** Use only the top-level fields (`continue`, `decision`, `reason`, `systemMessage`, etc.). The `hookSpecificOutput` field is ignored/not applicable.

### Exit Code Semantics

Exit code handling:

| Exit Code | Outcome | Behavior |
|---|---|---|
| **0** | `"success"` | Hook completed successfully. If stdout is JSON, it's processed by `Wd4()`. If plain text, logged. |
| **2** | `"blocking"` | **VETO.** Yields `{ blockingError: { blockingError: "[command]: stderr", command } }`. Prevents the triggering action. |
| **any other** | `"non_blocking_error"` | Warning only. Logged but does not prevent the action. Stderr included in warning message. |
| **exception** | `"non_blocking_error"` | Process failed to spawn/run. Error message logged, action proceeds. |

**Exit 2 blocking error format:**
```
[shell_command]: stderr_output_or_"No stderr output"
```

**For TeammateIdle:** Blocking error formatted by `okA()`:
```
TeammateIdle hook feedback:
[command]: stderr
```

**For TaskCompleted:** Blocking error formatted by `$Q1()`:
```
TaskCompleted hook feedback:
[command]: stderr
```

### I/O Flow Diagram

```
  Hook Event Triggered
         |
         v
  sh() orchestrator
    |-- Gets matching hooks via ikA()
    |-- For each matching hook:
         |
         v
  zW6() spawner
    |-- Spawns: child_process.spawn(command, [], { shell: true })
    |-- stdin.write(JSON.stringify(hookInput), "utf8")
    |-- stdin.end()
    |-- Waits for process exit (or timeout, default 600s)
    |
    +-- stdout captured -> jd4() parser
    |     |-- Starts with "{"? -> JSON.parse -> yO6.safeParse()
    |     |     |-- Valid? -> { json: parsed_data }
    |     |     +-- Invalid? -> { validationError: message }
    |     +-- Doesn't start with "{"? -> { plainText: raw_stdout }
    |
    +-- Exit code check:
          |-- 0: Process Wd4() on JSON output, yield success
          |-- 2: yield { blockingError: "[cmd]: stderr" }
          +-- other: yield non_blocking_error warning
         |
         v
  Results aggregated via xO6() (unlimited concurrency)
    |-- All hooks for event run in parallel
    |-- blockingErrors, messages, permissions collected
    +-- Yielded back to caller (KLA/OQ1)
```

### Hook Configuration Schema

Hooks are configured in settings files (`.claude/settings.json`, `.claude/settings.local.json`, project `.claude/settings.json`, etc.).

**Top-level settings schema (`dE`):**

```typescript
// dE = Record<HookEventName, HookMatcher[]>
hooks: {
  "TeammateIdle": [ ...matchers ],
  "TaskCompleted": [ ...matchers ],
  // ... any of the 15 event names
}
```

**Valid event names (`LjY`):**
```
PreToolUse, PostToolUse, PostToolUseFailure, Notification,
UserPromptSubmit, SessionStart, SessionEnd, Stop,
SubagentStart, SubagentStop, PreCompact, PermissionRequest,
Setup, TeammateIdle, TaskCompleted
```

**Hook Matcher (`Vz8`):**

```json
{
  "matcher": "string (optional) - pattern to match against (e.g. tool name, agent type)",
  "hooks": [ ...hook_definitions ]
}
```

For TeammateIdle and TaskCompleted, `matcher` is **not used** -- `ikA()` sets the match query to `undefined` for both events, so ALL registered hooks fire regardless of matcher value.

**Hook Definitions (`fz8` = discriminated union on `type`):**

Three types available:

**1. Command hook (`o2K`):**
```json
{
  "type": "command",
  "command": "string - shell command to execute",
  "timeout": "number (optional) - timeout in seconds",
  "statusMessage": "string (optional) - spinner message",
  "once": "boolean (optional) - run once then remove",
  "async": "boolean (optional) - run in background"
}
```

**2. Prompt hook (`a2K`):**
```json
{
  "type": "prompt",
  "prompt": "string - LLM prompt, use $ARGUMENTS for hook input JSON",
  "timeout": "number (optional) - timeout in seconds",
  "model": "string (optional) - model ID, defaults to small fast model",
  "statusMessage": "string (optional) - spinner message",
  "once": "boolean (optional) - run once then remove"
}
```

**3. Agent hook (`s2K`):**
```json
{
  "type": "agent",
  "prompt": "string - what to verify, use $ARGUMENTS for hook input JSON",
  "timeout": "number (optional) - timeout in seconds, default 60",
  "model": "string (optional) - model ID, defaults to Haiku",
  "statusMessage": "string (optional) - spinner message",
  "once": "boolean (optional) - run once then remove"
}
```

**Complete example configuration:**

```json
{
  "hooks": {
    "TeammateIdle": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 /path/to/on-idle.py",
            "timeout": 30
          }
        ]
      }
    ],
    "TaskCompleted": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash /path/to/on-task-done.sh",
            "timeout": 60
          },
          {
            "type": "prompt",
            "prompt": "Review whether the task described by $ARGUMENTS was completed correctly",
            "model": "claude-sonnet-4-5-20250929"
          }
        ]
      }
    ]
  }
}
```

### Hook Type Compatibility

Not all hook types work with TeammateIdle and TaskCompleted. The execution path inside `sh()` requires specific context parameters that the executor functions may or may not provide.

**Requirements by hook type:**
- **command**: Only needs the spawner `zW6()` -- no special context needed
- **prompt**: Requires `toolUseContext` (parameter `w` in `sh()`), throws if missing
- **agent**: Requires BOTH `toolUseContext` AND `messages` (parameter `H`), throws if either missing

**What each executor passes to `sh()`:**

`KLA()` (TeammateIdle):
```typescript
yield* sh({ hookInput: w, toolUseID: bv(), signal: Y, timeoutMs: z });
// No toolUseContext, no messages
```

`OQ1()` (TaskCompleted):
```typescript
yield* sh({
  hookInput: _, toolUseID: bv(), signal: H, timeoutMs: $,
  toolUseContext: O,   // passed!
  // No messages
});
```

**Compatibility matrix:**

| Hook Type | TeammateIdle | TaskCompleted | Failure Mode |
|---|---|---|---|
| **command** | Works | Works | N/A |
| **prompt** | FAILS | Works | `"ToolUseContext is required for prompt hooks. This is a bug."` |
| **agent** | FAILS | FAILS | `"ToolUseContext/Messages required for agent hooks. This is a bug."` |

Failures are caught by the try/catch and yielded as `non_blocking_error` -- Claude Code won't crash, but the hook won't execute and a warning is logged.

### Matcher Behavior for These Events

The `matcher` field in hook configuration (`Vz8`) is **accepted by the schema but functionally ignored** for both TeammateIdle and TaskCompleted.

`ikA()` handles these events with a bare `break`:

```typescript
case "TeammateIdle":
case "TaskCompleted":
  break;   // H (match query) remains undefined
```

Then:
```typescript
let O = (H ? w.filter((W) => !W.matcher || ARY(H, W.matcher)) : w).flatMap(...)
```

When `H` is `undefined`, the ternary takes the `else` branch -- `w` is used directly without filtering. This means **ALL registered hooks fire for every TeammateIdle/TaskCompleted event**, regardless of matcher value.

Contrast with events like `PreToolUse` where `H = Y.tool_name` enables filtering by tool name (e.g., `matcher: "Write"` only fires for the Write tool).

**What matchers COULD filter (if supported):**
- TeammateIdle: teammate_name or team_name (but neither is wired)
- TaskCompleted: task_id, task_subject, teammate_name (but none are wired)

### Settings Precedence

Hook configurations are merged from 6 sources by `qRY()`:

1. **Policy settings** (`eT7()`) - highest priority, if `allowManagedHooksOnly` is true, ONLY these are used
2. **Merged settings** (`$V1()` / `P8()`) - combined user/project/local settings
3. **Plugin hooks** - from installed plugins (skipped if managed-only mode)
4. **Session hooks** - temporary hooks added programmatically during a session

If `policySettings.allowManagedHooksOnly === true`, all non-policy hooks are ignored.

---

## TeammateIdle Hook

### TI: Schema Definition

**Variable:** `QjY`

Extends the base schema `NZ` (defined):

```typescript
// Base schema NZ (all hooks share this)
NZ = b.object({
  session_id: b.string(),
  transcript_path: b.string(),
  cwd: b.string(),
  permission_mode: b.string().optional(),
})

// TeammateIdle schema
QjY = NZ.and(
  b.object({
    hook_event_name: b.literal("TeammateIdle"),
    teammate_name: b.string(),    // required
    team_name: b.string(),        // required
  })
)
```

**No matcher metadata** -- this hook fires for ALL teammate idle events. You cannot filter by teammate name or team name via hook config matchers.

### TI: stdin Payload Construction

Built by `hX()`:

```typescript
function hX(A, q) {
  let K = q ?? U6();      // session ID
  return {
    session_id: K,
    transcript_path: p$(K),  // path to transcript file
    cwd: y6(),               // current working directory
    permission_mode: A,      // e.g. "default", "plan", etc.
  };
}
```

Then merged with event-specific fields in `KLA()`:

```typescript
async function* KLA(A, q, K, Y, z = cj) {
  let w = {
    ...hX(K),                            // base fields
    hook_event_name: "TeammateIdle",
    teammate_name: A,                     // teammate agent name
    team_name: q,                         // team name
  };
  yield* sh({ hookInput: w, toolUseID: bv(), signal: Y, timeoutMs: z });
}
```

### TI: Complete Trigger Chain

```
Agent turn completes -> VR() returns
  -> tp() updates state
  -> _t4() stop hook executor
    -> First: fires TaskCompleted for all in_progress tasks owned by this teammate
    -> Then: KLA() fires TeammateIdle
      -> sh() (generic hook executor)
        -> zW6() (child_process.spawn with shell)
```

**Critical sequence:**

```typescript
if (Kz()) {  // if in team mode
  let m = B5() ?? "",      // current teammate name
      x = Q3() ?? "",      // team name
      U = [];              // collects blocking errors

  // FIRST: fire TaskCompleted for all in_progress tasks owned by this teammate
  let g = nM(),
      p = zX(g).filter((c) => c.status === "in_progress" && c.owner === m);
  for (let c of p) {
    let Y1 = OQ1(c.id, c.subject, c.description, m, x, P, ...);
    for await (let f1 of Y1) {
      if (f1.blockingError) U.push(g6({ content: $Q1(f1.blockingError), isMeta: true }));
    }
  }

  // THEN: fire TeammateIdle
  let r = KLA(m, x, P, H.abortController.signal);
  for await (let c of r) {
    if (c.blockingError) U.push(g6({ content: okA(c.blockingError), isMeta: true }));
  }

  // If ANY blocking errors from either hook: continue conversation
  if (U.length > 0)
    yield* EZ({
      messages: [...A, ...q, ...U],   // original messages + blocking feedback
      systemPrompt: K,
      // ... continues the teammate's conversation
    });
}
```

### TI: Exit Code Behavior

| Exit Code | Behavior |
|---|---|
| **0** | Success -- teammate goes idle normally |
| **2** | **BLOCKING** -- prevents idle entirely |
| **other** | Non-blocking warning -- teammate still goes idle |

**Exit 2 mechanism (the veto):**

1. `KLA()` yields a `blockingError` object
2. catches it: `U.push(g6({ content: okA(c.blockingError), isMeta: true }))`
3. Error formatted by `okA()`:
   ```
   TeammateIdle hook feedback:
   [command]: stderr
   ```
4. `yield* EZ({...})` -- continues the teammate's conversation with the feedback messages appended
5. The teammate gets **another turn** with hook's stderr as a meta-message
6. **Team lead sees NOTHING** -- the feedback goes only to the teammate

### TI: Idle Detection Flow (when NOT vetoed)

After `_t4()` completes without blocking errors:

```typescript
tp(K, (t) => {
  t.onIdleCallbacks?.forEach((j1) => j1());    // fire idle callbacks
  return { ...t, isIdle: true, onIdleCallbacks: [] };
}, j);

if (!D1)   // if wasn't already idle
  zI4(q.agentName, q.color, q.teamName, {
    idleReason: f1 ? "interrupted" : "available",
    summary: sm1(V),
  });
```

**Key insight:** The hook fires **BEFORE** `isIdle` is set to `true`. Exit 2 triggers `EZ()` continuation which loops back to the agent turn, so the `isIdle = true` code never executes.

Idle notification chain: `zI4()` -> `nm1()` -> `gGY()` -> `X9()` writes to team inbox.

---

## TaskCompleted Hook

### TC: Schema Definition

**Variable:** `UjY`

```typescript
UjY = NZ.and(
  b.object({
    hook_event_name: b.literal("TaskCompleted"),
    task_id: b.string(),                     // required
    task_subject: b.string(),                // required
    task_description: b.string().optional(), // optional
    teammate_name: b.string().optional(),    // optional
    team_name: b.string().optional(),        // optional
  })
)
```

**No matcher metadata** -- fires for ALL task completions regardless of task ID, subject, or owner.

### TC: stdin Payload Construction

Built by `OQ1()`:

```typescript
async function* OQ1(A, q, K, Y, z, w, H, $ = cj, O) {
  let _ = {
    ...hX(w),                         // base fields (session, transcript, cwd, permission)
    hook_event_name: "TaskCompleted",
    task_id: A,                        // task ID
    task_subject: q,                   // task subject text
    task_description: K,               // task description (optional)
    teammate_name: Y,                  // which teammate (optional)
    team_name: z,                      // team name (optional)
  };
  yield* sh({
    hookInput: _,
    toolUseID: bv(),
    signal: H,
    timeoutMs: $,
    toolUseContext: O,
  });
}
```

### TC: Trigger Points

TaskCompleted fires from **two distinct locations**:

#### Trigger 1: TaskUpdate tool handler

When a task's status is changed to `"completed"` via the TaskUpdate tool:

```typescript
if (z !== X.status) {                          // if status is changing
  if (z === "completed") {                     // if changing TO completed
    let j = [],
        W = OQ1(A, X.subject, X.description,  // fire TaskCompleted hook
              B5(), Q3(), void 0,
              _?.abortController?.signal, void 0, _);
    for await (let G of W)
      if (G.blockingError) j.push($Q1(G.blockingError));
    if (j.length > 0)                          // if ANY blocking errors
      return {
        data: {
          success: false,                       // REJECT the completion
          taskId: A,
          updatedFields: [],
          error: j.join("\n"),
        },
      };
  }
  // Only reaches here if no blocking errors:
  M.status = z;                                // NOW persist the status change
  D.push("status");
}
if (Object.keys(M).length > 0) QC(J, A, M);   // QC() writes to disk
```

#### Trigger 2: Stop hook executor

When a teammate goes idle, TaskCompleted is fired for ALL of its in_progress tasks:

```typescript
let p = zX(g).filter((c) => c.status === "in_progress" && c.owner === m);
for (let c of p) {
  let Y1 = OQ1(c.id, c.subject, c.description, m, x, P, ...);
  // ... blocking errors collected into U[]
}
```

Note: This fires BEFORE TeammateIdle in the stop hook sequence.

### TC: Exit Code Behavior

| Exit Code | Behavior |
|---|---|
| **0** | Success -- task completion proceeds |
| **2** | **BLOCKING** -- prevents completion entirely |
| **other** | Non-blocking warning -- task still completes |

**Exit 2 mechanism (the veto):**

1. `OQ1()` yields `blockingError`
2. Error formatted by `$Q1()`:
   ```
   TaskCompleted hook feedback:
   [command]: stderr
   ```
3. **From TaskUpdate handler:** Returns `{ success: false, error: ... }` -- the task status **never changes**, `QC()` (disk write) **never executes**. No rollback needed because the change never happened.
4. **From stop hook:** Blocking error collected into `U[]`, which then feeds into the `EZ()` continuation (teammate gets another turn).

**Key insight:** The hook fires **BEFORE** the status change is persisted. `M.status = z` only executes if all hooks returned non-blocking. The disk write `QC()` only fires if mutations exist.

### TC: Unused Notification Infrastructure

Two functions exist but are **never called** anywhere in the codebase:

```typescript
function SWY(A, q, K) {
  return {
    type: "task_completed",
    from: A,
    taskId: q,
    taskSubject: K,
    timestamp: new Date().toISOString(),
  };
}

function hWY(A) {
  try {
    let q = MA(A);
    if (q && q.type === "task_completed") return q;
  } catch {}
  return null;
}
```

Both are **exported** but have **zero callers**. The team lead discovers task completion by polling the shared task list, NOT via inbox messages. This appears to be unused/deprecated infrastructure -- possibly planned but never wired up.

---

## Side-by-Side Comparison

| Aspect | TeammateIdle | TaskCompleted |
|---|---|---|
| **Schema var** | `QjY` | `UjY` |
| **Executor fn** | `KLA()` | `OQ1()` |
| **Error formatter** | `okA()` | `$Q1()` |
| **Trigger points** | 1 (stop hook only) | 2 (TaskUpdate + stop hook) |
| **Required fields** | `teammate_name`, `team_name` | `task_id`, `task_subject` |
| **Optional fields** | none | `task_description`, `teammate_name`, `team_name` |
| **Has matchers** | No | No |
| **Fires BEFORE state change** | Yes (before `isIdle = true`) | Yes (before `M.status = z`) |
| **Exit 2 effect** | Prevents idle, teammate gets another turn | Prevents completion, task stays in previous status |
| **Exit 2 feedback target** | Teammate only (team lead sees nothing) | Depends on trigger point |
| **Notification infrastructure** | `nm1()` / `zI4()` (active, used) | `SWY()` / `hWY()` (exported but never called) |

---

## Common Hook Infrastructure

### Generic Hook Executor: `sh()`

Both hooks delegate to `sh()`, which:
1. Checks `P8().disableAllHooks` -- returns immediately if hooks disabled
2. Gets the event name from `hookInput.hook_event_name`
3. Calls `ARY()` for matching (supports wildcard, pipe-separated, exact, regex)
4. Loads hook configurations from 6 settings locations (policy > local > project > user > session > plugin)
5. Spawns commands via `zW6()` using `child_process.spawn` with shell
6. Default timeout: `cj` (600,000ms = 10 minutes)
7. Parallel execution via `xO6` with unlimited concurrency
8. Validates output with Zod schema `Gdw = union(cjY, AWY)`

### Exit Code Interpretation

Common across all hooks:
- Exit 0: Success, no action needed
- Exit 2: Blocking -- yields `{ blockingError: "[cmd]: stderr" }`
- Any other exit: Non-blocking warning, logged but not blocking

### Permission Aggregation

When multiple hooks match:
- `deny` > `ask` > `allow` (most restrictive wins)
- Permission results aggregated across all matching hooks

### Base Schema `NZ` (shared by all 15 hook events)

Defined:

```typescript
NZ = b.object({
  session_id: b.string(),
  transcript_path: b.string(),
  cwd: b.string(),
  permission_mode: b.string().optional(),
})
```

Populated by `hX()` which reads:
- `U6()` for session ID
- `p$()` for transcript path
- `y6()` for current working directory
- Permission mode from the calling context

### Error Formatting Functions

Each hook event has a dedicated error formatter:

| Function | Format |
|---|---|
| `nkA(A, q)` | `"[A] hook error: [q.blockingError]"` |
| `rkA(A)` | `"Stop hook feedback:\n[A.blockingError]"` |
| `okA(A)` | `"TeammateIdle hook feedback:\n[A.blockingError]"` |
| `$Q1(A)` | `"TaskCompleted hook feedback:\n[A.blockingError]"` |
| `akA(A)` | `"UserPromptSubmit operation blocked by hook:\n[A.blockingError]"` |

---

## Key Architectural Insights

1. **Pre-state-change firing pattern:** Both TeammateIdle and TaskCompleted fire BEFORE their respective state changes are applied. This gives hooks full veto power without needing rollback logic.

2. **Sequential ordering in stop hooks:** TaskCompleted fires first (for all in_progress tasks), THEN TeammateIdle fires. Both contribute to the same blocking error array `U[]`.

3. **Asymmetric feedback routing:** TeammateIdle feedback goes to the teammate only (team lead is unaware). TaskCompleted feedback depends on trigger point -- from TaskUpdate it returns as a tool error, from stop hooks it goes to the teammate.

4. **No matcher support:** Neither hook supports matchers, meaning you cannot selectively filter which teammates or tasks trigger hooks. Every idle event and every completion event fires the hook.

5. **Dead code:** `SWY()` and `hWY()` represent task completion notification infrastructure that was built but never wired into the actual flow. The team lead polls for task completion rather than receiving push notifications.
