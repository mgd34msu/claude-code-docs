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
