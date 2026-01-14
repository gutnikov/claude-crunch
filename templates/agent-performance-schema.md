# Agent Performance Tracking Schema

<!-- TEMPLATE: Defines the schema for tracking agent performance metrics -->

## Overview

Agent performance tracking enables adaptive routing by recording success rates, quality scores, and trends for each agent across domains and phases.

## Storage Location

Performance metrics are stored in `.claude/knowledge/index.json` under the `agent_metrics` key.

## Schema Definition

### Full Agent Metrics Schema

```json
{
  "agent_metrics": {
    "{agent_name}": {
      "total_invocations": 0,
      "success_rate": 0.0,
      "average_duration_ms": 0,

      "by_domain": {
        "{domain}": {
          "count": 0,
          "successes": 0,
          "failures": 0,
          "partials": 0,
          "success_rate": 0.0,
          "avg_duration_ms": 0,
          "quality_score": 0,
          "last_invoked": "ISO-8601",
          "trend": "up|down|stable"
        }
      },

      "by_phase": {
        "{phase}": {
          "count": 0,
          "successes": 0,
          "failures": 0,
          "success_rate": 0.0,
          "avg_duration_ms": 0
        }
      },

      "quality_indicators": {
        "review_pass_rate": 0.0,
        "rework_rate": 0.0,
        "user_overrides": 0,
        "veto_rate": 0.0
      },

      "routing_weight": 1.0,

      "recent_performance": {
        "last_7_days": {
          "count": 0,
          "success_rate": 0.0,
          "trend": "up|down|stable"
        },
        "last_30_days": {
          "count": 0,
          "success_rate": 0.0,
          "trend": "up|down|stable"
        }
      },

      "last_updated": "ISO-8601"
    }
  }
}
```

### Field Definitions

| Field | Type | Description |
|-------|------|-------------|
| `total_invocations` | int | Total times agent was invoked |
| `success_rate` | float | Overall success rate (0.0-1.0) |
| `average_duration_ms` | int | Average task completion time |
| `by_domain.{domain}` | object | Performance metrics per domain |
| `by_phase.{phase}` | object | Performance metrics per workflow phase |
| `quality_indicators` | object | Quality-related metrics |
| `routing_weight` | float | Current routing preference (0.5-2.0) |
| `recent_performance` | object | Recent trends for adaptive routing |
| `last_updated` | string | ISO-8601 timestamp of last update |

## Success Criteria

### By Phase

| Phase | Success Definition |
|-------|-------------------|
| ENRICH | Spec approved without major rework |
| IMPLEMENTING | Tests pass, implementation accepted |
| VALIDATION | DoD checklist passes, review approved |
| Review | Findings accepted, not disputed |
| DOCS | Documentation approved |

### Outcome Values

| Outcome | Value | Description |
|---------|-------|-------------|
| SUCCESS | 1.0 | Task completed fully, accepted |
| PARTIAL | 0.5 | Task partially completed, some value delivered |
| FAILED | 0.0 | Task failed or completely rejected |

## Quality Score Calculation

Quality score combines multiple indicators:

```
quality_score = weighted_average(
  review_pass_rate     × 0.4,   // Reviews passed without major changes
  (1 - rework_rate)    × 0.3,   // Work accepted without rework
  (1 - override_rate)  × 0.2,   // User didn't override agent decisions
  on_time_rate         × 0.1    // Completed within timeout
)
```

### Indicator Definitions

| Indicator | Calculation | Good Threshold |
|-----------|-------------|----------------|
| review_pass_rate | approved_reviews / total_reviews | >= 0.8 |
| rework_rate | rework_requests / total_tasks | <= 0.2 |
| override_rate | user_overrides / total_decisions | <= 0.1 |
| on_time_rate | on_time_completions / total_tasks | >= 0.9 |

## Trend Calculation

Trends are calculated by comparing recent performance to historical:

```python
def calculate_trend(recent_rate, historical_rate):
    diff = recent_rate - historical_rate

    if diff > 0.05:    # 5% improvement
        return "up"
    elif diff < -0.05:  # 5% decline
        return "down"
    else:
        return "stable"
```

## Routing Weight Calculation

Routing weight determines agent selection priority:

```python
def calculate_routing_weight(agent_metrics, domain):
    base_weight = 1.0

    # Domain-specific success rate adjustment
    domain_rate = agent_metrics.by_domain[domain].success_rate
    if domain_rate > 0.9:
        base_weight += 0.3
    elif domain_rate > 0.8:
        base_weight += 0.1
    elif domain_rate < 0.6:
        base_weight -= 0.3
    elif domain_rate < 0.7:
        base_weight -= 0.1

    # Recent trend adjustment
    if agent_metrics.recent_performance.last_7_days.trend == "up":
        base_weight += 0.1
    elif agent_metrics.recent_performance.last_7_days.trend == "down":
        base_weight -= 0.1

    # Quality score adjustment
    quality = agent_metrics.quality_indicators
    if quality.review_pass_rate < 0.7:
        base_weight -= 0.2
    if quality.rework_rate > 0.3:
        base_weight -= 0.2

    # Clamp to valid range
    return max(0.5, min(2.0, base_weight))
```

## Update Events

### After Each Agent Invocation

