# Agent Communication Protocol (ACP) Schema

<!-- TEMPLATE: Defines the formal message contracts for multi-agent communication -->

## Overview

The Agent Communication Protocol (ACP) enables structured communication between agents in the claude-crunch system. All agent interactions use these message formats for consistent, traceable coordination.

## Message Types

| Type        | Direction             | Purpose                                               |
| ----------- | --------------------- | ----------------------------------------------------- |
| `REQUEST`   | Orchestrator -> Agent | Task assignment with context and constraints          |
| `RESPONSE`  | Agent -> Orchestrator | Task completion with results and findings             |
| `HANDOFF`   | Agent -> Agent        | Direct delegation when another agent is better suited |
| `ESCALATE`  | Agent -> Supervisor   | Issue needs higher authority or human input           |
| `BROADCAST` | Orchestrator -> All   | Context update to all active agents                   |
| `VOTE`      | Agent -> Orchestrator | Input for conflict resolution                         |

## Message Structure

### Base Message Schema

All messages share this base structure:

```json
{
  "id": "MSG-{timestamp}-{hash}",
  "type": "REQUEST|RESPONSE|HANDOFF|ESCALATE|BROADCAST|VOTE",
  "from": "{agent_name}",
  "to": "{agent_name|*}",
  "correlation_id": "{parent_message_id}",
  "timestamp": "ISO-8601",
  "priority": "CRITICAL|HIGH|NORMAL|LOW",
  "payload": {},
  "metadata": {
    "issue_number": 0,
    "phase": "{workflow_phase}",
    "attempt": 1,
    "checkpoint_id": "{optional_checkpoint}"
  }
}
```

### Field Definitions

| Field            | Type   | Required | Description                                                |
| ---------------- | ------ | -------- | ---------------------------------------------------------- |
| `id`             | string | Yes      | Unique message identifier (MSG-{unix_ms}-{first_8_sha256}) |
| `type`           | enum   | Yes      | One of the six message types                               |
| `from`           | string | Yes      | Sending agent name                                         |
| `to`             | string | Yes      | Receiving agent name or `*` for broadcast                  |
| `correlation_id` | string | No       | Parent message ID for threading                            |
| `timestamp`      | string | Yes      | ISO-8601 timestamp                                         |
| `priority`       | enum   | Yes      | Message priority level                                     |
| `payload`        | object | Yes      | Type-specific payload                                      |
| `metadata`       | object | Yes      | Contextual metadata                                        |

---

## Type-Specific Payloads

### REQUEST Payload

```json
{
  "payload": {
    "action": "{action_type}",
    "context": {
      "issue": {
        "number": 42,
        "title": "Issue title",
        "body": "Issue body content",
        "labels": ["type:feature", "domain:security"],
        "state": "enrich"
      },
      "knowledge_brief": {
        "related_issues": [],
        "patterns": [],
        "decisions": []
      },
      "files": ["relevant/file/paths"]
    },
    "constraints": {
      "timeout_ms": 300000,
      "max_output_tokens": 4000,
      "required_outputs": ["spec", "test_plan"],
      "forbidden_actions": []
    },
    "deadline": "ISO-8601"
  }
}
```

**Action Types**:

- `create_spec` - Create technical specification
- `review_code` - Perform code review
- `threat_model` - Create threat model
- `implement` - Implement changes
- `create_test_plan` - Create test plan
- `validate` - Validate implementation
- `diagnose` - Diagnose anomaly
- `document` - Create documentation

### RESPONSE Payload

```json
{
  "payload": {
    "status": "SUCCESS|PARTIAL|BLOCKED|FAILED",
    "result": {
      "summary": "Brief summary of what was done",
      "outputs": {
        "spec": "...",
        "findings": []
      },
      "files_modified": [],
      "files_created": []
    },
    "findings": [
      {
        "id": "F-001",
        "severity": "CRITICAL|HIGH|MEDIUM|LOW|INFO",
        "category": "security|quality|performance|style",
        "title": "Finding title",
        "description": "Detailed description",
        "location": "file:line",
        "recommendation": "How to fix",
        "confidence": 85
      }
    ],
    "metrics": {
      "duration_ms": 45000,
      "tokens_used": 3500
    },
    "handoff": {
      "to": "{agent_name}",
      "reason": "Why handoff is needed"
    }
  }
}
```

**Status Values**:

- `SUCCESS` - Task completed fully
- `PARTIAL` - Task partially completed, some outputs available
- `BLOCKED` - Cannot proceed, needs input or different agent
- `FAILED` - Task failed, see error in result

### HANDOFF Payload

```json
{
  "payload": {
    "reason": "Why this agent cannot complete the task",
    "recommended_agent": "{agent_name}",
    "context_transfer": {
      "work_completed": "What has been done",
      "remaining_work": "What needs to be done",
      "relevant_findings": []
    },
    "urgency": "IMMEDIATE|NORMAL"
  }
}
```

### ESCALATE Payload

```json
{
  "payload": {
    "escalation_type": "VETO|CONFLICT|BLOCKED|HUMAN_REQUIRED",
    "severity": "CRITICAL|HIGH",
    "reason": "Why escalation is needed",
    "context": {
      "decision_needed": "What decision is required",
      "options": [
        {
          "option": "Option A",
          "pros": ["benefits"],
          "cons": ["drawbacks"]
        }
      ],
      "recommendation": "Recommended option if any",
      "deadline": "When decision is needed by"
    },
    "veto_details": {
      "target": "What is being vetoed",
      "vulnerability": "CWE-XXX or description",
      "alternative": "Secure alternative"
    }
  }
}
```

### BROADCAST Payload

