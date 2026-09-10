# Sync.Mesh

Masterless peer-to-peer file sync. No cloud, no account, no relay by
default — every device is an equal node. LAN discovery via mDNS,
remote sync over WireGuard/Tailscale.

## Shape of the project

**Architecture**
- Core: Onion Architecture — pure domain, zero external dependencies
- Features: Vertical Slice — each feature is self-contained end to end
- UI: Blazor Hybrid, shared across desktop and mobile

**Sync engine**
- Change detection: OS-native (inotify / FSEvents / FileSystemWatcher /
  ReadDirectoryChangesW), debounced ~500ms
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
Core sync is free. Git Sync (a shadow repo via libgit2sharp, tracking
every sync folder's history as commits) is a paid feature, gated by a
signed JWT with a bundled public key — validated fully offline, no
phone-home required for normal operation.

**Tech stack**
.NET 9 end to end for the application layer — MAUI/Blazor shell, DI,
EF Core, gRPC client/server. Under active evaluation: a Rust-native
core for the sync engine specifically (change detection, hashing,
delta computation, WireGuard integration) compiled as a native library
and called from the .NET shell via FFI, keeping Rust's correctness
guarantees where race conditions and raw throughput matter most while
keeping the .NET ecosystem's velocity for UI, DI, and platform
integration everywhere else. See `brainstorms/` for the full reasoning.

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
