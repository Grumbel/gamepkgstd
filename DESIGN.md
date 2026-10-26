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
| Env for dlopen | Mirror package wrapper (`LD_LIBRARY_PATH`, plugin paths, `--data-dir`) | Handled by wrapper |

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

Use the project's chosen license consistently, preserve third-party license notices, and follow REUSE/SPDX conventions where adopted or required by the project.

---

## 3. Path Handling

Keep the *kind of data* separate from the *physical directory used on a platform*. A platform may place several kinds in one writable root, but code and options must retain the distinction. Never infer a writable location from the current working directory.

### 3.1 Path categories

| Category | Contents | Desktop Linux / Unix default | Writable? |
|---|---|---|---|
| Installed assets | Shipped levels, images, audio, shaders, built-in content | Install prefix, commonly `share/<app>` | No |
| User data | User-installed mods/add-ons and content the user explicitly imports | `$XDG_DATA_HOME/<app>`; default `~/.local/share/<app>` | Yes |
| Config | Preferences, key bindings, UI settings | `$XDG_CONFIG_HOME/<app>`; default `~/.config/<app>` | Yes |
| State | Save games, progress, high scores, session history | `$XDG_STATE_HOME/<app>`; default `~/.local/state/<app>` | Yes |
| Cache | Re-creatable downloads, shader caches, thumbnails | `$XDG_CACHE_HOME/<app>`; default `~/.cache/<app>` | Yes |
| Temporary files | Incomplete downloads, extraction staging, disposable scratch output | OS temporary-directory API, with a private per-app subdirectory if needed | Yes, ephemeral |
| Runtime files | Locks, sockets, other short-lived coordination files | `$XDG_RUNTIME_DIR/<app>` when valid and available | Yes, ephemeral |

`XDG_DATA_HOME`, `XDG_CONFIG_HOME`, `XDG_STATE_HOME`, and `XDG_CACHE_HOME` are *base directories*, not the application's final directory. Append the application-specific directory exactly once. Honor an XDG variable only when it is set to an absolute path; otherwise use the specification's default. Do not treat `XDG_RUNTIME_DIR` as a general temp directory: it is optional, user-owned, private, and intended for runtime objects. Validate its ownership and permissions before using it. Use the platform's temporary-directory API for scratch files.

The XDG Base Directory Specification is a desktop Unix convention, not a universal path API. Do not impose XDG semantics on Windows, Android, web builds, or a handheld port merely because the code uses SDL.

### 3.2 One consistent path-option vocabulary

Use hyphenated long options consistently. The recommended overrides are:

| Option | Meaning |
|---|---|
| `--data-dir PATH` | Root of the shipped/read-only game assets; preferred replacement for legacy `--datadir` |
| `--user-data-dir PATH` | User-managed content, including installed mods and add-ons |
| `--config-dir PATH` | Preferences and input bindings only |
| `--state-dir PATH` | Saves, progress, scores, and other durable gameplay state |
| `--cache-dir PATH` | Re-creatable cache files |
| `--temp-dir PATH` | Scratch files for this process; mainly useful for diagnostics, testing, and constrained launchers |
| `--show-paths` | Print all resolved paths and their source, then exit |

These are optional common conventions, not a requirement to expose every override in every game. Keep the spelling consistent for every path option that is exposed. If compatibility requires the legacy `--datadir`, retain it as a deprecated alias for `--data-dir`, document the alias, and make both feed the same internal field. Do not silently reinterpret `--data-dir` as the user's writable content directory.

Path resolution order for each independently overridable category is:

1. Explicit CLI option for that category.
2. Documented application-specific environment override, if provided by the launcher.
3. Explicit portable-mode root, if portable mode is enabled.
4. Native platform default.

Do not combine these sources ad hoc. `--config-dir` must not relocate saves, mods, cache, or temp files. A portable root may intentionally group categories below one root, but they should still use separate subdirectories. `--show-paths` should report the resolved location and whether it came from CLI, environment, portable mode, or a platform default.

### 3.3 Linux and Unix desktop: XDG

For a normal installed desktop build, use the XDG paths in the category table above. Shipped assets are found through the install layout or an explicit `--data-dir`; they do not belong in a writable XDG directory. Do not write into the install prefix or current working directory by default.

