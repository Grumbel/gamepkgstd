# Cross-Platform Project Compliance Checklist

Use this checklist as the sole compliance and release gate for projects following [`DESIGN.md`](DESIGN.md). It applies to games and other distributable software. For each item record **Pass**, **Fail**, **N/A with a reason**, or **Unknown** when evidence is insufficient. Do not mark an item passed merely because a build succeeds: record the command, target, and evidence for checks that matter. Platform-specific sections apply only to declared targets. Game-specific UX checks live in [`GAME-DESIGN.md`](GAME-DESIGN.md). For LLM-driven remediation, follow [`LLM-IMPLEMENTATION-GUIDE.md`](LLM-IMPLEMENTATION-GUIDE.md); do not edit away failed requirements to create an appearance of compliance.

## 1. Project contract and support matrix

- [ ] Project purpose, supported features, platforms, architectures, minimum OS/API versions, and known limitations are documented.
- [ ] Every required target has a declared artifact format, runtime requirements, and validation method.
- [ ] Build, test, package, run, deploy, and clean commands are documented and work from a clean checkout.
- [ ] Version source, release numbering, license, third-party notices, and asset licensing are defined.
- [ ] A release is built from a clean, identified revision; dirty-tree builds are rejected or clearly treated as development builds.
- [ ] Required and optional dependencies/features are explicit; accidental host discovery cannot silently change the product.

## 2. One-command Nix workflow

- [ ] `flake.nix` is the authoritative definition for development tools, builds, tests, cross-compilation, packaging, and deployment.
- [ ] The documented top-level command builds all required release artifacts, or a single documented aggregate output does so.
- [ ] Each supported target has an explicit, discoverable flake output and target-specific dependencies/options.
- [ ] A clean checkout can build required outputs without relying on undeclared files, local state, or network access during the build.
- [ ] GitHub Actions and other CI providers are thin shims: install/setup Nix, select flake outputs, configure cache access, and upload/publish results. Build logic is not duplicated in workflow YAML.
- [ ] Deployment is invoked through the flake-defined workflow; credentials are injected at deployment time and are not embedded in derivations or artifacts.
- [ ] Build outputs and checks are distinguished: CI explicitly builds every required package rather than assuming `nix flake check` builds all `packages` outputs.
- [ ] `checks.<system>` covers applicable formatting/linting, evaluation, compilation, automated tests, package/artifact smoke checks, and static-site checks; each check runs reproducibly with declared inputs.
- [ ] The documented `nix flake check` command is understood as evaluating the flake and building declared checks, not as proof that every package, target runtime, or deployment works.
- [ ] Any `--all-systems` use matches the intended coverage; target executables are only claimed runtime-tested when run on suitable hardware or an explicitly configured emulator.
- [ ] Build and deployment steps fail clearly on errors; a failed build cannot publish stale artifacts as if they were current.

## 3. Language and library choices

- [ ] Language and major libraries fit the declared support matrix; the reason for target-specific exceptions is documented.
- [ ] C++ is used by default for native interactive projects unless a concrete project/target reason supports another language.
- [ ] SDL2 is used as the default game/input layer when R36S/PortMaster compatibility is required; any SDL3 choice has explicit runtime/dependency evidence and target testing.
- [ ] SDL1.2 legacy targets (including GP2X/Wiz-era systems) and OLPC XO-1's SDL/X11 options are evaluated against their actual APIs and performance limits.
- [ ] Conventional desktop GUI framework choice is deliberate; Qt is the default in this project family, and GTK or another framework has a concrete justification if selected.
- [ ] Resizing, viewport changes, UI scale, aspect ratio, and minimum supported resolution are tested; low-resolution handhelds are not assumed to support desktop-sized layouts.
- [ ] Cursor visibility is intentional for controller-only and mouse-driven modes; SDL cursor hiding is tested on the actual target, and a visible cursor that survives SDL hiding is traced to the owning layer before choosing a fix.
- [ ] Desktop mouse interaction can restore the pointer, and handheld virtual-mouse/overlay behavior is tested without breaking other SDL applications.

