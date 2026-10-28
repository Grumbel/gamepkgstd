# Directory Locations Across Platforms

Where each kind of data lives on each target platform, and why. [`DESIGN.md`](DESIGN.md) section 4 defines the path-option vocabulary, resolution order, asset lookup, and path-correctness rules; this document defines the categories and their physical locations.

Keep the *kind of data* separate from the *physical directory used on a platform*. A platform may place several kinds in one writable root, but code and options must retain the distinction. Never infer a writable location from the current working directory or the executable's directory.

## Categories

| Category | Contents | Roams with the user? | Writable? |
|---|---|---|---|
| Installed assets | Shipped resources, templates, models, images, audio, shaders, built-in content | No (part of the install) | No |
| Config | Preferences, application settings, key bindings, UI settings | Yes | Yes |
| State | Savegames, autosave/resume files, saved documents, progress, high scores, history, and other durable user state the program creates | Yes | Yes |
| User data | User-installed mods/add-ons and content the user explicitly imports | Optional; may be large and re-obtainable | Yes |
| Cache | Re-creatable downloads, shader caches, thumbnails | No | Yes |
| Temporary files | Incomplete downloads, extraction staging, disposable scratch output | No | Yes, ephemeral |
| Runtime files | Locks, sockets, other short-lived coordination files | No | Yes, ephemeral |
| User documents | Files the user wants to find and handle outside the program, such as screenshots or exports | Per platform | Yes |

The test for "state" is: losing it destroys something the user made or earned, and it cannot be recreated. Savegames are the canonical example. Treat state as at least as valuable as config, never as disposable.

## Location matrix

`<app>` is the application's directory name. Windows paths may add a vendor level (`<Vendor>\<App>`); pick one and keep it stable across releases.

| Category | Linux / Unix desktop (XDG) | Windows | Android | Web (Emscripten) |
|---|---|---|---|---|
| Installed assets | Install prefix, commonly `share/<app>` | Package data directory, resolved relative to the executable | APK assets (read-only, virtual) | Preloaded/packaged virtual filesystem |
| Config | `$XDG_CONFIG_HOME/<app>`, default `~/.config/<app>` | `%APPDATA%\<app>` (Roaming) | Internal storage (`getFilesDir()` or equivalent) | Persistent mount, e.g. IndexedDB-backed |
| State | `$XDG_STATE_HOME/<app>`, default `~/.local/state/<app>` | `%APPDATA%\<app>` (Roaming), optionally a `state` subdirectory | Internal storage | Persistent mount, with export/import |
| User data | `$XDG_DATA_HOME/<app>`, default `~/.local/share/<app>` | `%LOCALAPPDATA%\<app>` (Local) | Internal storage, or app-specific external storage when size requires it | Persistent mount, if supported at all |
| Cache | `$XDG_CACHE_HOME/<app>`, default `~/.cache/<app>` | `%LOCALAPPDATA%\<app>\cache` (Local) | Cache directory (`getCacheDir()`) | In-memory or non-persisted mount |
| Temporary files | OS temporary-directory API, private per-app subdirectory if needed | OS temporary-directory API (`GetTempPath`) | Cache directory | In-memory filesystem |
| Runtime files | `$XDG_RUNTIME_DIR/<app>` when valid and available | Not usually needed; use named objects or Local AppData | Not usually needed | Not applicable |
| User documents (screenshots) | `XDG_PICTURES_DIR` from `xdg-user-dirs`, falling back to state | Pictures known folder (`FOLDERID_Pictures`) | MediaStore / shared storage APIs | Browser download |

Portable builds and handheld launchers override this matrix; see below.

## Linux and Unix desktop: XDG

`XDG_DATA_HOME`, `XDG_CONFIG_HOME`, `XDG_STATE_HOME`, and `XDG_CACHE_HOME` are *base directories*, not the application's final directory. Append the application-specific directory exactly once. Honor an XDG variable only when it is set to an absolute path; otherwise use the specification's default. Use the home directory only to construct that documented fallback. If neither a valid base directory nor a usable home directory can be resolved, report the problem rather than falling back to a relative path. Create application subdirectories as needed and report actionable errors when they cannot be created.

Do not treat `XDG_RUNTIME_DIR` as a general temp directory: it is optional, user-owned, private, and intended for runtime objects. Validate its ownership and permissions before using it.

Shipped assets are found through the install layout or an explicit `--data-dir`; they do not belong in a writable XDG directory. Do not write into the install prefix or current working directory by default.

### Why savegames go in `XDG_STATE_HOME`, not `XDG_DATA_HOME`

The specification describes `$XDG_STATE_HOME` as data that should persist between restarts but is "not important or portable enough to the user that it should be stored in `$XDG_DATA_HOME`", with examples like history and recent files. Read literally, that would put savegames in `XDG_DATA_HOME`, and `SDL_GetPrefPath()` and many Linux games do exactly that.

This contract deliberately chooses `XDG_STATE_HOME` instead, for practical reasons:

- In real use, `~/.local/share` behaves like a user-local `/usr/share`. Flatpak, Steam, pip, fonts, icons, `.desktop` files, and other installed material end up there. Savegames placed there are mixed with re-installable data.
- `~/.local/state` behaves like a user-local `/var/lib`: data the program itself produces and maintains. Savegames, autosave/resume files, and high scores fit that model. A resume file is literally the "current state of the application" the specification lists as an example.
- Both directories sit under `~/.local`, so backups and migrations that cover `~` or `~/.local` capture either one.

