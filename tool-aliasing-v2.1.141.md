# Tool Aliasing and Dispatch in Claude Code 2.1.141

2.1.141 has tool-name aliases, legacy permission-rule normalization, SDK
compatibility naming, and MCP/builtin de-duplication. There is no broad
parameter-alias system comparable to older notes; the important aliasing layer
is tool identity.

## Primary Lookup

`source/src/Tool.ts` defines:

- `toolMatchesName(tool, name)`
- `findToolByName(tools, name)`

Lookup succeeds when `name` equals the tool primary name or any value in the
tool's `aliases` array.

## Tool Aliases Observed

2.1.141 aliases:

- `Task` is an alias for `Agent`.
- `KillShell` is an alias for `TaskStop`.
- `AgentOutputTool` is an alias for `TaskOutput`.
- `BashOutputTool` is an alias for `TaskOutput`.
- `Brief` is an alias for `SendUserMessage`.

These aliases support older configs, SDK consumers, and docs.

## Permission Normalization

`source/src/utils/permissions/permissionRuleParser.ts` normalizes legacy names:

- `Task` -> `Agent`
- `KillShell` -> `TaskStop`
- `AgentOutputTool` -> `TaskOutput`
- `BashOutputTool` -> `TaskOutput`
- `Brief` -> `SendUserMessage` in brief/Kairos contexts.

This matters for allow/deny rules, `--allowedTools`, `--disallowedTools`, and
settings-defined permission rules.

## SDK Compatibility Name

`source/src/utils/messages/systemInit.ts` maps the canonical `Agent` name back
to legacy `Task` in SDK-facing system init/tool list contexts. This is a
compatibility layer; it does not mean the internal canonical tool name changed
back to `Task`.

## Dispatch Pipeline

Tool execution paths use `findToolByName()`:

- `source/src/services/tools/toolOrchestration.ts`
- `source/src/services/tools/toolExecution.ts`
- `source/src/services/tools/StreamingToolExecutor.ts`
- `source/src/tools/ToolSearchTool/ToolSearchTool.ts`

This means aliases can resolve during execution, not only at display time.

## Builtins vs MCP Tools

`assembleToolPool()` merges built-ins and MCP tools. Built-ins win conflicts.
The merged tool list is sorted/stabilized where possible to preserve prompt
cache behavior.

Practical consequence:

- An MCP server should not rely on shadowing a built-in tool name.
- Built-in names and aliases should be treated as reserved.
- MCP server-prefix permission rules can still blanket-deny server tools.

## SDK MCP No Prefix

The environment variable `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` appears in 2.1.141 and
is related to SDK MCP naming behavior. Use it carefully: removing prefixes can
increase collision risk with built-ins and other MCP tools.

## Matcher Guidance

For new 2.1.141 configs:

- use `Agent`, not `Task`.
- use `TaskStop`, not `KillShell`.
- use `TaskOutput`, not `AgentOutputTool` or `BashOutputTool`.
- use `SendUserMessage`, not `Brief`.

Legacy names remain useful when a config must run against old and new versions.

## 2.1.141 Source Index

- `source/src/Tool.ts`
- `source/src/tools/AgentTool/constants.ts`
- `source/src/tools/AgentTool/AgentTool.tsx`
- `source/src/tools/TaskStopTool/TaskStopTool.ts`
- `source/src/tools/TaskOutputTool/TaskOutputTool.tsx`
- `source/src/tools/BriefTool/BriefTool.ts`
- `source/src/utils/permissions/permissionRuleParser.ts`
- `source/src/utils/messages/systemInit.ts`
- `source/src/services/tools/toolExecution.ts`
- `source/src/tools.ts`
