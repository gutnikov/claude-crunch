---
name: crunch
description: Process issues through development workflow states. Supports creating new issues, non-interactive mode for automation, and subagent invocation. Use with issue number, input text, or parameters like -i, -t, --input.
---

# Crunch Issue Workflow

Process an issue through its development workflow until reaching a target state.

## Syntax

```
/crunch [options] [issue_number | "input text"]
```

### Parameters

| Parameter | Short | Description |
|-----------|-------|-------------|
| `--issue` | `-i` | Issue number to process |
| `--target` | `-t` | Target state (skips interactive ask) |
| `--input` | | Raw input text for new issue (skips negotiation) |
| `--type` | | Issue type: `bug` or `feature` |

### Invocation Modes

| Mode | Example | Use Case |
|------|---------|----------|
| Existing issue (interactive) | `/crunch 5` | User processing known issue |
| Existing issue (direct) | `/crunch -i 5 -t ready` | Subagent or scripted workflow |
| New issue (negotiate) | `/crunch "Add feature"` | User with rough idea |
| New issue (direct) | `/crunch --input "Fix X"` | Subagent or clear requirement |
| Full automation | `/crunch --input "Fix Y" --type bug -t ready` | Complete non-interactive |

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

### Phase 1: Parse Input & Load/Create Issue

1. **Parse arguments**:
   - If bare number or `-i`: existing issue mode
   - If quoted text (no `--input`): new issue, interactive mode
   - If `--input`: new issue, non-interactive mode

2. **For existing issue**:
   - Fetch issue using the project's CI MCP server
   - Extract from labels:
     - Type: `type:bug` or `type:feature` (see CLAUDE.md Task Types)
     - State: `state:*` label or no state label = INPUT (see CLAUDE.md Task States)
     - Domain: `domain:security`, `domain:infra`, etc. (for agent selection)
   - **Detect domain** from keywords if no domain label (see workflow.md)

3. **For new issue**:
   - Create issue via CI MCP with input text as title/body
   - If `--type` provided: add type label immediately
   - If interactive mode (no `--input`): proceed to INPUT negotiation
   - If non-interactive (`--input`): infer type from text, skip to BACKLOG

4. **Display summary** (interactive modes only):
   - Issue number and title
   - Current type and state (or "INPUT" if no state)
   - Detected domain(s) and agents that will be used
   - Brief description

### Phase 2: Determine Target State

1. **If `--target` provided**: use directly, skip to validation
2. **If interactive mode**: use `AskUserQuestion` with valid transitions
3. **Validation**: ensure target is reachable from current state

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

For interactive mode, always include "Stay here (just review)" option.

**Note**: INPUT phase requires 2-3 turns of negotiation unless `--input` flag is used (non-interactive).

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
- **Stop at target** - Never execute beyond the target state
- **Dynamic agent selection** - Choose agents based on issue domain (see workflow.md)
- **Multi-agent support** - Use multiple agents for complex or cross-domain issues
- **Reference CLAUDE.md** - For agents, states, types, CI config
- **Progress tracking** - Use TodoWrite to track phases
- **TDD required** - All code must have tests (see CLAUDE.md TDD section)
- **Non-interactive support** - With `-t` and `--input`, runs without user interaction

## Subagent Invocation

This skill can be invoked by other agents during workflows, not just by users.

### From Review Skill

After code review completes:
```
/crunch -i 15 -t validation
```

### From Another Crunch Instance

Chaining work across domains:
```
/crunch --input "Follow-up: add tests for feature #15" --type feature -t ready
```

### Non-Interactive Requirements

When invoked by subagents, always use:
- `-i` or `--input` (explicit issue source)
- `-t` (explicit target, no interactive asking)
- `--type` with `--input` if type isn't obvious from text

## Error Handling

| Error | Action |
|-------|--------|
| Issue not found | Report error, abort |
| No type label | Interactive: ask to classify. Non-interactive: infer from text |
| No state label | Interactive: negotiate. Non-interactive: skip to BACKLOG |
| Invalid target | Interactive: re-ask. Non-interactive: report error, abort |
| Workflow failure | Document progress in issue body, report blocker |
| Missing `--type` with `--input` | Attempt inference, or report error if ambiguous |

## Example Usage

### Example 1: Existing issue (interactive)
```
User: /crunch 10

Claude:
1. Reads CLAUDE.md to identify CI platform (e.g., GitHub)
2. Fetches issue #10 using CI MCP
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

### Example 2: Existing issue with target (non-interactive)
```
Subagent: /crunch -i 15 -t validation

Claude:
1. Reads CLAUDE.md to identify CI platform
2. Fetches issue #15 - type:bug, state:implementing
3. Target provided: validation (no interactive ask)
4. Validates transition: implementing -> validation is valid
5. Executes transition:
   - Invokes /review skill for code review
   - Review passes, proceeds to validation
   - Runs DoD checklist verification on staging
6. Returns: "Issue #15 transitioned to VALIDATION"
```

### Example 3: New issue (interactive negotiation)
```
User: /crunch "Add dark mode toggle to settings"

Claude:
1. Reads CLAUDE.md to identify CI platform
2. Creates new issue via CI MCP with title from input
3. No state label → INPUT phase
4. Begins negotiation (2-3 turns):
   Turn 1: "What problem are you solving?"
   Turn 2: "What does success look like?"
   Turn 3: "Confirmed - this is a feature for UI settings"
5. Updates issue body with clarified requirements
6. Adds labels: type:feature, state:backlog
7. Asks: "Where to take this issue?"
8. User selects: "ready"
9. Executes: backlog -> enrich -> ready
```

### Example 4: New issue (non-interactive)
```
Subagent: /crunch --input "Fix null pointer in auth handler" --type bug -t implementing

Claude:
1. Reads CLAUDE.md to identify CI platform
2. Creates new issue via CI MCP
3. --input flag: skip negotiation
4. --type bug: add type:bug label
5. Infers domain: security (from keywords: auth)
6. Transitions: INPUT -> backlog -> enrich -> ready -> implementing
   - Creates branch: fix/issue-42-null-pointer-auth
   - Runs investigation with security-analyst
   - Creates spec and DoD checklist
   - Begins implementation
7. Returns: "Issue #42 created and transitioned to IMPLEMENTING"
```

### Example 5: Security issue with domain agents
```
User: /crunch -i 15 -t validation

Claude:
1. Fetches issue #15 - "Fix authentication bypass vulnerability"
2. Detects domain: security (from keywords)
3. Target: validation (from -t parameter)
4. Executes with security-focused agents:
   - ENRICH: security-analyst investigates vulnerability
   - IMPLEMENTING: dev agent with security-analyst review
   - Code Review: security-analyst + reviewer (via /review skill)
   - VALIDATION: security-analyst verifies fix
5. Reports completion with security verification summary
```

### Example 6: Subagent chaining
```
Context: After completing issue #15, create follow-up

Crunch agent: /crunch --input "Add regression tests for auth fix #15" --type feature -t ready

Claude:
1. Creates issue #16 via CI MCP
2. Non-interactive: skips negotiation
3. Links to parent issue #15 in body
4. Transitions to READY:
   - Creates branch: feature/issue-16-auth-regression-tests
   - Spec: test cases based on #15 vulnerability
   - DoD: test coverage for auth paths
5. Returns: "Issue #16 created, ready for implementation"
```
