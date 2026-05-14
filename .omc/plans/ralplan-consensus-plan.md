# RALPLAN: Claude Code Configuration Profile Switcher (Revised)

## Project Identity

- **Name**: `claude-profile` (binary: `claude-profile`)
- **Type**: Greenfield Rust CLI/TUI tool
- **Repository**: `/Users/sun/Developer/claude-profile/`
- **Spec Source**: `/Users/sun/Developer/.omc/specs/deep-interview-claude-config-switcher.md`

---

## RALPLAN-DR Summary (Revised)

### Principles

1. **File-level atomicity over process-level tricks**: Switch by writing `settings.json` atomically via temp-file + rename. Do not attempt process injection, signal hacks, or IPC to force Claude Code runtime reload. The tool operates on files; Claude Code reads them on next start.

2. **Profile defines the desired configuration. The tool applies it atomically. Full replacement is default; selective merge is opt-in**: Each Profile is a standalone, human-editable TOML file that can express the complete desired `settings.json` state via a `[settings]` section. The tool never mutates Profile files during a switch. Full-file replacement is the default strategy because the user's existing `settings-*.json` files are complete, standalone configurations. Merge mode (if ever implemented) would be a v2 enhancement.

3. **Defensive by default**: Every switch operation creates a timestamped backup before mutation. Rollback is always one command away. Validation runs before write, not after. A dirty check compares current `settings.json` against the last backup and warns if manual edits are detected, requiring `--force` to proceed.

4. **Progressive interface depth**: CLI for scripting and speed, TUI for discovery and browsing, Hook for in-session awareness. All three share the same core domain logic.

5. **Zero external runtime dependencies for the core CLI binary. Hook integration leverages the Node.js runtime already required by Claude Code**: The CLI binary is a single static executable with no Node, Python, or daemon dependencies. The optional Hook companion feature requires Node.js, which is already a dependency of Claude Code itself.

### Decision Drivers

| Rank | Driver | Weight |
|------|--------|--------|
| 1 | **Safety of user data**: `settings.json` contains API keys and complex hook configurations. Corruption or loss is unacceptable. Backup/rollback must be bulletproof. | High |
| 2 | **Cross-platform consistency**: macOS, Linux, and Windows must behave identically for config discovery, XDG paths, and file operations. | High |
| 3 | **Single-pass correctness**: The user explicitly rejected MVP phasing ("全部功能一次性实现"). CLI + TUI + Hook + validation + backup must ship together. | High |
| 4 | **Schema fidelity**: The Profile TOML must map 1:1 to `settings.json` keys to prevent data loss. Fields like `enabledPlugins`, `extraKnownMarketplaces`, `statusLine`, `attribution`, `language` must be preserved. | High |
| 5 | **Honest limitations**: The Hook does not promise in-session switching. It informs the user that a restart is required. False expectations create more friction than explicit restart instructions. | Medium |

### Viable Options

#### Option A: Direct `settings.json` overwrite with atomic write (SELECTED)

Profile data is written as a complete `settings.json` replacement via temp-file + rename. The Profile TOML uses a `[settings]` section that maps 1:1 to `settings.json` keys.

**Pros:**
- Simple, predictable, no hidden state
- Works on all platforms identically
- Easy to debug: the file on disk is the truth
- Backup/rollback is just file copy
- No configuration drift across successive switches

**Cons:**
- Requires Claude Code restart to take effect (accepted per spec Round 4)
- Concurrent edits to `settings.json` outside the tool could be overwritten (mitigated by dirty check)

**Invalidation of alternatives:**
- Option B (inotify/fswatch hot-reload trigger): Claude Code does not expose a reload API. File change events do not trigger config re-read. Attempting to signal the Claude Code process is fragile and platform-specific.
- Option C (symlink swap): Windows requires elevated privileges for symlinks in many configurations. Adds complexity without benefit.
- Option D (overlay merge preserving existing keys): Adds significant complexity for marginal benefit. User's existing workflow is full-file replacements (evidenced by `settings-*.json` files in `~/.claude/`). Merge as default would cause configuration drift. Relegated to potential v2 enhancement.

#### Option B: Hook-based in-session trigger (ENHANCEMENT ONLY)

A Claude Code Hook (`UserPromptSubmit`) detects a "switch" request and responds with informational instructions. It does NOT modify `settings.json` in-session.

**Pros:**
- Enables awareness of switch capability without leaving the Claude Code session
- Leverages existing hook infrastructure the user already has
- Avoids race conditions with hook reloading

**Cons:**
- Cannot force reload; still requires restart
- Adds a Node.js hook dependency (the hook itself is a Node script)
- The hook entry must be preserved across all Profile switches

