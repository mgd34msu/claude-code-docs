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

## Detailed Runtime Control Model

2.1.141 has three distinct control layers:

1. Build-time `feature('NAME')` flags. These can remove imports and entire
   code paths from a build.
2. Runtime boolean/string/object values fetched through GrowthBook/Statsig
   helpers. These control product rollout without rebuilding.
3. Local environment/settings overrides. These can force or block behavior
   before or after a runtime gate is consulted.

The same feature may use more than one layer. For example, cron tools require a
build flag, a runtime gate, and a local environment kill switch.

## Build-Time Feature Flags by Area

Agent and team behavior:

- `FORK_SUBAGENT`
- `COORDINATOR_MODE`
- `BG_SESSIONS`
- `AGENT_TRIGGERS`
- `AGENT_TRIGGERS_REMOTE`
- `TEAMMEM`
- `AGENT_MEMORY_SNAPSHOT`
- `BUILTIN_EXPLORE_PLAN_AGENTS`

Kairos and external surfaces:

- `KAIROS`
- `KAIROS_BRIEF`
- `KAIROS_CHANNELS`
- `KAIROS_DREAM`
- `KAIROS_PUSH_NOTIFICATION`
- `PROACTIVE`

Tools and execution:

- `BASH_CLASSIFIER`
- `CONTEXT_COLLAPSE`
- `HISTORY_SNIP`
- `MONITOR_TOOL`
- `QUICK_SEARCH`
- `TREE_SITTER_BASH`
- `TREE_SITTER_BASH_SHADOW`
- `VOICE_MODE`
- `TERMINAL_PANEL`
- `MCP_SKILLS`
- `MCP_RICH_OUTPUT`

Prompt/context/memory:

- `EXTRACT_MEMORIES`
- `PROMPT_CACHE_BREAK_DETECTION`
- `TOKEN_BUDGET`
- `REACTIVE_COMPACT`
- `CACHED_MICROCOMPACT`
- `COMPACTION_REMINDERS`
- `ULTRAPLAN`
- `ULTRATHINK`

Bridge/remote/daemon:

- `BRIDGE_MODE`
- `CCR_AUTO_CONNECT`
- `CCR_MIRROR`
- `CCR_REMOTE_SETUP`
- `DAEMON`
- `BYOC_ENVIRONMENT_RUNNER`

UI/product:

- `MESSAGE_ACTIONS`
- `AUTO_THEME`
- `STREAMLINED_OUTPUT`
- `TEMPLATES`
- `DOWNLOAD_USER_SETTINGS`
- `UPLOAD_USER_SETTINGS`
- `ENHANCED_TELEMETRY_BETA`

## Runtime Gates and Dynamic Configs by Area

### Agent Teams and Background Agents

- `tengu_amber_flint`: external gate for agent teams.
- `tengu_auto_background_agents`: background-agent behavior.
- `tengu_agent_list_attach`: agent list attachments.
- `tengu_bg_classifier_config`: background classifier dynamic config.

### Auto Mode and Permissions

- `tengu_auto_mode_config`: enabled/disabled/opt-in state, model allowlist, and
  auto-mode behavior config.
- `tengu_disable_bypass_permissions_mode`: disables bypass mode when active.
- `tengu_destructive_command_warning`: destructive command warning controls.
- `tengu_sandbox_disabled_commands`: dynamic disabled command/substrings list.

### Brief, Channels, and Cron

- `tengu_kairos`: broader Kairos product gate.
- `tengu_kairos_brief`: brief entitlement gate.
- `tengu_kairos_brief_config`: slash command and UI behavior config.
- `tengu_harbor`: MCP channel runtime gate.
- `tengu_harbor_ledger`: channel plugin allowlist ledger.
- `tengu_harbor_permissions`: channel permission behavior.
- `tengu_kairos_cron`: scheduled tasks gate.
- `tengu_kairos_cron_durable`: durable scheduled tasks gate.
- `tengu_kairos_cron_config`: jitter/max-age scheduling config.
- `tengu_kairos_dream`: bundled dream skill gate.

### Prompt Suggestion, Speculation, and Dream

- `tengu_chomp_inflection`: prompt suggestion default gate.
- `tengu_onyx_plover`: auto dream thresholds and enablement config.
- `tengu_sedge_lantern`: away summary gate.
- `tengu_sedge_lantern_config`: away summary config.

