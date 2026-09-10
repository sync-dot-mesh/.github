# 2026-09-10 22:00 — GitHub Feature Survey for Open Source and AI Workflow

*Reconstructed sequential time — falls between org creation (21:45)
and the repo bootstrap that acted on it.*

## The goal

Establish a productive flow specifically for a human-plus-AI team
shape: Claude doing the heavy lifting, the human refining ideas,
testing locally, and occasionally editing Rust they're still learning.
Researched current (2026) GitHub features rather than relying on
stale assumptions, ranked by actual impact on that shape.

## Tier 1 — directly build the offload loop

- **`anthropics/claude-code-action`** (official) — runs the real
  Claude Code runtime inside an Actions runner, triggered by `@claude`
  mentions or issue assignment, opens draft PRs. Two auth paths:
  `ANTHROPIC_API_KEY` (pay-per-token) or `CLAUDE_CODE_OAUTH_TOKEN`
  (uses an existing Claude Pro/Max subscription instead of separate
  billing). Decided to skip wiring this up for now — the human is
  already relaying between this chat and GitHub manually, which covers
  the same loop "just backwards."
- **`github/gh-aw`** (GitHub Agentic Workflows, official GitHub Next
  project) — natural-language Markdown + YAML frontmatter compiled
  into hardened Actions workflows, for *recurring* automation rather
  than on-demand conversation. Supports Claude as an engine.
  Security model ("safe outputs") sandboxes writes by default. This is
  what got implemented — the weekly issue triage workflow.

## Tier 2 — free, high yield, no real setup cost

- **GitHub Actions on the public repo** — unlimited minutes on
  standard hosted runners, 20 concurrent jobs on the free plan, 6-hour
  single-job cap, no automatic retry (manual re-run or
  `continue-on-error` needed).
- **Dependabot** — native Cargo ecosystem support, free, no reason not
  to enable.
- **CodeRabbit** — free AI code review for public repos, complements
  rather than replaces Claude review.

## Tier 3 — conditional, verified rather than assumed

- **Copilot coding agent** — real and mature, but new Pro/Pro+/Max
  signups were paused starting April 20, 2026 (existing subscribers
  unaffected), and the free-for-OSS-maintainers program only kicks in
  once a repo is already popular (~2,500+ stars by community
  consensus) — not applicable to a brand-new project. Told the human
  to check `github.com/settings/copilot` directly rather than either
  of us guessing their entitlement.
- **GitHub Projects (v2)** — ranked low for a two-person team. Built-in
  automation is solid for simple cases; custom workflows need
  hand-rolled GraphQL + Actions glue, and there's visible community
  frustration it hasn't evolved much. Plain Issues + labels judged
  sufficient at this scale.
- **Codespaces** — lower priority given the Nix devenv already solves
  environment consistency locally; also, the free personal quota
  doesn't cleanly extend to org-owned repos (sources conflicted on
  exact behavior, flagged rather than asserted).

## What actually got built from this list

Items 2, 3, 4 in the original ranking — CI (test/clippy/fmt), Dependabot,
and the `gh-aw` weekly triage job — implemented in the repo bootstrap
session immediately following this one. `claude-code-action` and
Copilot's coding agent both deliberately deferred, Projects and
Codespaces skipped for now.
