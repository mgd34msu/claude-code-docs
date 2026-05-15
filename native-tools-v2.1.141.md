# Native Tools in Claude Code 2.1.141

This is the 2.1.141 built-in tool registry and behavior summary. The source of
truth is `source/src/tools.ts`, with lookup/alias behavior in
`source/src/Tool.ts`.

## Registry Pipeline

`getAllBaseTools()` builds the exhaustive built-in tool list for the current
environment. `getTools()` filters that list by simple mode, deny rules, REPL
visibility, and each tool's `isEnabled()` check. `assembleToolPool()` combines
built-ins with MCP tools and de-duplicates names so built-ins win conflicts.

## Core Tools

Always or commonly present:

- `Agent`
- `TaskOutput`
- `Bash`
- `Read`
- `Edit`
- `Write`
- `NotebookEdit`
- `WebFetch`
- `TodoWrite`
- `WebSearch`
- `TaskStop`
- `AskUserQuestion`
- `Skill`
- `EnterPlanMode`
- `ExitPlanMode`
- `SendUserMessage`
- `ListMcpResourcesTool`
- `ReadMcpResourceTool`

`Glob` and `Grep` are omitted when embedded search tools are available in the
native Bun binary.

## Conditional Tools

Conditional registrations include:

- `Config` for Anthropic internal users.
- `TaskCreate`, `TaskGet`, `TaskUpdate`, `TaskList` when todo-v2 tasks are
  enabled.
- `CtxInspect` when `feature('CONTEXT_COLLAPSE')` is present.
- `LSP` when `ENABLE_LSP_TOOL` is truthy.
- `EnterWorktree` and `ExitWorktree` when worktree mode is enabled.
- `TeamCreate` and `TeamDelete` when agent swarms are enabled.
- `REPL` for Anthropic internal REPL mode.
- `CronCreate`, `CronDelete`, `CronList` when `feature('AGENT_TRIGGERS')` is
  present and runtime cron gates allow use.
- `RemoteTrigger` when `feature('AGENT_TRIGGERS_REMOTE')` is present.
- `Monitor` when `feature('MONITOR_TOOL')` is present.
- `PushNotification` when Kairos push notification support is present.
- `PowerShell` when PowerShell support is enabled.
- `Snip` when `feature('HISTORY_SNIP')` is present.
- `ToolSearch` when optimistic tool search enablement says it may be available.

## Simple Mode

When `CLAUDE_CODE_SIMPLE` is truthy, the tool set is reduced to:

- `Bash`
- `Read`
- `Edit`

In REPL simple mode, `REPL` may replace those raw primitives. Coordinator mode
can add coordination-specific tools even under simple mode.

## Tool Metadata

Tool definitions support:

- `name`
- `aliases`
- `description`
- `prompt`
- `inputSchema`
- `isReadOnly`
- `isConcurrencySafe`
- `isEnabled`
- `shouldDefer`
- `alwaysLoad`
- `searchHint`
- permission and rendering hooks.

`ToolSearch` uses `searchHint` and deferred-tool metadata to expose a smaller
schema set to the model when tool search is active.

## Aliases

2.1.141 aliases include:

- `Task` -> `Agent`
- `KillShell` -> `TaskStop`
- `AgentOutputTool` -> `TaskOutput`
- `BashOutputTool` -> `TaskOutput`
- `Brief` -> `SendUserMessage`

Tool lookup checks both the primary name and aliases.

## Permission Filtering

`filterToolsByDenyRules()` removes tools that are blanket-denied by the current
permission context before the model sees the tool list. It uses the same
matching semantics as runtime permission checks, so MCP server-prefix deny rules
can hide whole MCP server tool families.

## MCP Tools

MCP tools are merged with built-ins by `assembleToolPool()`.

Important behavior:

- built-ins win duplicate-name conflicts.
- MCP tools are sorted to keep prompt caching stable.
- special internal tools are excluded from some model-facing lists.
- MCP resource tools are built-ins in 2.1.141.

## Shell Tools

`Bash` is the primary shell tool. `PowerShell` is available only when enabled by
platform/build/environment checks in `source/src/utils/shell/shellToolUtils.ts`.
`SHELL_TOOL_NAMES` includes both `Bash` and `PowerShell`.

PowerShell is default-on for Anthropic internal Windows paths, but external use
requires explicit enablement through `CLAUDE_CODE_USE_POWERSHELL_TOOL`.

## 2.1.141 Source Index

- `source/src/tools.ts`
- `source/src/Tool.ts`
- `source/src/tools/*`
- `source/src/services/tools/toolExecution.ts`
- `source/src/services/tools/toolOrchestration.ts`
- `source/src/services/tools/StreamingToolExecutor.ts`
- `source/src/utils/permissions`
- `source/src/utils/shell/shellToolUtils.ts`
