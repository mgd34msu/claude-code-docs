# Claude Code v2.1.81 — Environment Variables Reference

Generated from source: `cc-2.1.81` (build `2026-03-20T21:25:42Z`)

---

## 1. Overview

Claude Code reads environment variables at startup and at runtime to configure authentication, provider selection, feature flags, plugin behavior, and subprocess environment handling. Variables are consumed directly via `process.env.<NAME>` throughout the codebase.

### Boolean Parsing

Two parsing functions normalize string boolean values. **Source:** `src/cli/args-1.ts`

**`a6(value)` — truthy check**

```
a6("1")     → true
a6("true")  → true   (case-insensitive)
a6("yes")   → true   (case-insensitive)
a6("on")    → true   (case-insensitive)
a6("")      → false
a6(undefined) → false
a6("false") → false
a6("0")     → false
```

**`dY(value)` — explicit-false check** (used to test if something is *disabled*)

```
dY("0")     → true   (the flag IS disabled)
dY("false") → true   (case-insensitive)
dY("no")    → true   (case-insensitive)
dY("off")   → true   (case-insensitive)
dY(undefined) → false  (NOT disabled — absence is not the same as disabled)
dY("")      → false
dY("true")  → false
```

Note: `dY` is used for opt-out flags (e.g., `dY(process.env.USE_BUILTIN_RIPGREP)` means "explicitly disabled"). Absence of the variable is treated as *not disabled*, which differs from `a6` where absence means *not enabled*.

### Variable Precedence

1. **Environment variables** — highest precedence for most settings
2. **CLI flags** — take precedence over env vars for `--model`, `--debug`, etc.
3. **Settings files** (`~/.claude/settings.json`, `.claude/settings.json`) — lowest precedence
4. **Compiled defaults** — fallbacks when nothing else is set

---

## 2. Authentication & API Configuration

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `ANTHROPIC_API_KEY` | string | — | `src/core/config-1.ts:12511`, `src/core/validateforceloginorg-2.ts:172` |
| `ANTHROPIC_AUTH_TOKEN` | string | — | `src/conversation/session-1.ts:1820`, `src/core/config-1.ts:5219` |
| `ANTHROPIC_BASE_URL` | string | `https://api.anthropic.com` | `src/llm/api-1.ts:37` |
| `ANTHROPIC_MODEL` | string | — | `src/core/resolveskillmodeloverride-1.ts:64` |
| `ANTHROPIC_BETAS` | string | — | `src/cli/args-1.ts:2348` |
| `ANTHROPIC_CUSTOM_HEADERS` | string | — | `src/conversation/session-1.ts:1825`, `src/core/config-1.ts:5221` |
| `ANTHROPIC_SMALL_FAST_MODEL` | string | — | `src/core/resolveskillmodeloverride-1.ts:48` |
| `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` | string | — | `src/conversation/session-1.ts:1736` |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | string | — | `src/llm/tokenization-1.ts:370` |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | string | — | `src/llm/tokenization-1.ts:394` |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | string | — | `src/llm/tokenization-1.ts:443` |
| `ANTHROPIC_CUSTOM_MODEL_OPTION` | string | — | `src/llm/tokenization-1.ts:575` |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_NAME` | string | — | `src/llm/tokenization-1.ts:579` |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION` | string | — | `src/llm/tokenization-1.ts:581` |
| `ANTHROPIC_UNIX_SOCKET` | string | — | `src/networking/agent-1.ts:4387` |
| `CLAUDE_CONFIG_DIR` | string | `~/.claude` | `src/core/r8-32.ts:17` |
| `CLAUDE_CODE_OAUTH_TOKEN` | string | — | `src/core/validateforceloginorg-2.ts:146`, `src/core/config-1.ts:5218` |
| `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | integer | — | `src/core/config-2.ts:251` |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | string | — | `src/core/auth-1.ts:12055` |
| `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` | integer | — | `src/core/auth-1.ts:12010` |
| `CLAUDE_CODE_ORGANIZATION_UUID` | string | — | `src/core/auth-1.ts:12064` |
| `CLAUDE_CODE_CLIENT_CERT` | string (path) | — | `src/core/filesystem-2.ts:175` |
| `CLAUDE_CODE_CUSTOM_OAUTH_URL` | string (URL) | — | `src/mcp/getoauthconfig-2.ts:61` |
| `CLAUDE_CODE_OAUTH_CLIENT_ID` | string | — | `src/mcp/getoauthconfig-2.ts:80` |
| `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` | string | — | `src/vendor/dom.ts:59127` |
| `CLAUDE_CODE_OAUTH_SCOPES` | string | — | `src/vendor/dom.ts:59129` |
| `CLAUDE_CODE_ACCOUNT_TAGGED_ID` | string | — | `src/conversation/session-2.ts:68` |

### Details

**`ANTHROPIC_API_KEY`** — The Anthropic API key used for first-party authentication. When set and `CLAUDE_CODE_SIMPLE` mode is not active, it is read and validated. In simple/bare mode the key is accepted but keychain and OAuth are never consulted. The key must start with `sk-ant-`. Superseded at runtime by OAuth token if both are present and conditions allow.

**`ANTHROPIC_AUTH_TOKEN`** — Alternative authentication token. Checked before the keychain lookup at `src/conversation/session-1.ts:1820`. Included in the subprocess scrub list (`nD9`).

**`ANTHROPIC_BASE_URL`** — Overrides the default Anthropic API endpoint (`https://api.anthropic.com`). When set to any URL other than `api.anthropic.com`, Claude Code skips first-party feature checks and disables automatic tool-search unless `ENABLE_TOOL_SEARCH` is explicitly set. **Source:** `src/llm/api-1.ts:37`.

**`ANTHROPIC_MODEL`** — Forces a specific model string. Overrides the resolved model from settings. **Source:** `src/core/resolveskillmodeloverride-1.ts:64`.

**`ANTHROPIC_BETAS`** — Comma-separated list of beta feature flags appended to API requests. **Source:** `src/cli/args-1.ts:2348`.

**`ANTHROPIC_CUSTOM_HEADERS`** — JSON string of additional HTTP headers merged into every API request. Included in subprocess env scrub. **Source:** `src/conversation/session-1.ts:1825`.

**`ANTHROPIC_SMALL_FAST_MODEL`** — Override for the small/fast model used in lightweight tasks. Falls back to compiled default if unset. **Source:** `src/core/resolveskillmodeloverride-1.ts:48`.

**`ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION`** — AWS region override specifically for the small/fast model when using Bedrock. Takes precedence over `AWS_REGION`/`AWS_DEFAULT_REGION` for that model only. **Source:** `src/conversation/session-1.ts:1736`.

**`ANTHROPIC_UNIX_SOCKET`** — Path to a Unix domain socket. When set, HTTP connections to the Anthropic API route through this socket rather than TCP. **Source:** `src/networking/agent-1.ts:4387`.

**`CLAUDE_CONFIG_DIR`** — Overrides the default Claude configuration directory (`~/.claude`). Affects all config file reads and writes. **Source:** `src/core/r8-32.ts:17`.

**`CLAUDE_CODE_OAUTH_TOKEN`** — Pre-provisioned OAuth access token. Bypasses the interactive login flow. Takes precedence over keychain tokens. Included in subprocess scrub. **Source:** `src/core/validateforceloginorg-2.ts:146`.

**`CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR`** — File descriptor number from which to read an OAuth token at startup. Used in containerized environments where secrets are passed via file descriptors rather than environment variables. **Source:** `src/core/config-2.ts:251`.

**`CLAUDE_CODE_SESSION_ACCESS_TOKEN`** — Session-scoped access token (begins with `sk-ant-sid`). Used in remote/agent contexts. Also settable programmatically. **Source:** `src/core/auth-1.ts:12055`.

**`CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR`** — Integer file descriptor number from which to read a WebSocket authentication token. Platform-aware: uses `/dev/fd/N` on macOS/FreeBSD, `/proc/self/fd/N` on Linux. **Source:** `src/core/auth-1.ts:12010`.

**`CLAUDE_CODE_ORGANIZATION_UUID`** — When set alongside a session token, adds an `X-Organization-Uuid` header to API requests. **Source:** `src/core/auth-1.ts:12064`.

**`CLAUDE_CODE_CLIENT_CERT`** — Path to a PEM-encoded client certificate file for mTLS (mutual TLS). Loaded via `fs.readFileSync`. **Source:** `src/core/filesystem-2.ts:175`.

**`CLAUDE_CODE_CUSTOM_OAUTH_URL`** — Custom OAuth endpoint URL. Must be an approved endpoint; otherwise throws. **Source:** `src/mcp/getoauthconfig-2.ts:61`.

**`CLAUDE_CODE_OAUTH_SCOPES`** — Required companion to `CLAUDE_CODE_OAUTH_REFRESH_TOKEN`. Specifies OAuth scopes as a string. **Source:** `src/vendor/dom.ts:59129`.

---

## 3. LLM Provider Selection

### Provider Selection Order

The function `QA()` in `src/llm/api-1.ts:24` determines the active provider:

```
CLAUDE_CODE_USE_BEDROCK=1  →  "bedrock"
  else CLAUDE_CODE_USE_VERTEX=1  →  "vertex"
    else CLAUDE_CODE_USE_FOUNDRY=1  →  "foundry"
      else  →  "firstParty" (default Anthropic API)
```

Only one provider flag should be set at a time.

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `CLAUDE_CODE_USE_BEDROCK` | boolean (`a6`) | `false` | `src/llm/api-1.ts:25` |
| `CLAUDE_CODE_USE_VERTEX` | boolean (`a6`) | `false` | `src/llm/api-1.ts:27` |
| `CLAUDE_CODE_USE_FOUNDRY` | boolean (`a6`) | `false` | `src/llm/api-1.ts:29` |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | boolean (`a6`) | `false` | `src/conversation/session-1.ts:1742` |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | boolean (`a6`) | `false` | `src/conversation/session-1.ts:1781` |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | boolean (`a6`) | `false` | `src/conversation/session-1.ts:1766` |

### AWS / Bedrock Variables

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `AWS_REGION` | string | `us-east-1` | `src/cli/args-1.ts:69` |
| `AWS_DEFAULT_REGION` | string | `us-east-1` | `src/cli/args-1.ts:69` |
| `AWS_BEARER_TOKEN_BEDROCK` | string | — | `src/conversation/session-1.ts:1745`, `src/core/config-1.ts:5228` |
| `AWS_SECRET_ACCESS_KEY` | string | — | `src/core/config-1.ts:5226` (scrub list) |
| `AWS_SESSION_TOKEN` | string | — | `src/core/config-1.ts:5227` (scrub list) |
| `ANTHROPIC_BEDROCK_BASE_URL` | string | — | `src/conversation/anthropic-1.ts:26` |

Region resolution (`V76()` in `src/cli/args-1.ts`): `AWS_REGION` → `AWS_DEFAULT_REGION` → `"us-east-1"`

**`CLAUDE_CODE_SKIP_BEDROCK_AUTH`** — When set, passes `skipAuth: true` to the Bedrock client constructor, skipping AWS credential validation. Useful in environments with pre-signed tokens or custom auth. **Source:** `src/conversation/session-1.ts:1742`.

**`AWS_BEARER_TOKEN_BEDROCK`** — Static bearer token for Bedrock authentication. When set, the `Authorization: Bearer <token>` header is added directly and AWS SDK credential chain is bypassed. Included in subprocess scrub. **Source:** `src/conversation/session-1.ts:1745`.

**`ANTHROPIC_BEDROCK_BASE_URL`** — Custom base URL for AWS Bedrock API calls. **Source:** `src/conversation/anthropic-1.ts:26`.

### Google Cloud / Vertex Variables

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `CLOUD_ML_REGION` | string | `us-east5` | `src/cli/args-1.ts:73` |
| `ANTHROPIC_VERTEX_PROJECT_ID` | string | — | `src/conversation/session-1.ts:1800` |
| `GCLOUD_PROJECT` | string | — | `src/conversation/session-1.ts:1787` |
| `GOOGLE_CLOUD_PROJECT` | string | — | `src/conversation/session-1.ts:1788` |
| `GOOGLE_APPLICATION_CREDENTIALS` | string (path) | — | `src/conversation/session-1.ts:1792`, `src/core/config-1.ts:5229` |
| `GOOGLE_CLOUD_QUOTA_PROJECT` | string | — | `src/core/config-1.ts:5033` |

Region resolution (`l68()` in `src/cli/args-1.ts`): `CLOUD_ML_REGION` → `"us-east5"`

Model-specific region: `i68(model)` checks if the model matches a prefix in `aaq` lookup table; falls back to `l68()`. **Source:** `src/cli/args-1.ts`.

**`GOOGLE_APPLICATION_CREDENTIALS`** — Path to a Google service account JSON file. Included in subprocess scrub list (`nD9`) as it contains credentials. **Source:** `src/core/config-1.ts:4728`.

**`CLAUDE_CODE_SKIP_VERTEX_AUTH`** — Skip Vertex AI credential refresh (`IB6()`). **Source:** `src/conversation/session-1.ts:1781`.

### Azure Foundry Variables

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `ANTHROPIC_FOUNDRY_BASE_URL` | string | — | `src/core/anthropic-1.ts:22` |
| `ANTHROPIC_FOUNDRY_API_KEY` | string | — | `src/core/anthropic-1.ts:23`, `src/core/config-1.ts:5220` |
| `ANTHROPIC_FOUNDRY_RESOURCE` | string | — | `src/core/anthropic-1.ts:24` |
| `ANTHROPIC_FOUNDRY_AUTH_TOKEN` | string | — | `src/tools/tool-1.ts:24567` |
| `AZURE_AUTHORITY_HOST` | string | — | `src/core/auth-1.ts:9093` |
| `AZURE_POD_IDENTITY_AUTHORITY_HOST` | string | — | `src/core/auth-1.ts:9224` |
| `AZURE_CLIENT_SECRET` | string | — | `src/core/config-1.ts:5230` (scrub list) |
| `AZURE_CLIENT_CERTIFICATE_PATH` | string (path) | — | `src/core/config-1.ts:5231` (scrub list) |

**`ANTHROPIC_FOUNDRY_API_KEY`** — API key for Azure AI Foundry. Required when `CLAUDE_CODE_USE_FOUNDRY=1` and `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` is not set. Included in subprocess scrub. **Source:** `src/conversation/session-1.ts:1765`.

**`ANTHROPIC_FOUNDRY_RESOURCE`** — Azure resource identifier. Required when `ANTHROPIC_FOUNDRY_BASE_URL` is not provided. **Source:** `src/core/anthropic-1.ts:24`.

---

