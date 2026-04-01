# Feature: Platform Integration Services

## Summary
Implement platform-specific integration services: native installer (binary download, version locking, package manager detection — ~2K LOC), sleep prevention (macOS caffeinate — 165 LOC), deep linking (claude:// protocol handler — ~300 LOC), session teleportation (environment selection + git bundle transfer — ~400 LOC), and DXT utilities (ZIP bundling — ~100 LOC). These are scattered utility subsystems that ensure Claude Code integrates correctly with the host operating system.

## Requirements

### Native Installer (`claude-utils`)
- REQ-1: Binary download must support two sources: Artifactory/npm (ANT, with `npm ci` + integrity verification) and GCS binary repository (external, with SHA256 checksum verification). Stall timeout detection at 60s without data, 3 retry attempts. Platform detection for binary names
- REQ-2: PID-based version locking must implement: atomic write (temp file + rename), PID reuse mitigation (command validation via `/proc/{pid}/cmdline`), 2-hour stale timeout fallback, `is_process_running()` via signal 0, `try_acquire_lock()` with CAS semantics, legacy `proper-lockfile` cleanup
- REQ-3: Package manager detection must identify 9 sources: homebrew (Caskroom path), winget (WinGet Packages path), mise (.mise/installs), asdf (.asdf/installs), pacman (pacman -Qo, Arch only), deb (dpkg -S, Debian only), rpm (rpm -qf, Fedora/RHEL/SUSE only), apk (apk info, Alpine only), unknown. Memoized, with `/etc/os-release` parsing for distro family detection

### Sleep Prevention (`claude-utils`)
- REQ-4: macOS-only sleep prevention via `caffeinate` process: reference counting for nested calls, 5-minute timeout with 4-minute restart interval, `unref()` to avoid keeping process alive, SIGKILL for immediate termination, cleanup on process exit

### Deep Linking (`claude-utils`)
- REQ-5: `claude://` protocol handler must support: URI parsing (`parse_deep_link`), OS protocol registration (`register_protocol`), terminal preference detection, terminal launcher (open Claude Code in user's preferred terminal), and onboarding banner display

### Session Teleportation (`claude-utils`)
- REQ-6: Session teleportation must support: environment selection picker, environment definitions, teleport API client for session transfer, git bundle creation and transfer for repository context

### DXT Utilities (`claude-utils`)
- REQ-7: DXT (likely Desktop Extension) utilities must support: ZIP creation/extraction for bundling, helper functions for extension packaging

## Acceptance Criteria

- [ ] AC-1: Binary download with checksum verification succeeds — verified by mock GCS test
- [ ] AC-2: PID lock prevents concurrent installations — verified by race condition test
- [ ] AC-3: Package manager detection correctly identifies homebrew/apt/pacman — verified by path mock tests
- [ ] AC-4: Sleep prevention reference counting handles nested start/stop — verified by 5 nested calls
- [ ] AC-5: Deep link parsing extracts valid URI components — verified by fixture tests
- [ ] AC-6: Git bundle creation produces valid bundle — verified by `git bundle verify`

## Architecture

All modules live in `claude-utils` as utility subsystems:

```
crates/claude-utils/src/
├── installer/
│   ├── download.rs     (Artifactory + GCS download, checksum)
│   ├── lock.rs         (PID-based version locking)
│   ├── package_manager.rs (9-source detection)
│   └── cleanup.rs      (old version removal)
├── platform/
│   ├── sleep.rs        (caffeinate wrapper, macOS only)
│   └── deep_link.rs    (claude:// protocol, terminal launcher)
├── teleport/
│   ├── api.rs          (teleport API client)
│   ├── environments.rs (environment definitions)
│   └── git_bundle.rs   (git bundle creation/transfer)
└── dxt/
    ├── helpers.rs
    └── zip.rs          (ZIP create/extract)
```

## Open Questions

None.

## Out of Scope

- Auto-update orchestration (lives in `claude-cli` startup)
- Version display (lives in `claude-commands`)
