# Claude Code CLI - Telemetry Reference

> Info as of Claude Code CLI v2.1.34

## Overview

Claude Code CLI implements **400+ unique telemetry events** (all prefixed with `tengu_`) covering virtually every user action and system event. Telemetry is **enabled by default** and uses an opt-out model.

---

## Transport Configuration

| Property | Value |
|----------|-------|
| **Destination** | DataDog Logs API |
| **Endpoint** | `https://http-intake.logs.us5.datadoghq.com/api/v2/logs` |
| **Region** | US5 (United States) |
| **API Key** | `pub[REDACTED]` (public intake key) |
| **Flush Interval** | 15 seconds (configurable via `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS`) |
| **Batch Size** | Up to 100 events per request |
| **Max Queue** | 5,000 events |
| **Retry** | Up to 30 attempts with exponential backoff |
| **Failed Events** | Persisted to disk, retried later |
| **Shutdown Flush** | 2-second timeout before process exit |

Optional **OpenTelemetry** is also supported but requires explicit opt-in via `CLAUDE_CODE_ENABLE_TELEMETRY=true`. Configured via `OTEL_METRICS_EXPORTER`, `OTEL_EXPORTER_OTLP_HEADERS`, etc.

### Debug Flags

| Flag | Purpose |
|------|--------|
| `tengu_log_segment_events` | Log Segment analytics events |
| `tengu_log_datadog_events` | Log DataDog events |

---

## Common Metadata (Every Event)

Every telemetry event includes these 14 properties:

| Property | Description |
|----------|-------------|
| `arch` | System architecture (x64, arm64, etc.) |
| `clientType` | CLI client identifier |
| `errorType` | Error classification |
| `http_status_range` | HTTP response status range |
| `http_status` | Specific HTTP status code |
| `model` | AI model being used |
| `platform` | Operating system (linux, darwin, win32) |
| `provider` | API provider (Anthropic, Bedrock, etc.) |
| `subscriptionType` | User subscription tier |
| `toolName` | Tool being invoked |
| `userBucket` | A/B test bucket |
| `userType` | User classification |
| `version` | CLI version |
| `versionBase` | Base version number |

### Event Envelope Structure

```typescript
{
  event_name: string,
  event_id: string,
  core_metadata: { /* system-level */ },
  user_metadata: { /* sanitized user info */ },
  event_metadata: { /* event-specific properties */ },
  user_id?: string  // SHA-256 hash of API key, truncated to 8 hex chars, modulo 30
}
```

User ID is a **one-way hash** -- non-reversible, used only for anonymous bucketing.

---

## All Telemetry Events

### Session Lifecycle (~28 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_init` | platform, version, config | Application initialization |
| `tengu_startup_telemetry` | platform details, versions | Startup telemetry batch |
| `tengu_startup_perf` | timing metrics | Startup performance metrics |
| `tengu_exit` | exit reason, duration | Application exit |
| `tengu_session_resumed` | entrypoint (cli_flag/direct), session_id | Session resumption |
| `tengu_session_renamed` | - | Session title changed |
| `tengu_session_tagged` | - | Session tagged |
| `tengu_session_linked_to_pr` | prNumber | Session linked to pull request |
| `tengu_session_forked_branches_fetched` | total_sessions, sessions_with_branches, max_branches_per_session, avg_branches_per_session, total_transcript_count | Branch/fork metadata |
| `tengu_session_group_expanded` | - | Session group UI expanded |
| `tengu_session_all_projects_toggled` | enabled | All projects filter toggled |
| `tengu_session_branch_filter_toggled` | enabled | Branch filter toggled |
| `tengu_session_worktree_filter_toggled` | enabled | Worktree filter toggled |
| `tengu_session_search_toggled` | enabled | Session search toggled |
| `tengu_session_rename_started` | - | Rename UI opened |
| `tengu_session_preview_opened` | - | Session preview opened |
| `tengu_session_tag_filter_changed` | - | Tag filter changed |
| `tengu_session_file_read` | is_session_memory, is_session_transcript | Session file accessed by tool |
| `tengu_session_memory_loaded` | content_length | Session memory file loaded |
| `tengu_session_memory_accessed` | subagent_name? | Session memory accessed by agent |
| `tengu_session_memory_file_read` | content_length | Session memory read via tool |
| `tengu_session_memory_extraction` | input_tokens, output_tokens, cache_read_input_tokens, cache_creation_input_tokens, config values | Memory extraction performed |
| `tengu_session_persistence_failed` | - | Failed to persist session |
| `tengu_session_quality_classification` | classification | Session quality auto-classification |

### API Call Telemetry (~14 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_api_query` | model, messagesLength, temperature, provider, buildAgeMins, betas, permissionMode, querySource, queryChainId, queryDepth, effortValue | Every API call to Claude |
| `tengu_api_success` | - | Successful API response |
| `tengu_api_error` | error details | API error occurred |
| `tengu_api_retry` | attempt number | API retry attempted |
| `tengu_api_opus_fallback_triggered` | - | Fallback to Opus model |
| `tengu_api_before_normalize` | - | Before message normalization |
| `tengu_api_after_normalize` | - | After message normalization |
| `tengu_api_cache_breakpoints` | - | Prompt cache breakpoint calculation |
| `tengu_api_key_saved_to_config` | - | API key saved to config file |
| `tengu_api_key_saved_to_keychain` | - | API key saved to system keychain |
| `tengu_api_key_keychain_error` | - | Keychain access error |
| `tengu_api_custom_*` | - | Custom API configuration events |

