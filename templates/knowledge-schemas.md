# Knowledge Entry Schemas

<!-- TEMPLATE: Reference schemas for knowledge entries -->

## Entry ID Format

```
KE-{type}-{YYYYMMDD}-{hash}

type: decision | resolution | pattern | feedback
hash: first 4 chars of SHA-256 of content
```

## Index Schema

**File**: `.claude/knowledge/index.json`

```json
{
  "version": "1.0",
  "last_updated": "ISO-8601 timestamp",
  "stats": {
    "total_entries": 0,
    "by_type": {
      "decision": 0,
      "resolution": 0,
      "pattern": 0,
      "feedback": 0
    },
    "by_domain": {}
  },
  "entries": [],
  "tag_index": {},
  "issue_index": {},
  "agent_metrics": {}
}
```

## Entry Schemas

### Base Entry Structure

All entries share this base structure:

```json
{
  "id": "KE-{type}-{date}-{hash}",
  "type": "decision | resolution | pattern | feedback",
  "created_at": "ISO-8601",
  "updated_at": "ISO-8601",
  "decay_score": 1.0,
  "tags": ["domain:security", "auth"],
  "source": {
    "type": "issue | review | manual | accumulated",
    "ref": "#42",
    "phase": "enrich | done | review"
  },
  "content": {},
  "relationships": {
    "related_issues": [],
    "related_entries": [],
    "supersedes": null,
    "superseded_by": null
  }
}
```

### Decision Entry

```json
{
  "id": "KE-decision-20250114-a3f2",
  "type": "decision",
  "content": {
    "title": "Decision title",
    "decision": "What was decided",
    "rationale": "Why this decision",
    "context": "What led to needing this decision",
    "alternatives": [
      {
        "option": "Alternative option",
        "rejected_because": "Reason for rejection"
      }
    ],
    "consequences": {
      "positive": ["Benefits"],
      "negative": ["Drawbacks"]
    },
    "status": "active | superseded | deprecated",
    "review_date": "Optional date to revisit"
  }
}
```

### Resolution Entry

```json
{
  "id": "KE-resolution-20250114-b7c9",
  "type": "resolution",
  "content": {
    "issue_title": "Issue title",
    "issue_type": "bug | feature",
    "problem": "Description of what needed fixing/building",
    "root_cause": {
      "category": "concurrency | validation | auth | logic | config",
      "description": "What caused the issue",
      "code_location": "file:line"
    },
    "solution": {
      "approach": "Brief summary",
      "description": "Detailed explanation",
      "code_example": "Optional key snippet"
    },
    "files_changed": ["file paths"],
    "tests_added": ["test descriptions"],
    "time_to_resolve": "duration",
    "agents_used": ["agent names"]
  }
}
```

### Pattern Entry

```json
{
  "id": "KE-pattern-20250114-c8d3",
  "type": "pattern",
  "content": {
    "name": "Pattern name",
    "pattern_type": "pattern | anti-pattern",
    "description": "What this pattern is",
    "problem_it_solves": "For patterns",
    "problem_it_causes": "For anti-patterns",
    "when_to_use": "Applicability",
    "when_not_to_use": "Exceptions",
    "example": {
      "language": "typescript",
      "code": "Example code",
      "explanation": "Why this works"
    },
    "counter_example": {
      "language": "typescript",
      "code": "What NOT to do",
      "explanation": "Why this is bad"
    },
    "related_patterns": ["pattern names"],
    "domains": ["applicable domains"],
    "occurrences": 0
  }
}
```

### Feedback Entry

```json
{
  "id": "KE-feedback-20250114-d9e4",
  "type": "feedback",
  "content": {
    "finding": "What was found",
    "severity": "CRITICAL | HIGH | MEDIUM | LOW | INFO",
    "category": "security | quality | performance | style",
    "file_context": {
      "file": "file path",
      "line_start": 0,
      "line_end": 0,
      "code_snippet": "relevant code"
    },
    "recommendation": "How to fix",
    "remediation_applied": false,
    "similar_findings_count": 0,
    "may_become_pattern": false
  }
}
```

## Agent Metrics Schema

Stored in index.json under `agent_metrics`:

```json
{
  "agent_metrics": {
    "security-analyst": {
      "total_invocations": 0,
      "by_domain": {
        "security": {
          "count": 0,
          "success_rate": 0.0,
          "avg_duration": "0h"
        }
      },
      "by_phase": {
        "enrich": { "count": 0, "success_rate": 0.0 },
        "review": { "count": 0, "success_rate": 0.0 }
      }
    }
  }
}
```
