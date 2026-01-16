## Gitea MCP Commands

These Gitea MCP commands are commonly used during workflow and CI/CD operations.

### Issue Management

#### Get Label IDs

Before updating labels, retrieve current label IDs:

```
mcp__gitea__list_repo_labels(
    owner: "gitadmin",
    repo: "homebase"
)
```

#### Get Issue Details

Read issue information including current labels:

```
mcp__gitea__get_issue(
    owner: "gitadmin",
    repo: "homebase",
    index: {issue_number}
)
```

#### Update Issue Labels (State Transitions)

Replace all labels on an issue (used for state transitions):

```
mcp__gitea__replace_issue_labels(
    owner: "gitadmin",
    repo: "homebase",
    index: {issue_number},
    labels: [{type_label_id}, {new_state_label_id}]
)
```

**Example - Bug TODO → IN-PROGRESS:**

```
mcp__gitea__replace_issue_labels(
    owner: "gitadmin",
    repo: "homebase",
    index: 3,
    labels: [1, 5]  // type:bug=1, state:implementing=5
)
```

#### Update Issue Body

Update issue description (for progress notes, blockers):

```
mcp__gitea__update_issue(
    owner: "gitadmin",
    repo: "homebase",
    index: {issue_number},
    body: "{updated markdown body}"
)
```

#### Close Issue

Close an issue when work is complete:

```
mcp__gitea__update_issue(
    owner: "gitadmin",
    repo: "homebase",
    index: {issue_number},
    state: "closed"
)
```

#### Create Issue

Create a new bug, feature, or decomposition issue:

```
mcp__gitea__create_issue(
    owner: "gitadmin",
    repo: "homebase",
    title: "{issue title}",
    body: "{issue description with DOD}",
    labels: [{type_label_id}, {state_label_id}]
)
```

**Available type labels:** `type:bug`, `type:feature`, `type:decomposition`

**DOD Requirement:** Every issue body MUST contain a "Definition of Done (DOD)" section. Each DOD item MUST have a verification method. See `templates/5-workflow.md` for full DOD format and examples.

**Example issue body with DOD:**

```markdown
Implement user login functionality.

## Definition of Done (DOD)

- [ ] Login endpoint returns JWT token
  - **Verification:** POST /api/login with valid credentials returns 200 with token
- [ ] Invalid credentials return 401
  - **Verification:** POST /api/login with wrong password returns 401
- [ ] Unit tests pass
  - **Verification:** Run `go test ./auth/...`, all tests pass
```

#### Add Issue Comment

Add a comment to an issue (for status updates, questions):

```
mcp__gitea__create_issue_comment(
    owner: "gitadmin",
    repo: "homebase",
    index: {issue_number},
    body: "{comment text}"
)
```

#### List Issue Comments

Get all comments on an issue:

```
mcp__gitea__list_issue_comments(
    owner: "gitadmin",
    repo: "homebase",
    index: {issue_number}
)
```

---

### Pull Request Commands

#### Create Pull Request

Create a PR linking to the issue:

```
mcp__gitea__create_pull_request(
    owner: "gitadmin",
    repo: "homebase",
    title: "Fix #3: gitea container restart",
    body: "## Summary\n\nFixes #3\n\n## Changes\n\n- Fixed restart policy",
    head: "fix/issue-3-gitea-container-restart",
    base: "main"
)
```

#### Get PR Details

Read pull request information:

```
mcp__gitea__get_pull_request(
    owner: "gitadmin",
    repo: "homebase",
    index: {pr_number}
)
```

#### List Pull Requests

List open PRs in repository:

```
mcp__gitea__list_pull_requests(
    owner: "gitadmin",
    repo: "homebase",
    state: "open"
)
```

#### Merge Pull Request

Merge a PR (after review approval):

```
mcp__gitea__merge_pull_request(
    owner: "gitadmin",
    repo: "homebase",
    index: {pr_number},
    merge_style: "merge"  // or "rebase", "squash"
)
```

#### Add PR Review Comment

Add a review comment to a PR:

```
mcp__gitea__create_pull_review(
    owner: "gitadmin",
    repo: "homebase",
    index: {pr_number},
    body: "{review comment}",
    event: "COMMENT"  // or "APPROVE", "REQUEST_CHANGES"
)
```

---

### Repository Commands

#### Get Repository Info

Get repository details:

```
mcp__gitea__get_repo(
    owner: "gitadmin",
    repo: "homebase"
)
```

#### List Branches

List all branches in repository:

```
mcp__gitea__list_branches(
    owner: "gitadmin",
    repo: "homebase"
)
```

#### Get Branch Info

Get details about a specific branch:

```
mcp__gitea__get_branch(
    owner: "gitadmin",
    repo: "homebase",
    branch: "main"
)
```

#### List Commits

Get recent commits on a branch:

```
mcp__gitea__list_commits(
    owner: "gitadmin",
    repo: "homebase",
    sha: "main"
)
```

---

### CI/CD Commands

> **Note:** CI commands are used to trigger deployments and check pipeline status.

#### Trigger Workflow (Staging Deploy)

Trigger a workflow run for staging deployment:

```
mcp__gitea__trigger_workflow(
    owner: "gitadmin",
    repo: "homebase",
    workflow: "deploy-staging.yml",
    ref: "main"
)
```

#### Trigger Workflow (Production Deploy)

Trigger a workflow run for production deployment:

```
mcp__gitea__trigger_workflow(
    owner: "gitadmin",
    repo: "homebase",
    workflow: "deploy-production.yml",
    ref: "main"
)
```

#### List Workflow Runs

Get status of recent workflow runs:

```
mcp__gitea__list_workflow_runs(
    owner: "gitadmin",
    repo: "homebase"
)
```

#### Get Workflow Run Status

Check status of a specific workflow run:

```
mcp__gitea__get_workflow_run(
    owner: "gitadmin",
    repo: "homebase",
    run_id: {run_id}
)
```

---

## Quick Reference Table

### Issue Management

| Action        | MCP Command                        |
| ------------- | ---------------------------------- |
| Get labels    | `mcp__gitea__list_repo_labels`     |
| Read issue    | `mcp__gitea__get_issue`            |
| Create issue  | `mcp__gitea__create_issue`         |
| Update issue  | `mcp__gitea__update_issue`         |
| Close issue   | `mcp__gitea__update_issue`         |
| Change state  | `mcp__gitea__replace_issue_labels` |
| Add comment   | `mcp__gitea__create_issue_comment` |
| List comments | `mcp__gitea__list_issue_comments`  |

### Pull Request Management

| Action    | MCP Command                       |
| --------- | --------------------------------- |
| Create PR | `mcp__gitea__create_pull_request` |
| Get PR    | `mcp__gitea__get_pull_request`    |
| List PRs  | `mcp__gitea__list_pull_requests`  |
| Merge PR  | `mcp__gitea__merge_pull_request`  |
| Review PR | `mcp__gitea__create_pull_review`  |

### Repository Operations

| Action        | MCP Command                 |
| ------------- | --------------------------- |
| Get repo info | `mcp__gitea__get_repo`      |
| List branches | `mcp__gitea__list_branches` |
| Get branch    | `mcp__gitea__get_branch`    |
| List commits  | `mcp__gitea__list_commits`  |

### CI/CD Operations

| Action              | MCP Command                      |
| ------------------- | -------------------------------- |
| Trigger workflow    | `mcp__gitea__trigger_workflow`   |
| List workflow runs  | `mcp__gitea__list_workflow_runs` |
| Get workflow status | `mcp__gitea__get_workflow_run`   |
