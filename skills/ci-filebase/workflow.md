# File-based CI Workflow Implementation

Detailed instructions for implementing file-based CI operations. This skill provides local file storage as an alternative to MCP-based CI platforms.

---

## Directory Structure

```
.claude/ci-filebase/
├── issues/
│   ├── 1.json
│   ├── 2.json
│   └── ...
├── prs/
│   ├── 1.json
│   └── ...
├── labels.json
└── counter.json
```

---

## JSON Schemas

### Issue Schema (`issues/{id}.json`)

```json
{
  "id": 1,
  "title": "Issue title",
  "body": "Issue description in markdown",
  "state": "open",
  "labels": ["type:feature", "state:backlog"],
  "created_at": "2024-01-14T10:00:00Z",
  "updated_at": "2024-01-14T10:00:00Z",
  "closed_at": null,
  "comments": [
    {
      "id": 1,
      "body": "Comment text",
      "created_at": "2024-01-14T10:05:00Z"
    }
  ]
}
```

### PR Schema (`prs/{id}.json`)

```json
{
  "id": 1,
  "title": "PR title",
  "body": "PR description",
  "state": "open",
  "head_branch": "feature/issue-1-auth",
  "base_branch": "main",
  "issue_ref": 1,
  "labels": [],
  "created_at": "2024-01-14T10:00:00Z",
  "merged_at": null,
  "reviews": [
    {
      "id": 1,
      "state": "APPROVED",
      "body": "LGTM",
      "created_at": "2024-01-14T11:00:00Z"
    }
  ]
}
```

### Labels Schema (`labels.json`)

```json
{
  "labels": [
    {"name": "type:bug", "color": "d73a4a", "description": "Bug fix"},
    {"name": "type:feature", "color": "0075ca", "description": "New feature"},
    {"name": "state:backlog", "color": "fbca04", "description": "In backlog"},
    {"name": "state:enrich", "color": "c2e0c6", "description": "Being enriched"},
    {"name": "state:ready", "color": "0e8a16", "description": "Ready for implementation"},
    {"name": "state:implementing", "color": "1d76db", "description": "In progress"},
    {"name": "state:validation", "color": "5319e7", "description": "Validating"},
    {"name": "state:docs", "color": "006b75", "description": "Documentation"},
    {"name": "state:ready-to-merge", "color": "0e8a16", "description": "Ready to merge"},
    {"name": "state:done", "color": "ffffff", "description": "Completed"},
    {"name": "domain:security", "color": "b60205", "description": "Security-related"},
    {"name": "domain:infra", "color": "5319e7", "description": "Infrastructure-related"},
    {"name": "domain:frontend", "color": "1d76db", "description": "Frontend-related"},
    {"name": "domain:api", "color": "0e8a16", "description": "API-related"}
  ]
}
```

### Counter Schema (`counter.json`)

```json
{
  "issue": 0,
  "pr": 0,
  "comment": 0
}
```

---

## Command Implementations

### `init` Command

**Purpose**: Initialize ci-filebase directory structure.

```
1. Check if .claude/ci-filebase/ exists
   IF exists:
     Show warning: "ci-filebase already initialized"
     Ask user to confirm re-initialization
     IF not confirmed: abort

2. Create directory structure:
   mkdir -p .claude/ci-filebase/issues
   mkdir -p .claude/ci-filebase/prs

3. Create labels.json with default labels (see schema above)

4. Create counter.json:
   {
     "issue": 0,
     "pr": 0,
     "comment": 0
   }

5. Report success:
   "Initialized ci-filebase at .claude/ci-filebase/
    - Created issues/ directory
    - Created prs/ directory
    - Created labels.json with {N} default labels
    - Created counter.json"
```

### `issue create` Command

**Syntax**: `/ci-filebase issue create "title" [--body "..."] [--labels l1,l2]`

```
1. Read counter.json
2. Increment issue counter
3. Create issue object:
   {
     "id": {new_id},
     "title": {title},
     "body": {body or ""},
     "state": "open",
     "labels": {parsed labels array or []},
     "created_at": {ISO timestamp},
     "updated_at": {ISO timestamp},
     "closed_at": null,
     "comments": []
   }

4. Write to .claude/ci-filebase/issues/{id}.json
5. Update counter.json with new issue count
6. Report success:
   "Created issue #{id}: {title}
    State: open
    Labels: {labels or 'none'}"
```

### `issue get` Command

