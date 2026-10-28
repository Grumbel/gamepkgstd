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

## Dev shell

**FLAKE-6.** `devShells.default` is the development shell described in
[DEVELOPMENT.md](DEVELOPMENT.md).

## Packaging

**FLAKE-7.** The native package is built with `CMAKE_BUILD_TYPE=RelWithDebInfo`,
passes `-DPROJECT_VERSION_FULL=${version}` ([VERSIONING.md](VERSIONING.md)) and
sets `doCheck = true`, so `nix build` runs the CTest unit tests. Nixpkgs skips
`checkPhase` automatically for cross builds.

**FLAKE-8.** `meta.mainProgram = "<game>"` and `meta.license` are set.

## Checks

`nix flake check` evaluates every output for every system, but only builds
`checks.<current system>`. Packages that are not in `checks` are not built.

**FLAKE-9.** `nix flake check` builds every package. `checks` is the package
set plus the extra checks below:

```nix
checks = packages // appChecks // {
  version = …;
  integration = …;
  …
};
```

Reason: a broken Windows, Android or web build otherwise goes unnoticed until
release. Nix caches each output, so a re-run only rebuilds what changed. For a
quick edit-compile-test loop use `<game>-build` and `ctest` in `nix develop`,
not `nix flake check`.

**FLAKE-10.** Every app has an `app-<name>` check that its `program` exists
(apps cannot be in `checks` directly because their names collide with
packages).

**FLAKE-11.** Extra checks, one small derivation each, so a failure names what
broke:

| Check | Verifies |
|-------|----------|
| `version` | `<game> --version` prints `<game> ${version}` (OPT-2, VER-3) |
| `help` | `<game> --help` exits 0 |
| `layout` | The installed native package has every file of DIR-1 and INT-6 |
| `integration` | `desktop-file-validate` on the `.desktop` file, `appstreamcli validate --no-net` on the metainfo |
| `man` | `mandoc -T lint` on the man page |
| `reuse` | `reuse lint` |
| `wasm-layout` | `<game>-wasm` has `index.html`, `<game>.js`, `<game>.wasm` and its icon (WEB-5, INT-16) |
| `windows-icon` | The `.exe` contains an icon and `VERSIONINFO` resource (INT-10, INT-11), e.g. with `wrestool` |

**FLAKE-12.** Checks need no network, display, GPU, audio device or game
controller. A check that has to start the game sets
`SDL_VIDEODRIVER=dummy SDL_AUDIODRIVER=dummy`. Tests that need real hardware
are not checks; they are listed in the README.

**FLAKE-13.** An output that needs input files that cannot be in the store
without user action (e.g. `requireFile` for original game data) is left out of
`checks`; the README says how to build it.

## CI

**FLAKE-14.** CI (GitHub Actions) runs `nix flake check` on `x86_64-linux` and
nothing else. No build logic in workflow YAML or separate shell scripts.
