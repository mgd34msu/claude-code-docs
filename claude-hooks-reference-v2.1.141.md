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

## Event Quick Reference

Tool events:

- `PreToolUse`: before a tool executes; can inspect/modify/deny.
- `PostToolUse`: after a tool succeeds; can inject context.
- `PostToolUseFailure`: after a tool fails; can inject failure context.

Prompt/session events:

- `UserPromptSubmit`: after user prompt submission and before model processing.
- `SessionStart`: session initialization.
- `SessionEnd`: session shutdown/end.
- `Stop`: normal stop boundary after model response.
- `StopFailure`: stop hook failure path.

Subagent/team/task events:

- `SubagentStart`
- `SubagentStop`
- `TeammateIdle`
- `TaskCompleted`

Compaction/config/worktree:

- `PreCompact`
- `PostCompact`
- `ConfigChange`
- `WorktreeCreate`
- `WorktreeRemove`
- `InstructionsLoaded`

Interaction events:

- `Notification`
- `PermissionRequest`
- `Setup`
- `Elicitation`
- `ElicitationResult`

## Hook Types

Command hooks:

- run a local command.
- receive JSON on stdin.
- return JSON on stdout.
- use exit codes plus output schema.
- have timeout and environment behavior.

Prompt hooks:

- run a model prompt hook.
- useful for semantic classification or generated context.
- more expensive than command hooks.

Agent hooks:

- run an agent-style hook.
- can use broader model/tool behavior depending on config.

HTTP hooks:

- POST hook payloads to configured URLs.
- blocked for events where HTTP hooks are unsafe.
- include private IP blocking and URL allowlist checks.

Internal hooks:

- not user-configurable.
- used by product features and analytics.

## Output Concepts

Common output concepts:

- `decision`: event-specific allow/block/deny/continue behavior.
- `reason`: human/model-facing reason text.
- `additionalContext`: extra model context for supported events.
- `updatedInput`: tool-input mutation for supported pre-tool paths.
- async hook responses and progress.

Decision aggregation is event-specific. A deny/block decision generally wins
over passive context additions.

## Security and Policy

2.1.141 hook security includes:

- managed setting policies.
- `disableAllHooks`.
- managed-hooks-only style policy.
- workspace trust checks.
- simple/bare mode reductions.
- HTTP private IP blocking.
- HTTP URL allowlists.
- plugin hook variable substitution boundaries.
- subprocess environment construction/scrubbing.

Hooks are powerful enough to alter model context and tool execution. Treat them
as code execution, not as passive configuration.

## Additional Context Rules

Use `additionalContext` when the hook has concise, current, trustworthy
information the model would not otherwise have.

Good:

- command output summary.
- file generated by a tool.
- policy note based on current repo state.
- subagent result follow-up.

Bad:

- large copied documents.
- secrets.
- stale project notes.
- instructions that conflict with user prompt.
- logs printed accidentally to stdout.

## 2.1.141 Compatibility Note

The canonical subagent tool name is `Agent`. Legacy docs and SDK surfaces may
still use `Task`. For hooks written specifically for 2.1.141 and later, prefer
`Agent` matchers.

## Event Selection Guide

Use `PreToolUse` when:

- you need to block a tool before it runs.
- you need to adjust tool input before execution.
- you need to enforce repo-specific policy.
- you need to add current context before a tool result exists.

Use `PostToolUse` when:

- you need the successful tool result.
- you want to add context based on generated files.
- you want to summarize a subagent result.
- you want to react to a completed edit/read/search.

Use `PostToolUseFailure` when:

- a failed tool call should produce recovery guidance.
- failures need logging or notification.
- a failed command should inject diagnostic context.

Use `Stop` when:

- the main assistant turn is about to stop.
- you need final validation of the overall response.
- you need to inject final reminders or prevent completion.

Use `SubagentStop` when:

- the lifecycle boundary is inside a subagent.
- the hook should apply to subagent-specific cleanup.
- the parent does not need immediate `PostToolUse` context.

Use `SessionStart`/`Setup` when:

- the session needs initialization.
- external tools need one-time setup.
- policy needs to run before normal interaction.

## Matcher Rules

Tool matchers should use canonical tool names in new configs:

- use `Agent`, not `Task`.
- use `TaskStop`, not `KillShell`.
- use `TaskOutput`, not `AgentOutputTool` or `BashOutputTool`.
- use `SendUserMessage`, not `Brief`.

Legacy names can still work where aliases are supported, but canonical names are
the safer long-term choice.

## Output Handling

Hook output has two different audiences:

- machine-readable JSON fields for Claude Code.
- stderr/debug output for humans.

Do not print human logs to stdout if stdout is expected to contain JSON. If a
hook needs to emit both, send diagnostics to stderr and structured control data
to stdout.

## Security Model

Hooks are code execution. Treat them as privileged:

- they can observe tool inputs.
- they can block tool execution.
- they can inject model context.
- they can run subprocesses.
- they can read local files if the hook script does so.
- they can leak secrets if written carelessly.

Managed policy, workspace trust, hook disablement, and simple/bare mode can all
change whether hooks run.

## Troubleshooting

If a hook does not fire:

- verify hooks are not disabled.
- verify simple/bare mode is not active.
- verify managed policy permits hooks.
- verify the event name is correct.
- verify the matcher uses the canonical tool name.
- verify JSON config shape.
- verify the command exits successfully.
- verify stdout is valid JSON when JSON is expected.
- verify the hook file is executable or invoked through an interpreter.
- verify the relevant tool actually ran in this session mode.

## Future Diff Checklist

For a later release:

1. Inspect hook schemas.
2. Inspect hook execution environment construction.
3. Inspect hook event names.
4. Inspect hook output parsing.
5. Inspect managed policy controls.
6. Inspect plugin hook substitution behavior.
7. Inspect HTTP hook allow/block behavior.
8. Inspect SDK stream-json hook event inclusion.
9. Inspect tool alias changes that affect matchers.
10. Reconcile this quick reference with the full hooks document.