### Tool Usage (~15 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_tool_use_success` | toolName, isMcp, durationMs, mcpServerType?, mcpServerBaseUrl? | Tool executed successfully |
| `tengu_tool_use_error` | error, errorDetails?, errorCode?, messageID, toolName, isMcp, queryChainId, queryDepth, mcpServerType?, mcpServerBaseUrl?, durationMs | Tool execution failed |
| `tengu_tool_use_progress` | messageID, toolName, isMcp, queryChainId, queryDepth, mcpServerType?, mcpServerBaseUrl? | Tool execution progress |
| `tengu_tool_use_cancelled` | messageID, toolName, isMcp, toolUseID, queryChainId, queryDepth | Tool execution cancelled |
| `tengu_tool_use_granted_by_permission_hook` | tool details, permanent | Permission granted by hook |
| `tengu_tool_use_denied_in_config` | tool details | Permission denied in config |
| `tengu_tool_use_granted_in_config` | tool details | Permission granted in config |
| `tengu_tool_use_rejected_in_prompt` | tool details, isHook?, hasFeedback? | User rejected tool use |
| `tengu_tool_use_show_permission_request` | messageID, toolName, isMcp, decisionReasonType, sandboxEnabled | Permission prompt shown |
| `tengu_tool_use_diff_computed` | isEditTool/isWriteTool, durationMs, hasDiff | Diff computed for edit/write |
| `tengu_tool_use_tool_result_mismatch_error` | toolUseId, normalizedSequence, preNormalizedSequence, message counts, indices | Tool result sequence mismatch |
| `tengu_tool_use_can_use_tool_allowed` | - | Permission check allowed |
| `tengu_tool_use_can_use_tool_rejected` | - | Permission check rejected |
| `tengu_tool_result_persisted` | - | Tool result saved |
| `tengu_tool_search_mode_decision` | - | Tool search mode selected |
| `tengu_tool_search_outcome` | - | Tool search completed |
| `tengu_tool_prompt_changed` | - | Tool prompt modified |

### Model & Configuration (~20 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_model_command_menu` | action (cancel/model change), from_model?, to_model? | Model selection menu interaction |
| `tengu_model_command_inline` | args | Inline model command |
| `tengu_model_command_inline_help` | args | Inline model help command |
| `tengu_model_command_menu_effort` | - | Effort level menu |
| `tengu_model_fallback_triggered` | - | Model fallback activated |
| `tengu_model_picker_hotkey` | - | Model picker hotkey used |
| `tengu_model_response_keyword_detected` | - | Special keyword in response |
| `tengu_model_whitespace_response` | - | Empty/whitespace response |
| `tengu_startup_manual_model_config` | config | Manual model configuration |
| `tengu_unknown_model_cost` | - | Unknown model cost calculation |
| `tengu_config_changed` | - | Config file changed |
| `tengu_config_cache_stats` | - | Config cache statistics |
| `tengu_config_lock_contention` | - | Config file lock contention |
| `tengu_config_stale_write` | - | Stale config write detected |
| `tengu_context_size` | - | Context size calculation |
| `tengu_max_tokens_context_overflow_adjustment` | - | Max tokens adjusted |

### Authentication & OAuth (~26 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_oauth_flow_start` | - | OAuth flow initiated |
| `tengu_oauth_success` | - | OAuth completed successfully |
| `tengu_oauth_error` | - | OAuth failed |
| `tengu_oauth_api_key` | - | API key OAuth method |
| `tengu_oauth_api_key_error` | - | API key OAuth error |
| `tengu_oauth_auth_code_received` | - | Auth code received |
| `tengu_oauth_automatic_redirect` | - | Auto redirect to OAuth |
| `tengu_oauth_automatic_redirect_error` | - | Auto redirect failed |
| `tengu_oauth_claudeai_forced` | - | Claude.ai forced |
| `tengu_oauth_claudeai_selected` | - | Claude.ai selected |
| `tengu_oauth_console_forced` | - | Console forced |
| `tengu_oauth_console_selected` | - | Console selected |
| `tengu_oauth_manual_entry` | - | Manual OAuth entry |
| `tengu_oauth_platform_selected` | - | Platform selected |
| `tengu_oauth_profile_fetch_success` | - | Profile fetched |
| `tengu_oauth_roles_stored` | - | Roles stored |
| `tengu_oauth_storage_warning` | - | Storage warning |
| `tengu_oauth_token_exchange_error` | - | Token exchange failed |
| `tengu_oauth_token_exchange_success` | - | Token exchanged |
| `tengu_oauth_token_refresh_*` | - | Multiple token refresh events |
| `tengu_oauth_tokens_inference_only` | - | Inference-only tokens |
| `tengu_oauth_tokens_not_claude_ai` | - | Non-Claude.ai tokens |
| `tengu_oauth_tokens_saved` | - | Tokens saved |
| `tengu_oauth_tokens_save_exception` | - | Save exception |
| `tengu_oauth_tokens_save_failed` | - | Save failed |
| `tengu_oauth_user_roles_error` | - | User roles error |

