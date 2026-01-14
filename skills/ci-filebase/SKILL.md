---
name: ci-filebase
description: Manage local file-based issue and PR tracking. Use for projects without access to hosted CI platforms (GitHub/GitLab/Gitea). Supports issue create/update/label/close, PR management, and full workflow integration with /crunch.
---

# File-based CI Management

Manage issues, PRs, and labels using local JSON files instead of remote CI platforms.

## Syntax

```
/ci-filebase <command> [options]
```

### Commands

| Command | Description |
|---------|-------------|
| `init` | Initialize ci-filebase directory structure |
| `issue` | Issue management (create, get, update, label, close, list, comment) |
| `pr` | Pull request management (create, get, merge, list, review) |
| `labels` | List available labels |
| `docker` | Local Docker CI/CD (init, ci, deploy, logs, health, stop, rebuild) |

### Issue Subcommands

| Subcommand | Example | Description |
|------------|---------|-------------|
| `create` | `/ci-filebase issue create "Title"` | Create new issue |
| `get` | `/ci-filebase issue get 1` | Get issue details |
| `update` | `/ci-filebase issue update 1 --body "..."` | Update issue body/title |
| `label` | `/ci-filebase issue label 1 type:bug` | Set issue labels |
| `close` | `/ci-filebase issue close 1` | Close an issue |
| `list` | `/ci-filebase issue list` | List all issues |
| `comment` | `/ci-filebase issue comment 1 "text"` | Add comment to issue |

### PR Subcommands

| Subcommand | Example | Description |
|------------|---------|-------------|
| `create` | `/ci-filebase pr create --title "..."` | Create new PR |
| `get` | `/ci-filebase pr get 1` | Get PR details |
| `merge` | `/ci-filebase pr merge 1` | Mark PR as merged |
| `list` | `/ci-filebase pr list` | List all PRs |
| `review` | `/ci-filebase pr review 1 --state APPROVE` | Add review to PR |

### Docker Subcommands

| Subcommand | Example | Description |
|------------|---------|-------------|
| `init` | `/ci-filebase docker init` | Initialize Docker CI infrastructure |
| `ci` | `/ci-filebase docker ci` | Run local CI pipeline (lint, test, build) |
| `deploy` | `/ci-filebase docker deploy staging` | Deploy to local Docker staging |
| `logs` | `/ci-filebase docker logs` | View staging container logs |
| `health` | `/ci-filebase docker health` | Check staging health status |
| `stop` | `/ci-filebase docker stop` | Stop local staging environment |
| `rebuild` | `/ci-filebase docker rebuild` | Rebuild and restart staging |

## Prerequisites

Before using this skill:

1. Run `/ci-filebase init` to create the directory structure
2. Ensure `CLAUDE.md` CI Platform section shows `Filebase` as active

For Docker CI/CD features (optional but recommended for VALIDATION):

3. Docker installed and running (`docker --version`)
4. Docker Compose installed (`docker-compose --version`)
5. Run `/ci-filebase docker init` to set up Docker CI

## File Structure

After initialization, the following structure is created:

```
.claude/ci-filebase/
├── issues/           # Individual issue JSON files
│   └── {id}.json
├── prs/              # Individual PR JSON files
│   └── {id}.json
├── labels.json       # Label definitions
├── counter.json      # ID counters
└── docker/           # Docker CI/CD (after docker init)
    ├── docker-compose.staging.yaml
    ├── Dockerfile.staging
    ├── .env.staging
    └── ci-config.json
```

## Workflow Phases

### Phase 1: Parse Command

1. **Parse command**: Identify operation (init, issue, pr, labels)
2. **Parse subcommand**: For issue/pr, identify CRUD operation
3. **Parse arguments**: Extract IDs, flags, and values

### Phase 2: Execute Operation

#### For `init`:
1. Create `.claude/ci-filebase/` directory structure
2. Create `labels.json` with default workflow labels
3. Create `counter.json` with zeroed counters
4. Report success

