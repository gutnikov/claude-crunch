## Remediation Playbook

Configuration for staging auto-remediation during VALIDATION phase. Copy to `.claude/config/remediation-playbook.md` and customize.

---

### Purpose

This playbook defines:
1. Which remediation actions are allowed on staging
2. How to execute each action for your infrastructure
3. When each action is appropriate
4. Safety limits to prevent remediation loops

**Scope**: Staging environment only. These actions help distinguish transient issues from real bugs during deployment validation.

---

### Allowed Actions

Enable/disable actions for your environment:

| Action | Enabled | Notes |
|--------|---------|-------|
| `restart_pod` | yes | Kubernetes deployments |
| `restart_container` | yes | Docker containers |
| `flush_cache` | yes | Redis, Memcached, app cache |
| `reload_config` | yes | Configuration refresh |
| `scale_up` | no | Not recommended for staging |

---

### Action Definitions

#### restart_pod

Restart Kubernetes deployment pods.

**When to use**:
- Memory leak symptoms (high memory, OOM)
- Stuck process (not responding, deadlock)
- Connection pool exhaustion
- Cold start errors that persist

**Command**:
```bash
kubectl rollout restart deployment/{deployment_name} -n staging
```

**Variables**:
- `{deployment_name}`: Name of the deployment (usually same as service name)

**Timeout**: 60 seconds

**Verification**:
```bash
kubectl rollout status deployment/{deployment_name} -n staging --timeout=60s
```

**Fallback** (if kubectl unavailable):
```bash
# Delete pods to trigger restart
kubectl delete pods -l app={app_label} -n staging
```

---

#### restart_container

Restart Docker container directly.

**When to use**:
- Docker Compose environments
- Single-container deployments
- When Kubernetes is not used

**Command**:
```bash
docker restart {container_name}
```

**Variables**:
- `{container_name}`: Container name or ID

**Timeout**: 30 seconds

**Verification**:
```bash
docker ps --filter name={container_name} --format "{{.Status}}"
# Should show "Up X seconds"
```

---

#### flush_cache

Clear application or external cache.

**When to use**:
- Stale data symptoms
- Cache corruption after schema change
- Inconsistent responses

**Commands** (choose based on your stack):

**Redis**:
```bash
redis-cli -h {redis_host} -p {redis_port} FLUSHDB
```

**Memcached**:
```bash
echo "flush_all" | nc {memcached_host} {memcached_port}
```

**Application cache endpoint**:
```bash
curl -X POST http://{app_host}:{app_port}/admin/cache/flush \
  -H "Authorization: Bearer {admin_token}"
```

**Variables**:
- `{redis_host}`: Redis server hostname (default: redis.staging)
- `{redis_port}`: Redis port (default: 6379)
- `{app_host}`: Application hostname
- `{app_port}`: Application port
- `{admin_token}`: Admin API token (from vault)

**Timeout**: 30 seconds

**Verification**:
```bash
# Check cache is empty or reset
redis-cli -h {redis_host} DBSIZE
# Should return 0 or small number
```

---

#### reload_config

Trigger configuration reload without full restart.

**When to use**:
- Config changes not picked up
- Feature flag not reflecting
- Environment variable refresh needed

**Commands** (choose based on your stack):

**SIGHUP reload**:
```bash
# Find process ID and send HUP signal
kubectl exec -n staging deployment/{deployment_name} -- kill -HUP 1
```

**HTTP reload endpoint**:
```bash
curl -X POST http://{app_host}:{app_port}/admin/config/reload \
  -H "Authorization: Bearer {admin_token}"
```

**Consul/etcd watch** (if using dynamic config):
```bash
# Config should auto-reload, just verify
curl http://{app_host}:{app_port}/health/config
```

**Timeout**: 15 seconds

**Verification**:
```bash
# Check config version or timestamp
curl http://{app_host}:{app_port}/admin/config/version
```

---

### Safety Limits

Prevent remediation loops and excessive actions:

| Limit | Default | Description |
|-------|---------|-------------|
| `max_attempts_per_type` | 1 | Max remediation attempts per anomaly type |
| `max_total_attempts` | 2 | Max total remediations per monitoring window |
| `cooldown_minutes` | 5 | Minutes between same action type |
| `stabilization_wait` | 2m | Wait time after action before checking |
| `monitoring_timeout` | 20m | Total monitoring window |

**If limits exceeded**:
```
Auto-remediation limit reached.
Treating as bug requiring code fix.
Returning to IMPLEMENTING phase.
```

