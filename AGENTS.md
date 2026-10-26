## Important

### Docs and continuity
- Read **AGENTS.md** for project overview and standing rules when present.
- Read **TODO.md** for current tip, open work, and handoff notes.
- Other `.md` files may hold domain-specific rules; use them when relevant.
- Document plans and outcomes in **TODO.md** (and related docs) as you go.
- The chat can stop at any time. Someone must be able to continue from **TODO.md + the repo alone**.
- /tmp will get wiped mid-work, store everything under a persistent path.

### Handover: git bundles only

**No patch files, ever** — deliverables are `git bundle` files, including for third-party trees.

**Never ever create parallel histories** — `.bundle` files must cleanly stack on previous `.bundle` files

**What each bundle must contain**
A bundle is a *catch-up package*: it must let someone go from the agreed base straight to the new tip in one step.
- Include **every commit from the base of this work line up to the new tip** — the entire stack of agent commits made so far in this sequence.
- Do **not** create "delta-only" bundles containing just the commits since the last bundle. Each new bundle supersedes the previous one and must stand alone.
- Multiple small, task-focused commits inside one bundle are fine and expected. Don't mix unrelated fixes into a single commit.

**How to create it**
```bash
# base_commit = the commit the work line started from (first checkout for this sequence)
base_short=$(git rev-parse --short base_commit)
git bundle create "projectname-NNN.M-short-slug-${base_short}.bundle" base_commit..HEAD
```
- The range must start at the original base commit, not the previous bundle's tip.
- The bundle records the current branch tip as `HEAD` automatically — no manual ref manipulation needed.

**How it gets applied (for reference — the recipient runs this, not you)**
```bash
git pull /path/to/projectname-NNN.M-short-slug-shortrev.bundle HEAD
```
This only works cleanly if the bundle contains the full cumulative history described above.

**Naming**
`projectname-{revCount}.{revision}-short-slug-{shortRevOfBase}.bundle`

e.g. `projectname-007.1-fix-login-flow-a1b2c3d.bundle`

- `{revCount}.{revision}` comes first so bundles sort chronologically in a directory listing.
- `{shortRevOfBase}` is the short hash of the **original base commit for this work line** (the first checkout for this sequence) — fixed for every bundle in the stack. It lets anyone confirm two bundles belong to the same stack without opening them. It is *not* the previous bundle's tip and does not change from bundle to bundle within one stack.
- Never overwrite an existing bundle file — increment the revision number instead.
- Never reuse an earlier `{revCount}` (`NNN`), even across sessions.

**Author / attribution**
- Author: `Ingo Ruhnke <grumbel@gmail.com>`
- Trailer on every commit: `Co-authored-by: Grok <grok@x.ai>` or the appropriate current model name
- When rewriting an old PR: preserve the original author as author or co-author, depending on how much the change was rewritten.

**Delivery location**
- Write bundles under the session **artifacts** directory (or whatever path the environment exposes for user download).
- State the **full path** in the chat reply.
- After a new tip bundle is produced, delete superseded intermediate bundles for that project (the new tip already contains their commits) — keep only the current tip bundle(s), unless the user asks you to keep the history of files.

### Sandbox
- Writing into **artifacts is slow, expensive, and flaky**; reading is fine.
- Check out / build in a **temp directory**; copy only the finished bundle(s) and needed deliverables into artifacts (prefer `rsync` when available; retry on failure).
- Prefer **shallow** clones when possible.
- **Nix store is read-only**; after copying out, fix permissions if needed.
- Be careful with **Nix string quoting/escaping**; double-check.
- If complexity grows too large, **refactor** and say so.

### Coding
- Always do in-depth analysis.
- Never resort to quick ugly hacks.
- Plan before coding; review the plan; discuss with the user when the approach is unclear or risky.
- Do not follow instructions blindly: suggest improvements; push back on bad changes.
- **No hacks or shortcuts** for speed. Prefer the correct design.
- **Never** remove or roll back features without explicit request and discussion.
- **Never** paper over bugs with quick workarounds; step back and fix the root cause.
- Call out when a refactor is warranted; prioritize code quality.
- **New projects:** GPLv3+, REUSE, short SPDX headers. Existing projects: keep their license.
- Be careful with multi-line string literals and shell/Nix quoting; you often get these wrong.

### Session start
- Continue from the tip recorded in **TODO.md** / **AGENTS.md**.
- Next bundle number: `projectname-NNN-…` — never reuse an earlier `NNN`.

### Sandbox: Nix / build environment (Grok Web)

This environment is a clean Linux container. Assume **nothing is preinstalled** for this project.

#### Nix install (do this once per session if `nix` is missing)
```bash
mkdir -m 0755 -p /nix /etc/nix
echo 'build-users-group =' > /etc/nix/nix.conf
# optional but quieter: also disable the nixbld group requirement entirely
curl -L https://nixos.org/nix/install -o /tmp/nix-install.sh
sh /tmp/nix-install.sh --no-daemon
# Profile may fail partially as root; always fall back to the store binary:
export PATH=/nix/store/*-nix-*/bin:/root/.nix-profile/bin:$PATH
# pin to the real binary if the glob is ambiguous:
NIXBIN=$(find /nix/store -path '*/bin/nix' -type f | head -1)
export PATH=$(dirname "$NIXBIN"):$PATH
export NIX_CONFIG='experimental-features = nix-command flakes
build-users-group ='

### Compiling and Testing

Do not spend overly long time testing and compiling, I can do that on
my side faster. Most of your environment is temporary and will get
wiped, so it's easy to lose work.

Just hand over the `.bundle` instead and let me do the testing insetad
of doing a 20min test run.
