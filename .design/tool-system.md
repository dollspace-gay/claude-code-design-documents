# Feature: Tool System (`claude-tools` crate)

## Summary
Implement all ~54 tool implementations as the `claude-tools` crate. Each tool implements the `Tool` trait from `claude-types` using the `build_tool!` macro pattern (Rust equivalent of TypeScript's `buildTool()` helper). The BashTool has the most complex implementation with 23 security check IDs, command classification (search/read/list/write), wildcard permission matching, and sandboxing. Tools are dynamically registered in a registry with feature-flag gating matching the TypeScript `getAllBaseTools()` function.

## Requirements

- REQ-1: Every tool from the TypeScript `getAllBaseTools()` registry must have a Rust implementation. The always-included set: AgentTool, TaskOutputTool, BashTool, ExitPlanModeTool, FileReadTool, FileEditTool, FileWriteTool, NotebookEditTool, WebFetchTool, TodoWriteTool, WebSearchTool, TaskStopTool, AskUserQuestionTool, SkillTool, EnterPlanModeTool, BriefTool, ListMcpResourcesTool, ReadMcpResourceTool, GlobTool, GrepTool. Feature-gated tools: ConfigTool (internal), LSPTool (ENABLE_LSP_TOOL env), TaskCreateTool/TaskGetTool/TaskUpdateTool/TaskListTool (todoV2), EnterWorktreeTool/ExitWorktreeTool (worktreeMode), SendMessageTool (lazy), TeamCreateTool/TeamDeleteTool (agent swarms), SleepTool (PROACTIVE/KAIROS), ScheduleCronTool/CronDeleteTool/CronListTool (AGENT_TRIGGERS), RemoteTriggerTool (AGENT_TRIGGERS_REMOTE), MonitorTool (MONITOR_TOOL), SendUserFileTool (KAIROS), PushNotificationTool (KAIROS_PUSH_NOTIFICATION), SubscribePRTool (KAIROS_GITHUB_WEBHOOKS), PowerShellTool (isPowerShellToolEnabled), SnipTool (HISTORY_SNIP), ToolSearchTool (tool search optimization), WebBrowserTool (WEB_BROWSER_TOOL), ListPeersTool (UDS_INBOX), WorkflowTool (WORKFLOW_SCRIPTS), REPLTool (internal), SuggestBackgroundPRTool (internal), VerifyPlanExecutionTool (VERIFY_PLAN env)
- REQ-2: The `build_tool!` macro must provide defaults matching TypeScript's `TOOL_DEFAULTS`: `is_enabled() → true`, `is_concurrency_safe() → false`, `is_read_only() → false`, `is_destructive() → false`, `check_permissions() → Allow`, `to_auto_classifier_input() → ""`, `user_facing_name() → tool.name`
- REQ-3: BashTool security analysis must implement all 23 security check IDs from `bashSecurity.ts`: INCOMPLETE_COMMANDS (1), JQ_SYSTEM_FUNCTION (2), JQ_FILE_ARGUMENTS (3), OBFUSCATED_FLAGS (4), SHELL_METACHARACTERS (5), DANGEROUS_VARIABLES (6), NEWLINES (7), DANGEROUS_PATTERNS_COMMAND_SUBSTITUTION (8), DANGEROUS_PATTERNS_INPUT_REDIRECTION (9), DANGEROUS_PATTERNS_OUTPUT_REDIRECTION (10), IFS_INJECTION (11), GIT_COMMIT_SUBSTITUTION (12), PROC_ENVIRON_ACCESS (13), MALFORMED_TOKEN_INJECTION (14), BACKSLASH_ESCAPED_WHITESPACE (15), BRACE_EXPANSION (16), CONTROL_CHARACTERS (17), UNICODE_WHITESPACE (18), MID_WORD_HASH (19), ZSH_DANGEROUS_COMMANDS (20), BACKSLASH_ESCAPED_OPERATORS (21), COMMENT_QUOTE_DESYNC (22), QUOTED_NEWLINE (23)
- REQ-4: BashTool command classification must categorize commands into: BASH_SEARCH_COMMANDS (find, grep, rg, ag, ack, locate, which, whereis), BASH_READ_COMMANDS (cat, head, tail, less, more, jq, awk, cut, sort, uniq, tr), BASH_LIST_COMMANDS (ls, tree, du), BASH_SEMANTIC_NEUTRAL_COMMANDS (echo, printf, true, false, :). The `is_search_or_read_command()` method must return `{is_search, is_read, is_list}` based on these classifications
- REQ-5: BashTool input schema must accept: command (required), timeout (optional, milliseconds), description (optional), run_in_background (optional bool), dangerously_disable_sandbox (optional bool). Output must include: stdout, stderr, raw_output_path, interrupted, is_image, background_task_id, backgrounded_by_user, assistant_auto_backgrounded, return_code_interpretation, structured_content, persisted_output_path, persisted_output_size
- REQ-6: BashTool permission matching must support wildcard patterns via `match_wildcard_pattern(pattern, command)` and prefix extraction via `permission_rule_extract_prefix()`. Permission rules evaluate to allow/ask/deny with decision reasons
- REQ-7: FileEditTool must implement: file existence validation, old_string uniqueness check (reject if not unique unless replace_all=true), content replacement, and output with file_path, content, success, error. `max_result_size_chars = 100_000`
- REQ-8: AgentTool must support: single-agent spawning (description + prompt + subagent_type), team spawning (when team_name and name provided), background execution (run_in_background flag), isolation mode (worktree), and progress reporting. Output status enum: Completed, AsyncLaunched, TeammateSpawned, RemoteLaunched
- REQ-9: WebFetchTool must implement: URL fetching with reqwest, preapproved host bypass for permission checks, redirect handling (return message instead of following), output with bytes, status_code, status_text, result, duration_ms, url
- REQ-10: The tool registry function `get_all_base_tools()` must return a `Vec<Arc<dyn Tool>>` with tools conditionally included based on feature flags and runtime environment checks (env vars, settings), matching the exact conditional logic in TypeScript's `getAllBaseTools()`
- REQ-11: Git operation tracking (from `src/tools/shared/gitOperationTracking.ts`) must be implemented as shared infrastructure: detect commit (sha + kind: committed/amended/cherry-picked), push (branch), branch operations (merged/rebased), and PR operations (created/edited/merged/commented/closed/ready) from bash command + output

## Acceptance Criteria

- [ ] AC-1: `get_all_base_tools()` with all features enabled returns the same tool name set as TypeScript's `getAllBaseTools()` — verified by comparison test
- [ ] AC-2: BashTool rejects all 23 security check patterns — verified by test cases for each check ID
- [ ] AC-3: BashTool command classification correctly categorizes 30+ commands across search/read/list/neutral/write — verified by table-driven tests
- [ ] AC-4: BashTool permission wildcard matching produces identical results to TypeScript — verified by 50+ pattern/command test pairs extracted from TypeScript tests
- [ ] AC-5: FileEditTool rejects non-unique old_string when replace_all=false — verified by unit test
- [ ] AC-6: AgentTool spawns a sub-agent that executes tools independently — verified by integration test with mock query engine
- [ ] AC-7: WebFetchTool handles redirect responses without following — verified by mock HTTP server test
- [ ] AC-8: Each feature flag correctly includes/excludes its gated tools — verified by testing with each flag combination
- [ ] AC-9: Git operation detection correctly extracts commit SHA, branch name, PR number from real git command outputs — verified by fixture tests from captured outputs
- [ ] AC-10: All tools that TypeScript marks `isConcurrencySafe: true` are also marked concurrency-safe in Rust — verified by assertion test

## Architecture

**File structure:**
```
crates/claude-tools/
├── src/
│   ├── lib.rs              (re-exports, get_all_base_tools() registry)
│   ├── macros.rs           (build_tool! macro)
│   ├── bash/
│   │   ├── mod.rs          (BashTool implementation)
│   │   ├── security.rs     (23 security checks, ValidationContext)
│   │   ├── permissions.rs  (wildcard matching, permission evaluation)
│   │   └── classify.rs     (search/read/list/neutral classification)
│   ├── file/
│   │   ├── read.rs         (FileReadTool)
│   │   ├── edit.rs         (FileEditTool)
│   │   ├── write.rs        (FileWriteTool)
│   │   └── notebook.rs     (NotebookEditTool)
│   ├── search/
│   │   ├── glob.rs         (GlobTool — uses globset + ignore)
│   │   └── grep.rs         (GrepTool — uses grep-regex + grep-searcher)
│   ├── web/
│   │   ├── fetch.rs        (WebFetchTool)
│   │   └── search.rs       (WebSearchTool)
│   ├── agent/
│   │   ├── agent.rs        (AgentTool — sub-agent spawning)
│   │   ├── send_message.rs (SendMessageTool — inter-agent messaging)
│   │   ├── team_create.rs  (TeamCreateTool)
│   │   └── team_delete.rs  (TeamDeleteTool)
│   ├── task/
│   │   ├── create.rs       (TaskCreateTool)
│   │   ├── get.rs          (TaskGetTool)
│   │   ├── update.rs       (TaskUpdateTool)
│   │   ├── list.rs         (TaskListTool)
│   │   ├── stop.rs         (TaskStopTool)
│   │   ├── output.rs       (TaskOutputTool)
│   │   └── todo_write.rs   (TodoWriteTool)
│   ├── planning/
│   │   ├── enter_plan.rs   (EnterPlanModeTool)
│   │   └── exit_plan.rs    (ExitPlanModeTool)
│   ├── worktree/
│   │   ├── enter.rs        (EnterWorktreeTool)
│   │   └── exit.rs         (ExitWorktreeTool)
│   ├── scheduling/
│   │   ├── cron_create.rs  (ScheduleCronTool)
│   │   ├── cron_delete.rs  (CronDeleteTool)
│   │   ├── cron_list.rs    (CronListTool)
│   │   └── remote.rs       (RemoteTriggerTool)
│   ├── mcp/
│   │   ├── mcp_tool.rs     (MCPTool wrapper)
│   │   ├── list_resources.rs (ListMcpResourcesTool)
│   │   ├── read_resource.rs  (ReadMcpResourceTool)
│   │   ├── auth.rs         (McpAuthTool)
│   │   └── tool_search.rs  (ToolSearchTool)
│   ├── user/
│   │   ├── ask_question.rs (AskUserQuestionTool)
│   │   └── skill.rs        (SkillTool)
│   ├── lsp.rs              (LSPTool — goToDefinition, findReferences, hover, symbols)
│   ├── config.rs           (ConfigTool — get/set settings)
│   ├── brief.rs            (BriefTool — generate summaries)
│   ├── sleep.rs            (SleepTool)
│   ├── powershell.rs       (PowerShellTool — Windows)
│   └── shared/
│       └── git_tracking.rs (git operation detection: commit, push, branch, PR)
```

**Common tool implementation pattern:**
```rust
pub fn bash_tool() -> impl Tool {
    build_tool! {
        name: "Bash",
        max_result_size_chars: 100_000,
        input_schema: BashToolInput::schema(),
        
        is_concurrency_safe: |input| { /* check if read-only command */ },
        is_read_only: |input| { is_search_or_read_bash_command(&input.command).is_read },
        
        check_permissions: |input, ctx| {
            let analysis = analyze_bash_command(&input.command);
            check_bash_permissions(&input.command, &analysis, &ctx.tool_permission_context)
        },
        
        call: |input, ctx, can_use_tool, parent_msg, on_progress| {
            // Execute via tokio::process::Command
            // Stream stdout/stderr
            // Handle timeout, background, sandboxing
        },
    }
}
```

**Dependencies:**
- `claude-types` (Tool trait, PermissionResult, etc.)
- `claude-state` (SessionState for tool context)
- `claude-mcp` (MCP client for MCP wrapper tools)
- `tokio` (process::Command for BashTool, JoinSet for parallel)
- `globset` + `ignore` (GlobTool)
- `grep-regex` + `grep-searcher` (GrepTool)
- `reqwest` (WebFetchTool)
- `regex` (bash security analysis patterns)

## Open Questions

### Q1: Tree-sitter for bash security analysis
**Decision**: Use `tree-sitter` with `tree-sitter-bash` grammar. Bash security analysis is safety-critical — regex-based parsing misses nested quoting, command substitution, and heredoc edge cases that tree-sitter handles correctly. The C library dependency is acceptable since the binary is already statically linked. The TypeScript source uses tree-sitter for this purpose, and matching its parsing accuracy is essential for security parity.

## Out of Scope

- Tool trait definition (lives in `claude-types`)
- Tool execution pipeline / tool loop (lives in `claude-core`)
- Hook execution around tool calls (lives in `claude-hook-engine`)
- UI rendering of tool results (lives in `claude-tui`)
- Permission mode management (lives in `claude-state`)
