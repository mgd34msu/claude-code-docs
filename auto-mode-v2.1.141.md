# Auto Mode in Claude Code 2.1.141

Auto mode is an internal permission mode that uses static permission rules plus
a classifier-driven decision layer. In 2.1.141 it is still intentionally
externalized as `default` in places where the public permission-mode surface
should not expose it.

## Permission Mode Model

Permission types are defined in `source/src/types/permissions.ts`.

External modes:

- `default`
- `acceptEdits`
- `bypassPermissions`
- `dontAsk`
- `plan`

Internal mode:

- `auto`

There is also a `bubble` type in internal permission handling, but it is not a
normal user-selected mode.

The mode config in `source/src/utils/permissions/PermissionMode.ts` gives auto
mode the title `Auto mode`, but `toExternalPermissionMode(auto)` returns
`default`. That keeps internal auto-mode state out of surfaces that expect only
public permission modes.

## Activation Paths

Observed activation/configuration surfaces:

- Hidden deprecated CLI flag: `--enable-auto-mode`.
- `--permission-mode auto` when the build exposes the internal mode.
- Settings default mode via `permissions.defaultMode`.
- Auto-mode subcommands behind `feature('TRANSCRIPT_CLASSIFIER')`:
  `auto-mode defaults`, `auto-mode config`, and `auto-mode critique`.

The startup path is in `source/src/utils/permissions/permissionSetup.ts`.
`initialPermissionModeFromCLI()` applies precedence roughly as:

- `--dangerously-skip-permissions`
- explicit `--permission-mode`
- settings `permissions.defaultMode`
- fallback default

Bypass mode can be disabled by managed settings or by the runtime gate
`tengu_disable_bypass_permissions_mode`.

## Gate Access

Auto mode is gated by dynamic config, settings, model support, and cached
circuit-breaker state.

Key source areas:

- `verifyAutoModeGateAccess()`
- `getAutoModeUnavailableNotification()`
- `isAutoModeDisabledBySettings()`
- `getAutoModeEnabledState...`
- `source/src/state/autoModeState.ts`
- `source/src/components/AutoModeOptInDialog.tsx`

The runtime config key is `tengu_auto_mode_config`. The config can place auto
mode into enabled/disabled/opt-in states, can disable fast mode behavior, and can
override allowed models through beta handling.

Unavailable reasons observed in source include:

- settings disabled auto mode.
- circuit breaker disabled auto mode.
- current model not supported.

## Dangerous Rule Stripping

2.1.141 strips rules that would make auto mode unsafe before the classifier can
reason about the request. The stripping and restore logic lives in
`source/src/utils/permissions/permissionSetup.ts`.

Dangerous examples:

- Bash allow-all rules.
- Bash wildcard or broad interpreter-prefix rules.
- PowerShell allow-all rules.
- PowerShell rules for broad execution primitives such as `iex`, invoke/start
  process patterns, and similar shell escapes.
- Any `Agent` allow rule, because it could approve subagent creation before the
  classifier sees the spawn request.
- Anthropic-only `Tmux` rules.

`stripDangerousPermissionsForAutoMode()` removes unsafe rules and stashes them
in `strippedDangerousRules`; `restoreDangerousPermissions()` puts them back when
leaving auto mode.

## Transition State Machine

`transitionPermissionMode()` handles entering and leaving plan, auto, and other
permission modes.

Important behavior:

- Entering auto mode can strip dangerous permissions.
- Leaving auto mode restores stripped rules.
- Plan mode interaction is explicit; plan-mode transitions can create follow-up
  attachment/reminder state.
- Auto mode tracks active state separately in `autoModeState.ts`.

State variables include:

- `autoModeActive`
- `autoModeFlagCli`
- `autoModeCircuitBroken`
- `needsAutoModeExitAttachment`

## Static Rules Plus Classifier

Auto mode still uses the regular permission rule machinery. The difference is
that unresolved or risky decisions can be routed through classifier-aware logic.
This keeps normal rule matching fast while allowing model/classifier judgment for
requests that are not clean static matches.

Important related files:

- `source/src/hooks/useCanUseTool.tsx`
- `source/src/utils/permissions/permissions.ts`
- `source/src/utils/permissions/permissionRuleParser.ts`
- `source/src/types/permissions.ts`

