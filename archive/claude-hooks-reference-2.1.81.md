# Claude Code CLI v2.1.81 — Hooks System Reference

> **Single source of truth** for the hooks system in Claude Code CLI v2.1.81.  
> All enum values, required/optional fields, and execution semantics verified against Zod schemas.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Hook Event List](#2-hook-event-list)
3. [Hook Configuration Schema](#3-hook-configuration-schema)
4. [Hook Type Schemas](#4-hook-type-schemas)
5. [Event Input Schemas](#5-event-input-schemas)
6. [Hook Output / Return Value Schemas](#6-hook-output--return-value-schemas)
7. [Execution Engine Internals](#7-execution-engine-internals)
8. [Command Hook Execution](#8-command-hook-execution)
9. [Prompt Hook Execution](#9-prompt-hook-execution)
10. [Agent Hook Execution](#10-agent-hook-execution)
11. [HTTP Hook Execution](#11-http-hook-execution)
12. [Session Hooks](#12-session-hooks)
13. [Plugin Hook Hot Reload](#13-plugin-hook-hot-reload)
14. [Security and Policy](#14-security-and-policy)
15. [Practical Examples](#15-practical-examples)
16. [Event Dispatch Reference](#16-event-dispatch-reference)
17. [Error Handling and Recovery](#17-error-handling-and-recovery)
18. [Configuration Patterns and Anti-Patterns](#18-configuration-patterns-and-anti-patterns)
19. [Hook Execution Timing Diagrams](#19-hook-execution-timing-diagrams)
20. [Appendices](#appendix-a-full-event--input-field-map)

---

## 1. Overview

The hooks system in Claude Code CLI v2.1.81 provides a mechanism for intercepting, observing, and influencing the agent lifecycle at well-defined points. Hooks can execute shell commands, call external HTTP endpoints, trigger sub-prompts, or launch sub-agents.

### Architecture

```
User / Agent Action
       |
       v
  Hook Dispatcher (xx / ux)
       |
       +-- Matcher: does this hook apply?
       |     +-- * / falsy  -> always matches
       |     +-- literal(s) -> pipe-separated string match
       |     +-- regex      -> RegExp test
       |
       +-- Execution by type:
       |     +-- command  -> shell subprocess (bash / powershell)
       |     +-- prompt   -> inline LLM call (no tool use)
       |     +-- agent    -> sub-agent with dontAsk mode
       |     +-- http     -> HTTP POST to external URL
       |
       +-- Output processing
             +-- exit code 2  -> blocking error (command hooks)
             +-- decision     -> allow / deny / ask / passthrough
             +-- additionalContext -> injected into next prompt
```

### Design Principles

The hooks system is designed around several core principles:

1. **Non-interference by default:** Hooks that return nothing / empty output produce no side effects on the agent's behavior. The baseline decision is always `passthrough`.

2. **Composability:** Multiple hooks can be configured for the same event. Their decisions are merged using a defined precedence rule.

3. **Separation of concerns:** Hook configuration (what to run) is separate from hook execution (how it runs). The `xx`/`ux` dispatchers handle all execution; hook configs are declarative.

4. **Security by design:** HTTP hooks have IP blocking, URL allowlisting, and env var allowlisting built into the execution path, not bolted on after.

5. **Async-capable:** Long-running observability hooks can run in the background without blocking the agent with `async: true`.

### Key Constants

| Constant | Value | Source |
|----------|-------|--------|
| `UO` (default timeout) | `600000` ms (10 min) | `hasworktreecreatehook-1.ts:2373` |
| Prompt hook default timeout | `30000` ms (30 s) | `tool-2.ts:7667` |
| Agent hook default timeout | `60000` ms (60 s) | `tool-2.ts:7830` |
| Agent hook max turns | `50` | `tool-2.ts:7830` |
| Prompt hook model | `tH()` (configured model) | `tool-2.ts:7672` |
| Agent hook model | `tH()` (configured model) | `tool-2.ts:7838` |

### Source Files

| File | Contents |
|------|----------|
| `src/tools/hasworktreecreatehook-1.ts` | Execution engine, dispatcher, all `execute*Hooks` functions |
| `src/tools/writetomailbox-1.ts` | Event input Zod schemas, base schema, error enum |
| `src/tools/tool-1.ts` | Output schema, session hook helpers |
| `src/tools/tool-2.ts` | Prompt and agent hook execution |
| `src/core/ok-3.ts` | Policy logic, plugin var substitution |
| `src/core/auth-1.ts` | Hook type Zod schemas (command/prompt/agent/http) |
| `src/core/u48-1.ts` | Top-level hooks config schema |
| `src/ui/markdown-1.ts` | HTTP hook execution, IP blocking, env var interpolation |
| `src/tools/setuppluginhookhotreload-3.ts` | Plugin hook hot reload |

---

## 2. Hook Event List

All 23 events. Verified against the `Aq_` array at `writetomailbox-1.ts:243-267`.

| # | Event Name | Category | Direction |
|---|-----------|----------|-----------|
| 1 | `PreToolUse` | Tool lifecycle | Before tool call |
| 2 | `PostToolUse` | Tool lifecycle | After successful tool call |
| 3 | `PostToolUseFailure` | Tool lifecycle | After failed tool call |
| 4 | `Notification` | Agent comms | Agent notification |
| 5 | `UserPromptSubmit` | User input | User submits prompt |
| 6 | `SessionStart` | Session lifecycle | Session begins |
| 7 | `SessionEnd` | Session lifecycle | Session ends |
| 8 | `Stop` | Agent control | Agent stops |
| 9 | `StopFailure` | Agent control | Agent stop fails |
| 10 | `SubagentStart` | Subagent lifecycle | Subagent begins |
| 11 | `SubagentStop` | Subagent lifecycle | Subagent ends |
| 12 | `PreCompact` | Memory | Before context compaction |
| 13 | `PostCompact` | Memory | After context compaction |
| 14 | `PermissionRequest` | Permissions | Permission check |
| 15 | `Setup` | Initialization | Tool setup |
| 16 | `TeammateIdle` | Collaboration | Teammate becomes idle |
| 17 | `TaskCompleted` | Task lifecycle | Task finishes |
| 18 | `Elicitation` | User interaction | Elicitation request |
| 19 | `ElicitationResult` | User interaction | Elicitation response |
| 20 | `ConfigChange` | Configuration | Config modified |
| 21 | `WorktreeCreate` | Git | Worktree created |
| 22 | `WorktreeRemove` | Git | Worktree removed |
| 23 | `InstructionsLoaded` | Instructions | Instructions loaded |

### HTTP Hook Support by Event

HTTP hooks are **not** supported for the following events (verified `hasworktreecreatehook-1.ts:830-840`):
- `SessionStart`
- `Setup`

All other 21 events support HTTP hooks.

### Event Categories Explained

#### Tool Lifecycle Events

These three events bracket every tool invocation:

- **`PreToolUse`** — Fires before any tool executes. Can deny the tool call, modify its input via `hookSpecificOutput.modified_tool_input`, or inject context. This is the primary event for access control hooks.

- **`PostToolUse`** — Fires after a tool completes successfully. Receives both the tool input and response. Can modify the response via `hookSpecificOutput.modified_tool_response`. Used for audit logging and response transformation.

- **`PostToolUseFailure`** — Fires when a tool throws an error. Receives the error message. Used for error monitoring and alerting.

#### Session Lifecycle Events

- **`SessionStart`** — Fires when a new Claude Code session begins. HTTP hooks are not supported here (early in lifecycle, no HTTP context yet). Suitable for environment initialization, workspace setup.

- **`SessionEnd`** — Fires when a session terminates. Used for cleanup, session telemetry, persisting session summaries.

#### Agent Control Events

- **`Stop`** — Fires when the agent decides to stop. Hook can return `hookSpecificOutput.continue: true` to prevent stopping and resume the agent.

- **`StopFailure`** — Fires when the agent fails to stop cleanly. Receives the error. Used for diagnostics.

#### Subagent Events

- **`SubagentStart`** — Fires when a sub-agent (spawned by the main agent) begins. Receives `subagent_id`.

- **`SubagentStop`** — Fires when a sub-agent terminates. Receives `subagent_id`.

#### Memory Events

- **`PreCompact`** — Fires before context compaction begins. Receives `trigger` (what caused compaction). Hook can prepare or snapshot context.

- **`PostCompact`** — Fires after compaction completes. Receives `summary` (the compaction summary).

#### User Interaction Events

- **`UserPromptSubmit`** — Fires when the user submits a prompt. Can modify the prompt via `hookSpecificOutput.modified_prompt`. Used for prompt augmentation, content filtering.

- **`Notification`** — Fires when the agent emits a notification. Receives `message`.

- **`Elicitation`** — Fires when an elicitation (structured user input request) is created.

- **`ElicitationResult`** — Fires when the user responds to an elicitation.

#### Permission Event

- **`PermissionRequest`** — Fires for permission checks. Hook decision directly determines the permission outcome. The only event where the hook decision has immediate security implications.

#### Configuration Event

- **`ConfigChange`** — Fires when any configuration key changes. Receives `key`, `old_value`, `new_value`.

#### Git Events

- **`WorktreeCreate`** — Fires when a git worktree is created. Receives `worktree_path` and `branch`.

- **`WorktreeRemove`** — Fires when a git worktree is removed. Receives `worktree_path`.

#### Initialization Events

- **`Setup`** — Fires during tool initialization. HTTP hooks are not supported here. Used for setup validation and environment checks.

- **`InstructionsLoaded`** — Fires when instructions (CLAUDE.md or similar) are loaded. Receives `instructions_path` and `source`.

#### Collaboration Events

- **`TeammateIdle`** — Fires when a teammate becomes idle in a multi-agent collaboration. Receives `teammate_id`.

- **`TaskCompleted`** — Fires when a task completes. Receives `result`.

---

## 3. Hook Configuration Schema

**Source:** `src/core/u48-1.ts:1-38`

### Top-Level Shape

```typescript
// JL -- full hooks config
// partialRecord(enum(Qu), array(O0A()))
// Maps each HookEvent -> array of matcher entries
type HooksConfig = Partial<Record<HookEvent, MatcherEntry[]>>
```

### Matcher Entry Schema (`O0A`)

**Source:** `src/core/u48-1.ts:26-36`

```typescript
interface MatcherEntry {
  matcher?: string  // optional -- if omitted, matches all
  hooks: HookSpec[] // one or more hook specs
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `matcher` | `string` | No | Pattern to match against tool name, URL, etc. |
| `hooks` | `HookSpec[]` | Yes | Array of hook definitions |

### Matcher Logic (`Bn_`)

**Source:** `src/tools/hasworktreecreatehook-1.ts:649-667`

The matcher string is evaluated as follows:

| Matcher Value | Behavior |
|---------------|----------|
| Falsy / omitted | Matches everything (`*`) |
| `"*"` | Matches everything |
| Pipe-separated string | `"Bash|Read|Write"` -- matches any listed literal |
| Regex pattern | Tested via `RegExp.test()` |

### Matcher Examples

```json
// Match all tools
{ "hooks": [...] }

// Match specific tool by name
{ "matcher": "Bash", "hooks": [...] }

// Match multiple tools
{ "matcher": "Bash|Write|Edit", "hooks": [...] }

// Match by regex (any tool ending in File)
{ "matcher": "File$", "hooks": [...] }

// Match all (explicit wildcard)
{ "matcher": "*", "hooks": [...] }
```

### Full Configuration Shape in JSON

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'About to run Bash'"
          }
        ]
      },
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "http",
            "url": "https://audit.example.com/pre-write"
          }
        ]
      },
      {
        "hooks": [
          {
            "type": "command",
            "command": "/usr/local/bin/log-all-tools.sh",
            "async": true
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/usr/local/bin/audit-tool.sh"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo Session started: $CLAUDE_SESSION_ID",
            "once": true
          }
        ]
      }
    ]
  }
}
```

### Configuration in `settings.json` vs `settings.local.json`

Hook configuration can appear in:
- **`settings.json`** (project-level, committed to repo)
- **`settings.local.json`** (project-level, not committed, takes precedence)
- **User-level settings** (`~/.claude/settings.json`)

Hooks from all levels are merged. More specific scopes take precedence over broader ones.

---

## 4. Hook Type Schemas

All four user-configurable hook types. The schemas are constructed by `wDK()` factory function.

**Source:** `src/core/auth-1.ts:500-636`

### 4.1 Command Hook (`BashCommandHookSchema`)

```typescript
interface CommandHook {
  type: 'command'
  command: string          // Required
  shell?: 'bash' | 'powershell'  // Optional, default: 'bash'
  timeout?: number         // Optional, ms, default: UO (600000)
  statusMessage?: string   // Optional, displayed during execution
  once?: boolean           // Optional, run only once per session
  async?: boolean          // Optional, background execution
  asyncRewake?: boolean    // Optional, rewake agent when done
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | `'command'` | Yes | -- | Discriminant |
| `command` | `string` | Yes | -- | Shell command to execute |
| `shell` | `'bash'|'powershell'` | No | `'bash'` | Shell interpreter |
| `timeout` | `number` | No | `600000` | Execution timeout (ms) |
| `statusMessage` | `string` | No | -- | UI status text during run |
| `once` | `boolean` | No | `false` | Run at most once per session |
| `async` | `boolean` | No | `false` | Fire and forget |
| `asyncRewake` | `boolean` | No | `false` | Rewake agent after async completes |

### 4.2 Prompt Hook (`PromptHookSchema`)

```typescript
interface PromptHook {
  type: 'prompt'
  prompt: string           // Required
  timeout?: number         // Optional, ms, default: 30000
  model?: string           // Optional, default: tH() (configured model)
  statusMessage?: string   // Optional
  once?: boolean           // Optional
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | `'prompt'` | Yes | -- | Discriminant |
| `prompt` | `string` | Yes | -- | Prompt text for inline LLM call |
| `timeout` | `number` | No | `30000` | LLM call timeout (ms) |
| `model` | `string` | No | `tH()` | Model ID override |
| `statusMessage` | `string` | No | -- | UI status text during run |
| `once` | `boolean` | No | `false` | Run at most once per session |

**Note:** Thinking is disabled for prompt hooks (`tool-2.ts:7672`).

### 4.3 Agent Hook (`AgentHookSchema`)

```typescript
interface AgentHook {
  type: 'agent'
  prompt: string           // Required
  timeout?: number         // Optional, ms, default: 60000
  model?: string           // Optional, default: tH()
  statusMessage?: string   // Optional
  once?: boolean           // Optional
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | `'agent'` | Yes | -- | Discriminant |
| `prompt` | `string` | Yes | -- | Sub-agent system prompt |
| `timeout` | `number` | No | `60000` | Agent timeout (ms) |
| `model` | `string` | No | `tH()` | Model ID override |
| `statusMessage` | `string` | No | -- | UI status text during run |
| `once` | `boolean` | No | `false` | Run at most once per session |

**Notes:**
- Agent hooks run with `dontAsk` permission mode (`tool-2.ts:7830`)
- Maximum 50 turns (`tool-2.ts:7830`)

### 4.4 HTTP Hook (`HttpHookSchema`)

```typescript
interface HttpHook {
  type: 'http'
  url: string              // Required
  timeout?: number         // Optional, ms, default: UO (600000)
  headers?: Record<string, string>  // Optional
  allowedEnvVars?: string[]  // Optional
  statusMessage?: string   // Optional
  once?: boolean           // Optional
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | `'http'` | Yes | -- | Discriminant |
| `url` | `string` | Yes | -- | Endpoint URL (POST) |
| `timeout` | `number` | No | `600000` | Request timeout (ms) |
| `headers` | `Record<string,string>` | No | -- | Additional HTTP headers |
| `allowedEnvVars` | `string[]` | No | -- | Env vars to include in request |
| `statusMessage` | `string` | No | -- | UI status text during run |
| `once` | `boolean` | No | `false` | Run at most once per session |

**Not supported for:** `SessionStart`, `Setup` events.

### 4.5 Internal Hook Types (Non-Configurable)

These types exist in the discriminated union (`w0A` at `u48-1.ts:24`) but are not user-configurable:

| Type | Description |
|------|-------------|
| `callback` | Internal JavaScript callback function |
| `function` | Internal named function reference |

### Discriminated Union (`w0A`)

**Source:** `src/core/u48-1.ts:24`

```typescript
// w0A -- discriminated union on "type" field
type HookSpec =
  | CommandHook    // type: 'command'
  | PromptHook     // type: 'prompt'
  | AgentHook      // type: 'agent'
  | HttpHook       // type: 'http'
  | CallbackHook   // type: 'callback' (internal)
  | FunctionHook   // type: 'function' (internal)
```

### Common Fields Across All Types

The following fields appear in all four user-configurable hook types:

| Field | Type | Default | Meaning |
|-------|------|---------|----------|
| `type` | string | -- | Discriminant, required |
| `timeout` | number | varies | Max execution time (ms) |
| `statusMessage` | string | -- | UI progress message |
| `once` | boolean | `false` | Run at most once per session |

---

## 5. Event Input Schemas

All event input schemas extend the base schema `kH`. Each event schema is verified against Zod definitions in `src/tools/writetomailbox-1.ts:243-918`.

### 5.1 Base Schema (`kH`)

**Source:** `writetomailbox-1.ts:269-288`

```typescript
interface BaseHookInput {
  session_id: string        // Required
  transcript_path: string   // Required
  cwd: string               // Required
  permission_mode?: string  // Optional
  agent_id?: string         // Optional
  agent_type?: string       // Optional
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `session_id` | `string` | Yes | Unique session identifier |
| `transcript_path` | `string` | Yes | Path to transcript file |
| `cwd` | `string` | Yes | Current working directory |
| `permission_mode` | `string` | No | Current permission mode |
| `agent_id` | `string` | No | Agent identifier (subagents) |
| `agent_type` | `string` | No | Agent type identifier |

### 5.2 PreToolUse

**Source:** `writetomailbox-1.ts`

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `tool_name` | `string` | Yes | Name of the tool being called |
| `tool_input` | `object` | Yes | Input parameters for the tool |

**Example stdin:**
```json
{
  "session_id": "abc123",
  "transcript_path": "/tmp/transcript.json",
  "cwd": "/home/user/project",
  "permission_mode": "default",
  "tool_name": "Bash",
  "tool_input": {
    "command": "ls -la"
  }
}
```

### 5.3 PostToolUse

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `tool_name` | `string` | Yes | Name of the tool that ran |
| `tool_input` | `object` | Yes | Input parameters used |
| `tool_response` | `unknown` | Yes | Response from the tool |

**Example stdin:**
```json
{
  "session_id": "abc123",
  "transcript_path": "/tmp/transcript.json",
  "cwd": "/home/user/project",
  "tool_name": "Bash",
  "tool_input": { "command": "ls -la" },
  "tool_response": { "stdout": "file1.txt\nfile2.txt", "exit_code": 0 }
}
```

### 5.4 PostToolUseFailure

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `tool_name` | `string` | Yes | Name of the tool that failed |
| `tool_input` | `object` | Yes | Input parameters used |
| `error` | `string` | Yes | Error message from tool failure |

### 5.5 Notification

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `message` | `string` | Yes | Notification message content |

### 5.6 UserPromptSubmit

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `prompt` | `string` | Yes | User-submitted prompt text |

**Example stdin:**
```json
{
  "session_id": "abc123",
  "transcript_path": "/tmp/transcript.json",
  "cwd": "/home/user/project",
  "prompt": "Write a function that sorts a list"
}
```

### 5.7 SessionStart

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |

No additional fields beyond base schema.

### 5.8 SessionEnd

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |

No additional fields beyond base schema.

### 5.9 Stop

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `stop_reason` | `string` | Yes | Reason the agent stopped |

### 5.10 StopFailure

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `error` | `string` | Yes | Error that caused stop failure |

### 5.11 SubagentStart

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `subagent_id` | `string` | Yes | Identifier of the starting subagent |

### 5.12 SubagentStop

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `subagent_id` | `string` | Yes | Identifier of the stopping subagent |

### 5.13 PreCompact

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `trigger` | `string` | Yes | What triggered compaction |

### 5.14 PostCompact

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `summary` | `string` | Yes | Compaction summary |

### 5.15 PermissionRequest

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `tool_name` | `string` | Yes | Tool requesting permission |
| `tool_input` | `object` | Yes | Tool input for the permission request |
| `request_id` | `string` | Yes | Unique permission request identifier |

### 5.16 Setup

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |

No additional fields beyond base schema.

### 5.17 TeammateIdle

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `teammate_id` | `string` | Yes | Identifier of the idle teammate |

### 5.18 TaskCompleted

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `result` | `unknown` | Yes | Task completion result |

### 5.19 Elicitation

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `message` | `string` | Yes | Elicitation prompt message |
| `request_id` | `string` | Yes | Unique elicitation request ID |

### 5.20 ElicitationResult

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `request_id` | `string` | Yes | Matching elicitation request ID |
| `result` | `unknown` | Yes | User's elicitation response |

### 5.21 ConfigChange

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `key` | `string` | Yes | Configuration key that changed |
| `old_value` | `unknown` | Yes | Previous value |
| `new_value` | `unknown` | Yes | New value |

### 5.22 WorktreeCreate

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `worktree_path` | `string` | Yes | Path to the created worktree |
| `branch` | `string` | Yes | Branch name for the worktree |

### 5.23 WorktreeRemove

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `worktree_path` | `string` | Yes | Path to the removed worktree |

### 5.24 InstructionsLoaded

Extends base schema.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| *(base fields)* | -- | Yes | All base schema fields |
| `instructions_path` | `string` | Yes | Path to the loaded instructions file |
| `source` | `string` | Yes | Source of the instructions |

---

## 6. Hook Output / Return Value Schemas

**Source:** `src/tools/tool-1.ts:1520-1610`, `src/tools/writetomailbox-1.ts:586-706`

### 6.1 Output Schema (`my9`)

**Source:** `tool-1.ts:1520-1610`

```typescript
interface HookOutput {
  // Decision fields
  decision?: 'allow' | 'deny' | 'ask' | 'passthrough'
  reason?: string

  // Context injection
  additionalContext?: string

  // Output suppression
  suppressOutput?: boolean

  // Hook-specific output (varies by event)
  hookSpecificOutput?: HookSpecificOutput
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `decision` | `'allow'|'deny'|'ask'|'passthrough'` | No | Permission decision |
| `reason` | `string` | No | Human-readable reason for decision |
| `additionalContext` | `string` | No | Text injected into agent's next prompt |
| `suppressOutput` | `boolean` | No | Suppress tool output display |
| `hookSpecificOutput` | `object` | No | Event-specific output data |

### 6.2 Decision Values

| Decision | Meaning | Precedence |
|----------|---------|------------|
| `deny` | Block the action | Highest (1) |
| `ask` | Prompt user for confirmation | 2 |
| `allow` | Permit the action | 3 |
| `passthrough` | No opinion / defer | Lowest (4) |

**Permission precedence rule:** When multiple hooks run for the same event, the most restrictive decision wins: `deny > ask > allow > passthrough`.

### 6.3 Async Hook Union (`ff6`)

**Source:** `tool-1.ts:1611-1616`

```typescript
// ff6 -- async union
type AsyncHookResult = HookOutput | Promise<HookOutput>
```

### 6.4 Hook-Specific Output Variants (`pq_` / `hq_`)

**Source:** `writetomailbox-1.ts:586-706`

Each event may define a `hookSpecificOutput` schema variant. The following events have defined hook-specific outputs:

#### PreToolUse

```typescript
interface PreToolUseSpecificOutput {
  modified_tool_input?: object  // Override tool input
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `modified_tool_input` | `object` | No | Replacement input for the tool |

**Use case:** Sanitize or augment tool inputs before execution. E.g., strip dangerous flags from a Bash command.

#### PostToolUse

```typescript
interface PostToolUseSpecificOutput {
  modified_tool_response?: unknown  // Override tool response
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `modified_tool_response` | `unknown` | No | Replacement response for the tool |

**Use case:** Filter sensitive data from tool responses before they reach the agent.

#### UserPromptSubmit

```typescript
interface UserPromptSubmitSpecificOutput {
  modified_prompt?: string  // Override the user prompt
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `modified_prompt` | `string` | No | Replacement for the user's prompt |

**Use case:** Augment user prompts with additional context, translate languages, apply content policies.

#### PermissionRequest

```typescript
interface PermissionRequestSpecificOutput {
  decision: 'allow' | 'deny' | 'ask'  // Required for permission events
  reason?: string
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `decision` | `'allow'|'deny'|'ask'` | Yes | Permission outcome |
| `reason` | `string` | No | Explanation shown to user |

**Note:** `passthrough` is not a valid decision for `PermissionRequest` hooks in the `hookSpecificOutput`. Use `ask` as the minimum, or omit for passthrough behavior.

#### Stop

```typescript
interface StopSpecificOutput {
  continue?: boolean  // True to prevent stopping
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `continue` | `boolean` | No | Set `true` to prevent agent from stopping |

**Use case:** Implement "keep-alive" logic that resumes the agent with additional tasks when it would otherwise stop.

### 6.5 Error Enum (`BC1`)

**Source:** `writetomailbox-1.ts:908-918`

```typescript
enum HookErrorType {
  authentication_failed = 'authentication_failed',
  billing_error = 'billing_error',
  rate_limit = 'rate_limit',
  invalid_request = 'invalid_request',
  server_error = 'server_error',
  unknown = 'unknown',
  max_output_tokens = 'max_output_tokens',
}
```

| Value | Description |
|-------|-------------|
| `authentication_failed` | Auth credentials invalid or expired |
| `billing_error` | Billing/payment issue |
| `rate_limit` | API rate limit exceeded |
| `invalid_request` | Malformed request |
| `server_error` | Server-side failure |
| `unknown` | Unclassified error |
| `max_output_tokens` | Output token limit reached |

### 6.6 Output Examples

**Deny with reason:**
```json
{"decision": "deny", "reason": "Command contains dangerous pattern 'rm -rf'"}
```

**Allow with context:**
```json
{"decision": "allow", "additionalContext": "Note: git status shows 3 modified files"}
```

**Passthrough (empty):**
```json
{}
```

**Modify tool input:**
```json
{
  "hookSpecificOutput": {
    "modified_tool_input": {
      "command": "ls -la --color=never"
    }
  }
}
```

**Ask with reason:**
```json
{"decision": "ask", "reason": "This command will delete 15 files. Are you sure?"}
```

**Prevent stop:**
```json
{
  "hookSpecificOutput": {
    "continue": true
  },
  "additionalContext": "Please also run the tests before finishing."
}
```

---

## 7. Execution Engine Internals

**Source:** `src/tools/hasworktreecreatehook-1.ts`

### 7.1 Main Dispatcher (`xx`)

**Source:** `hasworktreecreatehook-1.ts:871-1529`

The `xx` function is the primary hook dispatcher used in REPL contexts.

```
xx(event, input, options)
  |
  +-- Policy check: ZU6() -- disableAllHooks?
  +-- Policy check: Dh() -- managedHooksOnly?
  +-- Workspace trust: eh8() -- trusted?
  |
  +-- Retrieve hooks: w44(event)
  |     +-- settings.hooks[event]
  |     +-- session hooks from appState.sessionHooks
  |     +-- plugin hooks
  |
  +-- For each MatcherEntry:
  |     +-- Bn_(matcher, input) -- does it match?
  |     +-- For each hook in entry.hooks:
  |           +-- once check -- already run?
  |           +-- async check -- background?
  |           +-- execute by type
  |
  +-- Aggregate results -> final output
```

### 7.2 Non-REPL Dispatcher (`ux`)

**Source:** `hasworktreecreatehook-1.ts:1533-1698`

The `ux` function is used outside of REPL contexts (e.g., during setup, batch operations). Same logic as `xx` but operates without interactive UI elements.

**Differences from `xx`:**
- No progress spinner / statusMessage display
- No interactive prompts for `ask` decisions
- Used by `Setup`, `SessionStart` (early lifecycle)

### 7.3 Hook Collector (`Oi1`)

**Source:** `hasworktreecreatehook-1.ts:715-851`

Collects all applicable hooks for an event from three sources, in priority order:

1. **Settings-level hooks** (`settings.hooks[event]`) — from `settings.json` / `settings.local.json`
2. **Session-level hooks** (`appState.sessionHooks[event]`) — programmatically registered
3. **Plugin hooks** (hot-loaded from enabled plugins)

Applies HTTP hook blocking for `SessionStart` and `Setup` events: any HTTP-type hooks in the collected set are filtered out for these two events.

### 7.4 Command Execution (`AS8`)

**Source:** `hasworktreecreatehook-1.ts:381-647`

Handles shell subprocess creation for command hooks. See Section 8 for full details.

### 7.5 Execute Functions

**Source:** `hasworktreecreatehook-1.ts:1700-2374`

Each event has a dedicated execute function:

| Function Pattern | Events |
|-----------------|--------|
| `executePreToolUseHooks` | `PreToolUse` |
| `executePostToolUseHooks` | `PostToolUse` |
| `executePostToolUseFailureHooks` | `PostToolUseFailure` |
| `executeNotificationHooks` | `Notification` |
| `executeUserPromptSubmitHooks` | `UserPromptSubmit` |
| `executeSessionStartHooks` | `SessionStart` |
| `executeSessionEndHooks` | `SessionEnd` |
| `executeStopHooks` | `Stop` |
| `executeStopFailureHooks` | `StopFailure` |
| `executeSubagentStartHooks` | `SubagentStart` |
| `executeSubagentStopHooks` | `SubagentStop` |
| `executePreCompactHooks` | `PreCompact` |
| `executePostCompactHooks` | `PostCompact` |
| `executePermissionRequestHooks` | `PermissionRequest` |
| `executeSetupHooks` | `Setup` |
| `executeTeammateIdleHooks` | `TeammateIdle` |
| `executeTaskCompletedHooks` | `TaskCompleted` |
| `executeElicitationHooks` | `Elicitation` |
| `executeElicitationResultHooks` | `ElicitationResult` |
| `executeConfigChangeHooks` | `ConfigChange` |
| `executeWorktreeCreateHooks` | `WorktreeCreate` |
| `executeWorktreeRemoveHooks` | `WorktreeRemove` |
| `executeInstructionsLoadedHooks` | `InstructionsLoaded` |

### 7.6 Async Hook Semantics

When `async: true` is set on a command hook:

1. The hook process is spawned
2. The main execution **does not wait** for the process to complete
3. The hook's output and exit code are **not** used for decision-making
4. If `asyncRewake: true`, a completion callback is registered that will re-trigger the agent when the process exits
5. The agent continues immediately after spawning

**Important:** `async: true` is only available on `command` hooks. `prompt`, `agent`, and `http` hooks are always synchronous.

### 7.7 `once` Semantics

When `once: true` is set:

1. A per-session record tracks which hooks have already run (keyed by hook identity)
2. On subsequent triggers for the same hook, it is skipped
3. Scope is the current session — resets when the session ends
4. Applies independently to each hook entry; `once: true` on hook A does not affect hook B

### 7.8 Decision Aggregation

When multiple hooks run for the same event, decisions are aggregated:

```
results = [hook1.decision, hook2.decision, hook3.decision]
final = most_restrictive(results)

precedence: deny(1) > ask(2) > allow(3) > passthrough(4)
```

If no hook returns a decision (or all return `passthrough`), the final result is `passthrough`.

**Example:**
- Hook 1: `allow`
- Hook 2: (no decision / passthrough)
- Hook 3: `deny`
- Result: **`deny`** (most restrictive wins)

### 7.9 `additionalContext` Aggregation

All non-empty `additionalContext` strings from all hook results are **concatenated** and injected into the agent's next prompt as a single block. The concatenation order follows hook execution order.

### 7.10 Async Generator Pattern

The dispatcher uses an async generator pattern internally, yielding progress events to the UI layer during hook execution. This enables:
- Live progress display (`statusMessage` updates)
- Cancellation support
- Streaming of hook results as they complete

---

## 8. Command Hook Execution

**Source:** `src/tools/hasworktreecreatehook-1.ts:381-647`

### 8.1 Execution Flow

```
1. Apply plugin variable substitution (wg): ${CLAUDE_PLUGIN_ROOT}
2. Serialize event input as JSON -> stdin
3. Spawn subprocess:
   - bash: bash -c "<command>"
   - powershell: powershell -Command "<command>"
4. Write JSON to stdin
5. Close stdin (signals EOF to the process)
6. Wait for process exit (up to timeout ms)
7. Collect stdout and stderr
8. On timeout: kill process, treat as non-blocking error
9. Parse stdout:
   a. If empty -> empty output (passthrough)
   b. If valid JSON -> parse as HookOutput
   c. If invalid JSON (plain text) -> wrap as additionalContext
10. Check exit code:
    - 0 -> success, use parsed output
    - 2 -> blocking error, raise stderr as error message
    - other -> non-blocking error (log, continue)
```

### 8.2 Exit Code Semantics

| Exit Code | Meaning | Behavior |
|-----------|---------|----------|
| `0` | Success | Output processed normally |
| `1` | Non-blocking error | Logged, execution continues |
| `2` | Blocking error | Error surfaced to user, hook chain may abort |
| Other | Non-blocking error | Logged, execution continues |

**Exit code 2 details:** When a command hook exits with code 2, the content of stderr is used as the error message presented to the user. This is the primary mechanism for hooks to report fatal errors that should halt processing.

### 8.3 Stdin/Stdout Contract

**Stdin:** JSON-serialized event input (the full event schema object).

**Stdout:** One of:
- Empty / whitespace only -> no output, passthrough decision
- Valid JSON object -> parsed as `HookOutput`
- Non-JSON text -> entire text wrapped as `additionalContext`

**Stderr:** Captured for error reporting. On exit code 2, stderr is the error message. On other non-zero exit codes, stderr is logged but not propagated to the user.

### 8.4 Environment Variables

Command hooks inherit the full current process environment. The following Claude-specific variables are additionally available:

| Variable | Value |
|----------|-------|
| `CLAUDE_SESSION_ID` | Current session ID |
| `CLAUDE_CWD` | Current working directory |
| `CLAUDE_PERMISSION_MODE` | Current permission mode |

### 8.5 Shell Selection

| `shell` Value | Command Template | Platform |
|---------------|------------------|----------|
| `'bash'` (default) | `bash -c "<command>"` | Unix/macOS |
| `'powershell'` | `powershell -Command "<command>"` | Windows |

**Note:** `bash` is the default regardless of platform. If you need Windows compatibility, explicitly set `"shell": "powershell"`.

### 8.6 Plugin Variable Substitution (`wg`)

**Source:** `src/core/ok-3.ts:49-57`

In plugin hook commands, `${CLAUDE_PLUGIN_ROOT}` is interpolated with the plugin's root directory path before execution.

```bash
# Before substitution:
"node \"${CLAUDE_PLUGIN_ROOT}/hooks/scripts/dist/pre-tool-use.js\""

# After substitution (example):
"node \"/home/user/.claude/plugins/my-plugin/hooks/scripts/dist/pre-tool-use.js\""
```

This substitution is performed by `wg()` before the command is passed to the shell. It only applies to `type: 'command'` hooks. Other hook types do not receive this substitution.

### 8.7 Timeout Behavior

When a command hook exceeds `timeout` ms:

1. The subprocess is killed (SIGTERM, then SIGKILL if needed)
2. The timeout is treated as a **non-blocking** error (not exit code 2)
3. The hook result is `passthrough` (no decision, no context)
4. Execution continues with the next hook

To make timeout a blocking error, the hook script itself must handle timeouts and exit with code 2.

### 8.8 Working Directory

Command hooks run with `cwd` set to the current project directory (same as the `cwd` field in the event input).

---

## 9. Prompt Hook Execution

**Source:** `src/tools/tool-2.ts:7659-7803`

### 9.1 Execution Flow

```
1. Serialize event input as JSON string
2. Construct full prompt:
   "<hook.prompt>\n\nEvent data:\n<JSON.stringify(eventInput, null, 2)>"
3. Create LLM API request:
   - model: hook.model || tH() (configured model)
   - messages: [{ role: 'user', content: constructedPrompt }]
   - thinking: { type: 'disabled' }
   - tools: [] (no tool use)
   - max_tokens: model default
4. Send request with timeout: hook.timeout || 30000 ms
5. Await response
6. Extract text from response
7. Attempt JSON.parse(text)
8. If valid JSON -> return as HookOutput
9. If invalid JSON -> wrap as additionalContext
10. On error/timeout -> return empty HookOutput (passthrough)
```

### 9.2 Constraints

| Constraint | Value | Source |
|------------|-------|--------|
| Default timeout | `30000` ms (30 s) | `tool-2.ts:7667` |
| Thinking | Disabled | `tool-2.ts:7672` |
| Tool use | Disabled | `tool-2.ts:7672` |
| Max tokens | Model default | -- |

### 9.3 Model Selection

Prompt hooks use the configured model (`tH()`) by default. Override with `model` field in hook config.

Because the prompt hook makes a real LLM API call, it:
- Incurs API costs
- Is subject to rate limits
- Has latency proportional to the model's inference speed
- Times out after 30 seconds by default

### 9.4 Output Format

The LLM is expected to return a JSON object matching `HookOutput`. If the LLM returns plain text instead of JSON, the entire text is treated as `additionalContext`.

Recommended system prompt patterns for prompt hooks:

```
You are a hook that reviews tool calls for security issues.
Always respond with valid JSON only. No explanation, no markdown.
If no issues, respond: {}
If there's an issue: {"decision": "deny", "reason": "<explanation>"}
```

### 9.5 Prompt Hook vs Agent Hook

| Aspect | Prompt Hook | Agent Hook |
|--------|-------------|------------|
| LLM calls | Single call | Multi-turn |
| Tool use | Disabled | Enabled (dontAsk) |
| Timeout default | 30s | 60s |
| Max turns | 1 | 50 |
| Complexity | Simple analysis | Complex workflows |
| Cost | Lower | Higher |
| Latency | Lower | Higher |

---

## 10. Agent Hook Execution

**Source:** `src/tools/tool-2.ts:7821-7990`

### 10.1 Execution Flow

```
1. Serialize event input as JSON string
2. Construct sub-agent configuration:
   - system_prompt: hook.prompt
   - initial_message: JSON.stringify(eventInput)
   - permission_mode: 'dontAsk'
   - model: hook.model || tH()
   - max_turns: 50
   - timeout: hook.timeout || 60000 ms
3. Launch sub-agent
4. Sub-agent runs to completion (or timeout/error)
5. Collect sub-agent's final output message
6. Attempt JSON.parse(finalMessage)
7. If valid JSON -> return as HookOutput
8. If invalid JSON -> wrap as additionalContext
9. On error/timeout -> return empty HookOutput (passthrough)
```

### 10.2 Constraints

| Constraint | Value | Source |
|------------|-------|--------|
| Default timeout | `60000` ms (60 s) | `tool-2.ts:7830` |
| Permission mode | `dontAsk` | `tool-2.ts:7830` |
| Max turns | `50` | `tool-2.ts:7830` |

### 10.3 Permission Mode

Agent hooks run with `dontAsk` permission mode. This means:
- All tool use requests are auto-approved without user confirmation
- The sub-agent can read/write files, run shell commands, make network requests
- The sub-agent has full access to Claude Code's tool suite
- No interactive prompts during hook execution

**Security implication:** Agent hooks with malicious or poorly-written prompts could have significant side effects. Use `allowManagedHooksOnly` or careful prompt design to limit scope.

### 10.4 Output Contract

The sub-agent's final response (last message in the conversation) is parsed as `HookOutput` JSON. Intermediate tool use outputs, thinking blocks, and earlier messages are discarded.

Recommended final message pattern for agent hooks:

```
Task complete. Final decision:
{"decision": "allow", "additionalContext": "Analysis complete: no issues found"}
```

Only the last JSON object in the agent's final message is used.

### 10.5 Agent Hook Use Cases

Agent hooks are appropriate when the hook needs to:
- Make multiple tool calls (read files, run commands) before deciding
- Perform complex multi-step analysis
- Execute side effects (write audit logs, update databases) before returning a decision
- Interact with external services that require multiple API calls

For simple single-call analysis, use `prompt` hooks instead.

---

## 11. HTTP Hook Execution

**Source:** `src/ui/markdown-1.ts:1520-1758`

### 11.1 Execution Flow

```
1. Parse URL
2. Resolve hostname to IP address(es)
3. For each resolved IP:
   a. Check IPv4 private ranges (t2q)
   b. Check IPv6 private ranges (yn_)
   c. If any IP is private (non-loopback) -> abort with security error
4. Check URL against allowedHttpHookUrls allowlist (In_)
   a. If allowlist configured and URL doesn't match -> abort
5. Build headers:
   a. Start with default headers (Content-Type: application/json)
   b. Merge hook.headers
   c. Interpolate ${VAR} in header values (xn_)
      - Only vars in BOTH hook.allowedEnvVars AND httpHookAllowedEnvVars
6. Send HTTP POST:
   - URL: hook.url
   - Method: POST
   - Headers: merged headers
   - Body: JSON.stringify(eventInput)
   - Timeout: hook.timeout || UO (600000 ms)
7. Await response
8. Parse response body as JSON
9. Return parsed HookOutput or empty output on failure
```

### 11.2 Private IP Blocking (`t2q` / `yn_`)

**Source:** `markdown-1.ts:1539-1567`

#### IPv4 Ranges

| CIDR | Range | Allowed? | Notes |
|------|-------|----------|-------|
| `127.0.0.0/8` | 127.0.0.1 - 127.255.255.255 | **ALLOWED** | Localhost, explicitly permitted |
| `10.0.0.0/8` | 10.0.0.0 - 10.255.255.255 | BLOCKED | RFC 1918 private class A |
| `169.254.0.0/16` | 169.254.0.0 - 169.254.255.255 | BLOCKED | Link-local (APIPA) |
| `172.16.0.0/12` | 172.16.0.0 - 172.31.255.255 | BLOCKED | RFC 1918 private class B |
| `100.64.0.0/10` | 100.64.0.0 - 100.127.255.255 | BLOCKED | RFC 6598 shared address space |
| `192.168.0.0/16` | 192.168.0.0 - 192.168.255.255 | BLOCKED | RFC 1918 private class C |

#### IPv6 Ranges

| Prefix | Range | Allowed? | Notes |
|--------|-------|----------|-------|
| `::1` | Loopback | **ALLOWED** | IPv6 localhost, explicitly permitted |
| `fc00::/7` | fc00:: - fdff:: | BLOCKED | Unique local (fc/fd prefixes) |
| `fe80::/10` | fe80:: - febf:: | BLOCKED | Link-local |

**Key point:** Both `127.x.x.x` (IPv4) and `::1` (IPv6) loopback addresses are **allowed**. You can point HTTP hooks at localhost services (e.g., a local webhook receiver on port 8080).

### 11.3 URL Allowlist (`allowedHttpHookUrls`)

**Source:** `markdown-1.ts:1682-1684`

Function `In_` performs pattern matching between the hook URL and each entry in `allowedHttpHookUrls`.

**Pattern syntax:** Glob-style matching where `*` matches any sequence of characters (not just path segments).

```json
{
  "allowedHttpHookUrls": [
    "https://api.example.com/*",
    "http://localhost:*/hooks/*",
    "https://*.internal.company.com/*"
  ]
}
```

**Behavior:**
- If `allowedHttpHookUrls` is **not configured** -> all URLs allowed (subject to IP blocking)
- If `allowedHttpHookUrls` is **configured as empty array** -> no URLs allowed
- If `allowedHttpHookUrls` has entries -> URL must match at least one pattern

### 11.4 Environment Variable Interpolation in Headers (`xn_`)

**Source:** `markdown-1.ts:1689-1705`

Header values support `${VAR_NAME}` syntax for environment variable interpolation.

**Two-layer allowlist:** A variable is only interpolated if it appears in **both**:
1. `hook.allowedEnvVars` (declared in the hook config)
2. `httpHookAllowedEnvVars` (declared in global settings)

```
actual_available = intersection(hook.allowedEnvVars, settings.httpHookAllowedEnvVars)
```

If a referenced variable is not in the intersection, its `${VAR}` placeholder is left as a literal string (or omitted, depending on implementation).

**Example:**

Hook config:
```json
{
  "type": "http",
  "url": "https://api.example.com/hook",
  "headers": {
    "Authorization": "Bearer ${API_TOKEN}",
    "X-Secret": "${INTERNAL_SECRET}"
  },
  "allowedEnvVars": ["API_TOKEN"]
}
```

Settings:
```json
{
  "httpHookAllowedEnvVars": ["API_TOKEN", "PUBLIC_KEY"]
}
```

Result: Only `API_TOKEN` is in the intersection. `${API_TOKEN}` is substituted. `${INTERNAL_SECRET}` is not substituted (not in `httpHookAllowedEnvVars`).

### 11.5 Request Format

The HTTP hook sends a POST request with:

```
POST <hook.url>
Content-Type: application/json
<additional headers from hook.headers>

<JSON-serialized event input>
```

### 11.6 Response Format

The response body should be a JSON object matching `HookOutput`:

```json
{
  "decision": "allow",
  "reason": "Request validated successfully",
  "additionalContext": "Validation timestamp: 2024-01-15T10:30:00Z"
}
```

Empty response body or `{}` is treated as `passthrough`.

### 11.7 Events Where HTTP Hooks Are Blocked

**Source:** `hasworktreecreatehook-1.ts:830-840`

| Event | HTTP Hooks Supported? | Reason |
|-------|-----------------------|--------|
| `SessionStart` | No | HTTP context not yet available |
| `Setup` | No | Too early in initialization |
| All other 21 events | Yes | -- |

---

## 12. Session Hooks

**Source:** `src/tools/tool-1.ts:1620-1740`

Session hooks are hooks registered programmatically during a session, stored in `appState.sessionHooks`. They are not persisted in settings files and exist only for the lifetime of the current session.

### 12.1 Session Hook Storage

```typescript
// appState.sessionHooks shape
type SessionHooks = Partial<Record<HookEvent, MatcherEntry[]>>
```

Session hooks use the same `MatcherEntry` format as settings hooks (same `matcher` + `hooks` array structure).

### 12.2 Session Hook Functions

| Function | Description | Source |
|----------|-------------|--------|
| `z08` | Add a session hook for an event | `tool-1.ts:1620` |
| `w08` | Remove a session hook | `tool-1.ts:1640` |
| `Q44` | List session hooks for an event | `tool-1.ts:1660` |
| `d44` | Clear all session hooks for a specific event | `tool-1.ts:1680` |
| `U44` | Clear all session hooks for all events | `tool-1.ts:1700` |
| `O08` | Check if a session hook exists | `tool-1.ts:1720` |
| `c44` | Get merged hooks (settings + session + plugin) | `tool-1.ts:1730` |
| `l44` | Register a once-hook and track its run state | `tool-1.ts:1735` |
| `Gf6` | Async session hook result handler | `tool-1.ts:1740` |

### 12.3 Session vs Settings Hooks

| Aspect | Settings Hooks | Session Hooks |
|--------|---------------|---------------|
| Storage | `settings.json` | `appState.sessionHooks` |
| Persistence | Across sessions | Current session only |
| Registration | Config file edit | Programmatic API (`z08`) |
| Hot reload | Via plugin system | Immediate |
| Priority | Normal | Normal (merged with settings) |
| Scope | All sessions | This session only |

### 12.4 Session Hook Merging

The `c44` function returns a merged view of hooks from all sources:

```
c44(event) =
  settings.hooks[event]
  + appState.sessionHooks[event]
  + pluginHooks[event]
```

All three sources are concatenated; no deduplication. A hook from settings and an identical hook in session hooks would both run.

---

## 13. Plugin Hook Hot Reload

**Source:** `src/tools/setuppluginhookhotreload-3.ts:1-150`

### 13.1 Overview

Plugin hooks are hooks defined by installed Claude Code plugins. They are loaded when the plugin is enabled and hot-reloaded when the plugin list changes (e.g., when a plugin is enabled/disabled in settings).

### 13.2 Hot Reload Setup (`dt9`)

**Source:** `setuppluginhookhotreload-3.ts:74-92`

Plugin hooks are hot-reloaded when settings change. The mechanism:

1. Subscribe to `policySettings` change events
2. On change, compute an `enabledPlugins` fingerprint:
   ```typescript
   fingerprint = settings.enabledPlugins.sort().join(',')
   ```
3. If fingerprint differs from last-known fingerprint -> reload plugin hooks
4. Call `Ap()` (memoized loader) with new plugin list
5. Update `appState.pluginHooks` with new hooks
6. Changes take effect on the next hook dispatch

### 13.3 Plugin Hook Loader (`Ap`)

**Source:** `setuppluginhookhotreload-3.ts:109-147`

Memoized loader that:
1. Reads the enabled plugin list from settings
2. For each enabled plugin:
   a. Locates the plugin directory
   b. Reads `plugin.hooks` from the plugin manifest
   c. Applies `${CLAUDE_PLUGIN_ROOT}` substitution via `wg()`
   d. Tags hooks with plugin origin metadata
3. Returns merged hook config across all enabled plugins

The loader is memoized on the plugin list fingerprint to avoid redundant file I/O.

### 13.4 Plugin Hook Entry Builder (`Ut9`)

**Source:** `setuppluginhookhotreload-3.ts:22-62`

Builds a `MatcherEntry` from a plugin hook definition. Applies:
- `${CLAUDE_PLUGIN_ROOT}` substitution in command strings
- Metadata tagging for plugin-origin tracking (for `allowManagedHooksOnly` filtering)

### 13.5 Plugin Variable Substitution (`wg`)

**Source:** `src/core/ok-3.ts:49-57`

```typescript
// wg(hookConfig, pluginRoot)
// Substitutes ${CLAUDE_PLUGIN_ROOT} in command strings
function wg(hookConfig: HookSpec, pluginRoot: string): HookSpec {
  if (hookConfig.type === 'command') {
    return {
      ...hookConfig,
      command: hookConfig.command.replace(
        /\$\{CLAUDE_PLUGIN_ROOT\}/g,
        pluginRoot
      )
    }
  }
  return hookConfig
}
```

Only `command` hooks receive this substitution. Other hook types (`prompt`, `agent`, `http`) are returned unchanged.

### 13.6 Plugin Hook Configuration

Plugin authors define hooks in their plugin manifest:

```json
{
  "name": "my-security-plugin",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/scripts/dist/pre-tool-use.js\"",
            "statusMessage": "Security check..."
          }
        ]
      }
    ]
  }
}
```

---

## 14. Security and Policy

### 14.1 Policy Functions

**Source:** `src/core/ok-3.ts:103-120`

#### `w44()` -- Hook Retrieval with Policy

**Source:** `ok-3.ts:103-111`

The central function for fetching hooks, applying policy:

```
w44(event):
  1. ZU6() -> disableAllHooks? -> return []
  2. Dh() -> managedHooksOnly? -> return only plugin hooks
  3. else -> return settings + session + plugin hooks
```

#### `Dh()` -- Managed-Only Check

**Source:** `ok-3.ts:112-117`

Returns `true` if `allowManagedHooksOnly` policy flag is active. When true:
- User-configured hooks in `settings.json` are ignored
- Session hooks registered via `z08()` are ignored
- Only hooks from installed plugins execute

#### `ZU6()` -- Disable-All Check

**Source:** `ok-3.ts:118-120`

Returns `true` if `disableAllHooks` policy flag is active. When true:
- All hook execution is bypassed
- No hooks of any type run for any event
- Policy hooks (if any) are also bypassed

### 14.2 Policy Flags

| Flag | Type | Effect | Typical Use |
|------|------|--------|-------------|
| `disableAllHooks` | `boolean` | Complete hook system disable | Debugging, emergency bypass |
| `allowManagedHooksOnly` | `boolean` | Only plugin/managed hooks | Enterprise governance |

### 14.3 Workspace Trust Check (`eh8`)

**Source:** `hasworktreecreatehook-1.ts:119-122`

Before executing any hooks, the workspace trust level is checked:
- **Untrusted workspace** -> hooks do not execute (all hooks skipped)
- **Trusted workspace** -> hooks execute normally

Workspace trust is determined by whether the user has explicitly trusted the current directory. Untrusted workspaces cannot run hook commands, preventing malicious repositories from executing code via hooks.

### 14.4 HTTP Hook Security Layers

HTTP hooks have four distinct security controls:

1. **Private IP blocking (`t2q`/`yn_`)** -- SSRF prevention. Blocks requests to RFC 1918 addresses, link-local, and shared address spaces. Loopback (`127.x.x.x`, `::1`) is explicitly allowed.

2. **URL allowlist (`allowedHttpHookUrls`)** -- Restricts which URLs HTTP hooks can call. Glob-pattern matching via `In_`. If not configured, all public IPs are reachable.

3. **Env var allowlist intersection** -- Two-layer allowlist (`hook.allowedEnvVars` + `httpHookAllowedEnvVars`) prevents unintended env var leakage into HTTP headers.

4. **Event blocking** -- `SessionStart` and `Setup` cannot use HTTP hooks regardless of configuration.

### 14.5 Permission Mode Interaction

Hooks run within the current permission mode context:

| Permission Mode | Hook Behavior |
|-----------------|---------------|
| `default` | Normal hook execution |
| `acceptEdits` | Normal hook execution |
| `bypassPermissions` | Normal hook execution |
| `dontAsk` | Normal hook execution (agent hooks use this internally) |

The `PermissionRequest` event hook output directly determines permission outcomes, with precedence `deny > ask > allow > passthrough`.

### 14.6 Managed Hooks

Managed hooks are hooks that originate from installed plugins (the plugin system). When `allowManagedHooksOnly` is enabled:

- User-configured hooks in `settings.json` are ignored
- Session hooks registered programmatically are ignored
- Only plugin-provided hooks (tagged with plugin origin metadata) execute

This is a security/governance mechanism for enterprise deployments where administrators want to ensure only approved hooks run.

### 14.7 Hook Execution Isolation

Command hooks run as subprocesses with:
- Inherited process environment (with additions)
- Inherited working directory (`cwd`)
- **No additional sandboxing** -- the subprocess has full system access

Agent hooks run as sub-agents with `dontAsk` permission mode, meaning they have unrestricted tool access within Claude Code's tool suite.

**Security recommendation:** For production deployments with untrusted code repositories, consider:
1. Using `allowManagedHooksOnly` to restrict to plugin hooks only
2. Configuring `allowedHttpHookUrls` to restrict HTTP hook targets
3. Auditing plugin hook scripts before enabling plugins

---

## 15. Practical Examples

### Example 1: Audit All Tool Use

Log every tool call to a file:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '[.session_id, .tool_name, (.tool_input | tostring)] | @tsv' >> /tmp/tool-audit.tsv"
          }
        ]
      }
    ]
  }
}
```

**How it works:** Receives full event JSON on stdin. Uses `jq` to extract session ID, tool name, and input as tab-separated values appended to a log file. Returns exit 0 with no stdout -> passthrough decision, no effect on agent behavior.

**Verified fields used:** `session_id` (base), `tool_name` (PreToolUse), `tool_input` (PreToolUse).

---

### Example 2: Block Dangerous Bash Commands

Deny `rm -rf` patterns using a Python inline script:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 /usr/local/bin/claude-hooks/block-dangerous.py"
          }
        ]
      }
    ]
  }
}
```

Script at `/usr/local/bin/claude-hooks/block-dangerous.py`:

```python
import sys
import json

input_data = json.load(sys.stdin)
cmd = input_data.get('tool_input', {}).get('command', '')

DANGEROUS_PATTERNS = ['rm -rf /', 'dd if=/dev', ':(){:|:&};:']

for pattern in DANGEROUS_PATTERNS:
    if pattern in cmd:
        output = {
            'decision': 'deny',
            'reason': f'Command contains dangerous pattern: {pattern}'
        }
        print(json.dumps(output))
        sys.exit(0)

# No issues - passthrough
sys.exit(0)
```

**How it works:** Reads stdin as JSON, checks for dangerous patterns in `tool_input.command`. Outputs deny decision if found; empty output (passthrough) otherwise.

---

### Example 3: HTTP Webhook on Stop

Notify a webhook when the agent stops:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "https://hooks.example.com/claude-stopped",
            "headers": {
              "Authorization": "Bearer ${WEBHOOK_TOKEN}",
              "X-Session-Source": "claude-code"
            },
            "allowedEnvVars": ["WEBHOOK_TOKEN"],
            "timeout": 5000,
            "statusMessage": "Notifying webhook..."
          }
        ]
      }
    ]
  }
}
```

**Also required in settings:**
```json
{
  "httpHookAllowedEnvVars": ["WEBHOOK_TOKEN"]
}
```

**How it works:** POSTs the full Stop event JSON (including `stop_reason`) to the webhook. `WEBHOOK_TOKEN` interpolated into Authorization header only because it's in both `allowedEnvVars` and `httpHookAllowedEnvVars`. 5-second timeout prevents blocking session cleanup.

---

### Example 4: Git Status Context Injection

Inject current git status into every user prompt:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 -c \"import subprocess, json; status = subprocess.run(['git', 'status', '--short'], capture_output=True, text=True).stdout.strip(); print(json.dumps({'additionalContext': 'Current git status:\\n' + (status or 'Clean working tree')}))\"",
            "timeout": 3000
          }
        ]
      }
    ]
  }
}
```

**How it works:** Runs `git status --short` and returns the output as `additionalContext`. This context is injected into the agent's prompt window. If git is not available or the directory is not a repo, provides a fallback message.

---

### Example 5: Once-Only Session Initialization

Run a setup script only once per session:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/usr/local/bin/claude-session-init.sh",
            "once": true,
            "statusMessage": "Initializing session environment...",
            "timeout": 30000
          }
        ]
      }
    ]
  }
}
```

**How it works:** `once: true` ensures the script runs at most once even if `SessionStart` fires multiple times within a session. `statusMessage` displays in the UI during the 30-second initialization window.

---

### Example 6: Async Background Telemetry

Fire telemetry in the background without blocking the agent:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/usr/local/bin/telemetry-send.sh",
            "async": true
          }
        ]
      }
    ]
  }
}
```

**How it works:** The telemetry script is spawned as a background process. The agent continues immediately without waiting for it. Because `async: true`, the hook's exit code and output are **not** used for decision-making. This is appropriate for fire-and-forget observability hooks.

---

### Example 7: Plugin Hook with Variable Substitution

In a plugin's hook configuration, reference the plugin root directory:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/scripts/dist/pre-tool-use.js\""
          }
        ]
      }
    ]
  }
}
```

**How it works:** `${CLAUDE_PLUGIN_ROOT}` is substituted with the plugin's actual installation path by `wg()` at `ok-3.ts:49-57` before execution. This allows hook scripts to be referenced by relative path within the plugin bundle, making plugins portable across different installation paths.

---

### Example 8: Agent Hook for Complex Security Analysis

Use a sub-agent to analyze PostToolUse results:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "agent",
            "prompt": "You are a security analyzer. Review the bash command and its output provided in the event data. If you see any signs of data exfiltration (sending data to external IPs), privilege escalation (sudo/su with suspicious commands), or credential access (reading /etc/shadow, .ssh keys), output JSON: {\"decision\": \"deny\", \"reason\": \"<specific issue>\"}. Otherwise output: {}",
            "timeout": 30000,
            "statusMessage": "Analyzing output for security concerns..."
          }
        ]
      }
    ]
  }
}
```

**How it works:** After every Bash tool use, a sub-agent (running with `dontAsk` permission mode, max 50 turns, 30s timeout) reviews the command and output. Can return `{"decision": "deny"}` to block further processing or `{}` to pass through. The agent has full tool access during analysis if needed.

---

## 16. Event Dispatch Reference

This section documents when each event fires, what context is available, and which hook types make sense for each.

### 16.1 Tool Lifecycle Dispatch Points

```
Tool Call Request
       |
       v
  [PreToolUse fires]
  Hooks can: deny, allow, ask, modify tool_input
       |
       v
  Tool Executes
       |
       +-- Success path:
       |     v
       |   [PostToolUse fires]
       |   Hooks can: modify tool_response, inject context
       |
       +-- Failure path:
             v
           [PostToolUseFailure fires]
           Hooks can: inject context, alert
