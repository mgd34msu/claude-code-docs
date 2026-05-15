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
- other hook events covered in `hooks-v2.1.141.md`.

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
