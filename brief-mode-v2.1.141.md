# Brief Mode and SendUserMessage in Claude Code 2.1.141

Brief mode is the Kairos/user-message path. In normal mode Claude can emit
assistant text directly; in brief mode user-visible output is routed through the
`SendUserMessage` tool so non-terminal or remote surfaces can receive concise
messages with status and optional attachments.

## Tool Identity

The tool lives in `source/src/tools/BriefTool/BriefTool.ts`.

- Primary name: `SendUserMessage`
- Legacy alias: `Brief`
- Prompt: `source/src/tools/BriefTool/prompt.ts`
- Registered from: `source/src/tools.ts`

The legacy alias remains available through the tool alias system and permission
normalization.

## Input and Output

Input fields:

- `message`: user-facing text.
- `attachments`: optional array of paths or pre-uploaded attachment objects.
- `status`: `normal` or `proactive`.

Pre-uploaded attachment objects include:

- `file_uuid`
- `file_name`
- `size`
- `is_image`

Output includes:

- `message`
- `attachments`
- `sentAt`

The tool is marked read-only and concurrency safe.

## Entitlement and Enablement

Brief support is behind `feature('KAIROS') || feature('KAIROS_BRIEF')`.

`isBriefEntitled()` returns true when one of these applies:

- Kairos is active.
- `CLAUDE_CODE_BRIEF` is set.
- runtime gate `tengu_kairos_brief` is true.

`isBriefEnabled()` then requires user/session opt-in in addition to entitlement:

- Kairos active; or
- the user message opt-in state is enabled.

The `CLAUDE_CODE_BRIEF` environment variable is stronger than a normal runtime
gate because it force-grants entitlement for the local session and is also used
by startup activation.

## Activation Paths

2.1.141 supports several activation paths:

- `--brief` when Kairos/brief support is compiled in.
- `CLAUDE_CODE_BRIEF`.
- `/brief` slash command when the slash command is enabled by
  `tengu_kairos_brief_config.enable_slash_command`.
- keybinding/app action `app:toggleBrief`.
- SDK/tool listing paths that include `SendUserMessage` or legacy `Brief`.
- chat/default-view paths that activate user-message mode.

The slash command can turn brief mode off even if entitlement later disappears;
this lets a user escape a mode that is already active.

## Rendering Behavior

Rendering behavior is split between the tool and message renderer:

- In normal transcript rendering, turns with `SendUserMessage` suppress redundant
  plain text where appropriate.
- In brief-only rendering, visible output is filtered toward `SendUserMessage`
  blocks.
- The prompt tells the model that text outside `SendUserMessage` may only be
  visible in detailed views, not necessarily to the human user.

Key file:

- `source/src/components/Messages.tsx`

## Attachments

Brief mode supports attachments passed by local path or by pre-uploaded UUID
metadata. The upload path validates size/type, derives MIME information, and
returns attachment records for the tool result.

Important controls:

- `CLAUDE_CODE_BRIEF_UPLOAD`
- attachment disable paths such as `CLAUDE_CODE_DISABLE_ATTACHMENTS`
- normal API/provider attachment restrictions.

## Proactive Status

The `status` field distinguishes normal user messages from proactive messages.
The prompt for `SendUserMessage` includes a proactive section so the model can
route check-ins, status pings, or remote-session notifications without treating
them as ordinary assistant chat text.

## Telemetry

Observed events:

- `tengu_brief_mode_toggled`
- `tengu_brief_mode_enabled`
- `tengu_brief_send`

`tengu_brief_send` includes send metadata, attachment counts, and status class,
not arbitrary unredacted tool content.

## 2.1.141 Source Index

- `source/src/tools/BriefTool/BriefTool.ts`
- `source/src/tools/BriefTool/prompt.ts`
- `source/src/commands/brief.ts`
- `source/src/main.tsx`
- `source/src/components/Messages.tsx`
- `source/src/tools.ts`
- `source/src/utils/permissions/permissionRuleParser.ts`

## Detailed Tool Schema

`SendUserMessage` input:

- `message`: string. The user-facing message.
- `attachments`: optional. Either local file paths or already uploaded attachment
  descriptors.
- `status`: `normal` or `proactive`.

Uploaded attachment descriptor:

- `file_uuid`
- `file_name`
- `size`
- `is_image`

Output:

- `message`
- `attachments`
- `sentAt`

Tool metadata:

- read-only.
- concurrency safe.
- legacy alias `Brief`.
- logs sends through `tengu_brief_send`.
- hidden unless entitlement/enablement allows it.

## Entitlement vs Enablement

The code separates entitlement from active mode:

- Entitlement answers whether this session/account can use brief.
- Enablement answers whether the session is currently in a mode that should use
  `SendUserMessage`.

Entitlement inputs:

- build support: `feature('KAIROS') || feature('KAIROS_BRIEF')`.
- active Kairos state.
- `CLAUDE_CODE_BRIEF`.
- runtime gate `tengu_kairos_brief`.

Enablement inputs:

- Kairos active.
- user-message opt-in.
- CLI/env activation.
- tool-list activation in SDK mode.
- slash command/keybinding state.

## Activation Details

Activation routes:

- `--brief` CLI flag.
- `CLAUDE_CODE_BRIEF`.
- `/brief` slash command.
- `app:toggleBrief` keybinding.
- SDK/tool lists that request `SendUserMessage` or `Brief`.
- chat/default-view state in Kairos flows.

The slash command is itself gated by `tengu_kairos_brief_config` and may be
hidden even when the underlying tool exists.

## Prompt Contract

The prompt text tells the model:

- user-facing messages must go through `SendUserMessage`.
- text outside the tool can be hidden in non-terminal surfaces.
- proactive messages should use `status: proactive`.
- attachments should be used only when needed.

This is not just UI preference. In remote/non-terminal contexts, direct
assistant text may not reach the human in the intended way.

## Attachment Pipeline

Attachment handling includes:

- accepting local paths or pre-uploaded descriptors.
- validating files.
- deriving MIME/image state.
- upload timeout/size behavior.
- returning UUID-backed descriptors to the tool result.

Operational controls:

- `CLAUDE_CODE_BRIEF_UPLOAD`
- `CLAUDE_CODE_DISABLE_ATTACHMENTS`
- provider/API attachment limitations.

## Rendering Semantics

`components/Messages.tsx` has special handling:

- brief-only mode filters visible transcript toward `SendUserMessage`.
- normal mode can suppress redundant assistant text if a turn already contains
  a `SendUserMessage`.
- detailed views may show more internal assistant text than the user-facing
  brief channel.

This means brief mode changes both model instructions and rendering policy.

## State Transitions

Brief toggling updates:

- `isBriefOnly` or related app state.
- user-message opt-in.
- system reminders.
- render filtering behavior.

The `/brief` command can turn brief off even if entitlement has disappeared,
which prevents trapping a user in brief-only rendering.

## Telemetry Detail

Events:

- `tengu_brief_mode_toggled`
- `tengu_brief_mode_enabled`
- `tengu_brief_send`

Metadata includes mode/status/attachment counts and safe categorical fields.
The message body itself is not a telemetry event-name payload.

## Compatibility Notes

- `Brief` remains a tool alias for `SendUserMessage`.
- Permission parser can normalize legacy `Brief` to `SendUserMessage`.
- New 2.1.141 configs should prefer `SendUserMessage`.
- SDK/tool-list paths may use either name for compatibility.

## Source-Level Activation Matrix

Brief mode can be activated through several surfaces:

- `--brief` CLI option when present in the build.
- `CLAUDE_CODE_BRIEF=1`.
- `/brief` slash command when the dynamic config enables the command.
- default chat view behavior under KAIROS/brief gates.
- assistant/KAIROS activation paths that force or imply brief behavior.

The code distinguishes entitlement from explicit opt-in:

- `feature('KAIROS')` or `feature('KAIROS_BRIEF')` controls compiled code.
- `getKairosActive()` can imply assistant-mode behavior.
- `tengu_kairos_brief` controls brief availability for non-force paths.
- `CLAUDE_CODE_BRIEF=1` is an explicit dev/testing/user override path.
- `/brief` can turn off user-message opt-in even when entitlement later
  disappears.

This prevents users from being trapped in a rendering mode they can no longer
toggle through the ordinary enable path.

## Tool Contract

`SendUserMessage` is a model-callable communication tool. It is not a hidden
UI-only message. Its input carries:

- `message`: the text to send to the user.
- `attachments`: optional structured attachments.
- `status`: optional status/progress information.

The output is intentionally small. The tool's purpose is to create a user-facing
message boundary, not to return large data to the model.

## Rendering Contract