## 4. Native builds and cross-compilation

- [ ] Native clean build and applicable tests succeed.
- [ ] Cross-builds use explicit target systems, toolchains/sysroots, and target dependencies.
- [ ] Build-host tools and target binaries are separated; generators run on the build host unless a documented emulator is used.
- [ ] Build directories and caches are isolated by target, compiler, and materially different configuration.
- [ ] Cross-configuration does not accidentally execute target binaries or discover host libraries as target dependencies.
- [ ] Target architecture, ABI, minimum OS/API, linkage, and runtime dependencies are checked in the produced artifact.
- [ ] Tests that cannot run on the build host are marked as requiring emulation, a device, or manual validation, not falsely reported as passed.
- [ ] Release metadata records source revision, version, target, configuration, and artifact names.

## 5. Target coverage and deployment

For every target marked supported in the support matrix:

- [ ] The required artifact is produced by the documented flake command from a clean checkout.
- [ ] Artifact architecture, format, minimum runtime, dependencies, and required assets are checked automatically where possible.
- [ ] The artifact is deployed or installed using a documented, repeatable command.
- [ ] The exact artifact intended for release receives a smoke test; testing a different build is not sufficient.
- [ ] A target-specific test is automated where practical; remaining manual/device checks are listed explicitly.
- [ ] Unsupported combinations fail clearly rather than silently producing a host-native artifact.

### Windows

- [ ] Win32 and/or x86-64 support is explicitly declared; each declared architecture is cross-built.
- [ ] Required DLLs/runtime components are bundled or their installation requirements are documented.
- [ ] Paths work from directories containing spaces and non-ASCII characters.
- [ ] The package runs without the build environment, developer PATH, or source tree.
- [ ] GUI executables embed the expected multi-resolution `.ico`; the framework sets window/taskbar icons where needed.
- [ ] Version resources contain accurate product/file version, name, publisher/company, and description where applicable.
- [ ] Installer packages create the intended Start-menu shortcuts and optional desktop shortcut; uninstall removes installer-created integration without deleting user data. Portable archives are exempt from installer-only behavior.
- [ ] File associations, protocol handlers, startup entries, and other shell integration are registered only when implemented, documented, and removable.

### Android

- [ ] Minimum Android API level and supported ABIs are explicit; API 21 / Android 5.0 is the baseline only when claimed by the project.
- [ ] Manifest, package metadata, permissions, native libraries, assets, and signing are handled by the documented release workflow.
- [ ] Storage follows Android's sandbox and scoped-storage model; pause/resume, activity recreation, and process death do not corrupt persistent data.
- [ ] Durable progress is persisted incrementally; correctness does not depend on receiving `onPause`, `onStop`, or another final callback before process death.
- [ ] Transient session/UI state and durable progress are distinguished; returning after backgrounding or activity recreation restores the expected screen, selection, scroll/position, and in-progress work where applicable.
- [ ] Resume/recreation does not duplicate submissions, purchases, network requests, input events, timers, audio playback, or save operations.
- [ ] Graphics surfaces/contexts and other lifecycle-bound resources are recreated after loss; audio, input, networking, and background work are paused, resumed, or cancelled according to a documented policy.
- [ ] Lifecycle behavior is tested by backgrounding and resuming, recreating the activity where applicable, and killing the background process before reopening; recovery is verified on the minimum supported Android version and a current version.
- [ ] VR applications declare their separate device/API requirements instead of inheriting the ordinary Android baseline.
- [ ] Application ID remains stable across compatible updates; label, version metadata, permissions, exported components, and signing mode are correct.
- [ ] Launcher icons include adaptive resources and any fallback resources required by the declared minimum API; check their appearance under different masks/backgrounds.
- [ ] Release signing secrets are not stored in source control or embedded in reproducible build derivations; debug and release artifacts are distinguishable.

