# Game Packaging & Surrounding Infrastructure Design Document

This document condenses common patterns observed across:

- [Pingus](https://github.com/Pingus/pingus)
- [Kurvenrausch](https://github.com/Grumbel/kurvenrausch)
- [dualthrust](https://github.com/Grumbel/dualthrust)
- [KugelMatch](https://github.com/Grumbel/kugelmatch)
- SuperTux (modern and Milestone 1)

Focus is **packaging, paths, input conventions, CLI, menus, build/versioning** — not core gameplay.

Related documents in this repository:

- `VERSION-HANDLING.md` — version number rules (source of truth `VERSION`, `-dev` suffix, Nix/CMake)
- `NIX-DEVELOP-SHELL.md` — out-of-tree CMake develop-shell pattern (`project-configure` / `project-build` / `project-run`)
- `AGENTS.md` — agent/session continuity, git-bundle delivery rules

Where implementations conflict, the recommendations below prefer **XDG Base Directory**, **portable CLI**, **explicit platform special-cases**, and practices that work on desktop Linux, Windows, Android, Emscripten, and handhelds (R36S / PortMaster / ArkOS).

---

## 1. Version Number Handling

See `VERSION-HANDLING.md` for the full ruleset. Summary of best practice:

| Context | Rule |
|---------|------|
| Source of truth | Top-level plain-text `VERSION` (e.g. `0.2.0-dev`) |
| Development | Append `.{revCount}+g{shortRev}` (and `-dirty` when applicable) |
| Release | Use `VERSION` as-is (no git metadata) |
| CMake `project(VERSION …)` | Numeric prefix only |
| Nix | `self.revCount or 0`, pass full string as `-DPROJECT_VERSION_FULL=…` |
| Application | Expose full string via `--version` / About |

Do not hard-code the version in `CMakeLists.txt`, headers, or the flake.

External references:

- [Semantic Versioning](https://semver.org/) (pre-release / build metadata)
- [XDG Base Directory](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html) (orthogonal but often co-located with versioned config dirs)

---

## 2. Build System

### Preferred stack

- **CMake** (≥ 3.10, ideally 3.16+) as the primary build system
- **Nix flake** for reproducible builds, cross-compilation (Windows via MinGW, R36S/ArkOS sysroot), and `nix develop` shells
- Optional: Ninja generator, ccache via `ccacheStdenv` in the develop shell

### Develop vs package split

Follow the pattern in `NIX-DEVELOP-SHELL.md`:

| Concern | Develop (`nix develop`) | Package (`nix build`) |
|---------|-------------------------|------------------------|
| Build type | `Debug` | `RelWithDebInfo` (or Release) + `separateDebugInfo` |
| Build location | Out-of-tree, sticky (`/tmp/project-build` or `$HOME/.cache/…`) | Nix store |
| Helpers | `project-configure`, `project-build`, `project-run`, `project-run-gdb` as real PATH scripts | Wrappers via `makeWrapper` / `qtWrapperArgs` |
| Nested C++ deps | Live sibling checkouts when possible | Flake inputs / fixed store paths |
| Env for dlopen | Mirror package wrapper (`LD_LIBRARY_PATH`, plugin paths, `--datadir`) | Handled by wrapper |

### Packaging artefacts

Typical install layout (prefix-relative):

```
bin/<game>
share/<game>/…          # data, shaders, …
share/applications/<game>.desktop
share/icons/hicolor/…/<game>.png (or .svg)
share/metainfo/<game>.appdata.xml  (or .metainfo.xml)
share/man/man6/<game>.6
```

Also produce:

- Flatpak / AppStream metainfo
- Windows zip / installer (NSIS or simple zip from Nix)
- PortMaster / R36S port package (`*.sh` launcher + `conf/` next to the binary)

REUSE / SPDX headers on every source file; GPLv3+ for new projects unless the existing tree uses something else.

---

## 3. Path Handling

### 3.1 XDG Base Directory (desktop Linux & most Unix)

| Kind | Variable | Fallback | Example |
|------|----------|----------|---------|
| Config | `$XDG_CONFIG_HOME` | `~/.config` | `~/.config/<app>/config` |
| State / savegames / stats | `$XDG_STATE_HOME` | `~/.local/state` | `~/.local/state/<app>/stats` |
| Cache | `$XDG_CACHE_HOME` | `~/.cache` | rarely needed for these games |
| Data (read-only assets) | install prefix or `--datadir` | next to binary / `share/<app>` | |

**Best practice observed in dualthrust / kurvenrausch / kugelmatch:**

- Config → `$XDG_CONFIG_HOME/<app>/…`
- Mutable state (saves, stats, high-scores, fog-of-war) → `$XDG_STATE_HOME/<app>/…`
- Never write into the install prefix or the current working directory by default.
- Create parent directories with `mkdir -p` semantics; fail loudly if an absolute path cannot be derived (do not guess a relative location).

Pingus historically used a single user dir (with a versioned subdirectory `pingus-0.8/`) and still supports a legacy `~/.pingus/` for compatibility; newer projects prefer the split config/state model.

### 3.2 Windows

- Prefer `%APPDATA%\<VendorOrApp>\` for config and `%LOCALAPPDATA%\<App>\` for state, or the same XDG-style layout under the user profile when a portable XDG helper is present.
- SDL’s `SDL_GetPrefPath` / `SDL_GetBasePath` are acceptable fallbacks.
- Avoid writing next to the `.exe` unless the user explicitly chooses a portable mode.

### 3.3 Android

- Writable: `SDL_AndroidGetInternalStoragePath()` + `/<app>/` (or the equivalent JNI path).
- Assets: empty “datadir” that resolves against the APK asset tree (`datadir://…` style virtual paths).
- Do **not** attempt classic Unix `$HOME` or `/sdcard` without explicit permission handling.

### 3.4 Emscripten / Web

- Virtual FS (IDBFS / MEMFS). Persist under a fixed mount such as `/<app>-user/`.
- Call an explicit flush (IndexedDB sync) on quit / periodic save.

### 3.5 R36S / PortMaster / ArkOS handhelds

Special rules observed in kurvenrausch / dualthrust / Pingus (`PINGUS_NO_XDGCPP`):

1. Prefer an environment override such as `$KURVENRAUSCH_STATE_DIR` or `$APP_STATE_DIR` when set by the port launcher.
2. If the binary lives under `…/ports/<name>/`, keep state in `…/ports/<name>/conf/` (or `app_base/conf`) so a direct launch still works without XDG.
3. When `XDG_STATE_HOME` already ends in `conf/`, treat it as the final directory (PortMaster convention).
4. Fall back to `$HOME/.config/<app>/` or `$HOME/.local/state/<app>/` only when the above do not apply.
5. Clear a hostile system `LD_LIBRARY_PATH` that can pull the wrong SDL / GLES libraries; the ArkOS sysroot is used at link time.

Document the exact layout in `mk/r36s/README.md` (or equivalent) so PortMaster maintainers know where saves live.

### 3.6 Data / asset location

- Runtime search order (recommended):

  1. `--datadir PATH` (CLI)
  2. Environment variable `$<APP>_DATADIR` / `$PINGUS_DATADIR`
  3. Directory next to the executable (`SDL_GetBasePath()` + `data/`)
  4. Install prefix `share/<app>`
  5. Compile-time baked path (last resort)

- Virtual schemes (`datadir://…`) keep CMake/install independent of the physical layout and work on Android assets.

---

## 4. Keyboard Shortcuts (shared conventions)

| Key | Action | Notes |
|-----|--------|-------|
| **F11** | Toggle fullscreen | Universal across kugelmatch, kurvenrausch, dualthrust, SuperTux |
| **Alt+Enter** | Toggle fullscreen | Same as F11; common Windows / SDL expectation |
| **F8** | Cycle renderer / backend | Software ↔ GLES / CPU ↔ GPU (kurvenrausch, kugelmatch) |
| **F12** | Screenshot | Prefer `~/Pictures` when it exists, else CWD or XDG pictures dir |
| **Esc** | Pause → menu → quit (layered) | Never hard-quit on first press while in-game |
| **Enter / Space** | Confirm / start / respawn | |
| **P** | Pause | |
| **M** | Mute | |
| **+/−** or **[/]** | Volume | |

Fullscreen should restore the previous windowed size and position when possible. Prefer “desktop” fullscreen (borderless) over exclusive mode unless the user asks for exclusive.

These match common desktop HIG expectations (GNOME HIG, KDE HIG, Windows UX guidelines for media/games) even though the games themselves are not full desktop applications.

---

## 5. Gamepad / Joystick Handling

- Use SDL2 (or SDL3) game-controller API; prefer the **GameController** abstraction over raw Joystick for consistent button names (A/B/X/Y, triggers, sticks).
- Provide a **rebind UI** that stores bindings in the config file (kurvenrausch / dualthrust pattern: `key_<action> …` / `pad_<action> …` lines).
- Default mappings should work out-of-the-box for Xbox-style and PlayStation-style pads; document R36S / PortMaster button layout in the port README.
- Support simultaneous keyboard + gamepad; last input wins for focus, both can generate events.
- Optional: rumble / vibration when the hardware reports it (kurvenrausch stores a `rumble` preference).
- Hot-plug: open newly connected controllers; ignore disconnects gracefully.

For multi-player (SuperTux local multiplayer, KugelMatch 2P) keep per-player binding sets.

---

## 6. Command-Line Options

Common, portable set (observed across the projects; adopt the union of useful flags):

```
Usage: <game> [options]

  -h, --help              Show this help
  -v, --version           Print full version string and exit
  -f, --fullscreen        Start fullscreen
  -w, --windowed          Start windowed
  --datadir PATH          Override data directory
  --config-dir PATH       Override XDG config directory
  --renderer auto|software|gles   (or --cpu / --gpu)
  --vsync on|off
  --fps N                 Target FPS when vsync off (0 = uncapped)
  --mute
  --volume 0..10

Developer / testing (do not write to the user config):
  --screenshot FRAME,PATH
  --demo / --attract
  --no-save               Do not touch stats / world saves
```

CLI flags that set a preference should **override** the config file for that run but, by default, should **not** rewrite the file unless the user later changes the option inside the menu. Use a bitmask of “CLI overrides” (kugelmatch pattern) so the distinction is explicit.

Always print a short summary of resolved paths (config, state, datadir) at debug log level or behind a `--show-paths` flag; this dramatically reduces support friction on handhelds and Flatpaks.

---

## 7. Menu Structure & Options

A practical, shared skeleton that appears (with local names) in all of the examined games:

```
Main
├── Start / Continue / New Game
├── Options
│   ├── Video
│   │   ├── Fullscreen
│   │   ├── Resolution / Scale / HD vs SD
│   │   ├── Renderer / Backend
│   │   ├── VSync / Target FPS
│   │   └── UI Scale
│   ├── Audio
│   │   ├── Master / Music / SFX volumes
│   │   └── Mute
│   ├── Controls
│   │   ├── Keyboard rebind
│   │   ├── Gamepad rebind
│   │   └── Rumble on/off
│   ├── Game
│   │   ├── Difficulty
│   │   ├── Language (if applicable)
│   │   └── … game-specific toggles
│   └── Reset to defaults
├── Extras / Credits / High scores
└── Quit
```

In-game pause menu mirrors a subset (Resume, Options, Abort to title, Quit).

Persist every option that the player can change; do not reset debug toggles on every launch unless they are explicitly marked transient.

---

## 8. Packaging & Distribution Checklist

- [ ] `VERSION` file + CMake / Nix wiring (see `VERSION-HANDLING.md`)
- [ ] Desktop entry + icon + AppStream metainfo
- [ ] Man page (optional but cheap)
- [ ] `nix build` produces a FHS-like layout; `nix run` works
- [ ] Windows cross zip from the same flake
- [ ] R36S / PortMaster port (`mk/r36s/`, launcher script, conf dir)
- [ ] Flatpak manifest (or link to Flathub) when the audience justifies it
- [ ] REUSE.toml / SPDX headers
- [ ] `--version` prints the full development or release string
- [ ] Config and state directories created on first run; failures reported to stderr

---

## 9. External References

| Topic | Reference |
|-------|-----------|
| Config / state directories | [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html) |
| Desktop integration | [Desktop Entry Specification](https://specifications.freedesktop.org/desktop-entry-spec/desktop-entry-spec-latest.html) |
| App metadata | [AppStream](https://www.freedesktop.org/software/appstream/docs/) |
| Versioning | [Semantic Versioning](https://semver.org/) |
| Keyboard / windowing | SDL2 / SDL3 documentation; common practice F11 / Alt+Enter |
| Game controllers | SDL_GameController; Steam Input / SDL controller database |
| GNOME HIG (menus, shortcuts) | https://developer.gnome.org/hig/ |
| KDE HIG | https://develop.kde.org/hig/ |
| PortMaster | https://portmaster.games/ (handheld packaging conventions) |
| REUSE | https://reuse.software/ |
| Nix flakes | https://nixos.wiki/wiki/Flakes |

---

## 10. Implementation Notes & Anti-Patterns

**Do**

- Derive every version string from `VERSION`.
- Keep develop-shell helpers as real executables on `PATH`.
- Fail early when `PROJECT_SOURCE` / user directories cannot be resolved.
- Mirror package-wrapper environment variables inside `nix develop`.
- Document platform-specific path rules next to the code that implements them.

**Avoid**

- Writing saves into the install prefix or CWD by default.
- Hard-coding `project(… VERSION 0.1.0)`.
- Silent fallback to a relative path when `$HOME` / XDG is missing.
- Qt-only environment variables in non-Qt games.
- Bundling “delta-only” git history; always ship cumulative bundles (see `AGENTS.md`).

---

*Document generated by consolidating patterns from the listed repositories and the accompanying VERSION / Nix-develop / agent guidelines. Prefer the practices above when the projects diverge.*

## 11. Normative levels and project profiles

These notes describe a **shared baseline**, not a requirement to ship every platform or feature. Mark each rule in project-specific documentation as one of:

- **MUST**: required for a supported build or release; deviations need an explicit reason.
- **SHOULD**: the default recommendation; document a concrete reason when deviating.
- **MAY**: optional capability, implemented only when it fits the game.

Every game should declare its supported targets and input modes. Do not advertise a target merely because its toolchain can compile the code. A project can have a small desktop profile and add Windows, web, Android, or handheld profiles later.

## 12. Build and cross-compilation rules

- Keep configure, build, test, install, and package steps separable. The package must be built from an explicit source revision and must not depend on files in the developer's home directory.
- Use out-of-tree builds. Do not share a CMake build directory between host and target architectures, compilers, toolchains, or materially different build options.
- Cross-builds must use the target compiler, linker, sysroot, and target dependencies consistently. Build-time generators that execute during the build must be built for the **build host**, not the target. Make this distinction explicit for code generators and asset tools.
- Never run a target executable as an ordinary configure-time test during cross-compilation. Use a native host tool, a cross-compilation emulator when deliberately configured, or defer the test to target hardware/CI.
- Separate build-time tools from target runtime dependencies. Do not accidentally link a host library or pick up host headers through ambient environment variables.
- Provide a clear target selection convention, for example separate flake package attributes or documented cross-system outputs. Avoid a single ambiguous `build` script whose output architecture depends on hidden environment state.
- Treat compiler flags, feature switches, and dependency versions as part of the build definition. Do not silently use optional system libraries found on the build machine if that changes the intended package features.
- Test the installed package, not only the build-tree executable. Stage installation into a temporary prefix and run from an unrelated working directory to catch missing assets, accidental source-tree references, and incorrect relative paths.
- For targets that cannot be run in CI, validate the artifact format and architecture, inspect linked dependencies, and document the remaining manual smoke test. Do not report an unrun target test as passing.

### Release build expectations

A release artifact should be reproducible from a clean checkout with documented commands. Record the source revision, version, target, build options, and artifact names. Keep debug symbols available where practical, either in the artifact or as a separate debug package. Do not strip symbols before extracting or preserving separate debug information.

A release checklist should cover at least: clean configure/build, tests that are available, install-tree launch, version output, asset discovery, configuration/save directory creation, and a smoke test on each supported runtime family. Add signing, checksums, and reproducible-build verification where the distribution channel requires or supports them.

## 13. CLI and configuration contract

### General rules

- `--help` and `--version` must work without initializing graphics, audio, controllers, or loading game assets. They should exit successfully.
- Invalid options and invalid values must produce a concise error on stderr and a non-zero exit status. Help text must show accepted values and defaults where useful.
- Support `--` to end option parsing if positional arguments are supported. Do not reinterpret unknown options as filenames silently.
- Prefer long options with explicit values for settings (`--renderer gles`) over a growing collection of overlapping boolean aliases. Keep `-h` and `-v` for help and version; only add short aliases when they are unambiguous and stable.
- Paths supplied by the user must be treated as paths, not shell fragments. Quote paths in launcher scripts and preserve spaces and non-ASCII characters.
- Document whether a flag is startup-only, persistent, diagnostic, or intended for automation. Developer flags should not silently alter player saves or preferences.
- `--show-paths` (or equivalent diagnostics) should report resolved config, state, data, and save paths without dumping secrets or unrelated environment variables.

### Precedence and persistence

Use this precedence for ordinary preferences:

1. Built-in defaults
2. Saved configuration
3. Explicit command-line overrides for this invocation

An explicit command-line option must not rewrite the saved configuration merely because the game starts. A change made in the options menu should be persisted according to the game's save policy. Make this distinction visible in code, preferably through a parsed options structure that records which values were explicitly overridden, rather than scattered ad-hoc checks.

`--config-dir`, `--datadir`, and similar path overrides should affect only the paths they name. Do not let a config-directory override unexpectedly relocate saves unless that behavior is explicitly documented. `--no-save` should disable all persistent gameplay writes, including autosaves and high scores, and should be testable as a single contract.

### Suggested common option set

Treat these as candidates, not mandatory flags for every game:

- `-h, --help`, `-v, --version`
- `-f, --fullscreen`, `-w, --windowed`
- `--datadir PATH`, `--config-dir PATH`
- `--renderer auto|...` only when multiple renderers are supported
- `--vsync on|off` and `--fps N` only when meaningful for the renderer
- `--mute`, `--volume VALUE` when audio is supported
- `--no-save` for safe test/demo runs
- `--show-paths` or `--diagnostics` for troubleshooting

Do not expose every menu preference as a CLI option by default. Add flags when they help automation, debugging, accessibility, deployment, or reproducible testing.

## 14. Graphics and display settings

- `Auto` should be the default for renderer/backend selection unless the game has one supported backend. Detect capabilities at runtime; do not infer GPU features solely from the OS or device name.
- Unsupported settings should be hidden, disabled with an explanation, or rejected clearly. Never display an option as active when the requested mode was not actually created.
- If renderer initialization fails, report the failure and attempt a documented safe fallback where one exists. Preserve the requested setting only if the user can recover from it; otherwise revert to a known-good setting and explain what happened.
- A fullscreen/resolution change that could leave the display unusable should have a confirmation/revert flow when the platform allows it. Revert automatically after a short timeout unless confirmed. Where such a flow is not feasible, provide a documented reset path such as a safe-mode command-line flag or removal of the relevant config entry.
- Prefer borderless desktop fullscreen by default on desktop platforms unless exclusive fullscreen is needed. Respect the platform's available display modes and windowing model; web and handheld targets may expose fewer controls.
- Distinguish internal render resolution, window size, display resolution, pixel scaling, and UI scale. Do not label one as another. Preserve aspect ratio by default for pixel-art or fixed-aspect games; make stretching an explicit choice if offered.
- VSync and frame-rate limits should have predictable interaction. Explain whether the cap applies to simulation, rendering, or both. Avoid presenting an FPS cap as a fix for simulation timing bugs.
- Graphics settings should be persisted only after successful application when possible. If a change fails, restore the previous working value in both runtime state and the UI.
- Include a way to reset graphics settings without requiring a successful graphical startup. A `--safe-mode` or `--reset-video-settings` option is useful when the normal menu cannot be reached.

## 15. Input, focus, and menu behavior

- Every essential action must be reachable using the primary supported input method. Menus should support keyboard and controller navigation where those devices are supported; mouse-only UI is insufficient for a controller-first port.
- Show the currently focused item clearly. After a dialog closes, restore focus to the item that opened it where practical. Avoid relying on hover as the only indication of selection.
- Controller prompts should reflect the active controller mapping when practical. Do not assume all controllers label the same physical button as A/B or Cross/Circle; document whether prompts use SDL logical button names or physical positions.
- Handle controller add/remove events during play and menus. A disconnect must not crash the game or leave input permanently stuck. Provide a fallback input route where possible.
- Rebinding must detect conflicts and explain how they are resolved. Support canceling a rebind operation without losing the previous binding.
- Do not hard-code a shortcut that conflicts with text entry or common platform/window-manager behavior without a strong reason. List global shortcuts in the in-game help or controls screen.
- Escape/back should move one level outward or close the current dialog. In gameplay it may pause or open the pause menu; it should not immediately destroy progress. Make quit and abort actions distinct, and confirm destructive actions when there is meaningful unsaved progress.
- Pause menus should clearly distinguish Resume, Restart (if available), Abort/Return to title, and Quit. Do not show unavailable actions as if they were usable.
- Persist settings that players reasonably expect to survive restart. Clearly label temporary actions and per-session overrides. A reset-to-defaults action should state its scope and should not delete saves or high scores.

## 16. Configuration, saves, and diagnostics

- Keep user configuration, gameplay state (save games, progress, scores), cache, and immutable game assets conceptually separate even if a target platform forces them into one physical directory.
- Define what happens when configuration is missing, malformed, from a newer version, or contains unknown keys. Prefer safe defaults, a useful warning, and preservation or backup of the unreadable file over silently overwriting user data.
- Add configuration migrations when key names or meanings change. Avoid changing a setting's meaning while retaining the same key. Ignore unknown keys where feasible so newer configs can be opened by older builds without destroying data.
- Make directory-creation and write failures actionable: report the affected path and whether the game can continue without saving. Never claim a save succeeded if the write failed.
- Keep logs useful and bounded. Diagnostics should identify version/build, selected renderer, detected input devices when relevant, and resolved paths; avoid logging private data or full environment dumps by default.
- Save atomically where practical: write a temporary file in the destination directory, flush/close it, then rename it into place. Handle interrupted or failed writes without truncating the last valid save.

## 17. Packaging validation checklist

Use `PACKAGING-CHECKLIST.md` for the release gate and artifact matrix. The short rule is: a package is not complete merely because it compiled. It must install, locate its data, create writable directories, start from an unrelated working directory, and expose the expected version and diagnostics on the target platform.
