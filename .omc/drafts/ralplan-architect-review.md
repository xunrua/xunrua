# Architect Review

## Overall Verdict

**ITERATE**

The plan is directionally sound but has several architectural gaps that should be resolved before implementation begins. The merge strategy is underspecified for the actual data, the TOML-to-JSON schema mapping needs clarification, and the hook integration creates a circular dependency risk. These are fixable in a revision pass.

---

## Antithesis (Strongest Counterargument)

**The entire tool is solving the wrong problem with the wrong level of complexity.**

The user's actual workflow, evidenced by their existing `settings-*.json` files, is already working: they maintain complete, independent copies of `settings.json` and swap them manually. The pain is not the swap itself — it is the *manual* aspect. A 32-hour, 6-phase Rust project with TUI, hooks, validation, and backup retention is massive over-engineering for what amounts to "copy the right file to `~/.claude/settings.json`".

The strongest counterargument proceeds as follows:

1. **The merge strategy is a liability, not a feature.** The plan proposes reading the current `settings.json` as a base and overlaying Profile values. But the user's existing files (`settings-kimi.json`, `settings-glm.json`, `settings-opus.json`) are *complete, standalone configurations*. They differ not just in `env` and `model`, but in field ordering, presence/absence of keys (`effortLevel` exists in some, not others; `model` placement varies; `attribution` is empty objects vs missing), and hook configurations that may diverge per provider. A merge strategy that preserves "non-profile keys" assumes a stable partition between "profile" and "non-profile" keys that does not exist in practice. The user's actual need is "replace the whole file atomically", which is what `cp` already does.

2. **TOML adds friction without value.** The user already has working JSON files. Converting them to TOML introduces a translation layer, schema drift risk, and cognitive overhead (now the user must maintain two formats). The stated reason for TOML is "human-editable", but JSON is already human-editable, and the user is already editing it.

3. **The TUI is premature for a tool with ~5 profiles.** The user has at most 6 profiles (kimi, glm, opus, ali, der, mimo). A TUI with arrow-key navigation, preview panes, and confirmation dialogs is a solution in search of a problem. A simple `fzf`-style selector or even numbered list in CLI would suffice.

