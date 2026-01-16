# File-based CI Workflow Implementation

Detailed instructions for implementing file-based CI operations. This skill provides local file storage as an alternative to MCP-based CI platforms.

---

## Directory Structure

```
.claude/ci-filebase/
├── issues/
│   ├── 1.yaml
│   ├── 2.yaml
│   └── ...
├── prs/
│   ├── 1.yaml
│   └── ...
├── labels.yaml
└── counter.yaml
```

---

## YAML Schemas

### Issue Schema (`issues/{id}.yaml`)

```yaml
id: 1
title: "Issue title"
body: |
  Issue description in markdown.

  ## Definition of Done (DOD)

  - [ ] DOD item 1
    - **Verification:** How to verify this item is complete
  - [ ] DOD item 2
    - **Verification:** How to verify this item is complete
state: open
labels:
  - type:feature
  - state:backlog
created_at: "2024-01-14T10:00:00Z"
updated_at: "2024-01-14T10:00:00Z"
closed_at: null
comments:
  - id: 1
    body: "Comment text"
    created_at: "2024-01-14T10:05:00Z"
```

### PR Schema (`prs/{id}.yaml`)

```yaml
id: 1
title: "PR title"
body: "PR description"
state: open
head_branch: feature/issue-1-auth
base_branch: main
issue_ref: 1
labels: []
created_at: "2024-01-14T10:00:00Z"
merged_at: null
reviews:
  - id: 1
    state: APPROVED
    body: "LGTM"
    created_at: "2024-01-14T11:00:00Z"
```

### Labels Schema (`labels.yaml`)

```yaml
labels:
  - name: type:bug
    color: d73a4a
    description: Bug fix
  - name: type:feature
    color: "0075ca"
    description: New feature
  - name: type:decomposition
    color: "7057ff"
    description: Spec/idea to decompose into tasks
  - name: state:backlog
    color: fbca04
    description: In backlog
  - name: state:enrich
    color: c2e0c6
    description: Being enriched
  - name: state:ready
    color: 0e8a16
    description: Ready for implementation
  - name: state:implementing
    color: 1d76db
    description: In progress
  - name: state:e2e
    color: "5319e7"
    description: E2E check
  - name: state:docs
    color: "006b75"
    description: Documentation
  - name: state:ready-to-merge
    color: 0e8a16
    description: Ready to merge
  - name: state:done
    color: ffffff
    description: Completed
  - name: domain:security
    color: b60205
    description: Security-related
  - name: domain:infra
    color: "5319e7"
    description: Infrastructure-related
  - name: domain:frontend
    color: 1d76db
    description: Frontend-related
  - name: domain:api
    color: 0e8a16
    description: API-related
```

### Counter Schema (`counter.yaml`)

```yaml
issue: 0
pr: 0
comment: 0
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

3. Create labels.yaml with default labels (see schema above)

4. Create counter.yaml:
   issue: 0
   pr: 0
   comment: 0

5. Report success:
   "Initialized ci-filebase at .claude/ci-filebase/
    - Created issues/ directory
    - Created prs/ directory
    - Created labels.yaml with {N} default labels
    - Created counter.yaml"
```

### `issue create` Command

**Syntax**: `/ci-filebase issue create "title" [--body "..."] [--labels l1,l2]`

**IMPORTANT:** Every issue MUST contain a Definition of Done (DOD) list in the body. Each DOD item MUST have a verification method explaining how to check if the item is complete.

```
1. Read counter.yaml
2. Increment issue counter
3. Create issue YAML file:
   id: {new_id}
   title: "{title}"
   body: |
     {body or ""}

     ## Definition of Done (DOD)

     - [ ] {DOD item 1}
       - **Verification:** {How to verify this item is complete}
     - [ ] {DOD item 2}
       - **Verification:** {How to verify this item is complete}
   state: open
   labels:
     - {label1}
     - {label2}
   created_at: "{ISO timestamp}"
   updated_at: "{ISO timestamp}"
   closed_at: null
   comments: []

4. Write to .claude/ci-filebase/issues/{id}.yaml
5. Update counter.yaml with new issue count
6. Report success:
   "Created issue #{id}: {title}
    State: open
    Labels: {labels or 'none'}"
```

