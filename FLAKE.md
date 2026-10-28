# Flake outputs

`<game>` is the game's executable and package name, lowercase, e.g. `pingus`.

## Packages

**FLAKE-1.** Package names follow `<game>[-<variant>]-<target>[-<format>]`:

| Output | Content |
|--------|---------|
| `default`, `<game>` | Native Linux build (same derivation) |
| `<game>-<variant>` | Alternative native build, e.g. `<game>-gles2` |
| `<game>-win32`, `<game>-win64` | Unpacked Windows directory: `.exe`, DLLs, data |
| `<game>-win32-zip`, `<game>-win64-zip` | The same, zipped |
| `<game>-android` | APK |
| `<game>-wasm` | Static web site directory (`index.html`, `.js`, `.wasm`, data) |
| `<game>-r36s` | R36S/ArkOS aarch64 build |
| `<game>-r36s-portmaster` | PortMaster port directory |
| `<game>-r36s-portmaster-zip` | The same, zipped |

**FLAKE-2.** Dependencies built for a target are named `<library>-<target>`:
`sdl2-wasm`, `sdl2-android`. Sysroots are `<distro>-sysroot`:
`arkos-sysroot`.

**FLAKE-3.** A zip file is named `<game>-<version>-<target>.zip` and contains a
single top-level directory `<game>-<version>-<target>/`. Exception: a PortMaster
zip uses the layout PortMaster requires.

**FLAKE-4.** Cross-target outputs are only defined for build systems that can
build them (normally `x86_64-linux`). Never define a target-named output that
silently builds for the host.

## Apps

**FLAKE-5.**

| Output | Action |
|--------|--------|
| `default`, `<game>` | Run the native build |
| `<game>-wasm` | Serve `<game>-wasm` on localhost and open the browser |
| `install-android-<game>` | `adb install` the APK |

## Dev shell and checks

**FLAKE-6.** `devShells.default` is the development shell described in
[DEVELOPMENT.md](DEVELOPMENT.md).

**FLAKE-7.** Tests go in `checks.<system>.<name>`. `nix flake check` must pass
on `x86_64-linux`.

## Packaging

**FLAKE-8.** The native package is built with `CMAKE_BUILD_TYPE=RelWithDebInfo`
and passes `-DPROJECT_VERSION_FULL=${version}` ([VERSIONING.md](VERSIONING.md)).

**FLAKE-9.** `meta.mainProgram = "<game>"` and `meta.license` are set.

**FLAKE-10.** CI (GitHub Actions) only runs `nix build`/`nix flake check` on
named outputs. No build logic in workflow YAML or separate shell scripts.
