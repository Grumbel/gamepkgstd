# Cross-Platform Software Packaging and Runtime Guidelines

A reusable engineering and distribution contract for applications, games, libraries, command-line tools, and web projects. The goal is to make projects reproducibly buildable, testable, packageable, and deployable from one NixOS `flake.nix`, with GitHub Actions acting only as a thin invocation shim.

These guidelines are informed by several game projects, but the shared contract applies to applications, games, libraries, command-line tools, and web projects. Game-only behavior lives in [`GAME-DESIGN.md`](GAME-DESIGN.md). For an LLM-driven remediation task, follow [`LLM-IMPLEMENTATION-GUIDE.md`](LLM-IMPLEMENTATION-GUIDE.md): inspect first, establish applicability, make evidence-backed changes, test them, and report what remains unverified. Do not treat every example or optional capability as a universal requirement.

Related documents:

- `VERSION-HANDLING.md` — version source and propagation.
- `NIX-DEVELOP-SHELL.md` — reusable out-of-tree CMake development workflow.
- `PACKAGING-CHECKLIST.md` — sole compliance and release gate.
- `LLM-IMPLEMENTATION-GUIDE.md` — procedure for applying this contract to an existing repository.
- `GAME-DESIGN.md` — optional game-specific UI/input guidance.
- `AGENTS.md` — repository continuity and git-bundle handoff rules.

## 1. Goals and supported target profiles

The reference target profile for cross-platform interactive applications in this work line is one source tree and one authoritative flake capable of producing the applicable artifacts for:

