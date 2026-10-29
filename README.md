# gamepkgstd — packaging and runtime standard for games

A short set of rules that every game in this family follows, so that builds,
install layout, savegame locations, shortcuts and command line options are the
same everywhere.

## Files

| File | Covers |
|------|--------|
| [FLAKE.md](FLAKE.md) | `flake.nix` output names, what `nix flake check` builds and tests |
| [DIRECTORIES.md](DIRECTORIES.md) | Install layout; config, savegame, cache locations per platform |
| [INTEGRATION.md](INTEGRATION.md) | App ID, `.desktop`, AppStream, icons on Linux, Windows, Android, PortMaster, web |
| [SHORTCUTS.md](SHORTCUTS.md) | Keyboard and gamepad shortcuts every game provides |
| [OPTIONS.md](OPTIONS.md) | Shared settings and command line options |
| [VERSIONING.md](VERSIONING.md) | `VERSION` file, dev/release version strings, CMake and Nix |
| [DEVELOPMENT.md](DEVELOPMENT.md) | Running and debugging through `nix develop` |
| [STACK.md](STACK.md) | Language, libraries and APIs |
| [EMSCRIPTEN.md](EMSCRIPTEN.md) | WebAssembly builds: main loop, Asyncify, persistent storage |
| [R36S.md](R36S.md) | R36S / ArkOS cross builds: eoan sysroot, C++20 and libraries that work |

## Rule format

- Every rule has an ID (`DIR-3`). Cite IDs in audits, commits and code comments.
- Rules are mandatory. "Should" marks a default that a game may deviate from
  if it documents why in its README.
- A rule that does not apply to a game (e.g. no Android port) is skipped, not
  worked around.

## Auditing a game

To check a game against this standard (by hand or with an AI agent):

1. Read every file listed above.
2. Determine which targets the game builds (from its `flake.nix`).
3. For each rule that applies, find the code or build file that implements it.
   Record **pass**, **fail** (with file and line), or **unknown** (could not
   tell without running it). Do not mark something as passing because it
   compiles.
4. Fix failures in small, separate commits. Path changes are data migrations:
   keep reading the old location (DIR-15).
5. Report the result as a table of rule ID, status and evidence.

Example prompt:

> Audit this game against /path/to/gamepkgstd (start with README.md). Produce
> the rule table, then fix the failures one commit per rule.

## Reference games

[Pingus](https://github.com/Pingus/pingus),
[Kurvenrausch](https://github.com/Grumbel/kurvenrausch),
[dualthrust](https://github.com/Grumbel/dualthrust),
[KugelMatch](https://github.com/Grumbel/kugelmatch),
[SuperTux](https://github.com/SuperTux/supertux).
