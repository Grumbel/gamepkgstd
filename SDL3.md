# SDL3

What it takes to build a game with SDL3 on every target of this standard.
Found while porting sdl-jstest (2026-10-30), which builds the same tester
against SDL 1.2, SDL2 and SDL3 for Linux, R36S, Android, Windows and the
web. Reference: sdl-jstest `nix/r36s.nix`, `nix/android.nix`,
`nix/windows.nix`, `nix/wasm.nix`.

## Status per target

| Target | SDL3 | Tested |
|---|---|---|
| Linux | nixpkgs `sdl3` | yes |
| R36S / ArkOS | built for KMS/DRM and shipped in the port's `libs/` (ARK-8) | starts fullscreen on KMSDRM, GLES2 on the Mali-G31, the built-in pad with PortMaster's mapping; not tried yet: button presses on the device, audio (built without ALSA) |
| Android | built from SDL3's source tree with ndk-build (SDL3-3) | Fire 7 (Fire OS 5.1.1 = API 22, armv7): menu, Back, Quit, a gamepad |
| Windows | built from source with the mingw-w64 cross stdenv (SDL3-4) | under Wine (win32, win64) |
| Web | static library built with `emcmake` (SDL3-5) | Chromium, without a gamepad |

No target needed a change to SDL3's sources. SDL3 was less work than SDL2
on the R36S, because the game no longer depends on the SDL2 build the
firmware happens to ship (ARK-3, ARK-7).

## Rules

**SDL3-1.** SDL3 comes from nixpkgs' `sdl3` (its `src` for cross builds), the
same version on every target. A build from a source tarball has no git
tree: pass `-DSDL_REVISION=SDL-<version>`, otherwise SDL reports
`GIT-NOTFOUND` as its revision (LOGGING.md prints it).

**SDL3-2.** Dependencies are named after FLAKE-2: `sdl3-r36s`,
`sdl3-android`, `sdl3-win64`, `sdl3-wasm`.

**SDL3-3.** Android:

- The native library is ndk-built from SDL3's source tree (its own
  `Android.mk`), the Java side compiled from
  `android-project/app/src/main/java/org` with `javac --release 8` and `d8`.
  The Java side needs a newer SDK to compile against (API 35); the APK's
  `minSdkVersion` stays 21, and it runs on API 22.
- The program includes `<SDL3/SDL_main.h>` in the file with `main()`; that
  turns it into `SDL_main`, which `SDLActivity` calls. The Activity is
  `org.libsdl.app.SDLActivity`, as with SDL2.
- Patches written for SDL2 sources must not fail on SDL3's: NDK r26+
  rejects `ALooper_pollAll`, which SDL2's sensor code calls and SDL3's does
  not, so a `grep -l` that finds nothing must not abort a `pipefail`
  script (`$(grep -Rl … || true)`).
- `main()` can run again in the same process: reset globals at its start.
- `SDL_HINT_ANDROID_TRAP_BACK_BUTTON` = `1` delivers Back as `SDLK_AC_BACK`
  instead of closing the Activity.

**SDL3-4.** Windows: SDL3 (like SDL2) is built from source with
`pkgsCross.mingwW64` / `mingw32` and CMake; nixpkgs' mingw SDL packages pull
in X11, which does not cross-build. Ship `SDL3.dll` next to the `.exe`.

**SDL3-5.** Web: SDL3 is built as a static library with `emcmake cmake`
(`-DSDL_SHARED=OFF -DSDL_STATIC=ON -DSDL_TESTS=OFF -DSDL_TEST_LIBRARY=OFF
-DSDL_EXAMPLES=OFF`, plus SDL3-1), offline instead of Emscripten's port
(WEB-4). The game finds it through `-DSDL3_DIR=<prefix>/lib/cmake/SDL3`;
SDL3's CMake config provides `SDL3::SDL3` for the static library too. The
frame loop rules (WEB-1/2) are the same as with SDL2. Size: sdl-jstest's
`.wasm` is 1.0 MB with SDL3 against 0.7 MB with SDL2.

**SDL3-6.** Call `SDL_SetAppMetadata(<display name>, <version>, <game>)`
before `SDL_Init`. The identifier becomes the X11 `WM_CLASS` and Wayland
`app_id` (INT-3) without a hint.

## Notes for converting a game

- Gamepads: PortMaster's `SDL_GAMECONTROLLERCONFIG` still applies (same
  variable name), and `gamecontrollerdb.txt` loads through
  `SDL_AddGamepadMappingsFromIO` (the file only adds lines for the running
  platform; the browser gets none, it has the Gamepad API's standard
  mapping).
- The Android accelerometer is no joystick in SDL3 (SDL2 lists it as one,
  `SDL_HINT_ACCELEROMETER_AS_JOYSTICK`, which SDL3 removed); read it
  through the sensor API.
- The renderer has no "accelerated" flag any more; the startup report
  (LOGGING.md) names the render driver (`opengles2`, `software`) instead.
- Joystick and gamepad functions take an `SDL_JoystickID`, not a device
  index; `SDL_GetJoysticks` / `SDL_GetGamepads` return the IDs.
- Booleans: most functions return `bool` (true = success) where SDL2
  returned 0 / negative.
