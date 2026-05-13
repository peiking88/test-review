---
name: test-review
description: >-
  Review test plans and test cases across multiple quality dimensions. Evaluates coverage
  completeness, assertion depth, boundary conditions, error handling, mutation-kill strength,
  invariant protection, anti-patterns, and external dependency failure scenarios. Supports
  three modes: single-file deep review with scorecard, branch/PR multi-agent parallel review,
  and AI-generated test plan walkthrough. Use when the user mentions "review tests",
  "test walkthrough", "are these tests good", "check test quality", "test plan review",
  "/test-review", "评审测试", "测试走查", "用例评审", or wants to evaluate test quality.
argument-hint: "[test-file | branch | commit-range | uncommitted | PR-number | plan-file]"
user-invocable: true
---

# Test Review Skill

Multi-dimensional walkthrough and review of test plans and test cases. Produces structured
reports with actionable improvement recommendations.

## Mode Selection

The mode is selected automatically based on `$ARGUMENTS`:

| Condition | Mode | Description |
|-----------|------|-------------|
| Path to a test plan / test point list (`.md`, `.csv`, `.xlsx`, or paste) | **Plan Review** | Analyze AI-generated test points for dependency-failure gaps, weight distribution, and historical bug coverage |
| Single test file path | **Quick Mode** | Deep review of one test file with 7-dimension scorecard |
| Branch / PR / commit range / "uncommitted" | **Full Mode** | Multi-agent parallel review with triage-based agent dispatch |
| Empty (no argument) | **Auto-detect** | Check for open PR first, then count changed files: ≤5 test files → Quick, >5 → Full |

---

## Mode 1: Plan Review (AI-Generated Test Points)

When the user provides AI-generated test points, a test plan document, or a list of test scenarios,
apply the three checks below. Do NOT grade on the 7 dimensions — this mode answers the question:
**"What did the AI miss?"** not "How good is the test code?"

### Step 1: Identify the Test Plan

The test plan may come as:
- A file path (`docs/test-plan.md`, `specs/test-points.csv`)
- Pasted content in the user's message
- An auto-detected document (glob `**/test-plan*`, `**/test-points*`, `**/*.csv` with "test point" headers)

Extract all test points into a flat list for review. If the input is a CSV, read its columns.

### Step 2: Check 1 — External Dependency Failure Coverage

AI-generated test points consistently underrepresent infrastructure failure scenarios.
The model's training biases it toward functional specifications, not dependency failures.

**Actions:**
1. List every external dependency the system under test touches (database, cache, message queue,
   third-party API, file system, external service).
2. For each dependency, check whether the test plan covers three failure modes:
   - **Unavailable**: Dependency is down / unreachable
   - **Slow**: Dependency times out or exceeds latency budget
   - **Corrupt**: Dependency returns malformed, incomplete, or stale data
3. Record every dependency + failure-mode combination that is missing.

**Real-world baseline**: Teams typically find 0% dependency-failure coverage in AI-generated test
point lists. Treat any detected gap here as **blocker** severity.

### Step 3: Check 2 — Business Weight Distribution

AI assigns equal importance to all test points. In reality, core business paths need deeper
coverage than edge cases. **Target: core-path test depth should be at least 3× that of edge paths.**

**Actions:**
1. Identify the system's core business paths. Look for signals: monitoring alerts that fire first,
   revenue-critical flows, most frequently invoked endpoints.
2. Tag each test point as **HIGH** (core path), **MEDIUM** (regular), or **LOW** (edge case).
3. Calculate the ratio: HIGH count / total count. If below **40%**, the core path is
   under-tested — not because AI gave too few, but because it gave uniformly shallow coverage.
4. For HIGH-weight paths specifically: check whether each has coverage at these depths:
   - Happy path variation (different valid inputs)
   - Input boundary (0, -1, MAX, empty)
   - Error injection (dependency failure, bad input)
   - State transition (before/after, idempotency)

**If the user hasn't specified what counts as core paths**, note this in the report's
"Questions for the Author" section and skip the depth check — do not guess.

### Step 4: Check 3 — Historical Bug Cross-Reference

AI has no knowledge of the team's past incidents. Historical P0/P1 bugs are a gold standard
for what the test plan SHOULD cover, because each one represents a scenario that actually
broke production. **Industry data: AI-generated test points cover only 23-41% of historical
bug scenarios.**

**Actions:**
1. Search for historical bug records in the project:
   - `**/BUGS.md`, `**/known-issues.md`, `**/incidents/*.md`
   - `gh issue list --label "bug" --limit 50` (if in a GitHub repo)
   - Ask the user if no structured record exists
2. For each P0/P1 bug with a clear triggering scenario, check: is there a test point in the plan
   that would have caught it?
3. Calculate: covered / total historical bugs. Report the percentage.
4. For bugs NOT covered: classify them (dependency failure? concurrency? state-machine gap?
   special business rule?). This classification is the foundation for the team's
   **test-point supplement rulebook**.

**If no historical bug data is available**, note the gap and recommend the team start recording
a "bug scenario → missing test" mapping. Skip the check rather than fabricate results.

### Step 5: Output

Use the **Plan Review template** in `references/report-format.md`.

---

## Mode 2: Quick Mode (Single-File Deep Review)

Deep walkthrough of a single test file. Scoring rubrics are in `references/dimensions.md`.

### Step 1: Identify Target

If a test file was specified, read it. Otherwise, discover via Glob
(`**/*.test.*`, `**/*.spec.*`, `**/__tests__/**`) and pick the most recently modified one.

### Step 2: Read the Source Under Test

Infer the production code location from the test file's imports, naming convention, or
directory structure. Read both. **You cannot evaluate a test without understanding the
contract of the code it exercises.**

### Step 3: Evaluate Each Dimension

