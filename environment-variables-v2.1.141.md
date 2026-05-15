# Environment Variables in Claude Code 2.1.141

This reference is source-derived from 2.1.141. It focuses on variables that
affect runtime behavior, authentication, provider selection, telemetry, tools,
plugins, hooks, remote sessions, and hidden feature paths.

## Boolean Parsing

Boolean helpers live in `source/src/utils/envUtils.ts`.

Truthy values:

- `1`
- `true`
- `yes`
- `on`

Explicit false values:

- `0`
- `false`
- `no`
- `off`

Many flags are three-state:

- unset: use default/gate/settings.
- truthy: force enable.
- explicit false: force disable.

## Core Paths and Runtime Identity

- `CLAUDE_CONFIG_DIR` - base config directory instead of `~/.claude`.
- `CLAUDE_CODE_SESSION_ID` - current session id.
- `CLAUDE_CODE_SESSION_NAME` - display/session name.
- `CLAUDE_CODE_SESSION_KIND` - session class.
- `CLAUDE_CODE_ENTRYPOINT` - entrypoint identifier.
- `CLAUDE_CODE_PATH` - Claude Code binary/path context.
- `CLAUDE_CODE_SIMPLE` - simple/bare mode control.
- `CLAUDE_CODE_REPL` - REPL mode marker.
- `CLAUDE_REPL_MODE` / `CLAUDE_REPL_VARIANT` - REPL variants.
- `CLAUDE_CODE_SHELL` - shell override.
- `CLAUDE_CODE_SHELL_PREFIX` - shell command prefix.
- `CLAUDE_CODE_TMPDIR` / `CLAUDE_TMPDIR` - temp directory selection.
- `CLAUDE_CODE_OVERRIDE_DATE` - date override for deterministic/testing flows.
- `CLAUDE_CODE_DONT_INHERIT_ENV` - environment inheritance control.

## Authentication and API

- `ANTHROPIC_API_KEY`
- `ANTHROPIC_AUTH_TOKEN`
- `ANTHROPIC_BASE_URL`
- `ANTHROPIC_CUSTOM_HEADERS`
- `ANTHROPIC_BETAS`
- `ANTHROPIC_UNIX_SOCKET`
- `CLAUDE_CODE_API_BASE_URL`
- `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR`
- `CLAUDE_CODE_API_KEY_HELPER_TTL_MS`
- `CLAUDE_CODE_OAUTH_TOKEN`
- `CLAUDE_CODE_OAUTH_REFRESH_TOKEN`
- `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR`
- `CLAUDE_CODE_OAUTH_CLIENT_ID`
- `CLAUDE_CODE_OAUTH_SCOPES`
- `CLAUDE_CODE_CUSTOM_OAUTH_URL`
- `CLAUDE_CODE_SESSION_ACCESS_TOKEN`
- `CLAUDE_TRUSTED_DEVICE_TOKEN`
- `USE_LOCAL_OAUTH`
- `USE_STAGING_OAUTH`
- `CLAUDE_LOCAL_OAUTH_API_BASE`
- `CLAUDE_LOCAL_OAUTH_APPS_BASE`
- `CLAUDE_LOCAL_OAUTH_CONSOLE_BASE`

## Provider Selection

Bedrock:

- `CLAUDE_CODE_USE_BEDROCK`
- `ANTHROPIC_BEDROCK_BASE_URL`
- `BEDROCK_BASE_URL`
- `AWS_BEARER_TOKEN_BEDROCK`
- `AWS_REGION`
- `AWS_DEFAULT_REGION`
- `AWS_PROFILE`
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_CONTAINER_CREDENTIALS_FULL_URI`
- `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI`
- `CLAUDE_CODE_SKIP_BEDROCK_AUTH`
- `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION`

Vertex:

- `CLAUDE_CODE_USE_VERTEX`
- `ANTHROPIC_VERTEX_PROJECT_ID`
- `GOOGLE_APPLICATION_CREDENTIALS`
- `GOOGLE_CLOUD_PROJECT`
- `GCLOUD_PROJECT`
- `CLOUDSDK_CONFIG`
- `CLOUD_ML_REGION`
- `VERTEX_BASE_URL`
- `CLAUDE_CODE_SKIP_VERTEX_AUTH`
- per-model Vertex region overrides such as
  `VERTEX_REGION_CLAUDE_4_6_SONNET`, `VERTEX_REGION_CLAUDE_4_5_SONNET`,
  `VERTEX_REGION_CLAUDE_4_1_OPUS`, and related legacy model variables.

Foundry:

- `CLAUDE_CODE_USE_FOUNDRY`
- `ANTHROPIC_FOUNDRY_API_KEY`
- `ANTHROPIC_FOUNDRY_BASE_URL`
- `ANTHROPIC_FOUNDRY_RESOURCE`
- `CLAUDE_CODE_SKIP_FOUNDRY_AUTH`

## Model Overrides

- `ANTHROPIC_MODEL`
- `ANTHROPIC_SMALL_FAST_MODEL`
- `ANTHROPIC_DEFAULT_HAIKU_MODEL`
- `ANTHROPIC_DEFAULT_SONNET_MODEL`
- `ANTHROPIC_DEFAULT_OPUS_MODEL`
- `ANTHROPIC_CUSTOM_MODEL_OPTION`
- `CLAUDE_CODE_SUBAGENT_MODEL`
- `CLAUDE_CODE_AUTO_MODE_MODEL`
- `CLAUDE_CODE_MAX_OUTPUT_TOKENS`
- `CLAUDE_CODE_MAX_CONTEXT_TOKENS`
- `MAX_THINKING_TOKENS`
- `CLAUDE_CODE_EFFORT_LEVEL`
- `CLAUDE_CODE_ALWAYS_ENABLE_EFFORT`
- `CLAUDE_CODE_DISABLE_THINKING`
- `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING`
- `DISABLE_INTERLEAVED_THINKING`
- `CLAUDE_CODE_DISABLE_1M_CONTEXT`

## Telemetry and Privacy

- `DISABLE_TELEMETRY`
- `DISABLE_ERROR_REPORTING`
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`
- `DISABLE_GROWTHBOOK`
- `CLAUDE_CODE_ENABLE_TELEMETRY`
- `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA`
- `ENABLE_ENHANCED_TELEMETRY_BETA`
- `ANT_CLAUDE_CODE_METRICS_ENDPOINT`
- `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS`
- `CLAUDE_CODE_OTEL_FLUSH_TIMEOUT_MS`
- `CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS`
- `ANT_OTEL_EXPORTER_OTLP_ENDPOINT`
- `ANT_OTEL_EXPORTER_OTLP_HEADERS`
- `ANT_OTEL_EXPORTER_OTLP_PROTOCOL`
- `OTEL_EXPORTER_OTLP_ENDPOINT`
- `OTEL_EXPORTER_OTLP_HEADERS`
- `OTEL_LOGS_EXPORTER`
- `OTEL_METRICS_EXPORTER`
- `OTEL_TRACES_EXPORTER`
- `OTEL_LOG_USER_PROMPTS`
- `OTEL_LOG_TOOL_CONTENT`
- `OTEL_LOG_TOOL_DETAILS`