### WebAssembly

- [ ] Browser/runtime requirements, memory limits, asset delivery, and persistence behavior are documented.
- [ ] Asyncify or an alternative async integration is selected intentionally; it is not enabled without a demonstrated need or omitted when synchronous assumptions require it.
- [ ] Startup, loading progress, errors, input focus, resize, and browser visibility changes are handled.
- [ ] Persistent data is committed using the chosen browser storage mechanism; the user is not told data is saved before the commit succeeds.
- [ ] Hosting paths, MIME types, compression, cache policy, and deployment destination are tested.
- [ ] The HTML/CSS shell is responsive and usable at mobile and desktop sizes; canvas resolution follows its displayed size and device pixel ratio.
- [ ] Essential controls, loading/errors, and status are accessible outside the canvas where practical; keyboard focus, touch input, page scrolling, and fullscreen behavior are intentional.
- [ ] Relative assets work when hosted below a URL prefix; the page does not assume the site root or undeclared server-side APIs.
- [ ] The page handles module/asset load failure and storage unavailable/full/cleared states without falsely claiming that data was saved.
- [ ] Browser visibility/focus changes do not leave stuck input, audio, or simulation state; resizing and orientation changes are tested where relevant.

### Linux desktop

- [ ] Installed assets and executables use a documented prefix-relative layout; runtime dependencies are resolvable on the intended distribution/runtime.
- [ ] GUI apps install a valid `.desktop` entry with correct stable ID, `Name`, `Exec`, `Icon`, and suitable `Categories`; `Exec` works outside the source/build tree.
- [ ] Icons are installed in a standard or package-format-correct location, resolve from the installed entry, and remain recognizable at actual launcher sizes.
- [ ] AppStream metainfo is supplied and validated when relevant to the distribution/software-catalog workflow; otherwise its omission is justified.
- [ ] Desktop entries and AppStream metadata pass available validators (for example `desktop-file-validate` and `appstreamcli validate`), and the installed package launches from the desktop environment.
- [ ] MIME types, URI handlers, D-Bus activation, autostart, and desktop actions are declared only when implemented and appropriate.
- [ ] `nix build` and `nix run` work if exposed as project interfaces.
- [ ] Sandboxed package permissions are limited to required access when a sandboxed format is offered.

### R36S / PortMaster / ArkOS / dArkOS and similar handhelds

- [ ] SDL2/SDL3 runtime assumptions are explicit and tested on the target firmware; 640×480 is treated as a reference size, not a hard-coded universal display mode.
- [ ] Supported device/firmware combinations and CPU architecture/ABI are documented and verified; names alone are not treated as proof of compatibility.
- [ ] The launcher contract, working directory, environment variables, runtime libraries, controls, display mode, and save location are documented.
- [ ] Bundled libraries do not accidentally override incompatible system libraries; loader paths and environment changes are intentional.
- [ ] The package installs, launches, exits, saves, and resumes correctly on the declared target hardware or a documented equivalent test setup.
- [ ] A controller-operated in-game overlay/pause menu replaces essential keyboard-only options; its shortcut follows the target firmware/runtime conventions and is documented.
- [ ] Overlay input is isolated from gameplay input; pause/resume and close/back behavior do not leave held buttons stuck or leak menu presses into gameplay.
- [ ] Legacy keyboard-only commands/text entry have a documented handheld alternative or an explicit limitation (for example, a virtual keyboard where needed).
- [ ] Save-state support is based on the actual architecture and a real state-capture API/implementation, not assumed from the original platform; unsupported features are not advertised.
- [ ] Save states are distinct from ordinary game saves, use a documented writable location and slots, preserve manual slots from autosave, and report success/failure truthfully.
- [ ] Save/Load State is grouped in a submenu rather than crowding the overlay root menu; Load State and Save State each open a controller-navigable, scrollable slot list.
- [ ] Load State lists existing loadable states; Save State lists existing slots plus an explicit New Save Slot action.
- [ ] Overwriting an existing state requires explicit confirmation that identifies the slot; cancel leaves the prior state intact. Creating a new slot never silently overwrites another manual save.
- [ ] Slot lists remain usable beyond one screenful, keep selection visible while scrolling, distinguish empty/incompatible entries, and provide consistent Back/Cancel behavior and operation feedback.
- [ ] Save-state writes survive interruption safely; corrupt/incompatible states fail without damaging ordinary saves or the previous valid state.
- [ ] Save/load is tested across gameplay and transitions, application restart, corrupt files, and storage errors where applicable; untested hardware behavior is marked Unknown.

