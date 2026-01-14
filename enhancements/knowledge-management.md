# Knowledge Management System - Detailed Design

## Overview

The Knowledge Management system transforms claude-crunch from a stateless workflow executor into a learning system that accumulates institutional knowledge and applies it to future work.

---

## What Changes

### New Components

| Component | Type | Purpose |
|-----------|------|---------|
| `knowledge-manager` | Agent | Captures, retrieves, analyzes, and synthesizes knowledge |
| `/learn` | Skill | Manual and automatic knowledge capture |
| `/knowledge` | Skill | Query and retrieve past decisions, solutions, patterns |
| `/analyze` | Skill | Pattern detection and reporting |
| `.claude/knowledge/` | Storage | Structured knowledge base |

### Modified Components

| Component | Changes |
|-----------|---------|
| `/crunch` workflow | Add knowledge injection at ENRICH, capture at DONE |
| `/review` workflow | Add feedback capture after review, pattern promotion |
| Agent selection | Weight by effectiveness data from knowledge base |

---

## Architecture

```
                                    ┌──────────────────┐
                                    │  /learn          │
                                    │  /knowledge      │
                                    │  /analyze        │
                                    └────────┬─────────┘
                                             │
                                             ▼
┌─────────────────┐              ┌──────────────────────┐
│ /crunch         │◄────────────►│  knowledge-manager   │
│ /review         │   inject/    │  agent               │
└─────────────────┘   capture    └──────────┬───────────┘
                                            │
                                            ▼
                                 ┌──────────────────────┐
                                 │ .claude/knowledge/   │
                                 ├──────────────────────┤
                                 │ ├── index.json       │
                                 │ ├── decisions/       │
                                 │ ├── resolutions/     │
                                 │ ├── patterns/        │
                                 │ └── feedback/        │
                                 └──────────────────────┘
```

---

## New Agent: knowledge-manager

**File**: `agents/knowledge-manager.md`

### Responsibilities

| Responsibility | Description |
|----------------|-------------|
| **Capture** | Extract insights from issues, reviews, decisions |
| **Retrieve** | Search semantically, rank by relevance, synthesize briefs |
| **Analyze** | Identify patterns, track effectiveness, surface debt |
| **Inject** | Prepare context briefs for other agents |

### When Invoked

| Trigger | Action |
|---------|--------|
| ENRICH entry | Query for related past issues and ADRs |
| ENRICH exit (spec created) | Capture architectural decision |
| IMPLEMENTING entry | Query bug patterns for affected files |
| Code review complete | Capture high-confidence feedback |
| DONE transition | Capture full resolution |
| `/learn` | Manual capture |
| `/knowledge` | Query and retrieval |
| `/analyze` | Pattern analysis |

### Interaction with Other Agents

```
knowledge-manager
       │
       ├── PROVIDES context to ──► system-architect, security-analyst,
       │                          dev-*, reviewer, devops-engineer
       │
       ├── RECEIVES from ────────► All agents (outcomes, decisions, findings)
       │
       └── REPORTS to ───────────► User (queries, analysis)
```

---

## New Skills

### /learn - Knowledge Capture

**Directory**: `skills/learn/`

#### Syntax
```
/learn [options]
```

| Parameter | Description |
|-----------|-------------|
| `-i, --issue` | Issue number to capture from |
| `-t, --type` | Type: `decision`, `resolution`, `pattern`, `feedback` |
| `--from-review` | Capture from most recent code review |
| `-c, --content` | Explicit content to capture |
| `--tags` | Comma-separated tags |

#### Modes

| Mode | Example | Use Case |
|------|---------|----------|
| From issue | `/learn -i 42` | Capture resolution of completed issue |
| From review | `/learn --from-review` | Capture patterns from code review |
| Explicit | `/learn -t decision -c "Use PostgreSQL..."` | Manual decision |
| Interactive | `/learn` | Guided wizard |

---

### /knowledge - Query & Retrieval

**Directory**: `skills/knowledge/`

#### Syntax
```
/knowledge [query] [options]
```

| Parameter | Description |
|-----------|-------------|
| `-q, --query` | Search query (semantic) |
| `-t, --type` | Filter: `decision`, `resolution`, `pattern`, `feedback`, `all` |
| `-d, --domain` | Filter by domain |
| `--since` | Only entries after date |
| `-l, --limit` | Max results (default: 5) |
| `-r, --related` | Find entries related to issue or ID |
| `--brief` | Generate synthesized context brief |

