# Workflow Transitions

Detailed instructions for each state transition. Reference CLAUDE.md for:
- Task types and states (Task Workflow section)
- Available agents (Agent System section)
- TDD requirements (TDD section)
- Observability requirements (Observability section)

---

## Input Modes

The crunch skill supports multiple invocation modes for flexibility.

### Mode 1: Existing Issue (Interactive)

```
/crunch 5
/crunch #5
```

- Fetch issue from CI MCP
- Display summary to user
- Ask target state interactively via `AskUserQuestion`
- Execute workflow transitions

### Mode 2: Existing Issue (Direct)

```
/crunch -i 5 -t ready
/crunch --issue 5 --target validation
```

- Fetch issue from CI MCP
- Use provided target state (skip interactive ask)
- Validate transition is allowed
- Execute workflow transitions

### Mode 3: New Issue (Interactive)

```
/crunch "Add dark mode toggle"
```

- Create new issue via CI MCP with text as title/body
- Run INPUT negotiation (2-3 turns)
- Ask target state after negotiation
- Execute workflow transitions

### Mode 4: New Issue (Non-Interactive)

```
/crunch --input "Fix null pointer in handler" --type bug
/crunch --input "Add metrics endpoint" --type feature -t ready
```

- Create new issue via CI MCP
- Skip INPUT negotiation entirely
- Use provided `--type` or infer from keywords
- Transition directly to BACKLOG (or further if `-t` provided)

### Parameter Parsing

```
Arguments:
  [number]          Issue number (e.g., 5, #5)
  "text"            New issue description (interactive mode)

Options:
  -i, --issue NUM   Issue number to process
  -t, --target ST   Target state (skips interactive ask)
  --input TEXT      New issue text (non-interactive mode)
  --type TYPE       Issue type: bug | feature
```

### Type Inference

When `--type` is not provided with `--input`, infer from keywords:

**Bug indicators**: fix, bug, crash, error, broken, issue, problem, fails, null, exception, undefined

**Feature indicators**: add, implement, create, new, feature, support, enable, introduce, enhance

If ambiguous, default to `feature`.

---

## Git Branch Lifecycle

```
INPUT ────────────────────────────────────────► No branch
   └──► Negotiation with requestor (issue body only)

BACKLOG ──────────────────────────────────────► No branch
   └──► Awaiting triage

ENRICH ───────────────────────────────────────► CREATE BRANCH
   └──► {fix|feature}/issue-{number}-{short-description}
        Investigation, spec, DoD checklist

READY ────────────────────────────────────────► Same branch
   └──► Ready for implementation (no commits here)

IMPLEMENTING ─────────────────────────────────► Same branch
   └──► Implementation commits, test commits

VALIDATION ───────────────────────────────────► Same branch
   └──► Verify DoD checklist on staging

DOCS ─────────────────────────────────────────► Same branch
   └──► Documentation commits

READY-TO-MERGE
   ├──► PUSH: git push -u origin {branch}
   └──► CREATE PR via CI MCP

DONE (after PR merged)
   └──► DELETE BRANCH (optional)
```

---

## INPUT Phase (no state label)

**Purpose**: Process raw input before it becomes a proper backlog item.

This phase has two paths depending on invocation mode.

### Interactive Path (default)

When invoked with bare text or existing issue without state label:

**This is a conversation phase** - do 2-3 turns of negotiation to:
1. Understand what the requestor actually wants
2. Clarify ambiguous requirements
3. Identify missing information
4. Confirm understanding

#### Negotiation Process

**Turn 1: Initial Understanding**
- Read the raw input
- Ask clarifying questions:
  - What problem are you trying to solve?
  - What does success look like?
  - Any constraints or preferences?

**Turn 2: Details**
- Based on answers, dig deeper:
  - Specific behaviors expected?
  - Edge cases to consider?
  - Priority and urgency?

**Turn 3: Confirmation**
- Summarize understanding
- Confirm with requestor
- Identify issue type (bug or feature)

#### Actions on Completion

1. Update issue body with clarified requirements
2. Add type label (`type:bug` or `type:feature`)
3. Add state label (`state:backlog`)

Update issue body:
```markdown
## {Bug|Feature}: {title}

### Problem/Request
{clarified description}

### Expected Behavior
{what should happen}

### Context
{any relevant context from negotiation}

### Priority
{priority level}

**Status**: Ready for Backlog
```

