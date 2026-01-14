# Orchestration Workflow

This document defines how the orchestrate skill coordinates multi-agent workflows.

## Overview

The orchestration workflow consists of 6 phases:
1. **Analysis** - Assess issue complexity and requirements
2. **Composition** - Select optimal team of agents
3. **Planning** - Build execution graph with dependencies
4. **Execution** - Run tasks with parallel optimization
5. **Conflict** - Resolve disagreements when they arise
6. **Completion** - Capture metrics and clean up

---

## Phase 1: Issue Analysis

### Input

- Issue number from `/crunch` or direct invocation
- Current workflow phase (from crunch context)

### Process

1. **Fetch Issue Context**

```
Fetch issue via CI MCP:
  - Title, body, labels
  - Current state label
  - Branch status
  - Related PRs
```

2. **Detect Domains**

```
domains = detect_domains_from({
  labels: issue.labels,
  title_keywords: extract_keywords(issue.title),
  body_keywords: extract_keywords(issue.body)
})

Domain detection uses keyword mapping:
  security: [auth, token, JWT, password, XSS, injection, CVE]
  infrastructure: [deploy, docker, k8s, terraform, CI/CD]
  frontend: [React, component, UI, CSS, TypeScript]
  python: [Django, Flask, FastAPI, pandas]
  go: [goroutine, gRPC, go mod]
  cpp: [STL, CMake, memory, pointer]
```

3. **Calculate Complexity Score**

```
complexity_factors = {
  multi_domain: len(domains) >= 2,           # +2 if true
  high_file_count: estimate_files() >= 10,   # +1 if true
  security_involved: "security" in domains,  # +3 if true
  breaking_change: has_breaking_labels(),    # +2 if true
  prior_failures: get_failure_count() >= 1   # +2 if true
}

complexity_score = sum(weight for factor, weight in factors if factor)

complexity_level = (
  "LOW" if score < 3 else
  "MEDIUM" if score < 5 else
  "HIGH" if score < 8 else
  "CRITICAL"
)
```

4. **Query Knowledge Base**

```
knowledge_brief = query_knowledge({
  domains: detected_domains,
  similar_issues: true,
  agent_effectiveness: true,
  patterns: true
})
```

### Output

```json
{
  "issue_number": 42,
  "domains": ["security", "auth"],
  "complexity": {
    "score": 6,
    "level": "HIGH",
    "factors": {...}
  },
  "knowledge_brief": {...},
  "orchestration_required": true
}
```

---

## Phase 2: Team Composition

### Process

1. **Determine Required Roles**

```
Based on complexity and phase:

ENRICH phase roles:
  - coordinator (if HIGH+)
  - domain_supervisor (per domain)
  - specialist (for primary domain)
  - quality (qa-engineer)
  - context (knowledge-manager if HIGH+)

IMPLEMENTING phase roles:
  - coordinator (if HIGH+)
  - implementer (language-specific)
  - quality (qa-engineer)

VALIDATION phase roles:
  - coordinator (if HIGH+)
  - reviewers (multiple, parallel)
  - security (if security domain)
  - monitoring (responder agents)
```

2. **Select Agents for Roles**

```
For each role:
  candidates = get_capable_agents(role)

  For each candidate:
    score = calculate_routing_score({
      capability_match: 0.3,
      performance_history: 0.5,
      routing_weight: 0.1,
      trend_boost: 0.1
    })

  Select highest scoring candidate
  Apply fallback rules if score < 0.6
```

3. **Assign Phases**

```
For each team member:
  member.phases = determine_active_phases(member.role, current_phase)
```

### Output

```json
{
  "team_id": "TEAM-20250114-a3f2",
  "issue_number": 42,
  "complexity_level": "HIGH",
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
    }
  ]
}
```

---

## Phase 3: Execution Planning

### Process

1. **Identify Tasks for Current Phase**

```
Based on current workflow phase and team:

ENRICH tasks:
  - inject_knowledge (knowledge-manager)
  - investigate (system-architect)
  - create_spec (system-architect)
  - create_test_plan (qa-engineer)
  - security_review (security-analyst, if domain)

IMPLEMENTING tasks:
  - inject_knowledge (knowledge-manager)
  - implement (dev-* agent)
  - verify_coverage (qa-engineer)

VALIDATION tasks:
  - parallel_review (6 review agents)
  - merge_findings (orchestrator)
  - deploy_staging (devops-engineer)
  - monitor_staging (responder agents)
```

2. **Build Dependency Graph**

