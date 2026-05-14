# RALPLAN: Claude Code Configuration Profile Switcher

## Project Identity

- **Name**: `claude-profile` (binary: `claude-profile`)
- **Type**: Greenfield Rust CLI/TUI tool
- **Repository**: `/Users/sun/Developer/claude-profile/`
- **Spec Source**: `/Users/sun/Developer/.omc/specs/deep-interview-claude-config-switcher.md`

---

## RALPLAN-DR Summary

### Principles

1. **File-level atomicity over process-level tricks**: Switch by writing `settings.json` atomically. Do not attempt process injection, signal hacks, or IPC to force Claude Code runtime reload. The tool operates on files; Claude Code reads them on next start.

2. **Profile-as-source-of-truth**: Each Profile is a standalone, human-editable TOML file. The tool never mutates Profile files during a switch. The active configuration is always the result of merging a Profile into `settings.json`.

3. **Defensive by default**: Every switch operation creates a timestamped backup before mutation. Rollback is always one command away. Validation runs before write, not after.

4. **Progressive interface depth**: CLI for scripting and speed, TUI for discovery and browsing, Hook for in-session triggers. All three share the same core domain logic.

5. **Zero external runtime dependencies**: Single static binary. No Node, no Python, no daemon. Cross-compilation targets macOS, Linux, Windows from the same codebase.

### Decision Drivers

| Rank | Driver | Weight |
|------|--------|--------|
| 1 | **Safety of user data**: `settings.json` contains API keys and complex hook configurations. Corruption or loss is unacceptable. Backup/rollback must be bulletproof. | High |
| 2 | **Cross-platform consistency**: macOS, Linux, and Windows must behave identically for config discovery, XDG paths, and file operations. | High |
| 3 | **Single-pass correctness**: The user explicitly rejected MVP phasing ("全部功能一次性实现"). CLI + TUI + Hook + validation + backup must ship together. | High |

### Viable Options

#### Option A: Direct `settings.json` overwrite with atomic write (SELECTED)

Profile data is merged into a base `settings.json` template, then written atomically via temp-file + rename.

**Pros:**
- Simple, predictable, no hidden state
- Works on all platforms identically
- Easy to debug: the file on disk is the truth
- Backup/rollback is just file copy

**Cons:**
- Requires Claude Code restart to take effect (accepted per spec Round 4)
- Concurrent edits to `settings.json` outside the tool could be overwritten

**Invalidation of alternatives:**
- Option B (inotify/fswatch hot-reload trigger): Claude Code does not expose a reload API. File change events do not trigger config re-read. Attempting to signal the Claude Code process is fragile and platform-specific.
- Option C (symlink swap): Windows requires elevated privileges for symlinks in many configurations. Adds complexity without benefit.
- Option D (overlay merge preserving existing keys): Adds significant complexity for marginal benefit. User's existing workflow is full-file replacements (evidenced by `settings-*.json` files in `~/.claude/`).

#### Option B: Hook-based in-session trigger (ENHANCEMENT ONLY)

A Claude Code Hook (`UserPromptSubmit` or `SessionStart`) detects a "switch" request and shells out to the CLI. The switch still writes `settings.json`; the user is prompted to restart.

**Pros:**
- Enables switching without leaving the Claude Code session
- Leverages existing hook infrastructure the user already has

**Cons:**
- Cannot force reload; still requires restart
- Adds a Node.js hook dependency (the hook itself is a Node script)

**Rationale:** This is not a mutually exclusive alternative but an integration layer on top of Option A. The Hook feature in the spec is implemented as a companion hook script that delegates to the CLI binary.

---

## Requirements Summary