```json
{
  "event": "agent_invocation_complete",
  "timestamp": "ISO-8601",
  "agent": "security-analyst",
  "issue_number": 42,
  "phase": "enrich",
  "domain": "security",
  "outcome": "success|partial|failed",
  "duration_ms": 45000,
  "quality_indicators": {
    "review_passed": true,
    "rework_needed": false,
    "user_override": false
  }
}
```

### Processing Update

```python
def update_agent_metrics(event, metrics):
    agent = metrics.agent_metrics[event.agent]

    # Update totals
    agent.total_invocations += 1
    agent.success_rate = recalculate_rate(agent, event.outcome)
    agent.average_duration_ms = recalculate_avg(agent, event.duration_ms)

    # Update domain metrics
    domain = agent.by_domain.setdefault(event.domain, default_domain_metrics())
    domain.count += 1
    domain.successes += 1 if event.outcome == "success" else 0
    domain.failures += 1 if event.outcome == "failed" else 0
    domain.partials += 1 if event.outcome == "partial" else 0
    domain.success_rate = domain.successes / domain.count
    domain.avg_duration_ms = recalculate_avg(domain, event.duration_ms)
    domain.last_invoked = event.timestamp
    domain.trend = calculate_trend(domain, agent.recent_performance)

    # Update phase metrics
    phase = agent.by_phase.setdefault(event.phase, default_phase_metrics())
    phase.count += 1
    phase.successes += 1 if event.outcome == "success" else 0
    phase.failures += 1 if event.outcome == "failed" else 0
    phase.success_rate = phase.successes / phase.count

    # Update quality indicators
    update_quality_indicators(agent.quality_indicators, event.quality_indicators)

    # Recalculate routing weight
    agent.routing_weight = calculate_routing_weight(agent, event.domain)

    # Update recent performance
    update_recent_performance(agent.recent_performance, event)

    agent.last_updated = event.timestamp
```

## Initial Metrics (New Agents)

New agents start with default metrics:

```json
{
  "total_invocations": 0,
  "success_rate": 0.75,
  "average_duration_ms": 0,
  "by_domain": {},
  "by_phase": {},
  "quality_indicators": {
    "review_pass_rate": 0.75,
    "rework_rate": 0.25,
    "user_overrides": 0,
    "veto_rate": 0.0
  },
  "routing_weight": 1.0,
  "recent_performance": {
    "last_7_days": {"count": 0, "success_rate": 0.0, "trend": "stable"},
    "last_30_days": {"count": 0, "success_rate": 0.0, "trend": "stable"}
  },
  "last_updated": null
}
```

Default success rate of 0.75 gives new agents a fair chance while experienced agents are preferred.

## Metrics Decay

To prevent stale metrics from dominating:

```python
def apply_metrics_decay(agent_metrics, decay_rate=0.01):
    """Apply daily decay to move metrics toward neutral over time."""
    for agent in agent_metrics.values():
        days_since_update = (now - agent.last_updated).days
        decay_factor = (1 - decay_rate) ** days_since_update

        # Decay routing weight toward 1.0
        agent.routing_weight = 1.0 + (agent.routing_weight - 1.0) * decay_factor
```

## Example: Full Agent Metrics

```json
{
  "agent_metrics": {
    "security-analyst": {
      "total_invocations": 47,
      "success_rate": 0.92,
      "average_duration_ms": 45000,

      "by_domain": {
        "security": {
          "count": 35,
          "successes": 33,
          "failures": 1,
          "partials": 1,
          "success_rate": 0.94,
          "avg_duration_ms": 42000,
          "quality_score": 87,
          "last_invoked": "2025-01-14T10:30:00Z",
          "trend": "up"
        },
        "auth": {
          "count": 12,
          "successes": 11,
          "failures": 0,
          "partials": 1,
          "success_rate": 0.92,
          "avg_duration_ms": 50000,
          "quality_score": 85,
          "last_invoked": "2025-01-13T15:20:00Z",
          "trend": "stable"
        }
      },

      "by_phase": {
        "enrich": {
          "count": 20,
          "successes": 19,
          "failures": 1,
          "success_rate": 0.95,
          "avg_duration_ms": 60000
        },
        "review": {
          "count": 27,
          "successes": 24,
          "failures": 1,
          "success_rate": 0.89,
          "avg_duration_ms": 35000
        }
      },

      "quality_indicators": {
        "review_pass_rate": 0.91,
        "rework_rate": 0.08,
        "user_overrides": 2,
        "veto_rate": 0.15
      },

      "routing_weight": 1.15,

      "recent_performance": {
        "last_7_days": {
          "count": 8,
          "success_rate": 0.96,
          "trend": "up"
        },
        "last_30_days": {
          "count": 25,
          "success_rate": 0.92,
          "trend": "stable"
        }
      },

      "last_updated": "2025-01-14T10:30:00Z"
    }
  }
}
```

## Integration Points

### With Team Composition

Performance metrics inform agent selection:
- High routing_weight agents preferred for their strong domains
- Declining agents (trend: "down") may be deprioritized
- New agents get opportunities but aren't preferred over proven performers

### With Conflict Resolution

Performance affects voting weights:
- Agents with higher quality_score have more influence
- Agents with frequent overrides may have reduced weight

### With Knowledge Base

Performance feeds learning:
- Patterns of success/failure captured
- Recommendations for agent improvements generated
- Training data for future optimizations
