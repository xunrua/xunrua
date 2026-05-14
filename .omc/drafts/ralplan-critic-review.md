# Critic Evaluation

## Verdict
**ITERATE**

The plan is fundamentally sound in direction but contains multiple shallow alternatives, principle contradictions, vague risks, and weak verification that must be addressed before implementation. The Architect's review correctly identifies the most serious issues; I independently confirm all of them and add several additional concerns.

---

## Principle-Option Consistency

**Rating: POOR**

Multiple principle-option contradictions exist:

1. **Principle 1 (File-level atomicity) vs. Hook design**: Principle 1 explicitly rejects "process injection, signal hacks, or IPC to force Claude Code runtime reload." Yet Phase 5's hook design attempts exactly that — an in-session trigger that modifies `settings.json` while Claude Code is running, creating the illusion of runtime reload without the reality. This is not a technical violation of the letter (it's a file write, not process injection), but it is a direct violation of the spirit. The hook gives users false confidence that switching "works" in-session, when it actually requires restart.

2. **Principle 2 (Profile-as-source-of-truth) vs. Merge strategy**: Principle 2 states "The active configuration is always the result of merging a Profile into `settings.json`." But the merge strategy makes the final state a function of BOTH the Profile AND the current `settings.json` base. The Profile is NOT the sole source of truth — the merge operation is. After switching to Profile A, then Profile B, the result depends on what A left behind. This is configuration drift by design.

3. **Principle 5 (Zero external runtime dependencies) vs. FR-5 (Hook Integration)**: Principle 5 demands "No Node, no Python, no daemon." But the hook is explicitly a Node.js script (`hook/claude-profile-hook.mjs`). The plan acknowledges this in Option B's cons ("Adds a Node.js hook dependency") but still lists Hook Integration as a Must-level requirement. This is a direct contradiction, not a tension to be managed.

4. **Principle 3 (Defensive by default) vs. missing dirty check**: Principle 3 says "Validation runs before write, not after." Yet there is no mechanism to detect if `settings.json` has been manually edited since the last switch. The user could lose unsaved changes, and the backup would back up the modified file — not the original. The backup protects against tool failure but not against user workflow failure.

---

## Alternative Exploration

**Rating: INADEQUATE**

The plan explores four options (A-D) but several are strawmen and one critical alternative is never considered:

**Shallow alternatives presented:**
- Option B (inotify/fswatch): Correctly rejected, but framed as if it were ever viable. No one building a CLI tool would seriously consider inotify as a config-switching mechanism.
- Option C (symlink swap): Correctly rejected for Windows, but this is a well-known limitation — not a genuine tradeoff.
- Option D (overlay merge): Dismissed with "adds significant complexity for marginal benefit" without quantifying the complexity or the benefit. The user's actual `settings-*.json` files suggest they use complete replacements, making this dismissal actually correct — but the reasoning is hand-wavy.

**Critical missing alternative: Shell script / `fzf` approach**
The Architect correctly identifies this. The user's existing workflow (`settings-*.json` files + manual `cp`) is already 90% functional. A 50-line shell script with `fzf` would solve the core problem in 30 minutes. The plan never seriously considers whether a Rust tool with TUI, hooks, and 32 hours of implementation is justified. The spec says "全部功能一次性实现" (all features at once), but the plan should have pushed back on scope proportionality. The deep-interview spec's Round 8 Ontologist challenge ("What makes this better than `cp`?") was answered with "validation, backup, discovery" — but the plan never validates whether those features are worth 32 hours versus a simpler approach.

**Missing alternative: JSON-native profiles instead of TOML**
The user already has working JSON files. Converting to TOML introduces a translation layer, schema mapping risk, and cognitive overhead. The plan never explores keeping profiles as JSON (or JSON-compatible) files, which would eliminate the schema mismatch risk entirely and allow direct `cp`-style operations.

---

## Risk Mitigation

**Rating: WEAK**

Several risks are identified but mitigations are vague or incomplete:

| Risk | Plan's Mitigation | Critique |
|------|------------------|----------|
| `settings.json` schema changes | "Store schema version in Profile TOML; warn on unknown fields; maintain mapping layer" | Vague. "Maintain mapping layer" is hand-wavy. No concrete mechanism for schema evolution. The Architect's recommendation of `#[serde(flatten)] extra` is specific and actionable; the plan lacks this. |
| Concurrent switch operations | "Use file locking (`.switch.lock`) with timeout; atomic write prevents corrupted files" | **Not in implementation tasks.** The risk table mentions locking but Phase 2 tasks do not include it. This is a plan-to-implementation gap. |
| User has unsaved changes | "Always backup before switch; provide `--dry-run` flag; show diff preview in TUI" | `--dry-run` is mentioned in the risk table but NOT in the CLI command list (Phase 3). The TUI diff preview is aspirational, not specified. No "dirty check" mechanism is described. |
| Hook corrupts existing hooks | "Parse and append to existing hook array, never replace; validate JSON before write" | Insufficient. If the Profile switch uses full-file replacement (which the merge strategy implies), the hook entry could be lost unless the Profile includes it. The circular dependency (hook modifies the file that defines the hook) is not addressed. |
| TOML schema mismatch with JSON | **Not listed as a risk** | This is a serious omission. The Profile schema is significantly simpler than actual `settings.json`. Fields like `enabledPlugins`, `extraKnownMarketplaces`, `statusLine`, `attribution`, `language` are not in the domain model. A switch would DROP these fields, causing data loss. |
| User abandons tool due to complexity | **Not listed as a risk** | 32 hours for a file-copying tool is poor ROI. If the user finds `cp` easier, the tool fails by non-use. |
| Build complexity / binary size | **Not listed as a risk** | `reqwest` + `tokio` + `ratatui` + `crossterm` will produce a binary well over 10MB with compile times over 2 minutes. For a file-copying tool, this is disproportionate. |

**Concrete mitigations that ARE good:**
- Backup retention policy (50 files / 30 days) — though the Architect correctly notes dual-dimension retention creates edge cases
- 0600 permissions on backup files — correct
- Atomic write via temp-file + rename — correct

---

## Acceptance Criteria Quality

**Rating: MIXED — some concrete, some vague**

**Concrete and verifiable:**
- `claude-profile list` prints profiles with active marker — verifiable by running the command
- `claude-profile switch <profile>` creates backup — verifiable by checking backup directory
- `claude-profile rollback` restores backup — verifiable by diffing files
- Cross-platform XDG paths — verifiable by checking actual paths on each OS
- TUI `q`/`Esc` exits cleanly — verifiable by running TUI

**Vague or unverifiable:**
- "Validates target Profile before applying" — What constitutes "valid"? URL syntax? Key prefix? HTTP reachability? The criteria say "API key format validated (prefix checks for known providers)" but don't list the prefixes or what "known providers" means.
- "TOML schema validated (required fields present, no unknown fields warned)" — What are the required fields? The domain model shows `name` as required but `description` as optional — is that the full schema? What about `env` being required for the core use case (switching API providers)?
- "Hook detects natural language switch requests" — What patterns? The open questions section asks this, but the acceptance criteria should specify the minimum pattern set before implementation.
- "TUI handles terminal resize gracefully" — What does "gracefully" mean? No crash? Re-renders correctly? Minimum terminal size?
- "Switching to Profile validates, backs up, applies, and reports success" — What does "reports success" mean? Exit code 0? A message to stdout? Both?

**Missing criteria:**
- No criterion for `active_profile` tracking (how does `list` know which profile is active?)
- No criterion for what happens when switching to the already-active profile
- No criterion for handling malformed `settings.json` (not created by the tool)
- No criterion for the `import` command (not in the plan at all, but the Architect correctly identifies it as needed)

---

## Verification Steps

**Rating: WEAK — incomplete and under-specified**

**Unit Tests section:**
- Lists four test names (`profile::tests::parse_roundtrip`, etc.) but these are examples, not an exhaustive list.
- No coverage target specified (line coverage? branch coverage?)
- No mention of testing the merge strategy (the most complex logic)
- No mention of testing error paths (invalid TOML, missing files, permission denied)
- No mention of property-based testing for roundtrip serialization

**Integration Test (Manual):**
- Steps 1-9 are a reasonable smoke test, but:
  - Step 5 references `--dry-run` which is NOT in the CLI command list (Phase 3)
  - Step 6 does not specify how to verify `settings.json` was updated correctly
  - No step tests the `show` command (with API key masking)
  - No step tests `add` or `remove` commands
  - No step tests validation failure paths
  - No step tests rollback after multiple switches (does rollback go back one step or to initial state?)