**Syntax**: `/ci-filebase issue get {id}`

```
1. Read .claude/ci-filebase/issues/{id}.json
   IF not found: report error "Issue #{id} not found"

2. Display formatted output:
   "Issue #{id}: {title}
    State: {state}
    Labels: {labels}
    Created: {created_at}
    Updated: {updated_at}

    {body}

    Comments ({count}):
    ---
    [{comment.created_at}] {comment.body}
    ..."
```

### `issue update` Command

**Syntax**: `/ci-filebase issue update {id} [--title "..."] [--body "..."]`

```
1. Read existing issue
   IF not found: report error

2. Apply updates:
   IF --title provided: issue.title = new_title
   IF --body provided: issue.body = new_body

3. Update timestamp:
   issue.updated_at = {ISO timestamp}

4. Write back to file

5. Report success:
   "Updated issue #{id}
    {list of changed fields}"
```

### `issue label` Command

**Syntax**: `/ci-filebase issue label {id} label1 [label2 ...]`

```
1. Read existing issue
   IF not found: report error

2. Parse labels from arguments
   (space-separated or comma-separated)

3. Replace labels array:
   issue.labels = {new labels}

4. Update timestamp:
   issue.updated_at = {ISO timestamp}

5. Write back to file

6. Report success:
   "Updated issue #{id} labels:
    - {label1}
    - {label2}
    ..."
```

### `issue close` Command

**Syntax**: `/ci-filebase issue close {id}`

```
1. Read existing issue
   IF not found: report error
   IF already closed: report "Issue #{id} is already closed"

2. Update state:
   issue.state = "closed"
   issue.closed_at = {ISO timestamp}
   issue.updated_at = {ISO timestamp}

3. Write back to file

4. Report success:
   "Closed issue #{id}: {title}"
```

### `issue list` Command

**Syntax**: `/ci-filebase issue list [--state open|closed] [--labels l1,l2]`

```
1. Read all files from .claude/ci-filebase/issues/

2. Apply filters:
   IF --state provided:
     filter to matching state
   IF --labels provided:
     filter to issues containing ALL specified labels

3. Sort by ID descending (newest first)

4. Display formatted list:
   "{State} issues{filter info}:
    #{id}: {title} [{labels}]
    #{id}: {title} [{labels}]
    ...

    Total: {count} issues"
```

### `issue comment` Command

**Syntax**: `/ci-filebase issue comment {id} "comment text"`

```
1. Read existing issue
   IF not found: report error

2. Read counter.json
3. Increment comment counter

4. Create comment object:
   {
     "id": {new_comment_id},
     "body": {comment_text},
     "created_at": {ISO timestamp}
   }

5. Append to issue.comments array

6. Update issue.updated_at

7. Write back issue file
8. Update counter.json

9. Report success:
   "Added comment #{comment_id} to issue #{id}"
```

### `pr create` Command

**Syntax**: `/ci-filebase pr create --title "..." --head "branch" --base "branch" [--body "..."] [--issue N]`

```
1. Read counter.json
2. Increment PR counter

3. Create PR object:
   {
     "id": {new_id},
     "title": {title},
     "body": {body or ""},
     "state": "open",
     "head_branch": {head},
     "base_branch": {base},
     "issue_ref": {issue or null},
     "labels": [],
     "created_at": {ISO timestamp},
     "merged_at": null,
     "reviews": []
   }

4. Write to .claude/ci-filebase/prs/{id}.json
5. Update counter.json

6. Report success:
   "Created PR #{id}: {title}
    Head: {head_branch}
    Base: {base_branch}
    Linked issue: #{issue_ref or 'none'}"
```

### `pr get` Command

**Syntax**: `/ci-filebase pr get {id}`

```
1. Read .claude/ci-filebase/prs/{id}.json
   IF not found: report error

2. Display formatted output:
   "PR #{id}: {title}
    State: {state}
    Head: {head_branch} -> Base: {base_branch}
    Issue: #{issue_ref or 'none'}
    Created: {created_at}
    {IF merged: Merged: {merged_at}}

    {body}

    Reviews ({count}):
    ---
    [{review.state}] {review.body}
    ..."
```

### `pr merge` Command

**Syntax**: `/ci-filebase pr merge {id}`

```
1. Read existing PR
   IF not found: report error
   IF already merged: report "PR #{id} is already merged"

2. Update state:
   pr.state = "merged"
   pr.merged_at = {ISO timestamp}

3. Write back to file

4. Report success:
   "Merged PR #{id}: {title}
    Branch {head_branch} -> {base_branch}"
```

