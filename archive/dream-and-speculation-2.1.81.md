# Dream Mode, Speculation System, and Deferred Prefetch
## Claude Code CLI v2.1.81 — Source Reference

> **Source:** All claims sourced exclusively from `/home/buzzkill/Projects/lab/cc-2.1.81/cli.js` (619,056 lines, 18.4 MB).
> Line references are provided for every function and constant cited.

---

## Table of Contents

- [Part 1: Dream Mode — Background Memory Consolidation](#part-1-dream-mode--background-memory-consolidation)
  - [1.1 Overview](#11-overview)
  - [1.2 Trigger Conditions — All Must Be Met](#12-trigger-conditions--all-must-be-met)
  - [1.3 Enablement Check — QR8() and $F_()](#13-enablement-check--qr8-and-f_)
  - [1.4 The 4-Phase Workflow Prompt — VYq()](#14-the-4-phase-workflow-prompt--vyq)
  - [1.5 The Main Execution Function — RYq() / LYq](#15-the-main-execution-function--ryq--lyq)
  - [1.6 Lock System — z5q() / Zd1() / MR8()](#16-lock-system--z5q--zd1--mr8)
  - [1.7 Bash Tool Constraints — jF_()](#17-bash-tool-constraints--jf_)
  - [1.8 Agent Fork — af() call](#18-agent-fork--af-call)
  - [1.9 Results and User Notification — iR8() / j5q()](#19-results-and-user-notification--ir8--j5q)
  - [1.10 Telemetry Events](#110-telemetry-events)
  - [1.11 Configuration — OF_() and yYq defaults](#111-configuration--of_-and-yyq-defaults)
  - [1.12 Force Trigger — z flag](#112-force-trigger--z-flag)
  - [1.13 Session Discovery — w5q()](#113-session-discovery--w5q)
  - [1.14 Rollback — MR8()](#114-rollback--mr8)
- [Part 2: Speculation System — Predictive Code Execution](#part-2-speculation-system--predictive-code-execution)
  - [2.1 Overview](#21-overview)
  - [2.2 Prompt Suggestion Generation — Sc1() and Cc1()](#22-prompt-suggestion-generation--sc1-and-cc1)
  - [2.3 Speculation Enablement — bc1()](#23-speculation-enablement--bc1)
  - [2.4 Speculation Activation — xc1()](#24-speculation-activation--xc1)
  - [2.5 Overlay Filesystem — _h8() and PE()](#25-overlay-filesystem--_h8-and-pe)
  - [2.6 Tool Permission Boundary Logic](#26-tool-permission-boundary-logic)
  - [2.7 Edit Boundary](#27-edit-boundary)
  - [2.8 File Path Redirection](#28-file-path-redirection)
  - [2.9 Bash Boundary](#29-bash-boundary)
  - [2.10 Denied Tool Boundary](#210-denied-tool-boundary)
  - [2.11 Allowed Tool Sets — eF_ and AU_](#211-allowed-tool-sets--ef_-and-au_)
  - [2.12 Message Validation — YU_()](#212-message-validation--yu_)
  - [2.13 Merge on Accept — OU_() and wzq()](#213-merge-on-accept--ou_-and-wzq)
  - [2.14 Abort on User Input — hx()](#214-abort-on-user-input--hx)
  - [2.15 Pipelined Speculation — wU_()](#215-pipelined-speculation--wu_)
  - [2.16 State Object — L16 and speculation shape](#216-state-object--l16-and-speculation-shape)
  - [2.17 Constants — sF_, tF_](#217-constants--sf_-tf_)
  - [2.18 Telemetry — Yh8()](#218-telemetry--yh8)
  - [2.19 Suggestion Guards — hc1() and Ic1()](#219-suggestion-guards--hc1-and-ic1)
- [Part 3: Deferred Prefetch System — Context Warming](#part-3-deferred-prefetch-system--context-warming)
  - [3.1 Overview](#31-overview)
  - [3.2 Entry Point — wm8()](#32-entry-point--wm8)
  - [3.3 Identity Prefetch — nWA()](#33-identity-prefetch--nwa)
  - [3.4 System Context Prefetch — GLY() and t2()](#34-system-context-prefetch--gly-and-t2)
  - [3.5 User Context Prefetch — Vz()](#35-user-context-prefetch--vz)
  - [3.6 Spinner Tips — gu8()](#36-spinner-tips--gu8)
  - [3.7 Bedrock Auth — DP1()](#37-bedrock-auth--dp1)
  - [3.8 Vertex Auth — XP1()](#38-vertex-auth--xp1)
  - [3.9 File Count Prefetch — \$j8()](#39-file-count-prefetch--j8)
  - [3.10 Analytics Gates — Ft1()](#310-analytics-gates--ft1)
  - [3.11 Models Prefetch — EC7()](#311-models-prefetch--ec7)
  - [3.12 Settings Watcher — wX.initialize()](#312-settings-watcher--wxinitialize)
  - [3.13 Skill/Command File Watcher — pV6 / FMY()](#313-skillcommand-file-watcher--pv6--fmy)
- [Part 4: How They Relate](#part-4-how-they-relate)
  - [4.1 Comparison Table](#41-comparison-table)
  - [4.2 Independence and Simultaneous Execution](#42-independence-and-simultaneous-execution)
  - [4.3 Shared Infrastructure — af() and querySource Labels](#43-shared-infrastructure--af-and-querysource-labels)
  - [4.4 Shared State — appState and speculation field](#44-shared-state--appstate-and-speculation-field)
- [Appendix A: Key Functions Decoded](#appendix-a-key-functions-decoded)
- [Appendix B: Feature Flags and Configuration](#appendix-b-feature-flags-and-configuration)
- [Appendix C: Constants and Magic Numbers](#appendix-c-constants-and-magic-numbers)

---

## Part 1: Dream Mode — Background Memory Consolidation

### 1.1 Overview

Dream mode is **automated memory maintenance**, not speculative execution. When triggered, it forks a background Claude agent that reviews recent JSONL session transcripts and consolidates useful information into persistent memory files (`MEMORY.md` and associated topic files in the memory directory).

Key properties:

- Runs as a background fork (via `af()`) with `skipTranscript: true` — it does **not** appear in the user's conversation history
- Completely invisible to the user unless it actually modifies memory files
- Identified with `querySource: "auto_dream"` and `forkLabel: "auto_dream"` in the API call
- The currently active session is excluded from review; only past sessions are scanned
- The agent is bash-restricted to read-only commands only (see §1.7)

The name "dream" is a metaphor: just as humans consolidate memories during sleep, Claude consolidates learnings between sessions in a passive background process.

### 1.2 Trigger Conditions — All Must Be Met

All of the following conditions must be satisfied before a dream fires. Checked inside `RYq()` / `LYq`, defined at **cli.js:452319–452414**:

**Condition 1 — Feature Enabled**
Either `autoDreamEnabled` user setting is `true`, OR the `tengu_onyx_plover` feature flag has `enabled: true`.
```js
// cli.js:451862–451864
function QR8() {
  let A = kA().autoDreamEnabled;
  if (A !== void 0) return A;
  return l8("tengu_onyx_plover", null)?.enabled === !0;
}
```

**Condition 2 — Not in Kairos mode**
`cv()` returns `false` (Kairos/headless mode). Kairos mode is detected by `T8.kairosActive` flag.
```js
// cli.js:2453–2455
function cv() {
  return T8.kairosActive;
}
```

**Condition 3 — Not in Dangerous/Remote mode**
`d4()` returns `false` (not in remote/dangerous mode). Remote mode is detected by `T8.isRemoteMode`.
```js
// cli.js:2743–2745
function d4() {
  return T8.isRemoteMode;
}
```

**Condition 4 — Auto-memory feature enabled — F5()**
The auto-memory system must be active. Respects `CLAUDE_CODE_DISABLE_AUTO_MEMORY`, `CLAUDE_CODE_SIMPLE`, `CLAUDE_CODE_REMOTE` env vars, and `autoMemoryEnabled` user setting.
```js
// cli.js:49596–49609
function F5() {
  let A = process.env.CLAUDE_CODE_DISABLE_AUTO_MEMORY;
  if (a6(A)) return !1;
  if (dY(A)) return !0;
  if (a6(process.env.CLAUDE_CODE_SIMPLE)) return !1;
  if (a6(process.env.CLAUDE_CODE_REMOTE) && !process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR)
    return !1;
  let q = kA();
  if (q.autoMemoryEnabled !== void 0) return q.autoMemoryEnabled;
  return !0;
}
```

**Condition 5 — 24+ hours since last consolidation**
Default: `minHours: 24`. Configurable via `tengu_onyx_plover.minHours`. Checked against mtime of the `.consolidate-lock` file via `JR8()`.
```js
// cli.js:452319
let O = (Date.now() - w) / 3600000;
if (!z && O < Y.minHours) return;
```

**Condition 6 — 5+ sessions since last consolidation**
Default: `minSessions: 5`. Configurable via `tengu_onyx_plover.minSessions`. Sessions are listed by scanning transcript mtimes newer than the last consolidation.
```js
// cli.js:452338–452343
if (((H = H.filter((P) => P !== j)), !z && H.length < Y.minSessions)) {
  V(`[autoDream] skip — ${H.length} sessions since last consolidation, need ${Y.minSessions}`);
  return;
}
```

**Condition 7 — 10-minute scan throttle**
The variable `wF_` (defined at **cli.js:452454**) is `600000` ms (10 minutes). Even when the time/session gate passes, the scan itself can only run once per 10 minutes:
```js
// cli.js:452323–452328
var wF_ = 600000,
let $ = Date.now() - A; // A = last scan timestamp
if (!z && $ < wF_) {
  V(`[autoDream] scan throttle — time-gate passed but last scan was ${Math.round($ / 1000)}s ago`);
  return;
}
```

**Condition 8 — Lock available**
The `.consolidate-lock` file must not be held by a live PID. See §1.6.

### 1.3 Enablement Check — QR8() and $F_()

Two functions govern the enable check:

`QR8()` (**cli.js:451862**) — checks `autoDreamEnabled` user setting, falls back to `tengu_onyx_plover.enabled`.

`$F_()` (**cli.js:452305**) — the full gate combining all mode checks:
```js
function $F_() {
  if (cv()) return !1;  // Kairos mode blocks it
  if (d4()) return !1;  // Remote/dangerous mode blocks it
  if (!F5()) return !1; // Auto-memory must be on
  return QR8();         // Feature flag/user setting
}
```

There is also `HF_()` (**cli.js:452307**) which always returns `!1` in this build — it represents a "force" path used when the `z` flag is set:
```js
function HF_() {
  return !1;
}
```

The `z` variable in `RYq`/`LYq` is the force override — when `true`, all time/session/scan gates are bypassed.

### 1.4 The 4-Phase Workflow Prompt — VYq()

The dream agent receives a carefully crafted system prompt generated by `VYq(A, q, K)` (**cli.js:452218**), where:
- `A` = memory directory path
- `q` = session transcripts directory path  
- `K` = optional additional context string (session list)

The prompt text defines four explicit phases:

**Phase 1 — Orient** (cli.js:452234)
- `ls` the memory directory to see what exists
- Read `MEMORY.md` to understand the current index
- Skim existing topic files to avoid creating duplicates
- If `logs/` or `sessions/` subdirectories exist (assistant-mode layout), review recent entries

**Phase 2 — Gather Recent Signal** (cli.js:452242)
Sources in priority order:
1. Daily logs (`logs/YYYY/MM/YYYY-MM-DD.md`) — the append-only stream
2. Existing memories that drifted — facts contradicting current codebase state
3. Transcript search — `grep -rn "<narrow term>" <transcripts>/ --include="*.jsonl" | tail -50`

The agent is explicitly instructed: "Don't exhaustively read transcripts. Look only for things you already suspect matter."

**Phase 3 — Consolidate** (cli.js:452252)
- Merge new signal into existing topic files rather than creating duplicates
- Convert relative dates to absolute dates
- Delete contradicted facts at the source
- Use the memory file format conventions from the system prompt's auto-memory section

**Phase 4 — Prune and Index** (cli.js:452261)
- Update `MEMORY.md` so it stays under a configurable line limit (`DH` constant)
- The index links to memory files with one-line descriptions — never writes content directly
- Remove stale/wrong/superseded pointers
- Demote verbose entries: keep the gist in the index, move details to topic files
- Resolve contradictions between files

The prompt concludes: "Return a brief summary of what you consolidated, updated, or pruned. If nothing changed (memories are already tight), say so."

If additional context `K` is provided (the session list), it is appended under an `## Additional context` section.

### 1.5 The Main Execution Function — RYq() / LYq

`RYq()` is the initializer (**cli.js:452309**) that assigns to the module-level `LYq` variable. `LYq` is then called by `hYq(A, q)` (**cli.js:452460**) which is the public trigger point.

Full flow within the trigger function:

```
1. Check z (force) OR $F_() (normal enable check)
2. Read last consolidation time via JR8() (lock file mtime)
3. Check hours elapsed against minHours
4. Check scan throttle (wF_ = 600000ms)
5. List sessions touched since last consolidation via w5q()
6. Filter out current session ID (E8())
7. Check session count against minSessions
8. Acquire lock via z5q()
9. Log + emit telemetry: tengu_auto_dream_fired
10. Build the prompt via VYq()
11. Fork via af() with querySource: "auto_dream"
12. On success: notify user if files modified, emit telemetry: tengu_auto_dream_completed
13. On abort: log "aborted by user"
14. On error: log, emit tengu_auto_dream_failed, rollback via MR8()
```

The `LYq` variable is initialized to `null` (cli.js:452473) and only becomes callable after `RYq()` runs:
```js
// cli.js:452472–452473
var wF_ = 600000,
  yYq,
  LYq = null;
```

### 1.6 Lock System — z5q() / Zd1() / MR8()

The lock system prevents parallel dream executions.

**Lock file location — Zd1()** (cli.js:442388):
```js
function Zd1() {
  return fB_(aw(), TB_);  // TB_ = ".consolidate-lock"
}
```
Where `aw()` returns the memory directory path and `TB_` is `".consolidate-lock"` (cli.js:442447).

**Lock file constant** (cli.js:442447):
```js
var TB_ = ".consolidate-lock",
  kB_ = 3600000; // 1 hour expiry for stale locks
```

**Read last consolidation time — JR8()** (cli.js:442391):
```js
async function JR8() {
  try {
    return (await Y5q(Zd1())).mtimeMs;
  } catch {
    return 0;
  }
}
```
Returns `0` (epoch) if the lock file does not exist, meaning consolidation has never run.

**Acquire lock — z5q()** (cli.js:442398):
Checks if the lock file exists and is newer than `kB_` (1 hour = 3,600,000ms). If a lock is held by a **live PID** (verified via `JJ6(K)`) and the file is recent, acquisition fails and returns `null`. Otherwise it writes the current `process.pid` to the lock file and verifies the write succeeded (race condition protection):
```js
// cli.js:442408–442412 (approximate)
// Lock held by live PID check:
if (q !== void 0 && Date.now() - q < kB_) {
  if (K !== void 0 && JJ6(K))
    return (V(`[autoDream] lock held by live PID ${K} (mtime ${Math.round((Date.now() - q) / 1000)}s ago)`), null);
}
// Write lock:
await _5q(A, String(process.pid));
// Verify:
if (parseInt(_.trim(), 10) !== process.pid) return null;
```

**Rollback — MR8(A)** (cli.js:442427):
On failure, the lock is either deleted (if `A === 0`, first attempt) or its mtime is set to a past timestamp (`A / 1000` seconds ago) to force the next trigger to be delayed by `minHours`:
```js
async function MR8(A) {
  let q = Zd1();
  try {
    if (A === 0) {
      await GB_(q);  // delete the lock
      return;
    }
    await _5q(q, "");  // clear content
    let K = A / 1000;
    await ZB_(q, K, K);  // set mtime to past
  } catch (K) {
    V(`[autoDream] rollback failed: ${K.message} — next trigger delayed to minHours`);
  }
}
```

### 1.7 Bash Tool Constraints — jF_()

The dream agent's bash access is enforced by `jF_(A)` (**cli.js:452467**):

```js
function jF_(A) {
  let q = cR8(A);  // base permission checker
  return async (K, _, Y, z, w, O) => {
    if (K.name === S7) {  // S7 = "Bash"
      let $ = K.inputSchema.safeParse(_);
      if ($.success && K.isReadOnly($.data))
        return { behavior: "allow", updatedInput: _ };
      let H = "Only read-only shell commands are permitted in this context (ls, find, grep, cat, stat, wc, head, tail, and similar)";
      return { behavior: "deny", message: H, decisionReason: { type: "other", reason: H } };
    }
    return q(K, _, Y, z, w, O);
  };
}
```

Only commands that pass `K.isReadOnly()` are permitted. This restricts the dream agent to: `ls`, `find`, `grep`, `cat`, `stat`, `wc`, `head`, `tail`, and similar read-only operations. Any command that writes, redirects to a file, or modifies state is denied.

The full list from the prompt text (cli.js:452373): **"ls, find, grep, cat, stat, wc, head, tail, and similar"**.

### 1.8 Agent Fork — af() call

The dream agent is launched via `af()` with these parameters (cli.js:452385–452395):

```js
let G = await af({
  promptMessages: [F8({ content: Z })],  // Z = VYq() prompt
  cacheSafeParams: qy(K),
  canUseTool: jF_(P),                    // read-only bash enforcement
  querySource: "auto_dream",
  forkLabel: "auto_dream",
  skipTranscript: !0,                    // invisible to user
  overrides: { abortController: X },
  onMessage: JF_(D, M),                  // tracks written files
});
```

Key parameters:
- `skipTranscript: true` — the dream conversation never appears in the user's session history
- `querySource: "auto_dream"` — tags API calls for monitoring/billing
- `forkLabel: "auto_dream"` — labels the fork in debug logs
- `abortController` — can be aborted if the user cancels

**Message handler — JF_(D, M)** (cli.js:452473): Tracks all assistant messages, counting tool uses and collecting written file paths (from `Edit`/`Write` tool inputs) to build the `filesTouched` list.

### 1.9 Results and User Notification — iR8() / j5q()

After the dream fork completes (cli.js:452395–452402):

```js
let v = K.toolUseContext.getAppState().tasks?.[D];
if (_ && O5q(v) && v.filesTouched.length > 0)
  _({ ...iR8(v.filesTouched), verb: "Improved" });
```

If `_` (the notification callback) is provided, the task has `type: "dream"` (checked by `O5q(v)`), and at least one file was written, the user receives a system notification.

**`iR8(A)`** (cli.js:547681) constructs the notification message object:
```js
function iR8(A) {
  return {
    type: "system",
    subtype: "memory_saved",
    writtenPaths: A,
    timestamp: new Date().toISOString(),
    uuid: fN(),
    isMeta: !1,
  };
}
```

The `verb: "Improved"` field is added at the call site, producing a message like "Improved" with the list of written memory file paths.

**If nothing changed:** completely invisible. No notification, no message. The only trace is the lock file mtime being updated.

### 1.10 Telemetry Events

All telemetry emitted via `Q()` (the analytics logger):

**tengu_auto_dream_fired** (cli.js:452358–452362):
```js
Q("tengu_auto_dream_fired", {
  hours_since: Math.round(O),   // hours since last consolidation
  sessions_since: H.length,     // number of sessions reviewed
});
```

**tengu_auto_dream_completed** (cli.js:452398–452404):
```js
Q("tengu_auto_dream_completed", {
  cache_read: G.totalUsage.cache_read_input_tokens,
  cache_created: G.totalUsage.cache_creation_input_tokens,
  output: G.totalUsage.output_tokens,
  sessions_reviewed: H.length,
});
```

**tengu_auto_dream_failed** (cli.js:452410):
```js
Q("tengu_auto_dream_failed", {});
```

**tengu_auto_dream_toggled** (cli.js:487143–487145):
```js
Q("tengu_auto_dream_toggled", { enabled: N6 });
```
Emitted when the user toggles the `autoDreamEnabled` setting.

### 1.11 Configuration — OF_() and yYq defaults

**Default values — yYq** (cli.js:452472):
```js
yYq = { minHours: 24, minSessions: 5 };
```

**Config resolver — OF_()** (cli.js:452281): Reads the `tengu_onyx_plover` feature flag and validates overrides:
```js
function OF_() {
  let A = l8("tengu_onyx_plover", null);
  return {
    minHours:
      typeof A?.minHours === "number" && Number.isFinite(A.minHours) && A.minHours > 0
        ? A.minHours
        : yYq.minHours,   // fallback: 24
    minSessions:
      typeof A?.minSessions === "number" && Number.isFinite(A.minSessions) && A.minSessions > 0
        ? A.minSessions
        : yYq.minSessions, // fallback: 5
  };
}
```

**User setting — autoDreamEnabled** (cli.js:446721):
```js
autoDreamEnabled: {
  source: "settings",
  type: "boolean",
  description: "Enable background memory consolidation",
}
```
This is a settings-sourced boolean, meaning it is stored per-user and loaded via `kA()`.

### 1.12 Force Trigger — z flag

The `z` variable in `RYq()`/`LYq` is a force flag. When `true`, all gates (time, session count, scan throttle) are bypassed. Only the lock is still acquired:

```js
// cli.js:452319
let Y = OF_(),
  z = HF_();  // HF_() always returns false in production build
if (!z && !$F_()) return;   // normal enable check, skipped on force
// ...
if (!z && O < Y.minHours) return;   // time gate, skipped on force
if (!z && $ < wF_) { /* throttle */ return; }   // throttle, skipped on force
if (!z && H.length < Y.minSessions) return;     // session count, skipped on force
if (z) J = w;  // force uses prior mtime instead of acquiring lock
```

In the production build, `HF_()` always returns `false` (cli.js:452307), so the force path is never activated normally. It appears to be a testing/debugging escape hatch.

### 1.13 Session Discovery — w5q()

`w5q(A)` (**cli.js:442448**) lists all session IDs that have been active since timestamp `A`:

```js
async function w5q(A) {
  let q = IO(l1());
  return (await A5q(q, !0)).filter((_) => _.mtime > A).map((_) => _.sessionId);
}
```

Where `IO(l1())` computes the transcripts directory from the config directory (`l1()` returns `~/.claude`), and `A5q()` scans the JSONL transcript index. The result is filtered to only include sessions newer than the last consolidation timestamp and mapped to session IDs.

The current session ID (`E8()`) is always excluded from this list:
```js
// cli.js:452337
H = H.filter((P) => P !== j);  // j = E8() = current session id
```

### 1.14 Rollback — MR8()

On any failure in the dream fork (cli.js:452409–452412):
```js
(V(`[autoDream] fork failed: ${P.message}`),
  Q("tengu_auto_dream_failed", {}),
  J5q(D, M),      // clean up the task state
  await MR8(J));  // J = lock acquisition time (0 if using force path)
```

If `J === 0`, the lock file is deleted entirely. Otherwise the lock file content is cleared and its mtime is set back to `J / 1000` seconds ago, which effectively pushes the next trigger window to `minHours` from that timestamp.

---

## Part 2: Speculation System — Predictive Code Execution

### 2.1 Overview

The speculation system runs **full Claude inference ahead of user confirmation**. When the user is about to accept a prompt suggestion (a predicted next message), speculation starts a complete conversation inference in an isolated overlay filesystem. If the user accepts the suggestion, the speculated work is merged into the real conversation — saving an entire round-trip.

Key properties:
- Uses an isolated **overlay directory** at `PE()/speculation/{pid}/{unique-id}` to redirect writes
- File reads check the overlay first; if a speculated version exists, it is served instead of the real file
- Stops at defined boundaries: file edits (unless in acceptEdits mode), bash commands, denied tools, or natural completion
- Tracks `timeSavedMs` as the primary user-facing metric
- Invisible to the user until accepted
- Identified with `querySource: "speculation"` and `forkLabel: "speculation"`

### 2.2 Prompt Suggestion Generation — Sc1() and Cc1()

Before speculation can begin, a **prompt suggestion** must be generated. This happens after every assistant response.

**Sc1(A, q, K, _, Y)** — main suggestion driver (**cli.js:454110**):

Checks several skip conditions before calling `Cc1()`:
- `A.signal.aborted` — abort in progress
- Fewer than 2 assistant messages — too early in conversation (`early_conversation`)
- Last response was an API error (`last_response_error`)
- Cache is too cold — `VF_()` checks if >threshold% of tokens were cache creation (`cache_cold`)
- `hc1(O)` returns a non-null skip reason (see §2.19)

If all checks pass:
```js
let H = eR8(),  // H = "user_intent" (the prompt ID)
  { suggestion: j, generationRequestId: J } = await Cc1(A, H, _);
```

**Cc1(A, q, K)** — the actual suggestion inference (**cli.js:454173**):

```js
async function Cc1(A, q, K) {
  let _ = EF_[q],  // EF_["user_intent"] = the suggestion prompt template
    Y = async () => ({ behavior: "deny", message: "No tools needed for suggestion", ... }),
    z = await af({
      promptMessages: [F8({ content: _ })],
      cacheSafeParams: K,
      canUseTool: Y,               // no tools allowed during suggestion
      querySource: "prompt_suggestion",
      forkLabel: "prompt_suggestion",
      skipTranscript: !0,
      skipCacheWrite: !0,          // suggestion doesn't write to cache
    });
  // Extract first non-empty text from assistant messages
  for (let $ of z.messages) {
    if ($.type !== "assistant") continue;
    let H = $.message.content.find((j) => j.type === "text");
    if (H?.type === "text" && H.text.trim())
      return { suggestion: H.text.trim(), generationRequestId: O };
  }
  return { suggestion: null, generationRequestId: O };
}
```

The suggestion is a text prediction of what the user will type next. It is generated from a prompt template keyed by prompt ID `"user_intent"`.

**Post-suggestion trigger** (cli.js:454147):
```js
if (bc1() && _.suggestion)
  xc1(_.suggestion, A, A.toolUseContext.setAppState, !1, K);
```
If speculation is enabled (`bc1()`) and a suggestion was generated, speculation immediately starts.

### 2.3 Speculation Enablement — bc1()

**bc1()** (**cli.js:456031**):
```js
function bc1() {
  return (V("[Speculation] enabled=false"), !1);
}
```

In this build (v2.1.81), `bc1()` unconditionally returns `false` — speculation is **disabled**. The log message `"[Speculation] enabled=false"` is emitted every time it is checked. This indicates the feature exists in the codebase but is gated off in production. In builds where it is enabled, this function would check a feature flag such as `tengu_chomp_inflection` or similar.

### 2.4 Speculation Activation — xc1()

**xc1(A, q, K, _ = false, Y)** (**cli.js:456084**) — the main speculation launcher:

```js
async function xc1(A, q, K, _ = !1, Y) {
  if (!bc1()) return;  // guard: returns immediately in current build
  hx(K);               // abort any running speculation
  let z = iF_().slice(0, 8),  // z = unique 8-char ID
    w = jg(q.toolUseContext.abortController),  // abort signal
    ...
    j = _h8(z),  // j = overlay directory path
    J = eS();    // J = current working directory
```

Parameters:
- `A` — the suggestion text (predicted user input)
- `q` — current conversation context
- `K` — `setAppState` function (to update speculation status)
- `_` — `isPipelined` flag (true when called from pipelined chain)
- `Y` — optional cache params

### 2.5 Overlay Filesystem — _h8() and PE()

**PE()** — the base temp directory (**cli.js:543583**):
```js
PE = z1(function () {
  let q = process.env.CLAUDE_CODE_TMPDIR || (E1() === "windows" ? yHY() : "/tmp"),
    K = w8(),
    _ = q;
  try { _ = K.realpathSync(q); } catch {}
  return PN(_, qE1()) + LW;
});
```
Returns `$CLAUDE_CODE_TMPDIR` → `/tmp/<session>` (Unix) or Windows temp. The result is memoized.

**_h8(A)** — overlay path constructor (**cli.js:455959**):
```js
function _h8(A) {
  return Nw6(PE(), "speculation", String(process.pid), A);
}
```
Constructs: `PE()/speculation/<pid>/<unique-id>/`

Example: `/tmp/claude-xyz/speculation/12345/a1b2c3d4/`

**Directory creation** (cli.js:456093):
```js
try {
  await gc1(j, { recursive: !0 });  // mkdir -p overlay dir
} catch {
  V("[Speculation] Failed to create overlay directory");
  return;
}
```

**Cleanup** (via `qa6(J)`) on abort/complete:
```js
function qa6(A) {
  nF_(A, { recursive: !0, force: !0, maxRetries: 3, retryDelay: 100 }, () => {});
}
```
Removes the overlay directory recursively with retries.

### 2.6 Tool Permission Boundary Logic

The speculation fork uses a custom `canUseTool` function that intercepts all tool calls and either:
1. Redirects them to the overlay filesystem
2. Sets a boundary and aborts the speculation
3. Allows them to proceed normally

The decision is based on tool classification:

- **Write tools** (`eF_` = `{"Edit", "Write", "NotebookEdit"}`): check if in permissive mode; if not, set `"edit"` boundary and abort
- **Read tools** (`AU_` = `{"Read", "Glob", "Grep", "ToolSearch", "LSP", "TaskGet", "TaskList"}`): allow, redirect if overlay has the file
- **Bash**: allow only if read-only commands; otherwise set `"bash"` boundary and abort
- **Other tools**: set `"denied_tool"` boundary and abort

### 2.7 Edit Boundary

When a write tool is used and the mode is not `acceptEdits`/`bypassPermissions`/`plan+bypass` (cli.js:456015):
```js
if (!(
  v === "acceptEdits" ||
  v === "bypassPermissions" ||
  (v === "plan" && k)
)) {
  V(`[Speculation] Stopping at file edit: ${D.name}`);
  let E = "file_path" in P ? P.file_path : void 0;
  return (
    ik6(K, () => ({
      boundary: {
        type: "edit",
        toolName: D.name,
        filePath: E ?? "",
        completedAt: Date.now(),
      },
    })),
    w.abort(),
    Kh8("Speculation paused: file edit requires permission", "speculation_edit_boundary")
  );
}
```
The speculation is paused — not failed — allowing later merge when the user accepts.

### 2.8 File Path Redirection

When a write tool runs in overlay mode (in acceptEdits/bypassPermissions mode), paths are remapped to the overlay directory (cli.js:456045–456065):

```js
if (W) {  // W = is write tool
  if (!H.current.has(k)) {
    let N = Nw6(j, k);  // j = overlay dir, k = relative path
    await gc1(zzq(N), { recursive: !0 });  // create parent dirs
    try { await Yzq(Nw6(J, k), N); } catch {}  // copy original if exists
    H.current.add(k);  // mark as in overlay
  }
  P = { ...P, [G]: Nw6(j, k) };  // rewrite path to overlay
} else if (H.current.has(k)) {
  P = { ...P, [G]: Nw6(j, k) };  // read from overlay if present
}
```

This creates a copy-on-write overlay: reads go to the real filesystem unless the file has been written in speculation (in which case the overlay version is read).

**Paths outside cwd are handled separately:**
- Writes outside cwd: denied with `"speculation_write_outside_root"`
- Reads outside cwd: allowed through to real filesystem with reason `"speculation_read_outside_root"`

### 2.9 Bash Boundary

Bash commands are validated against a read-only allowlist (cli.js:456095):

```js
if (D.name === "Bash") {
  let G = "command" in P && typeof P.command === "string" ? P.command : "";
  if (!G || qh8({ command: G }, wd6(G)).behavior !== "allow")
    return (
      V(`[Speculation] Stopping at bash: ${G.slice(0, 50) || "missing command"}`),
      ik6(K, () => ({ boundary: { type: "bash", command: G, completedAt: Date.now() } })),
      w.abort(),
      Kh8("Speculation paused: bash boundary", "speculation_bash_boundary")
    );
  // read-only bash passes through:
  return { behavior: "allow", updatedInput: P, decisionReason: { type: "other", reason: "speculation_readonly_bash" } };
}
```

### 2.10 Denied Tool Boundary

Any tool that is neither in `eF_` (write tools), `AU_` (read tools), nor `Bash` is denied:

```js
V(`[Speculation] Stopping at denied tool: ${D.name}`);
let Z = String(("url" in P && P.url) || ("file_path" in P && P.file_path) || ...).slice(0, 200);
return (
  ik6(K, () => ({
    boundary: { type: "denied_tool", toolName: D.name, detail: Z, completedAt: Date.now() },
  })),
  w.abort(),
  Kh8(`Tool ${D.name} not allowed during speculation`, "speculation_unknown_tool")
);
```

### 2.11 Allowed Tool Sets — eF_ and AU_

Defined at **cli.js:456533–456543**:

```js
eF_ = new Set(["Edit", "Write", "NotebookEdit"]),  // write tools
AU_ = new Set([
  "Read",
  "Glob",
  "Grep",
  "ToolSearch",
  "LSP",
  "TaskGet",
  "TaskList",
]);
```

`eF_` = **write-capable tools** — checked against edit boundary logic
`AU_` = **read-only tools** — allowed to run with optional overlay redirect

Any tool outside both sets triggers the `denied_tool` boundary.

### 2.12 Message Validation — YU_()

**YU_(A)** (**cli.js:456031**) filters speculated messages before merging:

```js
function YU_(A) {
  // 1. Identify valid tool results (non-error, non-redacted)
  let _ = new Set(
    A.filter(Uc1)
      .flatMap((z) => z.message.content)
      .filter(isToolResult)
      .filter(isNonError)         // !is_error && content doesn't include redaction marker
      .map((z) => z.tool_use_id)
  );
  // 2. Filter predicate: keep only valid content
  let Y = (z) =>
    z.type !== "thinking" &&
    z.type !== "redacted_thinking" &&
    !(z.type === "tool_use" && !_.has(z.id)) &&       // drop dangling tool_uses
    !(z.type === "tool_result" && !_.has(z.tool_use_id)) &&  // drop orphaned results
    !(z.type === "text" && (z.text === Ci || z.text === nD)); // drop placeholder text
  // 3. Filter messages
  return A.map((z) => {
    if (!("message" in z) || !Array.isArray(z.message.content)) return z;
    let w = z.message.content.filter(Y);
    if (w.length === z.message.content.length) return z;
    if (w.length === 0) return null;
    if (!w.some($ => $.type !== "text" || ($.text !== void 0 && $.text.trim() !== "")))
      return null;
    return { ...z, message: { ...z.message, content: w } };
  }).filter((z) => z !== null);
}
```

This function:
- Removes `thinking`/`redacted_thinking` content blocks
- Removes `tool_use` blocks whose results never came back (boundary hit)
- Removes `tool_result` blocks that are orphaned
- Removes messages whose content is entirely empty after filtering

### 2.13 Merge on Accept — OU_() and wzq()

**OU_(A, q, K)** (**cli.js:456337**) — copy overlay files and reset speculation state:

```js
async function OU_(A, q, K) {
  if (A.status !== "active") return null;
  let { id: _, messagesRef: Y, writtenPathsRef: z, abort: w, startTime: O, ... } = A,
    j = Y.current,
    J = _h8(_),  // overlay dir
    M = Date.now();
  if ((w(), K > 0)) await qU_(J, z.current, eS());  // copy files if any written
  qa6(J);  // cleanup overlay directory
  let X = A.boundary,
    D = Math.min(M, X?.completedAt ?? 1/0) - O;  // time saved
  // Reset speculation state and accumulate timeSaved:
  q((P) => { return { ...P, speculation: L16, speculationSessionTimeSavedMs: P.speculationSessionTimeSavedMs + D }; });
  // Write speculation-accept to transcript:
  let P = { type: "speculation-accept", timestamp: ..., timeSavedMs: D };
  rF_(WY(), x6(P) + "\n", { mode: 384 });
  return { messages: j, boundary: X, timeSavedMs: D };
}
```

**qU_(J, paths, cwd)** copies each written path from overlay to real filesystem:
```js
async function qU_(A, q, K) {
  let _ = !0;
  for (let Y of q) {
    let z = Nw6(A, Y),   // overlay path
      w = Nw6(K, Y);     // real path
    try {
      (await gc1(zzq(w), { recursive: !0 }), await Yzq(z, w));
    } catch {
      ((_ = !1), V(`[Speculation] Failed to copy ${Y} to main`));
    }
  }
  return _;
}
```

**wzq(A, q, K, _, Y)** (**cli.js:456439**) — the full accept flow called when user accepts a suggestion:

```js
let $ = A.messagesRef.current,
  H = YU_($),              // validate messages
  j = F8({ content: _ });  // wrap user message
z((f) => [...f, j]);        // add user message to conversation
let J = await OU_(A, K, H.length),  // merge overlay, get time saved
  M = J?.boundary?.type === "complete";  // was speculation fully complete?
if (!M) {
  // If not complete, trim to last non-assistant message
  let f = H.findLastIndex((Z) => Z.type !== "assistant");
  H = H.slice(0, f + 1);
}
let X = J?.timeSavedMs ?? 0,
  D = q + X,            // cumulative time saved
  P = zU_(H, J?.boundary ?? null, X, D);  // build status message (returns null in this build)
z((f) => [...f, ...H]);
// Update file read state from speculated files
let W = lk6(H, O, ac);
if (((w.current = XW8(w.current, W)), P)) z((f) => [...f, P]);
```

If `boundary.type === "complete"` (speculation finished naturally), all messages are used and a **pipelined speculation** for the next turn starts immediately.

### 2.14 Abort on User Input — hx()

**hx(A)** (**cli.js:456397**) — called when user starts typing:

```js
function hx(A) {
  A((q) => {
    if (q.speculation.status !== "active") return q;
    let { id: K, abort: _, startTime: Y, boundary: z, suggestionLength: w, messagesRef: O, isPipelined: $ } = q.speculation;
    return (
      V(`[Speculation] Aborting ${K}`),
      Yh8(K, "aborted", Y, w, O.current, z, { abort_reason: "user_typed", is_pipelined: $ }),
      _(),                // abort the inference
      qa6(_h8(K)),        // cleanup overlay directory
      { ...q, speculation: L16 }  // reset state
    );
  });
}
```

This ensures that any running speculation is cleanly terminated when the user begins typing something different.

### 2.15 Pipelined Speculation — wU_()

**Pipelined speculation** creates a continuous chain of speculations. After a `"complete"` boundary is hit and the user accepts, the system immediately starts speculating on the next turn.

**wU_(A, q, K, _, Y)** (**cli.js:456036**) — generates the next pipelined suggestion:

```js
async function wU_(A, q, K, _, Y) {
  try {
    let z = A.toolUseContext.getAppState(),
      w = hc1(z);      // check skip conditions
    if (w) { WW(`pipeline_${w}`); return; }
    let O = { ...A, messages: [...A.messages, F8({ content: q }), ...K] },
      $ = jg(Y);
    if ($.signal.aborted) return;
    let H = eR8(),  // H = "user_intent"
      { suggestion: j, generationRequestId: J } = await Cc1($, H, qy(O));
    if ($.signal.aborted) return;
    if (Ic1(j, H)) return;
    (V(`[Speculation] Pipelined suggestion: "${j.slice(0, 50)}..."`),
      ik6(_, () => ({ pipelinedSuggestion: { text: j, promptId: H, generationRequestId: J } })));
  } catch (z) { ... }
}
```

The `pipelinedSuggestion` field is stored on the current speculation object. When the current speculation is accepted and has a `"complete"` boundary (cli.js:456499):

```js
if (M && A.pipelinedSuggestion) {
  let { text: f, promptId: Z, generationRequestId: G } = A.pipelinedSuggestion;
  V(`[Speculation] Promoting pipelined suggestion: "${f.slice(0, 50)}..."`);
  // Show the suggestion in the UI
  K((k) => ({ ...k, promptSuggestion: { text: f, promptId: Z, ... } }));
  // Start the next speculation with updated context
  let v = { ...A.contextRef.current, messages: [...A.contextRef.current.messages, ...] };
  xc1(f, v, K, !0);  // isPipelined = true
}
```

The `isPipelined: true` flag is tracked in telemetry to measure chained speculation performance.

### 2.16 State Object — L16 and speculation shape

**L16** — the idle/reset state (cli.js:454446):
```js
L16 = { status: "idle" };
```

**Active speculation state** (set at cli.js:456115):
```js
{
  status: "active",
  id: z,                    // unique 8-char ID
  abort: () => w.abort(),   // abort function
  startTime: O,             // Date.now() when started
  messagesRef: $,           // { current: [] } — live message accumulator
  writtenPathsRef: H,       // { current: new Set() } — overlay written paths
  boundary: null,           // set when a boundary is hit
  suggestionLength: A.length,
  toolUseCount: 0,
  isPipelined: _,           // from isPipelined parameter
  contextRef: M,            // { current: q } — context snapshot
}
```

The `appState` initial value (cli.js:454421):
```js
speculation: L16,
speculationSessionTimeSavedMs: 0,
```

### 2.17 Constants — sF_, tF_

Defined at **cli.js:456509**:
```js
var sF_ = 20,   // maxTurns: speculation limited to 20 turns
  tF_ = 100,   // max messages before abort (speculated conversation)
  eF_,         // write tool set (defined at 456533)
  AU_;         // read tool set (defined at 456534)
```

The `tF_ = 100` guard aborts the speculation if the message count reaches 100 (cli.js:456303):
```js
if (($.current.push(D), $.current.length >= tF_)) w.abort();
```

### 2.18 Telemetry — Yh8()

**Yh8()** (**cli.js:455982**) emits `tengu_speculation`:

```js
function Yh8(A, q, K, _, Y, z, w) {
  Q("tengu_speculation", {
    speculation_id: A,
    outcome: q,              // "accepted" | "aborted" | "error"
    duration_ms: Date.now() - K,
    suggestion_length: _,
    tools_executed: Fc1(Y),  // count of successful tool calls
    completed: z !== null,   // whether a boundary was reached
    boundary_type: z?.type,  // "edit" | "bash" | "denied_tool" | "complete"
    boundary_tool: KU_(z),   // tool name at boundary
    boundary_detail: _U_(z), // detail string
    ...w,                    // additional fields from call site
  });
}
```

Additional fields included at call sites:
- On accept: `message_count`, `time_saved_ms`, `is_pipelined`
- On abort: `abort_reason: "user_typed"`, `is_pipelined`
- On error: `error_type`, `error_message`, `error_phase`, `is_pipelined`

**`speculation-accept` transcript record** (written when `timeSavedMs > 0`):
```js
{ type: "speculation-accept", timestamp: "...", timeSavedMs: D }
```
This entry is appended to the JSONL transcript for session-level time savings tracking. At session start, accumulated time savings are totaled:
```js
// cli.js:533087
if (d.type === "speculation-accept") j += d.timeSavedMs;
```

### 2.19 Suggestion Guards — hc1() and Ic1()

**hc1(A)** (**cli.js:454101**) — returns skip reason string or null:
```js
function hc1(A) {
  if (!A.promptSuggestionEnabled) return "disabled";
  if (A.pendingWorkerRequest || A.pendingSandboxRequest) return "pending_permission";
  if (A.elicitation.queue.length > 0) return "elicitation_active";
  if (A.toolPermissionContext.mode === "plan") return "plan_mode";
  if (_v.status !== "allowed") return "rate_limit";
  return null;
}
```

**Ah8()** (**cli.js:454059**) — initialization check for prompt suggestion feature:
Checks `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` env var, `tengu_chomp_inflection` feature flag, non-interactive mode, swarm teammate mode, and `promptSuggestionEnabled` user setting.

---

## Part 3: Deferred Prefetch System — Context Warming

### 3.1 Overview

The deferred prefetch system pre-warms data that will be needed for the first conversation turn. It runs once after the first render completes, launching multiple parallel async operations that populate caches before the user submits their first message.

Key properties:
- All prefetches are fire-and-forget (not awaited by the caller)
- Runs in parallel via independent promise chains
- Completely invisible to the user (no UI state change)
- Skipped if `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` is set or in certain modes
- Entry point: `wm8()` called from the render pipeline (cli.js:598632, cli.js:617604)

### 3.2 Entry Point — wm8()

**wm8()** (**cli.js:616199**) — orchestrates all prefetches:

```js
function wm8() {
  if (a6(process.env.CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER) || zY()) return;
  if (
    (nWA(),            // 1. identity prefetch
    Vz(),              // 2. user context (CLAUDE.md)
    GLY(),             // 3. system context (git status)
    gu8(),             // 4. spinner tips
    a6(process.env.CLAUDE_CODE_USE_BEDROCK) &&
      !a6(process.env.CLAUDE_CODE_SKIP_BEDROCK_AUTH))
  )
    DP1();             // 5. bedrock auth (conditional)
  if (
    a6(process.env.CLAUDE_CODE_USE_VERTEX) &&
    !a6(process.env.CLAUDE_CODE_SKIP_VERTEX_AUTH)
  )
    XP1();             // 6. vertex auth (conditional)
  if (
    ($j8(G8(), AbortSignal.timeout(3000), []),  // 7. file count (3s timeout)
    Ft1(),             // 8. analytics gates
    EC7(),             // 9. model list
    wX.initialize(),   // 10. settings watcher
    !zY())
  )
    pV6.initialize();  // 11. skill/command watcher (non-headless only)
}
```

This function is registered as `startDeferredPrefetches` (cli.js:616117):
```js
N8(hdq, { startDeferredPrefetches: () => wm8, main: () => VLY });
```

`wm8()` is called in two places:
- After first render completes (cli.js:598632): `(A.render(q), wm8(), await A.waitUntilExit(), ...)`
- On state initialization (cli.js:617604): in the app startup path

### 3.3 Identity Prefetch — nWA()

**nWA()** (**cli.js:52009**) — pre-warms the authenticated user's email/identity:

```js
async function nWA() {
  if (ES6 === null && !VS6)
    ((VS6 = M0K()), (ES6 = await VS6), (VS6 = null), YQ.cache.clear?.());
}
```

Where `M0K()` fetches the email from the authentication context (`x3()?.emailAddress`). The result is cached in `ES6`. The `YQ.cache.clear?.()` call invalidates dependent memoized queries so they pick up the fresh identity.

### 3.4 System Context Prefetch — GLY() and t2()

**GLY()** (**cli.js:616191**) — conditional launcher for `t2()`:

```js
function GLY() {
  if (K7()) {  // non-interactive mode
    (o8("info", "prefetch_system_context_non_interactive"), t2());
    return;
  }
  if (aY()) (o8("info", "prefetch_system_context_has_trust"), t2());
  else o8("info", "prefetch_system_context_skipped_no_trust");
}
```

System context is prefetched in non-interactive mode (`K7()`) or when the user has established trust (`aY()`). Otherwise it is skipped.

**t2()** (**cli.js:264280**) — the actual system context fetcher (memoized via `z1`):

```js
t2 = z1(async () => {
  let A = Date.now();
  o8("info", "system_context_started");
  let q = a6(process.env.CLAUDE_CODE_REMOTE) || !kW8() ? null : await nE1(),  // git status
    K = null;
  return (
    o8("info", "system_context_completed", {
      duration_ms: Date.now() - A,
      has_git_status: q !== null,
      has_injection: K !== null,
    }),
    { ...(q ? { gitStatus: q } : {}), ...{} }
  );
});
```

Key behavior:
- **Memoized** via `z1` wrapper — runs exactly once per session
- Fetches git status via `nE1()` unless in remote mode or filesystem is inaccessible
- Returns `{ gitStatus: q }` or `{}` if no git status
- Telemetry: `system_context_started` and `system_context_completed` with `duration_ms`, `has_git_status`, `has_injection`

### 3.5 User Context Prefetch — Vz()

**Vz()** (**cli.js:264293**) — the user context fetcher (memoized via `z1`):

```js
Vz = z1(async () => {
  let A = Date.now();
  o8("info", "user_context_started");
  let q =
      process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS ||
      (zY() && UW().length === 0),  // headless with no project dirs
    K = q ? null : FE1(await bO());  // bO() reads CLAUDE.md files, FE1() formats them
  return (
    kg8(K || null),  // cache the claudeMd content
    o8("info", "user_context_completed", {
      duration_ms: Date.now() - A,
      claudemd_length: K?.length ?? 0,
      claudemd_disabled: Boolean(q),
    }),
    {
      ...(K ? { claudeMd: K } : {}),
      currentDate: `Today's date is ${dP6()}.`,
    }
  );
});
```

Key behavior:
- **Memoized** — runs exactly once per session
- Respects `CLAUDE_CODE_DISABLE_CLAUDE_MDS` env var
- Skipped in headless mode (`zY()`) with no project directories
- `bO()` reads all relevant CLAUDE.md files (user global, project, nested)
- `FE1()` formats the CLAUDE.md content for injection into the system prompt
- Returns `{ claudeMd: string, currentDate: string }` or `{ currentDate: string }` if disabled
- Telemetry: `user_context_started` and `user_context_completed` with `duration_ms`, `claudemd_length`, `claudemd_disabled`

### 3.6 Spinner Tips — gu8()

**gu8(A)** (**cli.js:588736**) — loads and filters spinner tips shown during inference:

```js
async function gu8(A) {
  let K = kA().spinnerTipsOverride,
    _ = YTY();  // load custom/override tips
  if (K?.excludeDefault && _.length > 0) return _;
  let Y = [...KTY, ..._TY],  // KTY = builtin tips, _TY = additional tips
    z = await Promise.all(Y.map((O) => O.isRelevant(A)));
  return [
    ...Y.filter((O, $) => z[$]).filter((O) => xu8(O.id) >= O.cooldownSessions),
    ..._,
  ];
}
```

Filters tips by:
- `isRelevant(A)` — async relevance check per tip
- `xu8(O.id) >= O.cooldownSessions` — cooldown: tip not shown again until N sessions have passed

Called without arguments in `wm8()` to pre-warm the tips cache so spinner display is instant.

### 3.7 Bedrock Auth — DP1()

**DP1()** (**cli.js:179367**) — prefetches AWS Bedrock credentials:

```js
function DP1() {
  let A = zP1(),    // check if bedrock credentials source is set
    q = OP1();      // check for another bedrock config
  if (!A && !q) return;
  if (wP1() || $P1()) {
    if (!aY() && !K7()) return;  // skip if no trust and not non-interactive
  }
  (ws(), H3());    // warm the credential fetchers
}
```

Only runs when `CLAUDE_CODE_USE_BEDROCK` is set and `CLAUDE_CODE_SKIP_BEDROCK_AUTH` is not set (checked in `wm8()`).

### 3.8 Vertex Auth — XP1()

**XP1()** (**cli.js:179360**) — prefetches Google Vertex AI credentials:

```js
function XP1() {
  if (!JP1()) return;      // GCP config check
  if (MP1()) {
    if (!aY() && !K7()) return;  // trust check for some modes
  }
  IB6();   // warm the GCP credential fetcher
}
```

Only runs when `CLAUDE_CODE_USE_VERTEX` is set and `CLAUDE_CODE_SKIP_VERTEX_AUTH` is not set (checked in `wm8()`).

### 3.9 File Count Prefetch — \$j8()

**$j8()** (**cli.js:181802**) — memoized function that counts files in the working directory via ripgrep:

```js
$j8 = z1(
  async (A, q, K = []) => {
    if (Ct.resolve(A) === Ct.resolve(q79())) return;  // skip home dir
    try {
      let _ = ["--files", "--hidden"];
      K.forEach((O) => { _.push("--glob", `!${O}`); });
      let Y = await w79(_, A, q);  // ripgrep count
      if (Y === 0) return 0;
      let z = Math.floor(Math.log10(Y)),
        w = Math.pow(10, z);
      return Math.round(Y / w) * w;  // round to nearest order of magnitude
    } catch (_) {
      if (_?.name !== "AbortError") H6(_);
    }
  },
  (A, q, K = []) => `${A}|${K.join(",")}`,  // cache key
);
```

Called in `wm8()` with a **3-second timeout** (`AbortSignal.timeout(3000)`), the cwd (`G8()`), and an empty exclusion list:
```js
$j8(G8(), AbortSignal.timeout(3000), []);
```

The result is rounded to the nearest power of 10 (e.g., 1,432 files → 1,000). This is used for context about project size.

### 3.10 Analytics Gates — Ft1()

**Ft1()** (**cli.js:550275**) — initializes analytics routing:

```js
async function Ft1() {
  ((gt1 = d_(oyq)),    // oyq = "tengu_log_segment_events"
   (pt1 = d_(syq)));  // syq = "tengu_log_datadog_events"
}
```

Checks two feature flags to determine whether to route telemetry to Segment and/or Datadog. Results cached in `gt1` and `pt1`. Used by `JJY()` (sync) and `MJY()` (async) to gate all telemetry routing.

### 3.11 Models Prefetch — EC7()

**EC7()** (**cli.js:171793**) — prefetches the available model list from the API:

```js
async function EC7() {
  if (!VC7()) return;  // check if model list feature is enabled
  if (process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC) return;
  try {
    let A = await UV({ maxRetries: 1 }),
      q = oA() ? [AD] : void 0,    // AD = beta header if applicable
      K = [];
    for await (let z of A.models.list({ betas: q })) {
      let w = TC7().safeParse(z);  // TC7() = model schema validator
      if (w.success) K.push(w.data);
    }
    // ... caches the model list
  }
```

Fetches the model list with a single retry. Respects `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` env var. The beta header is included if applicable.

### 3.12 Settings Watcher — wX.initialize()

`wX.initialize()` starts watching settings files for changes. This is always initialized in `wm8()`. The watcher monitors the user settings file (`~/.claude/settings.json` and related files) and reloads settings when they change.

### 3.13 Skill/Command File Watcher — pV6 / FMY()

**FMY()** (**cli.js:553823**) — the skill/command file watcher initializer:

```js
async function FMY() {
  if (Oe1 || $e1) return;  // already initialized or disposed
  if (((Oe1 = !0), !mLq))
    ((mLq = !0), kAq(() => { (Gs6(), gV6.forEach((q) => q())); }));  // initial skills load
  let A = await QMY();  // get directories to watch
  if (A.length === 0) return;
  V(`Watching for changes in skill/command directories: ${A.join(", ")}...`);
  uF = O96.watch(A, {   // O96 = chokidar
    persistent: !0,
    ignoreInitial: !0,
    depth: 2,
    awaitWriteFinish: {
      stabilityThreshold: Fs6?.stabilityThreshold ?? uMY,  // uMY = 1000ms
      pollInterval: Fs6?.pollInterval ?? mMY,              // mMY = 500ms
    },
    ignored: (q, K) => {
      if (K && !K.isFile() && !K.isDirectory()) return !0;
      return q.split(E26.sep).some((_) => _ === ".git");  // ignore .git
    },
    ignorePermissionErrors: !0,
    usePolling: pMY,              // pMY = typeof Bun !== "undefined"
    interval: Fs6?.chokidarInterval ?? gMY,  // gMY = 2000ms default
    atomic: !0,
  });
  uF.on("add", we1);
  uF.on("change", we1);
  uF.on("unlink", we1);
}
```

**Directories watched — QMY()** (cli.js:553880):
- `userSettings.skills` — user's global skills directory
- `userSettings.commands` — user's global commands directory  
- `projectSettings.skills` — project `.claude/skills`
- `projectSettings.commands` — project `.claude/commands`
- For each project dir in `UW()`: `<dir>/.claude/skills`

**Change handler — we1(A)** (cli.js:553908):
```js
function we1(A) {
  V(`Detected skill change: ${A}`);
  Q("tengu_skill_file_changed", { source: "chokidar" });
  dMY(A);  // debounced reload
}
```

**Debounced reload — dMY(A)** (cli.js:553913): After `BMY = 300ms` debounce delay:
- Checks `ConfigChange` hook for permission to reload
- Calls `EL8()`, `OF()`, `Rl()` to reload skills
- Notifies all subscribers via `gV6.forEach((_) => _())`

**Watcher constants** (cli.js:553947):
```js
var uMY = 1000,   // stabilityThreshold: 1 second
  mMY = 500,      // pollInterval: 500ms
  BMY = 300,      // reload debounce: 300ms
  gMY = 2000,     // chokidarInterval: 2 seconds
  pMY;            // usePolling: set to true when running under Bun
```

**pV6 interface** (cli.js:553958):
```js
pV6 = { initialize: FMY, dispose: BLq, subscribe: UMY, resetForTesting: cMY };
```

Subscribers (`UMY`) receive callbacks when skill files change:
```js
function UMY(A) {
  return (
    gV6.add(A),
    () => { gV6.delete(A); }  // returns unsubscribe function
  );
}
```

---

## Part 4: How They Relate

### 4.1 Comparison Table

| System | When | Trigger | What It Does | User Visibility | Modifies Files |
|--------|------|---------|-------------|-----------------|----------------|
| **Dream** | Background (24h + 5 sessions) | Post-turn check, time-gated | Memory consolidation from past session transcripts | System notification if files changed; otherwise invisible | Yes (memory files only) |
| **Speculation** | During user input | Immediately after prompt suggestion generated | Runs full inference ahead of confirmation | Invisible until accepted; `time_saved_ms` shown | Yes (via overlay → real fs on accept) |
| **Prefetch** | Startup (after first render) | `wm8()` called once | Context warming (git status, CLAUDE.md, models, credentials) | Invisible | No |

### 4.2 Independence and Simultaneous Execution

All three systems are **architecturally independent**:

- **Dream** is independent of speculation and auto mode. It is triggered by the time+session gate check, not by any user action. It does not affect or interact with speculation state.

- **Speculation** is independent of dream. It activates only when prompt suggestions are generated. `bc1()` can disable it without affecting dream.

- **Prefetch** runs regardless of permission mode, speculation state, or dream status. It is a one-shot initialization triggered by first render.

All three **can run simultaneously**:
- Dream can be running in a background fork while speculation is running in an overlay fork
- Prefetch has already completed by the time dream or speculation are triggered
- Each system has its own abort controller, state namespace, and telemetry labels

### 4.3 Shared Infrastructure — af() and querySource Labels

Both Dream and Speculation use the same `af()` fork infrastructure, but with different `querySource` labels:

| System | querySource | forkLabel | skipTranscript |
|--------|------------|-----------|----------------|
| Dream | `"auto_dream"` | `"auto_dream"` | `true` |
| Speculation | `"speculation"` | `"speculation"` | `true` |
| Suggestion | `"prompt_suggestion"` | `"prompt_suggestion"` | `true` |

All fork-based systems use `skipTranscript: true` to remain invisible to the user's conversation history.

### 4.4 Shared State — appState and speculation field

The `speculation` field lives in the central `appState` (cli.js:454421):
```js
speculation: L16,                    // { status: "idle" } initially
speculationSessionTimeSavedMs: 0,    // accumulates across the session
```

Dream does not have a corresponding `appState` field — its state is managed via the task system (`tasks?.[D]`) and the lock file on disk.

Prefetch does not use `appState` at all — it only populates memoized function caches (`t2`, `Vz`, etc.).

---

## Appendix A: Key Functions Decoded

| Minified Name | Location (cli.js line) | Purpose |
|---------------|------------------------|----------|
| `QR8()` | 451862 | Dream enablement: checks `autoDreamEnabled` setting + `tengu_onyx_plover.enabled` flag |
| `$F_()` | 452305 | Full dream gate: cv() + d4() + F5() + QR8() |
| `HF_()` | 452307 | Always returns `false`; represents force/debug override path |
| `RYq()` | 452309 | Dream initializer — assigns LYq |
| `LYq` | 452473 | Module-level dream trigger function (null until RYq() runs) |
| `hYq(A, q)` | 452460 | Public dream trigger — calls LYq?(A, q) |
| `VYq(A, q, K)` | 452218 | Builds the 4-phase dream prompt |
| `OF_()` | 452281 | Resolves dream config from tengu_onyx_plover + defaults |
| `yYq` | 452472 | Default dream config `{ minHours: 24, minSessions: 5 }` |
| `wF_` | 452454 | Scan throttle: 600000ms (10 minutes) |
| `TB_` | 442447 | Lock filename: `".consolidate-lock"` |
| `kB_` | 442447 | Lock expiry: 3600000ms (1 hour) |
| `Zd1()` | 442388 | Lock file path builder |
| `JR8()` | 442391 | Reads last consolidation timestamp from lock file mtime |
| `z5q()` | 442398 | Acquires the consolidation lock |
| `MR8(A)` | 442427 | Rollback: deletes or back-dates the lock file |
| `w5q(A)` | 442448 | Lists session IDs with mtime > A |
| `jF_(A)` | 452467 | Builds read-only bash canUseTool for dream agent |
| `JF_(D, M)` | ~452473 | Dream message handler — tracks written files |
| `iR8(A)` | 547681 | Builds `memory_saved` system notification message |
| `O5q(v)` | ~452393 | Checks if task is `type: "dream"` |
| `cv()` | 2453 | Kairos/headless mode check (`T8.kairosActive`) |
| `d4()` | 2743 | Remote/dangerous mode check (`T8.isRemoteMode`) |
| `F5()` | 49596 | Auto-memory system enablement check |
| `bc1()` | 456031 | Speculation enablement (returns `false` in v2.1.81) |
| `xc1()` | 456084 | Speculation launcher — creates overlay, starts af() fork |
| `hx(A)` | 456397 | Abort active speculation (user typed) |
| `OU_(A, q, K)` | 456337 | Merge speculation overlay into real filesystem |
| `wzq()` | 456439 | Full speculation accept flow |
| `YU_(A)` | 456031 | Validate/filter speculated messages |
| `wU_(A, q, K, _, Y)` | 456036 | Generate pipelined next suggestion |
| `Sc1()` | 454110 | Prompt suggestion generation driver |
| `Cc1()` | 454173 | Actual suggestion inference via af() |
| `bYq()` | ~454133 | Post-turn hook: generates suggestion + starts speculation |
| `hc1(A)` | 454101 | Suggestion skip-reason checker |
| `Ah8()` | 454059 | Prompt suggestion feature init check |
| `eR8()` | 454056 | Returns `"user_intent"` (prompt ID) |
| `Yh8()` | 455982 | Emits tengu_speculation telemetry |
| `Fc1(A)` | ~455988 | Counts successful tool calls in message list |
| `KU_(A)` | ~455994 | Gets boundary tool name for telemetry |
| `_h8(A)` | 455959 | Builds overlay directory path |
| `qU_()` | ~455962 | Copies overlay files to real filesystem |
| `qa6(A)` | ~455955 | Removes overlay directory (rm -rf with retries) |
| `Kh8(A, q)` | ~455965 | Builds deny result for canUseTool |
| `ik6(A, q)` | ~456072 | Updates speculation state via setAppState |
| `pc1(K)` | ~456504 | Resets speculation state on error |
| `eF_` | 456533 | Write tool set: `{"Edit", "Write", "NotebookEdit"}` |
| `AU_` | 456534 | Read tool set: `{"Read", "Glob", "Grep", "ToolSearch", "LSP", "TaskGet", "TaskList"}` |
| `sF_` | 456509 | Max turns for speculation: 20 |
| `tF_` | 456509 | Max messages before speculation abort: 100 |
| `L16` | 454446 | Idle speculation state `{ status: "idle" }` |
| `wm8()` | 616199 | Deferred prefetch entry point |
| `GLY()` | 616191 | System context conditional launcher |
| `nWA()` | 52009 | Identity/email prefetch |
| `t2` | 264280 | System context memoized fetcher (git status) |
| `Vz` | 264293 | User context memoized fetcher (CLAUDE.md + date) |
| `gu8(A)` | 588736 | Spinner tips loader |
| `DP1()` | 179367 | Bedrock auth prefetch |
| `XP1()` | 179360 | Vertex auth prefetch |
| `$j8()` | 181802 | File count prefetch via ripgrep (3s timeout) |
| `Ft1()` | 550275 | Analytics gates init (Segment + Datadog feature flags) |
| `EC7()` | 171793 | Model list prefetch from API |
| `FMY()` | 553823 | Skill/command file watcher init (chokidar) |
| `QMY()` | 553880 | Collects directories to watch for skill changes |
| `we1(A)` | 553908 | Chokidar change handler for skills |
| `dMY(A)` | 553913 | Debounced skill reload (300ms) |
| `UMY(A)` | 553872 | Subscribe to skill change notifications |
| `BLq()` | 553868 | Dispose/close chokidar watcher |
| `pV6` | 553958 | Skill watcher interface object |
| `PE()` | 543583 | Returns tmpdir base: `$CLAUDE_CODE_TMPDIR` or `/tmp/<session>` |
| `d1()` | 2983 | Config dir: `$CLAUDE_CONFIG_DIR` or `~/.claude` |
| `E8()` | — | Current session ID |
| `l1()` | — | Returns config directory path |
| `IO(path)` | — | Returns transcripts directory from config path |
| `aw()` | — | Returns memory directory path |

---

## Appendix B: Feature Flags and Configuration

### tengu_onyx_plover (Server-side, Dream)

```typescript
{
  enabled: boolean,       // master on/off switch for dream
  minHours: number,       // minimum hours between consolidations (default: 24)
  minSessions: number,    // minimum sessions required (default: 5)
}
```

Read by `l8("tengu_onyx_plover", null)` (cli.js:451864, 452282). Both `enabled` and the numeric parameters are validated before use — invalid values fall back to defaults.

### autoDreamEnabled (User Setting)

```typescript
autoDreamEnabled: {
  source: "settings",
  type: "boolean",
  description: "Enable background memory consolidation"
}
```
(cli.js:446721)

This user-level setting overrides `tengu_onyx_plover.enabled`. If set, it takes precedence (cli.js:451862–451864). The setting is toggled by the user via the UI, and toggling emits `tengu_auto_dream_toggled`.

### tengu_chomp_inflection (Server-side, Speculation)

Gates the prompt suggestion feature. Read in `Ah8()` (cli.js:454072):
```js
if (!l8("tengu_chomp_inflection", !0))
  return (Q("tengu_prompt_suggestion_init", { enabled: !1, source: "growthbook" }), !1);
```
Default behavior when the flag is missing is `!0` (enabled).

### bc1() — Speculation Gate

In v2.1.81, `bc1()` unconditionally returns `false` (cli.js:456031). This disables all speculation regardless of other flags. Prior or future versions may implement a flag check here.

### CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION (Env, Speculation)

Environment variable to force-enable or force-disable prompt suggestions:
- `"false"` → disabled
- `"1"` → enabled
- Absent → governed by `tengu_chomp_inflection` + user setting (cli.js:454059)

### tengu_auto_background_agents (Server-side, separate)

Controls automatic background agent spawning — this is **separate from dream mode** and is not documented here. Dream mode uses `tengu_onyx_plover`, not this flag.

### Prefetch Guards

| Env Var | Effect on Prefetch |
|---------|-------------------|
| `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` | Skips all prefetches entirely (wm8 returns early) |
| `CLAUDE_CODE_USE_BEDROCK` | Enables DP1() prefetch |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | Disables DP1() even when USE_BEDROCK is set |
| `CLAUDE_CODE_USE_VERTEX` | Enables XP1() prefetch |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | Disables XP1() even when USE_VERTEX is set |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Skips EC7() model list prefetch |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | Skips CLAUDE.md loading in Vz() |
| `CLAUDE_CODE_REMOTE` | Skips git status in t2(), affects Vz() |
| `CLAUDE_CODE_TMPDIR` | Base directory for PE() (speculation overlay location) |

---

## Appendix C: Constants and Magic Numbers

| Constant | Value | Location | Meaning |
|----------|-------|----------|---------|
| `wF_` | `600000` | cli.js:452454 | Dream scan throttle: 10 minutes in ms |
| `kB_` | `3600000` | cli.js:442447 | Dream lock expiry: 1 hour in ms |
| `TB_` | `".consolidate-lock"` | cli.js:442447 | Dream lock filename |
| `yYq.minHours` | `24` | cli.js:452472 | Dream: minimum hours between consolidations |
| `yYq.minSessions` | `5` | cli.js:452472 | Dream: minimum sessions since last consolidation |
| `sF_` | `20` | cli.js:456509 | Speculation: max turns |
| `tF_` | `100` | cli.js:456509 | Speculation: max messages before abort |
| `uMY` | `1000` | cli.js:553947 | Chokidar stability threshold: 1 second |
| `mMY` | `500` | cli.js:553947 | Chokidar poll interval: 500ms |
| `BMY` | `300` | cli.js:553947 | Skill reload debounce: 300ms |
| `gMY` | `2000` | cli.js:553947 | Chokidar interval: 2 seconds |
| `3000` (inline) | `3000` | cli.js:616216 | File count prefetch timeout: 3 seconds |

---

*End of reference. Source: `/home/buzzkill/Projects/lab/cc-2.1.81/cli.js` (619,056 lines). Version: 2.1.81 (per package.json).*