## 6. Application identity and OS integration

- [ ] The support matrix identifies which OS integration features are applicable; libraries and headless CLI tools are not forced to ship GUI launchers.
- [ ] Application ID, display name, executable name, icon, package metadata, and user-data identity are consistent where appropriate; any identity change has an explicit upgrade/migration plan.
- [ ] Required icons and metadata are included in the final artifact, not merely present in the source tree or build directory.
- [ ] A GUI app is discoverable and launchable through the normal OS interface on each declared desktop/mobile target, and shows the intended icon/name.
- [ ] Optional integrations (shortcuts, MIME/file associations, URI handlers, autostart, shell extensions, store publication) are included only when implemented and suitable; cleanup/uninstall behavior is checked.
- [ ] Version, publisher/developer, license, project URL, and release metadata are accurate wherever the package format supports or requires them.
- [ ] Validation evidence distinguishes static metadata validation from a real installed-package launch test.

## 7. Man pages and user documentation

- [ ] Every installed, user-facing executable has a maintained man page, including GUI applications and games; internal build-only scripts are excluded unless exposed as supported commands.
- [ ] Pages use appropriate sections (normally 1 for commands/apps, 5 for configuration/data formats, 3 for public library APIs) and are installed under the standard man directory on Unix-like targets.
- [ ] Pages document actual invocation, options/subcommands, environment variables, relevant paths, exit behavior, examples, and destructive actions; no placeholder/stub pages.
- [ ] `--help` and man pages agree with implemented behavior; option and configuration changes update the documentation and tests together.
- [ ] Man pages are built/installed by the authoritative flake/package workflow, and the final package contains them where applicable.
- [ ] Non-Unix releases retain the maintained documentation source and provide readable documentation appropriate to that artifact; lack of a native man viewer is not treated as a failure.
- [ ] Validation checks page presence and section/path correctness, and uses a formatter/linter or renders pages when available.

## 8. Installed artifact and runtime behavior

Run the installed artifact from a working directory outside the source and build trees.

- [ ] The program starts from a clean install with no pre-existing user configuration.
- [ ] Required assets and runtime libraries resolve without access to the source tree or build environment.
- [ ] `--help` and `--version` work without unnecessarily initializing graphics, audio, network, or other expensive subsystems.
- [ ] Invalid CLI arguments fail with a useful diagnostic and non-zero exit status.
- [ ] No writes go to the install prefix or current working directory by default unless that behavior is explicitly intended.
- [ ] Missing, unwritable, or malformed user directories/configuration produce useful errors and do not destroy unrelated data.
- [ ] Shutdown and interrupted writes do not leave persistent data corrupt; atomic writes use staging on the destination filesystem.
- [ ] Startup, shutdown, and long-running operations remain responsive or report progress; failures are visible and actionable.

## 9. Paths, configuration, and CLI

