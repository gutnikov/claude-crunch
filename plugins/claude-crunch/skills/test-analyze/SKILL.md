---
name: test-analyze
description: Analyze test quality, coverage, and identify testing gaps. Evaluates assertion meaningfulness, test isolation, determinism, and provides actionable recommendations.
---

# Test Analysis Skill

Perform comprehensive test quality analysis including coverage gaps, assertion quality, and flaky test detection.

## Syntax

```
/test-analyze [options] [path]
```

### Parameters

| Parameter | Short | Description |
|-----------|-------|-------------|
| `--coverage` | `-c` | Focus on coverage analysis |
| `--quality` | `-q` | Focus on test quality assessment |
| `--flaky` | `-f` | Focus on flaky test detection |
| `--mutation` | `-m` | Apply mutation testing concepts |
| `--threshold` | `-t` | Coverage threshold to check (default: from CLAUDE.md) |
| `--issue` | `-i` | Link analysis to issue number |
| `--output` | `-o` | Output format: `markdown` (default) or `json` |
| `--verbose` | `-v` | Include detailed findings with code snippets |

### Path Argument

Optional path to analyze. Defaults to current directory.

```
/test-analyze                     # Analyze entire project
/test-analyze src/auth/           # Analyze specific module
/test-analyze src/auth/handler.ts # Analyze specific file
```

## Analysis Modes

### Coverage Analysis (`--coverage` or `-c`)

Analyzes test coverage and identifies gaps.

```
/test-analyze -c
/test-analyze -c --threshold 85
/test-analyze -c src/core/
```

**Output includes**:
- Current line and branch coverage percentages
- Uncovered files and functions
- Critical path coverage assessment
- Coverage trend (if historical data available)
- Gap prioritization by risk

### Quality Assessment (`--quality` or `-q`)

Evaluates test quality beyond coverage numbers.

```
/test-analyze -q
/test-analyze -q src/auth/
```

**Output includes**:
- Assertion meaningfulness score (0-100)
- Test isolation assessment
- Test naming quality
- Setup/teardown patterns
- Test smell detection

### Flaky Test Detection (`--flaky` or `-f`)

Identifies tests with non-determinism indicators.

```
/test-analyze -f
/test-analyze -f --verbose
```

**Output includes**:
- Tests using sleep/delay
- Tests with timing sensitivity
- Tests with shared state
- Tests with external dependencies
- Tests with date/time sensitivity

### Mutation Testing Concepts (`--mutation` or `-m`)

Applies mutation testing principles without running actual mutations.

```
/test-analyze -m
/test-analyze -m src/core/
```

**Output includes**:
- Assertions that would survive mutations
- Dead code in test coverage
- Trivial assertions (always pass)
- Missing edge case assertions

### Full Analysis (default)

When no mode specified, runs all analyses:

```
/test-analyze
/test-analyze src/
```

## Output Formats

### Summary View (default)

```markdown
## Test Analysis Summary

**Scope**: src/
**Tests Found**: 156
**Test Files**: 24

### Coverage
| Metric | Current | Target | Status |
|--------|---------|--------|--------|
| Lines | 82% | 80% | PASS |
| Branches | 74% | 75% | FAIL |
| Functions | 88% | 80% | PASS |

### Quality Score: 72/100

| Aspect | Score | Issues |
|--------|-------|--------|
| Assertion Quality | 78/100 | 12 weak assertions |
| Test Isolation | 85/100 | 3 isolation issues |
| Determinism | 68/100 | 8 flaky indicators |
| Readability | 75/100 | 15 naming issues |

### Critical Gaps
1. **src/auth/handler.ts:45-60** - No tests for error handling path
2. **src/payments/processor.ts** - 0% coverage on refund logic

### Flaky Test Candidates
1. `auth.test.ts: should timeout on slow response` - uses setTimeout
2. `cache.test.ts: should expire entries` - time-dependent

### Recommended Actions
1. Add error handling tests for auth handler - Priority: HIGH
2. Fix timing dependency in auth timeout test - Priority: HIGH
3. Add branch coverage for payment refunds - Priority: MEDIUM
```

