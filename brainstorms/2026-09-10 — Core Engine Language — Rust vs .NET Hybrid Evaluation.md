# 2026-09-10 — Core Engine Language: Rust vs .NET Hybrid Evaluation

*Predates the org — dated by day only, reconstructed from conversation
order.*

## The question

The profile README describes .NET 9 end to end (MAUI/Blazor shell, DI,
EF Core, gRPC). Given the sync engine specifically — change detection,
Blake3 hashing, rsync-style delta computation, WireGuard integration —
is Rust a better fit for that piece, and if so, how far does that go?

## Where Rust is genuinely better for this specific workload

- **Binary size / idle memory / cold start** — a Rust daemon
  meaningfully beats even .NET Native AOT here (roughly 3-8MB vs
  15-25MB binaries, 5-15MB vs 30-60MB idle RSS, <5ms vs 10-30ms cold
  start). Matters because this is a daemon sitting in the background
  for hours, competing visibly with Syncthing/Resilio Sync in a task
  manager.
- **Concurrency correctness** — Tokio's async model plus the borrow
  checker make the exact race-condition classes relevant to a sync
  engine (shared state between the file watcher, differ, transfer
  engine, and UI) compile errors rather than a discipline problem.
- **The delta-sync algorithm and Blake3** — CPU-intensive, benefits
  from zero-GC-pause execution and SIMD; Blake3's reference
  implementation is Rust's.
- **WireGuard** — `boringtun` (Cloudflare's userspace implementation)
  is Rust-native; calling it from .NET means P/Invoke into a C library
  instead.

## Where Rust is worse, and why it's not a small problem

- **Android** — the named platform requirement with the most friction.
  No mature single-language path exists; realistic options are a
  Rust-core-plus-Kotlin-UI split via JNI, Tauri's still-early Android
  support, or Flutter+Rust via `flutter_rust_bridge` — every option
  means two languages and two build systems, versus MAUI's native,
  documented Android story (foreground service, battery APIs) in one
  language.
- **UI generally** — no Rust equivalent of Blazor Hybrid for the
  settings panel; system tray crates exist but are less battle-tested
  than the .NET ecosystem's.
- **Application-layer ecosystem depth** — DI, EF Core, Serilog,
  OpenTelemetry all have less mature Rust equivalents; Rust's ownership
  model makes container-style DI itself feel unnatural, pushing toward
  manual wiring.
- **Development speed** — rough estimate, 40-70% longer for the same
  scope in Rust, concentrated in application-layer code rather than
  systems-level code.

## The decision: hybrid

Rust core compiled as a native library (change detection, hashing,
delta computation, WireGuard/transport), called from the .NET MAUI
shell via FFI. This is the same pattern Firefox (Rust components, C++
shell), Dropbox (Rust sync engine), and 1Password (Rust core, native
UI per platform) use. Gets Rust's correctness guarantees exactly where
they matter most, keeps .NET's ecosystem depth and Android story
everywhere else. Cost: FFI boundary management — domain types crossing
the boundary become flat C-ABI structs or serialised payloads, not
rich Rust enums directly.

This is why `sync-mesh-core` exists as its own repo rather than living
inside a .NET solution.