## 4. Core Behavior Configuration

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `CLAUDE_CODE_SIMPLE` | boolean (`a6`) | `false` | `src/cli/args-1.ts:52` |
| `CLAUDE_CODE_ENTRYPOINT` | string | `"cli"` | `src/vendor/commander.ts:4710` |
| `CLAUDE_CODE_DEBUG_LOG_LEVEL` | string | `"debug"` | `src/cli/args-1.ts:104` |
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | boolean (`a6`) | `false` | `src/cli/args-1.ts:76` |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | boolean (`a6`) | `false` | `src/tools/repl-1.ts:372` |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | boolean (`a6`) | `false` | `src/llm/api-1.ts:416` |
| `CLAUDE_CODE_SHELL` | string (path) | auto-detected | `src/cli/args-1.ts:3852` |
| `CLAUDE_CODE_TMPDIR` | string (path) | `/tmp` | `src/cli/args-1.ts:3902` |
| `DEBUG` | boolean (`a6`) | `false` | `src/cli/args-1.ts:111` |
| `DEBUG_SDK` | boolean (`a6`) | `false` | `src/cli/args-1.ts:112` |

### `CLAUDE_CODE_SIMPLE` — Bare / Simple Mode

**Source:** `src/cli/args-1.ts:52` — `zY()` function.

Activated by `CLAUDE_CODE_SIMPLE=1` **or** `--bare` CLI flag.

When active:
- Disables hooks, LSP, plugin sync, attribution, auto-memory, background prefetches, keychain reads, CLAUDE.md auto-discovery
- Authentication is strictly `ANTHROPIC_API_KEY` or `apiKeyHelper` via `--settings` (OAuth and keychain are never read)
- 3P providers (Bedrock/Vertex/Foundry) use their own credentials
- Skills still resolve via `/skill-name`
- Useful for: CI/CD pipelines, containerized workloads, scripted automation

### `CLAUDE_CODE_ENTRYPOINT` — Entrypoint Identifier

**Source:** `src/vendor/commander.ts:4699-4710`, `src/core/of6-4.ts:148`

Set programmatically by the runtime based on how Claude Code was invoked. Can also be set externally to override the detected entrypoint.

| Value | Context |
|-------|---------|
| `cli` | Standard terminal invocation (default) |
| `sdk-ts` | TypeScript SDK |
| `sdk-py` | Python SDK |
| `sdk-cli` | SDK CLI wrapper |
| `mcp` | MCP server mode |
| `claude-code-github-action` | GitHub Actions |
| `claude-vscode` | VS Code extension |
| `claude-desktop` | Claude Desktop app |
| `local-agent` | Local agent mode |
| `remote` | Remote session |

Effects: Controls which background tasks run, whether prompts are shown, whether OAuth is disabled, and how the user-agent string is constructed.

### `CLAUDE_CODE_DEBUG_LOG_LEVEL`

**Source:** `src/cli/args-1.ts:104`

Accepts: `verbose`, `debug`, `info`, `warn`, `error` (case-insensitive). Default: `debug`.

Maps to numeric levels internally: `verbose=0`, `debug=1`, `info=2`, `warn=3`, `error=4`.

### `CLAUDE_CODE_SHELL`

**Source:** `src/cli/args-1.ts:3852`

Forces a specific shell binary path for bash command execution. Must be a valid bash or zsh path — if the value doesn't match, a warning is logged and detection falls back to `SHELL` env var.

### `CLAUDE_CODE_TMPDIR`

**Source:** `src/cli/args-1.ts:3902`, `src/tools/tool-1.ts:14326`

Overrides the temporary file directory. Defaults to `/tmp` on Linux/macOS, `TEMP` env var or `C:\Temp` on Windows.

---

## 5. Feature Flags & Toggles

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `CLAUDE_CODE_BRIEF` | boolean (`a6`) | `false` | `src/agents/startdeferredprefetches-1.ts:2662` |
| `CLAUDE_CODE_BRIEF_UPLOAD` | boolean (`a6`) | `false` | `src/core/f_4-2.ts:67` |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | boolean (truthy) | `false` | `src/conversation/session-1.ts:875` |
| `DISABLE_TELEMETRY` | boolean (truthy) | `false` | `src/conversation/session-1.ts:988` |
| `DISABLE_ERROR_REPORTING` | boolean (truthy) | `false` | `src/conversation/session-1.ts:874` |
| `DISABLE_COMPACT` | boolean (`a6`) | `false` | `src/conversation/session-2.ts:257` |
| `CLAUDE_AFTER_LAST_COMPACT` | boolean (`a6`) | `false` | `src/conversation/session-1.ts:2231` |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | boolean (`a6`) | `false` | `src/vendor/dom.ts:55940` |
| `ENABLE_TOOL_SEARCH` | string | — | `src/tools/modelsupportstoolreference-1.ts:66` |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | boolean (`a6`) | `false` | `src/conversation/session-1.ts:6283` |
| `CLAUDE_CODE_DISABLE_FAST_MODE` | boolean (`a6`) | `false` | `src/core/config-1.ts:2494` |
| `CLAUDE_CODE_SKIP_FAST_MODE_NETWORK_ERRORS` | boolean (`a6`) | `false` | `src/core/config-1.ts:2538` |
| `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK` | boolean (`a6`) | `false` | `src/core/config-1.ts:6003` |
| `DISABLE_INTERLEAVED_THINKING` | boolean (`a6`) | `false` | `src/cli/args-1.ts:2329` |
| `USE_API_CONTEXT_MANAGEMENT` | boolean (`a6`) | `false` | `src/cli/args-1.ts:2339` |
| `USE_BUILTIN_RIPGREP` | boolean (`dY` — opt-out) | `true` | `src/cli/args-1.ts:2390` |
| `CLAUDE_CODE_ENABLE_CFC` | boolean (`a6`/`dY`) | auto | `src/core/config-1.ts:11237` |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | boolean (`a6`) | `false` | `src/cli/args-1.ts:2368` |
| `CLAUDE_CODE_RESUME_INTERRUPTED_TURN` | string | — | `src/tools/runheadless-1.ts:373` |
| `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` | string | `"true"` | `src/tools/runheadless-1.ts:806` |
| `CLAUDE_CODE_USE_CCR_V2` | boolean (`a6`) | `false` | `src/tools/runheadless-1.ts:2162` |
| `CLAUDE_CODE_USE_POWERSHELL_TOOL` | boolean (`a6`) | `false` | `src/tools/processbashcommand-3.ts:21` |
| `CLAUDE_CODE_IDE_SKIP_VALID_CHECK` | boolean (`a6`) | `false` | `src/cli/args-1.ts:6578` |
| `CLAUDE_CODE_IDE_SKIP_AUTO_INSTALL` | boolean (`a6`) | `false` | `src/cli/args-1.ts:6874` |
| `CLAUDE_CODE_MAX_RETRIES` | integer | auto | `src/core/auth-1.ts:14626` |
| `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | boolean (`a6`) | `false` | `src/conversation/session-1.ts:2459` |
| `DISABLE_AUTO_MIGRATE_TO_NATIVE` | boolean (`a6`) | `false` | `src/cli/args-1.ts:8603` |
| `DISABLE_LOGIN_COMMAND` | boolean (truthy) | `false` | `src/core/auth-1.ts:14806` |
| `DISABLE_LOGOUT_COMMAND` | boolean (truthy) | `false` | `src/core/auth-1.ts:14823` |
| `CLAUDE_CODE_ADDITIONAL_PROTECTION` | boolean (`a6`) | `false` | `src/conversation/session-1.ts:1712` |
| `FALLBACK_FOR_ALL_PRIMARY_MODELS` | boolean (truthy) | `false` | `src/core/auth-1.ts:14427` |
| `CLAUDE_CODE_FORCE_FULL_LOGO` | boolean (`a6`) | `false` | `src/core/auth-1.ts:15495` |
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` | boolean (`a6`) | `false` | `src/cli/args-1.ts:4140` |

### `ENABLE_TOOL_SEARCH`

**Source:** `src/tools/modelsupportstoolreference-1.ts:66`

Controls the tool-search (tool reference) feature.

| Value | Behavior |
|-------|----------|
| `"true"` or `"1"` | Always enable tool search |
| `"false"` or `"0"` | Always disable |
| `"auto"` | Enable if model supports it and using first-party Anthropic host |
| `"auto:N"` | Enable with confidence threshold N (numeric) |
| unset | Auto mode — disabled when `ANTHROPIC_BASE_URL` is a non-Anthropic host |

When `ANTHROPIC_BASE_URL` is set to a custom proxy, tool search defaults to disabled unless explicitly set.

### `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`

**Source:** `src/conversation/session-1.ts:875`, `src/core/config-1.ts:2682`

Any truthy string value disables: telemetry, error reporting, non-critical background HTTP calls. Implies `DISABLE_TELEMETRY` and `DISABLE_ERROR_REPORTING`.

### `DISABLE_TELEMETRY` / `DISABLE_ERROR_REPORTING`

Any truthy string (not parsed through `a6`) — checked as `!!process.env.DISABLE_TELEMETRY`. These also suppress telemetry when any of the 3P provider flags (`CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`, `CLAUDE_CODE_USE_FOUNDRY`) are set.

### `CLAUDE_CODE_ENABLE_CFC`

**Source:** `src/core/config-1.ts:11237`

Controls the CFC (Claude for Chrome) feature. Checked with both `a6` and `dY`, making it a three-state flag: explicitly enabled, explicitly disabled, or auto (neither).

### `USE_BUILTIN_RIPGREP`

**Source:** `src/cli/args-1.ts:2390` — uses `dY()` (explicit-disable).

Default behavior is to use the bundled ripgrep binary. Set `USE_BUILTIN_RIPGREP=false` (or `0`, `no`, `off`) to fall back to system `rg`.

---

## 6. Plugin & Hook Environment

### Variables Set BY Claude Code for Hook Processes

These variables are injected into the environment of every hook subprocess. They are set in `src/tools/hasworktreecreatehook-1.ts:413-424`.

| Variable | Set By | Value | Source |
|----------|--------|-------|--------|
| `CLAUDE_PROJECT_DIR` | Runtime | Absolute path to current project root | `src/tools/hasworktreecreatehook-1.ts:413` |
| `CLAUDE_PLUGIN_ROOT` | Runtime | Absolute path to plugin directory | `src/tools/hasworktreecreatehook-1.ts:415` |
| `CLAUDE_PLUGIN_DATA` | Runtime | Plugin data directory (from source) | `src/tools/hasworktreecreatehook-1.ts:415` |
| `CLAUDE_PLUGIN_OPTION_<KEY>` | Runtime | Plugin config values (key uppercased) | `src/tools/hasworktreecreatehook-1.ts:420` |
| `CLAUDE_ENV_FILE` | Runtime | Path to session env file (SessionStart/Setup only) | `src/tools/hasworktreecreatehook-1.ts:424` |

The `${CLAUDE_PLUGIN_ROOT}` and `${CLAUDE_PLUGIN_DATA}` tokens in hook command strings are also interpolated before execution.

**`CLAUDE_PROJECT_DIR`** — The project root directory. Derived from `R9()` (resolves current working directory). Set for every hook execution regardless of plugin context.

**`CLAUDE_PLUGIN_ROOT`** — Absolute path to the plugin's installation directory. Only set when the hook belongs to a plugin that has a `path`. Also set via fallback at line 422 if `H` (override root) is present.

**`CLAUDE_PLUGIN_DATA`** — Plugin data directory. Computed from `Fc(source)` — the plugin source identifier. Only set when the plugin has a source (`$`).

**`CLAUDE_PLUGIN_OPTION_<KEY>`** — Plugin configuration key-value pairs from the plugin's `options` object. Key names are uppercased and non-alphanumeric characters replaced with `_`. For example, `{ "timeout": 30 }` becomes `CLAUDE_PLUGIN_OPTION_TIMEOUT=30`. **Source:** `src/mcp/mcp-2.ts:796` (documentation), `src/tools/hasworktreecreatehook-1.ts:420` (implementation).

**`CLAUDE_ENV_FILE`** — Path to a shell script that provides session environment variables. Written to a per-session directory. Only injected for `SessionStart` and `Setup` hook events. **Source:** `src/tools/hasworktreecreatehook-1.ts:423`.

