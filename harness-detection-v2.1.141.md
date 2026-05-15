# Harness and Host Detection in Claude Code 2.1.141

This document describes how the reconstructed 2.1.141 source detects or
classifies external harnesses, SDK hosts, remote workers, IDE hosts, CI
environments, and related execution surfaces.

The scan did not find literal detectors for `hermes`, `openclaw`, `open claw`,
or obvious case variants in the reconstructed 2.1.141 source. If those harnesses
are detected in this release, the visible mechanism is indirect: they must set
generic SDK/remote/entrypoint/client-app environment fields, run Claude through
the SDK stream protocol, expose IDE lockfiles, or otherwise appear through
terminal/process/environment heuristics.

Primary source files:

- `source/src/main.tsx`
- `source/src/core/Session.ts`
- `source/src/entrypoints/agentSdkTypes.ts`
- `source/src/cli/print.ts`
- `source/src/cli/structuredIO.ts`
- `source/src/cli/remoteIO.ts`
- `source/src/utils/env.ts`
- `source/src/utils/envDynamic.ts`
- `source/src/utils/ide.ts`
- `source/src/utils/http.ts`
- `source/src/constants/system.ts`
- `source/src/services/api/client.ts`
- `source/src/services/analytics/metadata.ts`
- `source/src/services/analytics/growthbook.ts`
- `source/src/utils/headlessProfiler.ts`
- `source/src/utils/permissions/filesystem.ts`
- `source/src/bridge/sessionRunner.ts`
- `source/src/daemon/main.ts`
- `source/src/server/backends/dangerousBackend.ts`

## Executive Summary

Claude Code 2.1.141 does not appear to have a named allowlist or classifier for
third-party harness names such as Hermes or OpenClaw. Instead, it identifies the
execution surface through a set of generic signals:

- `CLAUDE_CODE_ENTRYPOINT`
- `CLAUDE_AGENT_SDK_VERSION`
- `CLAUDE_AGENT_SDK_CLIENT_APP`
- `CLAUDE_CODE_REMOTE`
- `CLAUDE_CODE_ENVIRONMENT_KIND`
- `CLAUDE_CODE_SESSION_ACCESS_TOKEN`
- `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR`
- `CLAUDE_CODE_REMOTE_SESSION_ID`
- `CLAUDE_CODE_CONTAINER_ID`
- `GITHUB_ACTIONS` / `CLAUDE_CODE_ACTION`
- `process.stdout.isTTY`
- terminal and IDE environment variables
- IDE extension lockfiles under `~/.claude/ide`
- SDK control messages over print/stream-json

The main classification result is stored in bootstrap state as `clientType`.
The raw entrypoint remains in `process.env.CLAUDE_CODE_ENTRYPOINT` and is reused
by analytics, HTTP headers, attribution headers, behavior gates, and startup
decisions.

## Literal Search Result

Local source scan result:

- No literal `hermes`.
- No literal `openclaw`.
- No literal `open claw`.
- No specific third-party harness enum for those names.

Related generic terms do exist:

- `harness` appears in comments around settings/hooks, internal permission
  paths, Claude Code hint processing, tests/conformance, and remote harness
  metadata.
- `client-app` and `CLAUDE_AGENT_SDK_CLIENT_APP` are the intended generic way
  for SDK consumers to identify their app/library.
- `CLAUDE_CODE_ENTRYPOINT` is the primary execution-surface selector.

## Entrypoint Detection

`main.tsx` owns the CLI entrypoint initialization.

The function `initializeEntrypoint(isNonInteractive)` exits early if
`CLAUDE_CODE_ENTRYPOINT` is already set. This is important: SDKs, remote
workers, local agents, daemons, and other wrappers can pre-label the process.

If the variable is absent, Claude sets it as follows:

- `mcp` when argv contains `mcp serve`;
- `claude-code-github-action` when `CLAUDE_CODE_ACTION` is truthy;
- `sdk-cli` when the session is non-interactive;
- `cli` otherwise.

Non-interactive detection is based on:

- `-p`;
- `--print`;
- `--init-only`;
- `--sdk-url...`;
- `!process.stdout.isTTY`.

A normal third-party harness that launches `claude -p` without setting
`CLAUDE_CODE_ENTRYPOINT` will therefore be classified as `sdk-cli`.

## Client Type Detection

After entrypoint initialization, `main.tsx` derives a higher-level `clientType`:

- `GITHUB_ACTIONS` -> `github-action`
- `CLAUDE_CODE_ENTRYPOINT=sdk-ts` -> `sdk-typescript`
- `CLAUDE_CODE_ENTRYPOINT=sdk-py` -> `sdk-python`
- `CLAUDE_CODE_ENTRYPOINT=sdk-cli` -> `sdk-cli`
- `CLAUDE_CODE_ENTRYPOINT=claude-vscode` -> `claude-vscode`
- `CLAUDE_CODE_ENTRYPOINT=local-agent` -> `local-agent`
- `CLAUDE_CODE_ENTRYPOINT=claude-desktop` -> `claude-desktop`
- `CLAUDE_CODE_ENTRYPOINT=remote` -> `remote`
- presence of `CLAUDE_CODE_SESSION_ACCESS_TOKEN` -> `remote`
- presence of `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` -> `remote`
- fallback -> `cli`

The value is stored through `setClientType(clientType)` in bootstrap state.

Behavior affected by client type includes:

- question preview format defaults;
- remote session source tagging;
- analytics metadata;
- remote/non-interactive checks;
- attribution and commit behavior;
- UI/SDK feature gates.

## SDK Harness Detection

There are two SDK launch paths in the recovered source.

### Agent SDK Session Class

`source/src/core/Session.ts` defines an SDK-facing `Session` class. Its
environment builder sets:

```text
CLAUDE_CODE_ENTRYPOINT=sdk-ts
```

unless the caller already provided an entrypoint in `options.env` or the parent
environment.

It also injects trace context through `TRACEPARENT` and `TRACESTATE` unless the
caller explicitly supplied those environment variables.

The SDK session resolves a Claude executable by trying:

- `options.pathToClaudeCodeExecutable`;
- optional native package `@anthropic-ai/claude-agent-sdk-linux-x64`;
- optional native package `@anthropic-ai/claude-code-linux-x64`;
- local `./cli.js`.

### Agent SDK Query Implementation

`source/src/entrypoints/agentSdkTypes.ts` has another spawn path. It builds a
child environment that preserves or sets:

```text
CLAUDE_CODE_ENTRYPOINT=<opts.env value | parent env value | sdk-ts>
```

It spawns `claude` or `CLAUDE_CODE_PATH`, streams stdout NDJSON, and sends SDK
messages over stdin. In practice, a harness built on this SDK path appears as an
SDK entrypoint unless it deliberately overrides the env.

## SDK Client App Identification

There is a generic app/library identification variable:

```text
CLAUDE_AGENT_SDK_CLIENT_APP
```

The HTTP helper documents it as the way SDK consumers identify their app or
library, for example `my-app/1.0.0`.

When set, it is propagated through:

- HTTP `User-Agent` as `client-app/<value>`;
- MCP user agent as `client-app/<value>`;
- API request header `x-client-app`;
- analytics metadata indirectly through HTTP/request metadata and user-agent
  surfaces.

There is no validation that maps `CLAUDE_AGENT_SDK_CLIENT_APP=hermes` or
`openclaw` to a special internal enum in the recovered source. Such values would
travel as generic client-app metadata.

## SDK Version Identification

`CLAUDE_AGENT_SDK_VERSION` is another generic SDK harness signal.

When present:

- `utils/http.ts` adds `agent-sdk/<version>` to the API User-Agent;
- `getMCPUserAgent()` adds `agent-sdk/<version>`;
- `services/analytics/metadata.ts` adds `agentSdkVersion`;
- first-party event logging converts it to `agent_sdk_version`;
- `headlessProfiler.ts` can segment by entrypoint, and global metadata carries
  the SDK version.

## Print / Stream Harness Protocol

