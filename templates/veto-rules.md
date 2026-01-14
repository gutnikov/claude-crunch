# Veto Authority Rules

<!-- TEMPLATE: Defines which agents have veto power and under what conditions -->

## Overview

Veto authority allows designated agents to block recommendations that would introduce unacceptable risks. This is a safety mechanism to prevent security vulnerabilities, infrastructure failures, or compliance violations from being implemented.

## Veto Holders

### Security Analyst - Full Veto Authority

The `security-analyst` agent has veto authority on security matters.

**Veto Scope**:
| Domain | Can Veto |
|--------|----------|
| Authentication | Yes |
| Authorization | Yes |
| Cryptography | Yes |
| Data exposure | Yes |
| Input validation | Yes |
| Injection vulnerabilities | Yes |
| Session management | Yes |
| Compliance requirements | Yes |

**Cannot Veto**:

- Code style preferences (unless security-relevant)
- Performance-only optimizations (unless security-relevant)
- Documentation completeness
- Test coverage levels (unless for security tests)
- UI/UX decisions (unless they expose data)

### DevOps Engineer - Limited Veto Authority

The `devops-engineer` agent has limited veto authority on infrastructure matters.

**Veto Scope** (HIGH severity only, no CRITICAL):
| Domain | Can Veto |
|--------|----------|
| Deployment configurations that will fail | Yes |
| Resource exhaustion risks | Yes |
| Missing monitoring for critical paths | Yes |
| Compliance/audit logging gaps | Yes |

**Cannot Veto**:

- Application logic decisions
- Security implementations (defers to security-analyst)
- Code quality matters

## Veto Severity Levels

### CRITICAL - Immediate Block

**Effect**: Vetoed recommendation is immediately blocked. Veto agent's alternative is applied.

**Criteria for CRITICAL**:

- Exploitable vulnerability with known attack vector
- Direct path to data breach
- Authentication/authorization bypass
- Remote code execution risk
- Privilege escalation vulnerability
- Deployment that will cause outage

**Examples**:

```
CRITICAL VETO: SQL injection vulnerability
- Target: "Use string concatenation for user input in query"
- Reason: "CWE-89: SQL injection. User input directly in query enables data exfiltration."
- Alternative: "Use parameterized queries with prepared statements"

CRITICAL VETO: Hardcoded credentials
- Target: "Store API key in source code for convenience"
- Reason: "CWE-798: Hardcoded credentials. Keys in source will be exposed in git history."
- Alternative: "Use environment variables or Vault for secret management"

CRITICAL VETO: Auth bypass
- Target: "Skip token validation for internal endpoints"
- Reason: "Authentication bypass. Internal endpoints reachable via SSRF attacks."
- Alternative: "Validate tokens on all endpoints, use separate internal auth if needed"
```

### HIGH - User Escalation

**Effect**: User is notified with the veto reason and options. User can override with acknowledgment.

**Criteria for HIGH**:

- Security weakness without immediate exploit path
- Missing defense-in-depth layer
- Weak cryptographic choices
- Insufficient logging for security events
- Insecure default configurations

**Examples**:

```
HIGH VETO: Weak password policy
- Target: "Allow 6-character passwords for easier onboarding"
- Reason: "Weak password policy increases brute force risk."
- Alternative: "Require 12+ characters with complexity rules"
- User can override: Yes, with business justification

HIGH VETO: Missing rate limiting
- Target: "Skip rate limiting for MVP to reduce complexity"
- Reason: "Missing rate limiting enables credential stuffing attacks."
- Alternative: "Implement basic rate limiting (100 req/min per IP)"
- User can override: Yes, if accepting risk for MVP

HIGH VETO: Excessive permissions
- Target: "Grant admin role for easier debugging"
- Reason: "Principle of least privilege violated. Over-permissioned accounts increase blast radius."
- Alternative: "Create specific debug role with minimal required permissions"
- User can override: Yes, for development environments only
```

## Veto Process

### 1. Veto Declaration