## Tool and Feature Toggles

- `ENABLE_TOOL_SEARCH`
- `ENABLE_LSP_TOOL`
- `CLAUDE_CODE_USE_POWERSHELL_TOOL`
- `CLAUDE_CODE_ENABLE_TASKS`
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`
- `CLAUDE_CODE_BRIEF`
- `CLAUDE_CODE_BRIEF_UPLOAD`
- `CLAUDE_CODE_DISABLE_CRON`
- `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION`
- `CLAUDE_CODE_DISABLE_AGENT_VIEW`
- `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS`
- `CLAUDE_CODE_DISABLE_AUTO_MEMORY`
- `CLAUDE_CODE_DISABLE_CLAUDE_MDS`
- `CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS`
- `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK`
- `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING`
- `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS`
- `CLAUDE_CODE_DISABLE_FAST_MODE`
- `CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING`
- `CLAUDE_CODE_ENABLE_TOKEN_USAGE_ATTACHMENT`
- `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`
- `BASH_MAX_OUTPUT_LENGTH`
- `TASK_MAX_OUTPUT_LENGTH`
- `MAX_MCP_OUTPUT_TOKENS`
- `MCP_TIMEOUT`
- `MCP_TOOL_TIMEOUT`

## Plugins, Skills, and MCP

- `CLAUDE_CODE_PLUGIN_CACHE_DIR`
- `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS`
- `CLAUDE_CODE_PLUGIN_SEED_DIR`
- `CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE`
- `CLAUDE_CODE_SYNC_PLUGIN_INSTALL`
- `CLAUDE_CODE_SYNC_PLUGIN_INSTALL_TIMEOUT_MS`
- `FORCE_AUTOUPDATE_PLUGINS`
- `CLAUDE_AGENT_SDK_MCP_NO_PREFIX`
- `ENABLE_CLAUDEAI_MCP_SERVERS`
- `ENABLE_MCP_LARGE_OUTPUT_FILES`
- `MCP_SERVER_CONNECTION_BATCH_SIZE`
- `MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE`
- `MCP_CLIENT_SECRET`
- `MCP_OAUTH_CALLBACK_PORT`
- `MCP_OAUTH_CLIENT_METADATA_URL`
- `MCP_XAA_IDP_CLIENT_SECRET`

## Hooks and Subprocesses

- `CLAUDE_CODE_SAVE_HOOK_ADDITIONAL_CONTEXT`
- `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`
- `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB`
- `CLAUDE_CODE_DONT_INHERIT_ENV`
- `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR`
- `CLAUDE_ENV_FILE`
- `SHELL`
- `PATH`
- `PWD`
- `HOME`

Hook subprocesses also receive Claude-provided context variables; see
`claude-hooks-reference-v2.1.141.md` for exact hook execution contracts.

## Remote, Bridge, CCR, and Daemon

- `CLAUDE_CODE_REMOTE`
- `CLAUDE_CODE_REMOTE_ENVIRONMENT_TYPE`
- `CLAUDE_CODE_REMOTE_SESSION_ID`
- `CLAUDE_CODE_REMOTE_MEMORY_DIR`
- `CLAUDE_CODE_REMOTE_SEND_KEEPALIVES`
- `SESSION_INGRESS_URL`
- `CLAUDE_SESSION_INGRESS_TOKEN_FILE`
- `CLAUDE_BRIDGE_BASE_URL`
- `CLAUDE_BRIDGE_OAUTH_TOKEN`
- `CLAUDE_BRIDGE_SESSION_INGRESS_URL`
- `CLAUDE_BRIDGE_USE_CCR_V2`
- `CCR_ENABLE_BUNDLE`
- `CCR_FORCE_BUNDLE`
- `CCR_EGRESS_GATEWAY_ENABLED`
- `CCR_UPSTREAM_PROXY_ENABLED`
- `CLAUDE_CODE_CCR_MIRROR`
- `CLAUDE_DAEMON_JSON_PATH`

## Terminal and UI Detection

- `TERM`
- `TERM_PROGRAM`
- `TERM_PROGRAM_VERSION`
- `COLORTERM`
- `COLORFGBG`
- `TMUX`
- `TMUX_PANE`
- `STY`
- `WT_SESSION`
- `ITERM_SESSION_ID`
- `KITTY_WINDOW_ID`
- `KONSOLE_VERSION`
- `VTE_VERSION`
- `ALACRITTY_LOG`
- `ZED_TERM`
- `VSCODE_GIT_ASKPASS_MAIN`
- `CURSOR_TRACE_ID`
- `__CFB`
- `CLAUDE_CODE_DISABLE_MOUSE`
- `CLAUDE_CODE_DISABLE_MOUSE_CLICKS`
- `CLAUDE_CODE_DISABLE_TERMINAL_TITLE`
- `CLAUDE_CODE_SCROLL_SPEED`
- `CLAUDE_CODE_TMUX_PREFIX`
- `CLAUDE_CODE_TMUX_SESSION`

## CI and Hosting Detection

- `CI`
- `GITHUB_ACTIONS`
- `GITHUB_REPOSITORY`
- `GITHUB_EVENT_NAME`
- `GITLAB_CI`
- `BUILDKITE`
- `CIRCLECI`
- `CODESPACES`
- `GITPOD_WORKSPACE_ID`
- `REPL_ID`
- `REPL_SLUG`
- `PROJECT_DOMAIN`
- `VERCEL`
- `RAILWAY_ENVIRONMENT_NAME`
- `RAILWAY_SERVICE_NAME`
- `RENDER`
- `NETLIFY`
- `DYNO`
- `FLY_APP_NAME`
- `FLY_MACHINE_ID`
- `CF_PAGES`
- `DENO_DEPLOYMENT_ID`
- `KUBERNETES_SERVICE_HOST`
- `AWS_LAMBDA_FUNCTION_NAME`
- `AZURE_FUNCTIONS_ENVIRONMENT`

## Debug, Test, and Development

- `NODE_ENV`
- `DEBUG`
- `CLAUDE_DEBUG`
- `DEBUG_SDK`
- `CLAUDE_CODE_DEBUG_LOG_LEVEL`
- `CLAUDE_CODE_DEBUG_LOGS_DIR`
- `CLAUDE_CODE_DEBUG_REPAINTS`
- `CLAUDE_CODE_PROFILE_STARTUP`
- `CLAUDE_CODE_PROFILE_QUERY`
- `CLAUDE_CODE_PERFETTO_TRACE`
- `CLAUDE_CODE_FRAME_TIMING_LOG`
- `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER`
- `CLAUDE_CODE_STALL_TIMEOUT_MS_FOR_TESTING`
- `CLAUDE_CODE_TEST_FIXTURES_ROOT`
- `TEST_ENABLE_SESSION_PERSISTENCE`
- `FORCE_VCR`
- `VCR_RECORD`

## 2.1.141 Source Index

- `source/src/utils/envUtils.ts`
- `source/src/utils/env.ts`
- `source/src/main.tsx`
- `source/src/services/analytics/config.ts`
- `source/src/utils/privacyLevel.ts`
- `source/src/utils/model/providers.ts`
- `source/src/services/mcp`
- `source/src/tools`

## Detailed Variable Semantics

This section expands the reference into behavior groups. The complete observed
set is large, so variables are grouped by what they control rather than by file.

## Provider Decision Variables

Provider selection is not a single flag. The runtime considers explicit Claude
Code provider flags, provider-specific credentials, cloud SDK environment, and
skip-auth flags.

First-party/API:

- `ANTHROPIC_API_KEY`: direct API key.
- `ANTHROPIC_AUTH_TOKEN`: auth token path.
- `ANTHROPIC_BASE_URL`: base URL override.
- `ANTHROPIC_CUSTOM_HEADERS`: custom request headers.
- `ANTHROPIC_UNIX_SOCKET`: Unix socket transport.
- `CLAUDE_CODE_API_BASE_URL`: Claude Code API base override.
- `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR`: file-descriptor based key injection.
- `CLAUDE_CODE_API_KEY_HELPER_TTL_MS`: key helper cache lifetime.

Bedrock:

- `CLAUDE_CODE_USE_BEDROCK`: select Bedrock provider path.
- `ANTHROPIC_BEDROCK_BASE_URL` and `BEDROCK_BASE_URL`: endpoint overrides.
- `AWS_BEARER_TOKEN_BEDROCK`: bearer token auth path.
- `AWS_REGION`, `AWS_DEFAULT_REGION`, `AWS_PROFILE`: AWS selection.
- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`: credential environment.
- `AWS_CONTAINER_CREDENTIALS_FULL_URI`,
  `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI`: container credential discovery.
