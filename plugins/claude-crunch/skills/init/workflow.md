# Init Workflow Implementation

Detailed implementation instructions for the init skill.

## Plugin Directory Structure

This skill is part of the `claude-crunch` plugin. At runtime, locate resources relative to the plugin's installation directory:

```
claude-crunch/                  # Plugin root (find via skill's own path)
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── init/
│       ├── SKILL.md
│       └── workflow.md      # This file
├── templates/               # Template files for CLAUDE.md generation
│   ├── 1-project.md
│   ├── 2-agents.md
│   └── ...
└── agents/
```

**To find templates**: Navigate up from this skill's location (`skills/init/`) to the plugin root, then into `templates/`.

**Progress file**: Stored in the TARGET PROJECT's `.claude/init-progress.txt` (not in the plugin directory).

## Startup Behavior

### 1. Enter Planning Mode

Before any actions, enter planning mode to think through the process carefully.

### 2. Check for Resume

```
IF .claude/init-progress.txt exists:
  Parse YAML content
  Show: "Found saved progress from {last_updated}"
  Show: "You were on Phase {current_phase}: {phase_name}"

  AskUserQuestion: "How would you like to proceed?"
    Options:
    - Resume from Phase {N}
    - Restart from beginning

  IF restart:
    Delete init-progress.txt
    Create new progress file
    Start Phase 1
  ELSE:
    Jump to current_phase

ELSE:
  Create new init-progress.txt with:
    version: 1
    started_at: {now}
    current_phase: 1
    phases: (all pending)
  Start Phase 1
```

### 3. Progress File Format

Location: `.claude/init-progress.txt`

```yaml
version: 1
started_at: "2024-01-15T10:30:00Z"
last_updated: "2024-01-15T10:45:00Z"
current_phase: 3
current_step: "environment_variables"

phases:
  1_project_info:
    status: completed # pending | in_progress | completed | skipped
    completed_at: "2024-01-15T10:32:00Z"
    data:
      project_name: "my-awesome-project"
      project_description: "A distributed system for processing events"

  2_ci_platform:
    status: completed
    data:
      platform: "github" # github | gitlab | gitea
      owner: "myorg"
      repo: "my-awesome-project"
      lint_tools: ["prettier", "eslint"]
      typecheck_tools: ["tsc"]
      coverage_threshold: 80

  3_configuration:
    status: in_progress
    data:
      environment_variables:
        - name: DATABASE_URL
          description: PostgreSQL connection string
          secret: true
          source: vault
      feature_flags:
        - name: new_ui
          default: false

  4_deployment:
    status: pending
    data:
      staging:
        url: null
        api_url: null
        logs_url: null
        monitoring_url: null
        infrastructure: null
      production:
        url: null
        api_url: null
        logs_url: null
        monitoring_url: null
        status_page_url: null
        infrastructure: null

  5_generate_claude_md:
    status: pending

  6_secret_setup:
    status: pending
    data:
      approach: null # "vault" | "sops" | null
      vault_installed: false
      vault_tested: false
      sops_installed: false
      age_installed: false
      age_public_key: null

  7_ci_mcp:
    status: pending
    data:
      installed: false
      configured: false
      tested: false
      # Filebase-specific (when platform == "filebase")
      filebase_initialized: false
      docker_ci_enabled: false
      docker_ci_initialized: false
      docker_ci_tested: false

  8_observability_mcp:
    status: pending
    data:
      prometheus:
        url: null
        installed: false
        tested: false
      loki:
        url: null
        installed: false
        tested: false
      alertmanager:
        url: null
        installed: false
        tested: false
      skipped: false

  9_secrets:
    status: pending
    data:
      secrets_configured: []
      secrets_pending: []

  10_setup_ci_issue:
    status: pending
    data:
      issue_number: null
      issue_url: null
      final_state: null

  11_verify_checklist:
    status: pending
    data:
      secret_setup_working: false # Vault MCP or SOPS+age verified
      secrets_configured: false
      ci_mcp_working: false
      ci_config_ready: false
      observability_configured: false
      prometheus_working: false
      loki_working: false
      alertmanager_working: false
      setup_issue_created: false
      setup_issue_crunched: false

errors: []
```

---

## Phase 1: Project Info

**Purpose**: Collect basic project identification

### Questions

1. **Project Name**

   ```
   AskUserQuestion: "What is the name of your project?"

   Header: "Project name"
   Options:
   - (User types custom answer)

   Validation: lowercase, alphanumeric, hyphens allowed
   Example: tmux-claude-bot, payment-service, frontend-dashboard
   ```

2. **Project Description**

   ```
   AskUserQuestion: "Describe your project in 1-2 sentences:"

   Header: "Description"
   Options:
   - (User types custom answer)

   Validation: non-empty, 10-500 characters
   Example: A distributed Claude agent orchestration system that manages
   multiple Claude instances across physical machines.
   ```

### Actions

1. Store in progress file:

   ```yaml
   phases:
     1_project_info:
       status: completed
       completed_at: { now }
       data:
         project_name: { answer1 }
         project_description: { answer2 }
   ```

2. Update progress:

   ```yaml
   current_phase: 2
   last_updated: { now }
   ```

3. Proceed to Phase 2

---

## Phase 2: CI Platform

**Purpose**: Configure CI/CD integration

### Questions

1. **CI Platform**

   ```
   AskUserQuestion: "Which CI platform do you use?"

   Header: "CI platform"
   Options:
   - GitHub (GitHub Actions)
   - GitLab (GitLab CI)
   - Gitea (Gitea Actions)
   - Filebase (Local files - no external CI)
   ```

2. **Repository Owner** (skip for Filebase)

   ```
   IF platform != "filebase":
     AskUserQuestion: "What is your repository owner/organization?"

     Header: "Repo owner"
     Example: myorg, username, company-name
   ELSE:
     owner = null (not needed for filebase)
   ```

