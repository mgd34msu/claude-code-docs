# Undocumented Internals, Codenames, and Dormant Surfaces in Claude Code 2.1.141

This document is intentionally narrower than a feature inventory. It does not treat a normal user-facing capability as "undocumented" just because its implementation is interesting. For this file, "undocumented" means the 2.1.141 source exposes an internal codename, hidden gate, hidden protocol switch, explicit stub, dormant import, unfinished path, or future-facing surface that is not presented as a normal documented product feature.

All content below is based on the reconstructed 2.1.141 source tree, especially `source/src`. Older source maps and older documents were useful only as orientation. The names and behavior described here are taken from the 2.1.141 code, with uncertainty called out where the source only supports inference.

## Scope Rules

Included:

- Runtime codenames such as `tengu_*` values when they are used as GrowthBook flags, GrowthBook configs, hidden rollout switches, or internal experiment names.
- Build-time gates expressed through `feature("...")` when they expose a surface that is hidden, experimental, dormant, or internally named.
- Null tool exports, unavailable-tool stubs, `TBD` comments, and future-looking TODOs that leave an identifiable product surface in the code.
- Hidden CLI flags, SDK/protocol arguments, and internal environment variables that are not normal user-facing commands.
- Telemetry-only names only when they help identify an otherwise hidden subsystem or codename family.

Excluded:

- Normal documented product features unless the undocumented part is the codename, hidden control plane, dormant branch, or future surface behind the feature.
- Public hooks, public MCP behavior, documented `--print` usage, documented output modes, documented slash commands, and documented configuration fields unless a hidden variant or unreleased extension is visible in source.
- General implementation details that are not tied to a hidden gate, codename, stub, or future-facing branch.

## Classification

The same internal name can appear in more than one role. This file uses the following labels:

| Label | Meaning |
| --- | --- |
| `live-gated` | Implementation is present in 2.1.141, but execution depends on a feature flag, GrowthBook value, config object, or hidden environment variable. |
| `build-gated` | Code is compiled in or out through `feature("...")`. The reconstructed source shows the branch but the distributed bundle may contain only the selected side. |
| `dormant-stub` | The name exists as a tool, import, constant, schema, or extension point, but the 2.1.141 code wires it to `null`, an unavailable stub, or a `TBD` path. |
| `internal-protocol` | The surface exists for SDK, bridge, remote, daemon, harness, or internal transport use rather than normal CLI help. |
| `inferred-future` | The code strongly suggests planned or unfinished functionality, but the current source does not expose a complete user-facing path. |
| `telemetry-only` | The name is logged as an event or attribute and should not be treated as a user-controllable feature gate by itself. |

## Important Distinction: `tengu_*` Is Not One Thing

The 2.1.141 source uses many `tengu_*` identifiers. They are not all feature flags.

Some are runtime feature flags or configs:

- `tengu_auto_mode_config`
- `tengu_bridge_repl_v2`
- `tengu_bridge_repl_v2_config`
- `tengu_kairos_brief_config`
- `tengu_sedge_lantern`
- `tengu_sedge_lantern_config`
- `tengu_chrome_auto_enable`

Some are telemetry event names:

- `tengu_agent_created`
- `tengu_agent_tool_completed`
- `tengu_auto_mode_decision`
- `tengu_tree_sitter_load`
- `tengu_permission_request_option_selected`

Some are experiment codenames whose source usage reveals a subsystem but not necessarily a public-facing name:

- `tengu_harbor`
- `tengu_herring_clock`
- `tengu_moth_copse`
- `tengu_passport_quail`
- `tengu_slate_thimble`
- `tengu_paper_halyard`

When updating this file for later versions, classify a `tengu_*` name by usage first. A value passed into GrowthBook or a feature config helper is different from a value sent to telemetry.

## High-Confidence Runtime Codenames and Hidden Configs

These are source-observed codenames whose surrounding implementation gives a reasonably clear meaning.

