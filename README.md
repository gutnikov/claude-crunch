# claude-crunch

A Claude Code plugin providing development workflow automation with structured issue processing, multi-agent orchestration, and project initialization.

## Features

- **16 Specialized Agents** - Domain-specific agents with hierarchical coordination for architecture, security, DevOps, and language-specific development
- **Multi-Agent Orchestration** - Automatic coordination for complex, multi-domain issues with conflict resolution and adaptive routing
- **Issue Workflow (`/crunch`)** - Process issues through structured states from input to ready-to-merge
- **Project Initialization (`/init`)** - Interactive setup wizard that generates CLAUDE.md and configures MCP servers
- **Code Review (`/review`)** - Parallel multi-agent code review with 6 specialized reviewers
- **Templates** - Predefined sections for building comprehensive project documentation

## Installation

### Development (test locally)

```bash
claude --plugin-dir /path/to/claude-crunch
```

### Via Marketplace

```bash
# Add this repository as a marketplace
/plugin marketplace add owner/claude-crunch

# Install the plugin
/plugin install claude-crunch
```

### Scoped Installation

```bash
# User scope (all your projects)
claude plugin install claude-crunch --scope user

# Project scope (shared via git)
claude plugin install claude-crunch --scope project

# Local scope (gitignored)
claude plugin install claude-crunch --scope local
```

## Usage

Commands are namespaced with `claude-crunch:` when installed as a plugin.

### Initialize a Project

```
/claude-crunch:init
```

Interactive wizard that:
1. Gathers project information (name, description)
2. Configures CI platform (GitHub/GitLab/Gitea)
3. Sets up environment variables and feature flags
4. Configures deployment environments
5. Generates `CLAUDE.md` from templates
6. Guides Vault and CI MCP installation
7. Creates and processes a setup issue
8. Verifies the setup checklist

Progress is saved to `.claude/init-progress.txt` - resume anytime by running `/init` again.

### Process Issues

```
/claude-crunch:crunch 42
```

Moves issue #42 through workflow states:

```
INPUT → BACKLOG → ENRICH → READY → IMPLEMENTING → VALIDATION → DOCS → READY-TO-MERGE → DONE
```

Features:
- Dynamic agent selection based on issue domain
- TDD enforcement
- Automatic branch creation and PR workflow
- Issue body updates with progress tracking

## Agents

### Agent Hierarchy

Agents are organized in a 4-tier hierarchy:

| Tier | Role | Agents |
|------|------|--------|
| 1 | Orchestrator | `orchestrator` |
| 2 | Supervisors | `system-architect`, `security-analyst`, `devops-engineer` |
| 3 | Specialists | `dev-*`, `reviewer`, `qa-engineer`, `techwriter`, `knowledge-manager` |
| 4 | Responders | `incident-responder`, `log-analyst`, `code-health-analyst`, `dependency-manager` |

### Available Agents

| Agent | Purpose |
|-------|---------|
| `orchestrator` | Multi-agent coordination for complex issues |
| `system-architect` | System design, architecture decisions, technical specifications |
| `security-analyst` | Threat modeling, vulnerability assessment, secure code review (VETO authority) |
| `devops-engineer` | Infrastructure, CI/CD, monitoring, deployment |
| `dev-cpp` | C++ implementation, memory management, performance optimization |
| `dev-go` | Go services, concurrency patterns, CLI tools |
| `dev-python` | Python development, data processing, ML/AI |
| `dev-react` | TypeScript/React frontend, components, hooks |
| `reviewer` | Code review, documentation review, configuration review |
| `qa-engineer` | Test planning, coverage analysis, TDD enforcement |
| `techwriter` | Documentation, API docs, user guides |
| `knowledge-manager` | Learning from history, pattern analysis |
| `code-health-analyst` | Code metrics, tech debt detection |
| `log-analyst` | Log pattern detection, anomaly detection |
| `dependency-manager` | CVE scanning, dependency updates |
| `incident-responder` | Staging validation, anomaly diagnosis |

