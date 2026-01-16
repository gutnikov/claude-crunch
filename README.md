# claude-crunch-marketplace

A Claude Code plugin marketplace containing development workflow automation tools.

## Available Plugins

| Plugin | Description | Category |
|--------|-------------|----------|
| [claude-crunch](#claude-crunch) | Development workflow automation with multi-agent orchestration | development |

## Installation

### Add the Marketplace

```bash
# Add this repository as a marketplace
/plugin marketplace add owner/claude-crunch-marketplace
```

### Install Plugins

```bash
# Install claude-crunch from this marketplace
/plugin install claude-crunch@claude-crunch-marketplace

# Or with scope
/plugin install claude-crunch@claude-crunch-marketplace --scope user
```

### Development (test locally)

```bash
# Test the entire marketplace
claude --plugin-dir /path/to/claude-crunch-marketplace

# Or test a specific plugin
claude --plugin-dir /path/to/claude-crunch-marketplace/plugins/claude-crunch
```

---

## claude-crunch

Development workflow automation with structured issue processing, multi-agent orchestration, and project initialization.

### Features

- **16 Specialized Agents** - Domain-specific agents with hierarchical coordination for architecture, security, DevOps, and language-specific development
- **Multi-Agent Orchestration** - Automatic coordination for complex, multi-domain issues with conflict resolution and adaptive routing
- **Issue Workflow (`/crunch`)** - Process issues through structured states from input to ready-to-merge
- **Project Initialization (`/init`)** - Interactive setup wizard that generates CLAUDE.md and configures MCP servers
- **Code Review (`/review`)** - Parallel multi-agent code review with 6 specialized reviewers
- **Templates** - Predefined sections for building comprehensive project documentation

### Usage

Commands are namespaced with `claude-crunch:` when installed as a plugin.

#### Initialize a Project

```
/claude-crunch:init
```

Interactive wizard that:

1. Gathers project information (name, description)
2. Configures CI platform (GitHub/GitLab/Gitea/Filebase)
3. Sets up environment variables and feature flags
4. Selects secret management approach (Vault or SOPS)
5. Configures deployment environments
6. Generates `CLAUDE.md` from templates
7. Sets up secret management (Vault MCP or SOPS+age)
8. Configures CI (MCP servers or Filebase with Docker)
9. Creates and processes a setup issue
10. Verifies the setup checklist

Progress is saved to `.claude/init-progress.txt` - resume anytime by running `/init` again.

#### Process Issues

```
/claude-crunch:crunch 42
```

Moves issue #42 through workflow states:

```
INPUT → BACKLOG → ENRICH → READY → IMPLEMENTING → E2E → DOCS → READY-TO-MERGE → DONE
```

Features:

- Dynamic agent selection based on issue domain
- TDD enforcement
- Automatic branch creation and PR workflow
- Issue body updates with progress tracking

#### File-based CI with Docker

For projects without access to hosted CI platforms, use Filebase with local Docker:

```
/claude-crunch:ci-filebase init
/claude-crunch:ci-filebase docker init
```

This provides:

- **Local issue/PR tracking** - JSON files in `.claude/ci-filebase/`
- **CI pipeline simulation** - Run lint, test, build locally
- **Local Docker staging** - Deploy and validate in containers
- **Full E2E support** - Complete workflow without external services

##### Quick Start (Filebase + Docker)

```bash
# Initialize file-based CI
/ci-filebase init

# Set up Docker CI (requires Docker installed)
/ci-filebase docker init

# Run CI pipeline locally
/ci-filebase docker ci

# Deploy to local staging
/ci-filebase docker deploy staging

# Check staging health
/ci-filebase docker health

# Stop staging
/ci-filebase docker stop
```

##### Docker CI Commands

| Command                 | Description                               |
| ----------------------- | ----------------------------------------- |
| `docker init`           | Initialize Docker CI infrastructure       |
| `docker ci`             | Run local CI pipeline (lint, test, build) |
| `docker deploy staging` | Deploy to local Docker staging            |
| `docker logs`           | View staging container logs               |
| `docker health`         | Check staging health status               |
| `docker stop`           | Stop local staging environment            |
| `docker rebuild`        | Rebuild and restart staging               |

### Agents

#### Agent Hierarchy

Agents are organized in a 4-tier hierarchy:

| Tier | Role         | Agents                                                                           |
| ---- | ------------ | -------------------------------------------------------------------------------- |
| 1    | Orchestrator | `orchestrator`                                                                   |
| 2    | Supervisors  | `system-architect`, `security-analyst`, `devops-engineer`                        |
| 3    | Specialists  | `dev-*`, `reviewer`, `qa-engineer`, `techwriter`, `knowledge-manager`            |
| 4    | Responders   | `incident-responder`, `log-analyst`, `code-health-analyst`, `dependency-manager` |

#### Available Agents

| Agent                 | Purpose                                                                        |
| --------------------- | ------------------------------------------------------------------------------ |
| `orchestrator`        | Multi-agent coordination for complex issues                                    |
| `system-architect`    | System design, architecture decisions, technical specifications                |
| `security-analyst`    | Threat modeling, vulnerability assessment, secure code review (VETO authority) |
| `devops-engineer`     | Infrastructure, CI/CD, monitoring, deployment                                  |
| `dev-cpp`             | C++ implementation, memory management, performance optimization                |
| `dev-go`              | Go services, concurrency patterns, CLI tools                                   |
| `dev-python`          | Python development, data processing, ML/AI                                     |
| `dev-react`           | TypeScript/React frontend, components, hooks                                   |
| `reviewer`            | Code review, documentation review, configuration review                        |
| `qa-engineer`         | Test planning, coverage analysis, TDD enforcement                              |
| `techwriter`          | Documentation, API docs, user guides                                           |
| `knowledge-manager`   | Learning from history, pattern analysis                                        |
| `code-health-analyst` | Code metrics, tech debt detection                                              |
| `log-analyst`         | Log pattern detection, anomaly detection                                       |
| `dependency-manager`  | CVE scanning, dependency updates                                               |
| `incident-responder`  | Staging validation, anomaly diagnosis                                          |

#### Customizing Agents

The agents included in this plugin are **general-purpose templates**. For best results, customize them for your specific project.

**Agent files location:** `plugins/claude-crunch/agents/*.md`

**What to customize:**

- **Project-specific libraries** - Add your actual frameworks (e.g., replace generic "Django, Flask, FastAPI" with your specific stack)
- **Code conventions** - Add your naming conventions, file structure patterns, import ordering rules
- **Architecture patterns** - Document your specific patterns (repositories, services, DTOs, etc.)
- **Testing requirements** - Specify your test frameworks, coverage requirements, mocking approaches
- **Domain knowledge** - Add business domain context that agents should understand

**Example:** If your Python project uses FastAPI + SQLAlchemy + Pydantic, edit `agents/dev-python.md` to reference these specifically, including your project's patterns for models, schemas, and dependency injection.

### Templates

Templates in `plugins/claude-crunch/templates/` are used by `/init` to generate `CLAUDE.md`:

#### Core Templates

| Template                 | Content                                    |
| ------------------------ | ------------------------------------------ |
| `1-project.md`           | Project name and description               |
| `2-agents.md`            | Agent system and hierarchy reference       |
| `3-tdd.md`               | Test-driven development requirements       |
| `4-observability.md`     | Logging and metrics standards              |
| `5-workflow.md`          | Task types and state flow                  |
| `5-e2e-testability.md`   | Autonomous E2E validation requirements     |
| `5a-incremental-development.md` | Incremental milestone development    |
| `6-ci.md`                | CI/CD platform configuration               |
| `6-ci-github.md`         | GitHub MCP reference                       |
| `6-ci-gitlab.md`         | GitLab MCP reference                       |
| `6-ci-gitea.md`          | Gitea MCP reference                        |
| `6-ci-filebase.md`       | File-based CI reference (no MCP required)  |
| `7-configuration.md`     | Environment variables and feature flags    |
| `8-secret-management.md` | Secret management approach selection       |
| `8-secrets-vault.md`     | Vault MCP reference                        |
| `8-secrets-sops.md`      | SOPS+age reference (file-based encryption) |
| `9-deployment.md`        | Staging and production deployment          |
| `99-setup-checklist.md`  | Setup verification checklist               |

#### Orchestration Templates

| Template                      | Content                                      |
| ----------------------------- | -------------------------------------------- |
| `acp-schema.md`               | Agent Communication Protocol message schemas |
| `acp-contracts.md`            | Agent input/output contracts                 |
| `hierarchy-config.md`         | 4-tier agent hierarchy configuration         |
| `conflict-resolution.md`      | Structured debate and voting protocol        |
| `veto-rules.md`               | Security veto authority rules                |
| `team-composition.md`         | Dynamic team selection algorithm             |
| `agent-performance-schema.md` | Performance tracking for adaptive routing    |
| `routing-config.md`           | Adaptive agent routing configuration         |
| `execution-graph.md`          | Parallel execution dependency graphs         |
| `checkpoint-config.md`        | Workflow checkpoint and recovery             |

### Requirements

- Claude Code CLI
- For full functionality:
  - Vault MCP server OR SOPS+age (secret management)
  - CI MCP server (GitHub/GitLab/Gitea) OR Filebase for local-only workflows
  - Docker and Docker Compose (for Filebase local staging)

---

## Marketplace Structure

```
claude-crunch-marketplace/
├── .claude-plugin/
│   └── marketplace.json      # Marketplace catalog
├── plugins/
│   └── claude-crunch/        # First plugin
│       ├── .claude-plugin/
│       │   └── plugin.json   # Plugin manifest
│       ├── agents/           # 16 specialized agents
│       ├── skills/           # Workflow skills
│       └── templates/        # CLAUDE.md templates
├── LICENSE
└── README.md
```

## Adding New Plugins

To add a new plugin to this marketplace:

1. Create a new directory under `plugins/`:
   ```
   plugins/my-new-plugin/
   ```

2. Add a plugin manifest:
   ```json
   // plugins/my-new-plugin/.claude-plugin/plugin.json
   {
     "name": "my-new-plugin",
     "version": "1.0.0",
     "description": "What this plugin does",
     "agents": "./agents/",
     "skills": "./skills/"
   }
   ```

3. Add plugin content (agents, skills, etc.)

4. Register in marketplace catalog:
   ```json
   // .claude-plugin/marketplace.json - add to plugins array
   {
     "name": "my-new-plugin",
     "description": "What this plugin does",
     "version": "1.0.0",
     "source": "./plugins/my-new-plugin",
     "category": "development"
   }
   ```

## License

MIT
