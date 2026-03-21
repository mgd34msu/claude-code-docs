# Claude Code v2.1.71 — Environment Variables Reference

Complete reference for all environment variables used by Claude Code CLI v2.1.71.
Source: `cli.js`, 612,918 lines, build `2026-03-06T22:45:36Z`.

---

## Table of Contents

1. [CLAUDE_CODE_* Variables (117 vars)](#1-claude_code_-variables)
2. [CLAUDE_* Variables (15 vars)](#2-claude_-variables)
3. [ANTHROPIC_* Variables (17 vars)](#3-anthropic_-variables)
4. [Feature Toggle Gates](#4-feature-toggle-gates)
   - 4.1 [DISABLE_* / ENABLE_* Gates (29 vars)](#41-disable--enable-gates)
   - 4.2 [USE_* / LOCAL_* Gates (5 vars)](#42-use--local-gates)
5. [Telemetry & Observability](#5-telemetry--observability)
   - 5.1 [OTEL_* Variables (27 vars)](#51-otel_-variables)
   - 5.2 [Debug & Development (5 vars)](#52-debug--development)
6. [Cloud Providers](#6-cloud-providers)
   - 6.1 [AWS / Bedrock (8 vars)](#61-aws--bedrock)
   - 6.2 [Azure / Foundry (15 vars)](#62-azure--foundry)
   - 6.3 [GCP / Vertex AI (14 vars)](#63-gcp--vertex-ai)
7. [Terminal & Display Detection (27 vars)](#7-terminal--display-detection)
8. [System & Paths (13 vars)](#8-system--paths)
9. [CI/CD & Git (10 vars)](#9-cicd--git)
10. [MCP (Model Context Protocol) (5 vars)](#10-mcp-model-context-protocol)
11. [Bash Tool Variables (5 vars)](#11-bash-tool-variables)
12. [Build Tools & Vendor Libraries (3 vars)](#12-build-tools--vendor-libraries)
13. [Cloud Platform Detection (~26 vars)](#13-cloud-platform-detection)

## Summary Statistics

---

## 1. CLAUDE_CODE_* Variables

All variables with the `CLAUDE_CODE_` prefix. Gate type `boolean` means parsed via `$1()` (truthy: `"1"`, `"true"`, `"yes"`) unless noted. `dz()` is the negation form (gate is ON by default, env var disables it).

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `CLAUDE_CODE_ACCESSIBILITY` | boolean | off | Enable accessibility / screen-reader mode |
| `CLAUDE_CODE_ACCOUNT_UUID` | string | — | Account UUID override for telemetry |
| `CLAUDE_CODE_ACTION` | boolean | off | Indicates GitHub Actions mode; gates action-specific prefetch behavior |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` | boolean | off | Scan additional directories for CLAUDE.md files |
| `CLAUDE_CODE_ADDITIONAL_PROTECTION` | boolean | off | Enable additional security / protection layer |
| `CLAUDE_CODE_ALWAYS_ENABLE_EFFORT` | boolean | off | Always enable extended thinking / effort mode |
| `CLAUDE_CODE_API_BASE_URL` | url | — | Override the Anthropic API base URL (checked before `ANTHROPIC_BASE_URL`) |
| `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR` | number | — | File descriptor number to read the API key from at startup |
| `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` | number | — | TTL (ms) for cached results from the API key helper process |
| `CLAUDE_CODE_ATTRIBUTION_HEADER` | boolean (dz) | on | Set to falsy value to disable the `x-anthropic-billing-header` attribution header |
| `CLAUDE_CODE_AUTO_CONNECT_IDE` | boolean | off | Auto-connect to IDE on startup (uses both `$1` and `dz` checks) |
| `CLAUDE_CODE_BASE_REF` | string | — | Git base ref for diff operations |
| `CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR` | boolean | off | When sandboxed, changes tool's `userFacingName` to `"SandboxedBash"` |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | number | — | Override the context blocking limit |
| `CLAUDE_CODE_BUBBLEWRAP` | boolean (`=== "1"`) | off | Enable Bubblewrap sandboxing on Linux |
| `CLAUDE_CODE_CLIENT_CERT` | path | — | Path to client TLS certificate file |
| `CLAUDE_CODE_CLIENT_KEY` | path | — | Path to client TLS private key file |
| `CLAUDE_CODE_CLIENT_KEY_PASSPHRASE` | string | — | Passphrase for encrypted client TLS key |
| `CLAUDE_CODE_CONTAINER_ID` | string | — | Container identifier for remote sessions |
| `CLAUDE_CODE_CUSTOM_OAUTH_URL` | url | — | Custom OAuth server URL (overrides default) |
| `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS` | number | — | Datadog metrics flush interval (ms) |
| `CLAUDE_CODE_DEBUG_LOG_LEVEL` | string | — | Debug log verbosity level (`debug`, `info`, etc.) |
| `CLAUDE_CODE_DEBUG_LOGS_DIR` | path | — | Directory for debug log files |
| `CLAUDE_CODE_DIAGNOSTICS_FILE` | path | — | Path to write diagnostics output |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | boolean | off | Disable 1M token context window support |
| `CLAUDE_CODE_DISABLE_ATTACHMENTS` | boolean | off | Disable file attachment support |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | boolean | off | Disable automatic memory / CLAUDE.md loading |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | boolean | off | Disable background task processing |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | boolean (truthy) | off | Disable all CLAUDE.md file loading |
| `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK` | boolean | off | Skip the command-injection detection check in the bash tool |
| `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` | boolean | off | Suppress experimental beta API headers from being sent |
| `CLAUDE_CODE_DISABLE_FAST_MODE` | boolean | off | Disable fast mode (always use the full model) |
| `CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY` | boolean | off | Suppress the post-session feedback survey prompt |
| `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` | boolean | off | Disable file state checkpoint snapshots |
| `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP` | boolean | off | Disable legacy model name → new name remapping |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | boolean (truthy) | off | Block telemetry, analytics, and non-essential network calls |
| `CLAUDE_CODE_DISABLE_OFFICIAL_MARKETPLACE_AUTOINSTALL` | boolean | off | Prevent auto-installing plugins from the official marketplace |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | boolean | off | Disable writing the current task name to the terminal title bar |
| `CLAUDE_CODE_EAGER_FLUSH` | boolean | off | Enable eager flushing of streaming output (bypasses buffering) |
| `CLAUDE_CODE_EFFORT_LEVEL` | string | — | Set effort / thinking level explicitly |
| `CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES` | boolean | off | Emit tool use summary telemetry events |
| `CLAUDE_CODE_ENABLE_CFC` | boolean | off | Enable Claude for Chrome (checked with both `$1` for enable and `dz` for disable) |
| `CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING` | boolean | off | Enable fine-grained tool streaming (also gated by `tengu_fgts`) |
| `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` | string | — | Set to `"false"` to disable prompt suggestions |
| `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING` | boolean | off | Enable file checkpointing in SDK mode |
| `CLAUDE_CODE_ENABLE_TASKS` | boolean | off | Enable task list feature |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | boolean | off | Enable OTEL / third-party telemetry export |
| `CLAUDE_CODE_ENABLE_TOKEN_USAGE_ATTACHMENT` | boolean | off | Include token usage data in message attachments |
| `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA` | string | — | Enable enhanced telemetry beta features |
| `CLAUDE_CODE_ENTRYPOINT` | string | — | Identifies launch path: `cli`, `sdk-ts`, `sdk-py`, `sdk-cli` |
| `CLAUDE_CODE_ENVIRONMENT_KIND` | string | — | Environment type identifier (e.g., `bridge`) |
| `CLAUDE_CODE_ENVIRONMENT_RUNNER_VERSION` | string | — | Version string of the environment runner |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | boolean | off | Enable experimental multi-agent team support (also gated by `tengu_amber_flint`) |
| `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` | boolean | off | Exit the process immediately after the first UI render (testing / CI) |
| `CLAUDE_CODE_EXIT_AFTER_STOP_DELAY` | number | — | Delay (ms) before process exit after a stop signal |
| `CLAUDE_CODE_EXTRA_BODY` | string (JSON) | — | JSON object merged into every API request body (must be a JSON object string) |
| `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | number | — | Max tokens for file read tool output |
| `CLAUDE_CODE_FORCE_GLOBAL_CACHE` | boolean | off | Force use of global system prompt cache across sessions |
| `CLAUDE_CODE_GIT_BASH_PATH` | path | — | Custom path to git-bash executable (Windows) |
| `CLAUDE_CODE_GLOB_TIMEOUT_SECONDS` | number | — | Timeout in seconds for glob file searches |
| `CLAUDE_CODE_HOST_PLATFORM` | string | — | Override host platform identifier (`win32`, `darwin`, `linux`) |
| `CLAUDE_CODE_IDE_HOST_OVERRIDE` | string | — | Override IDE server hostname |
| `CLAUDE_CODE_IDE_SKIP_AUTO_INSTALL` | boolean | off | Skip auto-installation of IDE extension |
| `CLAUDE_CODE_IDE_SKIP_VALID_CHECK` | boolean | off | Skip IDE integration validation check |
| `CLAUDE_CODE_INCLUDE_PARTIAL_MESSAGES` | boolean | off | Include partial / in-progress messages in remote session output |
| `CLAUDE_CODE_IS_COWORK` | boolean | off | Marks session as a cowork (collaboration) session; enables eager flush |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | number | — | Override the maximum output tokens for API requests |
| `CLAUDE_CODE_MAX_RETRIES` | number | — | Maximum API retry attempts |
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` | number | 10 | Maximum concurrent tool calls |
| `CLAUDE_CODE_MCP_INSTR_DELTA` | boolean | off | Enable MCP instruction delta mode (incremental updates; also gated by `tengu_basalt_3kr`) |
| `CLAUDE_CODE_OAUTH_CLIENT_ID` | string | — | Custom OAuth client ID |
| `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` | string | — | Pre-set OAuth refresh token |
| `CLAUDE_CODE_OAUTH_SCOPES` | string | — | Override OAuth scopes |
| `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | number | — | File descriptor number to read the OAuth token from at startup |
| `CLAUDE_CODE_ORGANIZATION_UUID` | string | — | Organization UUID for multi-org setups |
| `CLAUDE_CODE_OTEL_FLUSH_TIMEOUT_MS` | number | 5000 | OTEL flush timeout (ms) |
| `CLAUDE_CODE_OTEL_HEADERS_HELPER_DEBOUNCE_MS` | number | — | Debounce interval for OTEL header helper |
| `CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS` | number | 2000 | OTEL shutdown timeout (ms) |
| `CLAUDE_CODE_PERFETTO_TRACE` | path | — | Path to output a Perfetto performance trace |
| `CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE` | string | — | Interview phase setting for plan mode (also gated by `tengu_plan_mode_interview_phase`) |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | boolean | off | Force plan-before-execute mode |
| `CLAUDE_CODE_PLAN_V2_AGENT_COUNT` | number | — | Number of agents for Plan v2 |
| `CLAUDE_CODE_PLAN_V2_EXPLORE_AGENT_COUNT` | number | — | Number of explore agents for Plan v2 |
| `CLAUDE_CODE_PLUGIN_CACHE_DIR` | path | — | Custom directory for plugin cache |
| `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS` | number | — | Timeout for plugin git operations (ms) |
| `CLAUDE_CODE_PLUGIN_SEED_DIR` | path | — | Seed directory for initial plugin data |
| `CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE` | boolean | off | Use zip-based plugin cache |
| `CLAUDE_CODE_PROFILE_STARTUP` | boolean (`=== "1"`) | off | Enable startup performance profiling |
| `CLAUDE_CODE_PROXY_RESOLVES_HOSTS` | boolean | off | Proxy server resolves hostnames (no local DNS lookup) |
| `CLAUDE_CODE_REMOTE` | boolean | off | Enable remote / headless mode |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | path | — | Directory for remote session memory storage |
| `CLAUDE_CODE_REMOTE_SEND_KEEPALIVES` | boolean | off | Send keepalive pings in remote mode |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | string | — | Remote session identifier |
| `CLAUDE_CODE_SEARCH_HINTS_IN_LIST` | boolean | off | Show search hints in deferred tool list UI (also gated by `tengu_tst_hint_m7r`) |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | string | — | Session access token for authentication |
| `CLAUDE_CODE_SHELL` | string | — | Override shell executable path |
| `CLAUDE_CODE_SHELL_PREFIX` | string | — | Prefix prepended to shell invocation commands |
| `CLAUDE_CODE_SIMPLE` | boolean | off | Enable simplified / lite mode (fewer features) |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | boolean | off | Skip Bedrock auth; use ambient AWS credentials instead |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | boolean | off | Skip Azure AI Foundry authentication |
| `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | boolean | off | Skip saving prompt history to disk |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | boolean | off | Skip Vertex AI authentication |
| `CLAUDE_CODE_SLOW_OPERATION_THRESHOLD_MS` | number | — | Threshold (ms) above which slow operations are logged |
| `CLAUDE_CODE_SSE_PORT` | number | — | Port for the Server-Sent Events (SSE) server |
| `CLAUDE_CODE_STALL_TIMEOUT_MS_FOR_TESTING` | number | — | Override stall detection timeout (testing only) |
| `CLAUDE_CODE_STREAMING_TEXT` | boolean | off | Enable streaming text output mode (also gated by `tengu_streaming_text`) |
| `CLAUDE_CODE_SUBAGENT_MODEL` | string | — | Override model used for subagents |
| `CLAUDE_CODE_SYNTAX_HIGHLIGHT` | string (dz) | on | Set to falsy value to disable syntax highlighting |
| `CLAUDE_CODE_TASK_LIST_ID` | string | — | Pre-set task list ID |
| `CLAUDE_CODE_TEST_FIXTURES_ROOT` | path | — | Root directory for test fixtures |
| `CLAUDE_CODE_TMPDIR` | path | `/tmp` | Override temp directory for Claude Code |
| `CLAUDE_CODE_TWO_STAGE_CLASSIFIER` | boolean | off | Enable two-stage safety classifier |
| `CLAUDE_CODE_DONT_INHERIT_ENV` | boolean (truthy) | off | Prevent inheriting parent process environment in subprocesses |
| `CLAUDE_CODE_USE_BEDROCK` | boolean | off | Use Amazon Bedrock as the API backend |
| `CLAUDE_CODE_USE_COWORK_PLUGINS` | boolean | off | Enable collaboration / cowork plugin system |
| `CLAUDE_CODE_USE_FOUNDRY` | boolean | off | Use Azure AI Foundry as the API backend |
| `CLAUDE_CODE_USE_VERTEX` | boolean | off | Use Google Vertex AI as the API backend |
| `CLAUDE_CODE_USER_EMAIL` | string | — | User email override for telemetry |
| `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` | string | — | File descriptor for WebSocket auth token |

---

## 2. CLAUDE_* Variables

Variables with the `CLAUDE_` prefix that are NOT `CLAUDE_CODE_*`.

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `CLAUDE_AFTER_LAST_COMPACT` | boolean | off | Post-compact behavior control flag |
| `CLAUDE_AGENT_SDK_CLIENT_APP` | string | — | Client app name injected into the User-Agent header |
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` | boolean | off | Disable built-in agent type definitions from SDK |
| `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` | boolean | off | Disable MCP tool name prefixing in SDK mode |
| `CLAUDE_AGENT_SDK_VERSION` | string | — | Agent SDK version string injected into the User-Agent header |
| `CLAUDE_AUTO_BACKGROUND_TASKS` | boolean | off | Auto-enable background task scheduling |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | string | — | Override autocompact threshold percentage |
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | boolean | off | Keep bash subshell in project working directory across calls |
| `CLAUDE_BRIDGE_USE_CCR_V2` | boolean | off | Use CCR v2 bridge protocol |
| `CLAUDE_DEBUG` | boolean | off | Enable Claude-specific debug output |
| `CLAUDE_ENABLE_STREAM_WATCHDOG` | boolean | off | Enable stream watchdog timeout |
| `CLAUDE_ENV_FILE` | path | — | Path to `.env` file to load at startup |
| `CLAUDE_FORCE_DISPLAY_SURVEY` | boolean (truthy) | off | Force display of user survey |
| `CLAUDE_REPL_MODE` | boolean | off | Enable REPL (interactive) mode |
| `CLAUDE_TMPDIR` | path | — | Temp directory override for the bash tool |

---

## 3. ANTHROPIC_* Variables

Variables with the `ANTHROPIC_` prefix — API configuration and model overrides. (17 vars)

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `ANTHROPIC_API_KEY` | string | — | Anthropic API key for authentication |
| `ANTHROPIC_AUTH_TOKEN` | string | — | Bearer token for Anthropic API (alternative to API key) |
| `ANTHROPIC_BASE_URL` | url | — | Override Anthropic API base URL |
| `ANTHROPIC_BEDROCK_BASE_URL` | url | — | Custom Bedrock endpoint URL |
| `ANTHROPIC_BETAS` | string | — | Comma-separated list of beta feature flags to enable |
| `ANTHROPIC_CUSTOM_HEADERS` | string | — | Additional headers to send with all API requests (JSON or key:value) |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | string | — | Override Haiku model identifier |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | string | — | Override Opus model identifier |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | string | — | Override Sonnet model identifier |
| `ANTHROPIC_FOUNDRY_API_KEY` | string | — | API key for Azure AI Foundry |
| `ANTHROPIC_FOUNDRY_BASE_URL` | url | — | Base URL for Azure AI Foundry endpoint |
| `ANTHROPIC_FOUNDRY_RESOURCE` | string | — | Azure Foundry resource name |
| `ANTHROPIC_MODEL` | string | — | Override the primary model name |
| `ANTHROPIC_SMALL_FAST_MODEL` | string | — | Override the small / fast model name |
| `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` | string | — | AWS region for small / fast model on Bedrock |
| `ANTHROPIC_VERTEX_BASE_URL` | url | — | Custom Vertex AI endpoint base URL |
| `ANTHROPIC_VERTEX_PROJECT_ID` | string | — | Google Cloud project ID for Vertex AI |


---

## 4. Feature Toggle Gates

### 4.1 DISABLE_* / ENABLE_* Gates

Explicit feature disable/enable toggles not covered by the `CLAUDE_CODE_*` prefix.

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `DISABLE_AUTO_COMPACT` | boolean | off | Disable automatic compaction (allow manual only) |
| `DISABLE_AUTO_MIGRATE_TO_NATIVE` | boolean | off | Skip migration to native auth method |
| `DISABLE_AUTOUPDATER` | boolean | off | Disable auto-update mechanism |
| `DISABLE_BUG_COMMAND` | boolean (truthy) | off | Hide the `/bug` command |
| `DISABLE_CLAUDE_CODE_SM_COMPACT` | boolean | off | Force-disable session memory compaction |
| `DISABLE_COMPACT` | boolean | off | Disable context compaction feature entirely |
| `DISABLE_COST_WARNINGS` | boolean | off | Suppress cost warning UI messages |
| `DISABLE_DOCTOR_COMMAND` | boolean (truthy) | off | Hide the `/doctor` command |
| `DISABLE_ERROR_REPORTING` | boolean (truthy) | off | Disable Sentry / error reporting |
| `DISABLE_EXTRA_USAGE_COMMAND` | boolean (truthy) | off | Hide the `/usage` command |
| `DISABLE_FEEDBACK_COMMAND` | boolean (truthy) | off | Hide the `/feedback` command |
| `DISABLE_INSTALL_GITHUB_APP_COMMAND` | boolean (truthy) | off | Hide the GitHub App install command |
| `DISABLE_INSTALLATION_CHECKS` | boolean | off | Skip installation validation checks |
| `DISABLE_INTERLEAVED_THINKING` | boolean | off | Disable interleaved thinking / reasoning tokens |
| `DISABLE_LOGIN_COMMAND` | boolean (truthy) | off | Hide the `/login` command |
| `DISABLE_LOGOUT_COMMAND` | boolean (truthy) | off | Hide the `/logout` command |
| `DISABLE_PROMPT_CACHING` | boolean | off | Disable prompt caching entirely |
| `DISABLE_PROMPT_CACHING_HAIKU` | boolean | off | Disable prompt caching for Haiku model |
| `DISABLE_PROMPT_CACHING_OPUS` | boolean | off | Disable prompt caching for Opus model |
| `DISABLE_PROMPT_CACHING_SONNET` | boolean | off | Disable prompt caching for Sonnet model |
| `DISABLE_TELEMETRY` | boolean (truthy) | off | Disable all telemetry |
| `DISABLE_UPGRADE_COMMAND` | boolean (truthy) | off | Hide the `/upgrade` command |
| `ENABLE_BETA_TRACING_DETAILED` | boolean | off | Enable detailed beta tracing output |
| `ENABLE_CLAUDE_CODE_SM_COMPACT` | boolean | off | Force-enable session memory compaction |
| `ENABLE_CLAUDEAI_MCP_SERVERS` | boolean (dz) | on | Enable Claude.ai MCP server integration (disabled when set to falsy) |
| `ENABLE_ENHANCED_TELEMETRY_BETA` | string | — | Enable enhanced telemetry beta features |
| `ENABLE_MCP_LARGE_OUTPUT_FILES` | boolean (dz) | on | Enable large output file handling for MCP (disabled when set to falsy) |
| `ENABLE_PROMPT_CACHING_1H_BEDROCK` | boolean | off | Enable 1-hour prompt caching on Bedrock |
| `ENABLE_TOOL_SEARCH` | string | — | Enable / configure tool search mode (`"standard"`, etc.) |

### 4.2 USE_* / LOCAL_* Gates

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `LOCAL_BRIDGE` | boolean | off | Use local bridge connection |
| `USE_API_CONTEXT_MANAGEMENT` | boolean | off | Delegate context management to the API |
| `USE_BUILTIN_RIPGREP` | boolean (dz) | on | Use bundled ripgrep binary (disabled when set to falsy) |
| `USE_LOCAL_OAUTH` | boolean | off | Use local OAuth server instead of remote |
| `USE_STAGING_OAUTH` | boolean | off | Use staging OAuth endpoint |

---

## 5. Telemetry & Observability

### 5.1 OTEL_* Variables

OpenTelemetry configuration variables.

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | url | — | OTLP exporter endpoint URL |
| `OTEL_EXPORTER_OTLP_HEADERS` | string | — | HTTP headers for OTLP exporter (all signals) |
| `OTEL_EXPORTER_OTLP_INSECURE` | string | — | Allow insecure (non-TLS) OTLP connections |
| `OTEL_EXPORTER_OTLP_LOGS_HEADERS` | string | — | Per-signal HTTP headers for OTLP logs exporter |
| `OTEL_EXPORTER_OTLP_LOGS_PROTOCOL` | string | — | OTLP protocol for logs (`grpc`, `http/protobuf`, `http/json`) |
| `OTEL_EXPORTER_OTLP_METRICS_CLIENT_CERTIFICATE` | path | — | Client certificate for mTLS on OTLP metrics exporter |
| `OTEL_EXPORTER_OTLP_METRICS_CLIENT_KEY` | path | — | Client key for mTLS on OTLP metrics exporter |
| `OTEL_EXPORTER_OTLP_METRICS_HEADERS` | string | — | Per-signal HTTP headers for OTLP metrics exporter |
| `OTEL_EXPORTER_OTLP_METRICS_PROTOCOL` | string | — | OTLP protocol for metrics |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | string | `delta` | Metrics temporality (forced to `delta` internally) |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | string | — | Default OTLP protocol for all signals |
| `OTEL_EXPORTER_OTLP_TRACES_HEADERS` | string | — | Per-signal HTTP headers for OTLP traces exporter |
| `OTEL_EXPORTER_OTLP_TRACES_PROTOCOL` | string | — | OTLP protocol for traces |
| `OTEL_EXPORTER_PROMETHEUS_HOST` | string | — | Prometheus exporter host |
| `OTEL_EXPORTER_PROMETHEUS_PORT` | number | — | Prometheus exporter port |
| `OTEL_LOG_TOOL_CONTENT` | boolean | off | Include tool call content in OTEL logs |
| `OTEL_LOG_TOOL_DETAILS` | boolean | off | Include tool call details in OTEL logs |
| `OTEL_LOG_USER_PROMPTS` | boolean | off | Include user prompts in OTEL logs (otherwise redacted) |
| `OTEL_LOGS_EXPORT_INTERVAL` | number | — | Logs export interval (ms) |
| `OTEL_LOGS_EXPORTER` | string | — | Logs exporter type |
| `OTEL_METRIC_EXPORT_INTERVAL` | number | — | Metrics export interval (ms) |
| `OTEL_METRICS_EXPORTER` | string | — | Metrics exporter type |
| `OTEL_METRICS_INCLUDE_ACCOUNT_UUID` | boolean | off | Include account UUID as a metric attribute |
| `OTEL_METRICS_INCLUDE_SESSION_ID` | boolean | off | Include session ID as a metric attribute |
| `OTEL_METRICS_INCLUDE_VERSION` | boolean | off | Include Claude Code version as a metric attribute |
| `OTEL_RESOURCE_ATTRIBUTES` | string | — | OTEL resource attribute key=value pairs (standard OTEL spec) |
| `OTEL_TRACES_EXPORT_INTERVAL` | number | — | Traces export interval (ms) |
| `OTEL_TRACES_EXPORTER` | string | — | Traces exporter type |

### 5.2 Debug & Development

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `BETA_TRACING_ENDPOINT` | url | — | Endpoint URL for detailed beta trace reporting (requires `ENABLE_BETA_TRACING_DETAILED` and `tengu_trace_lantern`) |
| `DEBUG` | boolean | off | Enable general debug logging |
| `DEBUG_SDK` | boolean | off | Enable SDK-specific debug logging |
| `EMBEDDED_SEARCH_TOOLS` | boolean | off | Enable embedded search tool experiments |
| `VCR_RECORD` | boolean | off | Enable VCR cassette recording in CI (bypass fixture replay) |

---

## 6. Cloud Providers

### 6.1 AWS / Bedrock

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `AWS_ACCESS_KEY_ID` | string | — | AWS access key (used or written by login flow) |
| `AWS_BEARER_TOKEN_BEDROCK` | string | — | Bearer token for Bedrock API |
| `AWS_DEFAULT_REGION` | string | `us-east-1` | AWS region fallback (checked after `AWS_REGION`) |
| `AWS_LAMBDA_BENCHMARK_MODE` | boolean (`=== "1"`) | off | Enable Lambda benchmark mode |
| `AWS_LOGIN_CACHE_DIRECTORY` | path | — | Directory for AWS login cache |
| `AWS_PROFILE` | string | — | AWS CLI profile name |
| `AWS_REGION` | string | `us-east-1` | AWS region for API calls |
| `AWS_SECRET_ACCESS_KEY` | string | — | AWS secret key (used or written by login flow) |
| `AWS_SESSION_TOKEN` | string | — | AWS session token (used or written by login flow) |

### 6.2 Azure / Foundry

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `AZURE_ADDITIONALLY_ALLOWED_TENANTS` | string | — | Additional allowed Azure tenant IDs |
| `AZURE_AUTHORITY_HOST` | url | — | Azure authority / AAD host URL |
| `AZURE_CLIENT_CERTIFICATE_PASSWORD` | string | — | Client certificate password |
| `AZURE_CLIENT_CERTIFICATE_PATH` | path | — | Path to client certificate |
| `AZURE_CLIENT_ID` | string | — | Azure app client ID |
| `AZURE_CLIENT_SECRET` | string | — | Azure app client secret |
| `AZURE_CLIENT_SEND_CERTIFICATE_CHAIN` | boolean | off | Send full certificate chain in requests |
| `AZURE_FEDERATED_TOKEN_FILE` | path | — | Path to federated identity token file |
| `AZURE_IDENTITY_DISABLE_MULTITENANTAUTH` | boolean (truthy) | off | Disable multi-tenant authentication |
| `AZURE_PASSWORD` | string | — | Azure password for user credential auth |
| `AZURE_POD_IDENTITY_AUTHORITY_HOST` | url | — | Pod identity authority host |
| `AZURE_REGIONAL_AUTHORITY_NAME` | string | — | Regional authority name |
| `AZURE_TENANT_ID` | string | — | Azure tenant ID |
| `AZURE_TOKEN_CREDENTIALS` | string | — | Token credential mode (`prod` or `dev`) |
| `AZURE_USERNAME` | string | — | Azure username for user credential auth |

### 6.3 GCP / Vertex AI

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `CLOUD_ML_REGION` | string | `us-east5` | Default Vertex AI region; used as fallback when no per-model `VERTEX_REGION_*` var is set |
| `DETECT_GCP_RETRIES` | number | `0` | Number of retries for GCP environment detection metadata probe |
| `GAE_MODULE_NAME` | string | — | App Engine module name; presence indicates a GAE deployment |
| `GAE_SERVICE` | string | — | App Engine service name; presence indicates a GAE deployment |
| `GCE_METADATA_HOST` | url | — | Override the GCE metadata server hostname / IP |
| `GCE_METADATA_IP` | string | — | Override the GCE metadata server IP address |
| `GCLOUD_PROJECT` | string | — | GCP project ID (legacy form; checked before `GOOGLE_CLOUD_PROJECT`) |
| `GOOGLE_APPLICATION_CREDENTIALS` | path | — | Path to GCP service account JSON credentials file |
| `GOOGLE_CLOUD_PROJECT` | string | — | GCP project ID (standard form) |
| `K_CONFIGURATION` | string | — | Cloud Run configuration name; presence indicates a Cloud Run deployment |
| `K_SERVICE` | string | — | Cloud Run service name; presence indicates a Cloud Run deployment |
| `METADATA_SERVER_DETECTION` | string | — | Controls GCE metadata probing behavior: `assume-present`, `none`, `bios-only`, `ping-only` |
| `VERTEX_BASE_URL` | url | — | Override base URL for Vertex AI API calls |
| `VERTEX_REGION_CLAUDE_3_5_HAIKU` | string | — | Per-model Vertex region override for `claude-3-5-haiku` |
| `VERTEX_REGION_CLAUDE_3_5_SONNET` | string | — | Per-model Vertex region override for `claude-3-5-sonnet` |
| `VERTEX_REGION_CLAUDE_3_7_SONNET` | string | — | Per-model Vertex region override for `claude-3-7-sonnet` |
| `VERTEX_REGION_CLAUDE_4_0_OPUS` | string | — | Per-model Vertex region override for `claude-opus-4` |
| `VERTEX_REGION_CLAUDE_4_0_SONNET` | string | — | Per-model Vertex region override for `claude-sonnet-4` |
| `VERTEX_REGION_CLAUDE_4_1_OPUS` | string | — | Per-model Vertex region override for `claude-opus-4-1` |
| `VERTEX_REGION_CLAUDE_4_5_SONNET` | string | — | Per-model Vertex region override for `claude-sonnet-4-5` |
| `VERTEX_REGION_CLAUDE_4_6_SONNET` | string | — | Per-model Vertex region override for `claude-sonnet-4-6` |
| `VERTEX_REGION_CLAUDE_HAIKU_4_5` | string | — | Per-model Vertex region override for `claude-haiku-4-5` |

---

## 7. Terminal & Display Detection

Read-only environment variables used to detect the active terminal emulator and display capabilities. Not user-settable feature flags, but they directly affect color support, terminal title writes, and IDE detection.

| Variable | Type | Default | Terminal / IDE Detected |
|----------|------|---------|-------------------------|
| `ALACRITTY_LOG` | string | — | Alacritty terminal |
| `COLORTERM` | string | — | `truecolor` → 16M-color support; any value → basic color |
| `ConEmuANSI` | string | — | ConEmu terminal (also checks `ConEmuPID`, `ConEmuTask`) |
| `ConEmuPID` | string | — | ConEmu process ID (used alongside `ConEmuANSI`) |
| `ConEmuTask` | string | — | ConEmu task name (used alongside `ConEmuANSI`) |
| `CURSOR_TRACE_ID` | string | — | Cursor IDE (AI code editor) |
| `FORCE_COLOR` | string/number | — | Force color output: `true`/`1-3` (level), `false`/`0` (none) |
| `GNOME_TERMINAL_SERVICE` | string | — | GNOME Terminal |
| `ITERM_SESSION_ID` | string | — | iTerm2 (macOS); detected via `TERM_PROGRAM=iTerm.app` |
| `KITTY_WINDOW_ID` | string | — | Kitty terminal (fallback; primary check is `TERM` containing `kitty`) |
| `KONSOLE_VERSION` | string | — | Konsole (KDE) terminal |
| `LC_TERMINAL` | string | — | Terminal identifier on macOS (e.g., `iTerm2`) |
| `MSYSTEM` | string | — | MSYS2 / Git Bash (returns value lowercased as terminal name) |
| `SESSIONNAME` | string | — | Windows Cygwin detection (checked with `TERM=cygwin`) |
| `SSH_CLIENT` | string | — | SSH session detection (also triggers `ssh-session` terminal name) |
| `SSH_CONNECTION` | string | — | SSH session detection (primary form) |
| `SSH_TTY` | string | — | SSH TTY detection |
| `STY` | string | — | GNU Screen session |
| `TERM` | string | — | Terminal type (`xterm-ghostty` → ghostty, `*kitty*` → kitty, `dumb` → no color, etc.) |
| `TERM_PROGRAM` | string | — | Terminal application name (`iTerm.app`, `Apple_Terminal`, etc.) |
| `TERM_PROGRAM_VERSION` | string | — | Terminal application version (used with `TERM_PROGRAM`) |
| `TERMINAL_EMULATOR` | string | — | `JetBrains-JediTerm` → pycharm/JetBrains IDE |
| `TEAMCITY_VERSION` | string | — | TeamCity CI; used to determine color level |
| `TILIX_ID` | string | — | Tilix terminal |
| `TMUX` | string | — | tmux multiplexer |
| `VTE_VERSION` | string | — | VTE-based terminal (e.g., Xfce4 Terminal) |
| `VSCODE_GIT_ASKPASS_MAIN` | string | — | VS Code Git helper path; inspected for `cursor`, `windsurf`, `antigravity` substrings |
| `WT_SESSION` | string | — | Windows Terminal |
| `XTERM_VERSION` | string | — | xterm terminal |
| `__CFBundleIdentifier` | string | — | macOS app bundle ID; used to detect VSCodium, Windsurf, Android Studio, Conductor |

---

## 8. System & Paths

System-level paths and OS identification variables read for configuration and path resolution.

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `APPDATA` | path | — | Windows Roaming AppData directory (config/cache location on Windows) |
| `BROWSER` | string | — | Browser executable or command for opening URLs |
| `HOME` | path | — | User home directory |
| `MSYSTEM` | string | — | MSYS2 subsystem (also used for terminal detection; see §7) |
| `OSTYPE` | string | — | OS type string; `cygwin` or `msys` indicate Cygwin/MSYS environment |
| `PATH` | string | — | Executable search path |
| `ProgramData` | path | — | Windows ProgramData directory (Azure arc agent token path) |
| `ProgramFiles` | path | — | Windows Program Files directory (Azure arc agent binary path) |
| `PWD` | path | — | Current working directory (used when `process.cwd()` is unavailable) |
| `SHELL` | path | — | User's shell executable |
| `SystemRoot` | path | — | Windows system root (also checks `SYSTEMROOT`) |
| `TEMP` | path | — | Windows temp directory fallback (defaults to `C:\Temp`) |
| `TMPDIR` | path | — | Unix temp directory (passed to Node.js temp operations) |
| `USER` | string | — | Unix username (falls back to `USERNAME`, then `"default"`) |
| `USERNAME` | string | — | Windows username (fallback for `USER`) |
| `USERPROFILE` | path | — | Windows user profile / home directory |
| `WSL_DISTRO_NAME` | string | — | WSL distribution name; presence indicates WSL environment |
| `XDG_CONFIG_HOME` | path | — | XDG config directory (used for config file location) |
| `XDG_RUNTIME_DIR` | path | — | XDG runtime directory |

---

## 9. CI/CD & Git

Variables read in CI/CD environments. Some are used for telemetry metadata; others affect runtime behavior.

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `CI` | string | — | Generic CI indicator; presence triggers 1-color-level support in color detection |
| `CI_NAME` | string | — | CI system name; `codeship` enables basic color support |
| `CLAUDECODE` | string (`=== "1"`) | — | Set by Claude Code to `"1"` at launch; blocks nested sessions (with error message) |
| `GITHUB_ACTIONS` | boolean | — | GitHub Actions environment; gates collection of GitHub metadata |
| `GITHUB_ACTOR` | string | — | GitHub Actions actor (username of the triggering user) |
| `GITHUB_ACTOR_ID` | string | — | GitHub Actions actor user ID |
| `GITHUB_REPOSITORY` | string | — | GitHub repository (`owner/repo`) |
| `GITHUB_REPOSITORY_ID` | string | — | GitHub repository ID |
| `GITHUB_REPOSITORY_OWNER` | string | — | GitHub repository owner |
| `GITHUB_REPOSITORY_OWNER_ID` | string | — | GitHub repository owner ID |
| `IS_SANDBOX` | string (`!== "1"`) | — | Sandbox environment marker; checked in plugin and permission code to block certain actions |
| `VisualStudioVersion` | string | — | Visual Studio version string; presence indicates VS developer prompt |

---

## 10. MCP (Model Context Protocol)

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `MAX_MCP_OUTPUT_TOKENS` | number | 25000 | Maximum token count for MCP tool output |
| `MAX_THINKING_TOKENS` | number | — | Maximum thinking tokens; if set to a positive integer, enables extended thinking |
| `MCP_CONNECTION_NONBLOCKING` | boolean | off | Use non-blocking MCP connection mode (affects connection lifecycle) |
| `MCP_TIMEOUT` | number | 30000 | MCP server connection / operation timeout (ms) |
| `MCP_TOOL_TIMEOUT` | number | — | Individual MCP tool call timeout (ms); overrides per-server defaults |

---

## 11. Bash Tool Variables

Variables that configure behavior of the Bash tool. These appear in the allowed env-var passthrough list and are read by the bash tool's execution context.

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `BASH_DEFAULT_TIMEOUT_MS` | number | — | Default timeout for bash command execution (ms) |
| `BASH_MAX_OUTPUT_LENGTH` | number | — | Maximum character length for bash tool output before truncation |
| `BASH_MAX_TIMEOUT_MS` | number | — | Maximum timeout cap for bash command execution (ms) |

---

## 12. Build Tools & Vendor Libraries

Variables used by bundled third-party libraries. Not Claude Code–specific, but present in the `cli.js` bundle.

| Variable | Type | Default | Purpose |
|----------|------|---------|----------|
| `CHOKIDAR_INTERVAL` | number | — | Polling interval (ms) for chokidar file watcher (polling mode) |
| `CHOKIDAR_USEPOLLING` | boolean | — | Force chokidar to use filesystem polling instead of native events |
| `GRACEFUL_FS_PLATFORM` | string | `process.platform` | Override the platform string used by `graceful-fs` |
| `PKG_CONFIG_PATH` | path | — | `pkg-config` search path; injected when running native addon build steps |
| `UNDICI_NO_FG` | boolean | — | Disable undici's diagnostics channel / foreground reporting |

---

## 13. Cloud Platform Detection

Read-only environment variables used at startup to identify the hosting environment. Not user-settable feature flags; they affect environment classification and telemetry tags.

| Variable | Platform Detected |
|----------|-------------------|
| `APP_URL` (contains `ondigitalocean.app`) | DigitalOcean App Platform |
| `APPVEYOR` | AppVeyor CI |
| `AWS_EXECUTION_ENV` | AWS ECS Fargate / ECS EC2 |
| `AWS_LAMBDA_FUNCTION_NAME` | AWS Lambda |
| `AZURE_FUNCTIONS_ENVIRONMENT` | Azure Functions |
| `BUILDKITE` | Buildkite CI |
| `CF_PAGES` | Cloudflare Pages |
| `CIRCLECI` | CircleCI |
| `CODESPACES` | GitHub Codespaces |
| `DENO_DEPLOYMENT_ID` | Deno Deploy |
| `DRONE` | Drone CI |
| `DYNO` | Heroku |
| `FLY_APP_NAME` / `FLY_MACHINE_ID` | Fly.io |
| `GITPOD_WORKSPACE_ID` | Gitpod |
| `GITLAB_CI` | GitLab CI |
| `GITHUB_ACTIONS` | GitHub Actions |
| `GOOGLE_CLOUD_PROJECT` | GCP (generic) |
| `K_SERVICE` | GCP Cloud Run |
| `KUBERNETES_SERVICE_HOST` | Kubernetes |
| `NETLIFY` | Netlify |
| `PROJECT_DOMAIN` | Glitch |
| `RAILWAY_ENVIRONMENT_NAME` / `RAILWAY_SERVICE_NAME` | Railway |
| `RENDER` | Render.com |
| `REPL_ID` / `REPL_SLUG` | Replit |
| `SPACE_CREATOR_USER_ID` | HuggingFace Spaces |
| `TRAVIS` | Travis CI |
| `VERCEL` | Vercel |
| `WEBSITE_SITE_NAME` / `WEBSITE_SKU` | Azure App Service |

---

## Summary Statistics

| Category | Count |
|----------|-------|
| `CLAUDE_CODE_*` variables | 117 |
| `CLAUDE_*` variables (non-CODE) | 15 |
| `ANTHROPIC_*` variables | 17 |
| `DISABLE_*` / `ENABLE_*` gates | 29 |
| `USE_*` / `LOCAL_*` gates | 5 |
| `OTEL_*` telemetry variables | 28 |
| Debug / misc vars | 5 |
| AWS credential / config vars | 9 |
| Azure SDK credential vars | 15 |
| GCP / Vertex AI vars | 21 |
| Terminal & display detection vars | 29 |
| System & path vars | 18 |
| CI/CD & Git vars | 12 |
| MCP vars | 5 |
| Bash tool vars | 3 |
| Build tools / vendor vars | 5 |
| Cloud platform detection vars | ~28 |
| **Total unique env vars** | **~360+** |