- [ ] Installed assets, user data, configuration, persistent state, cache, temporary files, and runtime files have distinct documented purposes.
- [ ] CLI path options use the consistent vocabulary in `DESIGN.md`; deprecated aliases, if retained, are tested and documented.
- [ ] Linux desktop paths follow XDG base-directory rules; Windows uses appropriate known folders; Android and web use their platform storage models.
- [ ] Portable mode has explicit detection/activation rules, keeps writable files in its documented location, and does not silently fall back to writing elsewhere.
- [ ] Handheld launchers' supplied paths and environment variables are respected.
- [ ] Relative-path resolution is deterministic and documented; it does not accidentally depend on the caller's working directory.
- [ ] Configuration precedence is documented and tested; command-line overrides do not silently rewrite saved preferences.
- [ ] Config migration is versioned and tested; unknown or invalid values have a defined handling policy.
- [ ] Reset options clearly state their scope and do not remove saves or unrelated user data.
- [ ] `--no-save`, if provided, disables all persistent gameplay/state writes covered by its documented meaning.

## 10. Data safety, updates, and recovery

- [ ] User data is kept separate from installed program files and replaceable cache data.
- [ ] Critical writes preserve the previous valid file on failure; staging, replacement, and durability guarantees match the platform and filesystem as closely as practical.
- [ ] Saves/state have a versioning and migration strategy; updates do not silently overwrite data that cannot be read.
- [ ] Interrupted saves, full disks, permission failures, and invalid/corrupt files are handled without avoidable data loss.
- [ ] Recovery from an interrupted or partially completed write is tested where persistent data exists.
- [ ] Uninstall/update behavior for user data is documented; uninstall does not unexpectedly delete user-created content.
- [ ] Downgrade support or incompatibility is declared; older versions cannot silently overwrite or destroy newer user data.
- [ ] A broken configuration or graphics setting can be reset without requiring the normal UI to start.
- [ ] Destructive actions are clearly identified and confirmed when data loss would be meaningful.
- [ ] Backward/forward compatibility limits are documented; downgrade behavior is tested or explicitly unsupported.

## 11. Extensions, mods, plugins, and add-ons

- [ ] The project documents whether extensions are supported, which types are allowed, and where they are discovered and installed.
- [ ] Each supported extension type has a stable identifier and a documented metadata/version format.
- [ ] Discovery and load order are deterministic; duplicate IDs and missing/incompatible dependencies produce useful diagnostics.
- [ ] Compatibility with the host application/API and extension dependencies is checked before activation where possible.
- [ ] Archive extraction rejects path traversal, unsafe links, and writes outside the intended destination.
- [ ] Downloaded executable code is not silently run; trust and security implications are communicated.
- [ ] Extension settings and user content are stored separately from global preferences and shipped assets.
- [ ] An extension can be disabled or removed without damaging unrelated data; startup recovery can bypass extensions if they prevent launch.
- [ ] Updates that remove or change extension content handle dependent saves/projects explicitly.

## 11. UI, accessibility, and interaction (all applicable applications)

- [ ] Supported input modes are declared per target; essential operations are discoverable and reachable using every claimed mode.
- [ ] Device events map to shared semantic actions where appropriate; switching input methods does not reset state, duplicate actions, or leave input stuck.
- [ ] Keyboard focus/navigation, mouse targeting, touchscreen hit targets/gestures, controller navigation, text entry, dialogs, and cancellation work for the modes actually supported.
- [ ] UI text remains legible at supported sizes, scaling levels, resolutions, and display configurations.
- [ ] Unicode paths/content and locale-sensitive formatting work where applicable; text input methods are tested on supported platforms.
- [ ] Color is not the sole way important status or errors are communicated.
- [ ] Error messages explain what failed and offer a useful next step where possible.
- [ ] Long operations expose progress or a responsive cancel/stop path when practical.
- [ ] Destructive or irreversible actions are clearly named and are not triggered by ambiguous controls.
- [ ] Accessibility features are documented and tested against the relevant project scope; applicable items from the Xbox Accessibility Guidelines or equivalent guidance are reviewed.

## 12. Game-specific behavior (if applicable)

Use [`GAME-DESIGN.md`](GAME-DESIGN.md) for the detailed checklist.

