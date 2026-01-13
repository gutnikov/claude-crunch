## GitLab MCP Commands

These GitLab MCP commands are commonly used during workflow and CI/CD operations.

### Issue Management

#### Get Label IDs

List all labels in the project:

```
mcp__gitlab__list_labels(
    project_id: "{project_id}"
)
```

#### Get Issue Details

Read issue information including current labels:

```
mcp__gitlab__get_issue(
    project_id: "{project_id}",
    issue_iid: {issue_iid}
)
```

#### Update Issue Labels (State Transitions)

Update labels on an issue (used for state transitions):

```
mcp__gitlab__update_issue(
    project_id: "{project_id}",
    issue_iid: {issue_iid},
    labels: ["type:bug", "state:implementing"]
)
```

**Example - Bug TODO → IN-PROGRESS:**

```
mcp__gitlab__update_issue(
    project_id: "{project_id}",
    issue_iid: 3,
    labels: ["type:bug", "state:implementing"]
)
```

#### Update Issue Description

Update issue description (for progress notes, blockers):

```
mcp__gitlab__update_issue(
    project_id: "{project_id}",
    issue_iid: {issue_iid},
    description: "{updated markdown body}"
)
```

#### Close Issue

Close an issue when work is complete:

```
mcp__gitlab__update_issue(
    project_id: "{project_id}",
    issue_iid: {issue_iid},
    state_event: "close"
)
```

#### Create Issue

Create a new bug or task issue:

```
mcp__gitlab__create_issue(
    project_id: "{project_id}",
    title: "{issue title}",
    description: "{issue description}",
    labels: ["type:bug", "state:backlog"]
)
```

#### Add Issue Note (Comment)

Add a note to an issue (for status updates, questions):

```
mcp__gitlab__create_issue_note(
    project_id: "{project_id}",
    issue_iid: {issue_iid},
    body: "{comment text}"
)
```

#### List Issue Notes

Get all notes on an issue:

```
mcp__gitlab__list_issue_notes(
    project_id: "{project_id}",
    issue_iid: {issue_iid}
)
```

---

### Merge Request Commands

#### Create Merge Request

Create an MR linking to the issue:

```
mcp__gitlab__create_merge_request(
    project_id: "{project_id}",
    title: "Fix #3: container restart issue",
    description: "## Summary\n\nFixes #3\n\n## Changes\n\n- Fixed restart policy",
    source_branch: "fix/issue-3-container-restart",
    target_branch: "main"
)
```

#### Get MR Details

Read merge request information:

```
mcp__gitlab__get_merge_request(
    project_id: "{project_id}",
    merge_request_iid: {mr_iid}
)
```

#### List Merge Requests

List open MRs in project:

```
mcp__gitlab__list_merge_requests(
    project_id: "{project_id}",
    state: "opened"
)
```

#### Merge Merge Request

Merge an MR (after approval):

```
mcp__gitlab__accept_merge_request(
    project_id: "{project_id}",
    merge_request_iid: {mr_iid},
    squash: false  // or true for squash merge
)
```

#### Approve Merge Request

Approve an MR:

```
mcp__gitlab__approve_merge_request(
    project_id: "{project_id}",
    merge_request_iid: {mr_iid}
)
```

#### Add MR Note (Comment)

Add a comment to an MR:

```
mcp__gitlab__create_merge_request_note(
    project_id: "{project_id}",
    merge_request_iid: {mr_iid},
    body: "{review comment}"
)
```

---

### Project Commands

#### Get Project Info

Get project details:

```
mcp__gitlab__get_project(
    project_id: "{project_id}"
)
```

#### List Branches

List all branches in project:

```
mcp__gitlab__list_branches(
    project_id: "{project_id}"
)
```

#### Get Branch Info

Get details about a specific branch:

```
mcp__gitlab__get_branch(
    project_id: "{project_id}",
    branch: "main"
)
```

#### List Commits

Get recent commits on a branch:

```
mcp__gitlab__list_commits(
    project_id: "{project_id}",
    ref_name: "main"
)
```

---

### CI/CD Pipeline Commands

> **Note:** Pipeline commands are used to trigger deployments and check pipeline status.

#### Trigger Pipeline (Staging Deploy)

Trigger a pipeline for staging deployment:

```
mcp__gitlab__create_pipeline(
    project_id: "{project_id}",
    ref: "main",
    variables: [
        { "key": "DEPLOY_ENV", "value": "staging" }
    ]
)
```

#### Trigger Pipeline (Production Deploy)

Trigger a pipeline for production deployment:

```
mcp__gitlab__create_pipeline(
    project_id: "{project_id}",
    ref: "main",
    variables: [
        { "key": "DEPLOY_ENV", "value": "production" }
    ]
)
```

#### List Pipelines

Get status of recent pipelines:

```
mcp__gitlab__list_pipelines(
    project_id: "{project_id}",
    ref: "main"
)
```

#### Get Pipeline Status

Check status of a specific pipeline:

```
mcp__gitlab__get_pipeline(
    project_id: "{project_id}",
    pipeline_id: {pipeline_id}
)
```

#### List Pipeline Jobs

Get jobs in a pipeline:

```
mcp__gitlab__list_pipeline_jobs(
    project_id: "{project_id}",
    pipeline_id: {pipeline_id}
)
```

#### Retry Failed Pipeline

Retry a failed pipeline:

```
mcp__gitlab__retry_pipeline(
    project_id: "{project_id}",
    pipeline_id: {pipeline_id}
)
```

#### Cancel Pipeline

Cancel a running pipeline:

```
mcp__gitlab__cancel_pipeline(
    project_id: "{project_id}",
    pipeline_id: {pipeline_id}
)
```

#### Play Manual Job

Trigger a manual job in a pipeline:

```
mcp__gitlab__play_job(
    project_id: "{project_id}",
    job_id: {job_id}
)
```

---

## Quick Reference Table

### Issue Management

| Action       | MCP Command                      |
| ------------ | -------------------------------- |
| Get labels   | `mcp__gitlab__list_labels`       |
| Read issue   | `mcp__gitlab__get_issue`         |
| Create issue | `mcp__gitlab__create_issue`      |
| Update issue | `mcp__gitlab__update_issue`      |
| Close issue  | `mcp__gitlab__update_issue`      |
| Add note     | `mcp__gitlab__create_issue_note` |
| List notes   | `mcp__gitlab__list_issue_notes`  |

### Merge Request Management

| Action     | MCP Command                              |
| ---------- | ---------------------------------------- |
| Create MR  | `mcp__gitlab__create_merge_request`      |
| Get MR     | `mcp__gitlab__get_merge_request`         |
| List MRs   | `mcp__gitlab__list_merge_requests`       |
| Merge MR   | `mcp__gitlab__accept_merge_request`      |
| Approve MR | `mcp__gitlab__approve_merge_request`     |
| Add note   | `mcp__gitlab__create_merge_request_note` |

### Project Operations

| Action           | MCP Command                  |
| ---------------- | ---------------------------- |
| Get project info | `mcp__gitlab__get_project`   |
| List branches    | `mcp__gitlab__list_branches` |
| Get branch       | `mcp__gitlab__get_branch`    |
| List commits     | `mcp__gitlab__list_commits`  |

### CI/CD Pipelines

| Action           | MCP Command                       |
| ---------------- | --------------------------------- |
| Trigger pipeline | `mcp__gitlab__create_pipeline`    |
| List pipelines   | `mcp__gitlab__list_pipelines`     |
| Get status       | `mcp__gitlab__get_pipeline`       |
| List jobs        | `mcp__gitlab__list_pipeline_jobs` |
| Retry pipeline   | `mcp__gitlab__retry_pipeline`     |
| Cancel pipeline  | `mcp__gitlab__cancel_pipeline`    |
| Play manual job  | `mcp__gitlab__play_job`           |
