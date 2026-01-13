# Review Workflow

Detailed instructions for performing automated code review. Reference CLAUDE.md for:
- CI Platform configuration (CI/CD section)
- Project-specific review guidelines
- Task types and conventions
- Agent System (for reviewer selection)

---

## Integration with Crunch Workflow

This skill is automatically invoked during the `/crunch` workflow at the **Code Review** phase (IMPLEMENTING → VALIDATION transition).

### Invocation Context

When crunch reaches the Code Review phase:

1. **Crunch identifies reviewers** based on issue domain (see crunch workflow.md "Dynamic Agent Selection")
2. **Crunch invokes this skill** with the PR number
3. **This skill performs review** using 5 parallel agents
4. **Returns result to crunch**:
   - If issues found: crunch loops dev agent to fix, then re-runs review
   - If no issues: crunch proceeds to VALIDATION phase

### Reviewer Selection (from crunch)

| Domain | Primary Reviewer | Secondary Reviewer |
|--------|------------------|-------------------|
| Security | security-analyst | reviewer |
| Infrastructure | devops-engineer | reviewer |
| Frontend | dev-react | reviewer |
| Python | dev-python | reviewer |
| C++ | dev-cpp | reviewer |
| Go | dev-go | reviewer |
| General | reviewer | - |

The domain-specific reviewer provides expertise; this skill provides systematic multi-agent analysis.

### Review Loop

```
IMPLEMENTING (complete)
       │
       ▼
┌─► Code Review (this skill)
│      │
│      ├── Issues found ──► Dev agent fixes ──► Re-commit ──┐
│      │                                                     │
│      └── No issues ──► VALIDATION                          │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

The loop continues until review passes (no high-confidence issues).

---

## Review Process Overview

```
ELIGIBILITY CHECK ──► Skip if closed/draft/trivial/reviewed
        │
        ▼
GATHER CONTEXT ────► CLAUDE.md files, PR summary
        │
        ▼
PARALLEL REVIEW ───► 5 agents review independently
        │
        ▼
SCORE ISSUES ──────► 0-100 confidence per issue
        │
        ▼
FILTER ────────────► Keep only 80+ confidence
        │
        ▼
RE-VERIFY ─────────► Double-check eligibility
        │
        ▼
POST COMMENT ──────► Format and post via CI MCP
```

---

## Phase 1: Eligibility Check

Use a fast agent (Haiku) to determine if review should proceed.

### Check Criteria

1. **Is PR closed?**
   - Fetch PR state via CI MCP
   - If merged or closed, skip review

2. **Is PR a draft?**
   - Check draft status via CI MCP
   - Draft PRs are work-in-progress, skip review

3. **Is it trivial?**
   - Automated PRs (bot authors like dependabot, renovate)
   - Single-file changes to non-code files (README, changelog)
   - Version bump only commits

4. **Already reviewed?**
   - List PR comments via CI MCP
   - Search for signature: "Generated with [Claude Code]"
   - If found, skip to avoid duplicate reviews

### Actions

- If any check fails: Report reason and stop
- If all checks pass: Proceed to Phase 2

---

## Phase 2: Gather Context

### Step 2a: Get PR Details

Use CI MCP to fetch:
- PR number, title, description
- Author information
- Base and head branches
- List of changed files
- Full diff content

### Step 2b: Find CLAUDE.md Files

Use a fast agent (Haiku) to locate relevant guideline files:

1. Check for root `CLAUDE.md`
2. For each directory containing changed files:
   - Check for `CLAUDE.md` in that directory
   - Check parent directories up to repo root
3. Return list of file paths

### Step 2c: Read Guidelines

Read all found CLAUDE.md files and extract:
- Coding standards
- Review requirements
- Project-specific rules
- Any sections relevant to changed files

### Step 2d: Summarize Changes

Use a fast agent (Haiku) to create a brief summary:
- What is the PR doing?
- What areas of code are affected?
- What is the scope (small fix, large feature, refactor)?

---

## Phase 3: Parallel Agent Review

Launch 5 agents simultaneously using Task tool. Each agent reviews independently and returns a list of issues.

### Agent #1: CLAUDE.md Compliance

**Model**: Sonnet

**Prompt**:
```
Review the following PR changes for compliance with the project's CLAUDE.md guidelines.

