# Development with `nix develop`

Typical session:

```sh
nix develop
<game>-run --windowed       # configure if needed, build, run
<game>-run-gdb              # same, under gdb
nix develop -c <game>-run   # without entering the shell
```

**DEV-1.** `devShells.default` provides the build inputs of the native package
(`inputsFrom`), plus `cmake`, `ninja`, `gdb`, `ccache`, and these commands:

| Command | Action |
|---------|--------|
| `<game>-configure` | `cmake -S $<GAME>_SOURCE -B $<GAME>_BUILD_DIR -G Ninja` |
| `<game>-build` | Configure if needed, then `cmake --build` |
| `<game>-run [ARGS]` | Build, then run the game from the build tree |
| `<game>-run-gdb [ARGS]` | Build, then run under gdb |

**DEV-2.** The commands are real executables (`pkgs.writeShellScriptBin`), not
shell functions in `shellHook`, so `nix develop -c <game>-run` works.

**DEV-3.** `shellHook` exports `<GAME>_SOURCE="$PWD"` and
`<GAME>_BUILD_DIR="${<GAME>_BUILD_DIR:-/tmp/<game>-build}"`. Every command
starts with `set -euo pipefail` and exits with an error if `<GAME>_SOURCE` is
unset ("enter the shell with: nix develop").

**DEV-4.** The build type is `Debug` (override with `CMAKE_BUILD_TYPE`). The
package build stays `RelWithDebInfo` (FLAKE-8).

**DEV-5.** `<game>-build` reconfigures when `CMAKE_HOME_DIRECTORY` in
`CMakeCache.txt` differs from `<GAME>_SOURCE` or no longer exists, so moving
the checkout never builds a stale tree.

**DEV-6.** `<game>-run` passes `--data-dir "$<GAME>_SOURCE/data"` (if the game
has a data directory), so it runs from the source tree without installing.
Anything the package sets through `makeWrapper` (e.g. `LD_LIBRARY_PATH` for
dlopen'd libraries) is set the same way in the dev shell.

**DEV-7.** `<game>-run-gdb` starts the game immediately, quits gdb when the
game exits with status 0, and stays at the gdb prompt on a crash or non-zero
exit:

```sh
gdb -q \
  -ex "set pagination off" \
  -ex "set confirm off" \
  -ex "set debuginfod enabled off" \
  -ex run \
  -ex 'python
try:
  if int(gdb.parse_and_eval("$_exitcode")) == 0:
    gdb.execute("quit")
except Exception:
  pass
' \
  --args "$<GAME>_BUILD_DIR/<game>" "$@"
```

**DEV-8.** `<game>-configure` links `$<GAME>_SOURCE/build` to the build
directory so clangd finds `compile_commands.json`
(`CMAKE_EXPORT_COMPILE_COMMANDS=ON`).

Reference implementation: the `devShells.default` in
[Kurvenrausch's flake.nix](https://github.com/Grumbel/kurvenrausch).