### Functional Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1 | Define multiple Profiles as independent TOML files | Must |
| FR-2 | Scan and list all available Profiles | Must |
| FR-3 | Switch active Profile via CLI command | Must |
| FR-4 | Switch active Profile via TUI interactive menu | Must |
| FR-5 | Switch active Profile via Claude Code Hook trigger | Must |
| FR-6 | Automatic backup of current `settings.json` before switch | Must |
| FR-7 | One-command rollback to previous configuration | Must |
| FR-8 | Validate target Profile before applying (API key format, base_url reachability) | Must |
| FR-9 | Profile storage follows XDG Base Directory specification | Must |
| FR-10 | Cross-platform support (macOS, Linux, Windows) | Must |

### Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-1 | Single static binary, no runtime dependencies |
| NFR-2 | Sub-100ms cold start for CLI commands |
| NFR-3 | TUI renders correctly in any terminal with 80x24 minimum |
| NFR-4 | All file operations are atomic or recoverable |
| NFR-5 | Backup retention policy: keep last 50 backups, max 30 days |

### Domain Model

```
ConfigSwitcher
  profiles_dir: Path        # XDG_CONFIG_HOME/claude-profile/profiles/
  backups_dir: Path         # XDG_DATA_HOME/claude-profile/backups/
  active_profile: Option<String>
  settings_json_path: Path  # CLAUDE_CONFIG_DIR/settings.json

Profile (stored as TOML)
  name: String              # filename stem, e.g. "kimi"
  description: Option<String>
  env: Map<String, String>  # ANTHROPIC_BASE_URL, ANTHROPIC_AUTH_TOKEN, etc.
  model: Option<String>
  effort_level: Option<String>
  hooks: Option<Hooks>      # partial or full hooks config
  # ... other Claude Code settings

APIProvider (validation target)
  base_url: String
  api_key: String
  name: String              # human-readable, e.g. "Kimi Coding"
```

---

## Acceptance Criteria

### CLI

- [ ] `claude-profile list` prints all profiles with active marker, description, and provider name
- [ ] `claude-profile switch <profile>` validates, backs up, applies, and reports success
- [ ] `claude-profile switch <profile>` rejects invalid profiles with specific error message
- [ ] `claude-profile rollback` restores the most recent backup and reports what was restored
- [ ] `claude-profile add <name> --interactive` creates a new profile TOML via prompts
- [ ] `claude-profile remove <name>` deletes a profile file after confirmation
- [ ] `claude-profile validate <profile>` checks profile without applying
- [ ] `claude-profile show <profile>` displays profile contents (with API key masked)

### TUI

- [ ] `claude-profile tui` opens an interactive menu with arrow-key navigation
- [ ] TUI shows profile list with preview pane showing selected profile details
- [ ] Pressing Enter on a profile triggers switch with confirmation dialog
- [ ] TUI handles terminal resize gracefully
- [ ] q/Esc exits TUI cleanly

### Hook Integration

- [ ] Installing the hook adds a `UserPromptSubmit` hook entry to `settings.json`
- [ ] Hook detects natural language switch requests (e.g., "switch to kimi profile")
- [ ] Hook shells out to `claude-profile switch` and reports result
- [ ] Hook installation is idempotent

### Cross-Platform / XDG