#### For `issue create`:
1. Increment issue counter
2. Create issue JSON file with:
   - ID, title, body
   - State: `open`
   - Labels from `--labels` flag
   - Timestamps
3. Return created issue ID

#### For `issue get`:
1. Read issue JSON file
2. Display formatted issue details

#### For `issue update`:
1. Read existing issue
2. Apply changes (title, body)
3. Update `updated_at` timestamp
4. Write back to file

#### For `issue label`:
1. Read existing issue
2. Replace labels array
3. Update `updated_at` timestamp
4. Write back to file

#### For `issue close`:
1. Read existing issue
2. Set state to `closed`
3. Set `closed_at` timestamp
4. Write back to file

#### For `issue list`:
1. Read all issue files from `issues/` directory
2. Apply filters (state, labels)
3. Display formatted list

#### For `issue comment`:
1. Read existing issue
2. Increment comment counter
3. Append comment to comments array
4. Write back to file

#### For `pr create`:
1. Increment PR counter
2. Create PR JSON file with:
   - ID, title, body
   - Head/base branches
   - Issue reference (optional)
   - State: `open`
3. Return created PR ID

#### For `pr merge`:
1. Read existing PR
2. Set state to `merged`
3. Set `merged_at` timestamp
4. Write back to file

#### For `pr review`:
1. Read existing PR
2. Append review to reviews array
3. Write back to file

### Phase 3: Report Result

- Display operation result
- Show created/updated resource details
- Report any errors

## Integration with /crunch

When CLAUDE.md CI Platform is set to `Filebase`, the `/crunch` skill uses these commands instead of MCP calls:

| Operation | MCP Command | Filebase Equivalent |
|-----------|-------------|---------------------|
| Create issue | `mcp__github__create_issue` | `/ci-filebase issue create` |
| Get issue | `mcp__github__get_issue` | `/ci-filebase issue get` |
| Update labels | `mcp__github__set_issue_labels` | `/ci-filebase issue label` |
| Update body | `mcp__github__update_issue` | `/ci-filebase issue update` |
| Close issue | `mcp__github__update_issue` | `/ci-filebase issue close` |
| Add comment | `mcp__github__create_issue_comment` | `/ci-filebase issue comment` |
| Create PR | `mcp__github__create_pull_request` | `/ci-filebase pr create` |
| Merge PR | `mcp__github__merge_pull_request` | `/ci-filebase pr merge` |

## Default Labels

The `init` command creates these workflow labels:

| Label | Description |
|-------|-------------|
| `type:bug` | Bug fix |
| `type:feature` | New feature |
| `state:backlog` | In backlog |
| `state:enrich` | Being enriched |
| `state:ready` | Ready for implementation |
| `state:implementing` | In progress |
| `state:validation` | Validating |
| `state:docs` | Documentation |
| `state:ready-to-merge` | Ready to merge |
| `state:done` | Completed |
| `domain:security` | Security-related |
| `domain:infra` | Infrastructure-related |
| `domain:frontend` | Frontend-related |
| `domain:api` | API-related |

## Error Handling

| Error | Action |
|-------|--------|
| Not initialized | Prompt user to run `/ci-filebase init` |
| Issue not found | Report error with issue ID |
| PR not found | Report error with PR ID |
| Invalid label | Warn but allow (labels are user-defined) |
| File permission error | Report error, suggest checking permissions |

## Example Usage

### Example 1: Initialize and create first issue

```
User: /ci-filebase init

Claude: Initialized ci-filebase at .claude/ci-filebase/
  - Created issues/ directory
  - Created prs/ directory
  - Created labels.json with 14 default labels
  - Created counter.json

User: /ci-filebase issue create "Add user authentication" --body "Implement login/logout flow" --labels type:feature,state:backlog

Claude: Created issue #1: "Add user authentication"
  State: open
  Labels: type:feature, state:backlog
```