### Variables That Configure Plugin Behavior

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` | boolean (`a6`) | `false` | `src/core/ux8-1.ts:20` |
| `CLAUDE_CODE_SYNC_PLUGIN_INSTALL` | boolean (`a6`) | `false` | `src/conversation/setup-1.ts:124` |
| `CLAUDE_CODE_SYNC_PLUGIN_INSTALL_TIMEOUT_MS` | integer (ms) | auto | `src/tools/runheadless-1.ts:608` |
| `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS` | integer (ms) | auto | `src/core/config-1.ts:6834` |
| `CLAUDE_CODE_OTEL_HEADERS_HELPER_DEBOUNCE_MS` | integer (ms) | auto | `src/core/validateforceloginorg-2.ts:852` |

**`CLAUDE_CODE_SUBPROCESS_ENV_SCRUB`** — When set, the `nB()` function in `src/core/ux8-1.ts` strips sensitive environment variables from the env passed to hook/subprocess invocations. Specifically removes all variables in the `nD9` list (see Section 14) plus their `INPUT_<NAME>` prefixed variants (for GitHub Actions context scrubbing). **Source:** `src/core/ux8-1.ts:20`.

**`CLAUDE_CODE_SYNC_PLUGIN_INSTALL`** — Forces plugin installation to be synchronous during setup, blocking startup until complete. Normally async. **Source:** `src/conversation/setup-1.ts:124`.

### User-Facing Hook Environment Variable

**`CLAUDE_ENV_FILE`** — Can also be set by the user (not just by Claude Code) to point to a shell script that will be sourced to populate session environment. Checked in `cn7()` at `src/conversation/session-1.ts:2500`. If the file doesn't exist, the error is silently ignored (only non-ENOENT errors are logged).

---

## 7. Sandboxing & Security

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `CLAUDE_CODE_BUBBLEWRAP` | string `"1"` | — | `src/conversation/setup-1.ts:149` |
| `IS_SANDBOX` | string `"1"` | — | `src/conversation/setup-1.ts:148`, `src/core/auth-1.ts:14439` |
| `CLAUDE_CODE_SHELL_PREFIX` | string | — | `src/tools/hasworktreecreatehook-1.ts:409`, `src/core/auth-1.ts:14265` |
| `IS_DEMO` | boolean (truthy) | — | `src/core/auth-1.ts:15557` |

**`CLAUDE_CODE_BUBBLEWRAP`** — Signals that Claude Code is running inside a Linux bubblewrap sandbox. When `CLAUDE_CODE_BUBBLEWRAP=1`, certain network setup steps are skipped during initialization. **Source:** `src/conversation/setup-1.ts:149`.

**`IS_SANDBOX`** — Indicates sandbox mode. When `IS_SANDBOX=1`, keychain operations and certain auth flows are disabled. Also suppresses some fallback behaviors. **Source:** `src/conversation/setup-1.ts:148`, `src/core/auth-1.ts:14439`.

**`CLAUDE_CODE_SHELL_PREFIX`** — A command prefix prepended to shell invocations. Used to wrap bash commands with sandbox tooling (e.g., `bwrap`, `firejail`, custom security wrappers). Applied to hook commands and interactive bash tool calls. **Source:** `src/tools/hasworktreecreatehook-1.ts:409`, `src/core/auth-1.ts:14265`.

**`IS_DEMO`** — Marks the session as a demo instance. Suppresses logout prompts and organization-name display. **Source:** `src/core/auth-1.ts:15557`.

---

## 8. Network & Proxy Configuration

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `HTTP_PROXY` | string (URL) | — | `src/core/tba-5.ts:48`, `src/networking/agent-1.ts:4315` |
| `HTTPS_PROXY` | string (URL) | — | `src/core/tba-5.ts:51`, `src/networking/agent-1.ts:4315` |
| `http_proxy` | string (URL) | — | `src/core/tba-5.ts:48`, `src/networking/agent-1.ts:4315` |
| `https_proxy` | string (URL) | — | `src/core/tba-5.ts:51`, `src/networking/agent-1.ts:4315` |
| `NO_PROXY` | string | — | `src/core/tba-5.ts:112`, `src/networking/agent-1.ts:4318` |
| `no_proxy` | string | — | `src/core/tba-5.ts:112`, `src/conversation/streaming-1.ts:8162` |
| `ANTHROPIC_UNIX_SOCKET` | string (path) | — | `src/networking/agent-1.ts:4387` |
| `CLAUDE_CODE_PROXY_RESOLVES_HOSTS` | boolean (`a6`) | `false` | `src/networking/agent-1.ts:4349` |
| `CLAUDE_CODE_HOST_HTTP_PROXY_PORT` | integer | — | `src/core/filesystem-1.ts:3443` |

### Proxy Resolution Order

For HTTPS: `https_proxy` → `HTTPS_PROXY` → `http_proxy` → `HTTP_PROXY` **Source:** `src/core/tba-5.ts:48-51`, `src/conversation/streaming-1.ts:8251-8266`.

For no-proxy: `no_proxy` → `NO_PROXY` **Source:** `src/core/tba-5.ts:112`, `src/networking/agent-1.ts:4318`.

Both uppercase and lowercase variants are checked; lowercase takes precedence in `tba-5.ts`.

**`CLAUDE_CODE_PROXY_RESOLVES_HOSTS`** — When set, enables a mode where the proxy server performs DNS resolution rather than the Claude Code process. **Source:** `src/networking/agent-1.ts:4349`.

**`CLAUDE_CODE_HOST_HTTP_PROXY_PORT`** — Proxy port passed as a `--setenv` argument to bubblewrap sandbox. Set by the host process for sandboxed subprocess networking. **Source:** `src/core/filesystem-1.ts:3443`.

**`ANTHROPIC_UNIX_SOCKET`** — Path to a Unix domain socket for direct API communication. When set, standard TCP connections are bypassed. **Source:** `src/networking/agent-1.ts:4387`, `src/vendor/commander.ts:1643`.

---

## 9. OpenTelemetry & Observability

### Claude Code-Specific OTEL Variables

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `OTEL_METRICS_INCLUDE_SESSION_ID` | boolean (`a6`) | `true` | `src/conversation/session-2.ts:86` |
| `OTEL_METRICS_INCLUDE_VERSION` | boolean (`a6`) | `false` | `src/conversation/session-2.ts:87` |
| `OTEL_METRICS_INCLUDE_ACCOUNT_UUID` | boolean (`a6`) | `true` | `src/conversation/session-2.ts:88` |
| `OTEL_LOG_USER_PROMPTS` | boolean (`a6`) | `false` | `src/conversation/session-2.ts:91` |
| `OTEL_LOG_TOOL_DETAILS` | boolean (`a6`) | `false` | `src/agents/agent-1.ts:829` |
| `OTEL_LOG_TOOL_CONTENT` | boolean (`a6`) | `false` | `src/ui/markdown-1.ts:1004` |
| `CLAUDE_CODE_WORKSPACE_HOST_PATHS` | string (pipe-separated) | — | `src/conversation/session-2.ts:114` |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | boolean (`a6`) | `false` | `src/vendor/dom.ts:55940` |
| `CLAUDE_CODE_ACCOUNT_TAGGED_ID` | string | — | `src/conversation/session-2.ts:68` |
| `OTEL_LOGS_EXPORT_INTERVAL` | integer (ms) | auto | `src/conversation/shutdown1peventlogging-2.ts:100` |

**`OTEL_METRICS_INCLUDE_SESSION_ID`** — Controls whether `session.id` is included as an attribute in OTEL metric data points. Default: `true`. **Source:** `src/conversation/session-2.ts:49,86`.

**`OTEL_METRICS_INCLUDE_VERSION`** — Controls whether `app.version` is included in OTEL metric attributes. Default: `false`. **Source:** `src/conversation/session-2.ts:50,87`.

**`OTEL_METRICS_INCLUDE_ACCOUNT_UUID`** — Controls whether `user.account_uuid` and `user.account_id` are included in OTEL metric attributes. Default: `true`. **Source:** `src/conversation/session-2.ts:65,88`.

**`OTEL_LOG_USER_PROMPTS`** — When `false` (default), user prompt text is replaced with `"<REDACTED>"` in telemetry. When `true`, prompts are logged in plaintext. **Source:** `src/conversation/session-2.ts:91`, `src/ui/markdown-1.ts:763`.

**`CLAUDE_CODE_WORKSPACE_HOST_PATHS`** — Pipe-separated (`|`) list of workspace host paths. Added as `workspace.host_paths` array attribute to telemetry events. **Source:** `src/conversation/session-2.ts:114`.

**`CLAUDE_CODE_ENABLE_TELEMETRY`** — Master switch for third-party OpenTelemetry export. When `false` (default), OTEL exporters are not initialized. Must be explicitly enabled. **Source:** `src/vendor/dom.ts:55940`.

### Standard OpenTelemetry SDK Variables

The following standard OTEL SDK variables are consumed by the bundled OpenTelemetry SDK:

| Variable | Purpose | Source |
|----------|---------|--------|
| `OTEL_EXPORTER_OTLP_HEADERS` | Auth/custom headers for OTLP exporter (all signals) | `src/core/config-1.ts:5222` (scrub list) |
| `OTEL_EXPORTER_OTLP_LOGS_HEADERS` | OTLP headers for logs | `src/core/config-1.ts:5223` (scrub list) |
| `OTEL_EXPORTER_OTLP_METRICS_HEADERS` | OTLP headers for metrics | `src/core/config-1.ts:5224` (scrub list) |
| `OTEL_EXPORTER_OTLP_TRACES_HEADERS` | OTLP headers for traces | `src/core/config-1.ts:5225` (scrub list) |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP endpoint URL | `src/vendor/dom.ts:31687` |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | Transport protocol (`grpc`, `http/protobuf`, etc.) | `src/vendor/dom.ts:12689` |
| `OTEL_EXPORTER_OTLP_TIMEOUT` | Export timeout (ms) | `src/vendor/dom.ts:31606` |
| `OTEL_EXPORTER_OTLP_COMPRESSION` | Compression type | `src/vendor/dom.ts:31619` |
| `OTEL_EXPORTER_OTLP_LOGS_PROTOCOL` | Protocol override for logs | `src/vendor/dom.ts:12684` |
| `OTEL_EXPORTER_OTLP_METRICS_PROTOCOL` | Protocol override for metrics | `src/vendor/dom.ts:12688` |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | `cumulative` or `delta` | `src/vendor/dom.ts:32044` |
| `OTEL_EXPORTER_PROMETHEUS_HOST` | Prometheus exporter host | `src/vendor/dom.ts:32513` |
| `OTEL_EXPORTER_PROMETHEUS_PORT` | Prometheus exporter port | `src/vendor/dom.ts:32517` |
| `OTEL_RESOURCE_ATTRIBUTES` | Resource attribute string | `src/core/msa-5.ts:66` |
| `OTEL_SERVICE_NAME` | Service name override | `src/core/msa-5.ts:67` |
| `OTEL_LOGS_EXPORTER` | Logs exporter type | `src/vendor/dom.ts:12693` |
| `OTEL_METRICS_EXPORTER` | Metrics exporter type | `src/vendor/dom.ts:12695` |
| `OTEL_METRIC_EXPORT_INTERVAL` | Metrics export interval (ms) | `src/vendor/dom.ts:12694` |
| `OTEL_TRACES_SAMPLER` | Sampling strategy | `src/vendor/dom.ts:33231` |
| `OTEL_TRACES_SAMPLER_ARG` | Sampler argument (0.0–1.0 for ratio) | `src/vendor/dom.ts:33257` |
| `OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT` | Max attribute value length | `src/vendor/dom.ts:33202` |
| `OTEL_ATTRIBUTE_COUNT_LIMIT` | Max attribute count per span | `src/vendor/dom.ts:33205` |
| `OTEL_SPAN_ATTRIBUTE_VALUE_LENGTH_LIMIT` | Span-level attribute length limit | `src/vendor/dom.ts:33209` |
| `OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT` | Span-level attribute count limit | `src/vendor/dom.ts:33212` |
| `OTEL_SPAN_LINK_COUNT_LIMIT` | Max links per span | `src/vendor/dom.ts:33214` |
| `OTEL_SPAN_EVENT_COUNT_LIMIT` | Max events per span | `src/vendor/dom.ts:33216` |
| `OTEL_LOGRECORD_ATTRIBUTE_VALUE_LENGTH_LIMIT` | Log record attribute length limit | `src/core/ica-14.ts:440` |
| `OTEL_LOGRECORD_ATTRIBUTE_COUNT_LIMIT` | Log record attribute count limit | `src/core/ica-14.ts:443` |
| `OTEL_BLRP_MAX_EXPORT_BATCH_SIZE` | Batch log processor batch size | `src/core/errors-1.ts:621` |
| `OTEL_BLRP_MAX_QUEUE_SIZE` | Batch log processor queue size | `src/core/errors-1.ts:625` |
| `OTEL_BLRP_SCHEDULE_DELAY` | Batch log processor schedule delay (ms) | `src/core/errors-1.ts:629` |
| `OTEL_BLRP_EXPORT_TIMEOUT` | Batch log processor export timeout | `src/core/errors-1.ts:633` |

Note: The four `OTEL_EXPORTER_OTLP_*_HEADERS` variables are in the subprocess scrub list (`nD9`) because they may contain authentication tokens.

---

## 10. System & Platform Variables

| Variable | Type | Used For | Source |
|----------|------|----------|--------|
| `HOME` | string | User home directory | OS standard |
| `USERPROFILE` | string | Windows user home | `src/cli/args-1.ts:6464` |
| `XDG_CONFIG_HOME` | string | Config dir (Linux) | `src/cli/args-1.ts:2501`, `src/vendor/axios.ts:535` |
| `XDG_DATA_HOME` | string | Data dir (Linux) | `src/vendor/axios.ts:534` |
| `XDG_CACHE_HOME` | string | Cache dir (Linux) | `src/vendor/axios.ts:536` |
| `PATH` | string | Command lookup | OS standard |
| `SHELL` | string | Default shell detection | `src/cli/args-1.ts:2477` |
| `USER` | string | Username fallback | `src/core/auth-1.ts:11646` |
| `TMPDIR` | string | Temp dir (macOS/Linux) | `src/core/git-1.ts:728` |
| `TEMP` | string | Temp dir (Windows) | `src/core/uploadbriefattachment-2.ts:30` |
| `APPDATA` | string | Roaming app data (Windows) | `src/core/config-1.ts:3459`, `src/vendor/axios.ts:521` |
| `LOCALAPPDATA` | string | Local app data (Windows) | `src/cli/args-1.ts:7018`, `src/vendor/axios.ts:522` |
| `NODE_OPTIONS` | string | Node.js process options | `src/cli/args-1.ts:34` |
| `NODE_TLS_REJECT_UNAUTHORIZED` | string | TLS cert validation | Node.js standard |
| `NODE_ENV` | string | Node environment | Node.js standard |
| `CLAUDE_CODE_HOST_PLATFORM` | string | Platform override for telemetry | `src/vendor/axios.ts:302` |
| `WSL_DISTRO_NAME` | string | WSL detection & path translation | `src/cli/args-1.ts:6475`, `src/tools/tool-3.ts:578` |
| `UV_THREADPOOL_SIZE` | integer | libuv thread pool | `src/tools/definitions-1.ts:3227` |

**`XDG_CONFIG_HOME`** — When set, overrides the default `~/.config` path for XDG-compliant config directory resolution. **Source:** `src/cli/args-1.ts:2501`.

**`CLAUDE_CODE_HOST_PLATFORM`** — Overrides `process.platform` for telemetry reporting. Used in containerized/remote environments where the host platform differs from the container platform. **Source:** `src/vendor/axios.ts:302`.

**`WSL_DISTRO_NAME`** — Indicates Windows Subsystem for Linux. Used to translate between Windows and Linux paths via `vG6` class. **Source:** `src/cli/args-1.ts:6475`, `src/tools/tool-3.ts:578`.

**`NODE_OPTIONS`** — Standard Node.js environment variable for process-wide V8/Node options. Claude Code reads this in `S$6()` to check if specific flags are present. **Source:** `src/cli/args-1.ts:34`.

---

## 11. Terminal & Display

| Variable | Type | Used For | Source |
|----------|------|----------|--------|
| `TERM` | string | Terminal type detection | `src/vendor/axios.ts:264` |
| `TERM_PROGRAM` | string | Terminal program name | `src/vendor/axios.ts:266` |
| `COLORTERM` | string | Color support detection | OS standard |
| `FORCE_COLOR` | string | Force color output | Node.js/chalk standard |
| `NO_COLOR` | string | Disable color output | Standard |
| `COLUMNS` | integer | Terminal width | OS standard |
| `LINES` | integer | Terminal height | OS standard |
| `TMUX` | string | tmux session detection | `src/vendor/axios.ts:267` |
| `STY` | string | GNU screen detection | `src/vendor/axios.ts:268` |
| `CLAUDE_CODE_TMUX_SESSION` | string | tmux session name | `src/core/auth-1.ts:15517` |
| `CLAUDE_CODE_TMUX_PREFIX` | string | tmux prefix key | `src/core/auth-1.ts:15531` |
| `CLAUDE_CODE_TMUX_PREFIX_CONFLICTS` | boolean | tmux prefix conflict flag | `src/core/auth-1.ts:15530` |
| `CLAUDE_CODE_SSE_PORT` | integer | IDE SSE server port | `src/cli/args-1.ts:6568` |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | boolean (`a6`) | Disable terminal title updates | `src/tools/repl-1.ts:372` |

### Terminal Emulator Detection

The function in `src/vendor/axios.ts` detects the terminal by checking these variables in order:

1. `CURSOR_TRACE_ID` → `"cursor"`
2. `VSCODE_GIT_ASKPASS_MAIN` containing `"cursor"` → `"cursor"`
3. `VSCODE_GIT_ASKPASS_MAIN` containing `"windsurf"` → `"windsurf"`
4. `VSCODE_GIT_ASKPASS_MAIN` containing `"antigravity"` → `"zed"`
5. `__CFBundleIdentifier` (macOS app bundle) → app-specific
6. `VisualStudioVersion` → `"visualstudio"`
7. `TERMINAL_EMULATOR === "JetBrains-JediTerm"` → `"jetbrains"`
8. `TERM === "xterm-ghostty"` → `"ghostty"`
9. `TERM` containing `"kitty"` → `"kitty"`
10. `TERM_PROGRAM` → value of `TERM_PROGRAM`
11. `TMUX` → `"tmux"`
12. `STY` → `"screen"`
13. `KONSOLE_VERSION` → `"konsole"`
14. `GNOME_TERMINAL_SERVICE` → `"gnome-terminal"`
15. `XTERM_VERSION` → `"xterm"`
16. `VTE_VERSION` → `"vte-based"`
17. `TERMINATOR_UUID` → `"terminator"`
18. `KITTY_WINDOW_ID` → `"kitty"`
19. `ALACRITTY_LOG` → `"alacritty"`
20. `TILIX_ID` → `"tilix"`
21. `WT_SESSION` → `"windows-terminal"`
22. `SESSIONNAME` + `TERM === "cygwin"` → `"cygwin"`
23. `MSYSTEM` → value of `MSYSTEM` (lowercased)
24. `ConEmuANSI`, `ConEmuPID`, `ConEmuTask` → `"conemu"`
25. `WSL_DISTRO_NAME` → `` `wsl-${WSL_DISTRO_NAME}` ``
26. `TERM` → value of `TERM`

### SSH Detection

SSH session is detected by the presence of `SSH_CONNECTION`, `SSH_CLIENT`, or `SSH_TTY`. **Source:** `src/vendor/axios.ts:296`, `src/conversation/session-1.ts:5861`.

---

## 12. CI/CD & Automation Detection

| Variable | Type | Used For | Source |
|----------|------|----------|--------|
| `CI` | boolean | General CI detection | Standard |
| `GITHUB_ACTIONS` | boolean (`a6`) | GitHub Actions detection | `src/conversation/session-1.ts:972` |
| `GITHUB_ACTOR` | string | GH Actions actor name | `src/conversation/session-1.ts:974` |
| `GITHUB_ACTOR_ID` | string | GH Actions actor ID | `src/conversation/session-1.ts:975` |
| `GITHUB_REPOSITORY` | string | GH Actions repository | `src/conversation/session-1.ts:976` |
| `GITHUB_REPOSITORY_ID` | string | GH Actions repo ID | `src/conversation/session-1.ts:977` |
| `GITHUB_REPOSITORY_OWNER` | string | GH Actions repo owner | `src/conversation/session-1.ts:978` |
| `GITHUB_REPOSITORY_OWNER_ID` | string | GH Actions repo owner ID | `src/conversation/session-1.ts:979` |
| `GITHUB_EVENT_NAME` | string | GH Actions event type | `src/conversation/session-1.ts:1952` |
| `GITHUB_ACTION_PATH` | string | GH Actions action path | `src/conversation/session-1.ts:1955` |
| `RUNNER_ENVIRONMENT` | string | GH Actions runner env | `src/conversation/session-1.ts:1953` |
| `RUNNER_OS` | string | GH Actions runner OS | `src/conversation/session-1.ts:1954` |
| `GITLAB_CI` | boolean (`a6`) | GitLab CI detection | `src/vendor/axios.ts:436` |
| `CIRCLECI` | boolean (truthy) | CircleCI detection | `src/vendor/axios.ts:437` |
| `BUILDKITE` | boolean (truthy) | Buildkite detection | `src/vendor/axios.ts:438` |
| `CODESPACES` | boolean (`a6`) | GitHub Codespaces | `src/vendor/axios.ts:397` |
| `GITPOD_WORKSPACE_ID` | string | Gitpod detection | `src/vendor/axios.ts:398` |
| `REPL_ID` / `REPL_SLUG` | string | Replit detection | `src/vendor/axios.ts:399` |
| `VERCEL` | boolean (`a6`) | Vercel detection | `src/vendor/axios.ts:401` |
| `RAILWAY_ENVIRONMENT_NAME` / `RAILWAY_SERVICE_NAME` | string | Railway detection | `src/vendor/axios.ts:403` |
| `RENDER` | boolean (`a6`) | Render detection | `src/vendor/axios.ts:407` |
| `NETLIFY` | boolean (`a6`) | Netlify detection | `src/vendor/axios.ts:408` |
| `DYNO` | string | Heroku detection | `src/vendor/axios.ts:409` |
| `FLY_APP_NAME` / `FLY_MACHINE_ID` | string | Fly.io detection | `src/vendor/axios.ts:410` |
| `CF_PAGES` | boolean (`a6`) | Cloudflare Pages | `src/vendor/axios.ts:411` |
| `AWS_EXECUTION_ENV` | string | AWS ECS/Fargate detection | `src/vendor/axios.ts:414` |
| `AWS_LAMBDA_FUNCTION_NAME` | string | AWS Lambda detection | `src/vendor/axios.ts:413` |
| `K_SERVICE` | string | GCP Cloud Run detection | `src/vendor/axios.ts:427` |
| `GOOGLE_CLOUD_PROJECT` | string | GCP detection | `src/vendor/axios.ts:428` |
| `KUBERNETES_SERVICE_HOST` | string | Kubernetes detection | `src/vendor/axios.ts:440` |
| `WEBSITE_SITE_NAME` / `WEBSITE_SKU` | string | Azure App Service | `src/vendor/axios.ts:429` |
| `AZURE_FUNCTIONS_ENVIRONMENT` | string | Azure Functions | `src/vendor/axios.ts:431` |
| `VSCODE_GIT_ASKPASS_MAIN` | string | VS Code / editor detection | `src/vendor/axios.ts:247` |
| `CURSOR_TRACE_ID` | string | Cursor editor detection | `src/vendor/axios.ts:246` |
| `SPACE_CREATOR_USER_ID` | string | HuggingFace Spaces | `src/vendor/axios.ts:434` |
| `DENO_DEPLOYMENT_ID` | string | Deno Deploy | `src/vendor/axios.ts:412` |
| `PROJECT_DOMAIN` | string | Glitch detection | `src/vendor/axios.ts:400` |
| `APP_URL` | string | DigitalOcean App Platform | `src/vendor/axios.ts:432` |
| `CLAUDE_CODE_ACTION` | boolean (`a6`) | Claude Code Action mode | `src/conversation/session-1.ts:1929` |
| `JEST_WORKER_ID` | string | Jest test runner detection | `src/networking/http-1.ts:1273` |
| `ACTIONS_ID_TOKEN_REQUEST_TOKEN` | string | GH Actions OIDC token | `src/core/config-1.ts:5232` (scrub) |
| `ACTIONS_ID_TOKEN_REQUEST_URL` | string | GH Actions OIDC URL | `src/core/config-1.ts:5233` (scrub) |
| `ACTIONS_RUNTIME_TOKEN` | string | GH Actions runtime token | `src/core/config-1.ts:5234` (scrub) |
| `ACTIONS_RUNTIME_URL` | string | GH Actions runtime URL | `src/core/config-1.ts:5235` (scrub) |
| `ALL_INPUTS` | string | GH Actions all inputs | `src/core/config-1.ts:5236` (scrub) |
| `OVERRIDE_GITHUB_TOKEN` | string | GH Actions token override | `src/core/config-1.ts:5237` (scrub) |
| `DEFAULT_WORKFLOW_TOKEN` | string | GH default workflow token | `src/core/config-1.ts:5238` (scrub) |

**GitHub Actions Integration:** When `GITHUB_ACTIONS=1`, Claude Code captures metadata from `GITHUB_ACTOR`, `GITHUB_REPOSITORY`, etc. into telemetry. It also reads `GITHUB_ACTION_PATH` to detect if the `claude-code-action` is being used and which version. **Source:** `src/conversation/session-1.ts:972-979`.

---

## 13. Testing & Development

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `CLAUDE_CODE_STALL_TIMEOUT_MS_FOR_TESTING` | integer (ms) | compiled default | `src/vendor/dom.ts:56906` |
| `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` | integer (ms) | compiled default | `src/tools/hasworktreecreatehook-1.ts:61` |
| `API_TIMEOUT_MS` | integer (ms) | `600000` | `src/conversation/session-1.ts:1726` |
| `DEBUG` | boolean (`a6`) | `false` | `src/cli/args-1.ts:111` |
| `DEBUG_SDK` | boolean (`a6`) | `false` | `src/cli/args-1.ts:112` |
| `DEBUG_AUTH` | boolean (truthy) | `false` | `src/networking/http-1.ts:8187` |
| `DEMO_VERSION` | string | — | `src/conversation/session-1.ts:6931` |

**`CLAUDE_CODE_STALL_TIMEOUT_MS_FOR_TESTING`** — Overrides the internal stall timeout used in testing environments. Allows tests to trigger stall detection faster than the production threshold. **Source:** `src/vendor/dom.ts:56906`.

**`CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`** — Overrides the timeout for `SessionEnd` hooks. Parsed as integer; falls back to compiled default (`mn_`) if not a finite positive number. **Source:** `src/tools/hasworktreecreatehook-1.ts:61` — `Sa6()` function.

**`API_TIMEOUT_MS`** — Overrides the HTTP request timeout for Anthropic API calls. Default: `600000` (10 minutes). **Source:** `src/conversation/session-1.ts:1726`.

**`DEBUG` / `DEBUG_SDK`** — Enable verbose debug logging. The combined debug mode (`gZ`) is active when any of: `DEBUG=1`, `DEBUG_SDK=1`, `--debug` flag, `-d` flag, `--debug-to-stderr`, `--debug-file=<path>`, or debug filter (`--debug=<filter>`). **Source:** `src/cli/args-1.ts:111`.

**`DEBUG_AUTH`** — When truthy, logs auth-related info to console. **Source:** `src/networking/http-1.ts:8187`.

**`DEMO_VERSION`** — When set, enables demo mode and overrides the version display path. **Source:** `src/conversation/session-1.ts:6931`.

---

## 14. Subprocess Environment Handling

### The `nB()` Function

**Source:** `src/core/ux8-1.ts:19-24`

```typescript
function nB() {
  if (!a6(process.env.CLAUDE_CODE_SUBPROCESS_ENV_SCRUB)) return process.env;
  let A = { ...process.env };
  for (let q of nD9) (delete A[q], delete A[`INPUT_${q}`]);
  return A;
}
```

When `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1`, `nB()` returns a scrubbed copy of `process.env` with sensitive variables removed. This copy is passed as the `env` to hook subprocess invocations. Without the flag, the full `process.env` is passed.

The `INPUT_<NAME>` prefix stripping is for GitHub Actions context: GH Actions passes inputs as both `<NAME>` and `INPUT_<NAME>` environment variables.

### The `nD9` Scrub List

**Source:** `src/core/config-1.ts:5216-5240`

The following variables (and their `INPUT_<NAME>` variants) are stripped from subprocess environments when `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` is enabled:

| Variable | Category |
|----------|----------|
| `ANTHROPIC_API_KEY` | Authentication |
| `CLAUDE_CODE_OAUTH_TOKEN` | Authentication |
| `ANTHROPIC_AUTH_TOKEN` | Authentication |
| `ANTHROPIC_FOUNDRY_API_KEY` | Authentication |
| `ANTHROPIC_CUSTOM_HEADERS` | Authentication |
| `OTEL_EXPORTER_OTLP_HEADERS` | Telemetry credentials |
| `OTEL_EXPORTER_OTLP_LOGS_HEADERS` | Telemetry credentials |
| `OTEL_EXPORTER_OTLP_METRICS_HEADERS` | Telemetry credentials |
| `OTEL_EXPORTER_OTLP_TRACES_HEADERS` | Telemetry credentials |
| `AWS_SECRET_ACCESS_KEY` | Cloud credentials |
| `AWS_SESSION_TOKEN` | Cloud credentials |
| `AWS_BEARER_TOKEN_BEDROCK` | Cloud credentials |
| `GOOGLE_APPLICATION_CREDENTIALS` | Cloud credentials |
| `AZURE_CLIENT_SECRET` | Cloud credentials |
| `AZURE_CLIENT_CERTIFICATE_PATH` | Cloud credentials |
| `ACTIONS_ID_TOKEN_REQUEST_TOKEN` | CI/CD secrets |
| `ACTIONS_ID_TOKEN_REQUEST_URL` | CI/CD secrets |
| `ACTIONS_RUNTIME_TOKEN` | CI/CD secrets |
| `ACTIONS_RUNTIME_URL` | CI/CD secrets |
| `ALL_INPUTS` | CI/CD inputs |
| `OVERRIDE_GITHUB_TOKEN` | CI/CD secrets |
| `DEFAULT_WORKFLOW_TOKEN` | CI/CD secrets |
| `SSH_SIGNING_KEY` | SSH credentials |

### Hook Environment Construction

For each hook execution, the environment is built as: `{ ...nB(), CLAUDE_PROJECT_DIR: <cwd> }` plus optionally `CLAUDE_PLUGIN_ROOT`, `CLAUDE_PLUGIN_DATA`, `CLAUDE_PLUGIN_OPTION_*`, and `CLAUDE_ENV_FILE`. **Source:** `src/tools/hasworktreecreatehook-1.ts:413`.

---

## 15. Remote & Agent SDK Variables

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `CLAUDE_CODE_REMOTE` | boolean (`a6`) | `false` | `src/conversation/session-1.ts:1911` |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | string | — | `src/conversation/session-1.ts:1697` |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | string (path) | — | `src/telemetry/events-1.ts:1746` |
| `CLAUDE_CODE_REMOTE_ENVIRONMENT_TYPE` | string | — | `src/conversation/session-1.ts:1914` |
| `CLAUDE_CODE_REMOTE_SEND_KEEPALIVES` | boolean (`a6`) | `false` | `src/conversation/session-1.ts:5037` |
| `CLAUDE_CODE_CONTAINER_ID` | string | — | `src/conversation/session-1.ts:1696` |
| `CLAUDE_CODE_TAGS` | string | — | `src/conversation/session-1.ts:1925` |
| `CLAUDE_AGENT_SDK_VERSION` | string | — | `src/agents/agent-1.ts:960` |
| `CLAUDE_AGENT_SDK_CLIENT_APP` | string | — | `src/conversation/session-1.ts:1698` |
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` | boolean (`a6`) | `false` | `src/cli/args-1.ts:4140` |
| `CLAUBBIT` | boolean (`a6`) | `false` | `src/conversation/session-1.ts:1910` |
| `CLAUDE_CODE_WORKSPACE_HOST_PATHS` | string (pipe-sep) | — | `src/conversation/session-2.ts:114` |

