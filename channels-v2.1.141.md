# MCP Channels in Claude Code 2.1.141

Channels let an MCP server push inbound messages into a Claude Code session.
The server still exposes normal MCP tools for outbound replies, but inbound
messages arrive through the `notifications/claude/channel` notification method
and are wrapped into a `<channel>` tag before being injected into the queue.

## Feature Flags and Runtime Gate

Channel support is compiled behind:

- `feature('KAIROS')`
- `feature('KAIROS_CHANNELS')`

The runtime feature gate is:

- `tengu_harbor`

Allowlist data is read from:

- `tengu_harbor_ledger`

Permission relay support is gated separately by channel permission capability
and related harbor permissions logic.

## CLI Flags

`source/src/main.tsx` defines hidden flags:

- `--channels <servers...>`
- `--dangerously-load-development-channels <servers...>`

Accepted entry formats:

- `plugin:<name>@<marketplace>`
- `server:<name>`

Bad entries produce a hard CLI error. Development channels are interactive-only
and require a confirmation dialog. Accepting development channels marks only
those entries as `dev: true`; it does not bypass org policy globally.

## Session State

Channel entries are stored in bootstrap state as:

- plugin entry: `{ kind: 'plugin', name, marketplace, dev? }`
- server entry: `{ kind: 'server', name, dev? }`

State also tracks whether development channels were used.

## Notification Schema

`source/src/services/mcp/channelNotification.ts` defines the inbound schema:

- method: `notifications/claude/channel`
- params:
  - `content`: string
  - `meta`: optional string map

`wrapChannelMessage()` renders:

```text
<channel source="server" ...safe_meta_attrs>
message
</channel>
```

Metadata keys must match a strict identifier pattern. This prevents a malicious
metadata key from breaking XML attribute structure.

## Gate Order

`gateChannelServer()` applies a defense-in-depth sequence:

- server declares `capabilities.experimental['claude/channel']`.
- API provider is first party.
- `tengu_harbor` runtime gate is enabled.
- team/enterprise orgs have managed setting `channelsEnabled: true`.
- the server was explicitly listed in the session `--channels` entries.
- plugin entries match the installed plugin marketplace.
- non-dev plugin entries appear in the effective allowlist.

For team/enterprise orgs, managed setting `allowedChannelPlugins` replaces the
GrowthBook ledger when set. Unmanaged users fall back to the ledger.

## Permission Relay

Channel permission relay is explicit; a normal text message cannot accidentally
approve a local permission request.

Methods:

- inbound permission response:
  `notifications/claude/channel/permission`
- outbound permission request:
  `notifications/claude/channel/permission_request`

Inbound response fields:

- `request_id`
- `behavior`: `allow` or `deny`

Outbound request fields:

- `request_id`
- `tool_name`
- `description`
- `input_preview`

The MCP server must declare the permission capability and deliberately emit the
permission response method.

## Reconnect and Print Mode

`source/src/cli/print.ts` includes channel-enablement support for IDE-triggered
or non-interactive paths. It re-gates and re-registers channel handlers after
MCP reconnect/toggle events.

## UI Notice

`source/src/components/LogoV2/ChannelsNotice.tsx` warns when channels are:

- disabled by gate.
- missing authentication support.
- blocked by org policy.
- unmatched by session allowlist.

The notice also surfaces prompt-injection risk. Channels are an intentional
inbound prompt surface.

## Telemetry

Observed events:

- `tengu_mcp_channel_flags`
- `tengu_mcp_channel_gate`
- `tengu_mcp_channel_enable`
- `tengu_mcp_channel_message`

The flag event counts entries and plugin identifiers; it avoids logging raw
server names as arbitrary strings.

## 2.1.141 Source Index

- `source/src/main.tsx`
- `source/src/bootstrap/state.ts`
- `source/src/interactiveHelpers.tsx`
- `source/src/services/mcp/channelNotification.ts`
- `source/src/services/mcp/channelAllowlist.ts`
- `source/src/components/LogoV2/ChannelsNotice.tsx`
- `source/src/cli/print.ts`

## CLI Parsing Detail

