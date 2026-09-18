# 2026-09-18 — argenv CI Polish: Merged, With One Org-Level Blocker

## What this pass covered

Per explicit instruction: `argenv`-only work at this stage (`atpret`
still not started, waiting on this). Polished `argenv-opencommons/argenv`
to the same CI/release discipline as `sync-mesh-core`:

- `git-cliff` + `release-plz` on top of the *existing* three CI jobs
  (build/test/lint, MSRV check, and a genuinely good one worth noting
  again: independent Python-`jsonschema` conformance checking of the
  emitted schema against something other than the code that produced
  it) — not a from-scratch pipeline. Replaced the old manual
  tag-triggered `publish-crate.yml` with an automated `release-plz.yml`
  that maintains a standing release PR on every push to `main`.
- `release-plz.toml`: workspace-level `publish` default (the `argenv`
  library publishes to crates.io by default; `argenv-cli` already
  opts out via its own `Cargo.toml`), not a per-crate override —
  the same lesson `sync-mesh-core` already learned the hard way
  (a per-crate override there went stale the moment the workspace
  was restructured and silently broke `release-plz` on two separate
  merges before anyone noticed).
- PR-title enforcement with scopes matching `argenv`'s real module
  shape (`model`/`cli`/`contract`/`lint`/`api`/`ci`/`deps`/`release`),
  not scopes copied from `sync-mesh-core`.
- A `brainstorms/` backlog folder inside `argenv` itself, matching
  `sync-dot-mesh`'s own convention.

Merged as [argenv PR #1](https://github.com/argenv-opencommons/argenv/pull/1)
— all four checks (including the new PR-title one) genuinely green
before merge, following the merge-discipline correction already
established for `sync-mesh-core`: full CI wait every time, no
shortcuts, branch protection locked in (required checks + squash-only
merges) only *after* those checks proved themselves on a real PR, not
assumed from local testing.

## The real catch — checking `main` after merge, not just the PR

Exactly the discipline that caught `sync-mesh-core`'s `release-plz.toml`
drift paid off again here: the PR's own checks were all green, but
`main`'s post-merge `Release` workflow failed. Investigated rather
than assumed fine:

```
Failed to open PR
Caused by: GitHub Actions is not permitted to create or approve
pull requests. (403 Forbidden)
```

**Root cause: an org-level GitHub setting**, not a config file —
`argenv-opencommons`'s Actions permissions don't allow workflows to
open pull requests, which is exactly what `release-plz` needs to do
to maintain its standing release PR. Attempted the repo-level fix
directly via the API (`PUT .../actions/permissions/workflow` with
`can_approve_pull_request_reviews: true`) — rejected: `"Write
permissions for workflows are disabled by the organization"`. The
repo-level setting can't override an org-level restriction, and this
session's token doesn't carry org-admin scope to change the org
setting itself.

**Still outstanding, needs a human with `argenv-opencommons`
org-admin access**: Organization Settings → Actions → General →
Workflow permissions → check "Allow GitHub Actions to create and
approve pull requests." Once set, `release-plz` should work
immediately with no code change — the workflow itself is already
correct, this is purely an org policy gate in front of it.

**Also noted, confirmed pre-existing and not a regression from this
pass**: a `Publish contract API` workflow failure on `main`, dated
2026-07-23 (commit `9170097`, before this session touched anything).
Not investigated further this session since it predates the work
here — worth a look whenever `argenv`'s GitHub Pages setup is next
touched, but out of scope for a CI-polish pass that didn't cause it.

## What's still pending for `argenv` before `atpret` can depend on it

1. **The org-level Actions permission above** — blocks `release-plz`
   entirely until fixed by someone with org access.
2. **`CARGO_REGISTRY_TOKEN` repo secret** — not yet added, flagged in
   the workflow's own comment. Needed for the actual `cargo publish`
   step once a release PR is merged (separate from the PR-opening
   step blocked above).
3. Once both are in place: cut a real `0.1.0` release to crates.io,
   then start the `atpret` repo with `argenv` as an ordinary
   `cargo add` dependency, as originally sequenced.
