# Analyze Skill - Workflow Implementation

## Overview

The `/analyze` skill generates reports from the knowledge base to surface patterns, trends, and actionable insights.

## Workflow Phases

### Phase 1: Parse Arguments

```
1. Parse command arguments:
   - [report-type]: bugs | feedback | effectiveness | decisions | debt | summary
   - --domain (-d): Filter to domain
   - --since: Date filter
   - --output (-o): markdown | json
   - --top: Limit items

2. Default values:
   - report-type: summary
   - output: markdown
   - top: 10
   - since: (none - all time)
```

### Phase 2: Load Knowledge Base

```
1. Check .claude/knowledge/index.json exists
   IF not: Report "Knowledge base empty"

2. Load full index

3. Load config for thresholds:
   - pattern_promotion_threshold
   - decay rates
   - evergreen tags
```

### Phase 3: Generate Report

Route to appropriate report generator based on type.

---

## Report Generators

### Bugs Report

```
1. Filter entries:
   - type = "resolution"
   - issue_type = "bug"
   - Apply --domain and --since filters

2. Aggregate by category:
   FOR each resolution:
     category = content.root_cause.category
     Increment category_counts[category]
     Add to category_resolutions[category]

3. Calculate metrics:
   FOR each category:
     - Count
     - Avg time_to_resolve
     - Trend (compare last 30 days vs previous 30)

4. Find hotspot files:
   FOR each resolution:
     FOR each file in files_changed:
       Increment file_bug_count[file]
   Sort by count DESC
   Take top N

5. Generate recommendations:
   - Files with 3+ bugs: "Consider review"
   - Categories trending up: "Investigate root cause"
   - Patterns that could help: Link to applicable patterns
```

### Feedback Report

```
1. Filter entries:
   - type = "feedback"
   - Apply --domain filter
   - Apply --since filter

2. Group similar feedback:
   FOR each feedback:
     Find similar (same category, similar finding text)
     Add to similarity groups

3. Identify pattern candidates:
   FOR each group with count >= 2:
     IF count >= threshold: "Ready to promote"
     ELSE: "N more to promote"

4. Aggregate by agent:
   FOR each feedback:
     agent = source.reviewer_agent
     Increment agent_counts[agent]
     Track top category per agent

5. Calculate domain distribution

6. Generate recommendations:
   - Groups at threshold: "Promote to pattern"
   - Repeated findings: "Update CLAUDE.md"
```

### Effectiveness Report

```
1. Load agent_metrics from index

2. Calculate per-agent stats:
   FOR each agent in metrics:
     total_issues = sum of all domain counts
     overall_success = weighted average of domain success rates
     avg_duration = weighted average of domain durations

3. Find domain affinities:
   FOR each domain:
     Find agent with highest success rate
     Compare to baseline (reviewer success rate)
     Calculate delta

4. Find underperforming combinations:
   FOR each agent-domain pair:
     IF success_rate < baseline - 10%:
       Flag as underperforming
       Suggest better agent

5. Generate recommendations:
   - Optimal pairings by domain
   - Combinations to avoid
   - Training opportunities
```

### Decisions Report

```
1. Filter entries:
   - type = "decision"
   - Apply --domain filter

2. Categorize by status:
   FOR each decision:
     IF superseded_by: status = "superseded"
     ELIF decay_score < 0.5 AND no recent references: status = "stale"
     ELIF evergreen tag: status = "evergreen"
     ELSE: status = "active"

3. Count references:
   FOR each decision:
     Count entries linking to this decision
     Count issues referencing this decision

4. Find conflicts:
   FOR each pair of decisions:
     IF same domain AND contradictory:
       Flag for review

5. Generate recommendations:
   - Stale decisions: "Review for relevance"
   - Superseded: "Archive"
   - High-reference: "Consider documenting in CLAUDE.md"
```

### Debt Report

```
1. Calculate file debt scores:
   FOR each file mentioned in resolutions or feedback:
     bug_score = bug_count * 10
     feedback_score = feedback_count * 5
     recency_weight = (recent bugs score higher)
     debt_score = (bug_score + feedback_score) * recency_weight

2. Calculate domain debt:
   FOR each domain:
     Sum file debt scores in domain
     Add pattern violation count * 3
     Add stale decision count * 2

3. Calculate trends:
   Compare current quarter to previous:
     debt_change = current_score - previous_score
     trend = "growing" | "shrinking" | "stable"

4. Find unaddressed patterns:
   Feedback groups at promotion threshold but not yet promoted

5. Generate recommended actions:
   Priority = debt_score * actionability
   Sort by priority DESC
   Take top N
```

### Summary Report

```
1. Calculate entry distribution:
   Count by type
   Count last 30 days by type
   Calculate trend per type

2. Calculate key metrics:
   - Avg time-to-resolve: mean of resolution.time_to_resolve
   - Pattern adherence: (resolutions citing patterns / total resolutions)
   - Agent effectiveness: weighted mean success rate
   - First-time resolution: (issues without rework / total issues)

3. Generate alerts:
   - Pattern candidates at threshold
   - Stale decisions count
   - Domains with increasing bug density

4. List recent patterns:
   Patterns created in last 30 days
   Sorted by created_at DESC
```

---

## Output Formatting

### Markdown Output (default)

```markdown
## {Report Title}

**Period**: {date range} | **Metric**: {value}

### Section 1
| Column 1 | Column 2 | Column 3 |
|----------|----------|----------|
| data | data | data |

### Section 2
- Point 1
- Point 2

### Recommendations
1. **Action 1** - reason
2. **Action 2** - reason
```

### JSON Output

```json
{
  "report_type": "bugs",
  "generated_at": "2025-01-14T10:00:00Z",
  "period": {
    "start": "2024-10-01",
    "end": "2025-01-14"
  },
  "data": {
    "by_category": [...],
    "hotspots": [...],
    "trends": [...]
  },
  "recommendations": [...]
}
```

---

## Trend Calculation

```
Calculate trend for a metric:

1. Get values for current period (last 30 days)
2. Get values for previous period (30-60 days ago)
3. Calculate change:
   change = (current - previous) / previous * 100

4. Determine trend:
   IF change > 10%: "increasing" (↑)
   ELIF change < -10%: "decreasing" (↓)
   ELSE: "stable" (→)
```

---

## Debt Scoring Formula

```
File Debt Score:
  base_score = (bug_count * 10) + (feedback_count * 5)
  recency_factor = 1 + (recent_bug_count / total_bug_count)
  complexity_factor = 1 + (0.1 * unique_issue_types)

  debt_score = base_score * recency_factor * complexity_factor

Domain Debt Score:
  file_debt = sum of file scores in domain
  pattern_violations = unaddressed_patterns * 3
  stale_decisions = stale_decision_count * 2

  domain_score = file_debt + pattern_violations + stale_decisions

Overall Debt Score (0-100):
  raw_score = sum of domain scores
  normalized = min(100, raw_score / expected_max * 100)
```

---

## Error Handling

| Condition | Handling |
|-----------|----------|
| Empty knowledge base | Report "No data available" |
| No matching entries | Report "No entries match filters" |
| Insufficient data for trends | Show "N/A" for trend |
| Missing agent metrics | Skip effectiveness calculations |

---

## Performance Considerations

1. **Index-first**: Use index for aggregation when possible
2. **Lazy load entries**: Only load full entries for detail views
3. **Cache calculations**: Store computed metrics in index.stats
4. **Limit output**: Always respect --top parameter
