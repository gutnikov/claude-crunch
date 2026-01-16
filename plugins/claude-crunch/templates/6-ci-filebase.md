## File-based CI Commands

File-based CI provides local issue and PR tracking without requiring external CI platforms (GitHub/GitLab/Gitea). All data is stored in `.claude/ci-filebase/` as YAML files for better readability.

> **When to use:** Projects without access to hosted CI platforms, local-only development, air-gapped environments, or rapid prototyping.

### Issue Management

#### Initialize Filebase

Set up the ci-filebase directory structure (run once per project):

```
/ci-filebase init
```

Creates:

- `.claude/ci-filebase/issues/` - Issue storage (YAML files)
- `.claude/ci-filebase/prs/` - PR storage (YAML files)
- `.claude/ci-filebase/labels.yaml` - Default labels
- `.claude/ci-filebase/counter.yaml` - ID counters

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

| Action       | Command                                    |
| ------------ | ------------------------------------------ |
| Initialize   | `/ci-filebase init`                        |
| Get labels   | `/ci-filebase labels`                      |
| Read issue   | `/ci-filebase issue get {id}`              |
| Create issue | `/ci-filebase issue create "{title}"`      |
| Update issue | `/ci-filebase issue update {id}`           |
| Close issue  | `/ci-filebase issue close {id}`            |
| Change state | `/ci-filebase issue label {id} {labels}`   |
| Add comment  | `/ci-filebase issue comment {id} "{text}"` |
| List issues  | `/ci-filebase issue list`                  |

### Pull Request Management

| Action    | Command                                |
| --------- | -------------------------------------- |
| Create PR | `/ci-filebase pr create --title "..."` |
| Get PR    | `/ci-filebase pr get {id}`             |
| List PRs  | `/ci-filebase pr list`                 |
| Merge PR  | `/ci-filebase pr merge {id}`           |
| Review PR | `/ci-filebase pr review {id}`          |

### File Storage

| Resource | Location                               |
| -------- | -------------------------------------- |
| Issues   | `.claude/ci-filebase/issues/{id}.yaml` |
| PRs      | `.claude/ci-filebase/prs/{id}.yaml`    |
| Labels   | `.claude/ci-filebase/labels.yaml`      |
| Counters | `.claude/ci-filebase/counter.yaml`     |

---

## Local Docker CI/CD

File-based CI includes local Docker simulation for CI pipelines and staging deployment when external CI platforms are unavailable.

### Initialize Docker CI

Set up local Docker staging environment:

```
/ci-filebase docker init
```

Creates:

- `.claude/ci-filebase/docker/docker-compose.staging.yaml` - Staging services
- `.claude/ci-filebase/docker/Dockerfile.staging` - App container (if needed)
- `.claude/ci-filebase/docker/.env.staging` - Environment variables

### Run CI Pipeline (Local Simulation)

Simulate CI pipeline locally before deployment:

```
/ci-filebase docker ci
```

Runs in sequence:

1. **Lint** - Code style checks
2. **Test** - Unit and integration tests
3. **Build** - Build application/container
4. **Security scan** - Basic vulnerability checks (optional)

**Example output:**

```
CI Pipeline - Local Simulation
==============================
[1/4] Lint........... PASS (12s)
[2/4] Test........... PASS (45s)
[3/4] Build.......... PASS (30s)
[4/4] Security....... SKIP (no scanner configured)

Pipeline: PASSED
Ready for staging deployment.
```

### Deploy to Local Staging

Deploy application to local Docker staging:

```
/ci-filebase docker deploy staging
```

Actions:

1. Build Docker image (if changed)
2. Start/update staging containers via docker-compose
3. Wait for health checks
4. Report deployment status

**Example:**

```
Deploying to local staging...
  Building image: myapp:staging-abc123
  Starting containers: docker-compose -f .claude/ci-filebase/docker/docker-compose.staging.yaml up -d
  Waiting for health check...

Deployment: SUCCESS
  App URL: http://localhost:8080
  Logs: docker-compose logs -f
```

### View Staging Logs

Monitor staging container logs:

```
/ci-filebase docker logs
```

Or with docker-compose directly:

```bash
docker-compose -f .claude/ci-filebase/docker/docker-compose.staging.yaml logs -f
```

### Staging Health Check

Check staging environment health:

```
/ci-filebase docker health
```

Checks:

- Container status (running/stopped)
- Health endpoint response
- Recent error logs
- Resource usage

### Stop Staging

Stop local staging environment:

```
/ci-filebase docker stop
```

### Docker Compose Template

Generated `docker-compose.staging.yaml`:

```yaml
version: "3.8"

services:
  app:
    build:
      context: ../../../..
      dockerfile: .claude/ci-filebase/docker/Dockerfile.staging
    image: ${APP_NAME:-myapp}:staging
    container_name: ${APP_NAME:-myapp}-staging
    ports:
      - "${STAGING_PORT:-8080}:${APP_PORT:-8080}"
    environment:
      - NODE_ENV=staging
      - LOG_LEVEL=debug
    env_file:
      - .env.staging
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:${APP_PORT:-8080}/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
    restart: unless-stopped
    volumes:
      - app-logs:/var/log/app

  # Optional: Add database, redis, etc. as needed
  # db:
  #   image: postgres:15
  #   environment:
  #     POSTGRES_DB: myapp_staging
  #     POSTGRES_USER: staging
  #     POSTGRES_PASSWORD: staging_password
  #   volumes:
  #     - db-data:/var/lib/postgresql/data

volumes:
  app-logs:
  # db-data:
```

### Quick Reference - Docker Commands

| Action             | Command                              |
| ------------------ | ------------------------------------ |
| Initialize Docker  | `/ci-filebase docker init`           |
| Run CI pipeline    | `/ci-filebase docker ci`             |
| Deploy to staging  | `/ci-filebase docker deploy staging` |
| View logs          | `/ci-filebase docker logs`           |
| Health check       | `/ci-filebase docker health`         |
| Stop staging       | `/ci-filebase docker stop`           |
| Rebuild containers | `/ci-filebase docker rebuild`        |

### Manual Docker Commands

For direct control, use docker-compose:

```bash
# Start staging
docker-compose -f .claude/ci-filebase/docker/docker-compose.staging.yaml up -d

# View logs
docker-compose -f .claude/ci-filebase/docker/docker-compose.staging.yaml logs -f

# Stop staging
docker-compose -f .claude/ci-filebase/docker/docker-compose.staging.yaml down

# Rebuild and restart
docker-compose -f .claude/ci-filebase/docker/docker-compose.staging.yaml up -d --build
```

---

## Comparison with MCP-based CI

| Feature          | GitHub/GitLab/Gitea MCP | File-based CI    |
| ---------------- | ----------------------- | ---------------- |
| Issue tracking   | Remote API              | Local YAML files |
| PR management    | Remote API              | Local YAML files |
| CI/CD pipelines  | Automated (cloud)       | Local Docker     |
| Staging deploy   | Remote infrastructure   | Local Docker     |
| Collaboration    | Multi-user              | Single-user      |
| Offline support  | Requires network        | Fully offline    |
| Setup complexity | MCP + tokens            | Docker only      |
