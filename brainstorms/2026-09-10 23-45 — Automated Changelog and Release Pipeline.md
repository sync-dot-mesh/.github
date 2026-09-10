# 2026-09-10 23:45 — Automated Changelog and Release Pipeline

## The problem

Manually written changelogs drift from reality or don't get written
at all. Wanted commit messages to be the single source of truth for
release notes, enforced (not just conventionally hoped for) and wired
into CI so a release's notes are a byproduct of normal work, not a
separate writing task.

## Two layers, kept deliberately separate

**Enforcement** — something has to actually reject a malformed commit
message before it reaches `main`. **Generation** — something parses
the conformant history into an actual changelog and decides the next
version number. Conflating these into one tool choice was the wrong
frame; they're separate problems with separate best answers.

## Enforcement: PR title only, not every commit

Given the actual workflow — Claude opening PRs with however many
messy intermediate commits it takes to land something working —
validating every commit would fight the iteration that makes the
AI-heavy workflow useful. Resolution: **squash-merge only** (disabled
merge-commit and rebase-merge at the repo level), so every PR becomes
exactly one commit on `main`, and that commit's subject/body are the
PR's title/description. Only the PR title needs validating —
`amannn/action-semantic-pull-request`, wired as a required status
check alongside the existing CI checks. Squash commit settings:
title = PR_TITLE, message = PR_BODY — meaning any footers
(`BREAKING CHANGE:`, our custom `User-Facing:`) belong in the PR
description, not buried in a branch's internal commits.

## Generation: release-plz over the generic alternatives

Surveyed `release-please` (Google, multi-language, Rust as one of
several supported ecosystems), `semantic-release` (Node-based, wrong
runtime for a project with zero JS anywhere), plain `git-cliff` alone
(excellent changelog generator, but no release/version-bump
orchestration by itself), and `release-plz` (Rust-native, purpose-
built for Cargo projects).

`release-plz` won specifically because it's the least generic option
for exactly this project type:
- Uses `git-cliff` internally, so we get git-cliff's templating and
  don't lose anything by not using it standalone.
- Cross-checks the conventional-commit-implied version bump against
  `cargo-semver-checks`, which actually inspects the public API for
  breaking changes — catches a mislabeled `fix:` that was actually
  breaking, not just trusting the commit type blindly.
- Maintains a continuously-updated Release PR rather than a one-shot
  CLI invocation — nobody writes a changelog by hand at any point.
- crates.io publishing is a per-package toggle, currently **off** —
  `sync-mesh-core` is an internal FFI-consumed component right now,
  not something meant for external `cargo add`. Revisit if that
  changes.

## The type/scope/footer design — solving "not descriptive enough"

The standard Conventional Commits type vocabulary (`feat`, `fix`,
`perf`, `refactor`, `docs`, `style`, `test`, `build`, `ci`, `chore`,
`revert`) was kept as-is — no reason to reinvent 11 well-understood
buckets. The actual descriptiveness problem lives one level down:

- **Scopes**, matching the real architecture: `sync-engine`,
  `transport`, `persistence`, `ui`, `git-sync`, `packaging`, plus
  `deps` (Dependabot's own commits) and `ci` (added after the first
  real test PR failed on a scope I'd forgotten to allow — see below).
  git-cliff groups by type and shows scope inline per entry, so a
  release reads as organized by system area, not a flat list.
- **Custom `User-Facing:` footer** — the actual fix for "the subject
  line alone never says enough." When present, the changelog pulls
  this footer's text into the entry instead of the raw commit
  subject. Verified for real: a PR titled
  `feat(sync-engine): scaffold change detection module` produced a
  changelog line reading "Sync.Mesh will detect file changes on
  Linux, macOS, and Windows through a single unified interface
  instead of three platform-specific code paths" — the footer text,
  not the terse subject.
- **Dependencies isolated** — Dependabot's `chore(deps)`/`build(deps)`
  commits get their own changelog section instead of diluting the
  main list with routine version bumps.

## What broke on the way to working — and why it's worth keeping

Three real bugs, each caught by actually running the pipeline rather
than trusting the config on paper:

1. **`pull_request_target` chicken-and-egg.** That trigger reads the
   workflow definition from the *base* branch, not the PR branch — so
   a workflow introduced in the same PR that adds it can never fire on
   that PR. Switched to plain `pull_request`, which evaluates from the
   PR branch itself.
2. **Missing `ci` scope.** The first real test PR used
   `chore(ci): ...` and failed its own title check — `ci` wasn't in
   the allowed scopes list, despite `cliff.toml` already having a
   dedicated CI/CD group. Added it.
3. **Org-level Actions policy blocking PR creation.** `release-plz`'s
   first real run failed with "GitHub Actions is not permitted to
   create or approve pull requests" — a security default that exists
   specifically to stop a compromised workflow from silently opening
   PRs, and it overrides even an explicit `permissions:` block in the
   workflow file itself. Had to be flipped at both the org level
   (`sync-dot-mesh`) and the repo level via the Actions API before
   `release-plz` could open its Release PR.

## Verified end to end with two real merges

PR #2 (`chore(ci): ...`, the pipeline setup itself) and PR #4
(`feat(sync-engine): ...`, a real scaffold module with a `User-Facing`
footer) both went through the full loop: PR-title check → CI → squash
merge → `release-plz` picked up the push and opened/updated a Release
PR with a correctly-grouped, footer-aware changelog. Confirmed by
reading the actual generated `CHANGELOG.md` diff, not by assuming the
config was right.

Also confirmed incidentally: merging a PR you authored yourself, with
`required_approving_review_count: 1` active and no review submitted,
succeeds via `enforce_admins: false` — the admin-bypass path works
through the API, not just the web UI's "merge without waiting" button.
Matches the design intent from the branch-protection setup — the
review gate is doing its real work on PRs that aren't yours (bots,
eventually Claude via the Action), and the bypass exists so a solo
maintainer isn't locked out of their own repo.

## Left undone, deliberately

The Release PR (`chore: release v0.1.0`) was not merged. Doing so
creates a real, permanent git tag and public GitHub Release, and the
project is still scaffolding — cutting an actual first release is
staying a deliberate human decision, not something to fold into a
tooling-verification pass.
