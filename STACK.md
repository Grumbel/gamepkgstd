# Language, libraries and APIs

**STACK-1.** C++17 or newer, set with `CMAKE_CXX_STANDARD`. Build with CMake
and Ninja.

**STACK-2.** SDL2 for window, input, audio and timing. Not SDL3: the
R36S/PortMaster runtimes ship SDL2. SDL 1.2 only for legacy targets
(GP2X/Wiz, OLPC XO-1); on the XO-1, plain X11 may be faster, so benchmark.

**STACK-3.** Rendering uses SDL_Renderer or OpenGL ES 2.0 / GLSL ES 1.00.
That is what R36S, Android and WebGL 1 provide. Desktop OpenGL is only an
additional backend (`<game>-<variant>` in FLAKE-1), never the only one.

**STACK-4.** Input goes through SDL's GameController API (KEY-8). Device events
are mapped to game actions in one place; gameplay code sees actions, not keys.

**STACK-5.** The game is fully playable at 640×480 (R36S). Larger windows scale
up with aspect ratio preserved; the window is resizable on desktop.

**STACK-6.** Tools and editors with a desktop GUI use Qt, not GTK.

**STACK-7.** Platform-specific code (`__EMSCRIPTEN__`, `__ANDROID__`, `_WIN32`)
lives in a few small files behind a common interface, not spread through the
game.

**STACK-8.** Dependencies come from nixpkgs or pinned flake inputs. No
`FetchContent`, `ExternalProject` downloads or other network access during the
build.

**STACK-9.** The license is the project's choice. Every file carries SPDX
headers (or a `REUSE.toml` entry) and `reuse lint` passes (FLAKE-11).