```
For each task:
  Determine dependencies based on:
    - Data dependencies (output → input)
    - Sequence dependencies (must run after)
    - Approval dependencies (needs sign-off)

  Create edges between tasks
```

3. **Generate Batches**

```
batches = []
scheduled = set()

while unscheduled tasks exist:
  ready = tasks where all dependencies in scheduled
  batch = create_batch(ready)
  batches.append(batch)
  scheduled.update(batch.tasks)
```

4. **Define Checkpoints**

```
checkpoints = []
For each batch:
  if batch contains critical task:
    checkpoints.append(checkpoint_before(batch))
  checkpoints.append(checkpoint_after(batch))
```

### Output

```json
{
  "graph_id": "GRAPH-20250114-a3f2",
  "phase": "enrich",
  "nodes": [...],
  "edges": [...],
  "batches": [
    {"batch_id": 1, "tasks": ["inject_knowledge", "investigate"]},
    {"batch_id": 2, "tasks": ["create_spec", "create_test_plan"]},
    {"batch_id": 3, "tasks": ["security_review"]}
  ],
  "checkpoints": ["CP-before-batch-1", "CP-after-batch-3"]
}
```

---

## Phase 4: Coordinated Execution

### Process

1. **Execute Batches**

```
For each batch in execution_graph.batches:

  # Pre-batch checkpoint
  if batch.has_pre_checkpoint:
    save_checkpoint("batch_start", batch.id)

  # Launch all tasks in batch (parallel)
  tasks = []
  for task_id in batch.tasks:
    task = graph.get_node(task_id)
    agent = team.get_member(task.agent)

    # Prepare ACP REQUEST message
    request = create_acp_request({
      to: agent.name,
      action: task.action,
      context: prepare_context(task, knowledge_brief),
      constraints: task.constraints
    })

    # Launch via Task tool
    tasks.append(launch_agent_task(agent, request))

  # Wait for all tasks to complete
  results = await_all(tasks)

  # Process results
  for task_id, result in zip(batch.tasks, results):
    task = graph.get_node(task_id)

    if result.status == "SUCCESS":
      task.status = "complete"
      task.result = result.output
      store_output(task.outputs, result.output)

    elif result.status == "BLOCKED":
      # Handle handoff or escalation
      handle_blocked_task(task, result)

    else:
      task.status = "failed"
      handle_failed_task(task, result)

  # Check for conflicts
  conflicts = detect_conflicts_in_batch(results)
  if conflicts:
    resolve_conflicts(conflicts)  # Phase 5

  # Post-batch checkpoint
  save_checkpoint("batch_complete", batch.id)

  # Update progress
  update_issue_progress(issue, batch)
```

2. **Handle Task Outputs**

```
For each completed task:

  # Store outputs for dependent tasks
  for output_key in task.outputs:
    output_store[output_key] = task.result[output_key]

  # Check if outputs satisfy dependencies
  for dependent in get_dependents(task.id):
    if all_dependencies_met(dependent):
      mark_ready(dependent)
```

3. **Update Issue Body**

```
After each batch:

  progress_section = format_progress({
    phase: current_phase,
    batch: current_batch,
    completed: completed_tasks,
    pending: pending_tasks,
    team: active_team_members
  })

  update_issue_body_section("orchestration_progress", progress_section)
```

### Execution Patterns

**Sequential Execution** (default for dependencies):
```
Task A completes → Start Task B → Task B completes → Start Task C
```

**Parallel Execution** (no dependencies):
```
Task A starts ─┐
Task B starts ─┼─ All complete → Continue
Task C starts ─┘
```

**Competitive Execution** (best solution wins):
```
Approach A (architect) ─┐
                        ├─ Compare → Select winner → Apply
Approach B (security)  ─┘
```

---

## Phase 5: Conflict Resolution

### When Invoked

- Multiple agents disagree on approach
- Contradicting findings in review
- Resource/priority competition
- Veto authority invoked

### Process

1. **Detect Conflict**

```
conflict_detected = (
  agent_a.recommendation != agent_b.recommendation AND
  recommendations overlap in scope AND
  both are actionable
)

if conflict_detected:
  conflict = create_conflict_record(agent_a, agent_b, topic)
  save_checkpoint("conflict_start", conflict.id)
```

2. **Structure Debate**

```
For each conflicting agent:
  request_position({
    agent: agent.name,
    topic: conflict.topic,
    required: ["position", "rationale", "evidence", "compromise"]
  })

  position = await_response(agent)
  conflict.positions.append(position)
```

3. **Calculate Scores**

