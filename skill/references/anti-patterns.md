# Test Anti-Pattern Catalog

Common test quality issues and anti-patterns, ordered by severity. Each entry includes:
detection method, why it's a problem, and fix direction.

---

## P0 — Blocker

### AP1. Ghost Mock

**Detection**: `patch("x.y")` where the target path doesn't resolve to a real attribute in the
source, or `create=True` used against a nonexistent symbol that silently creates an auto-mock.

**Why it's a problem**: Every assertion downstream of a ghost mock is asserting against an
auto-generated stub — mutation-blind by construction. The test verifies nothing about real
production behavior.

**Fix**: Correct the mock path to point at the real attribute, or delete the test if the
original seam no longer exists.

**Source**: SKILL (2) Rule 10


### AP2. Invariant Erosion

**Detection**:
- `@pytest.mark.skip` added to a previously passing test
- Assertion changed from `assert result == 42` to `assert result is not None`
- Test renamed in a way that overwrites the old constraint intent
- Test deleted without a replacement

**Why it's a problem**: Invariant tests encode architectural constraints and design decisions.
Weakening or deleting them abandons important quality guards, usually without documenting the
decision to do so.

**Fix** — three options:
1. **Preserve**: Revert the test change, fix the implementation to satisfy the invariant
2. **Layer**: Keep the invariant test, add new behavior test alongside it
3. **Revise**: The invariant is genuinely obsolete — remove the old test AND write a new one
   encoding the replacement invariant

**Source**: SKILL (3) Invariant-Encoding Tests


### AP3. Always-Passing Test

**Detection**: Test body contains zero `assert`/`expect` statements, or all assertions are
unconditionally true (e.g., `assert True`).

**Why it's a problem**: Gives false confidence. Coverage metrics look good, but nothing is
actually verified.

**Fix**: Add meaningful assertions or delete the test.

**Source**: SKILL (4) Red Flags


### AP4. Dead-Dependency Vendor Smoke Test

**Detection**: Test verifies behavior of a third-party library rather than this codebase's usage
of it, AND `git grep` confirms the dependency has zero production references in `src/`.

**Why it's a problem**: Testing third-party library behavior is the library vendor's
responsibility. A dead dependency means the test is pure maintenance overhead.

**Fix**: Delete the test and the unused dependency.

**Important**: If the dependency IS used in production → rewrite the test to pin the specific
flags/wrapper contract this project relies on, rather than testing the dependency's own
capabilities.

**Source**: SKILL (2) Rule 7


### AP5. Unreachable Guard Test (No Production Caller)

**Detection**: Test exercises an early `if not x: return` / `raise` guard on input that no
production call path can produce.

**Why it's a problem**: The test will never catch a regression in real usage, only adding
maintenance cost.

**Exception**: Keep if the guard is a public API contract (CLI command, SDK entry point that
real users can hit with bad input), or if the source has a "don't regress" comment.

**Fix**: Delete. If the guard IS a public API contract, keep and name the test to signal
this is a contract test.

**Source**: SKILL (2) Rule 4


---

## P1 — High Severity

### AP6. Mock-Echo

**Detection**:
```python
mock.return_value = 42
result = function_under_test()
assert result == 42  # asserts the mock's own return value, no transformation in between
```

**Why it's a problem**: The test verifies that a mock returns what it was told to return. If the
function under test is a pass-through, this assertion holds for any implementation — zero
mutation-kill value.

**Exception** (keep conditions):
- The pass-through applies a rename, merge, default-fill, or conditional branch
- The assertion is negative (`!=`, `not in`) — an adversarial guard
- The assertion is on the call signature (`mock.assert_called_with(...)`) not the return value

**Fix**: Delete or replace with a call-signature assertion.

**Source**: SKILL (2) Rule 1


### AP7. Template-Echo

**Detection**: The test's expected value uses the same constant, template, or `**input_dict`
spread that the production code uses:

```python
# Production
EXPECTED = {"status": "ok", "data": transform(input)}
# Test
assert result == {"status": "ok", "data": transform(input)}  # duplicates same logic
```

**Why it's a problem**: If the production code's computation has a bug, the test replicates the
same bug and passes. The test and production are coupled at the hip.

**Exception**: Stable contracts external consumers depend on (API response shape, file-format
schema) may keep exact assertions.

**Fix**: Replace with targeted field assertions (`assert result["status"] == "ok"`) or use a
hardcoded known-correct value.

**Source**: SKILL (2) Rule 3


### AP8. Integration Test That Mocks the Integration

**Detection**: Test is marked as integration (`@pytest.mark.live`) but mocks ≥2 internal
modules under `src/`.

**Why it's a problem**: Integration tests should verify real cross-module interaction. Mocking
internal seams degrades the test to a unit test — it loses integration value.

**Exception**: All mocks are at external boundaries (third-party SDK, subprocess, network,
filesystem) → legitimate.

**Fix**: Remove seam mocks and use real modules; or remove the integration mark and treat it
as a unit test.

**Source**: SKILL (2) Rule 6


### AP9. Unjustified Skipped Test

**Detection**: `@pytest.mark.skip` or `it.skip` with no comment explaining why, or the skip
reason says "temporary" but has existed for over a month.

**Why it's a problem**: Skipped tests provide zero protection but still incur maintenance cost.

**Fix**: Add a skip reason or delete the test.

**Source**: SKILL (3) Red Flag Patterns


---

## P2 — Medium Severity

### AP10. Shallow Assertion

**Detection**: `toBeDefined`, `toBeTruthy`, `toHaveBeenCalled()` (no argument matchers),
`assert x is not None`.

