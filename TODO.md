# Work log / handoff

## Current tip

- Base commit: `21382b841669253dac16c4a6d584384df46534a6`
- Current work: strengthen shared game packaging guidelines and add a release checklist.
- Latest completed change: see the latest commit and `PACKAGING-CHECKLIST.md`.
- This follow-up should continue the same work line and deliver a cumulative bundle from the original base commit.

## Implemented in this work line

- Clarified normative levels and project-specific supported-platform profiles.
- Added explicit cross-compilation and release-build rules.
- Specified CLI/configuration precedence and persistence semantics.
- Added graphics recovery, input/menu behavior, config migration, diagnostics, and save-safety guidance.
- Added a practical release checklist covering native and cross-builds, installed-runtime smoke tests, Linux, Windows, web/Android/handheld targets, and release artifact hygiene.

## Remaining follow-up ideas

- Validate these conventions against current versions of the reference projects and link to specific implementation examples.
- Validate the Nix-first CI contract against each reference project flake; GitHub Actions should remain only a thin invocation shim.
- Consider a machine-readable per-project support matrix if multiple games adopt this specification.
- Validate add-on manifest and dependency conventions against real games before defining a shared schema.

## Handoff

Deliver changes as a cumulative git bundle rooted at the original base commit, following `AGENTS.md`. Do not mark target builds as tested unless they were actually run.

## Follow-up review completed

- Unified path-option spelling around hyphenated names and documented the legacy `--datadir` alias policy.
- Separated installed assets, user data/add-ons, config, state, cache, temporary files, and runtime files.
- Clarified XDG base-directory semantics and distinguished desktop Unix, Windows, portable builds, Android/web, and PortMaster-style handheld profiles.
- Added add-on discovery, manifest, trust, dependency, extraction-safety, and recovery requirements.
- Made `flake.nix` the single source of truth for CI build/test/package logic and reduced GitHub Actions to a thin shim.
- Reviewed wording for overgeneralized platform assumptions and added path/add-on/CI checklist items.
