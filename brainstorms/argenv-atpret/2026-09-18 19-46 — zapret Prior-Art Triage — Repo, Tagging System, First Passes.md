# 2026-09-18 19:46 — zapret Prior-Art Triage — Repo, Tagging System, First Passes

Built `sync-dot-mesh/zapret-prior-art` (private): an atomised, resumable triage of **every issue and every
commit** of `bol-van/zapret` (z1), so the failure modes its users and maintainer discovered in practice can be
grouped and turned into automatic *catch → handle* rules for `atpret`. Standing instruction from this session:
**always backlog, and continue** — every session gets a dated entry here and the pass keeps going.

## 1. What exists

- One record file per issue (830) and per commit (974), in a controlled-vocabulary tag system
  (`facet:value`: kind, ctype, layer, cause, proto, engine, os, target, impact, outcome, constraint, auto, conf).
- A single CLI (`tools/pz.py`): `bootstrap`, `apply` (one line per record; a whole batch is validated first and
  **any error rejects it, writing nothing**), `next`, `status`, `validate`, `index`, `replay`. Progress is derived
  from the records, so the work can stop after any command and resume exactly where it left off.
- Every manual tagging pass is a replayable batch file **and its own git commit**; rebuilding the repo from scratch
  by replaying the batches reproduced the working state byte-for-byte (which also caught a real bug: git does not
  track empty directories, so a fresh clone crashed). CI runs `pz validate` on every push.
- All 830 issue threads are cached locally (git-ignored, re-fetchable) so reading is offline; raw third-party
  text is deliberately not committed, records hold our own summaries.

## 2. Decisions

- **Private by default.** No licence has been chosen and parts of the analysis are marked unverified; flipping to
  public is one setting.
- **Standalone repo for now.** The tools use relative paths, so it can also live at `atpret/prior-art/zapret/`;
  placement is still open.
- **Aligned to the finalized plan** (typed wrapper around the real `nfqws`/`winws`, strategy ADT that rejects
  illegal phase orderings and incompatible mode pairs at construction): added a `constraint` facet
  (`illegal-combo`, `phase-order`, `needs-cap`, `param-range`, `engine-only`, `scope-limit`) and a generated
  `indexes/constraints.md` catalog, seeded from already-analyzed records.
- Issues are processed in ascending order (2016 to 2026); commits are tagged at **subject + touched-file level only**
  (diffs not opened), recorded as `depth: title`.
- A tool bug was caught in review before anything was pushed: a batch summary *replaced* the earlier one, wiping
  detail on 35 records. Fixed with an append mode (`+ text`) and the affected commits were rebuilt.

## 3. Progress and findings so far

- 325/830 issues and 771/974 commits analyzed (issues up to #384; commits up to May 4 2025, plus all docs-only and merge commits by an explicit rule). Cursor: next issue #386, next commit seq 517.
  merge commits by an explicit rule). Cursor: next issue #325, next commit seq 426.
- The Aug-Oct 2024 issue surge is mostly unsupported-platform and "give me a config" requests; real defects are a minority.
- A maintainer-stated platform limit can be **superseded**: nfqws "cannot do methodeol" (2020) became a feature in
  Nov 2024. Re-verify before encoding a constraint; probe capabilities rather than trust static tables (a reporter
  also claims MSS works on macOS although the source disables it).
- The DPI may react differently to QUIC from different client libraries (a client-fingerprint dimension for probing).
- Maintainer rule worth copying into support-bundle design: reports without capture files of a working and a failing
  attempt plus the config are not actionable.
- Crafted-packet fidelity matters: an uninitialised IP TOS field in nfqws-generated packets made YouTube fail on a Samsung TV until fixed (#341), so every header field of crafted packets needs conformance testing.
- Fake TLS content is a moving target: it became parametric in Jan 2025 and the default and GGC/Kyber fakes changed several times; treat fake generation as configurable, not constant.
- Decisions reverse: HUP list reload was removed (Oct 2024) and returned (Jan 2025).

## 4. Open

- Standalone repo vs `atpret/prior-art/`; licence.
- **zapret2** is untouched, and `atpret` is described as its rewrite, so its history and issues are likely more
  relevant than z1's; likely next to bootstrap into the same structure.
- About 505 issues and 203 commits remain in z1; commit diffs still unread.