### `issue get` Command

**Syntax**: `/ci-filebase issue get {id}`

```
1. Read .claude/ci-filebase/issues/{id}.yaml
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

2. Read counter.yaml
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
8. Update counter.yaml

9. Report success:
   "Added comment #{comment_id} to issue #{id}"
```

### `pr create` Command

**Syntax**: `/ci-filebase pr create --title "..." --head "branch" --base "branch" [--body "..."] [--issue N]`

```
1. Read counter.yaml
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

4. Write to .claude/ci-filebase/prs/{id}.yaml
5. Update counter.yaml

6. Report success:
   "Created PR #{id}: {title}
    Head: {head_branch}
    Base: {base_branch}
    Linked issue: #{issue_ref or 'none'}"
```

### `pr get` Command

**Syntax**: `/ci-filebase pr get {id}`

```
1. Read .claude/ci-filebase/prs/{id}.yaml
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

2. Read counter.yaml
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
7. Update counter.yaml

8. Report success:
   "Added {state} review to PR #{id}"
```

### `labels` Command

**Syntax**: `/ci-filebase labels`

```
1. Read .claude/ci-filebase/labels.yaml

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

| /crunch needs to... | MCP call (GitHub example)                                               | Filebase equivalent                                                     |
| ------------------- | ----------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Fetch issue         | `mcp__github__get_issue(owner, repo, issue_number)`                     | `/ci-filebase issue get {id}`                                           |
| Create issue        | `mcp__github__create_issue(owner, repo, title, body, labels)`           | `/ci-filebase issue create "{title}" --body "{body}" --labels {labels}` |
| Update issue body   | `mcp__github__update_issue(owner, repo, issue_number, body)`            | `/ci-filebase issue update {id} --body "{body}"`                        |
| Set labels          | `mcp__github__set_issue_labels(owner, repo, issue_number, labels)`      | `/ci-filebase issue label {id} {labels}`                                |
| Close issue         | `mcp__github__update_issue(owner, repo, issue_number, state: "closed")` | `/ci-filebase issue close {id}`                                         |
| Add comment         | `mcp__github__create_issue_comment(owner, repo, issue_number, body)`    | `/ci-filebase issue comment {id} "{body}"`                              |
| List labels         | `mcp__github__list_labels(owner, repo)`                                 | `/ci-filebase labels`                                                   |
| Create PR           | `mcp__github__create_pull_request(...)`                                 | `/ci-filebase pr create --title "..." --head "..." --base "..."`        |
| Merge PR            | `mcp__github__merge_pull_request(...)`                                  | `/ci-filebase pr merge {id}`                                            |

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

| Condition        | Error Message                                                 | Recovery               |
| ---------------- | ------------------------------------------------------------- | ---------------------- |
| Not initialized  | "ci-filebase not initialized. Run `/ci-filebase init` first." | Run init               |
| Issue not found  | "Issue #{id} not found"                                       | Check ID               |
| PR not found     | "PR #{id} not found"                                          | Check ID               |
| File read error  | "Error reading {path}: {error}"                               | Check permissions      |
| File write error | "Error writing {path}: {error}"                               | Check permissions      |
| Invalid JSON     | "Corrupted file at {path}"                                    | Manual fix or recreate |

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

---

## Local Docker CI/CD Commands

File-based CI includes local Docker simulation for CI pipelines and staging deployment.

### Directory Structure (with Docker)

```
.claude/ci-filebase/
├── issues/
├── prs/
├── labels.yaml
├── counter.yaml
└── docker/
    ├── docker-compose.staging.yaml
    ├── Dockerfile.staging
    ├── .env.staging
    └── ci-config.yaml