### Customizing Agents

The agents included in this plugin are **general-purpose templates**. For best results, customize them for your specific project.

**Agent files location:** `agents/*.md` in the plugin directory

**What to customize:**

- **Project-specific libraries** - Add your actual frameworks (e.g., replace generic "Django, Flask, FastAPI" with your specific stack)
- **Code conventions** - Add your naming conventions, file structure patterns, import ordering rules
- **Architecture patterns** - Document your specific patterns (repositories, services, DTOs, etc.)
- **Testing requirements** - Specify your test frameworks, coverage requirements, mocking approaches
- **Domain knowledge** - Add business domain context that agents should understand

**Example:** If your Python project uses FastAPI + SQLAlchemy + Pydantic, edit `agents/dev-python.md` to reference these specifically, including your project's patterns for models, schemas, and dependency injection.

## Templates

Templates in `templates/` are used by `/init` to generate `CLAUDE.md`:

### Core Templates

| Template | Content |
|----------|---------|
| `1-project.md` | Project name and description |
| `2-agents.md` | Agent system and hierarchy reference |
| `3-tdd.md` | Test-driven development requirements |
| `4-observability.md` | Logging and metrics standards |
| `5-workflow.md` | Task types and state flow |
| `6-ci.md` | CI/CD platform configuration |
| `6-ci-github.md` | GitHub MCP reference |
| `6-ci-gitlab.md` | GitLab MCP reference |
| `6-ci-gitea.md` | Gitea MCP reference |
| `6-ci-filebase.md` | File-based CI reference (no MCP required) |
| `7-configuration.md` | Environment variables and feature flags |
| `8-secret-management.md` | Secret management approach selection |
| `8-secrets-vault.md` | Vault MCP reference |
| `8-secrets-sops.md` | SOPS+age reference (file-based encryption) |
| `9-deployment.md` | Staging and production deployment |
| `99-setup-checklist.md` | Setup verification checklist |

### Orchestration Templates

| Template | Content |
|----------|---------|
| `acp-schema.md` | Agent Communication Protocol message schemas |
| `acp-contracts.md` | Agent input/output contracts |
| `hierarchy-config.md` | 4-tier agent hierarchy configuration |
| `conflict-resolution.md` | Structured debate and voting protocol |
| `veto-rules.md` | Security veto authority rules |
| `team-composition.md` | Dynamic team selection algorithm |
| `agent-performance-schema.md` | Performance tracking for adaptive routing |
| `routing-config.md` | Adaptive agent routing configuration |
| `execution-graph.md` | Parallel execution dependency graphs |
| `checkpoint-config.md` | Workflow checkpoint and recovery |

## Requirements

- Claude Code CLI
- For full functionality:
  - Vault MCP server (secret management)
  - CI MCP server (GitHub/GitLab/Gitea) OR Filebase for local-only workflows

## Plugin Structure

```
claude-crunch/
├── .claude-plugin/
│   └── plugin.json      # Plugin manifest
├── agents/              # 16 specialized agent definitions
│   ├── orchestrator.md  # Multi-agent coordinator
│   ├── system-architect.md
│   ├── security-analyst.md  # Has veto authority
│   └── ...
├── skills/
│   ├── init/           # Project initialization skill
│   ├── crunch/         # Issue processing skill
│   ├── review/         # Multi-agent code review skill
│   ├── orchestrate/    # Multi-agent orchestration skill
│   ├── patrol/         # Continuous monitoring skill
│   ├── learn/          # Knowledge capture skill
│   ├── knowledge/      # Knowledge query skill
│   ├── test-analyze/   # Test quality analysis skill
│   ├── analyze/        # Pattern analysis skill
│   └── ci-filebase/    # File-based CI for local workflows
└── templates/          # CLAUDE.md section and orchestration templates
```

## License

MIT
