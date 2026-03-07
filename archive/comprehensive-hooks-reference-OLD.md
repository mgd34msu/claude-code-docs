# Claude Code CLI v2.1.59 — Comprehensive Hooks Reference

> **This is the definitive reference for the Claude Code hook system as of v2.1.59.** It covers every hook event, every hook type, every configuration field, and the full execution pipeline — verified directly from source code. A reader finishing this document should be able to build funtionality that relies on native Claude Code hooks, debug problems and troubleshoot any errors, and reason about any hook configuration without needing another source.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Quick Start](#2-quick-start)
3. [Configuration](#3-configuration)
   - 3.1 [settings.json Schema](#31-settingsjson-schema)
   - 3.2 [Configuration Locations and Merge Order](#32-configuration-locations-and-merge-order)
   - 3.3 [Hook Types](#33-hook-types)
4. [Hook Event Reference](#4-hook-event-reference)
   - 4.1 [PreToolUse](#41-pretooluse)
   - 4.2 [PostToolUse](#42-posttooluse)
   - 4.3 [PostToolUseFailure](#43-posttoolusefailure)
   - 4.4 [PermissionRequest](#44-permissionrequest)
   - 4.5 [Notification](#45-notification)
   - 4.6 [UserPromptSubmit](#46-userpromptsubmit)
   - 4.7 [SessionStart](#47-sessionstart)
   - 4.8 [SessionEnd](#48-sessionend)
   - 4.9 [Stop](#49-stop)
   - 4.10 [SubagentStart](#410-subagentstart)
   - 4.11 [SubagentStop](#411-subagentstopp)
   - 4.12 [PreCompact](#412-precompact)
   - 4.13 [Setup](#413-setup)
   - 4.14 [TeammateIdle](#414-teammateidle)
   - 4.15 [TaskCompleted](#415-taskcompleted)
   - 4.16 [ConfigChange](#416-configchange)
   - 4.17 [WorktreeCreate](#417-worktreecreate)
   - 4.18 [WorktreeRemove](#418-worktreeremove)
5. [Hook Matching](#5-hook-matching)
6. [Hook Execution Pipeline](#6-hook-execution-pipeline)
7. [Exit Code Reference](#7-exit-code-reference)
8. [Environment Variables](#8-environment-variables)
9. [Async and Background Hooks](#9-async-and-background-hooks)
10. [Plugin Hook System](#10-plugin-hook-system)
11. [Hook Policies and Disabling](#11-hook-policies-and-disabling)
12. [Error Handling](#12-error-handling)
13. [Appendix: hookSpecificOutput by Event](#13-appendix-hookspecificoutput-by-event)

---

## 1. Introduction

Hooks are user-defined scripts (or Claude agent invocations, or HTTP endpoints) that Claude Code calls at specific points during its operation. They let you observe, modify, block, or extend Claude's behavior without modifying the CLI itself.

**What hooks can do:**
- Block tool calls before they execute (e.g., deny writes to protected paths)
- Modify tool inputs before execution (e.g., add `--dry-run` flags automatically)
- React to tool outputs (e.g., run linters after every file write)
- Auto-approve or auto-deny permission dialogs
- Inject context into the model at session start, after tools run, or on user prompt submit
- Forward notifications to external systems (e.g., Slack, desktop notifications)
- Control whether Claude stops or continues after completing a response
- Customize worktree creation and cleanup
- Validate or block context compaction
- Block config changes from applying

**Where hooks are configured:**

Hooks are defined in `settings.json` files at multiple levels (user, project, local, policy). See [Section 3.2](#32-configuration-locations-and-merge-order) for the full merge order.

**Architecture summary:**

The entire hook system is implemented in `src/tools/hasworktreecreatehook-1.ts` (1753 lines). The central dispatch function is `wx()` (an async generator), which: checks preconditions, finds matching hooks via `Hd8()`, executes them in parallel, and yields results (blocking errors, context, permission decisions, updated inputs) to the caller. Plugin hook loading and hot-reload lives in `src/tools/setuppluginhookhotreload-2.ts`.

**Hook execution preconditions (`wx()`, lines 635-651):**

Before any hooks run, `wx()` evaluates four early-return conditions in order:

| Order | Check | Function | Returns Early When | Source |
|-------|-------|----------|--------------------|--------|
| 1 | Managed disable | `Kn6()` | `policySettings.disableAllHooks === true` | `src/core/gg-4.ts:33` |
| 2 | Simple mode | env var `CLAUDE_CODE_SIMPLE` | truthy string (`_1()` check) | `hasworktreecreatehook-1.ts:646` |
| 3 | Workspace trust | `RL1()` | Trust NOT accepted AND not in restricted mode (`!vw() && !b4()`) | `hasworktreecreatehook-1.ts:72` |
| 4 | No matches | — | `Hd8()` returns zero matching hooks | `hasworktreecreatehook-1.ts:656` |

- **`CLAUDE_CODE_SIMPLE`** (env var): Setting this environment variable to any truthy value disables ALL hooks unconditionally. Checked at `hasworktreecreatehook-1.ts:646` and `hasworktreecreatehook-1.ts:1156` (for `mN6()` outside-REPL path). This is the lightest-weight way to disable hooks globally.
- **`Kn6()`** (`src/core/gg-4.ts:33`): Reads `policySettings.disableAllHooks`. Returns `true` when a managed/organization policy has set `disableAllHooks = true`. In that case hooks are disabled for all users in the organization regardless of their personal settings.
- **`RL1()`** (`hasworktreecreatehook-1.ts:72`): Returns `true` (and skips hooks) when workspace trust has NOT been accepted (`!vw()`) AND the session is not already in restricted mode (`!b4()`). Once the user accepts workspace trust, `vw()` returns `true` and this guard passes.
- **`FI()`** (`src/core/gg-4.ts:27`): Returns `true` when `policySettings.allowManagedHooksOnly === true`, OR when user settings have `disableAllHooks: true` but policy settings do NOT. When `FI()` is true, only policy-defined hooks and (non-plugin-root) plugin hooks are loaded; all user and workspace hooks are skipped. This is the "managed-only" enforcement mode.

---

## 2. Quick Start

The minimal example: run a script after every Bash tool call.

**~/.claude/settings.json**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Bash ran' >> /tmp/claude-audit.log"
          }
        ]
      }
    ]
  }
}
```

What this does:
1. After every successful Bash tool call, Claude Code spawns a child process running the echo command.
2. The hook input JSON is written to the hook's stdin. The base fields are always present:
   ```json
   {
     "session_id": "<uuid>",
     "transcript_path": "/path/to/transcript/<session_id>.jsonl",
     "cwd": "/current/working/directory",
     "permission_mode": "default",
     "hook_event_name": "PostToolUse",
     "tool_name": "Bash",
     "tool_input": { "command": "ls -la" },
     "tool_response": { "stdout": "...", "stderr": "", "interrupted": false }
   }
   ```
   The four base fields (`session_id`, `transcript_path`, `cwd`, `permission_mode`) come from `EO()` at `hasworktreecreatehook-1.ts:76-84`. Event-specific fields (`hook_event_name`, `tool_name`, etc.) are added by each event's builder function.
3. Exit code 0 = success, no blocking effect.
4. Stdout is captured and shown in transcript mode (Ctrl+O).

To block a tool call instead:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/validate-bash.sh"
          }
        ]
      }
    ]
  }
}
```

**validate-bash.sh:**
```bash
#!/bin/bash
# Read hook input from stdin
input=$(cat)
command=$(echo "$input" | jq -r '.tool_input.command')

# Block rm -rf commands
if echo "$command" | grep -q 'rm -rf'; then
  echo "Dangerous rm -rf command blocked" >&2
  exit 2  # exit 2 = blocking error shown to model
fi

exit 0
```

The key rule: **exit 2 blocks**. Stderr on exit 2 is shown to the model as a blocking error message. Any other non-zero exit shows stderr to the user only (non-blocking).

**Full JSON output example:**

Hook scripts can print JSON to stdout for maximum control. If stdout begins with `{`, Claude Code attempts to parse it as structured output. The complete schema:

```json
{
  "continue": true,
  "suppressOutput": false,
  "stopReason": "optional string — used when continue is false",
  "decision": "approve",
  "reason": "optional explanation for approve/block decisions",
  "systemMessage": "optional text injected as system message into the model context",
  "permissionDecision": "allow",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "optional explanation",
    "updatedInput": { },
    "additionalContext": "optional context injected into model"
  }
}
```

All fields are optional. If stdout does not start with `{`, or if the JSON does not match the expected schema, the output is treated as plain text (shown in transcript only). Exit code is still the primary blocking mechanism — JSON output adds extra control on top.

For the `hookSpecificOutput.hookEventName` field, use the exact event name matching the event your hook is registered for (`"PreToolUse"`, `"PostToolUse"`, `"UserPromptSubmit"`, etc.). See [Section 14](#14-appendix-hookspecificoutput-by-event) for the full per-event field list.

---

## 3. Configuration

### 3.1 settings.json Schema

The top-level `hooks` key in any settings file maps event names to arrays of matcher groups:

```json
{
  "hooks": {
    "<EventName>": [
      {
        "matcher": "<pattern>",
        "hooks": [
          { "type": "command", "command": "..." },
          { "type": "command", "command": "..." }
        ]
      },
      {
        "matcher": "<other-pattern>",
        "hooks": [
          { "type": "command", "command": "..." }
        ]
      }
    ],
    "<OtherEventName>": [
      {
        "hooks": [
          { "type": "command", "command": "..." }
        ]
      }
    ]
  }
}
```

**Structure breakdown:**

| Level | Type | Description |
|-------|------|-------------|
| `hooks` | object | Top-level key. Maps event name → array of matcher groups |
| `hooks["EventName"]` | array | List of matcher groups for this event |
| Matcher group | object | Has `matcher` (optional) + `hooks` (required array) |
| `matcher` | string | Pattern to filter which tool/source/type triggers the group. Omit or use `"*"` for all |
| `hooks` (inner) | array | The actual hook definitions to run when matcher matches |

**Matcher pattern rules:**

The `matcher` string is tested against a specific field of the hook input, depending on the event type:

| Event | Matched against |
|-------|-----------------|
| `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest` | `tool_name` |
| `SessionStart` | `source` |
| `Setup` | `trigger` |
| `PreCompact` | `trigger` |
| `Notification` | `notification_type` |
| `SessionEnd` | `reason` |
| `SubagentStart`, `SubagentStop` | `agent_type` |
| `TeammateIdle`, `TaskCompleted` | *(no match field — matcher is ignored, group always matches)* |
| `ConfigChange` | `source` |
| `UserPromptSubmit`, `Stop`, `WorktreeCreate`, `WorktreeRemove` | *(no match field — matcher is ignored, group always matches)* |

**Matcher syntax (evaluated in order):**

1. **Omitted or `"*"`** — matches everything.
2. **Alphanumeric with `|`** (e.g., `"Write|Edit"`) — pipe-separated exact match. Matches if any segment equals the field value (case-insensitive via `wN()` normalization).
3. **Plain alphanumeric** (e.g., `"Bash"`) — exact match against the field value.
4. **Anything else** — treated as a JavaScript `RegExp`. Also tested against aliases of the value (e.g., a tool might have alternate names). If the regex is invalid, the group silently does not match.

Examples:
```json
{ "matcher": "Bash" }          // exact: only Bash tool
{ "matcher": "Write|Edit" }    // pipe: Write or Edit tool
{ "matcher": "/^mcp__.*$/" }   // regex: any MCP tool
{ "matcher": "idle_prompt" }   // exact: Notification type idle_prompt
```

**Full example with multiple events:**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "node /scripts/check-file-permissions.js",
            "timeout": 10,
            "statusMessage": "Checking permissions..."
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/scripts/validate-bash.sh",
            "timeout": 5
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx eslint --fix $(cat /dev/stdin | jq -r '.tool_input.file_path')"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "cat /project/.claude-context.md"
          }
        ]
      }
    ],
    "Notification": [
      {
        "matcher": "idle_prompt",
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude is waiting\" with title \"Claude Code\"'"
          }
        ]
      }
    ]
  }
}
```

**Hook definition Zod schemas (source: `src/vendor/react-ink.ts:6045-6145`):**

The four user-configurable hook types are defined as Zod schemas and combined into a discriminated union `vw4 = B.discriminatedUnion("type", [lT5, iT5, rT5, nT5])` at line 6136. Each matcher group is `kw4 = B.object({ matcher: B.string().optional(), hooks: B.array(vw4) })` at line 6137. The full hooks config is a partial record mapping event names to arrays of matcher groups.

**CommandHook (`lT5`, `react-ink.ts:6045-6061`):**

```typescript
// lT5 = B.object({...})
{
  type: z.literal("command"),           // required
  command: z.string(),                   // required — shell command to execute
  timeout: z.number().positive().optional(),  // optional — seconds
  statusMessage: z.string().optional(),  // optional — spinner text
  once: z.boolean().optional(),          // optional — remove after first run
  async: z.boolean().optional(),         // optional — background execution
}
```

**PromptHook (`iT5`, `react-ink.ts:6062-6084`):**

```typescript
// iT5 = B.object({...})
{
  type: z.literal("prompt"),            // required
  prompt: z.string(),                    // required — use $ARGUMENTS placeholder
  timeout: z.number().positive().optional(),
  model: z.string().optional(),          // e.g. "claude-sonnet-4-6"
  statusMessage: z.string().optional(),
  once: z.boolean().optional(),
}
```

**AgentHook (`rT5`, `react-ink.ts:6111-6135`):**

```typescript
// rT5 = B.object({...})
{
  type: z.literal("agent"),             // required
  prompt: z.string()                     // required — transformed to function at parse time:
    .transform((A) => (q) => A),         //   prompt string becomes () => string internally
  timeout: z.number().positive().optional(),  // default 60s for agent type
  model: z.string().optional(),          // default: Haiku
  statusMessage: z.string().optional(),
  once: z.boolean().optional(),
}
```

**HttpHook (`nT5`, `react-ink.ts:6085-6110`):**

```typescript
// nT5 = B.object({...})
{
  type: z.literal("http"),              // required
  url: z.string().url(),                // required — validated as URL
  timeout: z.number().positive().optional(),
  headers: z.record(z.string(), z.string()).optional(),  // env var interpolation supported
  allowedEnvVars: z.array(z.string()).optional(),  // whitelist for $VAR substitution in headers
  statusMessage: z.string().optional(),
  once: z.boolean().optional(),
}
```

> **Note:** HttpHook (`type: "http"`) is NOT supported for `SessionStart` or `Setup` events. When the event is `SessionStart` or `Setup`, HTTP hooks are filtered out before execution at `hasworktreecreatehook-1.ts:594-600`. For all other events, HTTP hooks also silently skip at execution time in v2.1.59 (not yet implemented).

**Zod response schema (`vL1`, `src/tools/tool-1.ts:10590-10665`):**

Hook output (whether from command stdout, prompt response, or agent final message) is validated through:

```typescript
vL1 = B.union([sHz, tHz])
```

Where `sHz` is the async acknowledgment schema and `tHz` is the synchronous response schema:

```typescript
// sHz (async path, tool-1.ts:10590-10593)
{
  async: z.literal(true),              // hook signaled async mode
  asyncTimeout: z.number().optional(), // optional async timeout override
}

// tHz (sync path, tool-1.ts:10594-10664)
{
  continue: z.boolean().optional(),        // false = stop Claude after hook
  suppressOutput: z.boolean().optional(),  // true = hide stdout from transcript
  stopReason: z.string().optional(),       // message when continue is false
  decision: z.enum(["approve", "block"]).optional(),
  reason: z.string().optional(),           // explanation for the decision
  systemMessage: z.string().optional(),    // warning message shown to user
  hookSpecificOutput: z.union([
    // PreToolUse
    { hookEventName: "PreToolUse", permissionDecision: z.enum(["allow","deny","ask"]).optional(),
      permissionDecisionReason: z.string().optional(),
      updatedInput: z.record(z.string(), z.unknown()).optional(),
      additionalContext: z.string().optional() },
    // UserPromptSubmit
    { hookEventName: "UserPromptSubmit", additionalContext: z.string().optional() },
    // SessionStart
    { hookEventName: "SessionStart", additionalContext: z.string().optional() },
    // Setup
    { hookEventName: "Setup", additionalContext: z.string().optional() },
    // SubagentStart
    { hookEventName: "SubagentStart", additionalContext: z.string().optional() },
    // PostToolUse
    { hookEventName: "PostToolUse", additionalContext: z.string().optional(),
      updatedMCPToolOutput: z.unknown().optional() },
    // PostToolUseFailure
    { hookEventName: "PostToolUseFailure", additionalContext: z.string().optional() },
    // Notification
    { hookEventName: "Notification", additionalContext: z.string().optional() },
    // PermissionRequest
    { hookEventName: "PermissionRequest", decision: z.union([
        { behavior: z.literal("allow"), updatedInput: z.record(...).optional(),
          updatedPermissions: z.array(VL1).optional() },
        { behavior: z.literal("deny"), message: z.string().optional(),
          interrupt: z.boolean().optional() },
      ]) },
  ]).optional(),
}
```

Events NOT listed in `hookSpecificOutput` (Stop, SessionEnd, SubagentStop, PreCompact, ConfigChange, TeammateIdle, TaskCompleted, WorktreeCreate, WorktreeRemove) produce no structured `hookSpecificOutput` — only the top-level fields (`continue`, `suppressOutput`, `stopReason`, `systemMessage`) apply.

### 3.2 Configuration Locations and Merge Order

Hooks are loaded from multiple sources and merged. Sources are loaded in this priority order:

| Priority | Location | Scope | Notes |
|----------|----------|-------|-------|
| 1 (lowest) | `~/.claude/settings.json` | User-global | Applied to all sessions |
| 2 | `.claude/settings.json` | Project | Checked in to version control |
| 3 | `.claude/settings.local.json` | Project-local | Gitignored, not shared |
| 4 | Policy settings | Organization | Managed externally, highest user-facing authority |
| 5 | Plugin hooks | Per-plugin | Loaded from `~/.claude/plugins/*/hooks/hooks.json` or plugin `hooksConfig` |
| 6 | Skill hooks | Per-skill | Defined in skill frontmatter |
| 7 | Session hooks | In-memory | Temporary, added programmatically, cleared on session end |

**Merge behavior (`wOz()`, `hasworktreecreatehook-1.ts:475-503`):**

The merge function `wOz()` builds the final hook map in a strict append order. The four named sources are:

| Source | Accessor | Variable | Source File |
|--------|----------|----------|-------------|
| Policy hooks | `TL1()` | `Y` in wOz | Managed/organization policy settings |
| Plugin hooks | `zk6()` | `w` in wOz | Loaded by `rB`/`loadPluginHooks` |
| User hooks | `PP1(A, q)` | Directly appended | User settings (`~/.claude/settings.json`) |
| Workspace hooks | `PX7(A, q)` | Directly appended | Project settings (`.claude/settings.json`, `.claude/settings.local.json`) |

Merge logic in `wOz()` (lines 475-503):

1. **Policy hooks** (`TL1()`): Initialize the map. All event arrays are set from policy hook entries.
2. **Plugin hooks** (`zk6()`): Append to each event array.
   - If `FI()` (managed-only mode) is active AND a plugin hook group has a `pluginRoot` field (`"pluginRoot" in H`), it is **skipped**. Only plugin hooks without a `pluginRoot` field survive in managed-only mode.
3. **User hooks** (`PP1(A, q)`): Appended only if `!FI()`. The `A` param is the current app state, `q` is the agent ID.
4. **Workspace hooks** (`PX7(A, q)`): Appended only if `!FI()`. Workspace hooks only have `matcher` and `hooks` fields (no pluginRoot metadata).

**Result:** Hooks from all sources run in parallel for matching events. There is no override or shadowing — all matched hooks execute. The ordering within a single event (policy → plugin → user → workspace) determines dispatch order.

**`allowManagedHooksOnly` policy (`FI()`, `src/core/gg-4.ts:27-31`):** When `policySettings.allowManagedHooksOnly === true`, `FI()` returns `true` and only policy-defined and non-plugin-root plugin hooks are loaded. All user settings and workspace/project settings hooks are completely ignored.

**Plugin hook loading (`rB`/`loadPluginHooks`, `setuppluginhookhotreload-2.ts:314-347`):**

The `rB` function (aliased as `loadPluginHooks`) is a memoized async function that:
1. Calls `Hz()` to get all enabled plugins
2. For each plugin, calls `Xr9(plugin)` to extract its hook config
3. Merges all plugin hook arrays into one combined map via `zA6(q)`
4. Returns the total count of registered hooks

**`Xr9()` extraction (`setuppluginhookhotreload-2.ts:232-267`):**

`Xr9(A)` takes a plugin object and returns a map of event name → hook groups. Each hook group gets three plugin-specific fields added:
- `pluginRoot: A.path` — filesystem path to the plugin root directory
- `pluginName: A.name` — plugin display name
- `pluginId: A.source` — plugin identifier (used for analytics and dedup)

These fields are what `${CLAUDE_PLUGIN_ROOT}` variable substitution uses in command hook strings.

**Hot reload (`Mr9`/`setupPluginHookHotReload`, `setuppluginhookhotreload-2.ts:279-296`):**

`Mr9()` sets up a subscription to the settings event bus (`_O.subscribe`). When the `"policySettings"` event fires (policy settings changed), it:
1. Checks if `enabledPlugins` actually changed (compares JSON of sorted keys)
2. If changed: calls `Ik()` (clear settings cache), `CP6()` (clear plugin hook cache), `rB()` (reload all plugin hooks)
3. This is the chokidar-driven hot reload path — file system changes to settings files trigger the `_O` event bus, which triggers `Mr9()`'s subscriber.

**Deduplication:** Within each hook type, hooks are deduplicated by a unique key before execution:
- `command` hooks: deduplicated by `command` string
- `prompt` hooks: deduplicated by `prompt` string
- `agent` hooks: deduplicated by resolved prompt string
- `http` hooks: deduplicated by `url` string
- `callback` and `function` hooks: not deduplicated

**HTTP hooks and event restrictions:** HTTP hooks are filtered out for `SessionStart` and `Setup` events before execution. For all other events, HTTP hooks appear in the resolved list but are silently skipped at execution time (not yet implemented).

**Plugin hook metadata:** Plugin-sourced hook groups carry additional fields not present in user-defined hooks: `pluginRoot` (filesystem path to plugin root), `pluginId` (plugin identifier string), and optionally `pluginName`. These are used for `${CLAUDE_PLUGIN_ROOT}` variable substitution in command strings and for analytics.

### 3.3 Hook Types

Four types are user-configurable. Two additional types are internal-only.

---

#### Type: `command`

The most common type. Spawns a shell process.

```json
{
  "type": "command",
  "command": "shell-command-or-script",
  "timeout": 30,
  "statusMessage": "Running validation...",
  "once": false,
  "async": false
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | yes | — | Must be `"command"` |
| `command` | string | yes | — | Shell command to execute. Receives hook input JSON on stdin. Supports `${CLAUDE_PLUGIN_ROOT}` variable substitution for plugin hooks |
| `timeout` | number | no | ~30s (`IX` constant) | Seconds to wait before aborting. Converted to milliseconds internally. On timeout, hook outcome = `"cancelled"` |
| `statusMessage` | string | no | System default | Custom text shown in the spinner while the hook runs |
| `once` | boolean | no | `false` | If `true`, the hook is removed from the hook list after it runs once. Useful for one-time initialization hooks |
| `async` | boolean | no | `false` | If `true`, the hook process is backgrounded immediately after stdin is written. Claude does not wait for the hook to complete. See [Section 10](#10-async-and-background-hooks) |

**Shell behavior:**
- On Linux/macOS: shell is `true` (uses `/bin/sh`)
- On Windows: uses the detected shell, and `.sh` scripts are auto-prefixed with `bash`
- If `CLAUDE_CODE_SHELL_PREFIX` env var is set, the command is prefixed with it
- Working directory: the current project directory (`E1()`). If that directory no longer exists, falls back to the original CWD

**Variable substitution in plugin hooks:**
- `${CLAUDE_PLUGIN_ROOT}` in the command string is replaced with the plugin's root path (escaped for Windows if needed)

---

#### Type: `prompt`

Uses Claude itself (via a lightweight API call) to evaluate the hook input and return a structured decision.

```json
{
  "type": "prompt",
  "prompt": "Review this tool call and decide if it should be allowed: $ARGUMENTS",
  "timeout": 30,
  "model": "claude-sonnet-4-6",
  "statusMessage": "Reviewing...",
  "once": false
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | yes | — | Must be `"prompt"` |
| `prompt` | string | yes | — | The prompt text sent to Claude. `$ARGUMENTS` is replaced with the full hook input JSON (via `JSON.stringify`). |
| `timeout` | number | no | ~30s | Seconds before aborting the API call |
| `model` | string | no | Default small fast model | Model to use. See accepted values below |
| `statusMessage` | string | no | System default | Custom spinner text |
| `once` | boolean | no | `false` | Run once then remove from hook list |

**Availability:** `PreToolUse`, `PostToolUse`, `PermissionRequest` ONLY.

**Behavior:** The `$ARGUMENTS` placeholder is replaced with the full hook input JSON (stringified with `JSON.stringify(hookInput)`). Claude receives this as a user message and must respond with a single JSON object. The response is parsed through the same schema as command hook stdout output and the result is applied identically.

**Response format:** The model must return a JSON object as its response. The schema is identical to the command hook stdout JSON schema:

```json
{
  "continue": true,
  "decision": "approve",
  "reason": "The command is safe to execute",
  "systemMessage": "Optional text injected into model context",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "Verified safe",
    "updatedInput": {}
  }
}
```

If the model returns text that is not valid JSON or does not start with `{`, the output is treated as plain text (hook is treated as successful with no blocking effect).

**`$ARGUMENTS` in `agent` and `prompt` hooks:** This placeholder works in both hook types. It is replaced at execution time with the complete hook input object. For example, a prompt like `"Should I allow this tool call? $ARGUMENTS"` becomes `"Should I allow this tool call? {\"hook_event_name\": \"PreToolUse\", \"tool_name\": \"Bash\", ...}"`.

**Accepted `model` values for `prompt` and `agent` hooks:**

| Short form | Full model ID |
|------------|---------------|
| `"sonnet"` | `claude-sonnet-4-6` |
| `"opus"` | `claude-opus-4-6` |
| `"haiku"` | `claude-haiku-4-5-20251001` |

Full model IDs (e.g., `"claude-sonnet-4-6"`) are also accepted directly.

**Requires:** A `toolUseContext` to be available (i.e., running in REPL context). If not available, an error is thrown.

---

#### Type: `agent`

Spawns an autonomous Claude agent (up to 50 turns) to evaluate and potentially act on the hook input.

```json
{
  "type": "agent",
  "prompt": "Verify this code change is safe: $ARGUMENTS",
  "timeout": 60,
  "model": "claude-haiku-4",
  "statusMessage": "Agent reviewing...",
  "once": false
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | yes | — | Must be `"agent"` |
| `prompt` | string | yes | — | Prompt for the agent. `$ARGUMENTS` is replaced with the full hook input JSON (via `JSON.stringify`). Internally transformed to a function at parse time by Zod |
| `timeout` | number | no | 60s | Seconds before aborting agent execution |
| `model` | string | no | Haiku | Model for the agent. See accepted values below |
| `statusMessage` | string | no | System default | Custom spinner text |
| `once` | boolean | no | `false` | Run once then remove from hook list |

**Availability:** `PreToolUse`, `PostToolUse`, `PermissionRequest` ONLY.

**Behavior:** Unlike `prompt` hooks (single inference), agent hooks spawn a full autonomous agent with tool access (Bash, Read, Write, etc.) and up to 50 turns. This is more powerful but slower and incurs more API cost. The agent's final output is processed as the hook result.

**Response format:** The agent's final message text is parsed as the hook result using the same JSON schema as command hook stdout:

```json
{
  "continue": true,
  "decision": "approve",
  "reason": "Tests passed, change is safe",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow"
  }
}
```

If the agent's final message does not parse as valid hook JSON, the agent run is treated as successful (no blocking). This means an agent that simply completes without outputting JSON will never block a tool call — it must explicitly output a `{"decision": "block", ...}` JSON object in its final message to block.

**Requires:** Both `toolUseContext` and `messages` to be available in the current session context.

---

#### Type: `http`

POSTs the hook input JSON to an HTTP endpoint and processes the response.

```json
{
  "type": "http",
  "url": "https://api.example.com/claude-hook",
  "timeout": 30,
  "headers": {
    "Authorization": "Bearer ${API_TOKEN}",
    "Content-Type": "application/json"
  },
  "allowedEnvVars": ["API_TOKEN"],
  "statusMessage": "Calling webhook...",
  "once": false
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | yes | — | Must be `"http"` |
| `url` | string | yes | — | A valid URL to POST the hook input JSON to. Must be a well-formed URL (Zod `.url()` validated) |
| `timeout` | number | no | ~30s | Seconds before aborting the request |
| `headers` | object | no | `{}` | Additional HTTP headers to include. Values may reference environment variables using `$VAR_NAME` or `${VAR_NAME}` syntax. Only variables listed in `allowedEnvVars` are interpolated; all other `$VAR` references resolve to empty string |
| `allowedEnvVars` | string[] | no | `[]` | Explicit whitelist of environment variable names that may be interpolated into `headers` values. Required for any env var substitution to work |
| `statusMessage` | string | no | System default | Custom spinner text |
| `once` | boolean | no | `false` | Run once then remove from hook list |

**Availability:** NOT available for `SessionStart` or `Setup` events (filtered out before execution). Available in the schema for all other events.

**NOT YET SUPPORTED in v2.1.59:** HTTP hooks are fully defined in the configuration schema and can be written in `settings.json` without error. They pass validation, appear in the resolved hook list, and are deduplicated by URL. However, when the execution pipeline encounters an HTTP hook, it logs `"Skipping HTTP hook [url] — HTTP hooks are not supported for [event]"` and silently skips it. No HTTP request is made. This applies to ALL events in v2.1.59, not just SessionStart/Setup.

**Intended behavior (when implemented):** Would POST the hook input JSON as the request body with `Content-Type: application/json`. The response body would be parsed as the same JSON schema used by command hook stdout. The `method` field is not in the schema — POST is the only intended method.

---

#### Type: `function` (internal only)

Not user-configurable. These are hooks registered programmatically within the Claude Code system itself. They take the form of a JavaScript function reference rather than a configuration object, and bypass the subprocess execution path entirely. The function is called directly with the hook input and returns a result synchronously or asynchronously.

You will not see these in settings files. They exist to allow core CLI subsystems to hook into the same event pipeline as user-defined hooks without going through shell subprocess overhead.

---

#### Type: `callback` (internal only)

Not user-configurable. Plugin hooks that invoke a JavaScript callback function rather than a shell command. Identified by `{ type: "callback", internal: true }` in the internal hook data structure. These are registered by the plugin system (`zk6()`) when a plugin provides a `hooksConfig` with callback-style entries rather than command strings.

Callback hooks execute in-process and are filtered from the analytics/telemetry "internal hook" category via the `BPq()` check (`A.hook.type === "callback" && A.hook.internal === !0`) at `hasworktreecreatehook-1.ts:456-458`. They are not subject to deduplication and always run if matched.

> **All six hook types at a glance:**
>
> | Type | User-configurable | Execution path | Deduplication key | Availability |
> |------|-------------------|---------------|-------------------|--------------|
> | `command` | Yes | Shell subprocess (`CL1()`) | `command` string | All events |
> | `prompt` | Yes | Claude API call (single inference) | `prompt` string | PreToolUse, PostToolUse, PermissionRequest only |
> | `agent` | Yes | Claude agent (up to 50 turns) | resolved `prompt` string | PreToolUse, PostToolUse, PermissionRequest only |
> | `http` | Yes (schema only) | HTTP POST — NOT YET IMPLEMENTED | `url` string | All events except SessionStart, Setup |
> | `function` | No (internal) | Direct JS function call | none | Internal use only |
> | `callback` | No (internal) | In-process JS callback | none | Internal use only |

---

## 4. Hook Event Reference

This section documents all 18 hook events in detail. For each event:
- **When** it fires and **why** you'd use it
- The **matcher field** used to filter which hooks run
- The exact **input schema** written to stdin
- The **output schema** Claude reads from stdout
- **Exit code behavior** — what each exit code does
- **hookSpecificOutput** fields for that event
- **Practical examples**

---

### 4.1 PreToolUse

**When:** Before ANY tool is executed. This fires before the tool code runs — the tool has NOT yet done anything.

**Why you'd use it:**
- Block dangerous commands (e.g., `rm -rf`, writes to protected paths)
- Modify tool inputs before execution (e.g., add flags, sanitize paths)
- Auto-approve or auto-deny permission dialogs for specific tools
- Log all tool calls for audit purposes
- Inject additional context into the model about what the tool will do

**Matcher field:** `tool_name`

Examples: `"Bash"`, `"Write"`, `"Write|Edit"`, `"*"`, `"/^mcp__.*"/`

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/session/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Write",
  "tool_input": {
    "file_path": "/path/to/file.ts",
    "content": "file content here..."
  },
  "tool_use_id": "toolu_abc123"
}
```

| Field | Description |
|-------|-------------|
| `session_id` | UUID of the current session |
| `transcript_path` | Path to the JSONL transcript file for this session |
| `cwd` | Current working directory |
| `permission_mode` | Current permission mode of the session — see values below |
| `hook_event_name` | Always `"PreToolUse"` |

**`permission_mode` values:**

| Value | Description |
|-------|-------------|
| `"default"` | Standard behavior. Claude prompts for dangerous operations according to configured rules. Most interactive sessions use this mode |
| `"acceptEdits"` | Auto-accept file edit operations (Write, Edit, MultiEdit) without prompting. Bash and other tools still use normal prompting. Set via `--yes-accept-edits` CLI flag |
| `"bypassPermissions"` | Skip all permission checks entirely. Tools run without any prompting. Requires `allowDangerouslySkipPermissions` setting. Set via `--yes-bypass-permissions` CLI flag |
| `"plan"` | Planning/analysis mode. Tools are proposed but not executed. Used for dry-run previews |
| `"dontAsk"` | Don't prompt for permissions; deny if not pre-approved by rules. Differs from `bypassPermissions`: pre-approved operations proceed, unapproved ones are denied (not prompted) |

| Field | Description |
|-------|-------------|
| `tool_name` | The tool being called (e.g., `"Bash"`, `"Write"`, `"Edit"`, `"mcp__server__toolname"`) |
| `tool_input` | The full tool input object. Structure varies per tool — see per-tool breakdown below |
| `tool_use_id` | The unique ID of this tool call |

**`tool_input` structure per built-in tool:**

| Tool name | `tool_input` fields |
|-----------|--------------------|
| `Bash` | `{ command: string, timeout?: number, description?: string, run_in_background?: boolean, dangerouslyDisableSandbox?: boolean }` |
| `Read` | `{ file_path: string, offset?: number, limit?: number, pages?: string }` |
| `Write` | `{ file_path: string, content: string }` |
| `Edit` | `{ file_path: string, old_string: string, new_string: string, replace_all?: boolean }` |
| `MultiEdit` | `{ file_path: string, edits: Array<{ old_string: string, new_string: string, replace_all?: boolean }> }` |
| `Glob` | `{ pattern: string, path?: string }` |
| `Grep` | `{ pattern: string, path?: string, glob?: string, type?: string, output_mode?: string }` |
| `Task` | `{ description: string, prompt: string }` |
| `TodoRead` | `{}` (no input) |
| `TodoWrite` | `{ todos: Array<{ id: string, content: string, status: string, priority: string }> }` |
| `WebFetch` | `{ url: string, prompt?: string }` |
| `WebSearch` | `{ query: string, allowed_domains?: string[], blocked_domains?: string[] }` |
| `NotebookEdit` | `{ notebook_path: string, cell_id: string, new_source: string, cell_type?: string }` |
| `mcp__{server}__{tool}` | Varies — defined by the MCP server's tool schema. Inspect with `claude mcp list` |

**Complete list of built-in tool names** that can appear as `tool_name` values (and thus as matcher patterns):

`Bash`, `Read`, `Write`, `Edit`, `MultiEdit`, `Glob`, `Grep`, `Task`, `TodoRead`, `TodoWrite`, `WebFetch`, `WebSearch`, `NotebookEdit`

MCP tools appear as `mcp__{server_name}__{tool_name}` (e.g., `mcp__filesystem__read_file`). Use `"/^mcp__/"` as a regex matcher to catch all MCP tools.

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Success. Stdout and stderr are hidden from the conversation transcript. If stdout contains valid JSON, it is parsed for `hookSpecificOutput` decisions (permission override, input modification, context injection) |
| `2` | **BLOCKING.** The tool call is DENIED. Stderr is shown to the model as a blocking error message. The model sees the error and can respond to it |
| Any other non-zero | Non-blocking error. Stderr is shown to the USER (in the terminal) but the tool call CONTINUES normally |

**hookSpecificOutput fields:**

If stdout is valid JSON containing `hookSpecificOutput` with `hookEventName: "PreToolUse"`, the following fields are processed:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "Approved: path is in allowed directories",
    "updatedInput": {
      "file_path": "/modified/path.ts",
      "content": "modified content"
    },
    "additionalContext": "Note: this file is auto-generated, be careful with formatting"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `hookEventName` | string | Must be `"PreToolUse"` |
| `permissionDecision` | `"allow"` \| `"deny"` \| `"ask"` | Override the permission decision. `"allow"` = proceed without asking user. `"deny"` = block the call. `"ask"` = show permission dialog to user |
| `permissionDecisionReason` | string | Human-readable reason shown in the permission UI |
| `updatedInput` | object | Replace the tool input with this object before execution. The tool will receive this modified input instead of the original. Only applied when `permissionDecision` is `"allow"` or `"ask"` |
| `additionalContext` | string | Inject this text into the model as additional context, associated with this tool call |

**Permission decision priority (when multiple hooks run):**
- `"deny"` always wins
- `"ask"` wins over `"allow"`
- `"allow"` only takes effect if no other hook denies or asks

**`permission_suggestions` in the input:**

This field is passed by the permission system when a PreToolUse hook runs in a context where a permission dialog would normally appear. It is an array of opaque objects (`B.array(B.unknown())`) — the exact structure is defined by the calling component and is not normalized. In practice this field may be empty or absent; hooks should not rely on it for decision-making. Use `tool_input` content to make permission decisions instead.

**Top-level `decision` field (legacy permission shorthand):**

The hook response may include a top-level `decision` field alongside `hookSpecificOutput`. This is a legacy interface:

| Value | Effect | Source |
|-------|--------|--------|
| `"approve"` | Maps to `permissionBehavior = "allow"`. Tool proceeds without prompting | `hasworktreecreatehook-1.ts:141-143` |
| `"block"` | Maps to `permissionBehavior = "deny"`. Tool is blocked; `reason` field used as error message | `hasworktreecreatehook-1.ts:145-151` |

Note: these values are `"approve"` / `"block"` (NOT `"allow"` / `"deny"`). The `hookSpecificOutput.permissionDecision` field uses `"allow"` / `"deny"` / `"ask"`. The top-level `decision` field is processed first (lines 140-156) then overridden by `hookSpecificOutput.permissionDecision` if both are present. Prefer `hookSpecificOutput.permissionDecision` in new hooks.

**Tool execution lifecycle:**

Where PreToolUse, PostToolUse, and PostToolUseFailure fit in the full flow:

```
Model decides to use a tool
          |
          v
   PermissionRequest hooks  <-- if permission check needed
          |
          v
   Permission dialog (if not auto-resolved by hooks)
          |
          v
   PreToolUse hooks  <-- BEFORE execution
     (can block or modify input)
          |
          v
   Tool execution
     /          \
  success      failure
    |              |
    v              v
PostToolUse    PostToolUseFailure
  hooks           hooks
    |              |
    v              v
  Model sees    Model sees
  tool result   error message
```

**Key timing details:**
- `PreToolUse` fires after permission is granted but before the tool runs
- If `PreToolUse` returns `permissionDecision: "deny"`, the tool is blocked (different from the PermissionRequest hook — this is a last-minute block)
- `PostToolUse` only fires on tool success
- `PostToolUseFailure` fires on any tool error, including timeouts and user interrupts (`is_interrupt: true`)
- Multiple hooks for the same event run in PARALLEL
- All hook results are collected before proceeding

**Internal builder function:**

`bm8()` at `src/tools/hasworktreecreatehook-1.ts:1297-1307` — constructs the PreToolUse hook input:
```
bm8(tool_name, tool_use_id, tool_input, toolUseContext, sessionContext, signal, timeoutMs)
  → builds { ...EO(sessionContext), hook_event_name: "PreToolUse", tool_name, tool_input, tool_use_id }
  → calls wx() with hookInput, matchQuery=tool_name
```

**Practical examples:**

```bash
#!/bin/bash
# Block writes to /etc and /usr
input=$(cat)
tool=$(echo "$input" | jq -r '.tool_name')
file=$(echo "$input" | jq -r '.tool_input.file_path // empty')

if [[ "$tool" == "Write" || "$tool" == "Edit" ]]; then
  if [[ "$file" == /etc/* || "$file" == /usr/* ]]; then
    echo "Writes to system directories are blocked" >&2
    exit 2
  fi
fi
exit 0
```

```bash
#!/bin/bash
# Modify Bash input: add --dry-run to certain commands
input=$(cat)
cmd=$(echo "$input" | jq -r '.tool_input.command')

if echo "$cmd" | grep -q 'deploy'; then
  # Return modified input
  echo "$input" | jq '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "ask",
      permissionDecisionReason: "Deploy commands require confirmation"
    }
  }'
fi
exit 0
```

---

### 4.2 PostToolUse

**When:** After a tool executes SUCCESSFULLY. Only fires on success — if the tool errored, `PostToolUseFailure` fires instead.

**Why you'd use it:**
- Run linters or formatters after file writes
- Log tool outputs for audit
- Inject additional context into the model based on tool output
- Transform MCP tool responses before the model sees them
- Trigger external systems when specific tools run

**Matcher field:** `tool_name` (same as PreToolUse)

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "PostToolUse",
  "tool_name": "Write",
  "tool_input": {
    "file_path": "/path/to/file.ts",
    "content": "file content here..."
  },
  "tool_response": {
    "success": true,
    "filePath": "/path/to/file.ts"
  },
  "tool_use_id": "toolu_abc123"
}
```

| Field | Description |
|-------|-------------|
| `tool_response` | The tool's return value. Structure varies per tool — see per-tool breakdown below |

(All other fields same as PreToolUse.)

**`tool_response` structure per built-in tool:**

| Tool name | `tool_response` fields |
|-----------|----------------------|
| `Bash` | `{ stdout: string, stderr: string, interrupted: boolean, rawOutputPath?: string, isImage?: boolean, backgroundTaskId?: string, backgroundedByUser?: boolean, dangerouslyDisableSandbox?: boolean, returnCodeInterpretation?: string, noOutputExpected?: boolean, structuredContent?: any[], persistedOutputPath?: string, persistedOutputSize?: number }` — full schema from `definitions-1.ts:2731-2760` |
| `Read` | Discriminated union: `{ type: "text", file: { filePath: string, content: string, numLines: number } }` or `{ type: "image", file: { filePath: string, ... } }` |
| `Write` | The file content string (raw text, not an object) |
| `Edit` | The updated file content string (same format as Write) |
| `MultiEdit` | The updated file content string |
| `Glob` | Array of matching file path strings |
| `Grep` | Array of match result strings |
| `Task` | The subagent's output text |
| `TodoRead` | Array of todo objects |
| `TodoWrite` | String confirmation message |
| `WebFetch` | Fetched content as string |
| `WebSearch` | Search results as structured text |
| `mcp__{server}__{tool}` | Defined by the MCP tool's output schema. May be string, object, or array |

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Success. Stdout is captured and shown in transcript mode (Ctrl+O view). If stdout is valid JSON, it is parsed for `hookSpecificOutput` |
| `2` | Show stderr to the model immediately as a message. The model can react to this message |
| Any other non-zero | Non-blocking. Stderr is shown to the user only |

**hookSpecificOutput fields:**

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "ESLint found 3 warnings in this file",
    "updatedMCPToolOutput": "replacement content for MCP tool response"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `hookEventName` | string | Must be `"PostToolUse"` |
| `additionalContext` | string | Text injected into the model's context after the tool result. The model sees this as additional information about the tool execution. Visible via Ctrl+O (transcript view) alongside the tool result |
| `updatedMCPToolOutput` | string | For MCP tool calls only: replaces the tool's response content with this string. The model sees this instead of the original MCP output |

**`additionalContext` behavior:**
- The string is injected into the model context alongside the tool result (not replacing it)
- The model sees it as extra information about what the tool did
- Multiple hooks can each contribute `additionalContext`; they are collected and passed as an array (`additionalContexts`)
- Visible in the conversation transcript (Ctrl+O) alongside the tool output
- Use this for annotations: lint warnings, security notices, test status, etc.

**`updatedMCPToolOutput` behavior:**
- Only works for MCP tool calls (tool names matching `mcp__*` pattern)
- Has no effect on built-in tools (Bash, Read, Write, etc.)
- Completely replaces what the model sees as the tool's response
- No schema validation is applied — the hook is responsible for producing a string the model can interpret correctly
- Processed at source line 1056-1058 in `src/tools/hasworktreecreatehook-1.ts`

**`suppressOutput` field (applies to all hook events):**

The hook response JSON may include a top-level `suppressOutput: true` field. When present and `true`, the hook's stdout is NOT shown in the transcript display (Ctrl+O view) even on exit code 0. This applies to ALL hook events, not just PostToolUse. Source: `hasworktreecreatehook-1.ts:885` — `if (kL1(A6) && !A6.suppressOutput && $6 && l.status === 0)` controls whether to yield the success progress message.

**Internal builder function:**

`xm8()` at `src/tools/hasworktreecreatehook-1.ts:1312-1326` — constructs the PostToolUse hook input:
```
xm8(tool_name, tool_use_id, tool_input, tool_response, toolUseContext, sessionContext, signal, timeoutMs)
  → builds { ...EO(sessionContext), hook_event_name: "PostToolUse", tool_name, tool_input, tool_response, tool_use_id }
  → calls wx() with hookInput, matchQuery=tool_name
```
Note: `tool_response` is the ONLY field present in PostToolUse but absent from PreToolUse. It contains the raw tool output object, not a summary.

**Practical examples:**

```bash
#!/bin/bash
# Run ESLint after TypeScript file writes and report to model
input=$(cat)
tool=$(echo "$input" | jq -r '.tool_name')
file=$(echo "$input" | jq -r '.tool_input.file_path // empty')

if [[ "$tool" == "Write" && "$file" == *.ts ]]; then
  result=$(npx eslint --format=compact "$file" 2>&1)
  if [[ -n "$result" ]]; then
    # Exit 2 to show ESLint output to the model
    echo "ESLint output for $file: $result" >&2
    exit 2
  fi
fi
exit 0
```

```bash
#!/bin/bash
# Inject context about test results after Bash runs
input=$(cat)
tool=$(echo "$input" | jq -r '.tool_name')
cmd=$(echo "$input" | jq -r '.tool_input.command // empty')

if [[ "$tool" == "Bash" && "$cmd" == *"npm test"* ]]; then
  echo '{"hookSpecificOutput": {"hookEventName": "PostToolUse", "additionalContext": "Tests have completed. Review any failures and fix them before continuing."}}'
fi
exit 0
```

---

### 4.3 PostToolUseFailure

**When:** After a tool execution FAILS — meaning the tool threw an error, timed out, or was interrupted.

**Why you'd use it:**
- Inject recovery context into the model (e.g., "the test failed because of X, try Y")
- Log failures to external systems
- Provide additional debugging information when specific tools fail

**Matcher field:** `tool_name`

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "PostToolUseFailure",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  },
  "tool_use_id": "toolu_abc123",
  "error": "Command failed with exit code 1: ENOENT no such file",
  "is_interrupt": false
}
```

| Field | Type | Description |
|-------|------|-------------|
| `error` | string | The error message from the failed tool. A plain string in most cases (e.g., `"Command failed with exit code 1"`). For tools that produce structured errors (MCP tools), may contain a JSON-encoded error with `message`, `code`, and optionally `stderr` and `stdout` fields |
| `is_interrupt` | boolean | `true` if the tool was interrupted (Ctrl+C / SIGINT / SIGTERM / timeout exceeded). `false` if the tool failed on its own (non-zero exit code, thrown exception, file not found, etc.) |

**`error` values in practice:**
- Most failures: a plain error string, e.g., `"ENOENT: no such file or directory, open '/missing/file'"`
- Bash non-zero exit: `"Command failed with exit code N: <stderr excerpt>"`
- Timeout: the `is_interrupt` flag is set to `true` and `error` contains the timeout message
- Ctrl+C: the `is_interrupt` flag is set to `true` and `error` contains `"interrupted"` or similar

**`is_interrupt` vs natural failure:**
- `is_interrupt: true` — caused by Ctrl+C, SIGINT, SIGTERM, or tool timeout. The user or system actively stopped the tool
- `is_interrupt: false` — tool ran to completion but exited with an error (non-zero exit code, exception thrown, permission denied, file not found)
- These are mutually exclusive. A tool either finishes with an error, or is stopped externally
- There is no separate `is_timeout` field: timeouts set `is_interrupt: true`

**Exit code behavior:** Same as PostToolUse.

**hookSpecificOutput fields:**

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUseFailure",
    "additionalContext": "This command requires the dev server to be running first"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `additionalContext` | string | Context injected into the model after the failure |

**Internal builder function:**

`um8()` at `src/tools/hasworktreecreatehook-1.ts:1328-1345` — constructs the PostToolUseFailure hook input:
```
um8(tool_name, tool_use_id, tool_input, error, is_interrupt, toolUseContext, sessionContext, signal, timeoutMs)
  → builds { ...EO(sessionContext), hook_event_name: "PostToolUseFailure", tool_name, tool_input, tool_use_id, error, is_interrupt }
  → calls wx() with hookInput, matchQuery=tool_name
```
Note: there is no `is_timeout` field. Timeouts set `is_interrupt: true` in the `error` string.

**Practical example:**

```bash
#!/bin/bash
# Provide helpful context when npm test fails
input=$(cat)
tool=$(echo "$input" | jq -r '.tool_name')
cmd=$(echo "$input" | jq -r '.tool_input.command // empty')
error=$(echo "$input" | jq -r '.error // empty')

if [[ "$tool" == "Bash" && "$cmd" == *"npm test"* ]]; then
  if echo "$error" | grep -q 'Cannot find module'; then
    context="Tests failed with 'Cannot find module'. Try running 'npm install' first."
    echo "{\"hookSpecificOutput\": {\"hookEventName\": \"PostToolUseFailure\", \"additionalContext\": \"$context\"}}"
  fi
fi
exit 0
```

---

### 4.4 PermissionRequest

**When:** When a permission dialog is about to be shown to the user. This fires just before Claude Code would display the "Allow/Deny" prompt.

**Why you'd use it:**
- Auto-approve specific tools or commands without user interaction
- Auto-deny dangerous operations
- Implement policy-based permission automation
- Modify the tool input before it runs (e.g., sanitize paths even while allowing)

**Matcher field:** `tool_name`

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "PermissionRequest",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf /tmp/test-artifacts"
  },
  "permission_suggestions": [
    { "type": "allow_once" },
    { "type": "allow_session" },
    { "type": "deny" }
  ],
  "tool_use_id": "toolu_abc123"
}
```

| Field | Description |
|-------|-------------|
| `permission_suggestions` | Array of permission suggestion objects the UI would have shown the user. The structure of each element is opaque (`B.array(B.unknown())`) and varies by context. Common fields in each suggestion object: `{ type: string }` where type is one of `"allow_once"`, `"allow_session"`, or `"deny"`. Hooks should not rely on this array for decision logic — inspect `tool_input` instead |

**`permission_suggestions` context:**

This array represents the permission options Claude Code would normally display to the user in an interactive permission dialog. It is provided to PermissionRequest hooks as informational context so hooks can understand what the user would have been asked. The array is schema-typed as `B.array(B.unknown())` in source — meaning it passes through without validation. Each element typically has a `type` string (`"allow_once"`, `"allow_session"`, `"deny"`) and may have additional fields depending on the tool and context.

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | If stdout contains a valid `decision` in `hookSpecificOutput`, that decision is used. Otherwise, show normal permission dialog |
| Any other non-zero | Non-blocking. Stderr shown to user, normal permission dialog proceeds |

**hookSpecificOutput fields:**

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow",
      "updatedInput": {
        "command": "rm -rf /tmp/test-artifacts"
      }
    }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `hookEventName` | string | Must be `"PermissionRequest"` |
| `decision` | object | The permission decision object |
| `decision.behavior` | `"allow"` \| `"deny"` \| `"ask"` \| `"passthrough"` | The permission action to take |
| `decision.updatedInput` | object | Optional. If `behavior` is `"allow"`, the tool runs with this input instead of the original. Has no effect on `"deny"` or `"ask"` |
| `decision.message` | string | Optional. Message shown to the user or model when `behavior` is `"deny"`. Explains why the operation was blocked |
| `decision.interrupt` | boolean | Optional. If `true` with `"deny"`, marks the denial as a user-initiated interrupt (affects how the model handles the refusal) |

**Permission behavior values — full explanations:**

| Value | Effect |
|-------|--------|
| `"allow"` | Auto-approve the tool call. The tool runs immediately without prompting. If `updatedInput` is also provided, the tool receives that modified input. If `updatedPermissions` (an array of permission rule objects) is provided, those rules are applied to the session's permission set |
| `"deny"` | Block the tool call. The tool does not run. If `message` is provided, it is surfaced to the user/model as the reason. If `interrupt: true`, the denial is treated as a user-initiated interrupt rather than a policy block |
| `"ask"` | Fall through to the normal user permission dialog. The user sees the permission prompt and decides manually. The hook has not made a decision |
| `"passthrough"` | Do nothing. Equivalent to the hook returning exit 0 with no JSON. Source: `src/tools/hasworktreecreatehook-1.ts` lines 1075-1076 — the `case "passthrough": break` branch does not set `f` (the aggregated decision), so the hook has no effect on the permission outcome |

**Permission system sequence:**

```
Tool call requires permission check
              |
              v
  PermissionRequest hooks run
  (can return decision: allow/deny + updatedInput)
              |
      Hooks returned allow? ---> proceed with tool, skip dialog
              |
      Hooks returned deny? ---> block tool, show reason to model
              |
      No decision? ---> show permission dialog to user
              |
              v
  PreToolUse hooks run
  (can also return permissionDecision: allow/deny/ask)
```

**Decision accumulation when multiple PermissionRequest hooks disagree:**

```typescript
case "deny":  f = "deny";       break;  // deny always wins
case "ask":   if (f !== "deny") f = "ask"; break;  // ask beats allow
case "allow": if (!f) f = "allow"; break;  // allow only if no other decision
```

**Priority when multiple hooks disagree:** `deny > ask > allow > passthrough`
- If any hook returns `deny`, the request is denied regardless of other hooks
- If no deny but any hook returns `ask`, the user is shown the dialog
- `allow` only wins if no other hook returned `deny` or `ask`
- `passthrough` never overrides anything

**`updatedPermissions` field:**

The `decision` object may also include an `updatedPermissions` array alongside `behavior: "allow"`. This array contains permission rule objects that should be added to the session's permission ruleset as a result of allowing this operation. The schema is defined as `B.array(B.unknown())` — the structure depends on the permission system and is not formally schema-validated by the hook system. In practice, permission rules follow the same format as entries in `settings.json` `permissions` blocks.

**Internal builder function:**

`z26()` at `src/tools/hasworktreecreatehook-1.ts:1535-1545` — constructs the PermissionRequest hook input:
```
z26(tool_name, tool_use_id, tool_input, sessionContext, permission_suggestions, signal, timeoutMs)
  → builds { ...EO(sessionContext), hook_event_name: "PermissionRequest", tool_name, tool_input, permission_suggestions }
  → calls wx() with hookInput, matchQuery=tool_name
```
Note: `tool_use_id` is NOT included in the PermissionRequest hookInput (unlike PreToolUse/PostToolUse). The `permission_suggestions` value is passed through as-is from the calling component.

**Practical example:**

```bash
#!/bin/bash
# Auto-allow read-only operations, auto-deny destructive ones
input=$(cat)
cmd=$(echo "$input" | jq -r '.tool_input.command // empty')

# Auto-allow safe commands
if echo "$cmd" | grep -qE '^(ls|cat|echo|pwd|git (status|log|diff))'; then
  echo '{"hookSpecificOutput": {"hookEventName": "PermissionRequest", "decision": {"behavior": "allow"}}}'
  exit 0
fi

# Auto-deny destructive patterns
if echo "$cmd" | grep -qE '(rm -rf|DROP TABLE|DELETE FROM.*WHERE 1)'; then
  echo '{"hookSpecificOutput": {"hookEventName": "PermissionRequest", "decision": {"behavior": "deny"}}}'
  exit 0
fi

# Let the user decide everything else
exit 0
```

---

### 4.5 Notification

**When:** When the system emits a notification — idle prompts, permission prompts, auth events, elicitation dialogs.

**Why you'd use it:**
- Forward Claude's idle notification to a desktop notification system
- Send Slack/Teams messages when Claude needs input
- Suppress certain notification types
- Log notification events

**Matcher field:** `notification_type`

**Known `notification_type` values:**

| Value | When it fires |
|-------|---------------|
| `"idle_prompt"` | Claude has been waiting for user input for `messageIdleNotifThresholdMs` (default: 60 seconds) |
| `"permission_prompt"` | A permission dialog is being shown to the user |
| `"auth_success"` | Authentication completed successfully |
| `"elicitation_dialog"` | A user input dialog (elicitation) is being shown |
| `"elicitation_url_dialog"` | An MCP elicitation dialog with a URL input (dispatches notifications but is NOT a matchable value — see note below) |
| `"worker_permission_prompt"` | A background agent needs permission for a tool use (UNLISTED — cannot be matched by hooks; see note below) |

**Note on unlisted notification types:** `worker_permission_prompt` is dispatched when a background agent requests permission (message: `"${agent_id} needs permission for ${tool_name}"`), but it is not included in the hook matcher values array. Hooks with `notification_type: "worker_permission_prompt"` matchers will never fire. `elicitation_url_dialog` is similarly dispatched via `ac()` but not present in the matcher values.

**Per-type details:**

**`idle_prompt`**
- Timer source: `src/tools/getcompletedvisualtext-1.ts:1395-1410`
- Fires when the user has been idle for longer than `messageIdleNotifThresholdMs` (default: 60000ms, configurable via `v1().messageIdleNotifThresholdMs`)
- Conditions that suppress the notification (will NOT fire): model is currently loading (`iq` flag), a local JSX command is active (`DK` flag), a modal dialog is currently shown (`SY.current !== void 0`)
- Message: `"Claude is waiting for your input"`
- Title: null

**`idle_prompt` full chain:**

1. **UI timer setup** (`getcompletedvisualtext-1.ts`, lines 1387-1409) — A `useEffect` hook runs whenever `iq` (isQuerying/loading), `DK` (disable flag), `f_` (queue length), or `J$` (last input timestamp) changes. It sets a `setTimeout` for `messageIdleNotifThresholdMs` milliseconds (default: 60000ms).

2. **Timer conditions** — all of these must be true when the timer fires, or the notification is suppressed:
   - `iq === false` — Claude is NOT currently processing/loading a response
   - `DK === false/undefined` — The disable flag is NOT set
   - `SY.current === undefined` — No modal or overlay is currently active
   - `Date.now() - J$ >= messageIdleNotifThresholdMs` — Enough idle time has elapsed

3. **Timer fires** — calls `ac({ message: "Claude is waiting for your input", notificationType: "idle_prompt" }, e)`

4. **Notification dispatch** — `executeNotificationHooks()` (`FE8()`) is called with `{ hook_event_name: "Notification", message: "Claude is waiting for your input", notification_type: "idle_prompt" }`

5. **Hook execution** — Any `Notification` hooks matching `"idle_prompt"` are run via `mN6()`

6. **System notification** — After hooks run, the notification routes to the user's preferred notification channel (`preferredNotifChannel` setting: `iterm2`, `kitty`, `terminal_bell`, `auto`, `notifications_disabled`)

**Why hooks for idle_prompt?** System notification delivery (iterm2, bell, etc.) is separate from hook execution. Hooks run FIRST, then the system notification fires. This means your hook can POST to a webhook, send Slack messages, or run arbitrary scripts — regardless of the terminal's notification support.

**`permission_prompt`**
- Source: `Ar6()` at `src/ui/components-2.ts:2134-2149`
- Fires when a permission dialog is being shown to the user, via polling with `setInterval` checking `VZz(Lkq)` readiness
- Message: dynamic context based on which tool is requesting permission
- Title: null

**`auth_success`**
- Source: `src/vendor/dom.ts:4045-4070`
- Fires after a successful OAuth token exchange (login completed)
- Message: `"Claude Code login successful"`
- Title: null
- Cleanup: after dispatching the notification, invokes `Hl7()` for session cleanup before continuing

**`elicitation_dialog`**
- Source: `src/mcp/mcp-1.ts:4345-4350`
- Fires when an MCP server requests structured user input via the elicitation protocol
- Message: `"Claude Code needs your input"`
- Title: null
- Hooks CAN match this type

**`elicitation_url_dialog`**
- Source: `src/mcp/mcp-1.ts:5159`
- A URL-based variant of elicitation. Dispatched via `ac()` but NOT in the hook matcher values
- Hooks with `notification_type: "elicitation_url_dialog"` will never match

**`worker_permission_prompt`**
- Source: `src/tools/tool-1.ts:18645-18670`
- Fires when a background agent (subagent/worker) needs permission to run a tool
- Message: `"${agent_id} needs permission for ${tool_name}"`
- NOT in matcher values. Hooks cannot intercept this type

**Notification dispatch chain (`ac()` function, `src/vendor/parse5.ts:22497-22502`):**

All matchable notification types flow through the same unified dispatch function. The hook execution step happens BEFORE the system notification channel fires:
```
ac(notificationInput, extra)              [parse5.ts:22497-22502]
    |
    v
Step 1: FE8(notificationInput)            [hasworktreecreatehook-1.ts:1348-1360]
        async function FE8(A, q = IX):
          builds hookInput = { ...EO(void 0), hook_event_name: "Notification",
                               message: A.message, title: A.title, notification_type: A.notificationType }
          calls mN6({ hookInput, timeoutMs: q, matchQuery: A.notificationType })
        Execute all matched Notification hooks via mN6() path
        (uses executeHooksOutsideREPL, not the main wx() pipeline)
        (only "command" hook type supported; prompt/agent hooks are NOT)
    |
    v
Step 2: AXY(preferredNotifChannel, notificationInput, extra)  [parse5.ts:22507-22545]
        Routes notification to terminal channel based on preferredNotifChannel setting
        Supports: auto, iterm2, iterm2_with_bell, kitty, ghostty, terminal_bell, notifications_disabled
    |
    v
Step 3: tengu_notification_method_used (telemetry event)
```

**Key implication:** Your Notification hook runs BEFORE the terminal bell or iTerm2 notification fires. This means if you suppress the exit code or handle the notification yourself (e.g., POST to Slack), the system notification still fires afterward regardless.

**`preferredNotifChannel` values** (configured in `settings.json` as `preferredNotifChannel`):

| Channel value | Behavior |
|--------------|----------|
| `"auto"` | Auto-detect based on `$TERM_PROGRAM`: `iTerm.app` → `iterm2`, `kitty` → `kitty`, `ghostty` → `ghostty`, `Apple_Terminal` → `terminal_bell`. Unrecognized terminals get no notification |
| `"iterm2"` | iTerm2 proprietary notification API. Only works in iTerm2 |
| `"iterm2_with_bell"` | iTerm2 notification API plus ASCII bell character (`\a`) |
| `"kitty"` | Kitty terminal notification protocol |
| `"ghostty"` | Ghostty terminal notification API |
| `"terminal_bell"` | ASCII bell character (`\a`) only. Works in any terminal |
| `"notifications_disabled"` | No system notification is sent at all |

Default if not configured: `"auto"`.

Hooks run BEFORE the system notification channel fires. This means your hook script can POST to a webhook, send Slack messages, or do anything else regardless of terminal notification support. The system notification channel fires whether or not your hook exits with 0.

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "Notification",
  "message": "Claude is waiting for your input",
  "title": null,
  "notification_type": "idle_prompt"
}
```

| Field | Description |
|-------|-------------|
| `message` | The notification message text |
| `title` | Optional title (may be null) |
| `notification_type` | The type of notification (see table above) |

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Success. Stdout and stderr are hidden |
| Any other non-zero | Stderr is shown to the user |

**hookSpecificOutput:** No hook-specific output fields for Notification. Return value is ignored beyond exit code.

**Important:** The Notification hook does NOT use the main `wx()` pipeline — it uses `mN6()` (the simpler `executeHooksOutsideREPL` path). This means prompt/agent hook types are NOT supported for Notification events — only `command` hooks work.

**Practical examples:**

```bash
#!/bin/bash
# Send macOS notification when Claude is waiting
input=$(cat)
type=$(echo "$input" | jq -r '.notification_type')
message=$(echo "$input" | jq -r '.message')

if [[ "$type" == "idle_prompt" ]]; then
  osascript -e "display notification \"$message\" with title \"Claude Code\""
fi
exit 0
```

```bash
#!/bin/bash
# Send terminal bell on any notification
echo -e "\a"
exit 0
```

```bash
#!/bin/bash
# Post to Slack webhook when Claude is idle
input=$(cat)
type=$(echo "$input" | jq -r '.notification_type')

if [[ "$type" == "idle_prompt" ]]; then
  curl -s -X POST "$SLACK_WEBHOOK_URL" \
    -H 'Content-type: application/json' \
    --data '{"text":"Claude Code is waiting for your input"}' \
    &>/dev/null
fi
exit 0
```

---

### 4.6 UserPromptSubmit

**When:** When the user submits a message/prompt, before Claude processes it.

**Why you'd use it:**
- Validate user input (e.g., block certain patterns, require specific formats)
- Automatically block prompts that violate policy
- Inject additional context alongside the user's message
- Pre-process or augment the user's intent

**Matcher field:** NONE — fires for all user prompt submissions unconditionally.

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "UserPromptSubmit",
  "prompt": "Please delete all test files in /tmp"
}
```

| Field | Description |
|-------|-------------|
| `prompt` | The raw, unprocessed text of the user's message exactly as typed. This is the pre-processing value — it has not been through any template expansion, command processing, or modification. It is the literal string the user submitted |

**`prompt` field clarification:**
- This is the raw user text, not any processed or templated version
- Slash commands (e.g., `/clear`, `/compact`) have NOT been evaluated yet
- File references have NOT been expanded
- Any transformations the system applies happen AFTER UserPromptSubmit hooks run
- What you read from `prompt` is exactly what the user typed and submitted

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Success. If stdout is valid JSON with `hookSpecificOutput.additionalContext`, that string is injected into the model's context alongside the user's message. Plain stdout text (non-JSON) also works as context injection |
| `2` | **BLOCKING — UNIQUE BEHAVIOR.** The prompt submission is CANCELLED entirely. The prompt text is ERASED from the input. Stderr is shown to the user as an error. Claude NEVER sees the prompt. This is the only hook event where exit 2 causes message erasure |
| Any other non-zero | Non-blocking. Stderr is shown to the user in the terminal, but the prompt PROCEEDS normally to Claude |

**Context injection timing (exit 0 with context):**

When a UserPromptSubmit hook returns `additionalContext` via JSON or plain stdout:
- The context is injected as additional content alongside the user's message
- The model receives both the user's original prompt AND the injected context
- The context does NOT replace the user's message — it supplements it
- The context appears in the model's view before it generates a response
- Multiple hooks can each contribute context; all contexts are collected together

**Exit 2 message erasure (unique to UserPromptSubmit):**

This event is the ONLY hook event where exit 2 causes the underlying message to be erased rather than simply showing an error. In all other events (PreToolUse, PostToolUse, etc.), exit 2 shows the error and allows the tool/operation to be seen by the model. Here, exit 2 means:
1. The user's typed message is discarded entirely
2. The input box is cleared (or the prompt entry is cancelled)
3. Stderr content is displayed to the user as the reason for rejection
4. Claude never sees the original prompt

This makes UserPromptSubmit the hook for enforcing prompt-level policies (content filtering, access control, required formats).

**hookSpecificOutput fields:**

```json
{
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "User is working in a production environment. Exercise extra caution."
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `hookEventName` | string | Must be `"UserPromptSubmit"` |
| `additionalContext` | string | **Required** if you want to inject context via the JSON path. This string is appended to the model's context alongside the user's prompt. Plain stdout text (non-JSON) also works but the `additionalContext` JSON field is the canonical and preferred form |

**`additionalContext` for UserPromptSubmit — two ways to provide it:**

1. **JSON form (preferred):**
   ```json
   {"hookSpecificOutput": {"hookEventName": "UserPromptSubmit", "additionalContext": "User context here"}}
   ```
2. **Plain stdout form (also works):**
   Simply print text to stdout without JSON wrapping. The text is injected as context

Both forms result in the same outcome: the text is available to the model as context before it generates a response. The model sees both the user's original message AND the hook-injected context. The injected context does not replace or modify the user's message — it is additive.

**Internal builder function:**

`Jd8()` at `src/tools/hasworktreecreatehook-1.ts:1422-1430` — constructs the UserPromptSubmit hook input:
```
Jd8(prompt, sessionContext, toolUseContext)
  → builds { ...EO(sessionContext), hook_event_name: "UserPromptSubmit", prompt: prompt }
  → calls wx({ hookInput, toolUseID: RE(), signal: toolUseContext.abortController.signal,
               timeoutMs: IX, toolUseContext })
```
Note: `prompt` is the raw user text string. `RE()` generates a fresh tool use ID for this hook invocation.

**Practical examples:**

```bash
#!/bin/bash
# Block prompts containing "production" unless in approved sessions
input=$(cat)
prompt=$(echo "$input" | jq -r '.prompt')

if echo "$prompt" | grep -qi 'production' && [[ -z "$APPROVED_PROD_SESSION" ]]; then
  echo "Production-related prompts require an approved session. Set APPROVED_PROD_SESSION=1 to proceed." >&2
  exit 2
fi
exit 0
```

```bash
#!/bin/bash
# Inject project context with every prompt
input=$(cat)
cwd=$(echo "$input" | jq -r '.cwd')

if [[ -f "$cwd/.claude-context.md" ]]; then
  context=$(cat "$cwd/.claude-context.md")
  # Use JSON output for additionalContext
  jq -n --arg ctx "$context" '{hookSpecificOutput: {hookEventName: "UserPromptSubmit", additionalContext: $ctx}}'
fi
exit 0
```

---

### 4.7 SessionStart

**Source:** `_v8()` async generator at `src/tools/hasworktreecreatehook-1.ts:1431`. Builds `{ hook_event_name: "SessionStart", source: A, agent_type: K, model: Y }` then calls `wx()` with `matchQuery: A` (the source value).

**When:** At the beginning of a session — when Claude Code starts, when a session resumes, when the user clears the conversation, or when context compaction triggers a session restart.

**Why you'd use it:**
- Inject project-specific context or instructions at session start
- Run environment initialization scripts
- Load dynamic configuration (e.g., from a database or API)
- Log session start events

**Matcher field:** `source`

| Source value | When |
|-------------|------|
| `"startup"` | Fresh CLI launch |
| `"resume"` | Resuming a previous session |
| `"clear"` | User ran /clear |
| `"compact"` | Session restarted after context compaction |

**Source value details:**

- **`"startup"`** (`src/tools/runheadless-1.ts:1652,1695` and `src/agents/startdeferredprefetches-1.ts:1279,1365`): Fresh CLI launch when `!q.resume && !q.resume_session_at`. Passes `agentType` and `model` in hook input.

- **`"resume"`** (`src/tools/getcompletedvisualtext-1.ts:469`): Loads prior conversation from JSONL via `c66(sessionId)`, restores state via `gn6()`, applies file history via `yF()`. Falls back to `"startup"` behavior if no JSONL found for the given session ID.

- **`"clear"`** (`src/vendor/dom.ts:53509`): User cleared the conversation. Session ID is preserved across the clear. The sequence is: fire `SessionEnd("clear")` first, clear `fileHistory` and MCP state, then fire `SessionStart("clear")` for the new empty session.

- **`"compact"`** (`src/vendor/highlight.js.ts:15214,15337,16156`): After compaction summary is generated via `uJ7()` and a boundary marker is written via `Cp6()`. This is a session restart with the conversation replaced by its summary.

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "SessionStart",
  "source": "startup",
  "agent_type": "cli",
  "model": "claude-sonnet-4-6"
}
```

| Field | Description |
|-------|-------------|
| `source` | Why the session is starting: `startup`, `resume`, `clear`, `compact` |
| `agent_type` | The type of agent: `"cli"` for main sessions, or agent type for subagents |
| `model` | The model being used in this session |

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Success. Stdout is shown to Claude as initial context. If stdout is JSON with `hookSpecificOutput.additionalContext`, that string is used as context |
| `2` | **IGNORED.** Blocking errors do not prevent session start. The session starts regardless |
| Any other non-zero | Stderr shown to user |

**hookSpecificOutput fields:**

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Project: my-app\nDatabase: PostgreSQL 15\nTest framework: Vitest"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `additionalContext` | string | Context injected into the model at session start. Plain stdout text also works |

**Environment variables available:**
- `CLAUDE_ENV_FILE`: Path to the env file for this session. Set **only for SessionStart and Setup** hooks at `src/tools/hasworktreecreatehook-1.ts:294`: `W.CLAUDE_ENV_FILE = await R$4(q, _)`. The `R$4()` function is defined at `src/vendor/react-ink.ts:12170` and resolves to `mK8(await y$4(), "${event_lowercase}-hook-${session_id}.sh")` — a path inside the session environment directory. Hook scripts can read env-specific configuration from this path. Not set for any other hook event type.

**HTTP hooks not supported:** SessionStart and Setup hooks filter out HTTP hook types at `src/tools/hasworktreecreatehook-1.ts:578-600`: `if (G.hook.type === "http") return false` with a log message `"Skipping HTTP hook — HTTP hooks are not supported for SessionStart/Setup"`. Only `command`, `prompt`, `agent`, `callback`, and `function` hook types fire for SessionStart.

**Session lifecycle flow:**

```
claude starts
      |
      v
SessionStart (source: "startup")
      |
      v
 [interactive session]
      |
   /clear
      |
      v
SessionEnd (reason: "clear")
      |
      v
SessionStart (source: "clear")  <-- new session
      |
   /compact
      |
      v
 [compaction runs]
      |
      v
SessionStart (source: "compact")  <-- restart after compaction
      |
   exit
      |
      v
SessionEnd (reason: "logout")

claude --resume [session-id]
      |
      v
SessionStart (source: "resume")
```

**Important:** SessionEnd fires on clear, not just on process exit. When the user clears the conversation, a `SessionEnd("clear")` fires, followed immediately by `SessionStart("clear")` for the new empty session.

**Practical examples:**

```bash
#!/bin/bash
# Inject project documentation at startup
cwd=$(cat /dev/stdin | jq -r '.cwd')

if [[ -f "$cwd/CLAUDE.md" ]]; then
  cat "$cwd/CLAUDE.md"
elif [[ -f "$cwd/.claude/context.md" ]]; then
  cat "$cwd/.claude/context.md"
fi
exit 0
```

```bash
#!/bin/bash
# Different context for new vs resumed sessions
input=$(cat)
source=$(echo "$input" | jq -r '.source')

if [[ "$source" == "startup" ]]; then
  echo "New session started. Remember to check the PR description before making changes."
elif [[ "$source" == "resume" ]]; then
  echo "Session resumed. The previous session's work is available in the transcript."
fi
exit 0
```

---

### 4.8 SessionEnd

**Source:** `yB8()` async function at `src/tools/hasworktreecreatehook-1.ts:1512`. Builds `{ hook_event_name: "SessionEnd", reason: A }` then calls `mN6()` (the outside-REPL path) with `matchQuery: A` (the reason value). After hooks complete, calls `HG6(Y, H)` to update app state.

**When:** When a session terminates — regardless of reason.

**Why you'd use it:**
- Log session completion events
- Save session summaries to external systems
- Cleanup temporary files created during the session
- Send notifications that a session has ended

**Matcher field:** `reason`

| Reason value | When |
|-------------|------|
| `"clear"` | User ran /clear |
| `"logout"` | User logged out |
| `"prompt_input_exit"` | User exited from prompt input |
| `"other"` | Any other reason |

**Reason value details:**

- **`"clear"`**: From the UI clear conversation callback. Fires `SessionEnd("clear")` before `SessionStart("clear")` — the session ID is preserved.

- **`"logout"`** (`src/vendor/dom.ts:990`, `MbY()` component): Full logout flow. Clears onboarding state, caches, and auth tokens. 200ms delay before process exit to allow hooks to complete.

- **`"prompt_input_exit"`**: User pressed Ctrl+D at the interactive prompt. Before the session ends, the terminal displays a random goodbye message selected from: `["Goodbye!", "See ya!", "Bye!", "Catch you later!"]` via `hYz()`.

- **`"other"`**: Default catch-all in `eq(A=0, q="other", K)` at `src/conversation/session-1.ts:7030`. Triggered on crashes, unhandled signals, and unclean shutdowns. Use this value in matchers to catch abnormal exits.

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "SessionEnd",
  "reason": "logout"
}
```

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Success |
| Any other | Non-zero exit: `process.stderr.write("SessionEnd hook [command] failed: output\n")` — written directly to process stderr. Cannot block session end |

**Note:** SessionEnd hooks run via `mN6()` (the outside-REPL path). Session termination cannot be blocked — the session ends regardless of hook result. Failures are logged to process stderr but don't throw exceptions.

**Practical example:**

```bash
#!/bin/bash
# Log session end with duration
input=$(cat)
session=$(echo "$input" | jq -r '.session_id')
reason=$(echo "$input" | jq -r '.reason')

echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) Session $session ended: $reason" >> ~/.claude/session-log.txt
exit 0
```

---

### 4.9 Stop

**Source:** `dm8()` async generator at `src/tools/hasworktreecreatehook-1.ts:1359`. This function serves dual purpose: when called without an `agent_id` parameter it builds `{ hook_event_name: "Stop", stop_hook_active: Y, last_assistant_message: O }` and calls `wx()`; when called with `agent_id` it builds a SubagentStop event instead (see 4.11).

**When:** Right before Claude finishes its response — before the response turn is finalized and control returns to the user.

**Why you'd use it:**
- Continue the conversation automatically (e.g., run additional validation steps)
- Prevent Claude from stopping prematurely
- Inject follow-up instructions or tasks
- Implement agentic loops that run until a condition is met

**Matcher field:** NONE — fires unconditionally before every stop.

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "Stop",
  "stop_hook_active": false,
  "last_assistant_message": "I've completed the implementation. The function now handles edge cases..."
}
```

| Field | Description |
|-------|-------------|
| `stop_hook_active` | `true` if a Stop hook is already running (prevents recursive Stop hooks). If `true`, this field prevents infinite loops — Stop hooks should check this |
| `last_assistant_message` | The full text of Claude's last response. Extracted from the last assistant message in the conversation; all text content blocks are joined with newlines |

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Success. Stdout and stderr are hidden. Claude stops normally |
| `2` | **CONTINUE.** Claude does NOT stop. Stderr is shown to the model as a new message. The model continues the conversation based on that message |
| Any other non-zero | Non-blocking. Stderr shown to user. Claude stops normally |

**Recursive loop prevention:** The `stop_hook_active` field is set to `true` when a Stop hook has triggered continuation (exit 2). Specifically, the `dm8()` function sets this flag and passes it as a parameter to the next invocation. Subsequent Stop invocations during that continuation will have `stop_hook_active: true`. Hooks MUST check this field and exit 0 when `stop_hook_active` is true, otherwise they create infinite loops.

**`transcript_path` during Stop hooks:** The `transcript_path` field points to the session JSONL file (`CH(session_id)` → `{session_dir}/{session_id}.jsonl`). Hook scripts can read this file to inspect the full conversation history, including the most recent assistant message.

**Practical examples:**

```bash
#!/bin/bash
# Run tests after every response and continue if they fail
input=$(cat)
active=$(echo "$input" | jq -r '.stop_hook_active')

# Prevent infinite loops
if [[ "$active" == "true" ]]; then
  exit 0
fi

# Check if tests pass
if ! npm test --silent 2>/dev/null; then
  echo "Tests are failing. Please fix the failing tests before finishing." >&2
  exit 2  # Continue the conversation
fi

exit 0  # Tests pass, allow Claude to stop
```

```bash
#!/bin/bash
# Verify all TODO comments were addressed before stopping
input=$(cat)
active=$(echo "$input" | jq -r '.stop_hook_active')

if [[ "$active" == "true" ]]; then exit 0; fi

todo_count=$(grep -r 'TODO' src/ 2>/dev/null | wc -l)
if [[ $todo_count -gt 0 ]]; then
  echo "There are still $todo_count TODO comments in the codebase. Please address them." >&2
  exit 2
fi
exit 0
```

---

### 4.10 SubagentStart

**Source:** `Uh8()` async generator at `src/tools/hasworktreecreatehook-1.ts:1457`. Builds `{ hook_event_name: "SubagentStart", agent_id: A, agent_type: q }` then calls `wx()` with `matchQuery: q` (the agent_type value).

**When:** When a Task tool call spawns a subagent (a new autonomous Claude instance).

**Why you'd use it:**
- Inject context or instructions into the subagent that the parent can't easily pass
- Configure subagent behavior (e.g., enforce constraints, set environment)
- Log subagent spawning for monitoring

**Matcher field:** `agent_type`

Known built-in `agent_type` values (all confirmed from source):

| Value | Source | Description |
|-------|--------|-------------|
| `"general-purpose"` | `src/agents/agent-4.ts:42` | Default Task tool subagent |
| `"Explore"` | `src/agents/agent-1.ts:55` | Explore agent for codebase investigation |
| `"Plan"` | `src/agents/agent-4.ts:252` | Planning agent |
| `"claude-code-guide"` | `src/agents/agent-4.ts:271` (`uW8 = "claude-code-guide"`) | Built-in guide agent for Claude Code documentation questions |
| `"statusline-setup"` | `src/agents/agent-4.ts:73` | Status line setup agent |
| `"main-session"` | `src/vendor/dom.ts:22310` | Main interactive session agent |
| `"subagent"` | `src/vendor/dom.ts:27657` | Generic subagent from SDK Task calls |
| `"teammate"` | `src/vendor/dom.ts:23749` | Teammate agent in multi-agent teams |
| `"magic-docs"` | `src/tools/tool-2.ts:1791` | Magic documentation agent |
| *(custom)* | User-defined | Any `agentType` string from custom agent definitions |

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "SubagentStart",
  "agent_id": "a1234567890abcdef",
  "agent_type": "general-purpose"
}
```

| Field | Description |
|-------|-------------|
| `agent_id` | Unique identifier for this subagent instance |
| `agent_type` | The type/role of the subagent |

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Success. Stdout is shown to the subagent as initial context |
| `2` | **IGNORED.** Blocking errors cannot prevent subagent start |
| Any other non-zero | Stderr shown to user |

**hookSpecificOutput fields:**

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SubagentStart",
    "additionalContext": "You are working as part of a team. Report back your findings clearly."
  }
}
```

**Subagent lifecycle:**

```
Parent agent calls Task tool
              |
              v
  SubagentStart hooks run
  (can inject context into subagent)
              |
              v
  Subagent receives context + runs
  (has its own tool hooks, Stop hooks, etc.)
              |
              v
  Subagent prepares to finish
              |
              v
  SubagentStop hooks run
  (can prevent subagent from stopping)
              |
     exit 2? ---> subagent continues, receives stderr as message
              |
              v
  Subagent concludes, returns result to parent
```

**Note on stop_hook_active:** Just like the main Stop hook, SubagentStop sets `stop_hook_active: true` when a continuation has been triggered. Always check this to prevent infinite loops.

---

### 4.11 SubagentStop

**Source:** `dm8()` async generator at `src/tools/hasworktreecreatehook-1.ts:1359`. This is the same function as Stop. When called with `agent_id` (parameter `z`) present, it builds `{ hook_event_name: "SubagentStop", stop_hook_active: Y, agent_id: z, agent_transcript_path: vb(z), agent_type: $ ?? "", last_assistant_message: O }` and calls `wx()` with no `matchQuery` override — the `matchQuery` defaults to `agent_type`. The `last_assistant_message` is extracted from the messages array by finding the last assistant message and joining all text content blocks with newlines.

**When:** Right before a subagent concludes its work (analogous to Stop but for subagents).

**Why you'd use it:**
- Validate subagent output before it's returned to the parent
- Continue a subagent that hasn't fully completed its task
- Enforce quality gates on subagent work

**Matcher field:** `agent_type`

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "SubagentStop",
  "stop_hook_active": false,
  "agent_id": "a1234567890abcdef",
  "agent_transcript_path": "/path/to/agent/transcript.jsonl",
  "agent_type": "general-purpose",
  "last_assistant_message": "I have completed the analysis..."
}
```

| Field | Description |
|-------|-------------|
| `stop_hook_active` | `true` if a SubagentStop hook is already running (prevents recursive loops) |
| `agent_id` | The subagent's unique identifier (hex string) |
| `agent_transcript_path` | Path to the subagent's own transcript file (see formula below) |
| `agent_type` | The subagent's type/role |
| `last_assistant_message` | The full text of the subagent's last response |

**`agent_transcript_path` formula:** `vb(agent_id)` resolves to:
```
{session_dir}/{session_id}/subagents/agent-{agent_id}.jsonl
```
This is the subagent's full conversation history in JSONL format. Hook scripts can read this file to inspect everything the subagent did.

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Success. Hidden. Subagent concludes normally |
| `2` | **CONTINUE.** The subagent does NOT stop. Stderr is shown to the subagent as a message, and it continues running |
| Any other non-zero | Non-blocking. Stderr shown to user |

**Note:** Same recursive loop prevention as Stop — always check `stop_hook_active`.

---

### 4.12 PreCompact

**Source:** `p01()` async function at `src/tools/hasworktreecreatehook-1.ts:1471`. Builds `{ hook_event_name: "PreCompact", trigger: A.trigger, custom_instructions: A.customInstructions }` then calls `mN6()` (the outside-REPL path) with `matchQuery: A.trigger`. Returns `{ newCustomInstructions, userDisplayMessage }` — hooks can set new compaction instructions via stdout.

**When:** Before conversation context is compacted (summarized to fit within the context window). Fires both on manual `/compact` and automatic compaction.

**Why you'd use it:**
- Inject custom instructions to guide what the compaction summary preserves
- Block compaction entirely (e.g., when in a critical section)
- Add domain-specific compaction hints

**Matcher field:** `trigger`

| Trigger value | When |
|--------------|------|
| `"manual"` | User explicitly ran `/compact` |
| `"auto"` | Context window reached capacity, auto-compact triggered |

**Trigger value details:**

- **`"manual"`** (`src/vendor/highlight.js.ts:15270`): User typed `/compact` in the UI.

- **`"auto"`** (`src/vendor/highlight.js.ts:16142`): Automatic compaction by Session Memory. Requires BOTH `tengu_session_memory` AND `tengu_sm_compact` settings to be `true`. Environment variable overrides:
  - `ENABLE_CLAUDE_CODE_SM_COMPACT=1` — force-enables auto-compact regardless of settings
  - `DISABLE_CLAUDE_CODE_SM_COMPACT=1` — force-disables auto-compact regardless of settings
  
  **Efficiency threshold:** Auto-compact is skipped if savings would be less than 20%. Specifically, if `postCompactTokens >= preCompactTokens * 0.8`, the compaction is aborted without running hooks. Only when the summary would save more than 20% does compaction proceed.

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "PreCompact",
  "trigger": "auto",
  "custom_instructions": null
}
```

| Field | Description |
|-------|-------------|
| `trigger` | `"manual"` or `"auto"` |
| `custom_instructions` | Any existing custom compaction instructions, or `null` if none |

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Success. Stdout is **appended** to the compaction instructions. These instructions guide the summary. Multiple hook outputs are joined with double newlines |
| `2` | **BLOCK compaction.** The compaction is cancelled entirely |
| Any other non-zero | Stderr shown to user, compaction proceeds |

**Compaction instruction processing:** When exit code 0 and stdout is non-empty, the hook output is joined with the existing `custom_instructions` (if any) and passed to the summarization model as guidance for what to preserve/prioritize.

**Compaction flow:**

```
Context window approaches capacity (or user runs /compact)
              |
              v
  PreCompact hooks run
  (trigger: "auto" or "manual")
              |
       exit 2? Yes ---> BLOCK compaction, stop here
              |
              v
  Collect hook outputs (stdout from exit-0 hooks)
              |
              v
  Join outputs with double-newlines
              |
              v
  Append to existing custom_instructions
              |
              v
  Run summarization model with combined instructions
              |
              v
  Replace conversation history with summary
```

**Hook output joining (from source):**
```typescript
let newCustomInstructions = w.length > 0
  ? w.join("\n\n")  // join multiple hook outputs
  : undefined;
```
Where `w` is an array of `output.trim()` from all succeeded hooks with non-empty output. The final compaction instructions are: `existingCustomInstructions + "\n\n" + newCustomInstructions`.

**Practical examples:**

```bash
#!/bin/bash
# Inject domain-specific preservation instructions
echo "When compacting, preserve:
- All file paths that were modified
- The current state of any ongoing refactoring
- Error messages and their resolutions
- The agreed-upon approach for the current task"
exit 0
```

```bash
#!/bin/bash
# Block auto-compact if a critical section marker exists
input=$(cat)
trigger=$(echo "$input" | jq -r '.trigger')

if [[ "$trigger" == "auto" && -f ".claude-critical-section" ]]; then
  echo "Auto-compact blocked: currently in a critical section" >&2
  exit 2
fi
exit 0
```

---

### 4.13 Setup

**When:** During repository/workspace setup. Fires on initial setup or periodic maintenance.

**Why you'd use it:**
- Install project-specific dependencies
- Run initialization scripts
- Provide setup status or instructions to Claude

**Matcher field:** `trigger`

| Trigger value | When |
|--------------|------|
| `"init"` | First-time setup of this workspace |
| `"maintenance"` | Periodic maintenance run |

**Trigger value details:**

The trigger is determined by CLI flags at launch (`src/agents/startdeferredprefetches-1.ts:1361`):
```typescript
let CY = l || U ? "init" : d ? "maintenance" : null;
```

- **`"init"`** (`src/agents/startdeferredprefetches-1.ts:1364`, `--init` CLI flag): First-time workspace setup. `forceSyncExecution: true` — Setup hooks block and the process **exits after hooks complete**. Claude does not continue to an interactive session.

- **`"maintenance"`** (`src/agents/startdeferredprefetches-1.ts:1361`, `--maintenance` CLI flag): Periodic maintenance run. Does NOT exit after hooks complete — continues with the normal interactive session.

**Setup input builder:** `$v8()` (`src/tools/hasworktreecreatehook-1.ts:1447`) constructs the hook input:
```typescript
async function* $v8(A, q, K = IX, Y) {
  let z = { ...EO(void 0), hook_event_name: "Setup", trigger: A };
  yield* wx({ hookInput: z, toolUseID: RE(), matchQuery: A, signal: q,
               timeoutMs: K, forceSyncExecution: Y })
}
```
Note: `matchQuery: A` means the `trigger` value is used as the matcher — hooks with a `matcher` field of `"init"` only fire on install, `"maintenance"` only on maintenance.

**HTTP hooks not supported:** Like SessionStart, HTTP hooks are filtered out for Setup events (`src/tools/hasworktreecreatehook-1.ts:594-600`):
```typescript
K === "SessionStart" || K === "Setup"
  ? P.filter((G) => {
      if (G.hook.type === "http") return false;  // Skipped with warning
      return true;
    })
  : P
```
HTTP hooks configured for Setup are silently skipped with a log message.

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "Setup",
  "trigger": "init"
}
```

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Success. Stdout is shown to Claude as context |
| `2` | **IGNORED.** Setup proceeds regardless |
| Any other non-zero | Stderr shown to user |

**hookSpecificOutput fields:**

```json
{
  "hookSpecificOutput": {
    "hookEventName": "Setup",
    "additionalContext": "Dependencies installed. Node 20, PostgreSQL 15 available."
  }
}
```

**Environment variables available:**
- `CLAUDE_ENV_FILE`: Path to the env file for this session. Set via `await R$4(q, _)` when `q === "SessionStart" || q === "Setup"` (`src/tools/hasworktreecreatehook-1.ts:293-294`). Provides a writable file path that persists environment variables across tool invocations.

**additionalContext support:** `hookSpecificOutput.additionalContext` is supported for Setup (`src/tools/hasworktreecreatehook-1.ts:221-222`). The additional context is collected and injected as a `hook_additional_context` block (`src/vendor/highlight.js.ts:15066-15079`).

---

### 4.14 TeammateIdle

**When:** A teammate agent (in the multi-agent "Teams" feature) is about to go idle.

**Why you'd use it:**
- Assign new work to a teammate before they go idle
- Keep teammates active for continuous processing
- Implement work distribution logic

**Matcher field:** NONE — fires for all teammates.

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "TeammateIdle",
  "teammate_name": "worker-1",
  "team_name": "my-team"
}
```

| Field | Description |
|-------|-------------|
| `teammate_name` | Name of the teammate going idle |
| `team_name` | Name of the team |

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Success. Hidden. Teammate goes idle |
| `2` | **PREVENT idle.** Stderr is shown to the teammate as a message, and they continue working |
| Any other non-zero | Stderr shown to user |

**hookSpecificOutput:** NOT supported. TeammateIdle has no case in the `hookSpecificOutput` switch statement (`src/tools/hasworktreecreatehook-1.ts:186-253`). This is a genuine stub — no `hookSpecificOutput` fields are processed for this event.

**Input builder:** `cm8()` (`src/tools/hasworktreecreatehook-1.ts:1397`) constructs the hook input:
```typescript
async function* cm8(A, q, K, Y, z = IX) {
  let w = { ...EO(K), hook_event_name: "TeammateIdle",
             teammate_name: A, team_name: q };
  yield* wx({ hookInput: w, toolUseID: RE(), signal: Y, timeoutMs: z });
}
```
Note: `EO(K)` passes the teammate's app state (not `void 0`) — teammate-scoped session context.

**No matcher field:** TeammateIdle has no matcher case in `Hd8()` (`src/tools/hasworktreecreatehook-1.ts:533-535`). The match variable `_` is never set, so matcher filtering is not applied — all TeammateIdle hooks fire for every teammate.

**TeammateIdle flow:**

```
Teammate agent completes its current task
              |
              v
  TeammateIdle hooks run
    exit 2? ---> teammate receives message, continues working
              |
              v
  Teammate goes idle (available for new tasks)
```

**Exit 2 feedback message:** When exit 2 is returned, the message is formatted as `"TeammateIdle hook feedback:\n{stderr}"` (`src/tools/hasworktreecreatehook-1.ts:623-626`, function `Um8()`) and shown to the teammate model.

**Idle detection architecture:** TeammateIdle hooks fire as part of the callback-based idle system (`src/conversation/waitforteammatestobecomeidle-1.ts:107`):
- `MW8()` waits for running teammates to become idle by registering callbacks on their `onIdleCallbacks` array
- When a teammate's `isIdle` flag transitions to `true`, all registered callbacks in `onIdleCallbacks` are fired
- The `gBY()` poll loop (`src/vendor/dom.ts:23616`) runs per-teammate and checks for pending messages, shutdown requests, and task completion before going idle

**Teammate task object fields** (from `src/vendor/parse5.ts:13490`):
```typescript
{
  type: "in_process_teammate",
  status: "running" | "idle" | "completed",
  isIdle: boolean,
  shutdownRequested: boolean,
  pendingUserMessages: Message[],
  onIdleCallbacks: (() => void)[],
  identity: string,
  prompt: string,
  model: string,
  abortController: AbortController,
  awaitingPlanApproval: boolean,
  lastReportedToolCount: number,
  lastReportedTokenCount: number,
  messages: Message[],
  localTaskId: string,
}
```

---

### 4.15 TaskCompleted

**When:** A task is being marked as complete by a teammate agent.

**Why you'd use it:**
- Validate task completion criteria
- Prevent a task from being marked done prematurely
- Log task completions

**Matcher field:** NONE — fires for all task completions.

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "TaskCompleted",
  "task_id": "task-uuid-string",
  "task_subject": "Fix authentication bug",
  "task_description": "The login flow breaks when the session expires...",
  "teammate_name": "worker-1",
  "team_name": "my-team"
}
```

| Field | Description |
|-------|-------------|
| `task_id` | UUID of the task |
| `task_subject` | Short title of the task |
| `task_description` | Full description of the task |
| `teammate_name` | Which teammate is completing the task |
| `team_name` | Name of the team |

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Success. Hidden. Task is marked complete |
| `2` | **PREVENT completion.** Stderr is shown to the model, task is NOT marked complete |
| Any other non-zero | Stderr shown to user |

**hookSpecificOutput:** NOT supported. TaskCompleted has no case in the `hookSpecificOutput` switch statement (`src/tools/hasworktreecreatehook-1.ts:186-253`). This is a genuine stub — no `hookSpecificOutput` fields are processed for this event.

**Input builder:** `Al6()` (`src/tools/hasworktreecreatehook-1.ts:1405`) constructs the hook input:
```typescript
async function* Al6(A, q, K, Y, z, w, _, $ = IX, H) {
  let O = {
    ...EO(w),
    hook_event_name: "TaskCompleted",
    task_id: A,
    task_subject: q,
    task_description: K,
    teammate_name: Y,
    team_name: z,
  };
  yield* wx({ hookInput: O, toolUseID: RE(), signal: _, timeoutMs: $,
               toolUseContext: H });
}
```
Note: `EO(w)` passes the teammate's app state (6th argument `w` is the app state getter).

**No matcher field:** TaskCompleted has no matcher case in `Hd8()` (`src/tools/hasworktreecreatehook-1.ts:533-535`). The match variable `_` is never set, so all TaskCompleted hooks fire for every task completion.

**TaskCompleted flow:**

```
Teammate agent signals task completion
              |
              v
  TaskCompleted hooks run
    exit 2? ---> task NOT marked complete, model receives reason
              |
              v
  Task marked as completed in the team's task list
```

**Exit 2 feedback message:** When exit 2 is returned, the message is formatted as `"TaskCompleted hook feedback:\n{stderr}"` (`src/tools/hasworktreecreatehook-1.ts:627-629`, function `ec6()`) and shown to the teammate model.

---

### 4.16 ConfigChange

**When:** A configuration file changes during a session (user reloads settings, policy updates, etc.).

**Why you'd use it:**
- Validate config changes before they apply
- Block unauthorized config modifications
- Log config changes for audit

**Matcher field:** `source`

| Source value | Description |
|-------------|-------------|
| `"user_settings"` | `~/.claude/settings.json` changed |
| `"project_settings"` | `.claude/settings.json` changed |
| `"local_settings"` | `.claude/settings.local.json` changed |
| `"policy_settings"` | Organization policy settings changed |
| `"skills"` | Skills configuration changed |

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "ConfigChange",
  "source": "project_settings",
  "file_path": "/project/.claude/settings.json"
}
```

| Field | Description |
|-------|-------------|
| `source` | Which config source changed |
| `file_path` | Absolute path to the changed file |

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Allow the change to apply |
| `2` | **BLOCK the change.** The config update is NOT applied |
| Any other non-zero | Stderr shown to user |

**Input builder:** `BN6()` (`src/tools/hasworktreecreatehook-1.ts:1552`) constructs the hook input and uses `mN6()` (outside-REPL execution) rather than `wx()`:
```typescript
async function BN6(A, q, K = IX) {
  let Y = { ...EO(void 0), hook_event_name: "ConfigChange",
             source: A, file_path: q };
  let z = await mN6({ hookInput: Y, timeoutMs: K, matchQuery: A });
  if (A === "policy_settings") return z.map((w) => ({ ...w, blocked: false }));
  return z;
}
```
Note: `matchQuery: A` means the `source` value is used as the matcher field.

**Important restriction:** Changes to `policy_settings` source CANNOT be blocked. Even if exit 2 is returned, the blocking status is overridden to non-blocking for policy settings changes (`src/tools/hasworktreecreatehook-1.ts:1560`):
```typescript
if (A === "policy_settings") return z.map((w) => ({ ...w, blocked: false }));
```

**File watcher configuration (chokidar):**

ConfigChange hooks are triggered by a file watcher (`Nj6.watch`) that monitors all configuration paths:

```javascript
Nj6.watch(paths, {
  persistent: true,
  ignoreInitial: true,
  depth: 2,
  awaitWriteFinish: { stabilityThreshold: 1000, pollInterval: 500 },
  ignored: (path) => path.split('/').includes('.git'),
  ignorePermissionErrors: true,
  usePolling: false,
  atomic: true
})
```

Events watched: `add`, `change`, `unlink` (file creation, modification, deletion all trigger ConfigChange).

**Source-to-path mapping:**

| Source value | Paths watched |
|-------------|---------------|
| `"user_settings"` | `~/.claude/settings.json`, `~/.claude/skills/`, `~/.claude/commands/` |
| `"project_settings"` | `<project>/.claude/settings.json` and related files |
| `"local_settings"` | Workspace-specific `.claude/` directories |
| `"policy_settings"` | Admin/policy config paths (managed externally) |
| `"skills"` | Dynamic skill file changes |

**Reload sequence after ConfigChange hook runs:**

After the hook executes (function `Cc8()` triggers `BN6()`):
1. `zD1()` — clear loaded skills
2. `gI()` — reload config from disk
3. `Gc()` — clear permission/tool cache
4. Notify all registered config listeners

---

### 4.17 WorktreeCreate

**When:** Claude Code is creating an isolated git worktree for a subagent. This happens in multi-agent scenarios where subagents work in isolated copies of the repository.

**Why you'd use it:**
- Use a custom directory structure for worktrees
- Apply custom naming conventions
- Run initialization in the new worktree
- Integrate with custom workspace management systems

**Matcher field:** NONE

**CRITICAL behavior:** This hook REPLACES the default worktree creation. If a WorktreeCreate hook is registered and returns successfully, the system uses the hook's output as the worktree path — it does NOT create a worktree itself. The hook is responsible for actually creating the directory.

**Worktree lifecycle:**

```
Task tool spawns subagent in isolated worktree
              |
              v
  hasWorktreeCreateHook() check
    Has WorktreeCreate hooks? --No--> Use default git worktree creation
              |
             Yes
              v
  WorktreeCreate hooks run
  (hook must create directory and return absolute path in stdout)
              |
     Hook succeeded? --No--> throw Error("WorktreeCreate hook failed: ...")
              |
             Yes
              v
  worktreePath = hook_stdout.trim()
              |
              v
  [subagent runs in worktree directory]
              |
              v
  Subagent completes
              |
              v
  WorktreeRemove hooks run
  (hook receives worktree_path, responsible for cleanup)
              |
  Hooks fail? ---> Log error, but don't throw
              |
              v
  Worktree cleanup complete
```

**Important:** If there are NO WorktreeCreate hooks registered, the system uses its built-in git worktree creation. The hooks ONLY take effect if at least one is configured. `hasWorktreeCreateHook()` checks `settings.WorktreeCreate?.length > 0`.

**Execution model:** Uses `mN6()` (outside-REPL execution), NOT `wx()` (`src/tools/hasworktreecreatehook-1.ts:1714`). This means it runs synchronously and returns an array of results rather than yielding them as a stream. Prompt, agent, and function hook types are not supported (only `command` and `callback` hooks work).

**Input builder:** `DN1()` (`src/tools/hasworktreecreatehook-1.ts:1712`) constructs the hook input:
```typescript
async function DN1(A) {
  let q = { ...EO(void 0), hook_event_name: "WorktreeCreate", name: A };
  let K = await mN6({ hookInput: q, timeoutMs: IX });
  let Y = K.find((w) => w.succeeded && w.output.trim().length > 0);
  if (!Y) {
    let w = K.filter((_) => !_.succeeded)
               .map((_) => `${_.command}: ${_.output.trim() || "no output"}`);
    throw Error(`WorktreeCreate hook failed: ${w.join("; ") || "no successful output"}`);
  }
  return { worktreePath: Y.output.trim() };
}
```

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "WorktreeCreate",
  "name": "suggested-worktree-slug"
}
```

| Field | Description |
|-------|-------------|
| `name` | A suggested slug/name for the worktree (e.g., derived from the task name) |

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | **Stdout MUST contain the absolute path to the created worktree directory.** This path is used as the worktree location. Example stdout: `/home/user/worktrees/task-abc123` |
| Any other non-zero | Creation FAILED. An error is thrown: `WorktreeCreate hook failed: [stderr or 'no successful output']` |

**Success condition:** The first hook that exits 0 with non-empty stdout wins. The `output.trim()` of that hook's stdout is used as the worktree path. If all hooks fail or return empty stdout, an error is thrown.

**Practical example:**

```bash
#!/bin/bash
# Create worktrees in a custom location with a timestamp
input=$(cat)
name=$(echo "$input" | jq -r '.name')
timestamp=$(date +%Y%m%d_%H%M%S)
worktree_dir="/tmp/claude-worktrees/${name}_${timestamp}"

mkdir -p "$worktree_dir"
git worktree add "$worktree_dir" HEAD 2>/dev/null || cp -r . "$worktree_dir"

# Output must be the absolute path
echo "$worktree_dir"
exit 0
```

---

### 4.18 WorktreeRemove

**When:** After a subagent completes and its worktree should be cleaned up.

**Why you'd use it:**
- Custom cleanup logic (archive instead of delete)
- Preserve worktrees for debugging
- Integrate with external workspace management

**Matcher field:** NONE

**Execution model:** Uses `mN6()` (outside-REPL execution), NOT `wx()` (`src/tools/hasworktreecreatehook-1.ts:1734`). Same as WorktreeCreate — runs synchronously outside the REPL. Prompt, agent, and function hook types are not supported.

**Input builder:** `MN1()` (`src/tools/hasworktreecreatehook-1.ts:1726`) constructs the hook input:
```typescript
async function MN1(A) {
  let K = TL1()?.WorktreeRemove;
  if (!K || K.length === 0) return false;
  let Y = { ...EO(void 0), hook_event_name: "WorktreeRemove",
             worktree_path: A };
  let z = await mN6({ hookInput: Y, timeoutMs: IX });
  if (z.length === 0) return false;
  for (let w of z)
    if (!w.succeeded)
      R(`WorktreeRemove hook failed [${w.command}]: ${w.output.trim()}`,
        { level: "error" });
  return true;
}
```
Note: `MN1()` pre-checks `TL1()?.WorktreeRemove` (policy hooks) before executing — returns `false` immediately if no policy hooks are configured. Unlike `DN1()`, failures do NOT throw exceptions; they are logged at `error` level.

**Input schema (written to stdin):**
```json
{
  "session_id": "uuid-string",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "WorktreeRemove",
  "worktree_path": "/absolute/path/to/worktree"
}
```

| Field | Description |
|-------|-------------|
| `worktree_path` | Absolute path to the worktree that should be removed |

**Exit code behavior:**

| Exit Code | Effect |
|-----------|--------|
| `0` | Success |
| Any other non-zero | Logged as error (`level: "error"`), but does NOT throw an exception |

**Practical example:**

```bash
#!/bin/bash
# Archive the worktree instead of deleting it
input=$(cat)
path=$(echo "$input" | jq -r '.worktree_path')
archive_dir="/tmp/claude-worktree-archive"

mkdir -p "$archive_dir"
mv "$path" "$archive_dir/$(basename $path)_$(date +%s)"
exit 0
```

---

## 5. Hook Matching

The matcher determines WHICH hook groups are triggered for a given event. Hook discovery is performed by `Hd8()` (`hasworktreecreatehook-1.ts:505-632`), which calls `wOz()` to collect hooks and `zOz()` to evaluate each matcher.

**`zOz(A, q)`** (`hasworktreecreatehook-1.ts:437-453`) — evaluates whether a matcher string matches a query value.

**`wOz(A, q)`** (`hasworktreecreatehook-1.ts:475-502`) — collects and merges hooks from all sources into an event-keyed map.

**`Hd8(A, q, K, Y)`** (`hasworktreecreatehook-1.ts:505-632`) — top-level hook discovery function:
- `A`: appState (or undefined for outside-REPL calls)
- `q`: agentId string
- `K`: event name string
- `Y`: hookInput object (used to extract the match query)
- Returns: array of `{ hook, pluginRoot, pluginId, skillRoot }` objects (deduplicated)
- On any error: returns `[]` (caught internally)

### Matcher Evaluation Logic

**Step 1:** Determine the query value from the hook input:

```
event                  query value
─────────────────────────────────────────────
PreToolUse             tool_name
PostToolUse            tool_name
PostToolUseFailure     tool_name
PermissionRequest      tool_name
Notification           notification_type
SessionStart           source
SessionEnd             reason
Setup                  trigger
PreCompact             trigger
SubagentStart          agent_type
SubagentStop           agent_type
ConfigChange           source
Stop                   (no matcher)
UserPromptSubmit       (no matcher)
TeammateIdle           (no matcher)
TaskCompleted          (no matcher)
WorktreeCreate         (no matcher)
WorktreeRemove         (no matcher)
```

Events with no matcher field always run all hooks in the group (matcher is ignored).

**Step 2:** Normalize the query value via `wN()` (likely lowercases or trims it).

**Step 3:** Evaluate the matcher string:

#### Case 1: No matcher or `"*"`
```typescript
if (!q || q === "*") return true;
```
All values match. Use this to catch everything.

#### Case 2: Simple alphanumeric (including pipe-separated)
```typescript
if (/^[a-zA-Z0-9_|]+$/.test(q)) {
  if (q.includes("|"))
    return q.split("|").map(Y => wN(Y.trim())).includes(A);
  return A === wN(q);
}
```
- `"Bash"` → exact match (case-insensitive via normalization)
- `"Write|Edit|Bash"` → match if any segment matches
- Pipe-separated values are split on `|`, each segment is normalized and compared

#### Case 3: Regex pattern
If the pattern contains characters outside `[a-zA-Z0-9_|]`, it is treated as a regex:
```typescript
let K = new RegExp(q);
if (K.test(A)) return true;
for (let Y of Pw4(A)) if (K.test(Y)) return true;
return false;
```
- The regex is tested against the query value
- It's also tested against all **aliases** of the query value (via `Pw4()`), which may include alternative names or normalized forms of tool names
- Invalid regex: logs error `"Invalid regex pattern in hook matcher: [pattern]"`, returns `false` (no match)

### Matcher Examples

| Matcher | Matches |
|---------|--------|
| `"*"` | Everything |
| (omitted) | Everything |
| `"Bash"` | Only the Bash tool |
| `"Write"` | Only the Write tool |
| `"Write\|Edit"` | Write OR Edit tool |
| `"Write\|Edit\|Bash"` | Write OR Edit OR Bash |
| `"/^mcp__/"` | All MCP tools (regex) |
| `"/^mcp__my_server__/"` | All tools from `my_server` MCP server |
| `"idle_prompt"` | Only idle_prompt notifications |
| `"startup\|resume"` | Session starts from startup or resume |
| `"manual"` | Manual compaction only |
| `"init"` | Setup init trigger only |

### Deduplication

After matching, hooks of the same type are deduplicated by their identifying key:
- `command` hooks: deduplicated by `command` string
- `prompt` hooks: deduplicated by `prompt` string
- `http` hooks: deduplicated by `url` string
- `agent` hooks: deduplicated by `prompt([])` result
- `callback`/`function` hooks: NOT deduplicated

This means if the same command appears in multiple matcher groups that all match, it only runs once.

**Hook merge order (`wOz`, `hasworktreecreatehook-1.ts:475-502`):**

Hooks from different sources are merged in this priority order:

| Priority | Source | Function | Notes |
|----------|--------|----------|-------|
| 1 (first) | Policy settings | `TL1` | Highest authority; blocks user hooks in `allowManagedHooksOnly` mode |
| 2 | Plugin hooks | `zk6` | Skipped when in managed-only mode |
| 3 | User hooks | `PP1` | From `~/.claude/settings.json` |
| 4 (last) | Workspace hooks | `PX7` | Project-level hooks |

Later sources ADD to earlier ones; earlier sources override on duplicate command strings. That is, if policy defines a command and user defines the same command, only the policy version runs (it appeared first and the duplicate is dropped).

**Output validation (Zod schema `vL1`):**

Hook stdout JSON is validated against a Zod union schema:
```
vL1 = union of:
  { async: true, asyncTimeout?: number }  <-- async signal
  {
    continue?: boolean,
    suppressOutput?: boolean,
    stopReason?: string,
    decision?: "approve" | "block",
    reason?: string,
    systemMessage?: string,
    permissionDecision?: "allow" | "deny" | "ask",
    hookSpecificOutput?: <per-event-type union>
  }
```

If stdout does not parse as JSON, it is treated as plain text `additionalContext`. If it parses as JSON but fails schema validation, the error is logged and output is ignored.

---

## 6. Hook Execution Pipeline

The core execution function is `wx()` (`hasworktreecreatehook-1.ts:635-1135`), an async generator. The outside-REPL path uses `mN6()` (`hasworktreecreatehook-1.ts:1149-1287`). Command subprocess execution uses `CL1()` (`hasworktreecreatehook-1.ts:277-415`). Response parsing uses `mPq()` (`hasworktreecreatehook-1.ts:120-245`) and `uPq()` (`hasworktreecreatehook-1.ts:96-120`).

**`wx()` parameters** (`hasworktreecreatehook-1.ts:635`):
```typescript
async function* wx({
  hookInput,          // hook event input object
  toolUseID,          // UUID for this hook invocation
  matchQuery,         // optional override for matcher (usually auto-derived from hookInput)
  signal,             // AbortSignal for cancellation
  timeoutMs = IX,     // default timeout from imported constant IX
  toolUseContext,     // required for prompt/agent hooks
  messages,           // required for function hooks
  forceSyncExecution, // if true: disables async/background detection
})
```

Here is the complete flow:

### Phase 1: Precondition Checks

```
wx() called
    |
    ├─> Kn6()? (disableAllHooks policy) --> return (skip all hooks)
    |
    ├─> CLAUDE_CODE_SIMPLE env var set? --> return (skip all hooks)
    |
    └─> RL1()? (workspace trust not accepted) --> return (skip all hooks)
```

### Phase 2: Hook Discovery

```
Hd8(appState, agentId, eventName, hookInput)
    |
    ├─> wOz(): collect all hooks from all sources (config, plugins, skills, session)
    ├─> Filter by event name
    ├─> For each matching group: filter by matcher (zOz)
    ├─> Flatten to list of {hook, pluginRoot, pluginId, skillRoot}
    ├─> Deduplicate by command/prompt/url
    ├─> Filter out HTTP hooks for SessionStart/Setup
    └─> Return deduplicated list
```

### Phase 3: Progress Events

Before execution, for each hook, yields a progress event:
```typescript
{ type: "progress", data: { type: "hook_progress", hookEvent, hookName, command, statusMessage } }
```
This drives the spinner/status display in the UI.

### Phase 4: Parallel Execution

All hooks run concurrently via `Promise.all`-style parallel execution (using `DT1()` which merges async generators). Each hook runs independently:

**For `command` hooks:**
1. Spawn child process (`spawn(command, [], { env, cwd, shell })`) with environment variables
2. Write hook input JSON to stdin
3. Monitor stdout/stderr streams
4. Check initial stdout for async signal (`{ async: true }`)
5. If async signaled and not forceSyncExecution: background the process via `bPq()`, return immediately
6. Wait for process to close
7. Parse stdout: `uPq()` (try JSON, fallback to plain text)
8. Handle exit code and output

**For `prompt` hooks:** Dispatch to `yPq()` — calls Claude API with the prompt

**For `agent` hooks:** Dispatch to `SPq()` — spawns autonomous agent

**For `callback` hooks:** Call `callback(hookInput, toolUseID, signal, hookIndex, appStateAccessors)` directly

**For `function` hooks:** Call `callback(messages, signal)` — used for internal Stop hooks

**For `http` hooks:** Currently logs skip message and returns without executing

### Phase 5: Result Processing

As each hook completes, results are yielded through the generator:

```
outcome = "success" | "blocking" | "non_blocking_error" | "cancelled"
    |
    ├─> outcome == "blocking": yield { blockingError }
    |
    ├─> additionalContext present: yield { additionalContexts: [context] }
    |
    ├─> updatedMCPToolOutput present: yield { updatedMCPToolOutput }
    |
    ├─> permissionBehavior present:
    │     accumulate: deny > ask > allow (f variable)
    │     yield { permissionBehavior, hookPermissionDecisionReason, updatedInput }
    |
    ├─> updatedInput present (without permissionBehavior): yield { updatedInput }
    |
    └─> permissionRequestResult present: yield { permissionRequestResult }
```

### Phase 6: Telemetry

After all hooks complete:
- Records `tengu_repl_hook_finished` event with counts (success, blocking, non_blocking_error, cancelled)
- Emits `hook_execution_complete` workflow event if workflow tracking is enabled
- Calls `sS1(totalDurationMs)` to record hook duration

### Response Parsing

**`uPq(A)` — stdout parser** (`hasworktreecreatehook-1.ts:96-120`):
- If stdout doesn't start with `{`: returns `{ plainText: A }` (treat as plain text context)
- If stdout starts with `{`: attempts JSON parse via `YOz()` then Zod validation via `vL1.safeParse()`
- On success: returns `{ json: parsedObject }`
- On validation failure: returns `{ plainText: A, validationError: errorMessage }` with full schema shown in error
- On JSON parse failure: returns `{ plainText: A }` (silently treats as plain text)

**`mPq({ json, command, hookName, toolUseID, hookEvent, expectedHookEvent, stdout, stderr, exitCode, durationMs })` — result assembler** (`hasworktreecreatehook-1.ts:120-245`):
- Processes parsed JSON into the structured result object `J`
- Handles: `continue: false` → `preventContinuation`, `decision: "approve"/"block"` → `permissionBehavior`, `systemMessage`, `hookSpecificOutput` dispatch
- `hookSpecificOutput` switch cases: `PreToolUse`, `UserPromptSubmit`, `SessionStart`, `Setup`, `SubagentStart`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest` — all others are no-ops (genuine stubs)
- Returns `{ ...J, message: Gq({...}) }` where `message` is either `hook_blocking_error` or `hook_success` event

### The Outside-REPL Path

Some events (Notification, SessionEnd, PreCompact, WorktreeCreate, WorktreeRemove, ConfigChange) use `mN6()` (`hasworktreecreatehook-1.ts:1149-1287`) instead of `wx()`. Key differences:
- Returns a flat array of results instead of async-yielding
- No progress events or telemetry integration
- Simpler result format: `{ command, succeeded, output, blocked }`
- `prompt` and `agent` hook types are NOT supported (fall through with error messages)
- `function` hooks log an error and return immediately

---

## 7. Exit Code Reference

Comprehensive table of exit code effects for all events:

| Event | Exit 0 | Exit 2 | Other Non-Zero |
|-------|--------|--------|-----------------|
| **PreToolUse** | Hidden; parse JSON for decisions | BLOCK tool; stderr shown to model | Stderr to user; tool continues |
| **PostToolUse** | Transcript mode; parse JSON | Stderr shown to model | Stderr to user |
| **PostToolUseFailure** | Transcript mode; parse JSON | Stderr shown to model | Stderr to user |
| **PermissionRequest** | Parse JSON for decision; use if provided | (same as other non-zero) | Stderr to user; show normal dialog |
| **Notification** | Hidden | (same as other non-zero) | Stderr to user |
| **UserPromptSubmit** | Stdout shown to Claude as context | BLOCK prompt; erase input; stderr to user | Stderr to user; prompt proceeds |
| **SessionStart** | Stdout shown to Claude as context | IGNORED (session starts regardless) | Stderr to user |
| **SessionEnd** | Success | (same as other non-zero) | Written to `process.stderr` directly |
| **Stop** | Hidden; Claude stops | CONTINUE; stderr shown to model | Stderr to user; Claude stops |
| **SubagentStart** | Stdout shown to subagent as context | IGNORED | Stderr to user |
| **SubagentStop** | Hidden; subagent stops | CONTINUE subagent; stderr to subagent | Stderr to user; subagent stops |
| **PreCompact** | Stdout appended to compact instructions | BLOCK compaction | Stderr to user; compaction proceeds |
| **Setup** | Stdout shown to Claude as context | IGNORED | Stderr to user |
| **TeammateIdle** | Hidden; teammate goes idle | PREVENT idle; stderr to teammate | Stderr to user |
| **TaskCompleted** | Hidden; task marked complete | PREVENT completion; stderr to model | Stderr to user |
| **ConfigChange** | Allow change | BLOCK change (except policy_settings) | Stderr to user |
| **WorktreeCreate** | stdout = worktree path (required) | (creation FAILED) | Error thrown |
| **WorktreeRemove** | Success | (same as other non-zero) | Log error; no throw |

### The "blocking" vs "non-blocking" distinction

**Exit 2 = blocking error.** The specific meaning varies by event:
- For tool events: the tool call is denied/blocked, model sees the error
- For lifecycle events (Stop, SubagentStop): conversation/subagent continues
- For permission events (ConfigChange): the config change doesn't apply
- For setup events (SessionStart, SubagentStart, Setup): IGNORED, cannot block

**Other non-zero = non-blocking error.** Stderr is visible to the user (in the terminal output), but the operation proceeds as if the hook didn't run. Use this for diagnostic information you want operators to see without affecting Claude's behavior.

**Exit 0 = success.** Hook ran without issues. Further effect depends on stdout content.

---

## 8. Environment Variables

### Variables Set in Hook Subprocess Environment

These are available in every hook subprocess:

| Variable | Value | Available In |
|----------|-------|--------------|
| `CLAUDE_PROJECT_DIR` | Platform-adjusted project root: `j(f$())` where `j` = `ES()` on Windows or identity on other platforms (`hasworktreecreatehook-1.ts:299`) | All hook types |
| `CLAUDE_PLUGIN_ROOT` | Platform-adjusted plugin root: `j($)` or `j(H)` where `$`/`H` are plugin root paths passed to `CL1()` (`hasworktreecreatehook-1.ts:302-303`) | Plugin hooks only |
| `CLAUDE_ENV_FILE` | Env file path from `await R$4(q, _)` where `q` is the event name and `_` is env file content (`hasworktreecreatehook-1.ts:304-305`) | `SessionStart` and `Setup` only |

Plus all existing `process.env` variables are passed through via spread (`{ ...process.env, CLAUDE_PROJECT_DIR: ... }`).

**`CL1()` subprocess environment setup** (`hasworktreecreatehook-1.ts:297-305`):
```typescript
let W = { ...process.env, CLAUDE_PROJECT_DIR: j(X) };
if ($) W.CLAUDE_PLUGIN_ROOT = j($);   // $ = plugin root from arg 7
if (H) W.CLAUDE_PLUGIN_ROOT = j(H);   // H = skill root from arg 8
if ((q === "SessionStart" || q === "Setup") && _ !== void 0)
  W.CLAUDE_ENV_FILE = await R$4(q, _);
```

### `CLAUDE_CODE_SIMPLE`

If this environment variable is set to a truthy value, all hook execution is skipped globally. The check happens at the top of the `wx()` execution path and the `mN6()` path:
```typescript
if (_1(process.env.CLAUDE_CODE_SIMPLE)) return;
```
This is a quick way to disable hooks entirely for debugging or simplified operation.

### `CLAUDE_CODE_SHELL_PREFIX`

If set, the command hook's shell command is prefixed with this value. Useful for running hooks through wrappers (e.g., a timeout utility or environment setup script).

### Variable Substitution in Plugin Hook Commands

For plugin-defined command hooks, `${CLAUDE_PLUGIN_ROOT}` in the command string is replaced with the plugin's root path. On Windows, the path is escaped appropriately. Example:
```json
{ "type": "command", "command": "${CLAUDE_PLUGIN_ROOT}/scripts/validate.sh" }
```

### allowedEnvVars for HTTP Hooks

The `allowedEnvVars` array in HTTP hook definitions specifies which environment variables can be interpolated into the `url` and `headers` fields. This is a security measure to prevent arbitrary env var access.

---

## 9. Async and Background Hooks

Normally, Claude Code waits for a hook to complete before proceeding. Async/background hooks allow Claude to continue immediately while the hook runs in the background.

**Key functions** (all in `hasworktreecreatehook-1.ts`):
- **`l16(A)`** (async detection) — checks if a parsed JSON response has `async: true`. Used at line ~356 to detect response-based async signaling.
- **`bPq({ processId, hookId, shellCommand, asyncResponse, hookEvent, hookName, command })`** (lines 49-68) — initiates backgrounding by calling `shellCommand.background(processId)`, then hands off to `VH7()` with the full process context.
- **`VH7({ processId, hookId, asyncResponse, hookEvent, hookName, command, shellCommand })`** — manages backgrounded process lifecycle (timeout enforcement, final cleanup). Imported from external module (`xN6`).

### Method 1: Config-based (`"async": true`)

```json
{
  "type": "command",
  "command": "my-slow-audit-script.sh",
  "async": true
}
```

**Behavior** (`CL1()`, `hasworktreecreatehook-1.ts:313-327`):
1. Process is spawned via `AOz` (Node.js `spawn`)
2. Async check: `if (A.async && !O)` — if `hook.async === true` AND `forceSyncExecution` is false
3. Hook input JSON is written to stdin: `N.stdin.write(Y, "utf8")`
4. `N.stdin.end()` is called
5. `bPq({ processId: X6, hookId: w, shellCommand: v, asyncResponse: { async: true, asyncTimeout: P }, ... })` — backgrounds the process
6. Returns `{ stdout: "", stderr: "", output: "", status: 0, backgrounded: true }` immediately
7. Claude does not wait for the hook to complete

### Method 2: Response-based async

The hook itself can signal that it wants to run in the background by returning JSON with `async: true` before finishing:

```bash
#!/bin/bash
# Signal async then do slow work in background
echo '{"async": true}'  # Claude reads this and backgrounds us
# Now do slow work...
sleep 30
./slow-operation.sh
```

**Behavior** (`CL1()`, `hasworktreecreatehook-1.ts:340-365`):
1. Process starts, Claude monitors stdout stream with `N.stdout.on("data", ...)`
2. Once stdout includes `}` and hasn't yet been checked: `p = true`, attempts `_8(S.trim())` JSON parse
3. If `l16(D6)` returns true AND `forceSyncExecution` (`O`) is false: async detected
4. `bPq({ processId: q6, hookId: w, shellCommand: v, asyncResponse: D6, ... })` — backgrounds process
5. `L = true`, resolves the `x` promise: `g?.({ stdout: S, stderr: m, output: h, status: 0 })`
6. Claude returns immediately with success via `Promise.race([x, J6, $6])`
7. The process continues running in background via `VH7()`

**Optional field:** The async response can include `asyncTimeout` to set a specific timeout for the background phase:
```json
{ "async": true, "asyncTimeout": 300000 }
```

### `forceSyncExecution`

Some internal callers (like `SessionStart` with the `forceSyncExecution` parameter) can override async behavior. When `forceSyncExecution: true` is passed to `wx()`, the async detection is skipped and hooks run synchronously regardless of their async configuration or response.

### What happens to backgrounded hooks

Backgrounded hooks:
- Continue running under their original timeout
- Their stdout/stderr/exit code are NOT processed for permission decisions or context injection
- They can still write to files, call APIs, etc.
- If they exceed their timeout, the process is killed
- Failures in backgrounded hooks are not surfaced to the user or model

---

## 10. Plugin Hook System

### Plugin Hook Configuration

Plugins define their hooks in one of two ways:

**1. Hooks configuration file** (`hooks/hooks.json` or path specified in manifest):
```json
{
  "PreToolUse": [
    {
      "matcher": "Write",
      "hooks": [
        { "type": "command", "command": "${CLAUDE_PLUGIN_ROOT}/validate-write.sh" }
      ]
    }
  ]
}
```

**2. `hooksConfig` field in plugin manifest:**
```json
{
  "name": "my-plugin",
  "hooksConfig": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "my-command" }
        ]
      }
    ]
  }
}
```

### Plugin Hook Metadata

Plugin hooks carry additional metadata that user-configured hooks don't have:

```typescript
{
  matcher: "...",
  hooks: [...],
  pluginRoot: "/path/to/plugin",   // Root directory of the plugin
  pluginName: "my-plugin",          // Plugin display name
  pluginId: "my-plugin@1.0.0"       // Plugin source/version identifier
}
```

This metadata is used for:
- `CLAUDE_PLUGIN_ROOT` environment variable substitution
- Telemetry (tracking which plugins are running hooks)
- The `allowManagedHooksOnly` policy behavior (plugin hooks ARE considered managed hooks)

### Loading Process (`setuppluginhookhotreload-2.ts:314-347`)

**Key functions:**
- **`rB`** / **`loadPluginHooks()`** (`setuppluginhookhotreload-2.ts:314`) — memoized async function (`j8()` wrapper) that loads all plugin hooks
- **`Xr9(A)`** (`setuppluginhookhotreload-2.ts:232-266`) — extracts hook config from a single plugin, initializing all 18 event type keys then populating from `A.hooksConfig`
- **`CP6`** / **`clearPluginHookCache()`** (`setuppluginhookhotreload-2.ts:268-270`) — clears the memoized `rB` cache and resets global hook state via `gh1()`
- **`Mr9`** / **`setupPluginHookHotReload()`** (`setuppluginhookhotreload-2.ts:279-296`) — subscribes to settings changes for hot reload; guarded by `Xf8` flag to prevent double-registration
- **`Dr9`** / **`resetHotReloadState()`** (`setuppluginhookhotreload-2.ts:271-273`) — resets `Xf8` and `yX1` for testing

**All 18 event types registered for plugins** (`setuppluginhookhotreload-2.ts:234-252`):
```typescript
// Initialized in both Xr9() and rB():
PreToolUse, PostToolUse, PostToolUseFailure, Notification,
UserPromptSubmit, SessionStart, SessionEnd, Stop,
SubagentStart, SubagentStop, PreCompact, PermissionRequest,
Setup, TeammateIdle, TaskCompleted, ConfigChange,
WorktreeCreate, WorktreeRemove
```

```
Session start
    |
    v
loadPluginHooks()  [rB, setuppluginhookhotreload-2.ts:314]
    |
    ├─> Hz(): get list of enabled plugins
    ├─> For each enabled plugin with hooksConfig:
    │     └─> Xr9(plugin): extract hook configuration [line 232]
    │           ├─> Initialize empty map with all 18 event types
    │           ├─> For each [event, groups] in hooksConfig:
    │           │     For each group with hooks.length > 0:
    │           │       push { matcher, hooks, pluginRoot: plugin.path,
    │           │               pluginName: plugin.name, pluginId: plugin.source }
    │           └─> Return completed map
    ├─> Merge all plugin hooks into master event-keyed map
    ├─> zA6(q): register the hooks in the global hook registry
    └─> Log: "Registered N hooks from M plugins"
```

### Hot Reload

`setupPluginHookHotReload()` subscribes to policy settings changes:

```typescript
_O.subscribe((A) => {
  if (A === "policySettings") {
    let q = BY7();  // serialize current enabledPlugins list
    if (q === yX1) {
      // enabledPlugins unchanged, skip reload
      return;
    }
    yX1 = q;
    // reload: clear cache, reload all plugin hooks
    Ik();
    CP6();  // clearPluginHookCache
    rB();   // loadPluginHooks
  }
});
```

**What triggers hot reload:**
- Policy settings change (which is how plugin enable/disable is managed)
- Only if `enabledPlugins` actually changed (prevents spurious reloads)

**What hot reload does:**
1. Resets the hook state (`Ik()`)
2. Clears the plugin hook cache (`clearPluginHookCache()`)
3. Calls `loadPluginHooks()` to re-read all plugin hook configurations

This means enabling or disabling a plugin takes effect immediately without restarting Claude Code.

### Plugin Hook Restrictions

- Plugin hooks **cannot** be removed through user `settings.json` — you must disable the plugin
- When `allowManagedHooksOnly` is active, user hooks are skipped but plugin hooks (coming from `policySettings`) still run
- Hot reload only triggers on `policySettings` changes, not on direct settings.json edits

---

## 11. Hook Policies and Disabling

**Source functions** (all in `hasworktreecreatehook-1.ts`):
- **`Kn6()`** — checks the `"disableAllHooks"` managed setting. Returns `true` if hooks are globally disabled. Called at `wx()` line ~650 and `mN6()` line ~1159.
- **`RL1()`** (lines 72-74) — workspace trust check:
  ```typescript
  function RL1() {
    if (!!b4()) return false;  // b4() = some bypass/override condition
    return !vw();              // vw() = workspace trust accepted
  }
  ```
  Returns `true` (skip hooks) when workspace trust has NOT been accepted AND `b4()` bypass is not active. Log message: `"Skipping [event] hook execution - workspace trust not accepted"`
- **`FI()`** — checks `allowManagedHooksOnly` policy. Returns `true` when organization policy restricts hook execution to managed (plugin/policy) hooks only. Used in `wOz()` to skip user/workspace hooks.

### Global Disable

**`disableAllHooks`** (policy setting):
```json
{ "disableAllHooks": true }
```
Disables ALL hook execution globally. Checked by `Kn6()` (`hasworktreecreatehook-1.ts`) at the top of `wx()` and `mN6()`.

**`CLAUDE_CODE_SIMPLE`** (environment variable):
If set, disables all hooks. Checked via `_1(process.env.CLAUDE_CODE_SIMPLE)`.

### Managed Hooks Only

**`allowManagedHooksOnly`** (policy setting):
```json
{ "allowManagedHooksOnly": true }
```
When set, checked by `FI()`. Effect:
- Plugin hooks that have `"pluginRoot" in G` (i.e., come from plugins/policy) are kept
- Non-plugin hooks from user/project/local settings are SKIPPED

From source code:
```typescript
if (z && "pluginRoot" in H) continue;  // z = FI() = managedOnly
// H is a non-plugin hook, skip it
```

### Per-Event Disable

**`disableHooksFor`** (policy setting):
```json
{ "disableHooksFor": ["PreToolUse", "PostToolUse"] }
```
Disables hooks for specific events only. Others continue to run.

### Workspace Trust

Hooks also won't run if workspace trust hasn't been accepted. Checked by `RL1()`:
```typescript
function RL1() {
  if (!!b4()) return false;  // b4() = some bypass condition
  return !vw();              // vw() = workspace trust accepted
}
```
If workspace trust is not accepted: `"Skipping [event] hook execution - workspace trust not accepted"`

### Policy Settings Override for ConfigChange

Even with exit 2, ConfigChange hooks cannot block `policy_settings` changes:
```typescript
if (A === "policy_settings") return z.map((w) => ({ ...w, blocked: false }));
```
This is hardcoded — policy settings always apply regardless of hook votes.

### Summary of Conditions That Skip Hooks

1. `disableAllHooks` policy = true
2. `CLAUDE_CODE_SIMPLE` env var = set
3. Workspace trust not accepted
4. No matching hooks for this event + matcher
5. `allowManagedHooksOnly` + hook is from user/project settings
6. Event is in `disableHooksFor` list
7. Hook type is `http` (currently not executed, silently skipped)
8. Hook type is `http` for `SessionStart` or `Setup` (filtered before execution)

---

## 12. Error Handling

### EPIPE Error

Occurs when a hook process closes its stdin before the full JSON input is written (the hook command exits very quickly). Source: `hasworktreecreatehook-1.ts` lines 362–367.

```typescript
if (D6.code === "EPIPE") {
  R("EPIPE error while writing to hook stdin (hook command likely closed early)");
  let q6 = "Hook command closed stdin before hook input was fully written (EPIPE)";
  return { stdout: "", stderr: q6, output: q6, status: 1 };
}
```

The log message is emitted at level `"info"` via `R()`. The result is propagated back to `wx()` as a non-zero exit, which yields `outcome: "non_blocking_error"`. The operation continues.

### ABORT_ERR / Timeout

When the hook times out (or the parent AbortSignal fires). Source: `hasworktreecreatehook-1.ts` lines 368–374.

```typescript
else if (D6.code === "ABORT_ERR")
  return {
    stdout: "",
    stderr: "Hook cancelled",
    output: "Hook cancelled",
    status: 1,
    aborted: true,
  };
```

Result: `outcome = "cancelled"`. Non-blocking. A `hook_cancelled` message type is yielded by `wx()`. `wx()` checks `l.aborted` after awaiting `CL1()` and emits `hook_cancelled` instead of processing stdout.

**Timeout enforcement mechanism** (source: `hasworktreecreatehook-1.ts` line 303):

For each individual hook in the `wx()` loop:
```typescript
let S = V.timeout ? V.timeout * 1000 : z,         // per-hook timeout (ms) or session default
  { signal: m, cleanup: h } = yE(AbortSignal.timeout(S), Y),  // merge with parent AbortSignal
```

- `V.timeout`: per-hook `timeout` field from hook config (in seconds, multiplied by 1000)
- `z`: the session-level `timeoutMs` passed to `wx()` (default `IX` = 60000 ms)
- `yE()`: combines the per-hook `AbortSignal.timeout(S)` with the parent abort signal `Y`, first-to-fire wins
- `CL1()` receives the merged signal. When it fires (either timeout or parent abort), `CL1()` throws with `code: "ABORT_ERR"`
- Inside `CL1()` at line 303: `v = q51(N, z, P, V)` — `q51` creates a subprocess shell command wrapper/tracker with the process ID, used for async hook management

### JSON Validation Error

When a hook exits 0 but its stdout is not valid JSON (or fails the Zod schema validation `vL1.safeParse()`):

```typescript
if (J6) {
  // yield hook_non_blocking_error with the validation error details
  yield { message: ..., outcome: "non_blocking_error", hook: V };
  return;
}
```

The full validation error message includes:
- What failed in the JSON
- The expected schema (shown as a JSON comment structure)
- The actual output received

This is logged as `"non_blocking_error"` — the operation continues.

### Generic Execution Errors

If `CL1()` (the subprocess execution function) throws for any reason other than EPIPE or ABORT_ERR. Source: `hasworktreecreatehook-1.ts` lines 375–377.

```typescript
else {
  let Y6 = `Error occurred while executing hook command: ${X6 instanceof Error ? X6.message : String(X6)}`;
  return { stdout: "", stderr: Y6, output: Y6, status: 1 };
}
```

The error is stringified via `X6 instanceof Error ? X6.message : String(X6)`. Result: status 1 = non-blocking error. Propagated through `wx()` as `outcome: "non_blocking_error"` with a `hook_non_blocking_error` message type.

### Hook Input Serialization Failure

If `JSON.stringify(hookInput)` throws:
```typescript
yield {
  message: { type: "hook_error_during_execution", content: "Failed to prepare hook input: ..." },
  outcome: "non_blocking_error",
  hook: V,
};
return;
```

The hook is skipped with a non-blocking error.

### SessionEnd Hook Failures

SessionEnd uses `mN6()` which has different error reporting:
```typescript
for (let H of $)
  if (!H.succeeded && H.output)
    process.stderr.write(`SessionEnd hook [${H.command}] failed: ${H.output}\n`);
```

Errors go directly to process stderr (terminal output), not to the Claude transcript. No exceptions are thrown.

### WorktreeCreate Hook Failure

If all WorktreeCreate hooks fail or return empty stdout:
```typescript
throw Error(`WorktreeCreate hook failed: ${errorMessages.join("; ") || "no successful output"}`);
```

This IS a hard failure — it throws and the worktree creation fails entirely.

### Error Outcomes Summary

| Error Type | Outcome | User Visible | Model Visible | Blocks Operation |
|------------|---------|--------------|---------------|------------------|
| EPIPE | `non_blocking_error` | Stderr to user | No | No |
| Timeout | `cancelled` | No | No | No |
| JSON validation | `non_blocking_error` | Via hook error | No | No |
| Exit 2 | `blocking` | Stderr to model | Yes | Yes (for blocking events) |
| Other non-zero | `non_blocking_error` | Stderr to user | No | No |
| Input serialize fail | `non_blocking_error` | Via hook error | No | No |
| WorktreeCreate fail | throws | Error message | N/A | Yes |

---

## 13. Appendix: hookSpecificOutput by Event

Quick lookup: which `hookSpecificOutput` fields are available for each event.

| Event | `hookEventName` | `permissionDecision` | `permissionDecisionReason` | `updatedInput` | `additionalContext` | `updatedMCPToolOutput` | `decision` |
|-------|----------------|---------------------|---------------------------|----------------|--------------------|-----------------------|------------|
| PreToolUse | `"PreToolUse"` | allow/deny/ask | string | object | string | — | — |
| PostToolUse | `"PostToolUse"` | — | — | — | string | string | — |
| PostToolUseFailure | `"PostToolUseFailure"` | — | — | — | string | — | — |
| Notification | `"Notification"` | — | — | — | string | — | — |
| PermissionRequest | `"PermissionRequest"` | — | — | — | — | — | `{behavior, updatedInput}` |
| UserPromptSubmit | `"UserPromptSubmit"` | — | — | — | **required** | — | — |
| SessionStart | `"SessionStart"` | — | — | — | string | — | — |
| Setup | `"Setup"` | — | — | — | string | — | — |
| SubagentStart | `"SubagentStart"` | — | — | — | string | — | — |

Source: `tool-1.ts` lines 10610–10657 (Zod union `vL1`). The following events do NOT have a `hookSpecificOutput` variant in the Zod schema and their behavior is controlled entirely by exit code: SessionEnd, Stop, SubagentStop, PreCompact, TeammateIdle, TaskCompleted, ConfigChange, WorktreeCreate, WorktreeRemove.

> **Note on TeammateIdle and TaskCompleted:** These events have event builders (`cm8` and `Al6` in `hasworktreecreatehook-1.ts` lines 1390–1430) but no `hookSpecificOutput` handling in `mPq()`. This is a genuine source stub — no `case` exists for them in the `mPq` switch at lines 213–245. Their hook inputs do include extra fields (`teammate_name`, `team_name` for TeammateIdle; `task_id`, `task_subject`, `task_description`, `teammate_name`, `team_name` for TaskCompleted) but no structured output parsing is implemented.

### Base Input Fields (all events)

Every hook event receives these fields in the JSON written to stdin. Built by `EO()` function at `hasworktreecreatehook-1.ts` lines 76–84.

| Field | Type | Notes | Source |
|-------|------|-------|--------|
| `session_id` | string | UUID v4 generated via `crypto.randomUUID()` (`RE()`). Generated fresh per session start; consistent within a session | `EO()` line 77: `g1()` or passed parameter |
| `transcript_path` | string | `CH(session_id)` → `{session_dir}/{session_id}.jsonl`. Full conversation history. Hooks can read this for context | `EO()` line 78 |
| `cwd` | string | `E1()` — current working directory at hook execution time. May differ from parent session CWD for subagents | `EO()` line 79 |
| `permission_mode` | string | One of: `"default"`, `"plan"`, `"bypassPermissions"`, `"autoAllow"` | `EO()` line 80 |
| `hook_event_name` | string | The event name (e.g., `"PreToolUse"`, `"SessionStart"`). Added by each event builder, not `EO()` | per-event builder |

### Top-level JSON fields (all events)

These top-level fields can be returned in stdout JSON regardless of event:

| Field | Type | Effect |
|-------|------|--------|
| `continue` | boolean | If `false`, sets `preventContinuation: true` (causes Claude to stop). Pairs with `stopReason` |
| `stopReason` | string | Reason string when `continue: false`. Shown to user |
| `suppressOutput` | boolean | If `true`, suppresses hook stdout from transcript display |
| `decision` | `"approve"` \| `"block"` | Permission decision (legacy; prefer `hookSpecificOutput.permissionDecision`) |
| `reason` | string | Reason for decision |
| `systemMessage` | string | System message to show as warning in UI |
| `statusMessage` | string | Text shown in terminal spinner during hook execution (overrides hook-level `statusMessage` config) |
| `permissionDecision` | `"allow"` \| `"deny"` \| `"ask"` | Permission decision at top level (also processed) |
| `hookSpecificOutput` | object | Event-specific output (see table above) |

### hookSpecificOutput field details

**`permissionDecision`** (PreToolUse only)
- `"allow"`: Tool proceeds without user confirmation
- `"deny"`: Tool is blocked; `permissionDecisionReason` (or `reason`) is shown as blocking error
- `"ask"`: Show permission dialog to user; decision pending

**`updatedInput`** (PreToolUse, PermissionRequest)
- Replaces the tool's input entirely before execution
- For PreToolUse: applied when permissionDecision is `"allow"` or `"ask"` (or independently)
- For PermissionRequest: applied only when `behavior: "allow"`
- Object must match the tool's expected input schema

**`additionalContext`** (PreToolUse, PostToolUse, PostToolUseFailure, Notification, UserPromptSubmit, SessionStart, Setup, SubagentStart)
- String text injected into the model's context
- For UserPromptSubmit: REQUIRED field (plain stdout also works but this is canonical)
- Multiple hooks providing context: all contexts are collected and passed as an array
- Source: `tool-1.ts` lines 10618, 10624, 10631, 10637, 10641, 10645, 10648, 10651 (each `additionalContext: B.string().optional()` in the Zod union)

**`updatedMCPToolOutput`** (PostToolUse only)
- Replaces the MCP tool's response content
- Only applies to MCP tool calls (tool names starting with `mcp__`)
- The model sees this string instead of the actual tool response

**`decision`** (PermissionRequest only)
- Object: `{ behavior: "allow" | "deny" | "ask" | "passthrough", updatedInput?: object, message?: string, interrupt?: boolean }`
- `behavior: "allow"`: Auto-approve the permission request (with optional `updatedInput` and `updatedPermissions`)
- `behavior: "deny"`: Auto-deny (optional `message` shown to user; `interrupt: true` marks as user-interrupted)
- `behavior: "ask"`: Fall through to normal user permission dialog
- `behavior: "passthrough"`: Pass without modification (source lines 1075-1076)
- Priority when multiple hooks disagree: `deny > ask > allow`

### Event-specific hook input fields

Each event extends the base input with additional fields. Source: `hasworktreecreatehook-1.ts` lines 1296–1530 (individual event builder functions).

| Event | Additional Fields | Source function |
|-------|-------------------|----------------|
| PreToolUse | `tool_name`, `tool_input`, `tool_use_id` | `bm8()` line 1296 |
| PostToolUse | `tool_name`, `tool_input`, `tool_use_id`, `tool_response` | `xm8()` line 1318 |
| PostToolUseFailure | `tool_name`, `tool_input`, `tool_use_id`, `error`, `is_interrupt` | `um8()` line 1340 |
| PermissionRequest | `tool_name`, `tool_input`, `tool_use_id`, `permission_suggestions` | separate builder |
| Notification | `message`, `title`, `notification_type` | `FE8()` line 1360 |
| UserPromptSubmit | `prompt` | `Jd8()` line 1455 |
| SessionStart | `source`, `agent_type`, `model` | `_v8()` line 1465 |
| SessionEnd | `reason` | `mN6()` caller |
| Stop | `stop_hook_active`, `last_assistant_message` | `dm8()` line 1380 (when no agent_id) |
| SubagentStart | `agent_type` | separate builder |
| SubagentStop | `stop_hook_active`, `agent_id`, `agent_transcript_path`, `agent_type`, `last_assistant_message` | `dm8()` line 1380 (when agent_id set) |
| PreCompact | `trigger` | separate builder |
| Setup | `trigger` | `$v8()` line 1480 |
| TeammateIdle | `teammate_name`, `team_name` | `cm8()` line 1390 |
| TaskCompleted | `task_id`, `task_subject`, `task_description`, `teammate_name`, `team_name` | `Al6()` line 1410 |
| ConfigChange | `source` | separate builder |
| WorktreeCreate | *(base fields only)* | `mN6()` caller |
| WorktreeRemove | *(base fields only)* | `mN6()` caller |

### Environment Variable Reference

Environment variables injected into the subprocess environment by `CL1()`. Source: `hasworktreecreatehook-1.ts` lines 284–294.

| Variable | Set When | Value | Source |
|----------|----------|-------|--------|
| `CLAUDE_PROJECT_DIR` | Always | Platform-adjusted project directory via `f$()` | line 289 |
| `CLAUDE_PLUGIN_ROOT` | Plugin hooks only | Plugin root path (`$` parameter or `H` parameter to `CL1`) | lines 290–291 |
| `CLAUDE_ENV_FILE` | `SessionStart` or `Setup` only | Env file path returned by `R$4(event, envFile)` | lines 292–293 |
| `CLAUDE_CODE_SIMPLE` | External (user-set) | Any value; presence disables ALL hook execution | checked at `wx()` entry |
| All `process.env` vars | Always | Full parent process environment spread as base | line 288 |

**`CLAUDE_CODE_SHELL_PREFIX`**: If set, the command is wrapped: `z51(process.env.CLAUDE_CODE_SHELL_PREFIX, D)` (line 287). Allows custom shell prefix injection for all hook commands.

### Hook Type Comparison

All hook types that can appear in a hook configuration. Source: `hasworktreecreatehook-1.ts` `Hd8()` and `wx()` functions; `tool-1.ts` Zod schemas.

| Type | Execution | Async Support | Works in SessionStart/Setup | Requires |
|------|-----------|---------------|----------------------------|----------|
| `command` | Subprocess via `CL1()` | Yes (via `async: true` response or config) | Yes | `command` string |
| `prompt` | LLM evaluation via `yPq()` | No | Yes | `prompt` string, requires `toolUseContext` |
| `agent` | Autonomous agent via `SPq()` | No | Yes | `prompt` function, requires `toolUseContext` + `messages` |
| `http` | Not implemented (silently skipped) | No | No (filtered out for SessionStart/Setup) | `url` |
| `function` | Internal JS function via `_Oz()` | No | Yes (but logs error if outside REPL context) | `messages` array |
| `callback` | Plugin JS callback via `$Oz()` | No | Yes | `callback` function |

**HTTP hooks** are filtered at two points (source: `hasworktreecreatehook-1.ts` lines 1495–1502 and lines 770–776):
1. For `SessionStart`/`Setup`: filtered before the hook list is returned from `Hd8()`
2. For all other events: silently skipped inside the `wx()` execution loop with log: `"skipping HTTP hook ... — HTTP hooks are not yet supported"`

### Tool-related input fields

| Field | Events | Notes |
|-------|--------|-------|
| `tool_use_id` | PreToolUse, PostToolUse, PostToolUseFailure, PermissionRequest | Format: `toulu_*`. Unique per tool call from the Claude API response |
| `tool_input` | PreToolUse, PostToolUse, PostToolUseFailure, PermissionRequest | Raw input object from the API. Can be modified via `updatedInput` in PreToolUse/PermissionRequest hooks |
| `tool_response` | PostToolUse | Tool execution output. Only present on success. Modifiable via `updatedMCPToolOutput` |
| `is_interrupt` | PostToolUseFailure | `true` when failure was caused by Ctrl+C, timeout, SIGINT, or SIGTERM |
| `error` | PostToolUseFailure | Error message string OR `{ message, code, stderr, stdout }` object |

---

*Source: Verified from Claude Code CLI v2.1.59 cli.js, prettified and converted to typescript modules.*
