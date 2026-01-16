# Test Plan Template

> **MUST:** Every feature and bug fix must have a Test Plan created during the ENRICH phase. The Test Plan is created alongside the Definition of Done (DoD) checklist.

## Purpose

The Test Plan artifact defines:

- What testing levels are required
- Which critical paths must have coverage
- What edge cases need explicit tests
- Coverage thresholds for the feature

## When Created

| Aspect   | Value                                |
| -------- | ------------------------------------ |
| Phase    | ENRICH (specification/investigation) |
| Creator  | qa-engineer agent                    |
| Location | Issue body, following DoD checklist  |

## Test Plan Structure

```markdown
### Test Plan

#### Required Test Levels

| Level       | Required | Rationale                                                    |
| ----------- | -------- | ------------------------------------------------------------ |
| Unit        | Yes/No   | {why - e.g., "Pure logic requires unit tests"}               |
| Integration | Yes/No   | {why - e.g., "Database interaction needs integration tests"} |
| E2E         | Yes/No   | {why - e.g., "User-facing flow requires e2e"}                |
| Contract    | Yes/No   | {why - e.g., "API boundary needs contract tests"}            |

#### Critical Paths (P0 - Must Test)

1. **{Path Name}**
   - Entry: {entry point or trigger}
   - Expected: {expected behavior}
   - Test: {test description}

2. **{Path Name}**
   - Entry: {entry point}
   - Expected: {expected behavior}
   - Test: {test description}

#### Edge Cases (P1 - Should Test)

| Edge Case | Why It Matters       | Test Approach |
| --------- | -------------------- | ------------- |
| {case}    | {impact if untested} | {how to test} |
| {case}    | {impact if untested} | {how to test} |

#### Boundary Conditions

| Boundary   | Values to Test                     |
| ---------- | ---------------------------------- |
| {boundary} | min, max, zero, negative, overflow |
| {boundary} | empty, null, whitespace            |

#### Error Scenarios

| Error Condition | Expected Behavior          | Test Approach   |
| --------------- | -------------------------- | --------------- |
| {error}         | {how it should be handled} | {how to verify} |
| {error}         | {how it should be handled} | {how to verify} |

#### Coverage Targets

| Metric                 | Threshold | Rationale                   |
| ---------------------- | --------- | --------------------------- |
| Line Coverage          | {X}%      | {why this threshold}        |
| Branch Coverage        | {Y}%      | {why this threshold}        |
| Critical Path Coverage | 100%      | Non-negotiable for P0 paths |

#### Test Data Requirements

- **Fixtures**: {required test fixtures}
- **Mocks**: {services/components to mock}
- **Test Database**: {Yes/No - specific data requirements}

#### Environment Requirements

- {Any special test environment needs}

#### E2E Validation Checklist

> Maps DOD items to autonomous verification methods. See `templates/5-e2e-testability.md`.

| DOD Item      | Verification Type     | Command/Query | Expected          |
| ------------- | --------------------- | ------------- | ----------------- |
| {description} | API/Metric/Log/Health | {command}     | {expected result} |
| {description} | API/Metric/Log/Health | {command}     | {expected result} |

#### Incremental Milestones

> Feature breakdown from simplest testable version. See `templates/5a-incremental-development.md`.

| Milestone | Scope                 | Tests Required | DOD Subset        |
| --------- | --------------------- | -------------- | ----------------- |
| M0        | {simplest happy path} | {count}        | {which DOD items} |
| M1        | {next layer}          | {count}        | {which DOD items} |
| M2        | {next layer}          | {count}        | {which DOD items} |
```

## Test Level Decision Matrix

Use this matrix to determine which test levels are required:

| Scenario                 | Unit              | Integration | E2E      |
| ------------------------ | ----------------- | ----------- | -------- |
| Pure logic/calculation   | Required          | Optional    | No       |
| Database interaction     | Required          | Required    | Optional |
| External API integration | Required (mocked) | Required    | Optional |
| User-facing workflow     | Required          | Optional    | Required |
| Security-sensitive       | Required          | Required    | Required |
| Performance-critical     | Required          | Optional    | Required |
| Configuration/setup      | Optional          | Required    | Optional |

## Coverage Threshold Guidelines

Default thresholds by component type (can be overridden in CLAUDE.md):

| Component Type           | Line Coverage | Branch Coverage |
| ------------------------ | ------------- | --------------- |
| Core business logic      | 90%           | 85%             |
| API handlers/controllers | 80%           | 75%             |
| Utility functions        | 85%           | 80%             |
| Configuration/setup      | 60%           | 50%             |
| Generated code           | Excluded      | Excluded        |
| Test utilities           | Excluded      | Excluded        |

## Integration with Issue Body

During ENRICH phase, the Test Plan is appended after the DoD checklist:

```markdown
## {Bug|Feature}: {title}

### Summary

{description}

### Investigation/Specification

{details}

### Implementation Notes

- Files to create: {list}
- Files to modify: {list}

### Definition of Done (Staging Validation)

**Infrastructure**:

- [ ] App on staging is running successfully
- [ ] No suspicious error logs
- [ ] Metrics and dashboards are healthy
- [ ] No alerts firing

**Functional**:

- [ ] {Specific check 1}
- [ ] {Specific check 2}

**Regression**:

- [ ] {Regression check 1}

### Test Plan

{Test Plan content as defined above}

**Status**: Ready
```

## Example Test Plans

### Example 1: API Endpoint Feature

```markdown
### Test Plan

#### Required Test Levels

| Level       | Required | Rationale                     |
| ----------- | -------- | ----------------------------- |
| Unit        | Yes      | Handler logic and validation  |
| Integration | Yes      | Database and service layer    |
| E2E         | No       | Internal API, not user-facing |

#### Critical Paths (P0)

1. **Happy Path - Create Resource**
   - Entry: POST /api/resources with valid payload
   - Expected: 201 response, resource in database
   - Test: `should create resource and return 201`

2. **Authentication Required**
   - Entry: POST /api/resources without auth token
   - Expected: 401 response
   - Test: `should reject unauthenticated requests`

#### Edge Cases (P1)

| Edge Case          | Why It Matters | Test Approach                |
| ------------------ | -------------- | ---------------------------- |
| Duplicate resource | Data integrity | Verify 409 conflict response |
| Max payload size   | DoS prevention | Test with oversized payload  |

#### Coverage Targets

| Metric          | Threshold |
| --------------- | --------- |
| Line Coverage   | 85%       |
| Branch Coverage | 80%       |
```

### Example 2: Bug Fix

```markdown
### Test Plan

#### Required Test Levels

| Level       | Required | Rationale               |
| ----------- | -------- | ----------------------- |
| Unit        | Yes      | Regression test for bug |
| Integration | No       | Bug is in pure logic    |
| E2E         | No       | Not user-facing         |

#### Critical Paths (P0)

1. **Regression Test - The Bug**
   - Entry: {exact conditions that triggered bug}
   - Expected: {correct behavior after fix}
   - Test: `should handle {condition} correctly`

#### Edge Cases (P1)

| Edge Case           | Why It Matters       | Test Approach        |
| ------------------- | -------------------- | -------------------- |
| Similar condition A | Related failure mode | Verify no regression |
| Similar condition B | Related failure mode | Verify no regression |

#### Coverage Targets

| Metric        | Threshold |
| ------------- | --------- |
| Line Coverage | 80%       |
| Changed lines | 100%      |
```
