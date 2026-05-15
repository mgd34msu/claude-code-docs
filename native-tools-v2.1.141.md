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

## Detailed Tool Registry Reconstruction

`getAllBaseTools()` in `source/src/tools.ts` is the registry root. It does not
represent the final model-visible list by itself; it is the full candidate set
after build-time dead code elimination, process environment checks, and lazy
requires. The final list is produced by `getTools(permissionContext)` and then
combined with MCP tools by `assembleToolPool()`.

Tool list construction, in order:

1. `Agent`
2. `TaskOutput`
3. `Bash`
4. `Glob` and `Grep` only when embedded search tools are not present
5. `ExitPlanMode`
6. `Read`
7. `Edit`
8. `Write`
9. `NotebookEdit`
10. `WebFetch`
11. `TodoWrite`
12. `WebSearch`
13. `TaskStop`
14. `AskUserQuestion`
15. `Skill`
16. `EnterPlanMode`
17. `Config` for `USER_TYPE=ant`
18. optional browser/computer tools when present in the build
19. todo-v2 task tools when `isTodoV2Enabled()`
20. `CtxInspect` when `feature('CONTEXT_COLLAPSE')`
21. terminal capture when present in the build
22. `LSP` when `ENABLE_LSP_TOOL` is truthy
23. `EnterWorktree` and `ExitWorktree` when worktree mode is enabled
24. `SendMessage`
25. `TeamCreate` and `TeamDelete` when swarms are enabled
26. optional verify-plan tooling when present
27. `REPL` for Anthropic internal users when REPL mode is available
28. optional workflow tool when present
29. `CronCreate`, `CronDelete`, `CronList` when `feature('AGENT_TRIGGERS')`
30. `RemoteTrigger` when `feature('AGENT_TRIGGERS_REMOTE')`
31. `Monitor` when `feature('MONITOR_TOOL')`
32. `SendUserMessage`
33. `PushNotification` when Kairos push notifications are compiled in
34. optional subscribe PR tool when present
35. `PowerShell` when `isPowerShellToolEnabled()`
36. `Snip` when `feature('HISTORY_SNIP')`
37. test-only permission tool in `NODE_ENV=test`
38. `ListMcpResourcesTool`
39. `ReadMcpResourceTool`
40. `ToolSearch` when optimistic tool search says it may be enabled

## Final Visibility Filters

`getTools(permissionContext)` applies additional filters:

- Simple mode (`CLAUDE_CODE_SIMPLE`) collapses the list to shell/read/edit, or
  to REPL wrapper tools when REPL mode is active.
- Coordinator mode can add `Agent`, `TaskStop`, and `SendMessage` to the simple
  set for coordinator/worker split behavior.
- `filterToolsByDenyRules()` hides tools blanket-denied by permission context
  before the model sees them.
- Each tool's `isEnabled()` is evaluated before final exposure.
- REPL-only primitives are hidden when the REPL tool is the exposed wrapper.
- Special tools like MCP resource tools and structured output are filtered out
  of some model-facing lists even though they exist in the runtime pool.

## Tool Interface Fields

The `Tool` type in `source/src/Tool.ts` is richer than a simple name/schema
pair. Fields observed in 2.1.141 include:

- `name`: canonical tool name.
- `aliases`: legacy lookup names accepted by `toolMatchesName`.
- `searchHint`: short natural-language phrase used by `ToolSearch`.
- `description`: model-facing capability text.
- `prompt`: longer instruction text, often dynamic.
- `inputSchema`: zod schema for tool input.
- `outputSchema`: optional zod schema for structured output.
- `isEnabled`: runtime availability predicate.
- `shouldDefer`: whether the tool is eligible for deferred loading/tool search.
- `alwaysLoad`: opt-out from deferral.
- `isReadOnly`: permission and concurrency metadata.
- `isConcurrencySafe`: whether concurrent execution is safe.
- `toAutoClassifierInput`: reduced text for auto permission classification.
- `validateInput`: custom validation before execution.
- `mapToolResultToToolResultBlockParam`: conversion to tool result content.
- `renderToolUseMessage` and `renderToolResultMessage`: UI rendering hooks.
- `userFacingName`: display string for tool UI.

## Built-In Tool Catalog

### Agent

