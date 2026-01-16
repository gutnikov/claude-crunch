---
name: learn
description: Capture knowledge from the current context or explicit input. Use to record decisions, lessons learned, patterns, or any insight worth preserving for future reference.
---

# Learn - Knowledge Capture

Capture and store knowledge from the current session, issue, or explicit user input.

## Syntax

```
/learn [options]
```

### Parameters

| Parameter | Short | Description |
|-----------|-------|-------------|
| `--issue` | `-i` | Issue number to capture from |
| `--type` | `-t` | Knowledge type: `decision`, `resolution`, `pattern`, `feedback` |
| `--from-review` | | Capture from most recent code review |
| `--content` | `-c` | Explicit content to capture |
| `--tags` | | Comma-separated tags |

### Invocation Modes

| Mode | Example | Use Case |
|------|---------|----------|
| From issue | `/learn -i 42` | Capture resolution of completed issue |
| From review | `/learn --from-review` | Capture patterns from code review |
| Explicit | `/learn -t decision -c "Use PostgreSQL..."` | Manual decision capture |
| Interactive | `/learn` | Guided capture wizard |

## Knowledge Types

### Decision (`decision`)

Architectural or technical decisions with rationale.

**Auto-captured at**: ENRICH phase when spec is created
**Required fields**: title, decision, rationale

**Example**:
```
/learn -t decision -c "Use event sourcing for audit log"
```

### Resolution (`resolution`)

Issue resolution including root cause and solution.

**Auto-captured at**: DONE transition
**Required fields**: issue_ref, problem, root_cause, solution

**Example**:
```
/learn -i 42
```

### Pattern (`pattern`)

Reusable pattern or anti-pattern identified from experience.

**Auto-captured from**: Repeated review feedback (3+ occurrences)
**Required fields**: name, type (pattern|anti-pattern), description

**Example**:
```
/learn -t pattern -c "Always use mutex for token operations" --tags auth,concurrency
```

### Feedback (`feedback`)

Review feedback worth preserving.

**Auto-captured at**: After code review if score >= 80
**Required fields**: finding, recommendation

**Example**:
```
/learn --from-review
```

## Examples

### Capture from completed issue

```
User: /learn -i 42

Claude:
Capturing resolution for issue #42...

Extracted:
- Problem: Race condition in token refresh
- Root Cause: Concurrent requests without synchronization
- Solution: Added mutex protection
- Files: src/auth/tokenService.ts

Created: KE-resolution-20250114-b7c9
Linked to: ADR-003, Pattern "Token Operation Mutex"
```

### Record architectural decision

```
User: /learn -t decision -c "Use PostgreSQL for user data"

Claude:
Recording decision...

Please provide additional context:
1. What alternatives were considered?
2. Why was this chosen over alternatives?
3. What are the consequences?

[User provides context]

Created: KE-decision-20250114-a3f2
Tags: [database, infrastructure]
```

### Capture from code review

```
User: /learn --from-review

Claude:
Found 3 findings from recent review of PR #45:

1. [HIGH] Missing null check - captured
2. [MEDIUM] Inconsistent error format - captured
3. [LOW] Style issue - skipped (below threshold)

Created: 2 feedback entries
Note: "Missing null check" now has 3 occurrences - promoting to pattern
Created: KE-pattern-20250114-c8d3
```

## Output

After capture, the skill reports:
- Knowledge entry ID created
- Summary of what was captured
- Related entries found and linked
- Tags assigned
- Pattern promotion (if triggered)