Use the home directory only to construct the documented fallback when the relevant XDG variable is unset or invalid. If neither a valid base directory nor a usable home directory can be resolved, report the problem rather than falling back to a relative path. Create application subdirectories as needed and report actionable errors when they cannot be created.

### 3.4 Windows

For an installed build, use Windows known-folder APIs (or a library abstraction that maps to them) rather than assuming Unix home paths:

- Config: Roaming AppData, typically `%APPDATA%/<Vendor>/<App>`.
- State and user data: Local AppData by default, typically `%LOCALAPPDATA%/<Vendor>/<App>`; use Roaming AppData only when roaming is actually desired.
- Cache and temporary files: Local AppData cache subdirectory and the OS temporary-directory API, respectively.
- Installed assets: the package's installed data directory, resolved relative to the executable or through an installed-resource lookup.

SDL path helpers may be used where their behavior matches the intended category, but do not use one generic preference path for every kind of data. Do not write beside the executable in an ordinary installation.

### 3.5 Portable builds and removable installations

Portable mode is an explicit distribution or launch profile, not a guess based only on the executable's location. Enable it through a documented marker file, launcher setting, or `--portable` option. The package must state which mechanism is authoritative.

- Keep the shipped application and assets separate from writable user data.
- Put writable files below a documented directory such as `userdata/config`, `userdata/state`, `userdata/data`, `userdata/cache`, and `userdata/tmp` beside the portable package, or under a launcher-supplied writable root.
- Never assume the executable directory is writable. If portable storage is read-only or full, explain the failure and offer the native user-directory mode where possible; do not silently start writing elsewhere.
- Portable mode must not unexpectedly import or modify the installed version's normal user profile. Make profile selection visible in diagnostics.
- Preserve paths containing spaces and non-ASCII characters. Do not build paths by shell-concatenating untrusted strings.

### 3.6 R36S / PortMaster / ArkOS and similar handheld ports

Treat these as a distinct *portable launcher profile*, even though the underlying system may be Linux. Follow the platform's current launcher contract rather than applying desktop XDG rules blindly.

- Prefer a documented launcher-provided path override for writable files. Use a project-specific variable or a shared `APP_*_DIR` convention, and define whether it names a base root or a final category directory.
- If the PortMaster contract supplies a final `conf/` directory through a variable such as `XDG_CONFIG_HOME` or a launcher-specific variable, do not append another application directory to it. This exception must be explicit in the port integration, not inferred globally from a path merely ending in `conf`.
- Keep save data and user content in the launcher/platform's expected writable location; do not assume the binary's directory is writable or that the current working directory is stable.
- Document the actual layout, environment overrides, and migration behavior in the port README and launcher script.
- Keep the runtime library environment intentional. Do not let a host `LD_LIBRARY_PATH` accidentally select incompatible SDL, GLES, or other shared libraries; clear or set it in the launcher only when required by the target runtime.

### 3.7 Android and web

- **Android:** use the app's sandboxed internal storage APIs for writable config/state/cache, and platform-approved external storage only when the feature and permissions require it. Treat APK assets as read-only virtual assets; do not assume `$HOME`, a normal executable-relative `data/` directory, or unrestricted `/sdcard` access.
- **Emscripten/web:** use the virtual filesystem and a documented mount for persistent data. Distinguish in-memory temporary files from persisted storage, and flush asynchronous persistence (for example IndexedDB-backed storage) according to the runtime's API. Do not promise persistence until the flush completes.

### 3.8 Asset lookup and virtual assets

Asset lookup must be deterministic. A typical order is: explicit `--data-dir`, documented application-specific environment override, package-relative asset location when the distribution layout defines one, then a compiled install location. Do not silently fall back to an unrelated directory or the current working directory. A virtual asset abstraction such as `datadir://...` may be useful for APK/web assets, but its scheme and semantics must be documented and must not leak into ordinary filesystem APIs.

### 3.9 Path correctness and migration

- Normalize and validate paths at the platform boundary, but preserve the user's intended path and avoid resolving symlinks unless required.
- Use native path APIs rather than assuming `/` separators or UTF-8 conversion behavior on every platform.
- Make directory creation idempotent. Use restrictive permissions for private saves/config where supported.
- Temporary files must be uniquely and safely created; never use predictable names in shared temp directories. For atomic replacement of a durable file, create the staging file in the destination directory so the final rename can remain on the same filesystem. `--temp-dir` does not redirect these atomic-save staging files.
- When a release changes a path convention, migrate or continue reading the old location deliberately. Never delete the old directory automatically. If both old and new locations contain data, define a deterministic conflict policy and explain it in release notes.

