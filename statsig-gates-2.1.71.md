# Claude Code v2.1.71 — Statsig & GrowthBook Feature Gates

Source bundle: `cli.js` v2.1.71, build `2026-03-06T22:45:36Z`, 612,918 lines.

---

## Overview

This document catalogs all Statsig and GrowthBook runtime feature flags found in Claude Code v2.1.71.

Flags are checked via four minified SDK wrapper functions:

- **`p8(flagName, default)`** — synchronous Statsig gate check (57 unique gates, 83 call-sites)
- **`jU(flagName, default, ttlMs)`** — TTL-cached Statsig gate check (5 gates)
- **`A_(experimentName)`** — GrowthBook cached experiment variant (7 experiments, 11 call-sites)
- **`UL(configName, default)`** — Statsig dynamic config object (4 configs)

**Total unique flag/config names: 73** across 103 call-sites.

---

## Table of Contents

1. [p8() Boolean Gates (57 gates)](#1-p8-boolean-gates-57-gates)
2. [jU() TTL-Cached Gates (5 gates)](#2-ju-ttl-cached-gates-5-gates)
3. [A_() GrowthBook Experiments (7 experiments)](#3-a_-growthbook-experiments-7-experiments)
4. [UL() Dynamic Configs (4 configs)](#4-ul-dynamic-configs-4-configs)
5. [VS Code Experiment Gates](#5-vs-code-experiment-gates)
6. [Summary Statistics](#6-summary-statistics)

---

## 1. p8() Boolean Gates (57 gates)

Total call-sites: 83. The `p8(flagName, defaultValue)` function is the primary Statsig gate checker — `defaultValue` is used when Statsig is unavailable.

### Agent Teams / Background Agents (2 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_amber_flint` | `true` | Guards the agent-teams feature. Returns `false` if off, blocking experimental multi-agent swarm mode even when `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` env var is set. |
| `tengu_auto_background_agents` | `false` | Enables automatic background agent tasks; when on, sets a 120s idle timeout. Also gated by `CLAUDE_AUTO_BACKGROUND_TASKS` env var. |

### Model / Effort / Fast Mode (8 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_grey_wool` | `true` | Controls legacy model remapping (e.g., opus-4 alias resolution). Disabled by `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP`. |
| `tengu_grey_step2` | object | Returns config object for a grey-step-2 mode (merged with default thinking config; likely effort/thinking level experiment). |
| `tengu_marble_sandcastle` | `true` | Fast mode gating: if user is NOT on native binary and this gate is on, blocks Fast mode with an install-prompt message. |
| `tengu_penguins_off` | `null` | Fast mode kill switch: if non-null, the value is used as the "Fast mode unavailable" reason string. |
| `tengu_tangerine_ladder_boost` | `true` | Fast mode: when billing status is `"disabled"` and this gate is on, shows a specific error message (e.g., subscription required). |
| `tengu_turtle_carbon` | `true` | Enables interleaved/adaptive thinking for supported models. Used to determine the `betas` API header. |
| `tengu_amber_prism` | `false` | Appends a suffix to model IDs in enterprise/org context. Likely routes to a special model variant. |
| `tengu_auto_mode_config` | `{}` | Returns config object controlling auto-mode: `model`, `enabled` (`"enabled"`/`"disabled"`/`"opt-in"`), `twoStageClassifier`. |

### Memory & Context Window (8 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_herring_clock` | `false` | Enables team memory directory feature. When on, allows team memdir tracking and fires `tengu_team_memdir_disabled` event if conditions aren't met. |
| `tengu_coral_fern` | `false` | Adds a "Searching past context" section to system prompt with grep instructions for searching memory files and session logs. |
| `tengu_swinburne_dune` | `false` | Switches memory prompt builder: when on, uses typed combined memory prompt; also selects `buildTypedCombinedMemoryPrompt` for team memory. |
| `tengu_paper_halyard` | `false` | When on, skips Project-type and Local-type CLAUDE.md files from the system prompt context block. |
| `tengu_session_memory` | `false` | Enables session memory feature. Also checked paired with `tengu_sm_compact`. |
| `tengu_sm_compact` | `false` | Enables compaction of session memory. Only active when `tengu_session_memory` is also on. |
| `tengu_tight_weave` | `true` | Controls subagent system prompt verbosity: concise reports when on; detailed writeups when off. Also controls file path instruction style. |
| `tengu_pebble_leaf_prune` | `false` | Changes conversation history pruning algorithm to a more sophisticated leaf-pruning approach to remove dead branches in the message tree. |

### Compact / Cache / Streaming (7 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_compact_cache_prefix` | `false` | Enables prompt-cache sharing during compact: tries a cache-prefixed request first, then falls back to normal compact. |
| `tengu_compact_streaming_retry` | `false` | Enables retry logic during streaming compact (retries up to the configured limit on stream errors). |
| `tengu_system_prompt_global_cache` | `false` | Enables global system prompt cache. Splits prompt into sections with `cacheScope: "org"`. |
| `tengu_prompt_cache_1h_config` | `{}` | Config with `allowlist` of model-name prefixes eligible for 1-hour prompt caching (Bedrock path). |
| `tengu_fgts` | `false` | Enables fine-grained tool streaming (`eager_input_streaming: true`). Also gated by `CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING`. |
| `tengu_disable_streaming_to_non_streaming_fallback` | `false` | Prevents fallback from streaming to non-streaming mode on stream errors; throws instead. |
| `tengu_streaming_text` | `false` | Enables streaming text output mode. Also gated by `CLAUDE_CODE_STREAMING_TEXT` env var. |

### Tool Search / TST (6 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_glacier_2xr` | `false` | Changes how deferred tool discovery is announced in system prompt: uses "system-reminder messages" wording. Also used for dynamic tool loading. |
| `tengu_defer_all_bn4` | `true` | When on, marks ALL tools (except ToolSearch itself) as deferred. |
| `tengu_tst_hint_m7r` | `false` | Enables search hints in the deferred tool list (shows `name — searchHint` format). Also gated by `CLAUDE_CODE_SEARCH_HINTS_IN_LIST`. |
| `tengu_basalt_3kr` | `false` | Enables MCP instructions delta feature: sends incremental MCP server instruction changes rather than full instructions. Also gated by `CLAUDE_CODE_MCP_INSTR_DELTA`. |
| `tengu_tool_search_unsupported_models` | `null` | Config array of model name substrings for which tool search is disabled. Falls back to hardcoded default list if null/empty. |
| `tengu_tst_kx7` | `false` | GrowthBook experiment fallback for tool search in tst-auto mode: when below auto-threshold but deferred tools are present, determines whether to enable tool search. |

### MCP / Elicitation / Bridge (7 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_mcp_elicitation` | `false` | Enables MCP elicitation feature. Allows MCP servers to elicit user input via form/URL modes. |
| `tengu_ccr_bridge` | `false` | Enables the Claude Remote Control bridge. Also used as an async blocking gate via `Ck6()`. |
| `tengu_copper_bridge` | `false` | Enables the WebSocket bridge: returns bridge URL (production/staging/local) for Claude in Chrome connection. |
| `tengu_marble_lantern_disabled` | `false` | Disables the marble-lantern feature (appears to relate to fast/opus model availability toggle). |
| `tengu_quartz_lantern` | `false` | In remote mode (`CLAUDE_CODE_REMOTE`), enables diff computation for file write operations. |
| `tengu_remote_backend` | `false` | Controls whether `--remote` CLI flag can be used without a description argument. |
| `tengu_marble_whisper` | `false` | Enables word-highlighting feature: finds and highlights specific words in content. |

### Output / Prompt Style (4 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_sotto_voce` | `false` | Adds an "Output efficiency" section to system prompt instructing Claude to be extra concise and lead with answers. |
| `tengu_bergotte_lantern` | `false` | Controls output polish instruction: when on, uses detailed polished-output guidance; when off, just says "Be short and concise." |
| `tengu_summarize_tool_results` | `false` | Adds instruction to write down important tool result info immediately, since tool results may be cleared. |
| `tengu_attribution_header` | `true` | Enables the `x-anthropic-billing-header` attribution header on API calls. Gated by `CLAUDE_CODE_ATTRIBUTION_HEADER` env. |

### Permissions / Safety (6 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_marble_anvil` | `false` | Enables API context management beta. Also controls `clear_thinking_20251015` edit in thinking models. Only applied when model supports it. |
| `tengu_permission_explainer` | `false` | Enables permission explanation feature: uses an LLM call to explain why a tool permission is needed before asking the user. |
| `tengu_destructive_command_warning` | `false` | Enables destructive command detection in bash tool permission UI, warning users before potentially destructive shell commands. |
| `tengu_granite_whisper` | `false` | When on, skips repo text file size collection — short-circuits git ls-tree scan. |
| `tengu_scarf_coffee` | `false` | Adds a beta header for supported models — likely an interleaved-thinking or input-examples beta. |
| `tengu_quiet_hollow` | `false` | Adds a beta header for hiding thinking summaries (when not in agent mode and user hasn't enabled `showThinkingSummaries`). |

### Tool Behavior (7 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_tool_input_aliasing` | `false` | Enables tool input parameter aliasing: maps deprecated parameter names to current ones using `inputParamAliases` from tool schema. |
| `tengu_chomp_inflection` | `true` | Controls prompt suggestion / chomp feature. When on, shows "Prompt suggestions" toggle in settings. |
| `tengu_plum_vx3` | `false` | In web search tool: when on, disables thinking and forces a specific model/tool-choice for the search sub-request. |
| `tengu_cork_m4q` | `false` | Controls bash permission system prompt format: when on, uses simplified "Command: X" format vs. the full spec-based format. |
| `tengu_lean_cast` | `false` | Switches compact summary system prompt template. |
| `tengu_amber_quartz` | `false` | Enables a feature related to settings-change handling. |
| `tengu_moth_copse` | `false` | Enables auto-memory retrieval: when on, searches for relevant memories based on the user's last message using semantic matching. |

### Tracing / Observability (4 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_trace_lantern` | `false` | Enables detailed beta tracing. Requires `ENABLE_BETA_TRACING_DETAILED` env and `BETA_TRACING_ENDPOINT`. |
| `tengu_cicada_nap_ms` | `0` | Configures a cooldown (ms) for startup prefetch operations. Skips background prefetches if last run was within this window. |
| `tengu_miraculo_the_bard` | `false` | When false, calls startup prefetch function A; when true, calls alternate startup fetch path B. |
| `tengu_miraculo_the_bard2` | `false` | Controls a second separate startup prefetch operation. |

### CI/CD / Version / Process (3 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_pid_based_version_locking` | `false` | Enables PID-based version locking: locks a specific Claude Code version to a process ID to prevent upgrade conflicts. |
| `tengu_immediate_model_command` | `false` | Enables the immediate model command feature. Allows `/model` command to take effect without restarting. |
| `tengu_chrome_auto_enable` | `false` | Auto-enables Claude in Chrome when the native binary is available. |

### Keybinding / UI (3 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_keybinding_customization_release` | `false` | Enables keybinding customization feature. Controls whether custom key binding loading and settings UI are active. |
| `tengu_quiet_fern` | `false` | Pushed to VS Code extension via `experiment_gates` notification. Inferred: quiet/silent mode for VS Code integration. |
| `tengu_slate_ridge` | `false` | Pushed to VS Code extension via `experiment_gates` notification. Inferred: a VS Code UI/layout experiment. |

### PR Status / Git (2 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_pr_status_cli` | `false` | Enables PR status footer in CLI. When on, shows a PR status section in the footer and adds a settings toggle. |
| `tengu_plan_mode_interview_phase` | `false` | Enables interview phase in plan mode: adds an exploration/interview step before implementation. Also gated by `CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE` env. |

---

## 2. jU() TTL-Cached Gates (5 gates)

The `jU(flagName, defaultValue, ttlMs)` function caches gate results for `ttlMs` milliseconds, reducing Statsig calls for frequently-checked gates.

| Flag Name | Default | TTL | Purpose |
|-----------|---------|-----|---------|
| `tengu_kairos_cron` | `false` | 300,000 ms (5 min) | Enables the Kairos cron/scheduling feature: allows creating one-shot and recurring scheduled tasks within a session. |
| `tengu_iron_gate_closed` | `true` | via `aSz` | Controls auto-mode classifier failure behavior: when on (fail-closed), denies with retry guidance when the classifier is unavailable; when off, allows through. |
| `tengu_bridge_poll_interval_config` | object | 300,000 ms (5 min) | Config object for bridge polling intervals: `poll_interval_ms_not_at_capacity`, `poll_interval_ms_at_capacity`, `heartbeat_interval_ms`. |
| `tengu_kairos_cron_config` | object | 60,000 ms (1 min) | Config object for cron jitter: `recurringFrac`, `recurringCapMs`, `oneShotMaxMs`, `oneShotFloorMs`, `oneShotMinuteMod`. |
| `tengu_bridge_initial_history_cap` | `200` | 300,000 ms (5 min) | Maximum number of initial history messages to load when a remote bridge session connects. |

---

## 3. A_() GrowthBook Experiments (7 experiments)

The `A_(experimentName)` function reads from `cachedGrowthBookFeatures[name]` — the primary GrowthBook experiment accessor. Features are re-fetched on auth changes and via periodic light refreshes.

| Experiment Name | Purpose |
|-----------------|----------|
| `tengu_vscode_review_upsell` | Pushed to VS Code extension via `experiment_gates`. Controls whether a review/upsell prompt is shown inside the VS Code integration. |
| `tengu_vscode_onboarding` | Pushed to VS Code extension via `experiment_gates`. Controls the onboarding flow shown to new VS Code users. |
| `tengu_streaming_tool_execution2` | Controls whether tool execution uses streaming. Reported in session gate metadata. |
| `tengu_thinkback` | Guards the `think-back` and `thinkback-play` slash commands. When truthy, enables the "2025 Claude Code Year in Review" thinkback feature. |
| `tengu_disable_bypass_permissions_mode` | Disables `bypassPermissions` mode. When truthy (or org policy set), the mode is unavailable and any existing bypass-permissions session is downgraded to default. |
| `tengu_tool_pear` | When truthy, adds a beta header to API requests for supported models. Also checked when building tool schemas. Likely an API tool-pear beta experiment. |
| `tengu_scratch` | Enables the scratchpad feature. Controls whether the per-session scratchpad directory is made available. |

---

## 4. UL() Dynamic Configs (4 configs)

The `UL(configName, defaultValue)` function fetches a full config object from Statsig (not a boolean gate — returns structured data).

| Config Name | Default | Purpose |
|-------------|---------|----------|
| `tengu_bridge_min_version` | `{ minVersion: "0.0.0" }` | Minimum Claude Code version required for the Remote Control bridge. If current version is older than `minVersion`, shows upgrade prompt. |
| `tengu_1p_event_batch_config` | `{}` | Config for first-party telemetry event batching: `scheduledDelayMillis`, `maxExportBatchSize`, `maxQueueSize`. Controls OTEL log export behavior. |
| `tengu_desktop_upsell` | object | Config for desktop app upsell dialog: contains `enable_startup_dialog` boolean and display params. Shown on macOS/Windows x64 up to 3 times. |
| `tengu_sm_config` | `{}` | Session memory configuration object. Consumed alongside the `tengu_session_memory` gate. |

---

## 5. VS Code Experiment Gates

These four gates are specifically pushed to the `claude-vscode` MCP server as a notification when it connects:

| Gate | Check Function |
|------|----------------|
| `tengu_vscode_review_upsell` | `A_()` GrowthBook |
| `tengu_vscode_onboarding` | `A_()` GrowthBook |
| `tengu_quiet_fern` | `p8()` Statsig |
| `tengu_slate_ridge` | `p8()` Statsig |

---

## 6. Summary Statistics

### Statsig / GrowthBook Feature Flags

| Category | Unique Flags | Call-Sites |
|----------|-------------|------------|
| `p8()` Statsig boolean gates | 57 | 83 |
| `jU()` TTL-cached gates | 5 | 5 |
| `A_()` GrowthBook experiments | 7 | 11 |
| `UL()` dynamic config objects | 4 | 4 |
| **Total unique flag/config names** | **73** | **103** |
