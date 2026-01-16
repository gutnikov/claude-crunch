## Design for E2E Testability with Maximum Autonomy

> **MUST:** All DOD items MUST be verifiable without human intervention during E2E phase.

Design everything from the perspective of autonomous validation. The E2E phase should be able to verify all acceptance criteria programmatically, without requiring human judgment calls.

### Core Principle

Before creating any DOD item, ask: "Can this be verified automatically?"

For every acceptance criterion:

- Define the exact command, API call, or metric query
- Specify the expected output or threshold
- Ensure the verification is deterministic and repeatable

### Autonomous Verification Types

| Type            | Description                      | Example                                                      |
| --------------- | -------------------------------- | ------------------------------------------------------------ |
| API Assertion   | HTTP call with expected response | `curl /api/users/1` returns 200 with JSON containing `id`    |
| Metric Check    | Prometheus query with threshold  | `rate(http_errors[5m]) < 0.01`                               |
| Log Pattern     | Loki query for absence/presence  | No errors matching `FATAL.*auth` in last 5 minutes           |
| Health Endpoint | Structured health check          | `/health` returns `{"status": "ok"}`                         |
| Command Output  | CLI with expected output         | `./cli status` exits 0 with "healthy" in output              |
| Database Query  | SQL with expected result         | `SELECT count(*) FROM users WHERE active = true` returns > 0 |
| File Check      | File existence or content        | Config file exists at `/etc/app/config.yaml`                 |

### DOD Anti-Patterns (AVOID)

| Anti-Pattern                | Problem              | Better Alternative                               |
| --------------------------- | -------------------- | ------------------------------------------------ |
| "Works correctly"           | Not verifiable       | "Returns 200 with user JSON schema"              |
| "No bugs"                   | Impossible to verify | "Error rate < 0.1% over 5min"                    |
| "Looks good"                | Subjective           | "P99 latency < 200ms"                            |
| "Performance is acceptable" | Vague threshold      | "Response time < 500ms at P95"                   |
| "Users can login"           | Missing specifics    | "POST /login with valid creds returns 200 + JWT" |
| "System is stable"          | Unmeasurable         | "No restarts in 20min, memory < 512MB"           |

### DOD Best Practices (USE)

| Good DOD Item               | Verification Method                                                                                |
| --------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| API returns user data       | `curl -H "Auth: $TOKEN" /api/users/1` returns 200 with `{"id": 1, "name": "..."}`                  |
| Error rate within SLA       | Prometheus: `rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) < 0.001` |
| No auth errors in logs      | Loki: `{app="myapp"}                                                                               | = "authentication failed"` returns 0 results in last 10min |
| Cache hit ratio healthy     | Prometheus: `redis_cache_hits / (redis_cache_hits + redis_cache_misses) > 0.8`                     |
| Database migrations applied | `SELECT version FROM schema_migrations ORDER BY version DESC LIMIT 1` returns expected version     |

### Health Endpoint Requirements

All services MUST expose a `/health` endpoint for autonomous validation.

**Required Response Format:**

```json
{
  "status": "healthy|degraded|unhealthy",
  "checks": {
    "database": { "status": "healthy", "latency_ms": 5 },
    "redis": { "status": "healthy", "latency_ms": 2 },
    "external_api": { "status": "healthy", "latency_ms": 150 }
  },
  "version": "1.2.3",
  "uptime_seconds": 3600
}
```

**Health Status Definitions:**

| Status      | Meaning                        | E2E Behavior          |
| ----------- | ------------------------------ | --------------------- |
| `healthy`   | All dependencies operational   | Continue validation   |
| `degraded`  | Non-critical dependency issues | Continue with warning |
| `unhealthy` | Critical failures              | Fail validation       |

### Verification Command Templates

**API Verification:**

```bash
# Basic API check
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/api/endpoint
# Expected: 200

# API with response validation
curl -s http://localhost:8080/api/users/1 | jq -e '.id == 1'
# Expected: exit 0
```

**Metric Verification:**

```promql
# Error rate check
rate(http_requests_total{status=~"5.."}[5m]) < 0.01

# Latency check
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) < 0.2

# Memory check
container_memory_usage_bytes{container="myapp"} < 512 * 1024 * 1024
```

**Log Verification:**

```logql
# No fatal errors
{app="myapp"} |= "FATAL" | count_over_time([10m]) == 0

# No authentication failures
{app="myapp"} |~ "auth.*fail" | count_over_time([10m]) == 0
```

### Autonomous Validation Workflow

```
For each DOD item:
  1. Parse verification type (API/Metric/Log/Health/Command)
  2. Execute verification command
  3. Compare result to expected value
  4. Record: PASS | FAIL | ERROR

If all items PASS:
  → Proceed to next phase

If any item FAIL:
  → Record failure details
  → Return to IMPLEMENTING

If any item ERROR (verification itself failed):
  → Log error
  → Flag for manual review
```

### Integration with Test Plan

The Test Plan (created during ENRICH) should include an E2E Validation section that maps directly to DOD items:

```markdown
#### E2E Validation Checklist

| DOD Item           | Verification Type | Command/Query      | Expected              |
| ------------------ | ----------------- | ------------------ | --------------------- | --------- |
| API returns users  | API               | `curl /api/users`  | 200, array length > 0 |
| Error rate healthy | Metric            | `rate(errors[5m])` | < 0.01                |
| No auth errors     | Log               | `{app}             | = "auth error"`       | 0 results |
| Service healthy    | Health            | `/health`          | status: healthy       |
```

### Benefits of Autonomous E2E

1. **Faster feedback** - No waiting for human verification
2. **Consistent validation** - Same checks every time
3. **Documented criteria** - Clear what "done" means
4. **Regression prevention** - Automated checks catch issues
5. **CI/CD ready** - Can run in pipeline without intervention
