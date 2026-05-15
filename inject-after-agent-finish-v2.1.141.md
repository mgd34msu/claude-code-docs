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
