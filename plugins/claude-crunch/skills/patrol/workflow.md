# Patrol Workflow

Detailed execution flow for each patrol mode.

---

## Execution Flow

### 1. Initialization

```
1. Parse arguments:
   - Detect mode (--code, --deps, --logs, --metrics, --security, or all)
   - Read options (--dry-run, --threshold, --max-issues)

2. Load configuration:
   - Check for .claude/config/patrol-config.md
   - Apply defaults for missing values

3. Initialize state:
   - findings = []
   - issues_created = 0
   - agents_invoked = []
```

### 2. Module Execution

Execute modules based on mode. Multiple modules run sequentially.

#### Code Health Module (`--code`)

```
1. Invoke code-health-analyst agent:
   - Target: changed files (if on branch) or full scan
   - Metrics: complexity, duplication, dead code
   - Markers: TODO, FIXME, HACK

2. For each finding:
   - Calculate confidence score
   - Apply exclusion rules
   - Add to findings list

3. Output format:
   {
     module: "code",
     type: "complexity|duplication|dead_code|tech_debt",
     severity: "CRITICAL|HIGH|MEDIUM|LOW",
     confidence: 0-100,
     location: "file:line",
     description: "...",
     recommendation: "..."
   }
```

#### Security Module (`--security`)

```
1. Invoke dependency-manager agent:
   - Scan: CVE vulnerabilities
   - Source: GitHub Advisory, NVD

2. Invoke security-analyst agent:
   - Scan: SAST patterns (injection, XSS, secrets)
   - Target: changed files or full codebase

3. Merge findings:
   - Deduplicate same CVE from different sources
   - Correlate secrets with affected files

4. Output format:
   {
     module: "security",
     type: "cve|sast|secrets|config",
     severity: "CRITICAL|HIGH|MEDIUM|LOW",
     confidence: 0-100,
     cve_id: "CVE-XXXX-XXXX" (if applicable),
     package: "name@version" (if applicable),
     location: "file:line",
     description: "...",
     recommendation: "..."
   }
```

#### Dependency Module (`--deps`)

```
1. Invoke dependency-manager agent:
   - Scan: All manifest files
   - Check: Outdated, vulnerable, abandoned, license

2. Categorize findings:
   - Security: CVEs (also appears in --security)
   - Maintenance: Outdated, abandoned
   - Legal: License issues

3. Output format:
   {
     module: "deps",
     type: "vulnerability|outdated|abandoned|license",
     severity: "CRITICAL|HIGH|MEDIUM|LOW",
     confidence: 0-100,
     package: "name@version",
     current_version: "...",
     recommended_version: "...",
     description: "...",
     recommendation: "..."
   }
```

#### Log Analysis Module (`--logs`)

```
1. Query log aggregator:
   - Use Loki MCP, Elasticsearch MCP, or CloudWatch MCP
   - Time range: last 15 minutes (default)
   - Focus: error and warning levels

2. Invoke log-analyst agent:
   - Detect: Error rate spikes, new exceptions
   - Compare: Against baseline (if available)

3. Output format:
   {
     module: "logs",
     type: "error_spike|new_exception|warning_storm|missing_logs",
     severity: "CRITICAL|HIGH|MEDIUM|LOW",
     confidence: 0-100,
     time_range: "start - end",
     count: N,
     baseline: M,
     sample: "log line...",
     description: "...",
     recommendation: "..."
   }
```

#### Metrics Module (`--metrics`)

```
1. Query Prometheus MCP:
   - Latency: http_request_duration_seconds
   - Errors: http_requests_total{status=~"5.."}
   - Resources: container_memory_usage_bytes, container_cpu_usage

2. Invoke devops-engineer agent:
   - Detect: Regressions, leaks, saturation
   - Compare: Against baseline (if available)

3. Output format:
   {
     module: "metrics",
     type: "latency_regression|memory_leak|cpu_spike|throughput_drop",
     severity: "CRITICAL|HIGH|MEDIUM|LOW",
     confidence: 0-100,
     metric: "metric_name",
     current_value: N,
     baseline_value: M,
     threshold: X,
     description: "...",
     recommendation: "..."
   }
```

