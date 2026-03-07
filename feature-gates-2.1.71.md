# Claude Code v2.1.71 — Feature Gates Reference

Compiled from two research files:
- `env-gates.md` — all `process.env.*` feature gates (438 lines, ~300+ unique vars)
- `statsig-gates.md` — all Statsig/GrowthBook feature flags (238 lines, 73 unique flags)

Source bundle: `cli.js` v2.1.71, build `2026-03-06T22:45:36Z`, 612,918 lines.

---

## Overview

This document catalogs all feature gates found in Claude Code v2.1.71:

- **~300+ environment variable gates** across CLAUDE_CODE_*, CLAUDE_*, ANTHROPIC_*, and misc prefixes
- **73 Statsig/GrowthBook runtime flags** across 4 check-function types
- **Total unique control surfaces: ~370+**

Environment variable gate types:
- `boolean` — parsed by `$1()` (truthy: `"1"`, `"true"`, `"yes"`) or `dz()` (negation form)
- `string` / `number` / `path` / `url` — raw value reads

---

## Table of Contents

1. [Environment Variable Gates](#1-environment-variable-gates)
   - 1.1 [CLAUDE_CODE_* Variables (100 vars)](#11-claude_code_-variables-100-vars)
   - 1.2 [CLAUDE_* Variables (15 vars)](#12-claude_-variables-15-vars)
   - 1.3 [ANTHROPIC_* Variables (16 vars)](#13-anthropic_-variables-16-vars)
   - 1.4 [Other Feature Gates (~130 vars)](#14-other-feature-gates-130-vars)
2. [Statsig Feature Gates](#2-statsig-feature-gates)
   - 2.1 [p8() Boolean Gates (57 gates)](#21-p8-boolean-gates-57-gates)
   - 2.2 [jU() TTL-Cached Gates (5 gates)](#22-ju-ttl-cached-gates-5-gates)
   - 2.3 [A_() GrowthBook Experiments (7 experiments)](#23-a_-growthbook-experiments-7-experiments)
   - 2.4 [UL() Dynamic Configs (4 configs)](#24-ul-dynamic-configs-4-configs)
3. [Summary Statistics](#3-summary-statistics)

---

## 1. Environment Variable Gates

### 1.1 CLAUDE_CODE_* Variables (100 vars)

All variables with the `CLAUDE_CODE_` prefix. Gate type `boolean` means parsed via `$1()` unless noted.

| Variable | Gate Type | Default | Purpose |
|----------|-----------|---------|----------|
| `CLAUDE_CODE_SLOW_OPERATION_THRESHOLD_MS` | number | — | Threshold (ms) to log slow operations |
| `CLAUDE_CODE_DEBUG_LOGS_DIR` | path | — | Directory for debug log files |
| `CLAUDE_CODE_DEBUG_LOG_LEVEL` | string | — | Debug log verbosity level (`debug`, `info`, etc.) |
| `CLAUDE_CODE_PROFILE_STARTUP` | boolean (`=== "1"`) | off | Enable startup performance profiling |
| `CLAUDE_CODE_CUSTOM_OAUTH_URL` | url | — | Custom OAuth server URL (overrides default) |
| `CLAUDE_CODE_OAUTH_CLIENT_ID` | string | — | Custom OAuth client ID |
| `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` | string | — | Pre-set OAuth refresh token |
| `CLAUDE_CODE_OAUTH_SCOPES` | string | — | Override OAuth scopes |
| `CLAUDE_CODE_HOST_PLATFORM` | string | — | Override host platform identifier |
| `CLAUDE_CODE_USE_BEDROCK` | boolean | off | Use Amazon Bedrock as the API backend |
| `CLAUDE_CODE_USE_VERTEX` | boolean | off | Use Google Vertex AI as the API backend |
| `CLAUDE_CODE_USE_FOUNDRY` | boolean | off | Use Azure AI Foundry as the API backend |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | boolean (truthy) | off | Block telemetry, analytics, and non-essential network calls |
| `CLAUDE_CODE_GIT_BASH_PATH` | path | — | Custom path to git-bash executable (Windows) |
| `CLAUDE_CODE_GLOB_TIMEOUT_SECONDS` | number | — | Timeout in seconds for glob file searches |
| `CLAUDE_CODE_USE_COWORK_PLUGINS` | boolean | off | Enable collaboration/cowork plugin system |
| `CLAUDE_CODE_PLUGIN_CACHE_DIR` | path | — | Custom directory for plugin cache |
| `CLAUDE_CODE_PLUGIN_SEED_DIR` | path | — | Seed directory for initial plugin data |
| `CLAUDE_CODE_BUBBLEWRAP` | boolean (`=== "1"`) | off | Enable Bubblewrap sandboxing on Linux |
| `CLAUDE_CODE_ENTRYPOINT` | string | — | Identifies how Claude Code was launched (`cli`, `sdk-ts`, `sdk-py`, `sdk-cli`) |
| `CLAUDE_CODE_DIAGNOSTICS_FILE` | path | — | Path to write diagnostics output |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | boolean | off | Disable automatic memory/CLAUDE.md loading |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | path | — | Directory for remote session memory storage |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | boolean | off | Enable experimental multi-agent team support |
| `CLAUDE_CODE_ALWAYS_ENABLE_EFFORT` | boolean | off | Always enable extended thinking/effort mode |
| `CLAUDE_CODE_EFFORT_LEVEL` | string | — | Set effort/thinking level explicitly |
| `CLAUDE_CODE_ATTRIBUTION_HEADER` | boolean (dz) | on | Disable attribution header when set (negation gate) |
| `CLAUDE_CODE_CLIENT_CERT` | path | — | Path to client TLS certificate file |
| `CLAUDE_CODE_CLIENT_KEY` | path | — | Path to client TLS private key file |
| `CLAUDE_CODE_CLIENT_KEY_PASSPHRASE` | string | — | Passphrase for encrypted client TLS key |
| `CLAUDE_CODE_PROXY_RESOLVES_HOSTS` | boolean | off | Proxy server resolves hostnames (no local DNS lookup) |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | boolean | off | Skip Bedrock auth and use ambient AWS credentials |
| `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP` | boolean | off | Disable legacy model name → new name remapping |
| `CLAUDE_CODE_DISABLE_FAST_MODE` | boolean | off | Disable fast mode (always use the full model) |
| `CLAUDE_CODE_ACCESSIBILITY` | boolean | off | Enable accessibility/screen-reader mode |
| `CLAUDE_CODE_SSE_PORT` | number | — | Port for Server-Sent Events (SSE) server |
| `CLAUDE_CODE_IDE_SKIP_VALID_CHECK` | boolean | off | Skip IDE integration validation check |
| `CLAUDE_CODE_IDE_SKIP_AUTO_INSTALL` | boolean | off | Skip auto-installation of IDE extension |
| `CLAUDE_CODE_IDE_HOST_OVERRIDE` | string | — | Override IDE server hostname |
| `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` | string | — | File descriptor for WebSocket auth token |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | string | — | Session access token for authentication |
| `CLAUDE_CODE_ORGANIZATION_UUID` | string | — | Organization UUID for multi-org setups |
| `CLAUDE_CODE_CONTAINER_ID` | string | — | Container identifier for remote sessions |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | string | — | Remote session identifier |
| `CLAUDE_CODE_ADDITIONAL_PROTECTION` | boolean | off | Enable additional security/protection layer |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | boolean | off | Skip Azure AI Foundry authentication |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | boolean | off | Skip Vertex AI authentication |
| `CLAUDE_CODE_TEST_FIXTURES_ROOT` | path | — | Root directory for test fixtures |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` | boolean | off | Scan additional directories for CLAUDE.md files |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | boolean (truthy) | off | Disable all CLAUDE.md file loading |
| `CLAUDE_CODE_SIMPLE` | boolean | off | Enable simplified/lite mode (fewer features) |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | boolean | off | Force plan-before-execute mode |
| `CLAUDE_CODE_ENABLE_TASKS` | boolean | off | Enable task list feature |
| `CLAUDE_CODE_TASK_LIST_ID` | string | — | Pre-set task list ID |
| `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | boolean | off | Skip saving prompt history to disk |
| `CLAUDE_CODE_MAX_RETRIES` | number | — | Maximum API retry attempts |
| `CLAUDE_CODE_DONT_INHERIT_ENV` | boolean (truthy) | off | Don't inherit parent process environment in subprocesses |
| `CLAUDE_CODE_SHELL_PREFIX` | string | — | Prefix commands for shell invocation |
| `CLAUDE_CODE_SHELL` | string | — | Override shell executable path |
| `CLAUDE_CODE_TMPDIR` | path | `/tmp` | Override temp directory |
| `CLAUDE_CODE_ENVIRONMENT_KIND` | string | — | Environment type identifier (`bridge`, etc.) |
| `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | number | — | Max tokens for file read tool output |
| `CLAUDE_CODE_SEARCH_HINTS_IN_LIST` | boolean | off | Show search hints in deferred tool list UI |
| `CLAUDE_CODE_MCP_INSTR_DELTA` | boolean | off | Enable MCP instruction delta mode (incremental updates) |
| `CLAUDE_CODE_REMOTE` | boolean | off | Enable remote/headless mode |
| `CLAUDE_CODE_DISABLE_ATTACHMENTS` | boolean | off | Disable file attachment support |
| `CLAUDE_CODE_ENABLE_TOKEN_USAGE_ATTACHMENT` | boolean | off | Enable token usage in attachments |
| `CLAUDE_CODE_REMOTE_SEND_KEEPALIVES` | boolean | off | Send keepalive pings in remote mode |
| `CLAUDE_CODE_SUBAGENT_MODEL` | string | — | Override model used for subagents |
| `CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE` | boolean | off | Use zip-based plugin cache |
| `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS` | number | — | Timeout for plugin git operations (ms) |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | number | — | Override the context blocking limit |
| `CLAUDE_CODE_PERFETTO_TRACE` | path | — | Path to output Perfetto performance trace |
| `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA` | string | — | Enable enhanced telemetry beta features |
| `CLAUDE_CODE_ACCOUNT_UUID` | string | — | Account UUID override for telemetry |
| `CLAUDE_CODE_USER_EMAIL` | string | — | User email override for telemetry |
| `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS` | number | — | Datadog metrics flush interval (ms) |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | boolean | off | Enable OTEL/third-party telemetry export |
| `CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS` | number | 2000 | OTEL shutdown timeout (ms) |
| `CLAUDE_CODE_OTEL_FLUSH_TIMEOUT_MS` | number | 5000 | OTEL flush timeout (ms) |
| `CLAUDE_CODE_OTEL_HEADERS_HELPER_DEBOUNCE_MS` | number | — | Debounce interval for OTEL header helper |
| `CLAUDE_CODE_STALL_TIMEOUT_MS_FOR_TESTING` | number | — | Override stall detection timeout (testing only) |
| `CLAUDE_CODE_TWO_STAGE_CLASSIFIER` | boolean | off | Enable two-stage safety classifier |
| `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` | boolean | off | Disable file state checkpoint snapshots |
| `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING` | boolean | off | Enable file checkpointing in SDK mode |
| `CLAUDE_CODE_SYNTAX_HIGHLIGHT` | string (dz) | on | Disable syntax highlighting when set to falsy value |
| `CLAUDE_CODE_BASE_REF` | string | — | Git base ref for diff operations |
| `CLAUDE_CODE_PLAN_V2_AGENT_COUNT` | number | — | Number of agents for Plan v2 |
| `CLAUDE_CODE_PLAN_V2_EXPLORE_AGENT_COUNT` | number | — | Number of explore agents for Plan v2 |
| `CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE` | string | — | Interview phase setting for plan mode |
| `CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES` | boolean | off | Emit tool use summary telemetry events |
| `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` | string | — | Enable/disable prompt suggestions (`"false"` to disable) |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | boolean | off | Disable background task processing |
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` | number | 10 | Maximum concurrent tool calls |
| `CLAUDE_CODE_ENVIRONMENT_RUNNER_VERSION` | string | — | Version of the environment runner |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | boolean | off | Disable 1M token context window support |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | boolean | off | Disable background task processing |
| `CLAUDE_CODE_STREAMING_TEXT` | boolean | off | Enable streaming text output mode (also gated by `tengu_streaming_text`) |
| `CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING` | boolean | off | Enable fine-grained tool streaming (also gated by `tengu_fgts`) |

---

### 1.2 CLAUDE_* Variables (15 vars)

Variables with the `CLAUDE_` prefix that are NOT `CLAUDE_CODE_*`.

| Variable | Gate Type | Default | Purpose |
|----------|-----------|---------|----------|
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | boolean | off | Keep bash subshell in project working directory across calls |
| `CLAUDE_AGENT_SDK_VERSION` | string | — | Agent SDK version string (injected into User-Agent header) |
| `CLAUDE_AGENT_SDK_CLIENT_APP` | string | — | Client app name (injected into User-Agent header) |
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` | boolean | off | Disable built-in agent type definitions from SDK |
| `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` | boolean | off | Disable MCP tool name prefixing in SDK mode |
| `CLAUDE_ENV_FILE` | path | — | Path to `.env` file to load at startup |
| `CLAUDE_TMPDIR` | path | — | Temp directory override for bash tool |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | string | — | Override autocompact threshold percentage |
| `CLAUDE_AFTER_LAST_COMPACT` | boolean | off | Post-compact behavior control flag |
| `CLAUDE_AUTO_BACKGROUND_TASKS` | boolean | off | Auto-enable background task scheduling |
| `CLAUDE_REPL_MODE` | boolean | off | Enable REPL (interactive) mode |
| `CLAUDE_ENABLE_STREAM_WATCHDOG` | boolean | off | Enable stream watchdog timeout |
| `CLAUDE_BRIDGE_USE_CCR_V2` | boolean | off | Use CCR v2 bridge protocol |
| `CLAUDE_DEBUG` | boolean | off | Enable Claude-specific debug output |
| `CLAUDE_FORCE_DISPLAY_SURVEY` | boolean (truthy) | off | Force display of user survey |

---

### 1.3 ANTHROPIC_* Variables (16 vars)

Variables with the `ANTHROPIC_` prefix — API configuration and model overrides.

| Variable | Gate Type | Default | Purpose |
|----------|-----------|---------|----------|
| `ANTHROPIC_BASE_URL` | url | — | Override Anthropic API base URL |
| `ANTHROPIC_API_KEY` | string | — | Anthropic API key for authentication |
| `ANTHROPIC_AUTH_TOKEN` | string | — | Bearer token for Anthropic API (alternative to API key) |
| `ANTHROPIC_CUSTOM_HEADERS` | string | — | Additional headers to send with all API requests |
| `ANTHROPIC_BEDROCK_BASE_URL` | url | — | Custom Bedrock endpoint URL |
| `ANTHROPIC_SMALL_FAST_MODEL` | string | — | Override the small/fast model name |
| `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` | string | — | AWS region for small/fast model on Bedrock |
| `ANTHROPIC_MODEL` | string | — | Override the primary model name |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | string | — | Override Opus model identifier |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | string | — | Override Sonnet model identifier |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | string | — | Override Haiku model identifier |
| `ANTHROPIC_FOUNDRY_API_KEY` | string | — | API key for Azure AI Foundry |
| `ANTHROPIC_FOUNDRY_BASE_URL` | url | — | Base URL for Azure AI Foundry endpoint |
| `ANTHROPIC_FOUNDRY_RESOURCE` | string | — | Azure Foundry resource name |
| `ANTHROPIC_VERTEX_PROJECT_ID` | string | — | Google Cloud project ID for Vertex AI |
| `ANTHROPIC_BETAS` | string | — | Comma-separated list of beta feature flags to enable |

---

### 1.4 Other Feature Gates (~130 vars)

#### DISABLE_* / ENABLE_* Gates (25 vars)

Explicit feature disable/enable toggles not covered by the CLAUDE_CODE_* prefix.

| Variable | Gate Type | Default | Purpose |
|----------|-----------|---------|----------|
| `DISABLE_ERROR_REPORTING` | boolean (truthy) | off | Disable Sentry/error reporting |
| `DISABLE_TELEMETRY` | boolean (truthy) | off | Disable all telemetry |
| `ENABLE_CLAUDE_CODE_SM_COMPACT` | boolean | off | Force-enable session memory compaction |
| `DISABLE_CLAUDE_CODE_SM_COMPACT` | boolean | off | Force-disable session memory compaction |
| `ENABLE_BETA_TRACING_DETAILED` | boolean | off | Enable detailed beta tracing output |
| `DISABLE_COMPACT` | boolean | off | Disable context compaction feature entirely |
| `DISABLE_AUTO_COMPACT` | boolean | off | Disable automatic compaction (allow manual only) |
| `ENABLE_ENHANCED_TELEMETRY_BETA` | string | — | Enable enhanced telemetry beta features |
| `ENABLE_CLAUDEAI_MCP_SERVERS` | boolean (dz) | on | Enable Claude.ai MCP server integration (disabled when set) |
| `DISABLE_INSTALLATION_CHECKS` | boolean | off | Skip installation validation checks |
| `DISABLE_EXTRA_USAGE_COMMAND` | boolean (truthy) | off | Hide the `/usage` command |
| `DISABLE_FEEDBACK_COMMAND` | boolean (truthy) | off | Hide the `/feedback` command |
| `DISABLE_BUG_COMMAND` | boolean (truthy) | off | Hide the `/bug` command |
| `DISABLE_DOCTOR_COMMAND` | boolean (truthy) | off | Hide the `/doctor` command |
| `DISABLE_LOGIN_COMMAND` | boolean (truthy) | off | Hide the `/login` command |
| `DISABLE_LOGOUT_COMMAND` | boolean (truthy) | off | Hide the `/logout` command |
| `DISABLE_INSTALL_GITHUB_APP_COMMAND` | boolean (truthy) | off | Hide the GitHub App install command |
| `DISABLE_UPGRADE_COMMAND` | boolean (truthy) | off | Hide the `/upgrade` command |
| `ENABLE_TOOL_SEARCH` | string | — | Enable/configure tool search mode (`"standard"`, etc.) |
| `ENABLE_MCP_LARGE_OUTPUT_FILES` | boolean (dz) | on | Enable large output file handling for MCP (disabled when set) |
| `DISABLE_PROMPT_CACHING` | boolean | off | Disable prompt caching entirely |
| `DISABLE_PROMPT_CACHING_HAIKU` | boolean | off | Disable prompt caching for Haiku model |
| `DISABLE_PROMPT_CACHING_SONNET` | boolean | off | Disable prompt caching for Sonnet model |
| `DISABLE_PROMPT_CACHING_OPUS` | boolean | off | Disable prompt caching for Opus model |
| `ENABLE_PROMPT_CACHING_1H_BEDROCK` | boolean | off | Enable 1-hour prompt caching on Bedrock |
| `DISABLE_INTERLEAVED_THINKING` | boolean | off | Disable interleaved thinking/reasoning tokens |
| `DISABLE_AUTOUPDATER` | boolean | off | Disable auto-update mechanism |
| `DISABLE_COST_WARNINGS` | boolean | off | Suppress cost warning UI messages |
| `DISABLE_AUTO_MIGRATE_TO_NATIVE` | boolean | off | Skip migration to native auth method |

#### USE_* / LOCAL_* Gates (5 vars)

| Variable | Gate Type | Default | Purpose |
|----------|-----------|---------|----------|
| `USE_BUILTIN_RIPGREP` | boolean (dz) | on | Use bundled ripgrep binary (disabled when set) |
| `USE_API_CONTEXT_MANAGEMENT` | boolean | off | Delegate context management to the API |
| `USE_LOCAL_OAUTH` | boolean | off | Use local OAuth server instead of remote |
| `USE_STAGING_OAUTH` | boolean | off | Use staging OAuth endpoint |
| `LOCAL_BRIDGE` | boolean | off | Use local bridge connection |

#### OTEL / Telemetry Variables (19 vars)

| Variable | Gate Type | Default | Purpose |
|----------|-----------|---------|----------|
| `OTEL_LOG_USER_PROMPTS` | boolean | off | Include user prompts in OTEL logs (otherwise redacted) |
| `OTEL_LOG_TOOL_CONTENT` | boolean | off | Include tool call content in OTEL logs |
| `OTEL_LOG_TOOL_DETAILS` | boolean | off | Include tool call details in OTEL logs |
| `OTEL_EXPORTER_OTLP_HEADERS` | string | — | HTTP headers for OTLP exporter |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | url | — | OTLP exporter endpoint URL |
| `OTEL_EXPORTER_OTLP_INSECURE` | string | — | Allow insecure OTLP connections |
| `OTEL_EXPORTER_PROMETHEUS_HOST` | string | — | Prometheus exporter host |
| `OTEL_EXPORTER_PROMETHEUS_PORT` | number | — | Prometheus exporter port |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | string | `delta` | Metrics temporality preference (forced to `delta`) |
| `OTEL_METRICS_EXPORTER` | string | — | Metrics exporter type |
| `OTEL_METRIC_EXPORT_INTERVAL` | number | — | Metrics export interval (ms) |
| `OTEL_EXPORTER_OTLP_METRICS_PROTOCOL` | string | — | Metrics OTLP protocol |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | string | — | Default OTLP protocol |
| `OTEL_LOGS_EXPORTER` | string | — | Logs exporter type |
| `OTEL_EXPORTER_OTLP_LOGS_PROTOCOL` | string | — | Logs OTLP protocol |
| `OTEL_TRACES_EXPORTER` | string | — | Traces exporter type |
| `OTEL_EXPORTER_OTLP_TRACES_PROTOCOL` | string | — | Traces OTLP protocol |
| `OTEL_LOGS_EXPORT_INTERVAL` | number | — | Logs export interval (ms) |
| `OTEL_TRACES_EXPORT_INTERVAL` | number | — | Traces export interval (ms) |

#### Debug / General (3 vars)

| Variable | Gate Type | Default | Purpose |
|----------|-----------|---------|----------|
| `DEBUG` | boolean | off | Enable general debug logging |
| `DEBUG_SDK` | boolean | off | Enable SDK-specific debug logging |
| `VCR_RECORD` | boolean | off | Enable VCR cassette recording in CI (bypass fixture replay) |

#### Embedded / Search (1 var)

| Variable | Gate Type | Default | Purpose |
|----------|-----------|---------|----------|
| `EMBEDDED_SEARCH_TOOLS` | boolean | off | Enable embedded search tool experiments |

#### Cloud Platform Detection (26 vars — environment detection, not feature flags)

These are read-only environment detection variables. Not user-settable feature gates, but they affect runtime behavior and are checked at startup.

| Variable | Platform Detected |
|----------|-------------------|
| `CODESPACES` | GitHub Codespaces |
| `GITPOD_WORKSPACE_ID` | Gitpod |
| `REPL_ID` / `REPL_SLUG` | Replit |
| `PROJECT_DOMAIN` | Glitch |
| `VERCEL` | Vercel |
| `RAILWAY_ENVIRONMENT_NAME` / `RAILWAY_SERVICE_NAME` | Railway |
| `RENDER` | Render.com |
| `NETLIFY` | Netlify |
| `DYNO` | Heroku |
| `FLY_APP_NAME` / `FLY_MACHINE_ID` | Fly.io |
| `CF_PAGES` | Cloudflare Pages |
| `DENO_DEPLOYMENT_ID` | Deno Deploy |
| `AWS_LAMBDA_FUNCTION_NAME` | AWS Lambda |
| `AWS_EXECUTION_ENV` | AWS ECS Fargate / ECS EC2 |
| `K_SERVICE` | GCP Cloud Run |
| `GOOGLE_CLOUD_PROJECT` | GCP |
| `WEBSITE_SITE_NAME` / `WEBSITE_SKU` | Azure App Service |
| `AZURE_FUNCTIONS_ENVIRONMENT` | Azure Functions |
| `APP_URL` (contains `ondigitalocean.app`) | DigitalOcean App Platform |
| `SPACE_CREATOR_USER_ID` | HuggingFace Spaces |
| `GITHUB_ACTIONS` | GitHub Actions |
| `GITLAB_CI` | GitLab CI |
| `CIRCLECI` | CircleCI |
| `BUILDKITE` | Buildkite |
| `KUBERNETES_SERVICE_HOST` | Kubernetes |

#### AWS Credentials / Config (8 vars)

| Variable | Gate Type | Default | Purpose |
|----------|-----------|---------|----------|
| `AWS_REGION` / `AWS_DEFAULT_REGION` | string | `us-east-1` | AWS region for API calls |
| `AWS_BEARER_TOKEN_BEDROCK` | string | — | Bearer token for Bedrock API |
| `AWS_PROFILE` | string | — | AWS CLI profile name |
| `AWS_ACCESS_KEY_ID` | string | — | AWS access key (written by login flow) |
| `AWS_SECRET_ACCESS_KEY` | string | — | AWS secret key (written by login flow) |
| `AWS_SESSION_TOKEN` | string | — | AWS session token (written by login flow) |
| `AWS_LOGIN_CACHE_DIRECTORY` | path | — | Directory for AWS login cache |
| `AWS_LAMBDA_BENCHMARK_MODE` | boolean (`=== "1"`) | off | Enable Lambda benchmark mode |

#### Azure SDK Credentials (15 vars)

| Variable | Gate Type | Default | Purpose |
|----------|-----------|---------|----------|
| `AZURE_IDENTITY_DISABLE_MULTITENANTAUTH` | boolean (truthy) | off | Disable multi-tenant authentication |
| `AZURE_AUTHORITY_HOST` | url | — | Azure authority / AAD host URL |
| `AZURE_POD_IDENTITY_AUTHORITY_HOST` | url | — | Pod identity authority host |
| `AZURE_REGIONAL_AUTHORITY_NAME` | string | — | Regional authority name |
| `AZURE_TENANT_ID` | string | — | Azure tenant ID |
| `AZURE_CLIENT_ID` | string | — | Azure app client ID |
| `AZURE_CLIENT_SECRET` | string | — | Azure app client secret |
| `AZURE_FEDERATED_TOKEN_FILE` | path | — | Path to federated identity token file |
| `AZURE_ADDITIONALLY_ALLOWED_TENANTS` | string | — | Additional allowed tenant IDs |
| `AZURE_CLIENT_SEND_CERTIFICATE_CHAIN` | boolean | off | Send full certificate chain |
| `AZURE_CLIENT_CERTIFICATE_PATH` | path | — | Path to client certificate |
| `AZURE_CLIENT_CERTIFICATE_PASSWORD` | string | — | Client certificate password |
| `AZURE_USERNAME` | string | — | Azure username for user credential auth |
| `AZURE_PASSWORD` | string | — | Azure password for user credential auth |
| `AZURE_TOKEN_CREDENTIALS` | string | — | Token credential mode (`prod` or `dev`) |

#### System / Node.js Variables (~20 vars)

| Variable | Gate Type | Default | Purpose |
|----------|-----------|---------|----------|
| `NODE_OPTIONS` | string | — | Node.js runtime options (checked for `--inspect`) |
| `NODE_EXTRA_CA_CERTS` | path | — | Additional CA certificates (read and injected) |
| `NODE_V8_COVERAGE` | path | — | V8 coverage output directory |
| `NODE_DEBUG` | string | — | Node.js debug namespace filter |
| `UV_THREADPOOL_SIZE` | number | — | libuv thread pool size |
| `HTTP_PROXY` / `http_proxy` | url | — | HTTP proxy URL |
| `HTTPS_PROXY` / `https_proxy` | url | — | HTTPS proxy URL |
| `NO_PROXY` / `no_proxy` | string | — | Proxy bypass list |
| `HOME` | path | — | User home directory |
| `PATH` | string | — | Executable search path |
| `SHELL` | path | — | User's shell executable |
| `XDG_CONFIG_HOME` | path | — | XDG config directory |
| `XDG_RUNTIME_DIR` | path | — | XDG runtime directory |
| `TERM` | string | — | Terminal type identifier |
| `TERM_PROGRAM` | string | — | Terminal application name |
| `TERM_PROGRAM_VERSION` | string | — | Terminal application version |
| `TERMINAL_EMULATOR` | string | — | Terminal emulator identifier (e.g., JetBrains) |
| `TERMINATOR_UUID` | string | — | Terminator terminal UUID |
| `SESSIONNAME` | string | — | Windows session name (used for cygwin detection) |
| `COLORTERM` | string | — | Color terminal capability |
| `npm_package_config_libvips` | string | — | libvips path from npm package config |
| `PATHEXT` | string | — | Windows executable extensions |

---

## 2. Statsig Feature Gates

All runtime feature flags checked via minified SDK wrappers. Statsig is the primary system; GrowthBook is used for A/B experiments.

- **`p8(flagName, default)`** — synchronous Statsig gate check
- **`jU(flagName, default, ttlMs)`** — TTL-cached Statsig gate check
- **`A_(experimentName)`** — GrowthBook cached experiment variant
- **`UL(configName, default)`** — Statsig dynamic config object

### 2.1 p8() Boolean Gates (57 gates)

Total call-sites: 83. The `p8(flagName, defaultValue)` function is the primary Statsig gate checker — `defaultValue` is used when Statsig is unavailable.

#### Agent Teams / Background Agents (2 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_amber_flint` | `true` | Guards the agent-teams feature. Returns `false` if off, blocking experimental multi-agent swarm mode even when `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` env var is set. |
| `tengu_auto_background_agents` | `false` | Enables automatic background agent tasks; when on, sets a 120s idle timeout. Also gated by `CLAUDE_AUTO_BACKGROUND_TASKS` env var. |

#### Model / Effort / Fast Mode (8 gates)

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

#### Memory & Context Window (8 gates)

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

#### Compact / Cache / Streaming (7 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_compact_cache_prefix` | `false` | Enables prompt-cache sharing during compact: tries a cache-prefixed request first, then falls back to normal compact. |
| `tengu_compact_streaming_retry` | `false` | Enables retry logic during streaming compact (retries up to the configured limit on stream errors). |
| `tengu_system_prompt_global_cache` | `false` | Enables global system prompt cache. Splits prompt into sections with `cacheScope: "org"`. |
| `tengu_prompt_cache_1h_config` | `{}` | Config with `allowlist` of model-name prefixes eligible for 1-hour prompt caching (Bedrock path). |
| `tengu_fgts` | `false` | Enables fine-grained tool streaming (`eager_input_streaming: true`). Also gated by `CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING`. |
| `tengu_disable_streaming_to_non_streaming_fallback` | `false` | Prevents fallback from streaming to non-streaming mode on stream errors; throws instead. |
| `tengu_streaming_text` | `false` | Enables streaming text output mode. Also gated by `CLAUDE_CODE_STREAMING_TEXT` env var. |

#### Tool Search / TST (6 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_glacier_2xr` | `false` | Changes how deferred tool discovery is announced in system prompt: uses "system-reminder messages" wording. Also used for dynamic tool loading. |
| `tengu_defer_all_bn4` | `true` | When on, marks ALL tools (except ToolSearch itself) as deferred. |
| `tengu_tst_hint_m7r` | `false` | Enables search hints in the deferred tool list (shows `name — searchHint` format). Also gated by `CLAUDE_CODE_SEARCH_HINTS_IN_LIST`. |
| `tengu_basalt_3kr` | `false` | Enables MCP instructions delta feature: sends incremental MCP server instruction changes rather than full instructions. Also gated by `CLAUDE_CODE_MCP_INSTR_DELTA`. |
| `tengu_tool_search_unsupported_models` | `null` | Config array of model name substrings for which tool search is disabled. Falls back to hardcoded default list if null/empty. |
| `tengu_tst_kx7` | `false` | GrowthBook experiment fallback for tool search in tst-auto mode: when below auto-threshold but deferred tools are present, determines whether to enable tool search. |

#### MCP / Elicitation / Bridge (7 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_mcp_elicitation` | `false` | Enables MCP elicitation feature. Allows MCP servers to elicit user input via form/URL modes. |
| `tengu_ccr_bridge` | `false` | Enables the Claude Remote Control bridge. Also used as an async blocking gate via `Ck6()`. |
| `tengu_copper_bridge` | `false` | Enables the WebSocket bridge: returns bridge URL (production/staging/local) for Claude in Chrome connection. |
| `tengu_marble_lantern_disabled` | `false` | Disables the marble-lantern feature (appears to relate to fast/opus model availability toggle). |
| `tengu_quartz_lantern` | `false` | In remote mode (`CLAUDE_CODE_REMOTE`), enables diff computation for file write operations. |
| `tengu_remote_backend` | `false` | Controls whether `--remote` CLI flag can be used without a description argument. |
| `tengu_marble_whisper` | `false` | Enables word-highlighting feature: finds and highlights specific words in content. |

#### Output / Prompt Style (4 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_sotto_voce` | `false` | Adds an "Output efficiency" section to system prompt instructing Claude to be extra concise and lead with answers. |
| `tengu_bergotte_lantern` | `false` | Controls output polish instruction: when on, uses detailed polished-output guidance; when off, just says "Be short and concise." |
| `tengu_summarize_tool_results` | `false` | Adds instruction to write down important tool result info immediately, since tool results may be cleared. |
| `tengu_attribution_header` | `true` | Enables the `x-anthropic-billing-header` attribution header on API calls. Gated by `CLAUDE_CODE_ATTRIBUTION_HEADER` env. |

#### Permissions / Safety (6 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_marble_anvil` | `false` | Enables API context management beta. Also controls `clear_thinking_20251015` edit in thinking models. Only applied when model supports it. |
| `tengu_permission_explainer` | `false` | Enables permission explanation feature: uses an LLM call to explain why a tool permission is needed before asking the user. |
| `tengu_destructive_command_warning` | `false` | Enables destructive command detection in bash tool permission UI, warning users before potentially destructive shell commands. |
| `tengu_granite_whisper` | `false` | When on, skips repo text file size collection — short-circuits git ls-tree scan. |
| `tengu_scarf_coffee` | `false` | Adds a beta header for supported models — likely an interleaved-thinking or input-examples beta. |
| `tengu_quiet_hollow` | `false` | Adds a beta header for hiding thinking summaries (when not in agent mode and user hasn't enabled `showThinkingSummaries`). |

#### Tool Behavior (7 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_tool_input_aliasing` | `false` | Enables tool input parameter aliasing: maps deprecated parameter names to current ones using `inputParamAliases` from tool schema. |
| `tengu_chomp_inflection` | `true` | Controls prompt suggestion / chomp feature. When on, shows "Prompt suggestions" toggle in settings. |
| `tengu_plum_vx3` | `false` | In web search tool: when on, disables thinking and forces a specific model/tool-choice for the search sub-request. |
| `tengu_cork_m4q` | `false` | Controls bash permission system prompt format: when on, uses simplified "Command: X" format vs. the full spec-based format. |
| `tengu_lean_cast` | `false` | Switches compact summary system prompt template. |
| `tengu_amber_quartz` | `false` | Enables a feature related to settings-change handling. |
| `tengu_moth_copse` | `false` | Enables auto-memory retrieval: when on, searches for relevant memories based on the user's last message using semantic matching. |

#### Tracing / Observability (4 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_trace_lantern` | `false` | Enables detailed beta tracing. Requires `ENABLE_BETA_TRACING_DETAILED` env and `BETA_TRACING_ENDPOINT`. |
| `tengu_cicada_nap_ms` | `0` | Configures a cooldown (ms) for startup prefetch operations. Skips background prefetches if last run was within this window. |
| `tengu_miraculo_the_bard` | `false` | When false, calls startup prefetch function A; when true, calls alternate startup fetch path B. |
| `tengu_miraculo_the_bard2` | `false` | Controls a second separate startup prefetch operation. |

#### CI/CD / Version / Process (3 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_pid_based_version_locking` | `false` | Enables PID-based version locking: locks a specific Claude Code version to a process ID to prevent upgrade conflicts. |
| `tengu_immediate_model_command` | `false` | Enables the immediate model command feature. Allows `/model` command to take effect without restarting. |
| `tengu_chrome_auto_enable` | `false` | Auto-enables Claude in Chrome when the native binary is available. |

#### Keybinding / UI (3 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_keybinding_customization_release` | `false` | Enables keybinding customization feature. Controls whether custom key binding loading and settings UI are active. |
| `tengu_quiet_fern` | `false` | Pushed to VS Code extension via `experiment_gates` notification. Inferred: quiet/silent mode for VS Code integration. |
| `tengu_slate_ridge` | `false` | Pushed to VS Code extension via `experiment_gates` notification. Inferred: a VS Code UI/layout experiment. |

#### PR Status / Git (2 gates)

| Flag Name | Default | Purpose |
|-----------|---------|----------|
| `tengu_pr_status_cli` | `false` | Enables PR status footer in CLI. When on, shows a PR status section in the footer and adds a settings toggle. |
| `tengu_plan_mode_interview_phase` | `false` | Enables interview phase in plan mode: adds an exploration/interview step before implementation. Also gated by `CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE` env. |

---

### 2.2 jU() TTL-Cached Gates (5 gates)

The `jU(flagName, defaultValue, ttlMs)` function caches gate results for `ttlMs` milliseconds, reducing Statsig calls for frequently-checked gates.

| Flag Name | Default | TTL | Purpose |
|-----------|---------|-----|---------|
| `tengu_kairos_cron` | `false` | 300,000 ms (5 min) | Enables the Kairos cron/scheduling feature: allows creating one-shot and recurring scheduled tasks within a session. |
| `tengu_iron_gate_closed` | `true` | via `aSz` | Controls auto-mode classifier failure behavior: when on (fail-closed), denies with retry guidance when the classifier is unavailable; when off, allows through. |
| `tengu_bridge_poll_interval_config` | object | 300,000 ms (5 min) | Config object for bridge polling intervals: `poll_interval_ms_not_at_capacity`, `poll_interval_ms_at_capacity`, `heartbeat_interval_ms`. |
| `tengu_kairos_cron_config` | object | 60,000 ms (1 min) | Config object for cron jitter: `recurringFrac`, `recurringCapMs`, `oneShotMaxMs`, `oneShotFloorMs`, `oneShotMinuteMod`. |
| `tengu_bridge_initial_history_cap` | `200` | 300,000 ms (5 min) | Maximum number of initial history messages to load when a remote bridge session connects. |

---

### 2.3 A_() GrowthBook Experiments (7 experiments)

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

### 2.4 UL() Dynamic Configs (4 configs)

The `UL(configName, defaultValue)` function fetches a full config object from Statsig (not a boolean gate — returns structured data).

| Config Name | Default | Purpose |
|-------------|---------|----------|
| `tengu_bridge_min_version` | `{ minVersion: "0.0.0" }` | Minimum Claude Code version required for the Remote Control bridge. If current version is older than `minVersion`, shows upgrade prompt. |
| `tengu_1p_event_batch_config` | `{}` | Config for first-party telemetry event batching: `scheduledDelayMillis`, `maxExportBatchSize`, `maxQueueSize`. Controls OTEL log export behavior. |
| `tengu_desktop_upsell` | object | Config for desktop app upsell dialog: contains `enable_startup_dialog` boolean and display params. Shown on macOS/Windows x64 up to 3 times. |
| `tengu_sm_config` | `{}` | Session memory configuration object. Consumed alongside the `tengu_session_memory` gate. |

---

## 3. Summary Statistics

### Environment Variable Gates

| Category | Count |
|----------|-------|
| `CLAUDE_CODE_*` variables | 100 vars |
| `CLAUDE_*` variables (non-CODE) | 15 vars |
| `ANTHROPIC_*` variables | 16 vars |
| `DISABLE_*` / `ENABLE_*` gates | 29 vars |
| `USE_*` / `LOCAL_*` gates | 5 vars |
| `OTEL_*` telemetry variables | 19 vars |
| Debug / VCR / misc gates | 4 vars |
| Cloud platform detection vars | ~26 vars |
| AWS credential / config vars | 8 vars |
| Azure SDK credential vars | 15 vars |
| System / Node.js vars | ~22 vars |
| **Total unique env vars** | **~300+** |

### Statsig / GrowthBook Feature Flags

| Category | Unique Flags | Call-Sites |
|----------|-------------|------------|
| `p8()` Statsig boolean gates | 57 | 83 |
| `jU()` TTL-cached gates | 5 | 5 |
| `A_()` GrowthBook experiments | 7 | 11 |
| `UL()` dynamic config objects | 4 | 4 |
| **Total unique flag/config names** | **73** | **103** |

### Combined Total

| Metric | Count |
|--------|-------|
| Unique environment variable gates | ~300+ |
| Unique Statsig/GrowthBook flags | 73 |
| **Total unique feature control surfaces** | **~370+** |

### VS Code Experiment Gates (pushed at connection time)

These four gates are specifically pushed to the `claude-vscode` MCP server as a notification when it connects:

| Gate | Check Function |
|------|----------------|
| `tengu_vscode_review_upsell` | `A_()` GrowthBook |
| `tengu_vscode_onboarding` | `A_()` GrowthBook |
| `tengu_quiet_fern` | `p8()` Statsig |
| `tengu_slate_ridge` | `p8()` Statsig |