---

## 4. Keyboard Shortcuts (shared conventions)

| Key | Action | Notes |
|-----|--------|-------|
| **F11** | Toggle fullscreen | Universal across kugelmatch, kurvenrausch, dualthrust, SuperTux |
| **Alt+Enter** | Toggle fullscreen | Same as F11; common Windows / SDL expectation |
| **F8** | Optional renderer/backend debug cycle | Only for games with multiple backends; document it and do not treat it as a universal shortcut |
| **F12** | Screenshot | Use the platform pictures directory (XDG user-dir lookup on Linux); do not default to CWD |
| **Esc** | Pause → menu → quit (layered) | Never hard-quit on first press while in-game |
| **Enter / Space** | Confirm / start / respawn | |
| **P** | Pause | |
| **M** | Mute | |
| **+/−** or **[/]** | Volume | |

Fullscreen should restore the previous windowed size and position when possible. Prefer “desktop” fullscreen (borderless) over exclusive mode unless the user asks for exclusive.

Treat these as suggested defaults, not universal requirements. Preserve established project controls where changing them would surprise players, and document intentional deviations.

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
  --data-dir PATH         Override shipped/read-only asset directory
  --user-data-dir PATH    Override user-installed content directory
  --config-dir PATH       Override preferences directory
  --state-dir PATH        Override saves and durable gameplay state
  --cache-dir PATH        Override re-creatable cache directory
  --temp-dir PATH         Override scratch-file directory
  --portable              Use the package-local writable profile, if supported
  --show-paths            Print resolved paths and exit
  --renderer auto|software|gles   (or --cpu / --gpu)
  --vsync on|off
  --fps N                 Target FPS when vsync off (0 = uncapped)
  --mute
  --volume VALUE          Use the documented range for this game

Developer / testing (do not write to the user config):
  --screenshot FRAME,PATH
  --demo / --attract
  --no-save               Disable persistent gameplay writes
