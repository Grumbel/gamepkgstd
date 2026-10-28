# Work log / handoff

## Current tip

- Base commit: `21382b841669253dac16c4a6d584384df46534a6`
- Branch `restructure-standard` (from `4b6cab1`): the long contract documents
  were replaced by eight short rule files with rule IDs (see README.md).
  Removed: DESIGN.md, GAME-DESIGN.md, PACKAGING-CHECKLIST.md,
  LLM-IMPLEMENTATION-GUIDE.md, NIX-DEVELOP-SHELL.md (content folded into the
  new files or dropped as out of scope; still in git history).
  VERSION-HANDLING.md became VERSIONING.md.

## Scope

Only: flake output names, install/user directories, shortcuts, shared
settings/options, versioning, `nix develop`, language/libraries, Emscripten.
Deliberately out of scope now: desktop-integration details beyond install
paths, Android lifecycle, mods/add-on manifests, save-state overlay menus,
release/CI checklists, accessibility.

## Known deviations in reference games (found while writing the rules)

- kurvenrausch `src/state.cpp`: detects PortMaster from `conf`/`/ports/` path
  names and `KURVENRAUSCH_STATE_DIR` (violates DIR-10).
- kurvenrausch web export file is `kurvenrausch-records.json` (WEB-8 says
  `<game>-saves.json`).
- dualthrust: man page in section 1 (DIR-1 says 6); dev shell uses
  `PROJECT_SOURCE` instead of `DUALTHRUST_SOURCE` (DEV-3).
- kugelmatch: dev shell lacks `-build` and `-run-gdb` (DEV-1);
  `kugelmatch-r36s` is the PortMaster build (FLAKE-1 wants
  `-r36s-portmaster`).
- pingus: `wasm-sdl2`, `android-sdl-libs` (FLAKE-2 wants `sdl2-wasm`,
  `sdl2-android`); Windows zip has no top-level directory (FLAKE-3).

## Open questions

- OPT-5 config key names and OPT-7 `.bad` handling are new conventions; no
  game implements them yet. Confirm before mass-applying.
- DIR-5 Windows/Android `config`/`state` subdirectories are new; existing
  games may need DIR-15 migration.