```

### CI Config Schema (`docker/ci-config.yaml`)

```yaml
app_name: myapp
app_port: 8080
staging_port: 8080
health_endpoint: /health
commands:
  lint: npm run lint
  test: npm test
  build: npm run build
  security: ""
dockerfile: Dockerfile.staging
compose_file: docker-compose.staging.yaml
```

### `docker init` Command

**Purpose**: Initialize local Docker CI/CD infrastructure.

```
1. Check if .claude/ci-filebase/docker/ exists
   IF exists:
     Show warning: "Docker CI already initialized"
     Ask user to confirm re-initialization
     IF not confirmed: abort

2. Create directory:
   mkdir -p .claude/ci-filebase/docker

3. Detect project type and configure commands:
   IF package.yaml exists:
     commands.lint = "npm run lint"
     commands.test = "npm test"
     commands.build = "npm run build"
   ELSE IF requirements.txt or pyproject.toml exists:
     commands.lint = "ruff check ." or "flake8"
     commands.test = "pytest"
     commands.build = "pip install -e ."
   ELSE IF go.mod exists:
     commands.lint = "golangci-lint run"
     commands.test = "go test ./..."
     commands.build = "go build -o app ."
   ELSE IF Cargo.toml exists:
     commands.lint = "cargo clippy"
     commands.test = "cargo test"
     commands.build = "cargo build --release"
   ELSE:
     Ask user for commands

4. Create ci-config.yaml with detected/provided commands

5. Generate docker-compose.staging.yaml:
   - Use template from 6-ci-filebase.md
   - Substitute app_name, ports from config

6. Generate Dockerfile.staging (if not exists at project root):
   - Basic Dockerfile based on project type
   - Or prompt user to provide existing Dockerfile path

7. Create .env.staging with default environment variables

8. Report success:
   "Initialized Docker CI at .claude/ci-filebase/docker/
    - Created docker-compose.staging.yaml
    - Created Dockerfile.staging
    - Created .env.staging
    - CI commands configured for {project_type}

    Next steps:
    1. Review and customize docker-compose.staging.yaml
    2. Run `/ci-filebase docker ci` to test pipeline
    3. Run `/ci-filebase docker deploy staging` to deploy"
```

### `docker ci` Command

**Purpose**: Run local CI pipeline simulation.

**Syntax**: `/ci-filebase docker ci [--skip-lint] [--skip-test] [--skip-build]`

```
1. Read ci-config.yaml
   IF not exists: error "Docker CI not initialized. Run `/ci-filebase docker init`"

2. Display header:
   "CI Pipeline - Local Simulation
    =============================="

3. Run pipeline stages:

   stage_results = {}

   # Stage 1: Lint
   IF not --skip-lint AND config.commands.lint:
     print "[1/4] Lint..........."
     result = run_command(config.commands.lint)
     stage_results.lint = result
     print result.success ? "PASS" : "FAIL"
     IF not result.success: show errors, abort

   # Stage 2: Test
   IF not --skip-test AND config.commands.test:
     print "[2/4] Test..........."
     result = run_command(config.commands.test)
     stage_results.test = result
     print result.success ? "PASS" : "FAIL"
     IF not result.success: show errors, abort

   # Stage 3: Build
   IF not --skip-build AND config.commands.build:
     print "[3/4] Build..........."
     result = run_command(config.commands.build)
     stage_results.build = result
     print result.success ? "PASS" : "FAIL"
     IF not result.success: show errors, abort

   # Stage 4: Security (optional)
   IF config.commands.security:
     print "[4/4] Security........"
     result = run_command(config.commands.security)
     stage_results.security = result
     print result.success ? "PASS" : "WARN"
   ELSE:
     print "[4/4] Security........ SKIP (no scanner configured)"

