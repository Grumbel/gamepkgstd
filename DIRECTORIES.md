# Directories

`<game>` is the lowercase game name. `<app-id>` is the reverse-DNS ID, e.g.
`io.github.grumbel.kurvenrausch`.

## Install layout

**DIR-1.** Linux (Nix, distro packages), relative to the install prefix:

```
bin/<game>
share/<game>/                                   game data
share/applications/<app-id>.desktop
share/metainfo/<app-id>.metainfo.xml
share/icons/hicolor/<size>x<size>/apps/<app-id>.png   (and/or scalable/…/.svg)
share/man/man6/<game>.6
```

Games use man section 6, not 1.

**DIR-2.** Windows zip: `<game>.exe`, its DLLs and a `data/` directory side by
side in the top-level directory (FLAKE-3).

**DIR-3.** Game data is found in this order: `--data-dir`, then a path
relative to the executable (`../share/<game>` on Linux, `data/` on Windows),
then the compiled-in install path. Never the current working directory.

## User files

**DIR-4.** Categories. Code keeps these separate even where they share a
directory:

| Category | Contents |
|----------|----------|
| config | Settings, key bindings |
| state | Savegames, autosave/resume, high scores, statistics, save states |
| data | User-installed levels, mods, add-ons |
| cache | Anything that can be regenerated |
| screenshots | F12 output |

Savegames are **state**, never config, never cache.

**DIR-5.** Locations:

| Category | Linux | Windows | Android |
|----------|-------|---------|---------|
| config | `$XDG_CONFIG_HOME/<game>` | `%APPDATA%\<game>\config` | `<internal>/config` |
| state | `$XDG_STATE_HOME/<game>` | `%APPDATA%\<game>\state` | `<internal>/state` |
| data | `$XDG_DATA_HOME/<game>` | `%LOCALAPPDATA%\<game>\data` | `<internal>/data` |
| cache | `$XDG_CACHE_HOME/<game>` | `%LOCALAPPDATA%\<game>\cache` | `getCacheDir()` |
| screenshots | `$(xdg-user-dir PICTURES)/<game>` | Pictures known folder `\<game>` | state `/screenshots` |

XDG defaults: `~/.config`, `~/.local/state`, `~/.local/share`, `~/.cache`.
`<internal>` is the app's internal storage (`SDL_AndroidGetInternalStoragePath()`).
Web: see EMSCRIPTEN.md (WEB-5). PortMaster: see DIR-10.

**DIR-6.** An XDG variable is used only if set to an absolute path; otherwise
use the default. Append `<game>` exactly once. If neither the variable nor
`$HOME` is usable, report an error; never fall back to a relative path.

**DIR-7.** Savegames go to `XDG_STATE_HOME`, not `XDG_DATA_HOME`. Reason:
`~/.local/share` fills up with installed things (Flatpak, Steam, fonts,
icons), `~/.local/state` holds what programs produce. Consequence: do not use
`SDL_GetPrefPath()` on Linux for config or state; it returns `XDG_DATA_HOME`.

**DIR-8.** Windows: config and state go to Roaming (`%APPDATA%`) because they
must move with the user to a new PC; only cache and large re-obtainable data go
to Local. Use `SHGetKnownFolderPath()` or the environment variables.

**DIR-9.** Never write to the install directory, the executable's directory or
the current working directory, except in portable mode (DIR-11).

**DIR-10.** PortMaster / handhelds: the launcher script sets the paths, the game
does not detect PortMaster. The launcher exports

```sh
export XDG_CONFIG_HOME="$GAMEDIR/conf/config"
export XDG_STATE_HOME="$GAMEDIR/conf/state"
export XDG_DATA_HOME="$GAMEDIR/conf/data"
export XDG_CACHE_HOME="$GAMEDIR/conf/cache"
```

and the game applies its normal Linux rules. No guessing from path names
(`/ports/`, `conf/`) in game code.

**DIR-11.** Portable mode is off unless enabled by `--portable` or a
`portable.txt` file next to the executable. It then uses
`<exe dir>/userdata/{config,state,data,cache}` and fails with an error if that
is not writable.

**DIR-12.** Each category can be overridden individually (OPT-3). Overriding
one never moves another.

## Writing files

**DIR-13.** Directories are created on first write. A failure names the path
and is shown to the player; never report a save as successful when it failed.

**DIR-14.** Savegames and config are written atomically: write
`<file>.tmp` in the same directory, flush, rename over `<file>`.

**DIR-15.** Changing a location is a migration: read the old location if the
new one is empty, never delete the old one, mention it in the release notes.
