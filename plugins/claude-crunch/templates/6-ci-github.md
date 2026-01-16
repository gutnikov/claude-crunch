## GitHub MCP Commands

These GitHub MCP commands are commonly used during workflow and CI/CD operations.

### Issue Management

#### Get Label IDs

List all labels in the repository:

```
mcp__github__list_labels(
    owner: "{org}",
    repo: "{repo}"
)
```

#### Get Issue Details

Read issue information including current labels:

```
mcp__github__get_issue(
    owner: "{org}",
    repo: "{repo}",
    issue_number: {issue_number}
)
```

#### Update Issue Labels (State Transitions)

Set labels on an issue (used for state transitions):

```
mcp__github__set_issue_labels(
    owner: "{org}",
    repo: "{repo}",
    issue_number: {issue_number},
    labels: ["type:bug", "state:implementing"]
)
```

**Example - Bug TODO → IN-PROGRESS:**

```
mcp__github__set_issue_labels(
    owner: "{org}",
    repo: "{repo}",
    issue_number: 3,
    labels: ["type:bug", "state:implementing"]
)
```

#### Update Issue Body

Update issue description (for progress notes, blockers):

```
mcp__github__update_issue(
    owner: "{org}",
    repo: "{repo}",
    issue_number: {issue_number},
    body: "{updated markdown body}"
)
```

#### Close Issue

Close an issue when work is complete:

```
mcp__github__update_issue(
    owner: "{org}",
    repo: "{repo}",
    issue_number: {issue_number},
    state: "closed"
)
```

#### Create Issue

Create a new bug, feature, or decomposition issue:

```
mcp__github__create_issue(
    owner: "{org}",
    repo: "{repo}",
    title: "{issue title}",
    body: "{issue description with DOD}",
    labels: ["type:bug", "state:backlog"]
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
  - **Verification:** Run `npm test auth`, all tests pass
```

#### Add Issue Comment

Add a comment to an issue (for status updates, questions):

```
mcp__github__create_issue_comment(
    owner: "{org}",
    repo: "{repo}",
    issue_number: {issue_number},
    body: "{comment text}"
)
```

#### List Issue Comments

Get all comments on an issue:

```
mcp__github__list_issue_comments(
    owner: "{org}",
    repo: "{repo}",
    issue_number: {issue_number}
)
```

---

### Pull Request Commands

#### Create Pull Request

Create a PR linking to the issue:

```
mcp__github__create_pull_request(
    owner: "{org}",
    repo: "{repo}",
    title: "Fix #3: container restart issue",
    body: "## Summary\n\nFixes #3\n\n## Changes\n\n- Fixed restart policy",
    head: "fix/issue-3-container-restart",
    base: "main"
)
```

#### Get PR Details

Read pull request information:

```
mcp__github__get_pull_request(
    owner: "{org}",
    repo: "{repo}",
    pull_number: {pr_number}
)
```

#### List Pull Requests

List open PRs in repository:

```
mcp__github__list_pull_requests(
    owner: "{org}",
    repo: "{repo}",
    state: "open"
)
```

#### Merge Pull Request

Merge a PR (after review approval):

```
mcp__github__merge_pull_request(
    owner: "{org}",
    repo: "{repo}",
    pull_number: {pr_number},
    merge_method: "merge"  // or "rebase", "squash"
)
```

#### Create PR Review

Add a review to a PR:

```
mcp__github__create_pull_request_review(
    owner: "{org}",
    repo: "{repo}",
    pull_number: {pr_number},
    body: "{review comment}",
    event: "COMMENT"  // or "APPROVE", "REQUEST_CHANGES"
)
```

---

### Repository Commands

#### Get Repository Info

Get repository details:

```
mcp__github__get_repo(
    owner: "{org}",
    repo: "{repo}"
)
```

#### List Branches

List all branches in repository:

```
mcp__github__list_branches(
    owner: "{org}",
    repo: "{repo}"
)
```

#### Get Branch Info

Get details about a specific branch:

```
mcp__github__get_branch(
    owner: "{org}",
    repo: "{repo}",
    branch: "main"
)
```

#### List Commits

Get recent commits on a branch:

```
mcp__github__list_commits(
    owner: "{org}",
    repo: "{repo}",
    sha: "main"
)
```

---

### GitHub Actions Commands

> **Note:** Actions commands are used to trigger deployments and check workflow status.

#### Trigger Workflow (Staging Deploy)

Trigger a workflow run for staging deployment:

```
mcp__github__create_workflow_dispatch(
    owner: "{org}",
    repo: "{repo}",
    workflow_id: "deploy-staging.yml",
    ref: "main"
)
```

#### Trigger Workflow (Production Deploy)

Trigger a workflow run for production deployment:

```
mcp__github__create_workflow_dispatch(
    owner: "{org}",
    repo: "{repo}",
    workflow_id: "deploy-production.yml",
    ref: "main"
)
```

#### List Workflow Runs

Get status of recent workflow runs:

```
mcp__github__list_workflow_runs(
    owner: "{org}",
    repo: "{repo}",
    workflow_id: "ci.yml"
)
```

#### Get Workflow Run Status

Check status of a specific workflow run:

```
mcp__github__get_workflow_run(
    owner: "{org}",
    repo: "{repo}",
    run_id: {run_id}
)
```

#### Download Workflow Artifacts

Download artifacts from a workflow run:

```
mcp__github__download_workflow_artifact(
    owner: "{org}",
    repo: "{repo}",
    artifact_id: {artifact_id}
)
```

---

## Quick Reference Table

### Issue Management

| Action        | MCP Command                         |
| ------------- | ----------------------------------- |
| Get labels    | `mcp__github__list_labels`          |
| Read issue    | `mcp__github__get_issue`            |
| Create issue  | `mcp__github__create_issue`         |
| Update issue  | `mcp__github__update_issue`         |
| Close issue   | `mcp__github__update_issue`         |
| Change state  | `mcp__github__set_issue_labels`     |
| Add comment   | `mcp__github__create_issue_comment` |
| List comments | `mcp__github__list_issue_comments`  |

### Pull Request Management

| Action    | MCP Command                               |
| --------- | ----------------------------------------- |
| Create PR | `mcp__github__create_pull_request`        |
| Get PR    | `mcp__github__get_pull_request`           |
| List PRs  | `mcp__github__list_pull_requests`         |
| Merge PR  | `mcp__github__merge_pull_request`         |
| Review PR | `mcp__github__create_pull_request_review` |

### Repository Operations

| Action        | MCP Command                  |
| ------------- | ---------------------------- |
| Get repo info | `mcp__github__get_repo`      |
| List branches | `mcp__github__list_branches` |
| Get branch    | `mcp__github__get_branch`    |
| List commits  | `mcp__github__list_commits`  |

### GitHub Actions

| Action             | MCP Command                               |
| ------------------ | ----------------------------------------- |
| Trigger workflow   | `mcp__github__create_workflow_dispatch`   |
| List workflow runs | `mcp__github__list_workflow_runs`         |
| Get run status     | `mcp__github__get_workflow_run`           |
| Get artifacts      | `mcp__github__download_workflow_artifact` |