Brief mode changes rendering in `components/Messages.tsx` and related message
components:

- assistant prose can be suppressed when a turn includes `SendUserMessage`.
- user-facing output prioritizes the brief tool message.
- transcript/detail views can still expose more internal assistant content.
- Agent View and transcript mode can change whether brief layout is used.
- spinner/status UI reads brief state to avoid duplicate or misleading output.

The model prompt and the renderer must agree. If the prompt tells the model to
use `SendUserMessage` but rendering does not filter ordinary assistant text, the
user sees duplicates. If rendering filters without the prompt contract, the user
may see nothing useful.

## Attachment Semantics

Brief attachments are not generic file attachments. They are structured payloads
that the brief tool and message renderer know how to display. The pipeline must:

- validate attachment shape.
- preserve ordering.
- avoid leaking raw internal tool state.
- count attachments for telemetry.
- handle missing/unsupported attachment types gracefully.

For future versions, audit attachment schema changes separately from the text
message field.

## State Machine

The practical state machine is:

1. build includes brief/KAIROS code.
2. startup evaluates gates, env, settings, and default view.
3. brief-only/user-message opt-in state is set in app state.
4. system reminder/prompt instructions are adjusted.
5. model receives `SendUserMessage` tool when enabled.
6. model calls the tool.
7. renderer filters or prioritizes brief output.
8. `/brief` or config can toggle state.
9. telemetry records enabled/toggled/send events.

Each step has an independent failure mode. For example, a tool can be present
while rendering is not in brief-only mode, or brief-only state can exist without
the slash command being enabled.

## SDK and Noninteractive Behavior

Brief mode is most visible in the TUI, but it also matters for SDK/print paths:

- tool schemas can include the canonical or compatibility name.
- stream-json output may include tool-use lifecycle events.
- noninteractive sessions do not run normal TUI rendering.
- the model prompt can still instruct brief-style user messages.

When documenting automation behavior, avoid saying brief mode is "only UI".
The rendering is UI-specific, but the tool call and prompt contract are model
and API behavior.

## Future Diff Checklist

For a later release:

1. Inspect `BriefTool/BriefTool.ts`.
2. Inspect `BriefTool/prompt.ts`.
3. Inspect `/brief` command implementation.
4. Inspect `main.tsx` brief startup activation.
5. Inspect `components/Messages.tsx` and user/tool message renderers.
6. Inspect `PromptInput`/spinner/status components for brief state checks.
7. Inspect permission parser alias handling.
8. Inspect telemetry event names and metadata fields.
9. Inspect SDK schemas for tool-name compatibility.
10. Validate that `Brief` legacy naming remains compatibility-only.

## Deep 2.1.141 Brief Reconstruction

Brief mode is implemented as both a tool and a display policy. The tool is
`SendUserMessage`; the mode determines whether model-facing visible output is
expected to go through that tool.

### Source Map

| Concern | Source |
| --- | --- |
| Tool schema and runtime gate | `source/src/tools/BriefTool/BriefTool.ts` |
| Tool prompt text | `source/src/tools/BriefTool/prompt.ts` |
| Attachment resolution | `source/src/tools/BriefTool/attachments.ts` |
| Bridge upload | `source/src/tools/BriefTool/upload.ts` |
| Slash command | `source/src/commands/brief.ts` |
| CLI activation | `source/src/main.tsx` |
| Registry placement | `source/src/tools.ts` |
| Message filtering/rendering | `source/src/components/Messages.tsx`, `source/src/utils/messages.ts` |
| Spinner/status layout | `source/src/components/Spinner.tsx`, `source/src/components/PromptInput/PromptInput.tsx` |
| Settings UI | `source/src/components/Settings/Config.tsx` |

### Tool Identity

| Field | Value |
| --- | --- |
| Canonical name | `SendUserMessage` |
| Legacy alias | `Brief` |
| Description | Send a message to the user. |
| Read-only | Yes. |
| Concurrency safe | Yes. |
| Max result size | `100000` chars. |
| Auto-classifier projection | The `message` field. |

New docs and configs should use `SendUserMessage`. The alias is compatibility
only and is mostly useful for old transcripts or SDK/tool lists that still ask
for `Brief`.

### Input And Output Schemas

Input fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `message` | string | Markdown-capable user-visible message. |
| `attachments` | optional array | Either local file path strings or uploaded attachment objects. |
| `status` | `normal` or `proactive` | Indicates whether this is a direct reply or unsolicited/proactive update. |