### MCP Integration (~27 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_mcp_cli_status` | - | MCP CLI status check |
| `tengu_mcp_cli_command_executed` | command, tool_name?, success, duration_ms, error_type? | MCP CLI command executed |
| `tengu_mcp_add` | - | MCP server added |
| `tengu_mcp_get` | - | MCP server info retrieved |
| `tengu_mcp_list` | - | MCP servers listed |
| `tengu_mcp_list_changed` | - | MCP server list changed |
| `tengu_mcp_servers` | - | MCP servers configuration |
| `tengu_mcp_dialog_choice` | - | MCP dialog interaction |
| `tengu_mcp_headers` | - | MCP headers configured |
| `tengu_mcp_auth_config_authenticate` | - | MCP auth configured |
| `tengu_mcp_auth_config_clear` | - | MCP auth cleared |
| `tengu_mcp_server_connection_succeeded` | - | MCP server connected |
| `tengu_mcp_server_connection_failed` | - | MCP server connection failed |
| `tengu_mcp_ide_server_connection_succeeded` | - | IDE MCP server connected |
| `tengu_mcp_ide_server_connection_failed` | - | IDE MCP server failed |
| `tengu_mcp_server_needs_auth` | - | MCP server needs auth |
| `tengu_mcp_tool_call_auth_error` | - | MCP tool auth error |
| `tengu_mcp_tools_commands_loaded` | - | MCP tools loaded |
| `tengu_mcp_oauth_flow_start` | - | MCP OAuth started |
| `tengu_mcp_oauth_flow_success` | - | MCP OAuth succeeded |
| `tengu_mcp_oauth_flow_error` | - | MCP OAuth failed |
| `tengu_claudeai_mcp_connectors` | - | Claude.ai MCP connectors |
| `tengu_claudeai_mcp_eligibility` | - | Claude.ai MCP eligibility |
| `tengu_claudeai_mcp_toggle` | - | Claude.ai MCP toggled |
| `tengu_claudeai_mcp_reconnect` | - | Claude.ai MCP reconnected |
| `tengu_claudeai_mcp_clear_auth_completed` | - | Auth cleared |
| `tengu_claudeai_limits_status_changed` | - | Limits status changed |

### File Operations (~12 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_file_operation` | - | File operation performed |
| `tengu_file_changed` | - | File change detected |
| `tengu_file_write_optimization` | - | Write optimization applied |
| `tengu_file_suggestions_git_ls_files` | - | Git ls-files for suggestions |
| `tengu_file_suggestions_ripgrep` | - | Ripgrep for suggestions |
| `tengu_atomic_write_error` | - | Atomic write failed |
| `tengu_watched_file_compression_failed` | - | File compression failed |
| `tengu_watched_file_stat_error` | - | File stat error |

### File History & Snapshots (~12 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_file_history_backup_file_created` | - | Backup created |
| `tengu_file_history_backup_file_failed` | - | Backup failed |
| `tengu_file_history_backup_deleted_file` | - | Backup of deleted file |
| `tengu_file_history_snapshot_success` | - | Snapshot created |
| `tengu_file_history_snapshot_failed` | - | Snapshot failed |
| `tengu_file_history_snapshots_setting_changed` | - | Snapshots setting changed |
| `tengu_file_history_track_edit_success` | - | Edit tracked |
| `tengu_file_history_track_edit_failed` | - | Edit tracking failed |
| `tengu_file_history_rewind_success` | - | File rewound |
| `tengu_file_history_rewind_failed` | - | Rewind failed |
| `tengu_file_history_rewind_restore_file_failed` | - | Restore failed |
| `tengu_file_history_resume_copy_failed` | - | Resume copy failed |

### Git Operations (~3 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_git_operation` | operation (commit/commit_amend/pr_create) | Git command executed |
| `tengu_git_index_lock_error` | - | Git index lock error |
| `tengu_worktree_detection` | - | Worktree detected |

### Bash Tool (~7 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_bash_tool_command_executed` | - | Bash command run |
| `tengu_bash_command_explicitly_backgrounded` | command_type | Command backgrounded by user |
| `tengu_bash_command_timeout_backgrounded` | - | Command backgrounded on timeout |
| `tengu_bash_command_interrupt_backgrounded` | - | Command backgrounded on interrupt |
| `tengu_bash_security_check_triggered` | - | Security check triggered |
| `tengu_bash_tool_simple_echo` | - | Simple echo detected |
| `tengu_bash_tool_haiku_file_paths_read` | - | Haiku file paths read |

### Prompt Compaction (~14 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_compact` | tokens_before, tokens_after, cache stats | Full compaction performed |
| `tengu_compact_failed` | reason, stage | Compaction failed |
| `tengu_partial_compact` | tokens_before, tokens_after | Partial compaction |
| `tengu_partial_compact_failed` | reason, stage | Partial compaction failed |
| `tengu_compact_cache_prefix` | - | Feature flag check |
| `tengu_compact_cache_sharing_success` | cache stats | Cache sharing succeeded |
| `tengu_compact_cache_sharing_fallback` | reason | Cache sharing fallback |
| `tengu_compact_streaming_retry` | - | Streaming retry after compact |
| `tengu_post_compact_file_restore_success` | - | File restored post-compact |
| `tengu_post_compact_file_restore_error` | - | File restore failed |
| `tengu_post_autocompact_turn` | - | Turn after auto-compact |
| `tengu_post_compact_survey` | - | Post-compact survey |
| `tengu_post_compact_survey_event` | - | Survey event |
| `tengu_auto_compact_setting_changed` | - | Auto-compact setting changed |
| `tengu_sm_compact_threshold_exceeded` | - | Compact threshold exceeded |
| `tengu_prompt_cache_break` | - | Prompt cache break |

