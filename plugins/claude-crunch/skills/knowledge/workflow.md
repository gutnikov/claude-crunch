# Knowledge Skill - Workflow Implementation

## Overview

The `/knowledge` skill queries the project's knowledge base and returns relevant entries or synthesized context briefs.

## Workflow Phases

### Phase 1: Parse Query

```
1. Parse command arguments:
   - query: Free-text search terms
   - --type (-t): Filter by entry type
   - --domain (-d): Filter by domain tag
   - --since: Date filter
   - --limit (-l): Max results
   - --related (-r): Issue or entry ID
   - --brief: Generate context brief

2. Determine query mode:
   - IF --brief → Context Brief Mode
   - IF --related with issue number → Issue Related Mode
   - IF --related with entry ID → Entry Related Mode
   - ELSE → Search Mode
```

### Phase 2: Load Knowledge Base

```
1. Check for .claude/knowledge/index.yaml
   - IF not exists: Report "Knowledge base empty"

2. Load index into memory

3. Apply decay scoring:
   - FOR each entry in index:
     - Calculate current decay based on age
     - Filter out entries below min_score threshold
```

### Phase 3: Execute Query

#### Search Mode

```
1. Build candidate set:
   - IF type filter: Filter to matching types
   - IF domain filter: Filter to matching domain tags
   - IF since filter: Filter to entries after date

2. Score candidates:
   - FOR each candidate:
     - Calculate semantic similarity to query
     - Apply relevance formula:
       score = similarity * 0.6 + decay * 0.2 + domain_match * 0.1 + occurrence * 0.1

3. Sort by score descending

4. Limit to --limit results (default: 5)

5. Load full entries for top results
```

#### Issue Related Mode

```
1. Look up issue in issue_index

2. Get directly linked entries:
   - Resolutions for this issue
   - Decisions made during this issue
   - Feedback from reviews on this issue

3. Find indirectly related:
   - Entries with same domain tags
   - Entries referencing same files
   - Similar issues by content

4. Rank by relationship strength + relevance
```

#### Entry Related Mode

```
1. Load the specified entry

2. Follow relationships:
   - related_issues
   - related_entries
   - supersedes / superseded_by

3. Find entries with overlapping:
   - Tags
   - File references
   - Domain

4. Return relationship graph
```

#### Context Brief Mode

```
1. Fetch issue details (if issue number provided)

2. Extract context:
   - Issue title and description
   - Domain labels
   - Files likely to be affected (from similar issues)

3. Query for each knowledge type:
   - Decisions: Active decisions in domain
   - Resolutions: Similar past issues
   - Patterns: Applicable patterns and anti-patterns
   - Feedback: Common review findings in domain

4. Query agent metrics:
   - Effectiveness by domain
   - Recommended agents

5. Synthesize brief (see template below)
```

### Phase 4: Format Output

#### Standard Results Format

```markdown
## Knowledge Search Results

**Query**: "{query}"
**Filters**: {type}, {domain}, since {date}
**Found**: {total} entries (showing {limit})

---

### {rank}. [{type}] {id} (relevance: {score})

**{title_field}**: {value}
**{key_field_1}**: {value}
**{key_field_2}**: {value}

---

[Repeat for each result]
```

#### Context Brief Format

```markdown
## Context Brief for Issue #{number}

### Summary

{One sentence describing the issue domain and what knowledge was found}

### Must-Know

{Numbered list of critical patterns and decisions}

### Related Past Issues

{List of similar issues with outcomes and resolution times}

### Anti-Patterns to Avoid

{List of anti-patterns applicable to this domain}

### Recommended Agents

{Agent recommendations based on effectiveness data}
```

---

## Relevance Scoring Algorithm

### Semantic Similarity

```
For each candidate entry:
  1. Extract searchable text:
     - title/name
     - description/problem/finding
     - solution/decision/recommendation
     - tags

  2. Calculate similarity:
     - Keyword overlap score (0-1)
     - Phrase match bonus (+0.2 for exact phrases)
     - Domain term weighting (domain-specific terms score higher)

  3. Return normalized similarity (0-1)
```

### Decay Score Calculation

```
decay_score = max(min_score, initial_score - (days_old * decay_rate))

Where:
  initial_score = 1.0
  decay_rate = config.decay.rate (default: 0.01)
  min_score = config.decay.min_score (default: 0.3)

Exception: Entries with evergreen tags don't decay
  evergreen_tags = ["architecture", "security", "core", "breaking"]
```

### Final Relevance Score

```
final_score = (
  semantic_similarity * 0.6 +
  decay_score * 0.2 +
  domain_match * 0.1 +
  occurrence_weight * 0.1
)

Where:
  domain_match = 1.0 if query domain matches entry domain, else 0.5
  occurrence_weight = min(1.0, occurrences / 10) for patterns, else 0.5
```

---

## Context Brief Generation

### Step 1: Gather Issue Context

```
IF issue number provided:
  1. Fetch issue via CI MCP
  2. Extract:
     - Title and description
     - Labels (type, domain)
     - Mentioned files or modules

ELSE:
  1. Use query terms as context
  2. Infer domain from keywords
```

### Step 2: Query Each Knowledge Type

```
decisions_query:
  type: decision
  filter: domain matches OR status = "active"
  limit: 3
  sort: relevance

resolutions_query:
  type: resolution
  filter: similar problem OR same files
  limit: 3
  sort: relevance + recency

patterns_query:
  type: pattern
  filter: domain matches
  limit: 5
  sort: occurrences DESC

anti_patterns_query:
  type: pattern
  filter: pattern_type = "anti-pattern" AND domain matches
  limit: 3
```

### Step 3: Query Agent Metrics

```
FROM index.agent_metrics:
  1. Filter to agents with domain experience
  2. Sort by success_rate DESC
  3. Return top 2-3 recommendations with stats
```

### Step 4: Synthesize Brief

```
Combine results into context brief:
  1. Must-Know: Top patterns + active decisions
  2. Related Issues: Similar resolutions with outcomes
  3. Anti-Patterns: Warnings for this domain
  4. Agents: Effectiveness-based recommendations
```

---

## Index Structure Reference

### tag_index

```json
{
  "domain:security": ["KE-resolution-...", "KE-pattern-..."],
  "auth": ["KE-resolution-...", "KE-decision-..."],
  "concurrency": ["KE-pattern-...", "KE-feedback-..."]
}
```

Enables O(1) lookup for domain/tag filters.

### issue_index

```json
{
  "#42": ["KE-resolution-...", "KE-feedback-..."],
  "#18": ["KE-decision-...", "KE-resolution-..."]
}
```

Enables O(1) lookup for issue-related queries.

### agent_metrics

```json
{
  "security-analyst": {
    "total_invocations": 34,
    "by_domain": {
      "security": { "count": 28, "success_rate": 0.94 }
    }
  }
}
```

Enables agent effectiveness queries.

---

## Error Handling

| Error                    | Handling                                                        |
| ------------------------ | --------------------------------------------------------------- |
| Knowledge base not found | Report "Knowledge base empty. Use /learn to capture knowledge." |
| No results found         | Report "No matching entries. Try broader search."               |
| Invalid entry ID         | Report "Entry not found: {id}"                                  |
| Issue not in index       | Fall back to content-based search                               |

---

## Performance Considerations

1. **Index-first queries**: Always filter using index before loading full entries
2. **Lazy loading**: Only load full entry JSON when needed for display
3. **Cache recent queries**: Store last 10 queries in memory for repeated access
4. **Limit by default**: Always apply reasonable limits to prevent overwhelming output