External harnesses commonly interact through `claude -p` with stream-json.

`--sdk-url` is the strongest built-in harness signal. It causes `main.tsx` to
auto-set:

- `inputFormat = stream-json`;
- `outputFormat = stream-json`;
- `verbose = true`;
- `print = true`.

`print.ts` then uses `RemoteIO` instead of plain `StructuredIO`.

Without `--sdk-url`, a harness can still use:

```bash
claude -p --input-format stream-json --output-format stream-json --verbose
```

In both cases, the same structured protocol is active:

- stdin user messages;
- stdout SDK messages;
- control requests;
- control responses;
- hook callbacks;
- MCP messages;
- permissions;
- elicitation;
- environment variable updates;
- keep-alives.

This is not a named harness detector; it is a protocol detector. A harness using
this path is classified by entrypoint/client-app/env metadata.

## Remote Harness Detection

Remote mode is recognized through environment and session-ingress credentials.

Important variables:

- `CLAUDE_CODE_REMOTE`
- `CLAUDE_CODE_REMOTE_SESSION_ID`
- `CLAUDE_CODE_REMOTE_ENVIRONMENT_TYPE`
- `CLAUDE_CODE_CONTAINER_ID`
- `CLAUDE_CODE_ENVIRONMENT_KIND`
- `CLAUDE_CODE_SESSION_ACCESS_TOKEN`
- `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR`
- `CLAUDE_CODE_ENVIRONMENT_RUNNER_VERSION`
- `CLAUDE_CODE_USE_CCR_V2`
- `CLAUDE_CODE_WORKER_EPOCH`

`main.tsx` classifies any process with `CLAUDE_CODE_ENTRYPOINT=remote` or a
session-ingress credential as `clientType=remote`.

`RemoteIO`:

- reads session-ingress auth;
- adds `Authorization: Bearer <token>`;
- adds `x-environment-runner-version` when present;
- chooses transport from the SDK URL;
- detects bridge mode through `CLAUDE_CODE_ENVIRONMENT_KIND=bridge`;
- initializes CCR v2 when `CLAUDE_CODE_USE_CCR_V2` is truthy.

Analytics environment context includes:

- `isClaudeCodeRemote`;
- `remoteEnvironmentType`;
- `claudeCodeContainerId`;
- `claudeCodeRemoteSessionId`;
- `tags`;
- `isLocalAgentMode`.

API request headers include:

- `x-claude-remote-container-id`;
- `x-claude-remote-session-id`.

## Bridge / Remote-Control Harness

The bridge session runner spawns child Claude processes as print-mode SDK
workers:

```text
--print
--sdk-url <url>
--session-id <session>
--input-format stream-json
--output-format stream-json
--replay-user-messages
```

The child environment includes:

- `CLAUDE_CODE_ENVIRONMENT_KIND=bridge`
- `CLAUDE_CODE_SESSION_ACCESS_TOKEN=<token>`
- `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2=1`
- optional `CLAUDE_CODE_USE_CCR_V2=1`
- optional `CLAUDE_CODE_WORKER_EPOCH=<epoch>`

`main.tsx` also tags sessions created through `claude remote-control` by calling
`setSessionSource('remote-control')` when
`CLAUDE_CODE_ENVIRONMENT_KIND=bridge`.

This is the clearest built-in "remote harness" path.

## Daemon, Server, Background, and Local-Agent Entrypoints

Several internal launchers pre-label child processes with
`CLAUDE_CODE_ENTRYPOINT`.

Observed values in 2.1.141 include:

- `daemon`
- `daemon-worker`
- `bg`
- `server`
- `local-agent`
- `claude-desktop`
- `claude-vscode`
- `remote`
- `remote_baku`
- `remote_desktop`
- `remote_mobile`
- `mcp`
- `sdk-cli`
- `sdk-ts`
- `sdk-py`
- `cli`
- `claude-code-github-action`
- `claude_in_slack`

Not every value is set in `main.tsx`; some are set by daemon/server/bridge/agent
launchers or are handled by HTTP platform mapping.

