# Work log / handoff

## Current tip

- Base commit: `21382b841669253dac16c4a6d584384df46534a6`
- Current work: strengthen shared game packaging guidelines and add a release checklist.
- Latest completed change: see the latest commit and `PACKAGING-CHECKLIST.md`.

## Implemented in this work line

- Clarified normative levels and project-specific supported-platform profiles.
- Added explicit cross-compilation and release-build rules.
- Specified CLI/configuration precedence and persistence semantics.
- Added graphics recovery, input/menu behavior, config migration, diagnostics, and save-safety guidance.
- Added a practical release checklist covering native and cross-builds, installed-runtime smoke tests, Linux, Windows, web/Android/handheld targets, and release artifact hygiene.

## Remaining follow-up ideas

- Validate these conventions against current versions of the reference projects and link to specific implementation examples.
- Add optional CI examples for native and cross-target package validation once a preferred CI system is selected.
- Consider a machine-readable per-project support matrix if multiple games adopt this specification.

## Handoff

Deliver changes as a cumulative git bundle rooted at the original base commit, following `AGENTS.md`. Do not mark target builds as tested unless they were actually run.
