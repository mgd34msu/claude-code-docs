# Token and Cache Accounting in Claude Code 2.1.141

2.1.141 tracks token usage at three layers: raw API usage, session/model usage
aggregation, and persisted project statistics. Prompt-cache reads and writes are
first-class counters rather than being folded into plain input tokens.

## API Usage Fields

The Anthropic API usage object is handled in `source/src/services/api/claude.ts`
and normalized in `source/src/utils/messages.ts`.

Core fields:

- `input_tokens`
- `output_tokens`
- `cache_creation_input_tokens`
- `cache_read_input_tokens`

Server tool fields:

- `server_tool_use.web_search_requests`
- `server_tool_use.web_fetch_requests`

Prompt-cache tier details:

- `cache_creation.ephemeral_1h_input_tokens`
- `cache_creation.ephemeral_5m_input_tokens`
- `cache_creation.cache_deleted_input_tokens`

Additional API-side usage fields observed in the streaming path include:

- `iterations`
- `speed`

Streaming usage is cumulative. The update logic merges input/cache values only
when positive values are present and updates output/server-tool usage from the
latest stream state.

## Cost Tracker

The main aggregator is `source/src/cost-tracker.ts`.

`addToTotalModelUsage()` accumulates:

- input tokens.
- output tokens.
- cache read tokens.
- cache creation tokens.
- web search request counts.
- model cost in USD.
- model context window and max output token metadata.

`addToTotalSessionCost()` updates app state and emits OpenTelemetry counters for:

- `input`
- `output`
- `cacheRead`
- `cacheCreation`

It also logs advisor token usage through `tengu_advisor_tool_token_usage` when
advisor tooling is involved.

## Persisted Project Stats

`saveCurrentSessionCosts()` persists project usage fields such as:

- `lastTotalInputTokens`
- `lastTotalOutputTokens`
- `lastTotalCacheCreationInputTokens`
- `lastTotalCacheReadInputTokens`
- `lastTotalWebSearchRequests`
- `lastModelUsage`

The stats reader in `source/src/utils/stats.ts` aggregates the same usage shape
from transcript/log data.

## Formatting

`formatTotalCost()` produces user-facing totals for:

- input tokens.
- output tokens.
- cache read tokens.
- cache write/cache creation tokens.
- web search request usage.
- unknown model warnings when cost cannot be calculated cleanly.

## Prompt Cache Control

Cache behavior is spread across request construction, prompt-cache break
detection, and feature gates.

Relevant controls observed in 2.1.141:

- `DISABLE_PROMPT_CACHING`
- `DISABLE_PROMPT_CACHING_HAIKU`
- `DISABLE_PROMPT_CACHING_SONNET`
- `DISABLE_PROMPT_CACHING_OPUS`
- `ENABLE_PROMPT_CACHING_1H_BEDROCK`
- `feature('PROMPT_CACHE_BREAK_DETECTION')`
- runtime configs around 1-hour cache handling.

The prompt suggestion and speculation systems explicitly avoid running when
cache state is cold or when a parent prompt-cache state would be expensive to
break. Prompt suggestion suppression treats large input/cache-write/output
combinations as a reason to avoid speculative follow-up work.

## Server Tool Usage

Web search and web fetch have dedicated server-tool counters in usage. These
are not normal input/output token counters and are aggregated separately:

- `web_search_requests`
- `web_fetch_requests`

The cost display includes web search requests as a distinct usage line.

## Metrics

OpenTelemetry metrics include:

- `claude_code.cost.usage`
- `claude_code.token.usage`

The token metric is tagged by token type, allowing prompt-cache reads and
prompt-cache writes to be separated from ordinary input/output.

## 2.1.141 Source Index

- `source/src/cost-tracker.ts`
- `source/src/services/api/claude.ts`
- `source/src/utils/messages.ts`
- `source/src/utils/stats.ts`
- `source/src/services/api/promptCacheBreakDetection.ts`
- `source/src/services/analytics`

## Usage Object Reference

The normalized usage object in 2.1.141 includes these top-level fields:

- `input_tokens`
- `output_tokens`
- `cache_creation_input_tokens`
- `cache_read_input_tokens`
- `server_tool_use`
- `cache_creation`

`server_tool_use` includes:

- `web_search_requests`
- `web_fetch_requests`

