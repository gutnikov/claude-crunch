# Knowledge Entry Schemas

<!-- TEMPLATE: Reference schemas for knowledge entries -->

## Entry ID Format

```
KE-{type}-{YYYYMMDD}-{hash}

type: decision | resolution | pattern | feedback | incident | remediation_pattern
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
      "feedback": 0,
      "incident": 0,
      "remediation_pattern": 0
    },
    "by_domain": {}
  },
  "entries": [],
  "tag_index": {},
  "issue_index": {},
  "agent_metrics": {},
  "remediation_metrics": {
    "total_attempts": 0,
    "successful": 0,
    "failed": 0,
    "by_action": {
      "restart_pod": { "attempts": 0, "success": 0 },
      "flush_cache": { "attempts": 0, "success": 0 },
      "reload_config": { "attempts": 0, "success": 0 }
    },
    "by_anomaly_type": {
      "error_spike": { "attempts": 0, "transient": 0, "bug": 0 },
      "latency_regression": { "attempts": 0, "transient": 0, "bug": 0 },
      "new_exception": { "attempts": 0, "transient": 0, "bug": 0 },
      "memory_growth": { "attempts": 0, "transient": 0, "bug": 0 }
    }
  }
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

### Staging Incident Entry

Records staging validation failures for future pattern matching.

```json
{
  "id": "KE-incident-20250114-e5f6",
  "type": "incident",
  "content": {
    "issue_ref": "#42",
    "environment": "staging",
    "phase": "validation",
    "cycle": "3 of 4",
    "anomalies": [
      {
        "type": "error_spike | latency_regression | new_exception | memory_growth",
        "current_value": "5.2/min",
        "baseline_value": "0.1/min",
        "threshold": "0.3/min",
        "severity": "CRITICAL | HIGH | MEDIUM"
      }
    ],
    "symptoms": {
      "error_pattern": "NullPointerException in AuthService",
      "stack_trace": "Truncated stack trace",
      "affected_endpoints": ["/api/auth/validate"],
      "log_samples": ["Sample log lines"]
    },
    "diagnosis": {
      "classification": "transient | likely_transient | likely_bug | bug",
      "confidence": 92,
      "root_cause": "Connection pool exhaustion after deploy",
      "affected_components": ["AuthService", "DatabasePool"]
    },
    "remediation": {
      "attempted": true,
      "action": "restart_pod",
      "success": true,
      "duration_seconds": 45,
      "metrics_before": {
        "error_rate": 5.2,
        "latency_p99": 500,
        "memory_mb": 512
      },
      "metrics_after": {
        "error_rate": 0.15,
        "latency_p99": 120,
        "memory_mb": 280
      }
    },
    "outcome": "resolved_by_remediation | returned_to_implementing | investigation_needed",
    "resolution_type": "transient | bug_fix | unknown"
  }
}
```

### Remediation Pattern Entry

Records successful remediation patterns for matching future incidents.

```json
{
  "id": "KE-remediation-20250114-f7g8",
  "type": "remediation_pattern",
  "content": {
    "name": "Cold Start Connection Pool Exhaustion",
    "anomaly_signature": {
      "type": "error_spike",
      "error_patterns": ["ConnectionPoolExhausted", "timeout acquiring connection"],
      "timing": "within 5 minutes of deploy",
      "affected_components": ["database", "connection pool"]
    },
    "classification": "transient",
    "recommended_action": "restart_pod",
    "success_rate": {
      "total_attempts": 12,
      "successful": 11,
      "failed": 1,
      "rate": 0.917
    },
    "typical_resolution_time": "2-3 minutes",
    "when_to_use": [
      "Error spike immediately after deploy",
      "Connection-related errors",
      "Errors decrease over time"
    ],
    "when_not_to_use": [
      "Errors persist after restart",
      "Stack trace shows application code bug",
      "Memory growing linearly"
    ],
    "related_incidents": ["KE-incident-20250110-a1b2", "KE-incident-20250112-c3d4"],
    "last_successful": "2025-01-14T10:30:00Z"
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