3. **Repository Name** (skip for Filebase)

   ```
   IF platform != "filebase":
     AskUserQuestion: "What is your repository name?"

     Header: "Repo name"
     Example: my-project, backend-api
   ELSE:
     repo = null (not needed for filebase)
   ```

4. **Linting Tools** (multi-select)

   ```
   AskUserQuestion: "What linting tools do you use?"

   Header: "Lint tools"
   Options (based on common tools):
   - prettier (JavaScript/TypeScript formatting)
   - eslint (JavaScript/TypeScript linting)
   - black (Python formatting)
   - ruff (Python linting)
   - golangci-lint (Go linting)
   - Other (specify)

   multiSelect: true
   ```

5. **Coverage Threshold**

   ```
   AskUserQuestion: "What is your minimum code coverage threshold?"

   Header: "Coverage %"
   Options:
   - 80% (Recommended)
   - 70%
   - 90%
   - Other (specify)
   ```

### Actions

1. Store in progress file:

   ```yaml
   phases:
     2_ci_platform:
       status: completed
       completed_at: { now }
       data:
         platform: { answer1 }
         owner: { answer2 }
         repo: { answer3 }
         lint_tools: { answer4 }
         coverage_threshold: { answer5 }
   ```

2. Update progress:

   ```yaml
   current_phase: 3
   last_updated: { now }
   ```

3. Proceed to Phase 3

---

## Phase 3: Configuration

**Purpose**: Define environment variables and feature flags

### Questions

1. **Environment Variables**

   ```
   AskUserQuestion: "List your environment variables"

   Header: "Env vars"

   Instruction: "Enter one per line in format: NAME - description - secret:yes/no"

   Example:
   DATABASE_URL - PostgreSQL connection string - secret:yes
   REDIS_URL - Redis cache connection - secret:yes
   LOG_LEVEL - Logging verbosity (debug/info/warn/error) - secret:no
   API_BASE_URL - External API endpoint - secret:no

   Options:
   - (User types multi-line answer)
   - Skip (no environment variables)
   ```

2. **Feature Flags**

   ```
   AskUserQuestion: "List your feature flags"

   Header: "Feature flags"

   Instruction: "Enter one per line in format: NAME - default:true/false"

   Example:
   new_ui - default:false
   beta_api - default:false
   debug_mode - default:false

   Options:
   - (User types multi-line answer)
   - Skip (no feature flags)
   ```

3. **Secret Management Approach** (if any secrets defined)

   ```
   IF environment_variables contains any secret:yes entries:
     AskUserQuestion: "How do you want to manage secrets?"

     Header: "Secrets"
     Options:
     - Vault (HashiCorp Vault with MCP integration)
     - SOPS (File-based encryption with age)
   ELSE:
     secret_approach = null (no secrets needed)
   ```

### Parsing

Parse structured data from responses:

```
FOR each line in env_vars:
  Split by " - "
  Extract: name, description, secret (yes/no)
  Add to environment_variables list

FOR each line in feature_flags:
  Split by " - "
  Extract: name, default (true/false)
  Add to feature_flags list
```

### Actions

1. Store in progress file:

   ```yaml
   phases:
     3_configuration:
       status: completed
       completed_at: {now}
       data:
         environment_variables: [parsed list]
         feature_flags: [parsed list]
         secret_approach: "vault" | "sops" | null
   ```

2. Identify secrets for Phase 9:

   ```
   secrets_needed = environment_variables.filter(v => v.secret == true)
   ```

3. Update progress:

   ```yaml
   current_phase: 4
   last_updated: { now }
   ```

4. Proceed to Phase 4

---

## Phase 4: Deployment

**Purpose**: Configure staging and production environments

### Questions for Staging

1. **Staging URL**

   ```
   AskUserQuestion: "What is your staging application URL?"

   Header: "Staging URL"
   Example: https://staging.example.com
   ```

2. **Staging API URL**

   ```
   AskUserQuestion: "What is your staging API URL?"

   Header: "Staging API"
   Example: https://api.staging.example.com
   ```

3. **Staging Logs URL**

   ```
   AskUserQuestion: "Where can staging logs be accessed?"

   Header: "Staging logs"
   Example: https://grafana.example.com/d/staging
   ```

4. **Staging Monitoring URL**

   ```
   AskUserQuestion: "Where is your staging monitoring dashboard?"

   Header: "Monitoring"
   Example: https://prometheus.example.com/staging
   ```

5. **Staging Infrastructure**

   ```
   AskUserQuestion: "What is your staging infrastructure type?"

   Header: "Infra type"
   Options:
   - Docker (docker-compose or standalone)
   - Kubernetes (K8s cluster)
   - Bare Metal (direct server deployment)
   ```

### Questions for Production

Same questions as staging, plus:

6. **Status Page URL** (optional)

   ```
   AskUserQuestion: "What is your status page URL? (optional)"

   Header: "Status page"
   Example: https://status.example.com
   Options:
   - (User types answer)
   - Skip (no status page)
   ```

### Actions

1. Store in progress file:

   ```yaml
   phases:
     4_deployment:
       status: completed
       completed_at: { now }
       data:
         staging:
           url: { staging_url }
           api_url: { staging_api }
           logs_url: { staging_logs }
           monitoring_url: { staging_monitoring }
           infrastructure: { staging_infra }
         production:
           url: { prod_url }
           api_url: { prod_api }
           logs_url: { prod_logs }
           monitoring_url: { prod_monitoring }
           status_page_url: { status_page }
           infrastructure: { prod_infra }
   ```

2. Update progress:

   ```yaml
   current_phase: 5
   last_updated: { now }
   ```

3. Proceed to Phase 5

---

## Phase 5: Generate CLAUDE.md

**Purpose**: Combine templates with collected data to create CLAUDE.md

### Template Order

Read and combine templates in this order:

1. `1-project.md`
2. `2-agents.md`
3. `3-tdd.md`
4. `4-observability.md`
5. `5-workflow.md`
6. `6-ci.md`
7. `6-ci-{platform}.md` (based on Phase 2 selection)
8. `7-configuration.md`
9. `8-secret-management.md`
10. `8-secrets-{approach}.md` (based on Phase 3 secret_approach selection)
11. `9-deployment.md`
12. `99-setup-checklist.md`

### Placeholder Replacement

Build replacement map from progress data:

```
{PROJECT_NAME} -> phases.1_project_info.data.project_name
{PROJECT_DESCRIPTION} -> phases.1_project_info.data.project_description
{org} -> phases.2_ci_platform.data.owner
{owner} -> phases.2_ci_platform.data.owner
{repo} -> phases.2_ci_platform.data.repo
{80} -> phases.2_ci_platform.data.coverage_threshold
```

### Template Comment Processing

For content between `<!--` and `-->` containing `TEMPLATE:`:

1. Remove the comment markers (`<!--` and `-->`)
2. Remove the `TEMPLATE:` instruction line
3. Keep the actual content
4. Replace placeholders with actual values

For `<!-- Example: ... -->` comments:

1. Remove entirely (these are just examples for the template user)

### CI Platform Selection (6-ci.md)

In the CI Platform table:

1. Keep only the row matching selected platform
2. Replace `{active}` with `active`
3. Remove the other platform rows
4. Remove the `<!--TEMPLATE:` comment block

Example for GitHub:

```markdown
| Platform | Status | Config File          | MCP Server                                                       |
| -------- | ------ | -------------------- | ---------------------------------------------------------------- |
| GitHub   | active | `.github/workflows/` | [github-mcp-server](https://github.com/github/github-mcp-server) |
```

### Environment Variables (7-configuration.md)

Replace the TEMPLATE section with actual data:

```markdown
### Environment Variables

| Variable     | Description                  | Secret | Source  |
| ------------ | ---------------------------- | ------ | ------- |
| DATABASE_URL | PostgreSQL connection string | Yes    | Vault   |
| REDIS_URL    | Redis cache connection       | Yes    | Vault   |
| LOG_LEVEL    | Logging verbosity            | No     | Ansible |
```

### Feature Flags (7-configuration.md)

Replace the TEMPLATE section with actual data:

````markdown
### Feature Flags

```yaml
defaults:
  new_ui: false
  beta_api: false
  debug_mode: false
```
````

```

### Deployment (9-deployment.md)

Replace TEMPLATE sections with actual staging and production data.

### Process

1. Read each template file
2. Apply placeholder replacements
3. Process TEMPLATE comments
4. Concatenate all sections with proper spacing
5. Write to `CLAUDE.md` in project root

### User Confirmation

Show preview to user:
```

"Here's your generated CLAUDE.md (first 50 lines):"

[Preview content]

AskUserQuestion: "Does this look correct?"
Options:

- Yes, continue (Recommended)
- Show full file
- Regenerate with changes

````

### Actions

1. Write CLAUDE.md to project root
2. Update progress:
   ```yaml
   phases:
     5_generate_claude_md:
       status: completed
       completed_at: {now}
   current_phase: 6
   last_updated: {now}
````

3. Proceed to Phase 6

---

## Phase 6: Secret Management Setup

**Purpose**: Install and configure secret management tools (Vault MCP or SOPS+age)

### Determine Approach

Based on Phase 3 secret_approach selection:

- `vault` → Install Vault MCP server
- `sops` → Set up SOPS with age encryption
- `null` → Skip (no secrets defined)

### Skip if No Secrets

```
IF phases.3_configuration.data.secret_approach == null:
  Show: "No secrets defined. Skipping secret management setup."
  Mark phase as skipped
  Skip to Phase 7
```

---

### Vault Approach

#### Check Existing Vault MCP

```
IF secret_approach == "vault":
  Try executing a vault MCP command (e.g., mcp__vault__status)

  IF success:
    Show: "Vault MCP is already installed and working!"
    Mark phase as completed
    Skip to Phase 7
```

#### Guide Vault Installation

If Vault MCP not installed, show instructions:

````markdown
To set up Vault MCP, you'll need to:

1. **Install the vault-mcp server**
   ```bash
   npm install -g @anthropic/vault-mcp
   ```
````

2. **Add to your Claude MCP configuration**

   Edit your Claude settings (usually `~/.config/claude/settings.json`):

   ```json
   {
     "mcpServers": {
       "vault": {
         "command": "vault-mcp",
         "args": ["--addr", "https://vault.example.com"]
       }
     }
   }
   ```

3. **Restart Claude Code** to load the MCP server.

```

#### Vault User Confirmation

```

AskUserQuestion: "Have you completed the Vault MCP installation?"

Options:

- Yes, I've installed and configured it
- Skip (I'll manage secrets manually)
- Help (show more details)

```

---

### SOPS Approach

#### Check Existing SOPS Setup

```

IF secret_approach == "sops":
Check if sops and age are installed:
Run: sops --version
Run: age --version

IF both installed:
Show: "SOPS and age are already installed!"

Check if .sops.yaml exists:
IF exists: Show: "SOPS configuration found."

````

#### Guide SOPS Installation

If SOPS/age not installed, show instructions:

```markdown
To set up SOPS with age encryption:

1. **Install SOPS and age**
   ```bash
   # macOS
   brew install sops age

   # Linux - download from releases:
   # https://github.com/getsops/sops/releases
   # https://github.com/FiloSottile/age/releases
````

2. **Generate age key pair**

   ```bash
   mkdir -p ~/.config/sops/age
   age-keygen -o ~/.config/sops/age/keys.txt
   ```

   Note your public key (starts with `age1...`)

3. **Create .sops.yaml in project root**

   ```yaml
   creation_rules:
     - path_regex: secrets/staging\.yaml$
       age: age1your_staging_public_key...
     - path_regex: secrets/production\.yaml$
       age: age1your_production_public_key...
   ```

4. **Create secrets directory**
   ```bash
   mkdir -p secrets
   ```

```

