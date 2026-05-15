# Claude Code Hooks in 2.1.141

This document describes the hook system present in the local 2.1.141
reconstruction at `/home/buzzkill/Projects/lab/cc-linux-141`. It is based on
the reconstructed local source, especially:

- `source/src/utils/hooks.ts`
- `source/src/schemas/hooks.ts`
- `source/src/types/hooks.ts`
- `source/src/entrypoints/sdk/coreTypes.ts`
- `source/src/entrypoints/sdk/coreSchemas.ts`
- `source/src/utils/hooks/hooksConfigManager.ts`
- `source/src/utils/hooks/hooksSettings.ts`
- `source/src/utils/hooks/hooksConfigSnapshot.ts`
- `source/src/utils/hooks/sessionHooks.ts`
- `source/src/utils/hooks/AsyncHookRegistry.ts`
- `source/src/utils/hooks/execPromptHook.ts`
- `source/src/utils/hooks/execAgentHook.ts`
- `source/src/utils/hooks/execHttpHook.ts`
- `source/src/utils/hooks/fileChangedWatcher.ts`
- `source/src/utils/plugins/loadPluginHooks.ts`
- `source/src/utils/sessionStart.ts`
- `source/src/utils/processUserInput/processUserInput.ts`
- `source/src/services/tools/toolHooks.ts`
- `source/src/query/stopHooks.ts`
- `source/src/services/mcp/elicitationHandler.ts`
- `source/src/utils/worktree.ts`

The scope here is user-facing and integration-facing hooks, not React hooks.
Where a type appears in generated SDK schemas but is not visibly dispatched by
the reconstructed runtime, this writeup calls that out explicitly.

## Executive summary

Claude Code 2.1.141 has a broad hook system centered on the `hooks` setting.
Hooks are configured by event name. Each event contains matcher blocks. Each
matcher contains one or more hook actions.

The runtime-supported settings/plugin/session hook events in the reconstructed
source are:

1. `PreToolUse`
2. `PostToolUse`
3. `PostToolUseFailure`
4. `PermissionDenied`
5. `Notification`
6. `UserPromptSubmit`
7. `SessionStart`
8. `SessionEnd`
9. `Stop`
10. `StopFailure`
11. `SubagentStart`
12. `SubagentStop`
13. `PreCompact`
14. `PostCompact`
15. `PermissionRequest`
16. `Setup`
17. `TeammateIdle`
18. `TaskCreated`
19. `TaskCompleted`
20. `Elicitation`
21. `ElicitationResult`
22. `ConfigChange`
23. `WorktreeCreate`
24. `WorktreeRemove`
25. `InstructionsLoaded`
26. `CwdChanged`
27. `FileChanged`

There are also two command-driven customization surfaces implemented through
the same command-execution machinery, but not through the `hooks` map:

- `statusLine`
- `fileSuggestion`

The hook action types exposed by the reconstructed settings schema are:

- `command`: run a shell command and feed the hook input JSON on stdin.
- `prompt`: ask a small model to evaluate a boolean condition.
- `agent`: run an agentic verifier that can inspect the workspace and return
  structured output.
- `http`: POST the hook input JSON to a configured URL and parse JSON response.

Internal code also supports:

- `callback`: in-process SDK or built-in callback hooks.
- `function`: in-process session-scoped boolean hooks, mainly used to enforce
  structured output in agent/prompt flows.

Hooks are powerful because they can:

- Block or allow tool use.
- Modify tool input before a tool runs.
- Add context to the conversation.
- Modify MCP tool output after a tool runs.
- Enforce stop conditions before Claude ends a turn.
- Respond to MCP elicitations.
- Observe settings, skill, file, cwd, worktree, compaction, and notification
  events.
- Register dynamic file watch paths.
- Populate `CLAUDE_ENV_FILE` for environment changes consumed by later bash
  commands.

## Basic configuration shape

Hooks live under `hooks` in settings:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash|Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "python3 .claude/hooks/pretool.py",
            "timeout": 30
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npm run format --silent",
            "if": "Write(*.ts)",
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

The schema for each event is:

```ts
Partial<Record<HookEvent, HookMatcher[]>>
```

Each `HookMatcher` is:

```ts
{
  matcher?: string
  hooks: HookCommand[]
}
```

Each hook command is one of:

```ts
CommandHook | PromptHook | AgentHook | HttpHook
```

## Configuration sources

Hooks can come from multiple sources:

- User settings: `~/.claude/settings.json`
- Project settings: `.claude/settings.json`
- Local project settings: `.claude/settings.local.json`
- Managed/policy settings
- Plugin hooks
- Skill frontmatter hooks
- Agent frontmatter hooks
- SDK callback hooks
- Temporary in-memory session hooks
- Built-in internal callbacks

The UI for `/hooks` is read-only in 2.1.141. It displays configured hooks and
their sources, but users edit settings files directly or ask Claude to update
settings.

Plugin hooks use the same format as settings hooks. Plugin hook configs can
come from `hooks/hooks.json` or plugin manifest hook declarations. When plugin
hooks run, command hooks receive plugin-specific substitutions and environment
variables.

Skill and agent frontmatter hooks are registered as session hooks. They are
temporary and in-memory. For agent frontmatter, `Stop` hooks are converted to
`SubagentStop`, because subagents fire `SubagentStop` rather than top-level
`Stop`.

## Security and policy gates

Hooks execute arbitrary code, so the runtime applies several gates.

### Workspace trust

All hooks require workspace trust in interactive mode. If the workspace trust
dialog has not been accepted, hooks are skipped. Non-interactive SDK sessions
treat trust as implicit.

### `CLAUDE_CODE_SIMPLE`

If `CLAUDE_CODE_SIMPLE` is truthy, hook execution returns early.