**Rationale:** This is not a mutually exclusive alternative but an integration layer on top of Option A. The Hook feature in the spec is implemented as a companion hook script that informs the user to run the CLI and restart Claude Code.

---

## Requirements Summary

### Functional Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1 | Define multiple Profiles as independent TOML files | Must |
| FR-2 | Scan and list all available Profiles | Must |
| FR-3 | Switch active Profile via CLI command | Must |
| FR-4 | Switch active Profile via TUI interactive menu | Must |
| FR-5 | Claude Code Hook integration (informational, not operational) | Must |
| FR-6 | Automatic backup of current `settings.json` before switch | Must |
| FR-7 | One-command rollback to previous configuration | Must |
| FR-8 | Validate target Profile before applying (API key format, base_url syntax) | Must |
| FR-9 | Profile storage follows XDG Base Directory specification | Must |
| FR-10 | Cross-platform support (macOS, Linux, Windows) | Must |
| FR-11 | Import existing `settings-*.json` files into Profile TOML format | Must |
| FR-12 | Track active profile via marker file | Must |

### Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-1 | Single static binary, no runtime dependencies for core CLI |
| NFR-2 | Sub-100ms cold start for CLI commands (verified by benchmark) |
| NFR-3 | TUI renders correctly in any terminal with 80x24 minimum |
| NFR-4 | All file operations are atomic or recoverable |
| NFR-5 | Backup retention: keep last N backups (default 50), configurable via `CLAUDE_PROFILE_BACKUP_COUNT` |
| NFR-6 | Binary size target: under 5MB for default build (no TUI, no HTTP validate) |

### Domain Model

```
ConfigSwitcher
  profiles_dir: Path        # XDG_CONFIG_HOME/claude-profile/profiles/
  backups_dir: Path         # XDG_DATA_HOME/claude-profile/backups/
  active_profile_file: Path # XDG_DATA_HOME/claude-profile/active_profile
  settings_json_path: Path  # CLAUDE_CONFIG_DIR/settings.json

Profile (stored as TOML)
  name: String              # filename stem, e.g. "kimi"
  description: Option<String>
  [settings]                # maps 1:1 to settings.json keys
    env: Map<String, String>
    model: Option<String>
    effortLevel: Option<String>
    hooks: Option<serde_json::Value>
    enabledPlugins: Option<HashMap<String, bool>>
    extraKnownMarketplaces: Option<Vec<Marketplace>>
    statusLine: Option<StatusLine>
    attribution: Option<Attribution>
    language: Option<String>
    skipDangerousModePermissionPrompt: Option<bool>
    # ... etc
    #[serde(flatten)]
    extra: HashMap<String, serde_json::Value>

APIProvider (validation target)
  base_url: String
  api_key: String
  name: String              # human-readable, e.g. "Kimi Coding"
```

---

## Acceptance Criteria

### CLI

- [ ] `claude-profile list` prints all profiles with active marker (asterisk), description, and provider name
- [ ] `claude-profile list` reads `active_profile` marker file to determine active profile; if marker missing, shows no active marker
- [ ] `claude-profile switch <profile>` validates target profile, creates backup, writes `settings.json` atomically, updates `active_profile` marker, prints success message including backup path
- [ ] `claude-profile switch <profile>` exits with code 1 and prints specific error if profile is invalid (malformed TOML, missing required fields, invalid URL syntax, unknown API key prefix)
- [ ] `claude-profile switch <profile>` warns and requires `--force` if `settings.json` has been modified since last backup (dirty check)
- [ ] `claude-profile switch <profile> --dry-run` prints the full `settings.json` that would be written, the backup path that would be created, and exits without modifying any files
- [ ] `claude-profile switch <profile> --no-verify` skips validation and applies profile immediately
- [ ] `claude-profile rollback` restores the most recent backup to `settings.json`, updates `active_profile` marker to previous profile name (if known from backup metadata), and prints what was restored
- [ ] `claude-profile rollback` exits with code 1 and prints "no backups found" if backup directory is empty
- [ ] `claude-profile import <path> --name <name>` converts a `settings-*.json` file to Profile TOML format, preserving all keys via `[settings]` section, and writes to profiles directory
- [ ] `claude-profile import <path> --name <name>` exits with code 1 if the JSON file cannot be parsed
- [ ] `claude-profile add <name> --interactive` creates a new profile TOML via prompts for name, description, base URL, API key, and model
- [ ] `claude-profile remove <name>` deletes a profile file after Y/n confirmation; exits with code 1 if profile is currently active
- [ ] `claude-profile validate <profile>` checks profile TOML parse, required fields, URL syntax, API key format; exits 0 on success, 1 on failure with specific error message
- [ ] `claude-profile show <profile>` displays profile contents with API key masked as `sk-***...***-last4` (shows first 3 chars, ellipsis, last 4 chars)
- [ ] `claude-profile show <profile>` exits with code 1 if profile does not exist

