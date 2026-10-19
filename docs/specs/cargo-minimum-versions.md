# cargo-minimum-versions: spec

Positioning in one line: like `cargo-minimal-versions`, but it never touches your
`Cargo.toml` or `Cargo.lock`, needs no `cargo-hack`, and runs on stable with
`--bootstrap`.

Vocabulary follows `GLOSSARY.md`: a *floor* is what a requirement admits, a *minimum
version* is what the resolver picks, and "minimal" appears only when naming cargo's
flags or taiki-e's tool.

## Name and shape

Crate and binary: `cargo-minimum-versions`. Invoked as a cargo subcommand:

```
cargo minimum-versions [OPTIONS] <CARGO_SUBCOMMAND> [ARGS]...
```

Options, all before the subcommand:

| Flag | Effect |
|---|---|
| `--bootstrap` | Set `RUSTC_BOOTSTRAP=1` on the resolve step. No-op on nightly. |
| `--all` | Use `-Z minimal-versions` (transitive) instead of `-Z direct-minimal-versions`. |
| `-v`, `--verbose` | Print every row of the version table, not only the rows that differ. |
| `-h`, `--help`, `-V`, `--version` | Clap defaults. |

There is no `--quiet`: the default output is one summary line plus the rows that
changed, which is empty when nothing did. Everything from the subcommand onward is
passed to cargo unchanged. The shape mirrors `cargo-minimal-versions` so users can
switch between the two without relearning. Toolchain selection is cargo's `+toolchain`
syntax; the tool has no flag for it.

## Child processes

Every cargo invocation uses the binary named by the `CARGO` environment variable,
which cargo sets when it runs an external subcommand, falling back to `cargo` on
`PATH` when unset (direct `cargo-minimum-versions` invocation). The `cargo_metadata`
crate already honours `CARGO`. This guarantees `cargo +nightly minimum-versions` checks
and drives the same binary.

The tool passes everything cargo needs as cargo-global flags (`--locked`, `--config`)
inserted immediately after the subcommand name, before the user's arguments. It never
appends after them, because a tail such as `test -- --nocapture` would hand anything
appended to the test binary. It never uses `--target-dir`, which `cargo tree`,
`cargo metadata`, and third-party subcommands reject. It sets no environment variable
on the subcommand run, so nothing leaks into build scripts or into cargo processes the
user's own tests spawn. `RUSTC_BOOTSTRAP` on the resolve step is the single exception.

## Steps

1. **Check cargo.** Run `$CARGO --version`. Below 1.97: fail, "needs cargo 1.97 or
   newer for resolver.lockfile-path". Record whether the version string contains
   `-nightly`; beta and `-dev` builds count as not nightly, which is right because
   they refuse `-Z` too.
2. **Check toolchain policy.** Not nightly and no `--bootstrap`: fail, naming both
   exits. Otherwise proceed.
3. **Locate the target directory.** `cargo metadata --no-deps` via the `cargo_metadata`
   crate; read `target_directory`. This honours `CARGO_TARGET_DIR` and a
   `build.target-dir` in any config file. The workspace root is not needed: cargo finds
   the manifest from the working directory as it does for the user. Set
   `LOCK = <target>/minimum-versions/Cargo.lock` and
   `BUILD = <target>/minimum-versions/build`, both absolute. Create the parent
   directory.
4. **Read the current graph.** `cargo metadata --locked --all-features` against the
   user's lockfile. On failure: exit 1, "Cargo.lock is missing or out of date; run
   `cargo generate-lockfile` and commit it". This runs before the resolve so the
   failure takes under a second instead of following an index update. `--locked` is
   what keeps the user's lockfile untouched: a plain `cargo metadata` creates or
   rewrites it.
5. **Resolve.** Delete `LOCK` if present (ignore not-found), so a failed resolve never
   leaves last run's file behind. Then
   `cargo update -Z direct-minimal-versions --config 'resolver.lockfile-path="<LOCK>"'`
   (or `-Z minimal-versions` with `--all`), with `RUSTC_BOOTSTRAP=1` in the child's
   environment whenever `--bootstrap` is set; it is a no-op on nightly. On failure:
   show cargo's stderr unchanged, exit with cargo's status. No lockfile exists in this
   case and the tool adds nothing.
6. **Read the minimum graph.** `cargo metadata --locked --all-features --config
   'resolver.lockfile-path="<LOCK>"'`.
7. **Print the version table.** From the two graphs, for each workspace member, for
   each direct dependency including dev, build, optional, and target-specific ones,
   take (crate, requirement, current version, minimum version), resolved through
   `resolve.nodes` so multiple-version graphs map correctly. Dedupe rows by (crate,
   requirement) across members; there is no member column, the crate name is enough
   to find the `Cargo.toml` line. Print to stderr, before the subcommand runs so it
   sits at the top of a CI log: one summary line,
   `42 direct dependencies, 7 resolve lower than your Cargo.lock`, then the rows whose
   current and minimum differ, aligned columns `crate  req  current  minimum`. With
   `--verbose`, every row. This is the evidence a user pastes into the commit that
   moves a floor.