The local-agent value receives special treatment:

- `main.tsx` preserves pre-set `local-agent`;
- analytics records `isLocalAgentMode`;
- some settings and setup behavior skip local-agent;
- embedded search tools are disabled for `local-agent`;
- SDK/default tool behavior can differ.

## HTTP Platform Mapping

`utils/http.ts` maps entrypoints to product platform strings:

- `claude-vscode` -> `claude_code_vscode`
- `remote`, `remote_baku`, `remote_desktop`, `remote_mobile` ->
  `claude_code_remote`
- `sdk-cli`, `sdk-ts`, `sdk-py` -> `claude_code_sdk`
- `mcp` -> `claude_code_mcp`
- `claude-code-github-action` -> `claude_code_github_action`
- `local-agent` -> `claude_code_local_agent`
- `claude_in_slack` -> `claude_in_slack`
- fallback -> `claude_code_cli`

This mapping is used by API/client surfaces that need a normalized product
platform, not by a Hermes/OpenClaw-specific detector.

## User-Agent Detection Surface

API User-Agent:

```text
claude-cli/<version> (<USER_TYPE>, <CLAUDE_CODE_ENTRYPOINT>[, agent-sdk/<version>][, client-app/<app>][, workload/<tag>])
```

MCP User-Agent:

```text
claude-code/<version> (<CLAUDE_CODE_ENTRYPOINT>, agent-sdk/<version>, client-app/<app>)
```

WebFetch User-Agent:

```text
Claude-User (claude-code/<version>; +https://support.anthropic.com/)
```

These are outbound identification surfaces. A third-party harness can make
itself visible by setting `CLAUDE_AGENT_SDK_CLIENT_APP`; otherwise it will be
indistinguishable from the generic SDK/CLI entrypoint it uses.

## Billing / Attribution Header

`constants/system.ts` builds an attribution header:

```text
x-anthropic-billing-header: cc_version=<version.fingerprint>; cc_entrypoint=<entrypoint>; ...
```

It reads:

- `CLAUDE_CODE_ENTRYPOINT`;
- workload from `utils/workloadContext.ts`;
- optional native client attestation stub.

This header is another outbound surface that records the generic entrypoint, not
a named harness.

## Analytics Metadata

`services/analytics/metadata.ts` enriches all analytics events with:

- `entrypoint`
- `agentSdkVersion`
- `clientType`
- `isInteractive`
- `envContext.terminal`
- `envContext.deploymentEnvironment`
- `envContext.isCi`
- `envContext.isClaudeCodeRemote`
- `envContext.isLocalAgentMode`
- `remoteEnvironmentType`
- `claudeCodeContainerId`
- `claudeCodeRemoteSessionId`
- `tags`
- GitHub Action fields
- SWE-bench fields
- subscription type
- process metrics

First-party event logging converts core fields to snake_case:

- `client_type`
- `entrypoint`
- `agent_sdk_version`
- `is_interactive`

This means a harness can be visible in telemetry if it sets:

- a unique `CLAUDE_CODE_ENTRYPOINT`;
- `CLAUDE_AGENT_SDK_VERSION`;
- `CLAUDE_AGENT_SDK_CLIENT_APP`;
- remote environment variables;
- CI/deployment variables.

But there is no recovered code that maps arbitrary client-app names to special
behavior.

## GrowthBook Feature Context

`services/analytics/growthbook.ts` includes `CLAUDE_CODE_ENTRYPOINT` in
GrowthBook context when present. Feature flags can therefore vary by entrypoint.

The source does not show a literal Hermes/OpenClaw feature branch. It does show
that entrypoint is available to the feature-flag system.

## Terminal Detection

`utils/env.ts` detects the terminal/editor host through environment variables.

Important checks:

- `CURSOR_TRACE_ID` -> `cursor`
- `VSCODE_GIT_ASKPASS_MAIN` containing `cursor` -> `cursor`
- `VSCODE_GIT_ASKPASS_MAIN` containing `windsurf` -> `windsurf`
- `VSCODE_GIT_ASKPASS_MAIN` containing `antigravity` -> `antigravity`
- `__CFBundleIdentifier` containing `vscodium` -> `codium`
- `__CFBundleIdentifier` containing `windsurf` -> `windsurf`
- `__CFBundleIdentifier` containing `com.google.android.studio` ->
  `androidstudio`
- `__CFBundleIdentifier` containing a JetBrains IDE name -> that IDE
- `VisualStudioVersion` -> `visualstudio`
- `TERMINAL_EMULATOR=JetBrains-JediTerm` -> JetBrains fallback
- `TERM=xterm-ghostty` -> `ghostty`
- `TERM` containing `kitty` -> `kitty`
- `TERM_PROGRAM` -> its value
- `TMUX` -> `tmux`
- `STY` -> `screen`
- `KONSOLE_VERSION` -> `konsole`
- `GNOME_TERMINAL_SERVICE` -> `gnome-terminal`
- `XTERM_VERSION` -> `xterm`
- `VTE_VERSION` -> `vte-based`
- `TERMINATOR_UUID` -> `terminator`
- `KITTY_WINDOW_ID` -> `kitty`
- `ALACRITTY_LOG` -> `alacritty`
- `TILIX_ID` -> `tilix`
- `WT_SESSION` -> `windows-terminal`
- `MSYSTEM` -> its lowercased value
- WSL env -> `wsl-<distro>`
- SSH env -> `ssh-session`
- `TERM` fallback
- non-TTY stdout -> `non-interactive`

This terminal value is included in analytics environment context. It is also
used by IDE integration code to infer supported IDE terminals.

No Hermes/OpenClaw terminal-specific environment variable was found.

## Deployment Environment Detection

`utils/env.ts` detects hosting/CI environments:

- Codespaces
- Gitpod
- Replit
- Glitch
- Vercel
- Railway
- Render
- Netlify
- Heroku
- Fly.io
- Cloudflare Pages
- Deno Deploy
- AWS Lambda
- AWS Fargate
- AWS ECS
- AWS EC2
- GCP Cloud Run
- GCP
- Azure App Service
- Azure Functions
- DigitalOcean App Platform
- HuggingFace Spaces
- GitHub Actions
- GitLab CI
- CircleCI
- Buildkite
- generic CI
- Kubernetes
- Docker
- platform-specific unknown fallback

Again, these are generic environment classifiers, not named coding harness
detectors.

## IDE Detection

`utils/ide.ts` supports these IDE types:

- Cursor
- Windsurf
- VS Code
- IntelliJ IDEA
- PyCharm
- WebStorm
- PhpStorm
- RubyMine
- CLion
- GoLand
- Rider
- DataGrip
- AppCode
- DataSpell
- Aqua
- Gateway
- Fleet
- Android Studio

The IDE detector has two layers.

### Terminal-Based IDE Type

`getTerminalIdeType()` returns the terminal IDE type when the terminal is a
supported VS Code-family or JetBrains terminal. It uses `env.terminal` and
`envDynamic.terminal`.

### IDE Lockfile Detection

`detectIDEs(includeInvalid)` reads lockfiles under `~/.claude/ide` and related
platform paths. Lockfiles include:

- workspace folders;
- port;
- PID;
- IDE name;
- transport type `ws` or `sse`;
- Windows/WSL marker;
- auth token.

Validation checks:

- `CLAUDE_CODE_IDE_SKIP_VALID_CHECK` can force validity.
- `CLAUDE_CODE_SSE_PORT` can make a port-valid match regardless of workspace.
- Otherwise current cwd must be inside a reported workspace folder.
- WSL paths are converted and distro-matched.
- In a supported IDE terminal, the IDE PID must be the parent or an ancestor,
  unless the port matches the env port.

The result is a `DetectedIDEInfo` with:

- name;
- port;
- workspace folders;
- URL;
- validity;
- auth token;
- Windows/WSL marker.

This is the path by which Cursor/Windsurf/VS Code/JetBrains integration is
detected. Hermes/OpenClaw would need to emulate an IDE lockfile or terminal
signal to show up here; no named support was found.

