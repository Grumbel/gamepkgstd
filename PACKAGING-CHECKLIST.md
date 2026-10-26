# Cross-Platform Packaging and Release Checklist

This is a release and deployment gate for projects following `DESIGN.md`. The baseline target ambition is Windows x86/x86-64, Android 5.0+ (API 21+), desktop Linux, WebAssembly, and R36S/PortMaster where the project and dependencies support them. Declare the actual support matrix; mark unsupported targets as **N/A with a reason** rather than claiming universal support.

## 1. Project contract

- [ ] Supported platforms, architectures, minimum OS/API levels, runtime requirements, and platform-specific features are documented.
- [ ] Build, test, install, run, and package commands are documented and work from a clean checkout.
- [ ] Top-level `VERSION` is the only manually maintained version source where the project uses this convention (`VERSION-HANDLING.md`).
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
- [ ] dArkOS/ArkOS compatibility is recorded from actual PortMaster/runtime testing, not inferred from the device name alone.
- [ ] WebAssembly builds use Asyncify only when required; JSPI/browser requirements and persistent-storage behavior are documented if used.
- [ ] Android minimum API is explicit (API 21 for Android 5.0); older-API compatibility is not inferred solely from successful compilation.

## 7. Nix-first CI

- [ ] The flake defines authoritative build, test, lint, package, cross-build, artifact-assembly, and deployment logic where applicable.
- [ ] The documented CI flake output actually builds/runs all required checks and required package outputs; `nix flake check` alone does not automatically build every `packages` output.
- [ ] GitHub Actions only provisions Nix, selects the documented aggregate flake output, configures optional caching, and uploads/publishes already-built artifacts using external credentials.
- [ ] One documented command builds all required artifacts/checks for the project profile; another deploys when applicable.
- [ ] Local commands reproduce CI checks without needing logic hidden in workflow YAML.
- [ ] Required target outputs are dependencies of the release aggregate; `nix flake check` alone is not assumed to build all `packages` outputs.
- [ ] Cross-target executables are run only on compatible runners/emulators; otherwise artifact validation and manual checks are recorded.
- [ ] Deployment consumes the exact flake-built output and does not duplicate build steps in CI YAML; credentials remain outside the flake.

## 8. Extensions and user-installed content

- [ ] Shipped assets, user add-ons, configuration, cache, and temporary downloads are kept distinct.
- [ ] Add-on discovery and load order are deterministic and documented.
- [ ] Manifests validate IDs, versions, application/API compatibility, and dependencies.
- [ ] Duplicate IDs, missing dependencies, malformed packages, and load failures produce useful diagnostics.
- [ ] Archive extraction prevents path traversal and installs from a validated staging directory.
- [ ] Third-party code is not executed by default; any executable-mod trust model is documented.
- [ ] A safe mode can start without third-party extensions where extensions are supported.

## 9. User settings and recovery

- [ ] Defaults, saved settings, and command-line override precedence are documented and tested.
- [ ] CLI overrides do not rewrite saved preferences just by launching the game.
- [ ] Unsupported settings cannot be selected as if they were active; interactive applications should provide a recovery path for settings that prevent startup.
- [ ] Failed renderer/display changes recover to a usable state or have a documented reset procedure.
- [ ] Reset-to-defaults does not delete saves, scores, or other unrelated user data.
- [ ] For interactive applications, keyboard/controller navigation covers essential UI paths on supported targets.
- [ ] Input-device disconnect/reconnect and rebinding cancellation do not leave input stuck or corrupt bindings, where those features apply.

## 10. Release artifact hygiene

- [ ] Artifact names include project, version, and target where appropriate.
- [ ] Checksums are published when distributing downloadable archives.
- [ ] Debug symbols are retained or split out where practical.
- [ ] Release notes describe user-visible changes, platform limitations, and known issues.
- [ ] The exact artifact tested is the artifact being published.
