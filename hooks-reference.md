# Claude Code Hooks: Comprehensive Reference Guide

Hooks are shell commands or LLM prompts that execute automatically at specific points in Claude Code's lifecycle. They provide deterministic control over Claude's behavior—unlike instructions in CLAUDE.md which are suggestions, hooks are guaranteed to run.

---

## Table of Contents

1. [Configuration](#configuration)
2. [Hook Events Reference](#hook-events-reference)
3. [Input Schemas](#input-schemas)
4. [Output & Decision Control](#output--decision-control)
5. [Working with MCP Tools](#working-with-mcp-tools)
6. [Best Practices](#best-practices)
7. [Debugging](#debugging)

---

## Configuration

### Settings File Locations

| Location | Scope | Priority |
|----------|-------|----------|
| `.claude/settings.json` | Project | Highest |
| `.claude/settings.local.json` | Project (not committed) | High |
| `~/.claude/settings.json` | User (all projects) | Lower |
| Enterprise managed policy | Organization | Varies |

### Basic Structure

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "ToolPattern",
        "hooks": [
          {
            "type": "command",
            "command": "your-command-here",
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

### Configuration Fields

| Field | Description |
|-------|-------------|
| `matcher` | Pattern to match (tool names, notification types, etc.). Supports regex. Use `*` or `""` for all. |
| `type` | `"command"` for shell commands, `"prompt"` for LLM-based evaluation |
| `command` | Shell command to execute (for `type: "command"`) |
| `prompt` | LLM prompt text (for `type: "prompt"`) |
| `timeout` | Timeout in seconds (default: 60) |

### Environment Variables

| Variable | Description |
|----------|-------------|
| `CLAUDE_PROJECT_DIR` | Absolute path to project root |
| `CLAUDE_CODE_REMOTE` | `"true"` if running in web environment |
| `CLAUDE_ENV_FILE` | File path for persisting env vars (SessionStart only) |

---

## Hook Events Reference

### Quick Reference Table

| Hook | When It Fires | Matcher Field | Matcher Values | Can Block |
|------|---------------|---------------|----------------|-----------|
| **PreToolUse** | Before tool executes | `tool_name` | Tool names | ✅ Exit 2 |
| **PostToolUse** | After tool succeeds | `tool_name` | Tool names | ✅ Exit 2 |
| **PostToolUseFailure** | After tool fails | `tool_name` | Tool names | ✅ Exit 2 |
| **PermissionRequest** | Permission dialog shown | `tool_name` | Tool names | ✅ JSON decision |
| **Notification** | System notification | `notification_type` | `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog` | ❌ |
| **UserPromptSubmit** | User submits prompt | ❌ | N/A | ✅ Exit 2 |
| **Stop** | Main agent finishes | ❌ | N/A | ✅ Exit 2 |
| **SubagentStart** | Subagent spawns | `agent_type` | Agent type names | ❌ |
| **SubagentStop** | Subagent finishes | ❌ | N/A | ✅ Exit 2 |
| **PreCompact** | Before compaction | `trigger` | `manual`, `auto` | ✅ Exit 2 |
| **SessionStart** | Session begins | `source` | `startup`, `resume`, `clear`, `compact` | ❌ |
| **SessionEnd** | Session ends | `reason` | `clear`, `logout`, `prompt_input_exit`, `other` | ❌ |
| **Setup** | Repo setup | `trigger` | `init`, `maintenance` | ❌ |

### Exit Code Reference

| Hook | Exit 0 | Exit 2 | Other Exit Codes |
|------|--------|--------|------------------|
| **PreToolUse** | stdout/stderr not shown | Show stderr to model, block tool call | Show stderr to user only, continue with tool call |
| **PostToolUse** | stdout shown in transcript mode (Ctrl+O) | Show stderr to model immediately | Show stderr to user only |
| **PostToolUseFailure** | stdout shown in transcript mode (Ctrl+O) | Show stderr to model immediately | Show stderr to user only |
| **PermissionRequest** | Use hook decision if provided | N/A | Show stderr to user only |
| **Notification** | stdout/stderr not shown | N/A | Show stderr to user only |
| **UserPromptSubmit** | stdout shown to Claude | Block processing, erase original prompt, show stderr to user only | Show stderr to user only |
| **Stop** | stdout/stderr not shown | Show stderr to model, continue conversation | Show stderr to user only |
| **SubagentStart** | stdout shown to subagent | Blocking errors ignored | Show stderr to user only |
| **SubagentStop** | stdout/stderr not shown | Show stderr to subagent, continue having it run | Show stderr to user only |
| **PreCompact** | stdout appended as custom compact instructions | Block compaction | Show stderr to user only, continue with compaction |
| **SessionStart** | stdout shown to Claude | Blocking errors ignored | Show stderr to user only |
| **SessionEnd** | Command completes successfully | N/A | Show stderr to user only |
| **Setup** | stdout shown to Claude | Blocking errors ignored | Show stderr to user only |

---

### 1. PreToolUse

**When it fires:** Before tool execution.

**Input:** JSON of tool call arguments.

**Use cases:**
- Block dangerous operations (rm -rf, .env access)
- Auto-approve safe operations
- Modify tool inputs before execution
- Validate commands against security policies

**Common matchers:**
- `Bash` - Shell commands
- `Read` - File reading
- `Write` - File creation
- `Edit` - File editing
- `Glob` - File pattern matching
- `Grep` - Content search
- `Task` - Subagent tasks
- `WebFetch`, `WebSearch` - Web operations

**Example:**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/validate-bash.py"
          }
        ]
      },
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/check-file-permissions.sh"
          }
        ]
      }
    ]
  }
}
```

**Decision control (JSON output):**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "Auto-approved documentation file",
    "updatedInput": {
      "command": "npm run lint --fix"
    }
  }
}
```

