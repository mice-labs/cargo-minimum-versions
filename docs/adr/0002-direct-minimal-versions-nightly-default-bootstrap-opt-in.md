# `-Z direct-minimal-versions` is the engine; nightly by default, `--bootstrap` opt-in

Cargo's `-Z direct-minimal-versions` does the resolving. It is unstable (tracking issue
rust-lang/cargo#5657, open since 2018, tagged needs-design, no backers), so cargo
accepts it only on a nightly build or on a stable build with `RUSTC_BOOTSTRAP=1`. The
tool requires nightly by default and sets the bootstrap variable, for the resolve step
only, when the user passes `--bootstrap`. The flag is a no-op on nightly. Choosing
nightly is cargo's own `+nightly` syntax, which rustup propagates to the subprocess;
the tool has no toolchain flag of its own.

## Considered options

- **Stable-only resolution via `cargo update -p <dep> --precise <floor>` per direct
  dependency.** Needs no unstable flag, but the floor of `tokio = "1"` is 1.0.0, which
  may be yanked or never published; cargo's resolver picks the lowest version that
  exists, and matching that means reading the registry index, which for a private
  registry means reimplementing cargo's credential handling. It also reimplements
  multiple-version graphs, `=`/`~`/range requirements, and renames, and becomes dead
  weight the day the flag stabilises.
- **Bootstrap by default, no nightly.** The original Makefile's choice. Rejected for a
  public tool: the default path should use only supported mechanisms, and the escape
  hatch should be named for what it is. The Rust project tolerates `RUSTC_BOOTSTRAP` in
  private scripts and dislikes it in distributed tooling.
- **A strict `--no-bootstrap` mode on top of bootstrap-by-default.** Dissolved: with
  nightly as the default, strict is the default.
- **Full `-Z minimal-versions` by default.** Cargo's own docs say it is not
  recommended because transitive floors across the ecosystem are wrong, and that
  direct-only is the intended future shape. Available as `--all`.

## Consequences

- On a stable or beta toolchain without `--bootstrap` the tool fails before resolving,
  naming both exits: `cargo +nightly minimum-versions ...` or `--bootstrap`.
- `--bootstrap` must set the variable for the resolve process only, never for the
  subcommand run, and the README states that cargo calls the variable unsupported.
- CI for the tool needs a nightly toolchain to exercise the default path.
- If the flag stabilises, the tool drops the nightly requirement and `--bootstrap`
  becomes a no-op with a deprecation note. Nothing else changes.