## Running IDE Process Detection

`detectRunningIDEs()` uses platform-specific process scans:

- macOS: `ps aux | grep -E` for VS Code, Cursor, Windsurf, JetBrains IDEs,
  Gateway, Fleet, Android Studio.
- Windows: `tasklist | findstr` for `Code.exe`, `Cursor.exe`, `Windsurf.exe`,
  JetBrains executables, Gateway, Fleet, Android Studio.
- Linux: `ps aux | grep -E` for lowercase process names such as `code`,
  `cursor`, `windsurf`, `idea`, `pycharm`, `webstorm`, `fleet`, and
  `android-studio`.

This supports onboarding, extension installation, and suggestions. It is not a
general harness detector.

## Claude Code Hint Harness Protocol

`utils/claudeCodeHints.ts` implements a harness-only side channel:

```xml
<claude-code-hint ... />
```

Shell tools scan stdout/stderr output for these tags, strip them before the
model sees output, and use them as harness-side metadata. Related code appears
in:

- `source/src/utils/claudeCodeHints.ts`
- `source/src/tools/BashTool/BashTool.tsx`
- `source/src/tools/PowerShellTool/PowerShellTool.tsx`
- `source/src/utils/plugins/hintRecommendation.ts`
- `source/src/hooks/useClaudeCodeHintRecommendation.tsx`

This is "Claude Code detects hints emitted by other CLIs/SDKs", not "Claude Code
detects a named harness".

## Internal Harness-Controlled Paths

`utils/permissions/filesystem.ts` uses "harness" language around internal paths.
This is not external harness detection, but it matters because the permission
engine treats some harness-owned files as safe to read.

`checkReadableInternalPath()` allows reads from:

- session memory directory;
- project directory for past session memory;
- plan files for the current session;
- tool results directory;
- scratchpad directory for current session;
- project temp directory;
- agent memory directory;
- auto memory directory;
- tasks directory;
- teams directory;
- bundled skill reference files.

The bundled skill path comment says content under that subtree is
"harness-controlled" because Claude writes it before reading it and the path has
a per-process nonce.

This is a filesystem trust mechanism, not a detector for third-party products.

## Behavior Gated by SDK/Entrypoint

Several behaviors change when the entrypoint indicates SDK or local-agent mode.

Examples:

- `getCLISyspromptPrefix()` uses a different system prompt prefix for
  non-interactive sessions:
  - with appended system prompt: "Claude Code ... running within the Claude
    Agent SDK";
  - without appended system prompt: "You are a Claude agent, built on
    Anthropic's Claude Agent SDK."
- REPL mode is default-on only for internal interactive `cli`, not SDK
  entrypoints.
- Embedded search tools are disabled for `sdk-ts`, `sdk-py`, `sdk-cli`, and
  `local-agent`.
- Built-in Code Guide agent is not included for SDK entrypoints.
- `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` can disable built-in agents in
  non-interactive SDK/API usage.
- Some setup/settings paths skip `local-agent` and `claude-desktop`.

These are generic entrypoint gates.

## Permission Mode From Remote Harnesses

`AppStateStore.ts` comments identify `isUltraplanMode` as set from the
remote-harness side through a `set_permission_mode` control request and pushed
to CCR external metadata by app-state change handling.

`print.ts` handles `set_permission_mode` control requests from stream-json/SDK
hosts. This is one of the runtime surfaces a remote harness can use to influence
Claude Code behavior.

## Control Protocol as Harness API

A stream-json harness can send control requests to:

- initialize hooks, agents, MCP servers, and schemas;
- interrupt or end sessions;
- set permission mode;
- set model;
- set thinking token budget;
- query MCP status;
- query context usage;
- send MCP messages;
- rewind files;
- cancel queued async messages;
- seed read-state cache;
- replace MCP servers;
- reload plugins;
- reconnect/toggle MCP servers;
- enable channels;
- run OAuth/authentication flows;
- stop tasks;
- apply flag settings;
- read settings;
- generate session titles;
- ask side questions;
- enable/disable proactive mode where gated;
- enable remote control.