`cache_creation` includes tier-level details:

- `ephemeral_1h_input_tokens`
- `ephemeral_5m_input_tokens`

Streaming API responses can also surface:

- `cache_deleted_input_tokens`
- `iterations`
- `speed`

## API-to-Session Mapping

`source/src/services/api/claude.ts` updates usage from streaming chunks. The
important behavior is that some usage values are cumulative and some are
conditionally merged:

- input token and cache token values update only when present and positive.
- output token values track the stream's latest cumulative output state.
- server tool requests are accumulated from server-side usage blocks.
- cache-creation tier details are preserved separately from aggregate cache
  creation input tokens.

This prevents zero/undefined incremental chunks from wiping previously observed
usage.

## Cost Aggregation Fields

`addToTotalModelUsage()` accumulates per-model totals:

- total input tokens.
- total output tokens.
- total cache-read tokens.
- total cache-creation tokens.
- total web-search requests.
- total cost in USD.
- context window metadata.
- max output token metadata.

`addToTotalSessionCost()` updates session totals and emits metrics. This is the
point where raw usage becomes both UI state and telemetry.

## Persisted Stats Fields

The persisted project/session cost snapshot includes:

- `lastTotalInputTokens`
- `lastTotalOutputTokens`
- `lastTotalCacheCreationInputTokens`
- `lastTotalCacheReadInputTokens`
- `lastTotalWebSearchRequests`
- `lastModelUsage`

Stats utilities aggregate the same fields from transcript/log data. This lets
the UI display current-session usage and also summarize historical sessions.

## Prompt Cache Controls

Environment controls:

- `DISABLE_PROMPT_CACHING`
- `DISABLE_PROMPT_CACHING_HAIKU`
- `DISABLE_PROMPT_CACHING_SONNET`
- `DISABLE_PROMPT_CACHING_OPUS`
- `ENABLE_PROMPT_CACHING_1H_BEDROCK`

Feature/config controls:

- `feature('PROMPT_CACHE_BREAK_DETECTION')`
- runtime prompt-cache dynamic configs.
- cache-break detection service under `source/src/services/api`.

Prompt-cache behavior is model/provider-sensitive. A cache-control decision that
is valid for one model/provider path may not be valid for another.

## Cache Reads vs Cache Writes

2.1.141 treats prompt-cache reads and writes as separate cost/usage categories:

- cache read tokens represent tokens served from cache.
- cache creation tokens represent tokens written into a cache entry.
- both are counted separately from ordinary input tokens.
- both have separate OpenTelemetry token metric categories.

This matters for cost analysis. A large cached prompt can produce a small input
bill but a large cache-read counter, or a large one-time cache-creation counter
followed by cheaper reads.

## Server Tool Accounting

Server-side tools are not normal tool-use token blocks. Web search/fetch request
counts are represented under `server_tool_use`.

Known counters:

- `web_search_requests`
- `web_fetch_requests`

The user-facing cost formatter displays web search separately because it is a
request count, not a token count.

## OpenTelemetry Metrics

Cost and token counters:

- `claude_code.cost.usage`
- `claude_code.token.usage`

Token metric types:

- `input`
- `output`
- `cacheRead`
- `cacheCreation`

These are emitted when session cost is updated, not merely when a raw API chunk
arrives.

## Formatting and Unknown Models

`formatTotalCost()` is responsible for user-visible cost text. It can show:

- input tokens.
- output tokens.
- cache read.
- cache write/cache creation.
- web search request count.
- cost USD.
- unknown-model warnings.

Unknown model warning is important during reconstruction and future version
diffing: a model can be valid for API calls while still missing local cost table
metadata.

## Prompt Suggestion and Cache Interaction

Prompt suggestion/speculation code suppresses work when cache state would make
the background request expensive or likely to break parent cache assumptions.
The suppression logic considers large input/cache-write/output totals and cold
parent cache state.

This is one of the less obvious links between token accounting and UX:
predictive work is gated by cache economics, not only by feature flags.

## Practical Interpretation

When auditing a transcript:

1. Sum `input_tokens` and `output_tokens` for raw model work.
2. Sum `cache_read_input_tokens` separately for cached prompt volume.
3. Sum `cache_creation_input_tokens` separately for cache write volume.
4. Add `server_tool_use` request counts separately.
5. Use model-specific cost tables for USD.
6. Treat missing model cost metadata as an accounting gap, not necessarily an API
   failure.