- `CLAUDE_CODE_SKIP_BEDROCK_AUTH`: bypass Bedrock auth preflight.

Vertex:

- `CLAUDE_CODE_USE_VERTEX`: select Vertex provider path.
- `ANTHROPIC_VERTEX_PROJECT_ID`: project override.
- `GOOGLE_APPLICATION_CREDENTIALS`: service account path.
- `GOOGLE_CLOUD_PROJECT`, `GCLOUD_PROJECT`: project discovery.
- `CLOUDSDK_CONFIG`: gcloud config root.
- `CLOUD_ML_REGION`: default Vertex region, defaulting to `us-east5`.
- `VERTEX_BASE_URL`: endpoint override.
- `CLAUDE_CODE_SKIP_VERTEX_AUTH`: bypass Vertex auth preflight.

Foundry:

- `CLAUDE_CODE_USE_FOUNDRY`: select Foundry provider path.
- `ANTHROPIC_FOUNDRY_API_KEY`
- `ANTHROPIC_FOUNDRY_BASE_URL`
- `ANTHROPIC_FOUNDRY_RESOURCE`
- `CLAUDE_CODE_SKIP_FOUNDRY_AUTH`

## Model and Thinking Controls

Model overrides:

- `ANTHROPIC_MODEL`
- `ANTHROPIC_SMALL_FAST_MODEL`
- `ANTHROPIC_DEFAULT_HAIKU_MODEL`
- `ANTHROPIC_DEFAULT_SONNET_MODEL`
- `ANTHROPIC_DEFAULT_OPUS_MODEL`
- `ANTHROPIC_CUSTOM_MODEL_OPTION`
- `CLAUDE_CODE_SUBAGENT_MODEL`
- `CLAUDE_CODE_AUTO_MODE_MODEL`

Model option display variables:

- `ANTHROPIC_DEFAULT_HAIKU_MODEL_NAME`
- `ANTHROPIC_DEFAULT_HAIKU_MODEL_DESCRIPTION`
- `ANTHROPIC_DEFAULT_SONNET_MODEL_NAME`
- `ANTHROPIC_DEFAULT_SONNET_MODEL_DESCRIPTION`
- `ANTHROPIC_DEFAULT_OPUS_MODEL_NAME`
- `ANTHROPIC_DEFAULT_OPUS_MODEL_DESCRIPTION`
- `ANTHROPIC_CUSTOM_MODEL_OPTION_NAME`
- `ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION`

