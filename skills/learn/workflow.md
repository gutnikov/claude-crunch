# Learn Skill - Workflow Implementation

## Overview

The `/learn` skill captures knowledge and stores it in the project's knowledge base at `.claude/knowledge/`.

## Workflow Phases

### Phase 1: Parse Arguments

```
1. Parse command line arguments:
   - --issue (-i): Issue number
   - --type (-t): Knowledge type
   - --from-review: Capture from review
   - --content (-c): Explicit content
   - --tags: Additional tags

2. Determine capture mode:
   - IF issue specified → Issue Resolution Mode
   - IF from-review → Review Feedback Mode
   - IF content + type specified → Explicit Mode
   - ELSE → Interactive Mode
```

### Phase 2: Gather Source Data

#### Issue Resolution Mode

```
1. Fetch issue via CI MCP:
   - Issue title and body
   - Labels (for domain tags)
   - State history
   - Linked PRs

2. Fetch branch diff:
   - Files changed
   - Commit messages

3. Extract from issue body:
   - Problem description
   - Root cause (if documented)
   - Solution approach
   - DoD checklist results

4. Calculate metrics:
   - Time to resolve (created → closed)
   - Agents used (from workflow log if available)
```

#### Review Feedback Mode

```
1. Find most recent review:
   - Check conversation context for /review output
   - OR fetch from PR comments via CI MCP

2. Extract findings:
   - Filter to confidence >= 80
   - Parse finding details:
     - Description
     - Severity
     - File context
     - Recommendation

3. For each finding:
   - Check for similar existing feedback
   - Calculate similarity score
```

#### Explicit Mode

```
1. Validate required fields by type:
   - decision: title, decision text
   - pattern: name, description
   - feedback: finding, recommendation

2. IF missing required fields:
   - Prompt user for missing information
```

#### Interactive Mode

```
1. Ask user: "What type of knowledge?"
   - Options: Decision, Resolution, Pattern, Feedback

2. Based on type, prompt for required fields

3. Ask for optional context:
   - Related issues
   - Domain tags
   - Additional notes
```

### Phase 3: Structure Entry

#### Generate Entry ID

```
id = "KE-{type}-{YYYYMMDD}-{hash}"

Where:
- type: decision | resolution | pattern | feedback
- YYYYMMDD: Current date
- hash: First 4 chars of SHA-256(content)
```

#### Build Entry Object

```json
{
  "id": "{generated_id}",
  "type": "{type}",
  "created_at": "{ISO-8601 timestamp}",
  "updated_at": "{ISO-8601 timestamp}",
  "decay_score": 1.0,
  "tags": ["{extracted + explicit tags}"],
  "source": {
    "type": "{issue | review | manual}",
    "ref": "{issue number or PR}",
    "phase": "{workflow phase if applicable}"
  },
  "content": {
    // Type-specific content - see schemas
  },
  "relationships": {
    "related_issues": [],
    "related_entries": [],
    "supersedes": null,
    "superseded_by": null
  }
}
```

### Phase 4: Check Duplicates

```
1. Load index from .claude/knowledge/index.json

2. Find candidates of same type

3. For each candidate:
   - Calculate content similarity
   - IF similarity > 0.85:
     - EXACT duplicate (> 0.95): Skip, report "Already captured"
     - Similar (0.85-0.95): Add to relationships, continue

4. Return duplicate status and related entries
```

### Phase 5: Store Entry

```
1. Ensure directory exists:
   .claude/knowledge/{type}/

2. Write entry file:
   .claude/knowledge/{type}/{id}.json

3. Update index.json:
   - Add entry summary to entries[]
   - Update tag_index
   - Update issue_index (if issue ref)
   - Increment stats counters
   - Update last_updated timestamp

4. Link related entries:
   - Update relationships in related entries
```

### Phase 6: Check Pattern Promotion

```
IF type == "feedback":
  1. Query similar feedback entries:
     - Same category
     - Similarity > 0.7

  2. IF count >= pattern_promotion_threshold (default: 3):
     - Invoke pattern promotion workflow
     - Create KE-pattern-* entry
     - Link all source feedback entries
     - Report promotion to user
```