### Bare mode

Session startup/setup paths explicitly skip hook work in bare mode.

### `disableAllHooks`

`disableAllHooks` affects hooks and the command-backed `statusLine` and
`fileSuggestion` surfaces.

If `disableAllHooks` is set in managed/policy settings, all hooks are disabled,
including managed hooks.

If `disableAllHooks` is set in non-managed settings, non-managed hooks are
disabled but managed hooks can still run. Runtime code treats this as
managed-only mode.

### `allowManagedHooksOnly`

If managed settings set `allowManagedHooksOnly: true`, only managed hooks run.
User, project, local, plugin, and session hooks are skipped or hidden.

### `strictPluginOnlyCustomization`

Managed settings can restrict customizations by surface. When the `hooks`
surface is restricted, user/project/local hooks are blocked and policy hooks
remain. Plugin hooks are handled separately by plugin policy.

### HTTP hook policy

HTTP hooks have additional network controls:

- `allowedHttpHookUrls`: optional allowlist of URL patterns. `undefined` means
  unrestricted, `[]` means block all, non-empty means target URL must match.
- `httpHookAllowedEnvVars`: optional allowlist of environment variables that
  HTTP hooks may interpolate into headers.
- SSRF guard blocks private/link-local/non-routable target IP ranges. Loopback
  is allowed for local dev hooks.
- If a sandbox network proxy or environment proxy is active, target DNS is
  performed by the proxy and the direct SSRF lookup guard is skipped.
- Redirects are disabled (`maxRedirects: 0`).

## Hook action types

### `command`

A command hook runs an external command. The hook input JSON is written to the
command's stdin, followed by a newline.

Schema fields in the reconstructed source:

```json
{
  "type": "command",
  "command": "string",
  "if": "optional permission-rule filter",
  "shell": "bash | powershell",
  "timeout": 30,
  "statusMessage": "optional spinner text",
  "once": false,
  "async": false,
  "asyncRewake": false
}
```

Important behavior:

- Default timeout is 10 minutes unless a caller supplies a smaller cap.
- `timeout` is in seconds.
- Default shell is the hook shell provider default, effectively bash unless
  configured otherwise.
- `shell: "powershell"` uses `pwsh` or `powershell` with `-NoProfile
  -NonInteractive -Command`.
- Bash hooks run through a shell. On Windows, bash hooks use Git Bash.
- PowerShell hooks skip bash-only path conversion, `.sh` auto-prefixing,
  `CLAUDE_CODE_SHELL_PREFIX`, and `CLAUDE_ENV_FILE`.
- Command hooks receive `CLAUDE_PROJECT_DIR`.
- Plugin command hooks receive `CLAUDE_PLUGIN_ROOT`; if the plugin has an ID,
  they also receive `CLAUDE_PLUGIN_DATA`.
- Plugin user config values are exposed as
  `CLAUDE_PLUGIN_OPTION_<SANITIZED_KEY>`.
- Skill hooks use `CLAUDE_PLUGIN_ROOT` for the skill root as well.
- `SessionStart`, `Setup`, `CwdChanged`, and `FileChanged` bash command hooks
  may receive `CLAUDE_ENV_FILE`.

Exit-code behavior is event dependent, but the common rule is:

- Exit `0`: success.
- Exit `2`: blocking feedback for events that support blocking.
- Other non-zero: non-blocking error shown/logged to the user.

Command hooks may output plain text or JSON. If stdout begins with `{`, Claude
tries to parse and validate it as hook JSON output. If stdout does not begin
with `{`, it is treated as plain text.

### `prompt`

A prompt hook asks a model to evaluate a condition. It is not a freeform
transform hook. The model must produce structured JSON:

```json
{
  "ok": true,
  "reason": "optional explanation"
}
```

Schema fields:

```json
{
  "type": "prompt",
  "prompt": "Verify that ... Use $ARGUMENTS if needed.",
  "if": "optional permission-rule filter",
  "timeout": 30,
  "model": "optional model name",
  "statusMessage": "optional spinner text",
  "once": false
}
```

Behavior:

- `$ARGUMENTS` is replaced with hook input JSON.
- Indexed forms like `$ARGUMENTS[0]` and `$0` are handled through the shared
  argument substitution helper.
- Default model is the small fast model.
- Default timeout is 30 seconds for prompt hook model calls.
- Prompt hooks call the model without streaming.
- For `Stop` and `SubagentStop`, Claude wraps the prompt as a stop-condition
  evaluator over the transcript.
- `ok: false` blocks and sets `preventContinuation`.
- Prompt hooks require `ToolUseContext`, so they are not supported in
  outside-REPL paths. Outside-REPL callers return an unsupported message for
  prompt hooks.

### `agent`

An agent hook runs an agentic verifier. It is heavier than a prompt hook: the
hook agent can use tools, inspect the codebase, and must finish by producing
structured output.

Schema fields:

```json
{
  "type": "agent",
  "prompt": "Verify that ... Use $ARGUMENTS if needed.",
  "if": "optional permission-rule filter",
  "timeout": 60,
  "model": "optional model name",
  "statusMessage": "optional spinner text",
  "once": false
}
```

Behavior:

- `$ARGUMENTS` is replaced with hook input JSON.
- Default model is the small fast model.
- Default timeout is 60 seconds.
- The hook agent gets a synthetic structured-output tool and must return:

```json
{
  "ok": true,
  "reason": "optional explanation"
}
```

- If the hook agent returns `ok: false`, the hook blocks.
- Agent hook execution is capped at 50 assistant turns.
- Agent hooks filter out tools disallowed for agents and avoid duplicate
  structured-output tools.
- The hook agent is allowed to read the transcript file.
- Agent hooks require `ToolUseContext`, so outside-REPL paths do not support
  them.