### 3. Alert Correlation

When multiple modules run, correlate findings:

```
1. Group by root cause:
   - CVE in package → security + deps findings merge
   - Memory leak → metrics + logs findings merge

2. Identify cascades:
   - Dependency vulnerability causing errors in logs
   - High complexity causing more bugs

3. Reduce duplicates:
   - Same issue detected by multiple modules
   - Keep highest severity, merge evidence

4. Update findings:
   - Add correlated_with: [finding_ids]
   - Adjust confidence based on correlation
```

### 4. Issue Creation

```
1. Filter findings:
   - confidence >= threshold (default: 80)
   - severity in allowed_severities (default: CRITICAL, HIGH, MEDIUM)
   - Not in exclusions list
   - Not duplicate of existing issue

2. Check knowledge base:
   - Query for similar past findings
   - Skip if already tracked
   - Link if related

3. Check existing issues:
   - Query CI MCP for open patrol-generated issues
   - Skip if duplicate CVE or same location

4. Create issues (if not --dry-run):
   - Use CI MCP to create issue
   - Apply template (see below)
   - Add labels: patrol-generated, {module}, {severity}
   - Track in findings[].issue_number

5. Respect limits:
   - Stop at max_issues_per_run
   - Prioritize by severity then confidence
```

### 5. Issue Template

```markdown
## [patrol:{module}] {title}

**Severity**: {severity}
**Detection Source**: {agent_name}
**Confidence**: {confidence}%
**First Detected**: {timestamp}

### Finding

{description}

### Evidence

{evidence based on type}

For CVE:

- Package: {package}@{version}
- CVSS: {score}
- Fixed in: {fixed_version}

For code health:

- Location: {file}:{line}
- Metric: {metric_name} = {value}

For logs:

- Time range: {start} - {end}
- Count: {n} occurrences
- Sample: {log_line}

### Impact

{impact_description}

### Recommendation

{specific_action}

### References

{links to CVE, docs, etc.}

---

**Auto-generated by /patrol**
Module: {module}
Run: {timestamp}
```

### 6. Reporting

```
1. Console output:
   - Per-module summary
   - Findings by severity
   - Issues created vs skipped

2. Return data:
   {
     timestamp: "...",
     modules_run: ["code", "deps", ...],
     findings: [...],
     issues_created: N,
     issues_skipped: M,
     dry_run: true|false
   }
```

---

## Configuration Schema

`.claude/config/patrol-config.md`:

```markdown
## Patrol Configuration

### Thresholds

Detection thresholds for each module:

| Module  | Metric                 | Value |
| ------- | ---------------------- | ----- |
| code    | complexity_threshold   | 15    |
| code    | duplication_min_lines  | 6     |
| code    | todo_age_days          | 30    |
| code    | nesting_depth_max      | 3     |
| logs    | error_rate_multiplier  | 2.0   |
| logs    | baseline_window        | 24h   |
| metrics | latency_regression_pct | 50    |
| metrics | memory_growth_pct      | 20    |

### Exclusions

Paths and patterns to skip:
```

paths:

- node_modules
- vendor
- dist
- build
- coverage
- .git

files:

- "\*.min.js"
- "_.generated._"
- "_.test._" # Optional: exclude tests

packages:

- internal-test-utils

cves:

- CVE-2024-XXXX # Known false positive with justification

```

### Issue Creation

| Setting | Value | Description |
|---------|-------|-------------|
| confidence_threshold | 80 | Minimum confidence to create issue |
| max_issues_per_run | 10 | Rate limit per patrol run |
| create_for_severities | CRITICAL, HIGH, MEDIUM | Which severities create issues |
| dry_run_default | false | Default to dry-run mode |
```

---

## Integration with E2E Phase

When used during `/crunch` E2E phase for continuous monitoring:

### Activation