### Example 2: State transition

```
User: /ci-filebase issue label 1 type:feature state:implementing

Claude: Updated issue #1 labels:
  - type:feature
  - state:implementing
```

### Example 3: List and filter issues

```
User: /ci-filebase issue list --state open --labels state:implementing

Claude: Open issues with state:implementing:
  #1: Add user authentication [type:feature, state:implementing]
  #3: Fix login timeout [type:bug, state:implementing]
```

### Example 4: Create PR for issue

```
User: /ci-filebase pr create --title "Feat #1: Add user authentication" --body "Implements login/logout" --head "feature/issue-1-auth" --base "main" --issue 1

Claude: Created PR #1: "Feat #1: Add user authentication"
  Head: feature/issue-1-auth
  Base: main
  Linked issue: #1
```

### Example 5: Complete workflow with /crunch

```
User: /crunch 1 -t ready

Claude:
1. Reads CLAUDE.md - CI Platform: Filebase
2. Uses /ci-filebase issue get 1 to fetch issue
3. Executes workflow transitions:
   - /ci-filebase issue label 1 type:feature state:enrich
   - (performs enrich activities)
   - /ci-filebase issue update 1 --body "..."
   - /ci-filebase issue label 1 type:feature state:ready
4. Reports: Issue #1 transitioned to READY
```

### Example 6: Initialize Docker CI/CD

```
User: /ci-filebase docker init

Claude: Initialized Docker CI at .claude/ci-filebase/docker/
  - Created docker-compose.staging.yaml
  - Created Dockerfile.staging
  - Created .env.staging
  - CI commands configured for Node.js project

  Detected project type: Node.js
  - Lint: npm run lint
  - Test: npm test
  - Build: npm run build

  Next steps:
  1. Review docker-compose.staging.yaml
  2. Run `/ci-filebase docker ci` to test pipeline
  3. Run `/ci-filebase docker deploy staging` to deploy
```

### Example 7: Run CI pipeline locally

```
User: /ci-filebase docker ci

Claude:
CI Pipeline - Local Simulation
==============================
[1/4] Lint........... PASS (8s)
[2/4] Test........... PASS (32s)
[3/4] Build.......... PASS (15s)
[4/4] Security....... SKIP (no scanner configured)

Pipeline: PASSED
Duration: 55s

Ready for staging deployment.
Run: /ci-filebase docker deploy staging
```

### Example 8: Deploy and validate on local Docker

```
User: /ci-filebase docker deploy staging

Claude:
Deploying to local staging...
  Building image: myapp:staging-abc1234
  Starting containers...
  Waiting for health check...

Deployment: SUCCESS
  App URL: http://localhost:8080
  Container: myapp-staging

User: /ci-filebase docker health

Claude:
Staging Health Check
====================
Container: running (up 2 minutes)
Health endpoint: 200 OK (23ms)
CPU: 0.5%
Memory: 128MB / 512MB

Recent errors: 0
```

### Example 9: VALIDATION phase with Docker

```
User: /crunch 1 -t docs
(assuming issue #1 is in IMPLEMENTING state)

Claude:
1. Transitions to VALIDATION phase
2. Runs CI pipeline: /ci-filebase docker ci
   → Pipeline PASSED
3. Deploys to staging: /ci-filebase docker deploy staging
   → Deployment SUCCESS
4. Executes DoD checklist:
   - App running: ✓
   - No error logs: ✓
   - Functional checks: ✓
5. Runs continuous monitoring (10 min, 5 cycles):
   - Cycle 1/5: HEALTHY
   - Cycle 2/5: HEALTHY
   - Cycle 3/5: HEALTHY
   - Cycle 4/5: HEALTHY
   - Cycle 5/5: HEALTHY
6. Validation PASSED
7. Stops staging: /ci-filebase docker stop
8. Transitions to DOCS phase
```
