---
name: analyze
description: Analyze patterns in the knowledge base. Generate reports on recurring issues, agent effectiveness, review feedback trends, and technical debt indicators.
---

# Analyze - Pattern Detection and Reporting

Analyze the knowledge base to identify patterns, trends, and actionable insights.

## Syntax

```
/analyze [report-type] [options]
```

### Report Types

| Report | Description |
|--------|-------------|
| `bugs` | Recurring bug patterns by category, domain, file |
| `feedback` | Common review findings approaching pattern threshold |
| `effectiveness` | Agent performance metrics by domain and phase |
| `decisions` | Architectural decision inventory and health |
| `debt` | Technical debt indicators from accumulated issues |
| `summary` | High-level project knowledge health overview |

### Parameters

| Parameter | Short | Description |
|-----------|-------|-------------|
| `--report` | `-r` | Report type (default: `summary`) |
| `--domain` | `-d` | Filter to specific domain |
| `--since` | | Only entries after date (YYYY-MM-DD) |
| `--output` | `-o` | Output format: `markdown`, `json` |
| `--top` | | Limit to top N items (default: 10) |

## Reports

### Bug Pattern Report (`bugs`)

Analyze resolution entries to identify recurring bug categories.

```
/analyze bugs
/analyze bugs -d security --since 2024-10-01
```

**Output includes**:
- Bug count by category (concurrency, validation, auth, etc.)
- Bug count by domain
- Hotspot files with bug frequency
- Average time-to-resolve by category
- Trend indicators (increasing/decreasing/stable)

**Example Output**:
```markdown
## Bug Pattern Analysis

**Period**: Last 90 days | **Total Bugs**: 45

### By Category
| Category | Count | Avg Resolution | Trend |
|----------|-------|----------------|-------|
| Concurrency | 8 | 4.2h | ↑ increasing |
| Validation | 12 | 1.8h | → stable |
| Auth | 6 | 5.1h | ↓ decreasing |

### Hotspot Files
1. `src/auth/tokenService.ts` - 4 bugs (3 concurrency)
2. `src/api/userController.ts` - 3 bugs (2 validation)

### Recommendations
- Consider architectural review of tokenService.ts
- Add mutex abstraction for auth operations
```

---

### Review Feedback Report (`feedback`)

Analyze feedback entries to surface common issues and pattern candidates.

```
/analyze feedback
/analyze feedback -d frontend
```

**Output includes**:
- Finding categories with counts
- Findings approaching pattern threshold (2+ occurrences)
- Distribution by reviewer agent
- Domain breakdown

**Example Output**:
```markdown
## Review Feedback Analysis

**Period**: Last 90 days | **Total Findings**: 127 (84 high-confidence)

### Pattern Candidates (near threshold)
| Finding Pattern | Count | Domains | Action |
|-----------------|-------|---------|--------|
| Missing null check | 4 | api, auth | Ready to promote |
| Inconsistent error format | 3 | api | Ready to promote |
| Unused import | 2 | frontend | 1 more to promote |

### By Reviewer Agent
| Agent | Findings | Top Category |
|-------|----------|--------------|
| bug-scanner | 45 | Null handling |
| CLAUDE.md Compliance | 28 | Naming conventions |
| Code Comment Checker | 12 | TODO tracking |

### Recommendations
- Promote "Missing null check" to pattern
- Review error handling standards in CLAUDE.md
```

---

### Agent Effectiveness Report (`effectiveness`)

Analyze agent usage patterns and outcomes.

```
/analyze effectiveness
/analyze effectiveness -d security
```

**Output includes**:
- Issues handled by agent
- Success rate (issues reaching DONE without regression)
- Average phase duration
- Domain-agent affinity scores

**Example Output**:
```markdown
## Agent Effectiveness Report

**Period**: All time | **Total Issues**: 156

### Performance by Agent
| Agent | Issues | Success Rate | Avg Duration |
|-------|--------|--------------|--------------|
| security-analyst | 34 | 94% | 4.2h |
| dev-react | 45 | 91% | 2.8h |
| reviewer | 89 | 88% | 1.5h |
| dev-python | 28 | 85% | 3.5h |

### Recommended Domain Pairings
| Domain | Best Agent | Success Rate | vs Baseline |
|--------|------------|--------------|-------------|
| Security | security-analyst | 94% | +16% |
| Frontend | dev-react | 91% | +9% |
| API | reviewer | 88% | baseline |

### Underperforming Combinations
- dev-python on infrastructure issues: 67% (consider devops-engineer)
- reviewer on security issues: 72% (consider security-analyst)
```

