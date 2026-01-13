# claude-crunch

A Claude Code plugin providing development workflow automation with structured issue processing, specialized agents, and project initialization.

## Features

- **9 Specialized Agents** - Domain-specific agents for architecture, security, DevOps, and language-specific development
- **Issue Workflow (`/crunch`)** - Process issues through structured states from input to ready-to-merge
- **Project Initialization (`/init`)** - Interactive setup wizard that generates CLAUDE.md and configures MCP servers
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

| Agent | Purpose |
|-------|---------|
| `system-architect` | System design, architecture decisions, technical specifications |
| `security-analyst` | Threat modeling, vulnerability assessment, secure code review |
| `devops-engineer` | Infrastructure, CI/CD, monitoring, deployment |
| `dev-cpp` | C++ implementation, memory management, performance optimization |
| `dev-go` | Go services, concurrency patterns, CLI tools |
| `dev-python` | Python development, data processing, ML/AI |
| `dev-react` | TypeScript/React frontend, components, hooks |
| `techwriter` | Documentation, API docs, user guides |
| `reviewer` | Code review, documentation review, configuration review |

## Templates

Templates in `templates/` are used by `/init` to generate `CLAUDE.md`:

| Template | Content |
|----------|---------|
| `1-project.md` | Project name and description |
| `2-agents.md` | Agent system reference |
| `3-tdd.md` | Test-driven development requirements |
| `4-observability.md` | Logging and metrics standards |
| `5-workflow.md` | Task types and state flow |
| `6-ci.md` | CI/CD platform configuration |
| `6-ci-github.md` | GitHub MCP reference |
| `6-ci-gitlab.md` | GitLab MCP reference |
| `6-ci-gitea.md` | Gitea MCP reference |
| `7-configuration.md` | Environment variables and feature flags |
| `8-deployment.md` | Staging and production deployment |
| `99-setup-checklist.md` | Setup verification checklist |

## Requirements

- Claude Code CLI
- For full functionality:
  - Vault MCP server (secret management)
  - CI MCP server (GitHub/GitLab/Gitea)

## Plugin Structure

```
claude-crunch/
├── .claude-plugin/
│   └── plugin.json      # Plugin manifest
├── agents/              # Specialized agent definitions
├── skills/
│   ├── init/           # Project initialization skill
│   └── crunch/         # Issue processing skill
└── templates/          # CLAUDE.md section templates
```

## License

MIT