### `http`

HTTP hooks POST hook input JSON to a URL and require JSON responses.

Schema fields:

```json
{
  "type": "http",
  "url": "https://example.com/hook",
  "if": "optional permission-rule filter",
  "timeout": 30,
  "headers": {
    "Authorization": "Bearer $MY_TOKEN"
  },
  "allowedEnvVars": ["MY_TOKEN"],
  "statusMessage": "optional spinner text",
  "once": false
}
```

Behavior:

- Request method is POST.
- Body is the hook input JSON string.
- `Content-Type` is `application/json`.
- HTTP 2xx is success.
- Non-2xx is a non-blocking hook error unless the caller interprets structured
  output separately.
- Empty response body is treated like `{}`.
- Non-JSON response bodies are validation errors.
- HTTP hooks are skipped for `SessionStart` and `Setup` in the reconstructed
  runtime because headless mode can deadlock before the structured input
  consumer is ready.

### Internal `callback`

Callback hooks are registered through SDK/control or internal code. They are
not persisted to settings JSON.

They receive:

```ts
(
  input: HookInput,
  toolUseID: string | null,
  abort: AbortSignal | undefined,
  hookIndex?: number,
  context?: HookCallbackContext
) => Promise<HookJSONOutput>
```

Callback hooks can return the same JSON output shape as command hooks.

Internal callbacks can be marked `internal: true`. Internal-only batches use a
fast path and are excluded from `tengu_run_hook` metrics.

### Internal `function`

Function hooks are session-scoped, in-memory boolean validators. They are used
for internal enforcement, for example making a hook agent call the
structured-output tool. They cannot be persisted to settings.

Function hook callbacks receive the conversation messages and an optional abort
signal. Returning `false` creates a blocking error with the configured
`errorMessage`.

## Matcher semantics

Most events have a `matcher` field. The event determines which input field is
matched.

Matching rules:

- Empty matcher or `*` matches everything.
- Simple alphanumeric names match exactly.
- Pipe-separated simple names match any exact name, for example
  `Write|Edit|MultiEdit`.
- More complex strings are treated as regular expressions.
- Tool names are normalized for legacy aliases where applicable.
- Regex matching also tests legacy tool names so old patterns can still match.
- Invalid regex patterns do not match.

Event matcher fields:

| Event | Matcher field |
| --- | --- |
| `PreToolUse` | `tool_name` |
| `PostToolUse` | `tool_name` |
| `PostToolUseFailure` | `tool_name` |
| `PermissionRequest` | `tool_name` |
| `PermissionDenied` | `tool_name` |
| `SessionStart` | `source` |
| `Setup` | `trigger` |
| `PreCompact` | `trigger` |
| `PostCompact` | `trigger` |
| `Notification` | `notification_type` |
| `SessionEnd` | `reason` |
| `StopFailure` | `error` |
| `SubagentStart` | `agent_type` |
| `SubagentStop` | `agent_type` |
| `Elicitation` | `mcp_server_name` |
| `ElicitationResult` | `mcp_server_name` |
| `ConfigChange` | `source` |
| `InstructionsLoaded` | `load_reason` |
| `FileChanged` | basename of `file_path` |
| `Stop` | no matcher |
| `TeammateIdle` | no matcher |
| `TaskCreated` | no matcher |
| `TaskCompleted` | no matcher |
| `WorktreeCreate` | no matcher |
| `WorktreeRemove` | no matcher |
| `CwdChanged` | no matcher |

## `if` condition semantics

`command`, `prompt`, `agent`, and `http` hooks may include an `if` field.

The `if` field uses permission-rule syntax:

```json
{
  "type": "command",
  "command": "npm run lint",
  "if": "Bash(npm *)"
}
```

The runtime only evaluates `if` for tool-related events:

- `PreToolUse`
- `PostToolUse`
- `PostToolUseFailure`
- `PermissionRequest`

If an `if` condition is configured for a non-tool event, it cannot be evaluated
and the hook is skipped.

The `if` matcher is optimized so expensive tool-specific permission matching
is prepared once per hook event batch and reused across hooks.

## Deduplication and ordering

After matching, hooks are deduplicated by type and identity:

- `command`: shell + command + `if`
- `prompt`: prompt + `if`
- `agent`: prompt + `if`
- `http`: URL + `if`

The dedup key is namespaced by plugin root or skill root. This prevents two
different plugins with the same templated command from collapsing into one
hook.

Callback and function hooks are not deduplicated.

Hooks in a batch run in parallel. The runtime aggregates results as they finish.
For permission behavior, precedence is:

1. `deny`
2. `ask`
3. `allow`

An `allow` from a hook does not bypass deny or ask rules in settings. The
permission resolver still checks rule-based permissions after a hook allow.

## Common hook input fields

Every hook input starts with common fields:

```json
{
  "session_id": "string",
  "transcript_path": "string",
  "cwd": "string",
  "permission_mode": "optional string",
  "agent_id": "optional string",
  "agent_type": "optional string"
}
```

Notes:

- `session_id` is the current session ID unless a caller supplies one.
- `transcript_path` points to the JSONL transcript for the session.
- `cwd` is the current working directory.
- `permission_mode` is included when a caller has one.
- `agent_id` is present only for subagent contexts.
- `agent_type` can be present for subagent contexts or for main-thread
  `--agent` sessions.
- Generated SDK schemas mention an optional `effort` field, but the
  reconstructed `createBaseHookInput()` function does not populate it directly.
  Treat `effort` as schema-present but not reliably present in observed hook
  construction unless separately verified for a call site.

## Common hook JSON output

If a hook emits JSON, it must match the hook output schema.

Common sync output:

```json
{
  "continue": true,
  "suppressOutput": false,
  "stopReason": "optional reason",
  "decision": "approve",
  "reason": "optional decision reason",
  "systemMessage": "optional warning or message",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse"
  }
}
```

Common fields:

- `continue: false`: requests that Claude stop continuing after the hook.
- `stopReason`: message shown when continuation is stopped.
- `suppressOutput`: suppresses hook stdout from transcript output for JSON
  command hooks.
- `decision: "approve"`: maps to permission behavior `allow`.
- `decision: "block"`: maps to permission behavior `deny` and creates a
  blocking error using `reason` or a default.
- `reason`: reason for a decision or block.
- `systemMessage`: shown to the user as a hook system message.
- `hookSpecificOutput`: event-specific structured output.

The generated SDK schema also includes `terminalSequence`. I did not find
handling for this field in `source/src/utils/hooks.ts`, so treat it as
schema-present but not clearly processed by the reconstructed runtime.

## Async hook protocol

There are two ways a command hook can be async:

1. Configure the hook with `"async": true`.
2. Emit an async JSON line as the first line of stdout:

```json
{"async": true, "asyncTimeout": 15000}
```

Behavior:

- Async hooks are backgrounded and return success immediately to the main hook
  batch.
- The async hook registry tracks pending background hooks.
- When a background hook finishes, Claude scans stdout for the first non-async
  JSON response line.
- `asyncTimeout` defaults to 15000 ms if omitted.
- The registry emits hook response/progress events when hook event streaming is
  enabled.
- Completed `SessionStart` async hooks invalidate the session environment
  cache.

`asyncRewake` is a command-hook setting:

```json
{
  "type": "command",
  "command": "./long-stop-check.sh",
  "asyncRewake": true
}
```

`asyncRewake` implies async behavior. It bypasses the normal async registry.
If the hook exits with status `2`, Claude enqueues a task notification that can
wake or re-enter the model with a system reminder containing the blocking
output.

## Hook command prompt requests

Command hooks can ask the user to choose from options while they are running.
The hook writes a prompt request JSON line to stdout:

```json
{
  "prompt": "request-id",
  "message": "Select a deployment target",
  "options": [
    {
      "key": "staging",
      "label": "Staging",
      "description": "Deploy to staging"
    },
    {
      "key": "prod",
      "label": "Production"
    }
  ]
}
```

Claude detects that line, asks the user through the bound prompt callback, and
writes this response to the hook stdin:

```json
{
  "prompt_response": "request-id",
  "selected": "staging"
}
```

Processed prompt request lines are removed from final stdout before hook output
parsing, so they do not accidentally become hook results.

## Event reference

### `PreToolUse`

Fires before a tool executes.

Matcher: `tool_name`

Input fields:

```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {},
  "tool_use_id": "string"
}
```

Capabilities:

- Block the tool call.
- Approve the tool call.
- Force an ask flow.
- Modify `tool_input`.
- Add context to the model.
- Stop continuation.

Event-specific output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "approved by policy",
    "updatedInput": {},
    "additionalContext": "extra context for Claude"
  }
}
```

`permissionDecision` can be `allow`, `deny`, `ask`, and the generated SDK schema
also includes `defer`; the reconstructed `processHookJSONOutput()` switch
handles `allow`, `deny`, and `ask`.

Important semantics:

- Plain exit code `2` blocks with stderr.
- JSON `decision: "block"` blocks.
- JSON `hookSpecificOutput.permissionDecision: "deny"` blocks.
- `updatedInput` is applied if the hook allows or asks, or as a passthrough if
  no permission decision is returned.
- A hook allow skips the interactive prompt only if no deny/ask rule overrides
  it.

### `PostToolUse`

Fires after a tool succeeds.

Matcher: `tool_name`

Input fields:

```json
{
  "hook_event_name": "PostToolUse",
  "tool_name": "Write",
  "tool_input": {},
  "tool_response": {},
  "tool_use_id": "string",
  "duration_ms": 123
}
```

Capabilities:

- Add context to the next model request.
- Block/stop continuation by returning blocking output.
- Modify MCP tool output.

Event-specific output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "context after the tool",
    "updatedMCPToolOutput": {}
  }
}
```

Generated SDK schemas also include `updatedToolOutput`, described as replacing
the output for all tools. The reconstructed `processHookJSONOutput()` only
extracts `updatedMCPToolOutput`, and `runPostToolUseHooks()` only applies it
when the tool is an MCP tool. Treat `updatedToolOutput` as schema-present but
not clearly implemented in the reconstructed runtime.

### `PostToolUseFailure`

Fires after a tool execution fails.

Matcher: `tool_name`

Input fields:

```json
{
  "hook_event_name": "PostToolUseFailure",
  "tool_name": "Bash",
  "tool_input": {},
  "tool_use_id": "string",
  "error": "error text",
  "is_interrupt": false,
  "duration_ms": 123
}
```

Capabilities:

- Add context after failure.
- Emit blocking feedback.
- Emit non-blocking errors.

Event-specific output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUseFailure",
    "additionalContext": "failure-specific context"
  }
}
```

### `PermissionRequest`

Fires when Claude Code is about to display a permission dialog.

Matcher: `tool_name`

Input fields:

```json
{
  "hook_event_name": "PermissionRequest",
  "tool_name": "Bash",
  "tool_input": {},
  "permission_suggestions": []
}
```

Capabilities:

- Programmatically allow the permission request.
- Programmatically deny the permission request.
- Update the input on allow.
- Return permission updates.

Event-specific output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow",
      "updatedInput": {},
      "updatedPermissions": []
    }
  }
}
```