4. Report summary:
   "Pipeline: {PASSED|FAILED}
    Duration: {total_time}

    {IF PASSED}
    Ready for staging deployment.
    Run: /ci-filebase docker deploy staging
    {ENDIF}"
```

### `docker deploy` Command

**Purpose**: Deploy to local Docker staging environment.

**Syntax**: `/ci-filebase docker deploy staging [--rebuild]`

```
1. Read ci-config.yaml
   IF not exists: error "Docker CI not initialized"

2. Verify docker-compose file exists:
   compose_file = .claude/ci-filebase/docker/{config.compose_file}
   IF not exists: error "Compose file not found"

3. Build image (if --rebuild or image doesn't exist):
   print "Building image: {app_name}:staging-{git_short_sha}"

   result = docker build -t {app_name}:staging -f {dockerfile} .
   IF not result.success: show errors, abort

4. Deploy with docker-compose:
   print "Starting containers..."

   result = docker-compose -f {compose_file} up -d
   IF not result.success: show errors, abort

5. Wait for health check (max 60 seconds):
   print "Waiting for health check..."

   for attempt in 1..12:
     sleep 5s
     result = curl -s http://localhost:{staging_port}{health_endpoint}
     IF result.status == 200:
       break

   IF health check failed after 60s:
     show container logs
     error "Health check failed"

6. Report success:
   "Deployment: SUCCESS
    App URL: http://localhost:{staging_port}
    Container: {app_name}-staging

    Commands:
    - View logs: /ci-filebase docker logs
    - Health check: /ci-filebase docker health
    - Stop: /ci-filebase docker stop"
```

### `docker logs` Command

**Purpose**: View staging container logs.

**Syntax**: `/ci-filebase docker logs [--tail N] [--follow]`

```
1. Read ci-config.yaml

2. Run docker-compose logs:

   args = []
   IF --tail: args.append("-n {N}")
   IF --follow: args.append("-f")

   docker-compose -f {compose_file} logs {args}
```

### `docker health` Command

**Purpose**: Check staging environment health.

**Syntax**: `/ci-filebase docker health`

```
1. Read ci-config.yaml

2. Check container status:
   container_status = docker inspect {app_name}-staging --format '{{.State.Status}}'

3. Check health endpoint:
   health_response = curl -s -w "%{http_code}" http://localhost:{staging_port}{health_endpoint}

4. Get recent error logs:
   error_logs = docker-compose logs --tail=50 2>&1 | grep -i "error\|exception\|fatal"

5. Get resource usage:
   stats = docker stats {app_name}-staging --no-stream --format "{{.CPUPerc}},{{.MemUsage}}"

6. Report:
   "Staging Health Check
    ====================
    Container: {status} ({running_time})
    Health endpoint: {http_code} {response_time}ms
    CPU: {cpu_percent}
    Memory: {mem_usage}

    Recent errors: {error_count}
    {IF error_count > 0}
    ---
    {error_log_sample}
    {ENDIF}"
```

### `docker stop` Command

**Purpose**: Stop local staging environment.

**Syntax**: `/ci-filebase docker stop`

```
1. Read ci-config.yaml

2. Stop containers:
   docker-compose -f {compose_file} down

3. Report:
   "Staging environment stopped.

    To restart: /ci-filebase docker deploy staging"
```

### `docker rebuild` Command

**Purpose**: Rebuild and restart staging containers.

**Syntax**: `/ci-filebase docker rebuild`

```
1. Read ci-config.yaml

2. Rebuild and restart:
   docker-compose -f {compose_file} up -d --build

3. Wait for health check (same as deploy)

4. Report status
```

---

## Integration with /crunch Workflow

### E2E Phase with Local Docker

When CI platform is Filebase, the E2E phase uses local Docker for staging validation.

```
E2E Phase (Filebase + Docker):

1. Deploy to local staging:
   /ci-filebase docker deploy staging

2. Execute DoD checklist:
   - App running: /ci-filebase docker health
   - Error logs: /ci-filebase docker logs --tail 100 | check for errors
   - Functional checks: Manual or via curl/test commands
   - Regression checks: Manual verification

3. Continuous monitoring (simplified for local Docker):
   duration: 10m (shorter than remote staging)
   interval: 2m

   For each cycle:
     - Check health endpoint
     - Check docker logs for new errors
     - Check container stability (no restarts)

   IF anomaly detected:
     - Show logs
     - Return to IMPLEMENTING (no auto-remediation for local)

4. On success:
   - Mark DoD items as checked
   - Proceed to DOCS phase
```

### Staging Remediation (Local Docker)

For local Docker staging, remediation is simpler:

| Action              | When             | Command                                                          |
| ------------------- | ---------------- | ---------------------------------------------------------------- |
| `restart_container` | Container issues | `/ci-filebase docker rebuild`                                    |
| `view_logs`         | Any issue        | `/ci-filebase docker logs --tail 100`                            |
| `stop_start`        | Stuck container  | `/ci-filebase docker stop && /ci-filebase docker deploy staging` |

### Operation Mapping (with Docker)

| /crunch needs to...  | MCP (GitHub)      | Filebase | Filebase + Docker                    |
| -------------------- | ----------------- | -------- | ------------------------------------ |
| Run CI pipeline      | Automatic         | Manual   | `/ci-filebase docker ci`             |
| Deploy to staging    | Remote deploy     | Manual   | `/ci-filebase docker deploy staging` |
| Check staging health | Observability MCP | Manual   | `/ci-filebase docker health`         |
| View staging logs    | Loki MCP          | Manual   | `/ci-filebase docker logs`           |
| Restart staging      | kubectl/API       | Manual   | `/ci-filebase docker rebuild`        |

---

## Validation Output Format (Local Docker)

**Success case:**

```
Continuous Monitoring - LOCAL DOCKER STAGING
============================================
Environment: local-docker
Issue: #5
Duration: 10 minutes (5 cycles)
Started: 2024-01-14T10:00:00Z

Deployment check:
  ✓ Container running: myapp-staging (up 2 minutes)
  ✓ Health endpoint: 200 OK (45ms)

Cycle 1/5 [10:02:00]:
  ✓ Health: 200 OK
  ✓ Logs: No new errors
  ✓ Container: Stable (no restarts)
  Status: HEALTHY

Cycle 2/5 [10:04:00]:
  ✓ Health: 200 OK
  ✓ Logs: No new errors
  ✓ Container: Stable
  Status: HEALTHY

... (cycles 3-5 similar)

============================================
Monitoring Complete
Duration: 10 minutes
Result: PASSED

Staging validated on local Docker.
Proceeding to DOCS phase.
```

**Failure case:**

```
Continuous Monitoring - LOCAL DOCKER STAGING
============================================
Environment: local-docker
Issue: #5
Duration: 10 minutes (5 cycles)
Started: 2024-01-14T10:00:00Z

Cycle 1/5 [10:02:00]:
  ✓ Health: 200 OK
  ✓ Logs: No errors
  Status: HEALTHY

Cycle 2/5 [10:04:00]:
  ✗ Health: 500 Internal Server Error
  ✗ Logs: NullPointerException in UserService.getProfile()
  Status: ANOMALY DETECTED

Container logs (last 20 lines):
---
2024-01-14T10:03:55 ERROR UserService - NullPointerException
    at com.example.UserService.getProfile(UserService.java:45)
    at com.example.ApiController.user(ApiController.java:23)
---

============================================
Monitoring Stopped Early
Duration: 4 minutes
Result: FAILED - Bug detected

Action Required:
  1. Review logs above
  2. Fix NullPointerException in UserService
  3. Re-deploy: /ci-filebase docker deploy staging
  4. Re-run validation

Returning to IMPLEMENTING phase.
```
