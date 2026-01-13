---
name: review
description: Automated code review for pull requests. This skill is invoked automatically during the crunch workflow at the IMPLEMENTING → VALIDATION transition. Domain-specific reviewers use this skill to perform thorough code review before staging deployment.
---

# Review Pull Request

Perform automated code review on a pull request using multiple specialized agents with confidence-based scoring.

## When This Skill Is Used

This skill is **automatically invoked** during the crunch workflow:

1. **Trigger**: IMPLEMENTING → VALIDATION transition in `/crunch`
2. **Invoker**: Domain-specific reviewer agents (see CLAUDE.md Agent System)
3. **Target**: The PR created at READY-TO-MERGE phase

The skill is NOT typically invoked directly by users. It runs as part of the Code Review phase in the crunch workflow.

## Integration with Crunch Workflow

```
IMPLEMENTING ──► Code Review (this skill) ──► VALIDATION
                     │
                     ├── Domain reviewer (security-analyst, devops-engineer, etc.)
                     └── General reviewer (reviewer agent)
```

See `/crunch` workflow.md for reviewer selection based on issue domain.

## Prerequisites

Before using this skill:

1. Read `CLAUDE.md` for project-specific configuration:
   - **CI Platform** section - determines which MCP server to use
   - **Task Workflow** section - project conventions
   - Any project-specific review guidelines

2. Identify the CI MCP server from CLAUDE.md:
   - GitHub: `mcp__github__*` commands
   - GitLab: `mcp__gitlab__*` commands
   - Gitea: `mcp__gitea__*` commands

## Workflow Phases

### Phase 1: Eligibility Check

Use a fast agent to determine if the PR should be reviewed:

1. Is the PR closed?
2. Is the PR a draft?
3. Is it trivial (automated, obviously simple)?
4. Has it already been reviewed by this tool?

**Action**: If any condition is true, skip review and report why.

### Phase 2: Gather Context

1. Get PR details via CI MCP (title, description, changed files)
2. Find relevant CLAUDE.md files:
   - Root CLAUDE.md (if exists)
   - CLAUDE.md files in directories modified by the PR
3. Summarize the changes for agent context

### Phase 3: Parallel Agent Review

Launch 5 agents in parallel, each independently reviewing changes:

| Agent | Focus |
|-------|-------|
| #1 CLAUDE.md Compliance | Audit changes against project guidelines |
| #2 Bug Scanner | Scan for obvious bugs in changes only |
| #3 Git History Analyzer | Review blame/history for context |
| #4 Previous PR Reviewer | Check prior PRs for applicable comments |
| #5 Code Comment Checker | Verify compliance with inline comments |

Each agent returns issues with descriptions and reasons.

### Phase 4: Score & Filter Issues

For each issue found, score confidence (0-100):

| Score | Meaning |
|-------|---------|
| 0 | False positive, pre-existing issue |
| 25 | Might be real, stylistic not in CLAUDE.md |
| 50 | Real but minor, nitpick |
| 75 | Verified real, important or in CLAUDE.md |
| 100 | Absolutely certain, frequent, confirmed |

**Filter**: Remove issues with score < 80.

### Phase 5: Post Comment

1. Re-verify PR is still open and not yet reviewed
2. Format issues with code links
3. Post comment via CI MCP
4. Report completion to user

## CI Platform Abstraction

The skill works with any CI platform configured in CLAUDE.md.

### Common Operations

| Operation | What to do |
|-----------|------------|
| Get PR | Use CI MCP to fetch PR by number |
| Get PR diff | Use CI MCP to get changed files and lines |
| List comments | Use CI MCP to check for existing reviews |
| Post comment | Use CI MCP to add review comment |

## Key Behaviors

- **Platform agnostic** - Uses CI MCP from CLAUDE.md
- **High signal, low noise** - Only reports issues with 80+ confidence
- **Parallel execution** - 5 agents review simultaneously
- **Avoids false positives** - See workflow.md for filter criteria
- **Code links included** - Each issue links to specific lines
- **Reference CLAUDE.md** - Project guidelines inform review
- **Progress tracking** - Use TodoWrite to track phases

## Error Handling

| Error | Action |
|-------|--------|
| PR not found | Report error, abort |
| PR is closed | Skip review, report status |
| PR is draft | Skip review, report status |
| Already reviewed | Skip review, report existing comment |
| No issues found | Post "no issues" comment |
| CI MCP unavailable | Report error, suggest checking CLAUDE.md |

## Example Usage

### Example 1: Security issue code review (in crunch workflow)
```
Context: /crunch 15 transitioning from IMPLEMENTING → VALIDATION
Issue #15: "Fix authentication bypass vulnerability"
Domain: security

Crunch workflow:
1. Implementation complete, CI passing
2. Invokes review skill for code review phase
3. Reviewers: security-analyst (primary) + reviewer (secondary)

Review skill execution:
1. Reads CLAUDE.md to identify CI platform (GitHub)
2. Gets PR for issue #15's branch
3. Checks eligibility - PR is open, not reviewed
4. Gathers CLAUDE.md from root and src/auth/
5. Launches 5 parallel review agents
6. Scores 8 issues, 3 pass 80+ threshold
7. Posts review comment with 3 issues
8. Returns to crunch: "Review posted, 3 issues found"

Crunch workflow continues:
9. Dev agent addresses feedback
10. Re-runs review skill
11. No issues remain → proceeds to staging deployment
```

### Example 2: Infrastructure feature review
```
Context: /crunch 20 at code review phase
Issue #20: "Add Prometheus metrics to payment service"
Domain: infra

Reviewers: devops-engineer (primary) + reviewer (secondary)

Review skill execution:
1. Reads CLAUDE.md, identifies GitLab platform
2. Fetches MR for issue #20's branch
3. Eligibility check passes
4. Gathers CLAUDE.md files including observability section
5. Parallel agents review with focus on operational concerns
6. 2 issues found (missing metric labels, no dashboard)
7. Posts review comment
8. Returns to crunch for dev to address
```

### Example 3: Clean code review
```
Context: /crunch 28 at code review phase
Issue #28: "Add dark mode toggle"
Domain: frontend

Reviewers: dev-react (primary) + reviewer (secondary)

Review skill execution:
1. Reads CLAUDE.md, identifies CI platform
2. Gets PR for current branch
3. Launches parallel review agents
4. No issues pass 80+ confidence threshold
5. Posts: "No issues found. Checked for bugs and CLAUDE.md compliance."
6. Returns to crunch → proceeds to VALIDATION
```

### Example 4: Review loop with fixes
```
Context: /crunch 33 at code review phase
Issue #33: "Optimize database queries"
Domain: general

Review Round 1:
1. Review skill finds 2 issues (N+1 query, missing index)
2. Posts comment, returns to crunch
3. Dev agent addresses issues
4. Commits fixes

Review Round 2:
1. Crunch re-invokes review skill
2. Checks eligibility - already reviewed? No (new commits since last review)
3. Re-runs parallel agents on updated diff
4. No issues found
5. Proceeds to VALIDATION phase
```