- [ ] The game is playable using each declared input method through startup, menus, gameplay, pause/dialogs, settings, and exit.
- [ ] Touch targets are usable without hover or precise pointer input; essential actions do not depend on gestures alone.
- [ ] Keyboard, mouse, touch, and controller can be mixed where claimed without stuck input, duplicate actions, or lost focus/state.
- [ ] Settings reflect the mode actually applied; unsupported graphics modes cannot appear active.
- [ ] Renderer/display changes recover from failure or provide a documented safe reset path.
- [ ] Controller disconnect/reconnect and rebinding cancellation do not leave input stuck or lose previous bindings.
- [ ] Pause, quit, abort, restart, and destructive save/progression actions are distinct and behave predictably.
- [ ] Audio/visual cues, subtitles, remapping, text size, contrast, motion and photosensitivity options are reviewed for relevance to the game.

## 13. Console-style quality and lifecycle checks

These are platform-independent quality checks inspired by public console certification requirements, not a claim of certification and not a substitute for the current requirements of any console/store. Apply the relevant checks to desktop, mobile, web, and handheld releases too.

- [ ] Every supported mode and major feature can be entered, used, and exited without a crash, hang, dead end, or permanently blocked progression.
- [ ] The application remains responsive to input; loading waits are bounded where possible and show progress or a clear status when lengthy.
- [ ] Every essential menu and flow is operable with the primary supported input device; focus is visible and navigation has no dead ends.
- [ ] Settings take effect immediately when promised and remain correct after restart when persistent.
- [ ] Closing, suspending, backgrounding, losing focus, sleeping, and resuming do not cause crashes, hangs, duplicate actions, or avoidable loss of progress.
- [ ] Where the platform can terminate a suspended process without a final callback, important state is already durable; resume restores the documented point in the workflow rather than merely restarting blindly. Session/UI restoration and durable saves have distinct, tested contracts.
- [ ] Network/service failure, offline startup, timeout, and reconnection are handled gracefully when networking exists; unavailable optional services do not make unrelated offline features unusable.
- [ ] Saves and settings remain associated with the correct user/profile where profiles exist; switching users does not leak or overwrite another user's data.
- [ ] Updates preserve compatible saves and user content; version transitions are tested with data created by the previous released version.
- [ ] If purchases, entitlements, downloadable content, achievements, or online accounts exist, missing services and invalid/unavailable entitlements are handled without corrupting local progress.
- [ ] Fresh install, existing-user upgrade, no-save/no-profile, offline, low-storage, permission-denied, and corrupted-config scenarios are tested where relevant.
- [ ] No critical crash, data-loss bug, softlock, inaccessible required flow, or severe rendering/input failure remains in the release candidate.

## 14. Release and deployment hygiene

- [ ] Artifact names identify project, version, and target.
- [ ] Checksums and signatures are published or verified where required by the distribution channel.
- [ ] Debug symbols are retained or split out where practical; release artifacts do not accidentally include secrets or development-only assets.
- [ ] License and third-party notices are shipped where required.
- [ ] Third-party dependency sources, versions, and licenses are tracked; builds do not perform undeclared network fetches.
- [ ] Release notes list user-visible changes, compatibility/migration notes, known issues, and supported targets.
- [ ] Deployment publishes only artifacts built and tested for the current revision; failed jobs cannot leave a misleading partial release.
- [ ] A post-deployment check verifies the published endpoint/artifact is reachable and corresponds to the intended revision.
- [ ] A rollback/recovery procedure exists for deployment failures where the hosting/distribution system supports it.

## 15. Evidence and sign-off

- [ ] Every required item is marked Pass or Fail; each N/A includes a reason.
- [ ] Automated checks are linked to the flake output/job and their logs are retained as appropriate.
- [ ] Manual checks identify tester, target/device/runtime, artifact revision, and result.
- [ ] Release sign-off identifies the exact source revision and artifact hashes.
- [ ] Platform/store certification is claimed only after the applicable official process has actually been completed.
