# Test Coverage CI Integration

## Overview

This template provides guidance for integrating test coverage collection and enforcement into your CI pipeline.

## CI Pipeline Configuration

### GitHub Actions

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests with coverage
        run: npm test -- --coverage --coverageReporters=json-summary --coverageReporters=lcov

      - name: Upload coverage artifact
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
          retention-days: 30

      - name: Check coverage threshold
        run: |
          COVERAGE=$(jq '.total.lines.pct' coverage/coverage-summary.json)
          THRESHOLD=80
          if (( $(echo "$COVERAGE < $THRESHOLD" | bc -l) )); then
            echo "::error::Coverage $COVERAGE% is below $THRESHOLD% threshold"
            exit 1
          fi
          echo "Coverage: $COVERAGE% (threshold: $THRESHOLD%)"
```

### GitLab CI

```yaml
stages:
  - test

test:
  stage: test
  image: node:20
  script:
    - npm ci
    - npm test -- --coverage --coverageReporters=cobertura --coverageReporters=text
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    paths:
      - coverage/
    expire_in: 30 days
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
```

### Gitea Actions

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests with coverage
        run: |
          npm ci
          npm test -- --coverage

      - name: Upload coverage
        uses: actions/upload-artifact@v3
        with:
          name: coverage
          path: coverage/
```

## Language-Specific Coverage Commands

### JavaScript/TypeScript (Jest)

```bash
# Generate coverage
npm test -- --coverage

# With specific reporters
npm test -- --coverage --coverageReporters=lcov --coverageReporters=json-summary

# Check threshold
npm test -- --coverage --coverageThreshold='{"global":{"lines":80,"branches":75}}'
```

### Python (pytest)

```bash
# Generate coverage
pytest --cov=src --cov-report=xml --cov-report=term

# With threshold
pytest --cov=src --cov-fail-under=80
```

### Go

```bash
# Generate coverage
go test -coverprofile=coverage.out ./...

# View coverage
go tool cover -func=coverage.out

# HTML report
go tool cover -html=coverage.out -o coverage.html

# Check threshold (using custom script)
COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
if [ $(echo "$COVERAGE < 80" | bc) -eq 1 ]; then exit 1; fi
```

### Java (JaCoCo with Maven)

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## Coverage Report Formats

### Supported Formats

| Format | File | Tools |
|--------|------|-------|
| LCOV | `lcov.info` | Jest, c8, nyc, genhtml |
| Cobertura | `coverage.xml` | pytest-cov, JaCoCo, coverage.py |
| JSON Summary | `coverage-summary.json` | Jest |
| Go Coverage | `coverage.out` | go test |
| Clover | `clover.xml` | PHPUnit |

### Unified Format for /test-analyze

The `/test-analyze` skill parses coverage into this format:

```json
{
  "timestamp": "2025-01-14T10:00:00Z",
  "summary": {
    "lines": { "covered": 850, "total": 1000, "pct": 85.0 },
    "branches": { "covered": 120, "total": 150, "pct": 80.0 },
    "functions": { "covered": 95, "total": 100, "pct": 95.0 }
  },
  "files": {
    "src/auth/handler.ts": {
      "lines": { "covered": 45, "total": 50, "pct": 90.0 },
      "branches": { "covered": 8, "total": 10, "pct": 80.0 },
      "uncovered_lines": [23, 45, 46, 47, 89]
    }
  }
}
```

## Fetching Coverage from CI

### From GitHub Actions

```
1. List workflow runs:
   GET /repos/{owner}/{repo}/actions/runs?status=success&per_page=1

2. Get artifacts:
   GET /repos/{owner}/{repo}/actions/runs/{run_id}/artifacts

3. Download coverage artifact:
   GET /repos/{owner}/{repo}/actions/artifacts/{artifact_id}/zip

4. Parse lcov.info or coverage-summary.json
```

### From GitLab CI

```
1. List pipelines:
   GET /projects/{id}/pipelines?status=success&per_page=1

2. Get jobs:
   GET /projects/{id}/pipelines/{pipeline_id}/jobs

3. Download artifacts:
   GET /projects/{id}/jobs/{job_id}/artifacts

4. Parse cobertura XML
```

## Coverage Thresholds in CLAUDE.md

Add this section to your project's CLAUDE.md:

```markdown
## Test Coverage

### Thresholds

| Metric | Threshold | Enforcement |
|--------|-----------|-------------|
| Line Coverage | 80% | CI blocks merge |
| Branch Coverage | 75% | CI warns |
| Critical Paths | 100% | qa-engineer blocks review |

### Excluded Paths

Paths excluded from coverage requirements:
- `**/generated/**`
- `**/migrations/**`
- `**/*.d.ts`
- `**/test/**`
- `**/__mocks__/**`

### Per-Module Overrides

| Module | Line Threshold | Rationale |
|--------|----------------|-----------|
| `src/core/` | 90% | Critical business logic |
| `src/utils/` | 85% | Widely used utilities |
| `src/config/` | 60% | Configuration code |
```

## Enforcement Points

| Point | Tool | Action on Failure |
|-------|------|-------------------|
| PR/MR creation | CI pipeline | Block merge |
| TEST-VERIFICATION | qa-engineer | Block code review |
| Code Review | test-quality-reviewer | Flag with confidence score |
| Pre-merge | CI gate | Final verification |

## Flaky Test Detection

To detect flaky tests from CI history:

```yaml
# GitHub Actions - Run tests multiple times
test:
  strategy:
    matrix:
      run: [1, 2, 3]
  steps:
    - run: npm test -- --json --outputFile=results-${{ matrix.run }}.json

    - uses: actions/upload-artifact@v4
      with:
        name: test-results-${{ matrix.run }}
        path: results-${{ matrix.run }}.json
```

The `/test-analyze --flaky` command can analyze these results to identify:
- Tests that pass and fail across runs
- Tests with high duration variance
- Tests that require retries
