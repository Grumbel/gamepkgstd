# Shortcuts

Shortcuts work in every screen (menus and gameplay) unless noted. Gameplay
bindings must not use these keys.

## Keyboard

**KEY-1.** Global keys:

| Key | Action |
|-----|--------|
| F11 | Toggle fullscreen |
| Alt+Enter | Toggle fullscreen |
| F12 | Screenshot to the screenshot directory (DIR-5), file `<game>-YYYYMMDD-HHMMSS.png` |
| Esc | Back one level; in gameplay open the pause menu |
| Pause | Pause / resume |

**KEY-2.** Esc never quits or aborts a running game directly. Quitting from
the pause or main menu asks for confirmation only if progress would be lost.

**KEY-3.** Fullscreen is borderless desktop fullscreen. Leaving it restores the
previous window size and position. The state is saved as the `fullscreen`
setting (OPT-5).

**KEY-4.** Menus: arrow keys move, Enter and Space confirm, Esc goes back.

**KEY-5.** Should, unless the game needs the key for gameplay:

| Key | Action |
|-----|--------|
| F1 | Show controls |
| P | Pause / resume |
| M | Mute / unmute |

**KEY-6.** Closing the window (window button, Alt+F4, `SDL_QUIT`) quits
without confirmation after writing autosave/config.

**KEY-7.** Losing window focus pauses gameplay.

## Gamepad

**KEY-8.** Use SDL's GameController API (logical button names), not raw
joystick indices.

| Button | Action |
|--------|--------|
| Start | Pause menu / resume |
| A | Confirm |
| B | Back |
| D-pad, left stick | Menu navigation |

**KEY-9.** Every menu, including settings and quit, is usable with the gamepad
alone. Controllers can be plugged and unplugged at any time; unplugging during
gameplay pauses.

## Rebinding

**KEY-10.** Gameplay keys and buttons are rebindable and stored in the config
file. The global keys of KEY-1 are not rebindable.