| Codename | Classification | Source area | What 2.1.141 shows |
| --- | --- | --- | --- |
| `tengu_auto_mode_config` | `live-gated` | Auto permission / classifier path | Runtime config for automatic permission decisions. The source logs auto-mode decisions separately from normal permission prompting. |
| `tengu_bg_classifier_config` | `live-gated` | Background status / async work classification | Configures a classifier used around background work status. It is not a public command surface by itself. |
| `tengu_bridge_repl_v2` | `live-gated` | Bridge REPL / remote bridge | Enables the v2 bridge REPL path. The source explicitly comments that the remote bridge path is gated by this GrowthBook flag. |
| `tengu_bridge_repl_v2_config` | `live-gated` | Bridge REPL / remote bridge | Runtime configuration for bridge REPL v2 behavior. |
| `tengu_bridge_poll_interval_config` | `live-gated` | Remote bridge polling | Controls remote bridge polling cadence. This is an internal transport tuning surface, not a user feature. |
| `tengu_ccr_bridge` | `live-gated` | Claude Code Remote / bridge | Enables CCR bridge behavior. The surrounding code handles session ingress, bridge startup, and remote transport state. |
| `tengu_ccr_bridge_multi_environment` | `live-gated` | Claude Code Remote / environments | Indicates multi-environment support for the CCR bridge path. |
| `tengu_ccr_bridge_multi_session` | `live-gated` | Claude Code Remote / sessions | Indicates multi-session support for the CCR bridge path. |
| `tengu_ccr_mirror` | `live-gated` | Remote mirroring | Controls mirror behavior for remote/CCR session replication. |
| `tengu_ccr_bundle_max_bytes` | `live-gated` | Remote bundle upload | Configures bundle size limits for CCR-related upload or seeding behavior. |
| `tengu_ccr_bundle_seed_enabled` | `live-gated` | Remote bundle upload | Controls whether seed bundles are enabled in the CCR path. |
| `tengu_remote_backend` | `live-gated` | Remote session backend | Selects or configures the remote backend path used by bridge/remote code. |
| `tengu_chrome_auto_enable` | `live-gated` | Claude-in-Chrome setup | Controls automatic enablement of the Chrome integration path. The public-facing integration is not the undocumented part; this codename and rollout switch are. |
| `tengu_copper_bridge` | `live-gated` | Claude-in-Chrome MCP server | Gates the Chrome bridge/MCP server path. The source also has Chrome-specific permission mode environment controls. |
| `tengu_chomp_inflection` | `live-gated` | Prompt suggestion / inflection | Controls an internal prompt-suggestion or prompt-inflection subsystem. This is not just generic prompt UX; the codename identifies the hidden rollout. |
| `tengu_sedge_lantern` | `live-gated` | Away summary | Gates away-summary behavior. This should not be documented as "away summary exists"; the hidden part is the sedge-lantern gate and holdback. |
| `tengu_sedge_lantern_config` | `live-gated` | Away summary | Runtime config for away summary. |
| `tengu_sedge_lantern_holdback` | `live-gated` | Away summary | Holdback gate for the away-summary experiment. |
| `tengu_kairos` | `live-gated` | Kairos family | Top-level Kairos codename family. In 2.1.141 it connects to assistant/proactive/time-based surfaces rather than a single user-facing command. |
| `tengu_kairos_brief` | `live-gated` | Kairos brief | Enables the brief sub-feature inside the Kairos family. |
| `tengu_kairos_brief_config` | `live-gated` | Kairos brief | Runtime config for brief generation or delivery behavior. |
| `tengu_kairos_cron` | `live-gated` | Kairos cron | Enables scheduled/cron-style Kairos behavior. |
| `tengu_kairos_cron_config` | `live-gated` | Kairos cron | Runtime config for Kairos cron behavior. |
| `tengu_kairos_cron_durable` | `live-gated` | Kairos cron persistence | Indicates a durable scheduling path rather than only in-process scheduling. |
| `tengu_kairos_dream` | `live-gated` | Kairos dream | Gates the dream sub-feature inside the Kairos family. |
| `tengu_kairos_push_notifications` | `live-gated` | Push notifications | Gates Kairos-related push notification behavior. |
| `tengu_harbor` | `live-gated` | MCP / notification / channel infrastructure | Harbor appears as a codenamed control surface around notification/channel behavior. The source does not justify treating it as a public feature name. |
| `tengu_harbor_ledger` | `live-gated` | MCP / notification ledger | Indicates persisted or ledgered state for Harbor-related notifications or channel events. |
| `tengu_harbor_permissions` | `live-gated` | MCP / notification permissions | Permission-side control for the Harbor family. |
| `tengu_herring_clock` | `live-gated` | Memory / session timing family | Source placement ties this to memory/session timing behavior. The exact product name is not exposed. |
| `tengu_moth_copse` | `live-gated` | Memory / memdir family | Source placement ties this to memory or memdir-style behavior. Treat the exact meaning as inferred. |
| `tengu_passport_quail` | `live-gated` | Memory / identity / access family | Source placement suggests memory, identity, or access control around memory/session paths. Exact public meaning is not exposed. |
| `tengu_slate_thimble` | `live-gated` | Memory / session family | Source placement suggests session-memory behavior. Exact public meaning is not exposed. |
| `tengu_session_memory` | `live-gated` | Session memory | Direct session-memory gate. |
| `tengu_sm_config` | `live-gated` | Session memory | Runtime config for session memory. |
| `tengu_sm_compact_config` | `live-gated` | Session memory compaction | Runtime config for session-memory compaction behavior. |
| `tengu_paper_halyard` | `live-gated` | Attachment / CLAUDE.md inclusion path | Controls or records project-level memory/attachment inclusion behavior. Source usage is around skip/inclusion decisions, not a public feature name. |
| `tengu_prompt_cache_1h_config` | `live-gated` | Prompt cache | Runtime config for one-hour prompt-cache behavior. |
| `tengu_cache_plum_violet` | `live-gated` | Prompt/cache experiment | Codenamed cache experiment. The source supports cache-path association, but the exact product label is not exposed. |
| `tengu_tool_search_unsupported_models` | `live-gated` | Tool search | Configures model exclusions or unsupported-model handling for tool search. |
| `tengu_tool_pear` | `live-gated` | Tool/tool-search family | Codenamed tool path. The exact mapping should be verified at use sites before treating it as a public feature. |
| `tengu_review_bughunter_config` | `live-gated` | Review / bug hunter | Runtime config for bug-hunting review behavior. |
| `tengu_lodestone_enabled` | `live-gated` | Lodestone | Enables Lodestone integration/background housekeeping. |
| `tengu_version_config` | `live-gated` | Version gating | Runtime version configuration. |
| `tengu_max_version_config` | `live-gated` | Version gating | Runtime max-version configuration. |
| `tengu_tern_alloy` | `telemetry-or-copy-experiment` | Tip/copy registry | Appears around copy/tip experimentation. Treat as UX experiment rather than product capability unless later source reveals more. |
| `tengu_timber_lark` | `telemetry-or-copy-experiment` | Tip/copy registry | Appears around copy/tip experimentation. Treat as UX experiment rather than product capability unless later source reveals more. |
| `tengu_onyx_plover` | `live-gated` | Internal experiment family | Source-observed codename with insufficient public mapping. Keep it in the unknown-codename bucket until a use site proves more. |