4. **The hook integration is architecturally unsound.** The hook script modifies `settings.json`, which contains the hook definition itself. If the hook is triggered during a switch, and the switch modifies `settings.json`, the running Claude Code process may re-read hooks mid-execution. This creates a race condition where the hook could be unloaded or reloaded while running. Additionally, the hook requires Node.js (contradicting Principle 5's "zero external runtime dependencies" — the hook is an external dependency even if the CLI binary is not).

5. **Backup/rollback is solving a problem `git` already solves.** The user's `settings-*.json` files are already backups. The plan's timestamped backup system with 50-file retention and 30-day pruning is a mini-version-control system. If safety is the concern, `git init ~/.claude` gives atomic commits, diffs, and infinite rollback for free.

**The steelman conclusion:** A 50-line shell script using `cp` and `fzf` would solve 90% of the user's actual problem in 30 minutes. The Rust tool, as specified, is a category error: it applies systems-programming rigor to a dotfiles management task that needs scripting ergonomics.

---

## Tradeoff Tensions

### Tension 1: Merge Preservation vs. Complete Replacement

**The conflict:** Principle 2 states "Profile-as-source-of-truth" and "The tool never mutates Profile files during a switch." The merge strategy (Phase 2) reads the current `settings.json` as a base and overlays Profile values to "preserve non-profile keys like hooks, enabledPlugins." But this creates a tension:

- **Goal A (Preservation):** Keep user's hooks, plugins, and other settings across switches so they don't have to reconfigure.
- **Goal B (Determinism):** A switch should produce a known, reproducible configuration state. If the base `settings.json` has been manually edited, the result of `switch` is non-deterministic — it depends on the current state of the file.

**Where they conflict:** If the user manually edits `settings.json` (adds a new plugin, changes a hook), then switches to Profile A, then switches to Profile B, the resulting `settings.json` after switching to B depends on what A left behind, not on B alone. The "preserved" keys accumulate across switches, creating configuration drift. This directly undermines the user's stated core need: reliable, predictable switching between API providers.

**Evidence from actual data:** Comparing `settings-kimi.json` and `settings-glm.json`:
- `settings-kimi.json` has `"effortLevel": "xhigh"` at the root level
- `settings-glm.json` does NOT have `effortLevel`
- `settings-kimi.json` has `"model": "opus[1m]"` at the root level
- `settings-glm.json` has `"model": "opus[1m]"` at the END of the file
- `settings-glm.json` has additional env vars: `ANTHROPIC_MODEL`, `ANTHROPIC_DEFAULT_HAIKU_MODEL`, `ANTHROPIC_DEFAULT_SONNET_MODEL`, `ANTHROPIC_DEFAULT_OPUS_MODEL`
- `settings-kimi.json` has `"skipDangerousModePermissionPrompt": true` which `settings-glm.json` also has

These are not "profile" vs "non-profile" keys. They are *all* configuration. The user's existing workflow is full-file replacement, not overlay merging.

### Tension 2: Zero Dependencies (Principle 5) vs. Hook Integration (FR-5)

**The conflict:** Principle 5 demands "Zero external runtime dependencies: Single static binary. No Node, no Python, no daemon." But FR-5 (Hook Integration) requires a Node.js hook script (`hook/claude-profile-hook.mjs`) that executes within Claude Code's hook system. Claude Code hooks are Node.js scripts.

**Where they conflict:** The hook script is not optional — it is a Must-level requirement. But it is also an external runtime dependency (Node.js). Even if the CLI binary itself is a static Rust binary, the *feature* requires Node. The plan acknowledges this in Option B's cons: "Adds a Node.js hook dependency" but then proceeds to include it as a Must requirement anyway.

### Synthesis

**For Tension 1 (Merge vs. Replace):** Support both modes. Add a `--full-replace` flag (defaulting based on user preference) and a `--merge` flag. Better yet, define the merge semantics explicitly:

1. **Category 1 - Profile-owned keys:** `env`, `model`, `effortLevel` — always overwritten by Profile.
2. **Category 2 - User-owned keys:** `hooks`, `enabledPlugins`, `extraKnownMarketplaces`, `statusLine` — preserved from base unless Profile explicitly overrides.
3. **Category 3 - Unknown keys:** Warn on first encounter, preserve by default, log to stderr.

But even better: **abandon merge entirely for v1.** The user's existing workflow is full-file replacement. Implement atomic full-file replacement first. Add merge as a v2 enhancement if requested. This aligns with the spec's "务实优先" (pragmatism first) constraint and the user's actual behavior.

**For Tension 2 (Zero Dependencies vs. Hook):** Restate Principle 5 more precisely: "Zero external runtime dependencies for the CLI binary. Hook integration is an optional companion feature that requires Node.js, which is already a dependency of Claude Code itself." This is honest and accurate. Then make the hook truly optional — installable via `claude-profile hook install` but not required for core functionality. The current plan already does this structurally; the principle just needs to match.

---

## Principle Violations

### Violation 1: Principle 5 (Zero Dependencies) contradicted by FR-5

As detailed in Tension 2, the hook integration requires Node.js. The principle should be amended to: "Zero external runtime dependencies for the core CLI binary. Hook integration leverages the Node.js runtime already required by Claude Code."

### Violation 2: Principle 2 (Profile-as-source-of-truth) contradicted by merge strategy

Principle 2 says "The tool never mutates Profile files during a switch." This is fine. But it also implies the Profile is the complete definition of the desired state. The merge strategy violates this by making the final state a function of both the Profile AND the current `settings.json`. The Profile is NOT the source of truth for the final state — the merge operation is.

**Recommended fix:** Restate Principle 2 as: "Profile defines the desired configuration. The tool applies it atomically. Full replacement is default; selective merge is opt-in."

### Violation 3: Principle 1 (File-level atomicity) partially undermined by hook

Principle 1 says "Do not attempt process injection, signal hacks, or IPC." The hook integration (Phase 5) is technically not process injection, but it IS an in-session trigger that attempts to modify configuration while Claude Code is running. The hook receives prompt text, shells out to the CLI, modifies `settings.json`, and then... what? Claude Code does not reload config. The user must still restart. The hook gives the *illusion* of in-session switching without the reality, which is worse than requiring an explicit restart. It violates the spirit of Principle 1 by pretending to do what Principle 1 explicitly rejects.

**Recommended fix:** Redesign the hook to be informational, not operational. The hook detects switch intent and responds with: "Switching to Profile X. Please restart Claude Code with `/exit` and restart for changes to take effect." This is honest about limitations and avoids the false promise of in-session switching.

---

## Concrete Recommendations

### 1. Replace merge strategy with full-file replacement as default

**Location:** Phase 2, `src/switcher.rs`, lines 217-223 of plan

**Current:**
```
Merge strategy:
Profile TOML defines a subset of settings. The switch operation constructs a new settings.json by:
1. Reading the current settings.json as the base
2. Overwriting env, model, effortLevel with Profile values
3. Writing the merged result
```

**Recommended:**
```
Switch strategy (default: full replacement):
1. Load the Profile's complete settings definition
2. Validate the Profile configuration
3. Atomically write it to settings.json (temp file + rename)

The Profile TOML should support a [settings] section that maps 1:1 to settings.json keys,
allowing the user to define the complete desired state. Example:

[settings]
model = "opus[1m]"
effortLevel = "xhigh"
skipDangerousModePermissionPrompt = true
language = "简体中文"

[settings.env]
ANTHROPIC_AUTH_TOKEN = "..."
ANTHROPIC_BASE_URL = "..."

[settings.hooks]
# ... full hooks definition if desired
```

**Rationale:** The user's existing `settings-*.json` files are complete configurations. The tool should consume these directly or allow TOML equivalents that are equally complete. Partial merge is an advanced feature for a future version.

### 2. Add a `from-json` migration command

**Location:** New command: `claude-profile import <file>`

**Recommended:** Since the user already has `settings-*.json` files, provide a one-command import:
```bash
claude-profile import ~/.claude/settings-kimi.json --name kimi
```

This converts the JSON to the tool's TOML format, preserving all keys. It lowers adoption friction and validates the "complete configuration" approach.

### 3. Fix the TOML schema to match actual settings.json structure

**Location:** Phase 1, `src/profile.rs`, Domain Model section

**Current:**
```
Profile (stored as TOML)
  name: String
  description: Option<String>
  env: Map<String, String>
  model: Option<String>
  effort_level: Option<String>
  hooks: Option<Hooks>
```

**Problem:** This schema does not match the actual `settings.json`. The actual file has:
- `effortLevel` (camelCase, not snake_case)
- `enabledPlugins` (nested object with boolean values)
- `extraKnownMarketplaces` (nested object with source/repo)
- `statusLine` (object with type/command)
- `attribution` (object with commit/pr)
- `language` (string)
- `skipDangerousModePermissionPrompt` (boolean)

**Recommended:** Define the Profile TOML schema as a superset/parallel of the actual settings.json schema. Use serde's `rename` attributes to handle camelCase. Consider using a generic `serde_json::Value` fallback for forward compatibility:

```rust
#[derive(Serialize, Deserialize)]
struct Profile {
    name: String,
    description: Option<String>,
    #[serde(flatten)]
    settings: ClaudeSettings,
}

#[derive(Serialize, Deserialize)]
struct ClaudeSettings {
    env: Option<HashMap<String, String>>,
    model: Option<String>,
    #[serde(rename = "effortLevel")]
    effort_level: Option<String>,
    hooks: Option<serde_json::Value>,
    #[serde(rename = "enabledPlugins")]
    enabled_plugins: Option<HashMap<String, bool>>,
    // ... etc
    #[serde(flatten)]
    extra: HashMap<String, serde_json::Value>,
}
```

### 4. Redesign hook to be informational, not operational

**Location:** Phase 5, `hook/claude-profile-hook.mjs`

**Current design:** Hook detects switch intent, shells out to CLI, modifies `settings.json`.

**Problem:** Creates false expectation of in-session switching. Risks race condition with hook reloading.

**Recommended design:**
```javascript
// hook/claude-profile-hook.mjs
const input = await readStdin();
const match = input.match(/switch\s+(?:to\s+)?(\w+)(?:\s+profile)?/i);
if (match) {
  const profile = match[1];
  // Do NOT modify settings.json from within the hook
  // Instead, inform the user what to do
  console.log(`\n[claude-profile] To switch to "${profile}" profile, run:\n`);
  console.log(`  claude-profile switch ${profile}`);
  console.log(`\nThen restart Claude Code with /exit and restart.\n`);
  // Optionally: return a modified prompt that includes the instruction
  return input + `\n\n[Note: Use "claude-profile switch ${profile}" to switch profiles]`;
}
```

Alternatively, if the hook MUST trigger the switch, add a `--notify-only` mode to the CLI that prints restart instructions after switching.

### 5. Add file locking for atomic operations

**Location:** Phase 2, `src/switcher.rs`

**Current:** The plan mentions "Use file locking (`.switch.lock`) with timeout" in the risk table but does not include it in implementation tasks.

**Recommended:** Add explicit file locking to the switch operation:

```rust
use fs4::FileExt; // or fd-lock crate

fn switch_with_lock(profile: &str) -> Result<()> {
    let lock_path = paths.backups_dir.join(".switch.lock");
    let lock_file = std::fs::OpenOptions::new()
        .write(true)
        .create(true)
        .open(&lock_path)?;
    
    // Timeout after 5 seconds
    lock_file.try_lock_exclusive()
        .map_err(|_| Error::ConcurrentSwitch)?;
    
    // ... perform switch ...
    
    // Lock released on drop
    Ok(())
}
```

Add `fs4` or `fd-lock` to dependencies. This is critical because the hook could trigger while a manual `switch` is in progress.

### 6. Change backup retention to count-based only

**Location:** NFR-5, Risk table

**Current:** "keep last 50 backups, max 30 days"

**Problem:** Two dimensions of retention create edge cases. What if a backup is 31 days old but the user only has 3 backups? Deleting it leaves them with 2. What if they have 60 backups all from the last week? The 50-count limit deletes 10 that are still recent.

**Recommended:** Simplify to count-based only: "keep last N backups (default 50), configurable via `CLADE_PROFILE_BACKUP_COUNT`". Time-based pruning is unnecessary complexity for a config backup system. If the user wants time-based retention, they can use a cron job or git.

### 7. Remove `reqwest` + `tokio` dependency if HTTP validation is optional

**Location:** Phase 1, Cargo.toml dependencies

**Current:** `reqwest` and `tokio` are listed as dependencies for optional HTTP validation.

**Problem:** These are the heaviest dependencies in the project. `reqwest` pulls in `hyper`, `tokio`, `rustls`, etc. For a tool whose core job is file operations, this is significant bloat.

**Recommended:** Make HTTP validation truly optional via a feature flag:

```toml
[features]
default = []
http-validate = ["reqwest", "tokio"]

[dependencies]
reqwest = { version = "0.12", features = ["rustls-tls"], default-features = false, optional = true }
tokio = { version = "1", features = ["rt-multi-thread", "macros"], optional = true }
```

Default build is static and lightweight. Users who want URL reachability checks compile with `--features http-validate`. URL *syntax* validation can be done with a lightweight regex or the `url` crate (much smaller than `reqwest`).

### 8. Add `--dry-run` to the CLI command list

**Location:** Phase 3, CLI commands

**Current:** `--dry-run` is mentioned in the risk table but not in the CLI command list.

**Recommended:** Add it explicitly:
```
claude-profile switch <name> [--dry-run] [--no-verify] [--full-replace]
```

And implement it in Phase 2, not as an afterthought. The dry-run should show:
- What file would be written
- What backup would be created
- A diff of changes (if merge mode) or full content preview (if replace mode)

### 9. Rename `effort_level` to `effortLevel` in domain model

**Location:** Domain Model, `Profile` struct

**Current:** `effort_level: Option<String>`

**Problem:** The actual settings.json uses camelCase (`effortLevel`). Using snake_case in the domain model creates a naming impedance mismatch. Serde can handle the rename, but the internal code should match the external schema for clarity.

**Recommended:** Use `effortLevel` everywhere, with `#[serde(rename = "effortLevel")]` if the TOML format uses snake_case.

### 10. Add `active_profile` tracking

**Location:** Domain Model, `ConfigSwitcher`

**Current:** `active_profile: Option<String>` is in the `ConfigSwitcher` struct, but there's no mechanism described for tracking which profile is active.

**Problem:** After a switch, how does `claude-profile list` know which profile is active? It can't reliably diff `settings.json` against all profiles (especially with merge mode).

**Recommended:** Write an `active_profile` marker file:
```
~/.local/share/claude-profile/active_profile  # contains "kimi" or is empty
```

Update this atomically with the switch. `list` reads this file to show the active marker. On rollback, restore the previous active profile name.

---

## Risk Assessment

| Risk | Plan Assessment | Architect Assessment | Mitigation |
|------|----------------|---------------------|------------|
| `settings.json` schema changes | Medium/High | **High** | The plan's "store schema version in Profile TOML" is insufficient. Add `#[serde(flatten)] extra: HashMap<String, Value>` to capture unknown keys. Use `serde_json::Value` for nested objects instead of strict structs. |
| Unsaved changes overwritten | Medium/High | **Medium** | The backup strategy is sound, but the plan lacks a "dirty check". Before switch, compare current `settings.json` against the last backup. If different, warn: "settings.json has been modified outside the tool. Continue?" |
| Concurrent switch race condition | Low/Medium | **Medium** | The risk table mentions file locking but implementation tasks do not include it. **Mandatory:** Add `fs4` or `fd-lock` dependency and implement exclusive lock during switch. |
| Hook corrupts existing hooks | Low/High | **High** | The plan says "Parse and append to existing hook array, never replace" but the hook install task modifies `settings.json` directly. This is the SAME file the switch operation modifies. If the hook is installed and a switch happens, the hook entry could be lost if the Profile doesn't include it. **Mitigation:** Treat hooks as Category 2 (preserved) keys in all switch operations, OR add the hook to every Profile template. |
| API key leaked in backups | Low/High | **Low** | Plan handles this correctly with 0600 permissions. Also ensure backup directory is created with 0700. |
| TOML schema mismatch with JSON | Not listed | **High** | The Profile TOML schema is significantly simpler than actual settings.json. This will cause data loss (e.g., `enabledPlugins`, `extraKnownMarketplaces` will be dropped on switch). **Mitigation:** Implement `import` command first to validate the schema against real data. |
| Build complexity / compile time | Not listed | **Medium** | `reqwest` + `tokio` + `ratatui` + `crossterm` will produce a binary well over 10MB and compile times over 2 minutes. For a file-copying tool, this is poor ROI. **Mitigation:** Feature-gate `reqwest`/`tokio`, consider `tui-rs` alternative if binary size matters. |
| User abandons tool due to complexity | Not listed | **Medium** | A 32-hour implementation for a problem solvable with shell aliases creates adoption risk. The user may find it easier to keep using `cp`. **Mitigation:** Ship a working CLI with `list`/`switch`/`rollback` in Phase 1-2 before investing in TUI/Hook. Validate user engagement before Phase 4+. |
