# Settings and command line options

## Command line

**OPT-1.** GNU-style options: `-x` short, `--long-name` long, values as
`--opt VALUE` or `--opt=VALUE`. `--` ends option parsing. An unknown option or
invalid value prints an error to stderr and exits non-zero.

**OPT-2.** `--help` and `--version` print to stdout and exit 0 without
initializing SDL, audio or loading data. `--version` prints `<game> <version>`
with the full version string (VER-3).

**OPT-3.** Options every game has (omit only what does not exist in the game,
e.g. audio options for a silent game):

| Option | Meaning |
|--------|---------|
| `-h`, `--help` | Usage |
| `-v`, `--version` | Version |
| `-f`, `--fullscreen` | Start fullscreen |
| `-w`, `--windowed` | Start windowed |
| `-g`, `--geometry WxH` | Window size |
| `--mute` | Start with audio muted |
| `--data-dir DIR` | Game data (DIR-3) |
| `--config-dir DIR` | Config directory |
| `--state-dir DIR` | Savegame/state directory |
| `--user-data-dir DIR` | User levels, mods |
| `--cache-dir DIR` | Cache directory |
| `--portable` | Portable mode (DIR-11) |
| `--no-save` | Write no state and no config this run |
| `--show-paths` | Print every resolved directory and where it came from (default, XDG variable, option, portable), then exit |

**OPT-4.** Optional, but if present use these names:

| Option | Meaning |
|--------|---------|
| `--vsync on\|off` | VSync |
| `--renderer NAME` | Renderer/backend; `auto` is the default |
| `--show-fps` | FPS counter |
| `-d`, `--debug` | Debug output and keys |

## Settings

**OPT-5.** Settings every game has in its options menu and config file, using
these key names:

| Key | Type |
|-----|------|
| `fullscreen` | bool |
| `window-width`, `window-height` | int |
| `vsync` | bool |
| `master-volume`, `music-volume`, `sound-volume` | 0–100 |
| `mute` | bool |
| key and button bindings | KEY-10 |

**OPT-6.** Precedence: built-in default < config file < command line. Command
line values apply to this run only and are never written to the config file.
A value changed in the menu is written when the player leaves the menu.

**OPT-7.** The config file is human-readable text in the config directory.
Unknown keys are ignored, not an error. An unreadable file is renamed to
`<file>.bad`, a warning is printed, and defaults are used.

**OPT-8.** "Reset to defaults" in the menu resets settings only, never
savegames or high scores.

**OPT-9.** All options are documented in `--help` and the man page (DIR-1).