### Streaming & API Response (~10 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_streaming_error` | - | Streaming error |
| `tengu_streaming_fallback_to_non_streaming` | - | Fallback to non-streaming |
| `tengu_streaming_stall` | - | Stream stalled |
| `tengu_streaming_stall_summary` | - | Stall summary |
| `tengu_streaming_tool_execution_used` | - | Streaming tool execution |
| `tengu_streaming_tool_execution_not_used` | - | Non-streaming tool execution |
| `tengu_stream_no_events` | - | No events in stream |
| `tengu_filtered_orphaned_thinking_message` | - | Orphaned thinking filtered |
| `tengu_filtered_trailing_thinking_block` | - | Trailing thinking filtered |
| `tengu_filtered_whitespace_only_assistant` | - | Whitespace message filtered |
| `tengu_fixed_empty_assistant_content` | - | Empty content fixed |

### Input & User Interaction (~12 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_input_prompt` | - | User submitted prompt |
| `tengu_input_command` | - | Command input |
| `tengu_input_slash_invalid` | - | Invalid slash command |
| `tengu_input_slash_missing` | - | Missing slash command |
| `tengu_input_background` | - | Background input mode |
| `tengu_single_word_prompt` | length | Single-word prompt submitted |
| `tengu_code_prompt_ignored` | - | Code prompt ignored |
| `tengu_query_before_attachments` | - | Query before attachments |
| `tengu_query_after_attachments` | - | Query after attachments |
| `tengu_continue` | - | Continue command |
| `tengu_continue_print` | - | Continue print |
| `tengu_resume_print` | - | Resume print |

### Attachments & Media (~8 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_attachments` | - | Attachments added |
| `tengu_attachment_file_too_large` | - | File too large |
| `tengu_attachment_compute_duration` | - | Compute duration |
| `tengu_image_api_validation_failed` | - | Image validation failed |
| `tengu_image_compress_failed` | - | Image compression failed |
| `tengu_image_resize_failed` | - | Image resize failed |
| `tengu_image_resize_fallback` | - | Resize fallback used |
| `tengu_pasted_image_resize_attempt` | - | Pasted image resize |
| `tengu_pdf_page_extraction` | - | PDF page extracted |
| `tengu_pdf_reference_attachment` | - | PDF as attachment |

### Agent System (~22 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_agent_created` | agent details | New agent created |
| `tengu_agent_definition_generated` | agent_identifier | Agent definition generated |
| `tengu_agent_flag` | - | Agent flag set |
| `tengu_agent_parse_error` | - | Agent parsing error |
| `tengu_agent_stop_hook_error` | - | Stop hook error |
| `tengu_agent_stop_hook_max_turns` | - | Max turns reached |
| `tengu_agent_stop_hook_success` | - | Stop hook succeeded |
| `tengu_agent_tool_completed` | - | Agent tool completed |
| `tengu_agent_tool_selected` | - | Agent tool selected |
| `tengu_agent_memory_loaded` | - | Agent memory loaded |
| `tengu_agent_color_set` | - | Agent color set |
| `tengu_agent_name_set` | - | Agent name set |
| `tengu_at_mention_agent_success` | - | @mention agent succeeded |
| `tengu_at_mention_agent_not_found` | - | @mention agent not found |
| `tengu_at_mention_extracting_filename_success` | - | Filename extracted from @mention |
| `tengu_at_mention_extracting_filename_error` | - | Filename extraction failed |
| `tengu_at_mention_extracting_directory_success` | - | Directory extracted |
| `tengu_at_mention_mcp_resource_success` | - | MCP resource @mention |
| `tengu_at_mention_mcp_resource_error` | - | MCP resource error |
| `tengu_subagent_at_mention` | - | Subagent @mentioned |
| `tengu_fork_agent_query` | - | Agent query forked |
| `tengu_teammate_mode_changed` | - | Teammate mode changed |

### Skills & Plugins (~12 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_skill_loaded` | skill details | Skill loaded |
| `tengu_skill_file_changed` | source | Skill file changed |
| `tengu_skill_tool_invocation` | - | Skill tool invoked |
| `tengu_skill_tool_slash_prefix` | - | Skill slash command |
| `tengu_plugin_installed` | - | Plugin installed |
| `tengu_plugin_installed_cli` | - | Plugin installed via CLI |
| `tengu_plugin_updated_cli` | - | Plugin updated via CLI |
| `tengu_plugin_uninstalled_cli` | - | Plugin uninstalled |
| `tengu_plugin_enabled_cli` | - | Plugin enabled |
| `tengu_plugin_disabled_cli` | - | Plugin disabled |
| `tengu_plugin_disabled_all_cli` | - | All plugins disabled |
| `tengu_plugin_list_command` | - | Plugin list command |
| `tengu_plugin_update_command` | - | Plugin update command |

### IDE Integration (~10 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_ext_installed` | - | Extension installed |
| `tengu_ext_install_error` | - | Extension install error |
| `tengu_ext_ide_command` | - | IDE command executed |
| `tengu_ext_at_mentioned` | - | IDE @mention |
| `tengu_ext_diff_accepted` | - | Diff accepted in IDE |
| `tengu_ext_diff_rejected` | - | Diff rejected in IDE |
| `tengu_ext_will_show_diff` | - | Will show diff in IDE |
| `tengu_auto_connect_ide_changed` | - | Auto-connect IDE setting |
| `tengu_auto_install_ide_extension_changed` | - | Auto-install setting |
| `tengu_vscode_onboarding` | - | VS Code onboarding |
| `tengu_vscode_review_upsell` | - | VS Code review upsell |

