# Test Review Dimensions

Seven dimensions for evaluating test quality. Each uses letter grades A+ through F
with calibrated descriptors.

---

## Dimension 1: Assertion Depth

**Core question: Do assertions check specific values or merely existence/truthiness?**

Shallow assertions verify "is there something?" Deep assertions verify "is it the RIGHT thing?"
Every shallow assertion is a potential false-pass — production returns a wrong value but the
test passes because it only checked for non-null.

### Scoring

| Grade | Criteria |
|-------|----------|
| **A** | All assertions are specific value assertions; each covers an independent behavioral dimension |
| **B** | Vast majority are value assertions; occasional existence checks used appropriately (e.g., verifying only that a side-effect occurred) |
| **C** | 2–3 shallow assertions present (`toBeDefined`, `toBeTruthy`, unqualified `toHaveBeenCalled`), but not on core verification paths |
| **D** | Core verification paths use shallow assertions; missing value comparisons could let bugs through |
| **F** | Majority of assertions only check existence/truthiness; tests barely verify behavior |

### Detection

Count occurrences of:
- `expect(x).toBeDefined()` / `assert x is not None` — existence-only
- `expect(x).toBeTruthy()` / `assert x` — truthiness-only
- `mock.assert_called()` without arguments — called-but-with-what?
- `expect(x).toHaveBeenCalled()` without parameter matchers

Each counts as one deduction from A.

---

## Dimension 2: Input Coverage

**Core question: Does coverage span canonical, empty, boundary, invalid, and dependency-failure cases?**

Happy-path-only tests give false confidence. Real-world inputs are diverse and hostile.

### Input Space Categories

For each parameter of the function under test, check coverage across:

| Category | Description | Priority |
|----------|-------------|----------|
| **Canonical** | Normal, typical inputs | Mandatory |
| **Empty** | `null`/`None`, `""`, `[]`, `{}` | Mandatory |
| **Boundary** | `0`, `-1`, `MAX_INT`, `MAX_LEN`, just-beyond | Mandatory |
| **Invalid** | Wrong type, malformed format, business-rule violation | Mandatory |
| **Adversarial** | Deliberately crafted malicious input (SQL injection, XSS, path traversal) | Recommended |
| **Dependency failure** | Each external dependency: unavailable, timeout, corrupt response | Mandatory |

### External Dependency Failure Coverage (NEW)

This is the single most common gap in AI-generated test points. For every external dependency
the system under test touches, check whether the test plan covers three failure modes:

| Failure Mode | Example Scenario |
|-------------|-----------------|
| **Unavailable** | Database connection refused, Redis is down, third-party API returns 503 |
| **Slow / Timeout** | Database query takes 4.3s, cache responds after timeout, API hangs |
| **Corrupt / Stale** | API returns malformed JSON, cache returns wrong version, DB returns partial rows |

**Real-world baseline**: Teams typically find 0 out of N dependency-failure scenarios covered
in AI-generated test plans. If ANY dependency-failure mode is missing for a production
dependency, that is a **blocker**.

### Scoring

| Grade | Criteria |
|-------|----------|
| **A** | All params covered: canonical + empty + boundary + at least one error case + dependency failures for every external dependency |
| **B** | All params covered: canonical + empty, most have boundary tests, major dependencies have failure coverage |
| **C** | All params have canonical, most have empty, boundary or dependency-failure tests are missing |
| **D** | Happy-path only; empty, boundary, or dependency failures clearly omitted |
| **F** | Only ideal inputs tested; not even basic error paths |

---

## Dimension 3: Error Testing

**Core question: For every throwable/rejectable path in the source, is there a corresponding error test?**

If the source has 5 `throw`/`raise`/`reject` paths but tests only cover 1, the remaining 4
error behaviors are unverified.

### Detection

1. Walk the source code; count all `throw`/`raise`/`reject`/error-return paths
2. Count how many of these paths have a corresponding test
3. Ratio below 1:1 is a gap
4. Check whether error assertions are specific (type + message) or generic (bare `.toThrow()`)

### Scoring

| Grade | Criteria |
|-------|----------|
| **A** | All error paths tested; assertions verify both error type and key message content |
| **B** | All error paths tested, but some assertions only check type, not message |
| **C** | 1–2 error paths missing, or most assertions are bare `.toThrow()` |
| **D** | Multiple error paths missing; error assertions are generic |
| **F** | No error tests at all, or all error assertions are bare `.toThrow()` |

---

## Dimension 4: Mock Health

**Core question: Are mocks confined to external boundaries or leaking into internal modules?**

The deeper the mocking, the more fragile the test. Good mocks sit at system boundaries
(HTTP, database, filesystem, third-party SDKs). Bad mocks infiltrate internal modules —
the test becomes "verify the mock configuration" rather than "verify real behavior."

### Classification

Classify each mock target:
- **Boundary**: third-party SDK, subprocess, network, filesystem, database driver — mocking is legitimate
- **Seam**: internal module under `src/` — mocking is suspect
- **Ghost**: mock path resolves to a nonexistent attribute or module — dangerous, needs immediate fix

### Scoring

