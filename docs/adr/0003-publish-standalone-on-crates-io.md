# Publish on crates.io as a standalone tool rather than contribute upstream

The gap is real: the cargo team's own comment on #5657 is that `direct-minimal-versions`
"fits minimal-version testing better but is harder to use", and no stabilisation PR
exists. The upstream tool predates the stable `resolver.lockfile-path` field and is
built around editing files in place. The two deltas, alternate lockfile and
bootstrap-on-stable, were judged unlikely to be accepted upstream together, and the
author has ten private Rust repos that need the gate now.

## Considered options

- **Copy the Makefile target into every repo.** Fine for one target; the real drift is
  the whole Makefile, which is a separate decision (parked in the chain-pipeline repo).
- **Propose a stable CLI shape on cargo#5657.** Highest leverage, slowest. Not exclusive
  with this decision.
- **PR copy-mode into `cargo-minimal-versions`.** Would make the name available by
  becoming that tool; the bootstrap half would likely be refused, and the author
  wanted the no-nightly path.
- **A private subcommand on the lynx registry.** Installing would need the same
  credentials that running already needs, and it would carry one target of ten.

## Consequences

- `MIT OR Apache-2.0`, a public issue tracker on GitHub, a changelog, semver.
- The name `cargo-minimum-versions` was chosen over `cargo-lowest` for discoverability:
  people search for "minimal versions". The singular `minimum-version` was rejected
  because it reads as MSRV. `cargo-minver` is taken. Both `cargo-minimum-versions` and
  `cargo-lowest` were confirmed free on crates.io on 2026-10-19.