## Build-Time Flags With Hidden or Experimental Meaning

The source has many `feature("...")` gates. Some simply select platform or release behavior. The ones below matter for undocumented-feature tracking because they expose codenamed, experimental, internal, or future-facing code paths.

### Agent, Team, and Orchestration Gates

| Build flag | Classification | What it exposes |
| --- | --- | --- |
| `AGENT_TRIGGERS` | `build-gated` | Local scheduled/triggered agent surfaces. This is separate from ordinary subagent execution. |
| `AGENT_TRIGGERS_REMOTE` | `build-gated` | Remote-triggered agent surfaces, including remote trigger tooling. |
| `AGENT_MEMORY_SNAPSHOT` | `build-gated` / `inferred-future` | Agent memory snapshot behavior. The name implies saved state for agent memory rather than normal transcript context. |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | `build-gated` | Built-in explore/plan agent behavior. This is an internal orchestration mode, not a public model setting. |
| `COORDINATOR_MODE` | `build-gated` | Coordinator-mode branch for agent orchestration. |
| `COWORKER_TYPE_TELEMETRY` | `build-gated` | Additional telemetry around coworker/agent type. This is not a user feature by itself. |
| `FORK_SUBAGENT` | `build-gated` | Forked-subagent behavior. Indicates an internal execution topology for agents. |
| `TEAMMEM` | `build-gated` | Team memory path. |
| `VERIFICATION_AGENT` | `build-gated` / `inferred-future` | Verification-agent path. This is especially notable because `VerifyPlanExecutionTool` is null in the main tool registry. |
| `ULTRAPLAN` | `build-gated` | Internal planning mode. |
| `ULTRATHINK` | `build-gated` | Internal thinking/planning mode gate. Do not conflate this with ordinary model reasoning UX. |

### Remote, Daemon, Bridge, and Environment Gates

| Build flag | Classification | What it exposes |
| --- | --- | --- |
| `BRIDGE_MODE` | `build-gated` | Bridge transport mode. |
| `CCR_AUTO_CONNECT` | `build-gated` | Automatic connection behavior for CCR. |
| `CCR_MIRROR` | `build-gated` | CCR mirroring path. |
| `CCR_REMOTE_SETUP` | `build-gated` | Remote setup behavior for CCR. |
| `DAEMON` | `build-gated` | Daemon execution path. |
| `BYOC_ENVIRONMENT_RUNNER` | `build-gated` | Bring-your-own-compute or custom environment runner path. |
| `FILE_PERSISTENCE` | `build-gated` | File persistence behavior for remote/session contexts. |
| `LODESTONE` | `build-gated` | Lodestone integration and background housekeeping. |

### Kairos and Proactive Assistant Gates

| Build flag | Classification | What it exposes |
| --- | --- | --- |
| `KAIROS` | `build-gated` | Top-level Kairos family. The codename is the undocumented part. |
| `KAIROS_BRIEF` | `build-gated` | Brief sub-feature inside Kairos. |
| `KAIROS_CHANNELS` | `build-gated` | Channel or communication path inside Kairos. |
| `KAIROS_DREAM` | `build-gated` | Dream sub-feature inside Kairos. |
| `KAIROS_PUSH_NOTIFICATION` | `build-gated` | Push notification path for Kairos. |
| `PROACTIVE` | `build-gated` | Proactive behavior gate. |

### Permission, Parsing, and Classifier Gates