### `pr list` Command

**Syntax**: `/ci-filebase pr list [--state open|merged]`

```
1. Read all files from .claude/ci-filebase/prs/

2. Apply filters:
   IF --state provided:
     filter to matching state

3. Sort by ID descending (newest first)

4. Display formatted list:
   "{State} PRs{filter info}:
    #{id}: {title} ({head} -> {base})
    #{id}: {title} ({head} -> {base})
    ...

    Total: {count} PRs"
```

### `pr review` Command

**Syntax**: `/ci-filebase pr review {id} --state APPROVE|COMMENT|REQUEST_CHANGES [--body "..."]`

```
1. Read existing PR
   IF not found: report error

2. Read counter.json
3. Increment comment counter (used for review IDs too)

4. Create review object:
   {
     "id": {new_review_id},
     "state": {state},
     "body": {body or ""},
     "created_at": {ISO timestamp}
   }

5. Append to pr.reviews array
6. Write back PR file
7. Update counter.json

8. Report success:
   "Added {state} review to PR #{id}"
```

### `labels` Command

**Syntax**: `/ci-filebase labels`

```
1. Read .claude/ci-filebase/labels.json

2. Display formatted list:
   "Available labels:

    Type labels:
    - type:bug - Bug fix
    - type:feature - New feature

    State labels:
    - state:backlog - In backlog
    - state:enrich - Being enriched
    ...

    Domain labels:
    - domain:security - Security-related
    ..."
```

---

## Integration with /crunch

When `/crunch` detects `Filebase` as the CI platform in CLAUDE.md, it should use these skill commands instead of MCP calls.

### Operation Mapping

| /crunch needs to... | MCP call (GitHub example) | Filebase equivalent |
|---------------------|---------------------------|---------------------|
| Fetch issue | `mcp__github__get_issue(owner, repo, issue_number)` | `/ci-filebase issue get {id}` |
| Create issue | `mcp__github__create_issue(owner, repo, title, body, labels)` | `/ci-filebase issue create "{title}" --body "{body}" --labels {labels}` |
| Update issue body | `mcp__github__update_issue(owner, repo, issue_number, body)` | `/ci-filebase issue update {id} --body "{body}"` |
| Set labels | `mcp__github__set_issue_labels(owner, repo, issue_number, labels)` | `/ci-filebase issue label {id} {labels}` |
| Close issue | `mcp__github__update_issue(owner, repo, issue_number, state: "closed")` | `/ci-filebase issue close {id}` |
| Add comment | `mcp__github__create_issue_comment(owner, repo, issue_number, body)` | `/ci-filebase issue comment {id} "{body}"` |
| List labels | `mcp__github__list_labels(owner, repo)` | `/ci-filebase labels` |
| Create PR | `mcp__github__create_pull_request(...)` | `/ci-filebase pr create --title "..." --head "..." --base "..."` |
| Merge PR | `mcp__github__merge_pull_request(...)` | `/ci-filebase pr merge {id}` |

### State Extraction

When parsing issue data from filebase, extract:

```
type = issue.labels.find(l => l.startsWith("type:"))
state = issue.labels.find(l => l.startsWith("state:"))
domain = issue.labels.filter(l => l.startsWith("domain:"))
```

If no `state:` label exists, treat as INPUT phase.

---

## Error Handling

| Condition | Error Message | Recovery |
|-----------|---------------|----------|
| Not initialized | "ci-filebase not initialized. Run `/ci-filebase init` first." | Run init |
| Issue not found | "Issue #{id} not found" | Check ID |
| PR not found | "PR #{id} not found" | Check ID |
| File read error | "Error reading {path}: {error}" | Check permissions |
| File write error | "Error writing {path}: {error}" | Check permissions |
| Invalid JSON | "Corrupted file at {path}" | Manual fix or recreate |

---

## Timestamps

All timestamps use ISO 8601 format in UTC:

```
2024-01-14T10:00:00Z
```

Generate with: `new Date().toISOString()` or equivalent.

---

## File Permissions

Files should be created with standard permissions:
- Directories: 755
- Files: 644

The `.claude/` directory is typically gitignored. If tracking ci-filebase in git is desired, add exception:

```gitignore
# In .gitignore
.claude/*
!.claude/ci-filebase/
```