Thinking/context controls:

- `MAX_THINKING_TOKENS`
- `CLAUDE_CODE_EFFORT_LEVEL`
- `CLAUDE_CODE_ALWAYS_ENABLE_EFFORT`
- `CLAUDE_CODE_DISABLE_THINKING`
- `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING`
- `DISABLE_INTERLEAVED_THINKING`
- `CLAUDE_CODE_DISABLE_1M_CONTEXT`
- `CLAUDE_CODE_MAX_CONTEXT_TOKENS`
- `CLAUDE_CODE_MAX_OUTPUT_TOKENS`
- `API_MAX_INPUT_TOKENS`
- `API_TARGET_INPUT_TOKENS`
- `API_TIMEOUT_MS`
- `CLAUDE_CODE_MAX_RETRIES`

## Core Runtime and Session Variables

- `CLAUDE_CONFIG_DIR`: config root.
- `CLAUDE_CODE_SESSION_ID`: session id.
- `CLAUDE_CODE_SESSION_NAME`: human-readable session name.
- `CLAUDE_CODE_SESSION_KIND`: session category.
- `CLAUDE_CODE_ENTRYPOINT`: entrypoint used for telemetry and behavior.
- `CLAUDE_CODE_ACTION`: action context.
- `CLAUDE_CODE_PATH`: binary/path context.
- `CLAUDE_CODE_SIMPLE`: bare/simple mode.
- `CLAUDE_CODE_REPL`: REPL marker.
- `CLAUDE_REPL_MODE`, `CLAUDE_REPL_VARIANT`: REPL variants.
- `CLAUDE_CODE_TMPDIR`, `CLAUDE_TMPDIR`: temp root selection.
- `CLAUDE_CODE_OVERRIDE_DATE`: deterministic date override.
- `CLAUDE_CODE_DONT_INHERIT_ENV`: subprocess inheritance behavior.
- `CLAUDE_CODE_RESUME_INTERRUPTED_TURN`: resume interrupted turn.
- `CLAUDE_CODE_SKIP_PROMPT_HISTORY`: prompt history suppression.
- `CLAUDE_CODE_JSONL_TRANSCRIPT`: JSONL transcript behavior.
- `CLAUDE_CODE_INCLUDE_PARTIAL_MESSAGES`: partial message inclusion.

## Tool and Execution Limits

- `BASH_MAX_OUTPUT_LENGTH`
- `TASK_MAX_OUTPUT_LENGTH`
- `MAX_MCP_OUTPUT_TOKENS`
- `MAX_STRUCTURED_OUTPUT_RETRIES`
- `SLASH_COMMAND_TOOL_CHAR_BUDGET`
- `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS`
- `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`
- `CLAUDE_CODE_GLOB_HIDDEN`
- `CLAUDE_CODE_GLOB_NO_IGNORE`
- `CLAUDE_CODE_GLOB_TIMEOUT_SECONDS`
- `CLAUDE_CODE_USE_NATIVE_FILE_SEARCH`
- `USE_BUILTIN_RIPGREP`
- `EMBEDDED_SEARCH_TOOLS`
- `USE_API_CLEAR_TOOL_RESULTS`
- `USE_API_CLEAR_TOOL_USES`
- `USE_API_CONTEXT_MANAGEMENT`

## Feature Enablement and Disablement

Enable:

- `ENABLE_TOOL_SEARCH`
- `ENABLE_LSP_TOOL`
- `ENABLE_CLAUDEAI_MCP_SERVERS`
- `ENABLE_MCP_LARGE_OUTPUT_FILES`
- `ENABLE_SESSION_PERSISTENCE`
- `ENABLE_PROMPT_CACHING_1H_BEDROCK`
- `CLAUDE_CODE_ENABLE_TASKS`
- `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION`
- `CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING`
- `CLAUDE_CODE_ENABLE_TOKEN_USAGE_ATTACHMENT`
- `CLAUDE_CODE_ENABLE_EXPERIMENTAL_ADVISOR_TOOL`
- `CLAUDE_CODE_ENABLE_AWAY_SUMMARY`
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`
- `CLAUDE_CODE_BRIEF`

Disable:

- `CLAUDE_CODE_DISABLE_AGENT_VIEW`
- `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS`
- `CLAUDE_CODE_DISABLE_AUTO_MEMORY`
- `CLAUDE_CODE_DISABLE_CLAUDE_MDS`
- `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK`
- `CLAUDE_CODE_DISABLE_CRON`
- `CLAUDE_CODE_DISABLE_FAST_MODE`
- `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING`
- `CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS`
- `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP`
- `CLAUDE_CODE_DISABLE_MESSAGE_ACTIONS`
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`
- `CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK`
- `CLAUDE_CODE_DISABLE_OFFICIAL_MARKETPLACE_AUTOINSTALL`
- `CLAUDE_CODE_DISABLE_POLICY_SKILLS`
- `CLAUDE_CODE_DISABLE_PRECOMPACT_SKIP`
- `CLAUDE_CODE_DISABLE_TERMINAL_TITLE`
- `CLAUDE_CODE_DISABLE_VIRTUAL_SCROLL`
- `DISABLE_PROMPT_CACHING`
- `DISABLE_PROMPT_CACHING_HAIKU`
- `DISABLE_PROMPT_CACHING_SONNET`
- `DISABLE_PROMPT_CACHING_OPUS`

## Plugin and MCP Variables