| Target family | Expected output | Key constraint |
|---|---|---|
| Windows x86 and x86-64 | Portable directory/ZIP and optionally installer | Build with explicit MinGW toolchains; include runtime DLLs and assets. |
| Android 5.0+ (API 21+) | Installable APK/AAB as appropriate | Keep the minimum API explicit and test old-device runtime compatibility. VR applications may have a separate profile; see [Daydrym](https://github.com/Grumbel/daydrym). |
| R36S-class handhelds | PortMaster-compatible package when applicable | Target the actual userspace ABI, graphics/input stack and PortMaster packaging contract, not merely “ARM Linux”. |
| Desktop Linux | Nix package and/or portable distribution formats | Keep runtime dependencies and data lookup deterministic. |
| WebAssembly | Browser-deployable site/artifact | Define browser requirements, storage model, and whether asynchronous JS integration is required. |

This reference profile is not a universal mandate. Each repository must declare its actual support matrix, architecture/ABI, minimum OS/API, relevant graphics/input backend, artifact format, and validation status. Targets may be excluded with a brief reason. A target is supported only when its artifact is built and its runtime assumptions are checked; a successful compile alone is insufficient.

**One-click** means one documented flake entry point can build all required artifacts and checks for a project profile, and a documented deployment entry point can publish the outputs that are safe to publish. It does not mean every target binary can run on the build host, that credentials belong in the flake, or that hardware testing can always be automated.

## Platform implementation notes

### Windows x86 and x86-64

Expose separate target artifacts for 32-bit and 64-bit Windows when both are genuinely supported. Use a pinned MinGW toolchain and target dependencies, include required DLLs and assets, and smoke-test from an extracted package with no developer environment. Keep architecture and minimum Windows version in artifact metadata. Do not assume that a successful cross-link proves runtime compatibility.

### Android 5.0 and later

Android 5.0 corresponds to API level 21. Treat this as an explicit compatibility target, not merely a manifest setting: use a compatible NDK/toolchain and avoid accidentally linking newer-only APIs. Build APK/AAB artifacts through the flake, but keep signing credentials outside reproducible derivations. VR apps may need a separate compatibility profile and are not automatically held to the same API/runtime assumptions; [Daydrym](https://github.com/Grumbel/daydrym) is the reference exception named by this project.

Treat lifecycle transitions as normal operation, not exceptional cases. Persist important progress incrementally; do not rely on `onPause`, `onStop`, or a final shutdown callback being delivered before process death. Save enough lightweight instance state to restore the user's current place after activity recreation, and restore it without duplicating actions or corrupting durable progress. Handle pause/background, resume/foreground, activity recreation, and process death separately. Stop or pause audio and active input appropriately, and recreate graphics/device resources after surface or context loss. Lifecycle callbacks may occur repeatedly, so handling them must be idempotent. Test by backgrounding the app, returning to it, rotating/recreating the activity where applicable, and killing the background process before reopening; verify both the visible session and durable progress recover according to the documented contract.

### R36S, PortMaster, ArkOS, and dArkOS

“R36S” is a device family, while ArkOS/dArkOS are firmware distributions and PortMaster is a packaging/runtime ecosystem. Define the target in terms of CPU ABI, userspace ABI, available graphics/input APIs, runtime libraries, and PortMaster launcher contract. Do not treat a generic `aarch64-linux` cross build as automatically suitable for all handhelds. Use the [PortMaster packaging guide](https://portmaster.games/packaging.html), [PortMaster porting guide](https://portmaster.games/porting.html), and [supported firmware list](https://portmaster.games/faq.html) as the live references. PortMaster currently lists dArkOS as supported; that does not remove the need to validate the particular R36S image, ABI, launcher paths, and bundled-library requirements. Prefer one PortMaster artifact where the contract genuinely permits it, rather than duplicating packages per firmware without a technical reason.

### WebAssembly and browser-hosted applications

Default to a non-Asyncify build when the program does not need synchronous-looking calls that wait for asynchronous JavaScript or yield back to the browser event loop. Add Asyncify only when required by the code/runtime design and test its code-size/performance cost. Emscripten also documents JSPI, which depends on support in the chosen browser/runtime and is not a universal replacement. Prefer event-driven/nonblocking integration when practical, especially for loading and persistence. Declare browser requirements and test startup, input, resize/focus, downloads, and persistence in a real browser.

#### Website shell and layout

Treat the HTML/CSS page as a real user interface around the WebAssembly program, not merely an empty page containing a canvas. Tailor the shell to the app: a game may use a mostly full-window canvas, while a tool may need a toolbar, panels, help, and ordinary page scrolling. Do not force every project into the same layout.

- Include a correct viewport declaration and a responsive layout that works at narrow mobile widths, desktop sizes, browser zoom, and landscape/portrait changes where relevant. Respect mobile safe areas. Avoid fixed pixel layouts that clip controls or make text unreadable.
- Size the canvas from its displayed CSS size and device pixel ratio, with a deliberate maximum-resolution/performance policy. Resize the rendering surface when the container or display changes; do not stretch a stale framebuffer or assume a fixed desktop resolution.
- Keep important controls and status outside the canvas when practical, using semantic HTML. Provide keyboard access, visible focus, touch-friendly controls where applicable, and accessible names/instructions. Do not make essential status, errors, or actions available only as pixels drawn inside the canvas.
- Define page scrolling and input ownership intentionally. Prevent accidental page scrolling only while it interferes with an active app interaction; preserve browser navigation, text selection, zoom, and normal scrolling outside that interaction. Fullscreen must be user-initiated and optional where practical.
- Show loading progress when startup/download time is material, a useful error if the WASM module or assets fail to load, and a retry/reload path where safe. Handle focus loss and page visibility changes without leaving stuck input, audio, or simulation state.
- State what happens when browser storage is unavailable, full, cleared, or private-mode restricted. Confirm persistence only after it succeeds; provide export/import or another recovery path for valuable user-created data when feasible.
- Build and test the exact static output intended for hosting. Verify relative asset URLs when hosted below a path prefix (including project pages), correct WASM MIME type, compression and cache headers, and that cached HTML cannot accidentally reference incompatible old/new assets. Do not assume the site is hosted at `/` or has a server-side API unless declared.
- Keep the page usable if the canvas cannot initialize: show a readable failure message and any available instructions or data-recovery actions.

References: [Emscripten Asyncify and JSPI](https://emscripten.org/docs/porting/asyncify.html), [Emscripten deployment guidance](https://emscripten.org/docs/compiling/Deploying-Pages.html), [Android NDK compatibility](https://github.com/android/ndk/wiki/Compatibility), [PortMaster packaging](https://portmaster.games/packaging.html).

## 2. One authoritative Nix flake

`flake.nix` is the source of truth for dependency versions, build options, native and cross builds, tests, packaging, artifact assembly, and deployment logic. Do not maintain a second authoritative implementation in shell scripts, Makefiles, or CI YAML. Thin project helper scripts may call named flake outputs; they must not duplicate the build logic.

A project should expose a small, discoverable interface, for example:

- `nix build .#default` for the default native package;
- `nix build .#<target>` (or the project's documented equivalent) for target artifacts;
- `nix flake check` for checks that are intentionally part of the standard verification path;
- a named aggregate output such as `.#ci` / `.#release` that depends on all required builds and tests;
- a named deployment output or app such as `.#deploy` for publishing, when deployment is applicable.

Choose output names that fit Nix conventions and the flake's supported systems; the examples are an interface idea, not a requirement to create every spelling. Document the exact one-command build/test/deploy invocations in the README. A CI aggregate must explicitly depend on every required package: `nix flake check` does not by itself build every derivation under `packages`.

The detailed CI/deployment contract is in section 14. In short, CI invokes named flake outputs and may provision Nix, configure caching, inject credentials at deployment time, and upload/publish the already-built output. It must not maintain a second implementation of build or test logic. For a website, build the static site through the flake and publish that known output; see [the Kurvenrausch website repository](https://github.com/kurvenrausch/kurvenrausch.github.io) for a project reference.

## 3. Build-system independence and development workflow




### Preferred stack (where applicable)

- Use the project’s natural build system (CMake is a good fit for C/C++ projects, not a universal requirement).
- Use the Nix flake to pin tools and dependencies and to orchestrate native/cross builds and packaging.
- Optional: Ninja, ccache, language-specific build tools, or generators when they materially help.

### Develop vs package split

Follow the pattern in `NIX-DEVELOP-SHELL.md`:

| Concern | Develop (`nix develop`) | Package (`nix build`) |
|---------|-------------------------|------------------------|
| Build type | `Debug` | `RelWithDebInfo` (or Release) + `separateDebugInfo` |
| Build location | Out-of-tree, sticky (`/tmp/project-build` or `$HOME/.cache/…`) | Nix store |
| Helpers | `project-configure`, `project-build`, `project-run`, `project-run-gdb` as real PATH scripts | Wrappers via `makeWrapper` / `qtWrapperArgs` |
| Nested C++ deps | Live sibling checkouts when possible | Flake inputs / fixed store paths |
| Env for dlopen | Mirror package wrapper (`LD_LIBRARY_PATH`, plugin paths, `--data-dir`) | Handled by wrapper |

### Packaging artefacts

For conventional Linux desktop packages, a typical install layout is prefix-relative:

```
bin/<app>
share/<app>/…          # data, shaders, …
share/applications/<app>.desktop
share/icons/hicolor/…/<app>.png (or .svg)
share/metainfo/<app>.appdata.xml  (or .metainfo.xml)
share/man/man1/<app>.1 (or the appropriate section)
```

Produce the formats applicable to the declared support profile, for example:

- Linux desktop metadata (desktop entry, icons, AppStream)
- Windows ZIP / installer assembled by the flake
- PortMaster-compatible handheld package and launcher
- Web site/artifact bundle and deployment output
- Android APK/AAB

Use the project's chosen license consistently, preserve third-party license notices, and follow REUSE/SPDX conventions where adopted or required by the project.

---

## 4. Path Handling

Keep the *kind of data* separate from the *physical directory used on a platform*. A platform may place several kinds in one writable root, but code and options must retain the distinction. Never infer a writable location from the current working directory.

### Path categories

| Category | Contents | Desktop Linux / Unix default | Writable? |
|---|---|---|---|
| Installed assets | Shipped resources, templates, models, images, audio, shaders, built-in content | Install prefix, commonly `share/<app>` | No |
| User data | User-installed mods/add-ons and content the user explicitly imports | `$XDG_DATA_HOME/<app>`; default `~/.local/share/<app>` | Yes |
| Config | Preferences, application settings, key bindings, UI settings | `$XDG_CONFIG_HOME/<app>`; default `~/.config/<app>` | Yes |
| State | Saved documents, progress, history, and other durable user state | `$XDG_STATE_HOME/<app>`; default `~/.local/state/<app>` | Yes |
| Cache | Re-creatable downloads, shader caches, thumbnails | `$XDG_CACHE_HOME/<app>`; default `~/.cache/<app>` | Yes |
| Temporary files | Incomplete downloads, extraction staging, disposable scratch output | OS temporary-directory API, with a private per-app subdirectory if needed | Yes, ephemeral |
| Runtime files | Locks, sockets, other short-lived coordination files | `$XDG_RUNTIME_DIR/<app>` when valid and available | Yes, ephemeral |

`XDG_DATA_HOME`, `XDG_CONFIG_HOME`, `XDG_STATE_HOME`, and `XDG_CACHE_HOME` are *base directories*, not the application's final directory. Append the application-specific directory exactly once. Honor an XDG variable only when it is set to an absolute path; otherwise use the specification's default. Do not treat `XDG_RUNTIME_DIR` as a general temp directory: it is optional, user-owned, private, and intended for runtime objects. Validate its ownership and permissions before using it. Use the platform's temporary-directory API for scratch files.

The XDG Base Directory Specification is a desktop Unix convention, not a universal path API. Do not impose XDG semantics on Windows, Android, web builds, or a handheld port merely because the code uses a cross-platform framework.

### One consistent path-option vocabulary

Use hyphenated long options consistently. The recommended overrides are:

| Option | Meaning |
|---|---|
| `--data-dir PATH` | Root of installed/read-only application assets; preferred replacement for legacy `--datadir` |
| `--user-data-dir PATH` | User-managed content, including installed mods and add-ons |
| `--config-dir PATH` | Preferences and input bindings only |
| `--state-dir PATH` | Persistent application state |
| `--cache-dir PATH` | Re-creatable cache files |
| `--temp-dir PATH` | Scratch files for this process; mainly useful for diagnostics, testing, and constrained launchers |
| `--show-paths` | Print all resolved paths and their source, then exit |

These are optional common conventions, not a requirement to expose every override in every application. Keep the spelling consistent for every path option that is exposed. If compatibility requires the legacy `--datadir`, retain it as a deprecated alias for `--data-dir`, document the alias, and make both feed the same internal field. Do not silently reinterpret `--data-dir` as the user's writable content directory.

Path resolution order for each independently overridable category is:

1. Explicit CLI option for that category.
2. Documented application-specific environment override, if provided by the launcher.
3. Explicit portable-mode root, if portable mode is enabled.
4. Native platform default.

Do not combine these sources ad hoc. `--config-dir` must not relocate persistent state, extensions, cache, or temp files. A portable root may intentionally group categories below one root, but they should still use separate subdirectories. `--show-paths` should report the resolved location and whether it came from CLI, environment, portable mode, or a platform default.

### Linux and Unix desktop: XDG

For a normal installed desktop build, use the XDG paths in the category table above. Shipped assets are found through the install layout or an explicit `--data-dir`; they do not belong in a writable XDG directory. Do not write into the install prefix or current working directory by default.

Use the home directory only to construct the documented fallback when the relevant XDG variable is unset or invalid. If neither a valid base directory nor a usable home directory can be resolved, report the problem rather than falling back to a relative path. Create application subdirectories as needed and report actionable errors when they cannot be created.

### Windows

For an installed build, use Windows known-folder APIs (or a library abstraction that maps to them) rather than assuming Unix home paths:

- Config: Roaming AppData, typically `%APPDATA%/<Vendor>/<App>`.
- State and user data: Local AppData by default, typically `%LOCALAPPDATA%/<Vendor>/<App>`; use Roaming AppData only when roaming is actually desired.
- Cache and temporary files: Local AppData cache subdirectory and the OS temporary-directory API, respectively.
- Installed assets: the package's installed data directory, resolved relative to the executable or through an installed-resource lookup.

SDL path helpers may be used where their behavior matches the intended category, but do not use one generic preference path for every kind of data. Do not write beside the executable in an ordinary installation.

### Portable builds and removable installations

Portable mode is an explicit distribution or launch profile, not a guess based only on the executable's location. Enable it through a documented marker file, launcher setting, or `--portable` option. The package must state which mechanism is authoritative.

- Keep the shipped application and assets separate from writable user data.
- Put writable files below a documented directory such as `userdata/config`, `userdata/state`, `userdata/data`, `userdata/cache`, and `userdata/tmp` beside the portable package, or under a launcher-supplied writable root.
- Never assume the executable directory is writable. If portable storage is read-only or full, explain the failure and offer the native user-directory mode where possible; do not silently start writing elsewhere.
- Portable mode must not unexpectedly import or modify the installed version's normal user profile. Make profile selection visible in diagnostics.
- Preserve paths containing spaces and non-ASCII characters. Do not build paths by shell-concatenating untrusted strings.

### R36S / PortMaster / ArkOS and similar handheld ports

Treat these as a distinct *portable launcher profile*, even though the underlying system may be Linux. Follow the platform's current launcher contract rather than applying desktop XDG rules blindly.

- Prefer a documented launcher-provided path override for writable files. Use a project-specific variable or a shared `APP_*_DIR` convention, and define whether it names a base root or a final category directory.
- If the PortMaster contract supplies a final `conf/` directory through a variable such as `XDG_CONFIG_HOME` or a launcher-specific variable, do not append another application directory to it. This exception must be explicit in the port integration, not inferred globally from a path merely ending in `conf`.
- Keep save data and user content in the launcher/platform's expected writable location; do not assume the binary's directory is writable or that the current working directory is stable.
- Document the actual layout, environment overrides, and migration behavior in the port README and launcher script.
- Keep the runtime library environment intentional. Do not let a host `LD_LIBRARY_PATH` accidentally select incompatible SDL, GLES, or other shared libraries; clear or set it in the launcher only when required by the target runtime.

### Android and web

- **Android:** use the app's sandboxed internal storage APIs for writable config/state/cache, and platform-approved external storage only when the feature and permissions require it. Treat APK assets as read-only virtual assets; do not assume `$HOME`, a normal executable-relative `data/` directory, or unrestricted `/sdcard` access. Persist durable progress incrementally and use platform instance-state mechanisms for transient UI/session restoration. Never depend on receiving a last-chance lifecycle callback before the OS kills the process.
- **Emscripten/web:** use the virtual filesystem and a documented mount for persistent data. Distinguish in-memory temporary files from persisted storage, and flush asynchronous persistence (for example IndexedDB-backed storage) according to the runtime's API. Do not promise persistence until the flush completes.

### Asset lookup and virtual assets

Asset lookup must be deterministic. A typical order is: explicit `--data-dir`, documented application-specific environment override, package-relative asset location when the distribution layout defines one, then a compiled install location. Do not silently fall back to an unrelated directory or the current working directory. A virtual asset abstraction such as `datadir://...` may be useful for APK/web assets, but its scheme and semantics must be documented and must not leak into ordinary filesystem APIs.

### Path correctness and migration

- Normalize and validate paths at the platform boundary, but preserve the user's intended path and avoid resolving symlinks unless required.
- Use native path APIs rather than assuming `/` separators or UTF-8 conversion behavior on every platform.
- Make directory creation idempotent. Use restrictive permissions for private saves/config where supported.
- Temporary files must be uniquely and safely created; never use predictable names in shared temp directories. For atomic replacement of a durable file, create the staging file in the destination directory so the final rename can remain on the same filesystem. `--temp-dir` does not redirect these atomic-save staging files.
- When a release changes a path convention, migrate or continue reading the old location deliberately. Never delete the old directory automatically. If both old and new locations contain data, define a deterministic conflict policy and explain it in release notes.

---

## 5. Command-Line Interface

Use a small, documented CLI that is stable enough for scripts, launchers, tests, and support. Adopt only options that make sense for the program. The canonical path-option spellings are defined in section 4; option semantics, precedence, diagnostics, and suggested non-path options are defined in section 11.

At minimum, provide `--help` and `--version` where appropriate. Applications with meaningful writable paths should provide a way to diagnose resolved paths, normally `--show-paths` or `--diagnostics`. CLI behavior must be deterministic, must not depend accidentally on the caller's working directory, and must not silently change saved preferences merely because an option was supplied at startup.

---

## 6. Compliance and release checklist

[`PACKAGING-CHECKLIST.md`](PACKAGING-CHECKLIST.md) is the sole compliance and release checklist. Do not maintain a second copy of checklist items here. This document defines the requirements; the checklist records their applicability, pass/fail status, and evidence.

---

## 7. External References
| Topic | Reference |
|-------|-----------|
| Config / state directories | [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html) |
| Desktop integration | [Desktop Entry Specification](https://specifications.freedesktop.org/desktop-entry-spec/desktop-entry-spec-latest.html) |
| App metadata | [AppStream](https://www.freedesktop.org/software/appstream/docs/) |
| Versioning | [Semantic Versioning](https://semver.org/) |
| PortMaster | https://portmaster.games/ (handheld packaging conventions) |
| REUSE | https://reuse.software/ |
| Nix flakes and `nix flake check` | https://nix.dev/manual/nix/latest/command-ref/new-cli/nix3-flake-check |
| Windows known folders | https://learn.microsoft.com/en-us/windows/win32/shell/knownfolderid |

---

## 8. Implementation Notes and Anti-Patterns
**Do**

- Derive every version string from `VERSION`.
- Keep develop-shell helpers as real executables on `PATH`.
- Fail early when `PROJECT_SOURCE` / user directories cannot be resolved.
- Mirror package-wrapper environment variables inside `nix develop`.
- Document platform-specific path rules next to the code that implements them.
- Track third-party dependency sources, versions, licenses, and required notices. Keep builds free of undeclared network fetches; define a process for updating dependencies and responding to known vulnerabilities.
- Preserve Unicode paths and text. Use locale-aware formatting for user-facing numbers/dates where appropriate, and test text input methods and non-ASCII content on supported platforms.

**Avoid**

- Writing saves into the install prefix or CWD by default.
- Hard-coding `project(… VERSION 0.1.0)`.
- Silent fallback to a relative path when `$HOME` / XDG is missing.
- Framework-specific environment variables in projects that do not use that framework.
- Bundling “delta-only” git history; always ship cumulative bundles (see `AGENTS.md`).

---

## 9. Normative Levels and Project Profiles
These notes describe a **shared baseline**, not a requirement to ship every platform or feature. Apply MUST/SHOULD/MAY to project changes consistently. A project profile must state which target families and features are supported, which requirements apply, and why any SHOULD/MUST exception exists. Mark each rule in project-specific documentation as one of:

- **MUST**: required for a supported build or release; deviations need an explicit reason.
- **SHOULD**: the default recommendation; document a concrete reason when deviating.
- **MAY**: optional capability, implemented only when it fits the project.

Every project should declare its supported targets and input modes. Do not advertise a target merely because its toolchain can compile the code. A project can have a small desktop profile and add Windows, web, Android, or handheld profiles later.

## 10. Build and Cross-Compilation Rules
- Keep configure, build, test, install, and package steps separable. The package must be built from an explicit source revision and must not depend on files in the developer's home directory.
- Use out-of-tree builds. Do not share a CMake build directory between host and target architectures, compilers, toolchains, or materially different build options.
- Cross-builds must use the target compiler, linker, sysroot, and target dependencies consistently. Build-time generators that execute during the build must be built for the **build host**, not the target. Make this distinction explicit for code generators and asset tools.
- Never run a target executable as an ordinary configure-time test during cross-compilation. Use a native host tool, a cross-compilation emulator when deliberately configured, or defer the test to target hardware/CI.
- Separate build-time tools from target runtime dependencies. Do not accidentally link a host library or pick up host headers through ambient environment variables.
- Provide a clear target selection convention through named flake outputs or documented target attributes. The default aggregate should build all required release targets; target-specific outputs should also be available individually. Avoid hidden environment state that changes what an ambiguous `build` command produces.
- Treat compiler flags, feature switches, and dependency versions as part of the build definition. Do not silently use optional system libraries found on the build machine if that changes the intended package features.
- Test the installed package, not only the build-tree executable. Stage installation into a temporary prefix and run from an unrelated working directory to catch missing assets, accidental source-tree references, and incorrect relative paths.
- For targets that cannot be run in CI, validate the artifact format and architecture, inspect linked dependencies, and document the remaining manual smoke test. Do not report an unrun target test as passing.

### Release build expectations

A release artifact MUST be produced from a clean, identified source revision (normally a release tag) with documented commands. Reject dirty release trees, or explicitly classify the output as a development build that cannot be published as a release. Record the source revision, version, target, build options, and artifact names. Keep debug symbols available where practical, either in the artifact or as a separate debug package. Do not strip symbols before extracting or preserving separate debug information.

A release checklist should cover at least: clean configure/build, tests that are available, install-tree launch, version output, asset discovery, configuration/save directory creation, and a smoke test on each supported runtime family. Add signing, checksums, and reproducible-build verification where the distribution channel requires or supports them.

## 11. CLI and Configuration Contract
### General rules

- `--help` and `--version` must work without initializing optional UI, audio, devices, or loading application assets. They should exit successfully.
- Invalid options and invalid values must produce a concise error on stderr and a non-zero exit status. Help text must show accepted values and defaults where useful.
- Support `--` to end option parsing if positional arguments are supported. Do not reinterpret unknown options as filenames silently.
- Prefer long options with explicit values for settings (`--renderer gles`) over a growing collection of overlapping boolean aliases. Keep `-h` and `-v` for help and version; only add short aliases when they are unambiguous and stable.
- Paths supplied by the user must be treated as paths, not shell fragments. Quote paths in launcher scripts and preserve spaces and non-ASCII characters.
- Document whether a flag is startup-only, persistent, diagnostic, or intended for automation. Developer flags should not silently alter player saves or preferences.
- `--show-paths` (or equivalent diagnostics) should report resolved asset, user-data, config, state, cache, temp, and runtime paths where applicable, without dumping secrets or unrelated environment variables.

### Precedence and persistence

Use this precedence for ordinary preferences:

1. Built-in defaults
2. Saved configuration
3. Explicit command-line overrides for this invocation

An explicit command-line option must not rewrite the saved configuration merely because the game starts. A change made in the UI should be persisted according to the application's configuration policy. Make this distinction visible in code, preferably through a parsed options structure that records which values were explicitly overridden, rather than scattered ad-hoc checks.

`--config-dir`, `--data-dir`, `--state-dir`, and similar path overrides should affect only the paths they name. Do not let a config-directory override unexpectedly relocate saves unless that behavior is explicitly documented. `--no-save` should disable all persistent application-state writes, including autosaves, progress, and history, and should be testable as a single contract.

### Suggested common option set

Treat these as candidates, not mandatory flags for every application:

- `-h, --help`, `-v, --version`
- `-f, --fullscreen`, `-w, --windowed`
- `--data-dir PATH`, `--user-data-dir PATH`, `--config-dir PATH`, `--state-dir PATH`, `--cache-dir PATH`, `--temp-dir PATH` (expose only the useful subset)
- `--renderer auto|...` only when multiple renderers are supported
- `--vsync on|off` and `--fps N` only when meaningful for the renderer
- `--mute`, `--volume VALUE` when audio is supported
- `--no-save` or an equivalent no-persistence mode for safe test/demo runs
- `--show-paths` or `--diagnostics` for troubleshooting
- `--portable` only for builds that implement a documented portable profile

Do not expose every menu preference as a CLI option by default. Add flags when they help automation, debugging, accessibility, deployment, or reproducible testing.

## 12. Configuration, Persistent State, and Diagnostics
- Keep user configuration, application state (saved documents, progress, scores, or equivalent durable user state), cache, and immutable application assets conceptually separate even if a target platform forces them into one physical directory.
- Define what happens when configuration is missing, malformed, from a newer version, or contains unknown keys. Prefer safe defaults, a useful warning, and preservation or backup of the unreadable file over silently overwriting user data.
- Add configuration migrations when key names or meanings change. Avoid changing a setting's meaning while retaining the same key. Ignore unknown keys where feasible so newer configs can be opened by older builds without destroying data.
- Make directory-creation and write failures actionable: report the affected path and whether the application can continue without persistence. Never claim a save succeeded if the write failed.
- Keep logs useful and bounded. Diagnostics should identify version/build, active backend when relevant, detected input devices where relevant, and resolved paths; avoid logging private data or full environment dumps by default.
- Save critical state atomically: write a temporary file in the destination directory, flush/close it, then replace the previous file. Where the platform/filesystem provides the needed primitives, ensure data and directory metadata are durably flushed. Handle interrupted or failed writes without truncating the last valid save; document any weaker guarantees.
- Distinguish durable user data from transient session/UI state. Persist important durable state incrementally; define what is restored after suspend/resume, activity recreation, crash, or process restart. Lifecycle callbacks are opportunities to flush or release resources, not the only time data may be saved. Test interrupted writes, full storage, permission failures, and recovery from invalid or partially written files where persistence exists.

## 13. Extensions, Plugins, and User-Installed Content
Plugins, extensions, themes, user scripts, custom content, and other add-ons are user-managed content, not shipped assets and not configuration. Keep them in the user-data category unless a specific platform requires a different physical layout. Upgrades must not silently overwrite user-created content; uninstall must not delete user data without clear, informed consent. Declare whether downgrades are supported and what happens when newer data cannot be read by an older version.

- Define the supported add-on types and whether each is data-only or executable. Prefer declarative data formats. Do not treat arbitrary scripts, native libraries, or downloaded binaries as harmless content.
- Keep built-in content, user-installed content, and temporary workshop/download staging separate. Shipped add-ons belong to the package; user-installed add-ons belong in the user-data directory; caches and incomplete downloads belong in cache/temp directories.
- Define deterministic discovery and precedence. A reasonable default is explicit command-line selection, enabled user extensions, then built-in content, but the application must document the actual order. Do not let filesystem enumeration order determine load order; use a stable sort or explicit priority/dependency metadata.
- Provide a manifest with a stable identifier, display name, version, supported application/API version, content type, and declared dependencies where applicable. Validate identifiers and paths. Detect duplicate IDs, missing dependencies, incompatible versions, and dependency cycles with actionable errors.
- Make enable/disable and load order discoverable in the UI or documented config. If load order affects compatibility, show it and make it reproducible. Do not silently overwrite one add-on with another sharing the same ID.
- Do not execute extension code by default. If executable extensions are supported, document the trust boundary and permissions; do not imply that a sandbox exists unless one is actually enforced. Warn before loading untrusted code and avoid automatic execution of newly downloaded content.
- Downloads and extraction must defend against path traversal, absolute paths, symlinks escaping the destination, archive bombs, and partial/corrupt packages. Extract into a staging directory, validate, then install atomically. Do not overwrite unrelated user files.
- Store add-on settings separately from global preferences where practical. Uninstalling an add-on must not delete saves or user-created content without an explicit confirmation. Define what happens to a save that references a missing or disabled add-on.
- Provide diagnostics that list add-on IDs, versions, enabled state, load order, and compatibility errors without dumping private paths or user content. A safe mode should be able to start without third-party extensions to recover from a broken installation.
- Test with no add-ons, one valid add-on, duplicate IDs, missing dependencies, incompatible versions, malformed manifests, and an add-on that fails to load. Document whether updates preserve user settings and whether old add-on versions are backed up.

## 14. Nix-First CI and Deployment
The Nix flake is the single source of truth for build, test, lint, package, cross-compilation, artifact assembly, and (where applicable) deployment logic. CI must invoke the flake rather than reimplementing those steps in workflow YAML or separate shell scripts that drift from local builds.

- Put buildable packages in `packages`, automated checks in `checks`, and reusable development tools in `devShells` as appropriate. Checks should encode the commands and dependencies they need as derivations so they run in a clean, reproducible environment.
- Make the intended CI entry point explicit, for example `nix flake check` or a named aggregate check. `nix flake check` validates the flake's standard output shapes and builds derivations under `checks` for the selected system(s); it does not automatically build every derivation in `packages`. Include required package builds as checks/dependencies or explicitly select them in the documented CI/release entry point. `nix flake check --no-build` checks evaluation only; it is not a test run. `--all-systems` broadens system coverage but does not magically provide target hardware or make foreign binaries runnable.
- Populate `checks.<system>` with the tests that should run in a clean build environment, as applicable: Nix formatting/linting and evaluation; native compile; unit and integration tests; package build and installed-artifact smoke tests; CLI/version/help checks; output-layout and required-asset checks; cross-artifact architecture/ABI/dependency inspection; and static-site link/path/asset checks. Keep each check focused and give it the tools and inputs it needs explicitly.
- Ensure the check set tests the real deliverables. A successful source build does not prove the packaged artifact runs, a WASM build does not prove the hosted page loads, and a cross-compiled binary does not prove compatibility with its target runtime. Add browser/device execution only where runners exist; otherwise record artifact-level checks and outstanding manual verification separately.
- GitHub Actions is a thin shim: provision the required Nix version/features, configure caching if desired, then invoke the documented flake entry point. Keep platform matrices and target selection minimal and declarative; do not duplicate compiler flags, dependency lists, build commands, or test logic in YAML.
- For cross-compilation, expose target packages/checks through the flake. The aggregate release output should depend on every required artifact; CI should build those outputs from the flake, and only run target executables when a suitable runner or explicitly configured emulator exists. Otherwise validate artifacts and record the manual test requirement.
- Keep secrets and signing credentials in the CI service, not in the flake. The flake describes reproducible build logic; publishing credentials and release permissions remain workflow concerns.
- Document the same local commands that CI uses. A developer should be able to reproduce a failed CI check with `nix build` / `nix flake check` without reverse-engineering workflow YAML.
- Avoid a second authoritative build script. Small GitHub workflow steps that install/configure Nix, select a flake output, or upload an already-built artifact are acceptable shims; build and test behavior belongs in the flake.

## 15. Validation
Use `PACKAGING-CHECKLIST.md` for the sole release gate and artifact matrix. A deliverable is not complete merely because it compiled: install it, run it outside the source/build tree where possible, verify assets and writable paths, and check version/diagnostics on the target runtime.
