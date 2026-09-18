# 2026-09-11 — Naming atpret, Language Choice, and the argenv Dependency

## The ask

Fork `flowseal/zapret-discord-youtube` (a curated set of `.bat` files
wrapping `bol-van/zapret`'s `winws.exe`) into a new `sync-dot-mesh`
project: fully automatic, cross-platform (Windows, macOS, OpenWrt),
strategies described as a typed model rather than scripts or a DSL,
automatic strategy testing and updates from GitHub, import/export of
user config, tray icon, UI later.

## What the source project actually is

Confirmed by reading both repos directly: `zapret-discord-youtube`
contains zero original DPI-bypass logic — ~20 `.bat` files, each a
curated flag combination for `winws.exe`. The real engine is
`bol-van/zapret`, and critically: that v1 engine is explicitly EOL
upstream (Russian-language notice: no longer developed, bugfixes
only, feature PRs not accepted). The maintained successor is
`bol-van/zapret2` — decided to target that, not the frozen v1.
OpenWrt is upstream's stated primary target; macOS support is
explicitly "partial" — set expectations accordingly rather than
promising equal support across all three platforms.

## Two decisions, in sequence, worth recording precisely

**First proposal**: wrap the real `nfqws`/`winws` binaries rather than
reimplement the DPI-desync engine — lower risk, ships against a
proven engine immediately, typed orchestration layer on top
(strategy ADT with illegal-phase-ordering and incompatible-mode-pair
prevention at construction, matching the "illegal states
unrepresentable" discipline already established in `sync-mesh-core`).

**Overridden**: full reimplementation of the engine itself in Rust,
inside `atpret`, from day one — explicit reasoning given: Rust is
"far superior and secure," the original devs "just don't know it."
Flagged the real risk once, plainly, before proceeding regardless:
zapret2's parameter surface encodes years of adversarial,
empirically-discovered knowledge against real deployed DPI (MediaTek
NIC driver behavior, kernel-version-specific IPv6 defrag differences,
conntrack timeout tuning, byte-precise TLS/QUIC ClientHello handling)
that isn't written down as a spec anywhere — a from-scratch rewrite
risks landing on *silently non-functional*, not just buggy, since
there's no easy way to tell the difference without the same
adversarial access the original author has. Proceeding anyway, per
explicit instruction, sequenced riskiest-assumption-first: Linux/
OpenWrt engine validated against real blocked targets before Windows
or macOS are touched at all.

## Language: Rust, not C#/F# — the actual reasoning

OpenWrt targets (often MIPS, severely resource-constrained routers)
rule out .NET as a practical matter, not just a preference — no
realistic story for even NativeAOT .NET on that hardware. F#'s
discriminated unions were the right instinct for the strategy-model
*shape*; Rust's `enum` gives the identical capability, already proven
in `sync-mesh-core` (`HealthState`, `ChangeEvent`). Tooling reuse is
concrete, not just convenient: `cliff.toml` and `pr-title.yml` copy
over close to verbatim; `release-plz` is Cargo-native and would not
transfer to a `.csproj`/NuGet setup at all.

## Naming: atpret

User's own coinage — reads as "automated" + "zapret," and as
wordplay in Russian ("open up," with the deliberate a-for-o
substitution). Adopted as-is.

## The argenv correction

Initially assumed a new `argenv` repo would need creating from a
provided spec doc under `sync-dot-mesh`. Corrected: a real,
substantially-built `argenv` already exists under its own separate
org, `argenv-opencommons` — ~3,365 lines, real tests, three working
CI jobs already, `MIT OR Apache-2.0` (correct for a crate meant for
external consumption, deliberately different from `atpret`'s own
AGPLv3). Sequencing decided explicitly: `argenv` gets polished and
published to crates.io *first*, so `atpret` pulls it in as an
ordinary `cargo add argenv` dependency rather than a path dependency
to a sibling repo — real usage inside `atpret` afterward is expected
to surface design friction a unit-test suite alone wouldn't catch.

## Status as of this entry

`atpret` repo not yet created — explicitly deferred until `argenv`'s
polish pass (CI, release automation, PR-title enforcement) is
complete and merged. See the next entry in this folder for that
pass's outcome.
