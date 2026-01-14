# Adaptive Routing Configuration

<!-- TEMPLATE: Defines how tasks are routed to agents based on performance -->

## Overview

Adaptive routing uses historical performance data to select the best agent for each task, with fallback rules for edge cases and learning feedback loops.

## Routing Algorithm

### Agent Selection Process

```python
def route_to_agent(task, candidates, knowledge_base):
    """Select the best agent for a task from candidates."""

    scores = {}

    for agent in candidates:
        # 1. Capability match (can agent do this?)
        if not agent.can_handle(task.action):
            continue

        # 2. Calculate routing score
        capability_score = agent.capability_match(task)
        performance_score = get_performance_score(agent, task.domain, knowledge_base)
        trend_boost = 0.1 if agent.recent_trend == "up" else (-0.1 if agent.recent_trend == "down" else 0)
        load_penalty = -0.1 if agent.current_tasks > 0 else 0

        scores[agent] = (
            capability_score * 0.3 +     # 30% capability match
            performance_score * 0.5 +     # 50% historical performance
            agent.routing_weight * 0.1 +  # 10% routing weight
            trend_boost +                 # Recent trend bonus/penalty
            load_penalty                  # Load balancing
        )

    if not scores:
        return None, "No capable agent found"

    # Select highest scoring agent
    best_agent = max(scores, key=scores.get)
    best_score = scores[best_agent]

    # Apply fallback rules
    if best_score < 0.6:
        return apply_fallback_rules(task, candidates, best_agent, best_score)

    return best_agent, None
```

### Score Components

| Component | Weight | Source | Range |
|-----------|--------|--------|-------|
| Capability match | 30% | Agent contract | 0.0 - 1.0 |
| Performance score | 50% | Knowledge base metrics | 0.0 - 1.0 |
| Routing weight | 10% | Calculated from metrics | 0.5 - 2.0 |
| Trend boost | +/-10% | Recent 7-day performance | -0.1 to +0.1 |
| Load penalty | -10% | Current active tasks | 0 or -0.1 |

### Capability Match

```python
def capability_match(agent, task):
    """Score how well agent's capabilities match task requirements."""

    # Check required capabilities
    required = set(task.required_capabilities)
    available = set(agent.capabilities)

    if not required.issubset(available):
        return 0.0  # Missing required capability

    # Score optional capabilities
    optional = set(task.optional_capabilities)
    optional_match = len(optional & available) / len(optional) if optional else 1.0

    # Domain match bonus
    domain_match = 1.0 if task.domain in agent.primary_domains else 0.8

    return 0.5 + (0.3 * optional_match) + (0.2 * domain_match)
```

### Performance Score

```python
def get_performance_score(agent, domain, knowledge_base):
    """Get agent's performance score for a domain."""

    metrics = knowledge_base.get_agent_metrics(agent.name)

    if not metrics or domain not in metrics.by_domain:
        return 0.75  # Default for unknown

    domain_metrics = metrics.by_domain[domain]

    # Weighted combination of metrics
    return (
        domain_metrics.success_rate * 0.6 +
        (domain_metrics.quality_score / 100) * 0.3 +
        confidence_from_count(domain_metrics.count) * 0.1
    )

def confidence_from_count(count):
    """More invocations = higher confidence in metrics."""
    if count >= 50: return 1.0
    if count >= 20: return 0.8
    if count >= 10: return 0.6
    if count >= 5: return 0.4
    return 0.2
```

## Fallback Rules

### Condition-Based Fallbacks

| Condition | Action | Rationale |
|-----------|--------|-----------|
| Best score < 60% | Try backup agent | Primary agent may be struggling |
| All scores < 60% | Escalate to orchestrator | Need coordination |
| No specialist for file type | Use system-architect | General-purpose fallback |
| Agent timed out previously | Prefer different agent | Avoid repeated timeouts |
| Agent had recent failure | Reduce priority | Investigate before reusing |

### Fallback Implementation

```python
def apply_fallback_rules(task, candidates, best_agent, best_score):
    """Apply fallback rules when best score is too low."""

    # Rule 1: Try backup agent
    backup = get_backup_agent(best_agent, task.domain)
    if backup and backup in candidates:
        backup_score = calculate_score(backup, task)
        if backup_score > best_score:
            return backup, "Fallback to backup agent (higher score)"

    # Rule 2: Escalate to orchestrator
    if all(calculate_score(a, task) < 0.6 for a in candidates):
        return "orchestrator", "Escalated (all candidates below threshold)"

    # Rule 3: Use with warning
    return best_agent, f"Warning: Low confidence ({best_score:.2f})"
```

