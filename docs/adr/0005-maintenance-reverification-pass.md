# ADR 0005 — Maintenance Re-Verification Pass (Folding In an Owner-Authored Commit)

Date: 2026-07-25
Status: Accepted

## Context

This is the next cycle of the recurring pattern established in ADR 0004: each session gets a
fresh, disposable branch, re-verifies the previous open PR's claims from scratch (rather than
trusting its description), and re-opens the fix on a branch the current session can actually
push to.

This session (`claude/vigilant-hypatia-1c8jj7`) re-verified #5 (which itself superseded
#4 → #3 → #2). Three things were checked live, not assumed from the task brief:

1. **Had `master` moved since #5 was opened?** `git fetch origin` + SHA comparison: no.
   `master` is still `16c9dbc`, the same commit #5 branched from.
2. **Was #5 actually clean?** Checked via the GitHub API: draft, no review threads, no
   PR comments, no new issues, and no `.github/dependabot.yml` in the repo (still absent).
3. **Was CI actually green?** Fetched check runs for #5's head commit (`bb9f3c5`) directly
   via the Actions API rather than trusting the PR body: two full runs (one `pull_request`
   event, one `push` event), each with `test (3.1)`, `test (3.2)`, `test (3.3)` all
   `conclusion: success` — 6 check runs total, all green.

## What Was Different This Time: An Extra Commit

Unlike ADR 0004's pass, #5 was **not** identical to what the task brief described. The brief
said #5 carried "5 commits" (the Minitest fix, the CI workflow, and the ADR-numbering fix).
Re-reading #5's commit list via the API turned up **6 commits** — a new one at the very
bottom of the history, authored by the repo owner (`fpr <fagnerfpr@gmail.com>`), predating
even the CI workflow commit chronologically:

```text
5f4f67f test: write access check
        Author: fpr <fagnerfpr@gmail.com>
        1 file changed: docs/.gitkeep (0 insertions, 0 deletions)
```

This is exactly the kind of drift this maintenance pattern exists to catch: a stale task
description ("5 commits") is a claim, not a fact, and this session's job is to check the
live PR rather than propagate a possibly-outdated count forward. Inspecting the commit
directly (`git show`) confirmed it is trivial and harmless — an empty placeholder file
under `docs/`, almost certainly the repo owner probing that they had push access to the
branch before Claude sessions started committing to it. It changes no code, adds no risk,
and does not conflict with anything the later commits touch (the `docs/adr/` files added
afterward live in a subdirectory; `docs/.gitkeep` and `docs/adr/*.md` coexist without issue).

**Decision: include it.** The rule this maintenance pattern follows is to recreate the PR's
*actual current commit set*, not a memorized or stale summary of it. Silently dropping a
human-authored commit because it wasn't in the brief would be exactly the kind of unverified
assumption ADR 0004 warned against — the fix is to re-derive the commit list from the live
PR every time, which is what happened here.

## Verification Performed This Session

- All 6 of #5's commits (`5f4f67f`, `5e346f5`, `839b06c`, `ca906cd`, `bbc8cf4`, `bb9f3c5`)
  cherry-picked onto `claude/vigilant-hypatia-1c8jj7`, in original order, with **zero
  conflicts** — expected, since the merge-base is identical to #5's.
- `bundle exec rake test` on Ruby 3.3.6: **34 runs, 34 assertions, 0 failures, 0 errors,
  0 skips** — identical to every prior session's count, confirming the cherry-picked tree
  behaves identically to #5's. (The sandbox's `rake` shim was not wired into `bundle exec`'s
  resolved `PATH` in this environment — a local tooling quirk, not a code issue — so the
  suite was additionally run by invoking the same `rake` test loader executable directly;
  output was byte-for-byte the same pass/fail summary.)
- No lint task exists in this gem's `Rakefile` (only `task :default => 'test'`), so none
  was skipped.
- No `.github/dependabot.yml` exists in this repository, and no Dependabot PRs or alerts
  were present — nothing to fold in on that front.
- No open review threads, no PR comments, and no other open PRs or issues existed at the
  time of this check.

## Decision

1. Cherry-pick all 6 of #5's commits — including the previously-undescribed
   `test: write access check` commit — unchanged, onto `claude/vigilant-hypatia-1c8jj7`.
2. Add this ADR documenting the re-verification and, specifically, the discrepancy between
   the task brief's commit count and the PR's actual live state.
3. Open a new draft PR from this branch, closing #5 with a pointer to the new PR (mirroring
   how #5 closed #4, #4 closed #3, and #3 closed #2).
4. Leave `master` and the fix's functional content untouched — this pass changes *where*
   the fix lives and folds in one harmless human commit; it does not change what the fix does.

## Consequences

- The open PR continues to point at a branch the current session can actually push to and
  stand behind.
- Future sessions inherit a commit history that matches what actually happened, including
  the owner's own write-access check — nothing was silently dropped to make the history
  look tidier than it is.
- **Rule of thumb reaffirmed**: task briefs and prior PR descriptions are a starting point,
  not a source of truth. Re-deriving the commit list, CI status, and review state from the
  live GitHub API — every session, even when "nothing is expected to have changed" — is what
  catches drift like this one before it compounds.
