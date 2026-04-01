# Feature: Task & Agent Orchestration (`claude-tasks` crate)

## Summary
Implement the background task management system that tracks and executes concurrent work items. Tasks come in 7 types: LocalShell (background bash commands), LocalAgent (sub-agent processes), RemoteAgent (CCR-spawned agents), InProcessTeammate (in-process agent contexts), Dream (memory consolidation), LocalWorkflow (workflow script execution), and MonitorMcp (MCP server health monitoring). The coordinator mode enables multi-agent orchestration where a coordinator agent delegates work to specialized worker agents.

## Requirements

- REQ-1: Task state must track 7 task types, each with type-specific state: LocalShell (process handle, stdout/stderr buffers), LocalAgent (agent context, tool set, message history), RemoteAgent (remote session ID, connection state), InProcessTeammate (agent ID, shared state references), Dream (consolidation state), LocalWorkflow (workflow script path, execution state), MonitorMcp (server name, health status)
- REQ-2: Task lifecycle must follow 5 statuses: Pending → Running → Completed/Failed/Killed. Status transitions must be atomic and observable (store subscription pattern from `claude-state`)
- REQ-3: Agent spawning must: (1) register task in AppState, (2) create child CancellationToken linked to parent, (3) build agent-specific system prompt, (4) run `query()` in isolated context (separate tool set, separate state scope), (5) track progress (tokens consumed, tool uses, activities), (6) on completion enqueue notification to UI
- REQ-4: Coordinator mode must implement the multi-agent orchestration pattern: a coordinator agent that only has access to Agent, SendMessage, and TaskStop tools, delegates research/implementation/verification to worker agents, follows a phase-based workflow (research → synthesis → implementation → verification), and generates self-contained worker prompts
- REQ-5: The coordinator system prompt (~370 lines in TypeScript) must be reproduced exactly, covering: role definition (coordinator not executor), available tools, worker capabilities, task workflow phases, writing worker prompts (self-contained, specific), and example session
- REQ-6: Task output retrieval must support: reading stdout/stderr from running shell tasks, reading the final message from completed agent tasks, and streaming progress updates for in-progress tasks
- REQ-7: Task cleanup must handle: killing running processes (SIGTERM then SIGKILL after timeout), cancelling in-progress agent queries via CancellationToken, deregistering teams created during the session, and cleaning up worktree resources

## Acceptance Criteria

- [ ] AC-1: LocalShell task runs a background command and captures output — verified by `sleep 1 && echo done` test
- [ ] AC-2: LocalAgent task executes a sub-query with isolated tool set — verified by spawning an agent that uses FileReadTool
- [ ] AC-3: Task cancellation stops a running agent within 1 second — verified by timing test
- [ ] AC-4: Coordinator mode spawns worker agents and collects results — verified by integration test with mock query
- [ ] AC-5: Task status transitions are atomic — verified by concurrent status reads during transition
- [ ] AC-6: Session cleanup kills all running tasks — verified by starting 5 tasks and calling cleanup
- [ ] AC-7: Coordinator system prompt matches TypeScript byte-for-byte — verified by snapshot comparison

## Architecture

```
crates/claude-tasks/
├── src/
│   ├── lib.rs
│   ├── types.rs            (TaskState 7-variant enum, TaskStatus 5-variant enum)
│   ├── registry.rs         (task storage, lookup, status updates)
│   ├── shell.rs            (LocalShell: process spawn, output capture)
│   ├── agent.rs            (LocalAgent: isolated query context, progress tracking)
│   ├── remote.rs           (RemoteAgent: CCR session management)
│   ├── teammate.rs         (InProcessTeammate: shared-state agent)
│   ├── dream.rs            (Dream: memory consolidation task)
│   ├── workflow.rs         (LocalWorkflow: script execution)
│   ├── monitor.rs          (MonitorMcp: server health checking)
│   ├── coordinator.rs      (coordinator mode: system prompt, agent delegation)
│   ├── cleanup.rs          (SIGTERM/SIGKILL, cancellation, worktree cleanup)
│   └── error.rs            (TaskError enum)
```

**Dependencies:**
- `tokio` (process management, CancellationToken, JoinSet)
- `nix` (Unix signal sending: SIGTERM, SIGKILL)
- `claude-types` (TaskState, Tool trait for coordinator tools)
- `claude-state` (AppState for task registry)

## Open Questions

None.

## Out of Scope

- TaskCreateTool/TaskStopTool/TaskGetTool implementations (live in `claude-tools`)
- Agent tool implementation (lives in `claude-tools`, calls into task spawning)
- Query engine (lives in `claude-core`, used by agent tasks)