Legacy tool aliases are normalized before permission rules are evaluated:

- `Task` becomes `Agent`.
- `KillShell` becomes `TaskStop`.
- `AgentOutputTool` and `BashOutputTool` become `TaskOutput`.
- `Brief` becomes `SendUserMessage` when brief/Kairos support is present.

## Admin and Policy Controls

2.1.141 includes settings and managed-policy paths that can disable or constrain
auto and bypass modes.

Relevant controls:

- `permissions.defaultMode`
- auto-mode settings/defaults commands.
- `disableAutoMode`-style settings.
- managed settings that disable bypass.
- runtime gates for auto-mode config and bypass disablement.

Remote sessions are stricter: the remote path supports only selected external
permission modes and does not expose all local interactive choices.

## Telemetry

Observed auto-mode and related events include:

- `tengu_auto_mode_opt_in_dialog_shown`
- `tengu_auto_mode_opt_in_dialog_accept`
- `tengu_auto_mode_opt_in_dialog_accept_default`
- `tengu_auto_mode_opt_in_dialog_decline`
- `tengu_auto_mode_subsequent_approval`
- auto-mode decision metadata attached to permission decisions.
- `tengu_migrate_reset_auto_opt_in_for_default_offer`

General permission/tool decisions also flow through:

- `tengu_tool_use_can_use_tool_allowed`
- `tengu_tool_use_can_use_tool_rejected`
- `tengu_internal_tool_permission_decision`

## 2.1.141 Source Index

- `source/src/types/permissions.ts`
- `source/src/utils/permissions/PermissionMode.ts`
- `source/src/utils/permissions/permissionSetup.ts`
- `source/src/state/autoModeState.ts`
- `source/src/cli/handlers/autoMode.ts`
- `source/src/components/AutoModeOptInDialog.tsx`
- `source/src/hooks/useCanUseTool.tsx`
- `source/src/utils/betas.ts`
- `source/src/main.tsx`

## Detailed Permission Mode Table

External permission modes:

- `default`: normal ask/allow/deny behavior.
- `acceptEdits`: auto-accepts edit-class operations while preserving other
  permission checks.
- `bypassPermissions`: dangerous mode intended to skip permission prompts where
  allowed by policy and gate state.
- `dontAsk`: non-interactive/no-prompt style behavior.
- `plan`: planning mode; execution requires leaving plan mode.

Internal permission modes:

- `auto`: classifier/rule-assisted mode.
- `bubble`: internal propagation state, not a user-facing mode.

`auto` is intentionally converted back to `default` by externalization helpers.
This matters for SDKs, settings display, and remote paths that should not expose
internal state.

## Startup Decision Order

`initialPermissionModeFromCLI()` gives highest priority to explicit dangerous
or CLI-provided modes:

1. `--dangerously-skip-permissions`
2. explicit `--permission-mode`
3. settings `permissions.defaultMode`
4. default fallback

Then additional policy and gate checks can modify the result:

- bypass may be unavailable.
- auto may be unavailable.
- remote sessions narrow the accepted mode set.
- model support can block auto mode.
- circuit-breaker state can block auto mode.

## Auto Mode Gate Context

Auto availability is not just `tengu_auto_mode_config`.

Inputs include:

- runtime dynamic config `tengu_auto_mode_config`.
- model support and beta/model allowlist.
- user settings.
- managed settings.
- cached circuit-breaker state.
- opt-in dialog state.
- fast-mode settings in the dynamic config.

`verifyAutoModeGateAccess()` returns both context update behavior and user-facing
notification state. This is why auto mode can be "configured" but still not
actually enterable for a given session/model/account.

## Auto Mode Commands

The hidden `auto-mode` command family is behind
`feature('TRANSCRIPT_CLASSIFIER')`.

Subcommands:

- `auto-mode defaults`: prints default external auto-mode rules.
- `auto-mode config`: prints effective auto-mode config, including custom
  sections with replacement semantics.
- `auto-mode critique`: runs a side query that critiques custom rules.

These commands are diagnostic/configuration surfaces, not the permission engine
itself.