```

### 16.2 Session Lifecycle Dispatch Points

```
Claude Code Starts
       |
       v
  [Setup fires] (HTTP hooks blocked)
  Hooks can: validate environment, deny setup
       |
       v
  [SessionStart fires] (HTTP hooks blocked)
  Hooks can: initialize environment, run once
       |
       v
  ... session activity ...
       |
       v
  [SessionEnd fires]
  Hooks can: cleanup, send summaries, persist data
```

### 16.3 Agent Control Dispatch Points

```
Agent decides to stop
       |
       v
  [Stop fires]
  hookSpecificOutput.continue: true -> agent resumes
  hookSpecificOutput.continue: false / omitted -> agent stops
       |
       +-- If stop fails:
             v
           [StopFailure fires]
```

### 16.4 Permission Dispatch Point

```
Permission check needed
       |
       v
  [PermissionRequest fires]
  decision: 'allow' -> granted
  decision: 'deny'  -> denied
  decision: 'ask'   -> user prompted
  decision: 'passthrough' / no decision -> normal permission flow
```

### 16.5 Recommended Hook Types by Event

| Event | Best Hook Type | Reason |
|-------|----------------|--------|
| `PreToolUse` | `command` | Fast, simple access control |
| `PostToolUse` | `command` | Fast audit logging |
| `PostToolUseFailure` | `http` | Alert to monitoring system |
| `UserPromptSubmit` | `command` | Fast context injection |
| `SessionStart` | `command` | `http` not available here |
| `SessionEnd` | `http` | Send summary to external service |
| `Stop` | `command` | Fast continue/stop decision |
| `PermissionRequest` | `command` | Fast, deterministic decisions |
| `SubagentStart` | `command` | Lightweight tracking |
| `SubagentStop` | `command` | Lightweight tracking |
| `PreCompact` | `command` | Save context snapshot |
| `PostCompact` | `command` | Record compaction event |
| `ConfigChange` | `command` | Config change audit |
| `WorktreeCreate` | `command` | Worktree initialization |
| `WorktreeRemove` | `command` | Worktree cleanup |
| `InstructionsLoaded` | `command` | Validate instructions |
| `TeammateIdle` | `http` | Notify team system |
| `TaskCompleted` | `http` | Report task outcome |
| Complex analysis | `agent` | Multi-step reasoning needed |
| Simple LLM check | `prompt` | Single inference needed |

---

## 17. Error Handling and Recovery

### 17.1 Hook Failure Modes

| Failure Type | Hook Type | Behavior |
|-------------|-----------|----------|
| Non-zero exit (not 2) | `command` | Logged, passthrough |
| Exit code 2 | `command` | Blocking error to user |
| Timeout | `command` | Non-blocking, passthrough |
| Timeout | `prompt` | Non-blocking, passthrough |
| Timeout | `agent` | Non-blocking, passthrough |
| Timeout | `http` | Non-blocking, passthrough |
| Parse error | All | Output treated as additionalContext |
| Invalid JSON output | `command` | Output treated as additionalContext |
| LLM error | `prompt` | Empty output (passthrough) |
| Agent error | `agent` | Empty output (passthrough) |
| Network error | `http` | Non-blocking, passthrough |
| IP blocked | `http` | Security error (blocking) |
| URL not in allowlist | `http` | Security error (blocking) |

### 17.2 Exit Code 2 Semantics

Exit code 2 is the only way for a `command` hook to produce a blocking (user-visible) error:

```bash
#!/bin/bash
# Example: deny with blocking error
INPUT=$(cat)
TOOL=$(echo "$INPUT" | jq -r '.tool_name')