- `CLAUDE_CODE_PLUGIN_CACHE_DIR`
- `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS`
- `CLAUDE_CODE_PLUGIN_SEED_DIR`
- `CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE`
- `CLAUDE_CODE_SYNC_PLUGIN_INSTALL`
- `CLAUDE_CODE_SYNC_PLUGIN_INSTALL_TIMEOUT_MS`
- `FORCE_AUTOUPDATE_PLUGINS`
- `CLAUDE_AGENT_SDK_MCP_NO_PREFIX`
- `MCP_TIMEOUT`
- `MCP_TOOL_TIMEOUT`
- `MCP_SERVER_CONNECTION_BATCH_SIZE`
- `MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE`
- `MCP_CLIENT_SECRET`
- `MCP_OAUTH_CALLBACK_PORT`
- `MCP_OAUTH_CLIENT_METADATA_URL`
- `MCP_XAA_IDP_CLIENT_SECRET`
- `CLAUDE_CODE_MCP_INSTR_DELTA`

## Hook/Subprocess Variables

- `CLAUDE_CODE_SAVE_HOOK_ADDITIONAL_CONTEXT`
- `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`
- `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB`
- `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR`
- `CLAUDE_ENV_FILE`
- `SHELL`
- `PATH`
- `PWD`
- `HOME`

Sensitive env vars are scrubbed or withheld from subprocesses depending on
subprocess construction and scrub configuration.

## Privacy, Telemetry, and Debug Variables

Privacy:

- `DISABLE_TELEMETRY`
- `DISABLE_ERROR_REPORTING`
- `DISABLE_GROWTHBOOK`
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`

OpenTelemetry:

- `ANT_OTEL_EXPORTER_OTLP_ENDPOINT`
- `ANT_OTEL_EXPORTER_OTLP_HEADERS`
- `ANT_OTEL_EXPORTER_OTLP_PROTOCOL`
- `ANT_OTEL_LOGS_EXPORTER`
- `ANT_OTEL_METRICS_EXPORTER`
- `ANT_OTEL_TRACES_EXPORTER`
- `OTEL_EXPORTER_OTLP_ENDPOINT`
- `OTEL_EXPORTER_OTLP_HEADERS`
- `OTEL_EXPORTER_OTLP_PROTOCOL`
- `OTEL_LOGS_EXPORTER`
- `OTEL_METRICS_EXPORTER`
- `OTEL_TRACES_EXPORTER`
- `OTEL_LOG_USER_PROMPTS`
- `OTEL_LOG_TOOL_CONTENT`
- `OTEL_LOG_TOOL_DETAILS`
- `OTEL_METRIC_EXPORT_INTERVAL`
- `OTEL_LOGS_EXPORT_INTERVAL`
- `OTEL_TRACES_EXPORT_INTERVAL`

Debug/profile:

- `DEBUG`
- `DEBUG_SDK`
- `CLAUDE_DEBUG`
- `CLAUDE_CODE_DEBUG_LOG_LEVEL`
- `CLAUDE_CODE_DEBUG_LOGS_DIR`
- `CLAUDE_CODE_DEBUG_REPAINTS`
- `CLAUDE_CODE_PROFILE_STARTUP`
- `CLAUDE_CODE_PROFILE_QUERY`
- `CLAUDE_CODE_PERFETTO_TRACE`
- `CLAUDE_CODE_FRAME_TIMING_LOG`
- `CLAUDE_CODE_DIAGNOSTICS_FILE`
- `BETA_TRACING_ENDPOINT`
- `ENABLE_BETA_TRACING_DETAILED`

## Remote, Bridge, and Harness Variables

- `CLAUDE_CODE_REMOTE`
- `CLAUDE_CODE_REMOTE_ENVIRONMENT_TYPE`
- `CLAUDE_CODE_REMOTE_SESSION_ID`
- `CLAUDE_CODE_REMOTE_MEMORY_DIR`
- `CLAUDE_CODE_REMOTE_SEND_KEEPALIVES`
- `SESSION_INGRESS_URL`
- `CLAUDE_SESSION_INGRESS_TOKEN_FILE`
- `CLAUDE_BRIDGE_BASE_URL`
- `CLAUDE_BRIDGE_OAUTH_TOKEN`
- `CLAUDE_BRIDGE_SESSION_INGRESS_URL`
- `CLAUDE_BRIDGE_USE_CCR_V2`
- `CLAUDE_CODE_USE_CCR_V2`
- `CLAUDE_CODE_CCR_MIRROR`
- `CCR_ENABLE_BUNDLE`
- `CCR_FORCE_BUNDLE`
- `CCR_EGRESS_GATEWAY_ENABLED`
- `CCR_UPSTREAM_PROXY_ENABLED`
- `CLAUDE_DAEMON_JSON_PATH`
- `AGENT_PROXY_URL`
- `AGENT_PROXY_AUTH_TOKEN`

## Terminal Detection Variables

- `TERM`
- `TERM_PROGRAM`
- `TERM_PROGRAM_VERSION`
- `COLORTERM`
- `COLORFGBG`
- `TMUX`
- `TMUX_PANE`
- `STY`
- `WT_SESSION`
- `ITERM_SESSION_ID`
- `KITTY_WINDOW_ID`
- `KONSOLE_VERSION`
- `VTE_VERSION`
- `ALACRITTY_LOG`
- `ZED_TERM`
- `TERMINAL`
- `TERMINAL_EMULATOR`
- `GNOME_TERMINAL_SERVICE`
- `LC_TERMINAL`
- `CLAUDE_CODE_TMUX_PREFIX`
- `CLAUDE_CODE_TMUX_PREFIX_CONFLICTS`
- `CLAUDE_CODE_TMUX_SESSION`
- `CLAUDE_CODE_TMUX_TRUECOLOR`

## Detection and Hosting Variables

Source detects CI/hosting context from:

- `CI`
- `GITHUB_ACTIONS`
- `GITHUB_EVENT_NAME`
- `GITHUB_REPOSITORY`
- `GITHUB_ACTOR`
- `GITLAB_CI`
- `BUILDKITE`
- `CIRCLECI`
- `CODESPACES`
- `GITPOD_WORKSPACE_ID`
- `REPL_ID`
- `REPL_SLUG`
- `PROJECT_DOMAIN`
- `VERCEL`
- `RAILWAY_ENVIRONMENT_NAME`
- `RAILWAY_SERVICE_NAME`
- `RENDER`
- `NETLIFY`
- `DYNO`
- `FLY_APP_NAME`
- `FLY_MACHINE_ID`
- `CF_PAGES`
- `DENO_DEPLOYMENT_ID`
- `KUBERNETES_SERVICE_HOST`
- `K_SERVICE`
- `AWS_LAMBDA_FUNCTION_NAME`
- `AZURE_FUNCTIONS_ENVIRONMENT`

## Secret and Token Variables

Treat these as secret-bearing:

- `ANTHROPIC_API_KEY`
- `ANTHROPIC_AUTH_TOKEN`
- `ANTHROPIC_FOUNDRY_API_KEY`
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `GOOGLE_APPLICATION_CREDENTIALS`
- `CLAUDE_CODE_OAUTH_TOKEN`
- `CLAUDE_CODE_OAUTH_REFRESH_TOKEN`
- `CLAUDE_CODE_SESSION_ACCESS_TOKEN`
- `CLAUDE_TRUSTED_DEVICE_TOKEN`
- `MCP_CLIENT_SECRET`
- `MCP_XAA_IDP_CLIENT_SECRET`
- `CLAUDE_BRIDGE_OAUTH_TOKEN`
- `CLAUDE_SESSION_INGRESS_TOKEN_FILE`
- `NODE_EXTRA_CA_CERTS`
- `SSL_CERT_FILE`

## Variable Interaction Rules

- Provider flags are mutually exclusive in practical use; setting multiple
  provider selectors can lead to provider-specific auth and telemetry surprises.
- Telemetry disablement also affects GrowthBook availability.
- Simple/bare mode disables or skips several expensive/background subsystems.
- Runtime gates may override or ignore an env variable if the build flag removed
  the feature.
- Some variables are only read in Anthropic internal builds or ant user paths.
- Some variables are not user-facing controls; they are set by the CLI itself to
  communicate with child processes, SDKs, hooks, or remote workers.

## Entrypoint and Session Variables

These variables describe how the process was launched or how child processes
should identify themselves:

- `CLAUDE_CODE_ENTRYPOINT`: `cli`, `sdk-cli`, `sdk-ts`, `sdk-py`, `mcp`,
  `remote`, `local-agent`, `claude-desktop`, `daemon`, or related host values.
- `CLAUDE_CODE_SESSION_KIND`: daemon/session kind in daemon workers.
- `CLAUDE_CODE_SESSION_LOG`: daemon worker log path.
- `CLAUDE_CODE_SESSION_ACCESS_TOKEN`: remote/session ingress token.
- `CLAUDE_CODE_REMOTE_SESSION_ID`: remote session id for file/session actions.
- `CLAUDE_CODE_AGENT`: selected agent name from CLI startup.
- `CLAUDE_CODE_TASK_LIST_ID`: task-list identifier passed into subcontexts.
- `CLAUDE_CODE_IS_COWORK`: cowork/teammate execution hint.

Most of these are internal propagation variables. They should be documented so
debuggers understand them, but users should not generally set them by hand.

## Bare/Simple Mode Effects

`CLAUDE_CODE_SIMPLE` has unusually broad effect because `--bare` sets it. It
causes or contributes to:

- reduced tool set.
- skipped hooks.
- skipped LSP.
- skipped plugin sync.
- skipped attribution.
- disabled auto-memory.
- skipped background prefetches.
- skipped keychain reads.
- skipped CLAUDE.md auto-discovery unless explicitly supplied.
- explicit auth behavior through API key or apiKeyHelper.

This variable is not just a UI simplification flag. It changes startup,
context-loading, credential, background, hook, and tool behavior.

## Credential and Provider Matrix

Provider selectors and credential variables should be interpreted together:

- Anthropic direct API uses Anthropic API/base URL variables.
- OAuth/keychain flows use Claude OAuth and local OAuth variables.
- Bedrock uses AWS region/profile/key/container metadata variables plus
  Bedrock-specific Anthropic overrides.
- Vertex uses Google credentials and project/location configuration.
- Foundry uses Foundry API key/base URL variables.
- remote/bridge/CCR paths can use session ingress tokens instead of ordinary
  user API keys.

Setting multiple provider selectors can produce confusing auth and telemetry
because startup may prefetch more than one provider's credential path before the
final provider is used.

## GrowthBook and Telemetry Coupling

Several feature gates depend on GrowthBook. GrowthBook availability is affected
by privacy/network controls:

- telemetry disabled.
- nonessential traffic disabled.
- offline or third-party provider contexts.
- missing auth/user identity.
- managed policy.
- cached gate state.

Some gates default true specifically so Bedrock/Vertex/Foundry and
telemetry-disabled users do not lose core features. Cron is one example. Always
read the default value at the call site rather than assuming a missing gate means
false.

## Hook Environment

Hooks run as subprocesses and receive a constructed environment. Important
categories:

- session identity.
- current working directory.
- tool/event payload through stdin.
- hook-specific variables.
- selected safe Claude Code variables.
- scrubbed or inherited process variables depending on execution path.

Hook environment docs should distinguish "visible to hook subprocess" from
"visible to model". Those are different surfaces.

## Remote and Bridge Environment

Remote/bridge/CCR variables are high-signal because they often indicate a
nonstandard harness:

- `CLAUDE_CODE_REMOTE`
- `CLAUDE_CODE_REMOTE_MEMORY_DIR`
- `CLAUDE_CODE_ENVIRONMENT_KIND`
- `CLAUDE_BRIDGE_BASE_URL`
- `CLAUDE_BRIDGE_OAUTH_TOKEN`
- `CLAUDE_CODE_CCR_MIRROR`
- `CLAUDE_CODE_USE_CCR_V2`
- `SESSION_INGRESS_URL`
- `CLAUDE_SESSION_INGRESS_TOKEN_FILE`
- `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR`

These variables can affect startup classification, memory persistence, transport
selection, token download, and stream-json lifecycle behavior.

## UI and Terminal Environment

Terminal behavior depends on common terminal variables:

- `TERM`
- `TERM_PROGRAM`
- `TERM_PROGRAM_VERSION`
- `TMUX`
- `WT_SESSION`
- `KITTY_WINDOW_ID`
- `ZED_TERM`
- `COLORTERM`
- `BAT_THEME`
- `CLAUDE_CODE_TMUX_TRUECOLOR`
- `CLAUDE_CODE_SYNTAX_HIGHLIGHT`

These influence color depth, glyph selection, terminal feature detection,
syntax highlighting, and special terminal integrations.

## Debug and Test Variables

Debug/test variables are intentionally powerful:

- `NODE_ENV=test` enables test-only paths.
- `CLAUDE_CODE_TEST_FIXTURES_ROOT` controls fixture roots.
- `FORCE_VCR` and `VCR_RECORD` affect recorded API fixtures.
- `CLAUDE_CODE_PROFILE_STARTUP` and `CLAUDE_CODE_PROFILE_QUERY` affect
  profiling.
- `CLAUDE_CODE_PERFETTO_TRACE` enables trace output.
- `CLAUDE_CODE_FRAME_TIMING_LOG` enables frame timing logs.
- `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` changes lifecycle behavior.

Do not document test variables as supported user configuration. They are useful
for reconstruction, automation, and source-map validation.

## Extraction Procedure

For a future version, rebuild this file by:

1. Searching for `process.env.` and `Bun.env`.
2. Searching for `isEnvTruthy`, `isEnvDefinedFalsy`, and env helper wrappers.
3. Reading CLI option handlers that set environment variables.
4. Reading daemon, SDK, remote, bridge, and local-agent entrypoints.
5. Reading auth provider files for credential variables.
6. Reading terminal/color/shell helpers for UI variables.
7. Reading hook execution code for subprocess env construction.
8. Reading setup and background prefetch for variables that skip work.
9. Marking each variable as user-facing, internal propagation, secret-bearing,
   test-only, or provider-specific.
10. Validating that stale variables from older releases are not carried forward
    unless 2.1.141 source still references them.

## Deep 2.1.141 Environment Variable Audit

The 2.1.141 source references hundreds of environment variables. A useful
reference must separate user-facing controls from internal propagation,
provider credentials, test flags, and build/runtime feature toggles.

### High-Impact User/Operator Variables

| Variable | Source area | Purpose |
| --- | --- | --- |
| `ANTHROPIC_API_KEY` | auth/client/onboarding | Direct API key auth. |
| `ANTHROPIC_AUTH_TOKEN` | API client/auth | OAuth/session auth token path. |
| `ANTHROPIC_BASE_URL` | API, bridge upload, GrowthBook | Base URL override. |
| `ANTHROPIC_MODEL` | model selection/logging | Main model override. |
| `ANTHROPIC_SMALL_FAST_MODEL` | model selection/logging | Small/fast model override. |
| `ANTHROPIC_BETAS` | beta header handling | Extra beta header control. |
| `ANTHROPIC_CUSTOM_HEADERS` | API client | Custom request headers. |
| `CLAUDE_CONFIG_DIR` | env/config/keychain/daemon | Root config directory override. |
| `CLAUDE_CODE_SIMPLE` | CLI/main/prompts | Minimal/bare runtime behavior. |
| `CLAUDE_CODE_USE_BEDROCK` | provider/auth/status | Select Bedrock provider path. |
| `CLAUDE_CODE_USE_VERTEX` | provider/auth/status | Select Vertex provider path. |
| `CLAUDE_CODE_USE_FOUNDRY` | provider/auth/status | Select Foundry provider path. |
| `CLAUDE_CODE_REMOTE` | remote/CCR/main/print | Remote execution mode flag. |
| `CLAUDE_CODE_BRIEF` | main/Brief/spinner | Force/activate Brief behavior for dev/testing. |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | context | Disable CLAUDE.md loading. |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | tools/prompts/background | Disable background task surface. |
| `CLAUDE_CODE_DISABLE_FAST_MODE` | fast mode/query config | Disable fast mode. |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | privacy/registry | Reduce nonessential network traffic. |
| `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` | print/stop hooks/prompt suggestion | Enable prompt suggestion surfaces. |
| `CLAUDE_CODE_ENABLE_TASKS` | task tools | Enable task list V2 behavior. |
| `CLAUDE_CODE_EFFORT_LEVEL` | effort command/runtime | Reasoning effort override. |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | query/API | Max output tokens override. |
| `CLAUDE_CODE_MAX_RETRIES` | API retry | Retry count override. |
| `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` | hooks | SessionEnd hook timeout override. |
| `CLAUDE_CODE_SHELL` | shell selection | Shell override. |
| `CLAUDE_CODE_TMPDIR` | shell/image/permissions/temp | Temp directory override. |
| `CLAUDE_CODE_WORKSPACE_HOST_PATHS` | telemetry/events | Host path mapping metadata. |

### Provider Variables

| Provider | Variables observed |
| --- | --- |
| Bedrock | `CLAUDE_CODE_USE_BEDROCK`, `AWS_REGION`, `AWS_DEFAULT_REGION`, `AWS_PROFILE`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_BEARER_TOKEN_BEDROCK`, `ANTHROPIC_BEDROCK_BASE_URL`, `BEDROCK_BASE_URL`, `CLAUDE_CODE_SKIP_BEDROCK_AUTH` |
| Vertex | `CLAUDE_CODE_USE_VERTEX`, `ANTHROPIC_VERTEX_PROJECT_ID`, `CLAUDE_CODE_SKIP_VERTEX_AUTH` |
| Foundry | `CLAUDE_CODE_USE_FOUNDRY`, `ANTHROPIC_FOUNDRY_API_KEY`, `ANTHROPIC_FOUNDRY_BASE_URL`, `ANTHROPIC_FOUNDRY_RESOURCE`, `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` |
| First-party/OAuth | `CLAUDE_CODE_OAUTH_TOKEN`, `CLAUDE_CODE_OAUTH_REFRESH_TOKEN`, `CLAUDE_CODE_OAUTH_CLIENT_ID`, `CLAUDE_CODE_OAUTH_SCOPES`, `CLAUDE_CODE_CUSTOM_OAUTH_URL`, `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR`, `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` |

