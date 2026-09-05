---
name: tuffite-development
description: Develop and validate tuffite-in-chromium in its nested gclient workspace, including local Framework builds, dependency rolls, patch stacks, and Distribution packaging. Use for Framework work rather than application-only Tuffite development.
---

# Tuffite Framework development

The outer workspace owns `.gclient`, Mise tools, pinned Depot Tools, and the
independent `www` website submodule. The
Framework is a separate Git repository at `src/tuffite`; Chromium is its parent
repository at `src`. Inspect each repository's status separately and preserve
unrelated work. Read its AGENTS.md for Chromium patch boundary rules.

Run Framework x.py and Cargo spo commands with `src/tuffite` as the working directory and the
outer Mise environment active. `.gclient` pins the Framework, Framework DEPS
is the sole Chromium URL/revision declaration. spo config uses Git `base`/`base_ref`;
`refs/spo/upstream` records the applied baseline, while `x.py roll` reads the desired
baseline from DEPS. First checkout uses outer `mise run setup`, which runs `gclient sync --no-history` to synchronize the pinned repositories; the Framework CLI does not bootstrap its own Rust.

- `python3 x.py roll`: reconcile current Framework DEPS and patches using gclient,
  preserving the Framework branch. There is no `x.py sync` command. Run after branch
  or dependency changes, not on every build.
- `python3 x.py build --run`: build and run Playground against local Framework sources.
  The task supplies a local Distribution and disables automatic download fallback.
- `cargo spo`: status and patch actions. Explicit commands are preferable for agents.

spo menus support `/` to filter, arrow navigation, and Enter to accept. Esc clears
an active filter; Esc outside filtering or Ctrl-C cancels.
File selection supports Space for multiple files. Outer Mise runs framework/dev/tuff tasks directly in `src/tuffite` and www tasks in `www`.
x.py uses only Python standard-library modules under scripts/ (native build/package code is scripts/framework.py); no venv
or pip packages. No arguments print help; required arguments must be explicit.
The Python gclient hook builds/runs spo for patch operations; there is no patch.py.
Default Ninja jobs are max(1, floor(2 * logical CPU cores / 3)); -j overrides.
Private base refs and associations are rebuilt from DEPS and committed patches on
a new device; they are not cloneable metadata.

Read [the development guide](../../../src/tuffite/docs/development.md) for the
specific operation's flags, recovery rules, target configuration, and CI examples.
Read command `--help` before constructing unfamiliar combinations.

## Editing and saving

Commit Framework sources here directly. Chromium-owned modifications become
commits in the parent repository and are exported into `patches/chromium` here.
Never include `tuffite/` files in a Chromium patch commit. `spo new` and `refresh`
require deliberate file selection; `spo export` reads HEAD and never commits WIP.

Use `cargo spo export --all --verify-apply` to stage and verify exported mailboxes;
review the Framework diff. Use `cargo spo check --all --strict` for canonical
output and tree replay. `.patches` is the committed ordered file list; each file
may contain several consecutive commits. Grouping does not squash commits.
There are no Spo-Id trailers and no StGit runtime dependency.

`spo edit` uses action/commit/filename rows with explicit drop, contiguous groups,
and Git-backed rebase recovery. If external history edits invalidate local
associations, reassociate deliberately; do not guess from commit subjects.

## Recovery and validation

Inspect pending operations before rolling or rewriting history. Continue or abort
with `cargo spo continue <target>` / `abort <target>` or
`python3 x.py roll --continue` / `--abort`; never delete journals or hard-reset past
conflicts. WIP restoration is a separate phase. Only use `--wip-restored` after
resolving and inspecting it. The tool retains exact stash objects until restoration.
Roll abort restores target stacks and declarations; other gclient dependencies
may need a later sync.

Use `--dry-run` to inspect plans and `--non-interactive` for automated runs. Specify
profile and CPU in CI. `python3 x.py test --suite framework` covers Framework checks;
`--suite native` and `--suite smoke` validate native and packaged integration
behavior respectively. Choose verification appropriate to the changed layer and
report what actually ran.

`python3 x.py package` packages only the Framework Distribution; there is no
`--artifact` or application `--format` flag. Package Playground applications with
`cargo run --package tuffite --bin tuff -- package --manifest playground/tuffite.json`. Upload is an
external publishing action: run it only when the user's task authorizes upload.
Distribution APIs live in `tuffite::build::framework` behind the build feature.
There is no xtask crate; SDK build helpers must not depend on spo.

Applications prefer compatible local release output through `TUFFITE_FRAMEWORK_DIR`,
then cache/download. Store machine paths in ignored `mise.local.toml`. Use the
existing `TUFFITE_FRAMEWORK_DIST` there for an exact artifact or another profile;
explicit paths never fall back. No separate source-selection flag is needed.

Application build scripts compose `Builder::new().add(framework::Resolver).add(shell_api::Generator)`.
`Generator::new()` defaults to `CARGO_MANIFEST_DIR/tuffite.json`; use `.manifest(path)`
for an explicit input. `.rust(directory, mode)` emits `bindings.rs` and fragments;
`.javascript(directory, mode)` emits JavaScript and TypeScript declarations together.
Application commands (`check`, `build`, `doctor`, `dev`, and `package`) discover
the nearest `tuffite.json` from cwd through its parents; `--manifest` overrides
discovery. Default package output belongs to the discovered project `dist/`.
Neither source nor packaged application `tuffite.json` has `schemaVersion`. Optional top-level `productName` defaults to the Cargo
package name and controls bundle/installer display names; `bundle.executable`
and application identity remain separate settings.

Source code-generation settings belong in `build.shellApi`; packaging emits runtime
`app.shellApi` namespace/command IDs without declaration paths. Windows installer
packaging provisions pinned NSIS automatically through `tuff`, without Mise or
a global `makensis.exe`. Cache `%LOCALAPPDATA%/Tuffite/tools` in CI;
`TUFFITE_TOOLS_CACHE` overrides it, `TUFFITE_TOOLS_OFFLINE=true` forbids downloads,
and `TUFFITE_NSIS_DIR` selects a complete explicit installation without fallback.