Channel entries are intentionally tagged. The parser accepts only:

- `plugin:<name>@<marketplace>`
- `server:<name>`

The tag is part of the trust model:

- `plugin:` means "I trust this installed plugin from this marketplace."
- `server:` means "I trust this server name for this session."

Unqualified names are rejected. This prevents a user from accidentally granting
channel-inbound permissions to an ambiguous runtime source.

## Gate Function Detail

`gateChannelServer(serverName, capabilities, pluginSource)` returns either:

- `{ action: 'register' }`
- `{ action: 'skip', kind, reason }`

Skip kinds:

- `capability`
- `disabled`
- `provider`
- `policy`
- `session`
- `marketplace`
- `allowlist`

Gate order:

1. Server declares `experimental['claude/channel']`.
2. API provider is first party.
3. Runtime gate `tengu_harbor` is enabled.
4. Managed team/enterprise org has `channelsEnabled: true`.
5. Session `--channels` includes this server/plugin.
6. Plugin marketplace matches the installed plugin source.
7. Plugin is allowed by org allowlist or GrowthBook ledger unless entry is dev.

The order matters. Normal MCP servers that do not declare the capability exit at
the capability check and do not incur policy/ledger behavior.

## Channel Message Schema

Inbound notification:

```json
{
  "method": "notifications/claude/channel",
  "params": {
    "content": "message text",
    "meta": {
      "thread_id": "optional",
      "user": "optional"
    }
  }
}
```

Rendering:

```xml
<channel source="server" thread_id="...">
message text
</channel>
```

Meta keys must match:

```text
^[a-zA-Z_][a-zA-Z0-9_]*$
```

Values are XML-escaped. Unsafe metadata keys are dropped rather than escaped
because the key becomes an XML attribute name.

## Permission Relay Schema

Inbound permission response:

```json
{
  "method": "notifications/claude/channel/permission",
  "params": {
    "request_id": "abcde",
    "behavior": "allow"
  }
}
```

Outbound permission request to server:

```json
{
  "request_id": "abcde",
  "tool_name": "Bash",
  "description": "Run command",
  "input_preview": "{\"command\":\"...\"}"
}
```

The server must opt into permission relay with the experimental permission
capability. Claude Code does not treat arbitrary channel text as permission
approval.

## Development Channels

Development channels:

- require the dangerous development flag.
- are interactive-only.
- show a confirmation dialog.
- set `dev: true` on accepted entries.
- bypass plugin allowlist for those entries only.
- do not bypass team/enterprise policy.
- do not globally mark all channels as development channels.

This keeps development bypass scoped to exactly the accepted entry.

## Managed Policy and Ledger

For unmanaged users, the allowlist comes from `tengu_harbor_ledger`.

For team/enterprise orgs:

- `channelsEnabled` must be true.
- `allowedChannelPlugins`, when set, replaces the GrowthBook ledger.
- absence of policy is not treated as opt-in.

This prevents an org account from accidentally inheriting unmanaged-user channel
availability.

## Print/SDK Path

`source/src/cli/print.ts` supports channel enablement for non-interactive
protocol paths:

- validate connected plugin-sourced MCP server.
- append a channel entry.
- run the full gate.
- register the handler.
- log enable/message telemetry.
- enqueue the inbound message with channel origin.
- re-register after MCP reconnects or toggles.

## Threat Model

Channels are prompt injection by design. The security model is not "make inbound
messages safe"; it is "only trusted, explicitly enabled sources can inject."

Defenses:

- capability opt-in.
- first-party provider requirement.
- managed org opt-in.
- explicit session flag.
- plugin marketplace verification.
- allowlist/ledger.
- XML attribute key filtering.
- structured permission relay instead of text approval.

Non-defenses:

- channel content is not semantically sanitized.
- model still sees the message as context.
- outbound reply tools are controlled by the model and permission system.
- a trusted channel can still relay untrusted humans.

## Telemetry Detail

- `tengu_mcp_channel_flags`: session flag summary. Counts entries and records
  plugin identifiers while avoiding raw server-name logging.