Consequences:

- Do not use `SDL_GetPrefPath()` for savegames on Linux; it resolves to `XDG_DATA_HOME`. It may be used for the user-data category.
- Because some players will look in `~/.local/share` first, print the resolved state directory in `--show-paths`/diagnostics and document it in the README and man page.
- Keep `XDG_DATA_HOME` for content the user brings in: mods, add-ons, imported levels.

## Windows

Use Windows known-folder APIs (`SHGetKnownFolderPath`) or the `%APPDATA%`/`%LOCALAPPDATA%` environment variables, which point to the same folders for an ordinary user session. Do not assume Unix home paths and do not write beside the executable in an ordinary installation.

### Roaming vs Local AppData

- **Roaming (`%APPDATA%`, `FOLDERID_RoamingAppData`)** follows the user between machines in a domain with roaming profiles, and is what users and migration tools carry over to a new PC. Use it for anything the user would want to take with them: config **and state, including savegames**.
- **Local (`%LOCALAPPDATA%`, `FOLDERID_LocalAppData`)** is bound to one machine. Use it for caches, large re-downloadable content, and data that is only meaningful on this machine (paths, hardware-specific settings, shader caches).

Do not map XDG categories to Windows folders mechanically ("state → Local, config → Roaming"). XDG state holds savegames in this contract, and savegames are exactly the data Roaming exists for. Roaming profiles have size quotas and sync on logon/logoff, so keep large or frequently rewritten data (caches, big mod collections, logs) in Local.

### Saved Games known folder

Windows also defines `FOLDERID_SavedGames` (`%USERPROFILE%\Saved Games`). It is the officially intended location and is user-visible, but few games use it, there is no environment variable for it, and it requires `SHGetKnownFolderPath`. Roaming AppData is an acceptable default; `Saved Games` is an acceptable alternative. Whichever is chosen, document it and do not change it between releases without a migration.

### SDL helpers

`SDL_GetPrefPath()` on Windows returns `%APPDATA%\<org>\<app>` (Roaming), which suits config and state. Use path helpers only where their behavior matches the intended category; do not use one generic preference path for every kind of data across platforms.

## Portable builds and removable installations

Portable mode is an explicit distribution or launch profile, not a guess based only on the executable's location. Enable it through a documented marker file, launcher setting, or `--portable` option. The package must state which mechanism is authoritative.

- Keep the shipped application and assets separate from writable user data.
- Put writable files below a documented directory such as `userdata/config`, `userdata/state`, `userdata/data`, `userdata/cache`, and `userdata/tmp` beside the portable package, or under a launcher-supplied writable root.
- Never assume the executable directory is writable. If portable storage is read-only or full, explain the failure and offer the native user-directory mode where possible; do not silently start writing elsewhere.
- Portable mode must not unexpectedly import or modify the installed version's normal user profile. Make profile selection visible in diagnostics.
- Preserve paths containing spaces and non-ASCII characters. Do not build paths by shell-concatenating untrusted strings.

## R36S / PortMaster / ArkOS and similar handheld ports

Treat these as a distinct *portable launcher profile*, even though the underlying system may be Linux. Follow the platform's current launcher contract rather than applying desktop XDG rules blindly.

- Prefer a documented launcher-provided path override for writable files. Use a project-specific variable or a shared `APP_*_DIR` convention, and define whether it names a base root or a final category directory.
- If the PortMaster contract supplies a final `conf/` directory through a variable such as `XDG_CONFIG_HOME` or a launcher-specific variable, do not append another application directory to it. This exception must be explicit in the port integration, not inferred globally from a path merely ending in `conf`.
- Keep save data and user content in the launcher/platform's expected writable location; do not assume the binary's directory is writable or that the current working directory is stable.
- Document the actual layout, environment overrides, and migration behavior in the port README and launcher script.
- Keep the runtime library environment intentional. Do not let a host `LD_LIBRARY_PATH` accidentally select incompatible SDL, GLES, or other shared libraries; clear or set it in the launcher only when required by the target runtime.

## Android

Use the app's sandboxed internal storage APIs for writable config and state, and the cache directory for cache. Use platform-approved external storage only when the feature and permissions require it. Treat APK assets as read-only virtual assets; do not assume `$HOME`, a normal executable-relative `data/` directory, or unrestricted `/sdcard` access. Persist durable progress incrementally and use platform instance-state mechanisms for transient UI/session restoration. Never depend on receiving a last-chance lifecycle callback before the OS kills the process. Internal storage is deleted on uninstall; say so where users might expect otherwise, and consider Android Auto Backup for small config/state.

## Web (Emscripten)

Use the virtual filesystem and a documented mount for persistent data. Distinguish in-memory temporary files from persisted storage, and flush asynchronous persistence (for example IndexedDB-backed storage) according to the runtime's API. Do not promise persistence until the flush completes. Browser storage can be cleared by the user or evicted by the browser; provide export/import for savegames and other valuable state.

## Changing a location

Moving a category to a different directory is a user-data migration. Follow the migration rules in [`DESIGN.md`](DESIGN.md) section 4: read or migrate the old location deliberately, never delete it automatically, define a conflict policy when both contain data, and explain the change in release notes. Unreleased projects may skip migration, but should say so.