- [ ] On macOS/Linux: profiles stored in `~/.config/claude-profile/profiles/`
- [ ] On Windows: profiles stored in `%APPDATA%/claude-profile/profiles/`
- [ ] Backups stored in `~/.local/share/claude-profile/backups/` (macOS/Linux) or `%LOCALAPPDATA%/claude-profile/backups/` (Windows)
- [ ] Respects `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, and `CLAUDE_CONFIG_DIR` env overrides

### Validation

- [ ] API key format validated (prefix checks for known providers: `sk-`, `sk-ant-`, etc.)
- [ ] `base_url` validated as valid URL syntax
- [ ] Optional: HTTP HEAD/GET to `base_url` confirms reachability (with `--no-verify` to skip)
- [ ] TOML schema validated (required fields present, no unknown fields warned)

---

## Implementation Steps

### Phase 1: Project Skeleton & Domain Layer

**Files:**
- `/Users/sun/Developer/claude-profile/Cargo.toml` -- workspace manifest with dependencies
- `/Users/sun/Developer/claude-profile/src/main.rs` -- entry point, CLI/TUI dispatch
- `/Users/sun/Developer/claude-profile/src/lib.rs` -- library root
- `/Users/sun/Developer/claude-profile/src/config.rs` -- XDG path resolution, `ConfigSwitcher` struct
- `/Users/sun/Developer/claude-profile/src/profile.rs` -- `Profile` struct, TOML serialization
- `/Users/sun/Developer/claude-profile/src/error.rs` -- unified error type using `thiserror`

**Dependencies:**
```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
ratatui = "0.29"
crossterm = "0.28"
toml = "0.8"
serde = { version = "1", features = ["derive"] }
dirs = "6"
reqwest = { version = "0.12", features = ["rustls-tls"], default-features = false }
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
thiserror = "2"
anyhow = "1"
chrono = "0.4"
```

**Tasks:**
1. `cargo init --name claude-profile`
2. Implement `ConfigPaths` using `dirs` crate with `CLAUDE_CONFIG_DIR` override
3. Define `Profile` struct with `serde` derive, implement `load()` / `save()` / `list()`
4. Define `ClaudeSettings` struct matching actual `settings.json` schema (from observed files)
5. Implement atomic file write helper (`write_atomic`)

### Phase 2: Core Switch Logic

**Files:**
- `/Users/sun/Developer/claude-profile/src/switcher.rs` -- `ConfigSwitcher` implementation
- `/Users/sun/Developer/claude-profile/src/backup.rs` -- backup creation and rollback
- `/Users/sun/Developer/claude-profile/src/validation.rs` -- profile validation

**Tasks:**
1. Implement `backup_current()` -- copies `settings.json` to `backups/` with timestamp
2. Implement `switch_to(profile_name)` -- validates, backs up, merges Profile into settings template, writes atomically
3. Implement `rollback()` -- restores most recent backup, with safety checks
4. Implement `validate_profile()` -- TOML parse, required fields, URL syntax, API key format
5. Implement optional HTTP validation for `base_url` (async, behind `--verify` flag)

**Merge strategy:**
Profile TOML defines a subset of settings. The switch operation constructs a new `settings.json` by:
1. Reading the current `settings.json` as the base (to preserve non-profile keys like `hooks`, `enabledPlugins`)
2. Overwriting `env`, `model`, `effortLevel` with Profile values
3. Writing the merged result

This preserves user hooks and plugin config while switching the provider-specific fields.

### Phase 3: CLI Interface

**Files:**
- `/Users/sun/Developer/claude-profile/src/cli.rs` -- `clap` derive-based command definitions
- `/Users/sun/Developer/claude-profile/src/commands/` -- per-command handlers

**Commands:**
```
claude-profile list              # List all profiles
claude-profile switch <name>     # Switch to profile
claude-profile rollback          # Rollback to previous
claude-profile add <name>        # Create new profile (interactive or --from-current)
claude-profile remove <name>     # Delete profile
claude-profile validate <name>   # Validate profile without switching
claude-profile show <name>       # Display profile (masked)
claude-profile tui               # Launch TUI
claude-profile hook install      # Install Claude Code hook
claude-profile hook uninstall    # Remove Claude Code hook
```

### Phase 4: TUI Interface

**Files:**
- `/Users/sun/Developer/claude-profile/src/tui/` -- TUI module
- `/Users/sun/Developer/claude-profile/src/tui/app.rs` -- TUI state machine
- `/Users/sun/Developer/claude-profile/src/tui/ui.rs` -- rendering functions
- `/Users/sun/Developer/claude-profile/src/tui/events.rs` -- input handling

**Layout:**
```
+------------------------------------------+
| claude-profile v0.1.0          [q] quit  |
+------------------------------------------+
| Profiles              | Preview          |
| > kimi               | Name: kimi        |
|   glm                | Provider: Kimi    |
|   ali                | Base URL: ...     |
|   opus               | Model: opus[1m]   |
|                      |                  |
|                      | [Enter] Switch    |
|                      | [b] Backup first  |
+------------------------------------------+
| Status: Ready                            |
+------------------------------------------+
```

**Tasks:**
1. Implement list widget with selection state
2. Implement preview pane showing selected profile details
3. Implement switch confirmation dialog
4. Handle terminal resize and graceful exit

### Phase 5: Hook Integration

**Files:**
- `/Users/sun/Developer/claude-profile/src/hook.rs` -- hook install/uninstall logic
- `/Users/sun/Developer/claude-profile/hook/claude-profile-hook.mjs` -- Node.js hook script

**Hook script behavior:**
1. Receives prompt text from Claude Code via stdin
2. Checks for switch intent via regex (e.g., `/switch\s+(?:to\s+)?(\w+)(?:\s+profile)?/i`)
3. If matched, shells out to `claude-profile switch <name>`
4. Returns modified prompt or status message

**Install behavior:**
1. Reads current `settings.json`
2. Adds `UserPromptSubmit` hook entry pointing to hook script
3. Copies hook script to `~/.claude/hooks/claude-profile-hook.mjs`
4. Backs up `settings.json` before modification

### Phase 6: Testing & Polish

**Files:**
- `/Users/sun/Developer/claude-profile/tests/integration.rs` -- integration tests
- `/Users/sun/Developer/claude-profile/tests/fixtures/` -- test fixtures

**Test coverage:**
- Profile TOML parse roundtrip
- Switch operation creates correct backup
- Rollback restores exact previous content
- Validation catches malformed URLs and API keys
- XDG path resolution on all platforms (mocked)
- Atomic write survives interruption

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| `settings.json` schema changes in future Claude Code versions | Medium | High | Store schema version in Profile TOML; warn on unknown fields; maintain mapping layer |
| User has unsaved changes in `settings.json` that get overwritten | Medium | High | Always backup before switch; provide `--dry-run` flag; show diff preview in TUI |
| Concurrent switch operations (race condition) | Low | Medium | Use file locking (`.switch.lock`) with timeout; atomic write prevents corrupted files |
| TUI fails on exotic terminal | Low | Low | Graceful fallback to CLI if TUI init fails; `--no-tui` flag |
| Windows path handling edge cases | Medium | Medium | Use `std::path::PathBuf` throughout; test on Windows CI; use `dirs` crate |
| Hook installation corrupts existing hooks | Low | High | Parse and append to existing hook array, never replace; validate JSON before write |
| API key leaked in backup files | Low | High | Backup files get same permissions as original (0600); mask keys in `show` output |

---

## Verification Steps

### Unit Tests

```bash
cd /Users/sun/Developer/claude-profile
cargo test
```

Expected: all tests pass, including:
- `profile::tests::parse_roundtrip`
- `switcher::tests::backup_and_rollback`
- `validation::tests::invalid_url_rejected`
- `config::tests::xdg_paths_respected`

### Integration Test (Manual)

```bash
# 1. Build
cargo build --release

