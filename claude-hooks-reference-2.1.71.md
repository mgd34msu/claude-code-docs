# Claude Code v2.1.71 — Hooks System Reference

> **Version:** 2.1.71  
> **Status:** Definitive Reference  
> **Scope:** Complete hooks system — all 21 events, 4 hook types, execution internals, schemas, policies

---

## Table of Contents

1. [Overview](#overview)
2. [Hook Types](#hook-types)
   - [Command Hook](#command-hook)
   - [Prompt Hook](#prompt-hook)
   - [Agent Hook](#agent-hook)
   - [HTTP Hook](#http-hook)
3. [Hook Events Reference](#hook-events-reference)
   - [PreToolUse](#pretooluse)
   - [PostToolUse](#posttooluse)
   - [PostToolUseFailure](#posttoolusefailure)
   - [PermissionRequest](#permissionrequest)
   - [Notification](#notification)
   - [UserPromptSubmit](#userpromptsubmit)
   - [SessionStart](#sessionstart)
   - [SessionEnd](#sessionend)
   - [Stop](#stop)
   - [SubagentStart](#subagentstart)
   - [SubagentStop](#subagentstop)
   - [PreCompact](#precompact)
   - [TeammateIdle](#teammateidle)
   - [TaskCompleted](#taskcompleted)
   - [Elicitation](#elicitation)
   - [ElicitationResult](#elicitationresult)
   - [ConfigChange](#configchange)
   - [InstructionsLoaded](#instructionsloaded)
   - [Setup](#setup)
   - [WorktreeCreate](#worktreecreate)
   - [WorktreeRemove](#worktreeremove)
4. [Output Schemas](#output-schemas)
   - [Top-Level Output](#top-level-output)
   - [Async Response Schema](#async-response-schema)
   - [hookSpecificOutput Union](#hookspecificoutput-union)
   - [Permission Rule Types](#permission-rule-types)
5. [Hook Execution Internals](#hook-execution-internals)
   - [Dispatcher Flow](#dispatcher-flow)
   - [Command Hook Execution](#command-hook-execution)
   - [Prompt Hook Execution](#prompt-hook-execution)
   - [Agent Hook Execution](#agent-hook-execution)
   - [HTTP Hook Execution](#http-hook-execution)
   - [Private IP Blocking](#private-ip-blocking)
6. [Hook Configuration](#hook-configuration)
   - [File Format](#file-format)
   - [Matcher Syntax](#matcher-syntax)
   - [Configuration Examples](#configuration-examples)
7. [Hook Collection and Merging](#hook-collection-and-merging)
   - [Priority Order](#priority-order)
   - [Loading Logic](#loading-logic)
   - [Deduplication](#deduplication)
8. [Plugin and Session Hooks](#plugin-and-session-hooks)
   - [Plugin Hooks](#plugin-hooks)
   - [Hot Reload](#hot-reload)
   - [Session Hooks](#session-hooks)
9. [Policies and Security](#policies-and-security)
   - [disableAllHooks](#disableallhooks)
   - [allowManagedHooksOnly](#allowmanagedhooksonly)
   - [URL Allowlist](#url-allowlist)
   - [Private IP Blocking Detail](#private-ip-blocking-detail)
   - [Environment Variable Interpolation](#environment-variable-interpolation)
   - [Workspace Trust](#workspace-trust)
10. [Exit Code Reference](#exit-code-reference)
11. [additionalContext](#additionalcontext)
12. [Result Aggregation](#result-aggregation)
13. [Common Input Fields](#common-input-fields)
14. [Practical Examples](#practical-examples)
15. [Quick Reference Tables](#quick-reference-tables)

---

## Overview

Hooks are user-defined scripts, LLM prompts, agent sessions, or HTTP endpoints that execute automatically at specific lifecycle events during a Claude Code session. They provide a programmable layer between Claude's internal operations and the outside world — enabling automation, auditing, permission control, context injection, and workflow integration without modifying Claude itself.

### How Hooks Work

At each lifecycle event, Claude Code:

1. Looks up all registered hooks matching the event (and optional matcher pattern)
2. Serializes the event data as JSON and passes it to each hook
3. Collects hook output (stdout for commands/agents, HTTP response body, LLM response)
4. Parses output as JSON matching the output schema
5. Applies results: block the action, inject context, modify inputs, grant/deny permissions

Hooks execute **sequentially** within an event. Multiple hooks for the same event all run, and their outputs are aggregated (see [Result Aggregation](#result-aggregation)).

### Hook Lifecycle

```
Event fires
  +-> Hook dispatcher (AB())
        +-> Workspace trust check (rb1())
        +-> Simple mode check (bI6())
        +-> Policy check (disableAllHooks, allowManagedHooksOnly)
        +-> Match hooks for this event + matcher (Oe8(), Ghz())
        +-> Deduplicate hooks by type
        +-> Execute each hook sequentially
              +-> Command: spawn() + stdin/stdout
              +-> Prompt: Anthropic API call
              +-> Agent: subagent session with tools
              +-> HTTP: axios POST with DNS validation
```

### Hook Scopes

Hooks can be defined at four scopes (highest to lowest priority):

| Scope | Location | Description |
|-------|----------|-------------|
| **Managed** | `~/.claude/managed-settings.json` or MDM | Policy-enforced, always runs |
| **Plugin** | `~/.claude/plugins/*/manifest.json` | Plugin-provided hooks |
| **Project** | `.claude/settings.json` | Per-project hooks |
| **Local/User** | `~/.claude/local-settings.json` | User personal hooks |

---

## Hook Types

There are exactly 4 hook types, forming a discriminated union on the `type` field.

### Command Hook

Executes a shell command. The hook input JSON is written to stdin. Output is read from stdout.

```typescript
interface CommandHook {
  type: "command";
  // Shell command to execute
  command: string;
  // Timeout in seconds for this specific command (overrides default 30s)
  timeout?: number;
  // Custom status message shown in spinner while hook runs
  statusMessage?: string;
  // If true, hook runs once and is removed after execution
  once?: boolean;
  // If true, hook runs in background without blocking
  async?: boolean;
  // If true, hook runs in background and wakes the model on exit code 2.
  // Implies async: true.
  asyncRewake?: boolean;
}
```

**Execution details:**
- Uses `spawn()` from `node:child_process`
- Platform-aware shell handling (Windows vs Unix)
- Inherits `process.env`, plus adds `CLAUDE_PROJECT_DIR`
- For SessionStart/Setup events: also adds `CLAUDE_ENV_FILE`
- Plugin hooks: also adds `CLAUDE_PLUGIN_ROOT`
- Hook input JSON written to **stdin**
- Output read from **stdout** as newline-delimited JSON lines
- Supports mid-execution prompt requests (hook can query the LLM)
- Default timeout: 30000ms
- First stdout line matching async schema (`{async: true}`) backgrounds the process

**Example:**
```json
{
  "type": "command",
  "command": "python3 ~/.claude/hooks/check_permissions.py",
  "timeout": 10,
  "statusMessage": "Checking permissions..."
}
```

### Prompt Hook

Evaluates a natural-language condition using an LLM. Returns `{ok: true}` or `{ok: false, reason: "..."}`. Useful for semantic checks that are hard to express in shell scripts.

```typescript
interface PromptHook {
  type: "prompt";
  // Prompt text. Use $ARGUMENTS as placeholder for the hook input JSON.
  prompt: string;
  // Timeout in seconds (default: 30s)
  timeout?: number;
  // Model to use (e.g., "claude-sonnet-4-6"). Defaults to small fast model.
  model?: string;
  // Custom status message shown in spinner while hook runs
  statusMessage?: string;
  // If true, hook runs once and is removed after execution
  once?: boolean;
}
```

**Execution details:**
- Calls Anthropic API via internal `fr()` wrapper
- System prompt instructs model to return `{ok: true}` or `{ok: false, reason: "..."}`
- Thinking/extended reasoning: **disabled**
- Default model: small fast model (`Fj()`)
- Default timeout: 30000ms
- Result: `ok: true` -> success; `ok: false` -> blocking with reason as message
- `$ARGUMENTS` in the prompt is replaced with the full hook input JSON

**System prompt used internally:**
```
You are evaluating a hook in Claude Code. Your response must be a JSON object
matching one of the following schemas:
1. If the condition is met, return: {"ok": true}
2. If the condition is not met, return: {"ok": false, "reason": "Reason..."}
```

**Example:**
```json
{
  "type": "prompt",
  "prompt": "Review this tool use: $ARGUMENTS. Is the file path within the project directory and not a system file? Return ok: true only if safe.",
  "model": "claude-haiku-4-5",
  "timeout": 15
}
```

### Agent Hook

Spawns a full subagent session with tool access to verify a condition. The most powerful hook type — can actually run commands, read files, and make complex determinations.

```typescript
interface AgentHook {
  type: "agent";
  // Description of what to verify. Use $ARGUMENTS placeholder for hook input JSON.
  prompt: string;
  // Timeout in seconds (default: 60s)
  timeout?: number;
  // Model to use (e.g., "claude-sonnet-4-6"). Defaults to Haiku.
  model?: string;
  // Custom status message shown in spinner while hook runs
  statusMessage?: string;
  // If true, hook runs once and is removed after execution
  once?: boolean;
}
```

**Execution details:**
- Spawns a real subagent session with filtered tool access
- Stop hook tool excluded from available tools (prevents recursion)
- Max turns: **50** (hard limit)
- Default timeout: **60000ms** (60 seconds)
- Default model: Haiku
- System prompt: "You are verifying a stop condition in Claude Code. Your task is to verify that the agent completed the given plan."
- Outputs structured result via stop hook tool: `{ok: boolean, reason?: string}`
- Outcomes:
  - Max turns reached -> `"cancelled"`
  - No output produced -> `"cancelled"`
  - `ok: false` -> `"blocking"` with reason
  - `ok: true` -> `"success"`
- Telemetry tracked: `tengu_agent_stop_hook_max_turns`, `tengu_agent_stop_hook_error`, `tengu_agent_stop_hook_success`

**Example:**
```json
{
  "type": "agent",
  "prompt": "Check whether unit tests actually ran and passed. The tool use was: $ARGUMENTS. Look at recent test output files or run the test suite to verify.",
  "timeout": 120,
  "statusMessage": "Verifying tests..."
}
```

### HTTP Hook

POSTs the hook input JSON to an HTTP endpoint. The response body is parsed as the hook output.

```typescript
interface HttpHook {
  type: "http";
  // URL to POST hook input JSON to (must be http:// or https://)
  url: string;
  // Timeout in seconds (default: 30s)
  timeout?: number;
  // Additional headers. Values may reference env vars via $VAR or ${VAR}.
  // Only vars listed in allowedEnvVars are interpolated.
  headers?: Record<string, string>;
  // Explicit list of env var names that may be interpolated in headers.
  // Unlisted vars become empty strings.
  allowedEnvVars?: string[];
  // Custom status message shown in spinner while hook runs
  statusMessage?: string;
  // If true, hook runs once and is removed after execution
  once?: boolean;
}
```

**Execution details:**
- Uses axios POST
- Content-Type: `application/json`
- `maxRedirects: 0` (no redirects followed)
- Uses sandbox proxy if enabled, otherwise `false`
- Custom DNS lookup (`XIq()`) blocks private IPs
- URL validated against `allowedUrls` config (wildcard matching)
- Response: HTTP 200-299 -> ok, body parsed as JSON
- Default timeout: 30000ms
- **NOT allowed** for `SessionStart` or `Setup` events

**Security constraints:**
- URL length limit: 2000 characters
- URL must not contain username or password
- Hostname must have at least 2 parts (no bare hostnames)
- Private IP ranges blocked via DNS
- Only env vars explicitly listed in `allowedEnvVars` are interpolated

**Example:**
```json
{
  "type": "http",
  "url": "https://hooks.example.com/claude-event",
  "timeout": 5,
  "headers": {
    "Authorization": "Bearer $HOOK_SECRET",
    "X-Team": "engineering"
  },
  "allowedEnvVars": ["HOOK_SECRET"],
  "statusMessage": "Notifying webhook..."
}
```

### Discriminated Union (TypeScript)

```typescript
type HookUnion =
  | CommandHook
  | PromptHook
  | AgentHook
  | HttpHook;

// Discriminated on the "type" field:
// type === "command" -> CommandHook
// type === "prompt"  -> PromptHook
// type === "agent"   -> AgentHook
// type === "http"    -> HttpHook
```

---

## Hook Events Reference

All 21 hook events are documented below. Each entry includes:
- **Summary** — what triggers this event
- **Matcher field** — which field is used for pattern matching (if any)
- **Blocking** — whether exit code 2 blocks the operation
- **Input schema** — full TypeScript interface of JSON sent to stdin/POST body
- **Output schema** — event-specific output fields (beyond top-level)
- **Trigger sites** — source files where this event is fired

### Common Base Fields

All events include these base fields (injected by `R$()` helper):

```typescript
interface BaseHookInput {
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
  hook_event_name: string;
  subagent_id?: string;
  agent_type?: string;
}
```

---

### PreToolUse

Fires **before** a tool is executed. Can block the tool call or modify its input.

| Property | Value |
|----------|-------|
| **Matcher field** | `tool_name` |
| **Blocking** | YES — exit code 2 blocks the tool call |
| **Source** | `src/tools/tool-1.ts` lines 5487, 5504, 5532 |

**Input Schema:**
```typescript
interface PreToolUseInput extends BaseHookInput {
  hook_event_name: "PreToolUse";
  tool_name: string;
  tool_input: unknown;
  tool_use_id: string;
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:**
```typescript
interface PreToolUseOutput {
  hookEventName: "PreToolUse";
  permissionDecision?: "allow" | "deny" | "ask";
  permissionDecisionReason?: string;
  updatedInput?: Record<string, unknown>;
  additionalContext?: string;
}
```

**Notes:**
- `permissionDecision: "deny"` with exit code 2 blocks the tool
- `updatedInput` replaces the tool's input arguments before execution
- `additionalContext` is injected as a hook system message into the conversation
- Multiple hooks: deny takes priority over ask, ask over allow

**Example use cases:** Block writes to sensitive paths, auto-approve reads from safe directories, sanitize tool arguments, audit logging.

---

### PostToolUse

Fires **after** a tool executes successfully.

| Property | Value |
|----------|-------|
| **Matcher field** | `tool_name` |
| **Blocking** | Partial — exit code 2 shows stderr to model |
| **Source** | `src/tools/tool-1.ts` lines 5259, 5270, 5281, 5293, 5316 |

**Input Schema:**
```typescript
interface PostToolUseInput extends BaseHookInput {
  hook_event_name: "PostToolUse";
  tool_name: string;
  tool_input: unknown;
  tool_response: unknown;
  tool_use_id: string;
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:**
```typescript
interface PostToolUseOutput {
  hookEventName: "PostToolUse";
  additionalContext?: string;
  // Override MCP tool output (only applies to MCP tools)
  updatedMCPToolOutput?: unknown;
}
```

**Notes:**
- `updatedMCPToolOutput` only takes effect for MCP (Model Context Protocol) tool calls
- Exit code 2 shows stderr to the model but does NOT roll back tool execution

---

### PostToolUseFailure

Fires **after** a tool execution **fails** (throws an error).

| Property | Value |
|----------|-------|
| **Matcher field** | `tool_name` |
| **Blocking** | Partial — exit code 2 shows stderr to model |
| **Source** | `src/tools/tool-1.ts` lines 5343, 5354, 5365, 5386 |

**Input Schema:**
```typescript
interface PostToolUseFailureInput extends BaseHookInput {
  hook_event_name: "PostToolUseFailure";
  tool_name: string;
  tool_input: unknown;
  tool_use_id: string;
  error: string;
  // True if failure was due to user interrupt (Ctrl+C)
  is_interrupt?: boolean;
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:**
```typescript
interface PostToolUseFailureOutput {
  hookEventName: "PostToolUseFailure";
  additionalContext?: string;
}
```

**Notes:**
- `is_interrupt: true` indicates the user pressed Ctrl+C during tool execution
- Useful for error recovery, alerting, or cleanup

---

### PermissionRequest

Fires **when a permission dialog would be displayed** to the user.

| Property | Value |
|----------|-------|
| **Matcher field** | `tool_name` |
| **Blocking** | YES — hook return can allow or deny |
| **Source** | `src/tools/hasworktreecreatehook-1.ts` line 1860 |

**Input Schema:**
```typescript
interface PermissionRequestInput extends BaseHookInput {
  hook_event_name: "PermissionRequest";
  tool_name: string;
  tool_input: unknown;
  permission_suggestions?: PermissionSuggestion[];
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:**
```typescript
interface PermissionRequestOutput {
  hookEventName: "PermissionRequest";
  decision: AllowDecision | DenyDecision;
}

interface AllowDecision {
  behavior: "allow";
  updatedInput?: Record<string, unknown>;
  updatedPermissions?: PermissionRule[];
}

interface DenyDecision {
  behavior: "deny";
  message?: string;
  interrupt?: boolean;
}
```

---

### Notification

Fires **when Claude Code sends a notification** to the user.

| Property | Value |
|----------|-------|
| **Matcher field** | `notification_type` |
| **Blocking** | NO |
| **Source** | `src/tools/hasworktreecreatehook-1.ts` line 1671 |

**Input Schema:**
```typescript
interface NotificationInput extends BaseHookInput {
  hook_event_name: "Notification";
  message: string;
  title?: string;
  notification_type:
    | "permission_prompt"
    | "idle_prompt"
    | "auth_success"
    | "elicitation_dialog"
    | "elicitation_complete"
    | "elicitation_response";
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:**
```typescript
interface NotificationOutput {
  hookEventName: "Notification";
  additionalContext?: string;
}
```

**Notes:**
- Non-blocking: hook output does not affect notification delivery
- Use for forwarding notifications to external systems (Slack, email, desktop)
- `notification_type` values: `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog`, `elicitation_complete`, `elicitation_response`

---

### UserPromptSubmit

Fires **when the user submits a prompt** before Claude processes it.

| Property | Value |
|----------|-------|
| **Matcher field** | None |
| **Blocking** | YES — exit code 2 blocks processing and erases the prompt |
| **Source** | `src/tools/selectableusermessagesfilter-3.ts` line 101; `src/tools/hasworktreecreatehook-1.ts` line 1743 |

**Input Schema:**
```typescript
interface UserPromptSubmitInput extends BaseHookInput {
  hook_event_name: "UserPromptSubmit";
  prompt: string;
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:**
```typescript
interface UserPromptSubmitOutput {
  hookEventName: "UserPromptSubmit";
  additionalContext?: string;
}
```

**Notes:**
- Exit code 2 **erases the prompt** — the user's message is discarded
- `additionalContext` is injected into the conversation before processing
- Useful for: prompt validation, content filtering, automatic context injection

---

### SessionStart

Fires **when a new session begins** (or resumes, or is compacted/cleared).

| Property | Value |
|----------|-------|
| **Matcher field** | `source` |
| **Blocking** | NO — blocking errors are ignored |
| **Source** | `src/agents/startdeferredprefetches-1.ts` lines 1370, 1459; `src/tools/setuppluginhookhotreload-3.ts` lines 235, 423, 567; `src/tools/computeisstreamingtextenabled-1.ts` line 547; `src/tools/tool-2.ts` line 2156; `src/core/clearconversation-1.ts` line 85; `src/tools/runheadless-1.ts` lines 2007, 2051 |

**Input Schema:**
```typescript
interface SessionStartInput extends BaseHookInput {
  hook_event_name: "SessionStart";
  source: "startup" | "resume" | "clear" | "compact";
  agent_type?: string;
  model?: string;
  subagent_id?: string;
}
```

**hookSpecificOutput:**
```typescript
interface SessionStartOutput {
  hookEventName: "SessionStart";
  additionalContext?: string;
}
```

**Notes:**
- `startup` = fresh session start; `resume` = resuming previous; `clear` = new conversation; `compact` = after compaction
- HTTP hooks are **NOT allowed** for `SessionStart` events
- Command hooks receive `CLAUDE_ENV_FILE` env var
- `model` field indicates which model is being used

---

### SessionEnd

Fires **when a session is ending**.

| Property | Value |
|----------|-------|
| **Matcher field** | `reason` |
| **Blocking** | NO |
| **Source** | `src/tools/hasworktreecreatehook-1.ts` line 1840 |

**Input Schema:**
```typescript
interface SessionEndInput extends BaseHookInput {
  hook_event_name: "SessionEnd";
  reason: "clear" | "logout" | "prompt_input_exit" | "other";
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:** None — only top-level fields apply.

**Notes:**
- `clear` = conversation cleared; `logout` = user logged out; `prompt_input_exit` = exited from prompt; `other` = any other reason
- Useful for cleanup, final logging, saving state

---

### Stop

Fires **right before Claude concludes its response**.

| Property | Value |
|----------|-------|
| **Matcher field** | None |
| **Blocking** | Partial — exit code 2 shows stderr to model and continues |
| **Source** | `src/tools/tool-2.ts` line 2546; `src/tools/hasworktreecreatehook-1.ts` line 1694 |

**Input Schema:**
```typescript
interface StopInput extends BaseHookInput {
  hook_event_name: "Stop";
  // True if a Stop hook is already active (prevents infinite loops)
  stop_hook_active: boolean;
  last_assistant_message?: string;
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:** None — only top-level fields apply.

**Notes:**
- `stop_hook_active: true` means a Stop hook is already running — prevents infinite loops
- Exit code 2 sends stderr to the model as feedback, causing it to continue processing
- Use `stop_hook_active` check to avoid recursive Stop hook invocations

---

### SubagentStart

Fires **when a subagent (Agent tool call) is started**.

| Property | Value |
|----------|-------|
| **Matcher field** | `agent_type` |
| **Blocking** | NO — blocking errors are ignored |
| **Source** | `src/tools/hasworktreecreatehook-1.ts` line 1781 |

**Input Schema:**
```typescript
interface SubagentStartInput extends BaseHookInput {
  hook_event_name: "SubagentStart";
  agent_id: string;
  agent_type: string;
  subagent_id?: string;
}
```

**hookSpecificOutput:**
```typescript
interface SubagentStartOutput {
  hookEventName: "SubagentStart";
  additionalContext?: string;
}
```

---

### SubagentStop

Fires **right before a subagent concludes its response**.

| Property | Value |
|----------|-------|
| **Matcher field** | `agent_type` |
| **Blocking** | Partial — exit code 2 shows stderr to subagent and continues |
| **Source** | `src/tools/hasworktreecreatehook-1.ts` line 1703 |

**Input Schema:**
```typescript
interface SubagentStopInput extends BaseHookInput {
  hook_event_name: "SubagentStop";
  stop_hook_active: boolean;
  agent_id: string;
  agent_transcript_path: string;
  agent_type: string;
  last_assistant_message?: string;
  subagent_id?: string;
}
```

**hookSpecificOutput:** None — only top-level fields apply.

**Notes:**
- `agent_transcript_path` gives direct access to the subagent's full conversation history
- Exit code 2 shows stderr to the subagent, causing it to continue

---

### PreCompact

Fires **before conversation compaction occurs**.

| Property | Value |
|----------|-------|
| **Matcher field** | `trigger` |
| **Blocking** | YES — exit code 2 blocks compaction |
| **Source** | `src/tools/setuppluginhookhotreload-3.ts` lines 350, 496 |

**Input Schema:**
```typescript
interface PreCompactInput extends BaseHookInput {
  hook_event_name: "PreCompact";
  trigger: "manual" | "auto";
  // Custom compaction instructions (nullable)
  custom_instructions: string | null;
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:** None — only top-level fields apply.

**Notes:**
- `trigger: "manual"` = user requested; `trigger: "auto"` = context window threshold
- `custom_instructions` is the user-provided compaction instruction text, or null

---

### TeammateIdle

Fires **when a teammate is about to go idle**.

| Property | Value |
|----------|-------|
| **Matcher field** | None |
| **Blocking** | YES — exit code 2 prevents the teammate from going idle |
| **Source** | `src/tools/tool-2.ts` line 2635 |

**Input Schema:**
```typescript
interface TeammateIdleInput extends BaseHookInput {
  hook_event_name: "TeammateIdle";
  teammate_name: string;
  team_name: string;
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:** None — only top-level fields apply.

---

### TaskCompleted

Fires **when a task is being marked as completed**.

| Property | Value |
|----------|-------|
| **Matcher field** | None |
| **Blocking** | YES — exit code 2 prevents task completion |
| **Source** | `src/tools/tool-2.ts` line 2610 |

**Input Schema:**
```typescript
interface TaskCompletedInput extends BaseHookInput {
  hook_event_name: "TaskCompleted";
  task_id: string;
  task_subject: string;
  task_description?: string;
  teammate_name?: string;
  team_name?: string;
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:** None — only top-level fields apply.

**Notes:**
- Exit code 2 **rejects task completion** — the agent must continue
- Combine with agent hooks to verify task completion criteria before accepting

---

### Elicitation

Fires **when an MCP server requests user input** (shows an elicitation dialog).

| Property | Value |
|----------|-------|
| **Matcher field** | `mcp_server_name` |
| **Blocking** | YES — exit code 2 denies the elicitation |
| **Source** | `src/tools/hasworktreecreatehook-1.ts` lines 1965, 1995 |

**Input Schema:**
```typescript
interface ElicitationInput extends BaseHookInput {
  hook_event_name: "Elicitation";
  mcp_server_name: string;
  message: string;
  mode?: "form" | "url";
  url?: string;
  elicitation_id?: string;
  requested_schema?: Record<string, unknown>;
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:**
```typescript
interface ElicitationOutput {
  hookEventName: "Elicitation";
  action?: "accept" | "decline" | "cancel";
  content?: Record<string, unknown>;
}
```

**Notes:**
- Exit code 2 denies the elicitation without showing the dialog
- Hook can pre-fill responses by returning `action: "accept"` with `content`
- `requested_schema` describes what fields the MCP server expects

---

### ElicitationResult

Fires **after the user responds to an MCP elicitation**.

| Property | Value |
|----------|-------|
| **Matcher field** | `mcp_server_name` |
| **Blocking** | YES — exit code 2 blocks response, forces action to `decline` |
| **Source** | `src/tools/hasworktreecreatehook-1.ts` lines 1965, 1995 |

**Input Schema:**
```typescript
interface ElicitationResultInput extends BaseHookInput {
  hook_event_name: "ElicitationResult";
  mcp_server_name: string;
  elicitation_id?: string;
  mode?: "form" | "url";
  action: "accept" | "decline" | "cancel";
  content?: Record<string, unknown>;
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:**
```typescript
interface ElicitationResultOutput {
  hookEventName: "ElicitationResult";
  action?: "accept" | "decline" | "cancel";
  content?: Record<string, unknown>;
}
```

**Notes:**
- Exit code 2 blocks the response and forces `action` to `"decline"`
- Hook can modify the action or content before it reaches the MCP server

---

### ConfigChange

Fires **when configuration files change during a session**.

| Property | Value |
|----------|-------|
| **Matcher field** | `source` |
| **Blocking** | YES — exit code 2 blocks the configuration change |
| **Source** | `src/tools/hasworktreecreatehook-1.ts` line 1878 |

**Input Schema:**
```typescript
interface ConfigChangeInput extends BaseHookInput {
  hook_event_name: "ConfigChange";
  source:
    | "user_settings"
    | "project_settings"
    | "local_settings"
    | "policy_settings"
    | "skills";
  file_path?: string;
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:** None — only top-level fields apply.

**Notes:**
- Exit code 2 **reverts the configuration change**
- `file_path` is the absolute path to the modified configuration file

---

### InstructionsLoaded

Fires **when a CLAUDE.md instruction file is loaded**.

| Property | Value |
|----------|-------|
| **Matcher field** | `load_reason` |
| **Blocking** | NO |
| **Source** | `src/tools/hasworktreecreatehook-1.ts` line 1902 |

**Input Schema:**
```typescript
interface InstructionsLoadedInput extends BaseHookInput {
  hook_event_name: "InstructionsLoaded";
  file_path: string;
  memory_type: "User" | "Project" | "Local" | "Managed";
  load_reason:
    | "session_start"
    | "nested_traversal"
    | "path_glob_match"
    | "include";
  globs?: string[];
  trigger_file_path?: string;
  parent_file_path?: string;
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:** None — only top-level fields apply.

**Notes:**
- Non-blocking — cannot prevent instruction loading
- `load_reason` values: `session_start`, `nested_traversal`, `path_glob_match`, `include`
- `memory_type` identifies the settings layer: `User`, `Project`, `Local`, `Managed`

---

### Setup

Fires for **repo setup — initialization and maintenance tasks**.

| Property | Value |
|----------|-------|
| **Matcher field** | `trigger` |
| **Blocking** | NO — blocking errors are ignored |
| **Source** | `src/agents/startdeferredprefetches-1.ts` line 1458; `src/tools/runheadless-1.ts` line 137 |

**Input Schema:**
```typescript
interface SetupInput extends BaseHookInput {
  hook_event_name: "Setup";
  trigger: "init" | "maintenance";
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:**
```typescript
interface SetupOutput {
  hookEventName: "Setup";
  additionalContext?: string;
}
```

**Notes:**
- `init` = initial repo setup; `maintenance` = periodic maintenance
- HTTP hooks are **NOT allowed** for `Setup` events
- Command hooks receive `CLAUDE_ENV_FILE` env var
- Blocking errors are silently ignored

---

### WorktreeCreate

Fires **when creating an isolated git worktree**.

| Property | Value |
|----------|-------|
| **Matcher field** | None |
| **Blocking** | NO |
| **Source** | `src/tools/hasworktreecreatehook-1.ts` line 2171 |

**Input Schema:**
```typescript
interface WorktreeCreateInput extends BaseHookInput {
  hook_event_name: "WorktreeCreate";
  name: string;
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:** None — only top-level fields apply.

---

### WorktreeRemove

Fires **when removing a previously created git worktree**.

| Property | Value |
|----------|-------|
| **Matcher field** | None |
| **Blocking** | NO |
| **Source** | `src/tools/hasworktreecreatehook-1.ts` line 2192 |

**Input Schema:**
```typescript
interface WorktreeRemoveInput extends BaseHookInput {
  hook_event_name: "WorktreeRemove";
  worktree_path: string;
  subagent_id?: string;
  agent_type?: string;
}
```

**hookSpecificOutput:** None — only top-level fields apply.

---

## Output Schemas

### Top-Level Output

All hooks can return these top-level fields. Schema name in source: `whz`

```typescript
interface HookOutput {
  // Whether Claude should continue after this hook (default: true)
  continue?: boolean;
  // Hide stdout from transcript (default: false)
  suppressOutput?: boolean;
  // Message shown when continue is false
  stopReason?: string;
  // Deprecated: backward compatibility only
  decision?: "approve" | "block";
  // Explanation for the decision
  reason?: string;
  // Warning message shown to the user
  systemMessage?: string;
  // Event-specific output (discriminated union by hookEventName)
  hookSpecificOutput?: HookSpecificOutput;
}
```

**Field behavior:**
- `continue: false` + `stopReason` — stops the conversation with a message
- `suppressOutput: true` — hook's stdout is not shown in the transcript
- `decision: "block"` — deprecated way to block (use `continue: false` or exit code 2)
- `systemMessage` — shown as a warning banner to the user

### Async Response Schema

Command hooks can optionally respond with an async marker as their **first stdout line**. Schema name in source: `oE6`

```typescript
interface AsyncMarker {
  async: true;
  asyncTimeout?: number;
}

type HookResponse = AsyncMarker | HookOutput;
```

**Async behavior:**
- If first stdout JSON line is `{async: true}`, the hook process is backgrounded immediately
- Claude Code does not wait for the process to complete
- `asyncTimeout` sets how long the backgrounded process may run

### hookSpecificOutput Union

The `hookSpecificOutput` field is a discriminated union on `hookEventName`:

```typescript
type HookSpecificOutput =
  | { hookEventName: "PreToolUse"; permissionDecision?: "allow"|"deny"|"ask"; permissionDecisionReason?: string; updatedInput?: Record<string,unknown>; additionalContext?: string; }
  | { hookEventName: "PostToolUse"; additionalContext?: string; updatedMCPToolOutput?: unknown; }
  | { hookEventName: "PostToolUseFailure"; additionalContext?: string; }
  | { hookEventName: "UserPromptSubmit"; additionalContext?: string; }
  | { hookEventName: "SessionStart"; additionalContext?: string; }
  | { hookEventName: "Setup"; additionalContext?: string; }
  | { hookEventName: "SubagentStart"; additionalContext?: string; }
  | { hookEventName: "Notification"; additionalContext?: string; }
  | { hookEventName: "PermissionRequest"; decision: AllowDecision | DenyDecision; }
  | { hookEventName: "Elicitation"; action?: "accept"|"decline"|"cancel"; content?: Record<string,unknown>; }
  | { hookEventName: "ElicitationResult"; action?: "accept"|"decline"|"cancel"; content?: Record<string,unknown>; };

// Events with NO hookSpecificOutput (use top-level only):
// SessionEnd, Stop, SubagentStop, PreCompact, TeammateIdle,
// TaskCompleted, ConfigChange, WorktreeCreate, WorktreeRemove, InstructionsLoaded
```

### Permission Rule Types

Used within `PermissionRequest` output's `updatedPermissions` field. Schema name in source: `db1`

```typescript
type PermissionDestination = "project" | "user" | "local";

interface Rule {
  tool: string;   // Tool name pattern (supports glob)
  path?: string;  // Path pattern (supports glob)
}

type PermissionRule =
  | { type: "addRules"; rules: Rule[]; behavior: "allow"|"deny"|"ask"; destination: PermissionDestination; }
  | { type: "replaceRules"; rules: Rule[]; behavior: "allow"|"deny"|"ask"; destination: PermissionDestination; }
  | { type: "removeRules"; rules: Rule[]; behavior: "allow"|"deny"|"ask"; destination: PermissionDestination; }
  | { type: "setMode"; mode: "acceptEdits"|"plan"|"bypassPermissions"|"default"; destination: PermissionDestination; }
  | { type: "addDirectories"; directories: string[]; destination: PermissionDestination; };
```

---

## Hook Execution Internals

### Dispatcher Flow

The main hook dispatcher is the async generator `AB()` in `hasworktreecreatehook-1.ts` (lines 800-1300).

**Parameters:** `hookInput, toolUseID, matchQuery, signal, timeoutMs, toolUseContext, messages, forceSyncExecution, requestPrompt, toolInputSummary`

**Dispatch sequence:**

```
1. Workspace trust check: rb1()
   -> Untrusted workspaces may have hooks restricted

2. Simple mode check: bI6()
   -> CLAUDE_CODE_SIMPLE env var may restrict hooks

3. Match hooks for this event: Oe8()
   -> Query hook settings
   -> Switch on event type to extract matcher key
   -> Pattern match via Ghz()

4. Deduplicate hooks by type:
   -> command: dedup by command string
   -> prompt:  dedup by prompt string
   -> agent:   dedup by prompt
   -> http:    dedup by url

5. Filter HTTP hooks for SessionStart/Setup events

6. Execute each hook sequentially:
   -> command -> ob1()
   -> prompt  -> wIq()
   -> agent   -> OIq()
   -> http    -> _e8()
```

**Matcher lookup (`Oe8()`) by event:**

| Events | Matcher Key |
|--------|------------|
| PreToolUse, PostToolUse, PostToolUseFailure, PermissionRequest | `tool_name` |
| Notification | `notification_type` |
| SessionStart | `source` |
| SessionEnd | `reason` |
| SubagentStart, SubagentStop | `agent_type` |
| PreCompact, Setup | `trigger` |
| ConfigChange | `source` |
| InstructionsLoaded | `load_reason` |
| Elicitation, ElicitationResult | `mcp_server_name` |
| UserPromptSubmit, Stop, TeammateIdle, TaskCompleted, WorktreeCreate, WorktreeRemove | *(no matcher — all hooks run)* |

### Command Hook Execution

Executor: `ob1()` in `hasworktreecreatehook-1.ts` (lines 370-600)

```
ob1(hook, hookInput, signal, timeoutMs, toolUseContext)

1. Platform detection:
   Windows: cmd.exe /d /s /c "<command>"
   Unix:    /bin/sh -c "<command>"

2. Environment setup:
   Inherits: process.env
   Adds: CLAUDE_PROJECT_DIR
   SessionStart/Setup: + CLAUDE_ENV_FILE
   Plugin hooks: + CLAUDE_PLUGIN_ROOT

3. stdin: hookInput serialized as JSON

4. stdout listener:
   Line 1: check for async marker {async: true}
     -> If found: background process, return immediately
   Remaining lines: collect JSON output
   Supports prompt request protocol:
     Hook writes: {promptRequest: "...", requestId: "..."}
     Claude Code responds with LLM answer via stdin

5. Timeout: hook.timeout * 1000 || 30000ms (Hj constant)

6. Exit codes:
   0     -> success, parse stdout as HookOutput
   2     -> blocking (event-specific behavior)
   other -> error, log stderr, continue to next hook
```

**Environment variables available to command hooks:**

| Variable | Present | Events |
|----------|---------|--------|
| `CLAUDE_PROJECT_DIR` | Always | All |
| `CLAUDE_ENV_FILE` | Conditional | SessionStart, Setup only |
| `CLAUDE_PLUGIN_ROOT` | Conditional | Plugin hooks only |
| All other `process.env` | Always | All |

### Prompt Hook Execution

Executor: `wIq()` in `tool-2.ts` (lines 4761-4870)

```
wIq(hook, hookInput, signal)

1. Replace $ARGUMENTS in hook.prompt with JSON.stringify(hookInput)

2. Call Anthropic API (fr() wrapper):
   system: "You are evaluating a hook in Claude Code..."
   user: hook.prompt (with $ARGUMENTS substituted)
   model: hook.model || Fj() (default small fast model)
   thinking: DISABLED
   output format: {ok: boolean, reason?: string}

3. Timeout: 30000ms

4. Result:
   ok: true  -> "success"
   ok: false -> "blocking" with reason as message
```

**Exact system prompt:**
```
You are evaluating a hook in Claude Code. Your response must be a JSON object
matching one of the following schemas:
1. If the condition is met, return: {"ok": true}
2. If the condition is not met, return: {"ok": false, "reason": "Reason..."}
```

### Agent Hook Execution

Executor: `OIq()` in `tool-2.ts` (lines 4923-5080)

```
OIq(hook, hookInput, signal)

1. Replace $ARGUMENTS in hook.prompt with JSON.stringify(hookInput)

2. Spawn subagent session:
   system: "You are verifying a stop condition in Claude Code.
            Your task is to verify that the agent completed the given plan."
   tools: all available tools EXCEPT stop hook tool
   max_turns: 50 (hard limit, const B = 50)
   model: hook.model || Haiku (default)

3. Timeout: hook.timeout * 1000 || 60000ms

4. Structured output via stop hook tool:
   {ok: boolean, reason?: string}

5. Outcomes:
   max turns (50) reached -> "cancelled"
   no structured output   -> "cancelled"
   ok: false              -> "blocking" with reason
   ok: true               -> "success"
```

**Telemetry events tracked:**
- `tengu_agent_stop_hook_max_turns` — agent hit the 50-turn limit
- `tengu_agent_stop_hook_error` — agent encountered an error
- `tengu_agent_stop_hook_success` — agent completed successfully

### HTTP Hook Execution

Executor: `_e8()` in `markdown-1.ts` (lines 976-1020)

```
_e8(hook, hookInput, signal)

1. URL validation:
   Length: must be <= 2000 characters
   No username/password in URL
   Hostname: must have >= 2 parts
   allowedUrls: must match at least one pattern (wildcard)

2. Headers:
   Content-Type: application/json (always)
   Custom headers from hook.headers
   Env var interpolation: only allowedEnvVars

3. axios POST:
   body: JSON.stringify(hookInput)
   maxRedirects: 0
   proxy: sandbox proxy if enabled, else false
   lookup: XIq() (custom DNS -- blocks private IPs)

4. Timeout: hook.timeout * 1000 || 30000ms

5. Response:
   200-299 -> ok, parse body as HookOutput
   other   -> error, log, continue to next hook
```

### Private IP Blocking

Custom DNS resolver `XIq()` used by HTTP hooks to prevent SSRF attacks.

**IPv4 blocked ranges:**

| Range | Description |
|-------|-------------|
| `0.0.0.0/8` | Unspecified |
| `10.0.0.0/8` | Private class A |
| `100.64.0.0/10` | Carrier-grade NAT (RFC 6598) |
| `169.254.0.0/16` | Link-local (APIPA) |
| `172.16.0.0/12` | Private class B |
| `192.168.0.0/16` | Private class C |

**IPv4 ALLOWED:** `127.0.0.0/8` (localhost for local development)

**IPv6 blocked:**

| Address/Range | Description |
|--------------|-------------|
| `::` | Unspecified address |
| `fc00::/7` | Unique local addresses (ULA) |
| `fe80::/10` | Link-local addresses |

**IPv6 ALLOWED:** `::1` (loopback)

**Hostname resolution:** For domain names, `dns.lookup()` resolves the hostname and ALL resolved IP addresses are checked. If any resolved address is in a blocked range, the request is rejected.

---

## Hook Configuration

### File Format

Hooks are configured in JSON settings files under a top-level `hooks` key:

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "optional_pattern",
        "hooks": [
          {
            "type": "command|prompt|agent|http"
          }
        ]
      }
    ]
  }
}
```

**TypeScript type:**
```typescript
interface HooksConfig {
  hooks: {
    [eventName: string]: HookConfigEntry[];
  };
}

interface HookConfigEntry {
  // Pattern to match against the event's matcher field.
  // Empty or omitted: matches all.
  matcher?: string;
  // List of hooks to execute when matcher matches
  hooks: HookUnion[];
}
```

### Matcher Syntax

Matchers are **string patterns** matched against the event-specific matcher field.

| Event | Matcher Field | Example Values |
|-------|--------------|----------------|
| PreToolUse | `tool_name` | `"Bash"`, `"Write\|Edit"` |
| PostToolUse | `tool_name` | `"mcp__.*"`, `"Read"` |
| PostToolUseFailure | `tool_name` | `"Bash"` |
| PermissionRequest | `tool_name` | `"Write"` |
| Notification | `notification_type` | `"permission_prompt"` |
| SessionStart | `source` | `"startup"` |
| SessionEnd | `reason` | `"logout"` |
| SubagentStart | `agent_type` | `"engineer"` |
| SubagentStop | `agent_type` | `"engineer"` |
| PreCompact | `trigger` | `"auto"` |
| ConfigChange | `source` | `"project_settings"` |
| InstructionsLoaded | `load_reason` | `"session_start"` |
| Elicitation | `mcp_server_name` | `"my-mcp-server"` |
| ElicitationResult | `mcp_server_name` | `"my-mcp-server"` |
| Setup | `trigger` | `"init"` |
| UserPromptSubmit | *(none)* | N/A |
| Stop | *(none)* | N/A |
| TeammateIdle | *(none)* | N/A |
| TaskCompleted | *(none)* | N/A |
| WorktreeCreate | *(none)* | N/A |
| WorktreeRemove | *(none)* | N/A |

**Matching rules:**
- Empty or omitted `matcher` -> matches all contexts
- Non-empty `matcher` -> treated as a **regex pattern** matched against the field value
- Invalid regex patterns are caught and logged (hook skipped)
- Example: `"Bash|Write|Edit"` matches any of those three tool names
- Example: `"mcp__.*"` matches any MCP tool

### Configuration Examples

**Block dangerous shell commands:**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.claude/hooks/check_bash_safety.py",
            "timeout": 10,
            "statusMessage": "Safety check..."
          }
        ]
      }
    ]
  }
}
```

**Inject project context at session start:**
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/inject_context.sh",
            "statusMessage": "Loading project context..."
          }
        ]
      }
    ]
  }
}
```

**HTTP webhook for file writes:**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "http",
            "url": "https://api.example.com/audit",
            "headers": {
              "Authorization": "Bearer $AUDIT_TOKEN"
            },
            "allowedEnvVars": ["AUDIT_TOKEN"]
          }
        ]
      }
    ]
  }
}
```

**Once-only init hook:**
```json
{
  "hooks": {
    "Setup": [
      {
        "matcher": "init",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/initial_setup.sh",
            "once": true
          }
        ]
      }
    ]
  }
}
```

**Agent verifier on Stop:**
```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "Verify the agent completed its task: $ARGUMENTS. Check that all required output files exist.",
            "timeout": 120,
            "statusMessage": "Verifying completion..."
          }
        ]
      }
    ]
  }
}
```

**Async background logging:**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/audit_logger.sh",
            "async": true
          }
        ]
      }
    ]
  }
}
```

---

## Hook Collection and Merging

### Priority Order

Hook settings are loaded from four sources in priority order:

```
Highest priority
     |
     v
1. Managed    -- ~/.claude/managed-settings.json or MDM policy
2. Plugin     -- ~/.claude/plugins/*/manifest.json
3. Project    -- .claude/settings.json (in project root)
4. Local/User -- ~/.claude/local-settings.json
     |
     v
Lowest priority
```

### Loading Logic

Function: `J18()` in `jm-2.ts`

```
J18()

1. Read policySettings

2. policySettings.disableAllHooks === true?
   -> YES: return {} (empty -- no hooks run at all)

3. policySettings.allowManagedHooksOnly === true?
   -> YES: return only managed hooks

4. Read user settings (BA())

5. user settings disableAllHooks === true?
   -> YES: return only managed hooks

6. Return merged: user hooks + managed hooks
   (managed always included at top)
```

### Deduplication

Within a single event and matcher, hooks are deduplicated:

- **Command hooks:** deduplicated by `command` string
- **Prompt hooks:** deduplicated by `prompt` string
- **Agent hooks:** deduplicated by `prompt` string
- **HTTP hooks:** deduplicated by `url` string

Deduplication process:
1. Matchers sorted **lexicographically**
2. Hooks within each matcher sorted by their **string representation**
3. Duplicates (same type + identifying field) removed, keeping first occurrence

---

## Plugin and Session Hooks

### Plugin Hooks

Plugins register hooks via their manifest's `hooksConfig` field.

**Plugin manifest structure:**
```json
{
  "name": "my-plugin",
  "id": "my-plugin-id",
  "hooksConfig": {
    "hooks": {
      "PreToolUse": [
        {
          "matcher": "Write",
          "hooks": [
            {
              "type": "command",
              "command": "$CLAUDE_PLUGIN_ROOT/hooks/pre-write.sh"
            }
          ]
        }
      ]
    }
  }
}
```

**Plugin hook tagging:** Each plugin hook is tagged with:
- `pluginRoot` — absolute path to plugin directory
- `pluginName` — plugin display name
- `pluginId` — plugin unique identifier

**Loading:** `QHY()` function in `setuppluginhookhotreload-3.ts`

**Restrictions:** HTTP hooks NOT allowed for `SessionStart` or `Setup` events

### Hot Reload

Plugin hooks support in-memory hot reload — no restart required.

```
policySettings change detected (o$.subscribe())
  -> Hash enabledPlugins list
  -> Compare to previous hash
  -> If changed:
       -> Clear plugin hook cache: of1()
       -> Reload plugin hooks: KQ()
```

**What triggers hot reload:** Enabling/disabling a plugin, changing plugin config, policy settings update.

### Session Hooks

Ephemeral hooks registered at runtime for a specific session. Used by skills and internal components.

**Storage:** `appState.sessionHooks[sessionId]`

**API functions:**

| Function | Description |
|----------|-------------|
| `fcA()` | Register hooks for a session (with matcher deduplication) |
| `H51()` / `VcA()` | Get hooks for a specific event |
| `TcA()` | Remove specific session hooks |
| `rM6()` | Clear all hooks for a session |

**Notes:**
- Skills call `registerSessionHook` / `addSessionHook`
- Session hooks are removed when the session ends
- Session hooks participate in normal matching and execution
- Merged with config-file hooks; priority = order of registration

---

## Policies and Security

### disableAllHooks

**Setting:** `policySettings.disableAllHooks` (boolean)

When `true`, hook system returns `{}` — no hooks execute for any event.

```json
{ "disableAllHooks": true }
```

**Checked by:** `bI6()` function (also checks `CLAUDE_CODE_SIMPLE` env var)

**Effect:** All hook events skipped. No commands spawned, no HTTP calls made, no prompts evaluated. Takes effect immediately.

### allowManagedHooksOnly

**Setting:** `policySettings.allowManagedHooksOnly` (boolean)

When `true`, only hooks defined in managed/policy settings execute.

```json
{ "allowManagedHooksOnly": true }
```

**Effect:** Project hooks, local/user hooks, and plugin hooks are all skipped. Only managed/policy hooks run.

### URL Allowlist

**Config:** `allowedUrls` array in hook settings

```json
{
  "allowedUrls": [
    "https://api.example.com/*",
    "https://*.mycompany.com/webhooks"
  ]
}
```

**Matching function:** `Xhz()` — wildcard glob matching. Loaded via `Dhz()`.

**Rules:**
1. If `allowedUrls` is defined and non-empty, HTTP hook URLs must match at least one pattern
2. URL length limit: **2000 characters**
3. URL must not contain username or password components
4. Hostname must have at least **2 parts** (no bare hostnames; use `127.0.0.1` not `localhost`)
5. Wildcard `*` matches within a path segment or subdomain

### Private IP Blocking Detail

HTTP hooks use custom DNS lookup function `XIq()` to prevent SSRF attacks.

**Complete blocked ranges:**

```
IPv4:
  0.0.0.0/8       -- "This" network
  10.0.0.0/8      -- Private class A
  100.64.0.0/10   -- Carrier-grade NAT (RFC 6598)
  169.254.0.0/16  -- Link-local (APIPA)
  172.16.0.0/12   -- Private class B
  192.168.0.0/16  -- Private class C

IPv4 ALLOWED:
  127.0.0.0/8     -- Localhost (for local development)

IPv6 BLOCKED:
  ::              -- Unspecified address
  fc00::/7        -- Unique local addresses (ULA)
  fe80::/10       -- Link-local addresses

IPv6 ALLOWED:
  ::1             -- Loopback
```

**Hostname resolution:** `dns.lookup()` resolves domain names; ALL resolved addresses are checked. Any match in a blocked range rejects the request.

### Environment Variable Interpolation

Only available in HTTP hook `headers` values.

**Syntax:** `$VAR_NAME` or `${VAR_NAME}`

**Security constraints:**
1. Only variables explicitly listed in `hook.allowedEnvVars` are interpolated
2. Variables not in `allowedEnvVars` become **empty strings** (not an error)
3. Further filtered by global `allowedEnvVars` configuration
4. Never available in command strings, prompt text, or URLs

**Example:**
```json
{
  "type": "http",
  "url": "https://api.example.com/hook",
  "headers": {
    "Authorization": "Bearer $API_TOKEN",
    "X-Client": "${CLIENT_NAME}"
  },
  "allowedEnvVars": ["API_TOKEN", "CLIENT_NAME"]
}
```

### Workspace Trust

**Function:** `rb1()` — checks workspace trust before hook execution.

Hooks in untrusted workspaces may be restricted or require explicit user confirmation. Trust check occurs in dispatcher `AB()` before any hook execution.

- Trusted workspace: all hooks execute normally
- Untrusted workspace: hooks may be skipped or require user approval

### CLAUDE_CODE_SIMPLE

**Environment variable:** `CLAUDE_CODE_SIMPLE`

When set, may restrict hook execution. Checked via `bI6()` in the dispatcher. Intended for simplified/restricted operation modes.

---

## Exit Code Reference

Complete matrix of exit code behavior by event:

| Event | Exit 0 | Exit 2 | Other non-zero |
|-------|--------|--------|----------------|
| **PreToolUse** | Success, apply output | **BLOCKS tool call** | Error, log stderr, next hook |
| **PostToolUse** | Success, apply output | Shows stderr to model | Error, log stderr, next hook |
| **PostToolUseFailure** | Success, apply output | Shows stderr to model | Error, log stderr, next hook |
| **PermissionRequest** | Apply decision from output | Handled via decision field | Error, log stderr, next hook |
| **UserPromptSubmit** | Success, apply output | **BLOCKS prompt, erases it** | Error, log stderr, next hook |
| **SessionStart** | Success, apply output | Ignored | Ignored |
| **SessionEnd** | Success | Ignored | Ignored |
| **Stop** | Success | Shows stderr to model, continues | Error, log stderr, next hook |
| **SubagentStart** | Success, apply output | Ignored | Ignored |
| **SubagentStop** | Success | Shows stderr to subagent, continues | Error, log stderr, next hook |
| **PreCompact** | Success | **BLOCKS compaction** | Error, log stderr, next hook |
| **TeammateIdle** | Success | **PREVENTS idle** | Error, log stderr, next hook |
| **TaskCompleted** | Success | **PREVENTS completion** | Error, log stderr, next hook |
| **Elicitation** | Success, apply output | **DENIES elicitation** | Error, log stderr, next hook |
| **ElicitationResult** | Success, apply output | **BLOCKS response**, forces decline | Error, log stderr, next hook |
| **ConfigChange** | Success | **BLOCKS config change** | Error, log stderr, next hook |
| **InstructionsLoaded** | Success | Ignored | Ignored |
| **Setup** | Success, apply output | Ignored | Ignored |
| **WorktreeCreate** | Success | Ignored | Ignored |
| **WorktreeRemove** | Success | Ignored | Ignored |

**Blocking events summary:**
- **Full block:** `PreToolUse`, `UserPromptSubmit`, `PreCompact`, `TeammateIdle`, `TaskCompleted`, `Elicitation`, `ElicitationResult`, `ConfigChange`
- **Partial (feedback to model):** `PostToolUse`, `PostToolUseFailure`, `Stop`, `SubagentStop`
- **No effect:** `SessionStart`, `SessionEnd`, `SubagentStart`, `InstructionsLoaded`, `Setup`, `WorktreeCreate`, `WorktreeRemove`

---

## additionalContext

### Which Events Support It

| Event | Field Location |
|-------|---------------|
| PreToolUse | `hookSpecificOutput.additionalContext` |
| PostToolUse | `hookSpecificOutput.additionalContext` |
| PostToolUseFailure | `hookSpecificOutput.additionalContext` |
| UserPromptSubmit | `hookSpecificOutput.additionalContext` |
| SessionStart | `hookSpecificOutput.additionalContext` |
| Setup | `hookSpecificOutput.additionalContext` |
| SubagentStart | `hookSpecificOutput.additionalContext` |
| Notification | `hookSpecificOutput.additionalContext` |

### Injection Mechanism

When a hook returns `additionalContext`, Claude Code:
1. Parses `hookSpecificOutput.additionalContext` string
2. Creates attachment of type `"hook_additional_context"` or `"hook_system_message"`
3. Injects attachment into the conversation message stream
4. Context becomes visible to Claude as part of the conversation

### Truncation

- Content truncated at `MAX_MARKDOWN_LENGTH`
- Truncated content gets suffix: `"[Content truncated due to length...]"`

### Usage Pattern

```bash
#!/bin/bash
read -r hook_input

branch=$(git branch --show-current 2>/dev/null || echo 'unknown')

jq -n --arg ctx "Git branch: $branch" '{
  hookSpecificOutput: {
    hookEventName: "SessionStart",
    additionalContext: $ctx
  }
}'
```

---

## Result Aggregation

### Permission Priority

For events that produce permission decisions (`PreToolUse`, `PermissionRequest`):

```
Final decision priority:  deny > ask > allow

If ANY hook returns deny  -> final result is deny
If ANY hook returns ask   -> final result is ask (unless a deny exists)
If ALL hooks return allow -> final result is allow
```

### updatedInput Merging

For `PreToolUse` hooks returning `updatedInput`:
- Multiple hooks' `updatedInput` values are merged sequentially
- Later hooks see the updated input from earlier hooks
- Only merged when `permissionBehavior` is not explicitly set to deny/ask

### Context Collection

`additionalContext` from all hooks is:
- Collected from every hook that returns it
- Combined into a single attachment stream
- Injected once into the conversation (not once per hook)
- Concatenated in hook execution order

### Error Handling in Aggregation

- Hook exits non-zero (not 2): logs stderr, continues to next hook
- Hook times out: terminated, next hook runs
- Hook produces invalid JSON: parse error logged, continues
- Aggregated result uses all successfully parsed outputs

---

## Common Input Fields

All hook events include base fields injected by the `R$()` helper function:

```typescript
interface BaseHookInput {
  // Unique identifier for the current session
  session_id: string;

  // Absolute path to the current session transcript file
  // Format: ~/.claude/transcripts/<session-id>.jsonl
  transcript_path: string;

  // Current working directory when the hook fires
  cwd: string;

  // Current permission mode
  // Values: "default", "bypassPermissions", "plan", "acceptEdits"
  // Not present if permission mode is unavailable
  permission_mode?: string;

  // The event type name -- always matches hook_event_name
  hook_event_name: string;

  // Present when the hook fires within a subagent context
  subagent_id?: string;

  // The agent type/role when running in an agent session
  // e.g., "engineer", "orchestrator"
  agent_type?: string;
}
```

**`transcript_path` notes:**
- Path to newline-delimited JSON (JSONL) transcript file
- Available in all hook types (command, prompt, agent, HTTP)
- Read for conversation history; write with care for custom logging

---

## Practical Examples

### Example 1: Block Dangerous Bash Commands

```python
#!/usr/bin/env python3
# ~/.claude/hooks/check_bash.py
import json, sys, re

hook_input = json.load(sys.stdin)
if hook_input.get('tool_name') != 'Bash':
    sys.exit(0)

command = hook_input.get('tool_input', {}).get('command', '')

dangerous = [
    r'rm\s+-rf\s+/',
    r'sudo\s+rm',
    r'dd\s+if=.*of=/dev/',
    r'mkfs\.',
]

for pattern in dangerous:
    if re.search(pattern, command):
        print(json.dumps({
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": f"Dangerous pattern blocked: {pattern}"
            }
        }))
        sys.exit(2)

sys.exit(0)
```

```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Bash", "hooks": [{ "type": "command", "command": "python3 ~/.claude/hooks/check_bash.py", "timeout": 5 }] }
    ]
  }
}
```

### Example 2: Auto-Approve Safe Reads

```bash
#!/usr/bin/env bash
read -r hook_input
tool=$(echo "$hook_input" | jq -r '.tool_name')
path=$(echo "$hook_input" | jq -r '.tool_input.path // empty')
cwd=$(echo "$hook_input" | jq -r '.cwd')

if [[ "$tool" == "Read" && "$path" == "$cwd"* ]]; then
  echo '{"hookSpecificOutput": {"hookEventName": "PermissionRequest", "decision": {"behavior": "allow"}}}'
fi
```

### Example 3: Inject Git Context at Session Start

```bash
#!/usr/bin/env bash
read -r hook_input
source=$(echo "$hook_input" | jq -r '.source')
[[ "$source" != "startup" ]] && exit 0

branch=$(git branch --show-current 2>/dev/null || echo 'not a git repo')
status=$(git status --short 2>/dev/null | head -10 || echo '')
recent=$(git log --oneline -5 2>/dev/null || echo '')

ctx="Git branch: $branch"
[ -n "$status" ] && ctx="$ctx
Modified:
$status"
[ -n "$recent" ] && ctx="$ctx
Recent commits:
$recent"

jq -n --arg c "$ctx" '{hookSpecificOutput:{hookEventName:"SessionStart",additionalContext:$c}}'
```

### Example 4: Force Continuation with Stop Hook

```bash
#!/usr/bin/env bash
read -r hook_input
stop_active=$(echo "$hook_input" | jq -r '.stop_hook_active')
[[ "$stop_active" == "true" ]] && exit 0

cwd=$(echo "$hook_input" | jq -r '.cwd')
if [[ ! -f "$cwd/output.json" ]]; then
  echo "Required file output.json does not exist. Please create it before finishing." >&2
  exit 2
fi
```

### Example 5: Handle Elicitation Automatically

```python
#!/usr/bin/env python3
import json, sys
from pathlib import Path

hook_input = json.load(sys.stdin)
server = hook_input.get('mcp_server_name', '')

config_file = Path.home() / '.claude' / 'elicitation_responses.json'
if not config_file.exists():
    sys.exit(0)

with open(config_file) as f:
    responses = json.load(f)

if server in responses:
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "Elicitation",
            "action": "accept",
            "content": responses[server]
        }
    }))
sys.exit(0)
```

### Example 6: Async Audit Logging

```bash
#!/usr/bin/env bash
# Signal async immediately
echo '{"async": true}'

hook_input=$(cat)
echo "$hook_input" >> "$HOME/.claude/audit.jsonl"

if [ -n "$AUDIT_ENDPOINT" ]; then
  curl -s -X POST "$AUDIT_ENDPOINT" \
    -H "Content-Type: application/json" \
    -d "$hook_input" > /dev/null 2>&1
fi
```

### Example 7: HTTP Webhook with Auth

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [{
          "type": "http",
          "url": "https://audit.company.com/api/v1/file-changes",
          "timeout": 3,
          "headers": {
            "Authorization": "Bearer $AUDIT_API_KEY",
            "X-Source": "claude-code"
          },
          "allowedEnvVars": ["AUDIT_API_KEY"]
        }]
      }
    ]
  }
}
```

### Example 8: Agent Verifier for Task Completion

```json
{
  "hooks": {
    "TaskCompleted": [
      {
        "hooks": [{
          "type": "agent",
          "prompt": "A task was marked complete: $ARGUMENTS\n\nVerify that:\n1. All mentioned files exist with meaningful content\n2. No TODO/placeholder text remains\n3. Any mentioned tests pass\n\nReturn ok: true only if all criteria are met.",
          "timeout": 120,
          "statusMessage": "Verifying task completion..."
        }]
      }
    ]
  }
}
```

### Example 9: Config Change Auditing

```bash
#!/usr/bin/env bash
read -r hook_input
source=$(echo "$hook_input" | jq -r '.source')
file=$(echo "$hook_input" | jq -r '.file_path // "unknown"')

echo "[$(date -Iseconds)] Config change: source=$source file=$file" >> ~/.claude/config_audit.log
echo "$hook_input" >> ~/.claude/config_audit.jsonl

if [[ "$source" == "policy_settings" ]]; then
  echo "Policy settings changes require administrator approval" >&2
  exit 2
fi
```

### Example 10: Prompt Hook for Permission Decisions

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "matcher": "Bash",
        "hooks": [{
          "type": "prompt",
          "prompt": "A permission request for Bash: $ARGUMENTS\n\nIs this safe to auto-approve? Consider: system file modifications, global software installs, unexpected network access, irreversible effects.\n\nReturn ok: true if safe, ok: false with reason if user should be asked.",
          "model": "claude-haiku-4-5",
          "timeout": 10
        }]
      }
    ]
  }
}
```

### Example 11: Modify Tool Input Before Execution

```python
#!/usr/bin/env python3
import json, sys

hook_input = json.load(sys.stdin)
if hook_input.get('tool_name') != 'Bash':
    sys.exit(0)

cmd = hook_input.get('tool_input', {}).get('command', '')
if not cmd.startswith('timeout '):
    safe_cmd = f'timeout 30 bash -c {json.dumps(cmd)}'
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "updatedInput": {"command": safe_cmd},
            "permissionDecision": "allow"
        }
    }))
    sys.exit(0)
sys.exit(0)
```

### Example 12: Subagent Context Injection

```bash
#!/usr/bin/env bash
read -r hook_input
agent_type=$(echo "$hook_input" | jq -r '.agent_type')

case "$agent_type" in
  engineer)
    ctx="Follow coding standards in CONTRIBUTING.md. Project: $(basename $PWD)."
    ;;
  reviewer)
    ctx="Review for security, performance, and correctness against team standards."
    ;;
  *) exit 0 ;;
esac

jq -n --arg c "$ctx" '{hookSpecificOutput:{hookEventName:"SubagentStart",additionalContext:$c}}'
```

### Example 13: asyncRewake for Background Test Verification

```bash
#!/usr/bin/env bash
# Config: asyncRewake: true

echo '{"async": true}'

if ! npm test --silent > /tmp/test_output.txt 2>&1; then
  echo "Tests failed after your changes:" >&2
  tail -20 /tmp/test_output.txt >&2
  exit 2  # asyncRewake causes model to wake and see this
fi
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [{ "type": "command", "command": "bash ~/.claude/hooks/bg_tests.sh", "asyncRewake": true, "timeout": 300 }]
      }
    ]
  }
}
```

---

## Quick Reference Tables

### Event Summary Table

| Event | Matcher Field | Blocking | HTTP Allowed |
|-------|--------------|----------|--------------|
| PreToolUse | `tool_name` | Full | Yes |
| PostToolUse | `tool_name` | Partial | Yes |
| PostToolUseFailure | `tool_name` | Partial | Yes |
| PermissionRequest | `tool_name` | Full | Yes |
| Notification | `notification_type` | None | Yes |
| UserPromptSubmit | *(none)* | Full | Yes |
| SessionStart | `source` | None | **No** |
| SessionEnd | `reason` | None | Yes |
| Stop | *(none)* | Partial | Yes |
| SubagentStart | `agent_type` | None | Yes |
| SubagentStop | `agent_type` | Partial | Yes |
| PreCompact | `trigger` | Full | Yes |
| TeammateIdle | *(none)* | Full | Yes |
| TaskCompleted | *(none)* | Full | Yes |
| Elicitation | `mcp_server_name` | Full | Yes |
| ElicitationResult | `mcp_server_name` | Full | Yes |
| ConfigChange | `source` | Full | Yes |
| InstructionsLoaded | `load_reason` | None | Yes |
| Setup | `trigger` | None | **No** |
| WorktreeCreate | *(none)* | None | Yes |
| WorktreeRemove | *(none)* | None | Yes |

### Hook Type Comparison

| Feature | Command | Prompt | Agent | HTTP |
|---------|---------|--------|-------|------|
| Execution | Shell process | Anthropic API | Subagent session | HTTP POST |
| Default timeout | 30s | 30s | 60s | 30s |
| Max turns | N/A | N/A | 50 | N/A |
| Async support | Yes | No | No | No |
| asyncRewake | Yes | No | No | No |
| File system access | Yes (shell) | No | Yes (tools) | No |
| Network access | Yes (shell) | API only | Yes (tools) | Yes (restricted) |
| Private IP blocked | No | No | No | Yes |
| Allowed: SessionStart | Yes | Yes | Yes | **No** |
| Allowed: Setup | Yes | Yes | Yes | **No** |
| Best for | Scripted logic | Semantic evaluation | Complex verification | External webhooks |

### hookSpecificOutput Support

| Event | additionalContext | permissionDecision | updatedInput | updatedMCPToolOutput | PermReq decision | Elicit action |
|-------|:---:|:---:|:---:|:---:|:---:|:---:|
| PreToolUse | YES | YES | YES | - | - | - |
| PostToolUse | YES | - | - | YES | - | - |
| PostToolUseFailure | YES | - | - | - | - | - |
| UserPromptSubmit | YES | - | - | - | - | - |
| SessionStart | YES | - | - | - | - | - |
| Setup | YES | - | - | - | - | - |
| SubagentStart | YES | - | - | - | - | - |
| Notification | YES | - | - | - | - | - |
| PermissionRequest | - | - | - | - | YES | - |
| Elicitation | - | - | - | - | - | YES |
| ElicitationResult | - | - | - | - | - | YES |
| All others | - | - | - | - | - | - |

### Settings File Locations

| Scope | File Path | Priority |
|-------|-----------|----------|
| Managed/Policy | `~/.claude/managed-settings.json` | Highest (1) |
| Plugin | `~/.claude/plugins/*/manifest.json` | 2 |
| Project | `.claude/settings.json` | 3 |
| Local/User | `~/.claude/local-settings.json` | Lowest (4) |

### Source File Reference

| Component | File | Lines / Function |
|-----------|------|------------------|
| Hook dispatcher | `hasworktreecreatehook-1.ts` | 800-1300 (AB()) |
| Hook matcher | `hasworktreecreatehook-1.ts` | 663-783 (Oe8()) |
| Pattern matching | `hasworktreecreatehook-1.ts` | Ghz() |
| Command executor | `hasworktreecreatehook-1.ts` | 370-600 (ob1()) |
| Prompt executor | `tool-2.ts` | 4761-4870 (wIq()) |
| Agent executor | `tool-2.ts` | 4923-5080 (OIq()) |
| HTTP executor | `markdown-1.ts` | 976-1020 (_e8()) |
| Hook type schemas | `cli.js` | 65223-65367 (c73()) |
| Hook loading/merging | `jm-2.ts` | J18() |
| Plugin hook loading | `setuppluginhookhotreload-3.ts` | QHY() |
| Hot reload | `setuppluginhookhotreload-3.ts` | KQ(), of1() |
| Top-level output schema | (internal) | whz |
| Async response schema | (internal) | oE6 |
| Permission rule types | (internal) | db1 |
| Private IP DNS check | `markdown-1.ts` | XIq() |
| URL allowlist | (internal) | Xhz(), Dhz() |
| Session hook storage | `appState` | sessionHooks[sessionId] |
| Session hook register | (internal) | fcA() |
| Session hook retrieve | (internal) | H51() / VcA() |
| Session hook cleanup | (internal) | TcA() / rM6() |
| Workspace trust check | (internal) | rb1() |
| Simple mode check | (internal) | bI6() |

### Environment Variables Available to Command Hooks

| Variable | Present | Events |
|----------|---------|--------|
| `CLAUDE_PROJECT_DIR` | Always | All |
| `CLAUDE_ENV_FILE` | Conditional | SessionStart, Setup only |
| `CLAUDE_PLUGIN_ROOT` | Conditional | Plugin hooks only |
| All `process.env` vars | Always | All (inherited) |

### Notification Types

| notification_type | When |
|-------------------|------|
| `permission_prompt` | Permission dialog shown to user |
| `idle_prompt` | Claude is idle, waiting for input |
| `auth_success` | Authentication succeeded |
| `elicitation_dialog` | MCP server opened elicitation UI |
| `elicitation_complete` | Elicitation dialog closed |
| `elicitation_response` | User submitted elicitation response |

### SessionStart Source Values

| source | When |
|--------|------|
| `startup` | First session launch |
| `resume` | Resuming a saved session |
| `clear` | Conversation cleared, new session |
| `compact` | After conversation compaction |

### SessionEnd Reason Values

| reason | When |
|--------|------|
| `clear` | User cleared the conversation |
| `logout` | User logged out |
| `prompt_input_exit` | User exited from prompt input |
| `other` | Any other reason |

### ConfigChange Source Values

| source | Description |
|--------|-------------|
| `user_settings` | User-level settings |
| `project_settings` | Project `.claude/settings.json` |
| `local_settings` | Local/device settings |
| `policy_settings` | Managed policy settings |
| `skills` | Skill configuration |

### InstructionsLoaded Reason Values

| load_reason | Description |
|-------------|-------------|
| `session_start` | Loaded at session initialization |
| `nested_traversal` | Found by traversing parent directories |
| `path_glob_match` | Matched a configured glob pattern |
| `include` | Explicitly included by another CLAUDE.md |

### Setup Trigger Values

| trigger | Description |
|---------|-------------|
| `init` | Initial repository setup |
| `maintenance` | Periodic maintenance tasks |

### PreCompact Trigger Values

| trigger | Description |
|---------|-------------|
| `manual` | User explicitly requested compaction |
| `auto` | Context window threshold triggered |

### InstructionsLoaded Memory Types

| memory_type | Description |
|-------------|-------------|
| `User` | User-level CLAUDE.md (`~/.claude/CLAUDE.md`) |
| `Project` | Project-level CLAUDE.md (`.claude/CLAUDE.md`) |
| `Local` | Local/device-level instructions |
| `Managed` | Policy-managed instructions |

### PermissionRequest Decision Behaviors

| behavior | Effect |
|----------|--------|
| `allow` | Permission granted; optionally modify input or add permission rules |
| `deny` | Permission denied; optionally show message or interrupt agent |

### PermissionRule Types

| type | Effect |
|------|--------|
| `addRules` | Add new permission rules to the specified destination |
| `replaceRules` | Replace existing rules at destination |
| `removeRules` | Remove matching rules from destination |
| `setMode` | Set overall permission mode |
| `addDirectories` | Add directories to allowed list |

### Elicitation Action Values

| action | Description |
|--------|-------------|
| `accept` | Accept the elicitation and provide content |
| `decline` | Decline the elicitation |
| `cancel` | Cancel the elicitation dialog |

### Agent Hook Outcome Values

| Outcome | Condition | Effect |
|---------|-----------|--------|
| `success` | Agent returns `ok: true` | Hook passes, execution continues |
| `blocking` | Agent returns `ok: false` | Hook blocks; `reason` shown as message |
| `cancelled` | 50-turn limit hit, or no output | Hook skipped, treated as non-blocking |

### Permission Mode Values

| permission_mode | Description |
|----------------|-------------|
| `default` | Standard permission prompting |
| `acceptEdits` | Automatically accept file edits |
| `plan` | Plan-only mode, no execution |
| `bypassPermissions` | All permissions auto-approved |

### Schema Reference Summary

| Schema Name | Source Name | Used For |
|-------------|-------------|----------|
| `HookOutput` | `whz` | Top-level output from all hooks |
| `HookResponse` | `oE6` | Union of AsyncMarker or HookOutput |
| `HookSpecificOutput` | discriminated union | Event-specific output fields |
| `PermissionRule` | `db1` | Permission modifications in PermissionRequest |
| `HookUnion` | discriminated union | The 4 hook types |
| `HookConfigEntry` | (inline) | Matcher + hooks array in config files |
| `BaseHookInput` | `R$()` output | Common fields injected in all event inputs |

---

*End of Claude Code v2.1.71 Hooks System Reference*