```

CLI flags that set a preference should **override** the config file for that run but, by default, should **not** rewrite the file merely because the game starts. Track which settings were explicitly overridden in the parsed options structure so the distinction is explicit; a bitmask is not required.

Make resolved paths (assets, user data, config, state, cache, temp, and runtime when applicable) available at debug log level or through `--show-paths`; this dramatically reduces support friction on handhelds and Flatpaks.

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
| Nix flakes and `nix flake check` | https://nix.dev/manual/nix/2.18/command-ref/new-cli/nix3-flake-check |
| Windows known folders | https://learn.microsoft.com/en-us/windows/win32/shell/knownfolderid |

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
- `--show-paths` (or equivalent diagnostics) should report resolved asset, user-data, config, state, cache, temp, and runtime paths where applicable, without dumping secrets or unrelated environment variables.

### Precedence and persistence

Use this precedence for ordinary preferences:

1. Built-in defaults
2. Saved configuration
3. Explicit command-line overrides for this invocation

An explicit command-line option must not rewrite the saved configuration merely because the game starts. A change made in the options menu should be persisted according to the game's save policy. Make this distinction visible in code, preferably through a parsed options structure that records which values were explicitly overridden, rather than scattered ad-hoc checks.

`--config-dir`, `--data-dir`, `--state-dir`, and similar path overrides should affect only the paths they name. Do not let a config-directory override unexpectedly relocate saves unless that behavior is explicitly documented. `--no-save` should disable all persistent gameplay writes, including autosaves and high scores, and should be testable as a single contract.

### Suggested common option set

Treat these as candidates, not mandatory flags for every game:

- `-h, --help`, `-v, --version`
- `-f, --fullscreen`, `-w, --windowed`
- `--data-dir PATH`, `--user-data-dir PATH`, `--config-dir PATH`, `--state-dir PATH`, `--cache-dir PATH`, `--temp-dir PATH` (expose only the useful subset)
- `--renderer auto|...` only when multiple renderers are supported
- `--vsync on|off` and `--fps N` only when meaningful for the renderer
- `--mute`, `--volume VALUE` when audio is supported
- `--no-save` for safe test/demo runs
- `--show-paths` or `--diagnostics` for troubleshooting
- `--portable` only for builds that implement a documented portable profile

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

## 17. Mods and add-ons

Mods, custom levels, translations, themes, and other add-ons are user-managed content, not shipped assets and not configuration. Keep them in the user-data category unless a specific platform requires a different physical layout.

- Define the supported add-on types and whether each is data-only or executable. Prefer declarative data formats. Do not treat arbitrary scripts, native libraries, or downloaded binaries as harmless content.
- Keep built-in content, user-installed content, and temporary workshop/download staging separate. Shipped add-ons belong to the package; user-installed add-ons belong in the user-data directory; caches and incomplete downloads belong in cache/temp directories.
- Define deterministic discovery and precedence. A reasonable default is explicit command-line selection, enabled user add-ons, then built-in content, but the game must document the actual order. Do not let filesystem enumeration order determine load order; use a stable sort or explicit priority/dependency metadata.
- Provide a manifest with a stable identifier, display name, version, supported game/API version, content type, and declared dependencies where applicable. Validate identifiers and paths. Detect duplicate IDs, missing dependencies, incompatible versions, and dependency cycles with actionable errors.
- Make enable/disable and load order discoverable in the UI or documented config. If load order affects compatibility, show it and make it reproducible. Do not silently overwrite one add-on with another sharing the same ID.
- Do not execute add-on code by default. If executable mods are supported, document the trust boundary and permissions; do not imply that a sandbox exists unless one is actually enforced. Warn before loading untrusted code and avoid automatic execution of newly downloaded content.
- Downloads and extraction must defend against path traversal, absolute paths, symlinks escaping the destination, archive bombs, and partial/corrupt packages. Extract into a staging directory, validate, then install atomically. Do not overwrite unrelated user files.
- Store add-on settings separately from global preferences where practical. Uninstalling an add-on must not delete saves or user-created content without an explicit confirmation. Define what happens to a save that references a missing or disabled add-on.
- Provide diagnostics that list add-on IDs, versions, enabled state, load order, and compatibility errors without dumping private paths or user content. A safe mode should be able to start without third-party add-ons to recover from a broken installation.
- Test with no add-ons, one valid add-on, duplicate IDs, missing dependencies, incompatible versions, malformed manifests, and an add-on that fails to load. Document whether updates preserve user settings and whether old add-on versions are backed up.

## 18. Nix-first CI and GitHub integration

The Nix flake is the single source of truth for build, test, lint, package, and cross-compilation logic. CI must invoke the flake rather than reimplementing those steps in workflow YAML or separate shell scripts that drift from local builds.

- Put buildable packages in `packages`, automated checks in `checks`, and reusable development tools in `devShells` as appropriate. Checks should encode the commands and dependencies they need as derivations so they run in a clean, reproducible environment.
- Make the intended CI entry point explicit, for example `nix flake check` or a named `checks.<system>.ci` derivation. `nix flake check` builds derivations in `checks`, but does not automatically build every derivation in `packages`; include required package builds as checks/dependencies or explicitly select them in the documented flake entry point. Merely evaluating the flake is not equivalent to building all packages or running all tests.
- GitHub Actions is a thin shim: provision the required Nix version/features, configure caching if desired, then invoke the documented flake entry point. Keep platform matrices and target selection minimal and declarative; do not duplicate compiler flags, dependency lists, build commands, or test logic in YAML.
- For cross-compilation, expose target packages/checks through the flake. CI should build those outputs from the flake, and only run target executables when a suitable runner or explicitly configured emulator exists. Otherwise validate artifacts and record the manual test requirement.
- Keep secrets and signing credentials in the CI service, not in the flake. The flake describes reproducible build logic; publishing credentials and release permissions remain workflow concerns.
- Document the same local commands that CI uses. A developer should be able to reproduce a failed CI check with `nix build` / `nix flake check` without reverse-engineering workflow YAML.
- Avoid a second authoritative build script. Small GitHub workflow steps that install/configure Nix, select a flake output, or upload an already-built artifact are acceptable shims; build and test behavior belongs in the flake.

## 19. Packaging validation checklist

Use `PACKAGING-CHECKLIST.md` for the release gate and artifact matrix. The short rule is: a package is not complete merely because it compiled. It must install, locate its data, create writable directories, start from an unrelated working directory, and expose the expected version and diagnostics on the target platform.