**`CLAUDE_CODE_REMOTE`** — Marks the session as a remote session. Enables keepalive behavior, adjusts memory directory handling, and modifies the auth flow. **Source:** `src/conversation/session-1.ts:1911`.

**`CLAUDE_CODE_REMOTE_MEMORY_DIR`** — Custom path for remote session memory storage. Used instead of the default local memory path in remote environments. **Source:** `src/telemetry/events-1.ts:1746`.

**`CLAUDE_CODE_REMOTE_SEND_KEEPALIVES`** — When set, the session sends periodic keepalive signals to maintain the remote connection. **Source:** `src/conversation/session-1.ts:5037`.

**`CLAUDE_CODE_CONTAINER_ID`** — Identifies the container instance. Added to telemetry as `claudeCodeContainerId`. **Source:** `src/conversation/session-1.ts:1696`.

**`CLAUDE_CODE_TAGS`** — Comma-separated tags attached to telemetry events for grouping/filtering. **Source:** `src/conversation/session-1.ts:1925`.

**`CLAUDE_AGENT_SDK_VERSION`** — Agent SDK version string, appended to the `User-Agent` header as `agent-sdk/<version>`. **Source:** `src/agents/agent-1.ts:960`, `src/core/auth-2.ts:903`.

**`CLAUDE_AGENT_SDK_CLIENT_APP`** — Identifies the client application using the Agent SDK. Appended to `User-Agent` as `client-app/<app>`. **Source:** `src/conversation/session-1.ts:1698`, `src/core/auth-2.ts:906`.