| Build flag | Classification | What it exposes |
| --- | --- | --- |
| `BASH_CLASSIFIER` | `build-gated` | Classifier path for bash/tool permission decisions. |
| `TRANSCRIPT_CLASSIFIER` | `build-gated` | Classifier path over transcripts. |
| `TREE_SITTER_BASH` | `build-gated` | Tree-sitter Bash parser integration. |
| `TREE_SITTER_BASH_SHADOW` | `build-gated` / `inferred-future` | Shadow parser mode for comparing tree-sitter behavior before fully switching parser behavior. |
| `ANTI_DISTILLATION_CC` | `build-gated` | Anti-distillation control path. The source shows the gate but not a public user surface. |

### Context, Cache, and Compaction Gates

| Build flag | Classification | What it exposes |
| --- | --- | --- |
| `CACHED_MICROCOMPACT` | `build-gated` | Cached micro-compaction behavior. |
| `COMPACTION_REMINDERS` | `build-gated` | Reminder behavior around compaction. |
| `CONTEXT_COLLAPSE` | `build-gated` | Context-collapse tool/path. |
| `PROMPT_CACHE_BREAK_DETECTION` | `build-gated` | Prompt-cache break detection. |
| `REACTIVE_COMPACT` | `build-gated` | Reactive compaction behavior. |
| `TOKEN_BUDGET` | `build-gated` | Token budget behavior. |
| `HISTORY_SNIP` | `build-gated` | Snip/history extraction tool path. |

### MCP, Connector, and Tooling Gates

| Build flag | Classification | What it exposes |
| --- | --- | --- |
| `CHICAGO_MCP` | `build-gated` | Codenamed MCP variant. |
| `MCP_RICH_OUTPUT` | `build-gated` | Rich MCP output support. |
| `MCP_SKILLS` | `build-gated` | MCP-connected skill behavior. |
| `CONNECTOR_TEXT` | `build-gated` | Connector text behavior. |
| `MONITOR_TOOL` | `build-gated` | Monitor tool registration. |
| `TEMPLATES` | `build-gated` | Template surface. |
| `HOOK_PROMPTS` | `build-gated` | Hook prompt behavior beyond plain hook execution. |

### UI, Native, Telemetry, and Miscellaneous Gates

| Build flag | Classification | What it exposes |
| --- | --- | --- |
| `AUTO_THEME` | `build-gated` | Automatic theme behavior. |
| `AWAY_SUMMARY` | `build-gated` | Away-summary code path. The hidden part is the gate and codename family. |
| `DOWNLOAD_USER_SETTINGS` | `build-gated` | User settings download path. |
| `UPLOAD_USER_SETTINGS` | `build-gated` | User settings upload path. |
| `ENHANCED_TELEMETRY_BETA` | `build-gated` | Enhanced telemetry beta. |
| `MESSAGE_ACTIONS` | `build-gated` | Message action UI/control path. |
| `NATIVE_CLIENT_ATTESTATION` | `build-gated` | Native attestation path. |
| `NATIVE_CLIPBOARD_IMAGE` | `build-gated` | Native clipboard image path. |
| `TERMINAL_PANEL` | `build-gated` | Terminal panel UI path. |
| `SHOT_STATS` | `build-gated` | Shot statistics collection. |
| `SLOW_OPERATION_LOGGING` | `build-gated` | Slow-operation logging. |
| `STREAMLINED_OUTPUT` | `build-gated` | Streamlined output path. |
| `UNATTENDED_RETRY` | `build-gated` | Unattended retry behavior. |
| `VOICE_MODE` | `build-gated` | Voice-mode path. |

## Explicit Dormant Tool Surfaces

The main tool registry in 2.1.141 includes several exports that are explicitly wired to `null`. These are higher-signal than a random unused import because the names exist in the central tool surface but are not live tools.

| Tool surface | Classification | What the source shows |
| --- | --- | --- |
| `SubscribePRTool` | `dormant-stub` | A pull-request subscription tool name exists, but the registry assigns it to `null`. |
| `VerifyPlanExecutionTool` | `dormant-stub` / `inferred-future` | A plan verification tool name exists, but the registry assigns it to `null`. This lines up with the separate `VERIFICATION_AGENT` build flag. |
| `OverflowTestTool` | `dormant-stub` | Internal or test-only tool name exists but is null in the registry. |
| `TerminalCaptureTool` | `dormant-stub` | Terminal capture tool name exists but is null in the registry. |
| `WebBrowserTool` | `dormant-stub` | Browser tool name exists but is null in the registry. This is a future-looking surface, not proof of a live browser tool. |
| `WorkflowTool` | `dormant-stub` | Workflow tool name exists but is null in the registry. |

The source also includes `tools/_stubs/createUnavailableTool.ts`. That path matters because some tool names can exist as deliberately unavailable tools rather than disappearing entirely. For future release extraction, do not assume an unavailable tool is dead code; it can be a compatibility placeholder, rollout placeholder, or model/plan-restricted surface.

## Conditional Tool Surfaces Worth Tracking

These are not null, but they are hidden enough that they belong in the undocumented tracker. The important part is the hidden/gated registration, not the plain existence of tools.