### Non-Interactive Path (`--input`)

When invoked with `--input` flag, skip negotiation entirely:

1. **Use input text as-is** for issue body
2. **Determine type**:
   - Use `--type` if provided
   - Otherwise infer from keywords (see Input Modes section)
3. **Create issue** via CI MCP with:
   - Title: first line or first 80 chars of input
   - Body: full input text
   - Labels: inferred/provided type
4. **Immediately transition to BACKLOG**:
   - Add `state:backlog` label
   - Skip negotiation turns

Update issue body (non-interactive):
```markdown
## {Bug|Feature}: {title}

### Description
{input text as provided}

### Source
Created via non-interactive crunch invocation.

**Status**: Ready for Backlog
```

If `-t` target is also provided, continue transitioning toward target state.

---

## Transition: BACKLOG → ENRICH

**When**: Issue needs investigation (bug) or specification (feature)

**Actions**:
1. Review issue description
2. Create branch: `{fix|feature}/issue-{number}-{description}`
3. Update labels to `state:enrich`
4. Begin Enrich Phase

---

## Transition: BACKLOG → READY

**When**: Simple issue with clear requirements, no investigation needed

**Actions**:
1. Verify issue has clear requirements
2. Create branch: `{fix|feature}/issue-{number}-{description}`
3. Create DoD checklist (see Enrich Phase)
4. Update labels to `state:ready`

**Validation**: Must have clear goal, acceptance criteria, and DoD checklist

---

## Transition: BACKLOG → CLOSE

**When**: Cancelled, duplicate, not valid

**Actions**:
1. Document reason in issue body
2. Update labels to `state:done`
3. Close issue

---

## Enrich Phase (state:enrich)

> **CRITICAL**: The most important output of this phase is the **Definition of Done (DoD) checklist**.
> This checklist will be used during VALIDATION to verify the work on staging before merging to main.

### For Bugs: Investigation

1. **Reproduce** the bug
2. **Root cause analysis** - find where and why
3. **Fix approach** - document proposed fix
4. **Create DoD checklist**

### For Features: Specification

Use planning mode and agents per CLAUDE.md:

1. **Exploration** (planning mode)
   - Explore codebase for patterns
   - Identify constraints
   - Document findings

2. **Specification** (architect agent)
   - Create technical spec
   - Define interfaces
   - List open questions

3. **Negotiation** (architect ↔ dev agent)
   - Dev agent reviews spec
   - Architect clarifies/updates
   - Repeat until converged

4. **Create DoD checklist**

### Definition of Done (DoD) Checklist

> **This is mandatory.** The DoD checklist defines what must be verified on staging before the work can be merged to main and deployed to production.

The checklist must include:

**Infrastructure Checks**:
- [ ] App on staging is running successfully
- [ ] No suspicious error logs (check log aggregator)
- [ ] Metrics are healthy (check dashboards)
- [ ] Alerts are not firing

**Functional Checks**:
- [ ] The requested functionality works as expected
- [ ] {Specific behavior 1 from requirements}
- [ ] {Specific behavior 2 from requirements}
- [ ] Edge cases handled correctly

**Regression Checks**:
- [ ] Existing functionality still works
- [ ] No performance degradation
- [ ] Related features unaffected

Update issue body:
```markdown
## {Bug|Feature}: {title}

### Summary
{description}

### Investigation/Specification
{findings or spec details}

### Implementation Notes
- Files to create: {list}
- Files to modify: {list}

### Definition of Done (Staging Validation)

**Infrastructure**:
- [ ] App on staging is running successfully
- [ ] No suspicious error logs
- [ ] Metrics and dashboards are healthy
- [ ] No alerts firing

**Functional**:
- [ ] {Specific check 1}
- [ ] {Specific check 2}
- [ ] {Specific check 3}

**Regression**:
- [ ] {Regression check 1}
- [ ] {Regression check 2}

**Status**: Enrich Complete
```

---

## Transition: ENRICH → READY

**When**: Investigation/specification complete AND DoD checklist created

**Actions**:
1. Verify DoD checklist exists in issue body
2. Commit spec/investigation:
   ```bash
   git commit -m "{Spec|Investigation}: {description}

   Issue: #{issue_number}"
   ```
3. Update labels to `state:ready`

---

## Transition: READY → IMPLEMENTING

**When**: Implementation begins

### Planning (planning mode)