```json
{
  "payload": {
    "event": "CONTEXT_UPDATE|CHECKPOINT|PHASE_CHANGE|ABORT",
    "content": {
      "message": "Human-readable message",
      "data": {}
    },
    "affects": ["agent_names_affected"]
  }
}
```

### VOTE Payload

```json
{
  "payload": {
    "conflict_id": "{conflict_identifier}",
    "position": {
      "choice": "Option identifier",
      "rationale": "Why this choice",
      "evidence": ["Supporting evidence"],
      "confidence": 85
    },
    "compromise": {
      "acceptable": true,
      "conditions": ["What would make other option acceptable"]
    },
    "veto": false,
    "veto_reason": null
  }
}
```

---

## Message ID Generation

```
MSG-{timestamp_ms}-{hash_8}

timestamp_ms: Unix timestamp in milliseconds
hash_8: First 8 characters of SHA-256(from + to + timestamp + random)

Example: MSG-1705234567890-a3f2b7c9
```

## Priority Levels

| Priority   | Use Case                      | Timeout Behavior                 |
| ---------- | ----------------------------- | -------------------------------- |
| `CRITICAL` | Security veto, blocking issue | No timeout, immediate processing |
| `HIGH`     | Important task, user waiting  | Shorter timeout, priority queue  |
| `NORMAL`   | Standard task                 | Default timeout                  |
| `LOW`      | Background task, optional     | Can be deferred                  |

---

## Message Log

Messages are logged to `.claude/acp/message-log.yaml`:

```yaml
version: "1.0"
session_id: "{session_uuid}"
issue_number: 42
started_at: "ISO-8601"
messages:
  - id: "MSG-..."
    type: REQUEST
    from: orchestrator
    to: security-analyst
    timestamp: "..."
    summary: Brief summary for log

pending:
  - MSG-ids-awaiting-response
completed:
  - MSG-ids-completed
conflicts:
  - id: CONF-001
    agents:
      - agent_a
      - agent_b
    topic: What they disagree on
    status: pending|resolved
    resolution: null
```

---

## Usage Examples

### Example 1: Orchestrator Assigns Task

```json
{
  "id": "MSG-1705234567890-a3f2b7c9",
  "type": "REQUEST",
  "from": "orchestrator",
  "to": "security-analyst",
  "correlation_id": null,
  "timestamp": "2025-01-14T10:30:00Z",
  "priority": "HIGH",
  "payload": {
    "action": "threat_model",
    "context": {
      "issue": {
        "number": 42,
        "title": "Add JWT refresh token support",
        "labels": ["type:feature", "domain:security", "domain:auth"]
      },
      "knowledge_brief": {
        "related_issues": ["#38 - JWT implementation"],
        "patterns": ["secure-token-storage"]
      }
    },
    "constraints": {
      "timeout_ms": 300000,
      "required_outputs": ["threat_model", "security_requirements"]
    }
  },
  "metadata": {
    "issue_number": 42,
    "phase": "enrich",
    "attempt": 1
  }
}
```

### Example 2: Agent Response

```json
{
  "id": "MSG-1705234600000-b4c5d6e7",
  "type": "RESPONSE",
  "from": "security-analyst",
  "to": "orchestrator",
  "correlation_id": "MSG-1705234567890-a3f2b7c9",
  "timestamp": "2025-01-14T10:31:00Z",
  "priority": "HIGH",
  "payload": {
    "status": "SUCCESS",
    "result": {
      "summary": "Threat model created for JWT refresh token feature",
      "outputs": {
        "threat_model": "...",
        "security_requirements": ["..."]
      }
    },
    "findings": [
      {
        "id": "F-001",
        "severity": "HIGH",
        "category": "security",
        "title": "Refresh token rotation required",
        "description": "Refresh tokens must be rotated on each use",
        "recommendation": "Implement one-time-use refresh tokens",
        "confidence": 95
      }
    ],
    "metrics": {
      "duration_ms": 45000
    }
  },
  "metadata": {
    "issue_number": 42,
    "phase": "enrich",
    "attempt": 1
  }
}
```

### Example 3: Security Veto

```json
{
  "id": "MSG-1705234700000-c5d6e7f8",
  "type": "ESCALATE",
  "from": "security-analyst",
  "to": "orchestrator",
  "correlation_id": "MSG-1705234650000-x9y8z7w6",
  "timestamp": "2025-01-14T10:35:00Z",
  "priority": "CRITICAL",
  "payload": {
    "escalation_type": "VETO",
    "severity": "CRITICAL",
    "reason": "Proposed implementation stores refresh tokens in localStorage",
    "veto_details": {
      "target": "Token storage approach in spec",
      "vulnerability": "CWE-922: Insecure Storage of Sensitive Information",
      "alternative": "Use httpOnly secure cookies for refresh token storage"
    }
  },
  "metadata": {
    "issue_number": 42,
    "phase": "enrich",
    "attempt": 1
  }
}
```

---

## Integration Points

### With Workflow

1. **ENRICH entry**: Orchestrator sends REQUEST to architect/specialists
2. **Code Review**: Orchestrator broadcasts CONTEXT_UPDATE, sends parallel REQUESTs
3. **Conflict detected**: Agents send VOTE messages, orchestrator resolves
4. **Phase transition**: Orchestrator broadcasts PHASE_CHANGE

### With Knowledge Base

- All RESPONSE findings with confidence >= 80 are candidates for knowledge capture
- ESCALATE resolutions are stored as decision entries
- Agent metrics updated from RESPONSE metrics

### With Checkpoints

- BROADCAST with `event: CHECKPOINT` triggers checkpoint save
- Checkpoint includes pending messages for resume
