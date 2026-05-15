# Context Injection Avenues in Claude Code 2.1.141

2.1.141 has many ways to add instructions or messages to the model context.
CLAUDE.md is only one layer. The safest way to reason about context is by
injection point, scope, and whether it affects parent sessions, subagents, or
external surfaces.

## System Prompt Construction

Base system prompt construction lives in `source/src/constants/prompts.ts`.
It conditionally includes product behavior sections, feature-gated instructions,
brief/Kairos messaging guidance, tool-use guidance, and environment-specific
instructions.

Feature and environment inputs include:

- simple/bare mode.
- brief mode.
- active tool set.
- model/provider decisions.
- feature flags and GrowthBook values.
- output style.

## CLAUDE.md Files

CLAUDE.md discovery is handled in `source/src/context.ts` and related context
helpers.

Controls:

- `CLAUDE_CODE_DISABLE_CLAUDE_MDS`
- `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD`
- simple/bare mode constraints.
- project trust and working-directory state.

CLAUDE.md is best for stable, project-wide or user-wide memory. It is not the
only way subagents get context.

## Agent Definitions

Agent definitions are loaded by `source/src/tools/AgentTool/loadAgentsDir.ts`.
They inject context through:

- agent description.
- agent system prompt.
- frontmatter tool restrictions.
- frontmatter hooks.
- frontmatter MCP server entries.
- frontmatter skills.
- frontmatter memory policy.

The execution path in `source/src/tools/AgentTool/runAgent.ts` builds the
subagent prompt and attaches additional contexts from hooks and agent config.

## Hook Additional Context

Hooks can inject runtime context using `additionalContext` on supported events.
The most important injection paths are:

- `UserPromptSubmit`
- `PreToolUse`
- `PostToolUse`
- `SubagentStart`
- `Stop`
- other hook events covered in `claude-hooks-reference-v2.1.141.md`.

Tool-hook aggregation happens in `source/src/services/tools/toolHooks.ts`.
Stop and prompt hook execution happens in `source/src/query/stopHooks.ts`.

## Output Styles

Output styles are loaded from:

- `source/src/outputStyles/loadOutputStylesDir.ts`
- plugin output-style loaders.

Frontmatter fields include:

- `name`
- `description`
- `keep-coding-instructions`
- `force-for-plugin`

Output styles modify response behavior, not the underlying tool registry.

## Skills

Skills are loaded from bundled, user, project, plugin, and policy paths. They
can inject instructions when selected, discovered, or invoked via `Skill`.

Important areas:

- `source/src/skills`
- `source/src/tools/SkillTool`
- `source/src/services/skillSearch`

Skills are more appropriate than CLAUDE.md for large procedural instructions
that should only enter context when relevant.

## MCP and Plugins

MCP servers add context through:

- tool schemas.
- server-provided prompts/resources.
- plugin scopes and output styles.
- channel notifications.
- elicitation and permission flows.

Channel notifications are a special inbound prompt surface and are documented in
`channels-v2.1.141.md`.

## Teams and Teammates

Agent teams add context through:

- team metadata.
- teammate identity.
- team task lists.
- teammate inbox messages.
- plan approval messages.
- viewed teammate transcript state.

Team behavior is documented in `agent-teams-v2.1.141.md` and Agent View in
`agent-view-2.1.141.md`.

## Print Mode and SDK Mode

`-p` / `--print` changes runtime shape: no normal interactive UI, but tools,
hooks, cron scheduler, SDK transport, and telemetry can still run depending on
options. See `claude-print-v2.1.141.md`.

## Priority Guidance

Use the smallest context surface that matches the need:

- CLAUDE.md for stable project memory.
- agent definitions for subagent-specific behavior.
- hooks for runtime-computed information.
- skills for large reusable procedures.
- output styles for response style.
- MCP for external systems.
- channels only for trusted inbound external messages.

## 2.1.141 Source Index

- `source/src/constants/prompts.ts`
- `source/src/context.ts`
- `source/src/tools/AgentTool/loadAgentsDir.ts`
- `source/src/tools/AgentTool/runAgent.ts`
- `source/src/services/tools/toolHooks.ts`
- `source/src/query/stopHooks.ts`
- `source/src/outputStyles`
- `source/src/skills`
- `source/src/services/mcp`
- `source/src/bootstrap/state.ts`