if [ "$TOOL" = "Bash" ]; then
  CMD=$(echo "$INPUT" | jq -r '.tool_input.command')
  if echo "$CMD" | grep -q 'rm -rf'; then
    echo "Blocked: rm -rf detected in command" >&2
    exit 2  # Blocking error -- user sees stderr message
  fi
fi

exit 0  # Non-blocking success
```

### 17.3 Graceful Degradation

The hooks system is designed to degrade gracefully:

1. A misbehaving hook (crash, timeout, invalid output) is treated as `passthrough`
2. Other hooks in the same event still run
3. The agent continues normally if no blocking error (exit code 2) is raised
4. Only exit code 2 can block the agent; all other failures are non-blocking

### 17.4 Debugging Hooks

To debug a failing hook:

1. **Test the command directly:**
   ```bash
   echo '{"session_id":"test","transcript_path":"/tmp/t","cwd":"/tmp","tool_name":"Bash","tool_input":{"command":"ls"}}' | bash -c 'your-hook-command'
   echo "Exit: $?"
   ```

2. **Check exit code and stdout/stderr:**
   ```bash
   echo '...' | your-hook-command; echo "Exit: $?" >&2
   ```

3. **Validate JSON output:**
   ```bash
   echo '...' | your-hook-command | python3 -c 'import sys, json; json.load(sys.stdin); print("Valid JSON")'
   ```

4. **Enable verbose logging:** Add `>&2` debug output to stderr (non-fatal), visible in hook error output.

---

## 18. Configuration Patterns and Anti-Patterns

### 18.1 Good Patterns

#### Pattern: Fast fail with blocking error

```bash
#!/bin/bash
# Use exit 2 only for truly blocking issues
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