#### SOPS User Confirmation

```

AskUserQuestion: "Have you completed the SOPS setup?"

Options:

- Yes, SOPS and age are installed
- Skip (I'll set up SOPS later)
- Help (show more details)

```

#### SOPS Configuration

If user confirms, collect age public key:

```

AskUserQuestion: "Enter your age public key (starts with age1...)"

Header: "age key"

Store in progress for .sops.yaml generation

```

---

### Verification

For Vault:
```

Try vault MCP command again
IF success: Mark as verified
IF fail: Show error, offer retry/skip

```

For SOPS:
```

Check sops --version and age --version
IF success: Mark as verified
IF fail: Show error, offer retry/skip

````

### Actions

1. Update progress:
   ```yaml
   phases:
     6_secret_setup:
       status: completed  # or skipped
       data:
         approach: "vault" | "sops" | null
         vault_installed: true/false  # if vault
         vault_tested: true/false     # if vault
         sops_installed: true/false   # if sops
         age_installed: true/false    # if sops
         age_public_key: "age1..."    # if sops
   current_phase: 7
   last_updated: {now}
````

2. **IMPORTANT**: If Vault MCP was just installed, inform user:

   ```
   "MCP configuration changed. Please restart Claude Code and run /init again to continue."
   ```

   Save progress and exit gracefully.

3. Proceed to Phase 7

---

## Phase 7: CI MCP Setup

**Purpose**: Install and configure CI platform MCP server (or initialize Filebase)

### Determine MCP Server

Based on Phase 2 platform selection:

- GitHub -> `github-mcp-server`
- GitLab -> GitLab Duo MCP (embedded)
- Gitea -> `gitea-mcp`
- Filebase -> No MCP needed, use `/ci-filebase init`

### Handle Filebase Platform

```
IF platform == "filebase":
  Show: "Filebase uses local file storage instead of MCP servers."

  Run /ci-filebase init to create directory structure:
  - Creates .claude/ci-filebase/issues/
  - Creates .claude/ci-filebase/prs/
  - Creates .claude/ci-filebase/labels.yaml
  - Creates .claude/ci-filebase/counter.yaml

  Show: "Filebase initialized at .claude/ci-filebase/"
  Mark filebase_initialized = true

  # Ask about Docker CI setup
  Proceed to "Docker CI Setup for Filebase" section below
```

### Docker CI Setup for Filebase

When using Filebase, offer to set up local Docker CI for E2E phase support.

```
AskUserQuestion: "Do you want to set up local Docker CI/CD?"

Header: "Docker CI"
Options:
- Yes, set up Docker CI (Recommended for E2E support)
- No, I'll validate manually
```

#### If User Selects Docker CI

```
IF user selects "Yes":
  # Check Docker prerequisites
  Show: "Checking Docker installation..."

  Run: docker --version
  IF fails:
    Show: "Docker not found. Please install Docker first."
    Show link: https://docs.docker.com/get-docker/
    AskUserQuestion: "Docker status?"
    Options:
    - I've installed Docker, continue
    - Skip Docker CI for now

  Run: docker-compose --version OR docker compose version
  IF fails:
    Show: "Docker Compose not found. Please install Docker Compose."
    AskUserQuestion: "Docker Compose status?"
    Options:
    - I've installed Docker Compose, continue
    - Skip Docker CI for now

  # Initialize Docker CI
  Run /ci-filebase docker init:
  - Detects project type (Node.js, Python, Go, Rust, etc.)
  - Creates .claude/ci-filebase/docker/docker-compose.staging.yaml
  - Creates .claude/ci-filebase/docker/Dockerfile.staging (if needed)
  - Creates .claude/ci-filebase/docker/.env.staging
  - Creates .claude/ci-filebase/docker/ci-config.yaml

  Show:
    "Docker CI initialized!

    Detected project type: {project_type}
    CI commands configured:
    - Lint: {lint_command}
    - Test: {test_command}
    - Build: {build_command}

    Files created:
    - .claude/ci-filebase/docker/docker-compose.staging.yaml
    - .claude/ci-filebase/docker/ci-config.yaml"

  # Test CI pipeline
  AskUserQuestion: "Would you like to test the CI pipeline now?"
  Options:
  - Yes, run test pipeline (Recommended)
  - Skip test

  IF user selects "Yes":
    Run /ci-filebase docker ci
    IF success:
      Show: "CI pipeline test passed!"
      Mark docker_ci_tested = true
    ELSE:
      Show: "CI pipeline failed. You may need to adjust ci-config.yaml."
      Show error output
      Mark docker_ci_tested = false

  # Test staging deployment (optional)
  AskUserQuestion: "Would you like to test staging deployment?"
  Options:
  - Yes, deploy to local staging
  - Skip (I'll test later)

  IF user selects "Yes":
    Run /ci-filebase docker deploy staging
    IF success:
      Show:
        "Staging deployment successful!
         App URL: http://localhost:{port}

         Cleaning up..."
      Run /ci-filebase docker stop
      Show: "Staging stopped. Docker CI setup complete!"
    ELSE:
      Show: "Staging deployment failed. Check docker-compose configuration."
      Show error output

  Mark docker_ci_enabled = true
  Mark docker_ci_initialized = true
```

#### If User Skips Docker CI

```
IF user selects "No" or skips Docker setup:
  Show:
    "Docker CI skipped.

    Note: Without Docker CI, the E2E phase will require manual testing.
    You can set up Docker CI later by running:
      /ci-filebase docker init

    Proceeding with basic Filebase setup."

  Mark docker_ci_enabled = false
  Mark phase as completed
  Skip to Phase 8
```

#### Docker CI Summary

