# ADR 0003 — Update Test Class to Inherit from `Minitest::Test`

Date: 2026-06-29
Status: Accepted

## Context

[ADR 0002](0002-modernize-minitest-and-declare-dev-dependencies.md) fixed the
_loading_ side of the Minitest modernisation: it replaced the obsolete
`require 'minitest/unit'` with `require 'minitest/autorun'` in `test/helper.rb`.

That fix is necessary but not sufficient. `test/test_haml2slim.rb` still opens its
test class with:

```ruby
class TestHaml2Slim < MiniTest::Unit::TestCase
```

`MiniTest::Unit::TestCase` was the test base class in **Minitest 4.x** (note the mixed
capitalisation: `MiniTest`, not `Minitest`). Minitest 5 renamed the module to use
consistent Ruby casing and rearchitected the class hierarchy:

| Version | Require       | Base class                  |
|---------|---------------|-----------------------------|
| 4.x     | minitest/unit | `MiniTest::Unit::TestCase`  |
| 5.x+    | minitest/autorun | `Minitest::Test`         |

`MiniTest::Unit::TestCase` is **not defined** in Minitest 5. When `test/helper.rb`
loads `require 'minitest/autorun'` (the Minitest 5 entry point) and then
`test/test_haml2slim.rb` tries to inherit from `MiniTest::Unit::TestCase`, Ruby raises:

```
test/test_haml2slim.rb:4:in `<main>': uninitialized constant MiniTest (NameError)
```

The error surfaces at class evaluation time — before any test method runs. CI fails with
zero tests executed.

## Why Is This a Two-Step Bug?

ADR 0002 landed in a previous PR, but this file was not updated at the same time. That
is the classic **partial migration** problem: a codebase has two places that embody the
same assumption ("we use Minitest 4"), and only one of them was updated.

Partial migrations are particularly dangerous because:

1. **They pass at the step that was fixed.** Once `test/helper.rb` requires the modern
   entry point, the file itself is clean. The breakage lives one level up in the caller.
2. **The failure mode is surprising.** The error says `uninitialized constant MiniTest`
   — it looks like Minitest is not installed, not like a class hierarchy problem.
   Developers unfamiliar with the Minitest 4→5 rename often spend time debugging gem
   installation when the root cause is a wrong constant name.
3. **Grep can miss it.** If you search for `minitest/unit` you find the already-fixed
   helper. The problematic constant `MiniTest::Unit::TestCase` looks nothing like the
   require statement.

**Rule of thumb**: when you modernise a dependency, grep the entire test directory (and
the production code) for _all_ symbols the old version exported, not just the
require path you changed.

```bash
grep -r "MiniTest" test/   # catch the old camelCase spelling
grep -r "Minitest" test/   # confirm the new lowercase spelling is consistent
```

## Decision

Change `test/test_haml2slim.rb` line 4 from:

```ruby
class TestHaml2Slim < MiniTest::Unit::TestCase
```

to:

```ruby
class TestHaml2Slim < Minitest::Test
```

`Minitest::Test` is the canonical base class for assertion-style tests in Minitest 5 and
all subsequent versions. It provides the same `assert_*` / `refute_*` DSL that
`MiniTest::Unit::TestCase` provided — the migration is purely a rename.

## Consequences

- The `uninitialized constant MiniTest (NameError)` boot error is resolved.
- All existing test methods (`assert_equal`, `assert_haml_to_slim`, `assert_valid?`)
  continue to work unchanged — `Minitest::Test` provides the same assertion DSL.
- Together with ADR 0002, the test suite is fully migrated to Minitest 5 and can run
  on any supported Ruby 3.x version.
- Future contributors who check `class ... < Minitest::Test` will find it consistent
  with every modern Minitest tutorial and README.
