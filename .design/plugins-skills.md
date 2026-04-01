# Feature: Plugin & Skill Systems (`claude-plugins` + `claude-skills` crates)

## Summary
Implement the extension systems that allow Claude Code to be customized with plugins (packages of commands, agents, skills, hooks, MCP servers, and output styles from marketplaces or local directories) and skills (markdown files with YAML frontmatter that define reusable workflows). Plugins load from 3 sources (builtin, marketplace, inline), skills load from 5 sources (bundled, managed, plugin, MCP, user/project). The system includes manifest validation, marketplace integration with enterprise allowlist/denylist, frontmatter parsing for skill metadata, conditional skill activation via path patterns, and 17 bundled skills.

## Requirements

- REQ-1: Plugin loading must support 3 sources: (1) builtin plugins registered via `register_builtin_plugin(definition)` with fields name, description, version, skills, hooks, mcp_servers, default_enabled, is_available; (2) marketplace plugins installed from `github`/`git`/`url`/`settings` sources with name@marketplace ID format; (3) inline plugins from `--plugin-dir` CLI flag (session-only). Enabled state persisted in `settings.enabled_plugins: HashMap<String, EnabledState>` where EnabledState is `bool | Vec<String> | None`
- REQ-2: Plugin manifest must validate: name, description, version. Loaded plugins carry: name, manifest, path, source, repository, enabled, is_builtin, sha, commands_path(s), agents_path(s), skills_path(s), output_styles_path(s), hooks_config, mcp_servers, lsp_servers, settings. PluginError must be a 20+ variant enum covering: path-not-found, git-auth-failed, manifest-parse-error, plugin-not-found, mcp-config-invalid, lsp-server-crashed, etc.
- REQ-3: Marketplace integration must support: enterprise `strict_known_marketplaces` allowlist and `blocked_marketplaces` denylist (checked before download), filesystem isolation for downloaded plugins, auto-update capability, and plugin component types: commands, agents, skills, hooks, output-styles
- REQ-4: Skill loading must search 5 sources in order: (1) managed skills from policy settings, (2) user skills from `~/.claude/skills/`, (3) project skills from `.claude/skills/` (+ parent directories), (4) additional directories from `--add-dir` flag, (5) legacy commands from `.claude/commands/`. Deduplicate by file identity (realpath). Separate conditional skills (with `paths` field) for lazy activation
- REQ-5: Skill frontmatter parsing must extract fields: name, description, when-to-use, model, allowed-tools, user-invocable, paths (conditional activation patterns), hooks, version, agent, effort, shell. The `paths` field triggers conditional activation — skill only loads when working in matching file paths (glob patterns with `/` separators, `/**` suffix)
- REQ-6: All 17 bundled skills must be embedded: batch, claudeApi (+content), claudeInChrome, debug, dream (KAIROS_DREAM), hunter (REVIEW_ARTIFACT), keybindings, loop, loremIpsum, remember, runSkillGenerator (RUN_SKILL_GENERATOR), scheduleRemoteAgents, simplify, skillify, stuck, updateConfig, verify (+content). Feature-gated skills must only register when their feature is enabled
- REQ-7: Skill commands must produce `PromptCommand` type with: type='prompt', name, description, source (LoadedFrom: commands_DEPRECATED, skills, plugin, managed, bundled, mcp), allowed_tools, when_to_use, user_invocable, paths, hooks, effort, context (inline or fork), progress_message, content_length, get_prompt function
- REQ-8: Hook validation for skills must use the same schema as settings hooks — skill frontmatter `hooks` field is parsed via `HooksSchema().safeParse()`

## Acceptance Criteria

- [ ] AC-1: Builtin plugin registration makes skills/hooks/mcp_servers available — verified by registering a test plugin and checking command availability
- [ ] AC-2: Marketplace plugin install downloads, validates manifest, and registers components — verified by mock marketplace server test
- [ ] AC-3: Enterprise denylist blocks plugin installation before download — verified by unit test
- [ ] AC-4: Skill loading from all 5 sources produces correct deduplicated list — verified by creating skills in each location and checking merged output
- [ ] AC-5: Conditional skills only activate when `paths` patterns match cwd — verified by changing cwd and checking active skills
- [ ] AC-6: All 17 bundled skills register successfully with correct metadata — verified by iteration test
- [ ] AC-7: Skill frontmatter parsing handles all fields including hooks validation — verified by fixture tests with valid/invalid frontmatter
- [ ] AC-8: Inline plugins from `--plugin-dir` are session-only and don't persist — verified by checking settings after session end

## Architecture

```
crates/claude-plugins/
├── src/
│   ├── lib.rs              (PluginLoader: load_all, install, uninstall)
│   ├── builtin.rs          (register_builtin_plugin, builtin registry)
│   ├── marketplace.rs      (marketplace sources, download, allowlist/denylist)
│   ├── manifest.rs         (PluginManifest validation)
│   ├── loader.rs           (LoadedPlugin construction from filesystem)
│   └── error.rs            (PluginError 20+ variant enum)

crates/claude-skills/
├── src/
│   ├── lib.rs              (get_skill_dir_commands: merge 5 sources)
│   ├── frontmatter.rs      (YAML frontmatter parser for skill files)
│   ├── loader.rs           (load skills from directory, deduplicate by realpath)
│   ├── conditional.rs      (path-based conditional activation)
│   ├── bundled/
│   │   ├── mod.rs          (init_bundled_skills, registry)
│   │   ├── batch.rs
│   │   ├── claude_api.rs   (+ claude_api_content.rs)
│   │   ├── claude_in_chrome.rs
│   │   ├── debug.rs
│   │   ├── dream.rs        (#[cfg(feature = "kairos-dream")])
│   │   ├── hunter.rs       (#[cfg(feature = "review-artifact")])
│   │   ├── keybindings.rs
│   │   ├── loop_skill.rs
│   │   ├── lorem_ipsum.rs
│   │   ├── remember.rs
│   │   ├── run_skill_generator.rs (#[cfg(feature = "run-skill-generator")])
│   │   ├── schedule.rs
│   │   ├── simplify.rs
│   │   ├── skillify.rs
│   │   ├── stuck.rs
│   │   ├── update_config.rs
│   │   └── verify.rs       (+ verify_content.rs)
│   └── error.rs
```

**Dependencies:**
- `serde_yml` (YAML frontmatter parsing)
- `pulldown-cmark` (markdown content processing)
- `globset` (path pattern matching for conditional activation)
- `claude-types` (Command, PromptCommand, HooksSettings)

## Open Questions

None.

## Out of Scope

- SkillTool implementation (lives in `claude-tools`)
- Hook execution engine (lives in `claude-hook-engine`)
- MCP server management for plugin-provided servers (lives in `claude-mcp`)
- Output style loading (lives in `claude-config`)
