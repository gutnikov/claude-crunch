---
name: init
description: Initialize a new project with CLAUDE.md configuration. Use when user says /init, wants to set up a new project, configure CLAUDE.md, complete setup checklist, or resume project initialization.
---

# Init - Project Initialization

Initialize your project with a complete CLAUDE.md configuration file and MCP server setup.

## Quick Start

```
/init
```

Starts the initialization wizard or resumes from where you left off.

## Prerequisites

- This skill is part of the `claude-crunch` plugin
- Templates are bundled in the plugin's `templates/` directory
- User must have access to configure MCP servers

## What Gets Set Up

1. **CLAUDE.md** - Project configuration file combining all templates
2. **Vault MCP** - Secret management server
3. **CI MCP** - GitHub/GitLab/Gitea integration
4. **Secrets** - Required credentials in Vault
5. **Setup CI Issue** - Initial issue processed through workflow

## Phases

The init process has 10 phases that can be resumed if interrupted:

| Phase | Name               | Description                                  |
| ----- | ------------------ | -------------------------------------------- |
| 1     | Project Info       | Gather project name and description          |
| 2     | CI Platform        | Select GitHub/GitLab/Gitea, repo info, tools |
| 3     | Configuration      | Environment variables, feature flags         |
| 4     | Deployment         | Staging/Production URLs and infrastructure   |
| 5     | Generate CLAUDE.md | Combine templates with collected data        |
| 6     | Vault MCP          | Guide installation and configuration         |
| 7     | CI MCP             | Guide installation and configuration         |
| 8     | Secrets            | Configure required secrets in Vault          |
| 9     | Setup CI Issue     | Create and crunch initial issue              |
| 10    | Verify Checklist   | Final verification and cleanup               |

## Resume Support

If Claude restarts during initialization, run `/init` again. Progress is saved in `.claude/init-progress.txt` and you'll be offered to resume from where you left off.

To start fresh, delete `.claude/init-progress.txt` before running `/init`.

## Workflow

See [workflow.md](workflow.md) for detailed implementation instructions.

## Example Usage

### Starting Fresh

```
User: /init

Claude: Welcome to project initialization!

I found templates in the claude-crunch. Let me guide you through setup.

Phase 1: Project Information

What is the name of your project?
(Example: tmux-claude-bot, payment-service, frontend-dashboard)

User: my-awesome-api

Claude: Great! Now describe your project in 1-2 sentences:

User: A REST API for managing user subscriptions and payments.

Claude: Excellent! Moving to Phase 2...
```

### Resuming After Restart

```
User: /init

Claude: I found saved progress from 2024-01-15 10:45.
You were on Phase 3: Configuration.

Would you like to:
1. Resume from Phase 3
2. Restart from beginning

User: Resume

Claude: Continuing from Phase 3: Configuration...
```

### Completing Setup

```
Claude: Phase 10: Verifying Setup Checklist

Checking all items...

[x] Vault MCP installed and working - Verified
[x] All required secrets configured - 3/3 secrets readable
[x] CI MCP installed and working - GitHub API connected
[x] CI config ready - .github/workflows/ci.yml exists
[x] Setup CI issue created - Issue #1 created
[x] Setup CI issue crunched - Issue #1 at ready-to-merge

All checklist items complete!

Project initialization finished. You're ready to start development!
```

## Key Behaviors

- **Planning Mode** - Skill enters planning mode at start for careful execution
- **Resumable** - Progress saved after each phase in `.claude/init-progress.txt`
- **Interactive** - Uses AskUserQuestion for all user inputs
- **Validates** - Checks MCP connections actually work before proceeding
- **Completes Checklist** - Verifies all setup items and updates CLAUDE.md

## Error Handling

| Error                      | Action                            |
| -------------------------- | --------------------------------- |
| Templates not found        | Report error, abort               |
| MCP installation fails     | Offer retry, skip, or help        |
| Secret configuration fails | Mark as pending, continue         |
| Issue creation fails       | Report error, offer retry         |
| Checklist incomplete       | Keep progress file, report status |
