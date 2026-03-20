# Claude Code Docs

Unofficial deep-dive reference documentation for [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code), derived from source analysis of the bundled `cli.js`.

> **Current version covered:** Claude Code v2.1.71 (built 2026-03-06)

## What's Here

Each document is a standalone, exhaustive reference on a specific subsystem of Claude Code. These are not tutorials — they are technical references produced by reading the minified source bundle (612,918 lines) and documenting the internal behavior, schemas, and data flows.

### Core References (v2.1.71)

| Document | Description |
|----------|-------------|
| [Native Tools Reference](native-tools-2.1.71.md) | All built-in tools (Read, Write, Edit, Bash, Grep, Glob, Agent, Task, etc.) — schemas, dispatch, parameter aliasing, permission checks |
| [Hooks System Reference](claude-hooks-reference-2.1.71.md) | All 21 hook events, 4 hook types (command, prompt, agent, MCP), execution internals, schemas, and policies |
| [Environment Variables Reference](environment-variables-2.1.71.md) | 117 `CLAUDE_CODE_*` vars, 15 `CLAUDE_*` vars, 17 `ANTHROPIC_*` vars, feature toggle gates, and third-party integrations |
| [Telemetry Reference](telemetry-2.1.71.md) | 6 telemetry pipelines, 598 named event types, OpenTelemetry SDK integration, sampling, gating, and kill-switch controls |
| [Statsig & Feature Gates](statsig-gates-2.1.71.md) | 73 unique feature flags/configs across 103 call-sites — Statsig gates, GrowthBook experiments, and dynamic configs |
| [/loop Command Deep-Dive](loop-command-2.1.71.md) | The `/loop` scheduling command, `CronCreate`/`CronDelete`/`CronList` tools, and the background cron scheduler (Statsig-gated) |
| [Tool Aliasing & Dispatch](tool-aliasing-2.1.71.md) | Parameter aliasing (`inputParamAliases`) and tool name aliasing (`aliases[]`) in the tool dispatch pipeline |

### Guides & Techniques

| Document | Description |
|----------|-------------|
| [CLAUDE.md Best Practices](claude-md-best-practices.md) | Structuring and maintaining CLAUDE.md files across enterprise, user, project, and local levels |
| [Context Injection Avenues](context-injection-avenues.md) | All methods for injecting system prompts and context into Claude Code and its subagents |
| [Agent Teams Reference](agent-teams-info.md) | Agent Teams architecture — team creation, agent types, spawning, lifecycle, permissions, and tool restrictions |
| [Injecting Context After Agent Finish](inject-after-agent-finish.md) | Using `PostToolUse` hooks to inject additional context after subagent completion |
| [Token Tracking System](cached-tokens-misc-info.md) | API-level token keys, prompt caching mechanics, and cost calculations |

## Who This Is For

- **Power users** building custom hooks, agent teams, or CLAUDE.md configurations
- **Developers** integrating Claude Code into CI/CD pipelines or enterprise toolchains
- **Researchers** studying Claude Code's internal architecture, telemetry, and feature gating

## Disclaimer

This is an **unofficial** community resource. It is not affiliated with or endorsed by Anthropic. The documentation is based on source analysis of the publicly distributed CLI bundle and may become outdated as new versions are released. Always refer to [Anthropic's official documentation](https://docs.anthropic.com/en/docs/claude-code) for authoritative guidance.