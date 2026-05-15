# Brief Mode / SendUserMessage — Reference (v2.1.81)

Sourced exclusively from Claude Code CLI v2.1.81 extracted source.
All claims are anchored to specific file:line references.

---

## Table of Contents

1. [Overview](#1-overview)
2. [The SendUserMessage Tool](#2-the-sendusermessage-tool)
   - 2.1 [Tool Identity & Names](#21-tool-identity--names)
   - 2.2 [Input Schema](#22-input-schema)
   - 2.3 [Output Schema](#23-output-schema)
   - 2.4 [Tool Behavior & Properties](#24-tool-behavior--properties)
   - 2.5 [Full Prompt Text](#25-full-prompt-text)
   - 2.6 [Proactive Section Prompt](#26-proactive-section-prompt)
3. [Entitlement & Access Control](#3-entitlement--access-control)
   - 3.1 [isBriefEntitled — Cf8()](#31-isbriefentitled--cf8)
   - 3.2 [isBriefEnabled — rX4()](#32-isbrief-enabled--rx4)
   - 3.3 [Feature Flag Cache](#33-feature-flag-cache)
4. [Activation Methods](#4-activation-methods)
   - 4.1 [/brief Slash Command](#41-brief-slash-command)
   - 4.2 [app:toggleBrief Keybinding](#42-apptogglebrief-keybinding)
   - 4.3 [CLAUDE_CODE_BRIEF Environment Variable](#43-claude_code_brief-environment-variable)
   - 4.4 [--brief CLI Flag](#44---brief-cli-flag)
   - 4.5 [tengu_kairos_brief Feature Flag](#45-tengu_kairos_brief-feature-flag)
5. [Session State](#5-session-state)
   - 5.1 [isBriefOnly Default](#51-isbriefonly-default)
   - 5.2 [System Reminders on Toggle](#52-system-reminders-on-toggle)
   - 5.3 [Related State Fields](#53-related-state-fields)
6. [Attachment Upload System](#6-attachment-upload-system)
   - 6.1 [Upload Trigger Conditions](#61-upload-trigger-conditions)
   - 6.2 [Upload Endpoint & Protocol](#62-upload-endpoint--protocol)
   - 6.3 [Size Limit & Timeout](#63-size-limit--timeout)
   - 6.4 [MIME Type Detection](#64-mime-type-detection)
   - 6.5 [Return Value & Error Handling](#65-return-value--error-handling)
   - 6.6 [Attachment Validation](#66-attachment-validation)
7. [UI Rendering](#7-ui-rendering)
   - 7.1 [Standard (non-Brief) Rendering](#71-standard-non-brief-rendering)
   - 7.2 [Brief Mode Rendering](#72-brief-mode-rendering)
8. [Tool Registration](#8-tool-registration)
9. [Telemetry Events](#9-telemetry-events)
   - 9.1 [tengu_brief_mode_toggled](#91-tengu_brief_mode_toggled)
   - 9.2 [tengu_brief_mode_enabled](#92-tengu_brief_mode_enabled)
   - 9.3 [tengu_brief_send](#93-tengu_brief_send)
   - 9.4 [Event Registry](#94-event-registry)
10. [Kairos System Context](#10-kairos-system-context)
11. [Integration Design & Patterns](#11-integration-design--patterns)
    - 11.1 [Non-Terminal UI Design](#111-non-terminal-ui-design)
    - 11.2 [REPL Bridge Connection](#112-repl-bridge-connection)
    - 11.3 [status Field Routing](#113-status-field-routing)
    - 11.4 [Attachment UUID Flow](#114-attachment-uuid-flow)
12. [Config Object: tengu_kairos_brief_config](#12-config-object-tengu_kairos_brief_config)

---

## 1. Overview

Brief mode is a condensed output mode for Claude Code where **all user-facing output is routed exclusively through the `SendUserMessage` tool**. Plain text emitted outside that tool is hidden from the default view — it remains accessible only through a "detail view" expansion in supported UIs.

The system enforces intentional, structured communication: the model must consciously choose what the user sees, rather than relying on raw streaming text. This design serves integrations and non-terminal UIs (Claude Desktop, VS Code extensions, browser extensions, and any REPL bridge consumer) where unstructured plaintext output is not useful.

Brief mode is part of the **Kairos** feature system, Anthropic's runtime-controlled feature delivery layer. It shares this namespace with `tengu_kairos_cron` (scheduled task execution).

The feature was introduced with two names: the original `Brief` (a legacy alias) and the current canonical name `SendUserMessage`. Both names are maintained as aliases on the tool object.

**Source files covering this system:**

| File | Description |
|------|-------------|
| `src/conversation/legacy_brief_tool_name-2.ts` | Tool name constants, prompt strings, aliases |
| `src/telemetry/isbriefentitled-1.ts` | Entitlement functions `Cf8()` / `rX4()` |
| `src/telemetry/events-2.ts` | `/brief` slash command definition |
| `src/tools/definitions-1.ts` | Full tool object `Hr9`, schema, `call()` impl |
| `src/core/f_4-2.ts` | Attachment resolution and upload dispatch |
| `src/core/uploadbriefattachment-2.ts` | HTTP upload to Anthropic file API |
| `src/ui/components-2.ts` | UI rendering branch for Brief mode |
| `src/conversation/session-1.ts` | Session state initialization |
| `src/agents/startdeferredprefetches-1.ts` | CLI flag / env-var Brief activation |
| `src/ui/components-1.ts` | `app:toggleBrief` keybinding registration |
| `src/vendor/dom.ts` | Canonical telemetry event name registry |

---

## 2. The SendUserMessage Tool

### 2.1 Tool Identity & Names

```
src/conversation/legacy_brief_tool_name-2.ts, lines 82-84
```

```typescript
var Cj6 = "SendUserMessage",   // BRIEF_TOOL_NAME    — current canonical name
    Wa8 = "Brief",              // LEGACY_BRIEF_TOOL_NAME — legacy alias
    fa8 = "Send a message to the user";  // DESCRIPTION
```

The tool object at `src/tools/definitions-1.ts:2108` declares both:

```typescript
Hr9 = {
  name: Cj6,         // "SendUserMessage"
  aliases: [Wa8],    // ["Brief"]
  searchHint: "send a message to the user — your primary visible output channel",
  ...
};
```

The alias `Brief` ensures backward compatibility with any stored conversation history or tool-use messages that reference the old name. The alias array is processed during tool lookup so both names resolve to the same tool implementation.

### 2.2 Input Schema

```
src/tools/definitions-1.ts, lines 2068-2086
```

The input schema is a strict Zod object (no extra keys allowed):

```typescript
wr9 = g6(() =>
  h.strictObject({
    message: h
      .string()
      .describe("The message for the user. Supports markdown formatting."),

    attachments: h
      .array(h.string())
      .optional()
      .describe(
        "Optional file paths (absolute or relative to cwd) to attach. " +
        "Use for photos, screenshots, diffs, logs, or any file the user should see alongside your message."
      ),

    status: h
      .enum(["normal", "proactive"])
      .describe(
        "Use 'proactive' when you're surfacing something the user hasn't asked for and needs to see now — " +
        "task completion while they're away, a blocker you hit, an unsolicited status update. " +
        "Use 'normal' when replying to something the user just said."
      ),
  })
);
```

**Parameter summary:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `message` | `string` | Yes | The response content. Supports markdown. |
| `attachments` | `string[]` | No | File paths (absolute or cwd-relative) to include. Images, diffs, logs. |
| `status` | `"normal" \| "proactive"` | Yes | `normal` = replying to user request; `proactive` = agent-initiated (task done, blocker surfaced, needs input). |

### 2.3 Output Schema

```
src/tools/definitions-1.ts, lines 2086-2107
```

The output schema (what `call()` returns, stored in tool result messages):

```typescript
Or9 = g6(() =>
  h.object({
    message: h.string().describe("The message"),

    attachments: h
      .array(
        h.object({
          path: h.string(),
          size: h.number(),
          isImage: h.boolean(),
          file_uuid: h.string().optional(),   // present if upload succeeded
        })
      )
      .optional()
      .describe("Resolved attachment metadata"),

    sentAt: h
      .string()
      .optional()
      .describe(
        "ISO timestamp captured at tool execution on the emitting process. " +
        "Optional — resumed sessions replay pre-sentAt outputs verbatim."
      ),
  })
);
```

The `file_uuid` field on each attachment is populated only when the upload to the Anthropic file API succeeds (see Section 6). The `sentAt` ISO timestamp enables proper ordering and deduplication when conversations are resumed from history.

### 2.4 Tool Behavior & Properties

```
src/tools/definitions-1.ts, lines 2108-2175
```

| Property | Value | Notes |
|----------|-------|-------|
| `maxResultSizeChars` | `100000` (1e5) | Max output size in chars |
| `userFacingName()` | `""` (empty string) | Intentionally blank — Brief output renders with custom "Claude" label |
| `isConcurrencySafe()` | `true` | Safe to call concurrently with other tool calls |
| `isReadOnly()` | `true` | Does not modify filesystem or system state |
| `isEnabled()` | `rX4()` | Delegates to `isBriefEnabled()` — returns false if Brief is not active |
| `checkPermissions()` | always `{ behavior: "allow" }` | Never blocked by permission system |

The `mapToolResultToToolResultBlockParam()` method produces the tool result content inserted back into the conversation:

```typescript
mapToolResultToToolResultBlockParam(A, q) {
  let K = A.attachments?.length ?? 0,
      _ = K === 0 ? "" : ` (${K} attachment${K === 1 ? "" : "s"} included)`;
  return {
    tool_use_id: q,
    type: "tool_result",
    content: `Message delivered to user.${_}`,
  };
}
```

### 2.5 Full Prompt Text

```
src/conversation/legacy_brief_tool_name-2.ts, line 86
```

Variable `Za8` (exported as `BRIEF_TOOL_PROMPT`):

```
Send a message the user will read. Text outside this tool is visible in the detail view,
but most won't open it — the answer lives here.

`message` supports markdown. `attachments` takes file paths (absolute or cwd-relative)
for images, diffs, logs.

`status` labels intent: 'normal' when replying to what they just asked; 'proactive' when
you're initiating — a scheduled task finished, a blocker surfaced during background work,
you need input on something they haven't asked about. Set it honestly; downstream routing
uses it.
```

### 2.6 Proactive Section Prompt

```
src/conversation/legacy_brief_tool_name-2.ts, lines 93-103
```

Variable `eXK` (exported as `BRIEF_PROACTIVE_SECTION`) — injected into system prompts that use Brief mode to reinforce correct usage:

```
## Talking to the user

SendUserMessage is where your replies go. Text outside it is visible if the user expands
the detail view, but most won't — assume unread. Anything you want them to actually see
goes through SendUserMessage. The failure mode: the real answer lives in plain text while
SendUserMessage just says "done!" — they see "done!" and miss everything.

So: every time the user says something, the reply they actually read comes through
SendUserMessage. Even for "hi". Even for "thanks".

If you can answer right away, send the answer. If you need to go look — run a command,
read files, check something — ack first in one line ("On it — checking the test output"),
then work, then send the result. Without the ack they're staring at a spinner.

For longer work: ack → work → result. Between those, send a checkpoint when something
useful happened — a decision you made, a surprise you hit, a phase boundary. Skip the
filler ("running tests...") — a checkpoint earns its place by carrying information.

Keep messages tight — the decision, the file:line, the PR number. Second person always
("your config"), never third.
```

---

## 3. Entitlement & Access Control

```
src/telemetry/isbriefentitled-1.ts, lines 28-37
```

### 3.1 isBriefEntitled — Cf8()

`Cf8()` is the entitlement gate. It returns `true` if **any one** of three conditions holds:

```typescript
function Cf8() {
  return (
    cv() ||                                              // 1. internal AI context
    a6(process.env.CLAUDE_CODE_BRIEF) ||                // 2. env var truthy
    cV("tengu_kairos_brief", false, $r9)                // 3. GrowthBook feature flag
  );
}
```

| Gate | Identifier | Notes |
|------|-----------|-------|
| Internal AI context | `cv()` | Anthropic-internal usage context; always entitled |
| Environment variable | `process.env.CLAUDE_CODE_BRIEF` | Any truthy value passes `a6()` |
| GrowthBook feature flag | `"tengu_kairos_brief"` | Server-side rollout flag; cached 300 000 ms (5 min) |

### 3.2 isBriefEnabled — rX4()

`rX4()` is the runtime check used by the tool's `isEnabled()` method:

```typescript
function rX4() {
  return (cv() || FN()) && Cf8();
}
```

`FN()` is the REPL bridge connection check. This means Brief mode is fully enabled when:
- The user is in an internal AI context (`cv()`), **or** the REPL bridge is connected (`FN()`)
- **And** the entitlement check passes (`Cf8()`)

In practice, an external UI client connecting via the REPL bridge enables `FN()`, which — combined with any entitlement path — makes the tool available.

### 3.3 Feature Flag Cache

```
src/telemetry/isbriefentitled-1.ts, line 40
```

```typescript
var $r9 = 300000;   // 300 000 ms = 5 minutes TTL for tengu_kairos_brief GrowthBook cache
```

The GrowthBook flag result is cached via `cV()` for 5 minutes to avoid excessive network calls per turn.

---

## 4. Activation Methods

### 4.1 /brief Slash Command

```
src/telemetry/events-2.ts, lines 313-388
```

The `/brief` command is registered as a `local-jsx` slash command:

```typescript
xOY = {
  type: "local-jsx",
  name: "brief",
  description: "Toggle brief-only mode",
  isEnabled: () => {
    return bOY().enable_slash_command;   // gated by tengu_kairos_brief_config
  },
  isHidden: false,
  immediate: true,
  load: () =>
    Promise.resolve({
      async call(A, q) {
        let _ = !q.getAppState().isBriefOnly;   // toggles current state
        if (_ && !Cf8())
          return (
            Q("tengu_brief_mode_toggled", { enabled: false, gated: true, source: "slash_command" }),
            A("Brief tool is not enabled for your account", { display: "system" }),
            null
          );
        // ... toggle state, fire telemetry, inject system reminder
      },
    }),
};
```

**Key behaviors:**
- `isEnabled()` is gated by `tengu_kairos_brief_config.enable_slash_command` (separate from entitlement)
- `immediate: true` means the command executes synchronously without a round-trip to the model
- If the user attempts to enable Brief without entitlement (`!Cf8()`), they receive the message: **"Brief tool is not enabled for your account"** and a `gated: true` telemetry event is fired
- Successfully toggling Brief injects a system reminder into the conversation context (see Section 5.2)

### 4.2 app:toggleBrief Keybinding

```
src/ui/components-1.ts, line 10548
```

```typescript
W1("app:toggleBrief", W, { context: "Global" })
```

Registered as a global keybinding action. `W` is the handler function that toggles `isBriefOnly` on the app state in the same manner as the slash command. The `"Global"` context means it is available in all UI contexts (Chat, Confirmation, etc.), not just specific panes.

The keybinding can be bound to a key sequence via the user's `keybindings.json` configuration file.

### 4.3 CLAUDE_CODE_BRIEF Environment Variable

```
src/telemetry/isbriefentitled-1.ts, line 31
src/agents/startdeferredprefetches-1.ts, line 2662
```

Setting `CLAUDE_CODE_BRIEF` to any truthy value (e.g., `CLAUDE_CODE_BRIEF=1`) both:
1. Passes the entitlement check in `Cf8()` (via `a6(process.env.CLAUDE_CODE_BRIEF)`)
2. Activates Brief mode at session start via the startup function `Lm8()` in `startdeferredprefetches-1.ts`

```typescript
function Lm8(A) {
  let q = A.brief,
      K = a6(process.env.CLAUDE_CODE_BRIEF);  // env var check
  if (!q && !K) return;                        // exit if neither flag nor env var
  let { isBriefEntitled: _ } = (mg(), o7(Ml)),
      Y = _();
  if (Y) fu(true);                             // fu() activates Brief mode globally
  Q("tengu_brief_mode_enabled", {
    enabled: Y,
    gated: !Y,
    source: K ? "env" : "flag",
  });
}
```

The `source` field in telemetry distinguishes `"env"` (environment variable) from `"flag"` (CLI flag `--brief`).

### 4.4 --brief CLI Flag

```
src/agents/startdeferredprefetches-1.ts, line 2661
```

The CLI accepts a `--brief` flag (`A.brief` in `Lm8()`). When present at startup, it follows the same activation path as the env var: calls `isBriefEntitled()` and if entitled, activates Brief mode, firing `tengu_brief_mode_enabled` with `source: "flag"`.

### 4.5 tengu_kairos_brief Feature Flag

The GrowthBook feature flag `"tengu_kairos_brief"` enables entitlement server-side. This is the primary rollout mechanism for external users. It is distinct from `tengu_kairos_brief_config` (which controls slash command availability) — see Section 12.

---

## 5. Session State

### 5.1 isBriefOnly Default

```
src/conversation/session-1.ts, line 6156
```

```typescript
return {
  settings: kA(),
  tasks: {},
  // ...
  isBriefOnly: false,              // default off
  // ...
  kairosEnabled: false,
  replBridgeEnabled: false,
  replBridgeExplicit: false,
  replBridgeConnected: false,
  replBridgeSessionActive: false,
  // ...
};
```

`isBriefOnly` defaults to `false` in every new session. It is a boolean field on the `AppState` object. When set to `true`, the UI rendering branches (Section 7), tool availability (Section 8), and telemetry all respond accordingly.

### 5.2 System Reminders on Toggle

```
src/telemetry/events-2.ts, lines 361-376
```

When Brief mode is toggled via `/brief` or `app:toggleBrief`, a system reminder is injected into the conversation context (unless the user is in an internal AI context, where `cv()` suppresses it):

**On enable:**
```
<system-reminder>
Brief mode is now enabled. Use the SendUserMessage tool for all user-facing output —
plain text outside it is hidden from the user's view.
</system-reminder>
```

**On disable:**
```
<system-reminder>
Brief mode is now disabled. The SendUserMessage tool is no longer available —
reply with plain text.
</system-reminder>
```

The system reminder is passed as `metaMessages` in the activation call, ensuring it appears in the conversation history at the correct turn boundary.

### 5.3 Related State Fields

The following `AppState` fields interact with Brief mode:

| Field | Type | Relevance |
|-------|------|-----------|
| `isBriefOnly` | `boolean` | Primary Brief mode flag |
| `replBridgeEnabled` | `boolean` | Enables attachment upload (Section 6.1) |
| `replBridgeConnected` | `boolean` | REPL bridge connection status |
| `replBridgeSessionActive` | `boolean` | Active REPL session |
| `kairosEnabled` | `boolean` | Parent Kairos feature system flag |

---

## 6. Attachment Upload System

### 6.1 Upload Trigger Conditions

```
src/core/f_4-2.ts, lines 59-78
```

Attachment upload to the Anthropic file API is triggered when the tool's `call()` method processes attachments and **either** of these conditions is true:

```typescript
let _ = q.replBridgeEnabled || a6(process.env.CLAUDE_CODE_BRIEF_UPLOAD);
```

| Condition | Variable | Description |
|-----------|----------|-------------|
| REPL bridge active | `q.replBridgeEnabled` | App state flag; true when REPL bridge is connected |
| Upload env var | `process.env.CLAUDE_CODE_BRIEF_UPLOAD` | Force-enables upload regardless of REPL bridge |

When neither condition is true, attachments are resolved to local metadata (path, size, isImage) but are **not uploaded** — `file_uuid` is absent.

### 6.2 Upload Endpoint & Protocol

```
src/core/uploadbriefattachment-2.ts, lines 241-280
```

```
Endpoint:  {ANTHROPIC_BASE_URL}/api/oauth/file_upload
Method:    POST
Body:      multipart/form-data
Field:     file (with filename and content-type)
```

The `ANTHROPIC_BASE_URL` falls back to the compiled default base URL if the env var is not set (`iA().BASE_API_URL`).

Request headers:
```
Authorization: Bearer {oauth_access_token}
Content-Type:  multipart/form-data; boundary={uuid_boundary}
Content-Length: {body_length}
```

The OAuth access token is obtained from `hA()?.accessToken`. If no token is available, upload is silently skipped with a debug log.

**Expected success response:** HTTP `201` with JSON body `{ file_uuid: string }`.

Non-201 responses are logged and the upload is treated as failed (attachment proceeds without `file_uuid`).

### 6.3 Size Limit & Timeout

```
src/core/uploadbriefattachment-2.ts, lines 286-288
```

```typescript
var g_4 = 31457280,   // 30 MB (30 * 1024 * 1024 bytes)
    Ru9 = 30000;      // 30 000 ms = 30 second HTTP timeout
```

Files exceeding 30 MB are skipped before the HTTP request is made. The debug log entry format is:

```
[brief:upload] skip {path}: {size} bytes exceeds {limit} limit
```

### 6.4 MIME Type Detection

```
src/core/f_4-2.ts, lines 20-26
src/core/uploadbriefattachment-2.ts, lines 220-223
```

MIME type is determined from the file extension:

```typescript
hu9 = {
  ".png":  "image/png",
  ".jpg":  "image/jpeg",
  ".jpeg": "image/jpeg",
  ".gif":  "image/gif",
  ".webp": "image/webp",
};
```

Files with unrecognized extensions fall back to `"application/octet-stream"`.

The `isImage` boolean field in the output schema is set by matching the path against:
```typescript
uW8 = /\.(png|jpe?g|gif|webp)$/i
```

### 6.5 Return Value & Error Handling

```
src/core/uploadbriefattachment-2.ts, lines 269-283
```

On success, `xu9()` (the `uploadBriefAttachment` function) returns the `file_uuid` string. On any failure, it returns `undefined` and the caller (`d_4` in `f_4-2.ts`) falls back to the local metadata without `file_uuid`:

```typescript
return K.map((w, O) => (z[O] === void 0 ? w : { ...w, file_uuid: z[O] }));
```

All failure paths log to `[brief:upload]` prefix via `Z_6()`:

```
[brief:upload] skip: no oauth token
[brief:upload] skip {path}: {size} bytes exceeds {limit} limit
[brief:upload] read failed for {path}: {error}
[brief:upload] upload failed for {path}: status={status} body={body_preview}
[brief:upload] unexpected response shape for {path}: {error}
[brief:upload] upload threw for {path}: {error}
[brief:upload] uploaded {path} → {file_uuid} ({size} bytes)   ← success log
```

### 6.6 Attachment Validation

```
src/core/f_4-2.ts, lines 29-57
```

Before upload, `Q_4()` validates each attachment path:

```typescript
async function Q_4(A) {
  let q = G8();
  for (let K of A) {
    let _ = H4(K);   // resolve to absolute path
    try {
      if (!(await U_4(_)).isFile())
        return { result: false, message: `Attachment "${K}" is not a regular file.`, errorCode: 1 };
    } catch (Y) {
      let z = Y.code;
      if (z === "ENOENT")
        return { result: false, message: `Attachment "${K}" does not exist. Current working directory: ${q}.`, errorCode: 1 };
      if (z === "EACCES" || z === "EPERM")
        return { result: false, message: `Attachment "${K}" is not accessible (permission denied).`, errorCode: 1 };
      throw Y;
    }
  }
  return { result: true };
}
```

This validation runs in `validateInput()` on the tool object before `call()` is invoked. It checks:
- Path exists (ENOENT)
- Path is a regular file (not directory)
- Path is accessible (EACCES / EPERM)

---

## 7. UI Rendering

```
src/ui/components-2.ts, lines 467-496
```

### 7.1 Standard (non-Brief) Rendering

When `isBriefOnly` is `false`, the SendUserMessage tool result renders as a standard row:

```typescript
return qO.default.createElement(
  B,
  { flexDirection: "row", marginTop: 1 },
  qO.default.createElement(B, { minWidth: 2 }),
  qO.default.createElement(
    B,
    { flexDirection: "column" },
    A.message ? qO.default.createElement(zw, null, A.message) : null,
    qO.default.createElement(ZL1, { attachments: A.attachments }),
  ),
);
```

No label is shown. The message content is rendered via `zw` (the markdown renderer), and attachments are rendered via `ZL1`.

### 7.2 Brief Mode Rendering

When `K?.isBriefOnly` is `true`, a distinct layout is used:

```typescript
if (K?.isBriefOnly) {
  let Y = A.sentAt ? Sf8(A.sentAt) : "";
  return qO.default.createElement(
    B,
    { flexDirection: "column", marginTop: 1, paddingLeft: 2 },
    qO.default.createElement(
      B,
      { flexDirection: "row" },
      qO.default.createElement(T, { color: "briefLabelClaude" }, "Claude"),
      Y ? qO.default.createElement(T, { dimColor: true }, " ", Y) : null,
    ),
    qO.default.createElement(
      B,
      { flexDirection: "column" },
      A.message ? qO.default.createElement(zw, null, A.message) : null,
      qO.default.createElement(ZL1, { attachments: A.attachments }),
    ),
  );
}
```

**Differences from standard rendering:**

| Aspect | Standard | Brief Mode |
|--------|----------|------------|
| Layout | `flexDirection: "row"` | `flexDirection: "column"` |
| Top margin | `marginTop: 1` | `marginTop: 1` |
| Left padding | none | `paddingLeft: 2` |
| Label | none | `"Claude"` in `briefLabelClaude` color |
| Timestamp | not shown | shown after label, `dimColor: true` |

The `briefLabelClaude` color token is a distinct theme color separate from regular text, providing visual differentiation for Brief mode messages. `Sf8(A.sentAt)` formats the ISO timestamp for display.

---

## 8. Tool Registration

```
src/tools/definitions-1.ts, line 2375
```

The Brief tool is registered in the tool list via:

```typescript
hg("brief", () => xr9())
```

`hg()` is the tool slot registration function. The `"brief"` key is the internal slot name; the tool's public name is `"SendUserMessage"` (from `Cj6`).

The factory function `xr9()` performs a two-stage guard:

```typescript
// src/tools/definitions-1.ts, lines 2533-2537
function xr9() {
  if (!oX4) return null;           // oX4 is the tool definition object (Hr9)
  if (!Xr9?.isBriefEnabled()) return null;   // Xr9 is the isBriefEnabled ref
  return oX4;
}
```

`oX4` and `Xr9` are module-level variables populated during initialization:
- `oX4` holds the `Hr9` tool definition (populated when the module loads)
- `Xr9` holds a reference to the `{ isBriefEnabled }` export from `isbriefentitled-1.ts`

When `xr9()` returns `null`, the tool slot is absent from the tool list sent to the model — the model has no knowledge of or access to `SendUserMessage`.

The registration occurs inside the list that builds the active tool set per session:

```typescript
// src/tools/definitions-1.ts, lines 2370-2376
hg("scratchpad", () => Cr9()),
hg("frc", () => Ir9(q)),
hg("summarize_tool_results", () => br9),
hg("brief", () => xr9()),   // Brief is last in this group
```

---

## 9. Telemetry Events

### 9.1 tengu_brief_mode_toggled

Fired by the `/brief` slash command when the user attempts to toggle Brief mode.

**Payload:**

```typescript
Q("tengu_brief_mode_toggled", {
  enabled: boolean,   // whether Brief was turned ON (false if blocked by gate)
  gated: boolean,     // true if the toggle was blocked by entitlement check
  source: "slash_command",
});
```

Fired in two cases:
1. Successful toggle: `enabled: true/false, gated: false`
2. Blocked toggle (no entitlement): `enabled: false, gated: true`

### 9.2 tengu_brief_mode_enabled

Fired at session initialization when Brief mode is activated via CLI flag or environment variable.

```
src/agents/startdeferredprefetches-1.ts, lines 2667-2671
```

**Payload:**

```typescript
Q("tengu_brief_mode_enabled", {
  enabled: boolean,   // true if entitled and activated
  gated: boolean,     // true if entitlement check failed
  source: "env" | "flag",  // "env" = CLAUDE_CODE_BRIEF, "flag" = --brief CLI flag
});
```

### 9.3 tengu_brief_send

Fired each time the `SendUserMessage` tool's `call()` method executes successfully.

```
src/tools/definitions-1.ts, lines 2163-2167
```

**Payload:**

```typescript
Q("tengu_brief_send", {
  proactive: boolean,          // true if status === "proactive"
  attachment_count: number,    // 0 if no attachments
});
```

This event is the primary signal for tracking Brief mode usage volume and distinguishing proactive vs. reactive invocations.

### 9.4 Event Registry

```
src/vendor/dom.ts, lines 9132-9134
```

All three Brief telemetry events appear in the canonical registered event name list:

```typescript
"tengu_brief_mode_enabled",
"tengu_brief_mode_toggled",
"tengu_brief_send",
```

This list is the compile-time schema for allowed telemetry event names — unregistered events are rejected at the call site.

---

## 10. Kairos System Context

Brief mode lives under the **Kairos** feature namespace — Anthropic's system for runtime-controlled feature delivery that extends Claude Code beyond the base CLI.

Kairos features share a naming convention: `tengu_kairos_{feature_name}` for entitlement flags and `tengu_kairos_{feature_name}_config` for per-feature configuration objects.

**Known Kairos features in v2.1.81:**

| Feature | Flag | Config |
|---------|------|--------|
| Brief / SendUserMessage | `tengu_kairos_brief` | `tengu_kairos_brief_config` |
| Cron / Scheduled tasks | `tengu_kairos_cron` | `tengu_kairos_cron_config` |

The Kairos system enables Anthropic to progressively roll out features to subsets of users without shipping new CLI versions, using GrowthBook as the feature flag backend.

The `kairosEnabled` field in `AppState` (initialized to `false` in `session-1.ts:6162`) is a top-level indicator of whether any Kairos feature is active for the current session.

---

## 11. Integration Design & Patterns

### 11.1 Non-Terminal UI Design

Brief mode is specifically designed for non-terminal UIs where:
- The UI displays tool results differently from streamed text
- Plain text output is either invisible or displayed in a less prominent location ("detail view")
- Structured data (message, attachments, status, timestamp) enables rich rendering

The tool's `userFacingName()` returns an empty string — in Brief mode, the message is the UI, not a tool call result badge.

### 11.2 REPL Bridge Connection

The REPL bridge (`replBridgeEnabled`, `replBridgeConnected`, `replBridgeSessionActive` in `AppState`) is the primary integration channel for non-terminal UIs. When a REPL bridge connection is established:

1. `FN()` returns `true`, satisfying part of the `rX4()` condition
2. `replBridgeEnabled` is set to `true` in `AppState`, enabling attachment upload
3. The tool becomes available to the model (assuming entitlement)

The REPL bridge provides bidirectional communication: the UI sends user input, the model responds via `SendUserMessage`, and attachments are uploaded to the shared file API for the UI to retrieve.

### 11.3 status Field Routing

The `status` parameter is a first-class routing signal for downstream consumers:

| Value | Meaning | UI Handling Pattern |
|-------|---------|---------------------|
| `"normal"` | Direct response to user's last input | Standard message display |
| `"proactive"` | Agent-initiated; user did not ask for this | Notification badge, push alert, separate inbox |

The `proactive` case covers:
- A scheduled task (Cron) completing while the user is away
- A blocker surfaced during background work
- The agent needing user input on something outside the current request scope

Downstream routing uses this field — it is the model's responsibility to set it honestly. The `tengu_brief_send` telemetry event records `proactive: boolean` to track usage patterns.

### 11.4 Attachment UUID Flow

The full attachment lifecycle for REPL bridge consumers:

```
Model calls SendUserMessage(attachments: ["/path/to/file.png"])
  → validateInput(): path existence and accessibility check
  → call(): d_4() resolves metadata
    → uploadBriefAttachment(): POST to /api/oauth/file_upload
      → returns file_uuid on success
  → Output: { message, attachments: [{ path, size, isImage, file_uuid }], sentAt }
  → UI receives tool result with file_uuid
  → UI fetches file content from Anthropic CDN using file_uuid
```

When upload is unavailable (no REPL bridge, no upload env var, or upload fails), the `file_uuid` field is absent and the UI must fall back to local file access if it has filesystem access.

---

## 12. Config Object: tengu_kairos_brief_config

```
src/telemetry/events-2.ts, lines 313-320
```

`bOY()` loads and validates the config object:

```typescript
function bOY() {
  let A = l8("tengu_kairos_brief_config", RVq),
      q = IOY().safeParse(A);
  return q.success ? q.data : RVq;
}
```

Default value `RVq`:

```typescript
var RVq = { enable_slash_command: false };
```

Schema (validated via `IOY()`):

```typescript
IOY = g6(() => h.object({ enable_slash_command: h.boolean() }))
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enable_slash_command` | `boolean` | `false` | Controls whether the `/brief` slash command appears in the command list |

**Critical distinction:**
- `tengu_kairos_brief` (entitlement flag) — controls whether the tool is available at all
- `tengu_kairos_brief_config.enable_slash_command` — controls whether the `/brief` command appears in the slash command menu

A user can be entitled to Brief mode (e.g., via `CLAUDE_CODE_BRIEF=1` or the `--brief` flag) without the slash command being enabled. In that case, Brief mode is activated at startup but cannot be toggled interactively.

Conversely, if `enable_slash_command` is `true` but the user lacks entitlement, the command appears in the menu but displays "Brief tool is not enabled for your account" when invoked.

---

## Quick Reference

### Constants

| Constant | Value | Source |
|----------|-------|--------|
| `Cj6` (BRIEF_TOOL_NAME) | `"SendUserMessage"` | `legacy_brief_tool_name-2.ts:82` |
| `Wa8` (LEGACY_BRIEF_TOOL_NAME) | `"Brief"` | `legacy_brief_tool_name-2.ts:83` |
| `fa8` (DESCRIPTION) | `"Send a message to the user"` | `legacy_brief_tool_name-2.ts:84` |
| `g_4` (upload size limit) | `31457280` (30 MB) | `uploadbriefattachment-2.ts:286` |
| `Ru9` (upload timeout) | `30000` ms | `uploadbriefattachment-2.ts:287` |
| `$r9` (flag cache TTL) | `300000` ms (5 min) | `isbriefentitled-1.ts:40` |

### Functions

| Function | Export Name | Source | Description |
|----------|-------------|--------|-------------|
| `Cf8()` | `isBriefEntitled` | `isbriefentitled-1.ts:28` | Three-gate entitlement check |
| `rX4()` | `isBriefEnabled` | `isbriefentitled-1.ts:35` | Runtime enabled check (requires bridge or internal) |
| `xr9()` | — | `definitions-1.ts:2533` | Tool factory; returns null if not enabled |
| `d_4()` | — | `f_4-2.ts:59` | Resolve attachments + dispatch upload |
| `xu9()` | `uploadBriefAttachment` | `uploadbriefattachment-2.ts:237` | HTTP upload to file API |
| `Q_4()` | — | `f_4-2.ts:29` | Validate attachment paths |
| `bOY()` | — | `events-2.ts:313` | Load `tengu_kairos_brief_config` |
| `Lm8()` | — | `startdeferredprefetches-1.ts:2660` | Session init activation via flag/env |

### Environment Variables

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_BRIEF` | Truthy value enables Brief entitlement and activates at startup |
| `CLAUDE_CODE_BRIEF_UPLOAD` | Truthy value forces attachment upload even without REPL bridge |
| `ANTHROPIC_BASE_URL` | Overrides base URL for the file upload endpoint |

### Telemetry Events

| Event | When | Key Fields |
|-------|------|------------|
| `tengu_brief_mode_toggled` | `/brief` command used | `enabled`, `gated`, `source` |
| `tengu_brief_mode_enabled` | Session start with Brief on | `enabled`, `gated`, `source` |
| `tengu_brief_send` | `SendUserMessage.call()` executes | `proactive`, `attachment_count` |
