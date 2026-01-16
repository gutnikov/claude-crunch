---
name: knowledge
description: Query the project knowledge base. Search for past solutions, decisions, patterns, and feedback. Use before starting work on similar issues or when seeking institutional wisdom.
---

# Knowledge - Query and Retrieval

Search and retrieve relevant knowledge from the project's knowledge base.

## Syntax

```
/knowledge [query] [options]
```

### Parameters

| Parameter | Short | Description |
|-----------|-------|-------------|
| `--query` | `-q` | Search query (semantic search) |
| `--type` | `-t` | Filter: `decision`, `resolution`, `pattern`, `feedback`, `all` |
| `--domain` | `-d` | Filter by domain (e.g., `security`, `frontend`) |
| `--since` | | Only entries after date (YYYY-MM-DD) |
| `--limit` | `-l` | Max results (default: 5) |
| `--related` | `-r` | Find entries related to issue number or entry ID |
| `--brief` | | Generate synthesized context brief |

### Query Modes

| Mode | Example | Use Case |
|------|---------|----------|
| Semantic | `/knowledge "authentication race conditions"` | Find conceptually related |
| By type | `/knowledge -t pattern -d security` | Browse patterns in domain |
| Related | `/knowledge -r 42` | Find knowledge linked to issue #42 |
| Brief | `/knowledge --brief -r 55` | Context brief for issue #55 |

## Query Syntax

### Simple Query

```
/knowledge authentication bugs
```

Searches all fields of all entry types for matching content.

### Filtered Query

```
/knowledge -t resolution -d security "token"
```

Only searches resolution entries tagged with security domain.

### Related Query

```
/knowledge -r KE-decision-20250114-a3f2
```

Returns all entries linked to the specified knowledge entry.

### Context Brief

```
/knowledge --brief -r 55
```

Generates a synthesized context brief for use before starting work on issue #55.

## Response Formats

### Standard Results

```markdown
## Knowledge Search Results

**Query**: "authentication race conditions"
**Found**: 4 entries (showing top 3)

---

### 1. [Resolution] KE-resolution-20250114-b7c9 (relevance: 95)
**Issue**: #42 - Race condition in token refresh causes logout
**Root Cause**: Concurrent refresh requests without synchronization
**Solution**: Mutex protection for token operations
**Files**: src/auth/tokenService.ts

---

### 2. [Pattern] KE-pattern-20250114-c8d3 (relevance: 88)
**Name**: Token Operation Mutex
**Type**: Pattern (recommended practice)
**Description**: All token operations should be protected by mutex
**Occurrences**: 3 issues

---

### 3. [Decision] KE-decision-20250112-x1y2 (relevance: 72)
**Title**: Use refresh token rotation
**Decision**: Implement sliding-window refresh token rotation
**Relevance**: Establishes auth token handling architecture
```

### Context Brief Format

```markdown
## Context Brief for Issue #55

### Summary
This issue involves authentication token handling. Found 4 highly relevant entries.

### Must-Know
1. **Token Operation Mutex** (pattern): All token ops need mutex protection
2. **ADR: Refresh Token Rotation** (decision): Current auth architecture

### Related Past Issues
- **#42**: Similar race condition - solved with mutex (4h resolution)
- **#18**: Auth flow rework - see architectural decisions

### Anti-Patterns to Avoid
- Direct token manipulation without service abstraction
- Assuming single-threaded token access

### Recommended Agents
Based on effectiveness data:
- security-analyst: 94% success on auth issues (23 issues)
- dev-react: 87% on frontend auth (8 issues)
```

## Examples

### Search before starting work

```
User: /knowledge --brief -r 55

Claude:
Generating context brief for issue #55...

Issue #55: "Session timeout not working correctly"
Domain detected: security, auth

## Context Brief for Issue #55

### Must-Know
1. **Token Operation Mutex**: All token ops need mutex protection
2. **Session Lifetime Decision**: 7-day sliding window

### Related Past Issues
- **#42** (relevance: 92): Race condition in token refresh
  - Root cause: Concurrent access
  - Solution: Mutex protection
  - Time to resolve: 4h

### Anti-Patterns
- Direct localStorage access for tokens
- Timer-based session checks without server validation

### Recommended Agents
- security-analyst (94% effectiveness on auth)
```

### Browse patterns by domain

```
User: /knowledge -t pattern -d security

Claude:
## Security Domain Patterns

Found 5 patterns:

1. **Token Operation Mutex** (3 occurrences)
   All token operations protected by mutex

2. **Input Validation at Boundary** (5 occurrences)
   Validate all external input at API boundaries

3. **Secure Defaults Configuration** (2 occurrences)
   Security settings should be secure by default

4. **Least Privilege Access** (4 occurrences)
   Request minimum necessary permissions

5. **Audit Log for Sensitive Operations** (2 occurrences)
   Log all authentication and authorization events
```

### Find related knowledge

```
User: /knowledge -r 42

Claude:
## Knowledge Related to Issue #42

### Direct References
- **KE-resolution-20250114-b7c9**: Resolution of this issue
- **KE-feedback-20250114-d9e4**: Review feedback during implementation

### Linked Entries
- **KE-decision-20250112-a3f2**: Token rotation decision (referenced in spec)
- **KE-pattern-20250114-c8d3**: Token Mutex pattern (derived from this fix)

### Similar Issues
- **#18**: Auth flow rework (same domain, related files)
- **#35**: Token expiration handling (similar root cause category)
```

## Relevance Scoring

Results are ranked by combined score:

```
final_score = semantic_similarity * 0.6 +
              recency_score * 0.2 +
              domain_match * 0.1 +
              occurrence_weight * 0.1

Where:
- semantic_similarity: 0-1 based on content match
- recency_score: decay_score from entry (0-1)
- domain_match: 1.0 if domain matches, 0.5 otherwise
- occurrence_weight: For patterns, higher occurrence = higher weight
```

## Integration with Workflow

The `/knowledge` skill is automatically invoked:

| Trigger | Query Type |
|---------|------------|
| ENRICH entry | `--brief -r {issue}` for context |
| IMPLEMENTING entry | `-t pattern -t resolution` for affected files |
| Before code review | `-t feedback -d {domain}` for common issues |