---

### Decision Inventory Report (`decisions`)

Catalog architectural decisions and their health status.

```
/analyze decisions
/analyze decisions -d infrastructure
```

**Output includes**:
- Active decisions by domain
- Most frequently referenced decisions
- Stale decisions (high decay, low references)
- Superseded or conflicting decisions

**Example Output**:
```markdown
## Architectural Decision Inventory

**Total Decisions**: 24 | **Active**: 22 | **Superseded**: 2

### By Domain
| Domain | Count | Most Referenced |
|--------|-------|-----------------|
| Security | 8 | Token rotation (12 refs) |
| API | 6 | REST over GraphQL (8 refs) |
| Database | 4 | PostgreSQL primary (6 refs) |

### Health Status
| Status | Count | Decisions |
|--------|-------|-----------|
| Evergreen | 12 | High reference, low decay |
| Active | 8 | Normal decay, recent refs |
| Stale | 4 | High decay, no recent refs |

### Recommendations
- Review stale decisions: ADR-005, ADR-008
- Archive superseded: ADR-003 (replaced by ADR-021)
- Consider documenting: 3 implicit decisions detected
```

---

### Technical Debt Report (`debt`)

Synthesize debt indicators from multiple knowledge types.

```
/analyze debt
/analyze debt --top 5
```

**Output includes**:
- High-debt areas (files, modules, domains)
- Bug frequency hotspots
- Review finding density
- Unaddressed patterns

**Example Output**:
```markdown
## Technical Debt Analysis

**Overall Score**: 68/100 (higher = more debt)

### High-Debt Areas
| Area | Indicators | Score |
|------|------------|-------|
| `src/auth/` | 4 bugs, 8 findings, 2 unresolved TODOs | 85 |
| `src/api/validation/` | 12 validation bugs | 72 |
| `src/legacy/` | 0 recent changes, 6 stale patterns | 65 |

### Debt Trend
- Overall: ↑ 6 points from last quarter
- Growing: Auth (+15), API (+8)
- Shrinking: Frontend (-12)

### Recommended Actions
1. **Refactor tokenService.ts** - 4 bugs in 90 days
2. **Create validation middleware** - 12 scattered issues
3. **Update legacy patterns** - 6 outdated approaches

### Unaddressed Patterns
| Pattern | Occurrences | Status |
|---------|-------------|--------|
| Token mutex | 3 | Detected, not codified |
| Error format | 3 | Inconsistent across codebase |
```

---

### Summary Report (`summary`)

High-level knowledge base health overview.

```
/analyze summary
/analyze
```

**Example Output**:
```markdown
## Knowledge Base Summary

**As of**: 2025-01-14 | **Total Entries**: 245

### Entry Distribution
| Type | Count | Last 30 Days | Trend |
|------|-------|--------------|-------|
| Resolutions | 156 | +23 | ↑ |
| Decisions | 24 | +2 | → |
| Patterns | 18 | +4 | ↑ |
| Feedback | 47 | +31 | ↑ |

### Key Metrics
| Metric | Value | Status |
|--------|-------|--------|
| Avg time-to-resolve | 3.2h | ↓ improving |
| Pattern adherence | 78% | → stable |
| Agent effectiveness | 89% | → stable |
| First-time resolution | 72% | ↑ improving |

### Alerts
- 3 pattern candidates ready for promotion
- 4 decisions may be stale (review recommended)
- Auth domain showing increased bug density (+40%)

### Recent Patterns
1. Token Operation Mutex (promoted 2025-01-10)
2. Input Validation Boundary (promoted 2025-01-05)
```

## Integration

The `/analyze` skill is designed for:

1. **Periodic health checks** - Run weekly to monitor knowledge quality
2. **Sprint retrospectives** - Review what was learned during sprint
3. **Onboarding** - Help new team members understand common patterns
4. **Planning** - Identify high-debt areas before major work

## Best Practices

- Run `summary` report weekly to catch emerging patterns
- Review `feedback` report to promote ready patterns
- Use `bugs` report before refactoring to target hotspots
- Check `effectiveness` when selecting agents for important issues
- Update CLAUDE.md based on `patterns` findings