### JSON Output (`--output json`)

```json
{
  "timestamp": "2025-01-14T10:00:00Z",
  "scope": "src/",
  "coverage": {
    "lines": { "current": 82, "target": 80, "status": "PASS" },
    "branches": { "current": 74, "target": 75, "status": "FAIL" },
    "functions": { "current": 88, "target": 80, "status": "PASS" }
  },
  "quality": {
    "overall": 72,
    "assertions": { "score": 78, "issues": 12 },
    "isolation": { "score": 85, "issues": 3 },
    "determinism": { "score": 68, "issues": 8 },
    "readability": { "score": 75, "issues": 15 }
  },
  "gaps": [...],
  "flaky": [...],
  "recommendations": [...]
}
```

## Examples

### Check coverage before code review

```
User: /test-analyze -c -i 42

Claude:
Analyzing test coverage for issue #42...

## Coverage Analysis

| Metric | Current | Target | Status |
|--------|---------|--------|--------|
| Lines | 85% | 80% | PASS |
| Branches | 78% | 75% | PASS |

### Changed Files Coverage
| File | Coverage | Status |
|------|----------|--------|
| src/auth/handler.ts | 92% | PASS |
| src/auth/validator.ts | 88% | PASS |

All coverage thresholds met. Proceeding to code review.
```

### Find flaky tests

```
User: /test-analyze -f

Claude:
## Flaky Test Detection

Found 5 tests with non-determinism indicators:

### High Risk (timing-dependent)
1. **auth.test.ts:45** - `should timeout after 5 seconds`
   - Uses `setTimeout` in assertion
   - Recommendation: Use fake timers

2. **cache.test.ts:112** - `should expire old entries`
   - Compares against `Date.now()`
   - Recommendation: Inject clock dependency

### Medium Risk (shared state)
3. **db.test.ts:78** - `should handle concurrent writes`
   - Modifies global connection pool
   - Recommendation: Isolate connection per test
```

### Assess test quality

```
User: /test-analyze -q src/payments/

Claude:
## Test Quality Assessment: src/payments/

### Overall Score: 65/100

### Assertion Analysis
| Classification | Count | Percentage |
|----------------|-------|------------|
| Strong | 23 | 58% |
| Weak | 12 | 30% |
| Trivial | 5 | 12% |

### Weak Assertions Found
1. **processor.test.ts:34** - `expect(result).toBeDefined()`
   - Should verify: specific result value

2. **refund.test.ts:56** - `expect(response).not.toBeNull()`
   - Should verify: response status and body

### Test Smells Detected
- 3 tests with excessive mocking
- 2 tests with logic in test body
- 1 test testing private method
```

## CI Integration

The skill can fetch coverage reports from CI artifacts.

### Supported Coverage Formats

| Format | File Pattern | Tools |
|--------|--------------|-------|
| LCOV | `lcov.info` | Jest, c8, nyc |
| Cobertura | `coverage.xml` | pytest-cov, JaCoCo |
| JSON Summary | `coverage-summary.json` | Jest |
| Go Coverage | `coverage.out` | go test |

### Coverage Lookup Order

1. Local files: `coverage/`, `./coverage.xml`, `./cover.out`
2. CI artifacts (if configured): Download from latest successful run
3. Manual: Prompt user to run tests with coverage

## Integration with Workflow

### TEST-VERIFICATION Gate

This skill is automatically invoked during `/crunch` at IMPLEMENTING → Code Review:

```
/test-analyze -c --threshold {from_Test_Plan} -i {issue_number}
```

If coverage fails, the workflow remains at IMPLEMENTING until tests are fixed.

### Knowledge Integration

Results are stored for trending:
- `.claude/test-analysis/{date}-{issue}.json`
- Queryable via `/knowledge -t test-analysis`