| Grade | Criteria |
|-------|----------|
| **A** | All mocks at boundaries, or no mocks needed (real lightweight dependencies used) |
| **B** | Very few seam mocks with clear justification (e.g., avoiding re-testing already-covered module) |
| **C** | Some seam mocks but not out of control; mock setup lines < assertion lines |
| **D** | Many seam mocks; mock setup code exceeds assertion code |
| **F** | Mock chain goes to the leaf (every downstream call mocked); ghost mocks present |

### Special Checks

- Integration test marked `@pytest.mark.live` mocking ≥2 internal seams → demote; it's not a real integration test
- `patch("x.y", create=True)` against a nonexistent symbol → **blocker** (ghost mock)

---

## Dimension 5: Specification Clarity

**Core question: Can the full contract be understood by reading only test names?**

Good test names are living documentation. Reading all test names in sequence should
reconstruct the module's complete behavioral specification.

### Evaluation

- Do names describe specific behavior? ("rejects negative order amount with minimum-value error") vs. vague ("test_error")
- Is there a consistent naming convention? (`should_<behavior>_when_<condition>`)
- Is the Given/When/Then or Arrange/Act/Assert structure visible?

### Scoring

| Grade | Criteria |
|-------|----------|
| **A** | Test names form a complete behavioral spec; no need to read test bodies to understand the contract |
| **B** | Most names are clear; a few are vague |
| **C** | Roughly one-third of names are vague or use technical jargon instead of business language |
| **D** | Over half of names are `test1`, `test_error`, `it_works` style |
| **F** | Test names carry almost no information |

---

## Dimension 6: Independence

**Core question: Can tests run in any order without affecting each other?**

Order-dependent tests cannot be parallelized, and one test's failure cascades to downstream
tests, wasting debugging time.

### Detection Signals

- Global mutable state (module-level variables, static class fields)
- Shared state not cleaned up in `beforeEach`/`teardown` (database, files, cache)
- Tests communicating implicitly through filesystem or environment variables
- Does `beforeEach` properly reset state, or does it accumulate?

### Scoring

| Grade | Criteria |
|-------|----------|
| **A** | All tests fully independent; `beforeEach` resets correctly; no shared mutable state |
| **B** | Shared read-only fixtures; no side effects |
| **C** | Some shared state but `beforeEach` cleans up; occasional omissions |
| **D** | Known execution-order dependencies; random-order runs would fail |
| **F** | Cannot run in any order; `beforeEach` has accumulation effects |

---

## Dimension 7: Invariant Protection

**Core question: Do tests encode important design decisions as unbreakable constraints?**

Some tests don't just verify behavior — they encode architectural constraints. A test
asserting "module A never imports from module B" encodes a layering boundary. A test
asserting "this function has no side effects" encodes a concurrency model. These tests
have value far beyond what coverage metrics capture.

### Invariant Types

| Type | Example | Signal |
|------|---------|--------|
| **Layer boundary** | Domain layer never imports infrastructure layer | Import-check test |
| **Data constraint** | Order amount must be > 0 | Validation test |
| **Concurrency model** | This function is pure (no side effects) | Side-effect check |
| **API contract** | Return value must be an immutable type | Type check |

### Scoring

| Grade | Criteria |
|-------|----------|
| **A** | Key design invariants all have corresponding tests; test failure unambiguously signals invariant violation |
| **B** | Most invariants tested; a few boundaries lack coverage |
| **C** | Some invariant tests exist but not comprehensive |
| **D** | No invariant tests |
| **F** | Invariant tests exist but have been weakened (skip, relaxed assertion) without replacement |

### Invariant Erosion Alerts

Mark as **blocker** when:
- `skip` was added to a previously passing test → invariant being silently dropped
- Assertion changed from specific value to broad match → constraint being relaxed
- Test renamed to describe new behavior → old invariant erased from history
- Test deleted "because it tested old code" → invariant removed without replacement

---

## Core-Path Depth Ratio (Plan Review)

When reviewing test plans (not code), apply the **3:1 depth principle**:

> Core business paths should have at least 3× the test depth of edge paths.

**Depth tiers for core paths:**
1. Happy path variation (different valid inputs)
2. Input boundary (0, -1, MAX, empty)
3. Error injection (dependency failure, bad input)
4. State transition (before/after, idempotency)

**Weight distribution target**: HIGH-weight test points should be ≥40% of total.

This is a guideline, not a hard scoring dimension — report violations in the "Should-Fix"
section with the concrete ratio.

---

## Composite Grade

Weighted calculation across the 7 dimensions:

| Dimension | Weight |
|-----------|--------|
| Assertion Depth | 20% |
| Input Coverage | 20% |
| Error Testing | 20% |
| Mock Health | 10% |
| Specification Clarity | 10% |
| Independence | 10% |
| Invariant Protection | 10% |

Composite mapping: A+(≥95) → A(≥85) → B(≥75) → C(≥60) → D(≥45) → F(<45)

Grade meaning:
- **A**: Excellent — can serve as a team benchmark
- **B**: Good — minor room for improvement
- **C**: Functional but has gaps that could let bugs through
- **D**: Needs work — critical-path testing is insufficient
- **F**: Unacceptable — tests provide no meaningful quality assurance