### Feature And Mode Variables

| Variable | Area |
| --- | --- |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | Agent swarms/teams enablement. |
| `CLAUDE_CODE_AGENT` | Main-thread agent type. |
| `CLAUDE_CODE_AGENT_LIST_IN_MESSAGES` | Agent prompt/message behavior. |
| `CLAUDE_AUTO_BACKGROUND_TASKS` | Agent/background task behavior. |
| `CLAUDE_CODE_COORDINATOR_MODE` | Coordinator mode. |
| `CLAUDE_CODE_DISABLE_AGENT_VIEW` | Agent View disable path. |
| `CLAUDE_CODE_DISABLE_CRON` | Cron/scheduled tasks disable path. |
| `CLAUDE_CODE_DISABLE_MESSAGE_ACTIONS` | UI message actions. |
| `CLAUDE_CODE_DISABLE_MOUSE` | Fullscreen/mouse handling. |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | Terminal title updates. |
| `CLAUDE_CODE_DISABLE_VIRTUAL_SCROLL` | Virtual scrolling. |
| `CLAUDE_CODE_ENABLE_AWAY_SUMMARY` | Away summary service. |
| `CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING` | API/tool streaming detail. |
| `CLAUDE_CODE_ENABLE_TOKEN_USAGE_ATTACHMENT` | Token usage attachment. |
| `CLAUDE_CODE_ENABLE_XAA` | MCP XAA path. |
| `CLAUDE_CODE_USE_NATIVE_FILE_SEARCH` | Native file search path. |