**`CLAUBBIT`** — Internal flag for the Claubbit variant/build. **Source:** `src/conversation/session-1.ts:1910`.

---

## Quick Reference Table

Core Variables Quick Reference.

| Variable | Type | Category | Default | Key Source |
|----------|------|----------|---------|------------|
| `ACTIONS_ID_TOKEN_REQUEST_TOKEN` | string | CI/CD (scrub) | — | `src/core/config-1.ts:5232` |
| `ACTIONS_ID_TOKEN_REQUEST_URL` | string | CI/CD (scrub) | — | `src/core/config-1.ts:5233` |
| `ACTIONS_RUNTIME_TOKEN` | string | CI/CD (scrub) | — | `src/core/config-1.ts:5234` |
| `ACTIONS_RUNTIME_URL` | string | CI/CD (scrub) | — | `src/core/config-1.ts:5235` |
| `ALL_INPUTS` | string | CI/CD (scrub) | — | `src/core/config-1.ts:5236` |
| `API_TIMEOUT_MS` | integer (ms) | Testing | `600000` | `src/conversation/session-1.ts:1726` |
| `ANTHROPIC_API_KEY` | string | Auth | — | `src/core/config-1.ts:12511` |
| `ANTHROPIC_AUTH_TOKEN` | string | Auth (scrub) | — | `src/conversation/session-1.ts:1820` |
| `ANTHROPIC_BASE_URL` | string (URL) | API | `https://api.anthropic.com` | `src/llm/api-1.ts:37` |
| `ANTHROPIC_BEDROCK_BASE_URL` | string (URL) | Bedrock | — | `src/conversation/anthropic-1.ts:26` |
| `ANTHROPIC_BETAS` | string (csv) | API | — | `src/cli/args-1.ts:2348` |
| `ANTHROPIC_CUSTOM_HEADERS` | string (JSON) | API (scrub) | — | `src/conversation/session-1.ts:1825` |
| `ANTHROPIC_CUSTOM_MODEL_OPTION` | string | Model | — | `src/llm/tokenization-1.ts:575` |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION` | string | Model | — | `src/llm/tokenization-1.ts:581` |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_NAME` | string | Model | — | `src/llm/tokenization-1.ts:579` |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | string | Model | — | `src/llm/tokenization-1.ts:443` |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | string | Model | — | `src/llm/tokenization-1.ts:394` |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | string | Model | — | `src/llm/tokenization-1.ts:370` |
| `ANTHROPIC_FOUNDRY_API_KEY` | string | Foundry (scrub) | — | `src/core/anthropic-1.ts:23` |
| `ANTHROPIC_FOUNDRY_AUTH_TOKEN` | string | Foundry | — | `src/tools/tool-1.ts:24567` |
| `ANTHROPIC_FOUNDRY_BASE_URL` | string (URL) | Foundry | — | `src/core/anthropic-1.ts:22` |
| `ANTHROPIC_FOUNDRY_RESOURCE` | string | Foundry | — | `src/core/anthropic-1.ts:24` |
| `ANTHROPIC_MODEL` | string | Model | — | `src/core/resolveskillmodeloverride-1.ts:64` |
| `ANTHROPIC_SMALL_FAST_MODEL` | string | Model | — | `src/core/resolveskillmodeloverride-1.ts:48` |
| `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` | string | Bedrock | — | `src/conversation/session-1.ts:1736` |
| `ANTHROPIC_UNIX_SOCKET` | string (path) | Network | — | `src/networking/agent-1.ts:4387` |
| `ANTHROPIC_VERTEX_PROJECT_ID` | string | Vertex | — | `src/conversation/session-1.ts:1800` |
| `APP_URL` | string | Env detect | — | `src/vendor/axios.ts:432` |
| `APPDATA` | string | System (Win) | — | `src/core/config-1.ts:3459` |
| `AZURE_AUTHORITY_HOST` | string | Foundry/Azure | — | `src/core/auth-1.ts:9093` |
| `AZURE_CLIENT_CERTIFICATE_PATH` | string (scrub) | Azure | — | `src/core/config-1.ts:5231` |
| `AZURE_CLIENT_SECRET` | string (scrub) | Azure | — | `src/core/config-1.ts:5230` |
| `AZURE_FUNCTIONS_ENVIRONMENT` | string | Env detect | — | `src/vendor/axios.ts:431` |
| `AZURE_POD_IDENTITY_AUTHORITY_HOST` | string | Azure | — | `src/core/auth-1.ts:9224` |
| `AWS_BEARER_TOKEN_BEDROCK` | string (scrub) | Bedrock | — | `src/conversation/session-1.ts:1745` |
| `AWS_DEFAULT_REGION` | string | AWS | `us-east-1` | `src/cli/args-1.ts:69` |
| `AWS_EXECUTION_ENV` | string | Env detect | — | `src/vendor/axios.ts:414` |
| `AWS_LAMBDA_FUNCTION_NAME` | string | Env detect | — | `src/vendor/axios.ts:413` |
| `AWS_REGION` | string | AWS | `us-east-1` | `src/cli/args-1.ts:69` |
| `AWS_SECRET_ACCESS_KEY` | string (scrub) | AWS | — | `src/core/config-1.ts:5226` |
| `AWS_SESSION_TOKEN` | string (scrub) | AWS | — | `src/core/config-1.ts:5227` |
| `BUILDKITE` | boolean | CI detect | — | `src/vendor/axios.ts:438` |
| `CF_PAGES` | boolean (`a6`) | Env detect | — | `src/vendor/axios.ts:411` |
| `CI` | boolean | CI detect | — | Standard |
| `CIRCLECI` | boolean | CI detect | — | `src/vendor/axios.ts:437` |
| `CLAUBBIT` | boolean (`a6`) | Remote | `false` | `src/conversation/session-1.ts:1910` |
| `CLAUDE_AFTER_LAST_COMPACT` | boolean (`a6`) | Feature | `false` | `src/conversation/session-1.ts:2231` |
| `CLAUDE_AGENT_SDK_CLIENT_APP` | string | Agent SDK | — | `src/conversation/session-1.ts:1698` |
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` | boolean (`a6`) | Agent SDK | `false` | `src/cli/args-1.ts:4140` |
| `CLAUDE_AGENT_SDK_VERSION` | string | Agent SDK | — | `src/agents/agent-1.ts:960` |
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | boolean (`a6`) | Behavior | `false` | `src/cli/args-1.ts:76` |
| `CLAUDE_CODE_ACCOUNT_TAGGED_ID` | string | OTEL | — | `src/conversation/session-2.ts:68` |
| `CLAUDE_CODE_ACTION` | boolean (`a6`) | CI | `false` | `src/conversation/session-1.ts:1929` |
| `CLAUDE_CODE_ADDITIONAL_PROTECTION` | boolean (`a6`) | Auth | `false` | `src/conversation/session-1.ts:1712` |
| `CLAUDE_CODE_BRIEF` | boolean (`a6`) | Feature | `false` | `src/agents/startdeferredprefetches-1.ts:2662` |
| `CLAUDE_CODE_BRIEF_UPLOAD` | boolean (`a6`) | Feature | `false` | `src/core/f_4-2.ts:67` |
| `CLAUDE_CODE_BUBBLEWRAP` | string `"1"` | Sandbox | — | `src/conversation/setup-1.ts:149` |
| `CLAUDE_CODE_CLIENT_CERT` | string (path) | Auth | — | `src/core/filesystem-2.ts:175` |
| `CLAUDE_CODE_CONTAINER_ID` | string | Remote | — | `src/conversation/session-1.ts:1696` |
| `CLAUDE_CODE_CUSTOM_OAUTH_URL` | string (URL) | Auth | — | `src/mcp/getoauthconfig-2.ts:61` |
| `CLAUDE_CODE_DEBUG_LOG_LEVEL` | string | Logging | `"debug"` | `src/cli/args-1.ts:104` |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | boolean (`a6`) | Feature | `false` | `src/llm/api-1.ts:416` |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | boolean (`a6`) | Feature | `false` | `src/conversation/session-1.ts:6283` |
| `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK` | boolean (`a6`) | Security | `false` | `src/core/config-1.ts:6003` |
| `CLAUDE_CODE_DISABLE_FAST_MODE` | boolean (`a6`) | Feature | `false` | `src/core/config-1.ts:2494` |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | boolean (truthy) | Network | `false` | `src/conversation/session-1.ts:875` |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | boolean (`a6`) | Display | `false` | `src/tools/repl-1.ts:372` |
| `CLAUDE_CODE_ENABLE_CFC` | boolean (`a6`/`dY`) | Feature | auto | `src/core/config-1.ts:11237` |
| `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` | string | Feature | `"true"` | `src/tools/runheadless-1.ts:806` |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | boolean (`a6`) | OTEL | `false` | `src/vendor/dom.ts:55940` |
| `CLAUDE_CODE_ENTRYPOINT` | string | Behavior | `"cli"` | `src/vendor/commander.ts:4710` |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | boolean (`a6`) | Feature | `false` | `src/cli/args-1.ts:2368` |
| `CLAUDE_CODE_FORCE_FULL_LOGO` | boolean (`a6`) | Display | `false` | `src/core/auth-1.ts:15495` |
| `CLAUDE_CODE_HOST_HTTP_PROXY_PORT` | integer | Proxy | — | `src/core/filesystem-1.ts:3443` |
| `CLAUDE_CODE_HOST_PLATFORM` | string | Platform | — | `src/vendor/axios.ts:302` |
| `CLAUDE_CODE_IDE_SKIP_AUTO_INSTALL` | boolean (`a6`) | IDE | `false` | `src/cli/args-1.ts:6874` |
| `CLAUDE_CODE_IDE_SKIP_VALID_CHECK` | boolean (`a6`) | IDE | `false` | `src/cli/args-1.ts:6578` |
| `CLAUDE_CODE_MAX_RETRIES` | integer | API | auto | `src/core/auth-1.ts:14626` |
| `CLAUDE_CODE_OAUTH_CLIENT_ID` | string | Auth | — | `src/mcp/getoauthconfig-2.ts:80` |
| `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` | string | Auth | — | `src/vendor/dom.ts:59127` |
| `CLAUDE_CODE_OAUTH_SCOPES` | string | Auth | — | `src/vendor/dom.ts:59129` |
| `CLAUDE_CODE_OAUTH_TOKEN` | string (scrub) | Auth | — | `src/core/validateforceloginorg-2.ts:146` |
| `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | integer | Auth | — | `src/core/config-2.ts:251` |
| `CLAUDE_CODE_OTEL_HEADERS_HELPER_DEBOUNCE_MS` | integer (ms) | OTEL | auto | `src/core/validateforceloginorg-2.ts:852` |
| `CLAUDE_CODE_ORGANIZATION_UUID` | string | Auth | — | `src/core/auth-1.ts:12064` |
| `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS` | integer (ms) | Plugins | auto | `src/core/config-1.ts:6834` |
| `CLAUDE_CODE_PROXY_RESOLVES_HOSTS` | boolean (`a6`) | Proxy | `false` | `src/networking/agent-1.ts:4349` |
| `CLAUDE_CODE_REMOTE` | boolean (`a6`) | Remote | `false` | `src/conversation/session-1.ts:1911` |
| `CLAUDE_CODE_REMOTE_ENVIRONMENT_TYPE` | string | Remote | — | `src/conversation/session-1.ts:1914` |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | string (path) | Remote | — | `src/telemetry/events-1.ts:1746` |
| `CLAUDE_CODE_REMOTE_SEND_KEEPALIVES` | boolean (`a6`) | Remote | `false` | `src/conversation/session-1.ts:5037` |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | string | Remote | — | `src/conversation/session-1.ts:1697` |
| `CLAUDE_CODE_RESUME_INTERRUPTED_TURN` | string | Feature | — | `src/tools/runheadless-1.ts:373` |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | string | Auth | — | `src/core/auth-1.ts:12055` |
| `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` | integer (ms) | Hooks | auto | `src/tools/hasworktreecreatehook-1.ts:61` |
| `CLAUDE_CODE_SHELL` | string (path) | Behavior | auto | `src/cli/args-1.ts:3852` |
| `CLAUDE_CODE_SHELL_PREFIX` | string | Sandbox | — | `src/tools/hasworktreecreatehook-1.ts:409` |
| `CLAUDE_CODE_SIMPLE` | boolean (`a6`) | Behavior | `false` | `src/cli/args-1.ts:52` |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | boolean (`a6`) | Bedrock | `false` | `src/conversation/session-1.ts:1742` |
| `CLAUDE_CODE_SKIP_FAST_MODE_NETWORK_ERRORS` | boolean (`a6`) | Feature | `false` | `src/core/config-1.ts:2538` |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | boolean (`a6`) | Foundry | `false` | `src/conversation/session-1.ts:1766` |
| `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | boolean (`a6`) | Feature | `false` | `src/conversation/session-1.ts:2459` |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | boolean (`a6`) | Vertex | `false` | `src/conversation/session-1.ts:1781` |
| `CLAUDE_CODE_SSE_PORT` | integer | IDE | — | `src/cli/args-1.ts:6568` |
| `CLAUDE_CODE_STALL_TIMEOUT_MS_FOR_TESTING` | integer (ms) | Testing | auto | `src/vendor/dom.ts:56906` |
| `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` | boolean (`a6`) | Security | `false` | `src/core/ux8-1.ts:20` |
| `CLAUDE_CODE_SYNC_PLUGIN_INSTALL` | boolean (`a6`) | Plugins | `false` | `src/conversation/setup-1.ts:124` |
| `CLAUDE_CODE_SYNC_PLUGIN_INSTALL_TIMEOUT_MS` | integer (ms) | Plugins | auto | `src/tools/runheadless-1.ts:608` |
| `CLAUDE_CODE_TAGS` | string | Remote | — | `src/conversation/session-1.ts:1925` |
| `CLAUDE_CODE_TMUX_PREFIX` | string | Display | — | `src/core/auth-1.ts:15531` |
| `CLAUDE_CODE_TMUX_PREFIX_CONFLICTS` | boolean | Display | — | `src/core/auth-1.ts:15530` |
| `CLAUDE_CODE_TMUX_SESSION` | string | Display | — | `src/core/auth-1.ts:15517` |
| `CLAUDE_CODE_TMPDIR` | string (path) | Behavior | `/tmp` | `src/cli/args-1.ts:3902` |
| `CLAUDE_CODE_USE_BEDROCK` | boolean (`a6`) | Provider | `false` | `src/llm/api-1.ts:25` |
| `CLAUDE_CODE_USE_CCR_V2` | boolean (`a6`) | Feature | `false` | `src/tools/runheadless-1.ts:2162` |
| `CLAUDE_CODE_USE_FOUNDRY` | boolean (`a6`) | Provider | `false` | `src/llm/api-1.ts:29` |
| `CLAUDE_CODE_USE_POWERSHELL_TOOL` | boolean (`a6`) | Feature | `false` | `src/tools/processbashcommand-3.ts:21` |
| `CLAUDE_CODE_USE_VERTEX` | boolean (`a6`) | Provider | `false` | `src/llm/api-1.ts:27` |
| `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` | integer | Auth | — | `src/core/auth-1.ts:12010` |
| `CLAUDE_CODE_WORKSPACE_HOST_PATHS` | string | OTEL | — | `src/conversation/session-2.ts:114` |
| `CLAUDE_CONFIG_DIR` | string (path) | System | `~/.claude` | `src/core/r8-32.ts:17` |
| `CLAUDE_ENV_FILE` | string (path) | Hooks | — | `src/conversation/session-1.ts:2500` |
| `CLAUDE_PLUGIN_DATA` | string (path) | Hooks (set by CC) | — | `src/tools/hasworktreecreatehook-1.ts:415` |
| `CLAUDE_PLUGIN_OPTION_<KEY>` | string | Hooks (set by CC) | — | `src/tools/hasworktreecreatehook-1.ts:420` |
| `CLAUDE_PLUGIN_ROOT` | string (path) | Hooks (set by CC) | — | `src/tools/hasworktreecreatehook-1.ts:415` |
| `CLAUDE_PROJECT_DIR` | string (path) | Hooks (set by CC) | — | `src/tools/hasworktreecreatehook-1.ts:413` |
| `CLOUD_ML_REGION` | string | Vertex | `us-east5` | `src/cli/args-1.ts:73` |
| `CODESPACES` | boolean (`a6`) | Env detect | — | `src/vendor/axios.ts:397` |
| `CURSOR_TRACE_ID` | string | Terminal detect | — | `src/vendor/axios.ts:246` |
| `DEBUG` | boolean (`a6`) | Logging | `false` | `src/cli/args-1.ts:111` |
| `DEBUG_AUTH` | boolean (truthy) | Logging | `false` | `src/networking/http-1.ts:8187` |
| `DEBUG_SDK` | boolean (`a6`) | Logging | `false` | `src/cli/args-1.ts:112` |
| `DEFAULT_WORKFLOW_TOKEN` | string (scrub) | CI/CD | — | `src/core/config-1.ts:5238` |
| `DEMO_VERSION` | string | Testing | — | `src/conversation/session-1.ts:6931` |
| `DENO_DEPLOYMENT_ID` | string | Env detect | — | `src/vendor/axios.ts:412` |
| `DISABLE_AUTO_MIGRATE_TO_NATIVE` | boolean (`a6`) | Feature | `false` | `src/cli/args-1.ts:8603` |
| `DISABLE_COMPACT` | boolean (`a6`) | Feature | `false` | `src/conversation/session-2.ts:257` |
| `DISABLE_ERROR_REPORTING` | boolean (truthy) | Telemetry | `false` | `src/conversation/session-1.ts:874` |
| `DISABLE_INTERLEAVED_THINKING` | boolean (`a6`) | Feature | `false` | `src/cli/args-1.ts:2329` |
| `DISABLE_LOGIN_COMMAND` | boolean (truthy) | Feature | `false` | `src/core/auth-1.ts:14806` |
| `DISABLE_LOGOUT_COMMAND` | boolean (truthy) | Feature | `false` | `src/core/auth-1.ts:14823` |
| `DISABLE_TELEMETRY` | boolean (truthy) | Telemetry | `false` | `src/conversation/session-1.ts:988` |
| `DYNO` | string | Env detect | — | `src/vendor/axios.ts:409` |
| `ENABLE_TOOL_SEARCH` | string | Feature | auto | `src/tools/modelsupportstoolreference-1.ts:66` |
| `FALLBACK_FOR_ALL_PRIMARY_MODELS` | boolean (truthy) | Feature | `false` | `src/core/auth-1.ts:14427` |
| `FLY_APP_NAME` | string | Env detect | — | `src/vendor/axios.ts:410` |
| `FLY_MACHINE_ID` | string | Env detect | — | `src/vendor/axios.ts:410` |
| `GCLOUD_PROJECT` | string | Vertex | — | `src/conversation/session-1.ts:1787` |
| `GITPOD_WORKSPACE_ID` | string | Env detect | — | `src/vendor/axios.ts:398` |
| `GITHUB_ACTION_PATH` | string | CI | — | `src/conversation/session-1.ts:1955` |
| `GITHUB_ACTIONS` | boolean (`a6`) | CI | — | `src/conversation/session-1.ts:972` |
| `GITHUB_ACTOR` | string | CI | — | `src/conversation/session-1.ts:974` |
| `GITHUB_ACTOR_ID` | string | CI | — | `src/conversation/session-1.ts:975` |
| `GITHUB_EVENT_NAME` | string | CI | — | `src/conversation/session-1.ts:1952` |
| `GITHUB_REPOSITORY` | string | CI | — | `src/conversation/session-1.ts:976` |
| `GITHUB_REPOSITORY_ID` | string | CI | — | `src/conversation/session-1.ts:977` |
| `GITHUB_REPOSITORY_OWNER` | string | CI | — | `src/conversation/session-1.ts:978` |
| `GITHUB_REPOSITORY_OWNER_ID` | string | CI | — | `src/conversation/session-1.ts:979` |
| `GITLAB_CI` | boolean (`a6`) | CI detect | — | `src/vendor/axios.ts:436` |
| `GNOME_TERMINAL_SERVICE` | string | Terminal detect | — | `src/vendor/axios.ts:270` |
| `GOOGLE_APPLICATION_CREDENTIALS` | string (scrub) | Vertex | — | `src/conversation/session-1.ts:1792` |
| `GOOGLE_CLOUD_PROJECT` | string | Vertex/Env | — | `src/conversation/session-1.ts:1788` |
| `GOOGLE_CLOUD_QUOTA_PROJECT` | string | Vertex | — | `src/core/config-1.ts:5033` |
| `HOME` | string | System | — | OS standard |
| `HTTP_PROXY` | string (URL) | Proxy | — | `src/core/tba-5.ts:48` |
| `HTTPS_PROXY` | string (URL) | Proxy | — | `src/core/tba-5.ts:51` |
| `http_proxy` | string (URL) | Proxy | — | `src/core/tba-5.ts:48` |
| `https_proxy` | string (URL) | Proxy | — | `src/core/tba-5.ts:51` |
| `IS_DEMO` | boolean (truthy) | Feature | — | `src/core/auth-1.ts:15557` |
| `IS_SANDBOX` | string `"1"` | Sandbox | — | `src/conversation/setup-1.ts:148` |
| `JEST_WORKER_ID` | string | Testing | — | `src/networking/http-1.ts:1273` |
| `K_SERVICE` | string | Env detect | — | `src/vendor/axios.ts:427` |
| `KITTY_WINDOW_ID` | string | Terminal detect | — | `src/vendor/axios.ts:274` |
| `KONSOLE_VERSION` | string | Terminal detect | — | `src/vendor/axios.ts:269` |
| `KUBERNETES_SERVICE_HOST` | string | Env detect | — | `src/vendor/axios.ts:440` |
| `LOCALAPPDATA` | string | System (Win) | — | `src/cli/args-1.ts:7018` |
| `MSYSTEM` | string | Terminal detect | — | `src/vendor/axios.ts:279` |
| `NETLIFY` | boolean (`a6`) | Env detect | — | `src/vendor/axios.ts:408` |
| `NO_PROXY` | string | Proxy | — | `src/core/tba-5.ts:112` |
| `NODE_ENV` | string | System | — | Node.js standard |
| `NODE_OPTIONS` | string | System | — | `src/cli/args-1.ts:34` |
| `NODE_TLS_REJECT_UNAUTHORIZED` | string | Security | — | Node.js standard |
| `no_proxy` | string | Proxy | — | `src/core/tba-5.ts:112` |
| `OTEL_ATTRIBUTE_COUNT_LIMIT` | integer | OTEL | `128` | `src/vendor/dom.ts:33205` |
| `OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT` | integer | OTEL | — | `src/vendor/dom.ts:33202` |
| `OTEL_BLRP_EXPORT_TIMEOUT` | integer (ms) | OTEL | — | `src/core/errors-1.ts:633` |
| `OTEL_BLRP_MAX_EXPORT_BATCH_SIZE` | integer | OTEL | — | `src/core/errors-1.ts:621` |
| `OTEL_BLRP_MAX_QUEUE_SIZE` | integer | OTEL | — | `src/core/errors-1.ts:625` |
| `OTEL_BLRP_SCHEDULE_DELAY` | integer (ms) | OTEL | — | `src/core/errors-1.ts:629` |
| `OTEL_EXPORTER_OTLP_CERTIFICATE` | string (path) | OTEL | — | `src/vendor/dom.ts:31725` |
| `OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE` | string (path) | OTEL | — | `src/vendor/dom.ts:31711` |
| `OTEL_EXPORTER_OTLP_CLIENT_KEY` | string (path) | OTEL | — | `src/vendor/dom.ts:31718` |
| `OTEL_EXPORTER_OTLP_COMPRESSION` | string | OTEL | — | `src/vendor/dom.ts:31619` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | string (URL) | OTEL | — | `src/vendor/dom.ts:31687` |
| `OTEL_EXPORTER_OTLP_HEADERS` | string (scrub) | OTEL | — | `src/core/config-1.ts:5222` |
| `OTEL_EXPORTER_OTLP_LOGS_HEADERS` | string (scrub) | OTEL | — | `src/core/config-1.ts:5223` |
| `OTEL_EXPORTER_OTLP_LOGS_PROTOCOL` | string | OTEL | — | `src/vendor/dom.ts:12684` |
| `OTEL_EXPORTER_OTLP_METRICS_CLIENT_CERTIFICATE` | string | OTEL | — | `src/vendor/dom.ts:12685` |
| `OTEL_EXPORTER_OTLP_METRICS_CLIENT_KEY` | string | OTEL | — | `src/vendor/dom.ts:12686` |
| `OTEL_EXPORTER_OTLP_METRICS_HEADERS` | string (scrub) | OTEL | — | `src/core/config-1.ts:5224` |
| `OTEL_EXPORTER_OTLP_METRICS_PROTOCOL` | string | OTEL | — | `src/vendor/dom.ts:12688` |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | string | OTEL | `cumulative` | `src/vendor/dom.ts:32044` |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | string | OTEL | — | `src/vendor/dom.ts:12689` |
| `OTEL_EXPORTER_OTLP_TIMEOUT` | integer (ms) | OTEL | — | `src/vendor/dom.ts:31606` |
| `OTEL_EXPORTER_OTLP_TRACES_HEADERS` | string (scrub) | OTEL | — | `src/core/config-1.ts:5225` |
| `OTEL_EXPORTER_PROMETHEUS_HOST` | string | OTEL | — | `src/vendor/dom.ts:32513` |
| `OTEL_EXPORTER_PROMETHEUS_PORT` | integer | OTEL | — | `src/vendor/dom.ts:32517` |
| `OTEL_LOG_TOOL_CONTENT` | boolean (`a6`) | OTEL | `false` | `src/ui/markdown-1.ts:1004` |
| `OTEL_LOG_TOOL_DETAILS` | boolean (`a6`) | OTEL | `false` | `src/agents/agent-1.ts:829` |
| `OTEL_LOG_USER_PROMPTS` | boolean (`a6`) | OTEL | `false` | `src/conversation/session-2.ts:91` |
| `OTEL_LOGRECORD_ATTRIBUTE_COUNT_LIMIT` | integer | OTEL | — | `src/core/ica-14.ts:443` |
| `OTEL_LOGRECORD_ATTRIBUTE_VALUE_LENGTH_LIMIT` | integer | OTEL | — | `src/core/ica-14.ts:440` |
| `OTEL_LOGS_EXPORT_INTERVAL` | integer (ms) | OTEL | — | `src/conversation/shutdown1peventlogging-2.ts:100` |
| `OTEL_LOGS_EXPORTER` | string | OTEL | — | `src/vendor/dom.ts:12693` |
| `OTEL_METRIC_EXPORT_INTERVAL` | integer (ms) | OTEL | — | `src/vendor/dom.ts:12694` |
| `OTEL_METRICS_EXPORTER` | string | OTEL | — | `src/vendor/dom.ts:12695` |
| `OTEL_METRICS_INCLUDE_ACCOUNT_UUID` | boolean (`a6`) | OTEL | `true` | `src/conversation/session-2.ts:88` |
| `OTEL_METRICS_INCLUDE_SESSION_ID` | boolean (`a6`) | OTEL | `true` | `src/conversation/session-2.ts:86` |
| `OTEL_METRICS_INCLUDE_VERSION` | boolean (`a6`) | OTEL | `false` | `src/conversation/session-2.ts:87` |
| `OTEL_RESOURCE_ATTRIBUTES` | string | OTEL | — | `src/core/msa-5.ts:66` |
| `OTEL_SERVICE_NAME` | string | OTEL | — | `src/core/msa-5.ts:67` |
| `OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT` | integer | OTEL | `128` | `src/vendor/dom.ts:33212` |
| `OTEL_SPAN_ATTRIBUTE_VALUE_LENGTH_LIMIT` | integer | OTEL | — | `src/vendor/dom.ts:33209` |
| `OTEL_SPAN_EVENT_COUNT_LIMIT` | integer | OTEL | `128` | `src/vendor/dom.ts:33216` |
| `OTEL_SPAN_LINK_COUNT_LIMIT` | integer | OTEL | `128` | `src/vendor/dom.ts:33214` |
| `OTEL_TRACES_SAMPLER` | string | OTEL | `ParentBasedAlwaysOn` | `src/vendor/dom.ts:33231` |
| `OTEL_TRACES_SAMPLER_ARG` | float (0–1) | OTEL | — | `src/vendor/dom.ts:33257` |
| `OVERRIDE_GITHUB_TOKEN` | string (scrub) | CI/CD | — | `src/core/config-1.ts:5237` |
| `PATH` | string | System | — | OS standard |
| `PROJECT_DOMAIN` | string | Env detect | — | `src/vendor/axios.ts:400` |
| `RAILWAY_ENVIRONMENT_NAME` | string | Env detect | — | `src/vendor/axios.ts:403` |
| `RAILWAY_SERVICE_NAME` | string | Env detect | — | `src/vendor/axios.ts:404` |
| `RENDER` | boolean (`a6`) | Env detect | — | `src/vendor/axios.ts:407` |
| `REPL_ID` | string | Env detect | — | `src/vendor/axios.ts:399` |
| `REPL_SLUG` | string | Env detect | — | `src/vendor/axios.ts:399` |
| `RUNNER_ENVIRONMENT` | string | CI | — | `src/conversation/session-1.ts:1953` |
| `RUNNER_OS` | string | CI | — | `src/conversation/session-1.ts:1954` |
| `SESSIONNAME` | string | Terminal detect | — | `src/vendor/axios.ts:278` |
| `SHELL` | string | System | — | `src/cli/args-1.ts:2477` |
| `SPACE_CREATOR_USER_ID` | string | Env detect | — | `src/vendor/axios.ts:434` |
| `SSH_CLIENT` | string | SSH detect | — | `src/vendor/axios.ts:297` |
| `SSH_CONNECTION` | string | SSH detect | — | `src/vendor/axios.ts:296` |
| `SSH_SIGNING_KEY` | string (scrub) | Security | — | `src/core/config-1.ts:5239` |
| `SSH_TTY` | string | SSH detect | — | `src/vendor/axios.ts:298` |
| `STY` | string | Terminal detect | — | `src/vendor/axios.ts:268` |
| `TEMP` | string | System (Win) | — | `src/core/uploadbriefattachment-2.ts:30` |
| `TERMINAL_EMULATOR` | string | Terminal detect | — | `src/vendor/axios.ts:260` |
| `TERM` | string | Terminal detect | — | `src/vendor/axios.ts:264` |
| `TERM_PROGRAM` | string | Terminal detect | — | `src/vendor/axios.ts:266` |
| `TERMINATOR_UUID` | string | Terminal detect | — | `src/vendor/axios.ts:273` |
| `TILIX_ID` | string | Terminal detect | — | `src/vendor/axios.ts:276` |
| `TMPDIR` | string | System | — | `src/core/git-1.ts:728` |
| `TMUX` | string | Terminal detect | — | `src/vendor/axios.ts:267` |
| `USE_API_CONTEXT_MANAGEMENT` | boolean (`a6`) | Feature | `false` | `src/cli/args-1.ts:2339` |
| `USE_BUILTIN_RIPGREP` | boolean (`dY`) | Feature | `true` | `src/cli/args-1.ts:2390` |
| `USER` | string | Auth | — | `src/core/auth-1.ts:11646` |
| `USERPROFILE` | string | System (Win) | — | `src/cli/args-1.ts:6464` |
| `UV_THREADPOOL_SIZE` | integer | System | — | `src/tools/definitions-1.ts:3227` |
| `VERCEL` | boolean (`a6`) | Env detect | — | `src/vendor/axios.ts:401` |
| `VisualStudioVersion` | string | Terminal detect | — | `src/vendor/axios.ts:259` |
| `VTE_VERSION` | string | Terminal detect | — | `src/vendor/axios.ts:272` |
| `VSCODE_GIT_ASKPASS_MAIN` | string | Terminal/CI detect | — | `src/vendor/axios.ts:247` |
| `WEBSITE_SITE_NAME` | string | Env detect | — | `src/vendor/axios.ts:429` |
| `WEBSITE_SKU` | string | Env detect | — | `src/vendor/axios.ts:429` |
| `WSL_DISTRO_NAME` | string | System | — | `src/cli/args-1.ts:6475` |
| `WT_SESSION` | string | Terminal detect | — | `src/vendor/axios.ts:277` |
| `XDG_CACHE_HOME` | string | System | — | `src/vendor/axios.ts:536` |
| `XDG_CONFIG_HOME` | string | System | — | `src/cli/args-1.ts:2501` |
| `XDG_DATA_HOME` | string | System | — | `src/vendor/axios.ts:534` |
| `XTERM_VERSION` | string | Terminal detect | — | `src/vendor/axios.ts:271` |

---

## Appendix A: Boolean Parsing Rules

### `a6(value)` — Truthy Check

**Source:** `src/cli/args-1.ts` (lines 37–42)

```javascript
function a6(A) {
  if (!A) return false;
  if (typeof A === "boolean") return A;
  let q = A.toLowerCase().trim();
  return ["1", "true", "yes", "on"].includes(q);
}
```

| Input | Result |
|-------|--------|
| `"1"` | `true` |
| `"true"` | `true` |
| `"TRUE"` | `true` (case-insensitive) |
| `"yes"` | `true` |
| `"YES"` | `true` |
| `"on"` | `true` |
| `"ON"` | `true` |
| `"0"` | `false` |
| `"false"` | `false` |
| `"no"` | `false` |
| `"off"` | `false` |
| `""` (empty) | `false` |
| `undefined` | `false` |
| `null` | `false` |
| `true` (boolean) | `true` |
| `false` (boolean) | `false` |

### `dY(value)` — Explicit-False Check

**Source:** `src/cli/args-1.ts` (lines 43–49)

```javascript
function dY(A) {
  if (A === undefined) return false;
  if (typeof A === "boolean") return !A;
  if (!A) return false;
  let q = A.toLowerCase().trim();
  return ["0", "false", "no", "off"].includes(q);
}
```

| Input | Result | Meaning |
|-------|--------|----------|
| `"0"` | `true` | Feature IS disabled |
| `"false"` | `true` | Feature IS disabled |
| `"no"` | `true` | Feature IS disabled |
| `"off"` | `true` | Feature IS disabled |
| `"1"` | `false` | Feature is NOT disabled |
| `"true"` | `false` | Feature is NOT disabled |
| `undefined` | `false` | Feature is NOT explicitly disabled (absent ≠ disabled) |
| `""` (empty) | `false` | Feature is NOT disabled |
| `false` (boolean) | `true` | Feature IS disabled |
| `true` (boolean) | `false` | Feature is NOT disabled |

**Key difference from `a6`:** `dY(undefined)` returns `false` (not disabled), while `a6(undefined)` also returns `false` (not enabled). For variables parsed with `dY`, the absence of the variable means the feature is **not explicitly disabled** — it may still be on by default.

---

## Appendix B: Provider Decision Tree

```
Startup
  │
  ├─ CLAUDE_CODE_USE_BEDROCK=1 ?
  │    YES → Provider: "bedrock"
  │          Uses: AWS_REGION / AWS_DEFAULT_REGION (default: us-east-1)
  │                ANTHROPIC_BEDROCK_BASE_URL (optional override)
  │                CLAUDE_CODE_SKIP_BEDROCK_AUTH → skip AWS credential chain
  │                AWS_BEARER_TOKEN_BEDROCK → static bearer token
  │                ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION → region for fast model
  │
  ├─ CLAUDE_CODE_USE_VERTEX=1 ?
  │    YES → Provider: "vertex"
  │          Uses: CLOUD_ML_REGION (default: us-east5)
  │                ANTHROPIC_VERTEX_PROJECT_ID
  │                GCLOUD_PROJECT / GOOGLE_CLOUD_PROJECT (fallbacks)
  │                GOOGLE_APPLICATION_CREDENTIALS
  │                CLAUDE_CODE_SKIP_VERTEX_AUTH → skip credential refresh
  │
  ├─ CLAUDE_CODE_USE_FOUNDRY=1 ?
  │    YES → Provider: "foundry" (Azure AI Foundry)
  │          Uses: ANTHROPIC_FOUNDRY_BASE_URL
  │                ANTHROPIC_FOUNDRY_API_KEY (required unless SKIP_FOUNDRY_AUTH)
  │                ANTHROPIC_FOUNDRY_RESOURCE
  │                AZURE_AUTHORITY_HOST
  │                CLAUDE_CODE_SKIP_FOUNDRY_AUTH → skip Azure credential validation
  │
  └─ (none set)
       Provider: "firstParty"
       Uses: ANTHROPIC_API_KEY or OAuth token
             ANTHROPIC_BASE_URL (default: https://api.anthropic.com)
             ANTHROPIC_AUTH_TOKEN (fallback)
             CLAUDE_CODE_OAUTH_TOKEN / CLAUDE_CODE_SESSION_ACCESS_TOKEN
```

**Auth token priority within firstParty:**
1. `CLAUDE_CODE_SESSION_ACCESS_TOKEN` (set programmatically or via env)
2. `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` (file descriptor token)
3. `ANTHROPIC_AUTH_TOKEN` (environment)
4. Keychain / stored OAuth token
5. `CLAUDE_CODE_OAUTH_TOKEN`
6. `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR`
7. `ANTHROPIC_API_KEY` (environment)

---

## Appendix C: Environment Variable Inheritance in Hook Processes

When Claude Code executes a hook command, it constructs the subprocess environment as follows:

```
Step 1: Base environment
  ├─ CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=0 → use full process.env
  └─ CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1 → use process.env minus nD9 list
                                           (also removes INPUT_<NAME> variants)

Step 2: Add project context
  └─ CLAUDE_PROJECT_DIR = absolute path to project root

Step 3: Add plugin context (if hook belongs to a plugin)
  ├─ CLAUDE_PLUGIN_ROOT = absolute path to plugin directory
  └─ CLAUDE_PLUGIN_DATA = plugin data directory

Step 4: Add plugin options (for each option key-value pair)
  └─ CLAUDE_PLUGIN_OPTION_<UPPERCASED_KEY> = String(value)

Step 5: Add session env file path (SessionStart and Setup hooks only)
  └─ CLAUDE_ENV_FILE = path to per-hook session env file

Step 6: Apply CLAUDE_CODE_SHELL_PREFIX (if set)
  └─ Prepend shell prefix to the command string before spawn
```

The working directory (`cwd`) for hook processes is the project directory if it exists, falling back to the original `cwd` of the Claude Code process.

---

## Appendix D: Additional Variables by Feature Area

### D.1 SDK & Bridge Mode Variables

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `CLAUDE_CODE_ENVIRONMENT_KIND` | string | — | `src/agents/startdeferredprefetches-1.ts:249` |
| `CLAUDE_BRIDGE_USE_CCR_V2` | boolean (`a6`) | `false` | `src/conversation/initenvlessbridgecore-2.ts:469` |
| `CLAUDE_CODE_INCLUDE_PARTIAL_MESSAGES` | boolean (`a6`) | `false` | `src/agents/startdeferredprefetches-1.ts:764` |
| `CLAUDE_CODE_QUESTION_PREVIEW_FORMAT` | string | — | `src/agents/startdeferredprefetches-1.ts:240` |
| `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` | boolean (`a6`) | `false` | `src/agents/startdeferredprefetches-1.ts:108` |
| `NoDefaultCurrentDirectoryInExePath` | string `"1"` | — | `src/agents/startdeferredprefetches-1.ts:203` |

**`CLAUDE_CODE_ENVIRONMENT_KIND`** — When set to `"bridge"`, activates bridge mode which modifies how the environment is initialized. **Source:** `src/agents/startdeferredprefetches-1.ts:249`.

**`CLAUDE_BRIDGE_USE_CCR_V2`** — Enables CCR v2 protocol in bridge mode. **Source:** `src/conversation/initenvlessbridgecore-2.ts:469`.

**`CLAUDE_CODE_INCLUDE_PARTIAL_MESSAGES`** — When set, includes partial/streaming messages in the output. Used in SDK contexts. **Source:** `src/agents/startdeferredprefetches-1.ts:764`.

**`CLAUDE_CODE_QUESTION_PREVIEW_FORMAT`** — Controls the format of question previews in non-interactive contexts. **Source:** `src/agents/startdeferredprefetches-1.ts:240`.

**`CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER`** — Causes the process to exit after the first render cycle. Used for snapshot testing and CI render validation. **Source:** `src/agents/startdeferredprefetches-1.ts:108`.

**`NoDefaultCurrentDirectoryInExePath`** — Windows-specific variable set to `"1"` to prevent the current directory from being included in the executable search path. This is a Windows security best practice. **Source:** `src/agents/startdeferredprefetches-1.ts:203`.

### D.2 Node.js-Related Variables

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `NODE_EXTRA_CA_CERTS` | string (path) | — | `src/agents/startdeferredprefetches-1.ts:64` |
| `NODE_OPTIONS` | string | — | `src/cli/args-1.ts:34` |
| `UV_THREADPOOL_SIZE` | integer | Node.js default | `src/tools/definitions-1.ts:3227` |

**`NODE_EXTRA_CA_CERTS`** — Path to additional CA certificates for TLS verification. When set, Claude Code records this in telemetry (`has_node_extra_ca_certs: true`) to help diagnose TLS issues in enterprise environments with custom certificate authorities. **Source:** `src/agents/startdeferredprefetches-1.ts:64`.

**`NODE_OPTIONS`** — Standard Node.js environment variable. Claude Code specifically checks whether `--inspect`, `--inspect-brk`, `--debug`, or `--debug-brk` flags are included, to detect if a debugger is attached. **Source:** `src/agents/startdeferredprefetches-1.ts:49`.

### D.3 Model Override Variables (Detail)

Model resolution follows a priority chain. **Source:** `src/core/resolveskillmodeloverride-1.ts`.

```
For any model slot (main, fast, haiku-class, sonnet-class, opus-class):

1. ANTHROPIC_MODEL          — universal override for main model
2. ANTHROPIC_SMALL_FAST_MODEL — override for the small/fast model slot
3. ANTHROPIC_DEFAULT_OPUS_MODEL   — override for opus-class model slot
4. ANTHROPIC_DEFAULT_SONNET_MODEL — override for sonnet-class model slot  
5. ANTHROPIC_DEFAULT_HAIKU_MODEL  — override for haiku-class model slot
6. ANTHROPIC_CUSTOM_MODEL_OPTION  — custom model identifier
   ANTHROPIC_CUSTOM_MODEL_OPTION_NAME — display name for custom model
   ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION — description for custom model
7. Compiled defaults per model slot
```

| Variable | Purpose | Source |
|----------|---------|--------|
| `ANTHROPIC_MODEL` | Main model override | `src/core/resolveskillmodeloverride-1.ts:64` |
| `ANTHROPIC_SMALL_FAST_MODEL` | Fast model override | `src/core/resolveskillmodeloverride-1.ts:48` |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Opus-class override | `src/core/resolveskillmodeloverride-1.ts:78` |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Sonnet-class override | `src/core/resolveskillmodeloverride-1.ts:84` |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Haiku-class override | `src/core/resolveskillmodeloverride-1.ts:90` |
| `ANTHROPIC_CUSTOM_MODEL_OPTION` | Custom model ID | `src/llm/tokenization-1.ts:575` |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_NAME` | Custom model label | `src/llm/tokenization-1.ts:579` |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION` | Custom model description | `src/llm/tokenization-1.ts:581` |
| `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP` | Disable legacy model name remapping | `src/core/resolveskillmodeloverride-1.ts:236` |

**`CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP`** — When set, disables automatic remapping of legacy model identifiers to current ones. Useful when targeting specific model versions precisely. **Source:** `src/core/resolveskillmodeloverride-1.ts:236`.

### D.4 Temp Directory Variables

| Variable | Type | Default | Source |
|----------|------|---------|--------|
| `CLAUDE_CODE_TMPDIR` | string (path) | `/tmp` | `src/cli/args-1.ts:3902` |
| `CLAUDE_TMPDIR` | string (path) | `/tmp/claude` | `src/core/filesystem-2.ts:1376` |
| `TMPDIR` | string (path) | OS default | `src/core/git-1.ts:728` |
| `TEMP` | string (path) | `C:\Temp` (Win) | `src/core/uploadbriefattachment-2.ts:30` |
| `TMPPREFIX` | string | auto | `src/core/auth-1.ts:14341` |

**`CLAUDE_TMPDIR`** — Sandbox-specific temp directory variable. Set in the bubblewrap sandbox environment to `/tmp/claude` by default. Used by processes running inside the sandbox. **Source:** `src/core/filesystem-2.ts:1376`.

**`TMPPREFIX`** — Set automatically for zsh shells to a subdirectory of `CLAUDE_CODE_TMPDIR`. **Source:** `src/core/auth-1.ts:14341`.

### D.5 Staging & Internal Variables

| Variable | Type | Purpose | Source |
|----------|------|---------|--------|
| `ANTHROPIC_BASE_URL` (staging) | string | When set to `https://api-staging.anthropic.com`, enables staging API mode | `src/conversation/session-3.ts:705` |
| `DISABLE_COMPACT` | boolean (`a6`) | Disables `/compact` command | `src/conversation/session-2.ts:257`, `src/ui/markdown-1.ts:1447` |

---

## Appendix E: GCP Metadata Service Variables

Claude Code bundles GCP auth libraries that read these variables for Vertex AI credential detection:

| Variable | Type | Purpose | Source |
|----------|------|---------|--------|
| `GCE_METADATA_IP` | string | Override GCE metadata server IP | `src/networking/http-1.ts:8040` |
| `GCE_METADATA_HOST` | string | Override GCE metadata server host | `src/networking/http-1.ts:8041` |
| `DETECT_GCP_RETRIES` | integer | Number of GCP detection retries | `src/networking/http-1.ts:8154` |
| `METADATA_SERVER_DETECTION` | string | Control GCP metadata detection mode | `src/networking/http-1.ts:8160` |

**`METADATA_SERVER_DETECTION`** — Controls GCP metadata server detection behavior. Accepted values (case-insensitive): `"none"` (skip detection), `"bios-only"` (BIOS-based detection only), `"ping-only"` (ping-based only), `"bios-and-ping"` (both methods). **Source:** `src/networking/http-1.ts:8160`.

**`GCE_METADATA_IP`** / **`GCE_METADATA_HOST`** — Override the default GCE metadata server endpoint. Useful in environments where the metadata server is at a non-standard address. When set, they also suppress ping-based detection. **Source:** `src/networking/http-1.ts:8040`.

---

## Appendix F: Windows-Specific Behavior

### Windows Path Resolution

Claude Code uses `APPDATA`, `LOCALAPPDATA`, and `USERPROFILE` for Windows-specific path resolution:

```
Config directory:
  APPDATA                    → %APPDATA%\.claude
  (fallback) USERPROFILE     → %USERPROFILE%\.claude

Local data directory:
  LOCALAPPDATA               → %LOCALAPPDATA%\Claude
  (fallback) APPDATA         → %APPDATA%\Claude

Temp directory:
  TEMP                       → used as temp location
  (fallback)                 → C:\Temp
```

### WSL Path Translation

When `WSL_DISTRO_NAME` is set, Claude Code translates between Windows paths and WSL Linux paths using the `vG6` class:

- `vG6(distro).toIDEPath(path)` — converts Linux path to Windows IDE path
- `vG6(distro).toLocalPath(path)` — converts Windows path to Linux path

**Source:** `src/tools/tool-3.ts:578`, `src/cli/args-1.ts:6475`.

---

## Appendix G: Variable Usage Patterns

### Pattern: Three-State Flags

Some variables use both `a6()` (enable) and `dY()` (disable) to create three-state behavior:

```javascript
// CLAUDE_CODE_ENABLE_CFC example pattern:
if (a6(process.env.CLAUDE_CODE_ENABLE_CFC)) return true;   // explicitly ON
if (dY(process.env.CLAUDE_CODE_ENABLE_CFC)) return false;  // explicitly OFF
// else: use heuristic/default
```

Variables using this pattern:
- `CLAUDE_CODE_ENABLE_CFC` (`src/core/config-1.ts:11237`)
- `USE_BUILTIN_RIPGREP` — `dY` only (opt-out from default-on)
- `ENABLE_TOOL_SEARCH` — multi-value string with auto modes

### Pattern: Compile-Time Scrubbed Variables

Variables in the `nD9` list are scrubbed at two levels:
1. **Subprocess env scrub** — removed from hook subprocess environment when `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1`
2. **GitHub Actions INPUT_ prefix** — `INPUT_<NAME>` variants also removed for CI security

### Pattern: Provider-Gated Variables

Many variables are only meaningful when the corresponding provider is active:

| Only meaningful when | Variables |
|---------------------|----------|
| `CLAUDE_CODE_USE_BEDROCK=1` | `AWS_REGION`, `AWS_DEFAULT_REGION`, `AWS_BEARER_TOKEN_BEDROCK`, `ANTHROPIC_BEDROCK_BASE_URL`, `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION`, `CLAUDE_CODE_SKIP_BEDROCK_AUTH` |
| `CLAUDE_CODE_USE_VERTEX=1` | `CLOUD_ML_REGION`, `ANTHROPIC_VERTEX_PROJECT_ID`, `GCLOUD_PROJECT`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_APPLICATION_CREDENTIALS`, `CLAUDE_CODE_SKIP_VERTEX_AUTH` |
| `CLAUDE_CODE_USE_FOUNDRY=1` | `ANTHROPIC_FOUNDRY_BASE_URL`, `ANTHROPIC_FOUNDRY_API_KEY`, `ANTHROPIC_FOUNDRY_RESOURCE`, `AZURE_AUTHORITY_HOST`, `AZURE_POD_IDENTITY_AUTHORITY_HOST`, `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` |
| `CLAUDE_CODE_ENABLE_TELEMETRY=1` | All `OTEL_EXPORTER_*` variables |

### Pattern: Entrypoint-Specific Behavior

| Entrypoint | Specific behavior |
|------------|------------------|
| `cli` | Interactive REPL, OAuth login available, hooks enabled |
| `sdk-ts` / `sdk-py` / `sdk-cli` | SDK mode, no interactive prompts, hooks enabled |
| `mcp` | MCP server, background operation, `CLAUDE_CODE_ENTRYPOINT` set to `"mcp"` |
| `claude-code-github-action` | CI mode, `GITHUB_ACTIONS` metadata captured |
| `local-agent` | Agent mode, no plugin sync (`setup-1.ts:116`) |
| `claude-vscode` | VS Code extension context |
| `claude-desktop` | Desktop app, OAuth via desktop auth |
| `remote` | Remote session, keepalives may be active |

---

## Appendix H: Sensitive Variables — Complete Security Reference

The following variables should be treated as secrets:

### Always Secret (never log, never pass to subprocesses)

| Variable | Why Secret |
|----------|------------|
| `ANTHROPIC_API_KEY` | API authentication key |
| `ANTHROPIC_AUTH_TOKEN` | Authentication token |
| `CLAUDE_CODE_OAUTH_TOKEN` | OAuth access token |
| `ANTHROPIC_FOUNDRY_API_KEY` | Azure AI Foundry API key |
| `ANTHROPIC_CUSTOM_HEADERS` | May contain auth tokens in headers |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM secret |
| `AWS_SESSION_TOKEN` | AWS temporary session credential |
| `AWS_BEARER_TOKEN_BEDROCK` | Bedrock bearer token |
| `AZURE_CLIENT_SECRET` | Azure service principal secret |
| `AZURE_CLIENT_CERTIFICATE_PATH` | Path to Azure client certificate |
| `GOOGLE_APPLICATION_CREDENTIALS` | Google service account key file |
| `OTEL_EXPORTER_OTLP_HEADERS` | May contain OTEL auth tokens |
| `OTEL_EXPORTER_OTLP_LOGS_HEADERS` | OTEL logs auth headers |
| `OTEL_EXPORTER_OTLP_METRICS_HEADERS` | OTEL metrics auth headers |
| `OTEL_EXPORTER_OTLP_TRACES_HEADERS` | OTEL traces auth headers |
| `SSH_SIGNING_KEY` | SSH private signing key |

### CI/CD Context Secrets (GitHub Actions)

| Variable | Why Secret |
|----------|------------|
| `ACTIONS_ID_TOKEN_REQUEST_TOKEN` | OIDC token for cloud provider auth |
| `ACTIONS_ID_TOKEN_REQUEST_URL` | OIDC token request endpoint |
| `ACTIONS_RUNTIME_TOKEN` | GitHub Actions runtime token |
| `ACTIONS_RUNTIME_URL` | Actions service URL |
| `ALL_INPUTS` | Contains all workflow inputs |
| `OVERRIDE_GITHUB_TOKEN` | GitHub token override |
| `DEFAULT_WORKFLOW_TOKEN` | Default GitHub workflow token |

### Session Tokens (treated as secrets at runtime)

| Variable | Notes |
|----------|-------|
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | Set/read at runtime; not in `nD9` scrub list |
| `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` | FD number pointing to secret |
| `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | FD number pointing to OAuth secret |
| `ANTHROPIC_FOUNDRY_AUTH_TOKEN` | Foundry auth token |

---

## Appendix I: Environment Detection Reference

### Cloud/Hosting Platform Detection

Claude Code detects the hosting environment for telemetry. Detection is performed in `src/vendor/axios.ts` checking these indicators:

| Environment | Detection Variables |
|-------------|--------------------|
| GitHub Codespaces | `CODESPACES=true` |
| Gitpod | `GITPOD_WORKSPACE_ID` present |
| Replit | `REPL_ID` or `REPL_SLUG` present |
| Glitch | `PROJECT_DOMAIN` present |
| Vercel | `VERCEL=true` |
| Railway | `RAILWAY_ENVIRONMENT_NAME` or `RAILWAY_SERVICE_NAME` present |
| Render | `RENDER=true` |
| Netlify | `NETLIFY=true` |
| Heroku | `DYNO` present |
| Fly.io | `FLY_APP_NAME` or `FLY_MACHINE_ID` present |
| Cloudflare Pages | `CF_PAGES=true` |
| Deno Deploy | `DENO_DEPLOYMENT_ID` present |
| AWS Lambda | `AWS_LAMBDA_FUNCTION_NAME` present |
| AWS ECS Fargate | `AWS_EXECUTION_ENV=AWS_ECS_FARGATE` |
| AWS ECS EC2 | `AWS_EXECUTION_ENV=AWS_ECS_EC2` |
| GCP Cloud Run | `K_SERVICE` present |
| GCP (generic) | `GOOGLE_CLOUD_PROJECT` present |
| Azure App Service | `WEBSITE_SITE_NAME` or `WEBSITE_SKU` present |
| Azure Functions | `AZURE_FUNCTIONS_ENVIRONMENT` present |
| DigitalOcean App Platform | `APP_URL` contains `ondigitalocean.app` |
| HuggingFace Spaces | `SPACE_CREATOR_USER_ID` present |
| GitHub Actions | `GITHUB_ACTIONS=true` |
| GitLab CI | `GITLAB_CI=true` |
| CircleCI | `CIRCLECI` present |
| Buildkite | `BUILDKITE` present |
| Kubernetes | `KUBERNETES_SERVICE_HOST` present |

### Editor/IDE Detection

Detection chain for the editor context (affects UI behavior and plugin behavior):

| Editor | Detection |
|--------|----------|
| Cursor | `CURSOR_TRACE_ID` present, OR `VSCODE_GIT_ASKPASS_MAIN` contains `"cursor"` |
| Windsurf | `VSCODE_GIT_ASKPASS_MAIN` contains `"windsurf"` |
| Zed | `VSCODE_GIT_ASKPASS_MAIN` contains `"antigravity"` |
| Conductor | `__CFBundleIdentifier=com.conductor.app` |
| Visual Studio | `VisualStudioVersion` present |
| JetBrains | `TERMINAL_EMULATOR=JetBrains-JediTerm` |
| VS Code | `VSCODE_GIT_ASKPASS_MAIN` present (without other matches) |

---

## Appendix J: Variable Interactions & Conflicts

### Conflicting Authentication

When multiple auth variables are set, the priority order is:

```
1. CLAUDE_CODE_SESSION_ACCESS_TOKEN   (highest priority)
2. CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR
3. ANTHROPIC_AUTH_TOKEN
4. Stored OAuth token (keychain)
5. CLAUDE_CODE_OAUTH_TOKEN
6. CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR
7. ANTHROPIC_API_KEY                  (lowest priority)
```

Note: In `CLAUDE_CODE_SIMPLE` mode, only `ANTHROPIC_API_KEY` and `apiKeyHelper` are checked. All OAuth paths are skipped.

### Provider Mutual Exclusivity

Only one of `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`, and `CLAUDE_CODE_USE_FOUNDRY` should be set at a time. If multiple are set, the first truthy one in the evaluation order wins (Bedrock → Vertex → Foundry).

### Telemetry Disable Precedence

Telemetry is disabled if **any** of these conditions is met:
- `DISABLE_TELEMETRY` is set to any truthy value
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` is set to any truthy value
- `CLAUDE_CODE_USE_BEDROCK=1` OR `CLAUDE_CODE_USE_VERTEX=1` OR `CLAUDE_CODE_USE_FOUNDRY=1` (provider-mode sessions disable telemetry)
- `DISABLE_ERROR_REPORTING` is set

Note: `CLAUDE_CODE_ENABLE_TELEMETRY=1` enables **third-party** OTEL export separately from Anthropic's internal telemetry.

### Debug Mode Activation

Debug mode (`gZ()`) activates if **any** of these is true:
- `DEBUG=1`
- `DEBUG_SDK=1`
- `--debug` CLI flag
- `-d` CLI flag
- `--debug-to-stderr` / `-d2e` CLI flag
- `--debug=<filter>` CLI flag
- `--debug-file=<path>` CLI flag

**Source:** `src/cli/args-1.ts:105-122`.

---

## Appendix K: Migration & Compatibility Notes

### `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`

This variable serves as a compound disable switch. Setting it:
- Disables Anthropic telemetry events
- Disables error reporting
- Disables all non-critical background HTTP calls (update checks, stats)
- Equivalent to setting both `DISABLE_TELEMETRY` and `DISABLE_ERROR_REPORTING`

Use this in air-gapped or strict corporate environments.

### API Key vs OAuth

- `ANTHROPIC_API_KEY` — for direct API access, automation, CI/CD
- `CLAUDE_CODE_OAUTH_TOKEN` — for human-authenticated sessions
- `CLAUDE_CODE_SESSION_ACCESS_TOKEN` — for programmatic/agent sessions where a session token is provisioned externally

In `CLAUDE_CODE_SIMPLE=1` mode, only `ANTHROPIC_API_KEY` is accepted for first-party auth.

### Provider Variables That Don't Exist (Common Mistakes)

| Variable (does NOT exist) | What to use instead |
|--------------------------|---------------------|
| `CLAUDE_CODE_BEDROCK_REGION` | `AWS_REGION` or `AWS_DEFAULT_REGION` |
| `CLAUDE_CODE_VERTEX_REGION` | `CLOUD_ML_REGION` |
| `CLAUDE_VERTEX_PROJECT_ID` | `ANTHROPIC_VERTEX_PROJECT_ID` |
| `BEDROCK_API_KEY` | `AWS_BEARER_TOKEN_BEDROCK` |
| `CLAUDE_OAUTH_TOKEN` | `CLAUDE_CODE_OAUTH_TOKEN` |
| `CLAUDE_CODE_API_KEY` | `ANTHROPIC_API_KEY` |

---

*End of document. Source: Claude Code v2.1.81, build 2026-03-20T21:25:42Z.*
