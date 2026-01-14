# Execution Dependency Graph

<!-- TEMPLATE: Defines the structure for parallel execution with dependencies -->

## Overview

The execution graph represents tasks and their dependencies, enabling the orchestrator to maximize parallel execution while respecting sequential constraints.

## Graph Structure

### Schema

```json
{
  "graph_id": "GRAPH-{timestamp}-{hash}",
  "issue_number": 42,
  "phase": "enrich|implementing|validation",
  "created_at": "ISO-8601",
  "nodes": [
    {
      "id": "task-001",
      "agent": "system-architect",
      "action": "create_spec",
      "status": "pending|queued|running|complete|failed|skipped",
      "dependencies": [],
      "outputs": ["spec_document"],
      "priority": "CRITICAL|HIGH|NORMAL|LOW",
      "timeout_ms": 300000,
      "started_at": null,
      "completed_at": null,
      "result": null
    }
  ],
  "edges": [
    {
      "from": "task-001",
      "to": "task-002",
      "type": "data|sequence|approval",
      "data_key": "spec_document"
    }
  ],
  "batches": [
    {
      "batch_id": 1,
      "tasks": ["task-001"],
      "status": "pending|running|complete"
    }
  ]
}
```

## Node Definition

### Status Values

| Status | Description |
|--------|-------------|
| `pending` | Not yet scheduled |
| `queued` | Ready to run, waiting for executor |
| `running` | Currently executing |
| `complete` | Successfully finished |
| `failed` | Execution failed |
| `skipped` | Skipped due to dependency failure |

### Node Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique task identifier |
| `agent` | string | Yes | Agent assigned to task |
| `action` | string | Yes | Action type from ACP contracts |
| `status` | enum | Yes | Current execution status |
| `dependencies` | array | Yes | Task IDs this depends on |
| `outputs` | array | Yes | Keys for outputs produced |
| `priority` | enum | No | Execution priority |
| `timeout_ms` | int | No | Task timeout override |
| `started_at` | string | No | ISO-8601 start time |
| `completed_at` | string | No | ISO-8601 completion time |
| `result` | object | No | Task result after completion |

## Edge Types

### Data Dependency

Output of one task feeds input of another.

```json
{
  "from": "create_spec",
  "to": "implement",
  "type": "data",
  "data_key": "spec_document"
}
```

**Behavior**: `implement` cannot start until `create_spec` produces `spec_document`.

### Sequence Dependency

One task must complete before another starts, regardless of data flow.

```json
{
  "from": "implement",
  "to": "review",
  "type": "sequence"
}
```

**Behavior**: `review` waits for `implement` to complete, even without direct data dependency.

### Approval Dependency

Requires explicit approval before proceeding.

```json
{
  "from": "security_review",
  "to": "deploy",
  "type": "approval",
  "approval_required_from": "security-analyst"
}
```

**Behavior**: `deploy` waits for both completion and explicit approval from the security review.

## Batch Scheduling

### Batch Formation

Tasks are grouped into batches based on dependencies:

```python
def create_batches(graph):
    batches = []
    scheduled = set()

    while len(scheduled) < len(graph.nodes):
        # Find all tasks whose dependencies are satisfied
        ready = [
            node for node in graph.nodes
            if node.id not in scheduled
            and all(dep in scheduled for dep in node.dependencies)
        ]

        if not ready:
            raise CyclicDependencyError()

        batch = Batch(
            batch_id=len(batches) + 1,
            tasks=[node.id for node in ready],
            status="pending"
        )
        batches.append(batch)
        scheduled.update(batch.tasks)

    return batches
```

### Execution Flow

```
Batch 1: [task-001]              → Run sequentially (single task)
Batch 2: [task-002, task-003]    → Run in parallel
Batch 3: [task-004]              → Depends on batch 2 completion
Batch 4: [task-005, task-006]    → Run in parallel
```

## Example Graphs

### ENRICH Phase Graph