1. Enter planning mode
2. Review spec/investigation
3. Design implementation steps
4. Get user approval

### Implementation (dev agent per CLAUDE.md)

1. Update labels to `state:implementing`
2. **Write tests first** (TDD per CLAUDE.md)
3. Implement changes
4. Ensure tests pass
5. Run CI checks

Update issue body:
```markdown
### Implementation

**Branch**: `{branch}`
**Changes**:
- `{file}` - {description}

**Tests**:
- `{test_file}` - {description}

**CI**: {status}

**Status**: Implementation Complete
```

---

## Transition: IMPLEMENTING → VALIDATION

**When**: Implementation complete, CI passing

### Code Review (reviewer agent per CLAUDE.md)

Review loop until approved:
1. Reviewer identifies issues
2. Dev agent addresses feedback
3. Re-review

### Deploy to Staging

1. Deploy to staging environment (per CLAUDE.md Deployment section)
2. Update labels to `state:validation`

Update issue body:
```markdown
### Review

**Round {N}**: {APPROVED|CHANGES_REQUESTED}
**Issues**: {resolved/total}

### Deployed to Staging
**Version**: {commit/tag}
**Time**: {timestamp}
```

---

## Validation Phase (state:validation)

**Purpose**: Verify the DoD checklist on staging before proceeding.

### Execute DoD Checklist

Go through each item in the DoD checklist:

1. **Infrastructure Checks**
   - Verify app is running (health endpoints)
   - Check logs for errors (per CLAUDE.md Observability section)
   - Review metrics dashboards
   - Confirm no alerts firing

2. **Functional Checks**
   - Test each specific behavior from requirements
   - Verify edge cases work correctly
   - Confirm the task does what was requested

3. **Regression Checks**
   - Test related functionality
   - Verify no performance issues

### Update Issue Body

Check off items as verified:
```markdown
### Validation Status

**Infrastructure**:
- [x] App on staging is running successfully
- [x] No suspicious error logs
- [x] Metrics and dashboards are healthy
- [x] No alerts firing

**Functional**:
- [x] {Specific check 1}
- [ ] {Specific check 2} ← FAILED: {reason}

**Status**: {Validation Passed|Validation Failed}
```

---

## Transition: VALIDATION → IMPLEMENTING (Fail)

**When**: Any DoD checklist item fails

**Actions**:
1. Document what failed in issue body
2. Update labels back to `state:implementing`
3. Fix and repeat

---

## Transition: VALIDATION → DOCS

**When**: All DoD checklist items pass

**Actions** (techwriter agent per CLAUDE.md):
1. Update documentation as needed
2. Add changelog entry
3. Update labels to `state:docs`

---

## Transition: DOCS → READY-TO-MERGE

**When**: Documentation complete

**Actions**:
1. Push branch: `git push -u origin {branch}`
2. Create PR via CI MCP:
   - Title: `{Fix|Feat} #{issue_number}: {description}`
   - Body: Summary, changes, DoD validation results
3. Update labels to `state:ready-to-merge`

---

## Transition: READY-TO-MERGE → DONE

**When**: PR approved and merged

**Actions**:
1. Merge PR via CI MCP
2. Update labels to `state:done`
3. Close issue
4. Delete branch (optional)

Update issue body:
```markdown
## Resolution: {Fixed|Implemented}

**Summary**: {description}
**PR**: #{pr_number}
**Merged**: {date}

### DoD Validation
All checks passed on staging.
```

---

## Dynamic Agent Selection

The crunch skill dynamically selects appropriate agents based on issue context. This enables specialized handling for security issues, infrastructure changes, and other domain-specific work.

### Domain Detection

Detect the issue domain from labels and content keywords:

| Domain | Labels | Keywords in Title/Body |
|--------|--------|------------------------|
| Security | `domain:security`, `security` | auth, login, password, token, JWT, OAuth, XSS, injection, vulnerability, CVE, encrypt, permission, access control |
| Infrastructure | `domain:infra`, `infra`, `devops` | deploy, CI/CD, pipeline, docker, kubernetes, k8s, terraform, monitoring, alert, nginx, server, AWS, GCP, Azure |
| Performance | `domain:performance`, `performance` | slow, latency, optimize, cache, memory, CPU, bottleneck, profiling |
| API | `domain:api`, `api` | endpoint, REST, GraphQL, rate limit, API key, webhook |
| Database | `domain:database`, `database` | query, migration, schema, index, PostgreSQL, MySQL, Redis |
| Frontend | `domain:frontend`, `frontend` | React, component, hook, UI, CSS, styled, tailwind, form, modal, TypeScript, tsx, jsx |
| Python | `domain:python`, `python` | Python, Django, Flask, FastAPI, pandas, numpy, pytest, pip, venv, poetry |
| C++ | `domain:cpp`, `cpp` | C++, STL, template, RAII, smart pointer, unique_ptr, shared_ptr, CMake, Boost, memory |
| Go | `domain:go`, `go`, `golang` | Go, goroutine, channel, gin, cobra, gRPC, go mod, defer, interface{} |

### Agent Capabilities

| Agent | Primary Expertise | Use For |
|-------|-------------------|---------|
| `system-architect` | System design, architecture decisions | Specification, complex feature design |
| `security-analyst` | Security review, threat modeling | Security-domain issues, auth changes, data handling |
| `devops-engineer` | Infrastructure, deployment, monitoring | Infra-domain issues, CI/CD, observability |
| `reviewer` | Code quality, best practices | General code review, quality gates |
| `dev-react` | React, TypeScript, frontend patterns | Frontend-domain issues, UI components, hooks |
| `dev-python` | Python, Django, Flask, data processing | Python-domain issues, backend services, scripts |
| `dev-go` | Go, concurrency, CLI tools | Go-domain issues, services, tooling |
| `dev-cpp` | C++, performance, memory management | C++-domain issues, systems programming |
| `techwriter` | Documentation | Documentation phase |

### Phase-to-Agent Mapping

Select agents dynamically based on detected domain:

#### Specification Phase (ENRICH)

| Domain | Primary Agent | Secondary Agent |
|--------|---------------|-----------------|
| Security | security-analyst | system-architect |
| Infrastructure | devops-engineer | system-architect |
| Frontend | dev-react | system-architect |
| Python | dev-python | system-architect |
| C++ | dev-cpp | system-architect |
| Go | dev-go | system-architect |
| General | system-architect | - |

#### Implementation Phase

| Domain | Primary Agent |
|--------|---------------|
| Frontend | dev-react |
| Python | dev-python |
| Go | dev-go |
| C++ | dev-cpp |
| General | (detect from file extensions) |

#### Code Review Phase (IMPLEMENTING → VALIDATION)

| Domain | Primary Reviewer | Secondary Reviewer |
|--------|------------------|-------------------|
| Security | security-analyst | reviewer |
| Infrastructure | devops-engineer | reviewer |
| Frontend | dev-react | reviewer |
| Python | dev-python | reviewer |
| C++ | dev-cpp | reviewer |
| Go | dev-go | reviewer |
| Performance | reviewer | (profiling tools) |
| General | reviewer | - |

#### Validation Phase

| Domain | Validator |
|--------|-----------|
| Security | security-analyst (penetration testing mindset) |
| Infrastructure | devops-engineer (operational validation) |
| Frontend | dev-react (UI/UX validation) |
| Python | dev-python (functional validation) |
| C++ | dev-cpp (memory/performance validation) |
| Go | dev-go (concurrency validation) |
| General | reviewer |

### Multi-Agent Workflows

Some phases benefit from multiple agents working together:

**Sequential (one after another)**:
```
Specification: architect creates spec → security-analyst reviews for security
Code Review: reviewer for quality → security-analyst for security
```

**Parallel (independent reviews)**:
```
Complex feature: architect + security-analyst + devops-engineer (all review spec)
Critical change: reviewer + security-analyst (both review code)
```

### Selection Algorithm

When entering a phase that uses agents:

1. **Detect domains** from issue labels and keywords
2. **Select primary agent** based on domain (see tables above)
3. **Select secondary agents** if:
   - Issue spans multiple domains
   - Phase benefits from multiple perspectives
   - Issue is high-risk or critical
4. **Run agents** sequentially or in parallel as appropriate
5. **Synthesize feedback** and proceed

### Asking User for Agent Preference

For ambiguous cases, use `AskUserQuestion`:

```
Which reviewers should analyze this change?

Options:
- [ ] General reviewer (code quality, best practices)
- [ ] Security analyst (vulnerability assessment)
- [ ] DevOps engineer (infrastructure, deployment)
- [ ] All applicable (comprehensive review)
```

