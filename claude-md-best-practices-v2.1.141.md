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

## 2.1.141 Loading Model

CLAUDE.md is one of several persistent context sources. In 2.1.141, loading is
affected by:

- current working directory.
- additional directories.
- config directory.
- simple/bare mode.
- disable environment variables.
- trust/project state.
- memory-related feature gates.

The most important local kill switch is:

```text
CLAUDE_CODE_DISABLE_CLAUDE_MDS
```

Simple mode (`CLAUDE_CODE_SIMPLE`) also skips or reduces several background and
context-loading behaviors.

## Recommended File Roles

User-level memory:

- durable personal preferences.
- preferred commands that apply across many repos.
- style preferences that do not conflict with project tooling.

Project-level memory:

- build/test/lint/typecheck commands.
- repository architecture summary.
- generated-file policy.
- dependency-management rules.
- deployment or packaging caveats.
- local conventions not obvious from code.

Local/private memory:

- machine-local paths.
- local service names.
- credentials instructions without actual credentials.
- temporary workflow notes that should not be committed.

Agent memory:

- role-specific behavior.
- agent-specific tool constraints.
- task-specific long-lived notes.

## What Good CLAUDE.md Looks Like

Good:

```md
# Project Commands

- `pnpm test`: run unit tests.
- `pnpm typecheck`: run TypeScript checks.
- `pnpm lint`: run lint.

# Architecture

- API routes live in `src/routes`.
- Background jobs live in `src/jobs`.
- Generated files in `src/generated` must not be edited by hand.
```

Bad:

```md
# Always do a good job

This repository is important. Be careful. Use best practices. Think hard.
```

The good version is concrete and testable. The bad version consumes context
without changing behavior.

## Import and Size Discipline

Keep persistent memory short. Large background documents should usually be:

- a skill.
- a file the model can read on demand.
- a slash command.
- a hook that injects only relevant runtime information.

Persistent memory is loaded frequently, so every token has repeated cost and
prompt-cache impact.

## Commands Section Rules

Commands should be:

- exact.
- copy-pasteable.
- scoped.
- annotated only when the command is non-obvious.

Avoid:

- "run tests" without command.
- outdated package managers.
- commands that require unmentioned services.
- commands that mutate production state.

## Subagent Implications

Subagents can receive:

- inherited system prompt state.
- selected agent definition prompt.
- agent memory.
- `SubagentStart` hook additional context.
- skills and MCP tools specified in frontmatter.

This means a subagent may not need every project-level detail in CLAUDE.md.
If a rule applies only to a code-review agent, put it in that agent definition
or skill instead.

## Anti-Patterns

- Storing secrets.
- Pasting whole API docs.
- Duplicating README content.
- Adding emotional or motivational instructions.
- Encoding temporary issue state.
- Contradicting checked-in formatter/linter config.
- Adding broad "always" rules that are false in some packages.
- Using CLAUDE.md to compensate for missing tests or scripts.

## Maintenance Checklist

Review CLAUDE.md when:

- package manager changes.
- test commands change.
- repo layout changes.
- generated file locations change.
- project moves from one framework to another.
- agents/skills/hooks replace persistent instructions.

Remove instructions when:

- they became obvious from code.
- they refer to deleted paths.
- they duplicate a skill.
- they only mattered for a completed migration.

## Debugging Loaded Context

Use:

- `/context` to inspect context where available.
- hook logs for `InstructionsLoaded`.
- transcript/system init inspection.
- `CtxInspect` when context collapse tooling is available.

If an expected CLAUDE.md instruction is not affecting behavior, check disable
env vars, simple mode, working directory, additional directory state, and
whether a more specific instruction overrides it.

## Source-Derived Loading Rules

The important 2.1.141 loading rules are:

- CLAUDE.md discovery can be disabled completely.
- bare/simple mode skips auto-discovery.
- explicit directories can be added.
- imports are resolved subject to loader limits.
- loaded instructions become part of context, not executable code.
- stale or overly large files still cost prompt/cache budget.
- subagents may receive different context from the leader.

Best-practice docs should therefore optimize for small, stable, high-signal
instructions rather than exhaustive project documentation.

## Recommended Structure

Use sections like:

- `Commands`: exact commands the agent should run.
- `Architecture`: short map of major directories.
- `Generated Files`: what not to edit manually.
- `Testing`: required fast and slow test commands.
- `Style`: repo-specific conventions not enforced by formatter.
- `Safety`: destructive operations that require explicit confirmation.
- `Release`: packaging or deploy steps if the agent is asked to release.

Avoid prose that sounds helpful but does not change behavior. If a human could
not verify whether the instruction was followed, it probably does not belong in
CLAUDE.md.

## Bad Instruction Patterns

Remove or avoid:

- "be careful" style instructions.
- "do a good job" style instructions.
- long copied README/API docs.
- stale migration notes.
- issue-specific temporary plans.
- secrets or credentials.
- huge lists of files.
- instructions that contradict checked-in scripts.
- formatting rules already enforced by tooling.
- broad absolutes such as "always run all tests" when that is not practical.

Every line in CLAUDE.md competes with user context and prompt cache stability.

## Subagent-Specific Guidance

If an instruction only applies to one role, prefer an agent definition over
global CLAUDE.md. Examples:

- code-review heuristics belong in a review agent.
- documentation style belongs in a docs agent.
- migration rules belong in a migration agent.
- test triage rules belong in a test agent.

This keeps the leader prompt lighter and prevents unrelated agents from obeying
irrelevant rules.

## Import Hygiene

Imports can make CLAUDE.md maintainable, but they also hide context cost. Good
use:

- import a short package-specific command list.
- import a generated architecture summary.
- import a stable safety policy.

Bad use:

- import entire API docs.
- import long logs.
- import generated code.
- import files that change every run.
- import deep chains that are hard to audit.

For future source-map work, always document import limits and failure behavior
from the current source rather than assuming older behavior.

## Review Checklist

A good 2.1.141 CLAUDE.md should pass these checks:

1. It is short enough to read quickly.
2. It contains exact commands.
3. It names generated and vendored paths.
4. It avoids secrets.
5. It avoids stale issue state.
6. It does not duplicate agent definitions or skills.
7. It has no instructions contradicted by scripts/config files.
8. It explains non-obvious repo conventions.
9. It is stable across ordinary commits.
10. It improves model behavior in a way a reviewer can observe.
