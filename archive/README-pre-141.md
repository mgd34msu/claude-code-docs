## I'm skipping 2.1.88 for obvious reasons...

I will resume analysis when 2.1.89 comes out, as I don't want to give even a slight impression that I may be using leaked source code.

# Claude Code Docs

Unofficial deep-dive reference documentation for [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code), derived from source analysis of the bundled `cli.js`.

> **Latest version covered:** Claude Code v2.1.81 (built 2026-03-20)  
> Also includes: v2.1.71 references for tools, feature gates, /loop, and tool aliasing

## What's Here

Each document is a standalone, exhaustive reference on a specific subsystem of Claude Code. These are not tutorials — they are technical references produced by reading the source bundle and documenting the internal behavior, schemas, and data flows.

### Core References (v2.1.81)

| Document | Description |
|----------|-------------|
| [Hooks System Reference](claude-hooks-reference-2.1.81.md) | Complete hooks system — all hook events, hook types, execution internals, Zod schemas, and policies |
| [Environment Variables Reference](environment-variables-2.1.81.md) | All `CLAUDE_CODE_*`, `CLAUDE_*`, `ANTHROPIC_*` vars, feature toggle gates, boolean parsing, and third-party integrations |
| [Telemetry Reference](telemetry-2.1.81.md) | Multi-layered telemetry with 781 unique `tengu_` event types, Segment, Datadog, and OpenTelemetry backends |
| [Auto Mode Reference](auto-mode-2.1.81.md) | Permission modes, auto mode activation, opt-in dialog, bypass permissions, and CLI flags |
| [Brief Mode / SendUserMessage](brief-mode-2.1.81.md) | The SendUserMessage tool, brief mode behavior, input/output schemas, and tool properties |
| [MCP Channels System](channels-2.1.81.md) | Experimental `--channels` feature — real-time MCP server notifications pushed into running sessions |
| [Dream Mode & Speculation](dream-and-speculation-2.1.81.md) | Background memory consolidation (dream mode), speculation system, and deferred prefetch |
| [Undocumented Features](undocumented-features-2.1.81.md) | Feature flags, hidden CLI options, internal mechanisms, stubs, and experimental features not in official docs |

### Core References (v2.1.71)

| Document | Description |
|----------|-------------|
| [Native Tools Reference](native-tools-2.1.71.md) | All 45+ built-in tools — schemas, dispatch, parameter aliasing, permission checks |
| [Statsig & Feature Gates](statsig-gates-2.1.71.md) | 73 unique feature flags/configs across 103 call-sites — Statsig gates, GrowthBook experiments, and dynamic configs |
| [/loop Command Deep-Dive](loop-command-2.1.71.md) | The `/loop` scheduling command, cron tools, and the background scheduler (Statsig-gated) |
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