| Tool surface | Classification | Gate or context |
| --- | --- | --- |
| `CtxInspectTool` | `build-gated` | Registered through context-inspection feature logic. |
| `SnipTool` | `build-gated` | Tied to `HISTORY_SNIP`. |
| `MonitorTool` | `build-gated` | Tied to `MONITOR_TOOL`. |
| `RemoteTriggerTool` | `build-gated` | Tied to agent trigger and remote trigger behavior. |
| `REPLTool` | `internal-protocol` | Bridge/REPL-related internal tool surface. |
| `PushNotificationTool` | `build-gated` | Tied to Kairos push notification behavior. |
| `PowerShellTool` | `platform-gated` | Platform/toolchain-specific conditional tool surface. |
| `TestingPermissionTool` | `internal-protocol` | Permission-test surface, not a normal user-facing tool. |
| `LSPTool` | `internal-protocol` | Language-server helper surface. The user-facing IDE story is not the undocumented part; this tool wiring is. |

## Async-Agent MCP Stubs

The tool constants include explicit `ENABLE LATER` / `TBD` comments for MCP-related tools in async-agent contexts.

| Surface | Classification | Meaning |
| --- | --- | --- |
| `MCPTool` in async-agent tooling | `dormant-stub` / `inferred-future` | The code acknowledges the MCP tool surface but marks it for later enablement in async agents. |
| `ListMcpResourcesTool` in async-agent tooling | `dormant-stub` / `inferred-future` | Resource listing is identified but not enabled in that context. |
| `ReadMcpResourceTool` in async-agent tooling | `dormant-stub` / `inferred-future` | Resource reading is identified but not enabled in that context. |

This should be tracked separately from ordinary MCP support, which is documented. The undocumented part is the async-agent MCP enablement plan.

## SDK Schema Residue and Future Hook-Like Surfaces

The source schema layer contains names that look like hook or protocol surfaces but are not present as normal hook events in the public hook event list.

| Schema or event-like name | Classification | What to infer |
| --- | --- | --- |
| `PostToolBatch` | `inferred-future` | Schema residue for a post-tool-batch event or protocol shape. Do not document as a live hook unless later source wires it into `HOOK_EVENTS`. |
| `UserPromptExpansion` | `inferred-future` | Schema residue for prompt-expansion behavior. It is not the same as the normal user-prompt-submit hook surface. |

The safe interpretation is that these are protocol/schema preparation or generated residue, not live public hooks in 2.1.141.

## Hidden CLI and Protocol Controls

The source contains a number of CLI options and protocol arguments that are not normal end-user commands. Some may appear in help under specific modes, but their role is internal, SDK, bridge, harness, or transport-oriented.

| Control | Classification | What it is for |
| --- | --- | --- |
| `--sdk-url` | `internal-protocol` | Direct SDK transport URL override. |
| `--workload` | `internal-protocol` | Workload identity/control argument. |
| `--resume-session-at` | `internal-protocol` | Resume a session at a specific point rather than ordinary `--resume` UX. |
| `--rewind-files` | `internal-protocol` | File rewind control tied to session replay/resume behavior. |
| `--system-prompt-file` | `internal-protocol` | System prompt injection from file. |
| `--append-system-prompt-file` | `internal-protocol` | System prompt append injection from file. |
| `--permission-prompt-tool` | `internal-protocol` | Custom permission prompt tool path for harness/SDK control. |
| `--dangerously-load-development-channels` | `internal-protocol` | Loads development channels. The name itself marks this as a nonstandard control. |
| `--channels` | `internal-protocol` | Channel selection path. Track this with the Kairos/channel family rather than as normal CLI UX. |
| `--remote-control-session-name-prefix` | `internal-protocol` | Remote-control session naming control. |
| `--agent-type` | `internal-protocol` | Agent identity/type control used by internal agent execution. |
| `--teammate-mode` | `internal-protocol` | Internal teammate/coworker mode control. |
| `--parent-session-id` | `internal-protocol` | Parent session relationship for subagent or child-session execution. |
| `--file` | `internal-protocol` | File argument path used by nonstandard invocation modes. |
| `--plugin-url` | `internal-protocol` | Plugin URL loading path. |
| `--maintenance` | `internal-protocol` | Maintenance-mode path. |
| `--teleport` | `internal-protocol` | Teleport/remote bridge path. |
| `--remote` | `internal-protocol` | Remote mode path. |

Do not treat documented basics like `--print`, `--output-format`, `--resume`, or public hook flags as undocumented just because they have interesting telemetry. Only hidden or internal variants belong here.

## Internal Environment Variables and Overrides

Environment variables are included here only when they expose hidden behavior, internal transport, debug controls, or nonstandard rollout overrides.

### Feature and Rollout Overrides