## Dangerous Rule Detection

Auto mode strips rules that could approve dangerous operations before the
classifier sees enough context.

Bash danger patterns:

- blanket allow for `Bash`.
- wildcard allow.
- broad shell/interpreter prefix rules.
- rules that effectively delegate arbitrary execution.

PowerShell danger patterns:

- blanket allow.
- broad invocation patterns.
- `iex` / invoke-expression style behavior.
- process launch and command execution primitives.

Agent danger:

- any `Agent` allow rule is dangerous in auto mode because it can approve
  subagent creation before the classifier evaluates the delegation.

Internal danger:

- Anthropic-only Tmux rules are treated as dangerous.

The stripped rules are stored, not discarded. Leaving auto mode restores them.

## Transition Semantics

`transitionPermissionMode()` is the main state-machine function.

Entering auto mode:

- verifies gate/model/settings state.
- strips dangerous rules.
- sets auto active state.
- may attach auto-mode exit reminder/attachment state.

Leaving auto mode:

- restores dangerous rules.
- clears active state.
- updates notification/reminder state.

Plan interactions:

- plan mode can override execution.
- `ExitPlanMode` is the canonical route out.
- auto mode must not convert plan approval into silent execution.

## Permission Decision Flow

Auto mode still starts with the normal permission context:

1. Normalize tool names.
2. Check explicit deny rules.
3. Check explicit allow rules.
4. Apply base tools.
5. Evaluate mode-specific behavior.
6. Run hooks such as `PermissionRequest` where applicable.
7. Use classifier/rule outcome if auto mode needs a learned decision.
8. Track denials/subsequent approvals.

This layering is why rule stripping is necessary: an overbroad explicit allow
would otherwise bypass the classifier layer entirely.

## Settings and Policy Controls

Relevant settings/policy controls:

- `permissions.defaultMode`
- `permissions.allow`
- `permissions.deny`
- `permissions.additionalDirectories`
- managed settings that disable auto mode.
- managed settings that disable bypass mode.
- local auto-mode defaults/custom config.
- auto-mode opt-in state.

Bypass and auto mode should be treated separately. A setting that disables one
does not automatically imply identical behavior for the other.

## Circuit Breaker

Auto mode has circuit-breaker state cached outside the live permission context.
If the backend/config marks auto unavailable, startup can show an unavailable
notification and fall back to `default`. This prevents repeatedly entering an
unsafe or unsupported mode after prior failures.

## Telemetry Detail

Auto-mode-specific events:

- `tengu_auto_mode_opt_in_dialog_shown`
- `tengu_auto_mode_opt_in_dialog_accept`
- `tengu_auto_mode_opt_in_dialog_accept_default`
- `tengu_auto_mode_opt_in_dialog_decline`
- `tengu_auto_mode_subsequent_approval`
- `tengu_migrate_reset_auto_opt_in_for_default_offer`

Related permission events:

- `tengu_internal_tool_permission_decision`
- `tengu_tool_use_can_use_tool_allowed`
- `tengu_tool_use_can_use_tool_rejected`

Related migration/config events:

- `tengu_migrate_bypass_permissions_accepted`
- managed settings load events.

## Edge Cases

- `auto` may appear in internal mode lists but externalize to `default`.
- An explicit CLI mode can still be rejected later by policy/gates.
- A dangerous allow rule can be present in settings but temporarily absent while
  auto mode is active.
- Legacy tool names are normalized before dangerous-rule checks.
- Remote sessions do not expose the full local interactive mode set.

## Source-Level Decision Algorithm

The 2.1.141 auto-mode path is best understood as a startup and permission
decision algorithm:

1. Parse CLI flags and settings.
2. Resolve managed settings and policy.
3. Resolve workspace trust and environment constraints.
4. Determine whether auto mode is available to this user/session.
5. Determine whether the user has explicitly opted in.
6. Select the effective permission mode.
7. Strip or reject dangerous inherited rules that would make auto mode unsafe.
8. During each tool call, run static permission rules.
9. For shell commands, run command parsing/read-only checks.
10. When static analysis is insufficient, invoke the classifier.
11. Apply ask/allow/deny behavior based on the result.
12. Emit telemetry for mode decisions and classifier usage.

