## Incremental Development: Simplest Testable First

> **MUST:** All feature implementations MUST start with the simplest testable version, then extend incrementally toward the full solution.

### Core Principle

Build features in layers, not all at once:

1. **Milestone 0 (M0)**: Minimal viable slice with one passing test
2. **Milestone 1-N**: Add complexity incrementally, each milestone with passing tests
3. **Final**: Complete feature with full test coverage

Each milestone must be:
- Independently deployable
- Fully testable
- Committed separately

### Why Incremental Development?

| Benefit | Explanation |
|---------|-------------|
| Early feedback | Catch architectural issues before investing deeply |
| Easier debugging | Small changes = smaller search space for bugs |
| Progress visibility | Stakeholders see actual progress, not just "in progress" |
| Reduced risk | Each commit is a safe rollback point |
| Better tests | Tests written for each layer catch more edge cases |
| Faster reviews | Smaller changes are easier to review |

### Implementation Pattern

For each feature:

```
1. Identify the simplest end-to-end path (happy path, no edge cases)
2. Implement ONLY that path
3. Write test(s) for that path
4. Commit with passing test
5. Identify next layer of complexity
6. Implement that layer
7. Write test(s) for that layer
8. Commit with passing tests
9. Repeat until complete
```

### Milestone Planning Guidelines

**M0 - The Simplest Testable Version:**
- Happy path only
- No error handling beyond basic guards
- Hardcoded values acceptable
- Minimal UI (if any)
- One test proving core behavior works

**M1+ - Incremental Layers:**
- Add error handling for one error type
- Add one configuration option
- Add one edge case
- Add one UI element
- Each with corresponding tests

**Final - Complete Feature:**
- All error cases handled
- All configuration options
- All edge cases
- Full UI
- Comprehensive test coverage

### Example: User Authentication Feature

| Milestone | Scope | DOD Subset | Tests |
|-----------|-------|------------|-------|
| M0 | Login returns token (happy path only) | `POST /login` with valid creds returns 200 + token | 1 unit, 1 integration |
| M1 | Invalid credentials rejected | Returns 401 for wrong password | +2 tests |
| M2 | Token validation | Protected endpoint accepts valid token | +2 tests |
| M3 | Token expiry | Expired token returns 401 | +2 tests |
| M4 | Refresh token | `/refresh` returns new token | +3 tests |
| M5 | Rate limiting | 5 failed attempts = 15min lockout | +2 tests |
| Final | All security hardening | All DOD items complete | Full coverage |

### Example: File Upload Feature

| Milestone | Scope | DOD Subset | Tests |
|-----------|-------|------------|-------|
| M0 | Upload single small file | `POST /upload` with <1MB file returns 200 | 1 test |
| M1 | File type validation | Only allow .jpg/.png, reject others with 400 | +2 tests |
| M2 | File size limit | Reject >10MB with 413 | +2 tests |
| M3 | Progress tracking | `/upload/status/:id` returns progress % | +2 tests |
| M4 | Multiple files | Accept array of files | +2 tests |
| M5 | Thumbnail generation | Thumbnails created async | +2 tests |
| Final | Cleanup + edge cases | Temp files cleaned, all errors handled | Full coverage |

### Example: Search Feature

| Milestone | Scope | DOD Subset | Tests |
|-----------|-------|------------|-------|
| M0 | Basic text search | `GET /search?q=term` returns matching items | 1 test |
| M1 | Case-insensitive | "Apple" matches "apple" | +1 test |
| M2 | Pagination | `?page=2&limit=10` works | +2 tests |
| M3 | Filters | `?category=books` filters results | +2 tests |
| M4 | Sorting | `?sort=price&order=asc` works | +2 tests |
| M5 | Fuzzy matching | Typos still return results | +2 tests |
| Final | Performance + caching | <100ms response, cache hits | Full coverage |

### Milestone DOD Template

Each milestone should have a mini-DOD:

```markdown
### Milestone {N}: {Name}

**Scope:** {What this milestone adds}

**DOD:**
- [ ] {Specific testable criterion 1}
- [ ] {Specific testable criterion 2}

**Tests Required:**
- [ ] {Test description 1}
- [ ] {Test description 2}

**Not In Scope (deferred to later milestones):**
- {Thing 1}
- {Thing 2}
```

### Commit Strategy

Each milestone = one commit (or small PR):

```
feat(auth): M0 - basic login returns token

- POST /login accepts username/password
- Returns JWT on success
- Single happy-path test

Issue: #42
Milestone: 0/5
```

```
feat(auth): M1 - invalid credentials handling

- Return 401 for wrong password
- Return 400 for missing fields
- Tests for both error cases

Issue: #42
Milestone: 1/5
```

### Integration with ENRICH Phase

During ENRICH, the specification MUST include milestone breakdown:

```markdown
### Incremental Milestones

| Milestone | Scope | Tests | Estimated Complexity |
|-----------|-------|-------|---------------------|
| M0 | {simplest path} | {count} | Low |
| M1 | {next layer} | {count} | Low |
| M2 | {next layer} | {count} | Medium |
| ... | ... | ... | ... |

### M0 Definition (Start Here)

**Scope:** {Minimal viable implementation}

**Acceptance:**
- [ ] {Criterion 1}
- [ ] {Criterion 2}

**Explicitly Out of Scope:**
- {Deferred item 1}
- {Deferred item 2}
```

### Anti-Patterns (AVOID)

| Anti-Pattern | Problem | Better Approach |
|--------------|---------|-----------------|
| "Big bang" implementation | No feedback until end | Start with M0, iterate |
| M0 too complex | Defeats the purpose | Simplify until trivially testable |
| Skipping tests per milestone | Defeats incremental benefit | Every milestone needs tests |
| Milestones without commits | Loses rollback safety | Commit after each milestone |
| Scope creep in milestones | Delays feedback | Defer to later milestone |

### Benefits Checklist

After implementing incrementally, you should have:

- [ ] Multiple safe rollback points (one per milestone)
- [ ] Tests at each complexity layer
- [ ] Clear progress history in git log
- [ ] Easier code review (each milestone reviewable separately)
- [ ] Documentation of incremental decisions
- [ ] Confidence that each layer works before building on it
