# 2026-09-11 01:00 — Integration Testing Strategy

## The goal

Real integration testing for a P2P sync application: 2 to N genuinely
independent instances, verified to discover each other, communicate,
exchange files, and (pending clarification — see Open Items) offload
computation. Must run identically on a contributor's own machine and
in CI, with zero tolerance for manual/by-hand verification — the
explicit reason being that hand-testing is tedious, error-prone, and
gives contributors nothing to build confidence against except whatever
the maintainer happens to remember. Automated tests are the actual
contract for expected behavior.

No real engine code exists yet. This session designs the test
architecture first, deliberately, since the shape of the test harness
influences how the engine itself needs to be structured (e.g., it
needs *some* externally callable status/health interface from day one).

## The four tiers

Genuinely different infrastructure, not one test type at increasing
scale:

```
Tier 0 — Single machine, plain OS processes
Tier 1 — Single machine, real containers (separate network namespaces)
Tier 2 — Real separate machines / VMs (only for things needing a real
          network boundary — WireGuard tunneling, NAT traversal,
          latency/partition behavior)
Tier 3 — Real Android hardware + PC, self-hosted runner
```

**Tier 0 carries almost all the weight.** N instances as plain OS
processes on one machine, each with an isolated temp data directory,
is a fully valid multi-instance test for anything not specifically
dependent on separate physical hosts — discovery, pairing, sync logic,
conflict resolution. Fast, deterministic, zero infrastructure, runs
identically locally and in CI.

**Tier 1 exists for exactly one reason**: verifying behavior that
depends on genuine network-namespace separation — real socket
isolation, real multicast propagation across a bridge — not just
separate processes sharing one loopback stack. `testcontainers-rs`
(confirmed current, actively maintained, v0.27, async-capable, custom
network support, can build our own image rather than only pulling
prebuilt ones) is the right tool when this tier gets built.

**Tier 2** deferred until there's a concrete need — rare early on.

**Tier 3** is Android + PC over real hardware, self-hosted runner. See
the security section below; this one is not a bigger version of the
others, it's different infrastructure with different risk.

**Decision for this pass: build Tier 0 only, right now.** Tiers 1-3
are architected for (see "pluggable backend" below) but not
implemented yet. Tier 1/2 infrastructure, when built, will run on
the maintainer's own hardware or a VPS — not assumed to require a home
PC specifically.

## Pluggable backend, not four separate test suites

If Tier 1/2/3 get built as unrelated code later, either every test
body gets duplicated per tier, or Tier 0 and Tier 3 drift apart on what
they actually check. Instead: a `testkit` crate exposes one
`TestCluster` abstraction with swappable backends —
`process` (Tier 0), `docker` (Tier 1), `remote` (Tier 2, SSH to a VPS),
`android` (Tier 3). The same test body — "spawn N nodes, wait for
mutual discovery, assert" — runs against whichever backend is
selected. Building a later tier means writing a new backend
implementation, not new tests. This is the concrete mechanism for
"capacity for other tiers" without paying tier 1-3 costs today.

## Workspace restructuring

`sync-mesh-core` is currently a single binary crate. Proposed split,
done now while there's no real code yet (cheapest possible time):

```
sync-mesh-core/           (Cargo workspace root)
├── crates/
│   ├── engine/           (lib — the actual sync engine, unit-testable
│   │                       in-process)
│   ├── daemon/           (bin — thin wrapper: loads engine, exposes
│   │                       gRPC control interface)
│   └── testkit/          (dev-dependency only, not shipped —
│                           TestCluster + pluggable backends)
```

Unit tests (in-process, fast, individual functions/modules) and
integration tests (real spawned processes, actual multi-instance
behavior) are different concerns that both matter — this structure
gives each a natural home rather than conflating them.

## Test 0 is also the first real engine milestone

There's no daemon to spawn yet, so Test 0's actual subject is: build a
minimal daemon exposing a gRPC health/status endpoint. Not test
infrastructure sitting apart from engine work — it's the first
concrete deliverable, and the test and the feature are co-designed.

## Isolation — two separate problems

1. **Test instances vs. a real instance on the same LAN.** Tests use a
   distinct mDNS service type (`_syncmesh-test._tcp`, not the real
   `_syncmesh._tcp`) so a test run on a dev machine can never discover,
   or be discovered by, a real running instance.
