# Test Analysis Workflow

## Overview

The `/test-analyze` skill performs comprehensive test quality analysis including coverage, assertion quality, isolation, determinism, and mutation testing concepts.

## Workflow Phases

### Phase 1: Parse Arguments

```
1. Parse command arguments:
   - mode flags: --coverage, --quality, --flaky, --mutation
   - path: target directory/file (default: current)
   - threshold: coverage threshold (default: from CLAUDE.md)
   - issue: linked issue number
   - output: markdown | json
   - verbose: include code snippets

2. If no mode specified: run all modes

3. Load configuration:
   - CLAUDE.md for thresholds and exclusions
   - Test Plan from issue (if --issue provided)
```

### Phase 2: Discover Tests

```
1. Find test files by convention:

   JavaScript/TypeScript:
   - **/*.test.{js,ts,jsx,tsx}
   - **/*.spec.{js,ts,jsx,tsx}
   - **/__tests__/**/*.{js,ts,jsx,tsx}

   Python:
   - **/test_*.py
   - **/*_test.py
   - **/tests/**/*.py

   Go:
   - **/*_test.go

   Java:
   - **/Test*.java
   - **/*Test.java
   - **/*Tests.java

2. Find test configuration:
   - jest.config.{js,ts,json}
   - pytest.ini, pyproject.toml, setup.cfg
   - go.mod
   - pom.xml, build.gradle

3. Map test files to source files:
   - By naming convention (foo.ts → foo.test.ts)
   - By import analysis
   - By directory structure
```

### Phase 3: Load Coverage Data

```
1. Search for local coverage files:
   PRIORITY ORDER:
   - coverage/lcov.info
   - coverage/coverage-final.json
   - coverage/coverage-summary.json
   - coverage.xml (Cobertura)
   - cover.out (Go)
   - .coverage (Python)

2. IF not found AND CI configured:
   a. Determine CI platform from CLAUDE.md
   b. Get latest successful workflow/pipeline
   c. Download coverage artifact
   d. Extract and locate coverage file

3. Parse into unified format:
   {
     "summary": {
       "lines": { "covered": N, "total": N, "pct": X },
       "branches": { "covered": N, "total": N, "pct": X },
       "functions": { "covered": N, "total": N, "pct": X }
     },
     "files": {
       "path/to/file": {
         "lines": { ... },
         "branches": { ... },
         "uncovered_lines": [line_numbers]
       }
     }
   }

4. IF no coverage found:
   Report warning: "No coverage data. Run tests with --coverage flag."
   Continue with quality analysis only.
```

### Phase 4: Coverage Analysis

```
IF --coverage mode:

1. Compare against thresholds:
   - Global thresholds from CLAUDE.md
   - Test Plan thresholds (if issue linked)

2. Identify gaps:
   - Files with 0% coverage
   - Files below threshold
   - Critical paths without coverage (from Test Plan)

3. Prioritize gaps:
   PRIORITY = risk_score * (threshold - current_coverage)

   risk_score factors:
   - In critical path: +50
   - Recently changed: +30
   - High complexity: +20
   - Security-related: +40

4. Calculate trend (if historical data):
   - Compare to previous analysis
   - Identify improving/declining areas
```

### Phase 5: Quality Analysis

```
IF --quality mode:

FOR each test file:
  FOR each test function:
    1. Count and classify assertions:

       STRONG patterns:
       - expect(x).toEqual(specificValue)
       - expect(x).toBe(specificValue)
       - assertEquals(expected, actual)
       - assert x == expected
       - expect(fn).toHaveBeenCalledWith(args)

       WEAK patterns:
       - expect(x).toBeDefined()
       - expect(x).toBeTruthy()
       - assertNotNull(x)
       - assert x is not None

       TRIVIAL patterns:
       - expect(true).toBe(true)
       - pass
       - No assertions in test body

    2. Calculate assertion score:
       score = (strong * 1.0 + weak * 0.5 + trivial * 0.1) / total * 100

    3. Detect test smells:
       - Logic in tests (if/else, loops)
       - Excessive mocking (>5 mocks)
       - Testing private methods
       - Duplicate test code
       - Missing assertions
       - Overly complex setup

    4. Assess naming quality:
       Good: "should return error when user not found"
       Bad: "test1", "testFunction", "it works"
```

### Phase 6: Isolation Analysis

```
IF --quality mode:

FOR each test file:
  1. Detect shared state:
     - Global variable modifications
     - Class-level mutable state
     - Database connections without cleanup
     - File system operations without cleanup

  2. Detect test dependencies:
     - Tests referencing data from other tests
     - Ordered test execution requirements
     - beforeAll/afterAll that affects test isolation

  3. Flag issues:
     - Shared mutable state
     - Missing cleanup/teardown
     - Cross-test data leakage
```