### TUI

- [ ] `claude-profile tui` opens an interactive menu with arrow-key navigation (Up/Down to select, Enter to confirm)
- [ ] TUI shows profile list (left pane, 40% width) with active marker and preview pane (right pane, 60% width) showing selected profile details
- [ ] Pressing Enter on a profile opens confirmation dialog showing profile name and provider; Y confirms switch, N or Esc cancels
- [ ] TUI re-renders within 100ms of terminal resize event; does not crash on resize to below 80x24 (shows minimum-size warning)
- [ ] `q` or `Esc` exits TUI cleanly, restoring terminal state
- [ ] TUI falls back to error message and exit code 1 if terminal does not support TUI mode

### Hook Integration

- [ ] `claude-profile hook install` adds a `UserPromptSubmit` hook entry to `settings.json`, copies hook script to `~/.claude/hooks/claude-profile-hook.mjs`, and backs up `settings.json` before modification
- [ ] `claude-profile hook install` is idempotent: running twice produces the same result without duplicate entries
- [ ] `claude-profile hook uninstall` removes only the tool's hook entry from `settings.json`, leaving all other hooks intact
- [ ] Hook script detects natural language switch requests matching regex `/switch\s+(?:to\s+)?(\w+)(?:\s+profile)?/i`
- [ ] Hook responds with informational message: "To switch to '<profile>' profile, run: claude-profile switch <profile>. Then restart Claude Code with /exit and restart."
- [ ] Hook does NOT modify `settings.json` or shell out to CLI

### Cross-Platform / XDG

- [ ] On macOS/Linux: profiles stored in `~/.config/claude-profile/profiles/`
- [ ] On Windows: profiles stored in `%APPDATA%/claude-profile/profiles/`
- [ ] Backups stored in `~/.local/share/claude-profile/backups/` (macOS/Linux) or `%LOCALAPPDATA%/claude-profile/backups/` (Windows)
- [ ] `active_profile` marker stored in `~/.local/share/claude-profile/active_profile` (macOS/Linux) or `%LOCALAPPDATA%/claude-profile/active_profile` (Windows)
- [ ] Respects `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, and `CLAUDE_CONFIG_DIR` env overrides

### Validation

- [ ] API key format validated by prefix: `sk-` (OpenAI), `sk-ant-` (Anthropic), `sk-kimi-` (Kimi), `sk-glm-` (GLM); unknown prefixes warn but do not block
- [ ] `base_url` validated as valid URL syntax using `url` crate (lightweight, no HTTP request)
- [ ] Optional HTTP HEAD/GET to `base_url` confirms reachability when compiled with `--features http-validate`; skipped by default
- [ ] TOML schema validated: `name` is required; all other fields optional; unknown fields in `[settings]` captured by `#[serde(flatten)] extra` with warning logged to stderr
- [ ] `settings.json` parse failure handled gracefully: if existing `settings.json` is malformed, tool refuses to operate and instructs user to fix or remove the file

---

## Implementation Steps

### Phase 1: Project Skeleton & Domain Layer (4h)

**Goal:** Establish the codebase structure, dependency graph, and core domain types.

**Files:**
- `/Users/sun/Developer/claude-profile/Cargo.toml` -- workspace manifest with feature-gated dependencies
- `/Users/sun/Developer/claude-profile/src/main.rs` -- entry point, CLI/TUI dispatch
- `/Users/sun/Developer/claude-profile/src/lib.rs` -- library root
- `/Users/sun/Developer/claude-profile/src/config.rs` -- XDG path resolution, `ConfigSwitcher` struct
- `/Users/sun/Developer/claude-profile/src/profile.rs` -- `Profile` struct, TOML serialization, `import` command
- `/Users/sun/Developer/claude-profile/src/error.rs` -- unified error type using `thiserror`

**Dependencies:**
```toml
[features]
default = []
http-validate = ["reqwest", "tokio"]

[dependencies]
clap = { version = "4", features = ["derive"] }
ratatui = "0.29"
crossterm = "0.28"
toml = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
dirs = "6"
url = "2"
thiserror = "2"
anyhow = "1"
chrono = "0.4"
fs4 = "0.12"
reqwest = { version = "0.12", features = ["rustls-tls"], default-features = false, optional = true }
tokio = { version = "1", features = ["rt-multi-thread", "macros"], optional = true }
```

