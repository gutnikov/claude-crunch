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
    status: completed  # pending | in_progress | completed | skipped
    completed_at: "2024-01-15T10:32:00Z"
    data:
      project_name: "my-awesome-project"
      project_description: "A distributed system for processing events"

  2_ci_platform:
    status: completed
    data:
      platform: "github"  # github | gitlab | gitea
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

  6_vault_mcp:
    status: pending
    data:
      installed: false
      configured: false
      tested: false

  7_ci_mcp:
    status: pending
    data:
      installed: false
      configured: false
      tested: false

  8_secrets:
    status: pending
    data:
      secrets_configured: []
      secrets_pending: []

  9_setup_ci_issue:
    status: pending
    data:
      issue_number: null
      issue_url: null
      final_state: null

  10_verify_checklist:
    status: pending
    data:
      vault_mcp_working: false
      secrets_configured: false
      ci_mcp_working: false
      ci_config_ready: false
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
       completed_at: {now}
       data:
         project_name: {answer1}
         project_description: {answer2}
   ```

2. Update progress:
   ```yaml
   current_phase: 2
   last_updated: {now}
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
   ```

2. **Repository Owner**
   ```
   AskUserQuestion: "What is your repository owner/organization?"

   Header: "Repo owner"
   Example: myorg, username, company-name
   ```

3. **Repository Name**
   ```
   AskUserQuestion: "What is your repository name?"

   Header: "Repo name"
   Example: my-project, backend-api
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
       completed_at: {now}
       data:
         platform: {answer1}
         owner: {answer2}
         repo: {answer3}
         lint_tools: {answer4}
         coverage_threshold: {answer5}
   ```

2. Update progress:
   ```yaml
   current_phase: 3
   last_updated: {now}
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
   ```

2. Identify secrets for Phase 8:
   ```
   secrets_needed = environment_variables.filter(v => v.secret == true)
   ```

3. Update progress:
   ```yaml
   current_phase: 4
   last_updated: {now}
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
       completed_at: {now}
       data:
         staging:
           url: {staging_url}
           api_url: {staging_api}
           logs_url: {staging_logs}
           monitoring_url: {staging_monitoring}
           infrastructure: {staging_infra}
         production:
           url: {prod_url}
           api_url: {prod_api}
           logs_url: {prod_logs}
           monitoring_url: {prod_monitoring}
           status_page_url: {status_page}
           infrastructure: {prod_infra}
   ```

2. Update progress:
   ```yaml
   current_phase: 5
   last_updated: {now}
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
9. `8-deployment.md`
10. `99-setup-checklist.md`

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
| Platform | Status | Config File          | MCP Server                              |
| -------- | ------ | -------------------- | --------------------------------------- |
| GitHub   | active | `.github/workflows/` | [github-mcp-server](https://github.com/github/github-mcp-server) |
```

### Environment Variables (7-configuration.md)

Replace the TEMPLATE section with actual data:
```markdown
### Environment Variables

| Variable | Description | Secret | Source |
|----------|-------------|--------|--------|
| DATABASE_URL | PostgreSQL connection string | Yes | Vault |
| REDIS_URL | Redis cache connection | Yes | Vault |
| LOG_LEVEL | Logging verbosity | No | Ansible |
```

### Feature Flags (7-configuration.md)

Replace the TEMPLATE section with actual data:
```markdown
### Feature Flags

```yaml
defaults:
  new_ui: false
  beta_api: false
  debug_mode: false
```
```

### Deployment (8-deployment.md)

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
```

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
   ```

3. Proceed to Phase 6

---

## Phase 6: Vault MCP Setup

**Purpose**: Install and configure Vault MCP server for secret management

### Check Existing

First, check if Vault MCP is already installed:
```
Try executing a vault MCP command (e.g., mcp__vault__status or similar)

IF success:
  Show: "Vault MCP is already installed and working!"
  Mark phase as completed
  Skip to Phase 7
```

### Guide Installation

If not installed, show instructions:

```markdown
To set up Vault MCP, you'll need to:

1. **Install the vault-mcp server**
   ```bash
   npm install -g @anthropic/vault-mcp
   ```

   Or if using a specific Vault provider, check their MCP documentation.

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

3. **Restart Claude Code**

   After configuring, you'll need to restart Claude to load the MCP server.
```

### User Confirmation

```
AskUserQuestion: "Have you completed the Vault MCP installation?"

Options:
- Yes, I've installed and configured it
- Skip (I'll manage secrets manually)
- Help (show more details)
```

### Verification

If user says yes:
```
Try vault MCP command again

IF success:
  Show: "Vault MCP verified and working!"
  Mark phase as completed

IF fail:
  Show error message
  AskUserQuestion: "Would you like to:"
  Options:
  - Retry verification
  - Skip and continue
  - See troubleshooting tips
```

### Actions

1. Update progress:
   ```yaml
   phases:
     6_vault_mcp:
       status: completed  # or skipped
       data:
         installed: true/false
         configured: true/false
         tested: true/false
   current_phase: 7
   last_updated: {now}
   ```

2. **IMPORTANT**: If MCP was just installed, inform user:
   ```
   "MCP configuration changed. Please restart Claude Code and run /init again to continue."
   ```
   Save progress and exit gracefully.

3. Proceed to Phase 7

---

## Phase 7: CI MCP Setup

**Purpose**: Install and configure CI platform MCP server

### Determine MCP Server

Based on Phase 2 platform selection:
- GitHub -> `github-mcp-server`
- GitLab -> GitLab Duo MCP (embedded)
- Gitea -> `gitea-mcp`

### Check Existing

```
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

```markdown
To set up GitHub MCP:

1. **Install github-mcp-server**
   ```bash
   npm install -g @github/mcp-server
   ```

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
```

#### GitLab

```markdown
GitLab uses Duo MCP which is embedded in GitLab.

1. **Enable GitLab Duo in your instance**

   See: https://docs.gitlab.com/user/gitlab_duo/model_context_protocol/

2. **No separate installation needed**

   GitLab Duo MCP is automatically available when using Claude with GitLab.
```

#### Gitea

```markdown
To set up Gitea MCP:

1. **Install gitea-mcp**
   ```bash
   npm install -g gitea-mcp
   ```

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
```

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
   ```

2. **IMPORTANT**: If MCP was just installed, inform user to restart Claude.

3. Proceed to Phase 8

---

## Phase 8: Configure Secrets

**Purpose**: Set up required secrets in Vault

### Collect Secrets

Get secrets from Phase 3:
```
secrets = phases.3_configuration.data.environment_variables
          .filter(v => v.secret == true)
```

### Skip if No Vault

If Vault MCP was skipped in Phase 6:
```
Show: "Vault MCP was skipped. Secrets must be configured manually."
Show list of secrets that need configuration
Mark phase as completed
Skip to Phase 9
```

### For Each Secret

```
FOR each secret in secrets:
  Show:
    "Please configure secret: {secret.name}

    Path: secret/data/{project_name}/{environment}
    Description: {secret.description}

    Using Vault CLI:
    vault kv put secret/{project_name}/staging {secret.name}='your_value'
    vault kv put secret/{project_name}/production {secret.name}='your_value'

    Or use the Vault UI to set these values."

  AskUserQuestion: "Have you configured this secret?"
  Options:
  - Yes, it's configured
  - Skip this secret

  IF yes:
    Try to verify secret is readable (don't log actual value!)
    IF readable:
      Add to secrets_configured list
    ELSE:
      Show warning, add to secrets_pending list
  ELSE:
    Add to secrets_pending list
```

### Summary

```
Show:
  "Secrets Summary:
   - Configured: {count} ({list})
   - Pending: {count} ({list})

   You can configure pending secrets later and re-run /init."
```

### Actions

1. Update progress:
   ```yaml
   phases:
     8_secrets:
       status: completed  # or partial
       data:
         secrets_configured: [list]
         secrets_pending: [list]
   current_phase: 9
   last_updated: {now}
   ```

2. Proceed to Phase 9

---

## Phase 9: Setup CI Issue

**Purpose**: Create initial issue and process it through the workflow

### Skip if No CI MCP

If CI MCP was skipped in Phase 7:
```
Show: "CI MCP was skipped. Cannot create setup issue automatically."
Show: "You can create the issue manually later."
Mark phase as skipped
Skip to Phase 10
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
         issue_number: {number}
         issue_url: {url}
         final_state: ready-to-merge
   current_phase: 10
   last_updated: {now}
   ```

2. Proceed to Phase 10

---

## Phase 10: Verify Checklist

**Purpose**: Final verification that all setup items are complete

### Check Each Item

1. **Vault MCP installed and working**
   ```
   IF phases.6_vault_mcp.data.tested == true:
     Try vault command
     IF success: PASS
     ELSE: FAIL
   ELSE IF phases.6_vault_mcp.status == skipped:
     SKIP (note: manual management)
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

3. **CI MCP installed and working**
   ```
   IF phases.7_ci_mcp.data.tested == true:
     Try CI command
     IF success: PASS
     ELSE: FAIL
   ELSE IF phases.7_ci_mcp.status == skipped:
     SKIP (note: manual setup needed)
   ELSE:
     FAIL
   ```

4. **CI config ready**
   ```
   Check for workflow file:
   - GitHub: .github/workflows/*.yml exists
   - GitLab: .gitlab-ci.yml exists
   - Gitea: .gitea/workflows/*.yml exists

   IF exists: PASS
   ELSE: FAIL (or PENDING if issue is in progress)
   ```

5. **Setup CI issue created**
   ```
   IF phases.9_setup_ci_issue.data.issue_number exists:
     Verify issue exists via CI MCP
     IF exists: PASS
     ELSE: FAIL
   ELSE IF phases.9_setup_ci_issue.status == skipped:
     SKIP
   ELSE:
     FAIL
   ```

6. **Setup CI issue crunched to ready-to-merge**
   ```
   IF phases.9_setup_ci_issue.data.final_state in [ready-to-merge, done]:
     PASS
   ELSE:
     FAIL (show current state)
   ```

### Update CLAUDE.md Checklist

Read CLAUDE.md, find "## Setup Checklist" section, update with results:

```markdown
## Setup Checklist

[x] Vault MCP installed and working
    Verified: mcp__vault__status returned healthy
[x] All required secrets configured
    Verified: 3/3 secrets readable
[x] CI MCP installed and working
    Verified: mcp__github__get_repo returned myorg/myproject
[x] CI config ready
    Verified: .github/workflows/ci.yml exists
[x] Setup CI issue created
    Verified: Issue #1 exists
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
  - Vault MCP: {Working / Skipped / Failed}
  - CI MCP: {Working / Skipped / Failed}
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
     10_verify_checklist:
       status: completed
       data:
         vault_mcp_working: true/false
         secrets_configured: true/false
         ci_mcp_working: true/false
         ci_config_ready: true/false
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

| File | Content | Customization |
|------|---------|---------------|
| `1-project.md` | Project header | Replace {PROJECT_NAME}, {PROJECT_DESCRIPTION} |
| `2-agents.md` | Agent system table | None (static) |
| `3-tdd.md` | TDD requirements | None (static) |
| `4-observability.md` | Logging/metrics | None (static) |
| `5-workflow.md` | Task workflow | None (static) |
| `6-ci.md` | CI platform selection | Keep selected platform row only |
| `6-ci-github.md` | GitHub reference | Include if platform=github |
| `6-ci-gitlab.md` | GitLab reference | Include if platform=gitlab |
| `6-ci-gitea.md` | Gitea reference | Include if platform=gitea |
| `7-configuration.md` | Config management | Fill in env vars and feature flags |
| `8-deployment.md` | Deployment procedures | Fill in staging/prod details |
| `99-setup-checklist.md` | Setup verification | Update with [x] marks |

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
