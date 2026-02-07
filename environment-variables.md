# Claude Code CLI - Environment Variables Reference

Complete reference for all environment variables used by Claude Code CLI v2.1.34.

---

## Claude Code Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `CLAUDE_CODE_ACCESSIBILITY` | (none) | Enable accessibility mode (disables raw mode features and affects color rendering) |
| `CLAUDE_CODE_ACTION` | (none) | Indicates GitHub Action mode |
| `CLAUDE_CODE_ADDITIONAL_PROTECTION` | (none) | Enable additional API protection header |
| `CLAUDE_CODE_API_BASE_URL` | "https://api.anthropic.com" | Claude Code API base URL override |
| `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR` | (none) | File descriptor for API key |
| `CLAUDE_CODE_ATTRIBUTION_HEADER` | (none) | Enable attribution billing header |
| `CLAUDE_CODE_AUTO_CONNECT_IDE` | (none) | Auto-connect to IDE on startup |
| `CLAUDE_CODE_BASE_REF` | (none) | Git base reference for diffs |
| `CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR` | (none) | Show "SandboxedBash" indicator |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | (none) | Override blocking token limit |
| `CLAUDE_CODE_BUBBLEWRAP` | (none) | Using bubblewrap sandboxing |
| `CLAUDE_CODE_CLIENT_CERT` | (none) | Path to mTLS client certificate |
| `CLAUDE_CODE_CLIENT_KEY` | (none) | Path to mTLS client key |
| `CLAUDE_CODE_CLIENT_KEY_PASSPHRASE` | (none) | Passphrase for mTLS client key |
| `CLAUDE_CODE_CONTAINER_ID` | (none) | Container identifier for remote sessions |
| `CLAUDE_CODE_CUSTOM_OAUTH_URL` | (none) | Custom OAuth URL override |
| `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS` | "15000" | DataDog flush interval in milliseconds |
| `CLAUDE_CODE_DEBUG_LOGS_DIR` | (auto) | Directory for debug logs |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | (none) | Disable background tasks execution |
| `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK` | (none) | Disable command injection security checks |
| `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` | (none) | Disable experimental beta features |
| `CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY` | (none) | Disable feedback survey prompts |
| `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` | (none) | Disable file checkpointing feature |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | (none) | Disable all non-essential network traffic |
| `CLAUDE_CODE_DISABLE_OFFICIAL_MARKETPLACE_AUTOINSTALL` | (none) | Disable auto-install from official marketplace |
| `CLAUDE_CODE_DONT_INHERIT_ENV` | (none) | Prevents inheriting environment variables |
| `CLAUDE_CODE_EAGER_FLUSH` | (none) | Enable eager flushing |
| `CLAUDE_CODE_EFFORT_LEVEL` | (none) | Override effort level setting |
| `CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES` | (none) | Emit tool use summaries |
| `CLAUDE_CODE_ENABLE_CFC` | (none) | Enable Claude for Chrome |
| `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` | (none) | When "false", disables prompt suggestions |
| `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING` | (none) | Enable SDK file checkpointing |
| `CLAUDE_CODE_ENTRYPOINT` | (none) | Specifies the entrypoint type (mcp, sdk-ts, sdk-py, sdk-cli, claude-vscode, local-agent, remote, cli) |
| `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` | (none) | Exit after first render |
| `CLAUDE_CODE_EXIT_AFTER_STOP_DELAY` | (none) | Delay before exit after stop (ms) |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | (none) | Enable experimental agent teams feature |
| `CLAUDE_CODE_EXTRA_BODY` | (none) | Additional JSON body to merge into API requests |
| `CLAUDE_CODE_FORCE_GLOBAL_CACHE` | (none) | Force global cache usage for system prompts |
| `CLAUDE_CODE_GIT_BASH_PATH` | (none) | Custom Git Bash path (Windows) |
| `CLAUDE_CODE_GLOB_TIMEOUT_SECONDS` | 20 (linux) / 60 (wsl) | Timeout for glob operations in seconds |
| `CLAUDE_CODE_INCLUDE_PARTIAL_MESSAGES` | (none) | Include partial messages |
| `CLAUDE_CODE_IS_COWORK` | (none) | Indicates cowork mode |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | (none) | Maximum output tokens |
| `CLAUDE_CODE_MAX_RETRIES` | (none) | Maximum API retry attempts |
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` | "10" | Maximum number of concurrent tool uses |
| `CLAUDE_CODE_OAUTH_CLIENT_ID` | (none) | OAuth client ID override |
| `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | (none) | File descriptor for OAuth token |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | (none) | Require plan mode for agents |
| `CLAUDE_CODE_PROFILE_STARTUP` | "0" | Enable startup profiling ("1" to enable) |
| `CLAUDE_CODE_PROXY_RESOLVES_HOSTS` | (none) | Let proxy resolve host names |
| `CLAUDE_CODE_REMOTE` | "false" | Indicates remote execution mode |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | (random UUID) | Remote session identifier |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | (none) | Session access token for remote mode |
| `CLAUDE_CODE_SESSION_ID` | (dynamic) | Session identifier for Claude Code |
| `CLAUDE_CODE_SHELL` | (none) | Shell override for bash/zsh |
| `CLAUDE_CODE_SHELL_PREFIX` | (none) | Prefix for shell commands |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | (none) | Skip AWS Bedrock authentication |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | (none) | Skip Foundry authentication |
| `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | (none) | Skip saving prompt history |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | (none) | Skip Vertex AI authentication |
| `CLAUDE_CODE_SUBAGENT_MODEL` | (none) | Override model selection for subagents |
| `CLAUDE_CODE_SYNTAX_HIGHLIGHT` | (none) | Control syntax highlighting |
| `CLAUDE_CODE_TMPDIR` | "/tmp" | Custom temporary directory |
| `CLAUDE_CODE_USE_BEDROCK` | (none) | Use AWS Bedrock for API calls |
| `CLAUDE_CODE_USE_COWORK_PLUGINS` | (none) | Use cowork_settings.json |
| `CLAUDE_CODE_USE_FOUNDRY` | (none) | Use Microsoft Foundry for API calls |
| `CLAUDE_CODE_USE_VERTEX` | (none) | Use Google Vertex AI for API calls |
| `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` | (none) | WebSocket auth file descriptor for remote mode |

## Anthropic API

| Variable | Default | Description |
|----------|---------|-------------|
| `ANTHROPIC_API_KEY` | (none) | Anthropic API key for authentication |
| `ANTHROPIC_AUTH_TOKEN` | (none) | Anthropic authentication token |
| `ANTHROPIC_BASE_URL` | "https://api.anthropic.com" | Base URL for Anthropic API |
| `ANTHROPIC_BEDROCK_BASE_URL` | (none) | Custom AWS Bedrock endpoint URL |
| `ANTHROPIC_BETAS` | (none) | Comma-separated list of beta features to enable |
| `ANTHROPIC_CUSTOM_HEADERS` | (none) | Custom headers for Anthropic API |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | (none) | Override default Haiku model |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | (none) | Override default Opus model |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | (none) | Override default Sonnet model |
| `ANTHROPIC_FOUNDRY_API_KEY` | (none) | API key for Microsoft Foundry |
| `ANTHROPIC_FOUNDRY_BASE_URL` | (none) | Microsoft Foundry base URL |
| `ANTHROPIC_FOUNDRY_RESOURCE` | (none) | Microsoft Foundry resource identifier |
| `ANTHROPIC_LOG` (none) | Anthropic SDK log level |
| `ANTHROPIC_MODEL` | (none) | Override model selection |
| `ANTHROPIC_SMALL_FAST_MODEL` | (none) | Override small/fast model selection |
| `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` | (dynamic) | AWS region for small/fast model on Bedrock |
| `ANTHROPIC_VERTEX_PROJECT_ID` | (none) | Override Vertex AI project ID |

## LLM Provider Selection

| Variable | Default | Description |
|----------|---------|-------------|
| `BEDROCK_BASE_URL` | (none) | Custom Bedrock API base URL |
| `VERTEX_BASE_URL` | (none) | Custom Vertex AI base URL |

## Authentication & Security

| Variable | Default | Description |
|----------|---------|-------------|
| `CLAUDE_AGENT_SDK_VERSION` | (none) | Agent SDK version |
| `NODE_EXTRA_CA_CERTS` | (none) | Additional CA certificates for Node.js |

## Proxy & Networking

| Variable | Default | Description |
|----------|---------|-------------|
| `HTTP_PROXY` / `http_proxy` | (none) | HTTP proxy URL |
| `HTTPS_PROXY` / `https_proxy` | (none) | HTTPS proxy URL |
| `NO_PROXY` / `no_proxy` | (none) | Comma-separated list of hosts to bypass proxy |

## Terminal & Display

| Variable | Default | Description |
|----------|---------|-------------|
| `ALACRITTY_LOG` | (none) | Detect Alacritty terminal |
| `COLORTERM` | (none) | Color terminal support ("truecolor" enables rich colors) |
| `ConEmuANSI` | (none) | Detect ConEmu terminal |
| `ConEmuPID` | (none) | Detect ConEmu terminal |
| `ConEmuTask` | (none) | Detect ConEmu terminal |
| `CURSOR_TRACE_ID` | (none) | Detect Cursor IDE environment |
| `EDITOR` | (none) | Default editor command |
| `FORCE_COLOR` | (none) | Force color output |
| `GNOME_TERMINAL_SERVICE` | (none) | Detect GNOME Terminal |
| `ITERM_SESSION_ID` | (none) | iTerm2 session identifier |
| `KITTY_WINDOW_ID` | (none) | Detect Kitty terminal |
| `KONSOLE_VERSION` | (none) | Detect KDE Konsole |
| `LC_TERMINAL` | (none) | Locale terminal identifier |
| `SESSIONNAME` | (none) | Detect Cygwin (with TERM=cygwin) |
| `SSH_CLIENT` | (none) | Detect SSH connection |
| `SSH_CONNECTION` | (none) | Detect SSH connection |
| `SSH_TTY` | (none) | Detect SSH connection |
| `STY` | (none) | Detect screen session |
| `TERM` | (none) | Terminal type (ghostty, kitty, xterm, etc.) |
| `TERM_PROGRAM` | (none) | Terminal program name |
| `TERM_PROGRAM_VERSION` | (none) | Terminal program version |
| `TERMINATOR_UUID` | (none) | Detect Terminator terminal |
| `TERMINAL_EMULATOR` | (none) | Terminal emulator identifier |
| `TILIX_ID` | (none) | Detect Tilix terminal |
| `TMUX` | (none) | Detect tmux session |
| `TMUX_PANE` | (none) | TMUX pane identifier |
| `VISUAL` | (none) | Visual editor command (takes precedence over EDITOR) |
| `VSCODE_GIT_ASKPASS_MAIN` | (none) | Detect Cursor/Windsurf via git askpass path |
| `VTE_VERSION` | (none) | Detect VTE-based terminal |
| `WT_SESSION` | (none) | Detect Windows Terminal |
| `XTERM_VERSION` | (none) | Detect xterm |
| `ZED_TERM` | (none) | Zed editor terminal |
| `__CFBundleIdentifier` | (none) | macOS bundle identifier (IDE detection) |

## System & Paths

| Variable | Default | Description |
|----------|---------|-------------|
| `APPDATA` | (none) | Windows application data directory |
| `BROWSER` | (none) | Browser command for opening URLs |
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | (none) | Maintain project working directory in bash |
| `CLAUDE_BASH_NO_LOGIN` | (none) | Skip login flag when spawning bash |
| `CLAUDE_CONFIG_DIR` | `~/.claude` | Claude configuration directory |
| `HOME` | (none) | User home directory |
| `MSYSTEM` | (none) | MSYS2 system type |
| `OSTYPE` | (none) | Operating system type |
| `PATH` | (OS default) | System PATH environment variable |
| `PATHEXT` | (Windows default) | Windows executable extensions |
| `ProgramData` | (none) | Windows program data directory |
| `ProgramFiles` | (none) | Windows program files directory |
| `PWD` | (none) | Present working directory |
| `SHELL` | (none) | User's default shell |
| `SystemRoot` / `SYSTEMROOT` | (Windows) | Windows system root directory |
| `TEMP` | "C:\\Temp" (Windows) | Windows temp directory |
| `TMPDIR` | (OS default) | Temporary directory |
| `USER` | (none) | Current username |
| `USERPROFILE` | (none) | Windows user profile directory |
| `WSL_DISTRO_NAME` | (none) | WSL distribution name |
| `XDG_CONFIG_HOME` | (none) | XDG config directory |
| `XDG_RUNTIME_DIR` | (none) | XDG runtime directory |

## CI/CD & Git

| Variable | Default | Description |
|----------|---------|-------------|
| `CI` | (none) | Indicates CI environment |
| `CLAUDECODE` | (none) | Set to "1" to indicate Claude Code environment |
| `GITHUB_ACTION_INPUTS` | (none) | GitHub Action inputs JSON |
| `GITHUB_ACTIONS` | (none) | Running in GitHub Actions CI |
| `GITHUB_ACTOR` | (none) | GitHub Actions actor username |
| `GITHUB_ACTOR_ID` | (none) | GitHub Actions actor ID |
| `GITHUB_REPOSITORY` | (none) | GitHub repository (owner/repo) |
| `GITHUB_REPOSITORY_ID` | (none) | GitHub repository ID |
| `GITHUB_REPOSITORY_OWNER` | (none) | GitHub repository owner |
| `GITHUB_REPOSITORY_OWNER_ID` | (none) | GitHub repository owner ID |
| `IS_SANDBOX` | (none) | Running in sandboxed environment |
| `VisualStudioVersion` | (none) | Detect Visual Studio environment |

## Cloud Provider - AWS

| Variable | Default | Description |
|----------|---------|-------------|
| `AWS_ACCESS_KEY_ID` | (none) | AWS access key ID |
| `AWS_BEARER_TOKEN_BEDROCK` | (none) | Bearer token for Bedrock API |
| `AWS_DEFAULT_REGION` | "us-east-1" | AWS default region |
| `AWS_LAMBDA_BENCHMARK_MODE` | (none) | AWS Lambda benchmark mode |
| `AWS_LOGIN_CACHE_DIRECTORY` | `~/.aws/login/cache` | AWS login cache directory |
| `AWS_PROFILE` | (none) | AWS profile name |
| `AWS_REGION` | "us-east-1" | AWS region |
| `AWS_SECRET_ACCESS_KEY` | (none) | AWS secret access key |
| `AWS_SESSION_TOKEN` | (none) | AWS session token |

## Cloud Provider - Azure

| Variable | Default | Description |
|----------|---------|-------------|
| `AZURE_AUTHORITY_HOST` | (none) | Azure authority host |
| `AZURE_CLIENT_ID` | (none) | Azure client ID |
| `AZURE_FEDERATED_TOKEN_FILE` | (none) | Azure federated token file path |
| `AZURE_IDENTITY_DISABLE_MULTITENANTAUTH` | (none) | Disable multi-tenant authentication |
| `AZURE_POD_IDENTITY_AUTHORITY_HOST` | (none) | Azure Pod Identity authority host |
| `AZURE_REGIONAL_AUTHORITY_NAME` | (none) | Azure regional authority name |
| `AZURE_TENANT_ID` | (none) | Azure tenant ID |
| `AZURE_TOKEN_CREDENTIALS` | (none) | Azure token credentials (prod/dev) |

## Cloud Provider - GCP

| Variable | Default | Description |
|----------|---------|-------------|
| `CLOUD_ML_REGION` | "us-east5" | Cloud ML region |
| `DETECT_GCP_RETRIES` | "0" | Number of retries for GCP detection |
| `FUNCTION_NAME` | (none) | Google Cloud Function name |
| `FUNCTION_TARGET` | (none) | Google Cloud Function target |
| `GAE_MODULE_NAME` | (none) | Google App Engine module name |
| `GAE_SERVICE` | (none) | Google App Engine service indicator |
| `GCE_METADATA_HOST` | (dynamic) | Google Compute Engine metadata host |
| `GCE_METADATA_IP` | (dynamic) | Google Compute Engine metadata IP |
| `GCLOUD_PROJECT` | (none) | GCP project ID |
| `GOOGLE_APPLICATION_CREDENTIALS` | (none) | Path to GCP credentials JSON |
| `GOOGLE_CLOUD_PROJECT` | (none) | GCP project ID (alternative) |
| `GOOGLE_CLOUD_QUOTA_PROJECT` | (none) | GCP quota project ID |
| `GOOGLE_EXTERNAL_ACCOUNT_ALLOW_EXECUTABLES` | (none) | Allow executable credential sources (must be "1") |
| `K_CONFIGURATION` | (none) | Cloud Run configuration indicator |
| `METADATA_SERVER_DETECTION` | (none) | Override metadata server detection behavior |
| `VERTEX_REGION_CLAUDE_3_5_HAIKU` | (auto) | Vertex AI region for Claude 3.5 Haiku |
| `VERTEX_REGION_CLAUDE_3_5_SONNET` | (auto) | Vertex AI region for Claude 3.5 Sonnet |
| `VERTEX_REGION_CLAUDE_3_7_SONNET` | (auto) | Vertex AI region for Claude 3.7 Sonnet |
| `VERTEX_REGION_CLAUDE_4_0_OPUS` | (auto) | Vertex AI region for Claude Opus 4.0 |
| `VERTEX_REGION_CLAUDE_4_0_SONNET` | (auto) | Vertex AI region for Claude Sonnet 4.0 |
| `VERTEX_REGION_CLAUDE_4_1_OPUS` | (auto) | Vertex AI region for Claude Opus 4.1 |
| `VERTEX_REGION_CLAUDE_4_5_SONNET` | (auto) | Vertex AI region for Claude Sonnet 4.5 |
| `VERTEX_REGION_CLAUDE_HAIKU_4_5` | (auto) | Vertex AI region for Claude Haiku 4.5 |
| `gcloud_project` | (none) | Lowercase GCP project ID |
| `google_application_credentials` | (none) | Lowercase variant of GCP credentials |
| `google_cloud_project` | (none) | Lowercase GCP project ID (alternative) |

## Development & Debug

| Variable | Default | Description |
|----------|---------|-------------|
| `API_TIMEOUT_MS` | "600000" | API request timeout in milliseconds |
| `CLAUBBIT` | (none) | Bypass permission dialogs |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | (none) | Override auto-compact percentage threshold |
| `CLAUDE_CHROME_PERMISSION_MODE` | (none) | Initial permission mode for Chrome extension |
| `CLAUDE_DEBUG` | "false" | Enable debug mode |
| `CLAUDE_FORCE_DISPLAY_SURVEY` | (none) | Force display feedback survey (testing) |
| `COREPACK_ENABLE_AUTO_PIN` | "0" | Disable Corepack auto-pinning |
| `DEBUG` | (none) | Debug mode flag (general) |
| `DEBUG_AUTH` | (none) | Enable authentication debugging output |
| `DEBUG_SDK` | (none) | SDK debug mode |
| `DEMO_VERSION` | (none) | Demo version string override |
| `DISABLE_AUTO_COMPACT` | (none) | Disable automatic compaction |
| `DISABLE_AUTO_MIGRATE_TO_NATIVE` | (none) | Disable auto-migration to native install |
| `DISABLE_AUTOUPDATER` | "1" | Disable auto-updater |
| `DISABLE_COMPACT` | (none) | Completely disable compact functionality |
| `DISABLE_ERROR_REPORTING` | (none) | Disable error reporting to Anthropic |
| `DISABLE_EXTRA_USAGE_COMMAND` | (none) | Disable extra usage command |
| `DISABLE_INSTALLATION_CHECKS` | (none) | Disable installation validation checks |
| `DISABLE_INSTALL_GITHUB_APP_COMMAND` | (none) | Disable GitHub app installation command |
| `DISABLE_INTERLEAVED_THINKING` | (none) | Disable interleaved thinking feature |
| `DISABLE_MICROCOMPACT` | (none) | Disable microcompact feature |
| `DISABLE_PROMPT_CACHING` | (none) | Disable all prompt caching |
| `DISABLE_PROMPT_CACHING_HAIKU` | (none) | Disable prompt caching for Haiku model |
| `DISABLE_PROMPT_CACHING_OPUS` | (none) | Disable prompt caching for Opus model |
| `DISABLE_PROMPT_CACHING_SONNET` | (none) | Disable prompt caching for Sonnet model |
| `DISABLE_TELEMETRY` | (none) | Disable telemetry |
| `DISABLE_UPGRADE_COMMAND` | (none) | Disable /upgrade command visibility |
| `ENABLE_CLAUDE_CODE_SM_COMPACT` | (none) | Force enable session memory compact |
| `DISABLE_CLAUDE_CODE_SM_COMPACT` | (none) | Force disable session memory compact |
| `ENABLE_EXPERIMENTAL_MCP_CLI` | (none) | Enable experimental MCP CLI features |
| `ENABLE_MCP_CLI` | (none) | Enable MCP CLI mode |
| `ENABLE_MCP_CLI_ENDPOINT` | (none) | Enable MCP CLI endpoint |
| `ENABLE_TOOL_SEARCH` | (none) | Enable tool search functionality (e.g., "auto:50" for 50% rollout) |
| `FALLBACK_FOR_ALL_PRIMARY_MODELS` | (none) | Enable fallback for all primary models |
| `IS_DEMO` | (none) | Running in demo mode (hides sensitive data) |
| `LOCAL_BRIDGE` | (none) | Use local bridge for development |
| `MAX_STRUCTURED_OUTPUT_RETRIES` | "5" | Maximum retries for structured output |
| `NODE_DEBUG` | (none) | Node.js debug output |
| `NODE_OPTIONS` | "" | Node.js runtime options |
| `NODE_V8_COVERAGE` | (none) | V8 coverage collection mode |
| `NoDefaultCurrentDirectoryInExePath` | "1" | Windows-specific: prevents default current directory in exe path |
| `OTEL_LOGS_EXPORT_INTERVAL` | (none) | OpenTelemetry logs export interval |
| `PERMISSION_EXPLAINER_ENABLED` | (none) | Enable permission explainer |
| `RIPGREP_EMBEDDED` | "false" | Use embedded ripgrep via Node execPath |
| `RIPGREP_NODE_PATH` | (none) | Path to ripgrep.node native module |
| `SRT_DEBUG` | (none) | Sandbox runtime debug mode |
| `TEST_ENABLE_SESSION_PERSISTENCE` | (none) | Enable session persistence in test mode |
| `USE_API_CONTEXT_MANAGEMENT` | (none) | Use API-side context management |
| `USE_BUILTIN_RIPGREP` | (none) | Use built-in ripgrep binary |
| `USE_LOCAL_OAUTH` | (none) | Use local OAuth for development |
| `USE_MCP_CLI_DIR` | (auto) | Override MCP CLI directory |

## MCP (Model Context Protocol)

| Variable | Default | Description |
|----------|---------|-------------|
| `MCP_CONNECTION_NONBLOCKING` | (none) | Make MCP connections non-blocking |
| `MCP_TOOL_TIMEOUT` | "600000" | MCP tool timeout in milliseconds |

## Build Tools & Native Libraries

| Variable | Default | Description |
|----------|---------|-------------|
| `CHOKIDAR_INTERVAL` | (none) | Polling interval in milliseconds for file watching |
| `CHOKIDAR_USEPOLLING` | (none) | Enable polling mode for file watching |
| `GRACEFUL_FS_PLATFORM` | (dynamic) | Override platform for graceful-fs |
| `PKG_CONFIG_PATH` | (default paths) | pkg-config search path |
| `SHARP_FORCE_GLOBAL_LIBVIPS` | (none) | Force use of global libvips |
| `SHARP_IGNORE_GLOBAL_LIBVIPS` | (none) | Ignore global libvips installation |

## Vendor / Third-Party

| Variable | Default | Description |
|----------|---------|-------------|
| `GRPC_DEFAULT_SSL_ROOTS_FILE_PATH` | (none) | Path to default SSL root certificates for gRPC |
| `GRPC_NODE_TRACE` | (none) | gRPC Node.js trace configuration |
| `GRPC_NODE_USE_ALTERNATIVE_RESOLVER` | "false" | Use alternative DNS resolver for gRPC |
| `GRPC_NODE_VERBOSITY` | (none) | gRPC Node.js verbosity level |
| `GRPC_SSL_CIPHER_SUITES` | (none) | SSL cipher suites for gRPC |
| `GRPC_TRACE` | (none) | gRPC trace configuration |
| `GRPC_VERBOSITY` | (none) | gRPC verbosity level |
| `JEST_WORKER_ID` | (none) | Jest worker ID (affects WebAssembly compilation) |
| `UNDICI_NO_FG` | (none) | Disable Undici FinalizationRegistry |

---

## Notes

### Variable Naming Patterns

- **Boolean flags**: Variables without explicit "true"/"false" values are typically checked for truthy/falsy presence
- **Disable/Enable pairs**: Some features have both `DISABLE_*` and `ENABLE_*` variants
- **Case variants**: Cloud provider variables often have both uppercase and lowercase variants (e.g., `GCLOUD_PROJECT` and `gcloud_project`)

### Critical Variables for Production

1. **Authentication**: `ANTHROPIC_API_KEY`, `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR`
2. **Security**: `CLAUDE_CODE_CLIENT_CERT`, `CLAUDE_CODE_CLIENT_KEY`, `NODE_EXTRA_CA_CERTS`
3. **Networking**: `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`
4. **Provider Selection**: `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`, `CLAUDE_CODE_USE_FOUNDRY`
5. **Configuration**: `CLAUDE_CONFIG_DIR`, `CLAUDE_CODE_TMPDIR`

### Provider-Specific Requirements

**AWS Bedrock**:
- `CLAUDE_CODE_USE_BEDROCK` (enable)
- `AWS_REGION` or `AWS_DEFAULT_REGION`
- `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` (or use IAM roles)
- Optional: `CLAUDE_CODE_SKIP_BEDROCK_AUTH`, `AWS_BEARER_TOKEN_BEDROCK`

**Google Vertex AI**:
- `CLAUDE_CODE_USE_VERTEX` (enable)
- `GOOGLE_APPLICATION_CREDENTIALS` or `google_application_credentials`
- `GCLOUD_PROJECT` or `GOOGLE_CLOUD_PROJECT`
- Optional: `VERTEX_REGION_*` variables for specific models
- Optional: `CLAUDE_CODE_SKIP_VERTEX_AUTH`

**Microsoft Foundry**:
- `CLAUDE_CODE_USE_FOUNDRY` (enable)
- `ANTHROPIC_FOUNDRY_API_KEY`
- Optional: `ANTHROPIC_FOUNDRY_BASE_URL`, `ANTHROPIC_FOUNDRY_RESOURCE`
- Optional: `CLAUDE_CODE_SKIP_FOUNDRY_AUTH`

### Feature Flags

Many experimental features can be controlled via environment variables:
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` - Agent teams
- `ENABLE_EXPERIMENTAL_MCP_CLI` - Experimental MCP CLI features
- `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` - Disable all beta features
- `ENABLE_TOOL_SEARCH` - Tool search with rollout percentage (e.g., "auto:50")

### Telemetry & Privacy

- `DISABLE_TELEMETRY` - Disable telemetry collection
- `DISABLE_ERROR_REPORTING` - Disable error reporting to Anthropic
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` - Disable all non-essential network traffic
- `IS_DEMO` - Demo mode (hides sensitive data in logs)

### Terminal Detection

The CLI performs extensive terminal detection using 30+ environment variables to provide optimal display and features for each terminal emulator. This includes detection for:
- IDEs (VS Code, Cursor, Windsurf, Zed)
- Terminal emulators (iTerm2, Kitty, Alacritty, WezTerm, Ghostty, Windows Terminal, etc.)
- Multiplexers (tmux, screen)
- SSH sessions
- CI environments
