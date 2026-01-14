---
name: patrol
description: Proactively detect issues in code health, dependencies, logs, and metrics before they become bugs. Auto-creates issues for findings that exceed confidence thresholds.
---

# Patrol - Proactive Issue Detection

Shift from reactive (wait for bugs) to proactive (detect issues before they become bugs). The patrol skill continuously monitors code health, dependencies, logs, and metrics to auto-create issues in the backlog.

## Syntax

```
/patrol [options]
```

### Modes

| Command | Description |
|---------|-------------|
| `/patrol` | Run all detection modules once |
| `/patrol --code` | Code health only (tech debt, complexity) |
| `/patrol --security` | Security vulnerabilities (CVEs, SAST) |
| `/patrol --deps` | Dependency updates and vulnerabilities |
| `/patrol --logs` | Log analysis for error patterns |
| `/patrol --metrics` | Performance anomalies from Prometheus |
| `/patrol --all` | Run all modules (same as bare `/patrol`) |

### Parameters

| Parameter | Short | Description |
|-----------|-------|-------------|
| `--code` | | Run code health analysis |
| `--security` | | Run security scan |
| `--deps` | | Run dependency audit |
| `--logs` | | Run log analysis |
| `--metrics` | | Run metrics analysis |
| `--dry-run` | `-n` | Show findings without creating issues |
| `--threshold` | `-t` | Minimum confidence for issue creation (default: 80) |
| `--max-issues` | `-m` | Maximum issues to create (default: 10) |

## Detection Modules

### 1. Code Health (`--code`)

Uses: `code-health-analyst` agent

Detects:
- Complexity hotspots (cyclomatic complexity > 15)
- Duplicated code blocks (> 6 lines)
- Dead code and unused imports
- TODO/FIXME/HACK markers (tech debt)
- God classes and long methods
- Deep nesting (> 3 levels)

### 2. Security Scan (`--security`)

Uses: `security-analyst` agent + `dependency-manager` agent

Detects:
- CVE vulnerabilities in dependencies
- SAST patterns (injection, XSS, etc.)
- Hardcoded secrets and credentials
- Insecure configurations
- Missing security headers

### 3. Dependency Audit (`--deps`)

Uses: `dependency-manager` agent

Detects:
- Outdated packages (major versions behind)
- Vulnerable dependencies (CVEs)
- Abandoned packages (no updates in 2+ years)
- License compliance issues
- Lock file inconsistencies

### 4. Log Analysis (`--logs`)

Uses: `log-analyst` agent

Detects:
- Error rate spikes (> 2x baseline)
- New exception types
- Stack trace patterns
- Warning storms
- Missing expected logs

### 5. Metrics Analysis (`--metrics`)

Uses: `devops-engineer` agent + Prometheus MCP

Detects:
- Latency regressions (P99 > 2x baseline)
- Memory leaks (increasing RSS over time)
- CPU spikes
- Throughput drops
- Resource saturation

### 6. Alert Correlation (automatic)

When multiple modules run:
- Deduplicate related findings
- Group by root cause
- Link symptoms to causes
- Reduce noise, increase signal

## Output

### Console Output

```
Patrol Report - {timestamp}
==========================

Code Health:
  [MEDIUM] High complexity in auth/tokenService.ts (CC: 24)
  [LOW] 12 TODO markers older than 30 days

Dependencies:
  [HIGH] CVE-2025-1234 in lodash@4.17.20
  [MEDIUM] 3 packages with major updates available

Logs:
  [OK] No anomalies detected

Metrics:
  [OK] All metrics within baseline

Summary:
  - Critical: 0
  - High: 1
  - Medium: 2
  - Low: 1

Issues Created: 1 (--dry-run: would create 1)
  - #142: [patrol:security] CVE-2025-1234 in lodash@4.17.20
```

### Auto-Created Issues

When findings exceed threshold and `--dry-run` is not set:

```markdown
## [patrol:{module}] {Finding Title}

**Severity**: {CRITICAL|HIGH|MEDIUM}
**Detection Source**: {agent_name}
**Confidence**: {percentage}%
**First Detected**: {timestamp}

### Finding
{detailed description}

### Evidence
{code snippet, log sample, or metric data}

### Impact
{why this matters}

### Recommendation
{specific action to take}

### Auto-Tags
- patrol-generated
- {module} (security|code-health|deps|logs|metrics)
- {severity}
```

## Integration Points

### With /crunch

Patrol-generated issues enter the /crunch workflow:
- Start at BACKLOG state with `patrol-generated` label
- ENRICH phase can query knowledge base for similar past issues
- Priority auto-assigned based on severity

### With Knowledge System

- Findings stored in knowledge base for deduplication
- Pattern recognition across patrol runs
- Learning from false positive feedback

### With Existing Infrastructure

| Integration | Purpose |
|-------------|---------|
| Prometheus MCP | Query metrics for anomaly detection |
| Loki MCP | Query logs for error patterns |
| CI MCP | Create issues, trigger scans |
| GitHub/GitLab | Fetch dependency data |

## Safety Features

1. **Confidence Scoring**: Only create issues above threshold (default: 80%)
2. **Deduplication**: Check knowledge base before creating duplicate issues
3. **Rate Limiting**: Maximum issues per run (default: 10)
4. **Severity Gating**: Low severity collected but not auto-issued
5. **Human Review Mode**: `--dry-run` shows findings without creating issues

## Configuration

Patrol reads configuration from `.claude/config/patrol-config.md` if present:

```markdown
## Patrol Configuration

### Thresholds
- complexity_threshold: 15
- duplication_min_lines: 6
- todo_age_days: 30
- error_rate_multiplier: 2.0

### Exclusions
- paths: [node_modules, vendor, dist, build]
- packages: [internal-test-utils]
- cves: [CVE-2024-XXXX] # Known false positives

### Issue Creation
- confidence_threshold: 80
- max_issues_per_run: 10
- create_for_severities: [CRITICAL, HIGH, MEDIUM]
```

## Examples

### Quick security check

```
User: /patrol --security --dry-run

Claude:
Running security scan...

Agents invoked:
- security-analyst: SAST patterns
- dependency-manager: CVE scan

Findings:
  [HIGH] CVE-2025-1234 in lodash@4.17.20 (confidence: 95%)
    - Prototype Pollution vulnerability
    - Fixed in: 4.17.21
    - Recommendation: npm update lodash

  [MEDIUM] Hardcoded API key in config.ts:42 (confidence: 88%)
    - Pattern: API_KEY = "sk-..."
    - Recommendation: Move to environment variable

Dry run - no issues created
Would create: 2 issues
```

### Full patrol with issue creation

```
User: /patrol

Claude:
Running all detection modules...

Code Health: 3 findings (1 medium, 2 low)
Dependencies: 1 finding (1 high)
Logs: 0 findings
Metrics: 0 findings

Created issues:
- #143: [patrol:security] CVE-2025-1234 in lodash@4.17.20 (HIGH)
- #144: [patrol:code] High complexity in tokenService.ts (MEDIUM)

Skipped (below threshold or low severity):
- 2 TODO markers (LOW, confidence: 65%)
```

## Best Practices

1. **Start with --dry-run**: Review findings before enabling auto-creation
2. **Configure exclusions**: Add known false positives to config
3. **Review periodically**: Even skipped findings may need attention
4. **Link to CI**: Run patrol in CI for consistent checks
5. **Monitor trends**: Use /analyze to track patrol findings over time
