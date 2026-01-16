---
name: orchestrate
description: "Coordinate multi-agent workflows for complex issues. Automatically invoked by /crunch when issue complexity is HIGH (score >= 5) or involves multiple domains. Can also be invoked directly for manual orchestration control."
---

# Orchestrate Skill

Coordinate agent collaboration for complex development tasks through structured team composition, execution planning, conflict resolution, and progress checkpointing.

## When Orchestration is Active

Orchestration is automatically activated when:

- Issue complexity score >= 5 (HIGH)
- Issue spans 2+ domains
- Previous attempt on the issue failed
- User explicitly requests coordination

## Syntax

```
/orchestrate [options] [issue_number]
```

## Parameters

| Parameter           | Type   | Description                                                         |
| ------------------- | ------ | ------------------------------------------------------------------- |
| `issue_number`      | int    | Issue number to orchestrate (optional if already in crunch context) |
| `--plan`            | flag   | Show execution plan without executing                               |
| `--resume`          | flag   | Resume from latest checkpoint                                       |
| `--checkpoint <id>` | string | Resume from specific checkpoint                                     |
| `--team <agents>`   | string | Override team composition (comma-separated)                         |
| `--debug`           | flag   | Enable detailed orchestration logging                               |

## Examples

### Show Execution Plan

```
/orchestrate --plan 42
```

Shows the proposed team composition and execution graph without executing.

### Resume from Checkpoint

```
/orchestrate --resume 42
```

Resumes the workflow from the latest checkpoint.

### Custom Team Override

```
/orchestrate --team security-analyst,dev-python,reviewer 42
```

Forces specific agents to be included in the team.

## Capabilities

### 1. Team Composition

Dynamically selects agents based on:

- Issue domains (security, frontend, infrastructure, etc.)
- Complexity score
- Historical agent effectiveness
- Phase requirements

### 2. Execution Planning

Creates dependency graphs for:

- Sequential tasks (spec → implementation → review)
- Parallel tasks (multiple reviewers simultaneously)
- Approval gates (security review before deploy)

### 3. Conflict Resolution

Handles disagreements through:

- Structured debate protocol
- Weighted voting based on expertise
- Veto authority for security matters
- User escalation when needed

### 4. Progress Management

Maintains workflow state through:

- Automatic checkpoints at key transitions
- Recovery from session interruptions
- Progress reporting to issue body

## Integration with /crunch

When `/crunch` detects HIGH+ complexity, it automatically invokes orchestration:

```
/crunch 42
  → Complexity analysis: score = 6 (HIGH)
  → Multi-domain detected: [security, auth]
  → Activating orchestrator...
  → Team composed: [orchestrator, security-analyst, system-architect, dev-python, qa-engineer, reviewer]
  → Execution graph created
  → Proceeding with coordinated workflow
```

## Output

Orchestration provides:

- Team composition summary
- Execution graph visualization
- Progress updates as batches complete
- Conflict resolution reports
- Final summary with metrics

## Related Templates

- `templates/hierarchy-config.md` - Agent tier structure
- `templates/team-composition.md` - Team selection algorithm
- `templates/conflict-resolution.md` - Debate protocol
- `templates/execution-graph.md` - Dependency graph structure
- `templates/checkpoint-config.md` - Progress persistence
- `templates/acp-schema.md` - Agent communication protocol
