## Patrol Configuration

Configuration for the `/patrol` skill. Copy this template to `.claude/config/patrol-config.md` and customize.

---

### Detection Thresholds

Adjust sensitivity for each detection module:

#### Code Health

| Metric                           | Default | Description                                 |
| -------------------------------- | ------- | ------------------------------------------- |
| `complexity_threshold`           | 15      | Cyclomatic complexity above this is flagged |
| `cognitive_complexity_threshold` | 20      | Cognitive complexity above this is flagged  |
| `duplication_min_lines`          | 6       | Minimum lines for duplicate detection       |
| `method_length_max`              | 50      | Lines per method before flagging            |
| `file_length_max`                | 500     | Lines per file before flagging              |
| `nesting_depth_max`              | 3       | Nesting levels before flagging              |
| `parameter_count_max`            | 4       | Function parameters before flagging         |
| `todo_age_days`                  | 30      | Days before TODO becomes tech debt          |

#### Log Analysis

| Metric                   | Default | Description                          |
| ------------------------ | ------- | ------------------------------------ |
| `error_rate_warning`     | 2.0     | Multiplier vs baseline for warning   |
| `error_rate_critical`    | 5.0     | Multiplier vs baseline for critical  |
| `new_exception_warning`  | 5       | Occurrences before warning           |
| `new_exception_critical` | 20      | Occurrences before critical          |
| `baseline_window`        | 24h     | Time window for baseline calculation |
| `analysis_window`        | 15m     | Default time range for log queries   |

#### Metrics Analysis

| Metric                       | Default | Description                          |
| ---------------------------- | ------- | ------------------------------------ |
| `latency_regression_pct`     | 50      | P99 increase percentage for warning  |
| `latency_critical_pct`       | 100     | P99 increase percentage for critical |
| `memory_growth_warning_pct`  | 20      | Memory increase for warning          |
| `memory_growth_critical_pct` | 50      | Memory increase for critical         |
| `throughput_drop_pct`        | 30      | Request rate decrease for warning    |

#### Dependency Audit

| Metric                  | Default | Description                             |
| ----------------------- | ------- | --------------------------------------- |
| `major_versions_behind` | 2       | Major versions behind for HIGH severity |
| `abandoned_years`       | 2       | Years without update to flag            |
| `cvss_critical`         | 9.0     | CVSS score for CRITICAL severity        |
| `cvss_high`             | 7.0     | CVSS score for HIGH severity            |
| `cvss_medium`           | 4.0     | CVSS score for MEDIUM severity          |

---

### Exclusions

#### Path Exclusions

Directories and files to skip during analysis:

```
paths:
  - node_modules
  - vendor
  - dist
  - build
  - coverage
  - .git
  - .next
  - __pycache__
  - .pytest_cache
  - target
  - bin
  - obj
```

#### File Pattern Exclusions

```
files:
  - "*.min.js"
  - "*.min.css"
  - "*.generated.*"
  - "*.pb.go"
  - "*.d.ts"
  - "*_generated.go"
  - "swagger.json"
  - "openapi.yaml"
```

#### Package Exclusions

Packages to skip in dependency analysis:

```
packages:
  - internal-test-utils
  - @company/internal-types
```

#### CVE Exclusions

Known false positives with justification:

```
cves:
  # CVE-XXXX-YYYY: Justification for exclusion
  # Example: Not exploitable in our usage pattern
```

---

### Issue Creation

Control how patrol creates issues:

| Setting                | Default | Description                          |
| ---------------------- | ------- | ------------------------------------ |
| `confidence_threshold` | 80      | Minimum confidence % to create issue |
| `max_issues_per_run`   | 10      | Rate limit per patrol run            |
| `dry_run_default`      | false   | Default to dry-run mode              |
| `issue_label_prefix`   | patrol  | Prefix for auto-labels               |
| `link_to_knowledge`    | true    | Link issues to knowledge entries     |

#### Severity Gating

Which severities auto-create issues:

```
create_for_severities:
  - CRITICAL
  - HIGH
  - MEDIUM
```

Low and INFO severity findings are reported but don't create issues.

---

### Continuous Monitoring (E2E Phase)

Settings for monitoring during E2E:

| Setting    | Default       | Description                       |
| ---------- | ------------- | --------------------------------- |
| `duration` | 20m           | Total monitoring duration         |
| `interval` | 5m            | Time between checks               |
| `modules`  | logs, metrics | Modules to run in continuous mode |

#### E2E Thresholds

More sensitive thresholds during deployment validation:

| Metric                | Warning | Critical |
| --------------------- | ------- | -------- |
| Error rate multiplier | 1.5x    | 3.0x     |
| P99 latency increase  | 30%     | 50%      |
| New exception count   | 3       | 10       |
| Memory growth         | 10%     | 25%      |

---

### MCP Configuration

Which MCPs to use for each data source:

| Data Source     | MCP                             | Fallback            |
| --------------- | ------------------------------- | ------------------- |
| Logs            | loki, elasticsearch, cloudwatch | Web search for CVEs |
| Metrics         | prometheus                      | None                |
| Issues          | github, gitlab, gitea           | Error               |
| Vulnerabilities | GitHub Advisory API             | NVD web search      |

---

### Baseline Storage

Where to store baseline metrics for comparison:

```
baseline:
  storage: .claude/patrol/baselines.yaml
  refresh_interval: 24h
  metrics:
    - http_request_duration_seconds
    - http_requests_total
    - process_resident_memory_bytes
```

---

### Notification (Future)

Future integration for alerting:

```
notifications:
  enabled: false
  channels:
    - type: slack
      webhook: ${SLACK_WEBHOOK_URL}
      severities: [CRITICAL, HIGH]
    - type: email
      to: security@example.com
      severities: [CRITICAL]
```

---

### Example Configuration

Minimal configuration for a Node.js project:

```markdown
## Patrol Configuration

### Detection Thresholds

| Module | Metric               | Value |
| ------ | -------------------- | ----- |
| code   | complexity_threshold | 20    |
| code   | todo_age_days        | 60    |
| logs   | error_rate_warning   | 3.0   |

### Exclusions

paths:

- node_modules
- dist
- coverage

### Issue Creation

| Setting              | Value |
| -------------------- | ----- |
| confidence_threshold | 85    |
| max_issues_per_run   | 5     |
| dry_run_default      | true  |
```