```json
{
  "graph_id": "GRAPH-20250114-a3f2",
  "phase": "enrich",
  "nodes": [
    {
      "id": "investigate",
      "agent": "system-architect",
      "action": "investigate",
      "dependencies": [],
      "outputs": ["investigation_report"]
    },
    {
      "id": "create_spec",
      "agent": "system-architect",
      "action": "create_spec",
      "dependencies": ["investigate"],
      "outputs": ["spec_document"]
    },
    {
      "id": "create_test_plan",
      "agent": "qa-engineer",
      "action": "create_test_plan",
      "dependencies": ["investigate"],
      "outputs": ["test_plan"]
    },
    {
      "id": "security_review",
      "agent": "security-analyst",
      "action": "security_review",
      "dependencies": ["create_spec"],
      "outputs": ["security_findings"]
    },
    {
      "id": "inject_knowledge",
      "agent": "knowledge-manager",
      "action": "generate_brief",
      "dependencies": [],
      "outputs": ["context_brief"]
    }
  ],
  "edges": [
    {"from": "investigate", "to": "create_spec", "type": "data", "data_key": "investigation_report"},
    {"from": "investigate", "to": "create_test_plan", "type": "data", "data_key": "investigation_report"},
    {"from": "create_spec", "to": "security_review", "type": "data", "data_key": "spec_document"}
  ],
  "batches": [
    {"batch_id": 1, "tasks": ["inject_knowledge", "investigate"]},
    {"batch_id": 2, "tasks": ["create_spec", "create_test_plan"]},
    {"batch_id": 3, "tasks": ["security_review"]}
  ]
}
```

Visual representation:
```
        ┌─────────────────┐
        │inject_knowledge │
        │ (knowledge-mgr) │
        └────────┬────────┘
                 │
    ┌────────────┴────────────┐
    │                         │
    ▼                         ▼
┌──────────┐           ┌──────────────────┐
│investigate│           │     (parallel)    │
│ (arch)    │           └──────────────────┘
└────┬─────┘
     │
     ├─────────────────┬─────────────────┐
     │                 │                 │
     ▼                 ▼                 │
┌──────────┐    ┌────────────┐          │
│create_spec│   │create_test │           │
│ (arch)    │   │  _plan     │           │
│           │   │ (qa-eng)   │           │
└────┬──────┘   └────────────┘          │
     │                                   │
     ▼                                   │
┌──────────────┐                        │
│security_review│                        │
│(sec-analyst) │◀────────────────────────┘
└──────────────┘
```

### Code Review Graph

> **Note**: The review agents below (r1-r6) are specialized sub-agents used by the `/review` skill for parallel code review. They are distinct from the 16 main agents in the agent system.

```json
{
  "graph_id": "GRAPH-20250114-b4c5",
  "phase": "validation",
  "nodes": [
    {"id": "r1", "agent": "claude-compliance", "action": "review_code", "dependencies": []},
    {"id": "r2", "agent": "bug-scanner", "action": "review_code", "dependencies": []},
    {"id": "r3", "agent": "git-history", "action": "review_code", "dependencies": []},
    {"id": "r4", "agent": "prev-pr", "action": "review_code", "dependencies": []},
    {"id": "r5", "agent": "comment-checker", "action": "review_code", "dependencies": []},
    {"id": "r6", "agent": "test-quality", "action": "review_tests", "dependencies": []},
    {"id": "merge", "agent": "orchestrator", "action": "merge_findings", "dependencies": ["r1", "r2", "r3", "r4", "r5", "r6"]}
  ],
  "edges": [
    {"from": "r1", "to": "merge", "type": "data", "data_key": "findings"},
    {"from": "r2", "to": "merge", "type": "data", "data_key": "findings"},
    {"from": "r3", "to": "merge", "type": "data", "data_key": "findings"},
    {"from": "r4", "to": "merge", "type": "data", "data_key": "findings"},
    {"from": "r5", "to": "merge", "type": "data", "data_key": "findings"},
    {"from": "r6", "to": "merge", "type": "data", "data_key": "findings"}
  ],
  "batches": [
    {"batch_id": 1, "tasks": ["r1", "r2", "r3", "r4", "r5", "r6"]},
    {"batch_id": 2, "tasks": ["merge"]}
  ]
}
```

