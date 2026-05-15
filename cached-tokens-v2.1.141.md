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
