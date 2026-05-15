# Injecting Context After an Agent Finishes in Claude Code 2.1.141

The 2.1.141 way to inject context after a subagent finishes is a `PostToolUse`
hook targeting the `Agent` tool. The older `Task` name is still accepted as a
legacy alias, but new configurations should use `Agent`.

## Why PostToolUse

`Agent` is implemented as a normal tool. When it finishes, the tool hook system
has access to:

- tool name.
- tool input.
- tool result.
- transcript/session identifiers.
- hook matcher metadata.

That makes `PostToolUse` the most precise point for adding follow-up context
after a subagent returns.

## Tool Name in 2.1.141

Canonical name:

- `Agent`

Legacy alias:

- `Task`

The alias is defined on the tool and legacy permission normalization maps
`Task` to `Agent`. SDK compatibility code still maps `Agent` back to `Task` for
older SDK-facing system init/tool lists, but internal tool lookup uses the
canonical name plus aliases.

## Basic Hook Shape

Settings shape:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Agent",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/script"
          }
        ]
      }
    ]
  }
}
```

The hook command receives JSON on stdin. It can return JSON on stdout containing
`additionalContext` when the event supports it. That additional context is
aggregated by the hook runtime and inserted into the conversation.

## Matching Legacy Configs

For compatibility with legacy configs:

```json
{
  "matcher": "Task"
}
```

The safer 2.1.141 config is:

```json
{
  "matcher": "Agent"
}
```

If a config must support both old and new releases, use separate entries or
verify matcher alias behavior in the target release.

## Example Script Output

```json
{
  "additionalContext": "The subagent finished. Apply these follow-up constraints before continuing..."
}
```

The script should not print unrelated text to stdout if stdout is expected to be
parsed as hook JSON.

## Practical Uses

Good uses:

- summarize subagent findings into a parent-facing constraint.
- add a review checklist after a specialized agent returns.
- record and re-inject a path to generated artifacts.
- ask the parent to run verification when an agent completed implementation.

Poor uses:

- replacing normal subagent return content.
- injecting large unrelated documents.
- trying to override permission decisions after the fact.
- using `Stop` hooks when the exact target is a finished `Agent` call.

## Events That Can Be Relevant

Primary event:

- `PostToolUse`

Sometimes relevant:

- `SubagentStop`
- `Stop`
- `TaskCompleted`

`PostToolUse` is still the direct match for "after the Agent tool returned".

## 2.1.141 Source Index

- `source/src/tools/AgentTool/constants.ts`
- `source/src/tools/AgentTool/AgentTool.tsx`
- `source/src/Tool.ts`
- `source/src/utils/messages/systemInit.ts`
- `source/src/utils/permissions/permissionRuleParser.ts`
- `source/src/services/tools/toolHooks.ts`
- `hooks-v2.1.141.md`

## Complete Execution Flow

1. Model calls `Agent`.
2. `findToolByName()` resolves `Agent` or legacy `Task`.
3. Permission system evaluates the agent spawn.
4. `PreToolUse` hooks run.
5. The subagent executes synchronously or launches in background.
6. The Agent tool returns a tool result.
7. `PostToolUse` hooks for `Agent` run.
8. Hook runtime collects `additionalContext`.
9. Additional context is appended to the conversation for the parent model.

This works because the Agent tool is still a tool. It is not a special hidden
conversation type from the hook system's point of view.

## Hook Input Fields to Expect

The exact schema is in `hooks-v2.1.141.md`, but for this use case the important
fields are:

- session id.
- transcript path.
- cwd.
- hook event name.
- tool name.
- tool input.
- tool response/result.

The `tool_input` for Agent can include:

- `description`
- `prompt`
- `subagent_type`
- `model`
- `run_in_background`
- `name`
- `team_name`
- `mode`
- `isolation`
- `cwd`

Not every field appears in every call.

## Hook Output Contract

The hook should write JSON to stdout:

```json
{
  "additionalContext": "Context for the parent model after the agent result."
}
```

If the hook needs to block or deny behavior, use the event-specific decision
fields documented in the hooks reference. For this use case, `additionalContext`
is usually enough.

## Example: Summarize Agent Result

Settings:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Agent",
        "hooks": [
          {
            "type": "command",
            "command": "$HOME/.claude/hooks/agent-postprocess.py"
          }
        ]
      }
    ]
  }
}
```

Script behavior:

- read JSON from stdin.
- confirm `tool_name` is `Agent`.
- inspect `tool_input.subagent_type`.
- inspect the result summary.
- print compact JSON with `additionalContext`.