# 2. Create a test profile
mkdir -p ~/.config/claude-profile/profiles
cat > ~/.config/claude-profile/profiles/test.toml << 'EOF'
name = "test"
description = "Test profile"
[env]
ANTHROPIC_BASE_URL = "https://api.anthropic.com"
ANTHROPIC_AUTH_TOKEN = "sk-test-1234567890"
model = "claude-sonnet-4"
EOF

# 3. List profiles
./target/release/claude-profile list
# Expected: shows "test" profile

# 4. Validate
./target/release/claude-profile validate test
# Expected: success (URL valid, key format ok)

# 5. Switch (dry run)
./target/release/claude-profile switch test --dry-run
# Expected: shows what would change, no file modification

# 6. Switch (for real)
./target/release/claude-profile switch test
# Expected: backup created, settings.json updated

# 7. Verify backup
ls ~/.local/share/claude-profile/backups/
# Expected: timestamped backup file exists

# 8. Rollback
./target/release/claude-profile rollback
# Expected: settings.json restored to pre-switch state

# 9. TUI smoke test
./target/release/claude-profile tui
# Expected: opens interactive menu, q exits cleanly
```

### Cross-Platform Verification

- [ ] macOS: native build and test
- [ ] Linux: `cargo test` in Docker (`rust:slim`)
- [ ] Windows: `cargo build --target x86_64-pc-windows-gnu` (cross-compile from macOS)

### Hook Verification

```bash
# Install hook
./target/release/claude-profile hook install

