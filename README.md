# Game Packaging Design Notes

Condensed design document describing shared packaging, path handling,
input conventions, CLI, menus, versioning and build patterns used across
Pingus, Kurvenrausch, dualthrust, KugelMatch and SuperTux.

## Contents

| File | Purpose |
|------|---------|
| **DESIGN.md** | Main design document (paths, shortcuts, gamepad, CLI, menus, add-ons, packaging, CI) |
| **VERSION-HANDLING.md** | Rules for the top-level `VERSION` file, CMake and Nix |
| **NIX-DEVELOP-SHELL.md** | Reusable `nix develop` out-of-tree CMake pattern |
| **PACKAGING-CHECKLIST.md** | Release gate for native/cross builds, runtime smoke tests, add-ons, CI, and target packages |
| **AGENTS.md** | Session continuity, git-bundle delivery rules for agents |

## Source projects

- https://github.com/Pingus/pingus
- https://github.com/Grumbel/kurvenrausch
- https://github.com/Grumbel/dualthrust
- https://github.com/Grumbel/kugelmatch
- https://github.com/SuperTux/supertux (and Milestone 1)


## How to use these notes

Treat `DESIGN.md` as the shared behavior and engineering contract. The Nix flake is the source of truth for build and CI logic; GitHub Actions is only a thin Nix invocation shim. Use `PACKAGING-CHECKLIST.md` as a release gate and mark genuinely unsupported targets as N/A with a reason. Project-specific deviations should be documented rather than hidden in build scripts or workflow YAML.
