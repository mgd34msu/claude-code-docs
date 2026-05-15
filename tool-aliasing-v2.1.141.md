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

## Alias Resolution Algorithm

The resolver is intentionally simple:

```ts
tool.name === requestedName || tool.aliases?.includes(requestedName)
```

There is no fuzzy matching, case folding, or pattern matching in
`toolMatchesName()`. Any compatibility name must be explicitly listed as an
alias or normalized before lookup.

## Alias Layers

2.1.141 has four separate compatibility layers:

1. Tool object aliases (`aliases` on the tool definition).
2. Permission-rule legacy normalization.
3. SDK/system-init compatibility naming.
4. MCP/builtin name de-duplication.

These solve different problems and should not be conflated.

## Tool Object Aliases

Tool aliases allow runtime lookup by an old name.

Observed:

- `Agent.aliases = ['Task']`
- `TaskStop.aliases = ['KillShell']`
- `TaskOutput.aliases = ['AgentOutputTool', 'BashOutputTool']`
- `SendUserMessage.aliases = ['Brief']`

If a model/tool call emits the legacy name, `findToolByName()` can still resolve
the intended tool.

## Permission Normalization

Permission parser normalization rewrites legacy names before rule matching:

- `Task` -> `Agent`
- `KillShell` -> `TaskStop`
- `AgentOutputTool` -> `TaskOutput`
- `BashOutputTool` -> `TaskOutput`
- `Brief` -> `SendUserMessage` where brief support is compiled in.

This affects:

- settings permission rules.
- `--allowedTools`.
- `--disallowedTools`.
- base tools.
- dangerous-rule stripping in auto mode.

## SDK Compatibility

`sdkCompatToolName()` maps `Agent` to `Task` for SDK-facing system init/tool
lists. That is a wire compatibility decision, not the internal canonical name.

Consequence:

- internal docs/configs should say `Agent`.
- SDK consumers may still observe `Task`.
- hook configs for 2.1.141 should prefer `Agent`.

## Parameter Aliasing Status

The old parameter-aliasing topic should not be treated as a major 2.1.141
feature. The reconstructed source does not show a broad generic
`inputParamAliases` mechanism used across native tools.

There are still tool-specific normalization paths, for example:

- `ExitPlanMode` SDK-facing input normalization.
- provider/API input shaping.
- MCP schema adaptation.
- tool-specific validation.

But these are not a general "alias arbitrary input parameter names" facility.

## MCP Name Collision Behavior

MCP tools enter the pool after built-ins. Built-ins win name conflicts.

Collision rules:

- built-in tool name beats MCP tool name.
- MCP server-prefix deny rules can hide server tools before prompt exposure.
- no-prefix SDK MCP mode increases collision risk.
- tool search/deferred schemas do not change canonical identity rules.

Practical advice:

- Do not name MCP tools `Agent`, `Bash`, `Read`, etc.
- Keep MCP prefixes unless there is a strong SDK compatibility reason.
- Treat native tool names and aliases as reserved.

## Matcher Examples

Recommended:

```json
{ "matcher": "Agent" }
```

Legacy-compatible:

```json
{ "matcher": "Task" }
```

Recommended permission rule:

```json
{ "allow": ["Agent(researcher)"] }
```

Legacy permission rule that normalizes:

```json
{ "allow": ["Task(researcher)"] }
```

For future versions, prefer canonical names because aliases can eventually be
removed while canonical names are much more stable.

## Dispatch Sites Using Alias Lookup

Alias-aware lookup appears in:

- ordinary tool orchestration.
- streaming tool executor.
- direct tool execution.
- tool search.
- Agent UI/progress rendering.
- plan mode checks that need to know if `Agent` is present.

This is why aliases affect more than one code path.

## Complete Alias Table

Concrete 2.1.141 tool aliases found in source:

- canonical `Agent`, alias `Task`.
- canonical `TaskStop`, alias `KillShell`.
- canonical `TaskOutput`, aliases `AgentOutputTool` and `BashOutputTool`.
- canonical `SendUserMessage`, alias `Brief`.

Other alias systems exist in the codebase, but they are not native tool aliases:

- slash command aliases such as `plugin`/`plugins`.
- shell aliases captured in shell snapshots.
- model aliases such as `sonnet` and `opus`.
- keybinding modifier aliases such as `ctrl`/`control`.
- skill aliases such as `dream`/`learn`.
- CLI command aliases such as `update`/`upgrade`.

Do not mix these categories. `toolMatchesName()` only applies to tool objects.

## Permission Parser Normalization

The permission rule parser has a legacy-name map. This is separate from
`toolMatchesName()` and exists so persisted permission rules can migrate:

- `KillShell` normalizes to `TaskStop`.
- `AgentOutputTool` normalizes to `TaskOutput`.
- `BashOutputTool` normalizes to `TaskOutput`.
- `Task` normalizes to `Agent` through the agent alias path.
- `Brief` normalizes to `SendUserMessage` when brief tooling is compiled.

Normalization is applied when loading settings and when comparing rules. That
means a settings file can contain an old name while UI/debug output shows the
canonical name.

## Deprecated Tool Execution Fallback

`toolExecution.ts` contains a specific fallback for deprecated alias calls:

1. runtime cannot find a tool by primary name.
2. it searches for a tool whose `aliases` include the requested name.
3. it only uses the fallback if the requested name was an alias, not a primary
   name miss.

This exists for old transcripts and old SDK callers. It is deliberately narrow
so typoed tool names do not silently map to unrelated tools.

## Alias Effects by Layer

Model-visible tool schema:

- new sessions should expose canonical names.
- aliases are not necessarily shown as separate tools.
- aliases do not duplicate the schema.

Permission rules:

- legacy names normalize.
- canonical names are preferred for new rules.
- blanket deny filtering must account for aliases and MCP prefixes.

Hooks:

- tool matchers should use canonical names for new configs.
- legacy `Task` matchers can still work because `Agent` has an alias.
- `PreToolUse`, `PostToolUse`, and failure events all depend on tool-name
  matching.

SDK/print:

- SDK compatibility can emit or accept legacy names in some paths.
- `systemInit` maps `Agent` to legacy `Task` for older SDK compatibility.
- new code should treat `Agent` as canonical.

UI:

- progress rendering and agent displays use alias-aware matching where they
  need to detect an available tool.
- display labels are still controlled by tool/UI code, not by alias strings.

## Alias Non-Effects

Aliases do not:

- rename input fields.
- rename output fields.
- change permissions for a tool once resolved.
- let an MCP tool shadow a built-in.
- create a second tool schema.
- imply slash command aliases.
- imply shell aliases.

For example, `TaskOutput` accepting `BashOutputTool` as an alias does not mean
every parameter from an older Bash-output schema is accepted. Parameter
compatibility is implemented separately where source explicitly normalizes it.

## Source Audit Procedure

For a later release:

1. Search `source/src/tools` for `aliases:`.
2. Search `Tool.ts` for alias-aware helpers.
3. Search permission parser files for legacy-name maps.
4. Search SDK schema/init files for compatibility names.
5. Search tool execution for alias fallback logic.
6. Search hooks for matcher behavior.
7. Search UI and attachment code for `toolMatchesName`.
8. Separate tool aliases from command/model/skill/keybinding aliases.
9. Verify canonical names in docs and examples.
10. Keep legacy names only in migration/compatibility sections.

## Deep 2.1.141 Alias Mechanics Addendum

Tool aliasing in 2.1.141 is explicit and narrow. A name is an alias only when a
tool declares `aliases` or a parser deliberately normalizes a legacy name.

### Tool Interface

`Tool.ts` defines `aliases?: string[]` and two alias-aware helpers:

- `toolMatchesName(tool, name)` returns true for primary name or alias.
- `findToolByName(tools, name)` uses `toolMatchesName`.

The comment is precise: aliases are for backwards compatibility when a tool is
renamed. They are not synonyms, shell aliases, or fuzzy names.

### Execution Fallback

`services/tools/toolExecution.ts` first searches the available tools presented
to the model. If no tool is found, it searches `getAllBaseTools()` and only
falls back when the found tool includes the requested name in `aliases`. This
supports old transcripts that call a deprecated name without allowing arbitrary
hidden tools to execute by primary name.

Example from source comments:

- old `KillShell` calls can route to `TaskStop`.

### Permission And Hook Normalization

Permission and hook code normalizes legacy names:

- `utils/permissions/permissionsLoader.ts` round-trips raw settings entries so legacy names match canonical forms.
- `utils/hooks.ts` normalizes matcher names and tests regex matchers against legacy names.
- `utils/permissions/yoloClassifier.ts` builds alias maps so classifier input can understand tool aliases.

This means a rule or hook written against an old name may still match a current
tool. It does not mean docs should recommend old names for new configs.

### Known Source-Backed Alias Families

The source scan shows these important alias categories:

- `TaskStop` has alias `KillShell`.
- message-queue functions keep backward-compatible notification aliases such as `subscribeToPendingNotifications`.
- Brief mode exposes the primary communication tool through `SendUserMessage`; treat `Brief` as a legacy reference unless the current source exposes it.
- `TaskOutput` compatibility appears in docs/source as Bash-output migration language and should be verified per release.
- skill aliases exist separately, for example `/dream` has alias `/learn`.
- command aliases exist separately, for example `--rc` is an alias for `--remote-control`.
- model aliases are separate frontmatter/model-resolver behavior.

Only the first category is tool aliasing in the strict `Tool.aliases` sense.

### Alias Scope Rules

Apply these rules in docs and future maps:

- Tool alias: model/API tool-use name compatibility.
- Slash command alias: user command invocation compatibility.
- CLI flag alias: commander option compatibility.
- Model alias: model resolver compatibility.
- Shell alias: user shell environment behavior.
- Permission legacy syntax: settings parser compatibility.
- SDK schema compatibility: generated type/schema compatibility.

Do not mix these. A shell alias named `rg` is not a Claude tool alias. A slash
skill alias `/learn` is not a model tool name. A permission legacy name can
match a tool without being shown as a current tool name.

### Future Extraction Guidance

When mapping a later minified release, preserve the primary canonical name as
the file/function name. Put legacy names in migration sections only. If a
function accepts both primary and legacy names, name it around normalization or
compatibility, not around one legacy alias.