**Tasks:**
1. `cargo init --name claude-profile`
2. Implement `ConfigPaths` using `dirs` crate with `CLAUDE_CONFIG_DIR` override
3. Define `Profile` struct with `serde` derive, implement `load()` / `save()` / `list()` / `import_from_json()`
4. Define `ClaudeSettings` struct matching actual `settings.json` schema with `#[serde(flatten)] extra: HashMap<String, serde_json::Value>` for forward compatibility
5. Implement atomic file write helper (`write_atomic`) using temp-file + rename
6. Implement `active_profile` marker file read/write

### Phase 2: Core Switch Logic (8h)

**Goal:** Implement the full switch pipeline: validation, dirty check, backup, atomic write, rollback, file locking.

**Files:**
- `/Users/sun/Developer/claude-profile/src/switcher.rs` -- `ConfigSwitcher` implementation
- `/Users/sun/Developer/claude-profile/src/backup.rs` -- backup creation, retention, rollback
- `/Users/sun/Developer/claude-profile/src/validation.rs` -- profile validation
- `/Users/sun/Developer/claude-profile/src/dirty_check.rs` -- compare current settings against last backup

**Tasks:**
1. Implement `backup_current()` -- copies `settings.json` to `backups/` with ISO-8601 timestamp filename; stores active profile name in backup metadata (sidecar `.meta` file or filename prefix)
2. Implement `switch_to(profile_name)` -- full pipeline: validate -> dirty check -> lock -> backup -> write atomic -> update marker -> unlock
3. Implement `rollback()` -- restores most recent backup, updates active profile marker from backup metadata
4. Implement `validate_profile()` -- TOML parse, required fields (`name`), URL syntax via `url` crate, API key prefix checks
5. Implement optional HTTP validation for `base_url` (async, behind `http-validate` feature flag)
6. Implement `dirty_check()` -- compare current `settings.json` against most recent backup; warn if different
7. Implement file locking using `fs4` crate: exclusive lock on `.switch.lock` with 5-second timeout
8. Implement backup retention: count-based only, default 50, configurable via `CLAUDE_PROFILE_BACKUP_COUNT`
9. Implement `settings.json` parse failure handling: if existing file is malformed JSON, refuse to operate with clear error message

**Switch strategy (default: full replacement):**
1. Load the Profile's complete settings definition from TOML `[settings]` section
2. Validate the Profile configuration
3. Check if current `settings.json` differs from last backup (dirty check)
4. Acquire exclusive file lock
5. Create timestamped backup of current `settings.json`
6. Atomically write Profile's `[settings]` as the new `settings.json`
7. Preserve hook entry: if hook is installed, ensure `settings.hooks` contains the claude-profile hook (re-add after write if missing)
8. Update `active_profile` marker file
9. Release lock

### Phase 3: CLI Interface (4h)

**Goal:** Expose all core operations via CLI commands. CLI ships before TUI/Hook for incremental validation.

**Files:**
- `/Users/sun/Developer/claude-profile/src/cli.rs` -- `clap` derive-based command definitions
- `/Users/sun/Developer/claude-profile/src/commands/` -- per-command handlers

**Commands:**
```
claude-profile list                          # List all profiles with active marker
claude-profile switch <name> [--dry-run] [--no-verify] [--force]  # Switch to profile
claude-profile rollback                      # Rollback to previous
claude-profile import <path> --name <name>   # Convert settings-*.json to Profile TOML
claude-profile add <name> --interactive      # Create new profile via prompts
claude-profile remove <name>                 # Delete profile after confirmation
claude-profile validate <name>               # Validate profile without switching
claude-profile show <name>                   # Display profile (API key masked)
claude-profile tui                           # Launch TUI
claude-profile hook install                  # Install Claude Code hook
claude-profile hook uninstall                # Remove Claude Code hook
```

**`--dry-run` behavior:**
- Print the full `settings.json` content that would be written (pretty-printed)
- Print the backup file path that would be created
- Print the `active_profile` marker value that would be written
- Exit with code 0 without creating or modifying any files

### Phase 4: TUI Interface (8h)

**Goal:** Interactive profile browser and switcher. Built after CLI is validated.

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
1. Implement list widget with selection state and active marker
2. Implement preview pane showing selected profile details (name, description, model, base URL, API key masked)
3. Implement switch confirmation dialog with Y/N
4. Handle terminal resize and graceful exit
5. Fallback to CLI error if TUI init fails (non-TTY, unsupported terminal)

### Phase 5: Hook Integration (4h)