### UI & Display Settings (~23 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_editor_mode_changed` | mode, source | Editor mode changed |
| `tengu_mode_cycle` | - | Mode cycled |
| `tengu_output_style_changed` | - | Output style changed |
| `tengu_output_style_command_inline` | - | Inline output style |
| `tengu_output_style_command_inline_help` | - | Inline help |
| `tengu_output_style_command_menu` | - | Output style menu |
| `tengu_diff_tool_changed` | - | Diff tool changed |
| `tengu_code_diff_cli` | - | Code diff CLI |
| `tengu_code_diff_footer_setting_changed` | - | Footer setting changed |
| `tengu_pr_status_cli` | - | PR status CLI |
| `tengu_pr_status_footer_setting_changed` | - | Footer setting changed |
| `tengu_thinking` | - | Thinking mode |
| `tengu_thinking_toggled` | - | Thinking toggled |
| `tengu_thinking_toggled_hotkey` | - | Thinking hotkey |
| `tengu_tips_setting_changed` | - | Tips setting changed |
| `tengu_tip_shown` | tip details | Tip displayed |
| `tengu_reduce_motion_setting_changed` | - | Reduce motion changed |
| `tengu_terminal_progress_bar_setting_changed` | - | Progress bar setting |
| `tengu_status_line_mount` | - | Status line mounted |
| `tengu_language_changed` | - | Language changed |
| `tengu_keybinding_fallback_used` | - | Keybinding fallback |
| `tengu_flicker` | - | UI flicker detected |

### Permissions & Security (~9 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_permission_request_escape` | - | Permission request escaped |
| `tengu_permission_request_option_selected` | - | Permission option selected |
| `tengu_permission_explainer_generated` | - | Explainer generated |
| `tengu_permission_explainer_error` | - | Explainer error |
| `tengu_bypass_permissions_mode_dialog_shown` | - | Bypass dialog shown |
| `tengu_bypass_permissions_mode_dialog_accept` | - | Bypass accepted |
| `tengu_disable_bypass_permissions_mode` | - | Bypass disabled |
| `tengu_trust_dialog_shown` | - | Trust dialog shown |
| `tengu_respect_gitignore_setting_changed` | - | Gitignore setting changed |

### Transcript & Todos (~7 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_transcript_accessed` | - | Transcript accessed |
| `tengu_transcript_exit` | - | Transcript exited |
| `tengu_transcript_toggle_show_all` | - | Show all toggled |
| `tengu_transcript_view_enter` | - | Transcript view entered |
| `tengu_transcript_view_exit` | - | Transcript view exited |
| `tengu_toggle_transcript` | - | Transcript toggled |
| `tengu_toggle_todos` | - | Todos toggled |
| `tengu_message_selector_opened` | - | Message selector opened |

### Memory & Context (~5 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_memdir_accessed` | tool, subagent_name? | Memory directory accessed |
| `tengu_memdir_file_read` | - | Memory file read |
| `tengu_memdir_file_write` | - | Memory file written |
| `tengu_memdir_file_edit` | - | Memory file edited |
| `tengu_memdir_loaded` | - | Memory directory loaded |

### Teleport / Session Migration (~12 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_teleport_started` | source | Teleport initiated |
| `tengu_teleport_cancelled` | - | Teleport cancelled |
| `tengu_teleport_resume_session` | source/mode | Resuming via teleport |
| `tengu_teleport_resume_error` | - | Resume error |
| `tengu_teleport_interactive_mode` | - | Interactive teleport |
| `tengu_teleport_error_branch_checkout_failed` | - | Branch checkout failed |
| `tengu_teleport_error_git_not_clean` | - | Git not clean |
| `tengu_teleport_error_repo_mismatch_sessions_api` | - | Repo mismatch |
| `tengu_teleport_error_repo_not_in_git_dir_sessions_api` | - | Not in git dir |
| `tengu_teleport_errors_detected` | - | Errors detected |
| `tengu_teleport_errors_resolved` | - | Errors resolved |

### Remote Backend (~3 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_remote_backend` | - | Feature flag check |
| `tengu_remote_create_session` | - | Remote session created |
| `tengu_remote_create_session_success` | session_id | Success |
| `tengu_remote_create_session_error` | - | Failed |

### Conversation Management (~4 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_conversation_forked` | - | Conversation forked |
| `tengu_slash_command_forked` | - | Slash command forked |
| `tengu_chain_parent_cycle` | - | Parent cycle detected |
| `tengu_concurrent_onquery_enqueued` | - | Query enqueued |

### Plan Mode & Interviews (~5 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_plan_exit` | - | Plan mode exited |
| `tengu_plan_mode_interview_phase` | - | Interview phase |
| `tengu_ask_user_question_accepted` | - | Question accepted |
| `tengu_ask_user_question_rejected` | - | Question rejected |
| `tengu_ask_user_question_respond_to_claude` | - | Responded to Claude |
| `tengu_ask_user_question_finish_plan_interview` | - | Interview finished |