```
For each position:
  agent = position.agent

  base_weight = AGENT_WEIGHTS[agent.name]  # From hierarchy
  domain_boost = get_domain_boost(agent, conflict.domain)
  confidence = position.confidence / 100
  relevance = position.domain_relevance

  score = (base_weight + domain_boost) * confidence * relevance

  position.score = score
```

4. **Check Veto**

```
for position in conflict.positions:
  if position.veto:
    veto_agent = position.agent

    if veto_agent.has_veto_authority(conflict.domain):
      if position.veto_severity == "CRITICAL":
        # Immediate block
        conflict.resolution = position
        conflict.veto_applied = true
        return

      elif position.veto_severity == "HIGH":
        # Escalate to user
        user_decision = escalate_to_user(conflict, position)
        if user_decision == "accept_veto":
          conflict.resolution = position
          conflict.veto_applied = true
          return
```

5. **Apply Resolution**

```
if not conflict.veto_applied:
  # Select highest scoring position
  winner = max(conflict.positions, key=lambda p: p.score)
  conflict.resolution = winner

# Record dissent
for position in conflict.positions:
  if position != conflict.resolution:
    conflict.dissent.append(position)

# Store in knowledge base
create_decision_entry(conflict)

# Apply winning position
apply_resolution(conflict.resolution)

save_checkpoint("conflict_resolved", conflict.id)
```

---

## Phase 6: Completion

### Process

1. **Verify Completion**

```
all_batches_complete = all(
  batch.status == "complete" for batch in graph.batches
)

if not all_batches_complete:
  handle_incomplete_workflow()
  return
```

2. **Capture Metrics**

```
For each agent in team:
  update_agent_metrics({
    agent: agent.name,
    domain: issue.primary_domain,
    phase: completed_phase,
    outcome: determine_outcome(agent),
    duration_ms: agent.total_duration,
    quality_indicators: calculate_quality(agent)
  })
```

3. **Record Knowledge**

```
# Capture resolution if appropriate
if phase == "DONE":
  create_resolution_entry(issue, workflow_summary)

# Capture any decisions made
for conflict in resolved_conflicts:
  ensure_decision_recorded(conflict)

# Update patterns if applicable
check_pattern_promotion(workflow_findings)
```

4. **Clean Up**

```
# Archive old checkpoints
cleanup_checkpoints(issue.number, keep_latest=true)

# Clear ACP message log for this issue
archive_acp_messages(issue.number)

# Update issue with completion summary
update_issue_completion_summary(issue, metrics)
```

5. **Report Summary**

```
Output completion summary:
  - Team composition used
  - Execution statistics (tasks, batches, duration)
  - Conflicts resolved (if any)
  - Agent performance highlights
  - Next phase recommendation
```

---

## Error Handling

### Task Failures

```
on task_failure(task, error):

  # Check if retryable
  if is_retryable(error) and task.attempts < 2:
    task.attempts += 1
    retry_task(task)
    return

  # Mark task and dependents as failed
  task.status = "failed"
  for dependent in get_dependents(task.id):
    dependent.status = "skipped"

  # Check criticality
  if task.priority == "CRITICAL":
    escalate_critical_failure(task)
  else:
    continue_with_degraded_workflow()
```

### Timeout Handling

```
on task_timeout(task):

  # Check if still running (possible network issue)
  if task.is_responsive():
    extend_timeout(task, factor=1.5)
    return

  # Mark as timed out
  task.status = "failed"
  task.result = {"error": "timeout", "duration": task.elapsed}

  # Try backup agent if available
  backup = get_backup_agent(task.agent)
  if backup:
    reassign_task(task, backup)
```

### Recovery

```
on session_interrupt():
  save_checkpoint("user_interrupt", immediate=true)

on resume():
  checkpoint = load_latest_checkpoint()
  validate_checkpoint_freshness()
  restore_workflow_state(checkpoint)
  continue_from_checkpoint()
```

---

## Integration Points

### With /crunch

```
/crunch automatically invokes orchestration when:
  complexity_score >= 5 OR
  len(domains) >= 2 OR
  prior_attempt_failed

Orchestration returns control to /crunch for:
  - Phase transitions
  - State label updates
  - User interactions
```

### With /review

```
/review uses orchestration for parallel review:
  - 6 review agents launched simultaneously
  - Findings merged by orchestrator
  - Conflicts resolved via debate protocol
```

### With Knowledge Base

```
Knowledge integration at multiple points:
  - Phase entry: Inject context brief
  - Task completion: Capture outputs
  - Conflict resolution: Store decisions
  - Workflow completion: Update metrics
```
