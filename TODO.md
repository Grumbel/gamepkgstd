# Work log / handoff

## Current tip

- Base commit: `21382b841669253dac16c4a6d584384df46534a6`
- Current goal: maintain a generic, LLM-actionable contract for cross-platform packaging/runtime behavior and a separate evidence-based compliance gate.
- Latest completed change: directory locations moved from `DESIGN.md` section 4 into `DIRECTORIES.md`, with a per-platform matrix and rationale. Windows savegames now go to Roaming AppData (was Local); savegames on Linux stay in `XDG_STATE_HOME` (not `XDG_DATA_HOME`, which in practice fills with installed `/usr/share`-like data).
- Deliver a cumulative git bundle from the original base commit, following `AGENTS.md`.

## Current document roles

- `DESIGN.md`: normative generic contract and platform implementation guidance.
- `DIRECTORIES.md`: path categories, per-platform locations, and the reasoning for them.
- `GAME-DESIGN.md`: optional game-specific behavior.
- `PACKAGING-CHECKLIST.md`: sole compliance/release checklist.
- `LLM-IMPLEMENTATION-GUIDE.md`: inspect-plan-implement-verify-report workflow for coding agents.
- `VERSION-HANDLING.md`: version source and clean release revision rules.
- `NIX-DEVELOP-SHELL.md`: out-of-tree CMake development-shell pattern.

## Remaining follow-up ideas

- `DIRECTORIES.md`: verify the Android/web columns and the Pictures-folder screenshot rule against the reference projects (MOAS currently writes screenshots to its state directory).
- Validate the OS integration and man-page guidance against actual Linux package formats, Android APK/AAB packaging, and Windows portable/installer outputs; distinguish static metadata checks from UI launch tests.

- Validate the conventions against current reference-project implementations and link to concrete examples.
- Validate the Nix-first CI contract against reference-project flakes; do not claim target builds or device tests without running them.
- Consider a machine-readable support matrix if multiple projects adopt the contract.
- Define a shared add-on manifest schema only after validating it against real projects.

## Handoff

Preserve user data and existing behavior when applying the contract to another project. Record failures and unknowns instead of editing the checklist to imply compliance. Run relevant checks and state exactly what was and was not verified.