### Feedback System (~7 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_accept_feedback_mode_entered` | - | Accept feedback mode |
| `tengu_accept_feedback_mode_collapsed` | - | Mode collapsed |
| `tengu_accept_submitted` | - | Accept submitted |
| `tengu_reject_feedback_mode_entered` | - | Reject feedback mode |
| `tengu_reject_feedback_mode_collapsed` | - | Mode collapsed |
| `tengu_reject_submitted` | - | Reject submitted |
| `tengu_feedback_survey_config` | - | Survey config |
| `tengu_feedback_survey_event` | - | Survey event |
| `tengu_bug_report_submitted` | report details | Bug report submitted |

### Rate Limits & Cost (~4 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_rate_limit_options_menu_cancel` | - | Rate limit menu cancelled |
| `tengu_rate_limit_options_menu_select_extra_usage` | - | Extra usage selected |
| `tengu_rate_limit_options_menu_select_upgrade` | - | Upgrade selected |
| `tengu_cost_threshold_acknowledged` | - | Cost threshold ack |
| `tengu_refusal_api_response` | - | API refusal response |

### Updates & Versioning (~22 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_update_check` | - | Update check performed |
| `tengu_version_check_success` | - | Version check succeeded |
| `tengu_version_check_failure` | - | Version check failed |
| `tengu_version_config` | - | Version config |
| `tengu_version_lock_acquired` | - | Version lock acquired |
| `tengu_version_lock_failed` | - | Version lock failed |
| `tengu_pid_based_version_locking` | - | PID-based locking |
| `tengu_autoupdate_enabled` | - | Auto-update enabled |
| `tengu_autoupdate_channel_changed` | - | Update channel changed |
| `tengu_auto_updater_success` | - | Auto-updater succeeded |
| `tengu_auto_updater_fail` | - | Auto-updater failed |
| `tengu_auto_updater_lock_contention` | - | Lock contention |
| `tengu_auto_updater_windows_npm_in_wsl` | - | Windows NPM in WSL |
| `tengu_native_auto_updater_start` | - | Native updater started |
| `tengu_native_auto_updater_success` | - | Native updater succeeded |
| `tengu_native_auto_updater_fail` | - | Native updater failed |
| `tengu_native_auto_updater_lock_contention` | - | Lock contention |
| `tengu_native_auto_updater_up_to_date` | - | Already up to date |
| `tengu_native_update_complete` | - | Update complete |
| `tengu_native_install_binary_success` | - | Binary install success |
| `tengu_native_install_binary_failure` | - | Binary install failure |
| `tengu_native_install_package_success` | - | Package install success |
| `tengu_native_install_package_failure` | - | Package install failure |

### Native Migration (~7 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_auto_migrate_to_native_attempt` | - | Migration attempted |
| `tengu_auto_migrate_to_native_success` | - | Migration succeeded |
| `tengu_auto_migrate_to_native_failure` | - | Migration failed |
| `tengu_auto_migrate_to_native_partial` | - | Partial migration |
| `tengu_auto_migrate_to_native_ui_shown` | - | Migration UI shown |
| `tengu_auto_migrate_to_native_ui_success` | - | Migration UI success |
| `tengu_auto_migrate_to_native_ui_error` | - | Migration UI error |

### Settings Sync (~5 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_settings_sync_download_success` | - | Settings synced |
| `tengu_settings_sync_download_error` | - | Sync error |
| `tengu_settings_sync_download_empty` | - | No settings to sync |
| `tengu_settings_sync_download_skipped` | - | Sync skipped |
| `tengu_settings_sync_download_fetch_failed` | - | Fetch failed |

### Shell & Terminal (~6 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_shell_set_cwd` | - | Shell CWD set |
| `tengu_shell_snapshot_error` | - | Snapshot error |
| `tengu_shell_snapshot_failed` | - | Snapshot failed |
| `tengu_shell_unknown_error` | - | Unknown shell error |
| `tengu_shell_completion_failed` | - | Shell completion failed |
| `tengu_stdin_interactive` | - | Interactive stdin mode |

### Marketplace (~5 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_marketplace_added` | - | Item added |
| `tengu_marketplace_removed` | - | Item removed |
| `tengu_marketplace_updated` | - | Item updated |
| `tengu_marketplace_updated_all` | - | All items updated |
| `tengu_official_marketplace_auto_install` | - | Auto-install from marketplace |

### GitHub Integration (~6 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_install_github_app_started` | - | GitHub app install started |
| `tengu_install_github_app_completed` | - | Install completed |
| `tengu_install_github_app_error` | - | Install error |
| `tengu_install_github_app_step_completed` | - | Step completed |
| `tengu_setup_github_actions_started` | - | Actions setup started |
| `tengu_setup_github_actions_completed` | - | Actions setup completed |
| `tengu_setup_github_actions_failed` | reason | Actions setup failed |

### Slack Integration (~1 event)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_install_slack_app_clicked` | - | Slack app install clicked |

### Grove (Privacy/Policy System) (~9 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_grove_policy_viewed` | - | Policy viewed |
| `tengu_grove_policy_submitted` | - | Policy submitted |
| `tengu_grove_policy_toggled` | - | Policy toggled |
| `tengu_grove_policy_dismissed` | - | Policy dismissed |
| `tengu_grove_policy_escaped` | - | Policy escaped |
| `tengu_grove_policy_exited` | - | Policy exited |
| `tengu_grove_print_viewed` | - | Grove print viewed |
| `tengu_grove_privacy_settings_viewed` | - | Privacy settings viewed |
| `tengu_grove_oauth_*` | - | Grove OAuth events |

