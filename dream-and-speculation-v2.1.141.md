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

## Auto Dream Trigger Model

Auto dream is not a timer that blindly fires. It checks:

- whether auto dream is enabled by settings/config.
- how long it has been since the last consolidation.
- whether enough sessions changed since last consolidation.
- whether scan throttling allows another scan.
- whether a consolidation lock can be acquired.
- whether the user aborted or the fork failed.

Thresholds come from `tengu_onyx_plover`, while explicit user setting
`autoDreamEnabled` can override the default state.

## Auto Dream Execution Flow

1. Read config.
2. Read last consolidated timestamp.
3. List sessions touched since last consolidation.
4. Skip if session count or time threshold is not met.
5. Acquire lock.
6. Register a dream task for UI/task state.
7. Build consolidation prompt.
8. Run a forked agent with `querySource: 'auto_dream'`.
9. Track edited/written memory files.
10. Complete or fail the dream task.
11. Record consolidation timestamp on success.
12. Roll back or delay on failure/abort.

The dream task is a background task type, so it participates in task registry
and cancellation behavior.

## Consolidation Lock

The lock prevents concurrent dream runs. It records lock holder information and
checks whether the holding process is still live. If rollback fails, the next
trigger is delayed to the minimum-hours threshold rather than spinning.

This is important because auto dream can edit memory files; concurrent runs
would risk inconsistent memory state.

## /dream Skill Modes

The bundled dream skill is gated by `tengu_kairos_dream`.

Observed behavior:

- can invoke consolidation directly.
- can schedule nightly consolidation.
- uses Cron tools when available.
- logs `tengu_dream_invoked`.
- if scheduling is unavailable, reports schedule-unavailable mode.

The skill does not implement its own scheduler; it composes with Cron tools.

## Prompt Suggestion Enablement

`shouldEnablePromptSuggestion()` checks:

- `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION=false` disables.
- `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION=true` enables.
- otherwise `tengu_chomp_inflection` controls default.
- non-interactive sessions are disabled.
- swarm teammates are disabled.
- settings `promptSuggestionEnabled === false` disables.

Initialization logs `tengu_prompt_suggestion_init`.

## Prompt Suggestion Guards

Prompt suggestion is suppressed when:

- fewer than two assistant turns exist.
- last assistant turn was an API error.
- parent cache is cold.
- parent input/cache-write/output size is too large.
- permission request is pending.
- elicitation is pending.
- session is in plan mode.
- query source is not the main REPL thread.
- external rate-limit logic says not to run.

Accepted/ignored suggestions log `tengu_prompt_suggestion` with interaction
metadata such as accept method and timing.

## Speculation Isolation

Speculation uses a temp overlay:

```text
<claude-temp>/speculation/<pid>/<id>
```

Write tools are redirected into the overlay. Reads are redirected only if the
file was already written in the overlay. On accept, overlay writes are copied
back to the real working tree.

Denied during speculation:

- writes outside cwd.
- writes requiring permission when mode cannot auto-accept edits.
- non-read-only Bash commands.
- unknown or non-allowed tools.

Allowed:

- safe read-only tools.
- read-only Bash commands.
- writes when permission mode can auto-accept edits.

## Speculation Boundaries

Completion boundary types include:

- complete.
- edit boundary.
- Bash boundary.
- denied-tool boundary.
- error.
- abort.

If speculation finishes completely and the user accepts, the speculated messages
can be injected directly. If it stops at a boundary, the accepted suggestion
still uses partial work where safe and then falls back to a normal query.

## Pipelined Speculation

When a speculation completes, 2.1.141 can generate a pipelined next suggestion
while waiting for user acceptance. If the user accepts the completed speculation,
the pipelined suggestion can be promoted and speculation can start again for the
next likely prompt.

This is a performance optimization and is separate from the basic prompt
suggestion feature.

## Deferred Prefetch Details

`startDeferredPrefetches()` warms:

- system context.
- user context.
- cloud-provider auth.
- official MCP metadata.
- file change detectors.
- settings and model/gate data.
- plugin/skill/command watchers.

It returns early in bare/simple mode because those modes are intended to avoid
expensive background setup.

## Telemetry Summary

- `tengu_auto_dream_fired`
- `tengu_auto_dream_completed`
- `tengu_auto_dream_failed`
- `tengu_dream_invoked`
- `tengu_prompt_suggestion_init`
- `tengu_prompt_suggestion`
- `tengu_speculation`