| Decision | Effect |
|----------|--------|
| `"allow"` | Bypasses permission system |
| `"deny"` | Prevents tool execution |
| `"ask"` | Shows user confirmation dialog |

---

### 2. PermissionRequest

**When it fires:** When a permission dialog is displayed.

**Input:** JSON with `tool_name`, `tool_input`, and `tool_use_id`.

**Output:** JSON with `hookSpecificOutput` containing decision to allow or deny.

**Use cases:**
- Auto-approve permissions programmatically
- Implement custom permission logic
- Deny permissions based on context

**Matchers:** Tool names (same as PreToolUse)

**Example:**
```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "matcher": "Read",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/auto-approve-reads.sh"
          }
        ]
      }
    ]
  }
}
```

**Decision control (JSON output):**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow",
      "updatedInput": {
        "command": "npm run lint"
      }
    }
  }
}
```

For deny:
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "deny",
      "message": "This operation is not allowed",
      "interrupt": true
    }
  }
}
```

---

### 3. PostToolUse

**When it fires:** After tool execution succeeds.

**Input:** JSON with fields `inputs` (tool call arguments) and `response` (tool call response).

**Use cases:**
- Run linters/formatters after file edits
- Execute tests after code changes
- Log tool results for auditing
- Trigger downstream workflows

**Matchers:** Tool names (same as PreToolUse)

**Example:**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.inputs.file_path' | xargs -I {} sh -c 'if echo {} | grep -qE \"\\.(ts|tsx)$\"; then npx prettier --write {}; fi'"
          }
        ]
      }
    ]
  }
}
```

**Decision control (JSON output):**
```json
{
  "decision": "block",
  "reason": "Linting failed: 3 errors found",
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "Error details: missing semicolons on lines 12, 45, 67"
  }
}
```

---

### 4. PostToolUseFailure

**When it fires:** After tool execution fails.

**Input:** JSON with `tool_name`, `tool_input`, `tool_use_id`, `error`, `error_type`, `is_interrupt`, and `is_timeout`.

**Use cases:**
- Custom error handling and recovery
- Failure logging and alerting
- Retry logic implementation
- Error analysis and reporting

**Matchers:** Tool names (same as PreToolUse)

**Example:**
```json
{
  "hooks": {
    "PostToolUseFailure": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/log-failure.sh"
          }
        ]
      }
    ]
  }
}
```

---

### 5. Notification

**When it fires:** When notifications are sent.

**Input:** JSON with notification message and type.

**Notification types (matchers):**
| Type | Description |
|------|-------------|
| `permission_prompt` | Permission requests from Claude |
| `idle_prompt` | After 60+ seconds of idle time |
| `auth_success` | Authentication success |
| `elicitation_dialog` | MCP tool elicitation |

**Use cases:**
- Desktop notifications when Claude needs attention
- Slack/Discord alerts for permission requests
- Custom sound alerts
- Mobile push notifications

**Example:**
```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "permission_prompt",
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude needs permission\" with title \"Claude Code\"'"
          }
        ]
      },
      {
        "matcher": "idle_prompt",
        "hooks": [
          {
            "type": "command",
            "command": "say 'Claude is waiting for your input'"
          }
        ]
      }
    ]
  }
}
```

---

### 6. UserPromptSubmit

**When it fires:** When the user submits a prompt.

**Input:** JSON with original user prompt text.

**Use cases:**
- Inject dynamic context (git status, TODOs, sprint info)
- Validate prompts for sensitive content
- Block certain types of requests
- Add security filtering

**Exit code behavior:**
- Exit 0: stdout shown to Claude
- Exit 2: Block processing, erase original prompt, show stderr to user only
- Other: Show stderr to user only

**Example:**
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/inject-context.py"
          }
        ]
      }
    ]
  }
}
```