**Goal:** Informational hook that detects switch intent and instructs the user. Built after CLI is validated.

**Files:**
- `/Users/sun/Developer/claude-profile/src/hook.rs` -- hook install/uninstall logic
- `/Users/sun/Developer/claude-profile/hook/claude-profile-hook.mjs` -- Node.js hook script

**Hook script behavior (informational only):**
```javascript
// hook/claude-profile-hook.mjs
const input = await readStdin();
const match = input.match(/switch\s+(?:to\s+)?(\w+)(?:\s+profile)?/i);
if (match) {
  const profile = match[1];
  // Do NOT modify settings.json from within the hook
  console.log(`\n[claude-profile] To switch to "${profile}" profile, run:\n`);
  console.log(`  claude-profile switch ${profile}`);
  console.log(`\nThen restart Claude Code with /exit and restart.\n`);
  return input + `\n\n[Note: Use "claude-profile switch ${profile}" to switch profiles]`;
}
```

**Install behavior:**
1. Read current `settings.json`
2. Add `UserPromptSubmit` hook entry pointing to hook script (idempotent: check before add)
3. Copy hook script to `~/.claude/hooks/claude-profile-hook.mjs`
4. Back up `settings.json` before modification
5. Ensure hook entry is preserved in all future switch operations

**Hook preservation in switch:**
- After writing new `settings.json` from Profile, check if hook script exists in `~/.claude/hooks/`
- If yes, ensure `settings.hooks` contains the claude-profile hook entry; re-add if missing
- This prevents the tool from self-breaking after a switch to a Profile without hooks

### Phase 6: Testing & Polish (6h)

**Goal:** Comprehensive test coverage, benchmarks, cross-platform verification.

**Files:**
- `/Users/sun/Developer/claude-profile/tests/integration.rs` -- integration tests
- `/Users/sun/Developer/claude-profile/tests/fixtures/` -- test fixtures
- `/Users/sun/Developer/claude-profile/benches/cold_start.rs` -- cold start benchmark

**Test coverage:**
- Profile TOML parse roundtrip (all known fields + unknown fields via `extra`)
- `import` command converts JSON to TOML preserving all keys
- Switch operation creates correct backup and updates active marker
- Rollback restores exact previous content and previous active profile
- Dirty check detects manual edits and requires `--force`
- File locking prevents concurrent switch operations
- Validation catches malformed URLs, invalid API key prefixes, missing required fields
- XDG path resolution on all platforms (mocked)
- Atomic write survives interruption (simulate SIGKILL mid-write)
- Backup retention prunes old backups correctly (count-based)
- Hook preservation: switch to profile without hooks does not remove hook entry
- `settings.json` parse failure: tool refuses to operate with clear error
- `--dry-run` produces no file modifications

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| `settings.json` schema changes in future Claude Code versions | Medium | High | `#[serde(flatten)] extra: HashMap<String, Value>` captures unknown keys. `import` command validates schema against real data on every use. |
| User has unsaved changes in `settings.json` that get overwritten | Medium | High | Dirty check compares current `settings.json` against last backup. Warns and requires `--force` if different. `--dry-run` shows what would change. |
| Concurrent switch operations (race condition) | Low | Medium | `fs4` exclusive file lock (`.switch.lock`) with 5-second timeout. Lock released on drop. |
| TUI fails on exotic terminal | Low | Low | Graceful fallback to CLI error if TUI init fails. Terminal capability check before TUI launch. |
| Windows path handling edge cases | Medium | Medium | Use `std::path::PathBuf` throughout; test on Windows CI; use `dirs` crate for known directories. |
| Hook installation corrupts existing hooks | Low | High | Hook install appends to existing hook array, never replaces. Hook uninstall removes only the tool's entry. Hook entry preserved across all switch operations. |
| API key leaked in backup files | Low | High | Backup files get same permissions as original (0600). Backup directory created with 0700. Mask keys in `show` output. |
| TOML schema mismatch with JSON | Not listed | High | Profile TOML uses `[settings]` section mapping 1:1 to `settings.json`. `import` command validates against real data. `#[serde(flatten)] extra` handles unknown fields. |
| User abandons tool due to complexity | Low | Medium | Ship working CLI with `list`/`switch`/`rollback` in Phase 1-3 first. TUI and Hook are Phase 4-5. Lower adoption friction with `import` command. |
| Build complexity / binary size | Low | Medium | Feature-gate `reqwest`/`tokio` (HTTP validate optional). Default build is lightweight. Target under 5MB for default build. |
| `settings.json` parse failure blocks tool | Low | High | Tool refuses to operate with clear error message. User must fix or remove malformed file. Never attempt to overwrite malformed JSON blindly. |

