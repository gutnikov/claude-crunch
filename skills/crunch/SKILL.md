---
name: crunch
description: Process issues through development workflow states. Use when user says /crunch, mentions issue processing, workflow transitions, or wants to move an issue through backlog, enrich, ready, implementing, validation, docs, or done states.
---

# Crunch Issue Workflow

Process an issue through its development workflow until reaching a target state.

## Quick Start

```
/crunch {issue_number}
```

Accepts issue number (e.g., `6` or `#6`). Fetches issue, shows current state, asks for target state, then executes transitions.

## Prerequisites

Before using this skill:

1. Read `CLAUDE.md` for project-specific configuration:
   - **CI Platform** section - determines which MCP server to use (GitHub/GitLab/Gitea)
   - **Task Workflow** section - task types and states
   - **Agent System** section - available agents

2. Identify the CI MCP server from CLAUDE.md:
   - GitHub: `mcp__github__*` commands
   - GitLab: `mcp__gitlab__*` commands
   - Gitea: `mcp__gitea__*` commands

## Workflow Phases

### Phase 1: Load Issue

1. Parse issue number from arguments (strip `#` if present)
2. Fetch issue using the project's CI MCP server
3. Extract from labels:
   - Type: `type:bug` or `type:feature` (see CLAUDE.md Task Types)
   - State: `state:*` label or no state label = INPUT (see CLAUDE.md Task States)
   - Domain: `domain:security`, `domain:infra`, etc. (for agent selection)
4. **Detect domain** from keywords if no domain label (see workflow.md)
5. Display summary to user:
   - Issue number and title
   - Current type and state (or "INPUT" if no state)
   - Detected domain(s) and agents that will be used
   - Brief description

### Phase 2: Ask Target State

Use `AskUserQuestion` to ask where to take the issue.

**Valid transitions** - refer to CLAUDE.md State Flow Diagram:

| Current State | Valid Targets |
|---------------|---------------|
| `input` (no label) | backlog |
| `backlog` | enrich, ready, done (close) |
| `enrich` | ready, done (close) |
| `ready` | implementing |
| `implementing` | validation |
| `validation` | docs, implementing (fail) |
| `docs` | ready-to-merge |
| `ready-to-merge` | done |

Always include "Stay here (just review)" option.

**Note**: INPUT phase requires 2-3 turns of negotiation with the requestor to clarify requirements before moving to backlog.

### Phase 3: Execute Workflow

1. Load workflow details from [workflow.md](workflow.md)
2. **Detect issue domain** from labels and keywords (see workflow.md Dynamic Agent Selection)
3. Execute transitions from current state toward target:
   - Follow workflow phases for each transition
   - **Select agents dynamically** based on domain:
     - Security issues → security-analyst + reviewer
     - Infrastructure issues → devops-engineer + reviewer
     - General issues → default agents per phase
   - Use multiple agents when issue spans domains or is high-risk
   - Update issue body with progress (NOT comments)
   - Update labels at each state transition
4. **Stop when target state is reached** - never go beyond
5. Use `TodoWrite` to track progress through phases

### Phase 4: Report Completion

Report to user:
- Final state achieved
- Summary of work done
- Artifacts created (branches, PRs)
- Next steps if applicable

## CI Platform Abstraction

The skill works with any CI platform configured in CLAUDE.md. Use the appropriate MCP commands:

### Common Operations

| Operation | What to do |
|-----------|------------|
| Get issue | Use CI MCP to fetch issue by number |
| Update labels | Use CI MCP to replace issue labels |
| Update body | Use CI MCP to update issue description |
| Close issue | Use CI MCP to change issue state |
| Create PR | Use CI MCP to create pull request |
| Get labels | Use CI MCP to list repository labels |

### Label Management

**Always get label IDs first** from the repository before transitions:
- Labels have different IDs in each repository
- Use the CI MCP "list labels" command to get current IDs
- Use "replace labels" (not "add") for state transitions to ensure single state

## Git Branch Conventions

| Type | Branch Prefix | Example |
|------|---------------|---------|
| Bug | `fix/issue-` | `fix/issue-3-login-crash` |
| Feature | `feature/issue-` | `feature/issue-10-dark-mode` |

Branch created at: `enrich` state (when investigation/spec starts)

## Key Behaviors

- **CI platform is source of truth** - Always fetch current state from labels
- **Issue body updates only** - Never add comments, update the body
- **Stop at target** - Never execute beyond user's target state
- **Dynamic agent selection** - Choose agents based on issue domain (see workflow.md)
- **Multi-agent support** - Use multiple agents for complex or cross-domain issues
- **Reference CLAUDE.md** - For agents, states, types, CI config
- **Progress tracking** - Use TodoWrite to track phases
- **TDD required** - All code must have tests (see CLAUDE.md TDD section)

## Error Handling

| Error | Action |
|-------|--------|
| Issue not found | Report error, abort |
| No type label | Ask user to classify (bug or feature) |
| No state label | Treat as INPUT phase, negotiate with requestor |
| Invalid target | Explain valid options per CLAUDE.md, re-ask |
| Workflow failure | Document progress in issue body, report blocker |

## Example Usage

### Example 1: New issue (INPUT phase)
```
User: /crunch 6

Claude:
1. Reads CLAUDE.md to identify CI platform
2. Fetches issue #6 - no state label (INPUT phase)
3. Shows: "Issue #6 is in INPUT phase"
4. Begins negotiation (2-3 turns):
   Turn 1: "What problem are you solving?"
   Turn 2: "What does success look like?"
   Turn 3: "Confirmed - this is a feature for X"
5. Updates issue body with clarified requirements
6. Adds labels: type:feature, state:backlog
7. Asks: "Continue to next state?"
```

### Example 2: Existing issue
```
User: /crunch 10

Claude:
1. Reads CLAUDE.md to identify CI platform (e.g., GitHub)
2. Fetches issue #10 using mcp__github__get_issue
3. Shows: type:feature, state:backlog
4. Asks: "Where to take this issue?"
5. User selects: "ready"
6. Executes: backlog -> enrich -> ready
   - Creates branch, uses architect agent for spec
   - Creates DoD checklist for staging validation
   - Updates issue body with progress
   - Updates labels at each transition
7. Stops at ready, reports completion
```

### Example 3: Security-related issue
```
User: /crunch 15

Claude:
1. Reads CLAUDE.md to identify CI platform
2. Fetches issue #15 - "Fix authentication bypass vulnerability"
3. Detects domain: security (from keywords: authentication, vulnerability)
4. Shows: type:bug, state:backlog, domain:security
   - Agents: security-analyst (primary), reviewer (secondary)
5. Asks: "Where to take this issue?"
6. User selects: "validation"
7. Executes with security-focused agents:
   - ENRICH: security-analyst investigates vulnerability
   - IMPLEMENTING: dev agent with security-analyst review
   - Code Review: security-analyst + reviewer (both)
   - VALIDATION: security-analyst verifies fix
8. Reports completion with security verification summary
```

### Example 4: Infrastructure issue
```
User: /crunch 20

Claude:
1. Fetches issue #20 - "Add Prometheus metrics to payment service"
2. Detects domain: infra (from keywords: Prometheus, metrics)
3. Shows: type:feature, state:backlog, domain:infra
   - Agents: devops-engineer (primary), architect (secondary)
4. Executes with infra-focused agents:
   - ENRICH: devops-engineer designs metrics + architect reviews
   - Code Review: devops-engineer + reviewer
   - VALIDATION: devops-engineer verifies operational correctness
```