**Cross-Platform Verification:**
- "macOS: native build and test" — insufficient detail. Which macOS version? Apple Silicon or Intel?
- "Linux: `cargo test` in Docker (`rust:slim`)" — good, but should specify the Docker command
- "Windows: cross-compile from macOS" — this will NOT verify runtime behavior, only compilation. Cross-compilation does not test path handling, file permissions, or terminal behavior on Windows.

**Hook Verification:**
- Step 3 says "Test trigger (in Claude Code session, type 'switch to test profile')" — this requires manual testing inside a running Claude Code session. There is no automated verification for the hook.
- No verification that hook installation is idempotent (claimed in acceptance criteria)
- No verification that hook uninstall removes only the tool's hook entry, not all hooks

**Missing verification:**
- No benchmark for "sub-100ms cold start" (NFR-2)
- No binary size check for "single static binary" (NFR-1)
- No test for backup retention policy (NFR-5)
- No test for concurrent switch race condition
- No test for TUI on minimum terminal size (80x24, NFR-3)

---

## Architect Feedback Assessment

**I independently confirm ALL of the Architect's findings.** The following are particularly critical:

### Agreed — High Priority

1. **Merge strategy is wrong for v1** (Recommendation 1): The user's existing `settings-*.json` files are complete configurations. Full-file replacement should be the default. Merge is an advanced feature for v2 if ever needed. The plan's merge strategy will cause configuration drift and data loss.

2. **TOML schema mismatch with actual `settings.json`** (Recommendation 3): The domain model is dangerously incomplete. Fields like `enabledPlugins`, `extraKnownMarketplaces`, `statusLine`, `attribution`, `language` are missing. A switch using the current schema would DROP these fields. This is a data loss bug waiting to happen.

3. **Hook redesign to informational** (Recommendation 4): The current hook design creates false expectations and a race condition risk. The informational approach is honest about limitations.

4. **File locking missing from implementation** (Recommendation 5): The risk table mentions locking but no implementation task includes it. This is a plan-to-code gap.

5. **`reqwest` + `tokio` bloat** (Recommendation 7): These are the heaviest dependencies. For optional HTTP validation, they should be feature-gated. URL syntax validation can use the lightweight `url` crate.

6. **`--dry-run` not in CLI commands** (Recommendation 8): Mentioned in risk mitigation but not in the actual command specification. This is an inconsistency.

7. **`active_profile` tracking** (Recommendation 10): The domain model has `active_profile: Option<String>` but no mechanism for tracking. After a switch, `list` cannot know which profile is active without a marker file.

### Agreed — Medium Priority

8. **Backup retention dual-dimension** (Recommendation 6): "50 backups, max 30 days" creates edge cases. Count-based only is simpler and sufficient.

9. **`effort_level` naming** (Recommendation 9): The actual JSON uses `effortLevel` (camelCase). Internal code should match external schema for clarity.

10. **`import` command** (Recommendation 2): The user already has `settings-*.json` files. An import command lowers adoption friction and validates the schema against real data.

### Partially Agreed

11. **The "shell script is enough" argument**: The Architect's antithesis is technically correct — a shell script would solve 90% of the problem. However, the spec explicitly demands "全部功能一次性实现" (all features at once) and the user has already invested in the deep-interview process. The Rust tool is justified IF it delivers genuine value over `cp`. The plan needs to be honest about this tradeoff, not pretend the complexity is necessary.

---

## Independent Findings

### Finding 1: Phase ordering creates delivery risk

The plan implements TUI (Phase 4, 8h) and Hook (Phase 5, 4h) BEFORE the core CLI is proven working. The spec says "全部功能一次性实现" but the plan should still deliver value incrementally for validation. If the user tries the tool after Phase 3 and finds the core switch logic doesn't match their workflow, 12 hours of TUI/Hook work may be wasted. **Recommendation:** Implement CLI `list`/`switch`/`rollback` first, get user feedback, then proceed to TUI/Hook.

### Finding 2: The `add` command is underspecified