The user-visible toggle is only one step in this chain. Availability, policy,
and command-level risk classification can still override the outcome.

## Permission Modes Compared

`default`:

- normal interactive permission behavior.
- ask/allow/deny rules are honored.
- classifier may still run for shell risk decisions.
- no broad automatic approval is implied.

`acceptEdits`:

- narrower than auto mode.
- optimized around file edit approval behavior.
- still does not mean arbitrary shell/network operations are automatically
  approved.

`auto`:

- attempts to reduce prompts for low-risk operations.
- relies on static rules and classifier decisions.
- strips dangerous inherited allow rules.
- remains constrained by managed policy and hard deny logic.

`plan`:

- separates planning from execution.
- plan exit/approval tools become important.
- auto mode does not erase plan-mode semantics.

`bypassPermissions` or dangerously-skip-permissions:

- broader and more dangerous than auto mode.
- intentionally named and gated differently.
- should not be documented as "auto mode".

## Dangerous Rule Stripping Detail

The auto-mode source treats some allow rules as incompatible with safe automatic
operation. The dangerous-rule logic is not just a UI warning. It prevents a
stored or inherited rule set from silently converting auto mode into broad
execution bypass.

Rules are dangerous when they create one of these effects:

- allow all shell behavior.
- allow command prefixes that can trivially wrap arbitrary payloads.
- allow tools that can mutate or exfiltrate outside expected boundaries.
- bypass the classifier's ability to understand the actual command.
- rely on shell aliases/functions/extension points to hide the real operation.

The classifier prompt text also calls out safety-check bypass patterns: using
flags, configs, aliases, or extension points to launch a different command than
the permission check appears to authorize.

## Shell Classifier Relationship

Auto mode depends heavily on shell analysis because `Bash` is a broad tool.
2.1.141 uses multiple layers:

- static shell prefix extraction.
- read-only validation for known safe commands.
- wildcard/prefix matching for permission rules.
- yolo classifier prompts for ambiguous commands.
- hard-deny rules for bypass, secrets, network, destructive, or unsafe
  behavior.
- debug info explaining which rule or classifier decision applied.

The classifier is not a replacement for hard denies. It is the fallback for
cases where static analysis cannot confidently classify the command.

## Managed Policy Effects

Managed settings can affect auto mode in several ways:

- disable auto-mode availability.
- force or remove permission-mode choices.
- disable settings writes that would otherwise persist opt-in.
- restrict hooks/plugins/tools in the same session.
- make UI toggles show managed state rather than editable state.

When reconstructing behavior, check both the mode code and the settings loader.
An auto-mode toggle can exist in UI while still being blocked by managed policy.

## Migration Effects

2.1.141 includes migrations that can reset or adjust prior mode selections. This
matters because docs based only on fresh installs miss upgrade behavior. The
auto-mode migration path can reset opt-in for specific default-offer cases and
emit migration telemetry.

For release-to-release audits:

- inspect migrations touching permission mode or auto opt-in.
- inspect default settings schema.
- inspect managed setting precedence.
- inspect prompt text shown to upgraded users.
- inspect telemetry added to migration files.

## User-Facing Troubleshooting

If auto mode does not behave as expected:

- check whether the session is in `auto`, `default`, `acceptEdits`, `plan`, or
  bypass mode.
- check managed settings and enterprise policy.
- check whether `--bare` or `CLAUDE_CODE_SIMPLE` removed background systems.
- check whether a dangerous allow rule was stripped.
- check whether the command is a shell command that requires classifier review.
- check whether shell aliases/functions make the command less analyzable.
- check whether the operation is blocked by hard-deny policy.
- check telemetry/debug logs for classifier failure versus classifier denial.

## Future Diff Checklist

For later versions, review:

1. CLI flags that set or permit permission modes.
2. settings schema for permission mode and opt-in fields.
3. migrations touching auto mode.
4. dangerous-rule stripping code.
5. shell static-prefix and read-only validators.
6. yolo classifier prompts and hard-deny text.
7. permission UI components.
8. managed settings enforcement.
9. telemetry events around auto-mode toggles and classifier decisions.
10. tests for alias/function/safety-check bypass behavior.