### Guest Passes & Upsells (~4 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_guest_passes_visited` | - | Guest passes page |
| `tengu_guest_passes_link_copied` | - | Link copied |
| `tengu_guest_passes_upsell_shown` | - | Upsell shown |
| `tengu_switch_to_subscription_notice_shown` | - | Subscription notice |

### Chrome Integration (~4 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_claude_in_chrome_setup` | platform | Chrome setup started |
| `tengu_claude_in_chrome_setup_failed` | platform | Setup failed |
| `tengu_claude_in_chrome_setting_changed` | - | Setting changed |
| `tengu_chrome_auto_enable` | - | Feature flag check |

### Code Indexing & Analysis (~5 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_code_indexing_tool_used` | - | Code indexing used |
| `tengu_dir_search` | - | Directory search |
| `tengu_tree_sitter_load` | - | Tree-sitter loaded |
| `tengu_ripgrep_availability` | - | Ripgrep availability check |
| `tengu_ripgrep_eagain_retry` | - | EAGAIN retry |

### Hooks System (~5 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_hook_created` | - | Hook created |
| `tengu_hook_deleted` | - | Hook deleted |
| `tengu_hooks_command` | - | Hooks command |
| `tengu_run_hook` | - | Hook executed |
| `tengu_repl_hook_finished` | - | REPL hook finished |

### CLAUDE.md Files (~5 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_write_claudemd` | - | CLAUDE.md written |
| `tengu_claude_md_external_includes_dialog_shown` | - | External includes dialog |
| `tengu_claude_md_external_includes_dialog_accepted` | - | Dialog accepted |
| `tengu_claude_md_external_includes_dialog_declined` | - | Dialog declined |
| `tengu_claude_md_permission_error` | - | Permission error |
| `tengu_claude_rules_md_permission_error` | - | Rules.md permission error |

### Errors & Exceptions (~7 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_uncaught_exception` | error details | Uncaught exception |
| `tengu_unhandled_rejection` | error details | Unhandled promise rejection |
| `tengu_unexpected_tool_result` | - | Unexpected tool result |
| `tengu_preflight_check_failed` | - | Preflight check failed |
| `tengu_node_warning` | - | Node.js warning |
| `tengu_react_vulnerability_notice_shown` | - | React vulnerability notice |
| `tengu_react_vulnerability_warning` | - | React vulnerability warning |

### System Prompts (~6 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_sysprompt_block` | - | Sysprompt block |
| `tengu_sysprompt_boundary_found` | - | Boundary found |
| `tengu_sysprompt_missing_boundary_marker` | - | Missing boundary |
| `tengu_sysprompt_no_stable_tool_for_cache` | - | No stable tool |
| `tengu_sysprompt_using_tool_based_cache` | - | Tool-based cache |
| `tengu_system_prompt_global_cache` | - | Feature flag |