```
/crunch workflow reaches E2E phase
  → DoD checklist includes "Monitor staging for 20 minutes"
  → Invoke patrol in continuous mode
```

### Continuous Monitoring Loop

```
Parameters:
  - duration: 20m (configurable)
  - interval: 5m (configurable)
  - modules: --logs --metrics (monitoring-relevant)

Execution:
  for cycle in 1..(duration / interval):
    1. Run log analysis:
       - Query last {interval} of logs
       - Detect error spikes, new exceptions
       - Compare to baseline

    2. Run metrics analysis:
       - Query current metrics
       - Check for regressions vs pre-deploy baseline

    3. Report cycle results:
       Cycle {n}/{total}: {timestamp}
         Logs: {OK|SPIKE|ANOMALY}
           - Error rate: {rate}/min (baseline: {baseline})
           - New exceptions: {count}
         Metrics: {OK|REGRESSION}
           - P99 latency: {value}ms (baseline: {baseline}ms)
           - Memory: {value}MB

    4. Decide:
       IF anomaly detected:
         → Report immediately
         → Stay in E2E
         → Recommend investigation
         → STOP loop (don't proceed)
       ELSE:
         → Sleep {interval}
         → Continue to next cycle

    5. After all cycles complete with no anomalies:
       → Report "Monitoring complete - no issues detected"
       → Proceed to next phase (DOCS)
```

### Anomaly Thresholds (E2E mode)

More sensitive than standard patrol to catch deployment issues:

| Metric              | Warning       | Critical       |
| ------------------- | ------------- | -------------- |
| Error rate          | 1.5x baseline | 3x baseline    |
| P99 latency         | 30% increase  | 50% increase   |
| New exception types | 3 occurrences | 10 occurrences |
| Memory growth       | 10% increase  | 25% increase   |

### Output Format (Continuous)

```
Continuous Monitoring - E2E Phase
========================================

Baseline captured: {timestamp}
  - Error rate: {n}/min
  - P99 latency: {ms}ms
  - Memory: {MB}MB

Cycle 1/4 [{timestamp}]:
  Logs:
    ✓ Error rate: 0.1/min (baseline: 0.15/min)
    ✓ No new exception types
  Metrics:
    ✓ P99 latency: 120ms (baseline: 115ms)
    ✓ Memory: 256MB (baseline: 250MB)
  Status: HEALTHY
  Sleeping 5 minutes...

Cycle 2/4 [{timestamp}]:
  Logs:
    ⚠ Error rate: 2.3/min (baseline: 0.15/min) - SPIKE DETECTED
    ! New exception: NullPointerException in AuthService
  Metrics:
    ✓ P99 latency: 125ms
    ✓ Memory: 258MB
  Status: DEGRADED

ANOMALY DETECTED - Stopping monitoring

Recommendation:
  - Investigate error spike in logs
  - Check AuthService for null pointer issue
  - Do not proceed to DOCS until resolved

Staying in E2E phase.
```

---

## Error Handling

| Error                | Handling                                 |
| -------------------- | ---------------------------------------- |
| MCP unavailable      | Skip module, warn user, continue         |
| Agent timeout        | Retry once, then skip with warning       |
| No config file       | Use defaults                             |
| Issue creation fails | Log error, continue with other findings  |
| Rate limit hit       | Stop creating, report remaining findings |

---

## Knowledge Integration

### Before Run

```
1. Query knowledge base for:
   - Past patrol findings (avoid duplicates)
   - Known false positives (apply exclusions)
   - Baseline metrics (for comparison)

2. Load from index.json:
   - patrol_history.last_run
   - patrol_history.known_issues
   - patrol_history.false_positives
```

### After Run

```
1. Store findings in knowledge base:
   - Entry type: patrol_finding
   - Link to created issues
   - Track confidence scores

2. Update metrics:
   - patrol_history.last_run = now
   - patrol_history.runs[date] = summary

3. Learn from feedback:
   - If issue closed as "not a bug" → add to false_positives
   - If issue fixed → record resolution pattern
```
