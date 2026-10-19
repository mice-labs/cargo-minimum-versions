# cargo-minimum-versions

A cargo subcommand that builds or tests a workspace with every direct dependency at
the lowest version its requirement allows, so an understated floor fails in CI rather
than in a downstream lockfile.

## Language

**Floor**:
The lowest version a dependency requirement in `Cargo.toml` admits (`"1"` → 1.0.0).
It may be yanked or never published.
_Avoid_: minimum bound, lower bound

**Minimum version**:
The version the resolver picks for a direct dependency when every floor is honoured.
Never below the floor; may be raised by another member's or a dev-dependency's
requirement.
_Avoid_: minimal version (use only when naming cargo's `-Z minimal-versions` flags or
the `cargo-minimal-versions` tool), lowest version

**Understated floor**:
A floor below the lowest version that has the API the crate actually uses. The defect
this tool exists to catch.

**Direct dependency**:
A crate named in a workspace member's `[dependencies]`, `[dev-dependencies]`, or
`[build-dependencies]`, including optional and target-specific ones.

**Transitive dependency**:
A crate in the dependency graph that no workspace member names.

**Minimum lockfile**:
The lockfile resolved with every direct dependency at its minimum version. The user's
own `Cargo.lock` is never this file.
_Avoid_: minimal lockfile, alternate lockfile (how it is produced, not what it is)