CLAUDE.md contents:
{claude_md_content}

PR diff:
{pr_diff}

For each violation found, return:
- Description of the issue
- Which CLAUDE.md guideline it violates (quote the relevant text)
- File path and line numbers

Notes:
- Not all CLAUDE.md instructions apply during code review
- Focus on coding standards, conventions, and explicit requirements
- Ignore guidelines about development process (those are for implementation)
```

### Agent #2: Bug Scanner

**Model**: Sonnet

**Prompt**:
```
Scan the following PR changes for obvious bugs.

PR diff:
{pr_diff}

Focus on:
- Logic errors
- Off-by-one errors
- Null/undefined handling issues
- Resource leaks
- Race conditions
- Incorrect error handling
- Security vulnerabilities

For each bug found, return:
- Description of the bug
- Why it's a bug (what could go wrong)
- File path and line numbers

IMPORTANT:
- Only report bugs in the changed lines
- Prioritize large bugs, avoid nitpicks
- Skip issues that linters/typecheckers will catch
- Skip code that looks buggy but works correctly
- Skip pre-existing bugs not introduced by this PR
```

### Agent #3: Git History Analyzer

**Model**: Sonnet

**Prompt**:
```
Analyze the git history context for the files changed in this PR.

Changed files:
{changed_files}

PR diff:
{pr_diff}

Tasks:
1. Run git blame on modified sections
2. Check commit history for patterns
3. Look for related previous changes
4. Identify if changes contradict recent work

For each issue found, return:
- Description of the concern
- Historical context that raises the concern
- File path and line numbers

Notes:
- Focus on context that reveals potential problems
- Skip if history provides no relevant insights
```

### Agent #4: Previous PR Reviewer

**Model**: Sonnet

**Prompt**:
```
Check previous PRs that touched the same files for applicable review comments.

Changed files:
{changed_files}

Current PR diff:
{pr_diff}

Tasks:
1. Use CI MCP to find recent merged PRs touching these files
2. Read review comments on those PRs
3. Check if any past feedback applies to current changes

For each applicable finding, return:
- The original review comment (summarized)
- How it applies to current changes
- File path and line numbers

Notes:
- Only report if past feedback directly applies
- Skip generic or resolved feedback
```

### Agent #5: Code Comment Checker

**Model**: Sonnet

**Prompt**:
```
Verify that changes comply with inline code comments in the modified files.

PR diff:
{pr_diff}

Tasks:
1. Read the full content of modified files
2. Find inline comments (TODO, FIXME, NOTE, IMPORTANT, etc.)
3. Check if changes violate any inline guidance
4. Check if changes should address TODOs they're near

For each issue found, return:
- The relevant inline comment
- How the change relates to it
- File path and line numbers

Notes:
- Focus on comments that provide guidance or warnings
- Skip standard documentation comments
```

### Issue Format

Each agent returns issues in this format:
```json
{
  "issues": [
    {
      "description": "Missing null check before accessing user.email",
      "reason": "Bug: user could be undefined when not authenticated",
      "file": "src/auth/handler.ts",
      "start_line": 45,
      "end_line": 47,
      "agent": "bug-scanner"
    }
  ]
}
```

---

## Phase 4: Score & Filter Issues

### Confidence Scoring

For each issue from Phase 3, launch a fast agent (Haiku) to score confidence.

**Scoring Prompt**:
```
Score the confidence level (0-100) for this potential issue:

Issue: {issue_description}
Reason: {issue_reason}
Code context:
{code_snippet}

Scoring guide:
- 0: Not confident. False positive or pre-existing issue not introduced by PR
- 25: Somewhat confident. Might be real but unverified. Stylistic issue not in CLAUDE.md
- 50: Moderately confident. Real issue but may be nitpick. Not important relative to PR
- 75: Highly confident. Verified real issue that happens in practice. Important or in CLAUDE.md
- 100: Absolutely certain. Definite issue, frequent occurrence, direct evidence confirmed

