# Launchers, icons and app identity

Every game ships a launcher entry and an icon on every platform. These are easy
to forget and only show up once installed, so they are checked by
`nix flake check` (FLAKE-11).

## Identity

**INT-1.** The app ID is the executable name, `<game>`. No reverse-DNS IDs on
Linux. It is used unchanged as:

| Where | Value |
|-------|-------|
| `.desktop` file name | `<game>.desktop` |
| `Icon=` and icon file names | `<game>` |
| AppStream `<id>`, metainfo file name | `<game>`, `<game>.metainfo.xml` |
| X11 `WM_CLASS`, Wayland `app_id`, `StartupWMClass=` | `<game>` |

`appstreamcli validate` may report `cid-desktopapp-is-not-rdns`. This is
expected and is not a reason to change the ID.

Where a dotted ID is required (Android `applicationId`), use
`org.grumbel.<game>`, all lowercase, with `-` replaced by `_`. Not
`io.github.…` and not a per-game domain: no domain is owned either way, and
one prefix is at least consistent and unique enough. IDs already published
(Pingus: `org.pingus.game`, Flathub `org.seul.pingus`) stay unchanged;
changing an ID makes it a different app and existing installs lose updates
and their saves.

**INT-2.** The display name (`Name=`, window title, Android label, PortMaster
title, `<title>` of the web page) is the same everywhere, e.g. `Kurvenrausch`.

**INT-3.** Before creating the window, set the IDs SDL2 uses for window
matching. Do not rely on SDL's default (the executable name): Nix's
`makeWrapper` renames the real binary to `.<game>-wrapped`, and then
Wayland and X11 show a generic icon and the dock does not group the window
with its launcher:

```cpp
SDL_SetHint(SDL_HINT_VIDEO_X11_WMCLASS, "<game>");
SDL_SetHint(SDL_HINT_VIDEO_WAYLAND_WMCLASS, "<game>");
SDL_SetHint(SDL_HINT_APP_NAME, "<Name>");
```

## Icon source

**INT-4.** One source image (`data/icons/<game>.png` or `.svg`). All other
icons are generated from it by a script in the repository
(`tools/make_icons.py`); the generated files are committed. Pixel art is
scaled up by whole multiples with nearest neighbour.

**INT-5.** The icon must be readable at 16×16 and must not contain text.

## Linux

**INT-6.** Installed files (DIR-1):

```
share/applications/<game>.desktop
share/metainfo/<game>.metainfo.xml
share/icons/hicolor/{16x16,32x32,48x48,64x64,128x128,256x256}/apps/<game>.png
share/icons/hicolor/scalable/apps/<game>.svg      (if an SVG exists)
```

48×48 is required: it is the hicolor fallback size.

**INT-7.** `.desktop` file:

```ini
[Desktop Entry]
Type=Application
Name=Kurvenrausch
GenericName=Racing Game
Comment=One line describing the game
Exec=<game>
Icon=<game>
Terminal=false
Categories=Game;ArcadeGame;
Keywords=racing;arcade;
StartupWMClass=<game>
SingleMainWindow=true
```

- `Exec=` is the bare command name, no path, no `%f`/`%u` unless the game
  opens files given on the command line.
- `Icon=` is the bare `<game>`, no path, no extension.
- `Categories=` contains `Game` plus one or more of the freedesktop game
  subcategories (`ActionGame`, `ArcadeGame`, `PuzzleGame`, `StrategyGame`, …).

**INT-8.** AppStream metainfo contains at least: `<id>`, `metadata_license`
(`CC0-1.0`), `project_license`, `name`, `summary`, `description`,
`<launchable type="desktop-id">`, `<url type="homepage">`, one screenshot,
`<content_rating type="oars-1.1"/>`, `<supports>`/`<recommends>` with
`<control>` entries for keyboard/gamepad, and `<releases>` with an entry for
each released `VERSION`.

**INT-9.** The game also calls `SDL_SetWindowIcon()` with the 32×32 or 64×64
icon. X11 uses it; Wayland ignores it and takes the icon only from the
`.desktop` file matched by `app_id` (INT-3), so a game run from the build tree
shows a generic icon on Wayland. That is expected.

## Windows

**INT-10.** The `.exe` embeds a multi-size `.ico` (16, 24, 32, 48, 64, 256;
256 PNG-compressed) as its first icon resource, via a `.rc` file added to the
executable's sources. Explorer, the taskbar and the window all use it.

```rc
IDI_ICON1 ICON "@ICO_PATH@"
```

**INT-11.** The `.rc` file also contains a `VERSIONINFO` resource:
`FileVersion`/`ProductVersion` = `PROJECT_VERSION_FULL`, `ProductName` =
display name, and `FileDescription` = display name, because Task Manager shows
`FileDescription` as the process name.

**INT-12.** The executable is a GUI program (`WIN32_EXECUTABLE TRUE`, i.e.
`-mwindows`), so no console window opens. To keep `--help` and `--version`
usable from a terminal, attach to the parent console at startup with
`AttachConsole(ATTACH_PARENT_PROCESS)` and reopen stdout/stderr.

## Android

**INT-13.** `AndroidManifest.xml`: `android:label` = display name,
`android:icon="@mipmap/ic_launcher"`.

**INT-14.** Legacy launcher PNGs in `res/mipmap-{mdpi,hdpi,xhdpi,xxhdpi,xxxhdpi}/ic_launcher.png`
(48, 72, 96, 144, 192 px) for API 21–25. Should: adaptive icon in
`res/mipmap-anydpi-v26/ic_launcher.xml` (108×108 dp foreground, artwork inside
the central 66 dp, plain background); without it Android 8+ shrinks the PNG
into a white shape.

## PortMaster

**INT-15.** The port contains `port.json` (`attr.title` = display name,
`attr.inst` names the full version, `arch`, `genres`, `porter`),
`screenshot.png`, and
`<game>/cover.png` (the 256×256 icon). The launcher script is
`<Name>.sh`, e.g. `Kurvenrausch.sh`.

## Web

**INT-16.** The shell page has `<title>` = display name and
`<link rel="icon" type="image/png" href="icon.png">` using the 64×64 icon,
which is copied into the `<game>-wasm` output.
