# Workflow Checkpoint Configuration

<!-- TEMPLATE: Defines checkpoint structure for progress persistence and recovery -->

## Overview

Checkpoints enable workflow persistence across sessions, failure recovery, and progress tracking. They capture the complete state needed to resume from any point.

## Checkpoint Schema

### Full Checkpoint Structure

```yaml
checkpoint_id: "CP-{phase}-{timestamp}-{hash}"
version: "1.0"
created_at: "ISO-8601"
issue_number: 42
session_id: "{session_uuid}"

workflow_state:
  current_phase: enrich|ready|implementing|validation|docs
  current_state_label: state:enrich
  phase_started_at: "ISO-8601"
  phase_attempt: 1

issue_snapshot:
  title: Issue title
  body: Full issue body at checkpoint
  labels:
    - state:enrich
    - type:feature
    - domain:security
  assignees: []
  branch: feature/issue-42-jwt-refresh

team_state:
  team_id: TEAM-20250114-a3f2
  members:
    - agent: orchestrator
      role: coordinator
      status: active
      tasks_completed: 2
      tasks_pending: 1

execution_graph:
  graph_id: GRAPH-20250114-a3f2
  completed_batches:
    - 1
    - 2
  current_batch: 3
  node_states: {}

agent_outputs:
  system-architect:
    create_spec:
      completed_at: "ISO-8601"
      output_summary: Spec created for JWT refresh token feature
      output_location: .claude/crunch/42/spec.md

acp_state:
  pending_messages:
    - MSG-ids
  awaiting_response:
    - MSG-ids
  conflicts_in_progress:
    - CONF-ids

knowledge_context:
  injected_entries:
    - KE-ids
  captured_entries:
    - KE-ids
  pending_captures: []

files_modified:
  - path: src/auth/tokens.py
    status: modified
    staged: true

recovery_info:
  can_resume: true
  resume_from: batch_3_task_1
  prerequisites_met: true
  warnings: []
```

## Checkpoint Triggers

### Automatic Triggers

| Event             | Checkpoint Type     | Description                    |
| ----------------- | ------------------- | ------------------------------ |
| Phase transition  | `phase_transition`  | Before moving to next phase    |
| Batch completion  | `batch_complete`    | After each execution batch     |
| Agent completion  | `agent_complete`    | After significant agent output |
| Conflict detected | `conflict_start`    | Before debate begins           |
| Conflict resolved | `conflict_resolved` | After resolution applied       |

### Manual Triggers

| Trigger                 | Checkpoint Type  | Description                 |
| ----------------------- | ---------------- | --------------------------- |
| User interrupt (Ctrl+C) | `user_interrupt` | Immediate save on interrupt |
| Session timeout         | `session_end`    | Before session terminates   |
| `/crunch --checkpoint`  | `manual`         | User-requested checkpoint   |

## Checkpoint Lifecycle

### Creation

```python
def create_checkpoint(workflow, trigger_type):
    """Create a new checkpoint."""

    checkpoint = {
        "checkpoint_id": generate_checkpoint_id(workflow.phase),
        "version": "1.0",
        "created_at": now_iso8601(),
        "issue_number": workflow.issue_number,
        "session_id": workflow.session_id,

        "workflow_state": capture_workflow_state(workflow),
        "issue_snapshot": capture_issue_snapshot(workflow.issue),
        "team_state": capture_team_state(workflow.team),
        "execution_graph": capture_graph_state(workflow.graph),
        "agent_outputs": capture_agent_outputs(workflow),
        "acp_state": capture_acp_state(workflow.acp),
        "knowledge_context": capture_knowledge_context(workflow),
        "files_modified": capture_file_state(workflow),

        "recovery_info": generate_recovery_info(workflow)
    }

    # Save checkpoint
    save_checkpoint_to_disk(checkpoint)

    # Update checkpoint index
    update_checkpoint_index(checkpoint)

    # Cleanup old checkpoints
    cleanup_old_checkpoints(workflow.issue_number)

    return checkpoint
```

### Storage

Checkpoints are stored in `.claude/crunch/{issue_number}/checkpoints/`:

```
.claude/crunch/42/checkpoints/
├── index.yaml                    # Checkpoint index
├── CP-enrich-20250114-a3f2.yaml  # Individual checkpoints
├── CP-enrich-20250114-b4c5.yaml
└── latest -> CP-enrich-20250114-b4c5.yaml  # Symlink to latest
```

### Index Structure

```yaml
issue_number: 42
checkpoints:
  - checkpoint_id: CP-enrich-20250114-a3f2
    created_at: "2025-01-14T10:00:00Z"
    trigger: phase_transition
    phase: enrich
    can_resume: true
  - checkpoint_id: CP-enrich-20250114-b4c5
    created_at: "2025-01-14T10:30:00Z"
    trigger: batch_complete
    phase: enrich
    can_resume: true
latest: CP-enrich-20250114-b4c5
total_checkpoints: 2
```

## Recovery Process

### Resume Algorithm

