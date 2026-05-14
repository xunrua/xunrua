# Architect Review (Round 2)

## Verdict
APPROVE

## Previous Recommendations — Status

| # | Recommendation | Status | Notes |
|---|---------------|--------|-------|
| 1 | Full-file replacement default | IMPLEMENTED | ADR-1 accepted; switch pipeline explicitly does atomic full replacement |
| 2 | Import command for existing JSON | IMPLEMENTED | `claude-profile import <path> --name <name>` in CLI commands |
| 3 | TOML schema matches settings.json | IMPLEMENTED | `ClaudeSettings` with all known fields + `#[serde(flatten)] extra` |
| 4 | Informational hook only | IMPLEMENTED | ADR-3 accepted; hook detects intent, prints instructions, does not modify |
| 5 | File locking | IMPLEMENTED | `fs4` crate, exclusive lock on `.switch.lock`, 5s timeout |
| 6 | Count-based backup retention | IMPLEMENTED | ADR-5 accepted; default 50, env-configurable |
| 7 | Feature-gate reqwest/tokio | IMPLEMENTED | `http-validate` feature flag; default build < 5MB target |
| 8 | Dry-run flag | IMPLEMENTED | `--dry-run` shows preview content, backup path, no modifications |
| 9 | effortLevel naming consistency | IMPLEMENTED | `#[serde(rename = "effortLevel")]` noted in domain model |
| 10 | active_profile tracking | IMPLEMENTED | Marker file at `XDG_DATA_HOME/claude-profile/active_profile` |

## New Issues

None. The revisions are clean and address all prior concerns without introducing regressions.

## Overall Assessment

The plan is now architecturally sound. Key improvements verified:

- **Merge strategy**: Correctly replaced with full-file replacement. The hook preservation logic (re-adding hook entry post-switch if script exists) is the right approach to prevent self-breaking.
- **Schema fidelity**: `[settings]` section with `#[serde(flatten)] extra` provides forward compatibility without data loss.
- **Hook design**: Informational-only, no in-session modification, no shell-outs. Honest about restart requirement.
- **Principle 5**: Correctly restated — core CLI is zero-dependency static binary; Node.js only for optional hook companion.
- **Safety mechanisms**: Dirty check + `--force`, file locking, atomic writes, timestamped backups, parse-failure refusal, backup retention — all present.
- **Phase ordering**: CLI validation (Phases 1-3) before TUI/Hook (Phases 4-5) eliminates integration risk.
- **Dependencies**: `url` crate for default validation; `reqwest`/`tokio` properly gated behind `http-validate`.

One minor observation: the plan is ambitious (34 hours estimated) but the phased approach mitigates delivery risk. Phase 1-3 deliver core value; Phases 4-5 are additive.

Proceed to execution.