### Examples

**Example 1: Security-related bug fix**
```
Issue: "Fix SQL injection vulnerability in user search"
Labels: type:bug, domain:security

ENRICH phase:
  Primary: security-analyst (vulnerability assessment)
  Secondary: system-architect (if architectural changes needed)

IMPLEMENTING phase:
  Primary: dev-python (or appropriate language agent)

Code Review:
  Primary: security-analyst (verify fix is secure)
  Secondary: reviewer (code quality)
```

**Example 2: Infrastructure feature**
```
Issue: "Add Prometheus metrics to API endpoints"
Labels: type:feature, domain:infra

ENRICH phase:
  Primary: devops-engineer (monitoring design)
  Secondary: system-architect (integration points)

IMPLEMENTING phase:
  Primary: dev-go (or appropriate language agent)

Code Review:
  Primary: devops-engineer (operational correctness)
  Secondary: reviewer (code quality)
```

**Example 3: General feature**
```
Issue: "Add dark mode support"
Labels: type:feature

ENRICH phase:
  Primary: system-architect

IMPLEMENTING phase:
  Primary: dev-react (or appropriate language agent)

Code Review:
  Primary: reviewer
```

---

## Agent Usage Summary (Quick Reference)

| Phase | Default Agent | Security Domain | Infra Domain |
|-------|---------------|-----------------|--------------|
| Input negotiation | (conversation) | (conversation) | (conversation) |
| Exploration | planning mode | + security-analyst | + devops-engineer |
| Specification | architect | + security-analyst | + devops-engineer |
| Spec Review | architect ↔ dev-* | + security-analyst | + devops-engineer |
| Implementation | dev-* (per language) | dev-* | dev-* |
| Code Review | reviewer | + security-analyst | + devops-engineer |
| Validation | reviewer | security-analyst | devops-engineer |
| Documentation | techwriter | techwriter | techwriter |
| Deployment | devops | devops | devops |

**Legend**: `+` means "in addition to default"

---

## Issue Body Templates

### After INPUT (entering Backlog)
```markdown
## {Bug|Feature}: {title}

### Problem/Request
{clarified description}

### Expected Behavior
{what should happen}

### Context
{from negotiation}

**Status**: Backlog
```

### After ENRICH
```markdown
## {Bug|Feature}: {title}

### Summary
{description}

### {Investigation|Specification}
{details}

### Implementation Notes
- Files: {list}

### Definition of Done (Staging Validation)

**Infrastructure**:
- [ ] App running successfully
- [ ] No error logs
- [ ] Metrics healthy
- [ ] No alerts

**Functional**:
- [ ] {check 1}
- [ ] {check 2}

**Regression**:
- [ ] {check 1}

**Status**: Ready
```

### After VALIDATION
```markdown
### Validation Results

**Infrastructure**:
- [x] App running successfully
- [x] No error logs
- [x] Metrics healthy
- [x] No alerts

**Functional**:
- [x] {check 1}
- [x] {check 2}

**Status**: Ready to Merge
```

---

## CI MCP Operations

### Issue Creation (for new issues)

When creating a new issue via `--input` or quoted text:

| Platform | MCP Operation | Example |
|----------|---------------|---------|
| GitHub | `create_issue` | `mcp__github__create_issue` |
| GitLab | `create_issue` | `mcp__gitlab__create_issue` |
| Gitea | `create_issue` | `mcp__gitea__create_issue` |

**Parameters**:
- `title`: First line of input (or first 80 chars)
- `body`: Full input text formatted per template
- `labels`: Type label if known (e.g., `type:bug`, `type:feature`)

### Subagent Invocation Patterns

When this skill is invoked by other agents:

**From review skill** (after code review):
```
/crunch -i {issue_number} -t validation
```

**Creating follow-up issues**:
```
/crunch --input "Follow-up from #{parent}: {description}" --type feature -t ready
```

**Hotfix pipeline**:
```
/crunch --input "Hotfix: {description}" --type bug -t implementing
```

### Non-Interactive Checklist

Before running in non-interactive mode, ensure:
- [ ] Issue number (`-i`) or input text (`--input`) provided
- [ ] Target state (`-t`) provided (skips interactive ask)
- [ ] Type (`--type`) provided or inferable from text
- [ ] CI MCP server is available (check CLAUDE.md)
- [ ] Required labels exist in repository