```
Show:
  "Filebase CI Setup Summary:
   - Issue tracking: .claude/ci-filebase/ ✓
   - Docker CI: {Enabled / Disabled}
   {IF docker_ci_enabled}
   - CI pipeline: {Tested / Not tested}
   - Staging: Local Docker
   {ELSE}
   - E2E: Manual testing required
   {ENDIF}"

Mark phase as completed
Skip to Phase 8
```

### Check Existing (for MCP-based platforms)

```
IF platform != "filebase":
  Try appropriate get_repo command:
  - GitHub: mcp__github__get_repo with owner and repo
  - GitLab: mcp__gitlab__get_project
  - Gitea: mcp__gitea__get_repo

  IF success:
    Show: "{Platform} MCP is already installed and working!"
    Mark phase as completed
    Skip to Phase 8
```

### Guide Installation

Show platform-specific instructions:

#### GitHub

````markdown
To set up GitHub MCP:

1. **Install github-mcp-server**
   ```bash
   npm install -g @github/mcp-server
   ```
````

2. **Create a Personal Access Token**

   Go to: https://github.com/settings/tokens

   Required scopes:
   - `repo` (full repository access)
   - `workflow` (GitHub Actions access)

3. **Add to Claude MCP configuration**

   ```json
   {
     "mcpServers": {
       "github": {
         "command": "github-mcp-server",
         "env": {
           "GITHUB_TOKEN": "ghp_your_token_here"
         }
       }
     }
   }
   ```

4. **Restart Claude Code**

````

#### GitLab

```markdown
GitLab uses Duo MCP which is embedded in GitLab.

1. **Enable GitLab Duo in your instance**

   See: https://docs.gitlab.com/user/gitlab_duo/model_context_protocol/

2. **No separate installation needed**

   GitLab Duo MCP is automatically available when using Claude with GitLab.
````

#### Gitea

````markdown
To set up Gitea MCP:

1. **Install gitea-mcp**
   ```bash
   npm install -g gitea-mcp
   ```
````

2. **Create an Access Token**

   Go to: {your_gitea_url}/user/settings/applications

   Create a token with appropriate scopes.

3. **Add to Claude MCP configuration**

   ```json
   {
     "mcpServers": {
       "gitea": {
         "command": "gitea-mcp",
         "args": ["--url", "https://gitea.example.com"],
         "env": {
           "GITEA_TOKEN": "your_token_here"
         }
       }
     }
   }
   ```

4. **Restart Claude Code**

```

### User Confirmation

```

AskUserQuestion: "Have you completed the {Platform} MCP installation?"

Options:

- Yes, I've installed and configured it
- Skip (I'll set it up later)
- Help (show more details)

````

### Verification

Same as Phase 6 - verify, retry, or skip.

### Actions

1. Update progress:
   ```yaml
   phases:
     7_ci_mcp:
       status: completed  # or skipped
       data:
         installed: true/false
         configured: true/false
         tested: true/false
   current_phase: 8
   last_updated: {now}
````

2. **IMPORTANT**: If MCP was just installed, inform user to restart Claude.

3. Proceed to Phase 8

---

## Phase 8: Observability MCP Setup

**Purpose**: Configure MCP servers for Prometheus, Loki, and Alertmanager to enable `/patrol` and E2E continuous monitoring.

### Determine Need

```
AskUserQuestion: "Do you have observability infrastructure (Prometheus, Loki, Alertmanager)?"

Header: "Observability"
Options:
- Yes, I have existing infrastructure
- No, I'll set it up later
- Skip (not needed for this project)
```

IF "Skip" or "No, I'll set it up later":

```
Show: "Observability MCPs skipped. The following features will be limited:"
  - /patrol --logs: Unavailable
  - /patrol --metrics: Unavailable
  - E2E continuous monitoring: Unavailable
  - /patrol --code and /patrol --deps: Still work

Mark phase as skipped
Skip to Phase 9
```

### Collect Infrastructure URLs

IF user has existing infrastructure:

1. **Prometheus URL**

   ```
   AskUserQuestion: "What is your Prometheus server URL?"

   Header: "Prometheus"
   Example: http://prometheus.your-infra.local:9090
   ```

2. **Loki URL**

   ```
   AskUserQuestion: "What is your Loki server URL?"

   Header: "Loki"
   Example: http://loki.your-infra.local:3100
   ```

3. **Alertmanager URL**

   ```
   AskUserQuestion: "What is your Alertmanager server URL?"

   Header: "Alertmanager"
   Example: http://alertmanager.your-infra.local:9093
   ```

### Guide MCP Installation

For each observability tool, guide installation:

#### Prometheus MCP

````markdown
To set up Prometheus MCP:

1. **Install the MCP server**
   ```bash
   npm install -g @anthropic/mcp-server-prometheus
   ```
````

2. **Add to Claude MCP configuration** (`.claude/mcp.json`):

   ```json
   {
     "mcpServers": {
       "prometheus": {
         "command": "mcp-server-prometheus",
         "args": [],
         "env": {
           "PROMETHEUS_URL": "{prometheus_url}"
         }
       }
     }
   }
   ```

3. **Verify connectivity**:
   ```bash
   curl -s {prometheus_url}/-/healthy
   curl -s '{prometheus_url}/api/v1/query?query=up'
   ```

````

#### Loki MCP

```markdown
To set up Loki MCP:

1. **Install the MCP server**
   ```bash
   npm install -g @anthropic/mcp-server-loki
````

2. **Add to Claude MCP configuration** (`.claude/mcp.json`):

   ```json
   {
     "mcpServers": {
       "loki": {
         "command": "mcp-server-loki",
         "args": [],
         "env": {
           "LOKI_URL": "{loki_url}"
         }
       }
     }
   }
   ```

3. **Verify connectivity**:
   ```bash
   curl -s {loki_url}/ready
   ```

````

#### Alertmanager MCP

```markdown
To set up Alertmanager MCP:

1. **Install the MCP server**
   ```bash
   npm install -g @anthropic/mcp-server-alertmanager
````

