# Game Packaging and Release Checklist

This is a release gate for games that follow `DESIGN.md`. Mark unsupported targets as **N/A with a reason**; do not treat every listed format as mandatory.

## 1. Project contract

- [ ] Supported platforms, architectures, input methods, and graphics backends are documented.
- [ ] Build, test, install, run, and package commands are documented and work from a clean checkout.
- [ ] Top-level `VERSION` is the only manually maintained version source (`VERSION-HANDLING.md`).
- [ ] Required runtime assets and dynamic libraries are documented.
- [ ] License, third-party notices, and asset licensing are included in the release process.

## 2. Build and cross-build

- [ ] Native clean build succeeds.
- [ ] Cross-build uses an explicit target toolchain/sysroot and target dependencies.
- [ ] Host tools and target binaries are kept distinct; build-time generators run on the build host.
- [ ] Build directories are separate for each compiler, target, and materially different configuration.
- [ ] Target-only executables are not accidentally run during cross-configuration.
- [ ] Optional dependencies/features are deliberate, not selected by accidental host discovery.
- [ ] Tests run where supported; unavailable target tests are listed as manual checks, not marked passing.
- [ ] Release metadata records source revision, version, target, options, and output artifacts.

## 3. Install-tree/runtime smoke test

Run the installed artifact from a working directory outside the source and build trees.

- [ ] The game finds assets without access to the source tree.
- [ ] Runtime libraries and plugins resolve on a clean target environment.
- [ ] `--help` and `--version` work without starting graphics/audio.
- [ ] Invalid CLI options fail with a useful message and non-zero status.
- [ ] Asset, user-data, config, state, cache, temp, and runtime paths resolve according to the target platform.
- [ ] Path overrides use consistent hyphenated names and affect only their own category.
- [ ] XDG base variables are validated as absolute paths; native Windows and portable/handheld layouts are tested separately.
- [ ] Portable mode has an explicit activation mechanism and handles read-only/full storage without silently switching profiles.
- [ ] Missing/unwritable user directories produce a useful diagnostic.
- [ ] No writes go into the install prefix or current working directory by default.
- [ ] `--no-save` (if supported) prevents all persistent gameplay writes.
- [ ] A missing or malformed config does not destroy saves or silently overwrite the original config.
- [ ] A clean install can start with no pre-existing user configuration.
- [ ] Temporary files are safely created and cleaned up; durable files are never stored only in temp directories.

## 4. Linux desktop package

- [ ] Executable and assets are installed to a documented prefix-relative layout.
- [ ] Desktop entry uses a valid executable name, icon reference, and launch arguments.
- [ ] Icon files are installed at the expected sizes/locations.
- [ ] AppStream metainfo is valid if supplied.
- [ ] Man page is included if the project maintains one.
- [ ] `nix build` and `nix run` work if Nix packaging is supported.
- [ ] Flatpak permissions and filesystem access are limited to what the game needs if Flatpak is supported.

## 5. Windows package

- [ ] Architecture and minimum supported Windows version are documented.
- [ ] Required runtime DLLs are bundled or their installation requirements are explicit.
- [ ] Asset paths work when installed under a directory containing spaces or non-ASCII characters.
- [ ] The package runs without the build environment or developer PATH.
- [ ] The archive/installer contains license and third-party notices.
- [ ] A clean Windows smoke test is performed for each supported architecture when possible.

## 6. Web / Android / handheld targets (when supported)

- [ ] Writable storage and asset access use the platform's actual storage model.
- [ ] Persistence flush/commit behavior is tested where storage is virtual or asynchronous.
- [ ] Input prompts, focus, pause/resume, and quit behavior fit the platform.
- [ ] No desktop-only assumptions are required for startup or configuration.
- [ ] Handheld launcher, environment variables, save location, and control mapping are documented.
- [ ] Bundled libraries do not override platform libraries accidentally; launcher environment is intentional.
- [ ] Artifact format, architecture, and runtime dependencies are checked even when the target cannot run in CI.
- [ ] PortMaster/handheld launcher path variables and the final-vs-base-directory convention are documented.

## 7. Nix-first CI

- [ ] The flake defines the authoritative build, test, lint, package, and cross-build logic.
- [ ] The documented CI flake output actually builds/runs all required checks and required package outputs; `nix flake check` alone does not automatically build every `packages` output.
- [ ] GitHub Actions only provisions Nix, selects the documented flake output, configures optional caching, and handles artifact upload/release credentials.
- [ ] Local commands reproduce CI checks without needing logic hidden in workflow YAML.
- [ ] Cross-target executables are run only on compatible runners/emulators; otherwise artifact validation and manual checks are recorded.

## 8. Mods and add-ons

- [ ] Shipped assets, user add-ons, configuration, cache, and temporary downloads are kept distinct.
- [ ] Add-on discovery and load order are deterministic and documented.
- [ ] Manifests validate IDs, versions, game/API compatibility, and dependencies.
- [ ] Duplicate IDs, missing dependencies, malformed packages, and load failures produce useful diagnostics.
- [ ] Archive extraction prevents path traversal and installs from a validated staging directory.
- [ ] Third-party code is not executed by default; any executable-mod trust model is documented.
- [ ] A safe mode can start without third-party add-ons where add-ons are supported.

## 9. Options and recovery

- [ ] Defaults, saved settings, and command-line override precedence are documented and tested.
- [ ] CLI overrides do not rewrite saved preferences just by launching the game.
- [ ] Unsupported graphics settings cannot be selected as if they were active.
- [ ] Failed renderer/display changes recover to a usable state or have a documented reset procedure.
- [ ] Reset-to-defaults does not delete saves, scores, or other unrelated user data.
- [ ] Keyboard and controller navigation cover essential menus on supported targets.
- [ ] Controller disconnect/reconnect and rebind cancellation do not leave input stuck or corrupt bindings.

## 10. Release artifact hygiene

- [ ] Artifact names include project, version, and target where appropriate.
- [ ] Checksums are published when distributing downloadable archives.
- [ ] Debug symbols are retained or split out where practical.
- [ ] Release notes describe user-visible changes, platform limitations, and known issues.
- [ ] The exact artifact tested is the artifact being published.
