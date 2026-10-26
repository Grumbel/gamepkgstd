# Version Number Handling

## Rules

* The version number is stored in a file `VERSION` in the top-level directory. That file is the **only source of truth**. Do not hardcode the version elsewhere; derive it from this file.
* Do **not** use `.gitattributes` `export-subst` (or similar) for `VERSION`. Keep it a plain text file (e.g. `0.2.0-dev`).
* **In git (development):** `VERSION` always carries a `-dev` suffix, e.g. `0.2.0-dev`.
* **Development builds** (when `VERSION` contains `-dev`) append the git revision count and short hash:

  * `0.2.0-dev.1615+gf1fb306`
  * `0.2.0-dev.1615+gf1fb306-dirty` (working tree dirty)

* **Release builds** (when `VERSION` has no `-dev`): use the file contents as-is, e.g. `0.2.0`. Do **not** append revision count or hash.
* **Revision count** (`revCount`): number of commits reachable from the current revision. It orders development builds. In Nix flakes this is `self.revCount` when available (see caveats below).
* **Short hash:** Nix `self.shortRev` or, if dirty, `self.dirtyShortRev` (already includes a `-dirty` suffix when applicable). Fallback: `"dirty"`.

### Format (development)

```text
{base}-dev.{revCount}+g{shortRev}
```

Roughly semver-compatible: `0.2.0-dev.1615` is a pre-release; `+gf1fb306` is build metadata.

### Release process

1. Set `VERSION` to the release number **without** `-dev` (e.g. `0.2.0`).
2. Commit that change in the intended release revision.
3. Tag the clean release tree as `v` + that number (e.g. `v0.2.0`). The tag must match `VERSION` with a `v` prefix.
4. Build and publish from that identified clean revision. Reject dirty working trees for release artifacts; if a dirty build is intentionally made for debugging, label it as a development artifact and do not publish it as the release.
5. After the release, bump `VERSION` to the next `-dev` (e.g. `0.2.1-dev`) on the main branch.

---

## CMake

* Read `VERSION` into `PROJECT_VERSION_FULL` unless packaging overrides it (`-DPROJECT_VERSION_FULL=...`).
* Nix should pass the **full** version (with `.revCount+g…` when applicable) via that flag so the build matches the package.
* `project(... VERSION ...)` only accepts a numeric `major[.minor[.patch[.tweak]]]`. Strip any `-dev` / `+…` suffix for CMake’s `PROJECT_VERSION`.
* Use the **numeric** `PROJECT_VERSION` for `write_basic_package_version_file` and similar.
* Keep `PROJECT_VERSION_FULL` for display / compile definitions if needed.

### Applications

* Expose the full version via `--version` and About dialogs.
* Prefer defining e.g. `{PROJECT}_VERSION` from `PROJECT_VERSION_FULL` (e.g. `JSTEST_VERSION`).

### Libraries

* No `--version` / About UI required.
* Still drive packaging from `VERSION`:

  * Nix: `version = …` (full string for store uniqueness on dev builds)
  * CMake package config version file: numeric `PROJECT_VERSION`
  * pkg-config `Version:`: numeric is enough

* Do not hardcode `project(foo VERSION 0.1.0)`; always derive from `VERSION` / `PROJECT_VERSION_FULL`.

### CMakeLists.txt example

```cmake
cmake_minimum_required(VERSION 3.10...4.1)

# Source of truth: top-level VERSION (e.g. "0.2.0-dev"), or
# -DPROJECT_VERSION_FULL=... from packaging (Nix may append .<revCount>+g<rev>).
if(NOT DEFINED PROJECT_VERSION_FULL)
  file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/VERSION" PROJECT_VERSION_FULL LIMIT_COUNT 1)
endif()

# project() needs a numeric version; strip -dev / +git / similar suffixes.
string(REGEX MATCH "^[0-9]+(\\.[0-9]+)*" _project_version_cmake "${PROJECT_VERSION_FULL}")

project(myproject VERSION ${_project_version_cmake} LANGUAGES CXX)

# Optional: expose full string to the code (applications; libraries as needed)
# target_compile_definitions(myapp PRIVATE MYPROJECT_VERSION="${PROJECT_VERSION_FULL}")
```

---

## Nix flake

* Read the base from `VERSION`.
* If the base contains `-dev`, append `.{revCount}+g{shortRev}`.
* **`self.revCount` is not always present** (shallow clones, some path/dirty evaluations). Always use a fallback:

  ```nix
  self.revCount or 0
  ```

* Coerce with `toString` when interpolating.
* Pass the result into the package and into CMake as `-DPROJECT_VERSION_FULL=${version}`.

### flake.nix example

```nix
versionBase = nixpkgs.lib.strings.removeSuffix "\n" (builtins.readFile ./VERSION);
# Alternative: versionBase = nixpkgs.lib.fileContents ./VERSION;

gitRev = "${self.shortRev or self.dirtyShortRev or "dirty"}";
isDev = nixpkgs.lib.strings.hasInfix "-dev" versionBase;

version =
  if isDev then
    "${versionBase}.${toString (self.revCount or 0)}+g${gitRev}"
  else
    versionBase;
```

Use `version` as the derivation `version` and pass it through to CMake.

**Do not** always append (as in older sketches like `"${versionBase}.${self.revCount}+g${gitRev}"`): that would tag releases with git metadata and can fail when `revCount` is missing.

---

## Caveats and pitfalls

| Topic | Detail |
|--------|--------|
| **Missing `revCount`** | Use `self.revCount or 0`. Otherwise evaluation can fail with `attribute 'revCount' missing`. |
| **Shallow clones** | `revCount` may be missing or not match a full-history count; uniqueness still comes mostly from the hash. |
| **Dirty trees** | Prefer `dirtyShortRev` when set; it typically already ends in `-dirty`. |
| **CMake `project(VERSION)`** | Never pass the full `0.2.0-dev.N+g…` string; only the leading numeric part. |
| **Package version files** | Prefer numeric `PROJECT_VERSION` so `find_package` version checks stay sane. |
| **Duplication** | Avoid a second hardcoded version in `project()`, the flake, or headers; generate from `VERSION`. |
| **Release vs tag** | Tag `v0.2.0` must match committed `VERSION` `0.2.0` (plus the `v` prefix only on the tag). |

---

## Quick checklist

**Library**

- [ ] Plain top-level `VERSION` (`x.y.z-dev` on the main branch)
- [ ] CMake reads `VERSION` → `PROJECT_VERSION_FULL`, numeric for `project()`
- [ ] Flake builds `version` with conditional `.revCount+g…` and `revCount or 0`
- [ ] `-DPROJECT_VERSION_FULL=${version}` from Nix
- [ ] No requirement for `--version` / About

**Application** (same as library, plus)

- [ ] `--version` prints `PROJECT_VERSION_FULL` (or equivalent)
- [ ] GUI About uses the same string if applicable
