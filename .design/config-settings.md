# Feature: Configuration & Settings (`claude-config` crate)

## Summary
Implement the configuration system that manages Claude Code's multi-layered settings, global config, project config, output styles, and data migrations. Settings follow a 5-level precedence hierarchy (user → project → local → flag → policy, with policy highest). The global config (`~/.claude.json`) tracks 30+ fields including auth, UI preferences, model selection, project configs, MCP servers, and startup metrics. All 11 data migrations must be preserved to handle upgrades from any previous version.

## Requirements

- REQ-1: Settings must load from 5 sources in precedence order (lowest to highest): (1) userSettings `~/.claude/settings.json` (or `cowork_settings.json` when use_cowork_plugins is set), (2) projectSettings `.claude/settings.json`, (3) localSettings `.claude/local_settings.json` (gitignored), (4) flagSettings from `--settings /path` CLI flag, (5) policySettings from `/opt/managed-settings.json` + `.d/*.json` directory (enterprise-managed, highest precedence). Merged result applies all levels with higher precedence overriding lower
- REQ-2: The Settings struct must contain 100+ fields organized into sections: env/auth (env, api_key_helper, aws/gcp/xaa config), MCP control (enabled/disabled/allowed/denied servers, enable_all_project), permissions (allow/deny/ask rules, default_mode, disable_bypass, disable_auto), hooks (hooks settings, disable_all, managed-only, allowed_http_urls, env vars), plugins (enabled, marketplaces, strict/blocked, configs), customization (attribution for commit/PR, output_style, language, spinner), model/performance (model, available_models, overrides, effort, advisor, fast_mode, always_thinking), UI (respect_gitignore, terminal_title, spinner_verbs, tips), remote/bridge (default_environment_id, control_at_startup), worktree (symlink_directories, sparse_paths), and advanced (cleanup_period_days, file_suggestion, status_line)
- REQ-3: GlobalConfig must contain 30+ fields: auth (oauth_account, primary_api_key, api_key_helper), UI (theme, editor_mode, verbose, preferred_notif_channel), models (model, available_models, overrides, effort), projects (HashMap<String, ProjectConfig>), MCP (servers, disabled/enabled), hooks, permissions, cache (statsig_gates, growthbook_features, dynamic_configs), tracking (tips_history, skill_usage, hints, experiment_notices), features (auto_compact, show_turn_duration, file_checkpointing, terminal_progress_bar), startup (num_startups, first_start_time, completed_onboarding, onboarding_version)
- REQ-4: ProjectConfig must contain per-directory settings: allowed_tools, mcp_context_uris, mcp_servers, last_api_duration, last_tool_duration, last_cost, trust_dialog_accepted, project_onboarding, enabled/disabled_mcp_servers, active_worktree_session, remote_control_spawn_mode, and session metrics
- REQ-5: All 11 data migrations must run sequentially on version mismatch (CURRENT_MIGRATION_VERSION = 11): (1) migrateAutoUpdatesToSettings, (2) migrateBypassPermissionsAcceptedToSettings, (3) migrateEnableAllProjectMcpServersToSettings, (4) resetProToOpusDefault, (5) migrateSonnet1mToSonnet45, (6) migrateLegacyOpusToCurrent, (7) migrateSonnet45ToSonnet46, (8) migrateOpusToOpus1m, (9) migrateReplBridgeEnabledToRemoteControlAtStartup, (10) resetAutoModeOptInForDefaultOffer, (11) migrateFennecToOpus. Each takes `(config, settings) → (config, settings)`
- REQ-6: Output style loading must parse markdown files from the output_styles_path and apply user-customizable response formatting to the system prompt
- REQ-7: Settings validation must use `serde` with `#[serde(deny_unknown_fields)]` for strict parsing where appropriate, and lenient parsing for forward-compatible fields. Invalid settings must produce actionable error messages, not silent failures
- REQ-8: Remote managed settings must be fetched from enterprise policy endpoints, cached locally, and merged at highest precedence. Policy settings support `.d/*.json` directory for split configuration files

## Acceptance Criteria

- [ ] AC-1: Settings merge follows correct precedence — policy overrides flag overrides local overrides project overrides user — verified by creating settings at all 5 levels with conflicting values
- [ ] AC-2: All 11 migrations run in order and produce correct output — verified by snapshot tests with pre-migration fixtures
- [ ] AC-3: GlobalConfig serialization round-trips identically with TypeScript — verified by loading TypeScript-generated `~/.claude.json`
- [ ] AC-4: ProjectConfig loads per-directory — verified by testing with 3 different project directories
- [ ] AC-5: Missing settings files are handled gracefully (defaults applied, no errors) — verified by running with no config files present
- [ ] AC-6: Invalid settings produce clear error messages — verified by testing with malformed JSON
- [ ] AC-7: Policy settings at `/opt/managed-settings.json` take highest precedence — verified by setting conflicting values

## Architecture

```
crates/claude-config/
├── src/
│   ├── lib.rs
│   ├── settings/
│   │   ├── mod.rs          (load_settings: merge 5 sources)
│   │   ├── schema.rs       (Settings struct with 100+ fields, serde)
│   │   ├── merge.rs        (precedence-based merging logic)
│   │   └── validate.rs     (validation and error messages)
│   ├── global.rs           (GlobalConfig: ~/.claude.json)
│   ├── project.rs          (ProjectConfig: .claude.json per-directory)
│   ├── migrations/
│   │   ├── mod.rs          (run_migrations: v1-v11 sequential runner)
│   │   ├── v01_auto_updates.rs
│   │   ├── v02_bypass_permissions.rs
│   │   ├── v03_mcp_servers.rs
│   │   ├── v04_pro_opus_default.rs
│   │   ├── v05_sonnet_1m_45.rs
│   │   ├── v06_legacy_opus.rs
│   │   ├── v07_sonnet_45_46.rs
│   │   ├── v08_opus_1m.rs
│   │   ├── v09_repl_bridge_remote.rs
│   │   ├── v10_auto_mode_opt_in.rs
│   │   └── v11_fennec_opus.rs
│   ├── output_style.rs     (markdown output style loading)
│   ├── managed.rs          (remote managed settings, policy endpoint)
│   └── error.rs            (ConfigError enum)
```

**Dependencies:**
- `serde` + `serde_json` (JSON config files)
- `serde_yml` (if any YAML config is used)
- `dirs` (platform-specific config directories)
- `claude-types` (PermissionSettings, HooksSettings, McpServerConfig)

## Open Questions

None.

## Out of Scope

- CLI flag parsing (lives in `claude-cli`, passes values to config system)
- OAuth configuration (lives in `claude-auth`, uses config for endpoint URLs)
- MCP server management (lives in `claude-mcp`, reads config for server definitions)
- Settings sync to remote (lives in `claude-config/managed.rs` for read, but write-back is separate)
