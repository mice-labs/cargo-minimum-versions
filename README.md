# cargo-minimum-versions

Test your crate with every direct dependency at the lowest version its requirement
allows, so an understated floor fails in your CI instead of in a downstream lockfile.

Like [cargo-minimal-versions], but it never touches your `Cargo.toml` or `Cargo.lock`,
needs no `cargo-hack`, and runs on stable with `--bootstrap`.

## Install

    cargo install cargo-minimum-versions

Needs cargo 1.97 or newer and a current `Cargo.lock` in your workspace.

## Use

    cargo +nightly minimum-versions test --workspace
    cargo minimum-versions --bootstrap test --workspace    # on stable

Resolution uses cargo's unstable `-Z direct-minimal-versions`, so it needs a nightly
cargo. `--bootstrap` sets `RUSTC_BOOTSTRAP=1` for the resolve step so a stable cargo
accepts the flag. Cargo calls that variable unsupported; it is an escape hatch, and the
flag is named for what it does.

The resolved lockfile is written to `target/minimum-versions/Cargo.lock` and the build
goes to `target/minimum-versions/build`. Your own files are never modified. Before the
subcommand runs, the tool prints each direct dependency whose minimum version differs
from the one in your `Cargo.lock`, so a red run tells you which floor moved.
`--verbose` prints every direct dependency.

`--all` uses `-Z minimal-versions` instead, minimising transitive dependencies too.
Cargo's documentation does not recommend it; most of the ecosystem's transitive floors
are wrong.

## Notes

- Dev-dependencies count as direct and are minimised. A dev-dependency floor can raise
  the resolved version of a normal dependency.
- Choose a toolchain with cargo's own syntax: `cargo +nightly minimum-versions ...`.
- Everything after the subcommand goes to cargo unchanged, including `--
  --nocapture`. Any cargo subcommand works, not only build ones:
  `cargo minimum-versions tree` shows the minimum graph.
- A separate build directory is used so running the gate does not invalidate your
  normal build cache.
- If `Cargo.lock` is missing or out of date the tool stops rather than writing one;
  run `cargo generate-lockfile` first.

[cargo-minimal-versions]: https://github.com/taiki-e/cargo-minimal-versions
