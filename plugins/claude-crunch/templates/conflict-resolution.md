# Conflict Resolution Protocol

<!-- TEMPLATE: Defines the 5-stage protocol for resolving disagreements between agents -->

## Overview

When multiple agents provide contradicting recommendations, the conflict resolution protocol ensures systematic, fair resolution while respecting domain expertise and security requirements.

## When Conflicts Occur

Conflicts arise when:

1. Multiple agents recommend different approaches
2. Agent recommendations contradict each other
3. Resource or priority competition exists
4. Security concerns conflict with other requirements
5. Design decisions require trade-off evaluation

## Resolution Stages

### Stage 1: Detection

Conflicts are detected automatically when:

```
conflict_detected = (
  agent_a.recommendation != agent_b.recommendation AND
  both recommendations address same scope AND
  both recommendations are valid/actionable
)
```

**Detection triggers**:

- Contradicting findings in code review
- Different approaches in spec creation
- Incompatible security vs. UX recommendations
- Resource allocation disagreements

**Non-conflicts** (do not trigger resolution):

- Complementary recommendations
- Different severity assessments of same issue
- Additive suggestions

### Stage 2: Structured Debate

Each conflicting agent provides a formal position:

```json
{
  "agent": "{agent_name}",
  "position": {
    "choice": "What I recommend",
    "rationale": "Why this approach is best",
    "evidence": [
      "Supporting fact 1 (from codebase, standards, or knowledge base)",
      "Supporting fact 2"
    ],
    "risks_of_alternative": ["Why the other approach is worse or risky"],
    "compromise": {
      "acceptable": true,
      "conditions": ["What would make other approach acceptable"],
      "hybrid_proposal": "Optional merged approach"
    }
  },
  "confidence": 85,
  "domain_relevance": 0.9
}
```

**Debate rules**:

- Maximum 3 rounds of position refinement
- Each round allows response to other positions
- Compromise proposals encouraged
- Evidence must be verifiable

### Stage 3: Weighted Voting

Calculate final scores for each position:

```
position_score = Σ(agent.weight × agent.confidence × agent.domain_relevance)
```

#### Agent Base Weights

| Agent             | Base Weight | Rationale                   |
| ----------------- | ----------- | --------------------------- |
| security-analyst  | 1.0         | Security is non-negotiable  |
| system-architect  | 0.9         | Design authority            |
| devops-engineer   | 0.8         | Infrastructure expertise    |
| domain dev-\*     | 0.8         | Language-specific expertise |
| reviewer          | 0.7         | Quality perspective         |
| qa-engineer       | 0.7         | Testing perspective         |
| techwriter        | 0.6         | Documentation perspective   |
| knowledge-manager | 0.5         | Historical context          |

#### Domain Boosts

Apply boost when conflict is in agent's domain:

| Agent            | Domain Boost | Applicable Domains                     |
| ---------------- | ------------ | -------------------------------------- |
| security-analyst | +0.5         | security, auth, crypto, data_exposure  |
| system-architect | +0.3         | design, architecture, api, integration |
| devops-engineer  | +0.3         | infra, deployment, monitoring, ci_cd   |
| domain dev-\*    | +0.3         | language-specific issues               |
| reviewer         | +0.2         | quality, style, best_practices         |
| qa-engineer      | +0.2         | testing, coverage, validation          |

#### Scoring Example

```
Conflict: Token storage approach
Agents: security-analyst vs dev-react

security-analyst position:
  base_weight: 1.0
  domain_boost: +0.5 (security domain)
  confidence: 95
  domain_relevance: 1.0
  score = (1.0 + 0.5) × 0.95 × 1.0 = 1.425

dev-react position:
  base_weight: 0.8
  domain_boost: +0.0 (not frontend domain)
  confidence: 80
  domain_relevance: 0.6
  score = (0.8 + 0.0) × 0.80 × 0.6 = 0.384

Winner: security-analyst (1.425 > 0.384)
```

### Stage 4: Veto Check

Before finalizing, check for veto authority:

```
IF any agent invokes veto:
  IF veto.severity == CRITICAL:
    # Immediate block - veto agent's position wins
    resolution = veto_agent.position
    dissent_recorded = true

  ELSE IF veto.severity == HIGH:
    # Escalate to user
    notify_user({
      conflict: conflict_summary,
      veto_reason: veto.reason,
      options: [veto_agent.position, other_positions],
      recommendation: veto_agent.position
    })
    await user_decision
```

**Veto authority**: Only `security-analyst` has veto power (see `veto-rules.md`)

### Stage 5: Resolution

Record and apply the resolution:

```json
{
  "conflict_id": "CONF-{timestamp}-{hash}",
  "topic": "Brief description of conflict",
  "timestamp": "ISO-8601",
  "participants": ["agent_a", "agent_b"],
  "positions": [
    {
      "agent": "agent_a",
      "position": "...",
      "score": 1.425,
      "evidence_count": 3
    },
    {
      "agent": "agent_b",
      "position": "...",
      "score": 0.384,
      "evidence_count": 2
    }
  ],
  "resolution": {
    "winner": "agent_a",
    "winning_position": "...",
    "reasoning": "Security considerations outweigh UX convenience",
    "veto_applied": false,
    "user_override": false
  },
  "dissent": {
    "recorded": true,
    "agent": "agent_b",
    "position": "...",
    "note": "Team chose security over UX in this case"
  },
  "outcome": {
    "applied": true,
    "knowledge_entry": "KE-decision-20250114-a3f2"
  }
}
```

**Resolution actions**:

1. Apply winning position
2. Record dissent for future reference
3. Create knowledge base entry (decision type)
4. Update issue body with resolution summary
5. Continue workflow with resolved approach

## Conflict Types and Handlers

| Conflict Type                   | Primary Arbiter  | Tiebreaker          |
| ------------------------------- | ---------------- | ------------------- |
| Design approach                 | system-architect | User preference     |
| Security vs. functionality      | security-analyst | Security wins       |
| Performance vs. maintainability | reviewer         | Risk assessment     |
| Test coverage scope             | qa-engineer      | Risk-based priority |
| Documentation detail            | techwriter       | Audience needs      |
| Infrastructure approach         | devops-engineer  | Cost/complexity     |

## Escalation Criteria

Escalate to user when:

1. **Veto with HIGH severity** - Security concern but not critical
2. **Score tie** - Positions within 10% of each other
3. **Max rounds exceeded** - 3 debate rounds without convergence
4. **Novel situation** - No historical precedent in knowledge base
5. **Business impact** - Decision affects business requirements
6. **Explicit request** - Agent requests human judgment

## User Override

Users can override any resolution:

```json
{
  "override": {
    "original_winner": "security-analyst",
    "user_choice": "dev-react",
    "reason": "UX is critical for MVP launch, will address security in v2",
    "acknowledged_risks": ["Token in localStorage is XSS-vulnerable"],
    "ticket_created": "#43 - Migrate to secure token storage"
  }
}
```

User overrides are:

- Recorded with full justification
- Flagged in knowledge base
- May create follow-up issues for deferred concerns

## Integration with Workflow

### In ENRICH Phase

```
Specification conflict detected
  → Debate: architect vs security-analyst
  → Resolution recorded
  → Spec updated with winning approach
  → Dissent noted in spec comments
```

### In Code Review

```
Reviewer findings conflict
  → Debate: reviewer vs dev-* agent
  → Resolution determines which findings to address
  → PR comment includes resolution reasoning
```

### In E2E Phase

```
DoD interpretation conflict
  → Debate: qa-engineer vs implementer
  → Resolution determines pass/fail criteria
  → Validation proceeds with resolved criteria
```

## Metrics and Learning

Track conflict resolution effectiveness:

```json
{
  "conflict_metrics": {
    "total_conflicts": 50,
    "resolved_by_voting": 35,
    "resolved_by_veto": 10,
    "escalated_to_user": 5,
    "user_overrides": 2,
    "average_debate_rounds": 1.8,
    "by_agent_pair": {
      "security-analyst_vs_dev-react": {
        "conflicts": 8,
        "security_wins": 7,
        "dev_wins": 1
      }
    }
  }
}
```

Use metrics to:

- Identify frequent conflict patterns
- Adjust agent weights if consistently overridden
- Detect need for clearer guidelines
- Inform training data for future improvements
