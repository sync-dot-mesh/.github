# 2026-09-10 — Dev Environment: Nix and Isolated VSCodium Boilerplate

*Predates the org — dated by day only, reconstructed from conversation
order.*

## The ask

A reusable dev environment for Rust work: pure vanilla Nix (no flakes,
no `experimental-features`), an isolated VSCodium with a full Rust IDE
extension set baked in, callable from a project directory with a
simple command, `vscodium` itself shadowed by the local isolated
instance.

## First design, and why it was scrapped

Initial version split this into two repos: a standalone
`rust-devenv-vscodium` module (published independently, importable via
`builtins.fetchGit` + `import`) and a consuming project that fetched
it. Rationale at the time: one thing to update, reusable across
projects. Rejected on the next pass — nobody actually wants a
boilerplate's dev environment to keep updating itself after a project
is bootstrapped, and the cross-repo `fetchGit` indirection added real
complexity (pinning a commit `rev`, a second repo to maintain) for a
benefit that didn't matter in practice. Collapsed into one flat,
self-contained `rust-boilerplate` repo — clone, rename, done. `nix/`
subfolder structure kept for readability, but nothing imports across
repo boundaries.

## direnv as the actual interface

Settled on `.envrc` (`use nix`, committed directly since its content
never varies) plus a single `init_devenv.sh` script that runs
`direnv allow` (trust) and `direnv reload` (forces immediate
evaluation rather than waiting for the next `cd`, which matters when
re-running after editing `shell.nix`). After that, the environment
loads ambiently on every `cd` into the directory in any terminal —
`vscodium .` directly, no `nix-shell --run` wrapper needed. A separate
`./dev` launcher script from the first draft was dropped entirely once
direnv made it redundant.

## Two real bugs found only by actually running it

1. **Extensions silently empty.** `pkgs.vscode-with-extensions` already
   bakes its own `--extensions-dir` into the wrapped `codium` binary,
   pointing at an immutable Nix store path containing the curated set.
   The wrapper script was *also* passing its own `--extensions-dir`
   (an empty, freshly-created project-local folder) — VSCodium's CLI
   keeps the *last* of two conflicting flags, so it silently used the
   empty one. Fix: stop specifying `--extensions-dir` at all; let the
   Nix-baked one through. Bonus consequence: extensions are now fully
   declarative — trying to Install via the GUI fails because that path
   is read-only, which is correct, not a regression.
2. **Extension/editor version mismatch.** `jnoortheen.nix-ide` >=0.3.7
   requires VS Code engine >=1.96.0; the VSCodium build nixpkgs'
   `nixos-24.05` branch provides is 1.94.1. Had to query the
   marketplace API for the full version history with engine
   constraints and pin to 0.3.5 — the newest version actually
   compatible — rather than blindly taking latest. Left the exact
   query in a comment in `extensions.nix` for whenever the nixpkgs pin
   moves to something shipping a newer VSCodium.

## Marketplace extension hashing

For any extension not in nixpkgs' own curated `vscode-extensions` set,
pinning requires a `sha256`. Rather than iterating through
`lib.fakeHash` failures one at a time (Nix's standard but slow
workflow — each `nix-shell` run reports one correct hash, you paste it,
repeat), fetched the actual `.vsix` files directly and computed the
SRI hash locally, verifying the method against one hash Nix had
already reported to confirm correctness before trusting it for the
rest. Resolved three extensions (`direnv`, `crates`, `even-better-toml`)
in one pass this way.

## Nix-side IDE support added afterward

The `.nix` files are the actual backbone of the boilerplate, not
incidental — they got the same tier of support as the Rust code:
`nixd` (eval-based LSP, real completions against nixpkgs itself, not
just static analysis), `alejandra` (formatter, wired both to the
editor's Format Document and to `nixd`'s own formatting command),
`statix` and `deadnix` (linting, on PATH for manual runs — not wired
to inline diagnostics, since `nix-ide` doesn't surface external
linters automatically; noted honestly rather than overclaiming
integration that doesn't exist).

## Where this ended up

Published as the standalone `rust-boilerplate` repo, later cloned and
"unwrapped" (renamed, reframed) into `sync-dot-mesh/core` — see the
repo bootstrap entry for that step.