### Phase 7: Determinism Analysis

```
IF --flaky mode:

FOR each test file:
  1. Detect timing sensitivity:
     PATTERNS:
     - sleep(), time.sleep(), delay()
     - setTimeout in assertions
     - Date.now(), new Date() in assertions
     - performance.now() comparisons

  2. Detect environment sensitivity:
     PATTERNS:
     - File system reads/writes
     - Network calls without mocks
     - Environment variable reads
     - Process/OS-specific code

  3. Detect randomness:
     PATTERNS:
     - Math.random() in assertions
     - UUID generation in expected values
     - Shuffled collections in assertions

  4. Score determinism:
     Start at 100, subtract:
     - Timing dependency: -15 each
     - Environment dependency: -10 each
     - Randomness: -20 each
     - Shared state: -10 each
```

### Phase 8: Mutation Analysis

```
IF --mutation mode:

WITHOUT running actual mutations, analyze:

1. Assertion strength:
   FOR each assertion:
     - Would negating condition still pass?
     - Would changing operator (<, >, ==, !=) still pass?
     - Is expected value specific enough?

   Flag weak assertions:
   - expect(x > 0) // x = 1 and x = 1000000 both pass
   - expect(arr.length).toBe(arr.length) // tautology
   - expect(result).toBeTruthy() // many values pass

2. Dead code detection:
   - Code paths never executed in tests
   - Branches never taken
   - Error handlers never triggered

3. Boundary analysis:
   - Are min/max values tested?
   - Are off-by-one scenarios covered?
   - Are null/undefined cases tested?
   - Are empty collection cases tested?
```

### Phase 9: Compile Results

```
1. Calculate overall quality score:
   quality_score = (
     coverage_score * 0.30 +
     assertion_score * 0.30 +
     isolation_score * 0.20 +
     determinism_score * 0.20
   )

   WHERE:
   - coverage_score = min(100, (current_coverage / target) * 100)
   - assertion_score = (strong_pct * 100)
   - isolation_score = 100 - (isolation_issues * 10)
   - determinism_score = calculated in Phase 7

2. Prioritize recommendations:
   SORT BY:
   - Impact (coverage gap * risk)
   - Effort (estimated fix complexity)
   - Category (coverage > quality > style)

3. Determine pass/fail:
   FAIL if:
   - Coverage below threshold
   - Critical paths not covered
   - Quality score below 50

   WARN if:
   - Quality score below 70
   - Flaky tests detected
   - Many weak assertions
```

### Phase 10: Report Results

```
1. Format output (markdown or json)

2. Include:
   - Summary metrics
   - Coverage table
   - Quality breakdown
   - Critical gaps
   - Flaky candidates
   - Recommendations

3. IF issue linked:
   - Compare against Test Plan requirements
   - Note which critical paths are covered
   - Flag missing Test Plan items

4. Store for trending:
   WRITE to .claude/test-analysis/{date}-{scope_hash}.json
   {
     "timestamp": "...",
     "scope": "...",
     "issue": N or null,
     "coverage": {...},
     "quality": {...},
     "recommendations": [...]
   }
```

## Integration with Crunch Workflow

### TEST-VERIFICATION Gate

When invoked from `/crunch` before code review:

```
1. Load Test Plan from issue body

2. Run coverage analysis:
   /test-analyze -c -i {issue_number}

3. Check results against Test Plan:
   - Coverage meets Test Plan threshold?
   - Critical paths (P0) have tests?
   - Quality score >= 60?

4. Gate decision:
   IF coverage_pass AND critical_paths_covered:
     PASS → proceed to code review
   ELSE:
     FAIL → report gaps, stay in IMPLEMENTING

5. Report to user:
   "TEST-VERIFICATION: {PASS|FAIL}
    Coverage: {X}% (target: {Y}%)
    Critical Paths: {covered}/{total}
    Quality Score: {score}/100

    {gaps if FAIL}"
```

### Invocation from Review

The Test Quality Reviewer (Agent #6) can invoke this skill:

```
1. Get changed files from PR diff
2. Run: /test-analyze -q {changed_test_files}
3. Include findings in review output
```

## Error Handling

| Condition                | Action                                  |
| ------------------------ | --------------------------------------- |
| No tests found           | Report warning, suggest test locations  |
| No coverage data         | Continue with quality analysis, warn    |
| CI artifacts unavailable | Fall back to local analysis             |
| Parse error              | Report specific error, continue partial |
| Threshold not configured | Use defaults (80% line, 75% branch)     |

## Performance Considerations

1. **Parallelize**: Run coverage and quality analysis in parallel
2. **Cache**: Store parsed coverage for reuse in same session
3. **Limit scope**: When path specified, only analyze that subtree
4. **Skip unchanged**: When issue linked, focus on changed files