---

### Anomaly to Action Mapping

Default mapping of anomaly types to remediation actions:

| Anomaly Type | Primary Action | When |
|--------------|----------------|------|
| `error_spike` | `restart_pod` | If errors are retryable (timeout, connection) |
| `error_spike` | none | If errors are fatal (NPE, assertion) |
| `latency_regression` | `restart_pod` | If GC or memory related |
| `latency_regression` | `flush_cache` | If cache-related (slow queries) |
| `new_exception` | none | Usually indicates bug |
| `memory_growth` | `restart_pod` | If growth stabilizes after restart |
| `memory_growth` | none | If linear growth (leak - needs fix) |

**Override mapping** (customize for your app):

```yaml
# Custom mappings for specific error patterns
mappings:
  - pattern: "ConnectionPoolExhausted"
    action: restart_pod
    classification: transient

  - pattern: "CacheStaleException"
    action: flush_cache
    classification: transient

  - pattern: "NullPointerException"
    action: none
    classification: bug

  - pattern: "OutOfMemoryError"
    action: restart_pod
    classification: transient
    note: "Temporary fix, investigate if recurring"
```

---

### Infrastructure Configuration

Configure for your staging environment:

#### Kubernetes

```yaml
kubernetes:
  namespace: staging
  context: staging-cluster  # kubectl context
  deployments:
    - name: api
      app_label: api
    - name: worker
      app_label: worker
    - name: web
      app_label: web
```

#### Docker Compose

```yaml
docker:
  compose_file: docker-compose.staging.yml
  project_name: myapp-staging
  containers:
    - name: myapp-api
    - name: myapp-worker
```

#### Cache

```yaml
cache:
  type: redis  # redis | memcached | application
  host: redis.staging.svc.cluster.local
  port: 6379
  flush_command: FLUSHDB  # or FLUSHALL for all DBs
```

#### Application

```yaml
application:
  host: api.staging.internal
  port: 8080
  health_endpoint: /health
  config_reload_endpoint: /admin/config/reload
  cache_flush_endpoint: /admin/cache/flush
  admin_token_path: secret/staging/admin-token  # vault path
```

---

### Verification Checks

After remediation, verify success with these checks:

#### Error Rate Check

```bash
# Query Prometheus for error rate
curl -s "http://{prometheus_url}/api/v1/query" \
  --data-urlencode 'query=rate(http_requests_total{status=~"5..",namespace="staging"}[2m])' \
  | jq '.data.result[0].value[1]'
```

**Success criteria**: Error rate <= baseline * 1.5

#### Health Check

```bash
curl -s -o /dev/null -w "%{http_code}" http://{app_host}:{app_port}/health
```

**Success criteria**: HTTP 200

#### Log Check

```bash
# Query Loki for recent errors
curl -s "http://{loki_url}/loki/api/v1/query_range" \
  --data-urlencode 'query={namespace="staging"} |= "error"' \
  --data-urlencode 'start='$(date -d '2 minutes ago' +%s)000000000 \
  --data-urlencode 'end='$(date +%s)000000000
```

**Success criteria**: Error count lower than before remediation

---

### Example Full Configuration

```yaml
# .claude/config/remediation-playbook.md

## Remediation Playbook - MyApp Staging

### Allowed Actions
| Action | Enabled |
|--------|---------|
| restart_pod | yes |
| restart_container | no |
| flush_cache | yes |
| reload_config | yes |

### Safety Limits
| Limit | Value |
|-------|-------|
| max_attempts_per_type | 1 |
| max_total_attempts | 2 |
| stabilization_wait | 2m |

### Infrastructure
kubernetes:
  namespace: staging
  deployments:
    - name: myapp-api
      app_label: myapp-api

cache:
  type: redis
  host: redis.staging
  port: 6379

### Custom Mappings
mappings:
  - pattern: "Redis connection refused"
    action: restart_pod
    classification: transient

  - pattern: "JWT token expired"
    action: flush_cache
    classification: transient
```

---

### Disabling Auto-Remediation

To disable auto-remediation entirely:

```yaml
## Remediation Playbook

### Allowed Actions
| Action | Enabled |
|--------|---------|
| restart_pod | no |
| restart_container | no |
| flush_cache | no |
| reload_config | no |

# All anomalies will be treated as bugs requiring investigation
```

Or skip per-run with `/crunch` flag:
```
/crunch {issue} --skip-remediation
```