### Onboarding (~4 events)

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_began_setup` | - | Setup started |
| `tengu_onboarding_step` | - | Onboarding step |
| `tengu_setup_token_command` | - | Token setup command |
| `tengu_doctor_command` | - | Doctor command |

### Miscellaneous

| Event | Data Fields | Description |
|-------|-------------|-------------|
| `tengu_structured_output_enabled` | - | Structured output enabled |
| `tengu_structured_output_failure` | - | Structured output failed |
| `tengu_managed_settings_loaded` | - | Managed settings loaded |
| `tengu_managed_settings_security_dialog_shown` | - | Security dialog shown |
| `tengu_notification_method_used` | - | Notification method used |
| `tengu_timer` | - | Timer event |
| `tengu_headless_latency` | - | Headless mode latency |
| `tengu_unary_event` | - | Generic unary event |
| `tengu_attribution_header` | - | Attribution header |
| `tengu_dynamic_skills_changed` | - | Dynamic skills changed |
| `tengu_speculation` | - | Speculation feature |
| `tengu_tst_names_in_messages` | - | TST names in messages |
| `tengu_prompt_suggestion` | - | Prompt suggestion shown |
| `tengu_prompt_suggestion_init` | - | Suggestions initialized |
| `tengu_agentic_search_cancelled` | - | Search cancelled |
| `tengu_aws` | - | AWS event |

### Internal Feature Flags (~17 events)

These are internal A/B test or feature flag checks, not user actions:

| Event | Purpose |
|-------|--------|
| `tengu_amber_flint` | Internal feature |
| `tengu_copper_lantern` | Internal feature |
| `tengu_copper_lantern_config` | Config check |
| `tengu_coral_fern` | Internal feature |
| `tengu_marble_anvil` | Internal feature |
| `tengu_marble_kite` | Internal feature |
| `tengu_marble_lantern_disabled` | Feature disabled |
| `tengu_oboe` | Internal feature |
| `tengu_opus` | Model feature |
| `tengu_quartz_lantern` | Internal feature |
| `tengu_quiet_fern` | Internal feature |
| `tengu_scarf_coffee` | Internal feature |
| `tengu_silver_lantern` | Internal feature |
| `tengu_thinkback` | Feature check |
| `tengu_tool_pear` | Tool feature |
| `tengu_vinteuil_phrase` | Internal feature |

---

## Privacy Analysis

### What IS Collected

- **Model/API metadata**: model name, temperature, token counts, cache statistics, effort value
- **Tool names and execution times**: which tools are used (Read, Write, Bash, etc.) and how long they take
- **File operation metadata**: file paths, content lengths, operation types (not file contents)
- **Error messages**: full error text including validation errors (up to 2,000 characters)
- **Session metadata**: session IDs, timing, message counts
- **System information**: platform, OS version, build age, Node.js version, architecture
- **User interactions**: button clicks, menu selections, hotkey usage, setting changes
- **Git metadata**: operation types (commit, pr_create), worktree detection
- **MCP server types**: server base URLs and connection types
- **Agent/subagent names**: when using the agent system
- **PR numbers**: when linking sessions to pull requests

### What is NOT Collected

- **User prompts**: the actual content of messages is NOT sent
- **AI responses**: Claude's response content is NOT sent
- **File contents**: only lengths, never contents
- **Code snippets**: code is NOT sent
- **API keys**: actively redacted before logging
- **Passwords**: never logged
- **Tool input/output data**: not sent in telemetry events

### Data Redaction

Active redaction via `cG1()` function:

| Pattern | Replacement |
|---------|-------------|
| Anthropic API keys (`sk-ant-*`) | `[REDACTED_API_KEY]` |
| AWS access keys (`AKIA*`) | `[REDACTED_AWS_KEY]` |
| GCP API keys (`AIza*`) | `[REDACTED_GCP_KEY]` |
| GCP service accounts (`*@*.iam.gserviceaccount.com`) | `[REDACTED_GCP_SERVICE_ACCOUNT]` |
| Generic `x-api-key` headers | `[REDACTED_API_KEY]` |

URL query parameters are also sanitized (non-allowlisted params redacted). Sensitive HTTP headers (Authorization, Cookie, etc.) are stripped.

### Error Reporting

The `K1()` function captures:
- Error stack trace or message
- Timestamp
- No user data included

Error reporting is skipped when using Bedrock, Vertex, Foundry, or when `DISABLE_ERROR_REPORTING` is set.

---

## Opt-Out Mechanisms

| # | Variable | Effect |
|---|----------|--------|
| 1 | `DISABLE_TELEMETRY=true` | Primary telemetry opt-out |
| 2 | `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=true` | Disables all non-essential network traffic |
| 3 | `DISABLE_ERROR_REPORTING=true` | Disables error reporting specifically |
| 4 | `CLAUDE_CODE_USE_BEDROCK=1` | Auto-disables telemetry (alternative provider) |
| 5 | `CLAUDE_CODE_USE_VERTEX=1` | Auto-disables telemetry (alternative provider) |
| 6 | `CLAUDE_CODE_USE_FOUNDRY=1` | Auto-disables telemetry (alternative provider) |

Check function: `PZ()`

Alternatively, block `*.datadoghq.com` at the firewall/DNS level.

---

## Data Flow

```
Claude CLI Event
     |
     v
PZ() check -------> [DISABLED] --> No telemetry sent
     |                              (6 env vars can trigger this)
     | [ENABLED]
     v
Redact sensitive data
     | (API keys, credentials, URLs, headers)
     v
Add common properties
     | (arch, platform, version, model, etc. -- 14 fields)
     v
Buffer events (max 100 per batch)
     |
     v
Flush every 15 seconds
     |
     v
POST to DataDog
     | https://http-intake.logs.us5.datadoghq.com/api/v2/logs
     v
Success --> Clear queue
     |
     | Failure
     v
Persist to disk --> Retry with backoff (max 30 attempts)
```

### Shutdown Behavior

- 5-second timeout for graceful shutdown
- Session end hooks executed
- Telemetry flushed with 2-second timeout
- Force exit after cleanup

---

## Key Findings

1. **Comprehensive Tracking**: 400+ distinct telemetry events covering virtually every user action and system event
2. **DataDog as Transport**: All telemetry flows to DataDog's US5 region endpoint
3. **Enabled by Default**: Opt-out model, not opt-in
4. **Metadata-Heavy**: Focuses on operational metadata rather than content (file paths and lengths, not contents; tool names and times, not inputs/outputs)
5. **Detailed Error Tracking**: Errors include full messages and stack traces (up to 2,000 chars)
6. **Session Tracking**: Extensive lifecycle tracking including PR linking, forking, and quality classification
7. **Performance Monitoring**: API call durations, tool execution times, startup performance, streaming stalls
8. **Feature Usage Analytics**: Detailed tracking of feature usage, feature flag status, UI interactions, and settings changes
9. **Strong Redaction**: Proactive removal of API keys, credentials, and sensitive data patterns
10. **Multiple Escape Hatches**: 6 different environment variables to disable telemetry

---

## Privacy Assessment

**Strengths:**
- Multiple opt-out mechanisms (6 environment variables)
- Automatic disable when using alternative cloud providers (Bedrock, Vertex, Foundry)
- Comprehensive API key redaction (Anthropic, AWS, GCP)
- URL and header sanitization
- No file contents or conversation data sent
- User ID is one-way hash (non-reversible)
- OpenTelemetry requires explicit opt-in

**Observations:**
- Telemetry is enabled by default (opt-out, not opt-in)
- DataDog endpoint is US-based (us5.datadoghq.com)
- Error stack traces include file paths
- No documented data retention policy in the code
- No evidence of anonymization beyond user ID hashing