---

## Verification Steps

### Unit Tests

```bash
cd /Users/sun/Developer/claude-profile
cargo test
```

Expected: all tests pass, including:
- `profile::tests::parse_roundtrip` -- TOML parse and serialize roundtrip
- `profile::tests::import_preserves_all_keys` -- JSON import preserves unknown fields
- `profile::tests::extra_fields_forward_compat` -- unknown fields captured by `#[serde(flatten)]`
- `switcher::tests::backup_and_rollback` -- backup created, rollback restores exact content
- `switcher::tests::active_marker_updated` -- active_profile file updated on switch
- `switcher::tests::dirty_check_detects_changes` -- manual edits detected, `--force` required
- `switcher::tests::concurrent_switch_blocked` -- file lock prevents concurrent operations
- `switcher::tests::hook_preserved_across_switch` -- hook entry not lost when switching
- `switcher::tests::malformed_settings_refused` -- parse failure handled gracefully
- `validation::tests::invalid_url_rejected` -- malformed URL fails validation
- `validation::tests::unknown_key_prefix_warned` -- unknown API key prefix logs warning
- `backup::tests::retention_prunes_old` -- count-based retention works correctly
- `config::tests::xdg_paths_respected` -- env overrides work correctly

### Integration Test (Manual)

```bash
# 1. Build (default, lightweight)
cargo build --release
# Verify binary size under 5MB
ls -la target/release/claude-profile

# 2. Import existing profile
./target/release/claude-profile import ~/.claude/settings-kimi.json --name kimi
# Expected: creates ~/.config/claude-profile/profiles/kimi.toml with [settings] section

# 3. List profiles
./target/release/claude-profile list
# Expected: shows "kimi" profile, no active marker

# 4. Validate
./target/release/claude-profile validate kimi
# Expected: success (URL valid, key format ok), exit code 0

# 5. Dry run switch
./target/release/claude-profile switch kimi --dry-run
# Expected: prints full settings.json content, backup path, active marker value; no files modified

# 6. Switch (for real)
./target/release/claude-profile switch kimi
# Expected: backup created, settings.json updated, active_profile marker written

# 7. Verify active marker
cat ~/.local/share/claude-profile/active_profile
# Expected: "kimi"

# 8. Verify backup
ls ~/.local/share/claude-profile/backups/
# Expected: timestamped backup file exists with 0600 permissions

# 9. List with active marker
./target/release/claude-profile list
# Expected: "kimi" marked with asterisk as active

# 10. Rollback
./target/release/claude-profile rollback
# Expected: settings.json restored, active_profile marker cleared or restored

# 11. TUI smoke test
./target/release/claude-profile tui
# Expected: opens interactive menu, q exits cleanly, terminal state restored

# 12. Hook install
./target/release/claude-profile hook install
# Expected: hook entry added to settings.json, script copied to ~/.claude/hooks/

# 13. Verify hook idempotent install
./target/release/claude-profile hook install
# Expected: no duplicate hook entries

# 14. Verify hook preserved after switch
./target/release/claude-profile switch kimi
grep -c "claude-profile-hook" ~/.claude/settings.json
# Expected: count is 1 (hook entry preserved)
```

### Benchmarks

```bash
# Cold start benchmark
cargo bench --bench cold_start
# Expected: mean cold start < 100ms on target hardware

# Binary size check (default build, no optional features)
cargo build --release
# Expected: target/release/claude-profile < 5MB

# Binary size check (full build with all features)
cargo build --release --features http-validate
# Expected: target/release/claude-profile < 15MB
```

### Cross-Platform Verification

- [ ] macOS (Apple Silicon): native build, `cargo test`, manual integration test
- [ ] macOS (Intel): native build, `cargo test`
- [ ] Linux: `cargo test` in Docker (`docker run --rm -v $(pwd):/app -w /app rust:slim cargo test`)
- [ ] Windows: native build on Windows machine or VM (cross-compile only verifies compilation, not runtime)

### Hook Verification

```bash
# Install hook
./target/release/claude-profile hook install

# Verify hook entry in settings.json
grep -A2 "claude-profile-hook" ~/.claude/settings.json

# Verify hook script exists
ls ~/.claude/hooks/claude-profile-hook.mjs

# In a Claude Code session, type "switch to kimi profile"
# Expected: hook responds with informational message, does NOT modify settings.json
```

---

## ADR

### ADR-1: Full-File Replacement as Default Switch Strategy

**Status:** Accepted

**Context:** The initial plan proposed a merge strategy that reads the current `settings.json` as a base and overlays Profile values. Both Architect and Critic identified this as problematic.