Score each dimension (A+ to F) using the rubrics in `references/dimensions.md`:

| Dimension | Core Question |
|-----------|--------------|
| **Assertion Depth** | Are assertions on specific values or just existence/truthiness? |
| **Input Coverage** | Are canonical, empty, boundary, invalid, and dependency-failure cases covered? |
| **Error Testing** | For each throwable/rejectable path in the source, is there a corresponding error test? |
| **Mock Health** | Are mocks confined to external boundaries or leaking into internal modules? |
| **Specification Clarity** | Can the full contract be understood by reading only test names? |
| **Independence** | Can tests run in any order without affecting each other? |
| **Invariant Protection** | Do tests encode architectural boundaries, data constraints, and design invariants? |

Every finding must cite a specific line number.

### Step 4: Anti-Pattern Detection

Check for common anti-patterns (full catalog in `references/anti-patterns.md`):
shallow assertions, mock-echo, template-echo, unreachable guard tests, sleep/hardcoded delays,
vendor smoke tests, unjustified skipped tests, frozen-fixture assertions, and invariant erosion.

### Step 5: Output

Use the **Quick Mode template** in `references/report-format.md`.

---

## Mode 3: Full Mode (Multi-Agent Parallel Review)

For reviewing changes on a branch, PR, or commit range. Inherits the parallel-agent
architecture from the original test-reviewer skill.

### Step 1: Determine Review Scope

Based on argument type:
1. **Branch name**: review changes on that branch vs. the base branch
2. **Commit range** (`abc123..def456`): review that range
3. **"uncommitted"**: review working-tree changes (`git diff HEAD`)
4. **PR number/URL**: fetch PR details and diff

When no argument is given, auto-detect:
```bash
gh pr list --head $(git branch --show-current) --json number,title,body,baseRefName,url --limit 1
```

### Step 2: Detect Base Branch

1. PR: use `baseRefName` from `gh pr view`
2. Branch without PR: default to `develop`
3. Commit range / uncommitted: not needed

### Step 3: Gather Context

```bash
git diff {base}...HEAD --name-only
git diff {base}...HEAD
git log {base}..HEAD --oneline
```

Store: `DIFF`, `CHANGED_FILES`, `COMMIT_LOG`, `PR_DESCRIPTION`, `REVIEW_SCOPE`.

### Step 4: Filter

Skip generated code (`generated-sources/`, `generated-test-sources/`, generated parser code, etc.).

### Step 5: Categorize Changes

Scan the full diff and assign categories to every changed file:

| Category | Signals |
|----------|---------|
| **storage-engine** | `storage/`, `cache/`, `wal/`, page read/write, `DiskStorage` |
| **concurrency** | `synchronized`, `Lock`, `Atomic*`, `volatile`, thread pools, `ConcurrentHashMap` |
| **crash-durability** | WAL ops, crash simulation, transaction atomicity, page corruption |
| **index-data-structures** | `index/`, B-tree, hash index, `SBTree` |
| **serialization** | Record serializers, binary format |
| **sql-query** | `sql/` (excluding `parser/`), query execution |
| **network-server** | `server/`, `driver/`, protocol handling, authentication |
| **public-api** | `api/`, public interfaces |
| **config-build** | `pom.xml`, CI config |
| **docs-only** | Markdown/comment-only changes |

### Step 6: Select and Dispatch Agents

| Agent | Launch Condition |
|-------|-----------------|
| **review-test-behavior** | Always (unless only docs/config-build) |
| **review-test-completeness** | Always (unless only docs/config-build) |
| **review-test-structure** | Any test files changed |
| **review-test-concurrency** | `concurrency` category detected |
| **review-test-crash-safety** | `crash-durability` category detected |

Log the triage summary, then launch all selected agents **in parallel**
(`subagent_type` = agent name, `model` = `opus`):

```
Review the following code changes from your specialized test quality perspective.

## Review Scope
{REVIEW_SCOPE}

## PR Description
{PR_DESCRIPTION}

## Commit Log
{COMMIT_LOG}

## Changed Files
{CHANGED_FILES}

## Skip (generated code)
[List file patterns to skip]

## Diff
{DIFF}
```

### Step 7: Synthesize

After all agents complete:
1. **Deduplicate**: merge the same issue flagged by multiple agents, note all dimensions
2. **Prioritize**: blocker → should-fix → suggestion
3. **Attribute**: each finding labels which dimension(s) caught it

Use the **Full Mode template** in `references/report-format.md`.

---

## Core Principles

These apply across all three modes:

1. **Understand before judging**: Always read both test code AND production code. Without
   understanding the contract, you cannot evaluate the test.
2. **Be specific**: Every finding must cite a concrete line number and scenario. Not
   "needs more boundary tests" but "missing test for `email` being empty — should verify
   `ValidationError` is thrown."
3. **False positives over false negatives**: Marking a non-issue is less harmful than
   missing a real problem. When in doubt, flag it.
4. **Mutation thinking**: Ask "if I change this line of production code, will any test fail?"
   A test that passes against any plausible implementation is zero-value.
5. **Conservative defaults**: Tests in files named `invariants`, `lockdown`, `reexport`, or
   those referenced by a "don't regress" source comment, should be kept by default.
6. **Don't modify code**: Report findings and suggestions only. Human review decides what to change.
7. **Dependency-failure mindset**: Production outages are more often caused by dependency
   failures than by functional defects. Always check whether failure modes of external
   dependencies are covered.

## Reference Files

- `references/dimensions.md` — Scoring rubrics for all 7 dimensions, including external
  dependency failure coverage and core-path depth ratio
- `references/anti-patterns.md` — Full catalog of test anti-patterns with detection rules
  and fix recommendations
- `references/report-format.md` — Templates for all three modes (Plan Review, Quick, Full)