8. **Run the subcommand.**
   `cargo <subcommand> --locked --config 'resolver.lockfile-path="<LOCK>"' --config
   'build.target-dir="<BUILD>"' <args>...`, with `CARGO_TARGET_DIR` and
   `CARGO_BUILD_TARGET_DIR` removed from the child's environment, because either
   would silently beat the config value (verified on 1.99); `BUILD` already honoured
   them in step 3. No bootstrap variable. Exit with cargo's status. On non-zero,
   print one line: `minimum lockfile: <LOCK>`.

## Exit codes

| Code | Meaning |
|---|---|
| 0 | Subcommand succeeded at minimum versions. |
| 1 | Tool error: cargo too old, stable without `--bootstrap`, not in a workspace, user's `Cargo.lock` missing or stale, metadata failed. Message carries the `anyhow` context chain. |
| 2 | Usage error, from clap. |
| cargo's | Resolution failed (cargo exits 101) or the subcommand failed. Passed through. |

## Dependencies of the tool

`clap` (derive), `cargo_metadata`, `anyhow`. `serde` and `serde_json` arrive through
`cargo_metadata` and are not direct. No `toml`, `toml_edit`, `semver`, or `cargo`
crates: the tool never parses or writes a manifest. Edition 2024; `rust-version =
"1.97"`, the same number as the runtime floor, so there is one number to explain.
Checked in CI with `+1.97`.

## Dev-dependencies

`direct-minimal-versions` minimises dev-dependencies because they are direct. A
dev-dependency floor can therefore raise a normal dependency's resolved version, and
the tool does nothing special about it. Documented in the README. This is what caught
a real understated floor in chain-pipeline PR #2.

## Things deliberately not in the first release

- A strict mode refusing `--bootstrap`. Strict is now the default.
- `--quiet`. The default output is already the diff; see "Name and shape".
- `--lockfile-out <path>`. The lockfile is at a known path under `target/`; `cp` works.
- `--format json` for the version table. Would make `serde_json` a direct dependency.
- Resolving the "current" column into `target/minimum-versions/current.lock` when the
  user has no committed `Cargo.lock`, instead of failing. The upgrade path if the hard
  failure bites someone; file an issue at first release.
- A non-git fallback. Nothing is copied, so there is no git dependency to fall back from.
- `--ignore-private` or any manifest filtering. The tool never edits manifests.
- Forwarding a `--manifest-path` from the pass-through tail into the metadata calls.
  Known gap; file an issue at first release.
- Prebuilt binaries. `cargo install` is the install path.
- Windows in CI. Nothing is OS-specific, but it is untested until someone reports.

## Tests

Integration tests over fixture workspaces in `tests/fixtures/`, hitting crates.io for
real with tiny crates, because resolution is the thing under test and a mocked registry
would test the mock. Every fixture test asserts the user's `Cargo.lock` is
byte-identical before and after, and the test harness scrubs `CARGO_TARGET_DIR` from
its own environment so the fixtures resolve their own `target/`.

- `passes/`: correct floors with at least one floor below the committed lockfile;
  `cargo minimum-versions check` exits 0, `target/minimum-versions/Cargo.lock` exists,
  stderr carries the summary line and exactly the differing rows; with `--verbose`,
  every direct dependency appears.
- `understated-floor/`: a direct dependency whose floor is below what a transitive
  dependency demands; expect cargo's resolver error, non-zero exit, no minimum lockfile.
- `api-too-new/`: floors resolve but the code uses an API added later; expect the
  subcommand to fail, the version table on stderr, and the `minimum lockfile:` line.
- `no-lockfile/`: a valid workspace with no `Cargo.lock`; expect exit 1 naming
  `cargo generate-lockfile`, and no `Cargo.lock` created.
- `not-a-workspace/`: run from a directory with no manifest; expect exit 1 with a
  clear message.
- Pass-through, on `passes/`: `cargo minimum-versions test -- --nocapture` reaches the
  test binary with `--nocapture` and nothing else; `cargo minimum-versions tree` exits
  0 and shows the minimum versions.
- `old-cargo`: unit test of the version check against captured `cargo --version`
  strings for 1.96, 1.97, a beta, and a nightly.
- Toolchain policy: unit test of the nightly detection and the `--bootstrap` decision
  table.

## CI (GitHub Actions)

- Linux and macOS.
- Nightly: the floating `nightly` channel, the default path, `cargo minimum-versions
  check` on the fixtures and on the tool itself (dogfood). A weekly `schedule`
  trigger so a nightly break surfaces without a push.
- Stable: the `--bootstrap` path on the same fixtures and on itself.
- `+1.97`: build and test, the `rust-version` floor.
- `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`.
- Release: push a tag `vX.Y.Z`, workflow runs `cargo publish`. Changelog in Keep a
  Changelog format.

## Licence and hosting

`MIT OR Apache-2.0`, two `LICENSE-*` files. Repository: `github.com/mice-labs/
cargo-minimum-versions` (remote already set). Cargo.toml keywords:
`minimal-versions`, `msrv`, `ci`, `cargo-subcommand`.

## Tickets

Implementation tickets are beads under epic bead `cargo-minimum-versions-bz2`
(Jira LAB-14): `bd children cargo-minimum-versions-bz2` lists them, `bd ready` is the
frontier. They are tracer-bullet slices, not the layers above: scaffold + pass-through
run, resolve into the alternate lockfile, CI skeleton, current-lockfile guard, version
table, version floor and toolchain policy, `--all`, failure paths, full CI matrix,
release 0.1.0, deferred issues.