**Decision:** Full-file replacement is the default. The Profile TOML supports a `[settings]` section that maps 1:1 to `settings.json` keys, allowing the user to define the complete desired state. Merge mode is relegated to a potential v2 enhancement.

**Consequences:**
- (+) Deterministic switch results: switching to Profile A then Profile B produces exactly Profile B's configuration
- (+) No configuration drift across successive switches
- (+) Matches user's existing workflow (complete `settings-*.json` files)
- (+) Simpler implementation, fewer edge cases
- (-) User must include all desired settings in the Profile; no automatic preservation of "non-profile" keys
- (-) Hook entry must be explicitly preserved by the tool after each switch

### ADR-2: Profile TOML with `[settings]` Section

**Status:** Accepted

**Context:** The initial domain model had a simplified `Profile` struct (`name`, `description`, `env`, `model`, `effort_level`, `hooks`) that did not match the actual `settings.json` schema. Fields like `enabledPlugins`, `extraKnownMarketplaces`, `statusLine`, `attribution`, `language` were missing.

**Decision:** Profile TOML uses a `[settings]` section that is a direct structural parallel to `settings.json`. The `ClaudeSettings` struct uses `#[serde(flatten)] extra: HashMap<String, serde_json::Value>` for forward compatibility with unknown fields.

**Consequences:**
- (+) No data loss: all known `settings.json` fields are preserved on switch
- (+) Forward compatible: unknown future fields are captured and preserved
- (+) `import` command can convert any existing `settings-*.json` without loss
- (-) Profile TOML files are slightly more verbose (nested `[settings]` section)
- (-) User must understand `settings.json` structure to hand-author profiles

### ADR-3: Informational Hook Design

**Status:** Accepted

**Context:** The initial plan proposed a hook that shells out to the CLI and modifies `settings.json` in-session. Both Architect and Critic identified this as creating false expectations of in-session switching and risking race conditions with hook reloading.

**Decision:** The hook is informational only. It detects switch intent and responds with instructions to run `claude-profile switch <name>` and restart Claude Code. It does NOT modify `settings.json` or shell out to the CLI.

**Consequences:**
- (+) Honest about limitations: user understands a restart is required
- (+) No race conditions with hook reloading
- (+) No circular dependency (hook does not modify the file that defines the hook)
- (-) Less "magical" user experience
- (-) User must still run CLI command manually

### ADR-4: Feature-Gated HTTP Validation

**Status:** Accepted

**Context:** The initial plan included `reqwest` and `tokio` as core dependencies for optional HTTP validation. Both reviews identified these as the heaviest dependencies, bloating the binary and compile times.

**Decision:** `reqwest` and `tokio` are optional dependencies behind the `http-validate` feature flag. Default build uses the lightweight `url` crate for URL syntax validation only. HTTP reachability checks require compiling with `--features http-validate`.

**Consequences:**
- (+) Default binary is lightweight (< 5MB target)
- (+) Faster compile times for default build
- (+) Sub-100ms cold start achievable for default build
- (-) HTTP validation is opt-in, not default
- (-) Users who want reachability checks must compile with feature flag

### ADR-5: Count-Based Backup Retention

**Status:** Accepted

**Context:** The initial plan specified "keep last 50 backups, max 30 days" dual-dimension retention. The Architect identified edge cases where these dimensions conflict.

**Decision:** Simplify to count-based only: keep last N backups (default 50), configurable via `CLAUDE_PROFILE_BACKUP_COUNT` environment variable. Remove time-based pruning.

**Consequences:**
- (+) Simpler logic, no edge cases
- (+) Predictable behavior
- (-) Very old backups are kept if user switches infrequently
- (-) Users who want time-based retention must use external tools (cron, git)

---

## Changelog (What Changed from Initial Plan)

### Critical Changes (Addressing Critic's 15 Required Changes)