### Memory and Context

- `tengu_coral_fern`: memory directory loading.
- `tengu_passport_quail`: memory directory filtering.
- `tengu_slate_thimble`: memory path/filter behavior.
- `tengu_moth_copse`: memory directory dynamic behavior.
- `tengu_session_memory`: session memory.
- `tengu_herring_clock`: team memory directory behavior.
- `tengu_verified_vs_assumed`: prompt behavior around evidence/assumptions.
- `tengu_hive_evidence`: prompt evidence behavior.
- `tengu_slim_subagent_claudemd`: subagent CLAUDE.md slimming behavior.

### Tool Behavior and Limits

- `tengu_tool_pear`: remote/tool search related behavior.
- `tengu_amber_json_tools`: JSON tools/betas.
- `tengu_amber_wren`: tool result limits.
- `tengu_read_dedup_killswitch`: file-read dedup behavior.
- `tengu_edit_minimalanchor_jrn`: edit prompt/minimal anchor behavior.
- `tengu_quartz_lantern`: diff computation behavior.
- `tengu_plum_vx3`: web search behavior.
- `tengu_otk_slot_v1`: token budget slot behavior.

### Model, Thinking, and Fast Mode

- `tengu_turtle_carbon`: ultrathink detection/config.
- `tengu_ultraplan_model`: ultraplanning model selection.
- `tengu_ant_model_override`: internal model override behavior.
- `tengu_willow_mode`: cache/cold-context behavior in REPL.
- `tengu_slate_prism`: beta selection behavior.

### Remote, Bridge, CCR, and Voice

- `tengu_remote_backend`: remote backend/TUI selection.
- `tengu_ccr_bridge`: CCR bridge behavior.
- `tengu_ccr_mirror`: CCR mirror behavior.
- `tengu_bridge_repl_v2`: bridge REPL v2.
- `tengu_bridge_system_init`: bridge system init.
- `tengu_cobalt_lantern`: remote/GitHub setup behavior.
- `tengu_surreal_dali`: remote sessions behavior.
- `tengu_amber_quartz_disabled`: voice/Quartz kill switch.

### Telemetry and Experiment Infrastructure

- `tengu_event_sampling_config`: event sampling config.
- `tengu_1p_event_batch_config`: first-party event batching config.
- `tengu_frond_boric`: telemetry sink kill switch.
- `tengu_log_datadog_events`: Datadog logging gate.
- `tengu_trace_lantern`: debug tracing.

### UI, Tips, and IDE

- `tengu_terminal_sidebar`
- `tengu_terminal_panel`
- `tengu_tern_alloy`
- `tengu_timber_lark`
- `tengu_keybinding_customization_release`
- `tengu_vscode_review_upsell`
- `tengu_vscode_onboarding`
- `tengu_vscode_cc_auth`
- `tengu_quiet_fern`

## GrowthBook Identity Inputs

GrowthBook attributes observed in source include:

- stable user/account identifiers.
- session id and device id.
- platform and app version.
- API base URL host.
- organization and account UUID.
- user type.
- subscription type.
- rate limit tier.
- first token time.
- email.
- GitHub Actions metadata.
- release channel.
- entrypoint.

This means gates can be targeted by account, organization, environment,
platform, release channel, or entrypoint.

## Override and Cache Behavior

Runtime gate calls use several helper families:

- cached may-be-stale reads for hot paths.
- cached-with-refresh reads for periodically refreshed values.
- blocking or cached reads for gates that must be current before proceeding.
- dynamic config reads for object-valued settings.

Local overrides can come from:

- environment variables.
- cached global config.
- settings and managed policy.
- test/development injection.

## Risk Notes

- A build-time `feature(...)` flag can make a runtime gate irrelevant if the
  code was removed.
- Runtime gates can be stale by design in hot paths.
- Some gates default true, some default false, and some default to object
  configs; default value is part of the call site, not inherent in the key.
- Managed settings can override GrowthBook ledger-style values for enterprise
  features such as channels.

## Gate Lookup Patterns

2.1.141 uses several GrowthBook helper patterns:

- cached may-be-stale reads for hot paths where blocking would hurt startup or
  typing latency.
- cached-with-refresh reads for gates that should update periodically, such as
  cron kill switches.