| Variable | Classification | Meaning |
| --- | --- | --- |
| `CLAUDE_INTERNAL_FC_OVERRIDES` | `internal-protocol` | Internal feature-control override input. Important for reproducing gated behavior locally. |
| `CLAUDE_CODE_ENABLE_AWAY_SUMMARY` | `live-gated` | Explicit local override for away-summary behavior. |
| `CLAUDE_CODE_ENABLE_CFC` | `live-gated` | Enables Claude-in-Chrome setup behavior. |
| `CLAUDE_CODE_ENABLE_TOKEN_USAGE_ATTACHMENT` | `live-gated` | Enables token-usage attachment behavior. |
| `CLAUDE_CODE_SAVE_HOOK_ADDITIONAL_CONTEXT` | `internal-protocol` | Saves additional hook context. Useful for harness/debug behavior, not normal hook usage. |
| `CLAUDE_CODE_STREAMLINED_OUTPUT` | `live-gated` | Overrides streamlined output behavior. |
| `CLAUDE_CODE_INCLUDE_PARTIAL_MESSAGES` | `internal-protocol` | Includes partial messages in streaming/protocol paths. |

### Remote, Bridge, CCR, and Session Ingress

| Variable | Classification | Meaning |
| --- | --- | --- |
| `CLAUDE_CODE_USE_CCR_V2` | `internal-protocol` | Selects CCR v2 transport behavior. |
| `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2` | `internal-protocol` | Selects POST behavior for session ingress v2. |
| `CLAUDE_CODE_REMOTE` | `internal-protocol` | Marks remote execution behavior. |
| `SESSION_INGRESS_URL` | `internal-protocol` | Session ingress endpoint. |
| `CLAUDE_SESSION_INGRESS_TOKEN_FILE` | `internal-protocol` | Token file for session ingress. |
| `CLAUDE_BRIDGE_*` | `internal-protocol` | Bridge-specific environment family. Exact members should be checked at call sites because they control transport details. |

### Chrome, Voice, Terminal, and Native Debug Paths

| Variable | Classification | Meaning |
| --- | --- | --- |
| `CLAUDE_CHROME_PERMISSION_MODE` | `internal-protocol` | Permission mode override for Claude-in-Chrome bridge behavior. |
| `VOICE_STREAM_BASE_URL` | `live-gated` | Voice streaming base URL override. |
| `CLAUDE_CODE_TERMINAL_RECORDING` | `internal-debug` | Enables terminal/asciicast recording path in internal builds. |
| `CLAUDE_CODE_PROFILE_STARTUP` | `internal-debug` | Startup profiling. |
| `CLAUDE_CODE_PROFILE_QUERY` | `internal-debug` | Query profiling. |
| `CLAUDE_CODE_PERFETTO_TRACE` | `internal-debug` | Perfetto trace output. |
| `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` | `internal-protocol` | Alters MCP tool naming/prefix behavior for Agent SDK contexts. |

## Inferred Future or Unfinished Work

These are the highest-value "undocumented" items because the source reveals planned or partial surfaces without a complete public feature.

### Workflow Tool

`WorkflowTool` exists in the central tool registry but is assigned `null`.

Interpretation:

- A workflow tool surface was intentionally named.
- It is not live in 2.1.141 through the normal tool registry.
- Future versions may either fill this with an actual tool implementation or keep it as a compatibility placeholder.

What not to infer:

- Do not claim Claude Code 2.1.141 has a live workflow tool.
- Do not copy a workflow implementation from another version unless the 2.1.141 source contains it.

### Browser Tool

`WebBrowserTool` exists in the central registry as `null`.

Interpretation:

- The source preserves a browser-tool slot.
- 2.1.141 does not expose a live browser tool through this registry.
- This may be a future planned surface, model-specific stub, or compatibility placeholder.

### Terminal Capture Tool

`TerminalCaptureTool` exists in the central registry as `null`.

Interpretation:

- A terminal-capture surface is named but dormant.
- This is distinct from existing shell/terminal UI or terminal recording debug paths.

### Subscribe PR Tool

`SubscribePRTool` exists in the central registry as `null`.

Interpretation:

- The source exposes a possible future PR subscription surface.
- It is not a live user-facing tool in 2.1.141.

### Plan Verification Tool

`VerifyPlanExecutionTool` exists in the central registry as `null`, while `VERIFICATION_AGENT` exists as a build-time gate.

Interpretation:

- There is a planned or experimental verification-agent path.
- The explicit tool surface is dormant in 2.1.141.
- If a future release activates this, diff both the tool registry and verification-agent feature gate.

### Async-Agent MCP Enablement

MCP-related tools are explicitly marked `TBD` / `ENABLE LATER` in async-agent tooling.

Interpretation:

- Ordinary MCP support exists elsewhere.
- Async-agent MCP tool execution is the hidden unfinished surface.
- Future release validation should check whether `MCPTool`, `ListMcpResourcesTool`, or `ReadMcpResourceTool` become enabled for async agents.

### Post-Tool Batch Schema

`PostToolBatch` appears in schema/protocol code but is not a live hook event in the normal hook event list.

Interpretation:

- The schema is prepared for a batched post-tool phase or protocol event.
- 2.1.141 does not justify documenting this as a public hook.

### User Prompt Expansion Schema