**Context injection script example:**
```python
#!/usr/bin/env python3
import json
import sys
import subprocess

# Read input
input_data = json.load(sys.stdin)
prompt = input_data.get("prompt", "")

# Inject git context
git_status = subprocess.run(["git", "status", "--short"], capture_output=True, text=True)
git_branch = subprocess.run(["git", "branch", "--show-current"], capture_output=True, text=True)

context = f"""
Current branch: {git_branch.stdout.strip()}
Modified files:
{git_status.stdout}
"""

# Output context (will be added to conversation)
print(context)
sys.exit(0)
```

**Decision control (JSON output):**
```json
{
  "decision": "block",
  "reason": "Prompt contains potential secrets. Please rephrase.",
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "Current sprint: Q1-2026 Auth Refactor"
  }
}
```

---

### 7. Stop

**When it fires:** Right before Claude concludes its response.

**Use cases:**
- Force Claude to continue working
- Run cleanup tasks
- Send completion notifications
- Ensure all tasks are complete before stopping

**Exit code behavior:**
- Exit 0: stdout/stderr not shown
- Exit 2: Show stderr to model and continue conversation
- Other: Show stderr to user only

**Important:** Check `stop_hook_active` to prevent infinite loops.

**Example:**
```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/verify-completion.sh"
          }
        ]
      }
    ]
  }
}
```