- Canonical name: `Agent`
- Legacy alias: `Task`
- Source: `source/src/tools/AgentTool/AgentTool.tsx`
- Search hint: delegate work to a subagent
- Max result size: 100,000 chars
- Supports sync, async/background, fork subagent, and teammate spawn paths.
- Team spawning occurs when a team name is resolved and `name` is provided.
- Teammates cannot spawn other teammates.
- In-process teammates cannot spawn background agents.
- Background agents register foreground/background task state and can be
  surfaced through task output and Agent View.

### TaskOutput

- Canonical name: `TaskOutput`
- Aliases: `AgentOutputTool`, `BashOutputTool`
- Reads output for background tasks and agents.
- Used by background agent workflows, SDK task output, and historical alias
  compatibility.

### TaskStop

- Canonical name: `TaskStop`
- Alias: `KillShell`
- Stops running tasks through the shared task stop path.
- Applies to background shell and agent tasks that are still running.

### Bash

- Primary shell tool for POSIX-style command execution.
- Participates in sandbox checks, command-injection detection, backgrounding,
  progress reporting, and shell snapshot behavior.
- Emits bash-specific telemetry such as command execution, backgrounding, git
  index-lock, shell snapshot, and tree-sitter parser events.

### PowerShell

- Windows-oriented shell tool.
- Included only when `isPowerShellToolEnabled()` returns true.
- External users generally need `CLAUDE_CODE_USE_POWERSHELL_TOOL=true`.
- Permission stripping in auto mode treats broad PowerShell primitives as
  dangerous.

### Read, Write, Edit

- File primitives for reading, writing, and patching files.
- `Read` includes file-size/page/PDF handling and dedup telemetry.
- `Write` and `Edit` log CLAUDE.md writes and diff computation telemetry.
- Write/edit are affected by workspace trust, permissions, auto mode, and
  speculation overlay behavior.

### Glob and Grep

- Omitted when `hasEmbeddedSearchTools()` detects embedded native search tools.
- Otherwise included as dedicated tools.
- Environment variables such as `CLAUDE_CODE_GLOB_HIDDEN`,
  `CLAUDE_CODE_GLOB_NO_IGNORE`, and `CLAUDE_CODE_GLOB_TIMEOUT_SECONDS` affect
  glob behavior.

### NotebookEdit

- Notebook-specific edit tool.
- Included in the core candidate set.
- Has separate path fields and write semantics from `Edit`/`Write`.

### WebFetch and WebSearch

- Web-facing tools with server-tool accounting.
- Usage can increment `server_tool_use.web_search_requests` and
  `server_tool_use.web_fetch_requests`.
- WebFetch logs host and HTTP error telemetry.

### TodoWrite

- Legacy/simple todo write tool.
- Separate from todo-v2 `TaskCreate`/`TaskUpdate`/`TaskList`/`TaskGet`.

### TaskCreate, TaskGet, TaskUpdate, TaskList

- Included when `isTodoV2Enabled()`.
- Back the team task-list workflow.
- `TaskUpdate` supports status changes, owner changes, metadata merge/delete,
  and dependency changes with `addBlocks` and `addBlockedBy`.
- `TaskCompleted` hooks can fire from task update behavior.

### AskUserQuestion

- User elicitation/planning tool.
- Connected to elicitation schemas and hooks.
- Interactive behavior depends on session/UI mode.

### EnterPlanMode and ExitPlanMode

- Plan-mode transition tools.
- `ExitPlanMode` has SDK-facing normalization for plan content and plan file
  paths.
- Plan mode interacts with auto mode and teammate permission modes.

### Skill

- Invokes bundled/user/project/plugin skills.
- Skill loading integrates with dynamic skill discovery and tool-search style
  discovery.
- Emits skill invocation telemetry.

### ToolSearch

- Deferred tool discovery tool.
- Uses `searchHint`, deferred tool metadata, and keyword matching to expose
  relevant tools when the full schema set is not sent initially.
- Logs `tengu_tool_search_outcome`.

### MCP Resource Tools

- `ListMcpResourcesTool`
- `ReadMcpResourceTool`
- Built-in tools for MCP resources, distinct from server-provided MCP tools.

### Cron Tools

- `CronCreate`
- `CronDelete`
- `CronList`
- Build-gated by `AGENT_TRIGGERS`.
- Runtime-gated by cron helpers and `CLAUDE_CODE_DISABLE_CRON`.
- Detailed in `loop-command-v2.1.141.md`.

### Team Tools

