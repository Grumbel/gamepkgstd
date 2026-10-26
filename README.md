# Cross-Platform Packaging Design Notes

Reusable rules for packaging, paths, CLI, configuration, cross-compilation, extensions, versioning, and deployment across applications, games, libraries, command-line tools, and web projects. The initial examples came from several games, but the core contract is generic.

## Contents

| File | Purpose |
|------|---------|
| **DESIGN.md** | Generic build, path, CLI, configuration, extension, packaging, and Nix-first CI contract |
| **VERSION-HANDLING.md** | Rules for the top-level `VERSION` file, CMake and Nix |
| **NIX-DEVELOP-SHELL.md** | Reusable `nix develop` out-of-tree CMake pattern |
| **PACKAGING-CHECKLIST.md** | Sole compliance and release gate for native/cross builds, runtime smoke tests, extensions, CI, and target artifacts |
| **LLM-IMPLEMENTATION-GUIDE.md** | Agent workflow for inspecting, fixing, testing, and reporting project compliance |
| **GAME-DESIGN.md** | Optional game-specific shortcuts, controller, menu, and graphics guidance |
| **AGENTS.md** | Session continuity, git-bundle delivery rules for agents |

## Source projects

- https://github.com/Pingus/pingus
- https://github.com/Grumbel/kurvenrausch
- https://github.com/Grumbel/dualthrust
- https://github.com/Grumbel/kugelmatch
- https://github.com/SuperTux/supertux (and Milestone 1)


## How to use these notes

Treat `DESIGN.md` as the generic engineering contract and `GAME-DESIGN.md` as optional game-specific guidance. When asking an LLM to modify a project, also provide `LLM-IMPLEMENTATION-GUIDE.md` and use `PACKAGING-CHECKLIST.md` as the evidence-backed gap report and release gate. `flake.nix` is the single source of truth for builds, tests, cross-compilation, packaging, and deployment; GitHub Actions is only a thin invocation shim. The default goal is a documented one-command build for every required artifact, plus a deployment command where applicable. Use `PACKAGING-CHECKLIST.md` as a release gate and mark unsupported targets N/A with a reason. Project-specific deviations should be documented rather than hidden in build scripts or workflow YAML.
