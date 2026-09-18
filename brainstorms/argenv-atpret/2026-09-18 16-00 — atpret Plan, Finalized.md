# 2026-09-18 16:00 — atpret Plan, Finalized

With `argenv` published (`0.1.0` on crates.io) and fully polished
(CI, Dependabot, Pages, backlog — all documented in `argenv`'s own
`brainstorms/` and cross-referenced in
[`argenv-opencommons/.github`](https://github.com/argenv-opencommons/.github)),
the deferred `atpret` plan was reviewed and finalized. Answers to the
open questions left hanging when `argenv` polish was prioritized:

## 1. Architecture reverted to the original lower-risk proposal

The full-reimplement-the-engine-in-Rust decision (made 2026-09-11) is
**reversed for the starting point**. `atpret` starts as a typed
wrapper around the real `nfqws`/`winws` binaries — the original
first proposal, before it was overridden. Explicit reasoning given:
start lower-risk, ship against a proven engine immediately, put the
typed orchestration layer (the strategy ADT, illegal-phase-ordering
and incompatible-mode-pair prevention at construction) on top of
binaries that already work, rather than betting the whole project on
a from-scratch reimplementation of DPI-evasion logic that took years
of adversarial, undocumented tuning to get right upstream. A full
native-Rust engine reimplementation is not ruled out permanently —
just not the starting point.

## 2. Strategy-pack ingestion (auto-convert vs. native-only): deferred

Explicitly **not a pressing decision right now** — "we'll cross that
bridge when we come to it." Left open on purpose, not forgotten.

## 3. Docs/backlog repo: separate, same org, migration deferred

`atpret` gets its own repo for docs/backlog (matching the
`argenv-opencommons/.github` pattern just built), but **stays inside
`sync-dot-mesh`** rather than its own org for now — explicit
intent to migrate `atpret` to its own org later, "hopefully," not
guaranteed. Noted so a future session doesn't assume the current org
placement is permanent.

## 4. iOS support / private-key storage (core project, unrelated to atpret)

Confirmed explicitly as **not part of this project's scope at all** —
these are open items on the original Sync.Mesh (`core`) backlog,
temporarily sharing the same org purely for convenience, not because
`atpret` and `core` are related products. Both deferred, "we'll cross
that bridge later" — same as item 2, but flagged as an unrelated
project's open item rather than atpret's own.

## Everything else in the original plan stands, confirmed as "solid"

Linux/OpenWrt-first build order validated against real blocked
targets before Windows is touched; macOS explicitly best-effort;
hexagonal shape (engine/cli/daemon) with `cli` wired to `argenv`;
daemon → tray icon → Tauri UI last; AGPLv3 licensing. See the
[2026-09-11 entry](./2026-09-11%20—%20Naming%2C%20Language%2C%20and%20the%20argenv%20Dependency.md)
in this same folder for the full original reasoning behind each.
