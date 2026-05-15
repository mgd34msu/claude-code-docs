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