- `TeamCreate`
- `TeamDelete`
- `SendMessage`
- Team creation/deletion are gated by agent swarms.
- `SendMessage` is registered as a base tool but meaningful only when a team or
  addressable agent context exists.
- Detailed in `agent-teams-v2.1.141.md`.

### Worktree Tools

- `EnterWorktree`
- `ExitWorktree`
- Included when worktree mode is enabled.
- Emit worktree create/enter/keep/remove telemetry.

### Internal/Conditional Tools

- `Config`: Anthropic internal user type only.
- `REPL`: Anthropic internal REPL paths.
- `CtxInspect`: context collapse feature builds.
- `RemoteTrigger`: remote scheduled-agent trigger paths.
- `Monitor`: monitor MCP task paths.
- `PushNotification`: Kairos push notification builds.
- `Snip`: history snip builds.
- `TestingPermissionTool`: test environment only.

## Dispatch Flow

The runtime path is consistent across stream and non-stream execution:

1. The model emits a tool use block with a name.
2. `findToolByName()` looks up the tool by canonical name or alias.
3. Permission context evaluates allow/deny/ask behavior.
4. Hooks such as `PreToolUse` can inspect, deny, or update input.
5. Tool-specific `validateInput()` runs when present.
6. The tool `call()` executes.
7. Tool results are mapped to Anthropic tool result blocks.
8. `PostToolUse` or `PostToolUseFailure` hooks run.
9. UI rendering and telemetry are emitted.

`StreamingToolExecutor` follows the same name-resolution primitive, so alias
behavior is not limited to the non-streaming path.

## Permission and Alias Interactions

Permission rules normalize legacy names before matching. This matters because
the visible tool may be named `Agent`, while older configs or SDK compatibility
paths may still say `Task`.

Canonical names to prefer in 2.1.141:

- `Agent`, not `Task`.
- `TaskStop`, not `KillShell`.
- `TaskOutput`, not `AgentOutputTool` or `BashOutputTool`.
- `SendUserMessage`, not `Brief`.

## MCP Shadowing Rules

`assembleToolPool()` merges built-ins and MCP tools. Built-ins have priority.
Practical consequences:

- MCP tools should not rely on using built-in names.
- Server-prefixed MCP tool names are still subject to permission deny rules.
- MCP tools are sorted/stabilized to preserve prompt cache behavior.
- MCP resources are exposed through built-in resource tools rather than being
  treated as ordinary MCP tool invocations.

## Tool Telemetry Reference

Representative tool event families in 2.1.141:

- `tengu_tool_use_success`
- `tengu_tool_use_error`
- `tengu_tool_use_cancelled`
- `tengu_tool_use_progress`
- `tengu_tool_use_can_use_tool_allowed`
- `tengu_tool_use_can_use_tool_rejected`
- `tengu_deferred_tool_schema_not_sent`
- `tengu_bash_tool_command_executed`
- `tengu_bash_command_timeout_backgrounded`
- `tengu_bash_command_assistant_auto_backgrounded`
- `tengu_bash_command_explicitly_backgrounded`
- `tengu_file_read_limits_override`
- `tengu_file_read_dedup`
- `tengu_pdf_page_extraction`
- `tengu_write_claudemd`
- `tengu_edit_string_lengths`
- `tengu_tool_use_diff_computed`
- `tengu_web_fetch_http_error`
- `tengu_web_fetch_host`
- `tengu_skill_tool_invocation`
- `tengu_skill_descriptions_truncated`
- `tengu_tool_search_outcome`
- `tengu_worktree_created`
- `tengu_worktree_entered_existing`
- `tengu_worktree_kept`
- `tengu_worktree_removed`

## Deep Source Inventory

The 2.1.141 tool tree under `source/src/tools` is the concrete source of truth
for the built-in tool surface. The important reconstruction point is that the
file list is larger than the default model-visible tool list. Several tools are
compiled in but hidden by runtime gates, by feature flags, by `USER_TYPE`, by
simple/bare mode, by deny rules, or by optimistic tool-search deferral.

Tool implementation files present in 2.1.141:

- `AgentTool/AgentTool.tsx`: canonical subagent launcher. The compatibility
  alias is `Task`.
- `AskUserQuestionTool/AskUserQuestionTool.tsx`: interactive question tool.
- `BashTool/BashTool.tsx`: shell execution tool and the center of shell
  permission classification.
