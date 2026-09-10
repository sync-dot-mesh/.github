# 2026-09-10 22:30 — sync-mesh-core Bootstrap: CI, Dependabot, Weekly Triage

*Reconstructed sequential time.*

## What this session did

Created `sync-dot-mesh/core` — the actual repo for the Rust sync
engine (see the hybrid-architecture decision entry) — by "unwrapping"
the standalone `rust-boilerplate` template: cloned its current live
state (not an old local copy — pulled fresh to pick up manual fixes
already made to it), renamed the package to `sync-mesh-core`, reframed
the README from generic-template language to describe the actual
project, kept the entire Nix/direnv dev environment setup verbatim.

## CI

Three parallel jobs — `cargo test`, `cargo clippy -- -D warnings`,
`cargo fmt --check` — using `dtolnay/rust-toolchain` and
`Swatinem/rust-cache` rather than routing through Nix in CI. Decision:
Nix's value is local dev-environment consistency and the isolated IDE;
CI doesn't need either, just Cargo, and a lightweight toolchain action
is faster and avoids impure-nixpkgs-fetch flakiness in the CI context.

**Bug found on first real push**: `.cargo/config.toml` (inherited from
the boilerplate) pins the `mold` linker, present in the local Nix
shell but absent on GitHub's bare `ubuntu-latest` runner. `test`
failed at the link step; `clippy` and `fmt` passed because neither
invokes the linker (clippy runs a check-only pass, fmt never touches
rustc at all) — that asymmetry is what made the root cause legible.
Fixed by installing `mold` in CI rather than weakening the local
config, since the local config is correct for its actual environment.

## Dependabot

Cargo and github-actions ecosystems, weekly, minor/patch bumps grouped
into one PR to reduce review noise for a two-person team, security
updates never grouped. Confirmed working immediately — Dependabot's
initial scan opened a real PR (`actions/checkout` bump) within minutes
of the config landing, before the weekly schedule even applied.

## Weekly issue triage via gh-aw

Installed the real `gh` CLI and `github/gh-aw` extension rather than
hand-writing the compiled output, specifically so the real compiler
would catch mistakes — which it did, twice, on the first attempt:
`timeout_minutes` should be `timeout-minutes` (typo), and a fixed
Monday-09:00 cron should be a fuzzy `weekly on monday` schedule
(GitHub's own suggestion, to avoid every repo using this pattern
firing at the identical instant). The compiler also gated on a new
secret reference (`ANTHROPIC_API_KEY`, required by `engine: claude`)
before treating the compilation as trustworthy — reviewed and approved
since it was exactly the expected secret, nothing unexpected. Workflow
is compiled and correct but dormant until the secret is actually added
to the repo.

## What this connects to

Branch protection (next session) and the changelog/release pipeline
both build directly on this CI setup — the same three check names
(`cargo test`, `cargo clippy`, `cargo fmt --check`) become the required
status checks, and `release-plz`'s own workflow runs alongside these.
