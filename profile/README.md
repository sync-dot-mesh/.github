# Sync.Mesh

Masterless peer-to-peer file sync. No cloud, no account, no relay by
default — every device is an equal node. LAN discovery via mDNS,
remote sync over WireGuard/Tailscale.

## Shape of the project

**Architecture**
Hybrid: a Rust core handles the sync engine — change detection,
hashing, delta computation, transport — where correctness under
concurrency and raw throughput matter most. A .NET MAUI/Blazor Hybrid
shell handles UI, dependency injection, persistence, and platform
integration, calling into the Rust core via FFI. Full technical
reference, including the reasoning behind this split, lives in
[`core/ARCHITECTURE.md`](https://github.com/sync-dot-mesh/core/blob/main/ARCHITECTURE.md).

**Sync engine**
- Change detection: proxied file access (FUSE-style passthrough) where
  available — hashes computed during the write itself, zero re-read —
  with automatic fallback to native OS event watching
  (inotify/FSEvents/ReadDirectoryChangesW) wherever a proxied mount
  isn't available or fails, including always on Android
- Delta sync: rsync rolling-checksum algorithm for files over 10MB
- Content addressing: Blake3 hashing for snapshots and change detection
- Node identity: Ed25519 self-signed certs, pinned on first pairing
- Conflict handling: LastWriteWins | KeepBoth | ManualResolution,
  selectable per sync folder

**Transport**
- LAN: mDNS/Zeroconf discovery, direct connection
- Remote: WireGuard or Tailscale, user-configured
- Protocol: gRPC — `Pair`, `GetStatus`, `GetSnapshot` (stream),
  `NotifyChange`, `PushFile` (stream), `RequestDelta` (stream),
  `WatchChanges` (bidirectional stream)

**Persistence**
SQLite via EF Core — node identity, paired nodes, sync folders,
per-node folder state, file snapshots (Blake3 hash), conflict records,
and an append-only sync event log for audit/debugging.

**Platforms**
| OS | Package formats |
|---|---|
| Linux | `.deb`, `.rpm`, `.AppImage`, NixOS flake |
| Windows | `.exe` (InnoSetup), MSIX |
| macOS | `.pkg`, `.dmg` (codesigned, launchd) |
| Android | `.apk`, `.aab` — Google Play + F-Droid |

Android gets a foreground service and a `SyncIntensity` power profile
(Full / Reduced / MetadataOnly) rather than syncing unconditionally in
the background.

**Licensing**
AGPLv3. The entire codebase — including paid-feature code — stays open
source through any fork; there is no separate closed or
source-available tier. Git Sync (a shadow repo tracking every sync
folder's history as commits) is a paid feature in the sense of an
officially issued, signed license key plus ongoing support — the
gating code itself is open and technically forkable, which is an
accepted consequence of the license, not something being worked
around. Validated fully offline against a bundled public key, no
phone-home required.

**Tech stack**
Rust for the sync engine core (change detection, hashing, delta
computation, transport), compiled as a native library and called from
a .NET 9 shell (MAUI/Blazor Hybrid, DI, EF Core, gRPC) via FFI. See
`brainstorms/` for the full Rust-vs-.NET evaluation that led here, and
[`core/ARCHITECTURE.md`](https://github.com/sync-dot-mesh/core/blob/main/ARCHITECTURE.md)
for the complete technical reference.

**Infrastructure (Azure)**
The sync path itself touches no cloud service by design. Azure is used
only at the edges: Blob Storage + CDN for distributing installers,
Key Vault + a Functions endpoint for signing and issuing Git Sync
licences, and — as an optional paid tier — a stateless relay on
Container Apps for peers that can't establish a direct connection.
The relay only ever forwards already-encrypted packets.

---

## Where the actual history lives

This README is the current, synthesized shape of the project. The
reasoning behind each decision — including the ones we changed our
minds on — lives chronologically in [`brainstorms/`](./brainstorms),
one file per session, newest at the bottom of the index there.