### Backup Agent Mapping

| Primary Agent | Backup Agent | Condition |
|---------------|--------------|-----------|
| security-analyst | system-architect | Non-critical security |
| dev-python | reviewer | Complex Python issue |
| dev-react | reviewer | Complex frontend issue |
| dev-go | reviewer | Complex Go issue |
| dev-cpp | system-architect | Complex C++ issue |
| qa-engineer | reviewer | Test-related issue |
| techwriter | reviewer | Documentation issue |

## Learning Feedback Loop

### After Task Completion

```python
def record_routing_outcome(task, agent, outcome, knowledge_base):
    """Record outcome to improve future routing."""

    # Update agent metrics
    knowledge_base.update_agent_metrics(
        agent=agent.name,
        domain=task.domain,
        phase=task.phase,
        outcome=outcome.status,
        duration_ms=outcome.duration,
        quality_indicators=outcome.quality
    )

    # Check if fallback was needed
    if task.routing_info.fallback_used:
        record_fallback_outcome(task, outcome)

    # Detect patterns
    detect_routing_patterns(agent, task.domain, outcome)
```

### Pattern Detection

```python
def detect_routing_patterns(agent, domain, outcome):
    """Detect patterns that might inform routing rules."""

    recent_outcomes = get_recent_outcomes(agent, domain, days=7)

    # Pattern: Consistent failures
    if count_failures(recent_outcomes) >= 3:
        alert("Agent {agent} has 3+ recent failures in {domain}")
        reduce_routing_weight(agent, domain)

    # Pattern: Consistent successes
    if count_successes(recent_outcomes) >= 10:
        increase_routing_weight(agent, domain)

    # Pattern: Timeout issues
    if count_timeouts(recent_outcomes) >= 2:
        flag_for_investigation(agent, "repeated timeouts")
```

## Configuration Options

### In `.claude/routing-config.json`:

```json
{
  "routing": {
    "minimum_score_threshold": 0.6,
    "capability_weight": 0.3,
    "performance_weight": 0.5,
    "routing_weight_factor": 0.1,
    "enable_trend_boost": true,
    "enable_load_balancing": true,
    "fallback_enabled": true,
    "escalation_threshold": 0.5
  },
  "overrides": {
    "domain:security": {
      "preferred_agent": "security-analyst",
      "minimum_score_threshold": 0.7
    },
    "phase:implementing": {
      "load_balancing_disabled": true
    }
  },
  "exclusions": {
    "dev-cpp": {
      "domains": ["frontend"],
      "reason": "Not applicable to frontend work"
    }
  }
}
```

### Per-Domain Routing Rules

```json
{
  "domain_routing": {
    "security": {
      "required_agent": "security-analyst",
      "secondary_agents": ["system-architect", "reviewer"],
      "minimum_review": true
    },
    "infrastructure": {
      "required_agent": "devops-engineer",
      "secondary_agents": ["system-architect"],
      "minimum_review": false
    }
  }
}
```

## Routing Metrics

Track routing effectiveness:

```json
{
  "routing_metrics": {
    "total_routings": 500,
    "successful_first_choice": 450,
    "fallback_used": 40,
    "escalated": 10,
    "success_rate_by_method": {
      "first_choice": 0.95,
      "fallback": 0.82,
      "escalation": 0.70
    },
    "by_domain": {
      "security": {
        "routings": 100,
        "first_choice_rate": 0.98,
        "avg_score": 0.87
      }
    }
  }
}
```

## Integration with Orchestrator

When orchestrator is active:

1. **Orchestrator controls routing**: For HIGH+ complexity issues
2. **Parallel routing**: Multiple agents selected for parallel execution
3. **Dynamic reallocation**: Orchestrator can reassign if agent struggles
4. **Routing decisions logged**: For conflict resolution and learning

```python
def orchestrator_route(task, team, knowledge_base):
    """Orchestrator-managed routing for complex workflows."""

    # Get all suitable agents from team
    suitable = [m for m in team if m.role in task.suitable_roles]

    if not suitable:
        # Need to add agent to team
        new_agent = route_to_agent(task, all_agents, knowledge_base)
        team.add(new_agent)
        return new_agent

    # Route among team members
    return route_to_agent(task, suitable, knowledge_base)
```

## Debugging Routing Decisions

Enable routing debug mode:

```bash
/crunch 42 --routing-debug
```

Output includes:
- All candidates considered
- Score breakdown for each
- Fallback decisions made
- Final selection rationale