1. **Merge strategy replaced with full-file replacement as default** (Critic #1, Architect #1): Profile TOML now supports a `[settings]` section mapping 1:1 to `settings.json` keys. Added `import` command (FR-11) to convert existing `settings-*.json` files. Merge mode relegated to potential v2.

2. **Profile TOML schema expanded** (Critic #2, Architect #3): `ClaudeSettings` struct now matches actual `settings.json` structure with all known fields (`enabledPlugins`, `extraKnownMarketplaces`, `statusLine`, `attribution`, `language`, `skipDangerousModePermissionPrompt`, etc.). Added `#[serde(flatten)] extra: HashMap<String, Value>` for forward compatibility.

3. **Hook redesigned to be informational** (Critic #3, Architect #4): Hook detects switch intent and tells user to run CLI + restart. Does NOT modify `settings.json` in-session. Avoids false expectations and race conditions.

4. **Principle 5 fixed** (Critic #4, Architect #2): Restated as "Zero external runtime dependencies for the core CLI binary. Hook integration leverages the Node.js runtime already required by Claude Code."

5. **File locking added** (Critic #5, Architect #5): `fs4` crate added to dependencies. Exclusive lock on `.switch.lock` with 5-second timeout implemented in Phase 2.

6. **`--dry-run` added to CLI** (Critic #6, Architect #8): Explicit command with defined behavior: show diff/content preview, backup path, no modification. Included in Phase 3 CLI commands.

7. **`active_profile` marker file added** (Critic #7, Architect #10): Writes to `~/.local/share/claude-profile/active_profile` atomically with each switch. `list` reads this file for active marker. Rollback restores previous active profile name from backup metadata.

8. **`reqwest`/`tokio` feature-gated** (Critic #8, Architect #7): HTTP validation optional via `--features http-validate`. Default build uses `url` crate for syntax validation. Target binary size < 5MB for default build.

9. **Backup retention simplified** (Critic #9, Architect #6): Count-based only (default 50), configurable via `CLAUDE_PROFILE_BACKUP_COUNT`. Time-based pruning removed.

10. **Dirty check added** (Critic #10, Architect risk table): Compares current `settings.json` against last backup before switch. Warns and requires `--force` if different.

11. **`settings.json` parse failure handling added** (Critic #11): Tool refuses to operate with clear error message if existing `settings.json` is malformed. Never overwrites malformed JSON blindly.

12. **Hook preservation resolved** (Critic #12, Architect risk table): Hook entry treated as preserved in all switch operations. Tool re-adds hook entry after every switch if hook script exists in `~/.claude/hooks/`.

13. **Tool name verification** (Critic #13): Added to open questions; must verify `claude-profile` availability on crates.io before proceeding.

14. **Phases reordered for incremental validation** (Critic #14): CLI `list`/`switch`/`rollback` (Phases 1-3) delivered first. TUI (Phase 4) and Hook (Phase 5) built after core workflow is validated.

15. **Verification benchmarks added** (Critic #15): Cold start < 100ms benchmark, binary size target (< 5MB default, < 15MB full), automated tests for error paths.

### Additional Changes (Addressing Architect's 10 Recommendations)

- **Recommendation 2 (import command)**: Added `claude-profile import <path> --name <name>` command. Converts JSON to TOML preserving all keys.
- **Recommendation 3 (TOML schema)**: `ClaudeSettings` struct expanded with all known fields, `#[serde(rename = "effortLevel")]`, and `#[serde(flatten)] extra`.
- **Recommendation 4 (informational hook)**: Hook script redesigned. No in-session modification.
- **Recommendation 5 (file locking)**: `fs4` crate, exclusive lock, 5s timeout.
- **Recommendation 6 (count-based retention)**: Simplified to count-only.
- **Recommendation 7 (feature-gate reqwest/tokio)**: `http-validate` feature flag.
- **Recommendation 8 (dry-run)**: Explicit CLI flag with defined output.
- **Recommendation 9 (effortLevel naming)**: Internal code uses `effortLevel` with `#[serde(rename = "effortLevel")]`.
- **Recommendation 10 (active_profile tracking)**: Marker file mechanism.

### Structural Changes

- **RALPLAN-DR Summary revised**: Principles 2 and 5 rewritten. Decision Driver 4 (schema fidelity) and 5 (honest limitations) added.
- **Acceptance Criteria rewritten**: All criteria now concrete and testable. Added specific exit codes, output formats, and edge case behaviors.
- **Verification Steps expanded**: Added benchmark section, binary size checks, hook preservation test, malformed settings test.
- **ADR section added**: 5 ADRs documenting key architectural decisions with context, decision, and consequences.
- **Changelog section added**: This section, documenting all changes from initial plan.
- **Domain Model updated**: `ClaudeSettings` with `extra` field, `active_profile_file` path, `import_from_json()` method.
- **Phase 2 expanded**: Added dirty check, file locking, hook preservation, parse failure handling, backup retention tasks. Hours increased from 6h to 8h.
- **Cargo.toml updated**: Feature flags, `fs4`, `url`, optional `reqwest`/`tokio`.

---

*Plan revised: 2026-05-12*
*Based on: Architect review + Critic evaluation of initial plan*
*Spec source: deep-interview-claude-config-switcher.md (11 rounds, 19.0% ambiguity, PASSED)*