Deny output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "deny",
      "message": "Denied by policy",
      "interrupt": true
    }
  }
}
```

### `PermissionDenied`

Fires after the auto-mode classifier denies a tool call.

Matcher: `tool_name`

Input fields:

```json
{
  "hook_event_name": "PermissionDenied",
  "tool_name": "Bash",
  "tool_input": {},
  "tool_use_id": "string",
  "reason": "Permission denied"
}
```

Capabilities:

- Tell the model it may retry.

Event-specific output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionDenied",
    "retry": true
  }
}
```

This is specifically wired for transcript-classifier auto mode denials. If
`retry: true` is returned, Claude adds a meta message saying the hook indicated
the tool call may be retried.

### `Notification`

Fires when Claude Code sends notification events.

Matcher: `notification_type`

Known notification type values from metadata:

- `permission_prompt`
- `idle_prompt`
- `auth_success`
- `elicitation_dialog`
- `elicitation_complete`
- `elicitation_response`

Input fields:

```json
{
  "hook_event_name": "Notification",
  "message": "message text",
  "title": "optional title",
  "notification_type": "permission_prompt"
}
```

Event-specific output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "Notification",
    "additionalContext": "optional context"
  }
}
```

Notification hooks run outside the REPL path. Their errors are logged/debugged
rather than injected into the model like tool hooks.

### `UserPromptSubmit`

Fires after the user submits a prompt and before processing continues.

Matcher: none.

Input fields:

```json
{
  "hook_event_name": "UserPromptSubmit",
  "prompt": "original user prompt",
  "session_title": "optional schema field"
}
```

Capabilities:

- Block prompt processing.
- Stop continuation.
- Add context to the user prompt.

Event-specific output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "context to add"
  }
}
```

In generated SDK schemas, `UserPromptSubmit` output also has `sessionTitle` and
`suppressOriginalPrompt`. I did not find runtime handling for those fields in
the reconstructed `processHookJSONOutput()` / `processUserInput()` path.

Blocking behavior:

- Exit `2` or JSON block prevents the prompt from being queried.
- The current reconstructed `processUserInput()` path returns a warning system
  message that includes the original prompt text in the block message.
- `continue: false` stops processing but keeps the original prompt in context
  with an "Operation stopped by hook" message.

### `SessionStart`

Fires when a session starts, resumes, clears, or starts after compaction.

Matcher: `source`

Source values:

- `startup`
- `resume`
- `clear`
- `compact`

Input fields:

```json
{
  "hook_event_name": "SessionStart",
  "source": "startup",
  "agent_type": "optional agent type",
  "model": "optional model"
}
```

Capabilities:

- Add context.
- Set an initial user message.
- Register file watch paths.
- Write env exports to `CLAUDE_ENV_FILE` for bash hooks.

Event-specific output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "context for Claude",
    "initialUserMessage": "message to inject",
    "watchPaths": ["/absolute/path"]
  }
}
```

Notes:

- Plugin hooks are explicitly loaded before `SessionStart` hooks unless
  managed-only policy blocks plugin hooks.
- Blocking errors are ignored by the session start processor.
- HTTP hooks are skipped for this event.
- Bash command hooks may receive `CLAUDE_ENV_FILE`.

### `Setup`

Fires for repo setup hooks.

Matcher: `trigger`

Trigger values:

- `init`
- `maintenance`

Input fields:

```json
{
  "hook_event_name": "Setup",
  "trigger": "init"
}
```

Event-specific output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "Setup",
    "additionalContext": "context for Claude"
  }
}
```

Notes:

- Plugin hooks are loaded before setup hooks unless managed-only policy blocks
  plugin hooks.
- Blocking errors are ignored by setup processing.
- HTTP hooks are skipped for this event.
- Bash command hooks may receive `CLAUDE_ENV_FILE`.

### `Stop`

Fires right before Claude concludes its response.

Matcher: none.

Input fields:

```json
{
  "hook_event_name": "Stop",
  "stop_hook_active": false,
  "last_assistant_message": "optional text of last assistant message"
}
```

Capabilities:

- Prevent Claude from stopping.
- Provide feedback to the model.
- Enforce stop conditions.

Behavior:

- Exit `2` creates "Stop hook feedback" and continues the conversation.
- `continue: false` prevents continuation and records a stop reason.
- Prompt hooks for `Stop` are evaluated as stop-condition checks over the
  transcript.
- Agent hooks for `Stop` run an agentic verifier.
- Stop hook progress and summary messages are shown in the UI, with error
  details available in transcript mode.

### `StopFailure`

Fires when a turn ends due to an API-level assistant error instead of a normal
stop.

Matcher: `error`

Known matcher values from metadata:

- `rate_limit`
- `authentication_failed`
- `billing_error`
- `invalid_request`
- `server_error`
- `max_output_tokens`
- `unknown`

Input fields:

```json
{
  "hook_event_name": "StopFailure",
  "error": "rate_limit",
  "error_details": "optional details",
  "last_assistant_message": "optional last assistant text"
}
```

This hook is fire-and-forget. Hook output and exit codes are ignored by the
caller beyond debug/error handling.

### `SubagentStart`

Fires when an `Agent` tool subagent starts.

Matcher: `agent_type`

Input fields:

```json
{
  "hook_event_name": "SubagentStart",
  "agent_id": "agent id",
  "agent_type": "general-purpose"
}
```

