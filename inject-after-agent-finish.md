# Injecting Context After an Agent Finishes Work

## Problem

You want to inject additional context into the model's conversation after a subagent (Task tool) completes its work. Neither `SubagentStop` nor `Stop` hooks support an `additionalContext` field in their output schema.

## Recommended Solution: `PostToolUse` Hook with `Task` Matcher

When an agent finishes, the `Task` tool returns its result to the parent conversation. This fires the `PostToolUse` hook event. The `PostToolUse` event fully supports `additionalContext` injection via `hookSpecificOutput`.

---

## How It Works

### Execution Flow

```
Orchestrator spawns agent via Task tool
         |
         v
   Agent runs autonomously
         |
         v
   Agent completes work
         |
         v
   Task tool returns result to parent
         |
         v
  *** PostToolUse fires with matcher "Task" ***
         |
         v
   Your hook receives the full task result
         |
         v
   Hook outputs JSON with additionalContext
         |
         v
   Context injected into model conversation
         |
         v
   Model sees agent result + your injected context
```

---

## The PostToolUse Hook

### Overview

`PostToolUse` runs immediately after any tool call succeeds. It is one of the most capable hook events, supporting:

- `additionalContext` — inject text into the model's context
- `updatedMCPToolOutput` — replace the tool's output entirely
- `systemMessage` — display a message to the user
- `suppressOutput` — hide stdout from transcript
- `continue` / `stopReason` — control whether processing continues

### Exit Code Behavior

| Exit Code | Behavior |
|-----------|----------|
| `0` | Success. Stdout shown in transcript mode (Ctrl+O). If stdout is valid JSON with `hookSpecificOutput`, those fields are processed. |
| `2` | Blocking. Stderr shown to the model immediately as context. |
| Other | Non-blocking error. Stderr shown to user only (not injected into model). |

### Input Schema (stdin)

Your hook receives this JSON on stdin:

```json
{
  "hook_event_name": "PostToolUse",
  "session_id": "<session-uuid>",
  "tool_name": "Task",
  "tool_input": {
    "description": "<task description>",
    "prompt": "<full prompt sent to agent>",
    "subagent_type": "<agent type, e.g. goodvibes:engineer>",
    "run_in_background": true,
    "model": "<model if specified>"
  },
  "tool_response": {
    "<the full result returned by the agent>"
  },
  "tool_use_id": "toolu_<id>",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/working/directory",
  "permission_mode": "<allowedTools|ask|deny>"
}
```

**Key fields for Task matcher:**
- `tool_name` — always `"Task"` when an agent completes
- `tool_input.subagent_type` — the agent type (e.g., `goodvibes:engineer`, `Explore`, `general-purpose`)
- `tool_input.prompt` — the full prompt that was sent to the agent
- `tool_input.description` — the short description of the task
- `tool_response` — the complete result from the agent

### Output Schema (stdout)

