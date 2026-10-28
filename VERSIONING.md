# Versioning

**VER-1.** The top-level file `VERSION` holds the version, e.g. `0.2.0-dev`.
It is the only place the version is written. No `export-subst`, no version in
`project()`, headers or `flake.nix`.

**VER-2.** On the main branch `VERSION` always ends in `-dev`.

**VER-3.** The full version string:

| `VERSION` | Full version |
|-----------|--------------|
| `0.2.0-dev` | `0.2.0-dev.<revCount>+g<shortRev>`, e.g. `0.2.0-dev.1615+gf1fb306` |
| `0.2.0-dev`, dirty tree | `0.2.0-dev.1615+gf1fb306-dirty` |
| `0.2.0` | `0.2.0` |

**VER-4.** Release:

1. Set `VERSION` to `0.2.0`, commit.
2. Tag that commit `v0.2.0`.
3. Build release artifacts from the clean tag only.
4. Set `VERSION` to `0.2.1-dev`, commit.

## Nix

**VER-5.**

```nix
versionBase = nixpkgs.lib.fileContents ./VERSION;
gitRev = self.shortRev or self.dirtyShortRev or "dirty";
version =
  if nixpkgs.lib.hasInfix "-dev" versionBase
  then "${versionBase}.${toString (self.revCount or 0)}+g${gitRev}"
  else versionBase;
```

`self.revCount` is missing in shallow clones; always use `or 0`. Pass
`version` as the derivation `version` and as `-DPROJECT_VERSION_FULL=${version}`.

## CMake

**VER-6.**

```cmake
if(NOT DEFINED PROJECT_VERSION_FULL)
  file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/VERSION" PROJECT_VERSION_FULL LIMIT_COUNT 1)
endif()
# project() only accepts numeric versions
string(REGEX MATCH "^[0-9]+(\\.[0-9]+)*" _version_numeric "${PROJECT_VERSION_FULL}")
project(<game> VERSION ${_version_numeric} LANGUAGES CXX)
target_compile_definitions(<game> PRIVATE <GAME>_VERSION="${PROJECT_VERSION_FULL}")
```

**VER-7.** `--version`, the title screen/about box, the Windows resource and
Android `versionName` all show `PROJECT_VERSION_FULL`. Android `versionCode`
is `self.revCount or 1`.
