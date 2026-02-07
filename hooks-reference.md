# Claude Code CLI - Hooks System Reference

Comprehensive technical reference for the Claude Code CLI hooks system. This document merges detailed research from multiple source files covering events, configuration, execution, matching, integration, and data flow.

**Last Updated:** 2026-02-06

---

## Table of Contents

1. [Overview](#1-overview)
2. [Hook Configuration](#2-hook-configuration)
3. [Hook Event Types](#3-hook-event-types)
4. [Matching System](#4-matching-system)
5. [Execution Engine](#5-execution-engine)
6. [Hook Output Format](#6-hook-output-format)
7. [Tool Pipeline Integration](#7-tool-pipeline-integration)
8. [Permission System Integration](#8-permission-system-integration)
9. [Hook Event Messages](#9-hook-event-messages)
10. [Telemetry Events](#10-telemetry-events)
11. [Key Functions Reference](#11-key-functions-reference)
12. [Configuration Examples](#12-configuration-examples)
13. [Architecture Diagram](#13-architecture-diagram)

---

## 1. Overview

### What Are Hooks?

Hooks in Claude Code CLI are user-defined scripts, prompts, or agents that execute at specific lifecycle events. They enable observing, modifying, blocking, or enhancing tool execution behavior through a well-defined integration architecture.

### How Hooks Work

1. **Event Triggering**: Lifecycle events (PreToolUse, PostToolUse, SessionStart, etc.) trigger configured hooks
2. **Matching**: Hooks are filtered based on matcher patterns (tool names, notification types, etc.)
3. **Execution**: Matching hooks execute in parallel with controlled concurrency
4. **Data Flow**: Hooks receive JSON via stdin and return JSON via stdout
5. **Integration**: Hook results modify tool execution flow (permissions, inputs, outputs)

### Hook Types

Claude Code CLI supports 5 hook types:

#### 1. Command Hooks
```typescript
{
  type: "command",
  command: string,              // Shell command to execute
  timeout?: number,             // Timeout in seconds (optional)
  statusMessage?: string,       // Custom spinner message (optional)
  once?: boolean,              // Run once then remove (optional)
  async?: boolean              // Run in background (optional)
}
```
Spawns external process via shell. Most common hook type.

#### 2. Prompt Hooks
```typescript
{
  type: "prompt",
  prompt: string,               // LLM prompt template (can use $ARGUMENTS)
  timeout?: number,             // Timeout in seconds (optional)
  model?: string,               // Model ID (e.g., "claude-sonnet-4-5-20250929")
  statusMessage?: string,       // Custom spinner message (optional)
  once?: boolean               // Run once then remove (optional)
}
```
Interactive prompts for user input.

#### 3. Agent Hooks
```typescript
{
  type: "agent",
  prompt: string,               // Agent instructions (can use $ARGUMENTS)
  timeout?: number,             // Timeout in seconds (default: 60)
  model?: string,               // Model ID (defaults to Haiku if not specified)
  statusMessage?: string,       // Custom spinner message (optional)
  once?: boolean               // Run once then remove (optional)
}
```
Spawns sub-agent to handle hook logic.

#### 4. Callback Hooks
```typescript
{
  type: "callback",
  // JavaScript function executed in-process (no spawn)
}
```
JavaScript functions executed in-process. Used internally for plugin hooks.


#### 5. Function Hooks (REPL-only)
```typescript
{
  type: "function"
  // Only valid in REPL context (Stop hooks)
}
```
Only valid in REPL context.



---

## 2. Hook Configuration

### Settings File Schema


Hooks are configured in JSON settings files with the following structure:

```typescript
{
  hooks?: {
    [EventName: HookEvent]: Array<{
      matcher?: string,           // Tool name pattern (e.g., "Bash", "Write|Edit")
      hooks: Array<Hook>          // Array of hook configurations
    }>
  },
  disableAllHooks?: boolean,
  allowManagedHooksOnly?: boolean
}
```

**Zod Schema Variable:** `dE` (defined at line 947)
```typescript
dE = b.partialRecord(b.enum(Lx), b.array(Vz8))
```

Where:
- `Lx` = enum of hook event names
- `Vz8` = hook container schema (matcher + hooks array)
- `fz8` = discriminated union of the three hook types

### Storage Locations


| Scope | File Path | Git Status | Purpose |
|-------|-----------|------------|------|
| User Settings | `~/.claude/settings.json` | Not in repo | Personal global preferences |
| Project Settings | `.claude/settings.json` | Should commit | Team-wide configuration |
| Local Settings | `.claude/settings.local.json` | Should gitignore | Personal project overrides |
| Policy Settings | Managed settings path | Varies | Enterprise/policy-enforced |
| Plugin Hooks | `~/.claude/plugins/*/hooks/hooks.json` | Plugin-specific | Plugin-provided hooks |
| Session Hooks | In-memory | Temporary | Runtime-only hooks |

**File Resolution Functions:**
- `Pw(scope)` - Returns full path for a given scope
- `b$1(scope)` - Returns base directory for a scope
- `u$1(scope)` - Returns filename for project/local settings

### Hook Precedence and Priority

#### Source Priority (Highest to Lowest)


1. **Policy Settings** (managed-settings.json)
   - If `allowManagedHooksOnly === true`, these are the ONLY hooks loaded
   - Cannot be overridden by user
2. **Local Settings** (.claude/settings.local.json)
   - Project-specific personal overrides
3. **Project Settings** (.claude/settings.json)
   - Team-wide configuration
4. **User Settings** (~/.claude/settings.json)
   - Personal global defaults
5. **Session Hooks** (in-memory)
   - Temporary, cleared on session end
6. **Plugin Hooks** (plugin-provided)
   - From enabled plugins

#### Loading Logic

**Loading Order (`iT7` function, line 8208):**
```typescript
1. Check if allowManagedHooksOnly === true
   - If true, ONLY load policy settings hooks
   - If false, load user/project/local hooks
2. Iterate through settings sources in order
3. For each source, extract hooks by event type
4. Add session hooks (runtime-only)
5. Add plugin hooks (from enabled plugins)
```

#### Execution Order Within a Single Event

1. Hooks execute in the order they appear in the `hooks` array
2. Multiple sources combine their hooks
3. Duplicates are preserved (hooks can run multiple times)
4. No explicit priority or weight system

### Configuration Flags


#### disableAllHooks
```typescript
disableAllHooks?: boolean  // Disable all hooks and statusLine
```
Completely disables the hooks system.

#### allowManagedHooksOnly
```typescript
allowManagedHooksOnly?: boolean  // If true, blocks user-defined hooks
```

**Warning Message:**
```
"Only hooks from managed settings can run. User-defined hooks from
~/.claude/settings.json, .claude/settings.json, and
.claude/settings.local.json are blocked."
```

### Hook Lifecycle Management

#### Adding Hooks

**Programmatic API** (`nT7` function, line 8240):
```typescript
async function nT7(
  event: HookEvent,
  hook: Hook,
  matcher: string = "",
  source: SettingsSource = "userSettings"
)
```

#### Removing Hooks

**Programmatic API** (`rT7` function, line 8255):
```typescript
async function rT7(hookMetadata: {
  source: SettingsSource,
  event: HookEvent,
  matcher: string,
  config: Hook
})
```

**Restrictions:**
- Plugin hooks cannot be removed via settings (disable plugin instead)
- Session hooks cannot be removed via settings (temporary only)

### Plugin Hooks


Plugins can provide hooks via:
```typescript
{
  description?: string,
  hooks: HookSchema  // Same schema as user hooks
}

// Or via file reference:
{
  hooks: string | HookSchema | Array<string | HookSchema>
  // String paths are relative to plugin root
}
```

**Plugin Hook Path:** `~/.claude/plugins/*/hooks/hooks.json`

---

## 3. Hook Event Types

**Source Files:**
- Hook event type definitions
- Hook event metadata and descriptions

All 15 hook event types are defined as Zod schemas and support both synchronous and asynchronous execution.

### Complete Event List

1. PreToolUse
2. PostToolUse
3. PostToolUseFailure
4. PermissionRequest
5. Notification
6. UserPromptSubmit
7. SessionStart
8. SessionEnd
9. Setup
10. Stop
11. SubagentStart
12. SubagentStop
13. PreCompact
14. TeammateIdle
15. TaskCompleted


### 1. PreToolUse

**Summary:** Before tool execution

**When it fires:** Immediately before a tool is called, allowing inspection and modification of tool arguments.

**Data Schema:**
```typescript
{
  hook_event_name: "PreToolUse",
  tool_name: string,
  tool_input: unknown,
  tool_use_id: string
}
```

**Matcher Metadata:**
- `fieldToMatch`: "tool_name"
- `values`: Array of all available tool names
- Allows hooks to match specific tools (e.g., only run for "Bash" or "Edit" tools)

**Exit Code Behavior:**
- Exit code 0: stdout/stderr not shown
- Exit code 2: show stderr to model and block tool call
- Other exit codes: show stderr to user only but continue with tool call

**Hook Specific Output Schema:**
```typescript
{
  hookEventName: "PreToolUse",
  permissionDecision: "allow" | "deny" | "ask" (optional),
  permissionDecisionReason: string (optional),
  updatedInput: Record<string, unknown> (optional),
  additionalContext: string (optional)
}
```


---

### 2. PostToolUse

**Summary:** After tool execution

**When it fires:** After a tool has successfully executed, allowing inspection of both inputs and outputs.

**Data Schema:**
```typescript
{
  hook_event_name: "PostToolUse",
  tool_name: string,
  tool_input: unknown,
  tool_response: unknown,
  tool_use_id: string
}
```

**Matcher Metadata:**
- `fieldToMatch`: "tool_name"
- `values`: Array of all available tool names
- Allows hooks to match specific tools

**Exit Code Behavior:**
- Exit code 0: stdout shown in transcript mode (ctrl+o)
- Exit code 2: show stderr to model immediately
- Other exit codes: show stderr to user only

**Hook Specific Output Schema:**
```typescript
{
  hookEventName: "PostToolUse",
  additionalContext: string (optional),
  updatedMCPToolOutput: unknown (optional)
}
```


---

### 3. PostToolUseFailure

**Summary:** After tool execution fails

**When it fires:** When a tool call fails with an error, allowing custom error handling.

**Data Schema:**
```typescript
{
  hook_event_name: "PostToolUseFailure",
  tool_name: string,
  tool_input: unknown,
  tool_use_id: string,
  error: string,
  is_interrupt: boolean (optional)
}
```

**Matcher Metadata:**
- `fieldToMatch`: "tool_name"
- `values`: Array of all available tool names
- Allows hooks to match specific tools

**Exit Code Behavior:**
- Exit code 0: stdout shown in transcript mode (ctrl+o)
- Exit code 2: show stderr to model immediately
- Other exit codes: show stderr to user only

**Hook Specific Output Schema:**
```typescript
{
  hookEventName: "PostToolUseFailure",
  additionalContext: string (optional)
}
```


---

### 4. PermissionRequest

**Summary:** When a permission dialog is displayed

**When it fires:** When the user is prompted to approve a tool call, allowing automated permission decisions.

**Data Schema:**
```typescript
{
  hook_event_name: "PermissionRequest",
  tool_name: string,
  tool_input: unknown,
  permission_suggestions: Array<PermissionSuggestion> (optional)
}
```

**Matcher Metadata:**
- `fieldToMatch`: "tool_name"
- `values`: Array of all available tool names
- Allows hooks to match specific tools requiring permissions

**Exit Code Behavior:**
- Exit code 0: use hook decision if provided
- Other exit codes: show stderr to user only

**Hook Specific Output Schema:**
```typescript
{
  hookEventName: "PermissionRequest",
  decision: {
    behavior: "allow",
    updatedInput?: Record<string, unknown>,
    updatedPermissions?: Array<PermissionSuggestion>
  } | {
    behavior: "deny",
    message?: string,
    interrupt?: boolean
  }
}
```


---

### 5. Notification

**Summary:** When notifications are sent

**When it fires:** When the system sends notifications to the user.

**Data Schema:**
```typescript
{
  hook_event_name: "Notification",
  message: string,
  title: string (optional),
  notification_type: string
}
```

**Matcher Metadata:**
- `fieldToMatch`: "notification_type"
- `values`: ["permission_prompt", "idle_prompt", "auth_success", "elicitation_dialog"]
- Allows hooks to match specific notification types

**Exit Code Behavior:**
- Exit code 0: stdout/stderr not shown
- Other exit codes: show stderr to user only

**Hook Specific Output Schema:**
```typescript
{
  hookEventName: "Notification",
  additionalContext: string (optional)
}
```


---

### 6. UserPromptSubmit

**Summary:** When the user submits a prompt

**When it fires:** When the user submits input to Claude.

**Data Schema:**
```typescript
{
  hook_event_name: "UserPromptSubmit",
  prompt: string
}
```

**Matcher Metadata:** None (no matcher field available)

**Exit Code Behavior:**
- Exit code 0: stdout shown to Claude
- Exit code 2: block processing, erase original prompt, and show stderr to user only
- Other exit codes: show stderr to user only

**Hook Specific Output Schema:**
```typescript
{
  hookEventName: "UserPromptSubmit",
  additionalContext: string (optional)
}
```


---

### 7. SessionStart

**Summary:** When a new session is started

**When it fires:** When Claude Code starts a new conversation session.

**Data Schema:**
```typescript
{
  hook_event_name: "SessionStart",
  source: "startup" | "resume" | "clear" | "compact",
  agent_type: string (optional),
  model: string (optional)
}
```

**Matcher Metadata:**
- `fieldToMatch`: "source"
- `values`: ["startup", "resume", "clear", "compact"]
- Allows hooks to match specific session start triggers

**Exit Code Behavior:**
- Exit code 0: stdout shown to Claude
- Blocking errors are ignored
- Other exit codes: show stderr to user only

**Hook Specific Output Schema:**
```typescript
{
  hookEventName: "SessionStart",
  additionalContext: string (optional)
}
```


---

### 8. SessionEnd

**Summary:** When a session is ending

**When it fires:** When the conversation session is terminating.

**Data Schema:**
```typescript
{
  hook_event_name: "SessionEnd",
  reason: "clear" | "logout" | "prompt_input_exit" | "other" | "bypass_permissions_disabled"
}
```

**Matcher Metadata:**
- `fieldToMatch`: "reason"
- `values`: ["clear", "logout", "prompt_input_exit", "other"]
- Allows hooks to match specific session end reasons

**Exit Code Behavior:**
- Exit code 0: command completes successfully
- Other exit codes: show stderr to user only

**Hook Specific Output Schema:** None defined (general hook output only)


---

### 9. Setup

**Summary:** Repo setup hooks for init and maintenance

**When it fires:** During repository initialization or maintenance operations.

**Data Schema:**
```typescript
{
  hook_event_name: "Setup",
  trigger: "init" | "maintenance"
}
```

**Matcher Metadata:**
- `fieldToMatch`: "trigger"
- `values`: ["init", "maintenance"]
- Allows hooks to match specific setup triggers

**Exit Code Behavior:**
- Exit code 0: stdout shown to Claude
- Blocking errors are ignored
- Other exit codes: show stderr to user only

**Hook Specific Output Schema:**
```typescript
{
  hookEventName: "Setup",
  additionalContext: string (optional)
}
```


---

### 10. Stop

**Summary:** Right before Claude concludes its response

**When it fires:** Just before Claude finishes generating a response.

**Data Schema:**
```typescript
{
  hook_event_name: "Stop",
  stop_hook_active: boolean
}
```

**Matcher Metadata:** None (no matcher field available)

**Exit Code Behavior:**
- Exit code 0: stdout/stderr not shown
- Exit code 2: show stderr to model and continue conversation
- Other exit codes: show stderr to user only

**Hook Specific Output Schema:** None defined (general hook output only)


Stop hook event handling

---

### 11. SubagentStart

**Summary:** When a subagent (Task tool call) is started

**When it fires:** When a Task tool creates a new subagent.

**Data Schema:**
```typescript
{
  hook_event_name: "SubagentStart",
  agent_id: string,
  agent_type: string
}
```

**Matcher Metadata:**
- `fieldToMatch`: "agent_type"
- `values`: [] (empty array, dynamically populated)
- Allows hooks to match specific agent types

**Exit Code Behavior:**
- Exit code 0: stdout shown to subagent
- Blocking errors are ignored
- Other exit codes: show stderr to user only

**Hook Specific Output Schema:**
```typescript
{
  hookEventName: "SubagentStart",
  additionalContext: string (optional)
}
```


---

### 12. SubagentStop

**Summary:** Right before a subagent (Task tool call) concludes its response

**When it fires:** Just before a subagent finishes its execution.

**Data Schema:**
```typescript
{
  hook_event_name: "SubagentStop",
  stop_hook_active: boolean,
  agent_id: string,
  agent_transcript_path: string,
  agent_type: string
}
```

**Matcher Metadata:**
- `fieldToMatch`: "agent_type"
- `values`: [] (empty array, dynamically populated)
- Allows hooks to match specific agent types

**Exit Code Behavior:**
- Exit code 0: stdout/stderr not shown
- Exit code 2: show stderr to subagent and continue having it run
- Other exit codes: show stderr to user only

**Hook Specific Output Schema:** None defined (general hook output only)


SubagentStop hook event handling

---

### 13. PreCompact

**Summary:** Before conversation compaction

**When it fires:** Before the conversation history is compacted to save tokens.

**Data Schema:**
```typescript
{
  hook_event_name: "PreCompact",
  trigger: "manual" | "auto",
  custom_instructions: string | null
}
```

**Matcher Metadata:**
- `fieldToMatch`: "trigger"
- `values`: ["manual", "auto"]
- Allows hooks to match manual vs automatic compaction

**Exit Code Behavior:**
- Exit code 0: stdout appended as custom compact instructions
- Exit code 2: block compaction
- Other exit codes: show stderr to user only but continue with compaction

**Hook Specific Output Schema:** None defined (general hook output only)


---

### 14. TeammateIdle

**Summary:** When a teammate is about to go idle

**When it fires:** When a teammate in a team session is about to become idle.

**Data Schema:**
```typescript
{
  hook_event_name: "TeammateIdle",
  teammate_name: string,
  team_name: string
}
```

**Matcher Metadata:** None (no matcher field available)

**Exit Code Behavior:**
- Exit code 0: stdout/stderr not shown
- Exit code 2: show stderr to teammate and prevent idle (teammate continues working)
- Other exit codes: show stderr to user only

**Hook Specific Output Schema:** None defined (general hook output only)


---

### 15. TaskCompleted

**Summary:** When a task is being marked as completed

**When it fires:** When a task is about to be marked as complete in the task system.

**Data Schema:**
```typescript
{
  hook_event_name: "TaskCompleted",
  task_id: string,
  task_subject: string,
  task_description: string (optional),
  teammate_name: string (optional),
  team_name: string (optional)
}
```

**Matcher Metadata:** None (no matcher field available)

**Exit Code Behavior:**
- Exit code 0: stdout/stderr not shown
- Exit code 2: show stderr to model and prevent task completion
- Other exit codes: show stderr to user only

**Hook Specific Output Schema:** None defined (general hook output only)


---

### Events with Matcher Support vs. No Matcher

**Events with matcher metadata:**
- PreToolUse, PostToolUse, PostToolUseFailure, PermissionRequest → match by `tool_name`
- Notification → match by `notification_type`
- SessionStart → match by `source`
- SessionEnd → match by `reason`
- Setup → match by `trigger`
- PreCompact → match by `trigger`
- SubagentStart, SubagentStop → match by `agent_type`

**Events without matcher metadata (run for all instances):**
- UserPromptSubmit
- Stop
- TeammateIdle
- TaskCompleted


---

## 4. Matching System

### Overview

The Claude Code CLI hooks system uses a two-level matching approach:
1. **Event matching**: Which lifecycle event (PreToolUse, PostToolUse, etc.)
2. **Matcher field matching**: Which specific tools/values trigger the hook

### The ARY Matching Function


This is the core matching function that determines if a tool name matches a hook matcher pattern:

```typescript
function ARY(A, q) {
  // A = actual value (e.g., tool name "Bash")
  // q = matcher pattern (e.g., "Bash|Write" or "*" or regex)
  
  // Match all: empty matcher or wildcard
  if (!q || q === "*") return true;
  
  // Pipe-separated values (e.g., "Bash|Write|Edit")
  if (/^[a-zA-Z0-9_|]+$/.test(q)) {
    if (q.includes("|"))
      return q
        .split("|")
        .map((Y) => Y.trim())
        .includes(A);
    // Single exact match
    return A === q;
  }
  
  // Regex pattern matching
  try {
    return new RegExp(q).test(A);
  } catch {
    h(`Invalid regex pattern in hook matcher: ${q}`);
    return false;
  }
}
```

### Matching Algorithm Features

#### 1. Wildcard Matching (`*` or empty matcher)
- Matches any value
- Useful for hooks that should run on all tools

#### 2. Pipe-Separated OR Matching (`"Tool1|Tool2|Tool3"`)
- Must contain only alphanumeric characters, underscores, and pipes
- Splits on `|` and checks if actual value equals any option
- Values are trimmed of whitespace

#### 3. Exact Matching (`"ToolName"`)
- Simple string equality check
- Only used if no pipes present

#### 4. Regex Pattern Matching
- Any pattern that doesn't fit the alphanumeric+pipe format
- Allows complex matching patterns
- Invalid regex patterns are logged and return false

### Matching Examples

| Matcher Pattern | Tool Name | Matches? | Reason |
|-----------------|-----------|----------|--------|
| `"Bash"` | `"Bash"` | Yes | Exact match |
| `"Bash"` | `"Write"` | No | No match |
| `"Bash|Write|Edit"` | `"Write"` | Yes | Pipe-separated match |
| `"Bash|Write|Edit"` | `"Read"` | No | Not in list |
| `"*"` | `"Bash"` | Yes | Wildcard |
| `""` (empty) | `"Read"` | Yes | Empty matches all |
| `"^mcp__.*"` | `"mcp__slack__read"` | Yes | Regex match |
| `"Read|Write"` | `"Read"` | Yes | First in pipe list |

### Field to Match Mapping


The `ikA` function retrieves and filters hooks for a specific event by extracting the appropriate field:

```typescript
function ikA(A, q, K, Y) {
  // Extract the field value based on event type
  switch (Y.hook_event_name) {
    case "PreToolUse":
    case "PostToolUse":
    case "PostToolUseFailure":
    case "PermissionRequest":
      H = Y.tool_name;  // Tool name for tool events
      break;
    case "SessionStart":
      H = Y.source;  // Session source
      break;
    case "Setup":
    case "PreCompact":
      H = Y.trigger;  // Trigger type
      break;
    case "Notification":
      H = Y.notification_type;  // Notification type
      break;
    case "SessionEnd":
      H = Y.reason;  // End reason
      break;
    case "SubagentStart":
    case "SubagentStop":
      H = Y.agent_type;  // Agent type
      break;
    // Events without matchers
    case "TeammateIdle":
    case "TaskCompleted":
      break;
  }
  
  // Filter hooks using the matching algorithm
  let O = (H
    ? w.filter((W) => !W.matcher || ARY(H, W.matcher))
    : w
  ).flatMap((W) => {
    // Map to execution format with plugin metadata
  });
}
```

**Key Logic:**
1. Extract the appropriate field value from the hook input based on event type
2. Get all configured hooks for this event
3. Filter hooks where: `!W.matcher` (no matcher = match all) OR `ARY(actualValue, matcher)` returns true
4. Flatten the results and return

### matcherMetadata Schema


Each hook event has optional `matcherMetadata` that defines how hooks are matched:

```typescript
{
  fieldToMatch: string,  // The field name to match on (e.g., "tool_name", "agent_type")
  values: string[]       // Possible values for UI autocomplete
}
```

### Hook Events and Their Matchers

| Event | fieldToMatch | Description |
|-------|--------------|-------------|
| **PreToolUse** | `tool_name` | Matches by tool name (Bash, Read, Edit, etc.) |
| **PostToolUse** | `tool_name` | Matches by tool name |
| **PostToolUseFailure** | `tool_name` | Matches by tool name |
| **PermissionRequest** | `tool_name` | Matches by tool name |
| **Notification** | `notification_type` | Matches notification types: `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog` |
| **SessionStart** | `source` | Matches session sources: `startup`, `resume`, `clear`, `compact` |
| **SessionEnd** | `reason` | Matches end reasons: `clear`, `logout`, `prompt_input_exit`, `other` |
| **SubagentStart** | `agent_type` | Matches by agent type |
| **SubagentStop** | `agent_type` | Matches by agent type |
| **PreCompact** | `trigger` | Matches compaction triggers: `manual`, `auto` |
| **Setup** | `trigger` | Matches setup triggers: `init`, `maintenance` |
| **UserPromptSubmit** | (none) | No matcher - applies to all |
| **Stop** | (none) | No matcher - applies to all |
| **TeammateIdle** | (none) | No matcher - applies to all |
| **TaskCompleted** | (none) | No matcher - applies to all |

### Tool Name Format

Tool names in matchers correspond to the tool's internal name:
- **Built-in tools:** `Bash`, `Read`, `Write`, `Edit`, `Grep`, `Glob`, `NotebookEdit`, `WebSearch`, `WebFetch`, `Skill`, `SendMessage`, etc.
- **MCP tools:** Full namespaced name with format `mcp__<server>__<tool>` (e.g., `mcp__slack__read_channel`, `mcp__filesystem__list_directory`)

### MCP Tool Hooks


MCP (Model Context Protocol) tools follow the same matching system:

```typescript
let z = $V1();  // Get MCP plugin hooks
if (z)
  for (let [w, H] of Object.entries(z)) {
    let $ = w,      // Event type
      O = K[$];     // Hook map for this event
    if (!O) continue;
    for (let _ of H) {
      let J = _.matcher || "";  // Matcher for MCP hook
      for (let X of _.hooks)
        if (X.type === "callback") {
          if (!O[J]) O[J] = [];
          O[J].push({
            event: $,
            config: { type: "command", command: "[Plugin Hook]" },
            matcher: _.matcher,
            source: "pluginHook",
            pluginName: _.pluginName,
          });
```

MCP plugin hooks:
- Are registered through the plugin system
- Follow the exact same matcher logic
- Can use pipe-separated values, wildcards, or regex
- Example: Match all tools from a specific MCP server with `"^mcp__myserver__.*"`

### Matching Priority and Ordering

**Execution Order:**
1. Hooks are stored in arrays within the map structure
2. When multiple hooks match, they execute in **configuration order**
3. No explicit priority or weight system
4. Plugin hooks are added after user-configured hooks

**Example:** If you configure:
```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Write", "hooks": [{ "command": "first.sh" }] },
      { "matcher": "Write|Edit", "hooks": [{ "command": "second.sh" }] },
      { "matcher": "*", "hooks": [{ "command": "third.sh" }] }
    ]
  }
}
```

When `Write` is executed, all three hooks match and run in order:
1. `first.sh` (exact match)
2. `second.sh` (pipe-separated match)
3. `third.sh` (wildcard match)

---

## 5. Execution Engine


The Claude Code CLI uses a sophisticated hook execution system built on Node.js `child_process.spawn` to run external commands during various lifecycle events.

### Hook Command Spawning

#### Primary Spawner Function: `zW6`


The hook command spawner uses Node.js child_process:
```typescript
import { spawn as eLY } from "node:child_process";
```

#### Spawn Configuration

```typescript
// Line 61548
let W = eLY(D, [], {
  env: j,           // Environment variables
  cwd: y6(),        // Working directory
  shell: !0,        // Run through shell
  windowsHide: !0   // Hide window on Windows
})
```

**Key Details:**
- Uses `spawn` from `node:child_process` (imported as `eLY`)
- Always runs through a shell (`shell: true`)
- Empty args array - command is passed as single string
- Windows processes are hidden by default

#### Command Preprocessing

```typescript
// Line 61535-61541
// Replace plugin root placeholder
if ($) X = X.replace(/\$\{CLAUDE_PLUGIN_ROOT\}/g, $);

// Add bash prefix for .sh files on Windows
if (oA() === "windows" && X.trim().match(/\.sh(\s|$|")/)) {
  if (!X.trim().startsWith("bash ")) X = `bash ${X}`;
}

// Apply shell prefix if configured
let D = process.env.CLAUDE_CODE_SHELL_PREFIX
  ? _O6(process.env.CLAUDE_CODE_SHELL_PREFIX, X)
  : X;
```

### Environment Variables Passed to Hooks

**Location:** Lines 61543-61547

```typescript
let j = {
  ...process.env,                    // Inherit all process env vars
  CLAUDE_PROJECT_DIR: J              // Project directory
};

if ($) j.CLAUDE_PLUGIN_ROOT = $;     // Plugin root (if applicable)
if (O) j.CLAUDE_PLUGIN_ROOT = O;     // Skill root (if applicable)

// For SessionStart/Setup hooks only
if ((q === "SessionStart" || q === "Setup") && H !== void 0)
  j.CLAUDE_ENV_FILE = nU7(q, H);     // Path to env file
```

**Standard Environment Variables:**
- `CLAUDE_PROJECT_DIR` - Current project directory
- `CLAUDE_PLUGIN_ROOT` - Plugin/skill root directory (context-dependent)
- `CLAUDE_ENV_FILE` - Env file path (SessionStart/Setup hooks only)
- Plus all inherited `process.env` variables

### stdin Data Passed to Hooks

**Location:** Lines 61556-61637 (stdin writing), 61950 (JSON stringification)

Hooks receive JSON-formatted data via stdin:

```typescript
// Line 61950 - Prepare JSON input
try {
  x = Q1(A);  // Stringify hookInput to JSON
} catch (r) {
  // Error handling
}

// Lines 61636-61639 - Write to stdin
W.stdin.write(Y, "utf8"),  // Y is the JSON string
W.stdin.end(),
```

**JSON Payload Structure:**

The `hookInput` object (variable `A`) varies by hook event but always includes:

```typescript
{
  hook_event_name: string,      // e.g., "PreToolUse", "PostToolUse", etc.
  session_id: string,
  cwd: string,
  permission_mode: string,
  transcript_path: string,

  // Event-specific fields:
  // For PreToolUse:
  tool_name: string,
  tool_input: object,
  tool_use_id: string,

  // For PostToolUse:
  tool_name: string,
  tool_input: object,
  tool_response: any,
  tool_use_id: string,

  // For UserPromptSubmit:
  prompt: string,

  // For SessionStart:
  source: string,
  agent_type: string,
  model: string,

  // etc.
}
```

**Base Schema Function:** `hX(A, q)` at line 61341 creates base hook context with session_id, transcript_path, cwd, permission_mode.

### stdout/stderr Handling

**Location:** Lines 61577-61620

```typescript
let Z = "",    // stdout accumulator
  N = "",      // stderr accumulator
  T = "";      // combined output accumulator

W.stdout.setEncoding("utf8");
W.stderr.setEncoding("utf8");

W.stdout.on("data", (p) => {
  Z += p;      // Accumulate stdout
  T += p;      // Add to combined output

  // Check for async response (lines 61586-61616)
  if (!k && Z.trim().includes("}")) {
    // Parse JSON to detect async hooks
  }
});

W.stderr.on("data", (p) => {
  N += p;      // Accumulate stderr
  T += p;      // Add to combined output
});
```

**Key Points:**
- UTF-8 encoding used for both streams
- All output is accumulated (not streamed to user during execution)
- Early parsing of stdout to detect async hooks
- stderr and stdout are combined in order received

### Timeout Mechanism

#### Default Timeout

**Location:** Line 62774

```typescript
var cj = 600000;  // 600000ms = 10 minutes (default)
```

#### Timeout Implementation

**Location:** Lines 61542, 61906-61907, 61944-61945

```typescript
// Per-hook timeout (line 61542)
let M = A.timeout ? A.timeout * 1000 : cj;  // Hook-specific or default

// Timeout signal creation (line 61945)
let { signal: u, cleanup: S } = oL(AbortSignal.timeout(y), Y);
  // y = hook timeout in ms
  // Y = parent abort signal
```

**Timeout behavior:**
1. Default timeout: 600000ms (10 minutes)
2. Can be overridden per-hook via `hook.timeout` (in seconds)
3. Uses `AbortSignal.timeout()` for timeout enforcement
4. Timeout is combined with parent abort signal (for cancellation)

#### Timeout Wrapper: `OO6`


```typescript
function OO6(A, q, K, Y, z = !1) {
  return new U0A(A, q, K, Y, z);
}
```

Wraps the spawned process with timeout/abort functionality.

### Exit Code Interpretation

**Location:** Lines 62097-62165

#### Exit Code 0: Success

```typescript
// Line 62097
if (U.status === 0) {
  yield {
    message: Zq({ type: "hook_success", ... }),
    outcome: "success",
    hook: Z,
  };
  return;
}
```

**Behavior:** Hook succeeded, continue execution normally.

#### Exit Code 2: Blocking Error

```typescript
// Line 62123
if (U.status === 2) {
  yield {
    blockingError: {
      blockingError: `[${Z.command}]: ${U.stderr || "No stderr output"}`,
      command: Z.command,
    },
    outcome: "blocking",
    hook: Z,
  };
  return;
}
```

**Behavior:** Hook blocks the operation (e.g., prevents tool execution).

#### Other Non-Zero Exit Codes: Non-Blocking Error

```typescript
// Line 62143+
yield {
  message: Zq({
    type: "hook_non_blocking_error",
    stderr: `Failed with non-blocking status code: ${U.stderr.trim() || "No stderr output"}`,
    exitCode: U.status,
    ...
  }),
  outcome: "non_blocking_error",
  hook: Z,
};
```

**Behavior:** Hook failed but doesn't block the operation.

#### Exit Code Summary Table

| Exit Code | Outcome | Behavior |
|-----------|---------|----------|
| 0 | Success | Continue normally |
| 2 | Blocking | Stop/block the operation |
| 1, 3+ | Non-blocking error | Log error but continue |

**Special cases:**
- SessionStart, Setup, SubagentStart: Blocking errors are ignored
- PreCompact: Exit 0 appends stdout as custom instructions
- PermissionRequest: Exit 0 uses hook decision if provided

### Error Handling When Hooks Fail

**Location:** Lines 62166-62191

```typescript
catch (x) {
  S?.();  // Cleanup timeout
  let U = x instanceof Error ? x.message : String(x);

  Yh({
    hookId: m,
    hookName: _,
    hookEvent: O,
    output: `Failed to run: ${U}`,
    stdout: "",
    stderr: `Failed to run: ${U}`,
    exitCode: 1,
    outcome: "error",
  });

  yield {
    message: Zq({
      type: "hook_error_during_execution",
      hookName: _,
      toolUseID: q,
      hookEvent: O,
      content: U,
    }),
    outcome: "non_blocking_error",
    hook: Z,
  };
}
```

**Error Scenarios:**
1. **EPIPE Error** (line 61660): Hook closed stdin early - non-fatal
2. **ABORT_ERR** (line 61667): Hook cancelled - returns aborted status
3. **Other errors**: Wrapped as non-blocking errors

### Parallel Execution and Concurrency

#### Parallel Execution by Default


```typescript
// Line 61904 - Create array of async generators
let G = D.map(async function* ({ hook: Z, ... }, k) {
  // Each hook returns an async generator
});

// Line 62196 - Process hooks in parallel with controlled concurrency
for await (let Z of xO6(G)) {  // xO6 = parallel iterator
  // Process results as they complete
}
```

#### Controlled Concurrency: `xO6`

```typescript
async function* xO6(A, q = 1 / 0) {  // q = concurrency limit (default: infinity)
  // ...
  while (z.size > 0) {
    let { done: w, value: H, generator: $, promise: O } = await Promise.race(z);
    // Yield results as they complete (not in order)
  }
}
```

**Key Points:**
- Hooks run in **parallel by default** (unlimited concurrency)
- Results are processed as they complete (not in order)
- The concurrency limit parameter defaults to `Infinity`
- Each hook runs independently

### Permission Decision Aggregation

**Location:** Lines 62215-62228

When multiple hooks return permission decisions, they're aggregated:

```typescript
let V;  // Aggregated permission behavior

// Priority: deny > ask > allow
switch (Z.permissionBehavior) {
  case "deny":
    V = "deny";
    break;
  case "ask":
    if (V !== "deny") V = "ask";
    break;
  case "allow":
    if (!V) V = "allow";
    break;
}
```

**Permission priority:**
1. `deny` - Highest priority (blocks if any hook denies)
2. `ask` - Medium priority (prompts if no denies)
3. `allow` - Lowest priority (only if no other decisions)

### Async Hook Support

**Location:** Lines 61586-61616, 56674-56675

Hooks can return `{"async": true}` to run in background:

```typescript
function gq1(A) {
  return "async" in A && A.async === !0;
}

// In stdout handler:
if (gq1(r) && !_) {  // _ = forceSyncExecution flag
  let c = `async_hook_${W.pid}`;
  // Background the process
  P = !0;
  // Return immediately without waiting
}
```

**Async behavior:**
- Hook can return `{"async": true}` early
- Process is backgrounded (tracked by PID)
- Execution continues without waiting
- Can be forced to wait with `forceSyncExecution: true`

### Execution Summary

The Claude Code CLI hook execution engine:

1. **Spawns hooks** using `child_process.spawn` with shell support
2. **Passes JSON data** via stdin with event-specific payloads
3. **Sets environment variables** (CLAUDE_PROJECT_DIR, CLAUDE_PLUGIN_ROOT, etc.)
4. **Captures stdout/stderr** with UTF-8 encoding
5. **Enforces timeouts** (default 10 minutes, configurable per-hook)
6. **Interprets exit codes**: 0=success, 2=blocking, others=non-blocking error
7. **Parses JSON output** for structured responses (permission decisions, context, etc.)
8. **Supports async hooks** via `{"async": true}` response
9. **Runs hooks in parallel** with unlimited concurrency by default
10. **Aggregates permission decisions** with deny > ask > allow priority
11. **Integrates with tool permission system** to allow/deny/modify tool execution

---

## 6. Hook Output Format


Hooks can respond via stdout with JSON. The response is parsed and validated against schemas.

### General Hook Output Schema

All hooks can return a general output structure:

```typescript
{
  continue: boolean (optional),
  suppressOutput: boolean (optional),
  stopReason: string (optional),
  decision: "approve" | "block" (optional),
  systemMessage: string (optional),
  reason: string (optional),
  hookSpecificOutput: HookSpecificOutput (optional)
}
```


### Async Response Schema

For async hooks:

```typescript
{
  async: true,
  asyncTimeout: number (optional)
}
```


- Tells Claude Code to background the hook process
- `asyncTimeout` is optional, defaults to hook timeout

### Standard Response Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `continue` | boolean | No | If false, stops hook execution chain |
| `suppressOutput` | boolean | No | If true, suppresses hook output from being shown to user |
| `stopReason` | string | No | Reason for stopping (if continue is false) |
| `decision` | "approve" \| "block" | No | General approval/block decision |
| `systemMessage` | string | No | Message to add to system context |
| `reason` | string | No | Reason for the decision |
| `hookSpecificOutput` | object | No | Hook-event-specific output (see below) |

### Hook-Specific Output Schemas

#### PreToolUse
```typescript
{
  hookEventName: "PreToolUse",
  permissionDecision: "allow" | "deny" | "ask",
  permissionDecisionReason: string,
  updatedInput: { [key: string]: unknown },  // Modified tool input
  additionalContext: string
}
```

#### PermissionRequest
```typescript
{
  hookEventName: "PermissionRequest",
  decision: {
    behavior: "allow",
    updatedInput: { [key: string]: unknown },
    updatedPermissions: Array<PermissionRule>
  }
  // OR
  decision: {
    behavior: "deny",
    message: string,
    interrupt: boolean
  }
}
```

#### PostToolUse
```typescript
{
  hookEventName: "PostToolUse",
  additionalContext: string,
  updatedMCPToolOutput: unknown  // Can modify the tool output
}
```

#### PostToolUseFailure
```typescript
{
  hookEventName: "PostToolUseFailure",
  additionalContext: string
}
```

#### UserPromptSubmit
```typescript
{
  hookEventName: "UserPromptSubmit",
  additionalContext: string  // REQUIRED for this event
}
```

#### SessionStart
```typescript
{
  hookEventName: "SessionStart",
  additionalContext: string
}
```

#### Setup
```typescript
{
  hookEventName: "Setup",
  additionalContext: string
}
```

#### SubagentStart
```typescript
{
  hookEventName: "SubagentStart",
  additionalContext: string
}
```

#### Notification
```typescript
{
  hookEventName: "Notification",
  additionalContext: string
}
```

### JSON Validation

**Location:** Lines 61358-61373

```typescript
let K = MA(q),              // Parse JSON
  Y = yO6.safeParse(K);     // Validate with Zod schema

if (Y.success)
  return { json: Y.data };
else {
  // Return validation error with schema details
  return { plainText: A, validationError: w };
}
```

**Validation failure:** Treated as plain text with error logged.

### Plain Text Output

If stdout doesn't start with `{`, treated as plain text (no special parsing).

### Output Processing Function

**Location:** Line 61398

`Wd4({json, command, ...})` - Processes validated hook JSON response

### Hook Capabilities Summary

#### Input Transformation
Hooks can modify tool inputs via:
- **PreToolUse**: Set `hookSpecificOutput.updatedInput`
- **PermissionRequest**: Set `decision.updatedInput`

#### Permission Control
Hooks can control permissions via:
- **PreToolUse**: Set `hookSpecificOutput.permissionDecision` to "allow", "deny", or "ask"
- **PermissionRequest**: Provide full `decision` with behavior "allow" or "deny"
  - Can include `updatedPermissions` array to add permission rules
  - Can set `interrupt: true` to abort current operation

#### Custom Messages
Hooks can provide user-facing feedback via:
- `systemMessage`: Added to system context
- `additionalContext`: Added to conversation context (most hook types support this)
- For deny decisions: `message` field explains why permission was denied

#### Output Modification
- **PostToolUse**: Can modify tool output via `updatedMCPToolOutput`

#### Execution Control
- `continue: false`: Stops further hook execution in the chain
- `suppressOutput: true`: Hides hook output from user
- `async: true`: Backgrounds the hook process

### Zod Schema Definitions

**Location:** Lines 19850-20150

- `NZ`: Base hook input schema with session_id, transcript_path, cwd, permission_mode
- `RjY`: PreToolUse input schema
- `yjY`: PermissionRequest input schema
- `CjY`: PostToolUse input schema
- `SjY`: PostToolUseFailure input schema
- `hjY`: Notification input schema
- `IjY`: UserPromptSubmit input schema
- `xjY`: SessionStart input schema
- `djY`: SessionEnd input schema
- `ujY`: Stop input schema
- `BjY`: SubagentStart input schema
- `mjY`: SubagentStop input schema
- `FjY`: PreCompact input schema
- `QjY`: TeammateIdle input schema
- `UjY`: TaskCompleted input schema
- `AWY`: Standard hook output schema
- `cjY`: Async hook output schema
- `Gdw`: Union of async and standard output
- `ljY`, `ijY`, `njY`, etc.: Hook-specific output schemas

---

## 7. Tool Pipeline Integration


The hooks system provides comprehensive integration points throughout the tool execution pipeline.

### Pipeline Flow


The main tool execution flow through `QQY()` and `cs4()` functions:

```typescript
// Line 289: Hook execution integrated into tool pipeline
for await (let y of cs4(Y, A, j, q, w.message.id, $, O, _))
  switch (y.type) {
    case "message":
      if (y.message.message.type === "progress") J(y.message.message);
      else M.push(y.message);
      break;
    case "hookPermissionResult":
      P = y.hookPermissionResult;
      break;
    case "hookUpdatedInput":
      j = y.updatedInput;
      break;
    case "preventContinuation":
      W = y.shouldPreventContinuation;
      break;
    case "stopReason":
      G = y.stopReason;
      break;
    case "additionalContext":
      M.push(y.message);
      break;
    case "stop":
      return [/* ... */];
  }
```

### PreToolUse Hook Integration (`cs4` function)


PreToolUse hooks are called before tool execution:

```typescript
async function* cs4(A, q, K, Y, z, w, H, $) {
  let O = Date.now();
  try {
    let _ = await A.getAppState();
    for await (let J of tkA(
      q.name,
      Y,
      K,
      A,
      _.toolPermissionContext.mode,
      z,
      A.abortController.signal,
      w,
      H,
      $
    ))
```

#### Permission Decision Flow

**Blocking Error:**
```typescript
// Lines 1160-1171: Hook can block with error
if (J.blockingError) {
  let X = nkA(`PreToolUse:${q.name}`, J.blockingError);
  yield {
    type: "hookPermissionResult",
    hookPermissionResult: {
      behavior: "deny",
      message: X,
      decisionReason: {
        type: "hook",
        hookName: `PreToolUse:${q.name}`,
        reason: X,
      },
    },
  };
}
```

**Permission Behavior:**
```typescript
// Lines 1184-1223: Hook can allow, deny, or ask for permission
if (J.permissionBehavior !== void 0) {
  h(`Hook result has permissionBehavior=${J.permissionBehavior}`);
  let X = {
    type: "hook",
    hookName: `PreToolUse:${q.name}`,
    reason: J.hookPermissionDecisionReason,
  };
  if (J.permissionBehavior === "allow")
    yield {
      type: "hookPermissionResult",
      hookPermissionResult: {
        behavior: "allow",
        updatedInput: J.updatedInput,
        decisionReason: X,
      },
    };
  else if (J.permissionBehavior === "ask")
    yield {
      type: "hookPermissionResult",
      hookPermissionResult: {
        behavior: "ask",
        updatedInput: J.updatedInput,
        message: J.hookPermissionDecisionReason ||
          `Hook PreToolUse:${q.name} ${$CA(J.permissionBehavior)} this tool`,
        decisionReason: X,
      },
    };
}
```

**Input Modification:**
```typescript
// Lines 1224-1225: Hook can modify input without changing permission
if (J.updatedInput && J.permissionBehavior === void 0)
  yield { type: "hookUpdatedInput", updatedInput: J.updatedInput };
```

**Additional Context:**
```typescript
// Lines 1226-1236: Hook can inject context into model
if (J.additionalContexts && J.additionalContexts.length > 0)
  yield {
    type: "additionalContext",
    message: {
      message: Zq({
        type: "hook_additional_context",
        content: J.additionalContexts,
        hookName: `PreToolUse:${q.name}`,
        toolUseID: Y,
        hookEvent: "PreToolUse",
      }),
    },
  };
```

#### Error Handling

```typescript
// Lines 1262-1280: PreToolUse hook error handling
catch (X) {
  K1(X instanceof Error ? X : Error(String(X)));
  let D = Date.now() - O;
  l("tengu_pre_tool_hook_error", {
    messageID: z,
    toolName: rq(q.name),
    isMcp: q.isMcp ?? !1,
    duration: D,
    queryChainId: A.queryTracking?.chainId,
    queryDepth: A.queryTracking?.depth,
    // ...
  });
  yield {
    type: "message",
    message: {
      message: Zq({
        type: "hook_error_during_execution",
        content: hG1(X),
        hookName: `PreToolUse:${q.name}`,
        toolUseID: Y,
        hookEvent: "PreToolUse",
      }),
    },
  };
  yield { type: "stop" });
}
```

### Hook Result Processing


**Permission Override:**
```typescript
if (
  P !== void 0 &&
  P.behavior === "allow" &&
  !A.requiresUserInteraction?.() &&
  !Y.requireCanUseTool
) {
  h(`Hook approved tool use for ${A.name}, bypassing permission check`);
  Z = P;
} else if (
  P !== void 0 &&
  P.behavior === "allow" &&
  (A.requiresUserInteraction?.() || Y.requireCanUseTool)
) {
  h(`Hook approved tool use for ${A.name}, but canUseTool is required`);
  if (P.updatedInput) j = P.updatedInput;
  Z = await z(A, j, Y, w, q);
} else if (P !== void 0 && P.behavior === "deny") {
  h(`Hook denied tool use for ${A.name}`);
  Z = P;
}
```

**Input Modification:**
```typescript
// Lines 357, 437-439: Hooks can modify tool input
if (P?.behavior === "ask" && P.updatedInput) j = P.updatedInput;
// ...
if (Z.updatedInput !== void 0) j = Z.updatedInput;
```

**Execution Blocking:**
```typescript
// Lines 395, 590-595: PreToolUse can block execution
if (W && !u) u = `Execution stopped by PreToolUse hook${G ? `: ${G}` : ""}`;
// Hook creates error result and stops continuation
```

### PostToolUse Hook Integration (`ps4` function)


PostToolUse hooks are called after tool execution:

```typescript
async function* ps4(A, q, K, Y, z, w, H, $, O) {
  let _ = Date.now();
  try {
    let X = (await A.getAppState()).toolPermissionContext.mode,
    D = w;
    for await (let M of ekA(q.name, K, z, D, A, X, A.abortController.signal))
```

#### Hook Result Processing

**Cancellation:**
```typescript
// Lines 1000-1012: Handle cancelled hooks
if (
  M.message?.type === "attachment" &&
  M.message.attachment.type === "hook_cancelled"
) {
  l("tengu_post_tool_hooks_cancelled", {
    toolName: rq(q.name),
    queryChainId: A.queryTracking?.chainId,
    queryDepth: A.queryTracking?.depth,
  });
  yield {
    message: Zq({
      type: "hook_cancelled",
      hookName: `PostToolUse:${q.name}`,
      toolUseID: K,
      hookEvent: "PostToolUse",
    }),
  };
  continue;
}
```

**Blocking Error:**
```typescript
// Lines 1014-1023: Blocking errors stop continuation
if (M.blockingError)
  yield {
    message: Zq({
      type: "hook_blocking_error",
      hookName: `PostToolUse:${q.name}`,
      toolUseID: K,
      hookEvent: "PostToolUse",
      blockingError: M.blockingError,
    }),
  };
```

**Stop Continuation:**
```typescript
// Lines 1024-1035: Hook can prevent further execution
if (M.preventContinuation) {
  yield {
    message: Zq({
      type: "hook_stopped_continuation",
      message: M.stopReason || "Execution stopped by PostToolUse hook",
      hookName: `PostToolUse:${q.name}`,
      toolUseID: K,
      hookEvent: "PostToolUse",
    }),
  };
  return;
}
```

**Additional Context:**
```typescript
// Lines 1037-1046: Add context to model
if (M.additionalContexts && M.additionalContexts.length > 0)
  yield {
    message: Zq({
      type: "hook_additional_context",
      content: M.additionalContexts,
      hookName: `PostToolUse:${q.name}`,
      toolUseID: K,
      hookEvent: "PostToolUse",
    }),
  };
```

**MCP Tool Output Modification:**
```typescript
// Lines 1047-1048: Hooks can modify MCP tool output
if (M.updatedMCPToolOutput && mv(q))
  ((D = M.updatedMCPToolOutput), yield { updatedMCPToolOutput: D });
```

### PostToolUseFailure Hook Integration (`ds4` function)


PostToolUseFailure hooks are called after tool execution fails:

```typescript
async function* ds4(A, q, K, Y, z, w, H, $, O, _) {
  let J = Date.now();
  try {
    let D = (await A.getAppState()).toolPermissionContext.mode;
    for await (let M of ALA(q.name, K, z, w, A, H, D, A.abortController.signal))
```

The processing is similar to PostToolUse but with failure-specific context.

### Key Integration Points Summary

1. **Tool Execution Pipeline**
   - **Entry Point:** `QQY()` at line 187
   - **Hook Integration:** `cs4()` at line 289
   - **Permission Override:** Lines 335-359
   - **Input Modification:** Lines 437-439

2. **PreToolUse Integration**
   - **Hook Executor:** `cs4()` at line 1144
   - **Permission Decision:** Lines 1160-1223
   - **Input Modification:** Line 1225
   - **Additional Context:** Lines 1226-1236

3. **PostToolUse Integration**
   - **Hook Executor:** `ps4()` at line 994
   - **Result Processing:** Lines 1000-1048
   - **Output Modification:** Line 1048

---

## 8. Permission System Integration


Hooks are integrated into the permission check workflow, allowing automated permission decisions before user interaction.

### Hook Permission Decision Flow


Hooks run during automated checks:

```typescript
let j = await K.toolUseContext.getAppState(),
  W = await K.runHooks(
    j.toolPermissionContext.mode,
    z.suggestions,
    z.updatedInput,
    X,
  );
if (!W || !O()) return;
(K.removeFromQueue(), H(W));
```

### Permission Result Override

#### Allow Override

**Location:** Lines 974-981

```typescript
if ((S_6(H), X.behavior === "allow")) {
  _.logDecision({ decision: "accept", source: "config" });
  O(
    _.buildAllow(X.updatedInput ?? Y, {
      decisionReason: X.decisionReason,
    }),
  );
  return;
}
```

#### Deny Override

**Location:** Lines 990-1003

```typescript
case "deny": {
  GT6(
    {
      tool: K,
      input: Y,
      toolUseContext: z,
      messageId: _.messageId,
      toolUseID: H,
    },
    { decision: "reject", source: "config" },
  );
  O(X);
  return;
}
```

#### Ask Override

**Location:** Lines 1004-1038

```typescript
case "ask": {
  if (D.toolPermissionContext.awaitAutomatedChecksBeforeDialog) {
    let W = await Hjq({
      ctx: _,
      updatedInput: X.updatedInput,
      suggestions: X.suggestions,
      permissionMode: D.toolPermissionContext.mode,
    });
    if (W) {
      O(W);
      return;
    }
  }
  // Fallback to UI prompt
  Xjq(
    {
      ctx: _,
      description: M,
      result: X,
      awaitAutomatedChecksBeforeDialog:
        D.toolPermissionContext.awaitAutomatedChecksBeforeDialog,
    },
    O,
  );
}
```

### PermissionRequest Hook Execution

**Function:** `XQ1(A, q, K, Y, z, w, H, $)` at line 62615

Executes PermissionRequest hooks when user is prompted to approve a tool call.

### runHooks Flow


```typescript
async runHooks(_, J, X, D) {
  for await (let M of XQ1(...)) {  // XQ1 = executePermissionRequestHooks
    if (M.permissionRequestResult) {
      let j = M.permissionRequestResult;

      if (j.behavior === "allow") {
        // Hook allowed - update input and continue
        let W = j.updatedInput ?? X ?? q;
        return await this.handleHookAllow(W, j.updatedPermissions ?? [], D);
      }
      else if (j.behavior === "deny") {
        // Hook denied - block and log
        if (j.interrupt) {
          K.abortController.abort();  // Abort entire session
        }
        return this.buildDeny(j.message || "Permission denied by hook", ...);
      }
    }
  }
  return null;  // No hook decision - continue normal flow
}
```

### Permission System Summary

1. **PreToolUse hooks run** before permission dialog
2. Hooks can:
   - **Allow** - Bypass permission prompt, optionally modify input
   - **Deny** - Block tool execution, optionally abort session
   - **Ask** - Show permission prompt (default behavior)
   - **No decision** - Continue normal flow
3. **PostToolUse hooks run** after tool completes
4. **PostToolUseFailure hooks run** if tool fails
5. Hook results can add context, modify outputs, or block operations

### Telemetry for Permission Hooks

```typescript
// Permission hook telemetry
l("tengu_tool_use_granted_by_permission_hook", {
  ...jd1(q, A.name, Y),
  permanent: K.permanent ?? !1,
});
```

---

## 9. Hook Event Messages


When hooks execute, they can produce various message types that are tracked and displayed to users.

### All Hook Message Types

1. **hook_success** - Hook executed successfully
   - **Exit code 0**
   - Output handling varies by hook type

2. **hook_blocking_error** - Hook encountered a blocking error
   - **Exit code 2**
   - Show stderr to model and block tool call

3. **hook_non_blocking_error** - Hook encountered a non-blocking error
   - **Other exit codes**
   - Show stderr to user only but continue

4. **hook_error_during_execution** - Hook failed during execution
   - Internal error in hook system

5. **hook_cancelled** - Hook was cancelled/aborted
   - User or system cancelled hook

6. **hook_stopped_continuation** - Hook stopped continuation
   - Hook returned `continue: false` or `preventContinuation: true`

7. **hook_additional_context** - Hook provided additional context
   - Context injected back to model

8. **hook_system_message** - System message from hook
   - General system-level message

9. **hook_permission_decision** - Permission decision made by hook
   - Hook made allow/deny/ask decision

10. **hook_progress** - Progress updates during hook execution
    - Real-time progress reporting

### Hook Result Message Format

Messages are created using the `Zq()` function:

```typescript
Zq({
  type: "hook_success" | "hook_blocking_error" | ...,
  hookName: string,
  toolUseID: string,
  hookEvent: string,
  content?: string,
  blockingError?: string,
  stderr?: string,
  exitCode?: number,
  // ...
})
```

---

## 10. Telemetry Events


All hook events generate telemetry for debugging and analytics.

### PreToolUse Telemetry


```typescript
l("tengu_pre_tool_hook_error", {
  messageID: z,
  toolName: rq(q.name),
  isMcp: q.isMcp ?? !1,
  duration: D,
  queryChainId: A.queryTracking?.chainId,
  queryDepth: A.queryTracking?.depth,
  // ...
});
```

### PostToolUse Telemetry


```typescript
l("tengu_post_tool_hook_error", {
  messageID: Y,
  toolName: rq(q.name),
  isMcp: q.isMcp ?? !1,
  duration: W,
  queryChainId: A.queryTracking?.chainId,
  queryDepth: A.queryTracking?.depth,
  // ...
});

l("tengu_post_tool_hooks_cancelled", {
  toolName: rq(q.name),
  queryChainId: A.queryTracking?.chainId,
  queryDepth: A.queryTracking?.depth,
});
```

### Stop Hook Telemetry


```typescript
l("tengu_stop_hook_error", {
  duration: G,
  queryChainId: H.queryTracking?.chainId,
  queryDepth: H.queryTracking?.depth,
});
```

### Permission Hook Telemetry

```typescript
l("tengu_tool_use_granted_by_permission_hook", {
  ...jd1(q, A.name, Y),
  permanent: K.permanent ?? !1,
});
```

### Tool Use Progress Telemetry


```typescript
l("tengu_tool_use_progress", {
  messageID: H,
  toolName: rq(A.name),
  isMcp: A.isMcp ?? !1,
  queryChainId: Y.queryTracking?.chainId,
  queryDepth: Y.queryTracking?.depth,
  // ...
});
```

### All Telemetry Events

1. `tengu_pre_tool_hook_error` - PreToolUse hook errors
2. `tengu_post_tool_hook_error` - PostToolUse hook errors
3. `tengu_post_tool_hooks_cancelled` - PostToolUse hooks cancelled
4. `tengu_stop_hook_error` - Stop hook errors
5. `tengu_tool_use_granted_by_permission_hook` - Permission granted by hook
6. `tengu_tool_use_progress` - Tool use progress (includes hook progress)

---

## 11. Key Functions Reference

### Hook Execution Functions

| Function | Description |
|----------|-------------|
| `zW6` | Spawns and executes command hooks |
| `sh` | Main async generator that executes hooks |
| `ikA` | Retrieves and filters hooks for a specific event |
| `ARY` | Core matching algorithm |
| `tkA` | Executes PreToolUse hooks |
| `ekA` | Executes PostToolUse hooks |
| `ALA` | Executes PostToolUseFailure hooks |
| `XQ1` | Executes PermissionRequest hooks |

### Hook Data Processing Functions

| Function | Description |
|----------|-------------|
| `hX` | Creates base hook context with session_id, transcript_path, cwd, permission_mode |
| `jd4` | Parses hook stdout output |
| `Wd4` | Processes validated hook JSON response |
| `gq1` | Detects async hook response |

### Hook Integration Functions

| Function | Description |
|----------|-------------|
| `cs4` | PreToolUse integration with tool pipeline |
| `ps4` | PostToolUse integration with tool pipeline |
| `ds4` | PostToolUseFailure integration with tool pipeline |
| `QQY` | Main tool execution pipeline entry point |
| `runHooks` | Runs permission hooks |

### Hook Configuration Functions

| Function | Description |
|----------|-------------|
| `iT7` | Loads all matching hooks for an event |
| `nT7` | Adds hooks programmatically |
| `rT7` | Removes hooks programmatically |
| `U9q` | Retrieves hooks for a specific event and matcher |
| `Q9q` | Returns all matchers for an event |
| `lt` | Returns matcherMetadata for an event |
| `g9q` | Returns summary for an event |
| `tg1` | Gets metadata for all events |

### Helper Functions

| Function | Description |
|----------|-------------|
| `OO6` | Wraps spawned process with timeout/abort functionality |
| `xO6` | Parallel async iterator with controlled concurrency |
| `Zq` | Creates hook message format |

---

## 12. Configuration Examples

### Example 1: Format Code on Write

**Auto-format files after Write/Edit:**
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "jq -r '.tool_response.filePath // .tool_input.file_path' | xargs prettier --write",
        "timeout": 30,
        "statusMessage": "Formatting code..."
      }]
    }]
  }
}
```

### Example 2: Log All Bash Commands

**Log all Bash commands:**
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "jq -r '.tool_input.command' >> ~/.claude/bash-log.txt"
      }]
    }]
  }
}
```

### Example 3: Verify Tests Pass with Agent

**Verify that unit tests ran successfully:**
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "agent",
        "prompt": "Verify that unit tests ran successfully: $ARGUMENTS",
        "timeout": 120,
        "model": "claude-sonnet-4-5-20250929"
      }]
    }]
  }
}
```

### Example 4: Session Start Notification

**Session start notification:**
```json
{
  "hooks": {
    "SessionStart": [{
      "hooks": [{
        "type": "command",
        "command": "echo 'Session started at $(date)'",
        "async": true
      }]
    }]
  }
}
```

### Example 5: Multiple Matchers for Same Event

**Multiple matchers for same event:**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "log-bash.sh" }]
      },
      {
        "matcher": "Write|Edit",
        "hooks": [{ "type": "command", "command": "format-file.sh" }]
      }
    ]
  }
}
```

### Example 6: Match All MCP Tools with Regex

**Match all MCP tools with regex:**
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "^mcp__.*",
      "hooks": [{
        "type": "command",
        "command": "echo 'MCP tool executed' >> mcp.log"
      }]
    }]
  }
}
```

### Example 7: Match All Tools

**Match all tools:**
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "log-all-tools.sh"
      }]
    }]
  }
}
```

