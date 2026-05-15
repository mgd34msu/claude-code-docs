# Claude Code v2.1.81 — Undocumented Features Reference

> **Source**: v2.1.81 bundle (`cli.js`, 619,056 lines; extracted `.ts` modules under `src/`)
> **Build time**: 2026-03-20T21:25:42Z
> **Scope**: Everything not in the official documentation — feature flags, hidden CLI options, environment variables, internal mechanisms, stubs, and experimental features.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Feature Flag System Internals](#2-feature-flag-system-internals)
3. [Fast Mode Controls](#3-fast-mode-controls)
4. [Extended Thinking Features](#4-extended-thinking-features)
5. [Agent Teams & Multi-Agent](#5-agent-teams--multi-agent)
6. [Remote Control (Quartz)](#6-remote-control-quartz)
7. [Team Collaboration](#7-team-collaboration)
8. [Model Compatibility](#8-model-compatibility)
9. [Response Processing & Compaction](#9-response-processing--compaction)
10. [Telemetry & Channels](#10-telemetry--channels)
11. [Session & Memory Features](#11-session--memory-features)
12. [Attribution & Billing Headers](#12-attribution--billing-headers)
13. [Unimplemented Features / Stubs](#13-unimplemented-features--stubs)
14. [Hidden CLI Options](#14-hidden-cli-options)
15. [Experimental Environment Variables](#15-experimental-environment-variables)
16. [GrowthBook Integration](#16-growthbook-integration)
17. [Complete Feature Flag Reference Table](#17-complete-feature-flag-reference-table)
18. [Appendix A: Flag Loading Flow Diagram](#appendix-a-flag-loading-flow-diagram)
19. [Appendix B: Dynamic Config Flags (`LG()`)](#appendix-b-dynamic-config-flags-lg)
20. [Appendix C: Legacy Model Remap Table](#appendix-c-legacy-model-remap-table)

---

## 1. Overview

Claude Code uses **GrowthBook** (with optional Statsig) for server-side feature flags. Flag names use random word pairs (e.g., `tengu_amber_flint`, `tengu_marble_sandcastle`) specifically to obstruct client-side guessing or overriding.

### Flag loading priority (highest to lowest)

1. **Local storage override** — `kP6()` reads a hard-coded object `KI7` populated at startup. Can inject any flag value before GrowthBook is initialized.
2. **Session override** — `NP6()` returns session-level overrides. In v2.1.81 the body is `return;` (returns `undefined`), so session overrides are disabled.
3. **In-memory GrowthBook cache** — `ER` Map, populated after GrowthBook `init()` completes.
4. **`P8().cachedGrowthBookFeatures`** — persisted flag cache written to disk, read on startup before network.
5. **Default value** — the second argument passed to `l8(flagName, default)`.

Two flag-reading APIs exist:

- `l8(flag, default)` — synchronous, uses cache (may be stale). Used everywhere performance matters.
- `cV(flag, default, ttlMs)` — wraps `l8()` with TTL semantics. Used for Brief and Cron.
- `d_(flag)` — synchronous, checks GrowthBook AND Statsig gates. Used for gates requiring security review.
- `qc(flag)` / `gX1(flag)` — async, blocks on GrowthBook init. Used for security restriction gates.

### A/B experiment tracking

When GrowthBook serves a flag from an experiment (source = `"experiment"`), `t56` Map stores `{ experimentId, variationId }` for that flag key. When the flag is read, `cB6(A)` fires `logGrowthBookExperimentTo1P()` to record exposure via 1P event pipeline. The set `xX1` tracks flags already logged to avoid duplicate exposure events.

---

## 2. Feature Flag System Internals

**Source**: `src/conversation/shutdown1peventlogging-2.ts`, module `kt` (cli.js lines 174305–174637)

### Core functions

#### `l8(flagName, defaultValue)` — Primary flag reader
```
function l8(A, q) {
  let K = kP6();          // 1. local storage override
  if (K && A in K) return K[A];
  let _ = NP6();          // 2. session override (disabled in 2.1.81)
  if (_ && A in _) return _[A];
  if (!ed()) return q;    // 3. bail if 1P logging disabled
  if (t56.has(A)) cB6(A); // 4. fire exposure event if experiment
  else TP6.add(A);        // 5. queue for deferred exposure logging
  if (ER.has(A)) return ER.get(A); // 6. in-memory GrowthBook cache
  try {
    let Y = P8().cachedGrowthBookFeatures?.[A];
    return Y !== void 0 ? Y : q;   // 7. disk cache or default
  } catch { return q; }
}
```

#### `kP6()` — Local storage override reader
- Returns `KI7` (a hard-coded object set at startup).
- Setting `uX1 = true` marks the override as initialized.
- Putting flags into `KI7` before `l8()` is called bypasses all GrowthBook logic.

#### `NP6()` — Session override reader
- In v2.1.81: `function NP6() { return; }` — **disabled**.
- Was intended to hold per-session flag overrides; infrastructure exists but is not wired.

#### `ER` Map — In-memory GrowthBook feature cache
- Populated by `_I7(growthBookClient)` after init.
- Maps `flagName → value` for all features returned by GrowthBook.

#### `P8().cachedGrowthBookFeatures` — Disk-persisted cache
- Written to `settings.json` under the Claude data directory.
- Read synchronously at startup for zero-latency flag reads before the async GrowthBook fetch completes.

#### `t56` Map — Experiment tracking
- Maps `flagName → { experimentId, variationId }`.
- Populated from GrowthBook payload when `source === "experiment"`.
- Drives A/B exposure logging via `cB6()` → `logGrowthBookExperimentTo1P()`.

#### `TP6` Set — Deferred exposure queue
- Flags accessed before GrowthBook initializes are queued here.
- After init, `cB6(flag)` is called for each, retroactively logging exposures.

#### `cV(flag, default, ttlMs)` — Cached with refresh
- Source: used in `isbriefentitled-1.ts` and `iskairoscronenabled-1.ts`.
- In v2.1.81 the body is `return l8(A, q)` — the TTL parameter exists in the signature but is unused (the TTL refresh mechanism is present in infrastructure but the caching layer isn't implemented here).

#### `d_(flag)` — Gate check (GrowthBook + Statsig)
```
function d_(A) {
  // checks kP6(), NP6(), then:
  let _ = P8();
  let Y = _.cachedGrowthBookFeatures?.[A];
  if (Y !== void 0) return Boolean(Y);
  return _.cachedStatsigGates?.[A] ?? false;
}
```
Used for `tengu_tool_pear` (remote control tool at startup gate).

### GrowthBook client initialization

- Client is created in `mX1()` with `apiHost: "https://api.anthropic.com/"`, `clientKey: lWA`, `remoteEval: true`.
- Caches on `organizationUUID` and `id`.
- Auth headers injected when trust is established (`Y$().headers`).
- Init timeout: **5 000 ms**.
- On success: `_I7(client)` processes features, `YI7()` persists to disk, `dB6()` fires deferred refresh callbacks.

---

## 3. Fast Mode Controls

**Source**: `src/core/config-1.ts`, function `fX6()` (cli.js lines 114541–114582)

Fast mode ("Penguin mode") is the premium high-speed inference tier using opus-4-6. The `fX6()` function gatekeeps entry in order. It returns `null` if fast mode is allowed, or an error string if blocked.

### 3.1 `tengu_penguins_off` — Fast Mode Kill-Switch

| Property | Value |
|----------|-------|
| **Type** | `string \| null` |
| **Default** | `null` |
| **Source** | `src/core/config-1.ts`, `fX6()`, cli.js line 114542 |
| **Kill switch** | Yes — disables fast mode for all users immediately |

**Behavior**: When non-null, the entire value of the flag is used verbatim as the error message shown to the user. This makes it a surgical kill-switch: Anthropic can set it to any string and that string appears in the Claude Code UI.

```js
let A = l8("tengu_penguins_off", null);
if (A !== null) return (V(`Fast mode unavailable: ${A}`), A);
```

**Example**: Setting to `"Fast mode temporarily unavailable for maintenance"` shows exactly that message.

### 3.2 `tengu_marble_sandcastle` — Native Binary Gate

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |
| **Source** | `src/core/config-1.ts`, `fX6()`, cli.js line 114544 |
| **Kill switch** | Conditional |

**Behavior**: When `true` AND native binary is absent (`!OY()`), blocks fast mode with the error:
> `"Fast mode requires the native binary · Install from: https://claude.com/product/claude-code"`

Purpose: Enforce native binary installation before allowing fast mode. When `false` (default), fast mode works without the native binary.

### 3.3 Fast mode prefetch environment bypass

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_SKIP_FAST_MODE_NETWORK_ERRORS` | When truthy, continues even if the org fast mode endpoint returns a network error |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Skips the `/api/claude_code_penguin_mode` prefetch entirely |

### 3.4 Fast mode org API endpoint

The org enablement check hits `${BASE_API_URL}/api/claude_code_penguin_mode` with OAuth bearer token or `x-api-key`. Response: `{ enabled: boolean, disabled_reason?: string }`. The disabled reason drives specific user-facing error messages.

### 3.5 Disable env var

`CLAUDE_CODE_DISABLE_FAST_MODE` — when truthy, disables fast mode at the process level without setting a flag.

---

## 4. Extended Thinking Features

**Source**: `src/cli/args-1.ts`, function `PX1()` (cli.js lines 181672–181760); `src/telemetry/events-4.ts`

`PX1(model)` builds the array of API beta headers sent with requests. Each flag below gates an additional header.

### 4.1 `tengu_marble_anvil` — Clear Thinking Mode

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |
| **Source** | `src/cli/args-1.ts` line 2340; `src/telemetry/events-4.ts` line 100 |

**Two effects when `true`**:

1. **In `PX1()`**: Adds `tq8` (the `API_CONTEXT_MANAGEMENT` beta header) to requests when thinking is enabled AND `da3(model)` returns true (model supports context management):
   ```js
   let w = da3(A) && l8("tengu_marble_anvil", !1);
   if (l56() && (z || w)) q.push(tq8);
   ```

2. **In `cu7()`** (`src/telemetry/events-4.ts` line 100): Adds `{ type: "clear_thinking_20251015", keep: "all" }` to response edits, which instructs the API to preserve all thinking tokens in the response.
   ```js
   if (q && l8("tengu_marble_anvil", !1))
     K.push({ type: "clear_thinking_20251015", keep: "all" });
   ```

**Purpose**: A new variant of extended thinking that preserves full reasoning traces rather than summarizing them.

### 4.2 `tengu_quiet_hollow` — Thinking Summaries

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |
| **Source** | `src/cli/args-1.ts` line 2335 |

**Behavior**: When ALL conditions are met, adds the thinking-summary beta header (`nIA`):

```js
if (
  l56()              &&  // thinking is enabled
  RC7(A)             &&  // deep search is active  
  !K7()              &&  // not in SDK mode
  kA().showThinkingSummaries !== !0 &&
  l8("tengu_quiet_hollow", !1)
) q.push(nIA);
```

**Purpose**: Intermediate thinking output for deep search debugging. The `showThinkingSummaries !== !0` check prevents double-applying when the user has already opted into summaries via settings.

### 4.3 `tengu_scarf_coffee` — Extended Thinking Beta Variant

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |
| **Source** | `src/cli/args-1.ts` line 2344 |

**Behavior**: Adds the `eq8` beta header when thinking is enabled (`l56()`) and flag is true:

```js
if (Y && l8("tengu_scarf_coffee", !1)) q.push(eq8);
```

`Y` is the result of `l56()` (thinking enabled). This is the simplest condition of the three thinking flags — just thinking active + flag.

**Purpose**: Likely gates a new extended thinking API variant or refinement. Minimal prerequisites vs. `tengu_marble_anvil`.

### 4.4 `tengu_turtle_carbon` — Ultrathink Detection

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `true` |
| **Source** | `src/telemetry/events-4.ts`, `Oc()` at line 123 |

**Behavior**: When `true` (default), enables detection of the `ultrathink` keyword in user prompts.

```js
function Oc() { return l8("tengu_turtle_carbon", !0); }
function iu7(A) { return /\bultrathink\b/i.test(A); }
function iH8(A) {
  let q = [], K = A.matchAll(/\bultrathink\b/gi);
  for (let _ of K)
    if (_.index !== void 0)
      q.push({ word: _[0], start: _.index, end: _.index + _[0].length });
  return q;
}
```

When `ultrathink` is detected in a prompt:
- `Gg6(model)` returns `"medium"` thinking budget for opus-4-6 models (overriding default).
- The positions of all occurrences are extracted by `iH8()` for telemetry.

**Note**: This flag is `true` by default — disabling it (`false`) turns off ultrathink detection entirely.

### 4.5 `tengu_grey_step2` — Ultrathink Configuration

| Property | Value |
|----------|-------|
| **Type** | `Object` |
| **Default** | `au7` (built-in default config) |
| **Source** | `src/telemetry/events-4.ts`, `Zg6()` at line 270 |

**Default config (`au7`)**:
```js
au7 = {
  enabled: true,
  dialogTitle: "We recommend medium effort for Opus",
  dialogDescription:
    "Effort determines how long Claude thinks for when completing your task. "
    + "We recommend medium effort for most tasks to balance speed and intelligence "
    + "and maximize rate limits. Use ultrathink to trigger high effort when needed.",
};
```

**Behavior**: `Zg6()` merges remote flag value with defaults: `{ ...au7, ...flagValue }`. Allows Anthropic to tune ultrathink behavior server-side without a code deploy.

```js
function Zg6() {
  let A = l8("tengu_grey_step2", au7);
  return { ...au7, ...A };
}
```

The merged result controls whether the ultrathink dialog appears and its copy.

### 4.6 Effort Level Environment Variables

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_EFFORT_LEVEL` | Set effort: `low`, `medium`, `high`, `max`, `unset`, `auto`. `unset`/`auto` = null (model decides). |
| `MAX_THINKING_TOKENS` | Override max thinking token count. Any positive integer. |
| `CLAUDE_CODE_DISABLE_THINKING` | Disable extended thinking entirely. |
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` | Disable adaptive thinking mode. |
| `CLAUDE_CODE_ALWAYS_ENABLE_EFFORT` | Force effort controls on all models, not just those that natively support it. |

---

## 5. Agent Teams & Multi-Agent

**Source**: `src/cli/args-1.ts`, functions `ca3()` and `I7()` (cli.js lines 181761–181774)

### 5.1 `tengu_amber_flint` — Agent Teams Gate

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `true` |
| **Source** | `src/cli/args-1.ts` line 2370 |

**Behavior**:
```js
function ca3() {
  return process.argv.includes("--agent-teams");
}
function I7() {
  if (!a6(process.env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS) && !ca3())
    return !1;
  if (!l8("tengu_amber_flint", !0)) return !1;
  return !0;
}
```

Requires EITHER the env var OR the CLI flag AND the flag to be `true`. Since it defaults to `true`, Anthropic can kill agent teams server-side by setting it to `false`.

**Activation**:
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` — env var enable
- `--agent-teams` — CLI flag (hidden from help)

### 5.2 `tengu_auto_background_agents` — Auto Background Agents

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |

Gates automatic spawning of background agents without explicit user invocation.

### 5.3 Hidden Agent Team CLI Options

All hidden from `--help` (`.hideHelp()`):

| Flag | Description |
|------|-------------|
| `--agent-teams` | Enable agent teams mode |
| `--agent-id <id>` | Teammate agent ID (set by orchestrator) |
| `--agent-name <name>` | Teammate display name |
| `--team-name <name>` | Team name for swarm coordination |
| `--agent-color <color>` | Teammate UI color |
| `--agent-type <type>` | Custom agent type for this teammate |
| `--plan-mode-required` | Require plan mode before implementation |
| `--parent-session-id <id>` | Parent session ID for analytics correlation |
| `--teammate-mode <mode>` | How to spawn teammates: `tmux`, `in-process`, or `auto` |

---

## 6. Remote Control (Quartz)

**Source**: `src/tools/tool-1.ts`, functions `bk6()`, `Qd1()`, `xk6()` (cli.js lines 448105–448120)

### 6.1 `tengu_amber_quartz_disabled` — Quartz Kill-Switch

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |
| **Source** | `src/tools/tool-1.ts` line 6405 |
| **Kill switch** | Yes (negated logic) |

**Important**: The flag name contains "disabled" and the logic is **negated**:
```js
function bk6() {
  return !l8("tengu_amber_quartz_disabled", !1);
}
function Qd1() {
  if (!Yj()) return !1;        // require native binary
  let A = hA();
  return Boolean(A?.accessToken); // require OAuth token
}
function xk6() {
  return Qd1() && bk6();  // OAuth token AND not disabled
}
```

- `bk6()` returns `true` when feature is **enabled** (flag is `false` = not disabled).
- `xk6()` = full gate: requires OAuth access token AND not disabled.
- Setting the flag to `true` **disables** Quartz.

**Quartz** is the Remote Control feature — allows external systems/UIs to control Claude Code sessions at startup. The voice warmup hint UI (`src/ui/voicewarmuphint-3.ts`) is also gated behind `bk6()`.

### 6.2 `tengu_tool_pear` — Remote Control Tool at Startup

| Property | Value |
|----------|-------|
| **Type** | `boolean` (checked via `d_()`) |
| **Default** | Statsig/GrowthBook gate |
| **Source** | `src/cli/args-1.ts` line 2342; `src/tools/definitions-1.ts` line 10016 |

```js
let O = d_("tengu_tool_pear");
if (Y && i56(A) && O) q.push(_o); // _o = Quartz startup beta header
```

Adds the Quartz startup beta header when thinking is enabled, model supports it, and gate passes.

### 6.3 `tengu_remote_backend` — Remote Backend

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |

Gates remote backend connectivity for session management.

### 6.4 `tengu_cobalt_lantern` / `tengu_surreal_dali` — Remote Sessions

Both gate `X2("allow_remote_sessions", ...)`. `tengu_cobalt_lantern` additionally controls GitHub connectivity messaging for remote agents.

### 6.5 Hidden Remote CLI Options

| Flag | Description |
|------|-------------|
| `--remote [description]` | Create a remote session with the given description |
| `--remote-control [name]` | Start an interactive session with Remote Control enabled (optionally named) |
| `--rc [name]` | Alias for `--remote-control` |
| `--sdk-url <url>` | Use remote WebSocket endpoint for SDK I/O streaming (requires `-p` and `stream-json`) |
| `--teleport [session]` | Resume a teleport session, optionally specify session ID |

---

## 7. Team Collaboration

**Source**: `src/networking/validateteammemwritepath-1.ts`, module `t1` (cli.js lines 174638–174823)

### 7.1 `tengu_herring_clock` — Team Memory Directory

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |
| **Source** | `src/networking/validateteammemwritepath-1.ts` line 118 |

**Behavior**:
```js
function JH8() {
  if (!F5()) return !1;  // F5() = team mode active
  return l8("tengu_herring_clock", !1);
}
```

`JH8()` is `isTeamMemoryEnabled()`. Requires team mode active AND flag true.

**When enabled**:
- `lV()` returns the team memory path: `{appDataDir}/team/` (NFC-normalized)
- `Bo3()` returns the team MEMORY.md entrypoint: `{appDataDir}/team/MEMORY.md`
- Full path traversal protection is in place: null bytes, URL-encoded traversal, Unicode normalization traversal, backslashes, absolute paths, and symlink loops are all rejected.

**Security**: `go3(path)` validates write paths with `validateTeamMemWritePath`, using `realpath()` and containment checks to prevent symlink escape.

---

## 8. Model Compatibility

**Source**: `src/core/resolveskillmodeloverride-1.ts`, function `BY8()` (cli.js line 115236)

### 8.1 `tengu_grey_wool` — Legacy Model Remapping

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `true` |
| **Source** | `src/core/resolveskillmodeloverride-1.ts` line 237 |
| **Env override** | `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP` |

**Behavior**:
```js
function BY8() {
  if (a6(process.env.CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP)) return !1;
  return l8("tengu_grey_wool", !0);
}
```

When `true` (default), old model names in `KM3` auto-remap to the current default opus model (`ET()` = `claude-opus-4-6`). This is applied in `Y5(model)` via `_M3()`.

**Legacy models that remap** (`KM3` array):

| Legacy Model Name | Remaps To |
|-------------------|----------|
| `claude-opus-4-20250514` | `claude-opus-4-6` |
| `claude-opus-4-1-20250805` | `claude-opus-4-6` |
| `claude-opus-4-0` | `claude-opus-4-6` |
| `claude-opus-4-1` | `claude-opus-4-6` |

**Disable**: Set `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP=1` or server-side set flag to `false`.

### 8.2 Model aliases

`Y5(model)` resolves short aliases:

| Alias | Resolves To |
|-------|------------|
| `opus` | `ET()` = `claude-opus-4-6` (or `ANTHROPIC_DEFAULT_OPUS_MODEL`) |
| `sonnet` / `opusplan` | `HG()` = `claude-sonnet-4-6` (or `ANTHROPIC_DEFAULT_SONNET_MODEL`) |
| `haiku` | `kX6()` = `claude-haiku-4-5` (or `ANTHROPIC_DEFAULT_HAIKU_MODEL`) |
| `best` | `yw7()` = `ET()` |

### 8.3 Model environment variable overrides

| Variable | Effect |
|----------|--------|
| `ANTHROPIC_MODEL` | Override main loop model |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Override default opus |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Override default sonnet |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Override default haiku |
| `ANTHROPIC_SMALL_FAST_MODEL` | Override the small/fast model used for subagent tasks |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Override subagent model specifically |

### 8.4 1M context mode

`sH()` returns `true` when 1M context is available and active:
- Not in free tier (`Yd()`), not over-context (`OI()`), first-party only
- When active, `[1m]` suffix is appended to model name: e.g., `claude-opus-4-6[1m]`
- `db6(A, q)`: if user's requested model lacks `[1m]` but their active session has it, appends the suffix

---

## 9. Response Processing & Compaction

### 9.1 `tengu_pewter_ledger` — Transcript Truncation Strategy

| Property | Value |
|----------|-------|
| **Type** | `string \| null` |
| **Valid values** | `"trim"`, `"cut"`, `"cap"`, `null` |
| **Default** | `null` |
| **Source** | `src/agents/startdeferredprefetches-1.ts` line 464 |

```js
let A = l8("tengu_pewter_ledger", null);
```

| Value | Behavior |
|-------|----------|
| `"trim"` | Trim message content to stay within token limit |
| `"cut"` | Cut older messages entirely |
| `"cap"` | Cap at a maximum message count |
| `null` | Default compaction behavior |

### 9.2 `tengu_hawthorn_window` / `tengu_hawthorn_steeple` — Tool Result Persistence

| Property | Value |
|----------|-------|
| **`tengu_hawthorn_window`** | `null` — controls the persistence window for tool results across compaction |
| **`tengu_hawthorn_steeple`** | `false` — controls persistence strategy |
| **Source** | `src/agents/startdeferredprefetches-1.ts` lines 2072, 2077 |

```js
let A = l8("tengu_hawthorn_window", null);
// ...
if (!l8("tengu_hawthorn_steeple", !1)) return;
```

`tengu_hawthorn_window` being non-null activates the window, and `tengu_hawthorn_steeple` gates the persistence strategy.

### 9.3 `tengu_satin_quoll` — Message-Level Tool Budget

| Property | Value |
|----------|-------|
| **Internal name** | `RI9 = "tengu_satin_quoll"` |
| **Source** | `src/agents/startdeferredprefetches-1.ts` line 2293 |

Controls per-message token budget for tool result preservation during compaction.

### 9.4 `tengu_sm_compact` / `tengu_session_memory` — Session Memory Compaction

| Flag | Default | Purpose |
|------|---------|--------|
| `tengu_session_memory` | `false` | Enable session memory feature |
| `tengu_sm_compact` | `false` | Enable session memory compaction |

```js
let A = l8("tengu_session_memory", !1),
    q = l8("tengu_sm_compact", !1);
```

When `tengu_sm_compact` is active, the system reads a summarized prior session, emitting telemetry events: `tengu_sm_compact_no_session_memory`, `tengu_sm_compact_empty_template`, `tengu_sm_compact_summarized_id_not_found`, `tengu_sm_compact_resumed_session`, `tengu_sm_compact_threshold_exceeded`, `tengu_sm_compact_error`.

### 9.5 `tengu_compact_cache_prefix` / `tengu_compact_streaming_retry`

| Flag | Default | Purpose |
|------|---------|--------|
| `tengu_compact_cache_prefix` | `false` | Enable cache prefix during compaction |
| `tengu_compact_streaming_retry` | `false` | Retry compaction on streaming failure |

### 9.6 `tengu_chomp_inflection` — Context Inflection

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `true` |

Controls context inflection point behavior (default enabled).

### 9.7 `tengu_system_prompt_global_cache` — System Prompt Caching

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |

Enables global caching of the system prompt across requests. When active, the system prompt gets a persistent `cache_control` block.

### 9.8 `tengu_prompt_cache_1h_config` — Prompt Cache Allow-List

| Property | Value |
|----------|-------|
| **Type** | `Object` |
| **Default** | `{}` |

```js
let A = l8("tengu_prompt_cache_1h_config", {});
// Uses A.allowlist to determine which content gets 1-hour cache TTL
```

### 9.9 `tengu_amber_wren` — Tool Result Limits

| Property | Value |
|----------|-------|
| **Type** | `Object` |
| **Default** | `{}` |
| **Source** | `src/tools/tool-1.ts`, `w_6()` at line 1897 |

Configurable properties:

| Sub-property | Type | Fallback | Description |
|-------------|------|---------|-------------|
| `maxSizeBytes` | `number` | `ar8` constant | Maximum tool result size in bytes |
| `maxTokens` | `number` | `kI9` constant (overridden by `NI9()`) | Maximum tool result tokens |
| `includeMaxSizeInPrompt` | `boolean` | `false` | Tell Claude about size limits in system prompt |
| `targetedRangeNudge` | `boolean` | `false` | Enable range nudging algorithm for large outputs |

```js
w_6 = z1(() => {
  let A = l8("tengu_amber_wren", {}),
    q = typeof A?.maxSizeBytes === "number" && ... ? A.maxSizeBytes : ar8,
    _ = NI9() ?? (typeof A?.maxTokens === "number" && ... ? A.maxTokens : kI9);
  ...
});
```

**Purpose**: Allows Anthropic to dynamically tune tool output limits without a code deploy.

---

## 10. Telemetry & Channels

**Source**: `src/telemetry/ischannelsenabled-1.ts`, `src/telemetry/isbriefentitled-1.ts`, `src/telemetry/iskairoscronenabled-1.ts`

### 10.1 `tengu_harbor` — MCP Channels Feature Gate

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |
| **Source** | `src/telemetry/ischannelsenabled-1.ts` line 28 |

```js
function Ho6() { return l8("tengu_harbor", !1); }  // isChannelsEnabled()
```

Gates the MCP channel notification system (inbound push notifications from MCP servers).

### 10.2 `tengu_harbor_ledger` — Channel Allow-List

| Property | Value |
|----------|-------|
| **Type** | `string[]` |
| **Default** | `[]` |
| **Source** | `src/telemetry/ischannelsenabled-1.ts` line 23 |

```js
function $o6() {
  let A = l8("tengu_harbor_ledger", []),
    q = O8Y().safeParse(A);
  return q.success ? q.data : [];
}
```

Server-side allowlist of MCP server names that can use channels. The array is validated with a Zod schema (`O8Y()`).

### 10.3 `tengu_harbor_permissions` — Channel Permissions Gate

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |

Additional permission gate on top of the harbor allowlist.

### 10.4 `tengu_kairos_brief` — Brief Mode Gate

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |
| **Source** | `src/telemetry/isbriefentitled-1.ts` line 32 |
| **TTL** | 300,000ms (5 minutes) |

```js
function Cf8() {
  return cv() || a6(process.env.CLAUDE_CODE_BRIEF) ||
    cV("tengu_kairos_brief", !1, $r9);  // $r9 = 300000
}
```

Three paths to enable Brief:
1. SDK mode (`cv()`)
2. `CLAUDE_CODE_BRIEF=1` env var
3. GrowthBook flag

### 10.5 `tengu_kairos_brief_config` — Brief Configuration

| Property | Value |
|----------|-------|
| **Type** | `Object` |
| **Default** | `RVq` (built-in config) |

Configuration for the Brief tool behavior.

### 10.6 `tengu_kairos_cron` — Scheduled Tasks (Cron)

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `true` |
| **Source** | `src/telemetry/iskairoscronenabled-1.ts` line 46 |
| **TTL** | 300,000ms (5 minutes) |
| **Kill switch** | `CLAUDE_CODE_DISABLE_CRON=1` |

```js
function Vh() {
  return !a6(process.env.CLAUDE_CODE_DISABLE_CRON) &&
    cV("tengu_kairos_cron", !0, qu9);
}
```

Gates the cron/scheduling system. Default: **enabled** — Anthropic can disable globally. User can also disable via env var.

**Cron defaults** (from module init):
```js
Lg = {
  recurringFrac: 0.1,      // fraction of max turns for recurring tasks
  recurringCapMs: 900000,  // 15 minute cap per recurring run
  oneShotMaxMs: 90000,     // 90 second cap for one-shot tasks
  oneShotFloorMs: 0,
  oneShotMinuteMod: 30,    // schedule at 30-minute boundaries
  recurringMaxAgeMs: 604800000,  // 7 days max age
};
```

**Tool names** exposed to Claude: `CronCreate`, `CronDelete`, `CronList`
**Storage**: `{dataDir}/.claude/scheduled_tasks.json`

### 10.7 `tengu_1p_event_batch_config` — 1P Event Batching

| Property | Value |
|----------|-------|
| **Type** | `Object` |
| **Default** | `{}` |
| **Source** | `src/conversation/shutdown1peventlogging-2.ts` line 91 |

Controls OTEL log batching for first-party event telemetry:

| Sub-property | Fallback |
|-------------|----------|
| `scheduledDelayMillis` | `OTEL_LOGS_EXPORT_INTERVAL` env or 10,000ms |
| `maxExportBatchSize` | 200 |
| `maxQueueSize` | `Lo3` constant |
| `skipAuth` | false |
| `maxAttempts` | default |
| `path` | default endpoint |
| `baseUrl` | default |

### 10.8 `tengu_event_sampling_config` — Event Sampling

Lookup key constant `ko3 = "tengu_event_sampling_config"`. Used by `$H8(eventName)` to determine sampling rate per event type. Shape: `{ [eventName]: { sample_rate: number } }` where sample rate is 0.0–1.0.

### 10.9 Startup Profiling

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_PROFILE_STARTUP=1` | Always record startup performance marks (normally 0.5% chance) |

When profiling is active, `tengu_startup_perf` telemetry event is emitted with timing for: `import_time`, `init_time`, `settings_time`, `total_time`, `checkpoint_count`.

---

## 11. Session & Memory Features

### 11.1 `tengu_coral_fern` — Memory Directory Loading

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |
| **Source** | `src/agents/startdeferredprefetches-1.ts` line 258 |

```js
if (!l8("tengu_coral_fern", !1)) return [];
```

When `false`, memory directory loading returns an empty array (disabled). Enables automatic memory file loading from the configured memory directory on session start.

### 11.2 `tengu_swinburne_dune` — Memory Directory Config

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |
| **Source** | `src/agents/startdeferredprefetches-1.ts` lines 115, 285, 1816 |

```js
let X = l8("tengu_swinburne_dune", !1);
// ...
let Y = l8("tengu_swinburne_dune", !1) ? x94 : I94;
```

Switches between two memory directory configuration objects (`x94` vs `I94`).

### 11.3 `tengu_passport_quail` — Memory Directory Filtering

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |
| **Source** | `src/agents/startdeferredprefetches-1.ts` lines 184, 295, 306 |

```js
if (!l8("tengu_passport_quail", !1)) return;
```

Controls filtering of memory directory entries. When `false`, filter function returns early (bypasses filtering).

### 11.4 `tengu_paper_halyard` — CLAUDE.md Parsing

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |
| **Source** | `src/agents/startdeferredprefetches-1.ts` lines 736, 9186 |

```js
_ = l8("tengu_paper_halyard", !1);
```

Gates additional CLAUDE.md parsing logic. When `false`, the extended parsing code path is skipped.

### 11.5 `tengu_tight_weave` — Response Format Instructions

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `true` |
| **Source** | `src/agents/startdeferredprefetches-1.ts` line 2505 |

```js
${l8("tengu_tight_weave", !0)
  ? "- In your final response, share file paths (always absolute, never relative) that are relevant to the task. Include code snippets only when the exact text is load-bearing (e.g., a bug you found, a function signature the caller asked for) — do not recap code you merely read."
  : "- In your final response always share relevant file names and code snippets. Any file paths you return in your response MUST be absolute. Do NOT use relative paths."
}
```

Controls which system prompt variant for file path and code snippet instructions is used. Default (`true`) uses the more concise "tight" wording.

### 11.6 `tengu_session_memory` — Session Memory

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |

Enables the session memory feature, which persists and loads session-level context. When active, emits `tengu_session_memory_file_read`, `tengu_session_memory_extraction`, `tengu_session_memory_loaded`, `tengu_session_memory_accessed` telemetry.

### 11.7 `tengu_trace_lantern` — Debug Tracing

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `false` |
| **Source** | `src/tools/hasworktreecreatehook-1.ts` line 1298 |

```js
return K7() || l8("tengu_trace_lantern", !1);
```

Enables debug tracing output. Also activated when in SDK mode (`K7()`).

---

## 12. Attribution & Billing Headers

### 12.1 `tengu_attribution_header` — Attribution Header

| Property | Value |
|----------|-------|
| **Type** | `boolean` |
| **Default** | `true` |
| **Source** | `src/telemetry/events-4.ts` line 311 |

```js
function BA9() {
  if (dY(process.env.CLAUDE_CODE_ATTRIBUTION_HEADER)) return !1;
  return l8("tengu_attribution_header", !0);
}
```

When enabled, adds the `x-anthropic-billing-header` to API requests:
```
x-anthropic-billing-header: cc_version=2.1.81.<entrypoint>; cc_entrypoint=<entrypoint>; cch=2a76f; [cc_workload=<tag>;]
```

The hash `cch=2a76f` appears to be a build/version fingerprint. `cc_workload` is set via `--workload <tag>` (hidden CLI flag).

**Disable**: `CLAUDE_CODE_ATTRIBUTION_HEADER=0` or server-side flag to `false`.

### 12.2 `--workload <tag>` — Workload Billing Tag (hidden)

Hidden CLI option that sets `cc_workload` in the billing header. Intended for SDK daemon callers that spawn subprocesses for cron work.

---

## 13. Unimplemented Features / Stubs

### 13.1 Python Package Plugins

**Location**: `src/core/config-1.ts` line 8069

```js
throw Error("Python package plugins are not yet supported");
```

The plugin system supports `npm` and `git` source types. A `python` type exists in the schema (package name / PyPI) but throws immediately. The MCP schema even has:
```js
.describe("Python package name as it appears on PyPI")
.describe("Python package as plugin source")
```

### 13.2 Prompt Stop Hooks Outside REPL

**Location**: `src/tools/hasworktreecreatehook-1.ts` lines 1607, 1614

```js
output: "Prompt stop hooks are not yet supported outside REPL"
output: "Agent stop hooks are not yet supported outside REPL"
```

`Prompt` and `Agent` stop hook types only work in interactive REPL mode. They return a stub error output when triggered in non-interactive (`--print`) mode.

### 13.3 Windows Session Environment

**Location**: `src/conversation/session-1.ts` line 2497

```js
return (V("Session environment not yet supported on Windows"), null);
```

`.env` file loading into session environment is not implemented on Windows. macOS and Linux work correctly.

### 13.4 `file://` URI Loading

**Location**: `src/core/auth-1.ts` line 1902

```js
return Promise.resolve(dz("not implemented... yet..."));
```

HTTP and HTTPS URLs work. The `file://` protocol handler exists in the routing code but resolves to a "not implemented" error.

### 13.5 Python Package Manager Detection

**Location**: `src/core/tmuxbackend-2.ts` line 33; `src/vendor/dom.ts` line 68426

```js
// tmuxbackend-2.ts:
return (V("[it2Setup] No Python package manager found"), null);
// dom.ts (Python plugin installer):
H("No Python package manager found (uvx, pipx, or pip)")
```

The system looks for `uvx`, `pipx`, or `pip` to install Python plugins, but since Python plugins aren't supported yet, this path is never reached via the public plugin API.

---

## 14. Hidden CLI Options

All options using `.hideHelp()` in `src/vendor/commander.ts`. These are valid flags but do not appear in `--help` output.

### 14.1 Main Command Hidden Options

| Flag | Description | Notes |
|------|-------------|-------|
| `-d2e, --debug-to-stderr` | Enable debug mode, output to stderr | Also `-d2e` shorthand |
| `--init` | Run Setup hooks with init trigger, then continue | |
| `--init-only` | Run Setup and SessionStart:startup hooks, then exit | |
| `--maintenance` | Run Setup hooks with maintenance trigger, then continue | |
| `--output-format <format>` | Output format with `--print`: `text`, `json`, `stream-json` | Hidden variant |
| `--max-thinking-tokens <tokens>` | **DEPRECATED** Max thinking tokens (use `--thinking` instead) | |
| `--max-turns <turns>` | Max agentic turns in non-interactive mode | |
| `--max-budget-usd <amount>` | Max dollar spend on API calls | |
| `--noedit` | (unnamed, line 4970) | |
| `--system-prompt <prompt>` | System prompt for the session | |
| `--system-prompt-file <file>` | Read system prompt from file | |
| `--append-system-prompt <prompt>` | Append to default system prompt | |
| `--permission-mode <mode>` | Permission mode for the session | |
| `--prefill <text>` | Pre-fill prompt input without submitting | |
| `--deep-link-origin` | Signal session launched from deep link | |
| `--deep-link-repo <slug>` | Repo slug resolved from deep link `?repo=` | |
| `--deep-link-last-fetch <ms>` | FETCH_HEAD mtime (epoch ms), precomputed by deep link trampoline | |
| `--from-pr [value]` | Resume session linked to PR (by number/URL) | |
| `--rewind-files <user-message-id>` | Restore files to state at user message (requires `--resume`) | |
| `--workload <tag>` | Workload tag for billing header attribution (cron subprocesses) | |
| `--enable-auto-mode` | Opt in to auto mode | |
| `--brief` | Enable `SendUserMessage` tool for agent-to-user communication | |
| `--channels <servers...>` | MCP channel notification servers (space-separated) | |
| `--dangerously-load-development-channels <servers...>` | Load unapproved channel servers (dev only, shows confirmation dialog) | |
| `--agent-id <id>` | Teammate agent ID | |
| `--agent-name <name>` | Teammate display name | |
| `--team-name <name>` | Team name for swarm coordination | |
| `--agent-color <color>` | Teammate UI color | |
| `--plan-mode-required` | Require plan mode before implementation | |
| `--parent-session-id <id>` | Parent session ID for analytics correlation | |
| `--teammate-mode <mode>` | Teammate spawn mode: `tmux`, `in-process`, `auto` | |
| `--agent-type <type>` | Custom agent type for teammate | |
| `--sdk-url <url>` | Remote WebSocket endpoint for SDK I/O streaming | |
| `--teleport [session]` | Resume a teleport session | |
| `--remote [description]` | Create remote session | |
| `--remote-control [name]` | Interactive session with Remote Control | |
| `--rc [name]` | Alias for `--remote-control` | |
| `--agent-teams` | Enable agent teams mode | |

### 14.2 Plugin Subcommand Hidden Options

All plugin/marketplace subcommands (`claude plugin validate`, `claude plugin list`, `claude marketplace add`, `claude marketplace rm`, etc.) accept:

| Flag | Description |
|------|-------------|
| `--cowork` | Use `cowork_plugins` directory instead of default plugins directory |

This appears throughout all plugin-related subcommand handlers (lines 6816, 6831, 6847, 6867, 6879, 6892, 6910, 6931, 6945, 6960).

### 14.3 Bare / Simple Mode

```js
function zY() {
  return a6(process.env.CLAUDE_CODE_SIMPLE) || process.argv.includes("--bare");
}
```

`--bare` CLI flag (also `CLAUDE_CODE_SIMPLE=1`) activates simple/bare mode: skips hooks, LSP, plugins, and OAuth flows. Not in `--help`.

---

## 15. Experimental Environment Variables

Complete list of `CLAUDE_CODE_*` environment variables found in source. Grouped by function.

### 15.1 Feature Activation

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | Enable agent teams (also requires `--agent-teams` or env var + `tengu_amber_flint` gate) |
| `CLAUDE_CODE_BRIEF` | Enable Brief mode (SendUserMessage tool) |
| `CLAUDE_CODE_SIMPLE` | Enable bare/simple mode (skip hooks, LSP, plugins) |
| `CLAUDE_CODE_PROFILE_STARTUP=1` | Force startup performance profiling (normally 0.5% chance) |
| `CLAUDE_CODE_ACCESSIBILITY` | Enable accessibility features |
| `CLAUDE_CODE_ENABLE_CFC` | Enable CFC (context for Claude?) feature |
| `CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING` | Fine-grained tool streaming |
| `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` | Prompt suggestion feature |
| `CLAUDE_CODE_ENABLE_TASKS` | Enable task system |
| `CLAUDE_CODE_USE_NATIVE_FILE_SEARCH` | Use native file search binary |
| `CLAUDE_CODE_USE_POWERSHELL_TOOL` | Use PowerShell as Bash replacement (Windows) |
| `CLAUDE_CODE_BUBBLEWRAP` | Enable bubblewrap sandboxing |
| `CLAUDE_CODE_AUTO_CONNECT_IDE` | Auto-connect to IDE on startup |

### 15.2 Feature Disablement

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP` | Disable legacy model name remapping (`tengu_grey_wool` override) |
| `CLAUDE_CODE_DISABLE_CRON` | Disable scheduled tasks/cron |
| `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` | Disable all beta API headers |
| `CLAUDE_CODE_DISABLE_FAST_MODE` | Disable fast mode (Penguin mode) |
| `CLAUDE_CODE_DISABLE_THINKING` | Disable extended thinking entirely |
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` | Disable adaptive thinking |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | Disable automatic memory file loading |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | Disable background task spawning |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | Disable CLAUDE.md file parsing |
| `CLAUDE_CODE_DISABLE_ATTACHMENTS` | Disable file attachment support |
| `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK` | Skip command injection security check |
| `CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY` | Disable feedback survey prompts |
| `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` | Disable file checkpoint system |
| `CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS` | Disable git-related system prompt instructions |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Skip non-essential network requests (fast mode org check, analytics) |
| `CLAUDE_CODE_DISABLE_OFFICIAL_MARKETPLACE_AUTOINSTALL` | Skip auto-install from official marketplace |
| `CLAUDE_CODE_DISABLE_PRECOMPACT_SKIP` | Disable pre-compact optimization |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | Disable terminal title updates |
| `CLAUDE_CODE_DISABLE_VIRTUAL_SCROLL` | Disable virtual scrolling in UI |
| `CLAUDE_CODE_ATTRIBUTION_HEADER` | Set to `0`/`false` to disable billing attribution header |

### 15.3 Model & API Configuration

| Variable | Effect |
|----------|--------|
| `ANTHROPIC_MODEL` | Override main loop model |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Override default opus model |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Override default sonnet model |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Override default haiku model |
| `ANTHROPIC_SMALL_FAST_MODEL` | Override the small/fast sub-agent model |
| `ANTHROPIC_BASE_URL` | Override API base URL |
| `ANTHROPIC_AUTH_TOKEN` | Override auth token |
| `ANTHROPIC_API_KEY` | API key (standard) |
| `ANTHROPIC_BETAS` | Additional beta headers (comma-separated) |
| `CLAUDE_CODE_API_BASE_URL` | Alt: override API base URL |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Override subagent model |
| `CLAUDE_CODE_USE_BEDROCK` | Use AWS Bedrock |
| `CLAUDE_CODE_USE_VERTEX` | Use Google Vertex |
| `CLAUDE_CODE_USE_FOUNDRY` | Use Anthropic Foundry |
| `AWS_REGION` / `AWS_DEFAULT_REGION` | AWS region (default: `us-east-1`) |
| `CLOUD_ML_REGION` | Vertex region (default: `us-east5`) |

### 15.4 Auth & Credentials

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR` | FD number to read API key from |
| `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` | TTL for API key helper cache |
| `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` | FD for WebSocket auth |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | Session-scoped access token |
| `CLAUDE_CODE_CLIENT_CERT` | TLS client certificate |
| `CLAUDE_CODE_CLIENT_KEY` | TLS client key |
| `CLAUDE_CODE_CLIENT_KEY_PASSPHRASE` | TLS client key passphrase |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | Skip Bedrock authentication |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | Skip Vertex authentication |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | Skip Foundry authentication |
| `CLAUDE_CODE_CUSTOM_OAUTH_URL` | Custom OAuth endpoint URL |
| `USE_API_CONTEXT_MANAGEMENT` | Force API context management header (also gated by `tengu_marble_anvil`) |

### 15.5 Effort & Thinking

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_EFFORT_LEVEL` | `low`, `medium`, `high`, `max`, `unset`, `auto` |
| `MAX_THINKING_TOKENS` | Override max thinking token budget |
| `CLAUDE_CODE_ALWAYS_ENABLE_EFFORT` | Force effort controls on all model types |

### 15.6 Debug & Logging

| Variable | Effect |
|----------|--------|
| `DEBUG` | Enable debug mode |
| `DEBUG_SDK` | Enable SDK debug mode |
| `CLAUDE_CODE_DEBUG_LOG_LEVEL` | Log level: `verbose`, `debug`, `info`, `warn`, `error` |
| `CLAUDE_CODE_DEBUG_LOGS_DIR` | Directory for debug log files |
| `CLAUDE_CODE_DIAGNOSTICS_FILE` | Path to write diagnostics JSON |
| `CLAUDE_CODE_SLOW_OPERATION_THRESHOLD_MS` | Threshold for slow-operation warnings |
| `OTEL_LOGS_EXPORT_INTERVAL` | OTEL log export interval (ms) |
| `CLAUDE_CODE_OTEL_FLUSH_TIMEOUT_MS` | OTEL flush timeout |
| `CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS` | OTEL shutdown timeout |
| `CLAUDE_CODE_OTEL_HEADERS_HELPER_DEBOUNCE_MS` | OTEL headers debounce |
| `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS` | Datadog flush interval |
| `CLAUDE_CODE_PERFETTO_TRACE` | Enable Perfetto tracing |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | Enable telemetry |
| `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA` | Enhanced telemetry beta |

### 15.7 Session & Behavior

| Variable | Effect |
|----------|--------|
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | Maintain project cwd in bash tool |
| `CLAUDE_CODE_SKIP_FAST_MODE_NETWORK_ERRORS` | Continue when fast mode org check fails with network error |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | Force plan mode |
| `CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE` | Enable plan mode interview phase |
| `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | Skip loading prompt history |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | Skip Bedrock auth check |
| `CLAUDE_CODE_RESUME_INTERRUPTED_TURN` | Resume interrupted turn on restart |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | Auto-compact window size |
| `CLAUDE_CODE_STALL_TIMEOUT_MS_FOR_TESTING` | Stall timeout (testing) |
| `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` | Timeout for session-end hooks |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | Override blocking limit |
| `CLAUDE_CODE_DONT_INHERIT_ENV` | Don't inherit parent environment |
| `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` | Scrub env vars from subprocesses |
| `CLAUDE_CODE_SHELL` | Override shell binary |
| `CLAUDE_CODE_SHELL_PREFIX` | Prefix for shell commands |
| `CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR` | Show sandbox indicator in bash |

### 15.8 Plugins & Marketplace

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_PLUGIN_CACHE_DIR` | Override plugin cache directory |
| `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS` | Git timeout for plugin installs |
| `CLAUDE_CODE_PLUGIN_SEED_DIR` | Seed directory for plugins |
| `CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE` | Use zip cache for plugins |
| `CLAUDE_CODE_USE_COWORK_PLUGINS` | Use cowork plugins directory |
| `CLAUDE_CODE_SYNC_PLUGIN_INSTALL` | Synchronous plugin install |
| `CLAUDE_CODE_SYNC_PLUGIN_INSTALL_TIMEOUT_MS` | Timeout for sync plugin install |

### 15.9 Remote & Networking

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_REMOTE` | Enable remote mode |
| `CLAUDE_CODE_REMOTE_ENVIRONMENT_TYPE` | Remote environment type |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | Remote memory directory |
| `CLAUDE_CODE_REMOTE_SEND_KEEPALIVES` | Send keepalive pings |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | Remote session ID |
| `CLAUDE_CODE_PROXY_RESOLVES_HOSTS` | Proxy handles host resolution |
| `CLAUDE_CODE_SSE_PORT` | SSE server port |
| `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` | Auth file descriptor for WS |

### 15.10 Identity & Org

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_USER_EMAIL` | Override user email |
| `CLAUDE_CODE_ACCOUNT_UUID` | Override account UUID |
| `CLAUDE_CODE_ACCOUNT_TAGGED_ID` | Override tagged account ID |
| `CLAUDE_CODE_ORGANIZATION_UUID` | Override organization UUID |
| `CLAUDE_CODE_TAGS` | Billing/attribution tags |
| `CLAUDE_CODE_ENTRYPOINT` | Entrypoint label (used in billing header) |
| `CLAUDE_CODE_CONTAINER_ID` | Container identifier |

---

## 16. GrowthBook Integration

### How flags are fetched

1. At startup, `mX1()` creates a GrowthBook client with `remoteEval: true` pointed at `https://api.anthropic.com/`.
2. `init({ timeout: 5000 })` is called. On success, features are processed by `_I7()`.
3. `_I7()` normalizes the feature payload: flags without `defaultValue` have it set from `value`. Experiment assignments are recorded in `t56`.
4. `YI7()` persists the processed features to `settings.json` as `cachedGrowthBookFeatures`.
5. Refresh callbacks (`HH8` Set) are fired via `dB6()`.

### Caching behavior

- **Disk cache** (`P8().cachedGrowthBookFeatures`): read synchronously at startup before network fetch completes. Stale by at most one session (refreshed at next startup).
- **In-memory cache** (`ER` Map): the active GrowthBook runtime values. Updated after `init()` and on periodic refresh.
- **Local override** (`KI7`): hardcoded at build time, highest priority, bypasses all remote logic.

### Periodic refresh

`HI7()` sets up periodic GrowthBook refresh. `pX1()` stops it. `$I7()` forces an immediate refresh. `e56()` refreshes after auth state changes.

### A/B testing implications

- Every flag served from a GrowthBook experiment gets an exposure event logged to the 1P telemetry pipeline.
- `t56` Map tracks `{ experimentId, variationId }` per flag.
- `xX1` Set prevents duplicate exposure logging per session.
- Exposure events include `userAttributes` with device ID, session ID, org UUID, account UUID, subscription type, rate limit tier.

### Local override injection

The `KI7` object (returned by `kP6()`) is consulted first. In the extracted source, it appears to be populated via the `Io3(flag, value)` function (which is a stub returning `undefined` in 2.1.81 — the public override API is disabled). However, the infrastructure allows injecting overrides by setting `KI7` at process startup.

---

## 17. Complete Feature Flag Reference Table

All flags found via `l8("tengu_*")` in the v2.1.81 source (85 unique flags).

| Flag | Type | Default | Purpose | Kill Switch |
|------|------|---------|---------|-------------|
| `tengu_1p_event_batch_config` | Object | `{}` | 1P OTEL event batching config | No |
| `tengu_amber_flint` | Boolean | `true` | Agent teams gate | Yes (→false) |
| `tengu_amber_prism` | Boolean | `false` | Model suffix decoration | No |
| `tengu_amber_quartz_disabled` | Boolean | `false` | Remote Control kill-switch (negated) | Yes (→true) |
| `tengu_amber_wren` | Object | `{}` | Tool result size/token limits | No |
| `tengu_attribution_header` | Boolean | `true` | Billing attribution header | Yes (→false) |
| `tengu_auto_background_agents` | Boolean | `false` | Auto background agent spawning | No |
| `tengu_auto_mode_config` | Object | `{}` | Auto mode configuration | No |
| `tengu_basalt_3kr` | Boolean | `false` | Unknown (3kr suffix = likely internal code) | No |
| `tengu_bramble_lintel` | Mixed | `null/1` | Unknown; null default + numeric gate | No |
| `tengu_bridge_repl_v2` | Boolean | `false` | Bridge REPL v2 | No |
| `tengu_bridge_system_init` | Boolean | `false` | Bridge system initialization | No |
| `tengu_ccr_bridge` | Boolean | `false` | CCR bridge (Claude Code Remote) | No |
| `tengu_ccr_bundle_max_bytes` | Number | `VR_` const | CCR bundle max size | No |
| `tengu_chomp_inflection` | Boolean | `true` | Context inflection behavior | No |
| `tengu_chrome_auto_enable` | Boolean | `false` | Chrome extension auto-enable | No |
| `tengu_cicada_nap_ms` | Number | `0` | Delay/sleep duration (testing?) | No |
| `tengu_cobalt_frost` | Boolean | `false` | Unknown | No |
| `tengu_cobalt_lantern` | Boolean | `false` | Remote session GitHub connectivity | No |
| `tengu_collage_kaleidoscope` | Boolean | `true` | Unknown (default true) | No |
| `tengu_compact_cache_prefix` | Boolean | `false` | Cache prefix during compaction | No |
| `tengu_compact_streaming_retry` | Boolean | `false` | Compaction streaming retry | No |
| `tengu_copper_bridge` | Boolean | `false` | Unknown bridge feature | No |
| `tengu_coral_fern` | Boolean | `false` | Memory directory loading | No |
| `tengu_cork_m4q` | Boolean | `false` | Unknown (m4q internal code) | No |
| `tengu_defer_all_bn4` | Boolean | `true` | Defer all (bn4 internal code) | No |
| `tengu_defer_caveat_m9k` | Boolean | `false` | Defer caveat (m9k internal code) | No |
| `tengu_destructive_command_warning` | Boolean | `false` | Show warning for destructive CLI commands | No |
| `tengu_disable_streaming_to_non_streaming_fallback` | Boolean | `false` | Disable streaming→non-streaming fallback | No |
| `tengu_event_sampling_config` | Object | `{}` | Event sampling rates | No |
| `tengu_fgts` | Boolean | `false` | Fine-grained tool streaming (fgts) | No |
| `tengu_glacier_2xr` | Boolean | `false` | Unknown (2xr internal code) | No |
| `tengu_granite_whisper` | Boolean | `false` | Unknown | No |
| `tengu_grey_step2` | Object | `au7` | Ultrathink configuration | No |
| `tengu_grey_wool` | Boolean | `true` | Legacy model name remapping | Yes (→false) |
| `tengu_harbor` | Boolean | `false` | MCP channels feature | No |
| `tengu_harbor_ledger` | Array | `[]` | MCP channel allowlist | No |
| `tengu_harbor_permissions` | Boolean | `false` | MCP channel permissions gate | No |
| `tengu_hawthorn_steeple` | Boolean | `false` | Tool result persistence strategy | No |
| `tengu_hawthorn_window` | Mixed | `null` | Tool result persistence window | No |
| `tengu_herring_clock` | Boolean | `false` | Team memory directory | No |
| `tengu_immediate_model_command` | Boolean | `false` | Immediate model command | No |
| `tengu_kairos_brief` | Boolean | `false` | Brief tool mode | No |
| `tengu_kairos_brief_config` | Object | `RVq` | Brief mode configuration | No |
| `tengu_kairos_cron` | Boolean | `true` | Scheduled tasks (cron) | Yes (→false) |
| `tengu_keybinding_customization_release` | Boolean | `false` | Keybinding customization feature | No |
| `tengu_lean_cast` | Boolean | `false` | Lean output casting | No |
| `tengu_marble_anvil` | Boolean | `false` | Clear thinking mode | No |
| `tengu_marble_sandcastle` | Boolean | `false` | Fast mode native binary gate | Conditional |
| `tengu_mcp_elicitation` | Boolean | `false` | MCP elicitation feature | No |
| `tengu_miraculo_the_bard` | Boolean | `false` | Unknown (Miraculo = likely AI persona feature) | No |
| `tengu_miraculo_the_bard2` | Boolean | `false` | Miraculo v2 | No |
| `tengu_moth_copse` | Boolean | `false` | Unknown | No |
| `tengu_onyx_plover` | Mixed | `null` | Unknown; null default with `.enabled` check | No |
| `tengu_paper_halyard` | Boolean | `false` | Extended CLAUDE.md parsing | No |
| `tengu_passport_quail` | Boolean | `false` | Memory directory filtering | No |
| `tengu_pebble_leaf_prune` | Boolean | `false` | Unknown pruning behavior | No |
| `tengu_penguins_off` | String/null | `null` | Fast mode kill-switch (message = error shown) | Yes |
| `tengu_permission_explainer` | Boolean | `false` | Permission explanation in UI | No |
| `tengu_pewter_ledger` | String/null | `null` | Transcript truncation strategy | No |
| `tengu_pid_based_version_locking` | Boolean | `false` | PID-based version locking | No |
| `tengu_plan_mode_interview_phase` | Boolean | `false` | Plan mode interview phase | No |
| `tengu_plum_vx3` | Boolean | `false` | Unknown (vx3 internal code) | No |
| `tengu_prompt_cache_1h_config` | Object | `{}` | 1-hour prompt cache allowlist | No |
| `tengu_pr_status_cli` | Boolean | `false` | PR status in CLI footer | No |
| `tengu_quartz_lantern` | Boolean | `false` | Unknown Quartz variant | No |
| `tengu_quiet_fern` | Boolean | `false` | Unknown quiet mode | No |
| `tengu_quiet_hollow` | Boolean | `false` | Thinking summaries for deep search | No |
| `tengu_remote_backend` | Boolean | `false` | Remote backend connectivity | No |
| `tengu_satin_quoll` | String | internal | Per-message tool budget config key | No |
| `tengu_scarf_coffee` | Boolean | `false` | Extended thinking beta variant | No |
| `tengu_sepia_heron` | Boolean | `false` | Unknown | No |
| `tengu_session_memory` | Boolean | `false` | Session memory feature | No |
| `tengu_slate_heron` | Mixed | `Ke9` | Unknown (complex default) | No |
| `tengu_slate_prism` | Boolean | `true` | Unknown (default true) | No |
| `tengu_sm_compact` | Boolean | `false` | Session memory compaction | No |
| `tengu_surreal_dali` | Boolean | `false` | Remote sessions (alternate gate) | No |
| `tengu_swinburne_dune` | Boolean | `false` | Memory directory config selector | No |
| `tengu_system_prompt_global_cache` | Boolean | `false` | Global system prompt caching | No |
| `tengu_tern_alloy` | String | `"off"` | A/B variant (copy_b vs off) | No |
| `tengu_tide_elm` | String | `"off"` | A/B variant (copy_b vs off) | No |
| `tengu_tight_weave` | Boolean | `true` | Concise response format instructions | No |
| `tengu_tool_input_aliasing` | Boolean | `false` | Tool input parameter aliasing | No |
| `tengu_tool_search_unsupported_models` | Mixed | `null` | Override for tool search on unsupported models | No |
| `tengu_trace_lantern` | Boolean | `false` | Debug tracing | No |
| `tengu_tst_hint_m7r` | Boolean | `false` | TST hint (m7r internal code) | No |
| `tengu_tst_kx7` | Boolean | `false` | TST variant (kx7 internal code) | No |
| `tengu_turtle_carbon` | Boolean | `true` | Ultrathink detection | Yes (→false) |
| `tengu_vscode_cc_auth` | Boolean | `false` | VS Code Claude Code auth integration | No |

---

## Appendix A: Flag Loading Flow Diagram

```
claude-code process starts
         │
         ▼
   kP6() — KI7 object
   (local override: highest priority)
         │
         │ not in KI7?
         ▼
   NP6() — session override
   [DISABLED in 2.1.81: returns undefined]
         │
         │ not in session override?
         ▼
   ed() — is 1P event logging enabled?
   [if false → return default value]
         │
         │ enabled?
         ▼
   t56.has(flag)?
   ├── YES → cB6(flag) — log experiment exposure
   └── NO  → TP6.add(flag) — queue for deferred exposure
         │
         ▼
   ER.has(flag)?
   ├── YES → return ER.get(flag)  [in-memory GrowthBook cache]
   └── NO  →
         ▼
   P8().cachedGrowthBookFeatures?.[flag]
   ├── value found → return value  [disk cache]
   └── undefined  → return defaultValue


GrowthBook async init (parallel to flag reads):
  mX1() creates GrowthBook client
    → init({ timeout: 5000 })
    → on success: _I7() processes features
      → populates ER Map
      → records experiments in t56
      → YI7() writes cachedGrowthBookFeatures to disk
      → dB6() fires refresh callbacks
    → on failure: uses disk cache for that session
```

---

## Appendix B: Dynamic Config Flags (`LG()`)

Some flags use `LG()` (getDynamicConfig) instead of `l8()`. These return structured objects:

| Config Key | Purpose |
|-----------|--------|
| `tengu_1p_event_batch_config` | OTEL log batching parameters |
| `tengu_event_sampling_config` | Per-event telemetry sampling rates |
| `tengu_sm_compact_config` | Session memory compaction config |
| `tengu_kairos_brief_config` | Brief mode tool configuration |
| `tengu_auto_mode_config` | Auto mode parameters, two-stage classifier, JSONL transcript |
| `tengu_amber_wren` | Tool result limit overrides |
| `tengu_grey_step2` | Ultrathink settings (enabled, dialog copy) |
| `tengu_prompt_cache_1h_config` | Prompt cache 1h allowlist |
| `tengu_slate_heron` | Unknown structured config |

---

## Appendix C: Legacy Model Remap Table

When `tengu_grey_wool` is `true` (default) and the user specifies one of these model names on `firstParty` auth, it silently remaps to the current default opus:

| Legacy Name | Remaps To |
|-------------|----------|
| `claude-opus-4-20250514` | `claude-opus-4-6` |
| `claude-opus-4-1-20250805` | `claude-opus-4-6` |
| `claude-opus-4-0` | `claude-opus-4-6` |
| `claude-opus-4-1` | `claude-opus-4-6` |

If the user's current session has 1M context active (`sH()` = true) and the model being remapped supports 1M context (`_Y1()`), the `[1m]` suffix is appended: `claude-opus-4-6[1m]`.

Disable remapping: `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP=1` or set `tengu_grey_wool` to `false` server-side.

---

*Reference compiled from Claude Code v2.1.81 source (`cli.js` 619,056 lines, build 2026-03-20T21:25:42Z). All line numbers reference the development source `.ts` files in `src/`.*
