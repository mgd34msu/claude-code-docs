# Auto Mode Reference — Claude Code CLI v2.1.81

Generated from source: `src/core/ys6-1.ts`, `src/core/automodeoptindialog-1.ts`,
`src/core/automodedefaultshandler-1.ts`, `src/core/verifyautomodegateaccess-1.ts`,
`src/core/bypasspermissionsmodedialog-1.ts`, `src/core/setautomodeflagcli-1.ts`,
`src/core/permission_modes-2.ts`

---

## Table of Contents

1. [Overview](#1-overview)
2. [Permission Modes — Complete List](#2-permission-modes--complete-list)
3. [Activation](#3-activation)
4. [Opt-In Dialog](#4-opt-in-dialog)
5. [Two-Stage Classifier](#5-two-stage-classifier)
6. [Configuration](#6-configuration)
7. [Safety and Dangerous Rule Stripping](#7-safety-and-dangerous-rule-stripping)
8. [Circuit Breaker](#8-circuit-breaker)
9. [Gate Access and Unavailability](#9-gate-access-and-unavailability)
10. [Mode Transition State Machine](#10-mode-transition-state-machine)
11. [Decision Flow and Precedence](#11-decision-flow-and-precedence)
12. [Admin and Policy Controls](#12-admin-and-policy-controls)
13. [Bypass Permissions Mode](#13-bypass-permissions-mode)
14. [Dream Mode and Background Agents](#14-dream-mode-and-background-agents)
15. [Telemetry Events](#15-telemetry-events)
16. [Quick Reference Tables](#16-quick-reference-tables)

---

## 1. Overview

Auto Mode is an ML-powered permission classifier integrated into Claude Code CLI v2.1.81 that
automatically decides whether to approve or deny tool invocations on the user's behalf, without
requiring an interactive prompt for each action.

**Source:** `src/core/verifyautomodegateaccess-1.ts` (Module: jyq, lines 544562–545114)

### What It Is

- A two-stage system: first, rule-based matching; second, an LLM classifier model
- Not the same as `dontAsk` mode, which passively skips permission prompts for everything
- Auto mode can actively *deny* dangerous operations or *escalate* to ask when uncertain
- Requires explicit user opt-in before first use
- Gated behind a Statsig feature flag (`tengu_auto_mode_config.enabled`)

### What It Is Not

- Auto mode is not a blanket approval mechanism. The classifier may return `shouldBlock: true`
  for dangerous tool calls even if no explicit deny rule matches.
- Auto mode is not always available. It requires a compatible model (checked via `DP6()`), the
  circuit breaker must not be active, and settings must not explicitly disable it.
- Auto mode is separate from `bypassPermissions` mode. `bypassPermissions` entirely skips all
  permission checks; auto mode runs an active classifier.

### Architecture Summary

```
Tool invocation
     |
     v
[Stage 1: Rule Matching]
  allow[] ──> auto-approve
  soft_deny[] ──> require confirmation
  no match ──> Stage 2
     |
     v
[Stage 2: LLM Classifier (kG8)]
  shouldBlock: false ──> allow
  shouldBlock: true  ──> deny / ask
  unavailable        ──> fallback to ask
```

---

## 2. Permission Modes — Complete List

**Source:** `src/core/ys6-1.ts` lines 13–15

```typescript
// ys6-1.ts line 13
((x48 = ["acceptEdits", "bypassPermissions", "default", "dontAsk", "plan"]),
  (QPA = [...x48, "auto"]),   // INTERNAL_PERMISSION_MODES — includes "auto"
  (nW = QPA));                // PERMISSION_MODES — the complete runtime list
```

The array `x48` is exported as `EXTERNAL_PERMISSION_MODES` (modes exposed to users in docs/UI).
The array `QPA` is `INTERNAL_PERMISSION_MODES`. The array `nW` is `PERMISSION_MODES` (all valid
runtime values).

**Source:** `src/core/permission_modes-2.ts` lines 106–109

```typescript
// permission_modes-2.ts lines 106-109
N8(Pa8, {
  PERMISSION_MODES: () => nW,
  INTERNAL_PERMISSION_MODES: () => QPA,
  EXTERNAL_PERMISSION_MODES: () => x48,
```

| Mode | Exported to Users | Description |
|------|-------------------|-------------|
| `acceptEdits` | Yes | Auto-accept file edits only; prompts for other tool types |
| `bypassPermissions` | Yes | Skip all permission checks entirely (dangerous) |
| `default` | Yes | Standard mode — ask user for approval on each tool call |
| `dontAsk` | Yes | Do not ask for approval; used internally by agent hooks |
| `plan` | Yes | Plan-only mode — no tool execution permitted |
| `auto` | No (internal) | ML classifier decides approval/denial per tool call |

Note: `auto` is not in `EXTERNAL_PERMISSION_MODES` (`x48`). It is added exclusively to
`INTERNAL_PERMISSION_MODES` (`QPA`) at line 14 of `ys6-1.ts`. This means it does not appear
in user-facing documentation derived from `x48`.

### Mode Validation

The function `RC(A)` in `ys6-1.ts` (line 26) validates a mode string:

```typescript
// ys6-1.ts line 26
function RC(A) {
  return nW.includes(A) ? A : "default";
}
```

If an unrecognized string is passed as a mode, it falls back to `"default"`.

### dontAsk Behavior Detail

`dontAsk` mode is handled in the main permission flow as a passthrough:

```typescript
// src/core/filesystem-1.ts line 4479
if (q.mode === "dontAsk")
  return {
    behavior: "passthrough",
    message: "DontAsk mode is handled in main permission flow",
  };
```

When a session is in `dontAsk`, mode-transition logic maps it to `"default"`
(`src/core/config-1.ts` line 11634). `dontAsk` is primarily used by internal agent hooks
that spawn sub-agents without requiring interactive permission prompts.

---

## 3. Activation

**Source:** `src/agents/startdeferredprefetches-1.ts` lines 886–893, 2077;
`src/core/verifyautomodegateaccess-1.ts` lines 261–285

Auto mode can be activated through three different entry points:

### 3.1 CLI Flag: `--enable-auto-mode`

```typescript
// startdeferredprefetches-1.ts line 2077
q.addOption(new AK("--enable-auto-mode", "Opt in to auto mode").hideHelp())
```

This flag is hidden (`.hideHelp()`) — it does not appear in `claude --help` output.
When supplied, it calls `setAutoModeFlagCli(true)` (line 892), which sets the `Sd1` state
variable to `true`. This flag persists only for the current session.

### 3.2 CLI Flag: `--permission-mode auto` (or `-p auto`)

```typescript
// startdeferredprefetches-1.ts line 512
"--permission-mode <mode>"
```

Passing `--permission-mode auto` (or its short form `-p auto`) sets `permissionModeCli` to
`"auto"`. The function `initialPermissionModeFromCLI()` (`verifyautomodegateaccess-1.ts`
line 261) processes this:

```typescript
// verifyautomodegateaccess-1.ts lines 261-285
function vt1({ permissionModeCli: A, dangerouslySkipPermissions: q }) {
  // ...
  if (A) {
    let j = RC(A);      // validate mode string
    if (j === "auto")
      if (w)            // w = circuit breaker active
        V("auto mode circuit breaker active (cached) — falling back to default", ...);
      else O.push("auto");
    else O.push(j);
  }
  // ...
}
```

If the circuit breaker is active at startup, auto mode is silently downgraded to `"default"`.

### 3.3 Settings: `permissions.defaultMode`

```json
// ~/.claude/settings.json
{
  "permissions": {
    "defaultMode": "auto"
  }
}
```

This is set programmatically when the user chooses "Yes, and make it my default mode" in the
opt-in dialog (`automodeoptindialog-1.ts` line 50). The `initialPermissionModeFromCLI` function
also reads this value:

```typescript
// verifyautomodegateaccess-1.ts lines 84-103
if (K.permissions?.defaultMode) {
  let j = K.permissions.defaultMode;
  if (
    a6(process.env.CLAUDE_CODE_REMOTE) &&
    !["acceptEdits", "plan", "default"].includes(j)
  ) {
    // CLAUDE_CODE_REMOTE: warn and ignore auto mode
    V(`settings defaultMode "${j}" is not supported in CLAUDE_CODE_REMOTE — only acceptEdits and plan are allowed`, ...);
    Q("tengu_ccr_unsupported_default_mode_ignored", { mode: j });
  } else if (j === "auto")
    if (w)    // circuit breaker
      V("auto mode circuit breaker active (cached) — falling back to default", ...);
    else O.push("auto");
  else O.push(j);
}
```

### 3.4 Activation Logic Summary

The startup logic (line 891 in `startdeferredprefetches-1.ts`) determines whether to set the
auto mode CLI flag active:

```typescript
// startdeferredprefetches-1.ts line 891
if (
  O.enableAutoMode || Z === "auto" || E6 === "auto" || (!Z && nb8())
)
  MLY?.setAutoModeFlagCli(!0);
```

Where:
- `O.enableAutoMode` — `--enable-auto-mode` CLI flag
- `Z === "auto"` — `--permission-mode auto` CLI flag
- `E6 === "auto"` — resolved effective mode is auto
- `nb8()` — `permissions.defaultMode === "auto"` in settings

---

## 4. Opt-In Dialog

**Source:** `src/core/automodeoptindialog-1.ts` (Module: Ybq, lines 578663–578777)

The opt-in dialog (`AutoModeOptInDialog`) is a React component rendered in the TUI before
auto mode is first used.

### 4.1 Dialog Properties

```typescript
// automodeoptindialog-1.ts line 111
{ title: "Enable auto mode?", color: "warning", onCancel: _ }
```

- **Title:** "Enable auto mode?"
- **Color:** `"warning"` (rendered in warning color, not error red)
- **Security link:** `https://code.claude.com/docs/en/security` (line 71)

### 4.2 Three Options

```typescript
// automodeoptindialog-1.ts lines 79-51 (rendered order: accept-default, accept, decline)
[
  { label: "Yes, and make it my default mode", value: "accept-default" },
  { label: "Yes, enable auto mode",            value: "accept" },
  { label: "No, exit" | "No, go back",         value: "decline" },   // label depends on context
]
```

#### Option 1: "Yes, and make it my default mode" (`accept-default`)

```typescript
// automodeoptindialog-1.ts lines 47-13
case "accept-default": {
  Q("tengu_auto_mode_opt_in_dialog_accept_default", {}),
  vA("userSettings", {
    skipAutoPermissionPrompt: !0,
    permissions: { defaultMode: "auto" },
  }),
  K();
}
```

Effect:
- Writes `skipAutoPermissionPrompt: true` to user settings
- Writes `permissions.defaultMode: "auto"` to user settings
- Calls `onAccept()` to proceed
- Future sessions start in auto mode by default

#### Option 2: "Yes, enable auto mode" (`accept`)

```typescript
// automodeoptindialog-1.ts lines 40-44
case "accept": {
  Q("tengu_auto_mode_opt_in_dialog_accept", {}),
  vA("userSettings", { skipAutoPermissionPrompt: !0 }),
  K();
}
```

Effect:
- Writes `skipAutoPermissionPrompt: true` to user settings
- Does NOT set `permissions.defaultMode`
- Auto mode active for this session only; future sessions must opt in again unless the
  user runs `claude` with `--enable-auto-mode` or `--permission-mode auto`

#### Option 3: "No, exit" / "No, go back" (`decline`)

```typescript
// automodeoptindialog-1.ts line 56
case "decline":
  Q("tengu_auto_mode_opt_in_dialog_decline", {}), _();
```

- Calls `onDecline()`
- The label is contextually "No, exit" (`declineExits: true`) or "No, go back" (line 48)

### 4.3 Dialog Display Lifecycle

The dialog is shown once via a `useEffect` that fires `tengu_auto_mode_opt_in_dialog_shown`:

```typescript
// automodeoptindialog-1.ts line 82-83
function FZY() {
  Q("tengu_auto_mode_opt_in_dialog_shown", {});
}
```

### 4.4 Opt-In Reset Guard

**Source:** `src/core/config-1.ts` lines 12629–12638

A one-time migration runs to clear stale `skipAutoPermissionPrompt` state:

```typescript
// config-1.ts lines 12629-12638
if (P8().hasResetAutoModeOptInForDefaultOffer) return;

// ...
if (q?.skipAutoPermissionPrompt && q?.permissions?.defaultMode !== "auto")
  vA("userSettings", { skipAutoPermissionPrompt: void 0 },  // clear stale flag
// ...
if (K.hasResetAutoModeOptInForDefaultOffer) return K;
return { ...K, hasResetAutoModeOptInForDefaultOffer: !0 };
```

This migration runs once. After running, `hasResetAutoModeOptInForDefaultOffer: true` is
written to user settings so it never runs again. The effect: if a user previously opted in
(session-only) and `skipAutoPermissionPrompt` is set but `defaultMode` is not `"auto"`, the
prompt flag is cleared so they will see the dialog again with the new default-mode offer.

---

## 5. Two-Stage Classifier

**Source:** `src/tools/tool-2.ts` lines 2124–2300; `src/tools/definitions-1.ts` lines 9720–9840

### 5.1 Stage 1 — Rule Matching

Before invoking the LLM classifier, the system evaluates three rule categories configured in
the `autoMode` settings block:

| Rule Category | Effect |
|---------------|--------|
| `allow[]` | Auto-approve tool call immediately; skip LLM classifier |
| `soft_deny[]` | Require user confirmation (block + ask) |
| `environment[]` | Context strings injected into classifier system prompt |

Rules in `allow[]` and `soft_deny[]` are matched against the tool name and arguments. If a
match is found, Stage 2 is skipped.

**Source:** `src/core/automodedefaultshandler-1.ts` lines 142–144

```
- allow: Actions the classifier should auto-approve
- soft_deny: Actions the classifier should block (require user confirmation)
- environment: Context about the user's setup that helps the classifier make decisions
```

#### Fast-Path Shortcuts (Stage 1 Variants)

Before calling the full classifier, two fast-path checks run:

1. **acceptEdits fast-path** (`src/tools/definitions-1.ts` line 9737): If the tool would be
   allowed in `acceptEdits` mode, it is auto-approved without invoking the classifier:
   ```typescript
   Q("tengu_auto_mode_decision", {
     decision: "allowed",
     toolName: Uq(A.name),
     confidence: "high",
     fastPath: "acceptEdits",
   })
   ```

2. **Allowlist fast-path** (`src/tools/definitions-1.ts` line 9761): Tools on the internal
   safe allowlist (`cHY.isAutoModeAllowlistedTool`) are auto-approved:
   ```typescript
   Q("tengu_auto_mode_decision", {
     decision: "allowed",
     toolName: Uq(A.name),
     confidence: "high",
     fastPath: "allowlist",
   })
   ```

### 5.2 Stage 2 — LLM Classifier (`kG8`)

**Source:** `src/tools/tool-2.ts` function `kG8` starting at line 2124

The classifier is invoked as an API call:

```typescript
// tool-2.ts lines 2208-2250
let b = {
  model: E,           // NZ4() — configured classifier model
  max_tokens: 4096 + S,
  system: [{ type: "text", text: $, cache_control: ... }],  // $ = classifier system prompt
  skipSystemPromptPrefix: !0,
  temperature: 0,
  thinking: R,
  messages: [...J, { role: "user", content: W }],
  tools: [ie9],
  tool_choice: { type: "tool", name: bd6 },
  maxRetries: Pd6(),
  signal: Y,
  querySource: "auto_mode",
};
let g = await Tk(b);
```

#### Classifier Input

- **System prompt:** Classifier system prompt (accessible via `LZ4()` and displayed by
  `claude auto-mode defaults`)
- **User rules:** `allow[]`, `soft_deny[]`, `environment[]` from settings, replacing defaults
  when present
- **Tool invocation:** Tool name and full argument content, serialized to text
- **Conversation context:** Recent message history (user prompts, assistant turns, tool results)
  truncated to fit context window

#### Classifier Output

The classifier responds with a tool call (forced via `tool_choice`) containing:

```typescript
// tool-2.ts lines 2449-2459
{
  shouldBlock: boolean,  // true = deny/ask, false = allow
  thinking: string,      // reasoning trace
  reason: string,        // human-readable explanation
}
```

#### Handling of Unavailable Classifier

If the classifier is unavailable (e.g., network error, model overload), the system falls back
to asking the user:

```typescript
// tool-2.ts
{
  shouldBlock: !1,
  reason: "Tool declares no classifier-relevant input",
  model: NZ4(),
}
// or on error:
{ unavailable: true, ... }
```

The `unavailable` outcome is reported in telemetry as `tengu_auto_mode_decision` with
`decision: "unavailable"`.

#### No Tool-Use Block Response

If the classifier returns a response with no tool-use block, the system defaults to blocking:

```typescript
// tool-2.ts line 2078
V("Auto mode classifier: No tool use block found", { level: "warn" }),
Q("tengu_auto_mode_outcome", {
  outcome: "parse_failure",
  failureKind: "no_tool_use",
  classifierModel: E,
}),
{
  shouldBlock: !0,
  reason: "Classifier returned no tool use block - blocking for safety",
  ...
}
```

### 5.3 Classifier Telemetry Detail

The `tengu_auto_mode_decision` event carries rich classifier metadata:

```typescript
Q("tengu_auto_mode_decision", {
  decision,                          // "allowed" | "blocked" | "unavailable"
  toolName,
  agentMsgId,
  classifierModel,
  consecutiveDenials,
  totalDenials,
  classifierInputTokens,
  classifierOutputTokens,
  classifierCacheReadInputTokens,
  classifierCacheCreationInputTokens,
  classifierDurationMs,
  classifierSystemPromptLength,
  classifierToolCallsLength,
  classifierToolResultsLength,
  classifierUserPromptsLength,
  sessionInputTokens,
  sessionOutputTokens,
  classifierCostUSD,
  classifierStage,                   // which stage (1 or 2)
  // ...stage1/stage2 usage breakdown fields
})
```

---

## 6. Configuration

**Source:** `src/core/automodedefaultshandler-1.ts`

### 6.1 Settings Schema

Auto mode configuration lives in the `autoMode` key of the settings file:

```json
{
  "autoMode": {
    "allow": ["Read", "Bash --dry-run"],
    "soft_deny": ["Write", "Edit"],
    "environment": ["project=python", "vcs=git"]
  }
}
```

- `allow`: Array of tool-name strings (optionally with argument qualifiers) that Stage 1
  should auto-approve. Custom values *replace* (not extend) the built-in defaults.
- `soft_deny`: Array of tool-name strings that Stage 1 should block and prompt the user about.
  Custom values *replace* the built-in defaults.
- `environment`: Array of free-form context strings injected into the classifier system prompt.
  Custom values *replace* the built-in defaults.

#### Rule Merge Logic

```typescript
// automodedefaultshandler-1.ts lines 36-38
{
  allow: A?.allow?.length ? A.allow : q.allow,           // user rules OR defaults
  soft_deny: A?.soft_deny?.length ? A.soft_deny : q.soft_deny,
  environment: A?.environment?.length ? A.environment : q.environment,
}
```

Where `A` = user-configured rules (`WS6()`) and `q` = built-in defaults (`TG8()`). If any
category in user rules is empty, the built-in default for that category is used.

### 6.2 CLI Commands

Three sub-commands are available under `claude auto-mode`:

**Source:** `src/core/automodedefaultshandler-1.ts` exports:
- `autoModeDefaultsHandler` (`KLY`) — `claude auto-mode defaults`
- `autoModeConfigHandler` (`_LY`) — `claude auto-mode config`
- `autoModeCritiqueHandler` (`zLY`) — `claude auto-mode critique [--model <model>]`

#### `claude auto-mode defaults`

Prints the built-in default rules as JSON:

```typescript
// automodedefaultshandler-1.ts line 29-30
function KLY() {
  Vdq(TG8());  // print defaults as formatted JSON
}
```

#### `claude auto-mode config`

Prints the *effective* configuration — user-defined rules where present, otherwise defaults:

```typescript
// automodedefaultshandler-1.ts lines 32-39
function _LY() {
  let A = WS6(),   // user settings
    q = TG8();     // defaults
  Vdq({
    allow: A?.allow?.length ? A.allow : q.allow,
    soft_deny: A?.soft_deny?.length ? A.soft_deny : q.soft_deny,
    environment: A?.environment?.length ? A.environment : q.environment,
  });
}
```

#### `claude auto-mode critique [--model <model>]`

Uses an LLM to evaluate the user's custom rules against four dimensions:

1. **Clarity** — Is the rule unambiguous? Could the classifier misinterpret it?
2. **Completeness** — Are there gaps or edge cases the rule does not cover?
3. **Conflicts** — Do any of the rules conflict with each other?
4. **Actionability** — Is the rule specific enough for the classifier to act on?

```typescript
// automodedefaultshandler-1.ts lines 40-74
async function zLY(A) {
  let q = WS6();
  if (!( (q?.allow?.length ?? 0) > 0 || ... )) {
    process.stdout.write(`No custom auto mode rules found...`);
    return;
  }
  let _ = A.model ? Y5(A.model) : KK(),  // use specified model or current default
    Y = TG8(),
    z = LZ4(),                           // classifier system prompt
    w = T1A("allow", ...) + T1A("soft_deny", ...) + T1A("environment", ...);
  // ...
  O = await Tk({ querySource: "auto_mode_critique", model: _, system: YLY, ... });
}
```

The critique system prompt (`YLY`, line 138) describes the role: "You are an expert reviewer
of auto mode classifier rules for Claude Code."

The critique shows each user rule alongside the defaults it replaces:

```
## allow (custom rules replacing defaults)
Custom:
- <user rule>

Defaults being replaced:
- <default rule>
```

---

## 7. Safety and Dangerous Rule Stripping

**Source:** `src/core/verifyautomodegateaccess-1.ts` lines 1–197 (functions: `dn`, `T26`,
`Xyq`, `Dyq`, `Pyq`, `Jyq`, `Gt1`)

### 7.1 Why Stripping Exists

Auto mode routes tool calls through an LLM classifier. Certain `allow` rules would bypass the
classifier entirely, defeating the purpose of auto mode. When entering auto mode, rules that
"bypass the classifier" are automatically stripped from the allow list.

As logged:

```
Ignoring dangerous permission ${Y.ruleDisplay} from ${Y.sourceDisplay} (bypasses classifier)
```

### 7.2 Dangerous Pattern Detectors

**`isDangerousBashPermission(toolName, ruleContent)` — `Xyq`** (line 78–92)

Returns `true` (dangerous) if `toolName` is `Bash` (`S7`) AND the rule content matches any
of these patterns:

```typescript
// The two-tier shell command lists:
oHY = [
  "python", "python3", "python2", "node", "deno", "tsx",
  "ruby", "perl", "php", "lua",
  "npx", "bunx",
  "npm run", "yarn run", "pnpm run", "bun run",
  "bash", "sh", "ssh",
]
Hyq = [...oHY, "zsh", "fish", "eval", "exec", "env", "xargs", "sudo", ...]

// Matches any of:
// - rule === "*"
// - rule === "<shell_cmd>"        (exact match)
// - rule === "<shell_cmd>:*"      (colon-wildcard)
// - rule === "<shell_cmd>*"       (suffix-wildcard)
// - rule === "<shell_cmd> *"      (space-wildcard)
// - rule starts with "<shell_cmd> -" AND ends with "*"
```

Additionally, if `ruleContent` is `undefined` or `""` (i.e., `Bash` with no argument
constraint), it is also considered dangerous (line 80–81).

**`isDangerousPowerShellPermission(toolName, ruleContent)` — `Dyq`** (line 93–95)

```typescript
function Dyq(A, q) {
  return !1;  // PowerShell detection not implemented in v2.1.81
}
```

Currently always returns `false` — PowerShell detection is a stub in this version.

**`isDangerousTaskPermission(toolName, ruleContent)` — `Pyq`** (line 96–98)

```typescript
function Pyq(A, q) {
  return sZ(A) === a4;  // returns true if toolName resolves to the Task/agent-spawn tool
}
```

Detects attempts to allow the Task tool (sub-agent spawning). Allowing `Task` unconditionally
would allow auto mode to spawn sub-agents without classifier oversight.

**Combined check — `Jyq`** (line 109–111)

```typescript
function Jyq(A, q) {
  return Xyq(A, q) || Dyq(A, q) || Pyq(A, q);
}
```

### 7.3 Strip Function: `stripDangerousPermissionsForAutoMode` (`dn`)

**Source:** `verifyautomodegateaccess-1.ts` lines 198–221

Called when transitioning *into* auto mode:

```typescript
function dn(A) {
  let q = [];
  for (let [Y, z] of Object.entries(A.alwaysAllowRules)) {
    if (!z) continue;
    for (let w of z) {
      let O = iH(w);  // parse rule
      q.push({ source: Y, ruleBehavior: "allow", ruleValue: O });
    }
  }
  let K = Gt1(q, []);  // find dangerous rules
  if (K.length === 0) return A;  // nothing to strip
  for (let Y of K)
    V(`Ignoring dangerous permission ${Y.ruleDisplay} from ${Y.sourceDisplay} (bypasses classifier)`);
  let _ = {};  // collect stripped rules by source
  for (let Y of K) {
    if (!fyq(Y.source)) continue;  // only strip from user-controllable sources
    (_[Y.source] ??= []).push(p5(Y.ruleValue));
  }
  return { ...Zyq(A, K), strippedDangerousRules: _ };  // store for later restoration
}
```

**Sources eligible for stripping** (`fyq`):
`"userSettings"`, `"projectSettings"`, `"localSettings"`, `"session"`, `"cliArg"`

**Sources NOT stripped** (e.g., `policySettings`, `flagSettings`): their rules are logged as
ignored but not added to `strippedDangerousRules` for restoration.

### 7.4 Restore Function: `restoreDangerousPermissions` (`T26`)

**Source:** `verifyautomodegateaccess-1.ts` lines 222–235

Called when transitioning *out of* auto mode:

```typescript
function T26(A) {
  let q = A.strippedDangerousRules;
  if (!q) return A;  // nothing to restore
  let K = A;
  for (let [_, Y] of Object.entries(q)) {
    if (!Y || Y.length === 0) continue;
    K = gY(K, {
      type: "addRules",
      rules: Y.map(iH),
      behavior: "allow",
      destination: _,
    });
  }
  return { ...K, strippedDangerousRules: void 0 };  // clear stored rules
}
```

---

## 8. Circuit Breaker

**Source:** `src/core/verifyautomodegateaccess-1.ts` lines 430–540;
`src/core/setautomodeflagcli-1.ts` lines 40–48

### 8.1 Gate Key

The circuit breaker state is controlled via a Statsig dynamic config:

```typescript
// verifyautomodegateaccess-1.ts line 433
let K = await yR("tengu_auto_mode_config", {}),
  _ = Nt1(K?.enabled),
```

The gate key is `tengu_auto_mode_config`. Its `enabled` field is read and normalized by
`Nt1()` (line 309):

```typescript
function Nt1(A) {
  if (A === "enabled" || A === "disabled" || A === "opt-in") return A;
  return _jY;  // _jY = "disabled" (line 357)
}
```

| `enabled` value | Meaning |
|------|--------------------|
| `"enabled"` | Auto mode is globally available |
| `"opt-in"` | Auto mode available only if user has opted in (`hasAutoModeOptInAnySource()`) |
| `"disabled"` | Circuit breaker active — auto mode unavailable |
| (any other / missing) | Defaults to `"disabled"` |

### 8.2 Circuit Breaker State

The circuit broken state is persisted in-process via `setAutoModeCircuitBroken(bool)` /
`isAutoModeCircuitBroken()` in `setautomodeflagcli-1.ts`:

```typescript
// setautomodeflagcli-1.ts lines 37-42, 46-48
function UB_(A) { Cd1 = A; }          // setAutoModeCircuitBroken
function QB_() { return Cd1; }          // isAutoModeCircuitBroken
var Cd1 = !1;                           // initial state: false (not broken)
```

The state is set during `verifyAutoModeGateAccess()` (line 239):

```typescript
// verifyautomodegateaccess-1.ts line 239
IF?.setAutoModeCircuitBroken(_ === "disabled" || Y);  // Y = kt1() = disabled in settings
```

### 8.3 Fast-Path Cache

The cached (synchronous) state is checked via `ib8()` and `ea6()`, which read a Statsig
session-level cache without awaiting the async gate check:

```typescript
// verifyautomodegateaccess-1.ts lines 313-320
function ea6() {                    // getAutoModeEnabledState (async read)
  let A = l8("tengu_auto_mode_config", {});
  return Nt1(A?.enabled);
}
function ib8() {                    // getAutoModeEnabledStateIfCached
  let A = l8("tengu_auto_mode_config", Myq);
  if (A === Myq) return;            // returns undefined if not cached yet
  return Nt1(A?.enabled);
}
```

The cached value (`ib8()`) is used during startup mode initialization (`vt1`) to apply the
circuit breaker check synchronously:

```typescript
// verifyautomodegateaccess-1.ts line 69
let w = ib8() === "disabled",
```

If the cache says `"disabled"`, auto mode is blocked immediately at startup even before
the async Statsig call completes.

### 8.4 Behavior When Circuit Broken

- `isAutoModeGateEnabled()` returns `false`
- Attempts to use `--permission-mode auto` log a warning and fall back to `"default"`
- Attempts to use `--enable-auto-mode` are silently ignored at the flag-processing stage
- `settings.defaultMode: "auto"` is similarly downgraded to `"default"` with a warning
- `getAutoModeUnavailableReason()` returns `"circuit-breaker"`
- `getAutoModeUnavailableNotification()` returns `"auto mode temporarily unavailable"`

---

## 9. Gate Access and Unavailability

**Source:** `src/core/verifyautomodegateaccess-1.ts` lines 297–315

### 9.1 `isAutoModeGateEnabled` (`WN`)

This is the primary gate function. It checks three conditions:

```typescript
// verifyautomodegateaccess-1.ts lines 297-301
function WN() {
  if (IF?.isAutoModeCircuitBroken() ?? !1) return !1;   // (1) circuit breaker not active
  if (kt1()) return !1;                                  // (2) not disabled in settings
  if (!DP6(KK())) return !1;                             // (3) model supports auto mode
  return !0;
}
```

| Check | Function | Condition for `false` |
|-------|----------|-----------------------|
| Circuit breaker | `isAutoModeCircuitBroken()` | Statsig gate returned `"disabled"` |
| Settings disabled | `kt1()` | `disableAutoMode === "disable"` in settings or policy |
| Model support | `DP6(KK())` | Current model does not pass the DP6 model-support check |

### 9.2 `getAutoModeUnavailableReason` (`k26`)

```typescript
// verifyautomodegateaccess-1.ts lines 303-308
function k26() {
  if (kt1()) return "settings";
  if (IF?.isAutoModeCircuitBroken() ?? !1) return "circuit-breaker";
  if (!DP6(KK())) return "model";
  return null;  // null = auto mode is available
}
```

| Reason | Meaning |
|--------|-------------------------------|
| `"settings"` | Explicitly disabled via `disableAutoMode: "disable"` |
| `"circuit-breaker"` | Statsig gate has circuit-broken auto mode |
| `"model"` | Current model does not support auto mode |
| `null` | Auto mode is available |

### 9.3 `getAutoModeUnavailableNotification` (`kA6`)

```typescript
// verifyautomodegateaccess-1.ts lines 220-233
function kA6(A) {
  let q;
  switch (A) {
    case "settings":       q = "auto mode disabled by settings"; break;
    case "circuit-breaker": q = "auto mode temporarily unavailable"; break;
    case "model":          q = "auto mode unavailable for this model"; break;
  }
  return q;
}
```

### 9.4 `verifyAutoModeGateAccess` (`Pn6`) — Async Check

**Source:** `verifyautomodegateaccess-1.ts` lines 235–286

This function is called at session startup to perform the async Statsig check and apply any
necessary context updates:

```typescript
async function Pn6(A, q) {
  let K = await yR("tengu_auto_mode_config", {}),    // async Statsig fetch
    _ = Nt1(K?.enabled),
    Y = kt1();                                         // settings disabled?
  IF?.setAutoModeCircuitBroken(_ === "disabled" || Y);
  let z = KK(),                                        // current model
    O = DP6(z) && !w,                                  // model supports auto mode?
    $ = !1;
  if (_ !== "disabled" && !Y && O)
    $ = _ === "enabled" || dS8();                      // available = enabled OR user opted in
  let H = _ !== "disabled" && !Y && O,
    j = IF?.getAutoModeFlagCli() ?? !1;
  if (H) return { updateContext: (Z) => J(Z, $) };    // gate open: update context availability
  // gate closed: determine reason and handle active sessions
  // ...
  return { updateContext: D, notification: X };
}
```

The function returns an `updateContext` function that is applied to the current
`toolPermissionContext`. If auto mode is unavailable and the current mode is `"auto"`, the
context is transitioned back to `"default"` with a notification message.

---

## 10. Mode Transition State Machine

**Source:** `src/core/verifyautomodegateaccess-1.ts` function `cn` (line 237);
`src/core/setautomodeflagcli-1.ts`

### 10.1 `transitionPermissionMode` (`cn`)

```typescript
// verifyautomodegateaccess-1.ts lines 237-253
function cn(A, q, K) {
  if (A === q) return K;           // no-op: same mode
  if ((PU(A, q), bg8(A, q, K.prePlanMode), A === "plan" && q !== "plan"))
    dN(!0);                        // exiting plan mode: set some flag
  {
    if (q === "plan" && A !== "plan") return Ik6(K);   // entering plan mode
    let _ = A === "auto" || (A === "plan" && K.prePlanMode === "auto"),
      Y = q === "auto";
    if (Y && !_) {                 // transitioning INTO auto mode
      if (!WN())
        throw Error("Cannot transition to auto mode: gate is not enabled");
      (IF?.setAutoModeActive(!0), (K = dn(K)));        // activate + strip dangerous rules
    } else if (_ && !Y)           // transitioning OUT OF auto mode
      (IF?.setAutoModeActive(!1), (K = T26(K)));       // deactivate + restore rules
  }
  if (A === "plan" && q !== "plan" && K.prePlanMode)
    return { ...K, prePlanMode: void 0 };              // clear prePlanMode on plan exit
  return K;
}
```

Parameters: `A` = from mode, `q` = to mode, `K` = current context.

### 10.2 Entering Auto Mode

1. `WN()` gate check — throws if gate is not enabled
2. `IF?.setAutoModeActive(!0)` — sets `isAutoModeActive` to `true`
3. `dn(K)` — strips dangerous rules and returns new context with `strippedDangerousRules`

### 10.3 Leaving Auto Mode

1. `IF?.setAutoModeActive(!1)` — sets `isAutoModeActive` to `false`
2. `T26(K)` — restores dangerous rules and returns new context with
   `strippedDangerousRules: void 0`

### 10.4 Plan Mode Interaction

Plan mode saves the current permission mode for restoration when plan mode exits.
This is handled by `prepareContextForPlanMode` (`Ik6`):

```typescript
// verifyautomodegateaccess-1.ts lines 348-355
function Ik6(A) {
  let q = A.mode;
  if (q === "plan") return A;                    // already in plan: no-op
  if (q === "auto") return { ...A, prePlanMode: "auto" };   // save auto
  if (nb8() && WN() && q !== "bypassPermissions")
    return (IF?.setAutoModeActive(!0), { ...dn(A), prePlanMode: "auto" });  // settings default is auto
  return { ...A, prePlanMode: q };               // save other mode
}
```

- If the session was in auto mode before entering plan, `prePlanMode: "auto"` is saved
- When plan mode exits, `cn` restores auto mode by reading `prePlanMode`
- The expression `A === "plan" && K.prePlanMode === "auto"` in `cn` (line 243) correctly
  identifies "was in auto mode before plan"

### 10.5 State Variables

**Source:** `src/core/setautomodeflagcli-1.ts` lines 46–48

```typescript
// setautomodeflagcli-1.ts lines 46-48
var hd1 = !1,   // isAutoModeActive: is auto mode currently the active permission mode?
    Sd1 = !1,   // autoModeFlagCli: was --enable-auto-mode or -p auto passed?
    Cd1 = !1;   // isAutoModeCircuitBroken: has the circuit breaker tripped?
```

| Variable | Getter | Setter | Meaning |
|----------|--------|--------|---------|
| `hd1` | `isAutoModeActive()` / `gB_` | `setAutoModeActive(bool)` / `Id1` | Active mode is currently `"auto"` |
| `Sd1` | `getAutoModeFlagCli()` / `FB_` | `setAutoModeFlagCli(bool)` / `pB_` | CLI flag `--enable-auto-mode` was passed |
| `Cd1` | `isAutoModeCircuitBroken()` / `QB_` | `setAutoModeCircuitBroken(bool)` / `UB_` | Gate returned `"disabled"` |

All three start as `false`. A `_resetForTesting()` function (`dB_`, line 43) resets all
three to `false` for use in tests.

---

## 11. Decision Flow and Precedence

**Source:** `src/tools/definitions-1.ts` lines 9440–9840;
`src/tools/tool-2.ts` lines 1700–2340

### 11.1 Decision Values

The classifier and rule-matching system produce one of four behavior values:

| Value | Meaning |
|-------|---------|
| `"allow"` | Tool call is approved; execute without prompting |
| `"deny"` | Tool call is rejected; do not execute |
| `"ask"` | Present to user for manual approval |
| `"passthrough"` | Not handled by this layer; defer to next handler |

In classifier terms, `shouldBlock: false` maps to `"allow"` and `shouldBlock: true` maps to
`"deny"` / `"ask"` depending on headless context.

### 11.2 Decision Precedence

When multiple permission hooks or handlers return decisions, the precedence is:

```
deny  >  ask  >  allow  >  passthrough
```

A `deny` from any handler overrides any `allow` from another. `passthrough` is only used
when no handler has an opinion — it defers to the next layer.

### 11.3 `PermissionRequest` Hook

Organizations can audit or override classifier decisions via the `PermissionRequest` hook,
which fires before the user is prompted. The pending requests are tracked in:

```typescript
// src/tools/tool-2.ts line 10701
pendingPermissionRequests = new Map();
```

### 11.4 Denial Limit Tracking

**Source:** `src/tools/definitions-1.ts` lines 9440–9480

Auto mode tracks consecutive and total denials to detect classifier feedback loops:

```typescript
// definitions-1.ts lines 9457-9467
Q("tengu_auto_mode_denial_limit_exceeded", {
  limit: O ? "total" : "consecutive",   // "total" or "consecutive"
  mode: $ ? "headless" : "cli",
  messageID,
  consecutiveDenials,
  totalDenials,
  toolName,
})
```

In headless mode (`shouldAvoidPermissionPrompts: true`), exceeding the denial limit
throws an `f_` (agent abort) exception:

```typescript
throw new f_("Agent aborted: too many classifier denials in headless mode");
```

In CLI (interactive) mode, the system logs a warning and falls back to prompting the user.

---

## 12. Admin and Policy Controls

**Source:** `src/core/permission_modes-2.ts`; `src/core/verifyautomodegateaccess-1.ts`

### 12.1 Priority Order

```typescript
// permission_modes-2.ts line 117
return (q.add("policySettings"), q.add("flagSettings"), Array.from(q));
```

The full priority stack (highest to lowest):

1. `policySettings` — enterprise managed settings (highest authority)
2. `flagSettings` — CLI flags
3. `userSettings` — `~/.claude/settings.json`
4. `projectSettings` — shared `.claude/settings.json` in project
5. `localSettings` — gitignored `.claude/settings.local.json`
6. Session / built-in (lowest)

### 12.2 `disableAutoMode` Setting

```typescript
// verifyautomodegateaccess-1.ts lines 290-295
function kt1() {
  let A = PA() || {};
  return (
    A.disableAutoMode === "disable" ||
    A.permissions?.disableAutoMode === "disable"
  );
}
```

Setting either `disableAutoMode: "disable"` at the root level OR
`permissions.disableAutoMode: "disable"` disables auto mode. Both fields are checked.

### 12.3 `CLAUDE_CODE_REMOTE` Environment Variable

When `CLAUDE_CODE_REMOTE` is set (remote/hosted environments), the allowed
`defaultMode` values are restricted:

```typescript
// verifyautomodegateaccess-1.ts lines 87-94
if (
  a6(process.env.CLAUDE_CODE_REMOTE) &&
  !["acceptEdits", "plan", "default"].includes(j)
) {
  V(`settings defaultMode "${j}" is not supported in CLAUDE_CODE_REMOTE — only acceptEdits and plan are allowed`, ...);
  Q("tengu_ccr_unsupported_default_mode_ignored", { mode: j });
}
```

In `CLAUDE_CODE_REMOTE` mode, only `acceptEdits`, `plan`, and `default` are permitted as
`defaultMode`. Auto mode and bypass permissions mode are silently ignored (with a warning
log and telemetry event `tengu_ccr_unsupported_default_mode_ignored`).

### 12.4 `disableBypassPermissionsMode` Setting

```typescript
// verifyautomodegateaccess-1.ts lines 306-316
if (j === "bypassPermissions" && z) {
  if (_)  // _ = Statsig gate tengu_disable_bypass_permissions_mode
    V("bypassPermissions mode is disabled by Statsig gate", ...);
  else
    V("bypassPermissions mode is disabled by settings", ...);
  continue;  // skip, don't set as active mode
}
```

Bypass permissions mode is disabled by either:
- Statsig gate `tengu_disable_bypass_permissions_mode` returning true, OR
- `permissions.disableBypassPermissionsMode: "disable"` in settings

### 12.5 Initialization Priority

```typescript
// verifyautomodegateaccess-1.ts lines 64-124 (vt1 function)
// Priority order for resolving effective permission mode at startup:
// 1. dangerouslySkipPermissions flag -> bypassPermissions
// 2. --permission-mode CLI flag -> explicit mode
// 3. settings.permissions.defaultMode -> configured default
// 4. "default" fallback
```

First non-empty mode in the priority list wins. Bypass permissions mode can be overridden
back out by the `disableBypassPermissionsMode` check (line 106–116).

---

## 13. Bypass Permissions Mode

**Source:** `src/core/bypasspermissionsmodedialog-1.ts` (Module: Wpq, lines 598223–598323)

Bypass Permissions mode is a separate, more aggressive mode from Auto mode.

### 13.1 Behavior

- Skips ALL permission checks — the classifier is not invoked at all
- The `BypassPermissionsModeDialog` component is shown before activation
- The user must explicitly accept responsibility

### 13.2 Warning Dialog

```typescript
// bypasspermissionsmodedialog-1.ts line 94
title: "WARNING: Claude Code running in Bypass Permissions mode"
color: "error"   // rendered in error red (vs. auto mode's "warning" yellow)
```

**Dialog body text (line 67–73):**

```
In Bypass Permissions mode, Claude Code will not ask for your approval before
running potentially dangerous commands.

This mode should only be used in a sandboxed container/VM that has restricted
internet access and can easily be restored if damaged.

By proceeding, you accept all responsibility for actions taken while running
in Bypass Permissions mode.
```

A security documentation link is also shown:
```typescript
url: "https://code.claude.com/docs/en/security"
```

### 13.3 Options

```typescript
// bypasspermissionsmodedialog-1.ts lines 85
[
  { label: "No, exit",    value: "decline" },
  { label: "Yes, I accept", value: "accept" },
]
```

Note the order: "No, exit" is first (default selection), "Yes, I accept" is second.
This is reversed compared to the auto mode opt-in dialog order.

### 13.4 On Accept

```typescript
// bypasspermissionsmodedialog-1.ts lines 43-47
case "accept": {
  Q("tengu_bypass_permissions_mode_dialog_accept", {}),
  vA("userSettings", { skipDangerousModePermissionPrompt: !0 }),
  K();
}
```

The setting `skipDangerousModePermissionPrompt: true` is written to user settings.
This prevents the warning dialog from appearing again in future sessions.

### 13.5 On Decline

```typescript
// bypasspermissionsmodedialog-1.ts line 11
case "decline":
  $K(1);  // exit with code 1
```

Decline causes the process to exit with code 1 (immediate exit, not just going back).

### 13.6 Activation

Bypass permissions is activated via:
- `--dangerously-skip-permissions` CLI flag (maps to `dangerouslySkipPermissions`)
- The internal flag sets mode to `"bypassPermissions"` in `vt1()`:
  ```typescript
  // verifyautomodegateaccess-1.ts line 72
  if (q) O.push("bypassPermissions");
  ```

### 13.7 Statsig Gate

`tengu_disable_bypass_permissions_mode` — when this gate is enabled, bypass permissions
mode is disabled org-wide. An async check also runs after initialization:

```typescript
// verifyautomodegateaccess-1.ts line 337-343
async function Vt1(A) {
  if (!A.isBypassPermissionsModeAvailable) return;
  if (!(await mE8())) return;
  V("bypassPermissions mode is being disabled by Statsig gate (async check)", ...);
  kq(1, "bypass_permissions_disabled");  // force exit code 1
}
```

---

## 14. Dream Mode and Background Agents

**Source:** `src/tools/definitions-1.ts` lines 7760–7870;
`src/vendor/dom.ts` line 73288;
`src/ui/components-1.ts` line 5579

### 14.1 Auto Dream Mode

Auto Dream is a background speculative execution feature where Claude reviews past sessions
and performs unsolicited improvements. It runs as a background agent fork.

**Telemetry events:**

```typescript
// definitions-1.ts lines 7776-7826
Q("tengu_auto_dream_fired", {
  hours_since: Math.round(O),    // hours since last dream run
  sessions_since: H.length,     // number of sessions to review
})

Q("tengu_auto_dream_completed", {
  cache_read,
  cache_created,
  output,
  sessions_reviewed,
})

Q("tengu_auto_dream_failed", {})
```

**UI toggle telemetry:**
```typescript
// ui/components-1.ts line 5579
Q("tengu_auto_dream_toggled", { enabled: N6 })
```

**Dream mode tool constraints** (line 7831–7833):
```
Tool constraints for this run: Bash is restricted to read-only commands
(ls, find, grep, cat, stat, wc, head, tail, and similar). Anything that
writes, redirects to a file, or modifies state will be denied.
```

Dream mode runs with a custom `canUseTool` function (`jF_`) that enforces read-only
Bash restrictions.

### 14.2 Background Agents Feature Flag

```typescript
// vendor/dom.ts line 73288
l8("tengu_auto_background_agents", !1)   // default: false
```

Also controlled by the environment variable `CLAUDE_AUTO_BACKGROUND_TASKS`:

```typescript
// vendor/dom.ts lines 73288-73291
function nR_() {
  if (
    a6(process.env.CLAUDE_AUTO_BACKGROUND_TASKS) ||
    l8("tengu_auto_background_agents", !1)
  )
    return 120000;   // 120 second timeout for background agents
  return 0;         // no background agents
}
```

When `tengu_auto_background_agents` is `true` (or `CLAUDE_AUTO_BACKGROUND_TASKS` env is set),
auto-spawning of background sub-agents is permitted without user request, with a 120-second
default task timeout.

---

## 15. Telemetry Events

**Sources:** `src/core/automodeoptindialog-1.ts`, `src/core/bypasspermissionsmodedialog-1.ts`,
`src/tools/definitions-1.ts`, `src/tools/tool-2.ts`, `src/vendor/dom.ts`,
`src/ui/components-1.ts`

### 15.1 Auto Mode Core Events

| Event | Source | When Fired |
|-------|--------|------------|
| `tengu_auto_mode_state` | `src/vendor/dom.ts:70188` | LSP experiment gates notification; carries state of `QL_()` |
| `tengu_auto_mode_decision` | `src/tools/definitions-1.ts:9737,9761,9796`; `src/tools/tool-2.ts:2663` | After each classifier decision (allow/block/unavailable) |
| `tengu_auto_mode_outcome` | `src/tools/tool-2.ts:1935,1956,1974,2019,2043,2072,2090,2251,2271,2299,2313,2337` | After tool execution completes with auto mode outcome |
| `tengu_auto_mode_config` | `src/core/verifyautomodegateaccess-1.ts:433,454,511,515` | Gate configuration read; circuit breaker state |
| `tengu_auto_mode_malformed_tool_input` | `src/tools/tool-2.ts:1719` | Tool input failed schema validation |
| `tengu_auto_mode_denial_limit_exceeded` | `src/tools/definitions-1.ts:9457` | Consecutive or total denial limit hit |

### 15.2 Opt-In Dialog Events

| Event | Source | When Fired |
|-------|--------|------------|
| `tengu_auto_mode_opt_in_dialog_shown` | `automodeoptindialog-1.ts:122` | Dialog rendered via `useEffect` |
| `tengu_auto_mode_opt_in_dialog_accept` | `automodeoptindialog-1.ts:41` | User chose "Yes, enable auto mode" |
| `tengu_auto_mode_opt_in_dialog_accept_default` | `automodeoptindialog-1.ts:47` | User chose "Yes, and make it my default mode" |
| `tengu_auto_mode_opt_in_dialog_decline` | `automodeoptindialog-1.ts:56` | User declined |

### 15.3 Bypass Permissions Dialog Events

| Event | Source | When Fired |
|-------|--------|------------|
| `tengu_bypass_permissions_mode_dialog_shown` | `bypasspermissionsmodedialog-1.ts:110` | Dialog rendered via `useEffect` |
| `tengu_bypass_permissions_mode_dialog_accept` | `bypasspermissionsmodedialog-1.ts:44` | User accepted bypass permissions mode |

### 15.4 Dream Mode Events

| Event | Source | When Fired |
|-------|--------|------------|
| `tengu_auto_dream_fired` | `definitions-1.ts:7776` | Dream mode sub-agent fork started |
| `tengu_auto_dream_completed` | `definitions-1.ts:7814` | Dream mode fork completed successfully |
| `tengu_auto_dream_failed` | `definitions-1.ts:7826` | Dream mode fork threw an error |
| `tengu_auto_dream_toggled` | `components-1.ts:5579` | User toggled dream mode in UI |

### 15.5 Background Agents Event

| Event | Source | When Fired |
|-------|--------|------------|
| `tengu_auto_background_agents` | `vendor/dom.ts:73288` | Feature flag read (not an event; Statsig feature flag name) |

### 15.6 Remote Environment Event

| Event | Source | When Fired |
|-------|--------|------------|
| `tengu_ccr_unsupported_default_mode_ignored` | `verifyautomodegateaccess-1.ts:94` | `CLAUDE_CODE_REMOTE` mode ignored unsupported `defaultMode` |

---

## 16. Quick Reference Tables

### 16.1 Permission Modes

| Mode | Public | CLI Flag | Auto-Approve | Classifier |
|------|--------|----------|--------------|------------|
| `default` | Yes | `-p default` | No | No |
| `acceptEdits` | Yes | `-p acceptEdits` | File edits only | No |
| `plan` | Yes | `-p plan` | No (no execution) | No |
| `dontAsk` | Yes | `-p dontAsk` | Yes (all) | No |
| `bypassPermissions` | Yes | `--dangerously-skip-permissions` | Yes (all, no check) | No |
| `auto` | No (internal) | `--enable-auto-mode` or `-p auto` | Classifier decides | Yes |

### 16.2 Decision Values

| Value | Behavior | Precedence |
|-------|----------|------------|
| `"deny"` | Reject tool call | Highest |
| `"ask"` | Prompt user | Second |
| `"allow"` | Approve call | Third |
| `"passthrough"` | Defer to next handler | Lowest |

### 16.3 CLI Flags

| Flag | Effect | Hidden |
|------|--------|--------|
| `--enable-auto-mode` | Opt into auto mode for this session; calls `setAutoModeFlagCli(true)` | Yes |
| `--permission-mode auto` / `-p auto` | Set permission mode to `"auto"` | No |
| `--dangerously-skip-permissions` | Enable `bypassPermissions` mode | No |
| `--permission-mode <mode>` | Set any permission mode explicitly | No |

### 16.4 Settings Keys

| Key | Type | Description |
|-----|------|-------------|
| `permissions.defaultMode` | `string` | Default permission mode at startup |
| `skipAutoPermissionPrompt` | `boolean` | Skip re-showing opt-in dialog |
| `skipDangerousModePermissionPrompt` | `boolean` | Skip re-showing bypass permissions warning |
| `permissions.disableAutoMode` | `"disable"` | Disable auto mode in this settings scope |
| `disableAutoMode` | `"disable"` | Disable auto mode (root level) |
| `permissions.disableBypassPermissionsMode` | `"disable"` | Disable bypass permissions mode |
| `hasResetAutoModeOptInForDefaultOffer` | `boolean` | Migration guard: ran once to clear stale opt-in |
| `autoMode.allow` | `string[]` | Custom allow rules (replace defaults) |
| `autoMode.soft_deny` | `string[]` | Custom soft deny rules (replace defaults) |
| `autoMode.environment` | `string[]` | Custom environment context strings |

### 16.5 Statsig Feature Gates / Dynamic Configs

| Gate / Config | Default | Effect |
|---------------|---------|--------|
| `tengu_auto_mode_config` | `{enabled: "disabled"}` | Dynamic config; `enabled` field controls circuit breaker |
| `tengu_auto_mode_config.enabled` | `"disabled"` | `"enabled"` / `"opt-in"` / `"disabled"` |
| `tengu_disable_bypass_permissions_mode` | `false` | Gate: when true, bypass permissions mode is org-disabled |
| `tengu_auto_background_agents` | `false` | Feature flag: allow auto-spawning background agents |

### 16.6 Telemetry Events Summary

| Event | Category |
|-------|----------|
| `tengu_auto_mode_state` | State |
| `tengu_auto_mode_decision` | Classifier decision |
| `tengu_auto_mode_outcome` | Tool execution outcome |
| `tengu_auto_mode_config` | Gate / config |
| `tengu_auto_mode_malformed_tool_input` | Error |
| `tengu_auto_mode_denial_limit_exceeded` | Safety |
| `tengu_auto_mode_opt_in_dialog_shown` | UX / opt-in |
| `tengu_auto_mode_opt_in_dialog_accept` | UX / opt-in |
| `tengu_auto_mode_opt_in_dialog_accept_default` | UX / opt-in |
| `tengu_auto_mode_opt_in_dialog_decline` | UX / opt-in |
| `tengu_bypass_permissions_mode_dialog_shown` | UX / bypass |
| `tengu_bypass_permissions_mode_dialog_accept` | UX / bypass |
| `tengu_auto_dream_fired` | Dream mode |
| `tengu_auto_dream_completed` | Dream mode |
| `tengu_auto_dream_failed` | Dream mode |
| `tengu_auto_dream_toggled` | Dream mode |
| `tengu_ccr_unsupported_default_mode_ignored` | Remote env |

### 16.7 State Variables (`setautomodeflagcli-1.ts`)

| Variable | Getter | Setter | Initial | Meaning |
|----------|--------|--------|---------|------|
| `hd1` | `isAutoModeActive()` | `setAutoModeActive(bool)` | `false` | Active mode is `"auto"` |
| `Sd1` | `getAutoModeFlagCli()` | `setAutoModeFlagCli(bool)` | `false` | `--enable-auto-mode` flag was passed |
| `Cd1` | `isAutoModeCircuitBroken()` | `setAutoModeCircuitBroken(bool)` | `false` | Gate returned `"disabled"` |

### 16.8 Key Functions Cross-Reference

| Function | Minified Name | Source | Purpose |
|----------|---------------|--------|---------|
| `isAutoModeGateEnabled` | `WN` | `verifyautomodegateaccess-1.ts:297` | Primary gate check (3 conditions) |
| `getAutoModeUnavailableReason` | `k26` | `verifyautomodegateaccess-1.ts:303` | Returns `"settings"` / `"circuit-breaker"` / `"model"` / `null` |
| `transitionPermissionMode` | `cn` | `verifyautomodegateaccess-1.ts:237` | Handle mode transitions, strip/restore rules |
| `stripDangerousPermissionsForAutoMode` | `dn` | `verifyautomodegateaccess-1.ts:198` | Remove dangerous allow rules when entering auto |
| `restoreDangerousPermissions` | `T26` | `verifyautomodegateaccess-1.ts:222` | Restore rules when leaving auto |
| `isDangerousBashPermission` | `Xyq` | `verifyautomodegateaccess-1.ts:78` | Detect dangerous Bash allow rules |
| `isDangerousPowerShellPermission` | `Dyq` | `verifyautomodegateaccess-1.ts:93` | Stub: always false in v2.1.81 |
| `isDangerousTaskPermission` | `Pyq` | `verifyautomodegateaccess-1.ts:96` | Detect Task tool allow rules |
| `verifyAutoModeGateAccess` | `Pn6` | `verifyautomodegateaccess-1.ts:235` | Async startup gate check + context update |
| `initialPermissionModeFromCLI` | `vt1` | `verifyautomodegateaccess-1.ts:261` | Resolve initial mode from flags + settings |
| `prepareContextForPlanMode` | `Ik6` | `verifyautomodegateaccess-1.ts:348` | Save prePlanMode before entering plan |
| `isDefaultPermissionModeAuto` | `nb8` | `verifyautomodegateaccess-1.ts:346` | Check if settings default is auto |
| `hasAutoModeOptInAnySource` | `dS8` | `verifyautomodegateaccess-1.ts:322` | CLI flag OR user settings opt-in |
| `kG8` | `kG8` | `src/tools/tool-2.ts:2124` | LLM classifier invocation |
| `autoModeDefaultsHandler` | `KLY` | `automodedefaultshandler-1.ts:29` | `claude auto-mode defaults` |
| `autoModeConfigHandler` | `_LY` | `automodedefaultshandler-1.ts:32` | `claude auto-mode config` |
| `autoModeCritiqueHandler` | `zLY` | `automodedefaultshandler-1.ts:40` | `claude auto-mode critique` |

---

*End of Auto Mode Reference — Claude Code CLI v2.1.81*