Output:

```json
{
  "additionalContext": "The research agent finished. Before implementing, verify the API names against source/src/services/api and run the relevant tests."
}
```

## Example: Conditional Context by Subagent Type

Use `tool_input.subagent_type`:

- `code-reviewer`: inject review checklist.
- `test-runner`: inject failed test summary path.
- `researcher`: inject source paths that need follow-up.
- undefined/fork subagent: inject generic summary.

The hook should be conservative. If it cannot parse the result, it should emit
no additional context rather than hallucinating a summary.

## Why Not Stop Hook

`Stop` runs at the end of the parent turn. That is broader than "an Agent just
finished". It can work for some global behaviors, but it is less precise:

- multiple tools may have run.
- the parent may have added more content after Agent returned.
- matching a specific subagent is harder.
- it can trigger when no Agent ran.

Use `Stop` for whole-turn policy. Use `PostToolUse` for tool-specific behavior.

## Why Not SubagentStop Hook

`SubagentStop` is useful inside subagent lifecycle logic, especially for
subagent-specific cleanup or reporting. But if the target is injecting context
into the parent after the Agent tool returns, `PostToolUse` is closer to the
parent-visible result boundary.

## Async Agents Caveat

If `Agent` launches an async/background task, `PostToolUse` fires when the
launch result returns, not necessarily when the background agent has completed.
For completed background work, use:

- `TaskOutput`
- task notifications.
- Agent View transcript.
- task completion hooks/events.

For synchronous agents, `PostToolUse` is the completion boundary.

## Matcher Safety

New configs should use:

```json
{ "matcher": "Agent" }
```

Legacy `Task` may still match because `Agent` has alias `Task`, but relying on
the old name makes future-version migration harder.

## Common Mistakes

- Printing non-JSON logs to stdout.
- Returning huge `additionalContext`.
- Matching `Task` in new-only configs.
- Using `Stop` when only Agent results matter.
- Assuming async Agent launch means async Agent completion.
- Injecting instructions that contradict the Agent result.
- Reading files from a hook without handling missing paths/timeouts.

## Recommended Hook Pattern

For synchronous subagents, the safest 2.1.141 pattern is:

1. match `PostToolUse` for `Agent`.
2. read the hook input JSON.
3. extract the agent description/type and tool result.
4. generate a short summary or policy note.
5. emit JSON with `additionalContext`.
6. keep human logs on stderr.

The hook should not attempt to rewrite the transcript. It should inject
contextual guidance for the next model step.

## Async Agent Pattern

For background agents, `PostToolUse` only sees launch success. Completion context
arrives later through:

- task notification messages.
- `TaskOutput`.
- Agent View transcript.
- task termination SDK events.
- task-specific hooks/events where available.

If the goal is "after the background agent is actually done", do not rely on
the `Agent` launch result. Use the task ID returned by the launch and follow the
task output/completion path.

## Additional Context Shape

Good post-agent injected context should include:

- which agent ran.
- what it concluded.
- what files or tasks it changed.
- what follow-up is required.
- any confidence/uncertainty that matters.
- a clear boundary between facts and recommendations.

Bad context includes:

- the entire subagent transcript.
- raw logs with no summary.
- secrets.
- unrelated project instructions.
- instructions that override the user request.
- stale state from earlier runs.

## Matcher Migration Notes

Older configs may say:

```json
{ "matcher": "Task" }
```

2.1.141 canonical config should say:

```json
{ "matcher": "Agent" }
```

The alias exists for compatibility, but using `Task` in new docs creates
avoidable confusion because `Task*` tools now also refer to task-list and task
output/stop functionality.

## Failure Handling

Pair `PostToolUse` with `PostToolUseFailure` when failures matter. A failed
agent launch is different from a successful launch whose background task later
fails:

- failed launch -> `PostToolUseFailure` on `Agent`.
- successful synchronous failure result -> `PostToolUse` sees the result.
- successful async launch followed by failure -> task completion/failure path.

Document which boundary your hook is targeting.

## Future Diff Checklist

For a later release:

1. Verify the canonical agent tool name.
2. Verify aliases in `AgentTool`.
3. Verify hook event names and output fields.
4. Verify async launch result shape.
5. Verify task notification message shape.
6. Verify `TaskOutput` aliases and schema.
7. Verify Agent View transcript state.
8. Verify SDK task termination events.
9. Verify hook additional-context merge behavior.
10. Test synchronous and async agents separately.