Uploaded attachment objects have `file_uuid`, `file_name`, `size`, and
`is_image`.

Output fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `message` | string | Delivered message. |
| `attachments` | optional array | Resolved attachment metadata, including optional `file_uuid`. |
| `sentAt` | optional ISO string | Captured at tool execution time. Optional for old resumed sessions. |

The output keeps `attachments` and `sentAt` optional to preserve replay
compatibility with transcripts created before those fields existed.

### Entitlement Vs Activation

2.1.141 splits access into two questions:

| Question | Function | Required conditions |
| --- | --- | --- |
| Is the user allowed to use Brief? | `isBriefEntitled()` | `feature("KAIROS")` or `feature("KAIROS_BRIEF")`, plus Kairos active, env override, or GrowthBook `tengu_kairos_brief`. |
| Is the tool active now? | `isBriefEnabled()` | Entitlement plus Kairos active or `userMsgOptIn`. |

The env var `CLAUDE_CODE_BRIEF` is a development/testing override that grants
entitlement and activation through startup handling. It should not be described
as just a display toggle; it participates in tool availability.

### Activation Paths

| Path | Source behavior |
| --- | --- |
| `--brief` | Calls `maybeActivateBrief()`, checks entitlement, sets `userMsgOptIn`, logs source `flag`. |
| `CLAUDE_CODE_BRIEF` | Same startup path, logs source `env`. |
| Settings `defaultView: "chat"` | In interactive sessions, entitled users get `userMsgOptIn` so chat view can use the tool. |
| `--tools SendUserMessage` or `--tools Brief` | SDK/tool list opt-in when entitled. |
| `/brief` | Toggles `isBriefOnly` and `userMsgOptIn`, but command visibility is separately controlled by `tengu_kairos_brief_config.enable_slash_command`. |
| Assistant/Kairos mode | Bypasses user opt-in because assistant mode depends on the tool. |

`/brief` can always turn off if currently on, even if entitlement later flips
off. That prevents users from becoming stuck in a gated mode.

### Rendering Policy

Brief mode changes what the user is expected to read:

| Area | Behavior |
| --- | --- |
| Assistant plain text | Hidden or de-emphasized in brief-only paths so the tool message is primary. |
| Trailing assistant text after `SendUserMessage` | Filtered to avoid duplicated visible output. |
| Spinner | Uses `BriefSpinner` and `BriefIdleStatus` with stable two-row footprint. |
| Prompt input gap | PromptInput removes its normal gap when brief owns the layout. |
| Todo/task reminder nags | Suppressed when `SendUserMessage` is available, because it becomes the primary communication channel. |
| Conversation recovery | Treats a turn ending in `SendUserMessage` as complete, avoiding phantom interrupted-turn recovery. |

The display behavior is why a model must put the actual answer in the tool
call. A plain text "done" plus detailed answer outside the tool is a bad brief
turn because the user may only see the tool payload.

### Attachments And Uploads

Attachment handling has two paths:

| Attachment kind | Runtime behavior |
| --- | --- |
| Local path string | Validated, stat/read checks run, then resolved to metadata. |
| Uploaded object | Passed through without local stat or upload; intended for device tools. |
| Bridge upload | If bridge mode is active and upload succeeds, a private API `file_uuid` is attached. |

Upload constraints:

| Constraint | Value |
| --- | --- |
| Max upload size | 30 MB. |
| Timeout | 30 seconds. |
| Image MIME whitelist | png, jpg, jpeg, gif, webp. |
| Non-image MIME | `application/octet-stream`. |
| Auth | Bridge/OAuth token required. |
| Failure behavior | Best effort; local attachment metadata remains useful if upload fails. |

`CLAUDE_CODE_BRIEF_UPLOAD` is part of the attachment behavior surface and
should be treated as a brief-specific environment variable, not a generic file
upload control.

### Telemetry

| Event | Meaning |
| --- | --- |
| `tengu_brief_mode_enabled` | Startup activation through flag or env. |
| `tengu_brief_mode_toggled` | Slash command or keybinding toggle, including gated failures. |
| `tengu_brief_send` | Tool call delivered, with proactive flag and attachment count. |

The important metadata split is activation source versus message send. A session
can be entitled but inactive, active without a slash command, or active through
Kairos; those should not be collapsed into one "brief enabled" concept.
