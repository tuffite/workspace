# Tuffite development guidance

## Chromium patch boundaries

- Keep Tuffite-owned implementation under `src/tuffite/` whenever possible.
- In Chromium-owned files, guard Tuffite additions with
  `tuffite_in_chromium` in GN or `BUILDFLAG(TUFFITE_IN_CHROMIUM)` in C/C++
  whenever the surrounding syntax permits it.
- If a Chromium-owned file must contain a Tuffite change that cannot be fully
  enclosed by that condition (for example, an unconditional GN import, build
  dependency, or JSON5 feature declaration), enclose the smallest practical
  patch region in matching `# TUFFITE_BEGIN` / `# TUFFITE_END` comments for GN
  and matching `// TUFFITE_BEGIN` / `// TUFFITE_END` comments for C/C++, JSON5,
  and other C-style sources.
- Do not add boundary markers inside `src/tuffite/`; those files are wholly
  Tuffite-owned.
- Keep each marked region narrow and balanced so Chromium rolls can locate,
  review, and reapply downstream changes mechanically.
- Do not list Tuffite source files or target labels directly in Chromium-owned
  build files. Define the lists in `tuffite_in_chromium.gni` beside the component
  integration sources and import that file directly from Chromium. Consume
  feature implementation variables inside `if (tuffite_in_chromium)`.
- Reusable component implementation and IDL sources belong under
  `src/tuffite/components/<feature>/`. Dedicated integration directories such as
  Explorer `browser/content` and `renderer/blink` retain their Chromium
  compilation targets. Scope private dependency exceptions to these directories
  and preserve generated binding paths. No mirrored forwarding `.gni` is needed.
- Prefix imported variables with `tuffite_` and include the owning feature when
  applicable, for example `tuffite_explorer_sources` or
  `tuffite_explorer_generated_interface_sources`. This keeps names concise while
  preventing collisions in GN's shared import scope.
- GN source strings imported from a `.gni` are resolved in the consuming
  target's directory, not implicitly relative to the `.gni`. Use complete
  source-absolute labels such as `//tuffite/.../file.cc` by default. A private
  directory-prefix variable is appropriate only for a long same-directory
  list where it materially reduces repetition. Do not use bare relative paths
  or `rebase_path()` for target source labels.
