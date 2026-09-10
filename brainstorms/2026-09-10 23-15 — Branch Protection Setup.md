# 2026-09-10 23:15 — Branch Protection Setup

*Reconstructed sequential time — falls between the repo bootstrap and
the changelog/release pipeline, both of which depend on this.*

## What was configured on `sync-dot-mesh/core`'s `main`

- No direct pushes — everything through a PR.
- Required status checks (`cargo test`, `cargo clippy`,
  `cargo fmt --check` at the time; `conventional commit format` added
  later in the changelog-pipeline session), `strict: true` so a PR
  must be rebased on current `main` before merging, not just green
  against a stale base.
- 1 required approval, stale reviews dismissed on new commits.
- Required conversation resolution before merge.
- Linear history, no force-pushes, no branch deletion.
- `enforce_admins: false`.

## The nuance worth recording: self-approval

GitHub does not allow approving your own pull request — a platform
rule, not something branch protection controls. With one human on the
project, any PR they author themselves has no one else available to
approve it. `enforce_admins: false` is specifically the answer: as org
owner, a "merge without waiting for requirements" path stays available
on your own PRs, so the protection doesn't lock out the person it's
also protecting.

The design intent: the approval requirement does its real work on PRs
that *aren't* the human's — Dependabot's, and eventually
`claude-code-action`'s if that gets wired up later — giving those a
genuine human gate before landing on `main`, while the admin bypass
exists for the human's own direct work. Confirmed later (during the
changelog-pipeline testing session) that this bypass actually works
through the API merge endpoint, not just the web UI's button.

## Open item, noted rather than resolved

If the bypass ends up being leaned on for *every* merge rather than
occasionally, that's a signal the required-approval setting isn't
doing useful work as configured and should be revisited — either
drop `required_approving_review_count` to 0, or bring in a second
reviewer. Not evaluated yet; worth revisiting once there's enough
merge history to tell which pattern is actually happening.