**Why it's a problem**: Verifies "something is there" but not "it's the right thing."
Wrong values pass these assertions without detection.

**Fix**: Replace with specific value assertions (`expect(result).toBe(42)`) or qualified call
assertions (`expect(mock).toHaveBeenCalledWith({id: 123})`).

**Source**: SKILL (1) Dimension 1


### AP11. Over-Specified Forwarding Assertion

**Detection**:
```python
mock.assert_called_with(big_dict_with_20_fields)
# or
assert result == RunResult(field1=..., field2=..., ...)  # compares every field
```
When the function under test merely forwards data and the real downstream contract only
cares about 1–2 fields.

**Why it's a problem**: Any unrelated field addition or modification breaks the test, even
when the actual contract hasn't changed.

**Exception**: The function's core job IS assembling that data shape → exact assertions are valid.

**Fix**: Relax to field-subset assertions (`assert result.error is None`,
`assert result.success is True`) or use `mock.assert_called_once()` + targeted kwarg checks.

**Source**: SKILL (2) Rule 8


### AP12. Frozen-Fixture Assertion

**Detection**: `assert_equal [a, b], scope` or `expect(scope).to eq([a, b])` — exact collection
comparisons that break when unrelated fixtures add entries.

**Why it's a problem**: The test couples to unrelated fixture changes, producing false-positive
failures.

**Fix**: Use `assert_includes` / `expect(...).to include(...)` instead of exact collection
equality.

**Source**: SKILL (4) Red Flags


### AP13. Sleep / Hardcoded Delay

**Detection**: `sleep(N)`, `time.sleep(N)`, `setTimeout(N)` anywhere in test code.

**Why it's a problem**: Primary source of flaky tests. Under CI load, the delay may be
insufficient and the test fails randomly.

**Fix**: Replace with polling (`waitFor`, `eventually`), event-driven waiting, or mock timers.

**Source**: SKILL (4) Anti-Patterns


### AP14. Non-Deterministic Data

**Detection**: Test uses random values (`Math.random()`, `random.randint()`) without a fixed seed.

**Why it's a problem**: Tests fail sporadically and can't be reproduced, eroding trust in the
test suite.

**Fix**: Set a fixed seed or use parameterized fixed values.

**Source**: SKILL (4) Anti-Patterns


---

## P3 — Suggestion

### AP15. Setter/Getter Parade

**Detection**: A 5-line dataclass/config class with a separate test function for every field.

**Why it's a problem**: Noise. 15 tests covering a no-logic config class while the genuinely
complex function in the same module has zero coverage.

**Fix**: Collapse into one smoke test (construct + check all fields), freeing maintenance
budget for valuable tests.

**Source**: SKILL (2) Rule 2


### AP16. Parallel One-Liner Tests

**Detection**: 5+ tests with identical setup that differ only by one input value.

**Before collapsing, ask three questions:**
1. Do the cases exercise different branches/guards/filters in the source?
2. Do they target different constants/boundaries?
3. Would `case 3 failed` in a parametrized run be less debuggable than the named `def test_`?

If any answer is YES → keep separate. If ALL are NO → safe to collapse into
`@pytest.mark.parametrize` with `pytest.param(..., id="readable description")` so
failure output remains clear.

**Source**: SKILL (2) Rule 5


### AP17. Implementation-Coupled Test

**Detection**: Test verifies "`_internalHelper()` was called" rather than "the correct result
was returned." Refactoring internal implementation breaks the test even though behavior is
unchanged.

**Why it's a problem**: Tests coupled to implementation details block refactoring.

**Fix**: Rewrite as behavioral assertions — verify outputs/side-effects rather than internal
call chains.

**Source**: SKILL (4) Red Flags


### AP18. Overly Complex Setup

**Detection**: A single test's Arrange phase exceeds 20 lines, or requires multiple readings
to understand the context.

**Why it's a problem**: Complex setup signals a design issue (class under test has too many
responsibilities, too many dependencies) and the test itself is hard to maintain.

**Fix**: Extract test fixtures/factories. If the setup itself signals a design issue, flag as
"suggest refactoring the code under test."

**Source**: SKILL (4) Red Flags


### AP19. Undifferentiated Weight (Plan Review)

**Detection**: AI-generated test plan treats every test point as equally important. Core
business paths and edge cases sit side by side with no depth differentiation.

**Why it's a problem**: Limited testing budget gets spread uniformly; core paths end up under-
tested while low-impact edge cases consume disproportionate time.

**Fix**: Tag test points by business weight (HIGH/MEDIUM/LOW). Ensure HIGH-weight paths
have at least 3× the test depth of LOW-weight paths, and HIGH count is ≥40% of total.

**Source**: test-skill Check 2


---

## Keep-As-Is Signals

The following signals indicate a test should be **kept rather than flagged**:

| Signal | Meaning | Source |
|--------|---------|--------|
| Source has a "don't regress" comment referencing the test | Test guards against a known regression defect | SKILL (2) |
| Filename contains `invariants` / `lockdown` / `reexport` | Deliberately designed refactoring guard test | SKILL (2) |
| Uses real lightweight dependencies (`tmp_path`, real YAML parsing, real subprocess against local scripts) | Low mock-tautology risk | SKILL (2) |
| Can point to a specific `if` / `match` / `except` branch in the source that the test covers | Test has a clear coverage target | SKILL (2) |
| Encodes an ordered contract (filter chain, layered guard sequence, fallback order) | Test names are living documentation of the ordering | SKILL (2) |
| Adversarial / false-positive guard (verifies a pattern does NOT match) | Prevents over-eager regex or fuzzy-match logic from silently accepting dangerous input | SKILL (2) |