#### Example Output

```markdown
## Knowledge Search Results

**Query**: "authentication race conditions"
**Found**: 4 entries

---

### 1. [Resolution] KE-resolution-20250114-b7c9 (relevance: 95)
**Issue**: #42 - Race condition in token refresh
**Root Cause**: Concurrent refresh requests
**Solution**: Mutex protection for token operations

---

### 2. [Pattern] KE-pattern-20250114-c8d3 (relevance: 88)
**Name**: Token Operation Mutex
**Occurrences**: 3 issues
```

---

### /analyze - Pattern Detection & Reporting

**Directory**: `skills/analyze/`

#### Report Types

| Report | Description |
|--------|-------------|
| `bugs` | Recurring bug patterns by category, domain, file |
| `feedback` | Review findings approaching pattern threshold |
| `effectiveness` | Agent performance metrics |
| `decisions` | Architectural decision inventory |
| `debt` | Technical debt indicators |
| `summary` | High-level project health |

#### Example: Bug Pattern Report

```markdown
## Bug Pattern Analysis

**Period**: Last 90 days | **Total**: 45 bugs

### By Category
| Category | Count | Avg Resolution | Trend |
|----------|-------|----------------|-------|
| Concurrency | 8 | 4.2h | ↑ increasing |
| Validation | 12 | 1.8h | → stable |

### Hotspot Files
1. `src/auth/tokenService.ts` - 4 bugs (3 concurrency)

### Recommendations
- Consider architectural review of tokenService.ts
- Add mutex abstraction for auth operations
```

---

## Storage Structure

```
.claude/knowledge/
├── index.json              # Master index for fast querying
├── config.json             # System configuration
├── decisions/              # Architectural decisions
│   └── KE-decision-*.json
├── resolutions/            # Issue resolutions
│   └── KE-resolution-*.json
├── patterns/               # Patterns and anti-patterns
│   └── KE-pattern-*.json
├── feedback/               # Review feedback
│   └── KE-feedback-*.json
└── reports/                # Generated reports
    └── summary-*.json
```

### Entry ID Format
```
KE-{type}-{YYYYMMDD}-{hash}

Examples:
- KE-decision-20250114-a3f2
- KE-resolution-20250114-b7c9
- KE-pattern-20250114-c8d3
```

---

## Knowledge Types

### 1. Issue Resolutions

**Captured at**: DONE transition (automatic)

```json
{
  "id": "KE-resolution-20250114-b7c9",
  "type": "resolution",
  "content": {
    "issue_title": "Race condition in token refresh causes logout",
    "problem": "Users randomly logged out when multiple tabs open",
    "root_cause": {
      "category": "concurrency",
      "description": "Concurrent refresh requests race condition",
      "code_location": "src/auth/tokenService.ts:145"
    },
    "solution": {
      "approach": "Implement request deduplication with mutex",
      "code_example": "await this.refreshLock.runExclusive(() => ...)"
    },
    "files_changed": ["src/auth/tokenService.ts"],
    "time_to_resolve": "4h",
    "agents_used": ["security-analyst", "dev-react"]
  }
}
```

### 2. Architectural Decisions

**Captured at**: ENRICH exit when spec created (automatic)

```json
{
  "id": "KE-decision-20250114-a3f2",
  "type": "decision",
  "content": {
    "title": "Use refresh token rotation for session management",
    "decision": "Implement sliding-window refresh token rotation",
    "rationale": "Balances security with UX",
    "alternatives": [
      {"option": "Long-lived token", "rejected_because": "Security risk"}
    ],
    "consequences": {
      "positive": ["Improved security"],
      "negative": ["Implementation complexity"]
    }
  }
}
```

### 3. Patterns

**Captured at**: Promoted from feedback (3+ similar findings)

```json
{
  "id": "KE-pattern-20250114-c8d3",
  "type": "pattern",
  "content": {
    "name": "Token Operation Mutex",
    "pattern_type": "pattern",
    "description": "All token operations should be mutex-protected",
    "when_to_use": "Any code modifying token state",
    "example": {
      "language": "typescript",
      "code": "await this.refreshLock.runExclusive(() => ...)"
    },
    "occurrences": 3
  }
}
```