`UserPromptExpansion` appears in schema/protocol code but is not a normal public hook event.

Interpretation:

- The protocol/schema layer has prompt-expansion residue or preparation.
- This is separate from documented prompt hooks.

### Tree-Sitter Bash Shadow Mode

`TREE_SITTER_BASH_SHADOW` appears alongside `TREE_SITTER_BASH`.

Interpretation:

- The shadow gate suggests side-by-side parser validation.
- Shadow mode is likely used to compare tree-sitter parsing against another parser before flipping behavior.
- This is an internal migration path, not a separate user-facing parser option.

### Attachment TODOs

The attachment utilities include TODOs around changed-file attachment generation and upstream computation.

Interpretation:

- The source acknowledges future improvements to attachment computation.
- These are not feature flags by themselves, but they are future-facing implementation notes.

Visible TODO themes:

- Generate attachments when messages are created rather than later.
- Compute attachments while the user types.
- Add offset/limit support for changed-file attachments.
- Compute upstream information more completely.

### API Generalization TODO

The API utility layer includes a TODO to generalize behavior to all tools.

Interpretation:

- A tool-specific API path is intended to become broader.
- The source does not expose a public feature name for this work.

## Codenamed Families That Need Careful Future Diffing

Some codename groups are real and important, but the 2.1.141 source does not expose a clean public name. These should be tracked as families rather than over-named.

### Kairos Family

Names:

- `KAIROS`
- `KAIROS_BRIEF`
- `KAIROS_CHANNELS`
- `KAIROS_DREAM`
- `KAIROS_PUSH_NOTIFICATION`
- `tengu_kairos`
- `tengu_kairos_brief`
- `tengu_kairos_brief_config`
- `tengu_kairos_cron`
- `tengu_kairos_cron_config`
- `tengu_kairos_cron_durable`
- `tengu_kairos_dream`
- `tengu_kairos_push_notifications`

What is source-supported:

- Kairos is a multi-part internal family.
- It includes brief, channels, cron/scheduling, dream, and push notification branches.
- Some parts are build-gated and some are runtime-gated.

What is not source-supported:

- Do not collapse the whole family into a single public feature.
- Do not claim every Kairos component is enabled for every user.

### CCR / Remote / Bridge Family

Names:

- `CCR_AUTO_CONNECT`
- `CCR_MIRROR`
- `CCR_REMOTE_SETUP`
- `BRIDGE_MODE`
- `DAEMON`
- `BYOC_ENVIRONMENT_RUNNER`
- `tengu_ccr_bridge`
- `tengu_ccr_mirror`
- `tengu_ccr_bridge_multi_environment`
- `tengu_ccr_bridge_multi_session`
- `tengu_bridge_repl_v2`
- `tengu_bridge_repl_v2_config`
- `tengu_bridge_poll_interval_config`
- `tengu_remote_backend`

What is source-supported:

- Remote/bridge behavior is a major internal subsystem.
- It has multiple control planes: build flags, GrowthBook gates, CLI protocol arguments, and environment variables.
- V2 transport/ingress behavior is controlled by environment variables and runtime config.

What is not source-supported:

- Do not treat every CCR or bridge branch as generally available user functionality.
- Do not infer a stable external protocol from internal variable names alone.

### Memory / Session Memory / Memdir Family

Names:

- `TEAMMEM`
- `EXTRACT_MEMORIES`
- `AGENT_MEMORY_SNAPSHOT`
- `tengu_session_memory`
- `tengu_sm_config`
- `tengu_sm_compact_config`
- `tengu_herring_clock`
- `tengu_moth_copse`
- `tengu_passport_quail`
- `tengu_slate_thimble`
- `tengu_paper_halyard`

What is source-supported:

- There are multiple memory-related gates, including explicit session memory and extraction paths.
- Some codenames are only inferable through source placement and should remain classified as codenames, not renamed as if they were public features.
- Agent memory snapshotting is a distinct internal path from ordinary saved project memory.

What is not source-supported:

- Do not document every memory codename as a public memory feature.
- Do not assume team memory, session memory, and agent snapshot behavior are the same path.

### Harbor / Channel / Notification Family

Names:

- `tengu_harbor`
- `tengu_harbor_ledger`
- `tengu_harbor_permissions`
- `KAIROS_CHANNELS`
- `KAIROS_PUSH_NOTIFICATION`
- `PushNotificationTool`

What is source-supported:

- Harbor is related to channel/notification/permission infrastructure.
- Ledger and permissions codenames indicate stateful and permissioned behavior.
- Push notifications are gated both by build-time feature flags and runtime Kairos codenames.

What is not source-supported:

- Do not claim Harbor is a public product name.
- Do not claim push notifications are globally enabled.

### Claude-in-Chrome / Copper Bridge Family

Names:

- `tengu_chrome_auto_enable`
- `tengu_copper_bridge`
- `CLAUDE_CODE_ENABLE_CFC`
- `CLAUDE_CHROME_PERMISSION_MODE`

What is source-supported:

- The source has a Chrome setup path and MCP bridge/server path.
- The Chrome bridge has its own permission-mode override.
- Auto-enable behavior is runtime-gated.

What is not source-supported:

- Do not describe the public Chrome integration as undocumented.
- The undocumented part is the hidden enablement and bridge-control layer.

## Unknown or Ambiguous Codenames

These names appear in 2.1.141, but the available source context is not strong enough to assign a precise public meaning without overclaiming.

| Codename | Safe handling |
| --- | --- |
| `tengu_onyx_plover` | Track as a source-observed experiment codename. Assign meaning only after reviewing all use sites in the exact target version. |
| `tengu_tool_pear` | Track as a tool-related codename. Verify whether it controls tool search, tool permission, or another tool subsystem before documenting more specifically. |
| `tengu_coral_fern` | Track with the memory/session family only if the current use site still supports that placement. |
| `tengu_marble_fox` | Track with attachment/token-usage behavior only if the current use site still supports that placement. |
| `tengu_paper_halyard` | Track with project-level attachment or memory inclusion behavior. Avoid stronger naming unless use sites are clearer. |
| `tengu_penguins_off` | Track as an internal kill switch or behavior-disable flag until its exact target is reverified. |

For future releases, these should be grepped directly and classified from local use sites before being carried forward.

## Telemetry Names That Should Not Be Promoted to Features

The source logs many `tengu_*` telemetry events. Some are useful breadcrumbs, but they are not feature gates.

Examples:

- `tengu_agent_created`
- `tengu_agent_tool_completed`
- `tengu_agent_tool_selected`
- `tengu_agent_tool_terminated`
- `tengu_auto_mode_decision`
- `tengu_auto_mode_outcome`
- `tengu_auto_mode_state`
- `tengu_permission_request_option_selected`
- `tengu_tree_sitter_load`
- `tengu_tree_sitter_parse_abort`

Safe handling:

- Use telemetry names to identify subsystem boundaries.
- Do not list telemetry-only events as undocumented features.
- If a telemetry event and a runtime gate share a codename family, document the gate and mention the telemetry only as supporting evidence.

## What Was Removed From the Previous Interpretation

The following categories should not be included in this document unless a hidden codename, stub, or internal control is the actual topic:

- Public hooks such as `PreToolUse`, `PostToolUse`, `Notification`, `Stop`, `SubagentStop`, `PreCompact`, `UserPromptSubmit`, `SessionStart`, and `SessionEnd`.
- Public MCP behavior.
- Public `--print` / `-p` behavior and output modes.
- Public slash commands.
- Public settings fields.
- Public agent/subagent concepts.
- Public IDE, LSP, or terminal UX unless the source exposes a hidden tool or internal protocol branch.

Those areas belong in their own documents. This file should stay focused on hidden source-visible surfaces.

## Future Release Extraction Checklist

When using 2.1.141 as a basis for a later release, use this checklist to avoid both false positives and false negatives.

1. Grep for `feature("` and compare the flag set against the 2.1.141 list.
2. Grep for `tengu_` and classify each new name as runtime gate, config, telemetry, copy experiment, or unknown.
3. Re-open the central tool registry and check every `null` tool assignment.
4. Re-open unavailable-tool stubs and check whether any previously unavailable tool became live.
5. Re-open async-agent tool constants and check whether MCP `TBD` paths were enabled.
6. Re-open schema files and compare hook/protocol event names against the live hook event list.
7. Re-open CLI option definitions and separate public documented flags from hidden protocol flags.
8. Re-open remote, bridge, CCR, daemon, and session ingress files because those areas have both build gates and runtime env controls.
9. Re-open memory/session-memory/memdir services because several codenames are only understandable from source placement.
10. Re-open Chrome bridge files and check `tengu_chrome_auto_enable`, `tengu_copper_bridge`, and permission-mode behavior.
11. Re-open Kairos files and classify each sub-feature separately instead of treating Kairos as one feature.
12. Re-run stale-copy checks to ensure no text or code assertions were carried over from older versions without a 2.1.141 source anchor.

## Bottom Line

The strongest undocumented surfaces in 2.1.141 are not ordinary features. They are the internal control plane around features: codenamed GrowthBook configs, build-time gates, dormant tool slots, hidden bridge/remote protocol arguments, debug environment variables, and schema residue for future hooks or protocol events.

The highest-confidence future-facing items are:

- Dormant tool slots: `WorkflowTool`, `WebBrowserTool`, `TerminalCaptureTool`, `SubscribePRTool`, and `VerifyPlanExecutionTool`.
- Async-agent MCP `TBD` enablement for MCP tool/resource access.
- Kairos subfeatures: brief, cron, channels, dream, and push notification paths.
- CCR/remote bridge v2 paths and multi-session/multi-environment bridge controls.
- Session memory, team memory, memory extraction, and agent memory snapshot gates.
- Tree-sitter Bash shadow mode as an internal parser migration path.
- Schema residue around `PostToolBatch` and `UserPromptExpansion`.
