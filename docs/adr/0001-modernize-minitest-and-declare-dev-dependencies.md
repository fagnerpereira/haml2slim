# ADR 0001 — Modernize Minitest Usage and Declare Dev Dependencies Explicitly

Date: 2026-06-28  
Status: Accepted

## Context

The test helper (`test/helper.rb`) contained two lines that worked under Ruby 2.x but break under Ruby 3.x:

```ruby
require 'minitest/unit'   # LoadError in Ruby 3.x
MiniTest::Unit.autorun    # Old API from Minitest 4.x
```

When CI upgraded to Ruby 3.x the test suite stopped running entirely — not a test failure, but a boot failure:

```
LoadError: cannot load such file -- minitest/unit
```

The `haml2slim.gemspec` also omitted `minitest` from its development dependencies.

## The Lesson: Bundled Gems vs. the Standard Library

Ruby ships with two categories of built-in code that are easy to confuse:

**Standard Library** — pure Ruby files always available regardless of Bundler (e.g., `json`, `csv`, `date`, `uri`). You can `require 'json'` in any Ruby script.

**Bundled Gems** — actual gems that shipped _alongside_ Ruby but live in gem-space. Before Ruby 3.0, they were installed automatically with each Ruby version so you rarely had to think about them. Minitest was one of these: it rode along with Ruby 2.x without needing a `Gemfile` entry.

Starting with Ruby 3.0, Ruby tightened this contract. Some bundled gems were removed entirely; others require explicit declaration to work with Bundler. Minitest is now just a gem like any other — if it is not in your gemspec or Gemfile, `require 'minitest/autorun'` raises `LoadError`.

**Rule of thumb**: if your code `require`s it, your gemspec (or Gemfile) must list it. Never rely on implicit transitive dependencies, because the Ruby core team can and does remove them between releases.

## The Lesson: Public API vs. Internal Files

`require 'minitest/unit'` loaded an internal file maintained for backwards compatibility with Test::Unit-style test runners. The public entry point since Minitest 5 has always been:

```ruby
require 'minitest/autorun'
```

`minitest/autorun` does three things in one line:
1. Loads the core Minitest library.
2. Registers an `at_exit` hook that discovers and runs all test classes.
3. Works uniformly with `Minitest::Test`, `Minitest::Spec`, and `describe` blocks.

The old `MiniTest::Unit.autorun` call was redundant — `minitest/autorun` already registers the hook. Calling it again created no harm, but it signalled to every reader that the author did not understand what `autorun` does.

When a library ships multiple `require` paths, prefer the one documented at the top of its README. Internal files (anything not in the public changelog) can vanish without a major version bump.

## Decision

1. Replace `require 'minitest/unit'` with `require 'minitest/autorun'` in `test/helper.rb`.
2. Remove the now-redundant `MiniTest::Unit.autorun` call.
3. Add `s.add_development_dependency 'minitest'` to `haml2slim.gemspec`.

## Consequences

- Tests run correctly under Ruby 3.x and future Ruby versions.
- CI is green.
- Any developer who clones the repo will have minitest installed via `bundle install` regardless of their system Ruby version.
- The gemspec now accurately describes what this gem needs to be developed and tested.
