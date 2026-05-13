# Report Templates

---

## Plan Review Template

Used when reviewing AI-generated test points / test plans:

```
## Test Plan Review: {plan-name-or-description}

### Overall Assessment
[2–3 sentences: is this plan production-ready? What are the biggest blind spots?]

### Check 1: External Dependency Failure Coverage

**Dependencies identified**: {list of external dependencies}

| Dependency | Unavailable | Timeout | Corrupt Data | Status |
|-----------|-------------|---------|--------------|--------|
| {dep} | {covered/missing} | {covered/missing} | {covered/missing} | {OK/Gap} |

**Gap summary**: {N} missing failure scenarios across {M} dependencies.

### Check 2: Business Weight Distribution

**Core paths identified**: {list, or "Not specified — see Questions for the Author"}

| Weight | Count | Percentage |
|--------|-------|------------|
| HIGH | {N} | {P}% |
| MEDIUM | {N} | {P}% |
| LOW | {N} | {P}% |

**Depth ratio**: Core path avg depth = {X} scenarios per point, Edge path = {Y}.
Target is ≥3:1. Current ratio = {X/Y}:1.

{If HIGH < 40%: "Core paths are underrepresented. Not because too few were generated,
but because AI gave uniform shallow coverage. Deepen HIGH-weight paths specifically."}

### Check 3: Historical Bug Cross-Reference

**Data source**: {BUGS.md / gh issues / not available}

| Bug ID | Scenario | Covered by Plan? | Test Point |
|--------|----------|-----------------|-------------|
| {id} | {scenario} | {Yes/No} | {ref or "—"} |

**Coverage rate**: {N}/{M} = {P}%

{If <50%: "Below the 50% threshold. These gaps represent scenarios that have actually
broken production before — each one is a known risk."}

### Gap Classification
[Group uncovered bugs by type: dependency failure, concurrency, state-machine gap,
special business rule, etc. This becomes the team's supplement rulebook.]

### Priority Actions

1. **Blocker**: [Missing dependency-failure scenarios]
2. **Should-Fix**: [Core-path depth gaps]
3. **Suggestion**: [Historical bug scenarios to add]

### Questions for the Author
[What core paths should be tagged HIGH? Where are historical bug records?]
```

---

## Quick Mode Template

Used for single-file deep review:

```
## Test Review — {test-file-name}

**Composite Grade: {letter}** — {one-line summary}

### Scorecard

| Dimension | Score | Finding |
|-----------|-------|---------|
| Assertion Depth | {A+–F} | {specific finding + count} |
| Input Coverage | {A+–F} | {missing input categories} |
| Error Testing | {A+–F} | {gap count: N error paths / M tested} |
| Mock Health | {A+–F} | {assessment: boundary/seam/ghost counts} |
| Specification Clarity | {A+–F} | {assessment} |
| Independence | {A+–F} | {assessment} |
| Invariant Protection | {A+–F} | {assessment} |

### Blockers
[Invariant erosion, ghost mocks, etc. Each with file:line]

1. **{file}:{line}** — {issue}
   **Fix**: {concrete fix}

### Should-Fix
[Missing boundary conditions, weak assertions, missing error paths]

1. **{file}:{line}** — {issue}
   **Fix**: {concrete fix}

### Suggestions
[Naming improvements, optional edge cases, parametrization candidates]

### Missing Tests
[Most important tests to add, with concrete input values and expected outputs]

1. **Missing: {scenario category}** — {specific scenario}
   Suggested test: {concrete test case with input and expected output}
```

---

## Full Mode Template

Used for branch/PR multi-agent parallel review:

```
## Test Quality Review: {REVIEW_SCOPE}

### Triage Summary
- **Categories detected**: {category list}
- **Agents launched**: {agent list}
- **Agents skipped**: {agent list + reason}

### Overall Assessment
[2–3 sentences: are the tests meaningful and thorough? What are the main gaps?]

### Blockers
[Tests that give false confidence, or missing tests for dangerous code paths]

1. **[Dimension]** `path/to/file.ext` (line X–Y)
   - **Issue**: ...
   - **Suggestion**: ...

### Should-Fix
[Missing corner cases for critical code, weak assertions that could hide bugs]

1. **[Dimension]** `path/to/file.ext` (line X–Y)
   - **Issue**: ...
   - **Suggestion**: ...

### Suggestions
[Recommended improvements: test data quality, readability, additional scenarios, naming]

1. **[Dimension]** `path/to/file.ext` (line X–Y)
   - **Issue**: ...
   - **Suggestion**: ...

### Anti-Patterns Detected
[Anti-patterns found, referencing `references/anti-patterns.md` IDs]

1. **[AP{N}]** `path/to/file.ext` (line X–Y)
   - **Issue**: {anti-pattern description}
   - **Fix**: {suggestion}

### Questions for the Author
[Clarifying questions aggregated from all reviewers]
```

---

## Severity Level Definitions

| Level | Definition | Example |
|-------|-----------|---------|
| **Blocker** | Tests that give false confidence, or dangerous code paths with zero coverage | Ghost mock, invariant erosion, core security path has no error test, dependency-failure scenario missing for production dependency |
| **Should-Fix** | Weak assertions that could hide bugs, or missing boundary conditions for critical code | Missing error-path test, assertion too shallow, critical boundary uncovered, core-path test depth below 3:1 |
| **Suggestion** | Nice-to-have improvements that don't affect current functional verification | Naming improvement, parametrization candidate, optional additional scenario |
