# Startup log

When a game misbehaves on one device, its log is often all there is: the
PortMaster `log.txt`, `adb logcat`, a terminal paste in a bug report. The
start of that log must say what the game actually runs on, not only what it
was built against. Example: xanatoo's R36S log line
`imgui: SDL 2.0.10 at runtime, per-triangle fallback renderer` showed that
ArkOS's default SDL2 is 2.0.10, although it also installs 2.30.10, and
explained why the UI was slow.

**LOG-1.** Every start prints a startup report to stderr, without needing a
flag (on a handheld the user cannot easily start the game again with other
options). PortMaster launchers keep it in `<game>/log.txt`; on Android it
goes to logcat; on Windows to the console attached per INT-12; on the web to
the browser console. `--help`, `--version` and `--show-paths` print no
report (OPT-2). `--debug` (OPT-4) may add more.

**LOG-2.** One line per item, `<topic>: <details>`, plain text without
colour codes. The report fits about one screen (≈ 40 lines) and keeps the
same wording from run to run, so two logs can be diffed and grepped.

**LOG-3.** The report covers, as far as the game uses them:

| Topic | Contents |
|-------|----------|
| Game | Name, full version (VER-3), build type, target (`x86_64-linux`, `aarch64` R36S, `wasm32`, …), compiler and its version |
| System | `SDL_GetPlatform()`, OS and kernel version (Android: API level, device model), CPU count, RAM |
| Libraries | Each library that can report its version. Where the runtime version can differ from the headers (shared libraries: SDL2, FreeType, HarfBuzz, OpenSSL, SQLite, …), both: `SDL 2.0.10 (built against 2.0.10)` |
| Paths | Data directory and the config / state / cache directories as resolved, with where each came from (as `--show-paths`) |
| Video | SDL video driver, display mode (size, refresh rate), window size and drawable size, fullscreen mode, scale / DPI |
| Renderer | SDL_Renderer: name, flags (accelerated, vsync, target textures), max texture size. OpenGL: `GL_VENDOR`, `GL_RENDERER`, `GL_VERSION`, `GL_SHADING_LANGUAGE_VERSION`, context profile, the extensions the game depends on. Vulkan or other APIs: the equivalent device and driver strings |
| Audio | Audio driver, device name, frequency, format, channels, buffer size |
| Input | Each controller found: name, GUID, where its mapping came from (SDL's database, `SDL_GAMECONTROLLERCONFIG`, the game) |
| Assets | What was picked at runtime with fallbacks: font files, theme, language |

**LOG-4.** Values are what was obtained, not what was requested: the
context version and vsync state the driver actually gave, the audio spec
SDL returned, the font file Fontconfig matched.

**LOG-5.** Each fallback or degraded path is logged once, when it is chosen,
with the reason: missing function or extension, render targets unavailable,
software renderer, optional library missing, feature disabled.

**LOG-6.** An optional subsystem that fails to start (audio, a controller, a
network client) prints a warning with `SDL_GetError()` or the equivalent and
what the game does instead, and the game keeps running.

**LOG-7.** The report contains no secrets (tokens, passwords, keys).
