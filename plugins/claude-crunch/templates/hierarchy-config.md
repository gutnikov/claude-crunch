# Agent Hierarchy Configuration

<!-- TEMPLATE: Defines the 4-tier agent hierarchy for multi-agent orchestration -->

## Overview

Agents are organized in a 4-tier hierarchy that determines authority, escalation paths, and coordination patterns.

## Hierarchy Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                          TIER 1: ORCHESTRATOR                        │
│                                                                      │
│  orchestrator                                                        │
│  - Meta-coordination for complex workflows                           │
│  - Team composition and execution planning                           │
│  - Conflict resolution and checkpoint management                     │
│  - Activated for HIGH+ complexity (score >= 5)                       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
       ┌───────────────────────┼───────────────────────┐
       │                       │                       │
       ▼                       ▼                       ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ TIER 2:         │  │ TIER 2:         │  │ TIER 2:         │
│ SUPERVISOR      │  │ SUPERVISOR      │  │ SUPERVISOR      │
│                 │  │                 │  │                 │
│ system-architect│  │security-analyst │  │ devops-engineer │
│                 │  │                 │  │                 │
│ Design authority│  │Security authority│ │Infra authority  │
│ Spec approval   │  │VETO power       │  │Deploy approval  │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        TIER 3: SPECIALISTS                          │
│                                                                      │
│  Implementation:  dev-cpp, dev-go, dev-python, dev-react            │
│  Quality:         reviewer, qa-engineer                              │
│  Documentation:   techwriter                                         │
│  Context:         knowledge-manager                                  │
│                                                                      │
│  - Execute assigned tasks within their domain                        │
│  - Escalate to supervisors when authority needed                     │
│  - Can hand off to other specialists                                 │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        TIER 4: RESPONDERS                           │
│                                                                      │
│  incident-responder   - Staging anomaly diagnosis                   │
│  log-analyst          - Log pattern detection                       │
│  code-health-analyst  - Code metrics and tech debt                  │
│  dependency-manager   - Vulnerability and update scanning           │
│                                                                      │
│  - Monitor and detect issues                                         │
│  - Report findings to specialists/supervisors                        │
│  - Support validation and patrol workflows                           │
└─────────────────────────────────────────────────────────────────────┘
```

## Tier Definitions

### Tier 1: Orchestrator

| Agent        | Capabilities                                  | When Active                                 |
| ------------ | --------------------------------------------- | ------------------------------------------- |
| orchestrator | coordinate, resolve, route, checkpoint, merge | Complexity >= HIGH, multi-domain, conflicts |

**Authority**:

- Assign tasks to any agent
- Resolve conflicts between agents
- Create and manage checkpoints
- Escalate to user when needed

### Tier 2: Supervisors

| Agent            | Domain                     | Authority                     |
| ---------------- | -------------------------- | ----------------------------- |
| system-architect | Design, architecture       | Final say on design decisions |
| security-analyst | Security, auth, crypto     | VETO on security matters      |
| devops-engineer  | Infrastructure, deployment | Final say on infra decisions  |

**Authority**:

- Make binding decisions in their domain
- Review and approve specialist work
- Escalate to orchestrator for cross-domain conflicts
- security-analyst has VETO power (see veto-rules.md)

### Tier 3: Specialists

| Agent             | Focus Area                      |
| ----------------- | ------------------------------- |
| dev-cpp           | C++ implementation              |
| dev-go            | Go implementation               |
| dev-python        | Python implementation           |
| dev-react         | TypeScript/React implementation |
| reviewer          | Code quality review             |
| qa-engineer       | Test planning and verification  |
| techwriter        | Documentation                   |
| knowledge-manager | Context and learning            |

**Authority**:

- Execute tasks within their expertise
- Provide recommendations (not binding without supervisor approval for critical decisions)
- Hand off to other specialists
- Request supervisor review

### Tier 4: Responders

| Agent               | Focus Area                |
| ------------------- | ------------------------- |
| incident-responder  | Staging anomaly diagnosis |
| log-analyst         | Log pattern detection     |
| code-health-analyst | Code metrics, tech debt   |
| dependency-manager  | CVE scanning, updates     |

**Authority**:

- Monitor and detect issues
- Report findings (not make decisions)
- Support other agents with data
- Flag issues for specialist/supervisor attention

## Escalation Paths

### Standard Escalation

```
Specialist/Responder → Supervisor → Orchestrator → User

Example:
dev-python detects security concern
  → escalates to security-analyst
    → security-analyst makes decision

If security-analyst and system-architect disagree:
  → orchestrator resolves via debate protocol

If debate cannot resolve:
  → escalate to user with options
```

### Veto Escalation

```
Any agent recommendation
  → security-analyst reviews
    → VETO if security risk
      → CRITICAL: Immediate block
      → HIGH: User notified, can override
```

### Emergency Escalation

```
Any tier can escalate directly to user when:
  - Human judgment explicitly required
  - Ethical concerns identified
  - Task exceeds agent capabilities
  - User explicitly requested involvement
```

## Domain Routing

| Domain Keywords                            | Primary Supervisor | Primary Specialist   |
| ------------------------------------------ | ------------------ | -------------------- |
| auth, token, JWT, security, XSS, injection | security-analyst   | (varies by language) |
| deploy, CI/CD, docker, k8s, terraform      | devops-engineer    | (varies by language) |
| architecture, design, API, schema          | system-architect   | (varies by language) |
| React, component, hook, UI, CSS            | system-architect   | dev-react            |
| Python, Django, Flask, FastAPI             | system-architect   | dev-python           |
| Go, goroutine, gRPC                        | system-architect   | dev-go               |
| C++, memory, STL, CMake                    | system-architect   | dev-cpp              |

## Concurrent Operations

| Tier         | Max Concurrent | Notes                        |
| ------------ | -------------- | ---------------------------- |
| Orchestrator | 1              | Singleton per workflow       |
| Supervisors  | 2 each         | Can work in parallel         |
| Specialists  | As needed      | Limited by task dependencies |
| Responders   | 2 each         | Background monitoring        |

## Authority Matrix

| Decision Type           | Authority Level          | Can Veto                         |
| ----------------------- | ------------------------ | -------------------------------- |
| Design approach         | Supervisor (architect)   | security-analyst                 |
| Security implementation | Supervisor (security)    | N/A (final authority)            |
| Infrastructure          | Supervisor (devops)      | security-analyst                 |
| Code style              | Specialist (reviewer)    | No                               |
| Test coverage           | Specialist (qa-engineer) | No                               |
| Documentation           | Specialist (techwriter)  | No                               |
| Conflict resolution     | Orchestrator             | security-analyst (security only) |
| Final escalation        | User                     | N/A                              |

## Configuration Options

### In `.claude/orchestration-config.yaml`:

```yaml
hierarchy:
  orchestrator_threshold: 5
  supervisor_approval_required:
    - security
    - breaking_change
  auto_escalate_on_conflict: true
  max_debate_rounds: 3

tier_overrides:
  dev-python:
    tier: supervisor
    reason: Project lead is Python expert
```

### Customization

Projects can adjust the hierarchy by:

1. Promoting specialists to supervisor tier for specific expertise
2. Adjusting complexity thresholds
3. Adding project-specific escalation rules
4. Configuring veto scope