To inject context, output this JSON to stdout:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "Your injected context string here"
  }
}
```

**Full output schema with all optional fields:**

```json
{
  "continue": true,
  "suppressOutput": false,
  "stopReason": "string (optional, used when continue=false)",
  "systemMessage": "string (optional, shown to user in UI)",
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "string (optional, injected into model context)",
    "updatedMCPToolOutput": "string (optional, replaces the tool output entirely)"
  }
}
```

**Important:** The `hookEventName` field MUST match the actual event (`"PostToolUse"`). If it doesn't match, the hook engine throws an error:
> `Hook returned incorrect event name: expected 'PostToolUse' but got '...'`

---

## The Task Matcher — In Detail

### How Matchers Work

The `PostToolUse` event uses `tool_name` as its matcher field. When Claude Code fires a PostToolUse event, it checks each configured hook entry's `matcher` string against the `tool_name` of the completed tool call.

### Matcher Syntax

```
"matcher": "Task"
```

- **Exact match**: `"Task"` matches only the Task tool
- **Pipe-separated OR**: `"Task|Bash"` matches Task OR Bash
- **Empty/omitted**: matches ALL tool calls (not recommended for this use case)

### What "Task" Matches

The `Task` tool is the internal name for the subagent/agent spawning tool. It fires when:

1. **Background agents complete** — `run_in_background: true` agents notify completion, then PostToolUse fires
2. **Foreground agents return** — agents run in foreground return their result, PostToolUse fires immediately
3. **Any subagent type** — all agent types (`goodvibes:engineer`, `goodvibes:reviewer`, `Explore`, `Plan`, `general-purpose`, etc.) use the same `Task` tool

### What "Task" Does NOT Match

- `Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep` — these are separate tools
- `WebSearch`, `WebFetch` — separate tools
- MCP tools (`mcp__*`) — separate tools
- The matcher is case-sensitive: `"task"` will NOT match `"Task"`

### All Available Tool Names for Matchers

For reference, these are the tool names you can match against:

| Tool Name | Description |
|-----------|-------------|
| `Task` | Subagent/agent spawning |
| `Bash` | Shell command execution |
| `Read` | File reading |
| `Write` | File creation |
| `Edit` | File editing |
| `Glob` | File pattern matching |
| `Grep` | Content search |
| `WebSearch` | Web search |
| `WebFetch` | URL fetching |
| `NotebookEdit` | Jupyter notebook editing |
| `TodoWrite` | Task list management |
| `ToolSearch` | Deferred tool discovery |
| `mcp__*` | MCP server tools (dynamic names) |

---

## Configuration

### Settings File Location

Hooks are configured in `settings.json` at one of these scopes:

| Scope | Path | Use Case |
|-------|------|----------|
| User (global) | `~/.claude/settings.json` | Personal hooks for all projects |
| Project | `.claude/settings.json` (in repo root) | Shared team hooks |
| Project local | `.claude/settings.local.json` | Personal hooks for this project |

### Basic Configuration

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Task",
        "hooks": [
          {
            "type": "command",
            "command": "your-script-here",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

### Configuration Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `matcher` | `string` | No | Tool name(s) to match. Pipe-separated for OR. Omit to match all. |
| `hooks` | `array` | Yes | Array of hook definitions to execute. |
| `hooks[].type` | `string` | Yes | `"command"`, `"prompt"`, `"agent"`, or `"callback"` |
| `hooks[].command` | `string` | Yes (for command type) | Shell command to execute |
| `hooks[].timeout` | `number` | No | Timeout in seconds (default: ~30s) |
| `hooks[].statusMessage` | `string` | No | Status text shown while hook runs |
| `hooks[].async` | `boolean` | No | Run asynchronously (default: false) |

---

## Practical Examples

### Example 1: Simple Context Injection

Inject a reminder after every agent completes:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Task",
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"hookSpecificOutput\": {\"hookEventName\": \"PostToolUse\", \"additionalContext\": \"REMINDER: Check the agent output for any <gv> directives and execute them immediately.\"}}'"
          }
        ]
      }
    ]
  }
}
```

### Example 2: Conditional Injection Based on Agent Type

Read the agent type from stdin and inject different context:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Task",
        "hooks": [
          {
            "type": "command",
            "command": "node /path/to/post-agent-hook.mjs"
          }
        ]
      }
    ]
  }
}
```

**`post-agent-hook.mjs`:**

```javascript
import { readFileSync } from 'fs';

const input = JSON.parse(readFileSync('/dev/stdin', 'utf-8'));
const agentType = input.tool_input?.subagent_type || 'unknown';
const result = input.tool_response || '';

let context = '';

if (agentType.includes('engineer')) {
  context = 'Agent completed engineering work. Run tests before proceeding.';
} else if (agentType.includes('reviewer')) {
  context = 'Review complete. Parse the review score and act on any issues found.';
} else if (agentType.includes('tester')) {
  context = 'Tests complete. Check coverage report before marking task done.';
}

if (context) {
  console.log(JSON.stringify({
    hookSpecificOutput: {
      hookEventName: 'PostToolUse',
      additionalContext: context
    }
  }));
}
```

### Example 3: Inject from a File

Read context from a project-specific file:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Task",
        "hooks": [
          {
            "type": "command",
            "command": "cat .claude/post-agent-context.txt | jq -Rs '{hookSpecificOutput: {hookEventName: \"PostToolUse\", additionalContext: .}}'"
          }
        ]
      }
    ]
  }
}
```