2. **Add to Claude MCP configuration** (`.claude/mcp.json`):

   ```json
   {
     "mcpServers": {
       "alertmanager": {
         "command": "mcp-server-alertmanager",
         "args": [],
         "env": {
           "ALERTMANAGER_URL": "{alertmanager_url}"
         }
       }
     }
   }
   ```

3. **Verify connectivity**:
   ```bash
   curl -s {alertmanager_url}/-/healthy
   ```

````

### Combined MCP Configuration

Show combined configuration for `.claude/mcp.json`:

```json
{
  "mcpServers": {
    "prometheus": {
      "command": "mcp-server-prometheus",
      "args": [],
      "env": {
        "PROMETHEUS_URL": "{prometheus_url}"
      }
    },
    "loki": {
      "command": "mcp-server-loki",
      "args": [],
      "env": {
        "LOKI_URL": "{loki_url}"
      }
    },
    "alertmanager": {
      "command": "mcp-server-alertmanager",
      "args": [],
      "env": {
        "ALERTMANAGER_URL": "{alertmanager_url}"
      }
    }
  }
}
````

### User Confirmation

```
AskUserQuestion: "Have you installed and configured the observability MCPs?"

Options:
- Yes, all configured
- Some configured (specify which)
- Skip for now
- Help (show troubleshooting)
```

### Verification

For each MCP that was configured:

```
Prometheus:
  Try querying: mcp__prometheus__query with query="up"
  IF success: PASS
  ELSE: FAIL (show error)

Loki:
  Try querying: mcp__loki__query_range with query="{job=~\".+\"}"
  IF success: PASS
  ELSE: FAIL (show error)

Alertmanager:
  Try querying: mcp__alertmanager__get_alerts
  IF success: PASS
  ELSE: FAIL (show error)
```

### Actions

1. Update progress:

   ```yaml
   phases:
     8_observability_mcp:
       status: completed # or skipped or partial
       data:
         prometheus:
           url: { prometheus_url }
           installed: true/false
           tested: true/false
         loki:
           url: { loki_url }
           installed: true/false
           tested: true/false
         alertmanager:
           url: { alertmanager_url }
           installed: true/false
           tested: true/false
         skipped: true/false
   current_phase: 9
   last_updated: { now }
   ```

2. **IMPORTANT**: If any MCP was just installed, inform user:

   ```
   "MCP configuration changed. Please restart Claude Code and run /init again to continue."
   ```

   Save progress and exit gracefully.

3. Proceed to Phase 9

---

## Phase 9: Configure Secrets

**Purpose**: Set up required secrets using the selected approach (Vault or SOPS)

### Collect Secrets

Get secrets from Phase 3:

```
secrets = phases.3_configuration.data.environment_variables
          .filter(v => v.secret == true)

secret_approach = phases.3_configuration.data.secret_approach
```

### Skip if No Secrets

```
IF secrets is empty OR secret_approach == null:
  Show: "No secrets to configure."
  Mark phase as completed
  Skip to Phase 10
