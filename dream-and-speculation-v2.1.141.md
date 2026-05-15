# Dream, Prompt Suggestion, Speculation, and Prefetch in Claude Code 2.1.141

2.1.141 has several background or predictive systems that can run outside the
main visible turn: auto dream, the `/dream` skill, prompt suggestion,
speculative execution, and startup/deferred prefetch. They are related because
they reuse forked-agent infrastructure and cache-sensitive request parameters,
but they are independently gated.

## Auto Dream

Auto dream lives under `source/src/services/autoDream`.

Primary files:

- `autoDream.ts`
- `config.ts`
- `consolidationLock.ts`
- `consolidationPrompt.ts`

Runtime config:

- `tengu_onyx_plover`
- user setting `autoDreamEnabled`

The config file deliberately avoids importing the full agent/task/message chain
so the setting check can remain lightweight.

Auto dream behavior:

- checks whether enough time has passed since last consolidation.
- checks whether enough sessions have changed.
- uses a lock file to avoid concurrent consolidation.
- forks work with `querySource: 'auto_dream'`.
- records successful consolidation.
- rolls back/delays when failures occur.

Observed telemetry:

- `tengu_auto_dream_fired`
- `tengu_auto_dream_completed`
- `tengu_auto_dream_failed`

## /dream Skill

Bundled dream behavior lives in `source/src/skills/bundled/dream.ts`.

Runtime gate:

- `tengu_kairos_dream`

The skill can schedule recurring dream consolidation through the Cron tools when
they are available. The nightly schedule persists through
`.claude/scheduled_tasks.json` when durable cron is enabled and selected.

Observed telemetry:

- `tengu_dream_invoked`

## Prompt Suggestion

Prompt suggestions live in `source/src/services/PromptSuggestion/promptSuggestion.ts`.

Enablement:

- `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION=true` forces enablement.
- `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION=false` disables it.
- otherwise `tengu_chomp_inflection` controls the default.
- disabled in non-interactive sessions.
- disabled for swarm teammates.
- disabled when settings set `promptSuggestionEnabled === false`.

Suppression/guard conditions include:

- fewer than two assistant turns.
- last assistant message was an API error.
- parent cache state is cold or expensive.
- pending permission or elicitation.
- plan mode.
- external rate-limit suppression.
- query source not being the main REPL thread.

Observed telemetry:

- `tengu_prompt_suggestion_init`
- `tengu_prompt_suggestion`
- prompt suggestion suppression/debug logging.

## Speculation

Speculation lives in `source/src/services/PromptSuggestion/speculation.ts`.
It runs a forked agent ahead of user acceptance of a suggested prompt, then
injects the speculated messages if the user accepts.

Enablement:

- `USER_TYPE === 'ant'`
- global config `speculationEnabled` not false.

Isolation:

- speculative writes go into an overlay directory under Claude temp space.
- writes are copied back only on accepted speculation.
- reads of files already written in the overlay are redirected to the overlay.
- reads outside the current working directory are allowed as reads.
- writes outside the current working directory are denied.

Allowed work:

- read-only tools.
- write tools only when the current permission mode can auto-accept edits.
- read-only Bash commands that pass shell read-only constraints.

Boundaries that stop speculation:

- edit permission boundary.
- non-read-only Bash boundary.
- denied/unknown tool boundary.
- message count limit.
- turn limit.
- abort on user typing.

Acceptance behavior:

- aborts the speculative run.
- copies overlay writes back when there are clean messages.
- injects cleaned speculative messages into the visible transcript.
- writes a `speculation-accept` transcript entry when time was saved.
- if speculation was incomplete, drops trailing assistant messages so the
  follow-up query ends in a user turn.
- can promote a pipelined next suggestion when speculation completed fully.

Observed telemetry:

- `tengu_speculation` with outcomes such as accepted, aborted, and error.

## Deferred Prefetch

Startup and deferred prefetch are rooted in `startDeferredPrefetches()` in
`source/src/main.tsx`.

It skips bare/simple mode and warms several caches or background resources,
including:

- system context when safe.
- user context.
- cloud-provider auth checks.
- official MCP URLs.
- file-change detectors.
- model/gate-related state.
- plugin/skill/command file watchers.

The headless/print path starts deferred prefetches earlier because there is no
interactive UI startup path to hide latency behind.

## Relationship Between Systems

- Prompt suggestion proposes a likely next user prompt.
- Speculation can run that prompt ahead of time.
- Auto dream consolidates memories/background information after session changes.
- `/dream` exposes dream consolidation as a user-invoked or scheduled skill.
- Prefetch warms data needed by normal turns and background systems.

They share forked-agent, cache-safe parameter, transcript, telemetry, and abort
infrastructure, but each has its own gates and suppressors.

## 2.1.141 Source Index

- `source/src/services/autoDream/autoDream.ts`
- `source/src/services/autoDream/config.ts`
- `source/src/services/autoDream/consolidationLock.ts`
- `source/src/services/autoDream/consolidationPrompt.ts`
- `source/src/skills/bundled/dream.ts`
- `source/src/services/PromptSuggestion/promptSuggestion.ts`
- `source/src/services/PromptSuggestion/speculation.ts`
- `source/src/query/stopHooks.ts`
- `source/src/main.tsx`