2. **Concurrent test runs vs. each other.** `cargo test` runs tests in
   parallel by default. Two different test functions each spinning up
   their own mini-cluster could cross-discover one another under the
   same test service type. Fix: each test run generates its own
   identifier, baked into the mDNS instance name / TXT record;
   assertions filter to peers carrying that ID. Chose to design this
   correctly from the start rather than force `--test-threads=1` as a
   workaround — not meaingfully harder, and keeps the suite fast as it
   grows.

## Port allocation

Bind to port 0, read back the OS-assigned port, rather than fixed
ports — same parallel-test-safety reasoning as above. Fixed ports
would flake under concurrent runs.

## Process teardown

`TestCluster`/`TestNode` guarantee child-process cleanup via `Drop`,
so a panicking assertion can't leak zombie processes or held ports
into the next test run.

## Cross-OS limitation — accepted, not solved

GitHub-hosted runners are isolated VMs with no network path between
them — a `windows-latest` job and an `ubuntu-latest` job in the same
workflow cannot reach each other over a socket. Genuine cross-OS P2P
interop (Linux instance discovering a Windows instance) cannot be
automated on GitHub-hosted infrastructure regardless of how the
workflow is written. What CI *can* do: run the same Tier 0 suite
separately per OS runner, proving correct behavior on each OS
individually. True cross-OS interop testing is deferred to the
maintainer's own multi-machine setup later, or nested VMs on
self-hosted infrastructure if it ever becomes worth the complexity.
Explicitly accepted as a known gap rather than something to fake.

## Self-hosted runner security (Tier 3, and any future Tier 1/2 that
uses one)

Verified directly against GitHub's own documentation: self-hosted
runners "should almost never be used for public repositories," because
any user — not just collaborators — can open a pull request that
triggers a `pull_request`-based workflow targeting the runner's label
and execute arbitrary code on it. `sync-dot-mesh/core` is public. This
is a live, immediate risk, not a hypothetical to harden against later.

Hard requirement for any workflow touching a self-hosted label: trigger
on `workflow_dispatch` only (a human deliberately clicking run), never
on `pull_request`. Applies identically whether the runner lives on the
maintainer's own PC or a VPS.

## No new CI infrastructure needed for Tier 0

It's `cargo test` — already covered by the existing `test` job in
`ci.yml`. Building Tier 0 does not require touching
`.github/workflows/` at all.

## The staged test plan

```
0. Lifecycle           — N instances start, report healthy via gRPC
                          status endpoint, shut down clean, no leaked
                          ports/processes. Includes a negative case:
                          two instances pointed at the same data dir
                          fail cleanly rather than corrupting state.
1. Discovery (2)        — two instances, same test-mDNS domain, mutual
                          visibility within timeout
2. Discovery (N)        — 3-5 instances, full mesh visibility
3. Sync Intensity policy — Tier 0: engine behavior actually changes
                          under Full / Reduced / MetadataOnly config,
                          independent of platform
4. Pairing              — Ed25519 identity exchange completes, trust
                          established
5. File sync            — one instance creates a file, verify it lands
                          on the other, hash matches
6. Conflict             — concurrent edit on both sides, configured
                          policy fires correctly
7. Delta sync           — modify part of a large file, verify a delta
                          transfers, not a full re-send
8. Disruption           — kill an instance mid-transfer, verify
                          retry/resume behavior
9. Cross-platform       — same Tier 0 suite, run separately per OS
                          runner (see limitation above)
10. Sync Intensity triggering — Tier 3 only: foreground service
                          actually responds to real battery/doze state
11. Android + PC        — Tier 3, self-hosted, workflow_dispatch only
```

Items 0-3 are buildable immediately with zero new infrastructure.
4 onward requires real engine features that don't exist yet, so those
tests and their corresponding features get built together,
incrementally, same pattern as Test 0/the daemon skeleton.

## Open items — not resolved, flagged deliberately rather than guessed

**"Offload computation"** — not yet defined precisely enough to design
a test for. Working assumption: distributing hash/delta computation
across peers with spare capacity during large sync operations. Needs
confirmation before whichever test is meant to exercise it gets built;
noted here rather than silently designed around a guess.

**SyncIntensity documentation** — flagged as under-documented on the
org profile relative to how real the feature is (Android foreground
service, three-level power profile). Not expanded in this pass since
this session's scope is testing strategy; worth a short follow-up to
the profile README separately.