- `BriefTool/BriefTool.ts`: canonical `SendUserMessage` tool for brief mode.
- `ConfigTool/ConfigTool.ts`: first-party config write/read surface, ant-only
  in the base registry.
- `CtxInspectTool/CtxInspectTool.ts`: context inspection/debug surface when
  compiled and enabled.
- `EnterPlanModeTool/EnterPlanModeTool.ts`: legacy plan-entry helper.
- `EnterWorktreeTool/EnterWorktreeTool.ts`: worktree mode entry tool.
- `ExitPlanModeTool/ExitPlanModeV2Tool.ts`: plan exit and plan approval tool.
- `ExitWorktreeTool/ExitWorktreeTool.ts`: worktree mode exit tool.
- `FileEditTool/FileEditTool.ts`: targeted text edit tool.
- `FileReadTool/FileReadTool.ts`: file and notebook read tool.
- `FileWriteTool/FileWriteTool.ts`: full file write tool.
- `GlobTool/GlobTool.ts`: path discovery tool, omitted when embedded search
  tools replace the need for dedicated glob/grep.
- `GrepTool/GrepTool.ts`: text search tool, also omitted when embedded search
  tools are available.
- `LSPTool/LSPTool.ts`: language-server tool behind `ENABLE_LSP_TOOL`.
- `ListMcpResourcesTool/ListMcpResourcesTool.ts`: MCP resource listing tool,
  treated as a special tool rather than a normal default tool.
- `MCPTool/MCPTool.ts`: MCP tool wrapper implementation.
- `McpAuthTool/McpAuthTool.ts`: MCP authentication helper.
- `MonitorTool/MonitorTool.ts`: monitor/background observation tool when
  compiled into the build.
- `NotebookEditTool/NotebookEditTool.ts`: notebook mutation tool.
- `PowerShellTool/PowerShellTool.tsx`: Windows/PowerShell execution tool when
  the factory returns a tool.
- `PushNotificationTool/PushNotificationTool.ts`: push/notification helper when
  available.
- `REPLTool/REPLTool.ts`: REPL VM wrapper; hides primitive tools when enabled.
- `ReadMcpResourceTool/ReadMcpResourceTool.ts`: MCP resource read tool, special
  rather than default.
- `RemoteTriggerTool/RemoteTriggerTool.ts`: remote/session trigger tool when
  compiled.
- `ScheduleCronTool/CronCreateTool.ts`: scheduled task creation.
- `ScheduleCronTool/CronDeleteTool.ts`: scheduled task deletion.
- `ScheduleCronTool/CronListTool.ts`: scheduled task listing.
- `SendMessageTool/SendMessageTool.ts`: teammate/agent messaging tool.
- `SkillTool/SkillTool.ts`: dynamic and bundled skill invocation surface.
- `SnipTool/SnipTool.ts`: terminal snip/capture helper when compiled.
- `SyntheticOutputTool/SyntheticOutputTool.ts`: synthetic output channel, a
  special tool and not exposed as a normal default tool.
- `TaskCreateTool/TaskCreateTool.ts`: task-list item creation.
- `TaskGetTool/TaskGetTool.ts`: task-list item retrieval.
- `TaskListTool/TaskListTool.ts`: task-list enumeration.
- `TaskOutputTool/TaskOutputTool.tsx`: task output reader. Compatibility
  aliases include `AgentOutputTool` and `BashOutputTool`.
- `TaskStopTool/TaskStopTool.ts`: task stop/kill tool. Compatibility alias is
  `KillShell`.
- `TaskUpdateTool/TaskUpdateTool.ts`: task-list item mutation.
- `TeamCreateTool/TeamCreateTool.ts`: team definition creation.
- `TeamDeleteTool/TeamDeleteTool.ts`: team definition removal.
- `TodoWriteTool/TodoWriteTool.ts`: todo-list mutation.
- `ToolSearchTool/ToolSearchTool.ts`: deferred tool discovery surface.
- `WebFetchTool/WebFetchTool.ts`: URL fetch/read tool.
- `WebSearchTool/WebSearchTool.ts`: search tool.
- `testing/TestingPermissionTool.tsx`: test-only permission fixture.

That inventory should not be treated as "all visible tools". It is the set of
candidate implementations. The visible set is the result of registry assembly,
mode filtering, deny-rule filtering, MCP merge, and per-request API deferral.

## Registry Order Audit

The base registry in `tools.ts` is intentionally ordered before later sorting.
The order matters because some consumers use the raw base list and because
comments around global system caching explicitly say the list must stay aligned
with the server-side cache configuration.

