# MCP Channels System — Claude Code CLI v2.1.81

> **Deep Dive: `--channels` and `--dangerously-load-development-channels`**

This document covers the MCP Channels feature as implemented in Claude Code CLI v2.1.81. This is NOT the auto-update channel system (which selects between stable/beta/nightly release tracks). This is a separate, experimental mechanism that allows MCP servers and plugins to push real-time notifications directly into a running Claude Code session.

All source references are verified against the extracted v2.1.81 source files located in `src/`.

---

## Table of Contents

1. [Overview](#1-overview)
2. [CLI Arguments](#2-cli-arguments)
3. [Channel Entry Structure](#3-channel-entry-structure)
4. [Parsing Logic — the `y8` function](#4-parsing-logic--the-y8-function)
5. [Session State Management](#5-session-state-management)
6. [Channel Validation Gate — `xXq`](#6-channel-validation-gate--xxq)
7. [Channel Registration and Message Handling](#7-channel-registration-and-message-handling)
8. [Channel Permission System](#8-channel-permission-system)
9. [Dev Channels Dialog — `DevChannelsDialog`](#9-dev-channels-dialog--devchannelsdialog)
10. [Channel Status UI — `ChannelsNotice`](#10-channel-status-ui--channelsnotice)
11. [Feature Detection and Storage](#11-feature-detection-and-storage)
12. [Enterprise Policy Configuration](#12-enterprise-policy-configuration)
13. [Telemetry Events](#13-telemetry-events)
14. [Security Model](#14-security-model)
15. [Complete Flow Diagram](#15-complete-flow-diagram)
16. [Quick Reference](#16-quick-reference)

---

## 1. Overview

MCP Channels is an **experimental, opt-in feature** in Claude Code v2.1.81 that enables MCP servers and marketplace plugins to push real-time notification messages into an active Claude Code session. When a registered channel server sends a `notifications/claude/channel` MCP notification, its content is injected as a prompt into the current session queue — effectively allowing external services to drive conversation turns.

### Core Characteristics

| Property | Value |
|----------|-------|
| Feature flag | `tengu_harbor` (local storage boolean) |
| Allowlist storage | `tengu_harbor_ledger` (local storage array) |
| Authentication requirement | claude.ai OAuth token required |
| MCP capability required | `experimental["claude/channel"]` |
| Session flag | `--channels` (production) |
| Dev flag | `--dangerously-load-development-channels` |
| Help visibility | Both flags hidden from `--help` |
| Injection priority | `"next"` — processed in next turn |
| Prompt injection warning | Explicitly shown in UI |

### Why It Exists

MCP Channels provides a controlled push-notification pathway for approved MCP plugins and servers. Rather than requiring the user to manually trigger tool calls or slash commands, a registered channel server can proactively inject messages — for example: a CI/CD server pushing a build failure, a monitoring plugin alerting about a threshold breach, or an automation plugin sending task completion signals.

### Key Constraints

- The entire feature is gated behind the `tengu_harbor` local storage flag, which is managed server-side by Anthropic and cached locally.
- Production use requires the plugin/server to appear on an Anthropic-managed allowlist (`tengu_harbor_ledger`).
- Claude.ai authentication (OAuth) is mandatory.
- Enterprise and team organization members additionally require `policySettings.channelsEnabled: true` set by an administrator.
- Channel messages explicitly bypass slash command processing (`skipSlashCommands: true`).
- The `--dangerously-load-development-channels` flag bypasses the allowlist check but shows a mandatory confirmation dialog (unless not authenticated or channels disabled, in which case it silently proceeds).

**Source**: `src/mcp/channelsnotice-1.ts` lines 1–266, `src/core/auth-1.ts` lines 15149–15209, `src/telemetry/ischannelsenabled-1.ts` lines 22–29.

---

## 2. CLI Arguments

### Option Definitions

Both channel flags are defined in `src/vendor/commander.ts` at lines 6595–6606 using Commander.js's `addOption` with `hideHelp()`:

```typescript
// src/vendor/commander.ts:6595-6606
q.addOption(
  new AK(
    "--channels <servers...>",
    "MCP servers whose channel notifications (inbound push) should register this session. Space-separated server names.",
  ).hideHelp(),
);
q.addOption(
  new AK(
    "--dangerously-load-development-channels <servers...>",
    "Load channel servers not on the approved allowlist. For local channel development only. Shows a confirmation dialog at startup.",
  ).hideHelp(),
);
```

The same definitions are mirrored (by construction of the extracted source) in `src/agents/startdeferredprefetches-1.ts` around line 996.

### `--channels <servers...>`

- **Type**: Variadic string array (`<servers...>`)
- **Purpose**: Registers one or more MCP servers/plugins as approved channel sources for the current session.
- **Visibility**: Hidden from `--help` output (`.hideHelp()`).
- **Validation**: Each entry must begin with `plugin:` or `server:` prefix. Invalid entries cause immediate process exit with a format error.
- **Session scope**: Entries are stored in session state only — they do not persist across sessions.
- **Production requirement**: Non-dev entries must additionally appear on the `tengu_harbor_ledger` allowlist to successfully register.

### `--dangerously-load-development-channels <servers...>`

- **Type**: Variadic string array (`<servers...>`)
- **Purpose**: Registers channel servers that bypass the global Anthropic-managed allowlist — intended for local plugin/server development.
- **Visibility**: Hidden from `--help` output (`.hideHelp()`).
- **Confirmation dialog**: When authenticated and channels are enabled, presents a `DevChannelsDialog` warning before proceeding.
- **Dev flag**: All entries from this option are tagged `dev: true` in the channel entry structure.
- **Allowlist bypass**: The `dev: true` tag causes the validation gate to skip the `tengu_harbor_ledger` check.

### Argument Format

Both flags accept space-separated entries on the command line:

```bash
# Production: approved plugin from a marketplace
claude --channels plugin:my-monitor@acme-marketplace

# Production: multiple entries
claude --channels plugin:alerts@acme-marketplace plugin:logs@acme-marketplace

# Manual MCP server (still requires allowlist in production)
claude --channels server:my-local-mcp

# Development: bypass allowlist (shows confirmation dialog)
claude --dangerously-load-development-channels plugin:my-dev-plugin@dev-marketplace

# Development: server entry (no marketplace required)
claude --dangerously-load-development-channels server:dev-server
```

### Format Requirements

| Entry Type | Format | Example |
|-----------|--------|---------|
| Plugin | `plugin:<name>@<marketplace>` | `plugin:ci-monitor@acme` |
| Server | `server:<name>` | `server:my-mcp-server` |
| Invalid | anything without prefix | `my-server` → immediate exit |

**Source**: `src/vendor/commander.ts` lines 6595–6606; `src/agents/startdeferredprefetches-1.ts` lines 995–1030.

---

## 3. Channel Entry Structure

After parsing, each CLI argument is transformed into a typed channel entry object:

```typescript
// Conceptual TypeScript (reconstructed from minified source)
type ChannelEntry =
  | { kind: "plugin"; name: string; marketplace: string; dev?: boolean }
  | { kind: "server"; name: string; dev?: boolean };
```

### Fields

| Field | Type | Present On | Description |
|-------|------|------------|-------------|
| `kind` | `"plugin" \| "server"` | All entries | Discriminator: plugin from marketplace or manually configured MCP server |
| `name` | `string` | All entries | Plugin name (without marketplace) or MCP server name |
| `marketplace` | `string` | Plugin entries only | The marketplace source identifier (e.g. `"acme-marketplace"`) |
| `dev` | `boolean \| undefined` | Optional | `true` when loaded via `--dangerously-load-development-channels`; absent/`undefined` for production entries |

### Entry Formatting

The `po6` function in `src/mcp/channelsnotice-1.ts` (line 234) serializes entries back to string form for display:

```typescript
// src/mcp/channelsnotice-1.ts:234-238
function po6(A) {
  return A.kind === "plugin"
    ? `plugin:${A.name}@${A.marketplace}`
    : `server:${A.name}`;
}
```

The same pattern appears in `QNY` in `src/ui/devchannelsdialog-2.ts` lines 108–112:

```typescript
// src/ui/devchannelsdialog-2.ts:108-112
function QNY(A) {
  return A.kind === "plugin"
    ? `plugin:${A.name}@${A.marketplace}`
    : `server:${A.name}`;
}
```

**Source**: `src/agents/startdeferredprefetches-1.ts` lines 999–1015; `src/mcp/channelsnotice-1.ts` lines 234–238.

---

## 4. Parsing Logic — the `y8` function

The inline `y8` arrow function in `src/agents/startdeferredprefetches-1.ts` (lines 995–1032) is the entry point for transforming raw CLI string arguments into typed `ChannelEntry` objects. It is defined anonymously within the startup block and called separately for `--channels` and `--dangerously-load-development-channels`.

### Full Source

```typescript
// src/agents/startdeferredprefetches-1.ts:995-1032
let y8 = (q7, v4) => {           // q7 = array of raw strings, v4 = flag name for error msg
  let QK = [],                    // valid entries accumulated here
    qq = [];                      // invalid (unprefixed) entries
  for (let mA of q7)
    if (mA.startsWith("plugin:")) {
      let Yq = mA.slice(7),       // strip "plugin:" prefix
        Xq = Yq.indexOf("@");     // find @ separator
      if (Xq <= 0 || Xq === Yq.length - 1)  // @ missing or at start/end
        qq.push(mA);
      else
        QK.push({
          kind: "plugin",
          name: Yq.slice(0, Xq),         // name = everything before @
          marketplace: Yq.slice(Xq + 1), // marketplace = everything after @
        });
    } else if (mA.startsWith("server:") && mA.length > 7)
      QK.push({ kind: "server", name: mA.slice(7) }); // strip "server:" prefix
    else
      qq.push(mA);  // no recognized prefix → invalid
  if (qq.length > 0)
    (process.stderr.write(
      Y8.red(
        `${v4} entries must be tagged: ${qq.join(", ")}\n` +
          `  plugin:<name>@<marketplace>  — plugin-provided channel (allowlist enforced)\n` +
          `  server:<name>                — manually configured MCP server\n`,
      ),
    ),
      process.exit(1));           // hard exit on any invalid entry
  return QK;
},
  n1 = O,                         // O = parsed CLI options object
  G7 = n1.channels,               // --channels array
  DA = n1.dangerouslyLoadDevelopmentChannels;  // --dangerously-load-development-channels array
if (!Z6) {  // Z6 = isNonInteractiveSession (K7()); channels only work in interactive mode
  if (DA && DA.length > 0)
    p6 = y8(DA, "--dangerously-load-development-channels");
  let q7 = [];
  if (G7 && G7.length > 0) ((q7 = y8(G7, "--channels")), R$6(q7));
  // ...
}
```

### Parsing Rules

| Input | Condition | Result |
|-------|-----------|--------|
| `plugin:name@market` | `@` present, not at start/end | `{kind:"plugin", name:"name", marketplace:"market"}` |
| `plugin:name` | No `@` (Xq === -1, so Xq <= 0) | Added to `qq` (invalid), exits with error |
| `plugin:@market` | `@` at position 0 | Added to `qq` (invalid), exits with error |
| `plugin:name@` | `@` at last position | Added to `qq` (invalid), exits with error |
| `server:name` | Length > 7 | `{kind:"server", name:"name"}` |
| `server:` | Length === 7 | NOT matched (`length > 7` fails), falls to invalid |
| `bare-name` | No recognized prefix | Added to `qq` (invalid), exits with error |

### Error Output Format

When invalid entries are detected, the error message written to stderr is:

```
--channels entries must be tagged: bare-name
  plugin:<name>@<marketplace>  — plugin-provided channel (allowlist enforced)
  server:<name>                — manually configured MCP server
```

### Non-Interactive Mode Gate

The entire channel parsing block is wrapped in `if (!Z6)` where `Z6 = K7()` detects the non-interactive (headless/print) session mode. Channels are exclusively an interactive-session feature and are silently ignored when Claude Code runs in `--print` or pipe mode.

**Source**: `src/agents/startdeferredprefetches-1.ts` lines 995–1048; line 820 for `Z6`.

---

## 5. Session State Management

Channel state is stored in the global session state object `T8`, which is the singleton state container for the active Claude Code session. Channel-specific fields were added at the end of the state object definition.

### T8 State Fields

```typescript
// src/agents/updatelastinteractiontime-1.ts:349-350
allowedChannels: [],   // ChannelEntry[] — initialized empty, populated at startup
hasDevChannels: false, // boolean — true if any dev channel entry was accepted
```

These fields appear at the bottom of the `T8` initialization block (lines 349–350), after the main session state fields, indicating they were added as a later extension.

### Getter/Setter Functions

Four dedicated accessor functions are defined in `src/agents/updatelastinteractiontime-1.ts` at lines 1051–1062:

```typescript
// src/agents/updatelastinteractiontime-1.ts:1051-1062
function ry() {
  return T8.allowedChannels;  // getter: returns current allowedChannels array
}
function R$6(A) {
  T8.allowedChannels = A;     // setter: replaces allowedChannels array
}
function Q68() {
  return T8.hasDevChannels;   // getter: returns hasDevChannels boolean
}
function d68(A) {
  T8.hasDevChannels = A;      // setter: sets hasDevChannels flag
}
```

### State Population Flow

The state is populated during startup in two stages:

**Stage 1 — Parsing (startdeferredprefetches-1.ts, ~line 1032):**
```typescript
// After y8() parses --channels entries:
R$6(q7);  // sets allowedChannels to the parsed production entries
// p6 holds dev entries but they are NOT yet in allowedChannels at this stage
```

**Stage 2 — Dev channels merge (config-1.ts, ~line 12550):**
```typescript
// After dev channels dialog accepted (or bypassed):
R$6([...ry(), ...z.map((j) => ({ ...j, dev: true }))]);  // merge with dev:true
d68(true);  // set hasDevChannels flag
```

This two-stage approach means production channels (`--channels`) are available in state immediately after CLI parsing, while dev channels (`--dangerously-load-development-channels`) are only added after the optional confirmation dialog resolves.

### Cross-Component Access

`ry()` (read allowedChannels) is called from multiple locations:
- `src/core/auth-1.ts:15187` — `xXq` gate: session allowlist check via `jo6(A, ry())`
- `src/mcp/channelsnotice-1.ts:213` — `P5Y()`: builds the display state for `ChannelsNotice`
- `src/mcp/channelsnotice-1.ts:239` — `W5Y()`: per-entry validation for warning list
- `src/core/config-1.ts:12550` — dev channel merge: `[...ry(), ...z.map(...)]`

**Source**: `src/agents/updatelastinteractiontime-1.ts` lines 349–350, 1051–1062; `src/agents/startdeferredprefetches-1.ts` lines 1030–1032; `src/core/config-1.ts` lines 12543–12566.

---

## 6. Channel Validation Gate — `xXq`

Every MCP server connection attempt runs through `xXq`, the channel registration gate defined in `src/core/auth-1.ts` at lines 15149–15209. This function implements 8 sequential validation checks; any failure short-circuits with a skip result.

### Function Signature

```typescript
// src/core/auth-1.ts:15149
function xXq(
  A,   // server name (string)
  q,   // server capabilities object (from MCP handshake)
  K    // server config (including pluginSource if applicable)
): { action: "register" } | { action: "skip"; kind: string; reason: string }
```

### The 8 Validation Checks

#### Check 1 — MCP Capability Declaration

```typescript
// src/core/auth-1.ts:15150-15154
if (!q?.experimental?.["claude/channel"])
  return {
    action: "skip",
    kind: "capability",
    reason: "server did not declare claude/channel capability",
  };
```

The server must declare support for `experimental["claude/channel"]` in its MCP capabilities response during the initialization handshake. This is an explicit opt-in — servers that do not declare this capability are silently skipped without telemetry.

#### Check 2 — Feature Flag

```typescript
// src/core/auth-1.ts:15155-15159
if (!Ho6())
  return {
    action: "skip",
    kind: "disabled",
    reason: "channels feature is not currently available",
  };
```

`Ho6()` reads the `tengu_harbor` boolean from local storage. This flag is Anthropic-controlled and can disable the entire channels feature server-side. Defined in `src/telemetry/ischannelsenabled-1.ts` line 27–29:
```typescript
function Ho6() {
  return l8("tengu_harbor", false);  // defaults to false if not set
}
```

#### Check 3 — Claude.ai Authentication

```typescript
// src/core/auth-1.ts:15160-15165
if (!hA()?.accessToken)
  return {
    action: "skip",
    kind: "auth",
    reason: "channels requires claude.ai authentication (run /login)",
  };
```

`hA()` returns the current OAuth token state. An `accessToken` is required — API key-only users cannot use MCP Channels. The user must authenticate via `claude login` (or `/login` within the session).

#### Check 4 — Organization Policy

```typescript
// src/core/auth-1.ts:15166-15175
let _ = sq();  // returns account type: "free" | "pro" | "team" | "enterprise"
if (_ === "team" || _ === "enterprise") {
  if (N1("policySettings")?.channelsEnabled !== !0)
    return {
      action: "skip",
      kind: "policy",
      reason: "channels not enabled by org policy (set channelsEnabled: true in managed settings)",
    };
}
```

Free and Pro accounts are not subject to this check. For team and enterprise accounts, an administrator must explicitly enable channels via managed settings.

#### Check 5 — Session Allowlist

```typescript
// src/core/auth-1.ts:15176-15183
let Y = jo6(A, ry());  // look up server in allowedChannels
if (!Y)
  return {
    action: "skip",
    kind: "session",
    reason: `server ${A} not in --channels list for this session`,
  };
```

The helper `jo6` (lines 15143–15147) searches the `allowedChannels` array:

```typescript
// src/core/auth-1.ts:15143-15147
function jo6(A, q) {
  let K = A.split(":");
  return q.find((_) =>
    _.kind === "server" ? A === _.name : K[0] === "plugin" && K[1] === _.name,
  );
}
```

For `server:` entries, the full server name is matched directly. For `plugin:` entries, the MCP server name is split on `:` and the plugin name portion is matched.

#### Check 6 — Marketplace Match (Plugin-only)

```typescript
// src/core/auth-1.ts:15184-15192
if (Y.kind === "plugin") {
  let z = K ? Hq(K).marketplace : void 0;  // extract marketplace from server config
  if (z !== Y.marketplace)
    return {
      action: "skip",
      kind: "marketplace",
      reason: `you asked for plugin:${Y.name}@${Y.marketplace} but the installed ${Y.name} plugin is from ${z ?? "an unknown source"}`,
    };
```

This cross-checks that the actually installed plugin's marketplace source matches what the user specified on the command line. Prevents a plugin from a different (potentially malicious) source from registering as a channel.

#### Check 7 — Global Allowlist (Plugin, non-dev)

```typescript
// src/core/auth-1.ts:15193-15200
if (
  !Y.dev &&
  !$o6().some((w) => w.plugin === Y.name && w.marketplace === Y.marketplace)
)
  return {
    action: "skip",
    kind: "allowlist",
    reason: `plugin ${Y.name}@${Y.marketplace} is not on the approved channels allowlist (use --dangerously-load-development-channels for local dev)`,
  };
```

`$o6()` reads the `tengu_harbor_ledger` from local storage and parses it with the Zod schema `O8Y()` (array of `{plugin: string, marketplace: string}`). Only plugins present in this Anthropic-managed list pass production use.

#### Check 7b — Global Allowlist (Server, non-dev)

```typescript
// src/core/auth-1.ts:15201-15206
} else if (!Y.dev)
  return {
    action: "skip",
    kind: "allowlist",
    reason: `server ${Y.name} is not on the approved channels allowlist (use --dangerously-load-development-channels for local dev)`,
  };
```

`server:` entries without `dev: true` are always rejected in production. This means manually configured MCP servers can ONLY be used as channels via `--dangerously-load-development-channels`.

#### Check 8 — Dev Bypass (implicit)

```typescript
// src/core/auth-1.ts:15208
return { action: "register" };
```

If `Y.dev === true`, checks 7 and 7b are skipped (guarded by `!Y.dev`), and execution falls through to the register result.

### Validation Summary Table

| # | Check | Gate Function | Failure Kind | Bypassed by dev: true |
|---|-------|---------------|--------------|------------------------|
| 1 | MCP capability declared | `q?.experimental["claude/channel"]` | `capability` | No |
| 2 | Feature flag enabled | `Ho6()` (tengu_harbor) | `disabled` | No |
| 3 | OAuth token present | `hA()?.accessToken` | `auth` | No |
| 4 | Org policy (team/enterprise) | `N1("policySettings")?.channelsEnabled` | `policy` | No |
| 5 | Session allowlist | `jo6(A, ry())` | `session` | No |
| 6 | Marketplace match (plugin) | `Hq(K).marketplace === Y.marketplace` | `marketplace` | No |
| 7 | Global allowlist (plugin) | `$o6().some(...)` | `allowlist` | Yes |
| 7b | Global allowlist (server) | `!Y.dev` | `allowlist` | Yes |

**Source**: `src/core/auth-1.ts` lines 15143–15209.

---

## 7. Channel Registration and Message Handling

When `xXq` returns `{action: "register"}`, the MCP connection setup in `src/mcp/mcp-4.ts` (lines 1650–1703) proceeds to configure the notification handlers.

### Registration

```typescript
// src/mcp/mcp-4.ts:1665-1703 (condensed)
case "register":
  a8(G.name, "Channel notifications registered"); // debug log
  G.client.setNotificationHandler(SXq(), async (u) => {
    let { content: b, meta: g } = u.params;
    // ... handles notifications/claude/channel
  });
  // If server also has permission capability:
  if (G.capabilities?.experimental?.["claude/channel/permission"] !== void 0)
    G.client.setNotificationHandler(CXq(), async (u) => {
      // ... handles notifications/claude/channel/permission
    });
```

`SXq` and `CXq` are schema factories that return the Zod-based notification schemas for `notifications/claude/channel` and `notifications/claude/channel/permission` respectively. They are initialized as `var SXq, CXq` in `src/core/auth-1.ts` lines 15210–15213.

### Notification Schema

The `notifications/claude/channel` notification params shape:

```typescript
// Reconstructed from usage at src/mcp/mcp-4.ts:1668
{
  content: string,  // the message text to inject as a prompt
  meta: Record<string, unknown> | null  // optional metadata key-value pairs
}
```

### Message Injection via OX()

When a channel notification arrives, `OX()` injects it into the session prompt queue:

```typescript
// src/mcp/mcp-4.ts:1679-1686
OX({
  mode: "prompt",
  value: bXq(G.name, b, g),   // formatted XML message
  priority: "next",           // processed in the next available turn
  isMeta: true,               // marked as metadata/system content
  origin: { kind: "channel", server: G.name }, // tracks the source
  skipSlashCommands: true,    // channel messages cannot trigger slash commands
});
```

### Message Formatting via `bXq()`

The `bXq` function in `src/core/auth-1.ts` (lines 15134–15142) wraps the channel message in an XML-like `<channel>` tag:

```typescript
// src/core/auth-1.ts:15134-15142
function bXq(A, q, K) {  // A=serverName, q=content, K=meta
  let _ = Object.entries(K ?? {})
    .filter(([Y]) => $8Y.test(Y))   // $8Y = /^[a-zA-Z_][a-zA-Z0-9_]*$/  (valid identifiers only)
    .map(([Y, z]) => ` ${Y}="${P3(z)}"`)  // P3 = XML attribute escaping
    .join("");
  return `<${Yj6} source="${P3(A)}"${_}>\n${q}\n</${Yj6}>`;
  //       Yj6 = "channel"  (from src/core/ua-3.ts:126)
}
```

Example output for a message `"Build failed on main"` from server `ci-monitor`:

```xml
<channel source="ci-monitor">
Build failed on main
</channel>
```

With meta `{branch: "main", run_id: "123"}`:

```xml
<channel source="ci-monitor" branch="main" run_id="123">
Build failed on main
</channel>
```

Note: The `$8Y` regex (`/^[a-zA-Z_][a-zA-Z0-9_]*$/`) filters meta keys to valid identifier names only — keys that don't match are silently excluded from the XML attributes. Values are XML-attribute-escaped via `P3()`.

**Source**: `src/mcp/mcp-4.ts` lines 1650–1703; `src/core/auth-1.ts` lines 15134–15142; `src/core/ua-3.ts` line 126.

---

## 8. Channel Permission System

In addition to the basic channel notification, the MCP Channels system supports an optional bi-directional permission request/response flow. Servers that declare `experimental["claude/channel/permission"]` can participate in pending permission resolution.

### Optional Capability

```typescript
// src/mcp/mcp-4.ts:1688-1698
if (G.capabilities?.experimental?.["claude/channel/permission"] !== void 0)
  G.client.setNotificationHandler(CXq(), async (u) => {
    let { request_id: b, behavior: g } = u.params,
      m = $.current?.resolve(b, g, G.name) ?? false;
    a8(
      G.name,
      `notifications/claude/channel/permission: ${b} → ${g} (${m ? "matched pending" : "no pending entry — stale or unknown ID"})`,
    );
  });
```

### Permission Queue — `FXq()`

The permission queue is created via `FXq()` defined in `src/telemetry/events-1.ts` lines 2594–2612:

```typescript
// src/telemetry/events-1.ts:2594-2612
function FXq() {
  let A = new Map();  // request_id.toLowerCase() → callback
  return {
    onResponse(q, K) {
      // registers a pending permission request listener
      let _ = q.toLowerCase();
      return (
        A.set(_, K),
        () => { A.delete(_); }  // returns cleanup function
      );
    },
    resolve(q, K, _) {
      // called when server sends permission response notification
      let Y = q.toLowerCase(),
        z = A.get(Y);
      if (!z) return false;  // stale or unknown ID
      return (A.delete(Y), z({ behavior: K, fromServer: _ }), true);
    },
  };
}
```

### Permission Flow

```
Claude Code session         MCP Channel Server
      │                           │
      │ ← (needs permission) ─────┤ (server wants to change behavior)
      │                           │
      │ registers pending entry   │
      │ in $.current (FXq map)    │
      │                           │
      │ ←── notifications/        │
      │     claude/channel/ ──────┤ sends {request_id, behavior}
      │     permission            │
      │                           │
      │ $.current.resolve()       │
      │ looks up request_id       │
      │ → matched: fires callback │
      │ → no match: logs stale ID │
```

### Notification Names

```typescript
// src/core/auth-1.ts:15210-15213
var SXq,                                    // schema for notifications/claude/channel
    rn1 = "notifications/claude/channel/permission",
    CXq,                                    // schema for rn1
    IXq = "notifications/claude/channel/permission_request";
```

The permission queue state is stored in the React ref `$.current` within the MCP connection component and is integrated into the global state via a Zustand store update:

```typescript
// src/mcp/mcp-4.ts:1486-1491
z((v) => {
  if (v.channelPermissionCallbacks === G) return v;
  return { ...v, channelPermissionCallbacks: G };
});
```

**Source**: `src/mcp/mcp-4.ts` lines 1480–1498, 1688–1698; `src/telemetry/events-1.ts` lines 2594–2612; `src/core/auth-1.ts` lines 15210–15213.

---

## 9. Dev Channels Dialog — `DevChannelsDialog`

The `DevChannelsDialog` React component is defined in `src/ui/devchannelsdialog-2.ts` (lines 25–115). It presents a blocking confirmation dialog when the user passes `--dangerously-load-development-channels`.

### When the Dialog is Shown

The dialog logic is in `src/core/config-1.ts` lines 12543–12565:

```typescript
// src/core/config-1.ts:12543-12565 (condensed)
if (z && z.length > 0) {   // z = parsed dev channel entries (p6)
  let [{ isChannelsEnabled: $ }, { getClaudeAIOAuthTokens: H }] =
    await Promise.all([
      Promise.resolve().then(() => (JC8(), hXq)),      // channels module
      Promise.resolve().then(() => (wA(), UI)),         // auth module
    ]);
  if (!$() || !H()?.accessToken)              // if NOT enabled OR NOT authenticated:
    (R$6([...ry(), ...z.map((j) => ({ ...j, dev: true }))]), d68(true)); // silent add
  else {
    // authenticated AND channels enabled: show dialog
    let { DevChannelsDialog: j } = await Promise.resolve().then(
      () => (vpq(), Gpq),
    );
    await Sy(A, (J) =>
      LN.default.createElement(j, {
        channels: z,
        onAccept: () => {
          (R$6([...ry(), ...z.map((M) => ({ ...M, dev: true }))]),
            d68(true),
            J());  // close dialog
        },
      }),
    );
  }
}
```

### Dialog Display Logic

| Condition | Behavior |
|-----------|----------|
| `!isChannelsEnabled() \|\| !accessToken` | Silently adds entries with `dev: true`, no dialog |
| `isChannelsEnabled() && accessToken` | Shows `DevChannelsDialog`, requires user choice |

### Dialog UI Structure

```typescript
// src/ui/devchannelsdialog-2.ts:47-107
x1({  // dialog container component
  title: "WARNING: Loading development channels",
  color: "error",
  onCancel: dNY,  // calls $K(0) — exits with code 0
},
  // Warning text 1:
  "--dangerously-load-development-channels is for local channel development only. "
  "Do not use this option to run channels you have downloaded off the internet.",
  // Warning text 2:
  "Please use --channels to run a list of approved channels.",
  // Channel list display:
  T({ dimColor: true }, "Channels: ", H),  // H = comma-joined channel names
  // Select menu:
  T1({ options: J, onChange: (D) => z(D) })  // J = options array
)
```

### User Options

```typescript
// src/ui/devchannelsdialog-2.ts:78-82
J = [
  { label: "I am using this for local development", value: "accept" },
  { label: "Exit",                                  value: "exit" },
];
```

### Option Handlers

```typescript
// src/ui/devchannelsdialog-2.ts:30-38
switch (P) {
  case "accept":
    _();   // calls onAccept → merges dev entries, sets hasDevChannels, closes dialog
    break;
  case "exit":
    $K(1); // calls process.exit(1)
}
```

```typescript
// src/ui/devchannelsdialog-2.ts:113-115
function dNY() {  // onCancel (Escape key)
  $K(0);  // process.exit(0) — clean exit
}
```

### Props

| Prop | Type | Description |
|------|------|-------------|
| `channels` | `ChannelEntry[]` | The dev channel entries to display |
| `onAccept` | `() => void` | Called when user selects "I am using this for local development" |

**Source**: `src/ui/devchannelsdialog-2.ts` lines 1–115; `src/core/config-1.ts` lines 12543–12566.

---

## 10. Channel Status UI — `ChannelsNotice`

`ChannelsNotice` is the React component that renders channel status information in the Claude Code startup output. Defined in `src/mcp/channelsnotice-1.ts` (Module `Jfq`, lines 510727–510983 of original cli.js), extracted to lines 24–266.

### State Calculation — `P5Y()`

```typescript
// src/mcp/channelsnotice-1.ts:212-233
function P5Y() {
  let A = ry();  // read current allowedChannels
  if (A.length === 0)
    return { channels: A, disabled: false, noAuth: false, policyBlocked: false, list: "" };
  let q = A.map(po6).join(", "),    // format channel list as string
    K = sq(),                        // account type
    _ = K === "team" || K === "enterprise",
    Y = N1("policySettings");        // managed policy settings
  return {
    channels: A,
    disabled: !Ho6(),                              // tengu_harbor is false
    noAuth: !hA()?.accessToken,                    // no OAuth token
    policyBlocked: _ && Y?.channelsEnabled !== true, // org blocked
    list: q,                                       // display string
  };
}
```

### Flag Label Logic

```typescript
// src/mcp/channelsnotice-1.ts:33-39
let H = K.some(D5Y),   // D5Y checks: !A.dev (i.e., has any non-dev entry)
  j =
    Q68() && H          // hasDevChannels AND has non-dev entries
      ? "Channels"                              // mixed: "Channels"
      : Q68()                                    // hasDevChannels but no non-dev
        ? "--dangerously-load-development-channels"  // only dev
        : "--channels";                          // only production
```

| Condition | Flag Label |
|-----------|------------|
| `hasDevChannels` AND has non-dev entries | `"Channels"` |
| `hasDevChannels` AND no non-dev entries | `"--dangerously-load-development-channels"` |
| No dev channels | `"--channels"` |

### Display Modes

#### 1. Early Return (empty)
```typescript
// src/mcp/channelsnotice-1.ts:32
if (K.length === 0) return null;  // no channels → render nothing
```

#### 2. Disabled State
```
<flag-label> ignored (channels are not currently available)
Channels are not currently available
```

Triggered when `disabled === true` (i.e., `!Ho6()`, `tengu_harbor` is false).

#### 3. No Authentication
```
<flag-label> ignored (Channels require claude.ai authentication · run /login, then restart)
Channels require claude.ai authentication · run /login, then restart
```

Triggered when `noAuth === true` (no `accessToken`).

#### 4. Policy Blocked
```
<flag-label> blocked by org policy (<channel-list>)
Inbound messages will be silently dropped
Have an administrator set channelsEnabled: true in managed settings to enable
  <entry>: <reason>     (per-entry warnings via W5Y())
```

Triggered when `policyBlocked === true` (team/enterprise without `channelsEnabled: true`).

#### 5. Active (normal operation)
```
Listening for channel messages from: plugin:name@marketplace, server:name
Experimental · inbound messages will be pushed into this session, this carries prompt injection risks. Restart Claude Code without <flag-label> to disable.
  <entry>: <reason>     (per-entry warnings via W5Y())
```

Source line 154: `"Listening for channel messages from: "` + the formatted list.
Source line 165: `"Experimental · inbound messages will be pushed into this session, this carries prompt injection risks. Restart Claude Code without "` + flag label + `" to disable."`

### Per-Entry Validation — `W5Y()`

```typescript
// src/mcp/channelsnotice-1.ts:239-266
function W5Y(A) {
  if (A.length === 0) return [];
  let q = ["enterprise", "user", "project", "local"],
    K = new Set();
  for (let w of q)                           // collect all configured MCP server names
    for (let O of Object.keys(GH(w).servers))
      K.add(O);
  let _ = new Set(Object.keys(HM().plugins)),  // installed plugin names
    Y = $o6(),                                  // tengu_harbor_ledger allowlist
    z = [];
  for (let w of A) {
    if (w.kind === "server") {
      if (!K.has(w.name))
        z.push({ entry: w, why: "no MCP server configured with that name" });
      if (!w.dev)
        z.push({ entry: w, why: "server: entries need --dangerously-load-development-channels" });
      continue;
    }
    // plugin:
    if (!_.has(`${w.name}@${w.marketplace}`))
      z.push({ entry: w, why: "plugin not installed" });
    if (
      !w.dev &&
      !Y.some((O) => O.plugin === w.name && O.marketplace === w.marketplace)
    )
      z.push({ entry: w, why: "not on the approved channels allowlist" });
  }
  return z;
}
```

### Per-Entry Warning Reasons

| Entry Type | Condition | Warning |
|-----------|-----------|----------|
| `server:` | Not in MCP config | `"no MCP server configured with that name"` |
| `server:` | No `dev: true` | `"server: entries need --dangerously-load-development-channels"` |
| `plugin:` | Not installed | `"plugin not installed"` |
| `plugin:` | Not on allowlist AND not dev | `"not on the approved channels allowlist"` |

Warnings are rendered by `M5Y` (for active mode) and `X5Y` (for policy-blocked mode), both identical in structure (lines 191–207):

```typescript
// src/mcp/channelsnotice-1.ts:191-207
function M5Y(A) {
  return zz.createElement(
    T,
    { key: `${po6(A.entry)}:${A.why}`, color: "warning" },
    po6(A.entry), " · ", A.why,
  );
}
```

**Source**: `src/mcp/channelsnotice-1.ts` lines 24–266.

---

## 11. Feature Detection and Storage

### Local Storage Keys

| Key | Type | Default | Purpose |
|-----|------|---------|----------|
| `tengu_harbor` | `boolean` | `false` | Master feature flag for MCP Channels |
| `tengu_harbor_ledger` | `Array<{plugin: string, marketplace: string}>` | `[]` | Allowlist of approved plugins |

### `tengu_harbor` — Feature Flag

```typescript
// src/telemetry/ischannelsenabled-1.ts:27-29
function Ho6() {
  return l8("tengu_harbor", false);  // l8 = localStorage getter with default
}
```

This flag is the single ON/OFF switch for the entire MCP Channels feature. When `false` (the default), all channel registrations fail at gate check 2 with `kind: "disabled"`. The flag is managed server-side by Anthropic and synced to local storage; individual users cannot enable it themselves.

### `tengu_harbor_ledger` — Allowlist

```typescript
// src/telemetry/ischannelsenabled-1.ts:22-26
function $o6() {
  let A = l8("tengu_harbor_ledger", []),
    q = O8Y().safeParse(A);
  return q.success ? q.data : [];  // returns empty array on parse failure
}
```

### `O8Y()` — Allowlist Schema

```typescript
// src/core/auth-1.ts:15131-15133
O8Y = g6(() =>
  h.array(h.object({ marketplace: h.string(), plugin: h.string() }))
);
```

`O8Y` is a lazily-initialized Zod schema (`g6` = memoized factory). It validates that `tengu_harbor_ledger` is an array of objects with `marketplace: string` and `plugin: string` fields. Invalid data in local storage is treated as an empty allowlist (`q.success ? q.data : []`).

### Schema Entries

Allowlist entries use a flat structure:

```typescript
interface AllowlistEntry {
  plugin: string;      // plugin name (without marketplace)
  marketplace: string; // marketplace identifier
}
```

Matching at gate check 7: `$o6().some((w) => w.plugin === Y.name && w.marketplace === Y.marketplace)`

### Data Flow

```
Anthropic servers
       │
       │  (sync to local storage during auth/update)
       ▼
local storage: "tengu_harbor" = true/false
local storage: "tengu_harbor_ledger" = [{plugin,marketplace}, ...]
       │
       ├── Ho6()  ──►  isChannelsEnabled check
       └── $o6()  ──►  allowlist check in xXq()
```

**Source**: `src/telemetry/ischannelsenabled-1.ts` lines 22–29; `src/core/auth-1.ts` lines 15131–15133.

---

## 12. Enterprise Policy Configuration

### The `channelsEnabled` Policy Setting

For organizations on team or enterprise plans, the `channelsEnabled` boolean in `policySettings` controls whether any member can use MCP Channels.

```typescript
// src/core/auth-1.ts:15166-15175
let _ = sq();  // account type
if (_ === "team" || _ === "enterprise") {
  if (N1("policySettings")?.channelsEnabled !== !0)
    return {
      action: "skip",
      kind: "policy",
      reason: "channels not enabled by org policy (set channelsEnabled: true in managed settings)",
    };
}
```

### Effect Scope

| Account Type | Policy Check Applied |
|-------------|----------------------|
| `free` | No — channels enabled if `tengu_harbor` is true |
| `pro` | No — channels enabled if `tengu_harbor` is true |
| `team` | Yes — requires `channelsEnabled: true` in managed settings |
| `enterprise` | Yes — requires `channelsEnabled: true` in managed settings |

### Admin Configuration

Administrators set this via the managed settings JSON:

```json
{
  "policySettings": {
    "channelsEnabled": true
  }
}
```

`N1("policySettings")` reads this from the managed settings layer.

### UI Guidance

The `ChannelsNotice` component (line 125) shows org members the admin action required:

```
Have an administrator set channelsEnabled: true in managed settings to enable
```

### Interaction With Dev Channels

The policy check is NOT bypassed by `dev: true`. Even with `--dangerously-load-development-channels`, team/enterprise users without `channelsEnabled: true` will still fail gate check 4 during server registration. However, the dev channels dialog behavior is different: if `!isChannelsEnabled() || !accessToken`, it silently adds entries without showing the dialog (line 12550):

```typescript
if (!$() || !H()?.accessToken)
  // Silent add — channels are blocked anyway, dialog would be confusing
  (R$6([...ry(), ...z.map((j) => ({ ...j, dev: true }))]), d68(true));
```

This means the dialog is only shown to users who WOULD be able to use channels (authenticated + feature enabled), where the dev bypass is actually meaningful.

**Source**: `src/core/auth-1.ts` lines 15166–15175; `src/mcp/channelsnotice-1.ts` lines 116–148; `src/core/config-1.ts` lines 12543–12566.

---

## 13. Telemetry Events

### Event: `tengu_mcp_channel_flags`

Emitted once at startup in `src/agents/startdeferredprefetches-1.ts` lines 1040–1047, whenever at least one channel entry (production or dev) is specified:

```typescript
// src/agents/startdeferredprefetches-1.ts:1040-1047
Q("tengu_mcp_channel_flags", {
  channels_count: q7.length,       // number of --channels entries
  dev_count: p6?.length ?? 0,      // number of --dangerously-load-development-channels entries
  plugins: v4(q7),                 // comma-sorted plugin names (format: name@marketplace)
  dev_plugins: v4(p6 ?? []),       // comma-sorted dev plugin names
});
```

The `v4` helper (defined inline):
```typescript
let v4 = (QK) => {
  let qq = QK.flatMap((mA) =>
    mA.kind === "plugin" ? [`${mA.name}@${mA.marketplace}`] : [],
  );
  return qq.length > 0 ? qq.sort().join(",") : void 0;  // undefined if no plugins
};
```

| Field | Type | Description |
|-------|------|-------------|
| `channels_count` | `number` | Count of `--channels` entries (includes both plugin: and server:) |
| `dev_count` | `number` | Count of `--dangerously-load-development-channels` entries |
| `plugins` | `string \| undefined` | Comma-separated sorted plugin names from `--channels`; undefined if none |
| `dev_plugins` | `string \| undefined` | Comma-separated sorted plugin names from dev flag; undefined if none |

**Note**: `server:` entries are counted in `channels_count`/`dev_count` but NOT included in `plugins`/`dev_plugins` (which are plugin-only).

### Event: `tengu_mcp_channel_gate`

Emitted in `src/mcp/mcp-4.ts` lines 1656–1663 for each server connection attempt that reaches the gate check, with a condition that silently skips `capability` failures:

```typescript
// src/mcp/mcp-4.ts:1651-1663
let E = xXq(G.name, G.capabilities, G.config.pluginSource),
  R = jo6(G.name, ry()),
  S = R?.kind === "plugin" ? `${R.name}@${R.marketplace}` : void 0;
if (E.action === "register" || E.kind !== "capability")  // skip silent capability failures
  Q("tengu_mcp_channel_gate", {
    registered: E.action === "register",
    skip_kind: E.action === "skip" ? E.kind : void 0,
    entry_kind: R?.kind,       // "plugin" | "server" | undefined
    is_dev: R?.dev ?? false,
    plugin: S,                 // "name@marketplace" string or undefined
  });
```

| Field | Type | Description |
|-------|------|-------------|
| `registered` | `boolean` | `true` if registration succeeded |
| `skip_kind` | `string \| undefined` | Reason for skip: `disabled`, `auth`, `policy`, `session`, `marketplace`, `allowlist` |
| `entry_kind` | `"plugin" \| "server" \| undefined` | Kind of the matched allowedChannels entry |
| `is_dev` | `boolean` | Whether the entry has `dev: true` |
| `plugin` | `string \| undefined` | Plugin identifier `"name@marketplace"` if applicable |

**Telemetry condition**: Only emitted when `E.action === "register"` OR `E.kind !== "capability"`. Pure capability misses (servers that don't declare the channel capability at all) are NOT logged to reduce noise from ordinary MCP servers.

### Event: `tengu_mcp_channel_message`

Emitted in `src/mcp/mcp-4.ts` lines 1672–1678 for each inbound channel notification received:

```typescript
// src/mcp/mcp-4.ts:1672-1678
Q("tengu_mcp_channel_message", {
  content_length: b.length,              // character length of the message content
  meta_key_count: Object.keys(g ?? {}).length,  // number of meta object keys
  entry_kind: R?.kind,                   // "plugin" | "server" | undefined
  is_dev: R?.dev ?? false,
  plugin: S,                             // plugin identifier or undefined
});
```

| Field | Type | Description |
|-------|------|-------------|
| `content_length` | `number` | Character length of the channel message content |
| `meta_key_count` | `number` | Number of keys in the `meta` object (0 if no meta) |
| `entry_kind` | `"plugin" \| "server" \| undefined` | Channel entry kind |
| `is_dev` | `boolean` | Whether the channel is a dev entry |
| `plugin` | `string \| undefined` | Plugin identifier if applicable |

**Source**: `src/agents/startdeferredprefetches-1.ts` lines 1040–1047; `src/mcp/mcp-4.ts` lines 1651–1678.

---

## 14. Security Model

### Defense in Depth

The MCP Channels security model applies multiple independent layers of control:

#### Layer 1 — Feature Flag (Server-Controlled)

The `tengu_harbor` flag is the outermost gate. It defaults to `false` and can only be enabled by Anthropic distributing a value of `true` to local storage. Users cannot self-enable this feature.

#### Layer 2 — Authentication

Claude.ai OAuth authentication is required. This ties channel usage to a verified Anthropic account and provides an audit trail. API-key-only setups (non-claude.ai) cannot use channels.

#### Layer 3 — Organization Policy

Enterprise and team administrators retain full control via `channelsEnabled`. This gives organizations a kill switch even when Anthropic enables the feature globally.

#### Layer 4 — Session Allowlist

The user explicitly opts in per-session by listing specific servers on the command line. There is no ambient channel registration — every channel must be explicitly specified at session start.

#### Layer 5 — Allowlist Verification

Production plugins must appear on the Anthropic-managed `tengu_harbor_ledger`. This prevents arbitrary plugins from declaring channel capability and registering without Anthropic's approval.

#### Layer 6 — Marketplace Verification

The installed plugin's marketplace source must match what was specified on the command line. This prevents supply-chain substitution attacks where a different (malicious) plugin from a different marketplace registers under the same name.

#### Layer 7 — MCP Capability Declaration

Servers must explicitly declare `experimental["claude/channel"]` in their capabilities. Ordinary MCP servers cannot accidentally become channels.

#### Layer 8 — Dev Bypass Controls

The dev bypass (`--dangerously-load-development-channels`) requires:
- Explicit use of the long-form flag name (the word "dangerously" is deliberate)
- A mandatory confirmation dialog (when auth + feature are present)
- Confirmation that the user is doing local development

### Prompt Injection Risk

The system explicitly acknowledges and warns about prompt injection. Channel messages become prompts in the session context with no filtering. The `ChannelsNotice` component shows:

```
Experimental · inbound messages will be pushed into this session,
this carries prompt injection risks.
```

Mitigations implemented:
1. `skipSlashCommands: true` — channel messages cannot trigger slash commands
2. XML wrapping — content is wrapped in `<channel source="...">` tags, giving the model context about the message origin
3. `isMeta: true` — marked as metadata, not user input
4. `origin: { kind: "channel", server: G.name }` — origin tracking in the prompt queue

### What Is NOT Mitigated

- The model itself can be influenced by the channel message content. A malicious channel server that passes all security checks can still attempt prompt injection attacks against the model.
- There is no content filtering or sanitization of the `content` field.
- Meta values are only filtered by key name validity (`$8Y = /^[a-zA-Z_][a-zA-Z0-9_]*$/`), not by value content.

### Security Properties Table

| Threat | Mitigation | Strength |
|--------|-----------|----------|
| Unauthorized channel feature use | `tengu_harbor` flag (server-controlled) | Strong |
| Unauthenticated channel use | OAuth token required | Strong |
| Org-wide abuse | `channelsEnabled` admin policy | Strong |
| Arbitrary server registration | Session allowlist | Moderate |
| Malicious plugin from wrong marketplace | Marketplace verification | Moderate |
| Unapproved plugin in production | `tengu_harbor_ledger` allowlist | Strong |
| Accidental channel registration by ordinary servers | MCP capability requirement | Strong |
| Slash command injection via channel | `skipSlashCommands: true` | Strong |
| Dev channels without user awareness | Mandatory confirmation dialog | Moderate |
| Model prompt injection | XML wrapping, `isMeta`, origin tracking | Weak (advisory only) |

**Source**: `src/core/auth-1.ts` lines 15149–15209; `src/mcp/mcp-4.ts` lines 1679–1686; `src/mcp/channelsnotice-1.ts` lines 163–168.

---

## 15. Complete Flow Diagram

```
╔══════════════════════════════════════════════════════════════════════╗
║                    MCP CHANNELS FLOW — v2.1.81                      ║
╚══════════════════════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 1: CLI ARGUMENT PARSING                                      │
│  src/agents/startdeferredprefetches-1.ts:995-1048                   │
└─────────────────────────────────────────────────────────────────────┘

  CLI invocation:
    claude --channels plugin:name@market server:my-mcp
           --dangerously-load-development-channels plugin:dev@local
          │
          ▼
  Z6 = K7() — is non-interactive?  ──YES──► Skip all channel processing
          │
         NO
          │
          ▼
  y8(DA, "--dangerously-...")       ──► parse dev entries → p6[]
  y8(G7, "--channels")             ──► parse prod entries → q7[]
          │
          ├── Invalid entry found? ──YES──► process.stderr.write(error)
          │                                process.exit(1)
         NO
          │
          ▼
  R$6(q7)                          ──► T8.allowedChannels = q7[]
  Q("tengu_mcp_channel_flags", {   ──► telemetry
    channels_count, dev_count,
    plugins, dev_plugins
  })

┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 2: DEV CHANNELS DIALOG (if --dangerously flag used)          │
│  src/core/config-1.ts:12543-12566                                   │
└─────────────────────────────────────────────────────────────────────┘

  p6.length > 0?
       │
      YES
       │
       ▼
  isChannelsEnabled() && accessToken?  ──NO──► R$6([...ry(), ...p6.dev]) ; d68(true)
       │                                       (silent add, no dialog)  │
      YES                                                                │
       │                                                                 │
       ▼                                                                 │
  Show DevChannelsDialog:                                               │
    "WARNING: Loading development channels"                            │
    Options:                                                            │
    [I am using this for local development] ──────────────────────────►│
    [Exit] ──► process.exit(1)                                         │
                                                                        │
  onAccept():                             ◄──────────────────────────── │
    R$6([...ry(), ...p6.map(dev:true)])   (merge dev entries with dev flag)
    d68(true)                             (set hasDevChannels = true)

┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 3: MCP CONNECTION + CHANNEL GATE                             │
│  src/mcp/mcp-4.ts:1650-1703                                         │
│  src/core/auth-1.ts:15149-15209                                     │
└─────────────────────────────────────────────────────────────────────┘

  For each MCP server G that connects:
       │
       ▼
  E = xXq(G.name, G.capabilities, G.config.pluginSource)
       │
       ├─ Check 1: G.capabilities?.experimental["claude/channel"]?
       │           NO ──► {action:"skip", kind:"capability"}
       │                  (no telemetry — ordinary server)
       ├─ Check 2: Ho6() — tengu_harbor flag?
       │           NO ──► {action:"skip", kind:"disabled"}
       ├─ Check 3: hA()?.accessToken?
       │           NO ──► {action:"skip", kind:"auth"}
       ├─ Check 4: if team/enterprise: policySettings.channelsEnabled?
       │           NO ──► {action:"skip", kind:"policy"}
       ├─ Check 5: jo6(G.name, ry()) — in allowedChannels?
       │           NO ──► {action:"skip", kind:"session"}
       ├─ Check 6: if plugin: marketplace match?
       │           NO ──► {action:"skip", kind:"marketplace"}
       ├─ Check 7: if !dev: on tengu_harbor_ledger allowlist?
       │           NO ──► {action:"skip", kind:"allowlist"}
       │
       └─► {action: "register"}
              │
              ▼
  Q("tengu_mcp_channel_gate", {registered, skip_kind, entry_kind, is_dev, plugin})
              │
  E.action === "register"?
  YES ──────────────────────────────────────────────────────────────────────►│
                                                                              │
  NO (skip):                                                                  │
    G.client.removeNotificationHandler("notifications/claude/channel")        │
    G.client.removeNotificationHandler(rn1)                                   │
    a8(G.name, "Channel notifications skipped: " + E.reason)                 │
    (some skip kinds shown in ChannelsNotice warnings)                        │

┌─────────────────────────────────────────────────────────────────────┐  ◄──┘
│  PHASE 4: CHANNEL REGISTRATION                                      │
│  src/mcp/mcp-4.ts:1664-1699                                         │
└─────────────────────────────────────────────────────────────────────┘

  G.client.setNotificationHandler(SXq(), handler)   ──► registers channel handler
  if experimental["claude/channel/permission"]:
    G.client.setNotificationHandler(CXq(), handler) ──► registers permission handler
  a8(G.name, "Channel notifications registered")

┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 5: STARTUP UI — ChannelsNotice                               │
│  src/mcp/channelsnotice-1.ts:24-266                                 │
└─────────────────────────────────────────────────────────────────────┘

  P5Y() computes display state:
    ry().length === 0? ──► null (render nothing)
    disabled?    ──► "<flag> ignored (channels are not currently available)"
    noAuth?      ──► "<flag> ignored (Channels require claude.ai authentication...)"
    policyBlocked? ──► "<flag> blocked by org policy" + admin instructions
    else:        ──► "Listening for channel messages from: <list>"
                     + injection risk warning
                     + W5Y() per-entry warnings

┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 6: RUNTIME — Inbound Message Handling                        │
│  src/mcp/mcp-4.ts:1667-1686                                         │
└─────────────────────────────────────────────────────────────────────┘

  MCP Server sends: notifications/claude/channel
    { content: "Build failed on main", meta: {branch: "main"} }
          │
          ▼
  a8(G.name, `notifications/claude/channel: ${b.slice(0, 80)}`)
  Q("tengu_mcp_channel_message", {content_length, meta_key_count, ...})
          │
          ▼
  OX({
    mode: "prompt",
    value: bXq(G.name, content, meta),   ──► "<channel source=\"...\">\ncontent\n</channel>"
    priority: "next",                        processed in next session turn
    isMeta: true,
    origin: { kind: "channel", server: G.name },
    skipSlashCommands: true,
  })
          │
          ▼
  Session receives injected prompt in next turn
  Model processes: <channel source="server-name" branch="main">\nBuild failed on main\n</channel>
```

---

## 16. Quick Reference

### All Validation Gate Checks

| # | Function | Check | Kind on Failure | Dev Bypass |
|---|---------|-------|-----------------|------------|
| 1 | `xXq` | `q?.experimental["claude/channel"]` present | `capability` | No |
| 2 | `Ho6()` | `tengu_harbor` local storage = true | `disabled` | No |
| 3 | `hA()?.accessToken` | OAuth token present | `auth` | No |
| 4 | `N1("policySettings")?.channelsEnabled` | Policy allows (team/enterprise only) | `policy` | No |
| 5 | `jo6(A, ry())` | Server in session's allowedChannels | `session` | No |
| 6 | `Hq(K).marketplace === Y.marketplace` | Plugin marketplace matches (plugin only) | `marketplace` | No |
| 7 | `$o6().some(...)` | Plugin on `tengu_harbor_ledger` (plugin, non-dev) | `allowlist` | Yes |
| 7b | `!Y.dev` | Server on `tengu_harbor_ledger` (server, non-dev) | `allowlist` | Yes |

### All Telemetry Events

| Event | Emitted When | Source |
|-------|-------------|--------|
| `tengu_mcp_channel_flags` | Startup, if any channels specified | `startdeferredprefetches-1.ts:1040` |
| `tengu_mcp_channel_gate` | Each server connection (excluding capability-only failures) | `mcp-4.ts:1656` |
| `tengu_mcp_channel_message` | Each inbound channel notification | `mcp-4.ts:1672` |

### Channel Entry Format

| Format | Kind | Required Fields | Optional Fields |
|--------|------|----------------|------------------|
| `plugin:name@marketplace` | `"plugin"` | `name`, `marketplace` | `dev?: true` |
| `server:name` | `"server"` | `name` | `dev?: true` |

### Feature Flags and Storage Keys

| Key | Storage | Default | Controlled By | Purpose |
|-----|---------|---------|---------------|---------|
| `tengu_harbor` | localStorage | `false` | Anthropic | Master ON/OFF for MCP Channels |
| `tengu_harbor_ledger` | localStorage | `[]` | Anthropic | Allowlist of approved plugins |
| `policySettings.channelsEnabled` | Managed settings | `undefined` | Org admins | Team/enterprise policy opt-in |

### Key Functions and Locations

| Function | File | Lines | Role |
|----------|------|-------|------|
| `y8` (inline arrow) | `startdeferredprefetches-1.ts` | 995–1032 | Parse CLI args to ChannelEntry[] |
| `ry()` | `updatelastinteractiontime-1.ts` | 1051–1053 | Get `T8.allowedChannels` |
| `R$6(A)` | `updatelastinteractiontime-1.ts` | 1054–1056 | Set `T8.allowedChannels` |
| `Q68()` | `updatelastinteractiontime-1.ts` | 1057–1059 | Get `T8.hasDevChannels` |
| `d68(A)` | `updatelastinteractiontime-1.ts` | 1060–1062 | Set `T8.hasDevChannels` |
| `xXq(A, q, K)` | `auth-1.ts` | 15149–15209 | 8-check validation gate |
| `jo6(A, q)` | `auth-1.ts` | 15143–15147 | Find entry in allowedChannels |
| `bXq(A, q, K)` | `auth-1.ts` | 15134–15142 | Format channel message as XML |
| `Ho6()` | `ischannelsenabled-1.ts` | 27–29 | Read `tengu_harbor` flag |
| `$o6()` | `ischannelsenabled-1.ts` | 22–26 | Read `tengu_harbor_ledger` |
| `O8Y()` | `auth-1.ts` | 15131–15133 | Zod schema for allowlist |
| `J5Y()` (ChannelsNotice) | `channelsnotice-1.ts` | 24–189 | Startup status UI component |
| `P5Y()` | `channelsnotice-1.ts` | 212–233 | Compute display state |
| `W5Y(A)` | `channelsnotice-1.ts` | 239–266 | Per-entry warning validation |
| `UNY()` (DevChannelsDialog) | `devchannelsdialog-2.ts` | 25–107 | Dev channels confirmation dialog |
| `FXq()` | `telemetry/events-1.ts` | 2594–2612 | Permission queue factory |

### State Object Fields (T8)

| Field | Type | Default | Getter | Setter |
|-------|------|---------|--------|--------|
| `T8.allowedChannels` | `ChannelEntry[]` | `[]` | `ry()` | `R$6(A)` |
| `T8.hasDevChannels` | `boolean` | `false` | `Q68()` | `d68(A)` |

### CLI Flag Summary

| Flag | Variadic | Hidden | Production Allowlist | Dev Dialog | Entries Tagged |
|------|----------|--------|---------------------|------------|----------------|
| `--channels <servers...>` | Yes | Yes | Required | No | `dev: undefined` |
| `--dangerously-load-development-channels <servers...>` | Yes | Yes | Bypassed | Yes (if auth+enabled) | `dev: true` |

### MCP Notification Names

| Notification | Constant | Direction | Purpose |
|-------------|----------|-----------|----------|
| `notifications/claude/channel` | `SXq()` schema | Server → Client | Push message to session |
| `notifications/claude/channel/permission` | `rn1`, `CXq()` schema | Server → Client | Respond to pending permission |
| `notifications/claude/channel/permission_request` | `IXq` | Client → Server | (request constant) |

---

*Document generated from Claude Code CLI v2.1.81 extracted source files. All line references verified against files in `src/`.*
