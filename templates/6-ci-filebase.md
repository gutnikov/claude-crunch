## File-based CI Commands

File-based CI provides local issue and PR tracking without requiring external CI platforms (GitHub/GitLab/Gitea). All data is stored in `.claude/ci-filebase/` as JSON files.

> **When to use:** Projects without access to hosted CI platforms, local-only development, air-gapped environments, or rapid prototyping.

### Issue Management

#### Initialize Filebase

Set up the ci-filebase directory structure (run once per project):

```
/ci-filebase init
```

Creates:
- `.claude/ci-filebase/issues/` - Issue storage
- `.claude/ci-filebase/prs/` - PR storage
- `.claude/ci-filebase/labels.json` - Default labels
- `.claude/ci-filebase/counter.json` - ID counters

#### List Labels

List all available labels:

```
/ci-filebase labels
```

#### Get Issue Details

Read issue information including current labels:

```
/ci-filebase issue get {issue_id}
```

#### Update Issue Labels (State Transitions)

Set labels on an issue (used for state transitions):

```
/ci-filebase issue label {issue_id} type:bug state:implementing
```

**Example - Bug TODO -> IN-PROGRESS:**

```
/ci-filebase issue label 3 type:bug state:implementing
```

#### Update Issue Body

Update issue description (for progress notes, blockers):

```
/ci-filebase issue update {issue_id} --body "{updated markdown body}"
```

Or update title:

```
/ci-filebase issue update {issue_id} --title "{new title}"
```

#### Close Issue

Close an issue when work is complete:

```
/ci-filebase issue close {issue_id}
```

#### Create Issue

Create a new bug or task issue:

```
/ci-filebase issue create "{issue title}" --body "{issue description}" --labels type:bug,state:backlog
```

**Example:**

```
/ci-filebase issue create "Fix container restart issue" --body "Container fails to restart after config change" --labels type:bug,state:backlog
```

#### Add Issue Comment

Add a comment to an issue (for status updates, questions):

```
/ci-filebase issue comment {issue_id} "{comment text}"
```

#### List Issues

List all issues (optionally filter by state or labels):

```
/ci-filebase issue list
/ci-filebase issue list --state open
/ci-filebase issue list --labels state:implementing
```

---

### Pull Request Commands

#### Create Pull Request

Create a PR record linking to an issue:

```
/ci-filebase pr create --title "Fix #3: container restart issue" --body "## Summary\n\nFixes #3" --head "fix/issue-3-container-restart" --base "main" --issue 3
```

#### Get PR Details

Read pull request information:

```
/ci-filebase pr get {pr_id}
```

#### List Pull Requests

List PRs (optionally filter by state):

```
/ci-filebase pr list
/ci-filebase pr list --state open
```

#### Merge Pull Request

Mark a PR as merged:

```
/ci-filebase pr merge {pr_id}
```

#### Add PR Review

Add a review to a PR:

```
/ci-filebase pr review {pr_id} --body "{review comment}" --state APPROVE
```

Review states: `COMMENT`, `APPROVE`, `REQUEST_CHANGES`

---

### Repository Commands

> **Note:** File-based CI relies on local git commands for repository operations. These are not managed by `/ci-filebase`.

#### List Branches

```bash
git branch -a
```

#### Get Current Branch

```bash
git branch --show-current
```

#### List Commits

```bash
git log --oneline -10
```

---

### CI/CD Commands

> **Note:** File-based CI does not include automated pipelines. Use manual commands or local scripts for build/test/deploy operations.

#### Run Tests (Manual)

```bash
# Run your project's test command
npm test
pytest
go test ./...
```

#### Build (Manual)

```bash
# Run your project's build command
npm run build
go build
make
```

---

## Quick Reference Table

### Issue Management

| Action        | Command                                |
| ------------- | -------------------------------------- |
| Initialize    | `/ci-filebase init`                    |
| Get labels    | `/ci-filebase labels`                  |
| Read issue    | `/ci-filebase issue get {id}`          |
| Create issue  | `/ci-filebase issue create "{title}"`  |
| Update issue  | `/ci-filebase issue update {id}`       |
| Close issue   | `/ci-filebase issue close {id}`        |
| Change state  | `/ci-filebase issue label {id} {labels}` |
| Add comment   | `/ci-filebase issue comment {id} "{text}"` |
| List issues   | `/ci-filebase issue list`              |

### Pull Request Management

| Action    | Command                                |
| --------- | -------------------------------------- |
| Create PR | `/ci-filebase pr create --title "..."`  |
| Get PR    | `/ci-filebase pr get {id}`              |
| List PRs  | `/ci-filebase pr list`                  |
| Merge PR  | `/ci-filebase pr merge {id}`            |
| Review PR | `/ci-filebase pr review {id}`           |

### File Storage

| Resource | Location                              |
| -------- | ------------------------------------- |
| Issues   | `.claude/ci-filebase/issues/{id}.json` |
| PRs      | `.claude/ci-filebase/prs/{id}.json`    |
| Labels   | `.claude/ci-filebase/labels.json`      |
| Counters | `.claude/ci-filebase/counter.json`     |

---

## Comparison with MCP-based CI

| Feature | GitHub/GitLab/Gitea MCP | File-based CI |
|---------|------------------------|---------------|
| Issue tracking | Remote API | Local JSON files |
| PR management | Remote API | Local JSON files |
| CI/CD pipelines | Automated | Manual scripts |
| Collaboration | Multi-user | Single-user |
| Offline support | Requires network | Fully offline |
| Setup complexity | MCP + tokens | Zero config |
