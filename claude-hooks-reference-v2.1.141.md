# Claude Code Hooks in 2.1.141

The full 2.1.141 hooks writeup is `hooks-v2.1.141.md`. This file exists as the
versioned companion for the older `claude-hooks-reference-*` docs and summarizes
the 2.1.141 source shape.

## Hook System Shape

Hooks are configured through settings and plugin-provided configuration, then
executed by the hook runtime under:

- `source/src/types/hooks.ts`
- `source/src/services/tools/toolHooks.ts`
- `source/src/query/stopHooks.ts`
- `source/src/commands/hooks`
- `source/src/utils/hooks`

The system supports command hooks, prompt hooks, agent hooks, HTTP hooks, and
internal non-configurable hooks.

## Important Event Families

The 2.1.141 hook list includes tool, prompt, session, compact, permission,
subagent, teammate, task, worktree, config, and elicitation events. Events
covered in detail in the full writeup include:

- `PreToolUse`
- `PostToolUse`
- `PostToolUseFailure`
- `Notification`
- `UserPromptSubmit`
- `SessionStart`
- `SessionEnd`
- `Stop`
- `StopFailure`
- `SubagentStart`
- `SubagentStop`
- `PreCompact`
- `PostCompact`
- `PermissionRequest`
- `Setup`
- `TeammateIdle`
- `TaskCompleted`
- `Elicitation`
- `ElicitationResult`
- `ConfigChange`
- `WorktreeCreate`
- `WorktreeRemove`
- `InstructionsLoaded`

## Output Semantics

Hook output can affect execution through:

- decisions such as allow/deny/block/continue depending on event type.
- `additionalContext` injection for supported events.
- async hook progress.
- exit-code semantics for command hooks.
- cancellation behavior when a hook exits with interrupting status.

Tool hooks aggregate decisions and additional context in
`source/src/services/tools/toolHooks.ts`. Stop hooks and prompt hooks are
handled in `source/src/query/stopHooks.ts`.

## Environment and Execution

Command hooks receive JSON on stdin, run with a constructed hook environment,
and can be configured with timeout and matcher behavior. Hook process
environment construction is covered in detail in `hooks-v2.1.141.md` and the
environment-variable reference.

## Version-Specific Note

In 2.1.141 the canonical subagent tool name is `Agent`; legacy `Task` still
exists as an alias. Hook matchers should prefer `Agent` for new configs and use
`Task` only when compatibility with legacy configs is required.

## Canonical Full Doc

Use `hooks-v2.1.141.md` for exhaustive schemas, examples, matcher semantics,
HTTP hook behavior, command-hook exit codes, and per-event input/output shapes.