- cached-or-blocking checks where an in-flight GrowthBook initialization must be
  awaited before making a security-sensitive decision.
- dynamic config reads for object-valued configuration.
- explicit initialization and refresh after auth/trust changes.

The helper choice is part of the behavior. A gate read with a stale cache can
lag behind a server-side rollback; a blocking gate can delay startup or
interaction but gives a stronger freshness guarantee.

## Dynamic Config Examples

Object-valued configs in 2.1.141 include areas such as:

- auto mode configuration.
- bridge polling and v2 bridge config.
- feedback survey config.
- prompt cache one-hour allowlists.
- session memory config.
- session-memory compaction config.
- background classifier config.
- tool search thresholds and unsupported model maps.
- tool-result storage threshold overrides.
- desktop upsell and emergency tip config.
- cron durable and scheduler kill switch behavior.
- auto-dream thresholds in `tengu_onyx_plover`.

Defaults are supplied at the call site. A missing config does not have a global
meaning without reading that default.

## Build-Time Feature Flags

`feature('...')` calls are stronger than runtime gates because bundling can
dead-code-eliminate entire imports. Important flags visible in 2.1.141 include:

- `AGENT_TRIGGERS`
- `AGENT_TRIGGERS_REMOTE`
- `BASH_CLASSIFIER`
- `BG_SESSIONS`
- `BREAK_CACHE_COMMAND`
- `BRIDGE_MODE`
- `CACHED_MICROCOMPACT`
- `CCR_AUTO_CONNECT`
- `CCR_MIRROR`
- `CHICAGO_MCP`
- `COMMIT_ATTRIBUTION`
- `CONTEXT_COLLAPSE`
- `COORDINATOR_MODE`
- `EXTRACT_MEMORIES`
- `HISTORY_SNIP`
- `KAIROS`
- `KAIROS_BRIEF`
- `KAIROS_CHANNELS`
- `KAIROS_DREAM`
- `LODESTONE`
- `MONITOR_TOOL`
- `NATIVE_CLIENT_ATTESTATION`
- `PROACTIVE`
- `PROMPT_CACHE_BREAK_DETECTION`
- `REACTIVE_COMPACT`
- `TEAMMEM`
- `TOKEN_BUDGET`
- `TRANSCRIPT_CLASSIFIER`
- `UNATTENDED_RETRY`
- `UPLOAD_USER_SETTINGS`
- `VOICE_MODE`

Some strings are feature-flag names; others are runtime gate keys. Do not merge
the two categories when building a source map.

## High-Risk Gate Areas

Some gates affect safety or data movement and deserve extra scrutiny:

- `TRANSCRIPT_CLASSIFIER`: permission classifier and auto-mode behavior.
- `BASH_CLASSIFIER`: shell classifier behavior.
- `KAIROS_CHANNELS`: inbound channel context injection.
- `BRIDGE_MODE`: remote bridge and control surfaces.
- `CCR_MIRROR`: mirrored remote session behavior.
- `TEAMMEM`: team memory path handling and secret guards.
- `PROMPT_CACHE_BREAK_DETECTION`: prompt-cache diagnostics.
- `NATIVE_CLIENT_ATTESTATION`: attestation header behavior.
- cron gates: unattended scheduled prompt execution.
- tool search configs: whether tool schemas are deferred or sent.

These gates can change security posture, not just UI visibility.

## Identity and Trust Inputs

GrowthBook targeting can depend on:

- user UUID.
- organization UUID.
- subscription/account type.
- authenticated versus unauthenticated state.
- workspace trust state.
- provider/backend.
- entrypoint.
- platform.
- internal/external user type.
- remote versus local session state.

The source refreshes GrowthBook after login/trust changes because those inputs
alter which gates should apply.

## Future Gate Extraction Checklist

For a later version:

1. Search for `feature('`.
2. Search for `getFeatureValue_`, `checkGate_`, and `useDynamicConfig`.
3. Search for literal `tengu_` config/gate names.
4. Separate build-time feature flags from runtime keys.
5. Record default values at each call site.
6. Record refresh intervals and stale/blocking helper choice.
7. Record whether telemetry/network disablement affects availability.
8. Record managed setting overrides.
9. Record auth/trust refresh behavior.
10. Diff high-risk gate areas manually, not only by string name.