The base list begins with core conversation and editing tools:

- `Agent`
- `TaskOutput`
- `Bash`
- `Glob` and `Grep` when embedded search tools are not present
- `ExitPlanMode`
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
- `Plan`

Conditional blocks then add first-party, experimental, or environment-specific
tools:

- `Config` only for `USER_TYPE=ant`.
- Browser/web tool when the build exports it.
- task-list tools only when todo-v2 is enabled.
- overflow/context/terminal capture test or debug tools when present.
- `LSP` only with `ENABLE_LSP_TOOL`.
- worktree enter/exit tools only when worktree mode is enabled.
- `SendMessage` through the tool factory.
- team create/delete only when agent swarms are enabled.
- plan verification, REPL, workflow, cron, remote trigger, monitor, brief, push,
  PR subscribe, PowerShell, snip, test-only permission, MCP resource tools, and
  tool search as separate conditional additions.

The registry then removes special tools from the normal default tool list:

- `ListMcpResources`
- `ReadMcpResource`
- `SyntheticOutput`

Those are still real tools, but they are inserted through specific flows rather
than being ordinary model-visible defaults.

## Visibility Algorithm

The effective 2.1.141 visibility algorithm is:

1. If `CLAUDE_CODE_SIMPLE` is truthy, return a reduced tool set.
2. If simple mode and REPL mode are both enabled, return `REPL` rather than raw
   primitives.
3. If simple mode and coordinator mode are active, add coordinator-facing
   `Agent`, `TaskStop`, and `SendMessage` as needed.
4. Otherwise build the full candidate list with `getAllBaseTools()`.
5. Remove special tools that should not be in the ordinary default list.
6. Apply blanket deny-rule filtering before the model sees tools.
7. If REPL mode is active, hide primitive tools once `REPL` is available.
8. Call each tool's `isEnabled()` and remove disabled tools.
9. Merge MCP tools later via `assembleToolPool()`.
10. Sort built-ins and MCP tools in separate partitions for prompt-cache
    stability, then deduplicate by name with built-ins winning collisions.

This means a reconstruction that only scans `source/src/tools` will overcount
visible tools, while a reconstruction that only scans default prompts will miss
compiled but gated tools.

## Simple/Bare Mode Tool Set

Simple mode is set by `--bare` and by code paths that set
`CLAUDE_CODE_SIMPLE=1`. Its default raw primitive set is intentionally tiny:

- `Bash`
- `Read`
- `Edit`

When REPL is enabled, simple mode returns `REPL` instead of those primitives.
That is not cosmetic. The REPL tool wraps shell/read/edit behavior inside the VM
context and prevents the model from calling the primitives directly.

Coordinator mode is an exception to the tiny-set rule. The coordinator needs to
manage workers, so simple mode can include:

- `Agent`
- `TaskStop`
- `SendMessage`

Worker filtering then narrows what each worker can actually call.

## Agent and Teammate Tool Policy

`constants/tools.ts` defines three tool sets that matter for subagent and team
behavior:

- `ALL_AGENT_DISALLOWED_TOOLS` blocks tools that ordinary agents must not call.
- `ASYNC_AGENT_ALLOWED_TOOLS` is the explicit allowlist for async agents.
- `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` adds task-list, messaging, and optionally
  cron tools for teammates.

Important 2.1.141 details:

- Ordinary async agents get file read/search/edit/write, notebook edit, web
  search/fetch, todo, shell tools, skills, synthetic output, tool search, and
  worktree enter/exit.
- Async agents do not get recursive agent/task-output/task-stop/plan-mode
  primitives.
- In-process teammates can additionally use task-list and messaging tools.
- When `AGENT_TRIGGERS` is compiled/enabled, in-process teammates can use the
  cron tools so scheduled work can route back to their pending message queues.
- `USER_TYPE=ant` is special for nested agent permission: the source comments
  explicitly allow nested `Agent` for ant users while non-ant users keep it in
  the disallowed set.

This is one of the strongest examples of why "native tools" must be documented
by runtime context, not just by implementation filename.

## Permission Filtering Details

Blanket deny rules are applied before model exposure. A rule with a tool name
and no rule content removes that tool from the visible list. The filter uses the
same matching model as runtime permission checks, including MCP server-prefix
behavior. Consequences:

- A blanket deny for a built-in canonical tool removes the built-in before the
  model sees it.
- Legacy aliases normalize through the permission parser for renamed tools.
- MCP server-prefix deny rules can strip all tools from a server before call
  time.
- Runtime permission checks still run even if a tool is visible.
- Visibility filtering is not a substitute for per-call validation.

The important distinction is "not exposed" versus "exposed but blocked at
execution". 2.1.141 does both, depending on the rule and call path.

## MCP Merge and Collision Rules

MCP tools are merged with built-ins in `assembleToolPool()`:

- built-ins are produced first through `getTools()`.
- MCP tools are separately deny-filtered.
- built-ins are sorted by name as one partition.
- allowed MCP tools are sorted by name as a second partition.
- the partitions are concatenated.
- `uniqBy(..., 'name')` keeps the first instance of a name.

Because built-ins are first, a built-in wins when an MCP tool has the same tool
name. The partitioned sort is specifically for prompt-cache stability. A flat
sort could insert an MCP tool between built-ins and invalidate downstream cache
keys whenever MCP availability changes.

## Tool Search Interaction

`ToolSearch` is optimistic in the base registry. The actual decision to defer
tools is made later at API-request time. That distinction matters:

- `ToolSearch` can be present in the candidate list even when no tools are
  finally deferred.
- token counting and threshold calculations need the complete built-in plus MCP
  universe.
- deferred tools are appended/discovered, not treated as a completely separate
  registry.
- prompt-cache stability still depends on deterministic ordering and stable
  tool schemas.

When reconstructing a later version, check both the registry inclusion point and
the request-time deferral policy.

## Compatibility Aliases

The concrete tool aliases in 2.1.141 include:

- `Agent` has alias `Task`.
- `TaskStop` has alias `KillShell`.
- `TaskOutput` has aliases `AgentOutputTool` and `BashOutputTool`.
- `SendUserMessage` has alias `Brief`.

Aliases are not display labels. They are accepted by alias-aware lookup and by
permission normalization. New documentation should use canonical names, while
migration notes should still mention aliases because old transcripts and config
files may contain them.

## Source Reconstruction Checklist

For future releases, rebuild the native-tool surface in this order:

1. List all files under `source/src/tools`.
2. Read `tools.ts` for base registry inclusion, special-tool removal, simple
   mode behavior, REPL behavior, and MCP merge behavior.
3. Read `constants/tools.ts` for agent, async-agent, teammate, and coordinator
   allowed/disallowed sets.
4. Read `Tool.ts` for interface fields and alias lookup.
5. Read `toolExecution.ts`, `toolOrchestration.ts`, and `QueryEngine.ts` for
   execution paths.
6. Read `permissionRuleParser.ts`, `permissionsLoader.ts`, and
   `yoloClassifier.ts` for permission normalization and alias handling.
7. Read each conditional tool's `isEnabled()` and surrounding feature gates.
8. Reconcile native built-ins with MCP tool merge and tool-search deferral.
9. Verify simple/bare mode separately from default interactive mode.
10. Verify print/SDK mode separately from interactive mode.

## Per-Tool Behavior Notes

`Agent`:

- launches subagents.
- can run synchronously or asynchronously depending on input and context.
- has legacy alias `Task`.
- is blocked inside ordinary async agents to prevent recursion.
- is permitted for coordinator contexts.
- integrates with task state, Agent View, task notifications, and memory.

`TaskOutput`:

- reads output from background tasks.
- compatibility aliases cover older agent/bash output names.
- should be documented with background task semantics, not as a shell-only tool.
- interacts with Agent View and task transcript state.

`TaskStop`:

- stops running tasks.
- compatibility alias is `KillShell`.
- applies to more than shell background jobs.
- is disallowed inside ordinary async agents because it needs main task state.

`Bash`:

- executes shell commands.
- depends on shell snapshot behavior.
- uses static parsing, read-only validation, permission rules, and classifier
  paths.
- can background commands or be auto-backgrounded.
- has the broadest safety surface of any native tool.

`Read`:

- reads text, notebooks, images/PDFs through supported paths.
- has size and token-budget protections.
- participates in tool result storage and cache-stability logic.
- can be used by agents and async agents.

`Edit`:

- applies targeted string edits.
- emits string-length/diff telemetry.
- is a central write tool for both main and agent contexts.
- must be considered separately from `Write` because permission and diff
  behavior differ.

`Write`:

- writes full file contents.
- can create or replace files.
- is higher-risk than targeted edit for some permission policies.
- is used by memory/dream and agent contexts when allowed.

`NotebookEdit`:

- mutates notebook content.
- follows notebook-specific schema and rendering.
- should not be collapsed into ordinary text edit docs.

`Glob` and `Grep`:

- provide file discovery and text search when embedded search tools are absent.
- can disappear from the visible native set in ant-native builds with embedded
  search aliases.
- should be audited together with shell `find`/`grep` alias behavior.

`WebFetch`:

- fetches URL content.
- has host/error telemetry.
- can interact with network policy and prompt-injection risks.
- is separate from `WebSearch`.

`WebSearch`:

- performs search.
- contributes server-side tool usage counts when API usage reports them.
- should be accounted separately in token/cost docs.

`TodoWrite`:

- manages the conversation todo list.
- is visible in many modes and agents.
- should not be confused with task-list tools.

`TaskCreate`, `TaskGet`, `TaskList`, `TaskUpdate`:

- manage task-list work items.
- are gated by todo-v2 enablement.
- are especially important for teams/teammates.
- are not the same as background task process state.

`SendMessage`:

- sends messages to teammates/agents.
- is part of the team coordination surface.
- can be available to in-process teammates and coordinator contexts.

`TeamCreate` and `TeamDelete`:

- manage team definitions.
- are gated by agent swarms/team enablement.
- interact with durable team files and in-process teammate runtime.

`CronCreate`, `CronDelete`, `CronList`:

- implement the scheduling surface used by `/loop` and `/dream nightly`.
- are gated by `AGENT_TRIGGERS` and runtime cron gates.
- may be session-only or durable depending on durable gate.
- can route scheduled prompts to teammates when created by teammates.

`Skill`:

- invokes bundled or disk skills.
- has its own dynamic discovery and truncation behavior.
- can carry large procedural context without always loading full content.

`ToolSearch`:

- provides deferred tool discovery.
- is included optimistically in the base registry.
- final use depends on request-time thresholds and model/tool conditions.

`SendUserMessage`:

- brief-mode user-facing message tool.
- alias is `Brief`.
- changes rendering and prompt contract.
- should be documented with brief mode, not only native tools.

`AskUserQuestion`:

- interactive question mechanism.
- not suitable for all noninteractive/print contexts.
- can be affected by SDK/stream behavior.

`EnterPlanMode` and `ExitPlanMode`:

- implement planning transitions.
- `ExitPlanMode` is the approval boundary.
- plan behavior interacts with permission mode and auto mode.

`EnterWorktree` and `ExitWorktree`:

- manage worktree mode.
- worktree creation/removal emits telemetry.
- interacts with session state and git/worktree helpers.

`Config`:

- ant-only in the base registry.
- should be treated as internal unless the user context is explicitly ant.

`CtxInspect`:

- context inspection/debug tool.
- tied to context collapse/debug builds.
- useful for reconstruction but not ordinary public tool behavior.

`LSP`:

- enabled by `ENABLE_LSP_TOOL`.
- depends on language server availability and environment.

`REPL`:

- wraps primitive tool behavior in a VM-like context.
- hides raw primitives when enabled.
- changes simple-mode tool exposure.

`Monitor`, `RemoteTrigger`, `PushNotification`, `Snip`, `PowerShell`:

- compiled or enabled only in specific builds/environments.
- should be documented as conditional until the relevant build/gate proves
  default availability.

`ListMcpResources` and `ReadMcpResource`:

- special MCP resource tools.
- not ordinary default tools.
- excluded from the normal visible base list and inserted through MCP/resource
  flows.

`SyntheticOutput`:

- special internal output channel.
- not a normal user-facing tool.
- allowed in some agent/coordinator contexts for output plumbing.

## Behavior Test Matrix

A complete native-tool audit should test these contexts separately:

- default interactive session.
- `--print` text output.
- `--print --output-format=json`.
- `--print --output-format=stream-json`.
- `--bare`.
- `CLAUDE_CODE_SIMPLE=1`.
- REPL mode.
- coordinator mode.
- ordinary subagent.
- async/background subagent.
- in-process teammate.
- remote/CCR session.
- MCP tools present.
- MCP name collision with built-in.
- blanket deny rule for a built-in.
- blanket deny rule for an MCP server prefix.
- legacy alias call from an old transcript.

The expected tool set can differ in each row.
