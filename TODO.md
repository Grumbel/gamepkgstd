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
INTEGRATION.md (launchers, icons, app ID) was added back on request.
Deliberately out of scope now: Android lifecycle, mods/add-on manifests, save-state overlay menus,
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

- kurvenrausch: no 48x48 hicolor icon (INT-6), no adaptive Android icon
  (INT-14), `.rc` has no `VERSIONINFO` (INT-11), flake has no `checks`
  (FLAKE-9).
- kurvenrausch, kugelmatch: reverse-DNS IDs on Linux
  (`io.github.grumbel.kurvenrausch.desktop`, `org.kugelmatch.KugelMatch`
  metainfo); INT-1 says plain `<game>`.
- kurvenrausch, kugelmatch: Android package `io.github.grumbel.<game>`;
  INT-1 says `org.grumbel.<game>`. Not in any store, so switching only means
  reinstalling; copy saves out of app storage first.
- dualthrust, kugelmatch: no `checks` (FLAKE-9).
- pingus: `checks = packages // appChecks` already matches FLAKE-9/10; extra
  checks of FLAKE-11 missing.

## R36S.md (2026-10-29)

- Added from the xanatoo R36S build: C++20 against libstdc++ 9 works for
  language features (no newer headers), SQLite via eoan debs + `libs/`,
  no libvips. The C++20 xanatoo build runs on the device.
- STACK-1 says C++17 or newer; ARK-6 bounds what C++20 an R36S game can use.

## Open questions

- OPT-5 config key names and OPT-7 `.bad` handling are new conventions; no
  game implements them yet. Confirm before mass-applying.
- FLAKE-9 makes `nix flake check` build every target (slow first run,
  cached afterwards). Alternative would be native + tests only, with a
  separate aggregate for all targets.
- DIR-5 Windows/Android `config`/`state` subdirectories are new; existing
  games may need DIR-15 migration.