## Usage Field Semantics

The Anthropic usage object has multiple input-related fields:

- `input_tokens`: uncached input tokens for the request.
- `cache_creation_input_tokens`: tokens written into prompt cache.
- `cache_read_input_tokens`: tokens read from prompt cache.
- `output_tokens`: generated output tokens.
- `server_tool_use`: server-side tool request counts.
- `cache_creation.ephemeral_1h_input_tokens`: one-hour cache write component.
- `cache_creation.ephemeral_5m_input_tokens`: five-minute cache write component.

Total prompt volume is not `input_tokens`. The useful audit formula is:

```text
prompt_volume = input_tokens + cache_creation_input_tokens + cache_read_input_tokens
```

Cost is not the same as volume because cache reads, cache writes, and uncached
input have different rates.

## Streaming Usage Merge

Streaming responses can deliver usage in deltas. 2.1.141 merges per-part usage
into an aggregate usage object:

- positive delta values override or increment the matching aggregate field.
- cache creation subfields are preserved even when SDK types omit them.
- total usage is accumulated across messages.
- empty usage objects explicitly set zero values to avoid undefined math.

This matters for reconstruction because a single final message object is not the
only source of usage data.

## Cost Tracker State

`cost-tracker.ts` maintains session-level totals and model-level totals:

- total cost USD.
- total input tokens.
- total output tokens.
- total cache read tokens.
- total cache creation tokens.
- total web search request count.
- per-model cost and token breakdown.

It also persists last-session values to project config:

- `lastTotalInputTokens`
- `lastTotalOutputTokens`
- `lastTotalCacheCreationInputTokens`
- `lastTotalCacheReadInputTokens`
- `lastTotalWebSearchRequests`
- `lastModelUsage`

Those fields power status/usage displays and help compare the current session
against the prior one.

## Cache Stability Controls

2.1.141 has many explicit prompt-cache stability comments. Stable-cache behavior
depends on:

- stable system prompt bytes.
- stable tool schema ordering.
- stable model name and budget parameters.
- stable forked-agent cache-critical params.
- stable tool-result replacement decisions.
- avoiding timestamps/UUIDs in cached prompt sections.
- sorting built-ins and MCP tools in separate partitions.
- not mutating parent context when forked agents share cache.

Cache accounting docs should therefore include both the reported fields and the
engineering rules that keep those fields healthy.

## Cache Sharing Paths

Several background/forked paths intentionally share the parent prompt cache:

- forked subagents.
- agent summaries.
- memory extraction.
- auto dream.
- prompt suggestions.
- side questions/side queries.
- compaction variants.

These paths pass cache-safe parameters from parent to child so the prefix can
hit the same server-side cache. Diverging system prompt, tools, model, or budget
parameters can turn a cheap background request into a large cache creation.

## Cache-Break Detection

Prompt-cache break detection tracks cache-read drops and emits diagnostic events
when the cache appears to have been invalidated unexpectedly. It treats some
query sources as untracked because background requests such as speculation or
prompt suggestion intentionally differ from the main turn.

Important interpretation:

- a cache-read drop can be expected after a deliberate cache deletion.
- cache creation can spike after tool/schema/prompt changes.
- a model alias change can invalidate a previous cache entry.
- a late-connected chrome/MCP/tool surface can alter prompt bytes.

The metric is a diagnostic, not a proof of user-visible failure.

## Future Diff Checklist

For a later release:

1. Inspect API usage merge functions.
2. Inspect `emptyUsage`.
3. Inspect `cost-tracker.ts`.
4. Inspect bootstrap metric creation.
5. Inspect model cost tables.
6. Inspect forked-agent cache-safe parameter code.
7. Inspect tool-pool sorting and tool schema cache.
8. Inspect prompt-cache break detection.
9. Inspect SDK usage schemas.
10. Verify all cache fields are documented with exact names and not collapsed
    into a single "cached tokens" number.

## Deep 2.1.141 Token And Cache Accounting Addendum

The 2.1.141 source has several distinct concepts that are easy to collapse
incorrectly:

- API usage fields reported by model responses.
- local cost tracking.
- prompt-cache reads and writes.
- cache deletion from cached microcompact.
- read-file state cache for file tools.
- memoized config/auth/tool/schema caches.
- cache-safe parameters for prompt suggestions and speculation.

