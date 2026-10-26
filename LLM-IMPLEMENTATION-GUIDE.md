# LLM Implementation Guide

Use this guide when asking a coding LLM or agent to bring an existing project into compliance with `DESIGN.md`, `VERSION-HANDLING.md`, and the applicable parts of `GAME-DESIGN.md`. The goal is to improve the project's actual behavior, not merely to make its documentation look compliant.

## Operating rules

- Inspect the repository before editing. Read its `AGENTS.md`, `README`, build files, flake, platform launchers, tests, and relevant source code. Follow repository-specific instructions unless they conflict with a higher-priority safety or user requirement.
- Treat the declared support matrix and project purpose as facts to discover and confirm, not assumptions. Do not add a platform or feature solely because this contract lists it as an option.
- Distinguish **requirements** from **examples**. Apply MUST requirements to supported features; apply SHOULD recommendations unless there is a concrete reason not to; implement MAY features only when they fit the project. Record justified exceptions.
- Prefer small, coherent changes that fix root causes. Do not rewrite unrelated code, silently remove features, change public behavior unnecessarily, or introduce a new framework just to match an example.
- Preserve existing user data and compatibility. Before changing paths, config formats, save formats, command-line options, packaging layout, or add-on behavior, identify migration and rollback risks. Never delete user data as an automatic cleanup step.
- Do not claim a build, test, target device, deployment, or runtime behavior passed unless it was actually verified. Clearly separate automated results, manual checks, and untested assumptions.
- Do not add secrets, credentials, private data, or machine-specific paths to source, derivations, logs, or generated artifacts.

## Required workflow

### 1. Establish the baseline

1. Read the project instructions and relevant design documents.
2. Identify the language/build system, current flake outputs, CI entry points, supported platforms, artifact formats, install/runtime paths, persistent data, and existing tests.
3. Run lightweight status and discovery commands first. Do not overwrite local changes or reset the worktree.
4. Build a baseline where practical. Record existing failures separately from regressions introduced by the work.

### 2. Create a gap report before changing behavior

For each applicable checklist item, classify it as:

- **Pass**: implementation and evidence demonstrate compliance.
- **Fail**: observed behavior or missing implementation violates the requirement.
- **N/A**: genuinely inapplicable, with a short reason.
- **Unknown**: insufficient evidence. Do not convert unknown to pass or N/A merely to shorten the report.

For each failure or unknown, record the relevant file/symbol, observed behavior, desired behavior, impact, and a proposed verification method. Prioritize data loss/security risks, broken supported targets, release/deployment correctness, then usability and lower-risk cleanup. Do not modify the checklist to make a failure disappear.

### 3. Plan a bounded implementation

Before proposing a new framework or replacing a library, check the language/library guidance in `DESIGN.md` against the real support matrix. Verify SDL2/SDL3 availability for PortMaster, legacy SDL1.2 and OLPC XO-1/X11 constraints where relevant, the rationale for Qt versus GTK in desktop GUIs, and actual low-resolution/resizing behavior. Do not migrate solely for modernity; account for runtime availability, footprint, porting effort, and target tests.

Group related fixes into small stages, for example:

1. Build and CI reproducibility.
2. Runtime paths, configuration, and persistence safety.
3. Platform lifecycle, input, graphics, and packaging behavior. For input, verify the declared keyboard/mouse/touch/gamepad modes per target, then test each through essential workflows; do not treat device detection as proof of usable support. Keep semantic actions separate from device events and avoid broad input rewrites without evidence.
4. Extensions, diagnostics, accessibility, and documentation.

Adapt the order to the project's actual risks. Avoid broad rewrites when a focused fix is safer. If a requirement is ambiguous or conflicts with established project behavior, document the conflict and choose the least destructive compatible solution; ask the user when the decision is consequential and cannot be inferred safely.

### 4. Implement and verify

- Make the implementation and its tests/docs agree. Prefer a regression test for every bug that can reasonably be tested automatically.
- Use the repository's documented Nix flake commands as the authoritative build/test/package path. Do not move build logic into CI YAML or a parallel shell script.
- Inspect what `nix flake check` actually covers. It builds declared `checks` for the selected system(s), but does not automatically build every `packages` output or prove target-device/browser behavior. Ensure required package builds are explicit in checks or the documented aggregate CI/release command; distinguish evaluation-only checks from builds and tests.
- Test the installed/package artifact from outside the source/build tree, not only the development executable. For WASM, inspect and smoke-test the actual static website output, including URL-prefix hosting and browser loading, rather than treating successful compilation as proof that the site works.
- Exercise failure paths that could corrupt data: interrupted writes, full/unwritable storage, invalid configuration, migration from an older version, and recovery after process termination where relevant.
- For cross-target behavior that cannot be executed locally, validate artifact metadata/architecture/dependencies and explicitly leave device/runtime checks unverified.
- Re-run relevant tests after the final code change. Report the exact commands and their outcomes.

### 5. Review the final change

Check for accidental API/CLI changes, duplicate path resolution logic, duplicated build definitions, data deletion, hard-coded versions, platform assumptions, stale docs, secrets, and unrelated churn. Review the diff and run whitespace/static checks. Keep changes minimal enough that a maintainer can understand and review them.

### 6. Report the result

Provide a concise handoff with:

- What changed and why, grouped by behavior rather than by file alone.
- Tests/builds actually run, with exact commands and pass/fail results.
- Supported targets verified, targets only artifact-checked, and targets not tested.
- Checklist statuses: Pass, Fail, N/A with reason, or Unknown.
- Remaining risks, compatibility/migration notes, and suggested next steps.
- Changed files and the resulting commit/revision when applicable.

Do not describe the project as fully compliant if any required item is Fail or Unknown. State the remaining blockers plainly.

## Suggested task prompt

> Inspect this repository and apply `DESIGN.md`, `VERSION-HANDLING.md`, and the applicable sections of `GAME-DESIGN.md`. Follow `LLM-IMPLEMENTATION-GUIDE.md`. First establish the current support matrix and write a concise evidence-backed gap report against `PACKAGING-CHECKLIST.md`. Then implement the highest-priority applicable fixes in small, reviewable stages. Preserve user data and existing behavior unless a change is required and its migration is handled. Use the Nix flake as the authoritative build/test/package workflow. Run the relevant checks, update documentation, and report exact verification results. Do not claim tests or targets passed unless actually tested; mark unverified items Unknown and explain why.