Return only the numeric score (0-100).
```

### Filtering

1. Collect all scored issues
2. Remove issues with score < 80
3. Sort remaining issues by score (highest first)

### False Positive Criteria

Automatically filter out:

- **Pre-existing issues**: Problems that existed before this PR
- **Code that looks buggy but isn't**: Intentional patterns, framework conventions
- **Pedantic nitpicks**: Issues a senior engineer wouldn't mention
- **Linter-catchable issues**: Type errors, style issues, formatting
- **General quality issues**: Unless explicitly required in CLAUDE.md
- **Silenced issues**: Code with lint ignore comments
- **Unchanged lines**: Issues on lines the PR didn't modify

---

## Phase 5: Post Comment

### Step 5a: Re-verify Eligibility

Before posting, repeat eligibility check:
- Is PR still open?
- Was a review posted while we were working?

If either check fails, abort and report.

### Step 5b: Format Comment

#### If Issues Found

```markdown
### Code review

Found {N} issues:

1. {description} (CLAUDE.md says "{guideline}")

   {code_platform_link}

2. {description} (bug due to {reason})

   {code_platform_link}

---
Generated with [Claude Code](https://claude.ai/code)

_If useful, react with a thumbs up. Otherwise, react with a thumbs down._
```

#### If No Issues Found

```markdown
### Code review

No issues found. Checked for bugs and CLAUDE.md compliance.

Generated with [Claude Code](https://claude.ai/code)
```

### Step 5c: Code Link Format

Links must point to the specific lines in the PR's head commit.

**Format**: `{repo_url}/blob/{full_sha}/{file_path}#L{start}-L{end}`

**Requirements**:
- Use full git SHA (not branch names or abbreviated SHA)
- Include at least 1 line of context before and after
- Platform-specific URL format:
  - GitHub: `https://github.com/{owner}/{repo}/blob/{sha}/{path}#L{start}-L{end}`
  - GitLab: `https://gitlab.com/{owner}/{repo}/-/blob/{sha}/{path}#L{start}-{end}`
  - Gitea: `https://{host}/{owner}/{repo}/src/commit/{sha}/{path}#L{start}-L{end}`

### Step 5d: Post via CI MCP

Use the appropriate CI MCP command to add the comment:
- GitHub: `mcp__github__add_pull_request_comment` or equivalent
- GitLab: `mcp__gitlab__create_merge_request_note` or equivalent
- Gitea: `mcp__gitea__add_pull_request_comment` or equivalent

---

## CI MCP Operations Reference

Abstract operations mapped to CI platforms:

| Operation | GitHub MCP | GitLab MCP | Gitea MCP |
|-----------|------------|------------|-----------|
| Get PR/MR | `get_pull_request` | `get_merge_request` | `get_pull_request` |
| Get diff | `get_pull_request_diff` | `get_merge_request_changes` | `get_pull_request_diff` |
| List comments | `list_pull_request_comments` | `list_merge_request_notes` | `list_pull_request_comments` |
| Add comment | `add_pull_request_comment` | `create_merge_request_note` | `add_pull_request_comment` |
| Get file | `get_file_contents` | `get_file` | `get_file_contents` |
| Search PRs | `search_pull_requests` | `list_merge_requests` | `list_pull_requests` |

**Note**: Refer to CLAUDE.md CI Platform section for the specific MCP server configured for the project.

---

## Configuration

### Adjusting Confidence Threshold

The default threshold is 80. To adjust:

1. Lower threshold (e.g., 70): More issues reported, more potential false positives
2. Higher threshold (e.g., 90): Fewer issues, only highest confidence

To change, modify the filter condition in Phase 4:
```
Remove issues with score < {threshold}
```

### Customizing Review Focus

Add or modify agents in Phase 3 for specific needs:
- Security-focused agent for sensitive repos
- Performance analysis for critical paths
- Accessibility checking for frontend
- Documentation quality for public APIs

---

## Troubleshooting

### Review Takes Too Long

- Normal for large PRs - 5 agents run in parallel
- Consider splitting large PRs into smaller ones
- Each agent processes independently for thoroughness

### Too Many False Positives

- Default threshold is 80, already filters most false positives
- Make CLAUDE.md more specific about what matters
- Consider if flagged issues are actually valid

### No Comment Posted

Check if:
- PR was closed during review
- PR already has a review comment from this tool
- All issues scored below 80 (no comment needed if clean)

### Wrong Platform Commands

- Verify CLAUDE.md has correct CI Platform section
- Check MCP server is properly configured
- Refer to platform-specific MCP documentation