# Verify hook entry in settings.json
grep -A2 "claude-profile-hook" ~/.claude/settings.json

# Test trigger (in Claude Code session, type "switch to test profile")
# Expected: hook detects intent, runs switch, reports result
```

---

## File Inventory

| File | Purpose | Phase |
|------|---------|-------|
| `Cargo.toml` | Dependencies and metadata | 1 |
| `src/main.rs` | Entry point | 1 |
| `src/lib.rs` | Module declarations | 1 |
| `src/error.rs` | Error types | 1 |
| `src/config.rs` | Path resolution, ConfigSwitcher | 1 |
| `src/profile.rs` | Profile domain model | 1 |
| `src/settings.rs` | Claude settings.json model | 2 |
| `src/switcher.rs` | Switch/merge logic | 2 |
| `src/backup.rs` | Backup/rollback | 2 |
| `src/validation.rs` | Profile validation | 2 |
| `src/cli.rs` | Clap command definitions | 3 |
| `src/commands/mod.rs` | Command dispatcher | 3 |
| `src/commands/list.rs` | List command | 3 |
| `src/commands/switch.rs` | Switch command | 3 |
| `src/commands/rollback.rs` | Rollback command | 3 |
| `src/commands/add.rs` | Add command | 3 |
| `src/commands/remove.rs` | Remove command | 3 |
| `src/commands/validate.rs` | Validate command | 3 |
| `src/commands/show.rs` | Show command | 3 |
| `src/commands/hook.rs` | Hook install/uninstall | 5 |
| `src/tui/mod.rs` | TUI module root | 4 |
| `src/tui/app.rs` | TUI state | 4 |
| `src/tui/ui.rs` | TUI rendering | 4 |
| `src/tui/events.rs` | TUI input | 4 |
| `src/hook.rs` | Hook management | 5 |
| `hook/claude-profile-hook.mjs` | Node.js hook script | 5 |
| `tests/integration.rs` | Integration tests | 6 |
| `tests/fixtures/*.toml` | Test fixtures | 6 |

---

## Open Questions

1. **Tool name**: Spec notes "cc-switch" is taken. `claude-profile` is proposed. Confirm before proceeding.
2. **Settings merge granularity**: Should Profile TOML support partial overrides (e.g., only `env.ANTHROPIC_BASE_URL`) or require full env block? Recommend: partial overrides with explicit merge rules.
3. **Backup retention**: Spec says automatic backup. Should old backups be pruned? Recommend: keep 50 most recent, prune on every switch.
4. **HTTP validation default**: Should `base_url` reachability be checked by default? Recommend: yes, with `--no-verify` to skip.
5. **Hook trigger phrases**: What natural language patterns should the hook recognize? Recommend: "switch to <profile>", "use <profile> profile", "activate <profile>".

---

## Estimated Effort

| Phase | Estimated Hours | Complexity |
|-------|----------------|------------|
| 1: Skeleton & Domain | 4h | Low |
| 2: Core Switch Logic | 6h | Medium |
| 3: CLI | 4h | Low |
| 4: TUI | 8h | Medium |
| 5: Hook Integration | 4h | Medium |
| 6: Testing & Polish | 6h | Medium |
| **Total** | **32h** | |

---

*Plan generated: 2026-05-12*
*Based on: deep-interview-claude-config-switcher.md (11 rounds, 19.0% ambiguity, PASSED)*
