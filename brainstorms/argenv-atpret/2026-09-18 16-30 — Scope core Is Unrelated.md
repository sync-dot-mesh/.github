# 2026-09-18 16:30 — Scope: core Is Unrelated

Explicit correction: `sync-dot-mesh/core` (the P2P file-sync engine)
is **not part of the `argenv`/`atpret` project** and should not be
referenced when planning either one. It shares the `sync-dot-mesh` org
purely for convenience — same as the earlier note that `atpret` and
`core`'s open items (iOS, private-key storage) are unrelated — but a
previous "grand architecture" summary drew `core` into the picture as
a sibling project, which overstated the relationship. `core` shares
devenv/CI/backlog *conventions* with `argenv`/`atpret`, nothing more;
it should stay out of planning for this pair going forward.