Event-specific output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SubagentStart",
    "additionalContext": "context for the subagent"
  }
}
```

Blocking errors are ignored by the subagent start path.

### `SubagentStop`

Fires before a subagent concludes.

Matcher: `agent_type`

Input fields:

```json
{
  "hook_event_name": "SubagentStop",
  "stop_hook_active": false,
  "agent_id": "agent id",
  "agent_transcript_path": "path/to/subagent.jsonl",
  "agent_type": "general-purpose",
  "last_assistant_message": "optional text"
}
```

Behavior mirrors `Stop`, but feedback goes to the subagent. Agent frontmatter
`Stop` hooks are registered as `SubagentStop`.

### `PreCompact`

Fires before conversation compaction.

Matcher: `trigger`

Trigger values:

- `manual`
- `auto`

Input fields:

```json
{
  "hook_event_name": "PreCompact",
  "trigger": "manual",
  "custom_instructions": "optional existing custom instructions"
}
```

Behavior:

- Successful stdout from hooks is collected and appended as custom compact
  instructions.
- Hook execution results are summarized for user display.
- Exit `2` / JSON block is treated as a blocking result by the outside-REPL
  executor. Compaction callers check for blocked results and can prevent
  compaction.

### `PostCompact`

Fires after conversation compaction.

Matcher: `trigger`

Trigger values:

- `manual`
- `auto`

Input fields:

```json
{
  "hook_event_name": "PostCompact",
  "trigger": "manual",
  "compact_summary": "summary text"
}
```

Behavior:

- Hook success/failure is summarized for user display.
- Successful stdout is shown in the summary message.

### `SessionEnd`

Fires when a session ends.

Matcher: `reason`

Reason values in `ExitReason`:

- `clear`
- `resume`
- `logout`
- `prompt_input_exit`
- `other`
- `bypass_permissions_disabled`

Input fields:

```json
{
  "hook_event_name": "SessionEnd",
  "reason": "clear"
}
```

Behavior:

- Runs outside the REPL.
- During shutdown, failures are written to stderr.
- Session hooks are cleared after execution.
- Shutdown callers cap the whole event with
  `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`, default 1500 ms.
- A per-hook `timeout` can be configured, but shutdown callers also pass the
  overall abort signal cap.

### `TeammateIdle`

Fires when a teammate is about to go idle.

Matcher: none.

Input fields:

```json
{
  "hook_event_name": "TeammateIdle",
  "teammate_name": "name",
  "team_name": "team"
}
```

Behavior:

- Exit `2` blocks idle and sends feedback to the teammate/model.
- `continue: false` prevents continuation.

### `TaskCreated`

Fires when a task is being created.

Matcher: none.

Input fields:

```json
{
  "hook_event_name": "TaskCreated",
  "task_id": "id",
  "task_subject": "subject",
  "task_description": "optional description",
  "teammate_name": "optional teammate",
  "team_name": "optional team"
}
```

Behavior:

- Exit `2` blocks task creation.
- `continue: false` prevents continuation.

### `TaskCompleted`

Fires when a task is being marked complete.

Matcher: none.

Input fields:

```json
{
  "hook_event_name": "TaskCompleted",
  "task_id": "id",
  "task_subject": "subject",
  "task_description": "optional description",
  "teammate_name": "optional teammate",
  "team_name": "optional team"
}
```

Behavior:

- Exit `2` blocks task completion.
- Teammate stop processing runs this for in-progress tasks owned by the
  teammate before allowing the teammate to idle.

### `Elicitation`

Fires when an MCP server requests user input through elicitation.

Matcher: `mcp_server_name`

Input fields:

```json
{
  "hook_event_name": "Elicitation",
  "mcp_server_name": "server",
  "message": "prompt from server",
  "mode": "form",
  "url": "optional url",
  "elicitation_id": "optional id",
  "requested_schema": {}
}
```

Event-specific output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "Elicitation",
    "action": "accept",
    "content": {}
  }
}
```

Actions:

- `accept`
- `decline`
- `cancel`

Behavior:

- A hook can auto-respond instead of showing the dialog.
- `decline` creates a blocking error.
- Exit `2` blocks/denies the elicitation.

### `ElicitationResult`

Fires after the user responds to an MCP elicitation, before the response is
sent to the MCP server.

Matcher: `mcp_server_name`

Input fields:

```json
{
  "hook_event_name": "ElicitationResult",
  "mcp_server_name": "server",
  "elicitation_id": "optional id",
  "mode": "form",
  "action": "accept",
  "content": {}
}
```

Event-specific output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "ElicitationResult",
    "action": "decline",
    "content": {}
  }
}
```

Behavior:

- A hook can observe or override the action/content before sending to the MCP
  server.
- `decline` creates a blocking error.
- Exit `2` blocks the response.

### `ConfigChange`

Fires when configuration files change during a session.

Matcher: `source`

Source values:

- `user_settings`
- `project_settings`
- `local_settings`
- `policy_settings`
- `skills`

Input fields:

```json
{
  "hook_event_name": "ConfigChange",
  "source": "project_settings",
  "file_path": "optional changed path"
}
```

Behavior:

- Intended for auditing and policy reactions to settings/skills changes.
- Blocking is ignored for `policy_settings`; policy changes are enterprise
  managed and cannot be blocked by hooks.
- Skill config-change hooks can block skill changes from applying in-session.

### `InstructionsLoaded`

Fires when an instruction file is loaded into context.

Matcher: `load_reason`

Load reasons:

- `session_start`
- `nested_traversal`
- `path_glob_match`
- `include`
- `compact`

Memory types:

- `User`
- `Project`
- `Local`
- `Managed`

Input fields:

```json
{
  "hook_event_name": "InstructionsLoaded",
  "file_path": "/path/to/CLAUDE.md",
  "memory_type": "Project",
  "load_reason": "session_start",
  "globs": ["optional matching globs"],
  "trigger_file_path": "optional touched file path",
  "parent_file_path": "optional including file"
}
```

Behavior:

- Observability/audit only.
- Fire-and-forget.
- Does not support blocking.

Dispatch sites include eager session-start loads, reloads after compaction, and
lazy nested/conditional instruction loads when Claude touches a triggering file.

### `WorktreeCreate`

Fires when Claude Code needs a custom/VCS-agnostic isolated worktree.

Matcher: none.

Input fields:

```json
{
  "hook_event_name": "WorktreeCreate",
  "name": "suggested-worktree-slug"
}
```

Output:

- Command hooks should print the absolute worktree path on stdout.
- HTTP/callback hooks can use:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "WorktreeCreate",
    "worktreePath": "/absolute/path"
  }
}
```