Speculation telemetry includes outcome, time saved, message count, boundary, and
whether the speculation was pipelined.

## Auto Dream Gate Order

Auto dream uses the cheapest gates first:

1. disabled if KAIROS assistant mode is active because KAIROS uses the disk skill
   dream path.
2. disabled in remote mode.
3. disabled if auto-memory is off.
4. disabled if auto-dream is unavailable or user-disabled.
5. time gate: hours since last consolidation must meet `minHours`.
6. session gate: enough sessions must have changed since last consolidation.
7. lock gate: no other live process may be consolidating.

The defaults are 24 hours and 5 sessions when the GrowthBook object is absent or
malformed. The same `tengu_onyx_plover` config carries availability/enabled
state and scheduling thresholds.

## Consolidation Lock Behavior

The lock protects memory files from concurrent dream agents. Important behavior:

- live PID lock holders block new consolidation.
- failed acquisition logs debug information and skips the run.
- rollback exists for fork failures so the next trigger is delayed correctly.
- manual `/dream` records consolidation state optimistically at prompt-build
  time.
- scan throttling prevents repeated expensive transcript scans when the time
  gate passes but the session gate does not.

The lock is a correctness feature, not just an optimization. Without it, two
background agents could edit memory files simultaneously.

## Dream Agent Tool Policy

Auto dream runs as a forked agent with a constrained tool policy. It is allowed
to edit/write memory files, but the tool-use policy is not the ordinary full
interactive tool pool. Source points:

- it uses cache-safe fork parameters.
- it creates a dream task for UI/task tracking.
- it records dream turns with tool-use counts collapsed.
- it appends memory-saved messages when files were touched.
- it fails or completes the task explicitly.

Dream is therefore both a memory feature and a background task feature.

## Manual /dream Skill Modes

The bundled `/dream` skill covers:

- immediate consolidation.
- scheduled/nightly setup.
- unavailable scheduling fallback.
- telemetry for invocation mode.
- alias `learn`.
- coordination with cron tools.

The nightly path intentionally schedules `"/dream consolidate"` so the fired
prompt resolves to consolidation behavior rather than being interpreted as a new
scheduling request.

## Prompt Suggestion Gating

Prompt suggestions are gated by:

- app state `promptSuggestionEnabled`.
- settings and UI toggle state.
- environment variable `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION`.
- prompt input state.
- whether the user is typing a slash command.
- whether Agent View is focused on an agent transcript.
- cache economics and current usage.
- SDK `promptSuggestions` request flag in print/SDK mode.

The UI feature is small, but the code is deliberately conservative because
background suggestion requests can waste tokens or disturb prompt-cache
assumptions.

## Speculation Overlay Model

Speculation uses an overlay model for file changes:

- reads can map through overlay paths.
- writes land in a temporary speculation area.
- accepted speculation can copy results back.
- rejected/aborted speculation can be discarded.
- tool boundaries stop unsafe speculative continuation.

The overlay path is under a temp speculation directory keyed by process/session
state. This keeps speculative edits from mutating the real workspace unless the
user accepts the speculation.

## Speculation Boundaries

Speculation can stop at boundaries such as:

- edit boundary.
- bash boundary.
- denied tool boundary.
- completion boundary.
- stale task/output boundary.
- user typing or cancellation boundary.
- background task state change.

Boundaries are not all failures. Some are normal stop points where partial work
can still make the accepted prompt faster or safer.

## SDK and Print Mode Relationship

Print/SDK mode can request prompt suggestions and receive structured suggestion
events. This is separate from interactive ghost text:

- SDK request has `promptSuggestions`.
- print loop can enable prompt suggestions in app state for that request.
- stream schemas include predicted next-user-prompt events.
- prompt suggestion generation happens after turn completion.

This is important for automation harnesses because prompt suggestions are not
strictly a TUI-only feature.

## Future Diff Checklist

For a later release:

1. Inspect `services/autoDream/config.ts`.
2. Inspect `services/autoDream/autoDream.ts`.
3. Inspect `services/autoDream/consolidationLock.ts`.
4. Inspect bundled `dream` skill.
5. Inspect prompt suggestion service.
6. Inspect speculation service and overlay helpers.
7. Inspect prompt input and typeahead acceptance logic.
8. Inspect SDK/print schemas for prompt suggestion events.
9. Inspect session storage for speculation accept entries.
10. Inspect telemetry event names and cache-usage metadata.