### Example 4: Log Agent Results + Inject Context

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Task",
        "hooks": [
          {
            "type": "command",
            "command": "tee -a ~/.claude/agent-results.log | node -e \"const d=JSON.parse(require('fs').readFileSync('/dev/stdin','utf-8')); const ctx='Agent '+d.tool_input.subagent_type+' completed. Description: '+d.tool_input.description; console.log(JSON.stringify({hookSpecificOutput:{hookEventName:'PostToolUse',additionalContext:ctx}}))\""
          }
        ]
      }
    ]
  }
}
```

---

## Alternative Approaches (and Why PostToolUse + Task Is Better)

### Alternative 1: Stop Hook with Exit Code 2

```json
{
  "hooks": {
    "Stop": [{
      "hooks": [{
        "type": "command",
        "command": "echo 'Your context here' >&2; exit 2"
      }]
    }]
  }
}
```

**How it works:** Exit code 2 sends stderr to the model and prevents the stop (conversation continues).

**Why it's worse:**
- Fires on EVERY stop, not just after agents
- Prevents the model from stopping (forces continuation)
- No structured JSON output, just raw stderr text
- No way to conditionally inject based on what just happened
- Can create infinite loops if not careful

### Alternative 2: SubagentStop Hook with Exit Code 2

```json
{
  "hooks": {
    "SubagentStop": [{
      "hooks": [{
        "type": "command",
        "command": "echo 'Context for subagent' >&2; exit 2"
      }]
    }]
  }
}
```

**How it works:** Injects stderr into the subagent's context, preventing it from stopping.

**Why it's worse:**
- Injects into the SUBAGENT's context, not the parent orchestrator
- Prevents the subagent from completing (it keeps running)
- Not what you want if the goal is to inform the parent after completion

### Why PostToolUse + Task Wins

| Feature | PostToolUse+Task | Stop (exit 2) | SubagentStop (exit 2) |
|---------|-----------------|---------------|----------------------|
| Fires after agent completes | Yes | Yes (but also other stops) | No (fires in subagent) |
| Targets parent context | Yes | Yes | No (targets subagent) |
| Structured JSON output | Yes | No (raw stderr) | No (raw stderr) |
| Conditional on agent type | Yes (via stdin) | No | Partially |
| additionalContext field | Yes | No | No |
| updatedMCPToolOutput | Yes | No | No |
| Doesn't block normal flow | Yes | No (prevents stop) | No (prevents stop) |
| Can read agent result | Yes (tool_response) | No | No |

---

## Events That Support `additionalContext`

From the source code (`NMq` function in `hasworktreecreatehook-1.ts`, lines 179-244), these events support `additionalContext` via `hookSpecificOutput`:

| Event | additionalContext | updatedInput | updatedMCPToolOutput | permissionDecision |
|-------|:-:|:-:|:-:|:-:|
| PreToolUse | Yes | Yes | No | Yes |
| PostToolUse | Yes | No | Yes | No |
| PostToolUseFailure | Yes | No | No | No |
| UserPromptSubmit | Yes | No | No | No |
| SessionStart | Yes | No | No | No |
| Setup | Yes | No | No | No |
| SubagentStart | Yes | No | No | No |
| PermissionRequest | No | Yes (via decision) | No | No (uses decision.behavior) |
| **Stop** | **No** | **No** | **No** | **No** |
| **SubagentStop** | **No** | **No** | **No** | **No** |
| **SessionEnd** | **No** | **No** | **No** | **No** |
| **Notification** | **No** | **No** | **No** | **No** |
| **PreCompact** | **No** | **No** | **No** | **No** |
| **TeammateIdle** | **No** | **No** | **No** | **No** |
| **TaskCompleted** | **No** | **No** | **No** | **No** |
| **ConfigChange** | **No** | **No** | **No** | **No** |
| **WorktreeCreate** | **No** | **No** | **No** | **No** |
| **WorktreeRemove** | **No** | **No** | **No** | **No** |

---

## Source References

All findings verified against Claude Code v2.1.55 decompiled source:

| File | What's There |
|------|-------------|
| `src/tools/tool-1.ts:2916-3079` | All 18 hook event definitions, matchers, exit code docs |
| `src/tools/hasworktreecreatehook-1.ts:113-244` | `NMq()` — hook JSON output processing, hookSpecificOutput switch |
| `src/tools/hasworktreecreatehook-1.ts:100-106` | Expected output schema (validation error message) |
| `src/tools/hasworktreecreatehook-1.ts:628-810` | `Ub()` — hook execution engine |
| `src/tools/hasworktreecreatehook-1.ts:1030-1098` | Hook result yielding (additionalContext, updatedInput, etc.) |
| `src/tools/tool-4.ts:380-473` | Hook documentation and example configurations |
