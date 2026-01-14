# Dynamic Team Composition Algorithm

<!-- TEMPLATE: Defines how teams are composed based on issue complexity and domain -->

## Overview

The team composition algorithm dynamically selects the optimal set of agents for each issue based on complexity, domain requirements, and historical effectiveness.

## Input Factors

### Issue Analysis

```json
{
  "issue": {
    "number": 42,
    "title": "Add JWT refresh token support",
    "type": "feature|bug",
    "domains": ["security", "auth"],
    "labels": ["domain:security", "type:feature"],
    "keywords": ["JWT", "token", "refresh", "auth"],
    "files_affected_estimate": 8
  },
  "constraints": {
    "priority": "P0|P1|P2|P3",
    "deadline": "ISO-8601 or null",
    "budget": "agent_invocations_limit or null"
  },
  "history": {
    "prior_attempts": 0,
    "similar_issues": ["#38", "#25"],
    "known_patterns": ["secure-token-storage"]
  }
}
```

### Complexity Scoring

| Factor | Weight | Threshold | Example |
|--------|--------|-----------|---------|
| Multi-domain | 2 | >= 2 domains | security + frontend = 2 points |
| File count | 1 | >= 10 files | 15 files affected = 1 point |
| Security involved | 3 | Any security domain | domain:security = 3 points |
| Breaking change | 2 | API/schema changes | Changes public API = 2 points |
| Prior failures | 2 | >= 1 failed attempt | 1 retry = 2 points |

```
complexity_score = sum(factor_weight for each threshold met)

LOW:      score < 3   → Direct routing, minimal team
MEDIUM:   3 <= score < 5   → Standard team
HIGH:     5 <= score < 8   → Full team + orchestrator
CRITICAL: score >= 8   → Full team + user oversight
```

## Team Formation Rules

### Minimum Team (all issues)

Every issue gets at minimum:
- 1 domain lead (supervisor tier based on primary domain)
- 1 specialist for implementation (based on language/framework)

### Scaling Rules

| Complexity | Team Size | Composition |
|------------|-----------|-------------|
| LOW | 2 | domain-lead + implementer |
| MEDIUM | 3-4 | + reviewer OR qa-engineer |
| HIGH | 5-6 | + orchestrator + multiple specialists |
| CRITICAL | 6+ | Full team + user oversight gates |

### Domain Multiplier

Each additional domain adds:
- 1 domain-appropriate specialist
- Consider domain supervisor if cross-cutting concerns

## Team Selection Algorithm

```python
def compose_team(issue, knowledge_base):
    team = []

    # Step 1: Calculate complexity
    complexity = calculate_complexity(issue)

    # Step 2: Add orchestrator if needed
    if complexity.score >= 5 or len(issue.domains) >= 2:
        team.append({
            "agent": "orchestrator",
            "role": "coordinator",
            "phases": ["all"]
        })

    # Step 3: Add domain supervisors
    for domain in issue.domains:
        supervisor = get_domain_supervisor(domain)
        if supervisor not in [t["agent"] for t in team]:
            team.append({
                "agent": supervisor,
                "role": "supervisor",
                "phases": ["enrich", "review"]
            })

    # Step 4: Add specialists based on file types
    file_types = analyze_affected_files(issue)
    for file_type in file_types:
        specialist = get_file_specialist(file_type)
        if specialist not in [t["agent"] for t in team]:
            team.append({
                "agent": specialist,
                "role": "implementer",
                "phases": ["implementing"]
            })

    # Step 5: Add quality agents
    if issue.type == "feature" or complexity.score >= 3:
        team.append({
            "agent": "qa-engineer",
            "role": "quality",
            "phases": ["enrich", "implementing"]
        })

    # Always include reviewer
    team.append({
        "agent": "reviewer",
        "role": "quality",
        "phases": ["review"]
    })

    # Step 6: Add knowledge-manager for HIGH+ complexity
    if complexity.score >= 5:
        team.append({
            "agent": "knowledge-manager",
            "role": "context",
            "phases": ["enrich", "done"]
        })

    # Step 7: Apply effectiveness weighting
    team = reorder_by_effectiveness(team, issue.domains, knowledge_base)

    return team
```

## Domain-to-Agent Mapping

### Primary Domain Supervisors

| Domain | Primary Supervisor | Backup |
|--------|-------------------|--------|
| security | security-analyst | system-architect |
| infrastructure | devops-engineer | system-architect |
| architecture | system-architect | - |
| general | system-architect | reviewer |

### Primary Domain Specialists

