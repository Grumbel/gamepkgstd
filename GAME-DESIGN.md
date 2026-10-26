# Game-Specific UI, Input, and Runtime Guidelines

This document contains recommendations that are specific to games. The generic packaging, path, CLI, configuration, extension, cross-compilation, and Nix-first CI contract lives in [`DESIGN.md`](DESIGN.md). These recommendations are optional defaults: preserve established controls where changing them would surprise players, and document deliberate deviations.

The source projects that informed these examples include [Pingus](https://github.com/Pingus/pingus), [Kurvenrausch](https://github.com/Grumbel/kurvenrausch), [dualthrust](https://github.com/Grumbel/dualthrust), [KugelMatch](https://github.com/Grumbel/kugelmatch), and [SuperTux](https://github.com/SuperTux/supertux).


## Keyboard shortcuts

| Key | Action | Notes |
|-----|--------|-------|
| **F11** | Toggle fullscreen | Common default across the reference projects |
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


## Gamepad and joystick handling

- Use the SDL2 GameController API by default when PortMaster/R36S is in the support matrix; prefer the **GameController** abstraction over raw Joystick for consistent button names (A/B/X/Y, triggers, sticks). Use SDL3 only when each required target runtime is explicitly supported and tested. See `DESIGN.md` for legacy SDL1.2 and OLPC XO-1/X11 guidance.
- Provide a **rebind UI** that stores bindings in the config file (kurvenrausch / dualthrust pattern: `key_<action> …` / `pad_<action> …` lines).
- Default mappings should work out-of-the-box for Xbox-style and PlayStation-style pads; document R36S / PortMaster button layout in the port README.
- Support simultaneous keyboard + gamepad; last input wins for focus, both can generate events.
- Optional: rumble / vibration when the hardware reports it (kurvenrausch stores a `rumble` preference).
- Hot-plug: open newly connected controllers; ignore disconnects gracefully.

For multi-player (SuperTux local multiplayer, KugelMatch 2P) keep per-player binding sets.

---


## Menu structure and options

A practical menu skeleton to adapt to the game’s actual modes and features:

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


## Graphics and display settings

- `Auto` should be the default for renderer/backend selection unless the game has one supported backend. Detect capabilities at runtime; do not infer GPU features solely from the OS or device name.
- Unsupported settings should be hidden, disabled with an explanation, or rejected clearly. Never display an option as active when the requested mode was not actually created.
- If renderer initialization fails, report the failure and attempt a documented safe fallback where one exists. Preserve the requested setting only if the user can recover from it; otherwise revert to a known-good setting and explain what happened.
- A fullscreen/resolution change that could leave the display unusable should have a confirmation/revert flow when the platform allows it. Revert automatically after a short timeout unless confirmed. Where such a flow is not feasible, provide a documented reset path such as a safe-mode command-line flag or removal of the relevant config entry.
- Prefer borderless desktop fullscreen by default on desktop platforms unless exclusive fullscreen is needed. Respect the platform's available display modes and windowing model; web and handheld targets may expose fewer controls.
- Distinguish internal render resolution, window size, display resolution, pixel scaling, and UI scale. Do not label one as another. Preserve aspect ratio by default for pixel-art or fixed-aspect games; make stretching an explicit choice if offered.
- Make the desktop window resizable where supported and update rendering, layout, and pointer/controller coordinate mapping after every resize. Reflow menus and settings instead of clipping them. Handheld screens are fixed-size, so test at their actual native resolution; 640×480 is a useful ArkOS/R36S reference, not a universal assumption.
- VSync and frame-rate limits should have predictable interaction. Explain whether the cap applies to simulation, rendering, or both. Avoid presenting an FPS cap as a fix for simulation timing bugs.
- Graphics settings should be persisted only after successful application when possible. If a change fails, restore the previous working value in both runtime state and the UI.
- Include a way to reset graphics settings without requiring a successful graphical startup. A `--safe-mode` or `--reset-video-settings` option is useful when the normal menu cannot be reached.


## Input, focus, and menu behavior

- Every essential action must be reachable using the primary supported input method. Menus should support keyboard and controller navigation where those devices are supported; mouse-only UI is insufficient for a controller-first port.
- Show the currently focused item clearly. After a dialog closes, restore focus to the item that opened it where practical. Avoid relying on hover as the only indication of selection.
- Controller prompts should reflect the active controller mapping when practical. Do not assume all controllers label the same physical button as A/B or Cross/Circle; document whether prompts use SDL logical button names or physical positions.
- Handle controller add/remove events during play and menus. A disconnect must not crash the game or leave input permanently stuck. Provide a fallback input route where possible.
- Rebinding must detect conflicts and explain how they are resolved. Support canceling a rebind operation without losing the previous binding.
- Do not hard-code a shortcut that conflicts with text entry or common platform/window-manager behavior without a strong reason. List global shortcuts in the in-game help or controls screen.
- Escape/back should move one level outward or close the current dialog. In gameplay it may pause or open the pause menu; it should not immediately destroy progress. Make quit and abort actions distinct, and confirm destructive actions when there is meaningful unsaved progress.
- Pause menus should clearly distinguish Resume, Restart (if available), Abort/Return to title, and Quit. Do not show unavailable actions as if they were usable.
- Persist settings that players reasonably expect to survive restart. Clearly label temporary actions and per-session overrides. A reset-to-defaults action should state its scope and should not delete saves or high scores.
