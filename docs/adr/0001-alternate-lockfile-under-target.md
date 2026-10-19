# Minimum versions resolve into an alternate lockfile under `target/`

The tool never copies the workspace and never edits the user's `Cargo.toml` or
`Cargo.lock`. It resolves into `target/minimum-versions/Cargo.lock` through the
`resolver.lockfile-path` config field, stable since cargo 1.97 (2026-07-09), and runs
the user's subcommand `--locked` against that file with `target/minimum-versions/build`
as the build directory. Verified on 2026-10-19 with cargo 1.99 against a real
workspace: the minimum lockfile landed under `target/`, a locked compile succeeded
against it, and the user's lockfile was byte-identical throughout.

## Considered options

- **Copy the workspace under `target/`** (the chain-pipeline Makefile). Lockfile safe,
  but breaks on `path` dependencies outside the workspace and on `include_str!`
  reaching above it, requires git to know what to copy, and fails on an index entry
  deleted from the worktree.
- **Swap the manifest and lockfile in place and restore afterwards**
  (`cargo-minimal-versions`). Works with any layout, but a kill or crash mid-run leaves
  the user's files modified, and an IDE running `cargo check` in the same tree sees
  the swapped lockfile. That tool grew signal handling for exactly this.
- **Alternate lockfile** (chosen). Any layout, no git, nothing of the user's written,
  build cache survives between runs. Costs a cargo 1.97 floor.

## Consequences

- Cargo older than 1.97 silently ignores the config field and would rewrite the user's
  lockfile. The tool checks `cargo --version` first and refuses to run below 1.97.
- No TOML crate, no `semver` crate: the tool reads manifests through `cargo metadata`
  and never writes one.
- A separate build directory, not the user's `target/debug`. Cargo keys artifacts by
  dependency version so sharing would not corrupt anything, but the workspace's own
  crates are re-fingerprinted whenever their dependency set changes, and alternating
  the gate with normal builds in one directory would rebuild the workspace on every
  switch. Precedent: `cargo-llvm-cov` uses `target/llvm-cov-target`.
- The one cargo call that reads the user's lockfile (the "current" column of the
  version table) runs `--locked`, because a plain `cargo metadata` creates or updates
  `Cargo.lock` when it is missing or stale. A missing or stale lockfile is therefore a
  tool error, not something the tool repairs.