**Prompt-based hook example (LLM evaluation):**
```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Evaluate if Claude should stop. Context: $ARGUMENTS\n\nCheck if:\n1. All requested tasks are complete\n2. Any errors need addressing\n3. Tests pass\n\nRespond with JSON: {\"decision\": \"approve\" or \"block\", \"reason\": \"explanation\"}",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

**Decision control (JSON output):**
```json
{
  "decision": "block",
  "reason": "Tests are still failing. Please fix the 3 failing test cases before stopping."
}
```

---

### 8. SubagentStart

**When it fires:** When a subagent (Task tool call) is started.

**Input:** JSON with `agent_id` and `agent_type`.

**Matchers:** Agent type names (matches on `agent_type` field)

**Use cases:**
- Set up resources before subagent runs
- Initialize database connections
- Configure environment for specific subagents
- Inject subagent-specific context

**Exit code behavior:**
- Exit 0: stdout shown to subagent
- Blocking errors are ignored
- Other: Show stderr to user only

**Example:**
```json
{
  "hooks": {
    "SubagentStart": [
      {
        "matcher": "db-agent",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/setup-db-connection.sh"
          }
        ]
      },
      {
        "matcher": "test-runner",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/setup-test-env.sh"
          }
        ]
      }
    ]
  }
}
```

---

### 9. SubagentStop

**When it fires:** Right before a subagent (Task tool call) concludes its response.

**Use cases:**
- Ensure subagent tasks complete properly
- Clean up resources after subagent finishes
- Validate subagent output
- Coordinate between parallel agents

**Exit code behavior:**
- Exit 0: stdout/stderr not shown
- Exit 2: Show stderr to subagent and continue having it run
- Other: Show stderr to user only

**Example:**
```json
{
  "hooks": {
    "SubagentStop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/cleanup-subagent.sh"
          }
        ]
      }
    ]
  }
}
```

**Paired with SubagentStart for lifecycle management:**
```json
{
  "hooks": {
    "SubagentStart": [
      {
        "matcher": "db-agent",
        "hooks": [{ "type": "command", "command": "./scripts/setup-db-connection.sh" }]
      }
    ],
    "SubagentStop": [
      {
        "matcher": "db-agent",
        "hooks": [{ "type": "command", "command": "./scripts/cleanup-db-connection.sh" }]
      }
    ]
  }
}
```

---

### 10. PreCompact

**When it fires:** Before conversation compaction.

**Input:** JSON with compaction details.

**Matchers:**
| Value | Trigger |
|-------|---------|
| `manual` | From `/compact` command |
| `auto` | From automatic compaction |

**Use cases:**
- Backup transcripts before compaction
- Preserve important context
- Append custom compact instructions
- Block compaction if conditions not met

**Exit code behavior:**
- Exit 0: stdout appended as custom compact instructions
- Exit 2: Block compaction
- Other: Show stderr to user only, continue with compaction

**Example:**
```json
{
  "hooks": {
    "PreCompact": [
      {
        "matcher": "auto",
        "hooks": [
          {
            "type": "command",
            "command": "cp \"$(jq -r '.transcript_path' /dev/stdin)\" ~/backups/transcript-$(date +%s).jsonl"
          }
        ]
      }
    ]
  }
}
```

---

### 11. SessionStart

**When it fires:** When a new session is started.

**Input:** JSON with session start source.

**Matchers:**
| Value | Trigger |
|-------|---------|
| `startup` | Fresh start |
| `resume` | From `--resume`, `--continue`, or `/resume` |
| `clear` | From `/clear` |
| `compact` | After compaction |

**Use cases:**
- Load development context (git status, TODOs, recent issues)
- Install dependencies
- Set up environment variables
- Initialize project state

**Exit code behavior:**
- Exit 0: stdout shown to Claude
- Blocking errors are ignored
- Other: Show stderr to user only

**Special behavior:** Can write to `$CLAUDE_ENV_FILE` to persist environment variables.

**Example:**
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/load-context.sh"
          }
        ]
      }
    ]
  }
}
```

**Environment persistence script example:**
```bash
#!/bin/bash

# Capture current environment
ENV_BEFORE=$(export -p | sort)

# Run setup commands
source ~/.nvm/nvm.sh
nvm use 20

# Persist environment changes
if [ -n "$CLAUDE_ENV_FILE" ]; then
  ENV_AFTER=$(export -p | sort)
  comm -13 <(echo "$ENV_BEFORE") <(echo "$ENV_AFTER") >> "$CLAUDE_ENV_FILE"
fi

# Output context (added to conversation)
echo "Git branch: $(git branch --show-current)"
echo "Node version: $(node --version)"
echo "Recent commits:"
git log --oneline -5

exit 0
```

---

### 12. SessionEnd

**When it fires:** When a session is ending.

**Input:** JSON with session end reason.

**Matchers (reason values):**
| Value | Trigger |
|-------|---------|
| `clear` | `/clear` command |
| `logout` | User logged out |
| `prompt_input_exit` | User exited while prompt visible |
| `other` | Other exit reasons |

**Use cases:**
- Cleanup tasks
- Log session statistics
- Save session state
- Send session summary notifications

**Example:**
```json
{
  "hooks": {
    "SessionEnd": [
      {
        "matcher": "prompt_input_exit",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/session-cleanup.sh"
          }
        ]
      }
    ]
  }
}
```

---

### 13. Setup

**When it fires:** Repo setup hooks for init and maintenance.

**Input:** JSON with trigger (`init` or `maintenance`).

**Matchers:**
| Value | Trigger |
|-------|---------|
| `init` | Initial repository setup |
| `maintenance` | Maintenance operations |

**Use cases:**
- Initialize project dependencies
- Run database migrations
- Set up development environment
- Perform routine maintenance tasks

**Exit code behavior:**
- Exit 0: stdout shown to Claude
- Blocking errors are ignored
- Other: Show stderr to user only

**Example:**
```json
{
  "hooks": {
    "Setup": [
      {
        "matcher": "init",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/init-project.sh"
          }
        ]
      },
      {
        "matcher": "maintenance",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/run-maintenance.sh"
          }
        ]
      }
    ]
  }
}
```

---

## Input Schemas

All hooks receive JSON via stdin with common fields plus event-specific data.

### Common Fields (All Hooks)

