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