### Phase 7: Report Results

```
Output to user:
1. Entry ID created
2. Summary of captured content
3. Tags assigned
4. Related entries found
5. Pattern promotion (if triggered)
```

---

## Entry Templates

### Resolution Entry Template

```json
{
  "content": {
    "issue_title": "{from issue}",
    "issue_type": "{bug | feature}",
    "problem": "{extracted problem description}",
    "root_cause": {
      "category": "{concurrency | validation | auth | logic | config | other}",
      "description": "{root cause analysis}",
      "code_location": "{file:line if identified}"
    },
    "solution": {
      "approach": "{brief summary}",
      "description": "{detailed explanation}",
      "code_example": "{key snippet if applicable}"
    },
    "files_changed": ["{list from diff}"],
    "tests_added": ["{test descriptions}"],
    "time_to_resolve": "{duration}",
    "agents_used": ["{agents from workflow}"]
  }
}
```

### Decision Entry Template

```json
{
  "content": {
    "title": "{decision title}",
    "decision": "{what was decided}",
    "rationale": "{why this decision}",
    "context": "{what led to this decision}",
    "alternatives": [
      {
        "option": "{alternative}",
        "rejected_because": "{reason}"
      }
    ],
    "consequences": {
      "positive": ["{benefits}"],
      "negative": ["{drawbacks}"]
    },
    "status": "active",
    "review_date": null
  }
}
```

### Pattern Entry Template

```json
{
  "content": {
    "name": "{pattern name}",
    "pattern_type": "{pattern | anti-pattern}",
    "description": "{what this pattern is}",
    "problem_it_solves": "{for patterns}",
    "problem_it_causes": "{for anti-patterns}",
    "when_to_use": "{applicability}",
    "when_not_to_use": "{exceptions}",
    "example": {
      "language": "{language}",
      "code": "{example code}",
      "explanation": "{why this works}"
    },
    "counter_example": null,
    "related_patterns": [],
    "domains": ["{applicable domains}"],
    "occurrences": 1
  }
}
```

### Feedback Entry Template

```json
{
  "content": {
    "finding": "{what was found}",
    "severity": "{CRITICAL | HIGH | MEDIUM | LOW | INFO}",
    "category": "{security | quality | performance | style | other}",
    "file_context": {
      "file": "{file path}",
      "line_start": 0,
      "line_end": 0,
      "code_snippet": "{relevant code}"
    },
    "recommendation": "{how to fix}",
    "remediation_applied": false,
    "similar_findings_count": 0,
    "may_become_pattern": false
  }
}
```

---

## Auto-Capture Integration

### DONE Transition (Resolution)

When `/crunch` transitions an issue to DONE:

```
1. Automatically invoke /learn -i {issue_number}
2. Extract resolution details from issue body
3. Store as KE-resolution-*
4. Link to any decisions made during ENRICH
5. Update agent metrics
```

### ENRICH Exit (Decision)

When spec/investigation is created during ENRICH:

```
1. Analyze spec content for decisions
2. IF architectural decision detected:
   - Extract decision, rationale, alternatives
   - Invoke /learn -t decision -c "{extracted}"
   - Link to issue
```

### Code Review Complete (Feedback)

After /review posts findings:

```
1. FOR each finding with confidence >= 80:
   - Invoke /learn --from-review (targeting that finding)
   - Check pattern promotion threshold
2. Report captured feedback count
```

---

## Error Handling

| Error | Handling |
|-------|----------|
| Knowledge directory missing | Create `.claude/knowledge/` structure |
| Index file missing | Initialize empty index |
| Duplicate entry detected | Report and skip (or merge if requested) |
| Invalid entry schema | Report validation errors, don't store |
| CI MCP unavailable | Fall back to manual input |

---

## Progress Tracking

Use TodoWrite to track capture progress for batch operations:

```
- Fetching issue #42 details
- Extracting resolution information
- Checking for duplicates
- Storing entry
- Updating index
- Checking pattern promotion
```