if [ -n "$CMD" ] && echo "$CMD" | grep -qE '(rm -rf /|mkfs|dd if=/dev/zero)'; then
  echo "BLOCKED: Destructive command detected" >&2
  exit 2
fi
```

#### Pattern: Non-blocking audit with JSON output

```bash
#!/bin/bash
INPUT=$(cat)
SESSION=$(echo "$INPUT" | jq -r '.session_id')
TOOL=$(echo "$INPUT" | jq -r '.tool_name')
TS=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Append to audit log (non-blocking)
echo "$TS\t$SESSION\t$TOOL" >> /var/log/claude-audit.tsv

# Return passthrough
exit 0
```

#### Pattern: Conditional context injection

```python
#!/usr/bin/env python3
import sys, json, subprocess

data = json.load(sys.stdin)

# Only inject context for Bash tool
if data.get('tool_name') == 'Bash':
    try:
        branch = subprocess.check_output(
            ['git', 'branch', '--show-current'],
            cwd=data['cwd'], text=True, timeout=2
        ).strip()
        print(json.dumps({'additionalContext': f'Current git branch: {branch}'}))
    except Exception:
        pass  # Not a git repo, no context needed
```

### 18.2 Anti-Patterns

#### Anti-Pattern: Slow blocking hooks

```json
// BAD: Network call in synchronous hook blocks agent
{
  "type": "command",
  "command": "curl -s https://slow-api.example.com/check",
  "timeout": 600000
}