```json
{
  "session_id": "abc123",
  "transcript_path": "/Users/.../.claude/projects/.../session.jsonl",
  "cwd": "/Users/mike/projects/pellux",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse"
}
```

| Field | Description |
|-------|-------------|
| `session_id` | Unique session identifier |
| `transcript_path` | Path to conversation JSONL file |
| `cwd` | Current working directory |
| `permission_mode` | Current mode: `default`, `plan`, `acceptEdits`, `bypassPermissions` |
| `hook_event_name` | The hook event type |

### PreToolUse Input

```json
{
  "session_id": "abc123",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm run test"
  },
  "tool_use_id": "toolu_01ABC123..."
}
```

### PostToolUse Input

```json
{
  "session_id": "abc123",
  "hook_event_name": "PostToolUse",
  "tool_name": "Bash",
  "inputs": {
    "command": "npm run test"
  },
  "response": {
    "stdout": "All tests passed",
    "stderr": "",
    "exit_code": 0
  },
  "tool_use_id": "toolu_01ABC123..."
}
```

### PostToolUseFailure Input

```json
{
  "session_id": "abc123",
  "hook_event_name": "PostToolUseFailure",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm run test"
  },
  "tool_use_id": "toolu_01ABC123...",
  "error": "Command failed with exit code 1",
  "error_type": "execution_error",
  "is_interrupt": false,
  "is_timeout": false
}
```

### PermissionRequest Input

```json
{
  "session_id": "abc123",
  "hook_event_name": "PermissionRequest",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf node_modules"
  },
  "tool_use_id": "toolu_01ABC123..."
}
```

### Notification Input

```json
{
  "hook_event_name": "Notification",
  "message": "Claude needs your permission to use Bash",
  "notification_type": "permission_prompt"
}
```

### UserPromptSubmit Input

```json
{
  "hook_event_name": "UserPromptSubmit",
  "prompt": "Write a function to calculate factorial"
}
```

### Stop / SubagentStop Input

```json
{
  "hook_event_name": "Stop",
  "stop_hook_active": false
}
```

**Important:** `stop_hook_active` is `true` when Claude is already continuing due to a stop hook. Check this to prevent infinite loops.

### SubagentStart Input

```json
{
  "hook_event_name": "SubagentStart",
  "agent_id": "agent_abc123",
  "agent_type": "db-agent"
}
```

### PreCompact Input

```json
{
  "hook_event_name": "PreCompact",
  "trigger": "manual",
  "custom_instructions": "Focus on the authentication refactor"
}
```

### SessionStart Input

```json
{
  "hook_event_name": "SessionStart",
  "source": "startup"
}
```

### SessionEnd Input

```json
{
  "hook_event_name": "SessionEnd",
  "reason": "prompt_input_exit"
}
```

### Setup Input

```json
{
  "hook_event_name": "Setup",
  "trigger": "init"
}
```

---

## Output & Decision Control

### Exit Codes Summary

| Exit Code | General Behavior |
|-----------|-----------------|
| **0** | Success. Output handling varies by hook type (see Exit Code Reference table above) |
| **2** | Blocking action. Behavior varies by hook type (block tool, show to model, block prompt, etc.) |
| **Other** | Non-blocking error. stderr shown to user only in most cases |

### JSON Output Structure

Return structured JSON to `stdout` for sophisticated control:

```json
{
  "continue": true,
  "stopReason": "Message shown when continue is false",
  "suppressOutput": false,
  "systemMessage": "Warning message shown to user",
  "decision": "block",
  "reason": "Explanation for decision",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "Auto-approved",
    "updatedInput": {},
    "additionalContext": "Extra info for Claude"
  }
}
```

### Decision Control by Hook Type

| Hook | Available Decisions |
|------|---------------------|
| PreToolUse | `allow`, `deny`, `ask` + `updatedInput` |
| PermissionRequest | `allow`, `deny` + `updatedInput`, `message`, `interrupt` |
| PostToolUse | `block` + `reason`, `additionalContext` |
| UserPromptSubmit | `block` + `reason`, `additionalContext` |
| Stop / SubagentStop | `block` + `reason` |
| SessionStart | `additionalContext` only |
| PreCompact | stdout appended as custom instructions |

---

## Working with MCP Tools

MCP tools follow the pattern `mcp__<server>__<tool>`:

```
mcp__memory__create_entities
mcp__filesystem__read_file
mcp__github__search_repositories
mcp__playwright__browser_click
```

### Targeting MCP Tools

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "mcp__memory__.*",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Memory operation' >> ~/mcp.log"
          }
        ]
      },
      {
        "matcher": "mcp__.*__write.*",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/validate-mcp-write.py"
          }
        ]
      }
    ]
  }
}
```

---

## Best Practices

### Do

- **Validate and sanitize inputs** - Never trust input data blindly
- **Always quote shell variables** - Use `"$VAR"` not `$VAR`
- **Use absolute paths** - Use `$CLAUDE_PROJECT_DIR` for project scripts
- **Check `stop_hook_active`** - Prevent infinite loops in Stop hooks
- **Test hooks manually first** - Run commands in isolation before configuring
- **Use specific matchers** - Target specific tools rather than `*` for performance
- **Handle errors gracefully** - Provide clear error messages

### Don't

- **Block path traversal** - Check for `..` in file paths
- **Touch sensitive files** - Skip `.env`, `.git/`, keys, credentials
- **Run expensive operations on every tool** - Scope matchers precisely
- **Forget timeout configuration** - Long-running hooks can slow everything down
- **Ignore exit codes** - Use appropriate codes for your intent

### Performance Tips

- Hooks run in parallel when multiple match
- Timeout defaults to 60 seconds (configurable per hook)
- Identical hook commands are deduplicated automatically
- Use matchers to avoid running hooks unnecessarily

---

## Debugging

### Enable Debug Mode

```bash
claude --debug
```

### Check Hook Configuration

```
/hooks
```

### Debug Output Example

```
[DEBUG] Executing hooks for PostToolUse:Write
[DEBUG] Getting matching hook commands for PostToolUse with query: Write
[DEBUG] Found 1 hook matchers in settings
[DEBUG] Matched 1 hooks for query "Write"
[DEBUG] Found 1 hook commands to execute
[DEBUG] Executing hook command: ./scripts/format.sh with timeout 60000ms
[DEBUG] Hook command completed with status 0: Formatted successfully
```

### Common Issues

| Problem | Solution |
|---------|----------|
| Hook not running | Check `/hooks`, verify matcher pattern, ensure script is executable |
| JSON not parsing | Ensure exit code is 0, check JSON syntax |
| Command not found | Use absolute paths or `$CLAUDE_PROJECT_DIR` |
| Infinite loop | Check `stop_hook_active` in Stop hooks |
| Changes not taking effect | Hooks snapshot at startup; use `/hooks` to review changes |

---

## Complete Example Configuration

```json
{
  "hooks": {
    "Setup": [
      {
        "matcher": "init",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/init-project.sh"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/load-context.sh"
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/inject-sprint-context.py"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/validate-bash.py"
          }
        ]
      }
    ],
    "PermissionRequest": [
      {
        "matcher": "Read",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/auto-approve-reads.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/format-and-lint.sh"
          }
        ]
      }
    ],
    "PostToolUseFailure": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/log-failure.sh"
          }
        ]
      }
    ],
    "SubagentStart": [
      {
        "matcher": "db-agent",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/setup-db.sh"
          }
        ]
      }
    ],
    "SubagentStop": [
      {
        "matcher": "db-agent",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/cleanup-db.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Check if all tasks are complete. Context: $ARGUMENTS. Return {\"decision\": \"approve\" or \"block\", \"reason\": \"...\"}",
            "timeout": 30
          }
        ]
      }
    ],
    "Notification": [
      {
        "matcher": "permission_prompt",
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude needs permission\" with title \"Claude Code\"'"
          }
        ]
      }
    ],
    "PreCompact": [
      {
        "matcher": "auto",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/backup-transcript.sh"
          }
        ]
      }
    ],
    "SessionEnd": [
      {
        "matcher": "prompt_input_exit",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/session-summary.sh"
          }
        ]
      }
    ]
  }
}
```

---

## Resources

- [Official Hooks Reference](https://code.claude.com/docs/en/hooks)
- [Hooks Getting Started Guide](https://code.claude.com/docs/en/hooks-guide)
- [Subagents Documentation](https://code.claude.com/docs/en/sub-agents)
- [Settings Reference](https://code.claude.com/docs/en/settings)

---

*Last updated: January 2026*