Behavior:

- The first successful result with non-empty output is used.
- If no successful result provides output, worktree creation throws an error.

### `WorktreeRemove`

Fires when a previously created custom worktree should be removed.

Matcher: none.

Input fields:

```json
{
  "hook_event_name": "WorktreeRemove",
  "worktree_path": "/absolute/path"
}
```

Behavior:

- Returns true if hooks were configured and ran.
- Failures are logged with debug/error details.
- Non-zero exits do not necessarily stop the caller beyond logging.

### `CwdChanged`

Fires after the working directory changes.

Matcher: none.

Input fields:

```json
{
  "hook_event_name": "CwdChanged",
  "old_cwd": "/old/path",
  "new_cwd": "/new/path"
}
```

Capabilities:

- Write environment exports to `CLAUDE_ENV_FILE` for subsequent bash commands.
- Return file watch paths.
- Return system messages for the user.

Event-specific output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "CwdChanged",
    "watchPaths": ["/absolute/path"]
  }
}
```

Behavior:

- `clearCwdEnvFiles()` runs before `CwdChanged` hooks execute.
- If hooks return `watchPaths`, the file watcher is updated.
- System messages from hook JSON are shown through the env-hook notifier.

### `FileChanged`

Fires when a watched file changes, is added, or is unlinked.

Matcher: basename of `file_path`.

Input fields:

```json
{
  "hook_event_name": "FileChanged",
  "file_path": "/absolute/path/.env",
  "event": "change"
}
```

Event values:

- `change`
- `add`
- `unlink`

Capabilities:

- Write environment exports to `CLAUDE_ENV_FILE` for subsequent bash commands.
- Return new dynamic watch paths.
- Return system messages for the user.

Event-specific output:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "FileChanged",
    "watchPaths": ["/absolute/path"]
  }
}
```

Watcher behavior:

- Static watch paths come from `FileChanged` matcher strings.
- Matchers are split on `|`; relative names are resolved against current cwd.
- Dynamic watch paths from `CwdChanged`/`FileChanged` output are merged.
- The watcher uses `chokidar`, ignores initial events, and uses an
  `awaitWriteFinish` stability threshold.

## Status line command

`statusLine` is not an entry in `hooks`, but it uses hook command execution.

