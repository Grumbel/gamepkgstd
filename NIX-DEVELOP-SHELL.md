# Nix develop shell pattern (out-of-tree CMake projects)

Reusable pattern for day-to-day development with a Nix flake: **deps + env +
PATH scripts** in `nix develop`; packaged wrappers stay separate for
`nix build` / `nix run`.

Derived from [biltoo](https://github.com/Grumbel/biltoo) and applied to
[Pingus](https://github.com/Pingus/pingus).

---

## Goals

| Goal | Mechanism |
|------|-----------|
| One command for the common loop | `project-run` / `project-run-gdb` **build before run** |
| Fail clearly outside the shell | Preamble requires `PROJECT_SOURCE` (set only in `shellHook`) |
| No silent stale builds | Compare CMake cache vs current env; **reconfigure** on mismatch |
| Live deps while developing | Prefer in-tree / live checkouts for nested C++ libs; flake inputs for `nix build` |
| Unwrapped binary finds plugins | Mirror package-wrapper env (`LD_LIBRARY_PATH`, plugin paths, …) in the shell |
| Useful debug symbols | **Debug** in develop; **RelWithDebInfo** (or similar) in packages |
| Comfortable gdb | Auto-`run`; quit on clean exit; keep session on failure |

---

## Minimal flake shape

```nix
devShells.default = pkgs.mkShell.override { stdenv = pkgs.ccacheStdenv; } {
  inputsFrom = [ /* the main package derivation — pulls buildInputs */ ];
  packages = [
    pkgs.cmake pkgs.ninja pkgs.gdb pkgs.ccache
    (pkgs.writeShellScriptBin "project-configure" ''…'')
    (pkgs.writeShellScriptBin "project-build" ''…'')
    (pkgs.writeShellScriptBin "project-run" ''…'')
    (pkgs.writeShellScriptBin "project-run-gdb" ''…'')
  ];
  CMAKE_BUILD_TYPE = "Debug";
  shellHook = ''
    export PROJECT_SOURCE="$PWD"   # or the flake root
    export PROJECT_BUILD_DIR="''${PROJECT_BUILD_DIR:-/tmp/project-build}"
    # Optional: mirror anything the package wrapper sets for dlopen
    # export QT_PLUGIN_PATH=…
    # export LD_LIBRARY_PATH=…
    echo "project-configure | project-build | project-run | project-run-gdb"
  '';
};
```

**Helpers must be real PATH scripts** (`writeShellScriptBin`), not `shellHook`
functions. Only binaries work with `nix develop -c project-run`.

---

## Checklist (what each piece does)

### 1. Preamble requires `PROJECT_SOURCE`

Every helper starts with a shared preamble:

```bash
set -euo pipefail
if [ -z "${PROJECT_SOURCE:-}" ]; then
  echo "$0: PROJECT_SOURCE is not set (enter the shell with: nix develop)" >&2
  exit 1
fi
PROJECT_BUILD_DIR="${PROJECT_BUILD_DIR:-/tmp/project-build}"
```

Running a script outside the shell fails immediately instead of writing into a
random cwd or a stale absolute path.

### 2. Build before run

`project-run` and `project-run-gdb` call `project-build` first. One command for
the edit → compile → try loop; no forgotten rebuilds.

### 3. Reconfigure when paths change

CMake records `CMAKE_HOME_DIRECTORY` (and any nested `FOO_SOURCE_DIR`) in the
cache. `project-build` compares the cached path to the current env and
re-runs `project-configure` when they differ or the cached tree is gone.

Without this, moving a checkout or switching a nested flake input to a new
`/nix/store/…` hash leaves you compiling a ghost tree until someone notices.

### 4. Prefer live checkouts for nested C++ deps

When the project `add_subdirectory`s a sibling library (biltoo → thumtoo):

- **Develop:** resolve a live path (`../thumtoo`, `THUMTOO_SOURCE_DIR=…`) so
  edits rebuild incrementally.
- **Package:** flake inputs / fixed store paths for reproducibility.

Warn if the resolved path is under `/nix/store` (frozen snapshot: edits
invisible until reconfigure, and each new store hash forces a full rebuild).

Pingus vendors helpers under `external/` in the same tree, so CMake’s
`add_subdirectory(external/…)` already is the live path — no extra resolver.

### 5. Mirror wrapper env in the shell

Anything the **packaged** binary gets via `makeWrapper` / `qtWrapperArgs` /
`addDriverRunpath` must also be set for the **unwrapped** out-of-tree binary:

| Concern | Example |
|---------|---------|
| Qt plugins | `QT_PLUGIN_PATH` |
| dlopen libs | `LD_LIBRARY_PATH` (PipeWire, etc.) |
| Data / assets | `--data-dir`, `XDG_DATA_DIRS`, baked `DATA_PREFIX` |
| Models / side tools | `PATH`, project-specific `*_MODEL` vars |

Do **not** copy Qt-only variables into a non-Qt project. Pingus is SDL2/OpenAL;
the develop shell mainly clears a hostile system `LD_LIBRARY_PATH` and passes
`--data-dir "$PINGUS_SOURCE/data"`.

### 6. Build type split

| Context | Type |
|---------|------|
| `nix develop` / out-of-tree | `Debug` (override with `CMAKE_BUILD_TYPE=…`) |
| `nix build` package | `RelWithDebInfo` + `separateDebugInfo` where useful |

### 7. gdb helper

```bash
gdb -q \
  -ex "set pagination off" \
  -ex "set confirm off" \
  -ex "set debuginfod enabled off" \
  -ex run \
  -ex 'python
try:
  ec = gdb.parse_and_eval("$_exitcode")
  if int(ec) == 0:
    gdb.execute("quit")
except Exception:
  pass
' \
  --args "$PROJECT_BUILD_DIR/project" "$@"
```

- Starts the inferior immediately (no manual `run`).
- Quiet until stop.
- **Exit 0** → gdb quits (no dangling prompt after a successful run).
- **Crash / signal / non-zero** → interactive prompt for `bt`, etc.

---

## What *not* to copy blindly

- **Qt-only** env (`QT_PLUGIN_PATH`, offscreen platform) into non-Qt games.
- **Default build dir** location — any writable sticky path is fine
  (`/tmp/project-build`, `$HOME/.cache/…`).
- **Nested-library resolvers** unless the flake has the same dual
  live-tree / store-input setup as biltoo’s thumtoo.
- **`inputsFrom = [ package ]`** content — always verify the package’s
  `buildInputs` match what local cmake needs (and that Windows-only deps
  stay out of the Linux shell).

---

## Pingus mapping

| biltoo | Pingus |
|--------|--------|
| `BILTOO_SOURCE` | `PINGUS_SOURCE` |
| `BILTOO_BUILD_DIR` | `PINGUS_BUILD_DIR` (default `/tmp/pingus-build`) |
| `biltoo-configure` | `pingus-configure` |
| `biltoo-build` | `pingus-build` |
| `biltoo-run` | `pingus-run` (`--data-dir "$PINGUS_SOURCE/data"`) |
| `biltoo-run-gdb` | `pingus-run-gdb` |
| `THUMTOO_SOURCE_DIR` | *(n/a — `external/` is in-tree)* |
| `QT_PLUGIN_PATH` / PipeWire | clear system `LD_LIBRARY_PATH`; SDL/OpenAL via `inputsFrom` |

```sh
nix develop
pingus-configure && pingus-build
pingus-run
# or: nix develop -c pingus-run-gdb
```

Package builds stay `nix build .#pingus` (RelWithDebInfo, no ccache).

---

## Reference

- biltoo flake develop shell: https://github.com/Grumbel/biltoo
- biltoo `docs/ENVIRONMENT.md` (dev-shell env table + gdb notes)
- This pattern’s reusable idea: **develop shell = deps + env + PATH scripts;
  scripts drive out-of-tree configure/build/run/gdb; packaged wrappers stay
  separate for `nix build` / `nix run`.**
