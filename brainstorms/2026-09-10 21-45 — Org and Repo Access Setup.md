# 2026-09-10 21:45 — Org and Repo Access Setup

## What happened

Set up the `sync-dot-mesh` GitHub org as the project's public home.

- Verified a fine-grained PAT scoped to the org actually has working
  access (org read, repo creation, contents write).
- Created the `.github` repo — GitHub's special convention for an org
  profile: a public repo named exactly `.github` with a
  `profile/README.md` inside it renders as the org's homepage.
- Established the layout going forward:
  - `profile/README.md` — the synthesized, current shape of the
    project. Gets edited/rewritten as understanding evolves; not a
    changelog.
  - `brainstorms/` — this folder. One dated file per session,
    append-only, never edited after the fact. The reasoning trail
    behind whatever the README currently says.
- Filename convention: `YYYY-MM-DD HH-MM — Subject.md`, human-readable
  date and time plus a short subject, newest entries added to the
  bottom of the index table above.

## Open question

Whether to backfill the substantive design discussions that happened
before this org existed — the Rust vs .NET evaluation for the sync
engine, the DDD/domain-model discussion, the Azure usage plan — as
their own dated entries here, or leave them folded into the profile
README's synthesis only and start the session log fresh from here.