Settings shape:

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline.sh",
    "padding": 0
  }
}
```

Input type:

```json
{
  "session_id": "string",
  "transcript_path": "string",
  "cwd": "string",
  "permission_mode": "optional",
  "agent_id": "optional",
  "agent_type": "optional",
  "session_name": "optional",
  "model": {
    "id": "string",
    "display_name": "string"
  },
  "workspace": {
    "current_dir": "string",
    "project_dir": "string",
    "added_dirs": []
  },
  "version": "string",
  "output_style": {
    "name": "string"
  },
  "cost": {
    "total_cost_usd": 0,
    "total_duration_ms": 0,
    "total_api_duration_ms": 0,
    "total_lines_added": 0,
    "total_lines_removed": 0
  },
  "context_window": {
    "total_input_tokens": 0,
    "total_output_tokens": 0,
    "context_window_size": 200000,
    "current_usage": 0,
    "used_percentage": 0,
    "remaining_percentage": 100
  },
  "exceeds_200k_tokens": false,
  "rate_limits": {
    "five_hour": {
      "used_percentage": 0,
      "resets_at": "timestamp"
    },
    "seven_day": {
      "used_percentage": 0,
      "resets_at": "timestamp"
    }
  },
  "vim": {
    "mode": "INSERT"
  },
  "agent": {
    "name": "string"
  },
  "remote": {
    "session_id": "string"
  },
  "worktree": {
    "name": "string",
    "path": "string",
    "branch": "string",
    "original_cwd": "string",
    "original_branch": "string"
  }
}
```

Behavior:

- Default timeout is 5000 ms.
- Only stdout from exit `0` is used.
- Output is trimmed, empty lines are dropped, and remaining lines are joined
  with newlines.
- If managed policy disables all hooks, no status line runs.
- If non-managed `disableAllHooks` is set, only managed `statusLine` can run.
- The same trust check applies.

## File suggestion command

`fileSuggestion` is also not an entry in `hooks`, but it uses hook command
execution.

Settings shape:

```json
{
  "fileSuggestion": {
    "type": "command",
    "command": "~/.claude/file-suggest.sh"
  }
}
```

Input type:

```json
{
  "session_id": "string",
  "transcript_path": "string",
  "cwd": "string",
  "query": "partial @ mention query",
  "permission_mode": "optional",
  "agent_id": "optional",
  "agent_type": "optional"
}
```

Behavior:

- Default timeout is 5000 ms.
- Exit `0` stdout is split by newline.
- Non-empty trimmed lines become file suggestions.
- Non-zero, abort, or errors return no suggestions.
- Managed-only/disable/trust gates mirror `statusLine`.

## Hook event streaming

The hook event subsystem can emit hook execution events separate from the main
message stream.

Events:

- `started`
- `progress`
- `response`

Always emitted events:

- `SessionStart`
- `Setup`

All other hook event emissions require all hook events to be enabled, which is
done when SDK `includeHookEvents` is set or when remote mode enables it.

The progress interval defaults to 1000 ms. It emits only when output changes.

`response` events include:

- stdout
- stderr
- combined output
- exit code
- outcome: `success`, `error`, or `cancelled`

All hook responses are logged to the debug log even if they are not emitted to
an SDK hook-event handler.

## Telemetry and metrics

Hook execution records several analytics/telemetry signals.

Generic hook execution:

- `tengu_run_hook`
- `tengu_repl_hook_finished`

Hook error/cancel event names observed in source:

- `tengu_pre_tool_hook_error`
- `tengu_pre_tool_hooks_cancelled`
- `tengu_post_tool_hook_error`
- `tengu_post_tool_hooks_cancelled`
- `tengu_post_tool_failure_hook_error`
- `tengu_post_tool_failure_hooks_cancelled`
- `tengu_pre_stop_hooks_cancelled`

Agent hook telemetry:

- `tengu_agent_stop_hook_success`
- `tengu_agent_stop_hook_error`
- `tengu_agent_stop_hook_max_turns`

Beta tracing/OpenTelemetry hook events:

- `hook_execution_start`
- `hook_execution_complete`

Stats store observations:

- `hook_duration_ms`
- `pre_tool_hook_duration_ms`

Telemetry includes hook name, hook type counts, hook count, success/block/error
counts, durations, and sanitized plugin hook counts. Third-party plugin names
are sanitized to `third-party` unless they belong to official marketplaces.

## Event-specific output summary

| Event | `hookSpecificOutput` support in reconstructed runtime |
| --- | --- |
| `PreToolUse` | `permissionDecision`, `permissionDecisionReason`, `updatedInput`, `additionalContext` |
| `UserPromptSubmit` | `additionalContext` |
| `SessionStart` | `additionalContext`, `initialUserMessage`, `watchPaths` |
| `Setup` | `additionalContext` |
| `SubagentStart` | `additionalContext` |
| `PostToolUse` | `additionalContext`, `updatedMCPToolOutput` |
| `PostToolUseFailure` | `additionalContext` |
| `PermissionDenied` | `retry` |
| `Notification` | schema includes `additionalContext`; reconstructed `processHookJSONOutput()` does not handle a dedicated Notification branch, but generic `systemMessage` still works |
| `PermissionRequest` | `decision` allow/deny |
| `Elicitation` | `action`, `content` |
| `ElicitationResult` | `action`, `content` |
| `CwdChanged` | `watchPaths` |
| `FileChanged` | `watchPaths` |
| `WorktreeCreate` | `worktreePath` |

## Schema-only or inconsistent artifacts to audit

The local reconstructed source shows a few inconsistencies worth preserving in
the map for future versions:

- `source/src/entrypoints/sdk/coreSchemas.ts` includes `PostToolBatch` and
  `UserPromptExpansion` in its schema-level `HOOK_EVENTS`, and defines input
  and output schemas for both. The runtime `HOOK_EVENTS` exported from
  `source/src/entrypoints/sdk/coreTypes.ts` does not include them, the settings
  hook manager does not group/display them, and I did not find dispatch
  functions for them in `source/src/utils/hooks.ts`. Treat them as schema
  artifacts, not runtime-supported settings hooks, unless separately verified
  in the binary.
- Generated SDK schemas include `updatedToolOutput` for `PostToolUse`, but the
  reconstructed runtime only extracts and applies `updatedMCPToolOutput`.
- Generated SDK schemas include `terminalSequence`, but the reconstructed hook
  processor does not visibly handle it.
- Generated SDK schemas include `sessionTitle` and `suppressOriginalPrompt` for
  `UserPromptSubmit`, but the reconstructed hook processor does not visibly
  handle those fields.
- A map-only/minified snippet under `rebuild/snippets-2.1.141` appears to show
  a richer hook schema with command `args` and an `mcp_tool` hook type. The
  checked reconstructed `source/src/schemas/hooks.ts` does not include those
  fields/types and `source/src/utils/hooks.ts` has no execution branch for
  `mcp_tool`. This should be audited against the actual minified span before
  declaring `mcp_tool` publicly usable in this reconstruction.

## Practical examples

### Block dangerous bash commands before execution

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 .claude/hooks/block-dangerous-bash.py",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

The hook can return:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Command touches production credentials"
  }
}
```

### Modify tool input before normal permission flow

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "updatedInput": {
      "command": "git status --short"
    }
  }
}
```

If no permission decision is returned, the updated input is passed through to
normal permission evaluation.

### Add context after a file write

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "Formatter changed the file after Write; read before editing again."
  }
}
```

### Register env-file watch paths at session start

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "watchPaths": [
      "/home/me/project/.env",
      "/home/me/project/.envrc"
    ]
  }
}
```

### Auto-answer an MCP elicitation

```json
{
  "hookSpecificOutput": {
    "hookEventName": "Elicitation",
    "action": "accept",
    "content": {
      "environment": "staging"
    }
  }
}
```

### Custom file suggestions

```json
{
  "fileSuggestion": {
    "type": "command",
    "command": "~/.claude/file-suggestions.sh"
  }
}
```

The script reads JSON from stdin and prints one suggestion per line.

## Bottom line

Claude Code 2.1.141 exposes 27 runtime hook events through `hooks`, plus the
command-backed `statusLine` and `fileSuggestion` surfaces. The strongest
integration points are `PreToolUse`, `PostToolUse`, `UserPromptSubmit`,
`Stop`/`SubagentStop`, `SessionStart`, `PermissionRequest`,
`Elicitation`/`ElicitationResult`, and the env/file watcher pair
`CwdChanged`/`FileChanged`.

The system is intentionally broad: settings, managed policy, plugins,
frontmatter, SDK callbacks, and in-memory session hooks all merge into the same
execution path. The critical implementation details are that hooks run in
parallel, workspace trust is required in interactive mode, managed settings can
restrict or disable hooks, HTTP hooks have dedicated network controls, and
not every field present in generated SDK schemas is necessarily processed by
the reconstructed 2.1.141 runtime.