### Auto Mode And Permission Variables

| Variable | Area |
| --- | --- |
| `CLAUDE_CODE_AUTO_MODE_MODEL` | Auto-mode classifier model override. |
| `CLAUDE_CODE_DUMP_AUTO_MODE` | Dumps classifier request/response JSON for internal diagnostics. |
| `CLAUDE_CODE_TWO_STAGE_CLASSIFIER` | Two-stage classifier mode selection. |
| `CLAUDE_CODE_JSONL_TRANSCRIPT` | Auto-mode classifier transcript serialization mode. |
| `CLAUDE_CODE_ADDITIONAL_PROTECTION` | API/client additional protection. |
| `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` | Subprocess environment scrubbing. |
| `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK` | Bash permission/security path. |
| `CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR` | Bash sandbox UI indicator. |
| `CLAUDE_CODE_BUBBLEWRAP` | Bubblewrap sandbox path. |

### Hooks And Context Variables

| Variable | Area |
| --- | --- |
| `CLAUDE_ENV_FILE` | Hook/session environment side channel. |
| `CLAUDE_CODE_SAVE_HOOK_ADDITIONAL_CONTEXT` | Persist hook additional context. |
| `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` | SessionEnd hook timeout. |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` | Additional CLAUDE.md directories. |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | Disable CLAUDE.md loading. |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | Disable auto-memory. |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | Remote memory directory. |
| `CLAUDE_CODE_REMOTE_SEND_KEEPALIVES` | Remote session activity. |

### Bridge, Remote, And Harness Variables

| Variable | Area |
| --- | --- |
| `CLAUDE_BRIDGE_BASE_URL` | Bridge base URL. |
| `CLAUDE_BRIDGE_OAUTH_TOKEN` | Bridge OAuth token. |
| `CLAUDE_BRIDGE_SESSION_INGRESS_URL` | Bridge ingress. |
| `CLAUDE_BRIDGE_USE_CCR_V2` | Bridge CCR V2 path. |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | Session ingress/remote bridge auth. |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | Remote session id propagation. |
| `CLAUDE_CODE_REMOTE_ENVIRONMENT_TYPE` | Remote environment metadata. |
| `CLAUDE_CODE_ENVIRONMENT_KIND` | Environment kind metadata/output scanner. |
| `CLAUDE_CODE_ENVIRONMENT_RUNNER_VERSION` | Environment runner metadata. |
| `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2` | Session ingress transport. |
| `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` | WebSocket auth fd. |

### Internal Propagation Variables

These variables are meaningful in 2.1.141 source but are mostly used to pass
state between Claude Code processes, workers, bridge sessions, or launched
subprocesses:

| Variable | Meaning |
| --- | --- |
| `CLAUDE_CODE_ENTRYPOINT` | Entrypoint classification. |
| `CLAUDE_CODE_ACTION` | CLI action metadata. |
| `CLAUDE_CODE_PATH` | SDK/process path propagation. |
| `CLAUDE_CODE_TASK_LIST_ID` | Task list selection. |
| `CLAUDE_CODE_SESSION_ID` | Session id propagation. |
| `CLAUDE_CODE_SESSION_KIND` | Concurrent session kind. |
| `CLAUDE_CODE_SESSION_NAME` | Concurrent session display name. |
| `CLAUDE_CODE_SESSION_LOG` | Session log path. |
| `CLAUDE_CODE_AGENT_SDK_VERSION` | SDK version metadata. |
| `CLAUDE_AGENT_SDK_CLIENT_APP` | SDK client app metadata. |
| `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` | SDK MCP tool-name compatibility mode. |
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` | SDK built-in agent suppression. |

### Classification Rules For Future Docs

When adding variables for later releases, classify each source reference as:

| Class | Examples |
| --- | --- |
| Secret-bearing | API keys, OAuth tokens, fd-based credentials, client cert/key. |
| Provider selector | Bedrock, Vertex, Foundry, Mantle, first-party. |
| User-facing feature control | Brief, tasks, prompt suggestion, background tasks, shell/tool toggles. |
| Internal propagation | Session ids, bridge tokens, entrypoint/action state. |
| Test/dev diagnostic | dumps, profiling, fixtures, debug logs. |
| Build/runtime gate | Variables that expose code behind build flags or internal feature flags. |

This prevents stale docs from presenting internal process plumbing as a stable
user interface.