### Example 8: Block Dangerous Operations

**Block specific operations:**
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "if echo \"$1\" | jq -r '.tool_input.command' | grep -q 'rm -rf'; then exit 2; fi",
        "statusMessage": "Checking for dangerous commands..."
      }]
    }]
  }
}
```

### Example 9: Inject Context on Session Start

**Inject project context at session start:**
```json
{
  "hooks": {
    "SessionStart": [{
      "hooks": [{
        "type": "command",
        "command": "cat PROJECT_CONTEXT.md",
        "statusMessage": "Loading project context..."
      }]
    }]
  }
}
```

---

## 13. Architecture Diagram

### Hook Execution Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     Tool Execution Request                      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    PreToolUse Hook Phase                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 1. Load matching hooks (ikA + ARY)                       │  │
│  │ 2. Execute hooks in parallel (xO6)                       │  │
│  │ 3. Aggregate permission decisions (deny > ask > allow)   │  │
│  │ 4. Modify tool input (updatedInput)                      │  │
│  │ 5. Inject additional context                             │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
                 ┌─────────────────┐
                 │ Hook Decision?  │
                 └────────┬────────┘
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
     [allow]         [deny]          [ask]
         │               │               │
         │               │               ▼
         │               │        ┌─────────────┐
         │               │        │ Permission  │
         │               │        │   Prompt    │
         │               │        └──────┬──────┘
         │               │               │
         ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Tool Execution                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Execute tool with (possibly modified) input              │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                 ┌─────────┴─────────┐
                 │                   │
                 ▼                   ▼
         [Tool Success]      [Tool Failure]
                 │                   │
                 ▼                   ▼
┌─────────────────────────┐ ┌────────────────────────────────────┐
│  PostToolUse Hook Phase │ │ PostToolUseFailure Hook Phase      │
│  ┌───────────────────┐  │ │  ┌──────────────────────────────┐  │
│  │ 1. Execute hooks  │  │ │  │ 1. Execute hooks             │  │
│  │ 2. Inject context │  │ │  │ 2. Inject context            │  │
│  │ 3. Modify output  │  │ │  │ 3. Log/analyze error         │  │
│  └───────────────────┘  │ │  └──────────────────────────────┘  │
└─────────────────────────┘ └────────────────────────────────────┘
                 │                   │
                 └─────────┬─────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Return Result to Model                       │
└─────────────────────────────────────────────────────────────────┘
```