When a veto-holding agent identifies a vetoed condition:

```json
{
  "type": "VETO",
  "from": "security-analyst",
  "severity": "CRITICAL|HIGH",
  "target": {
    "recommendation": "What is being vetoed",
    "source_agent": "Agent that made the recommendation",
    "context": "Where this recommendation appears"
  },
  "reason": {
    "summary": "Brief explanation",
    "vulnerability_type": "CWE-XXX or description",
    "risk_assessment": "What could happen if implemented",
    "references": ["Links to standards, CVEs, or documentation"]
  },
  "alternative": {
    "recommendation": "Secure alternative approach",
    "implementation_notes": "How to implement the alternative",
    "trade_offs": "Any downsides of the alternative"
  }
}
```

### 2. Veto Processing

```
CRITICAL veto received:
  1. Block original recommendation immediately
  2. Record veto in conflict log
  3. Apply alternative automatically
  4. Notify user of veto (informational)
  5. Continue workflow with alternative

HIGH veto received:
  1. Pause recommendation implementation
  2. Record veto in conflict log
  3. Notify user with full context
  4. Present options:
     a. Accept veto (apply alternative)
     b. Override veto (acknowledge risk)
     c. Request clarification
  5. Await user decision
  6. Record decision and continue
```

### 3. User Override Process

For HIGH severity vetoes, users can override:

```json
{
  "override": {
    "veto_id": "VETO-20250114-a3f2",
    "user_decision": "override",
    "justification": "Required for MVP launch timeline",
    "risk_acknowledgment": [
      "Understood: Missing rate limiting increases credential stuffing risk",
      "Accepted: Will implement in v1.1 within 30 days"
    ],
    "mitigation_plan": {
      "follow_up_issue": "#43",
      "deadline": "2025-02-14",
      "interim_measures": [
        "Monitor for unusual login patterns",
        "Alert on >10 failed logins"
      ]
    }
  }
}
```

**Override requirements**:

- Written justification
- Explicit risk acknowledgment
- Mitigation plan (recommended)
- Follow-up issue (recommended)

**Override restrictions**:

- CRITICAL vetoes cannot be overridden
- Overrides are logged permanently
- Multiple overrides trigger review

## Veto Log

All vetoes are logged to `.claude/acp/veto-log.json`:

```json
{
  "vetoes": [
    {
      "id": "VETO-20250114-a3f2",
      "timestamp": "2025-01-14T10:30:00Z",
      "issue_number": 42,
      "veto_agent": "security-analyst",
      "severity": "HIGH",
      "target_summary": "Skip rate limiting for MVP",
      "outcome": "overridden",
      "override_by": "user",
      "follow_up_issue": "#43"
    }
  ],
  "statistics": {
    "total_vetoes": 15,
    "critical": 5,
    "high": 10,
    "overridden": 2,
    "by_category": {
      "injection": 3,
      "auth": 4,
      "crypto": 2,
      "exposure": 6
    }
  }
}
```

## Non-Vetoable Decisions

Some decisions cannot be vetoed:

1. **User explicit requests** - If user explicitly requests a known-risky approach with justification
2. **External requirements** - Compliance or regulatory requirements from external sources
3. **Business decisions** - Pure business logic unrelated to security/infrastructure
4. **Orchestrator rulings** - Final conflict resolutions after full debate

Even for non-vetoable decisions, veto-holding agents should:

- Document security concerns
- Recommend mitigations
- Create follow-up issues if risks accepted

## Appeal Process

If an agent believes a veto was incorrectly applied:

1. **Request review** - Provide counter-evidence
2. **Escalate to orchestrator** - For second opinion
3. **Escalate to user** - Final arbiter
4. **Record precedent** - Document outcome for future reference

## Integration with Knowledge Base

Vetoes contribute to learning:

1. **Pattern detection** - Repeated similar vetoes become anti-patterns
2. **Agent calibration** - Veto override rates inform weight adjustments
3. **Documentation** - Common veto reasons become CLAUDE.md guidance
