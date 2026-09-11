# 2026-09-11 12:00 — Test 0: the First Real Code

## What got built

`sync-mesh-core`'s first real code, merged via
[PR #6](https://github.com/sync-dot-mesh/core/pull/6), verified green
on actual GitHub Actions before merging, not just locally. Workspace
restructured into four crates applying Hexagonal Architecture (ports
and adapters) — the pattern chosen specifically because the stated
goal was expandable, interchangeable infrastructure, which is exactly
Hexagonal's own vocabulary:

- **`engine`** — pure domain and ports. Zero I/O, no gRPC, no
  filesystem access, enforced by what it's allowed to depend on, not
  just by convention.
- **`proto`** — the wire contract (`sync.proto` + generated bindings)
  as its own crate, owned by neither the server nor any client.
- **`daemon`** — the adapters (gRPC service, `SystemClock`, `DataDir`
  locking) and the composition root (`main.rs`) that wires them to
  engine's ports. The only place allowed to construct a concrete
  adapter.
- **`testkit`** — a `ClusterBackend` port with exactly one real
  implementation (`process`, Tier 0) — `docker`/`remote`/`android`
  slot in later as new implementations of the same trait, per the
  integration-testing backlog's pluggable-backend design.

## Illegal states unrepresentable, applied for real

Three concrete examples, not just the principle stated abstractly:

- **`NodeId(Uuid)`** — a newtype specifically so a future `FolderId`
  (also a `Uuid` underneath) can never be passed where a `NodeId` was
  expected. The mistake becomes a compile error, not a runtime bug
  discovered when a node is treated as a folder.
- **`HealthState`** — an enum (`Starting` / `Healthy` / `ShuttingDown`),
  not a `bool`. A bare `is_healthy: bool` can only say yes or no; it
  cannot distinguish "still starting, ask again" from "actually
  broken," and a second field invented to cover that gap can disagree
  with the first. The enum makes only the real states representable.
- **`DataDir`** — cannot be constructed except via `DataDir::acquire`,
  which only returns `Ok` once the exclusive OS-level lock is actually
  held. A `DataDir` value is proof the lock is held, not a path that
  merely should have one. This is what makes "two instances against
  the same directory" fail cleanly rather than racing.

## The Clock port

A small, deliberately narrow seam: `engine` never calls
`Instant::now()` directly, only through a `Clock` trait. `daemon`
provides the real `SystemClock`. Justified now, even though nothing
yet needs a fake clock in a test, because retrofitting this after
several call sites already call `Instant::now()` directly is real
work; adding the trait now costs almost nothing. Same pattern will
apply to the next port (a transport abstraction, a change-detection
source) once there's a second real implementation to justify it —
deliberately not built speculatively ahead of that.

## Real bugs found only by actually building and running this

**Orphan rule violation.** First attempt implemented the
proto-generated `SyncService` trait for `Arc<StatusService>` directly.
Illegal — `Arc` isn't local to this crate, and Rust's orphan rule
requires at least the outermost type or the trait to be local. Fixed
by making `StatusService` itself internally `Arc`-backed and cheaply
`Clone` (a standard Rust handle pattern) instead of wrapping it
externally — cleaner than the original attempt, not just a workaround.

**A bug in the test, not the product.** The negative lifecycle test
(two instances, same data directory) initially failed — but the
daemon's own stderr showed `AlreadyLocked` firing correctly in the
second process. The test itself was wrong: both instances shared a
data directory, so they'd also share the handshake-file path, and the
test's reachability check found the *first* instance's already-written
handshake file and mistook it for the second instance succeeding.
Fixed by deleting the stale handshake before the second attempt. Worth
recording because "the test passed" and "the test is correct" are
different claims, and this is a concrete case where they came apart.

**Missing environment dependencies, found by actually compiling.**
`protoc` (for compiling `.proto` files) was absent both in this
sandbox and in the local Nix dev shell — neither had ever needed it
before this session, since no `.proto` file existed yet. Added to CI
(alongside the already-known `mold` requirement) and to `shell.nix`.
Verified on real GitHub Actions before merging, not assumed from local
success alone.

**A clippy lint firing on generated code, not ours.**
`clippy::result_large_err` flagged `tonic::Status` as a large error
type — inside tonic's own generated server trait, regenerated on every
build. Suppressed at the exact boundary (`#![allow(...)]` on the
`include_proto!` call site) rather than anywhere that could be mistaken
for accepting a real lint against code we wrote.

**A genuinely useful clippy catch.** `clippy::suspicious_open_options`
flagged the lock file's `OpenOptions` for not specifying truncate
behaviour explicitly. Real ambiguity, not a false positive — fixed
with an explicit `.truncate(false)` and a comment explaining the lock
file's content is irrelevant, only the OS-level lock matters.

**Stale nightly-only rustfmt settings.** `imports_granularity` and
`group_imports` in `rustfmt.toml` (inherited from the original
boilerplate) require nightly rustfmt and had been silently no-op-ing
with a warning on every single `cargo fmt` run since the boilerplate
was first created — CI and the pinned dev toolchain are both stable.
Removed rather than kept as decoration that did nothing.

**A regression caught before it shipped.** The original single-crate
`src/change_detection.rs` scaffold got deleted during the workspace
restructure without its content being migrated anywhere — caught by
reviewing `git status` before committing, not by a test. Recovered via
`git show HEAD:src/change_detection.rs` and re-homed as
`engine::domain::ChangeEvent`, with its doc comment updated to
reference the actual current FUSE-with-fallback decision instead of
the stale notify-crate-only description it had before.

**An inaccurate `BREAKING CHANGE` footer, caught before merging.**
The first draft of the commit message declared a breaking change for
the workspace restructure. Reconsidered before opening the PR: nothing
has ever depended on `sync-mesh-core` as a published dependency (v0.1.0,
pre-first-release) — there's no real consumer contract being broken,
and leaving the footer in would have pushed `release-plz` toward an
inappropriate major-version bump on what should be an ordinary first
release. Removed.

## Process note: the original three-PR plan collapsed into one

Earlier planning proposed three separate PRs — a pure workspace
restructure, then the daemon, then testkit + Test 0 — specifically so
each could be reviewed independently. In practice the pieces were
built together and were too interdependent to usefully split after the
fact (the restructure alone doesn't compile without the daemon and
testkit crates that were supposed to come after it). Shipped as one
PR instead. Worth doing differently next time: commit and open a PR
for each increment *before* starting the next one, not after building
several together and discovering they don't decompose cleanly in
hindsight.

## What Test 0 actually proved

Not "the code compiles" — three real, separate OS processes started,
each answered `GetStatus` over an actual gRPC connection with a real
assigned port, each shut down cleanly, and a second instance pointed
at an already-locked directory genuinely failed to become reachable.
Verified on GitHub's own runner before merging. This is the first
piece of the integration-testing backlog's Tier 0 that is no longer a
plan.