// GOOD: Use async for fire-and-forget, or http hook with short timeout
{
  "type": "command",
  "command": "curl -s https://fast-api.example.com/check",
  "async": true
}
```

#### Anti-Pattern: Always-deny without reason

```bash
# BAD: Denies everything, no reason, confusing UX
echo '{"decision": "deny"}'
```

```bash
# GOOD: Deny only what needs to be denied, with reason
if is_dangerous "$CMD"; then
  echo "{\"decision\": \"deny\", \"reason\": \"Command '$CMD' matches dangerous pattern list\"}"
fi
```

#### Anti-Pattern: Invalid JSON output

```bash
# BAD: Plain text output is treated as additionalContext, not a decision
echo "BLOCKED"
exit 0

# GOOD: Always output valid JSON for decisions
echo '{"decision": "deny", "reason": "BLOCKED"}'
exit 0
```

#### Anti-Pattern: Using exit code 2 for non-critical errors

```bash
# BAD: Exit 2 for a log write failure blocks the agent
if ! write_audit_log "$INPUT"; then
  echo "Audit log write failed" >&2
  exit 2  # Blocks entire agent -- wrong!
fi

# GOOD: Log failure is non-critical, exit 0 or 1
if ! write_audit_log "$INPUT"; then
  echo "Audit log write failed (non-critical)" >&2
  exit 1  # Non-blocking, logged only