### Hook Matching Algorithm

```
┌─────────────────────────────────────────────────────────────────┐
│                       Hook Event Fired                          │
│                (e.g., PreToolUse with tool_name="Bash")          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              Extract Field Value (ikA function)                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ PreToolUse/PostToolUse → tool_name                       │  │
│  │ SessionStart → source                                    │  │
│  │ Notification → notification_type                         │  │
│  │ SubagentStart → agent_type                               │  │
│  │ etc.                                                     │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│           Retrieve All Hooks for Event (from hookMap)          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│            Filter Hooks Using ARY(actualValue, matcher)         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ For each hook:                                           │  │
│  │   if (!hook.matcher || hook.matcher === "*")            │  │
│  │     → MATCH                                              │  │
│  │   else if (pipe-separated: "Tool1|Tool2|Tool3")         │  │
│  │     → Check if actualValue in split list                │  │
│  │   else if (exact match)                                  │  │
│  │     → actualValue === matcher                            │  │
│  │   else (regex)                                           │  │
│  │     → new RegExp(matcher).test(actualValue)             │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                Execute Matching Hooks in Parallel               │
└─────────────────────────────────────────────────────────────────┘
```

### Hook Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     Claude Code Process                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 1. Build hookInput JSON object                           │  │
│  │    - Base: session_id, cwd, transcript_path, etc.        │  │
│  │    - Event-specific: tool_name, tool_input, etc.         │  │
│  └────────────────────────┬─────────────────────────────────┘  │
└───────────────────────────┼──────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              spawn(command, [], { env, shell: true })           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Environment Variables:                                   │  │
│  │   - CLAUDE_PROJECT_DIR                                   │  │
│  │   - CLAUDE_PLUGIN_ROOT (conditional)                     │  │
│  │   - CLAUDE_ENV_FILE (conditional)                        │  │
│  │   - All process.env                                      │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Hook Process                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ stdin ← JSON payload                                     │  │
│  │ ┌────────────────────────────────────────────────────┐   │  │
│  │ │ {                                                  │   │  │
│  │ │   "hook_event_name": "PreToolUse",                │   │  │
│  │ │   "tool_name": "Bash",                            │   │  │
│  │ │   "tool_input": {...},                            │   │  │
│  │ │   "session_id": "...",                            │   │  │
│  │ │   ...                                              │   │  │
│  │ │ }                                                  │   │  │
│  │ └────────────────────────────────────────────────────┘   │  │
│  │                                                          │  │
│  │ [Process hook logic]                                     │  │
│  │                                                          │  │
│  │ stdout → JSON response (optional)                        │  │
│  │ ┌────────────────────────────────────────────────────┐   │  │
│  │ │ {                                                  │   │  │
│  │ │   "hookSpecificOutput": {                         │   │  │
│  │ │     "hookEventName": "PreToolUse",                │   │  │
│  │ │     "permissionDecision": "allow",                │   │  │
│  │ │     "updatedInput": {...}                         │   │  │
│  │ │   }                                                │   │  │
│  │ │ }                                                  │   │  │
│  │ └────────────────────────────────────────────────────┘   │  │
│  │                                                          │  │
│  │ stderr → Error messages                                  │  │
│  │                                                          │  │
│  │ Exit code: 0 (success), 2 (blocking), other (error)     │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Claude Code Process                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 1. Parse stdout JSON (jd4 function)                      │  │
│  │ 2. Validate with Zod schemas                             │  │
│  │ 3. Process hook result (Wd4 function)                    │  │
│  │ 4. Apply effects to tool execution:                      │  │
│  │    - Permission decisions                                │  │
│  │    - Input modifications                                 │  │
│  │    - Additional context                                  │  │
│  │    - Blocking/continuation control                       │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Summary

The Claude Code CLI hooks system provides comprehensive integration points throughout the tool execution pipeline:
1. **PreToolUse hooks** intercept tool calls before execution, with full control over permission decisions, input modification, and execution blocking.
2. **PostToolUse hooks** process results after execution, can inject additional context, modify MCP output, and prevent continuation.
3. **Permission hooks** integrate directly into the permission system, allowing automated approval/denial decisions before user interaction.
4. **Lifecycle hooks** (SessionStart, UserPromptSubmit, Stop) provide observability and control over session lifecycle events.
5. **Hook results flow through** a well-defined pipeline with specific message types, telemetry events, and error handling.
6. **Progress reporting** keeps the UI informed of hook execution status through dedicated hook_progress events.
7. **Telemetry integration** tracks all hook execution, errors, and decisions for debugging and analytics.

The hooks system is designed to be extensible, observable, and safe - with comprehensive error handling, cancellation support, and UI feedback throughout the execution pipeline.

---

**Document Version:** 1.0  
**Date:** 2026-02-06  