```

### Skip if Setup Skipped

If secret management was skipped in Phase 6:

```
Show: "Secret management was skipped in Phase 6. Secrets must be configured manually."
Show list of secrets that need configuration
Mark phase as completed
Skip to Phase 10
```

---

### Vault Approach

```
IF secret_approach == "vault":
  FOR each secret in secrets:
    Show:
      "Please configure secret: {secret.name}

      Path: secret/data/{project_name}/{environment}
      Description: {secret.description}

      Using Vault CLI:
      vault kv put secret/{project_name}/staging {secret.name}='your_value'
      vault kv put secret/{project_name}/production {secret.name}='your_value'

      Or use the Vault MCP:
      mcp__vault__write(path='secret/data/{project_name}/staging/{secret.name}', data={value: '...'})

      Or use the Vault UI."

    AskUserQuestion: "Have you configured this secret?"
    Options:
    - Yes, it's configured
    - Skip this secret

    IF yes:
      Try to verify secret is readable via MCP (don't log actual value!)
      IF readable: Add to secrets_configured
      ELSE: Show warning, add to secrets_pending
    ELSE:
      Add to secrets_pending
```

---

### SOPS Approach

```
IF secret_approach == "sops":
  Show:
    "Create encrypted secrets files for each environment.

    For staging, create secrets/staging.yaml with these secrets:
    {list of secret names}

    For production, create secrets/production.yaml with these secrets:
    {list of secret names}"

  Show:
    "To create staging secrets file:

    1. Create the file:
       sops secrets/staging.yaml

    2. Add secrets in YAML format:
       DATABASE_URL: postgresql://user:password@host:5432/db
       API_KEY: your-api-key-here
       ...

    3. Save and close - SOPS will encrypt automatically."

  AskUserQuestion: "Have you created the encrypted secrets files?"
  Options:
  - Yes, secrets files created
  - Skip (I'll create them later)

  IF yes:
    Check if secrets/staging.yaml exists
    Check if secrets/production.yaml exists
    IF both exist:
      Try to decrypt and verify keys match expected secrets
      Add found secrets to secrets_configured
      Add missing secrets to secrets_pending
    ELSE:
      Show warning about missing files
      Add all to secrets_pending
  ELSE:
    Add all to secrets_pending
```

---

### Summary

```
Show:
  "Secrets Summary ({secret_approach}):
   - Configured: {count} ({list})
   - Pending: {count} ({list})

   You can configure pending secrets later and re-run /init."
```

### Actions

1. Update progress:

   ```yaml
   phases:
     9_secrets:
       status: completed  # or partial
       data:
         approach: "vault" | "sops"
         secrets_configured: [list]
         secrets_pending: [list]
   current_phase: 10
   last_updated: {now}
   ```

2. Proceed to Phase 10

---

## Phase 10: Setup CI Issue

**Purpose**: Create initial issue and process it through the workflow

### Skip if No CI MCP

If CI MCP was skipped in Phase 7:

```
Show: "CI MCP was skipped. Cannot create setup issue automatically."
Show: "You can create the issue manually later."
Mark phase as skipped
Skip to Phase 11
```

### Create Issue

Using CI MCP, create the "Setup CI Pipeline" issue:

```markdown
Title: Setup CI Pipeline

Body:

## Feature: Setup CI Pipeline

### Problem/Request

Configure the CI/CD pipeline for this project with automated testing,
linting, and deployment workflows.

### Expected Behavior

- All commits trigger validation (lint, typecheck, test)
- PRs require passing checks before merge
- Staging deploys automatically on develop branch
- Production deploys manually from main branch

### Acceptance Criteria

- [ ] CI config file created ({config_file} based on platform)
- [ ] Lint step configured with {lint_tools}
- [ ] Test step configured with {coverage_threshold}% coverage threshold
- [ ] Staging deploy job configured
- [ ] Production deploy job configured (manual trigger)

### Technical Notes

- Platform: {platform}
- Config location: {config_path}
- See CLAUDE.md CI/CD section for details

Labels: type:feature, state:backlog
```

### Process Issue

Invoke the crunch skill to process the issue:

```
/crunch {issue_number}

Target state: ready-to-merge

This will:
1. Enrich the issue with investigation
2. Create implementation branch
3. Implement CI configuration
4. Run through validation
5. Update documentation
6. Prepare for merge
```

### Actions

1. Update progress:

   ```yaml
   phases:
     9_setup_ci_issue:
       status: completed
       data:
         issue_number: { number }
         issue_url: { url }
         final_state: ready-to-merge
   current_phase: 11
   last_updated: { now }
   ```

2. Proceed to Phase 11

---

## Phase 11: Verify Checklist

**Purpose**: Final verification that all setup items are complete

### Check Each Item

1. **Secret management configured** (Vault or SOPS)

   ```
   IF phases.6_secret_setup.data.approach == "vault":
     IF phases.6_secret_setup.data.vault_tested == true:
       Try vault MCP command
       IF success: PASS
       ELSE: FAIL
     ELSE: FAIL
   ELSE IF phases.6_secret_setup.data.approach == "sops":
     Check sops --version and age --version
     IF both work: PASS
     ELSE: FAIL
   ELSE IF phases.6_secret_setup.status == skipped:
     SKIP (note: manual management or no secrets)
   ELSE:
     FAIL
   ```

2. **All required secrets configured**

   ```
   IF phases.8_secrets.data.secrets_pending is empty:
     PASS
   ELSE:
     PARTIAL (list pending secrets)
   ```

3. **CI MCP installed and working** (or Filebase initialized)

   ```
   IF platform == "filebase":
     Check .claude/ci-filebase/ directory exists
     Check .claude/ci-filebase/labels.yaml exists
     Check .claude/ci-filebase/counter.yaml exists
     IF all exist: PASS
     ELSE: FAIL (run /ci-filebase init)
   ELSE IF phases.7_ci_mcp.data.tested == true:
     Try CI command
     IF success: PASS
     ELSE: FAIL
   ELSE IF phases.7_ci_mcp.status == skipped:
     SKIP (note: manual setup needed)
   ELSE:
     FAIL
   ```

4. **Docker CI configured** (Filebase only)

   ```
   IF platform == "filebase":
     IF phases.7_ci_mcp.data.docker_ci_enabled == true:
       Check .claude/ci-filebase/docker/ directory exists
       Check .claude/ci-filebase/docker/docker-compose.staging.yaml exists
       Check .claude/ci-filebase/docker/ci-config.yaml exists
       IF all exist:
         # Optionally verify Docker is running
         Run: docker ps
         IF success: PASS
         ELSE: WARN (Docker not running, but config exists)
       ELSE: FAIL (run /ci-filebase docker init)
     ELSE:
       SKIP (note: Docker CI not enabled, manual E2E required)
   ELSE:
     SKIP (not applicable for MCP-based CI)
   ```

5. **CI config ready**

   ```
   Check for workflow file:
   - GitHub: .github/workflows/*.yml exists
   - GitLab: .gitlab-ci.yml exists
   - Gitea: .gitea/workflows/*.yml exists
   - Filebase: .claude/ci-filebase/ directory exists (already checked above)

   IF exists: PASS
   ELSE: FAIL (or PENDING if issue is in progress)
   ```

6. **Observability MCPs configured**

   ```
   IF phases.8_observability_mcp.status == skipped:
     SKIP (note: /patrol --logs and --metrics unavailable)
   ELSE:
     Check each MCP:

     Prometheus:
       IF phases.8_observability_mcp.data.prometheus.tested == true:
         Try prometheus MCP command
         IF success: PASS
         ELSE: FAIL
       ELSE: SKIP

     Loki:
       IF phases.8_observability_mcp.data.loki.tested == true:
         Try loki MCP command
         IF success: PASS
         ELSE: FAIL
       ELSE: SKIP

     Alertmanager:
       IF phases.8_observability_mcp.data.alertmanager.tested == true:
         Try alertmanager MCP command
         IF success: PASS
         ELSE: FAIL
       ELSE: SKIP
   ```

7. **Setup CI issue created**

   ```
   IF phases.10_setup_ci_issue.data.issue_number exists:
     Verify issue exists via CI MCP
     IF exists: PASS
     ELSE: FAIL
   ELSE IF phases.10_setup_ci_issue.status == skipped:
     SKIP
   ELSE:
     FAIL
   ```

8. **Setup CI issue crunched to ready-to-merge**
   ```
   IF phases.10_setup_ci_issue.data.final_state in [ready-to-merge, done]:
     PASS
   ELSE:
     FAIL (show current state)
   ```

### Update CLAUDE.md Checklist

Read CLAUDE.md, find "## Setup Checklist" section, update with results:

**For MCP-based CI (GitHub/GitLab/Gitea):**

```markdown
## Setup Checklist

[x] Vault MCP installed and working
Verified: mcp**vault**status returned healthy
[x] All required secrets configured
Verified: 3/3 secrets readable
[x] CI MCP installed and working
Verified: mcp**github**get_repo returned myorg/myproject
[x] CI config ready
Verified: .github/workflows/ci.yml exists
[x] Observability MCPs configured
Verified: Prometheus, Loki, Alertmanager MCPs working
[x] Setup CI issue created
Verified: Issue #1 exists
[x] Setup CI issue crunched to ready-to-merge
Verified: Issue #1 has label state:ready-to-merge
```

**For Filebase CI:**

```markdown
## Setup Checklist

[x] SOPS and age installed
Verified: sops --version, age --version work
[x] All required secrets configured
Verified: secrets/staging.yaml and secrets/production.yaml exist
[x] Filebase CI initialized
Verified: .claude/ci-filebase/ directory exists
[x] Docker CI configured
Verified: .claude/ci-filebase/docker/ directory exists
[x] Docker CI pipeline tested
Verified: /ci-filebase docker ci passed
[x] Local staging works
Verified: /ci-filebase docker deploy staging succeeded
[x] Setup CI issue created
Verified: Issue #1 exists in .claude/ci-filebase/issues/1.yaml
[x] Setup CI issue crunched to ready-to-merge
Verified: Issue #1 has label state:ready-to-merge
```

Use `[x]` for passed, `[ ]` for failed/pending, and add verification notes.

### Report Summary

```
Show:
  "Project Initialization Complete!

  Summary:
  - CLAUDE.md: Created with full configuration
  - Secret Management: {Vault / SOPS / Skipped}
  - CI Platform: {GitHub / GitLab / Gitea / Filebase}
  {IF platform == "filebase"}
  - Filebase CI: Initialized
  - Docker CI: {Enabled / Disabled}
    {IF docker_ci_enabled}
    - CI Pipeline: {Tested / Not tested}
    - Staging: Local Docker
    {ELSE}
    - E2E: Manual testing required
    {ENDIF}
  {ELSE}
  - CI MCP: {Working / Skipped / Failed}
  {ENDIF}
  - Secrets: {N} configured, {M} pending
  - Setup Issue: #{number} - {state}

  Checklist: {passed}/{total} items complete

  {If all passed}:
    All setup items verified! You're ready to start development.
    Deleting init-progress.txt...

  {If some failed/pending}:
    Some items are incomplete. Run /init again to resume.
    Progress saved in .claude/init-progress.txt"
```

### Actions

1. Update progress:

   ```yaml
   phases:
     11_verify_checklist:
       status: completed
       data:
         secret_setup_working: true/false # Vault MCP or SOPS+age
         secrets_configured: true/false
         ci_mcp_working: true/false
         ci_config_ready: true/false
         docker_ci_working: true/false/skip # Filebase only
         observability_configured: true/false
         prometheus_working: true/false
         loki_working: true/false
         alertmanager_working: true/false
         setup_issue_created: true/false
         setup_issue_crunched: true/false
   ```

2. Update CLAUDE.md checklist section

3. IF all items passed:
   - Delete `.claude/init-progress.txt`
   - Show success message

4. IF some items failed/pending:
   - Keep progress file
   - Show status and instructions to resume

---

## Error Handling

### MCP Command Failures

```
IF MCP command fails:
  Record error in progress file:
    errors:
      - phase: {N}
        timestamp: {now}
        message: {error}
        resolved: false

  AskUserQuestion: "Command failed: {error}"
  Options:
  - Retry
  - Skip this step
  - Get help

  IF retry: Try again
  IF skip: Mark as skipped, continue
  IF help: Show troubleshooting tips
```

### Validation Failures

```
IF user input invalid:
  Explain the validation rule
  Show correct format example
  Re-ask the question
```

### Interrupted Session

Progress is automatically saved after each phase. User can run `/init` again to resume.

---

## Template Reference

### Templates Directory

All templates are in the plugin's `templates/` directory (relative to plugin root):

| File                     | Content                    | Customization                                 |
| ------------------------ | -------------------------- | --------------------------------------------- |
| `1-project.md`           | Project header             | Replace {PROJECT_NAME}, {PROJECT_DESCRIPTION} |
| `2-agents.md`            | Agent system table         | None (static)                                 |
| `3-tdd.md`               | TDD requirements           | None (static)                                 |
| `4-observability.md`     | Logging/metrics            | None (static)                                 |
| `5-workflow.md`          | Task workflow              | None (static)                                 |
| `6-ci.md`                | CI platform selection      | Keep selected platform row only               |
| `6-ci-github.md`         | GitHub reference           | Include if platform=github                    |
| `6-ci-gitlab.md`         | GitLab reference           | Include if platform=gitlab                    |
| `6-ci-gitea.md`          | Gitea reference            | Include if platform=gitea                     |
| `6-ci-filebase.md`       | Filebase reference         | Include if platform=filebase                  |
| `7-configuration.md`     | Config management          | Fill in env vars and feature flags            |
| `8-secret-management.md` | Secret management overview | Keep selected approach row only               |
| `8-secrets-vault.md`     | Vault reference            | Include if secret_approach=vault              |
| `8-secrets-sops.md`      | SOPS reference             | Include if secret_approach=sops               |
| `9-deployment.md`        | Deployment procedures      | Fill in staging/prod details                  |
| `99-setup-checklist.md`  | Setup verification         | Update with [x] marks                         |

### Placeholder Map

```
{PROJECT_NAME}        -> Phase 1: project_name
{PROJECT_DESCRIPTION} -> Phase 1: project_description
{org}                 -> Phase 2: owner
{owner}               -> Phase 2: owner
{repo}                -> Phase 2: repo
{project_id}          -> Phase 2: owner/repo
{80}                  -> Phase 2: coverage_threshold
{active}              -> "active" for selected platform
```
