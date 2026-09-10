# 2026-09-10 — Infrastructure Strategy: Azure for Sync.Mesh

*This session happened before the `sync-dot-mesh` org existed — no
API-verified timestamp, dated by day only, reconstructed from
conversation order.*

## The question

Sync.Mesh is deliberately masterless — no cloud relay, no account
required, every device is an equal node. The question was where, if
anywhere, Azure legitimately fits without contradicting that design.

## The answer: Azure serves the business around the project, not the
sync path itself

The sync engine touches no cloud service by design. Azure's role is
entirely at the edges:

- **Distribution** — Blob Storage + CDN for hosting installer binaries
  across every packaging target (`.deb`, `.rpm`, `.AppImage`, `.exe`,
  `.pkg`, `.dmg`, `.apk`), landing page hosting via Static Web Apps,
  GitHub Actions → Azure Blob as the release pipeline.
- **Git Sync licensing** — the paid feature (a shadow repo tracking
  sync history as commits, gated by a signed JWT) needs licence
  issuance infrastructure. An Azure Function handles issuance on
  payment confirmation; the signing private key lives in Key Vault,
  read via the Function's managed identity — the key never touches
  application code or environment variables. Licences validate fully
  offline once issued; no phone-home required for normal operation.
- **Optional relay tier** — for peers that can't establish a direct
  connection (NAT traversal failure, no WireGuard/Tailscale
  configured), a stateless relay on Container Apps as a future paid
  feature. Scale-to-zero means it costs nothing when unused. Critically,
  it only ever forwards already-encrypted packets — it cannot read
  content, so it doesn't compromise the masterless/private design.
- **Opt-in telemetry** — Application Insights, off by default,
  anonymised, no file names/content/node IDs — a possible future
  addition for crash reports and performance metrics, contingent on
  genuine user opt-in.

## What was explicitly rejected

Any Azure service in the actual sync/transport/discovery path. The
architecture's core selling point — no cloud, no account, works
entirely peer-to-peer — doesn't get compromised for infrastructure
convenience anywhere in this plan.

## Rust/.NET stack question this connects to

General .NET-on-Azure knowledge (App Service, Container Apps,
Key Vault, Application Insights, managed identity patterns) covered
separately as background — the project-specific decisions above are
what actually apply here.
