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