| Domain Keywords | Primary Specialist | Detection Pattern |
|-----------------|-------------------|-------------------|
| React, TypeScript, component, hook, UI | dev-react | `*.tsx`, `*.jsx`, `*.css` |
| Python, Django, Flask, FastAPI | dev-python | `*.py`, `requirements.txt` |
| Go, goroutine, gRPC | dev-go | `*.go`, `go.mod` |
| C++, CMake, memory | dev-cpp | `*.cpp`, `*.hpp`, `CMakeLists.txt` |
| General/unknown | system-architect | (fallback) |

## Effectiveness Weighting

Query knowledge base for agent performance:

```json
{
  "agent_effectiveness": {
    "security-analyst": {
      "domain:security": {"success_rate": 0.94, "count": 35},
      "domain:auth": {"success_rate": 0.92, "count": 12}
    },
    "dev-python": {
      "domain:python": {"success_rate": 0.87, "count": 50},
      "domain:api": {"success_rate": 0.82, "count": 20}
    }
  }
}
```

### Reordering Algorithm

```python
def reorder_by_effectiveness(team, domains, knowledge_base):
    for member in team:
        agent = member["agent"]
        effectiveness = knowledge_base.get_effectiveness(agent, domains)
        member["routing_score"] = effectiveness.success_rate * effectiveness.count_weight

    # Sort within role groups by routing_score
    return sorted_by_role_then_score(team)
```

### Fallback Rules

| Condition | Action |
|-----------|--------|
| Primary agent success_rate < 60% | Use backup agent |
| All candidates < 60% | Escalate to orchestrator for decision |
| No specialist for file type | Use system-architect |
| Agent timeout on prior attempts | Prefer different agent |

## Team Output Format

```json
{
  "team_id": "TEAM-20250114-a3f2",
  "issue_number": 42,
  "complexity": {
    "score": 6,
    "level": "HIGH",
    "factors": {
      "multi_domain": true,
      "file_count": false,
      "security_involved": true,
      "breaking_change": false,
      "prior_failures": false
    }
  },
  "members": [
    {
      "agent": "orchestrator",
      "role": "coordinator",
      "phases": ["all"],
      "routing_score": 1.0
    },
    {
      "agent": "security-analyst",
      "role": "supervisor",
      "phases": ["enrich", "review"],
      "routing_score": 0.94
    },
    {
      "agent": "system-architect",
      "role": "supervisor",
      "phases": ["enrich"],
      "routing_score": 0.89
    },
    {
      "agent": "dev-python",
      "role": "implementer",
      "phases": ["implementing"],
      "routing_score": 0.87
    },
    {
      "agent": "qa-engineer",
      "role": "quality",
      "phases": ["enrich", "implementing"],
      "routing_score": 0.85
    },
    {
      "agent": "reviewer",
      "role": "quality",
      "phases": ["review"],
      "routing_score": 0.80
    },
    {
      "agent": "knowledge-manager",
      "role": "context",
      "phases": ["enrich", "done"],
      "routing_score": 0.75
    }
  ],
  "phase_assignments": {
    "enrich": ["orchestrator", "security-analyst", "system-architect", "qa-engineer", "knowledge-manager"],
    "implementing": ["orchestrator", "dev-python", "qa-engineer"],
    "review": ["orchestrator", "security-analyst", "reviewer"],
    "done": ["orchestrator", "knowledge-manager"]
  }
}
```

## Phase-Specific Composition

### ENRICH Phase

| Role | Agents | Purpose |
|------|--------|---------|
| Lead | system-architect OR domain supervisor | Create spec |
| Security | security-analyst (if security domain) | Review spec for security |
| Quality | qa-engineer | Create test plan |
| Context | knowledge-manager (HIGH+) | Inject historical context |

### IMPLEMENTING Phase

| Role | Agents | Purpose |
|------|--------|---------|
| Implementer | dev-* (language-specific) | Write code with TDD |
| Quality | qa-engineer | Verify test coverage |
| Context | knowledge-manager (HIGH+) | File-specific patterns |

### VALIDATION Phase

| Role | Agents | Purpose |
|------|--------|---------|
| Reviewers | 6 parallel review agents | Code review |
| Security | security-analyst | Security review |
| Quality | qa-engineer | Test quality review |
| Monitoring | log-analyst, incident-responder | Staging monitoring |

### DOCS Phase

| Role | Agents | Purpose |
|------|--------|---------|
| Documentation | techwriter | Create/update docs |
| Review | reviewer | Review documentation |

## User Override

Users can override team composition:

```bash
/crunch 42 --agents security-analyst,dev-python,reviewer
```

Override is logged and may be used for learning about preferences.

## Integration with Orchestrator

When orchestrator is active, it manages team through the workflow:

1. **Team announcement**: Orchestrator broadcasts team composition
2. **Phase transitions**: Orchestrator signals which agents to activate
3. **Conflict handling**: Orchestrator mediates when team members disagree
4. **Progress tracking**: Orchestrator maintains checkpoint with team state