### 4. Review Feedback

**Captured at**: After code review (findings with score >= 80)

```json
{
  "id": "KE-feedback-20250114-d9e4",
  "type": "feedback",
  "content": {
    "finding": "Missing mutex protection for token refresh",
    "severity": "HIGH",
    "category": "concurrency",
    "file_context": {
      "file": "src/auth/tokenService.ts",
      "line_start": 142
    },
    "recommendation": "Wrap in mutex",
    "similar_findings_count": 2,
    "may_become_pattern": true
  }
}
```

---

## Workflow Integration

### Injection Points (knowledge → agents)

| State | What's Injected |
|-------|-----------------|
| **ENRICH entry** | Context brief: related issues, ADRs, patterns, agent recommendations |
| **IMPLEMENTING entry** | Bug patterns for affected files, warnings |
| **DOCS entry** | Documentation patterns |

### Context Brief Format

```markdown
## Knowledge Context

### Related Past Issues
- **#32** (relevance: 92): Similar auth bug - mutex solution
- **#18** (relevance: 78): Login flow rework - see ADR-003

### Relevant Decisions
- **ADR-003**: Use refresh token rotation

### Known Patterns
- **PATTERN**: Token Operation Mutex - all token ops need protection
- **ANTI-PATTERN**: Direct DB calls in auth handlers

### Agent Effectiveness
- security-analyst: 94% on auth issues (23 issues)
```

### Capture Points (workflow → knowledge)

| Trigger | What's Captured |
|---------|-----------------|
| **ENRICH exit** | Decision (if spec created) |
| **Code review** | Feedback (score >= 80) |
| **DONE** | Full resolution |
| **Feedback count >= 3** | Promote to pattern |

---

## Modified Files

### skills/crunch/workflow.md

Add new section "Knowledge Integration":

```markdown
## Knowledge Integration

### On ENRICH Entry
1. Invoke knowledge-manager with issue details
2. Generate context brief
3. Inject into agent prompts

### On ENRICH Exit
1. If spec created, extract decision
2. Store as KE-decision-*
3. Link to issue

### On DONE Transition
1. Capture full resolution
2. Extract root cause, solution, metrics
3. Store as KE-resolution-*
4. Link to related entries
```

### skills/review/workflow.md

Add after Phase 5:

```markdown
## Knowledge Capture from Review

### After Review Complete
FOR each finding with confidence >= 80:
  1. Create KE-feedback-* entry
  2. Check pattern promotion threshold

IF similar findings >= 3:
  1. Promote to KE-pattern-*
  2. Link feedback entries
```

### Dynamic Agent Selection

Modify to include effectiveness weighting:

```markdown
### Knowledge-Aware Selection

1. Detect domains from issue
2. Query agent effectiveness for domain
3. Weight selection by past success rate
4. Include effectiveness in recommendations

Example:
  Security domain → security-analyst (94% on 23 issues)
  Suggest: Also include dev-react for frontend auth (87%)
```

---

## Configuration

**File**: `.claude/knowledge/config.json`

```json
{
  "version": "1.0",
  "decay": {
    "rate": 0.01,
    "min_score": 0.3,
    "evergreen_tags": ["architecture", "security", "core"]
  },
  "capture": {
    "auto_capture_resolutions": true,
    "auto_capture_decisions": true,
    "feedback_confidence_threshold": 80,
    "pattern_promotion_threshold": 3
  },
  "search": {
    "default_limit": 5,
    "similarity_threshold": 0.7,
    "boost_recent": true
  }
}
```

---

## Implementation Order

| Phase | Deliverable | Files |
|-------|-------------|-------|
| 1 | Storage structure | `.claude/knowledge/`, schemas |
| 2 | knowledge-manager agent | `agents/knowledge-manager.md` |
| 3 | /learn skill | `skills/learn/SKILL.md`, `workflow.md` |
| 4 | /knowledge skill | `skills/knowledge/SKILL.md`, `workflow.md` |
| 5 | Workflow integration | Modify `crunch/workflow.md`, `review/workflow.md` |
| 6 | /analyze skill | `skills/analyze/SKILL.md`, `workflow.md` |

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Recurring issue reduction | 50% within 6 months |
| First-time resolution rate | 80%+ |
| Review cycles reduction | 40% |
| Pattern coverage | 80% of issues match documented patterns |