```python
def resume_from_checkpoint(issue_number, checkpoint_id=None):
    """Resume workflow from a checkpoint."""

    # Load checkpoint
    if checkpoint_id:
        checkpoint = load_checkpoint(issue_number, checkpoint_id)
    else:
        checkpoint = load_latest_checkpoint(issue_number)

    if not checkpoint:
        raise NoCheckpointError(f"No checkpoint found for issue #{issue_number}")

    # Validate checkpoint
    if not checkpoint.recovery_info.can_resume:
        raise CannotResumeError(checkpoint.recovery_info.warnings)

    # Restore state
    workflow = create_workflow_from_checkpoint(checkpoint)

    # Verify prerequisites
    verify_prerequisites(workflow, checkpoint)

    # Restore issue state if needed
    sync_issue_state(workflow.issue, checkpoint.issue_snapshot)

    # Restore execution graph
    workflow.graph = restore_graph_state(checkpoint.execution_graph)

    # Resume from appropriate point
    resume_point = checkpoint.recovery_info.resume_from
    return workflow, resume_point
```

### State Restoration

```python
def restore_graph_state(graph_checkpoint):
    """Restore execution graph from checkpoint."""

    graph = ExecutionGraph(graph_checkpoint.graph_id)

    for node_id, node_state in graph_checkpoint.node_states.items():
        node = graph.get_node(node_id)
        node.status = node_state["status"]
        node.result = node_state.get("result")
        node.started_at = node_state.get("started_at")
        node.completed_at = node_state.get("completed_at")

    # Regenerate batches from pending tasks
    graph.batches = create_batches_from_pending(graph)
    graph.current_batch = graph_checkpoint.current_batch

    return graph
```

## Checkpoint Retention

### Retention Policy

| Checkpoint Type     | Retention              |
| ------------------- | ---------------------- |
| `phase_transition`  | Keep all (milestone)   |
| `batch_complete`    | Keep last 3 per phase  |
| `agent_complete`    | Keep last 1 per phase  |
| `conflict_resolved` | Keep all (audit trail) |
| `user_interrupt`    | Keep last 1            |
| `manual`            | Keep all               |

### Cleanup Logic

```python
def cleanup_old_checkpoints(issue_number, retention_policy):
    """Remove old checkpoints according to retention policy."""

    checkpoints = load_checkpoint_index(issue_number)

    for checkpoint_type, max_keep in retention_policy.items():
        type_checkpoints = [cp for cp in checkpoints if cp.trigger == checkpoint_type]

        if len(type_checkpoints) > max_keep:
            # Sort by created_at, keep newest
            to_remove = sorted(type_checkpoints, key=lambda x: x.created_at)[:-max_keep]

            for cp in to_remove:
                delete_checkpoint(cp.checkpoint_id)
```

## Configuration Options

### In `.claude/checkpoint-config.yaml`:

```yaml
checkpoint:
  enabled: true
  auto_checkpoint: true
  checkpoint_triggers:
    phase_transition: true
    batch_complete: true
    agent_complete: false
    conflict_events: true
  retention:
    phase_transition: -1
    batch_complete: 3
    agent_complete: 1
    conflict_resolved: -1
    user_interrupt: 1
    manual: -1
  storage:
    location: ".claude/crunch/{issue}/checkpoints"
    compress: false
    max_size_mb: 50
```

## Edge Cases

### Handling Stale Checkpoints

```python
def validate_checkpoint_freshness(checkpoint):
    """Check if checkpoint is still valid."""

    # Check if issue state matches
    current_issue = fetch_issue(checkpoint.issue_number)

    if current_issue.labels != checkpoint.issue_snapshot.labels:
        return {
            "can_resume": True,
            "warnings": ["Issue labels changed since checkpoint"],
            "action": "proceed_with_warning"
        }

    if current_issue.body != checkpoint.issue_snapshot.body:
        return {
            "can_resume": True,
            "warnings": ["Issue body modified since checkpoint"],
            "action": "prompt_user"
        }

    # Check if branch still exists
    if not branch_exists(checkpoint.issue_snapshot.branch):
        return {
            "can_resume": False,
            "warnings": ["Branch no longer exists"],
            "action": "start_fresh"
        }

    return {"can_resume": True, "warnings": [], "action": "resume"}
```

### Conflicting Changes

When external changes conflict with checkpoint:

1. **Issue body changed**: Prompt user to confirm resume
2. **Labels changed**: Update workflow state from current labels
3. **Branch deleted**: Cannot resume, must start fresh
4. **Files modified externally**: Show diff, ask user to reconcile

## Integration Points

### With Orchestrator

Orchestrator manages checkpoints during coordination:

- Creates checkpoint before each batch
- Restores from checkpoint on retry
- Includes checkpoint state in progress reports

### With ACP

ACP messages are checkpointed:

- Pending messages saved for retry
- Response messages preserved for recovery
- Conflict state captured for resolution resume

### With Knowledge Base

Knowledge context checkpointed:

- Injected entries recorded
- Captured entries tracked
- Prevents duplicate captures on resume