Visual representation:
```
   Batch 1 (all parallel)
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐             │
│  │ r1 │  │ r2 │  │ r3 │  │ r4 │  │ r5 │  │ r6 │             │
│  └──┬─┘  └──┬─┘  └──┬─┘  └──┬─┘  └──┬─┘  └──┬─┘             │
│     │       │       │       │       │       │                │
└─────┼───────┼───────┼───────┼───────┼───────┼────────────────┘
      │       │       │       │       │       │
      └───────┴───────┴───┬───┴───────┴───────┘
                          │
                          ▼
                    ┌───────────┐
                    │   merge   │
                    │ (orch)    │
                    └───────────┘
```

## Execution Scheduler

### Scheduler Algorithm

```python
async def execute_graph(graph, orchestrator):
    """Execute graph with parallel batches."""

    for batch in graph.batches:
        batch.status = "running"

        # Check for failures in previous batches
        failed_deps = get_failed_dependencies(batch, graph)
        if failed_deps:
            handle_batch_failure(batch, failed_deps)
            continue

        # Launch all tasks in batch
        tasks = [
            launch_task(graph.get_node(task_id))
            for task_id in batch.tasks
        ]

        # Wait for all to complete or fail
        results = await asyncio.gather(*tasks, return_exceptions=True)

        # Process results
        for task_id, result in zip(batch.tasks, results):
            node = graph.get_node(task_id)
            if isinstance(result, Exception):
                node.status = "failed"
                node.result = {"error": str(result)}
            else:
                node.status = "complete"
                node.result = result

        # Check for conflicts
        conflicts = detect_conflicts(batch, graph)
        if conflicts:
            resolution = await orchestrator.resolve_conflict(conflicts)
            apply_resolution(resolution, graph)

        batch.status = "complete"

        # Checkpoint after each batch
        save_checkpoint(graph)

    return graph
```

### Failure Handling

| Scenario | Action |
|----------|--------|
| Single task fails | Mark dependents as skipped, continue batch |
| Critical task fails | Abort remaining batches, escalate |
| Timeout | Retry once, then mark failed |
| All batch tasks fail | Checkpoint and escalate |

```python
def handle_task_failure(node, graph):
    """Handle a single task failure."""

    # Mark all dependent tasks as skipped
    for dependent_id in get_dependents(node.id, graph):
        dependent = graph.get_node(dependent_id)
        dependent.status = "skipped"
        dependent.result = {"skipped_reason": f"Dependency {node.id} failed"}

    # Check if this was a critical task
    if node.priority == "CRITICAL":
        raise CriticalTaskFailure(node)
```

## Graph Modification

### Adding Tasks Dynamically

```python
def add_task_to_graph(graph, new_task, after_task_id=None):
    """Add a new task, optionally after a specific task."""

    graph.nodes.append(new_task)

    if after_task_id:
        # Add dependency
        new_task.dependencies.append(after_task_id)

        # Add edge
        graph.edges.append({
            "from": after_task_id,
            "to": new_task.id,
            "type": "sequence"
        })

    # Regenerate batches
    graph.batches = create_batches(graph)
```

### Replaying Failed Tasks

```python
def replay_failed_tasks(graph):
    """Reset failed tasks for retry."""

    for node in graph.nodes:
        if node.status == "failed":
            node.status = "pending"
            node.result = None
            node.started_at = None
            node.completed_at = None

        if node.status == "skipped":
            # Check if dependencies now pass
            if all_deps_complete(node, graph):
                node.status = "pending"

    # Regenerate batches from pending tasks
    graph.batches = create_batches_from_pending(graph)
```

## Integration with Checkpoints

Graphs are saved as part of checkpoints:

```json
{
  "checkpoint_id": "CP-20250114-a3f2",
  "graph_state": {
    "graph_id": "GRAPH-20250114-a3f2",
    "completed_batches": [1, 2],
    "current_batch": 3,
    "node_states": {
      "task-001": {"status": "complete", "result": {...}},
      "task-002": {"status": "complete", "result": {...}},
      "task-003": {"status": "running", "started_at": "..."}
    }
  }
}
```

Recovery restores graph state and resumes from current batch.