Only the first three are "cached tokens" in the ordinary billing/accounting
sense.

### Usage Field Semantics

The query path reads usage from assistant messages and compaction responses. The
important fields surfaced in source include:

- `input_tokens`.
- `output_tokens`.
- `cache_read_input_tokens`.
- `cache_creation_input_tokens`.
- `cache_deleted_input_tokens`.

Compaction total token math in `query.ts` adds input, cache creation, cache
read, and output fields. Cached microcompact compares cumulative
`cache_deleted_input_tokens` against a baseline so it can compute how many
cached tokens were actually deleted by a cache edit.

### Cached Microcompact

When `feature('CACHED_MICROCOMPACT')` is active, microcompact can produce
pending cache edits. The main query loop waits until the next API response to
read actual `cache_deleted_input_tokens`. This is necessary because local token
estimation cannot know the server-side cache-deletion result.

Future documentation should not describe cached microcompact as "removes N
tokens" unless it specifies whether N is estimated, requested, or confirmed by
`cache_deleted_input_tokens`.

### Stale Usage Guard

The query loop explicitly avoids using stale `input_tokens` from kept messages
when checking compaction/context limits. The source comment calls out a prior
failure mode where a session could compact successfully but kept messages still
carried stale pre-compaction usage above the blocking limit. That means usage
metadata on old messages is not always the current context size.

### Cache-Safe Params

`utils/forkedAgent.ts` creates `CacheSafeParams`, and print/SDK paths use
`getLastCacheSafeParams()` for prompt suggestions. The goal is to run follow-on
requests with a stable prefix so prompt-cache hits are possible. Consumers
include:

- prompt suggestions.
- speculation.
- away summaries.
- SDK push suggestions after print turns.

Cache-safe does not mean "no state changed." It means the request has the data
needed to form a prompt prefix compatible with the cached parent request.

### Read File State Cache

`cli/print.ts` creates and maintains a `readFileState` cache across headless
turns. It also tracks pending seeds and merges them into the cache before
`ask()`. This cache is about file contents the model has already seen, not API
prompt-cache tokens. It prevents unnecessary rereads and helps edit correctness
in long-running print/SDK sessions.

### Tool Schema And Deferred Tool Cache

Tool schema token cost is influenced by tool selection and deferral:

- deferred tools do not all enter the initial prompt.
- `ToolSearch` can reveal deferred tools later.
- MCP tools can connect late or be refreshed by SDK control messages.
- plugin reload clears relevant command/agent/hook/tool caches.

Token accounting for tools should therefore identify whether it is measuring
all registered tools, currently loaded tools, deferred tool descriptions, or
MCP tools exposed to the current turn.

### Auth And Settings Caches Are Not Token Caches

The source has many caches around OAuth tokens, keychain reads, API key helpers,
Bedrock/GCP credentials, remote settings, policy limits, plugin state, commands,
and GrowthBook values. These affect behavior and latency, but they are not
cached-token billing metrics. They can indirectly affect prompt-cache hits by
changing system prompt, tools, settings, or provider, but they should be
documented separately.

### Practical Accounting Rules

Use these rules when reading 2.1.141 telemetry or future source:

- If the field is `cache_read_input_tokens`, it means input tokens served from prompt cache.
- If the field is `cache_creation_input_tokens`, it means input tokens written into prompt cache.
- If the field is `cache_deleted_input_tokens`, it means server-confirmed cache deletion.
- If the code says `readFileState`, it is a local file-content cache.
- If the code says `memoize`, `cache.clear`, or "cached" around config/auth/plugins, it is usually not token accounting.
- If a cost function receives usage, inspect whether it includes cache-read and cache-creation tokens in the total.
- If a prompt suggestion or speculation call reuses cache-safe params, inspect whether it sets `skipCacheWrite`.

### Future Map Guidance

For future releases, the high-value map anchors are `query.ts`,
`utils/tokens.ts`, `services/cost-tracker.ts`, `utils/forkedAgent.ts`,
`services/PromptSuggestion/*`, `cli/print.ts`, `utils/toolSearch.ts`, SDK usage
schemas, and any new API response normalizer. The map should preserve the exact
field names because downstream billing and performance docs depend on those
names.