## Injection Surface Matrix

2.1.141 context surfaces can be grouped by when they enter the conversation:

- Startup/system prompt: base system prompt, product mode, output style,
  environment, tools, and feature-gated instructions.
- Project context load: CLAUDE.md, additional directories, memory directory, and
  settings-derived context.
- User prompt preprocessing: prompt hooks, attachments, slash command expansion,
  and process-user-input behavior.
- Tool-time context: tool results, hook additional context, MCP resource reads,
  channel messages, and task notifications.
- Subagent construction: agent definition, agent frontmatter, MCP requirements,
  agent memory, skills, and SubagentStart hooks.
- Background reinjection: task notifications, Agent View transcript reads,
  scheduled task prompts, dream results, and speculation acceptance.

## System Prompt Inputs

System prompt construction depends on:

- main product prompt text.
- current date/environment.
- tool set.
- permission mode.
- brief/Kairos state.
- output style.
- model/provider decisions.
- feature-gated instruction sections.
- simple/bare mode.
- memory/context availability.

The system prompt is also cache-sensitive. Subagents can reuse a rendered parent
system prompt to preserve prompt-cache compatibility.

## CLAUDE.md and Memory Files

CLAUDE.md is loaded through the context system, but can be disabled or reduced.

Controls:

- `CLAUDE_CODE_DISABLE_CLAUDE_MDS`
- `CLAUDE_CODE_SIMPLE`
- additional directories passed by CLI/options.
- project trust and working directory.
- memory-related runtime gates.

Important distinction:

- CLAUDE.md is static context.
- hooks are dynamic context.
- skills are conditional procedural context.
- agent definitions are role-specific context.

Do not put everything into CLAUDE.md just because it is available.

## Agent Definition Injection

Agent definitions can inject:

- role description.
- system prompt.
- allowed/disallowed tool lists.
- model override.
- effort.
- color/background UI metadata.
- memory policy.
- isolation policy.
- max turns.
- hooks.
- MCP server requirements.
- skills.

The `Agent` tool filters definitions by MCP availability and permission denial
before presenting them in prompt text.

## Subagent Context Construction

`runAgent()` builds subagent execution context from:

- selected agent definition.
- inherited/cached parent prompt state.
- tool permission context.
- agent-specific tool filter.
- additional context from `SubagentStart`.
- preloaded skills.
- agent memory prompts.
- MCP server scope.
- output style/system prompt modifiers.

Subagents do not simply receive a raw copy of the parent conversation. The
context is reconstructed for the agent role and tool set.

## Hook Context Injection

Hook output can include `additionalContext` for supported events. Important
cases:

- `UserPromptSubmit`: inject before the prompt becomes a model turn.
- `PreToolUse`: inject around tool-use decision/execution.
- `PostToolUse`: inject after a tool succeeds.
- `PostToolUseFailure`: inject after a tool fails.
- `SubagentStart`: inject into a subagent.
- `Stop`: force/continue behavior at turn end.

The full per-event schema is in `claude-hooks-reference-v2.1.141.md`.

## MCP Context

MCP can affect context through:

- tool schemas.
- tool results.
- resources read by `ReadMcpResourceTool`.
- server prompts.
- elicitation requests.
- plugin-defined hooks.
- channel notifications.

MCP channels are special because they inject external inbound user messages,
not just tool results.

## Skills and Output Styles

Skills:

- bundled/user/project/plugin sources.
- invoked through `Skill`.
- searchable/deferred through skill tooling.
- can carry large procedural instructions without always loading them.

Output styles:

- loaded from output style directories and plugins.
- modify response style.
- may keep or replace default coding instructions.
- can be forced by plugin metadata.

## Teams, Tasks, and Background Context

Teams inject context through:

- team task lists.
- teammate messages.
- plan approval messages.
- shutdown messages.
- task notifications.
- teammate transcripts.

Background agents inject context when:

- a task notification is enqueued.
- `TaskOutput` is read.
- Agent View opens a transcript.
- a background agent completes/fails/is killed.

## Precedence and Conflict Guidance

If instructions conflict, prefer the most specific trusted surface:

- explicit current user prompt over persistent memory.
- agent definition over general CLAUDE.md for that subagent.
- managed policy over local settings.
- hook-injected current context over stale memory when the hook is trusted.
- tool result facts over speculative instructions.

Avoid duplicating the same rule across CLAUDE.md, hooks, skills, and agent
definitions. Duplication makes later diffs and prompt-cache behavior harder to
reason about.

## Injection Timeline

The 2.1.141 context timeline is roughly:

1. process starts and entrypoint/session variables are set.
2. settings and managed policy load.
3. workspace trust and project context resolve.
4. CLAUDE.md and additional directories are discovered unless disabled.
5. system prompt sections are assembled.
6. tools are assembled and filtered.
7. skills, agents, commands, plugins, and MCP resources are discovered.
8. attachments and memories are selected for a turn.
9. hooks can inject additional context at lifecycle boundaries.
10. background task notifications and tool results append context during the
    conversation.

Some steps run once at startup, some run every turn, and some are deferred until
after trust or idle time.

## System Prompt Versus Message Context

Context can enter as:

- system prompt sections.
- system reminders inside the message stream.
- user messages.
- assistant/tool result messages.
- hidden synthetic messages.
- hook `additionalContext`.
- task notifications.
- SDK-provided system prompt overrides.

This distinction matters for prompt-cache behavior. Changing early system prompt
bytes can invalidate the whole cached prefix, while appending a system reminder
later can preserve more of the cache.

## CLAUDE.md Disable and Add-Dir Controls

CLAUDE.md loading is affected by:

- `CLAUDE_CODE_DISABLE_CLAUDE_MDS`.
- `CLAUDE_CODE_SIMPLE`/bare mode.
- explicit `--add-dir`.
- `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD`.
- workspace trust.
- project root and original cwd.
- import/size limits in CLAUDE.md processing.

If an instruction is missing, check disable/env state before assuming the file
was parsed incorrectly.

## Agent Definition Context

Agent definitions can inject context through frontmatter and body content:

- `name`
- `description`
- `tools`
- `model`
- `background`
- `color`
- `memory`
- `isolation`
- `effort`
- `permissionMode`
- `maxTurns`
- `disallowedTools`
- `hooks`
- MCP tool references.
- skills.

Subagents do not simply inherit all parent context unchanged. The agent runtime
constructs a child context with its own definition, tool policy, and optional
cache-sharing parameters.

## Hook Context Safety

Hooks can inject additional context, but this is one of the easiest ways to
damage prompt quality:

- hook context should be current.
- hook context should be concise.
- hook context should identify provenance.
- hook context should not contain secrets.
- hook context should not duplicate large files.
- hook context should not contradict the current user prompt.

Hook stdout and hook JSON output are different. Only the structured output fields
intended for context should be treated as model-facing context.

## MCP and Plugin Context

MCP and plugins inject context through several surfaces:

- tool schemas.
- resource lists.
- resource contents.
- commands.
- plugin instructions.
- MCP channels.
- permission relay text.
- marketplace/plugin metadata.

MCP availability can change after startup. Deferred MCP resource prefetch and
reconnect behavior mean context can evolve during a session.

## Future Diff Checklist

For later releases:

1. Inspect `context.ts` and `constants/prompts.ts`.
2. Inspect CLAUDE.md loader and import handling.
3. Inspect agent loader/frontmatter parser.
4. Inspect hook schemas and execution.
5. Inspect skills loader and bundled skill registry.
6. Inspect MCP client resource/tool registration.
7. Inspect plugin loader instruction surfaces.
8. Inspect memory and relevant-memory prefetch.
9. Inspect task notification message builders.
10. Inspect SDK/print system prompt override handling.

## Deep 2.1.141 Injection Timeline Addendum

Context injection in 2.1.141 is not a single function. It is a timeline of
system prompt assembly, user-context prepending, file/memory attachments, hook
output, MCP/tool discovery, queued notifications, and SDK/remote control
messages.

### Startup And Setup Phase

Before the first query:

- `main.tsx` parses flags that can replace or append system prompts.
- `--bare` sets `CLAUDE_CODE_SIMPLE=1`, disabling auto-discovered context while preserving explicit inputs.
- setup initializes UDS messaging, file-changed hook watcher, context-collapse, session-file analytics, and team memory watchers where gated.
- `startDeferredPrefetches()` can warm system context, credentials, MCP registry URLs, passes, and fast-mode state.
- MCP config loading is started early but MCP resource prefetch is deferred until after trust-sensitive setup.
- command, agent, skill, plugin, and MCP loading can all affect model-visible capability context.

### System Context

`context.ts` builds system context with:

- git status snapshot when git instructions are enabled and the session is not remote.
- current branch.
- main/default branch.
- git user name when available.
- short status, truncated at 2000 characters.
- last five commits.
- optional cache-breaker injection when the build feature is active.

`appendSystemContext()` appends system context as `key: value` strings to the
system prompt. This is separate from user context and is not wrapped in the
same `<system-reminder>` block.

### User Context

`getUserContext()` builds user context with:

- CLAUDE.md content unless disabled.
- current local date.

It skips automatic CLAUDE.md discovery when `CLAUDE_CODE_DISABLE_CLAUDE_MDS` is
truthy, or in `--bare` mode with no explicit added directories. `--bare` still
honors explicit `--add-dir` directories. The function also writes the CLAUDE.md
content to bootstrap cache for the auto-mode classifier.

`prependUserContext()` injects user context as a meta user message containing a
`<system-reminder>` wrapper. The reminder explicitly tells the model that the
context may or may not be relevant and should not be answered unless relevant.

### CLAUDE.md And Memory

CLAUDE.md injection is assembled through `utils/claudemd.ts` and related memory
loaders. It can include:

- user/project/local memory files.
- additional directories from `--add-dir`.
- imported memory files.
- filtered injected memory files.
- cache state used by auto mode.

Memory prefetch can inject attachments later in the query loop, after tool
execution, if the prefetch has settled and the model has not already read or
edited the same memory file.

### Tool-Turn Injection

After each tool batch, `query.ts` can inject:

- queued command/task notifications.
- background task completion prompts.
- memory attachments.
- skill discovery prefetch attachments.
- hook blocking/error/cancel attachments.
- compact or max-token recovery messages.

The queue drain is scoped. Main-thread queries drain main prompts and
notifications; subagents only drain task notifications addressed to their own
agent id. Slash commands are excluded from mid-turn drain because they must go
through slash-command processing after the turn.

### Hook Injection

Hook outputs can inject:

- `additionalContext`.
- `systemMessage`.
- blocking errors.
- permission decisions.
- modified tool output.
- cancellation attachments.
- async follow-up messages.

The hook path matters because hook-injected context can affect the next API
request even when no user-visible text appears in the transcript. For future
release maps, hook output schemas and `utils/hooks.ts` execution order should
be treated as context-injection surfaces.

### MCP And Plugin Injection

MCP and plugins inject context through:

- tool schemas.
- deferred tool search descriptions.
- MCP resources.
- MCP commands.
- channel notifications.
- plugin-provided commands/skills/agents.
- marketplace/plugin metadata.
- dynamically managed MCP servers in SDK print mode.

The visible model context can change after startup because MCP servers can
connect, reconnect, be toggled, or be replaced through SDK control messages.

### Agent And Team Injection

Agent/team features inject:

- main-thread agent system prompts.
- custom SDK/CLI agent definitions.
- teammate identity and team context.
- task-list/team coordination context.
- Agent View task notifications.
- Brief/SendUserMessage communication instructions.
- channel/remote-control availability.

Agent context is especially version-sensitive. Future maps should verify both
agent definition schemas and runtime filtering in `getTools`, `AgentTool`, and
team spawn helpers.

### Print/SDK Injection

Print mode adds additional injection routes:

- SDK initialize can provide `systemPrompt`, `appendSystemPrompt`, `agents`, hooks, and JSON schema.
- stream-json input can include file attachments.
- dynamic MCP control messages can add/remove servers mid-session.
- SDK host permission responses can alter tool execution flow.
- prompt suggestions can be emitted after a result.
- `--rewind-files` can restore filesystem state and exit before a normal turn.

For documentation, do not treat `claude -p` as a context-lite path. In
2.1.141 it is one of the richest injection paths because it exposes control
protocol state that the TUI does not.
