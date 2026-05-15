# Feature Gates and GrowthBook/Statsig Controls in Claude Code 2.1.141

2.1.141 uses two kinds of feature control:

- build-time feature flags through `feature('NAME')`.
- runtime GrowthBook/Statsig values such as `tengu_*` feature gates and dynamic
  configs.

Runtime access is centralized mostly in `source/src/services/analytics/growthbook.ts`.

## Build-Time Feature Flags Observed

Examples of build-time `feature(...)` flags in 2.1.141:

- `AGENT_TRIGGERS`
- `AGENT_TRIGGERS_REMOTE`
- `BASH_CLASSIFIER`
- `BG_SESSIONS`
- `BRIDGE_MODE`
- `CCR_AUTO_CONNECT`
- `CCR_MIRROR`
- `CONTEXT_COLLAPSE`
- `COORDINATOR_MODE`
- `EXTRACT_MEMORIES`
- `FORK_SUBAGENT`
- `HISTORY_SNIP`
- `KAIROS`
- `KAIROS_BRIEF`
- `KAIROS_CHANNELS`
- `KAIROS_DREAM`
- `KAIROS_PUSH_NOTIFICATION`
- `MCP_SKILLS`
- `MESSAGE_ACTIONS`
- `MONITOR_TOOL`
- `PROACTIVE`
- `PROMPT_CACHE_BREAK_DETECTION`
- `QUICK_SEARCH`
- `STREAMLINED_OUTPUT`
- `TEAMMEM`
- `TEMPLATES`
- `TERMINAL_PANEL`
- `TOKEN_BUDGET`
- `TRANSCRIPT_CLASSIFIER`
- `ULTRAPLAN`
- `ULTRATHINK`
- `VOICE_MODE`

These flags can dead-code-eliminate imports and whole tool surfaces.

## Runtime Gate Families

Agent teams:

- `tengu_amber_flint`

Auto mode and permissions:

- `tengu_auto_mode_config`
- `tengu_disable_bypass_permissions_mode`
- `tengu_destructive_command_warning`
- `tengu_sandbox_disabled_commands`

Brief/Kairos:

- `tengu_kairos`
- `tengu_kairos_brief`
- `tengu_kairos_brief_config`
- `tengu_kairos_dream`
- `tengu_kairos_cron`
- `tengu_kairos_cron_durable`
- `tengu_kairos_cron_config`

Channels:

- `tengu_harbor`
- `tengu_harbor_ledger`
- `tengu_harbor_permissions`

Prompt suggestion/speculation:

- `tengu_chomp_inflection`

Dream:

- `tengu_onyx_plover`

Memory:

- `tengu_coral_fern`
- `tengu_passport_quail`
- `tengu_slate_thimble`
- `tengu_moth_copse`
- `tengu_session_memory`
- `tengu_herring_clock`

Telemetry:

- `tengu_event_sampling_config`
- `tengu_1p_event_batch_config`
- `tengu_frond_boric`
- `tengu_log_datadog_events`

Tools/model/response behavior:

- `tengu_tool_pear`
- `tengu_amber_json_tools`
- `tengu_amber_wren`
- `tengu_read_dedup_killswitch`
- `tengu_edit_minimalanchor_jrn`
- `tengu_quartz_lantern`
- `tengu_willow_mode`
- `tengu_ultraplan_model`
- `tengu_turtle_carbon`

Remote/bridge:

- `tengu_remote_backend`
- `tengu_ccr_bridge`
- `tengu_ccr_mirror`
- `tengu_cobalt_lantern`
- `tengu_surreal_dali`

UI/tips/sidebar:

- `tengu_terminal_sidebar`
- `tengu_terminal_panel`
- `tengu_sedge_lantern`
- `tengu_sedge_lantern_config`
- `tengu_tern_alloy`
- `tengu_timber_lark`

## GrowthBook Enablement

GrowthBook is disabled when:

- `DISABLE_GROWTHBOOK` is set.
- first-party event logging is disabled by privacy/telemetry controls.

User attributes include session id, device id, platform, API base host,
organization/account ids, user type, subscription type, rate-limit tier, email,
app version, GitHub Actions metadata, release channel, and entrypoint.

## Dynamic Configs

Dynamic configs are used where a boolean gate is not enough. Examples:

- auto-mode config.
- cron jitter/max-age config.
- telemetry event sampling.
- 1P event batching.
- sandbox disabled command lists.
- auto dream thresholds.
- away summary config.
- background classifier config.

## 2.1.141 Source Index

- `source/src/services/analytics/growthbook.ts`
- `source/src/utils/betas.ts`
- `source/src/tools.ts`
- `source/src/utils/permissions/permissionSetup.ts`
- `source/src/services/mcp/channelAllowlist.ts`
- `source/src/tools/ScheduleCronTool/prompt.ts`
- `source/src/services/autoDream/config.ts`
- `source/src/services/PromptSuggestion/promptSuggestion.ts`