`claude-profile add <name> --interactive` creates a new profile via prompts. But:
- What prompts? Which fields are required vs optional?
- How does the user specify the `[settings]` section (if using full-file replacement)?
- Does `--from-current` import the current `settings.json`? (This is the Architect's `import` command by another name.)
- The domain model shows `Profile` with `env`, `model`, `effort_level`, `hooks` — but if the schema should match `settings.json`, the `add` command needs to prompt for ALL possible fields, which is unwieldy.

### Finding 3: No error handling strategy for `settings.json` parse failures

If the user's existing `settings.json` is malformed JSON (or uses a schema the tool doesn't know), what happens?
- Does the tool refuse to operate?
- Does it attempt to preserve the file as raw bytes?
- Does it warn and proceed?
The plan's merge strategy requires parsing `settings.json` as structured data. If parsing fails, the entire tool is blocked.

### Finding 4: The name `claude-profile` may conflict

The spec notes "cc-switch" is taken. `claude-profile` is better, but a quick crates.io/npm search should be done before committing. This is in the open questions but should be resolved before implementation.

### Finding 5: Hook installation modifies the file being switched

The hook install command modifies `settings.json` to add the hook entry. But `settings.json` is also the target of the switch operation. This means:
- After installing the hook, every switch must preserve the hook entry
- If the user switches to a Profile that doesn't include hooks, the hook is LOST
- The tool becomes self-breaking

**Mitigation:** The hook entry must be treated as a preserved key in ALL switch operations, OR the tool must add the hook entry back after every switch. Neither is described in the plan.

### Finding 6: NFR-2 "sub-100ms cold start" is unverified

With `ratatui`, `crossterm`, `reqwest`, `tokio`, and `chrono` as dependencies, cold start time depends heavily on dynamic linking vs static linking. A statically-linked binary with these dependencies may exceed 100ms startup on slower systems. The verification section has no benchmark for this.

### Finding 7: The `show` command masking is underspecified

"`claude-profile show <profile>` displays profile contents (with API key masked)" — what is the masking rule? Replace with `***`? Show first/last N characters? This matters for debugging (user needs to verify they have the right key).

---

## Required Changes

1. **Replace merge strategy with full-file replacement as default**: The Profile TOML should support a `[settings]` section that maps 1:1 to `settings.json` keys. Merge mode should be a v2 enhancement, not the default.

2. **Expand Profile TOML schema to match actual `settings.json`**: Use `#[serde(flatten)] extra: HashMap<String, Value>` for forward compatibility. Add `import` command to convert existing `settings-*.json` files.

3. **Redesign hook to be informational, not operational**: The hook should tell the user to run `claude-profile switch <name>` and restart Claude Code, not attempt to modify `settings.json` in-session.

4. **Fix Principle 5 to be honest about Node.js dependency**: "Zero external runtime dependencies for the core CLI binary. Hook integration leverages the Node.js runtime already required by Claude Code."

5. **Add file locking to implementation tasks**: Use `fs4` or `fd-lock` crate. Add to Phase 2 tasks and dependencies.

6. **Add `--dry-run` to CLI command specification**: Include in Phase 3 with explicit behavior (show diff, show backup path, no file modification).

7. **Add `active_profile` marker file mechanism**: Write to `~/.local/share/claude-profile/active_profile` atomically with each switch. Read it for `list` active marker.

8. **Feature-gate `reqwest` and `tokio`**: Make HTTP validation optional. Use `url` crate for syntax validation in default build.

9. **Simplify backup retention to count-based only**: "Keep last N backups (default 50), configurable via env var." Remove time-based pruning.

10. **Add dirty check before switch**: Compare current `settings.json` against last backup. If different, warn user and require `--force` to proceed.

11. **Add `settings.json` parse failure handling**: Define behavior when existing `settings.json` cannot be parsed (refuse to operate? preserve as raw bytes? warn and proceed?).

12. **Resolve hook preservation in switch operations**: Either treat hooks as preserved keys in all switches, or re-add the hook entry after every switch. Document the behavior.

13. **Verify tool name availability** on crates.io before proceeding.

14. **Reorder phases for incremental validation**: Deliver CLI `list`/`switch`/`rollback` first, get user validation on the core workflow, then build TUI and Hook.

15. **Add specific verification benchmarks**: Cold start time (<100ms), binary size target, and automated tests for error paths (malformed TOML, missing files, permission denied).