This is the practical "harness interface" in 2.1.141.

## How a Hermes/OpenClaw-Like Harness Would Be Seen

Given the recovered code, a third-party harness named Hermes or OpenClaw would
be visible only through generic channels unless it patches Claude Code or uses a
recognized environment value.

Expected classification examples:

- Launches `claude -p` without custom env:
  - entrypoint: `sdk-cli`;
  - clientType: `sdk-cli`;
  - no app-specific name.

- Uses Agent SDK without custom env:
  - entrypoint: `sdk-ts` for the TypeScript SDK path;
  - clientType: `sdk-typescript`;
  - optional SDK version if set.

- Uses SDK and sets `CLAUDE_AGENT_SDK_CLIENT_APP=hermes/1.0.0`:
  - entrypoint still `sdk-ts`, `sdk-py`, or `sdk-cli`;
  - user agent includes `client-app/hermes/1.0.0`;
  - API requests include `x-client-app: hermes/1.0.0`.

- Uses SDK and sets `CLAUDE_CODE_ENTRYPOINT=hermes`:
  - raw analytics metadata includes `entrypoint=hermes`;
  - normalized clientType falls back to `cli` unless remote credentials or a
    recognized entrypoint branch also apply;
  - HTTP platform mapping falls back to `claude_code_cli`;
  - feature-flag context can still see `entrypoint=hermes`.

- Runs as a remote CCR/bridge worker:
  - clientType likely `remote`;
  - remote environment/session/container metadata is present;
  - `--sdk-url` and RemoteIO are active.

- Embeds inside Cursor/Windsurf/VS Code/JetBrains terminal:
  - terminal/IDE detection may identify the editor, not the harness.

## What Is Not Present

The recovered source does not show:

- a `Hermes` enum;
- an `OpenClaw` enum;
- a direct check for `HERMES_*` environment variables;
- a direct check for `OPENCLAW_*` environment variables;
- process-name scanning for Hermes/OpenClaw;
- package-name scanning for Hermes/OpenClaw;
- protocol branches named Hermes/OpenClaw;
- custom behavior specific to those names.

## Practical Audit Checklist

To investigate a new harness integration in this source, check these surfaces:

- Does it set `CLAUDE_CODE_ENTRYPOINT`?
- Does it set `CLAUDE_AGENT_SDK_VERSION`?
- Does it set `CLAUDE_AGENT_SDK_CLIENT_APP`?
- Does it use `-p --input-format stream-json --output-format stream-json`?
- Does it use `--sdk-url`?
- Does it set `CLAUDE_CODE_REMOTE` or session-ingress credentials?
- Does it set `CLAUDE_CODE_ENVIRONMENT_KIND=bridge`?
- Does it create IDE lockfiles under the Claude config IDE directory?
- Does it set terminal/editor env vars such as `CURSOR_TRACE_ID`,
  `VSCODE_GIT_ASKPASS_MAIN`, `TERM_PROGRAM`, `__CFBundleIdentifier`, or
  `TERMINAL_EMULATOR`?
- Does it emit `<claude-code-hint />` tags through shell output?
- Does it send SDK control requests after initialize?
- Does it set CI/deployment variables that classify the runtime environment?

## Key Takeaways

- In 2.1.141, "harness detection" is primarily environment/protocol detection.
- `CLAUDE_CODE_ENTRYPOINT` is the central raw label.
- `clientType` is a normalized internal classification, but it only recognizes a
  fixed set of Claude/SDK/remote surfaces.
- `CLAUDE_AGENT_SDK_CLIENT_APP` is the intended generic third-party app label.
- `CLAUDE_AGENT_SDK_VERSION` identifies SDK usage/version.
- `--sdk-url` and stream-json activate the richest external harness protocol.
- Cursor/Windsurf/VS Code/JetBrains are detected as IDE/editor hosts, not as
  general agent harnesses.
- No literal Hermes/OpenClaw detector was found in the reconstructed 2.1.141
  source.