fi
```

---

## 19. Hook Execution Timing Diagrams

### 19.1 Synchronous Hook Chain

```
Time -->

Event fires
   |
   v
[Hook 1 starts] ---timeout---> [timeout: passthrough]
   |                                |
   v (exit 0)                       |
[Hook 1 output processed]           |
   |                                |
   v                                |
[Hook 2 starts]                     |
   |                                |
   v (exit 0)                       |
[Hook 2 output processed]           |
   |                                |
   v                                |
[Decisions aggregated] <-----------+
   |
   v
[Final decision returned to agent]
```

### 19.2 Async Hook

```
Time -->

Event fires
   |
   v
[Async hook spawned] --> [background: runs independently]
   |                         |
   v (immediately)           v (eventually)
[Agent continues]     [asyncRewake? -> agent re-triggered]
```

### 19.3 Exit Code 2 Chain Abort

```
Time -->

Event fires
   |
   v
[Hook 1 starts]
   |
   v (exit 2)
[Blocking error raised]
   |
   v
[Hook 2 SKIPPED]
[Hook 3 SKIPPED]
   |
   v
[Error surfaced to user]
[Agent blocked]
```

### 19.4 `once` Hook Behavior

```
Session start
   |
   v
[Event fires #1]
   |
   v
[once: true hook] -> runs -> [marked as run]
   |
   v
[Event fires #2]
   |
   v
[once: true hook] -> SKIPPED (already ran this session)
   |
   v
[Other hooks still run normally]
```

---

## Appendix A: Full Event -> Input Field Map

Quick reference -- all events and their non-base fields:

| Event | Additional Fields |
|-------|------------------|
| `PreToolUse` | `tool_name`, `tool_input` |
| `PostToolUse` | `tool_name`, `tool_input`, `tool_response` |
| `PostToolUseFailure` | `tool_name`, `tool_input`, `error` |
| `Notification` | `message` |
| `UserPromptSubmit` | `prompt` |
| `SessionStart` | *(none beyond base)* |
| `SessionEnd` | *(none beyond base)* |
| `Stop` | `stop_reason` |
| `StopFailure` | `error` |
| `SubagentStart` | `subagent_id` |
| `SubagentStop` | `subagent_id` |
| `PreCompact` | `trigger` |
| `PostCompact` | `summary` |
| `PermissionRequest` | `tool_name`, `tool_input`, `request_id` |
| `Setup` | *(none beyond base)* |
| `TeammateIdle` | `teammate_id` |
| `TaskCompleted` | `result` |
| `Elicitation` | `message`, `request_id` |
| `ElicitationResult` | `request_id`, `result` |
| `ConfigChange` | `key`, `old_value`, `new_value` |
| `WorktreeCreate` | `worktree_path`, `branch` |
| `WorktreeRemove` | `worktree_path` |
| `InstructionsLoaded` | `instructions_path`, `source` |

---

## Appendix B: Hook Type Feature Matrix

| Feature | command | prompt | agent | http |
|---------|---------|--------|-------|------|
| `type` field | `'command'` | `'prompt'` | `'agent'` | `'http'` |
| `timeout` | Yes (600s default) | Yes (30s default) | Yes (60s default) | Yes (600s default) |
| `statusMessage` | Yes | Yes | Yes | Yes |
| `once` | Yes | Yes | Yes | Yes |
| `async` | Yes | No | No | No |
| `asyncRewake` | Yes | No | No | No |
| `shell` | Yes | No | No | No |
| `model` | No | Yes | Yes | No |
| `prompt` | No | Yes | Yes | No |
| `command` | Yes | No | No | No |
| `url` | No | No | No | Yes |
| `headers` | No | No | No | Yes |
| `allowedEnvVars` | No | No | No | Yes |
| Decision output | Yes | Yes | Yes | Yes |
| `additionalContext` | Yes | Yes | Yes | Yes |
| Background exec | `async: true` | No | No | No |
| Sub-agent | No | No | Yes | No |
| Exit code 2 | Yes | No | No | No |
| Tool use | No | No | Yes (dontAsk) | No |
| Multi-turn | No | No | Yes (max 50) | No |

---

## Appendix C: Settings Configuration Reference

Settings keys relevant to the hooks system:

| Key | Type | Description |
|-----|------|-------------|
| `hooks` | `HooksConfig` | Main hooks configuration map |
| `allowedHttpHookUrls` | `string[]` | URL glob patterns allowed for HTTP hooks |
| `httpHookAllowedEnvVars` | `string[]` | Global env var allowlist for HTTP hook headers |
| `disableAllHooks` | `boolean` | Completely disable hooks system |
| `allowManagedHooksOnly` | `boolean` | Only allow managed (plugin) hooks |

---

## Appendix D: Timeout Reference

| Hook Type | Default Timeout | Override Field | Source |
|-----------|----------------|----------------|--------|
| `command` | `600000` ms (10 min) | `timeout` | `UO` constant at `hasworktreecreatehook-1.ts:2373` |
| `prompt` | `30000` ms (30 s) | `timeout` | `tool-2.ts:7667` |
| `agent` | `60000` ms (60 s) | `timeout` | `tool-2.ts:7830` |
| `http` | `600000` ms (10 min) | `timeout` | `UO` constant at `hasworktreecreatehook-1.ts:2373` |

---

## Appendix E: Source File Index

| File | Lines | Primary Contents |
|------|-------|------------------|
| `src/tools/hasworktreecreatehook-1.ts` | 1-2374 | Main execution engine, dispatchers, collectors |
| `src/tools/writetomailbox-1.ts` | 243-918 | Event input Zod schemas, error enum |
| `src/tools/tool-1.ts` | 1515-1740 | Output schema, session hook helpers |
| `src/tools/tool-2.ts` | 7650-7990 | Prompt and agent hook execution |
| `src/core/ok-3.ts` | 1-131 | Policy logic, plugin var substitution |
| `src/core/auth-1.ts` | 500-636 | Hook type Zod schemas |
| `src/core/u48-1.ts` | 1-38 | Top-level hooks config schema |
| `src/ui/markdown-1.ts` | 1520-1758 | HTTP hook execution, IP blocking, env interp |
| `src/tools/setuppluginhookhotreload-3.ts` | 1-150 | Plugin hook hot reload |

### Critical Line References

| Symbol / Constant | File | Line(s) | Description |
|-------------------|------|---------|-------------|
| `Aq_` | `writetomailbox-1.ts` | 243-267 | Array of all 23 event names |
| `kH` | `writetomailbox-1.ts` | 269-288 | Base event input schema |
| `BC1` | `writetomailbox-1.ts` | 908-918 | Error type enum |
| `pq_`/`hq_` | `writetomailbox-1.ts` | 586-706 | Hook-specific output variants |
| `my9` | `tool-1.ts` | 1520-1610 | Main output schema |
| `ff6` | `tool-1.ts` | 1611-1616 | Async hook result union |
| Session hook functions | `tool-1.ts` | 1620-1740 | `z08`, `w08`, `Q44`, etc. |
| `c2q` | `tool-2.ts` | 7659-7803 | Prompt hook execution |
| `n2q` | `tool-2.ts` | 7821-7990 | Agent hook execution |
| `wg` | `ok-3.ts` | 49-57 | Plugin variable substitution |
| `w44` | `ok-3.ts` | 103-111 | Hook retrieval with policy |
| `Dh` | `ok-3.ts` | 112-117 | Managed-only check |
| `ZU6` | `ok-3.ts` | 118-120 | Disable-all check |
| `wDK` | `auth-1.ts` | 500-636 | Hook type schema factory |
| `w0A` | `u48-1.ts` | 24 | Discriminated union on `type` |
| `O0A` | `u48-1.ts` | 26-36 | Matcher entry schema |
| `JL` | `u48-1.ts` | 37 | Top-level hooks config |
| `t2q` | `markdown-1.ts` | 1539-1556 | IPv4 private range check |
| `yn_` | `markdown-1.ts` | 1558-1567 | IPv6 private range check |
| `In_` | `markdown-1.ts` | 1682-1684 | URL allowlist pattern match |
| `xn_` | `markdown-1.ts` | 1689-1705 | Env var interpolation |
| `zi1` | `markdown-1.ts` | 1707-1758 | HTTP hook execution |
| `AS8` | `hasworktreecreatehook-1.ts` | 381-647 | Command subprocess execution |
| `Bn_` | `hasworktreecreatehook-1.ts` | 649-667 | Matcher evaluation |
| `Oi1` | `hasworktreecreatehook-1.ts` | 715-851 | Hook collector |
| `xx` | `hasworktreecreatehook-1.ts` | 871-1529 | Main REPL dispatcher |
| `ux` | `hasworktreecreatehook-1.ts` | 1533-1698 | Non-REPL dispatcher |
| `UO` | `hasworktreecreatehook-1.ts` | 2373 | Default timeout constant (600000) |
| `eh8` | `hasworktreecreatehook-1.ts` | 119-122 | Workspace trust check |
| `Ap` | `setuppluginhookhotreload-3.ts` | 109-147 | Memoized plugin hook loader |
| `dt9` | `setuppluginhookhotreload-3.ts` | 74-92 | Hot reload setup |
| `Ut9` | `setuppluginhookhotreload-3.ts` | 22-62 | Plugin hook entry builder |
