# ADR-0001: Add GitHub Actions CI and Drop Ruby 3.0 from the Test Matrix

**Status:** Accepted
**Date:** 2026-06-28
**Context:** The gem shipped with no automated CI. Maintainers had to test locally.

---

## The Problem

haml2slim has no GitHub Actions workflow. Every contributor must run `bundle exec rake test`
locally and trust that they remembered. There is no gate that catches regressions before a
commit lands on `master`.

This is the most common form of **technical debt** in open-source gems: the code works today,
but the first person to upgrade a dependency or change Ruby version has no safety net.

---

## What We Added

A single CI job (`.github/workflows/ci.yml`) that:

1. Runs on every push to `master` and every pull request.
2. Tests against **Ruby 3.1, 3.2, and 3.3** — all versions currently receiving security patches.
3. Uses `actions/checkout@v4` and `ruby/setup-ruby@v1` with `bundler-cache: true` to keep
   runs fast by caching the installed gems between runs.

---

## Why Ruby 3.0 is Excluded

Ruby 3.0 reached **end-of-life in March 2024**. It no longer receives security patches.

Supporting an EOL Ruby version costs something:

- You cannot use language features introduced in 3.1+ (pattern matching improvements,
  `Data.define`, numbered block parameters, endless method definitions, etc.).
- CI has to warm up an old toolchain cache that grows stale.
- Any bug in Ruby 3.0 that is fixed in 3.1 becomes your problem, not Ruby's.

**Rule of thumb:** drop EOL Ruby versions promptly. The cut-off is when the ruby-lang.org
[maintenance page](https://www.ruby-lang.org/en/downloads/branches/) removes the version.
The cost of dropping it is asking users to upgrade a runtime they should have already upgraded.
The cost of keeping it is silently accumulating compatibility debt forever.

---

## Why `ruby/setup-ruby` with `bundler-cache: true`?

`ruby/setup-ruby` is the official GitHub Action maintained by the Ruby core team. It:

- Reads `.ruby-version` if present, or accepts the `ruby-version` matrix input.
- Automatically installs the right version of Bundler for the Ruby version.
- With `bundler-cache: true`, caches `vendor/bundle` between runs so `bundle install`
  on the second and subsequent runs takes seconds, not minutes.

Alternative approaches (`actions/cache` + manual `bundle install`) work but require more
boilerplate and are more likely to break when Bundler changes its lock-file format.

---

## The Broader Principle: CI is the First Line of Defense

A CI workflow is not a luxury. It is the contract that says "this code works on supported
platforms." Without it:

- Reviewers must trust that the author ran the tests.
- A broken `master` can persist for days before anyone notices.
- New contributors cannot know if their environment is the problem or the code is.

Add CI before writing a line of feature code. It costs 20 lines of YAML and pays off
on the first PR from anyone other than yourself.