- `tengu_mcp_channel_gate`: per-server gate outcomes.
- `tengu_mcp_channel_enable`: enablement in print/SDK channel path.
- `tengu_mcp_channel_message`: inbound channel message receipt.

## Full Channel Lifecycle

The 2.1.141 channel system is a side-channel transport for MCP-integrated
clients. It is not the same thing as ordinary MCP stdio/http tool calls. The
high-level lifecycle is:

1. CLI parses channel flags.
2. feature gates and dynamic config decide whether channels are allowed.
3. session state stores the enabled/disabled channel configuration.
4. the MCP client path creates channel-capable connection state.
5. tool notifications and permission requests can be routed across the channel.
6. permission results return through relay schemas.
7. reconnect and cleanup logic preserve or clear channel state.
8. print/SDK mode serializes compatible events when requested.

This makes channels both a transport feature and a permission feature.

## CLI Flag Semantics

Channel flags are parsed before the main interactive session is fully running.
Important behaviors:

- channel flags can be present without channels becoming enabled if gates fail.
- debug/development channels are separate from normal production channel state.
- channel configuration is stored in session/app state rather than being a
  one-off local variable.
- telemetry records the flag shape and whether it activated.
- invalid combinations fail early or fall back to disabled state.

The docs should therefore describe both the flags and the gate results. A flag
being accepted by the CLI parser does not prove the runtime channel is active.

## Gate Inputs

The source-level gate inputs include:

- build-time feature flags.
- GrowthBook/dynamic config.
- managed policy.
- session type.
- MCP server/client capabilities.
- remote/print/SDK mode constraints.
- development override flags.
- channel-specific CLI options.

The gate is not a single boolean. Reconstructing it requires reading CLI
parsing, MCP client setup, and permission relay code.

## Message Families

The channel schema covers more than one kind of message:

- generic channel notifications.
- tool-use lifecycle messages.
- permission request messages.
- permission result messages.
- connection/reconnect messages.
- development/testing messages.
- stream-json compatible lifecycle records.

For compatibility, channel messages need stable discriminators and safe payload
shapes. Tool-use payloads reference tool names and IDs. Permission payloads
reference the rule/tool decision context. They should not be treated as
free-form model messages.

## Permission Relay Flow

Permission relay is the highest-risk channel path:

1. a tool call needs permission.
2. local permission code creates a request object.
3. channel transport emits it to the connected client.
4. client displays or processes the request.
5. client returns an allow/deny/ask-style result.
6. local runtime validates the result.
7. execution resumes or fails based on the validated decision.

The local runtime remains the enforcement point. A remote channel can carry a
decision, but the local code still validates the shape and applies the local
permission model.

## Print and SDK Compatibility

Print/SDK mode changes how channel lifecycle data is surfaced:

- `--output-format=stream-json` can include lifecycle records.
- `--include-hook-events` controls hook lifecycle event inclusion.
- remote mode can force inclusion of some hook/channel-style events.
- SDK schemas include fields for prompt suggestions, plugin install progress,
  and related structured events.

The important point is that channel events may be serialized for automation
without implying a human TUI is present.

## Threat Model Extension

Channels can affect tool execution and permission decisions, so the threat model
must include:

- forged permission responses.
- stale responses replayed after reconnect.
- mismatched tool-use IDs.
- server/client capability confusion.
- debug channel accidentally enabled in production.
- private network or remote transport assumptions.
- policy bypass through remote UI surfaces.

2.1.141 mitigates this through gates, schemas, managed policy, local validation,
and explicit session state. Documentation should not present channels as a
simple notification feature.

## Reconstruction Checklist

For a later release:

1. Inspect CLI options around channel flags.
2. Search source for channel enablement gates and dynamic configs.
3. Read MCP client connection setup.
4. Read permission relay schemas.
5. Read stream-json/SDK output schemas for channel-compatible records.
6. Read remote/CCR transport code for channel interaction.
7. Verify managed policy handling.
8. Verify debug/development channel paths are not documented as normal user
   features.
9. Compare telemetry event names and metadata.
10. Test disabled, enabled, reconnect, and permission-denied cases separately.
