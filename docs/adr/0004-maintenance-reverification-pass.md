# ADR 0004 — Maintenance Re-Verification Pass (Superseding #4 With No New Content)

Date: 2026-07-24
Status: Accepted

## Context

This repository has a recurring pattern now, visible in the history of #1 → #2 → #3 → #4:
each maintenance session gets a fresh, disposable branch (`claude/vigilant-hypatia-*`), and
if nothing has changed upstream, the session's job is to re-verify the prior PR's claims from
scratch and re-open it on its own branch, rather than trust the previous session's PR
description at face value.

This session (`claude/vigilant-hypatia-pec4g1`) is exactly that case again. Before touching
anything, three things were checked live rather than assumed from the brief:

1. **Had `master` moved since #4 was opened?** `git fetch origin` + comparing SHAs: no.
   `master` is still `16c9dbc`, the same commit #3 and #4 both branched from.
2. **Was #4 actually clean?** No review threads, no comments, draft, mergeable. Checked via
   the GitHub API, not inferred from the PR body.
3. **Had anything new appeared?** No new PRs, no new issues, no Dependabot activity.

## Why Re-Verify a Claim Instead of Trusting It

#4's own PR body claims: *"CI (GitHub Actions, introduced by this PR's own commits) was green
across Ruby 3.1 / 3.2 / 3.3."* That is a claim worth being suspicious of on its face — #4's
commits added the CI workflow file itself, so the very first time that workflow could have run
was on #4's own branch. A prior session asserting "CI was green" without a fresh reader
confirming it is exactly the kind of unverified claim this maintenance pattern exists to catch.

This session checked it independently: `GET /actions/runs?branch=claude/vigilant-hypatia-zifohv`
returned two completed runs against #4's head commit (`09037a05`), one on `pull_request` and
one on `push`, both `conclusion: success`. The claim held up. But the point isn't that it
happened to be true — it's that **"the previous PR said so" is never treated as adequate
verification on its own.** A future session finding a red run here should not defer to a green
claim in a PR description; it should read the run.

## The Decision: Cherry-Pick Forward Even Though Nothing Changed

Since master hadn't moved and #4 was still clean, the same 5 commits (`ce67e73`, `bb01318`,
`f467e43`, `376af67`, `09037a0`) were cherry-picked, unmodified, onto this session's branch
`claude/vigilant-hypatia-pec4g1`. All five applied without a single conflict — expected, since
the merge-base is identical to #4's.

This might look like busywork: same code, same tests, same ADRs, just recommitted under a new
branch name. It isn't, for a structural reason specific to how these sessions are dispatched:

**Each session gets a new, disposable branch name it does not choose.** The aggregate PR that
tracks "the current state of this fix" has to point at *some* live branch, and that branch
changes every session. If a session finds nothing new and does nothing, the aggregate PR either
goes stale (pointing at a branch nobody is re-verifying anymore) or has to be left exactly as
#4, owned by a branch this session has no authority to push to. Re-creating the PR on the
session's own designated branch is what keeps "the current, re-verified state of this fix" and
"the branch a human can safely tell someone to review" the same thing.

**Rule of thumb**: a maintenance pass that finds nothing new is not a no-op. Its output is a
freshly re-verified branch pointer plus a paper trail (this ADR) of exactly what was checked and
when. That trail is what lets the *next* session — or a human — trust "still green" instead of
re-deriving it from nothing.

## Verification Performed This Session

- `bundle exec rake test` on Ruby 3.3.6: **34 runs, 34 assertions, 0 failures, 0 errors, 0
  skips** — identical to every prior session's count, confirming the cherry-picked tree behaves
  identically to #4's.
- No lint task exists in this gem's `Rakefile` (only `task :default => 'test'`), so none was run.
- No Dependabot config (`.github/dependabot.yml`) exists in this repository, and no Dependabot
  PRs or alerts were present — nothing to fold in on that front.

## Decision

1. Cherry-pick #4's 5 commits, unchanged, onto `claude/vigilant-hypatia-pec4g1`.
2. Add this ADR documenting the re-verification.
3. Open a new draft PR from this branch, closing #4 with a pointer to the new PR (mirroring how
   #4 closed #3, and #3 closed #2).
4. Leave `master` and the fix's actual content untouched — this pass changes *where* the fix
   lives, not *what* it does.

## Consequences

- The open PR always points at a branch the current session can actually push to and stand
  behind, instead of accumulating orphaned PRs against branches from sessions that no longer
  exist.
- Anyone reviewing the current PR gets a fresh confirmation timestamp (this ADR's date) instead
  of having to trust a claim made possibly weeks earlier.
- If master *had* moved, or something new *had* appeared, this same process is where that would
  have been caught and folded in — this ADR is also the record that says "checked, and there
  was nothing to fold in this time."
