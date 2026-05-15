# CLAUDE.md Behavior and Best Practices in Claude Code 2.1.141

This is a source-grounded 2.1.141 update to the older CLAUDE.md guidance. The
best practices remain simple: keep project memory short, concrete, and scoped;
but the exact behavior matters because CLAUDE.md content is only one of several
context-injection paths.

## Loading Controls

CLAUDE.md loading is managed through the context subsystem, especially:

- `source/src/context.ts`
- `source/src/utils/envUtils.ts`
- `source/src/constants/prompts.ts`
- `source/src/main.tsx`

Loading can be disabled by:

- `CLAUDE_CODE_DISABLE_CLAUDE_MDS`
- simple/bare mode paths such as `CLAUDE_CODE_SIMPLE`
- source-specific gates in context loading.

Additional directories for CLAUDE.md discovery can be injected by CLI/options
and environment-derived state, including `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD`.

## Practical Hierarchy

Use memory at the narrowest level that matches the instruction:

- User-level memory for durable personal preferences.
- Project-level `CLAUDE.md` for repository-specific build/test/style commands.
- Local/private memory for machine-local paths or unreproducible setup.
- Agent frontmatter/system prompts for subagent-specific instructions.
- Hooks for dynamic context injection that should be computed at runtime.

Do not use CLAUDE.md for secrets, long pasted docs, or instructions that are
more accurately represented by a slash command, skill, hook, or agent definition.

## What to Put in CLAUDE.md

Good project-level contents:

- exact test commands.
- exact lint/typecheck commands.
- repository layout.
- non-obvious architecture constraints.
- generated-file rules.
- conventions that the model regularly violates.
- setup caveats that are stable across machines.

Bad project-level contents:

- giant framework tutorials.
- copied issue history.
- secrets or tokens.
- instructions that contradict checked-in tooling.
- speculative preferences that should be decided per task.

## Subagent Behavior

Subagents can receive context through more paths than the parent CLAUDE.md
hierarchy:

- inherited/rendered system prompt state.
- agent definition frontmatter.
- agent-specific system prompt text.
- agent memory settings.
- `SubagentStart` hook additional context.
- tool restrictions and MCP server configuration from agent definitions.

Agent loading lives in `source/src/tools/AgentTool/loadAgentsDir.ts`, and
subagent execution is built in `source/src/tools/AgentTool/runAgent.ts`.

## Agent Definition Memory

Agent frontmatter in 2.1.141 supports fields including:

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
- `mcp__...`
- `skills`

When agent memory is enabled and an explicit tool list is present, the loader
can add read/write/edit capabilities required for memory access. This means an
agent memory choice can affect the eventual tool set.

## Import and Context Hygiene

Keep memory files importable and cache-friendly:

- Prefer stable, short sections.
- Put commands in copy-pasteable code blocks.
- Avoid huge generated text.
- Put volatile information in hooks or commands instead of memory.
- Split very large background information into files the model can read on
  demand rather than always loading it.

## Checking What Was Loaded

Useful source features for understanding context:

- `/context` command area under `source/src/commands/context`.
- `CtxInspect` when `feature('CONTEXT_COLLAPSE')` enables the tool.
- debug/session transcript inspection.
- hook logs for `InstructionsLoaded` and related events.

## 2.1.141 Source Index

- `source/src/context.ts`
- `source/src/constants/prompts.ts`
- `source/src/main.tsx`
- `source/src/tools/AgentTool/loadAgentsDir.ts`
- `source/src/tools/AgentTool/runAgent.ts`
- `source/src/services/tools/toolHooks.ts`
- `source/src/commands/context`
