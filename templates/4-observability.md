## Observability

> **MUST:** All development work MUST include proper logging and metrics. All code reviews MUST verify observability requirements.

### Structured Logging

Logs are the primary tool for investigating problems.

When writing code that can potentially fail, ask yourself: "Are these logs enough to investigate and reproduce the problem in the future?" Imagine you are the one who will be debugging this at 3 AM.

Design everything from the perspective of a future investigator:

- Before writing code, ask: "What would I need to see in logs to understand what happened?"
- For every decision point: Log the inputs, the decision made, and why
- For every error path: Log context that helps reproduce and diagnose

**Practical guidelines:**

- Use JSON structured logging with consistent fields for processing by systems like Loki or Kibana
- Log at appropriate levels (DEBUG for flow, INFO for events, WARN for issues, ERROR for failures)

### Metrics and Alerts

> **MUST:** All services MUST expose a `/metrics` endpoint in Prometheus format. This is non-negotiable for production deployments.

Design your project so that metrics can be collected by Prometheus and alerts configured in Alertmanager.

Ask yourself:

- "What metrics could help me investigate problems proactively?"
- "What alerts can signal a problem before users notice?"

Design everything from the perspective of a future investigator:

- Before deploying, ask: "What metrics would tell me the system is healthy?"
- For every critical path: Expose latency, error rate, and throughput metrics
- For every dependency: Track availability and response times

**Prometheus metrics requirements:**

- Expose a `/metrics` endpoint that Prometheus can scrape
- Use appropriate metric types: counters for totals, gauges for current values, histograms for distributions
- Follow naming conventions (e.g., `http_requests_total`, `request_duration_seconds`)
- Include standard labels: `service`, `environment`, `instance`
- Expose application-specific business metrics alongside technical metrics

**Alerting requirements:**

- Set up alerts for anomalies, not just failures
- Include runbook links in alert annotations
- Define severity levels (critical, warning, info) consistently

---

## Observability Infrastructure Integration

> **IMPORTANT:** The `/patrol` skill and VALIDATION phase continuous monitoring require access to observability infrastructure via MCP servers.

### Required MCP Servers

| MCP Server | Purpose | Required For |
|------------|---------|--------------|
| Prometheus MCP | Query metrics | `/patrol --metrics`, VALIDATION monitoring |
| Loki MCP | Query logs | `/patrol --logs`, VALIDATION monitoring, log-analyst agent |
| Alertmanager MCP | Query/manage alerts | Alert correlation, incident detection |

### MCP Installation

#### Prometheus MCP

```bash
# Install Prometheus MCP server
# Option 1: npm (if available)
npm install -g @anthropic/mcp-server-prometheus

# Option 2: From source
git clone https://github.com/anthropics/mcp-servers
cd mcp-servers/prometheus
npm install && npm run build
```

**Configuration** (add to `.claude/mcp.json`):

```json
{
  "mcpServers": {
    "prometheus": {
      "command": "mcp-server-prometheus",
      "args": [],
      "env": {
        "PROMETHEUS_URL": "http://prometheus.your-infra.local:9090"
      }
    }
  }
}
```

**Verify installation:**

```bash
# Test query via Claude Code
# Ask: "Query Prometheus for up metric"
# Expected: Returns list of up targets
```

#### Loki MCP

```bash
# Install Loki MCP server
npm install -g @anthropic/mcp-server-loki

# Or from source
git clone https://github.com/anthropics/mcp-servers
cd mcp-servers/loki
npm install && npm run build
```

**Configuration** (add to `.claude/mcp.json`):

```json
{
  "mcpServers": {
    "loki": {
      "command": "mcp-server-loki",
      "args": [],
      "env": {
        "LOKI_URL": "http://loki.your-infra.local:3100"
      }
    }
  }
}
```

**Verify installation:**

```bash
# Test query via Claude Code
# Ask: "Query Loki for recent error logs"
# Expected: Returns log entries
```

#### Alertmanager MCP

```bash
# Install Alertmanager MCP server
npm install -g @anthropic/mcp-server-alertmanager

# Or from source
git clone https://github.com/anthropics/mcp-servers
cd mcp-servers/alertmanager
npm install && npm run build
```

**Configuration** (add to `.claude/mcp.json`):

```json
{
  "mcpServers": {
    "alertmanager": {
      "command": "mcp-server-alertmanager",
      "args": [],
      "env": {
        "ALERTMANAGER_URL": "http://alertmanager.your-infra.local:9093"
      }
    }
  }
}
```

---

## External Infrastructure Setup

> **NOTE:** Prometheus, Loki, and Alertmanager are external infrastructure components. This section provides guidance for teams that need to set them up.

### Prometheus Setup

**Docker Compose (development):**

```yaml
# docker-compose.observability.yml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.enable-lifecycle'

volumes:
  prometheus_data:
```

**prometheus.yml (minimal config):**

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'app'
    static_configs:
      - targets: ['host.docker.internal:8080']  # Your app's metrics endpoint
    metrics_path: /metrics
```

**Kubernetes (production):**

```bash
# Using Prometheus Operator (recommended)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

### Loki Setup

**Docker Compose (development):**

```yaml
# Add to docker-compose.observability.yml
services:
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
      - loki_data:/loki
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:latest
    volumes:
      - ./promtail-config.yml:/etc/promtail/config.yml
      - /var/log:/var/log:ro
    command: -config.file=/etc/promtail/config.yml

volumes:
  loki_data:
```

**loki-config.yml (minimal):**

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h
```

**Kubernetes (production):**

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true
```

### Alertmanager Setup

**Docker Compose (development):**

```yaml
# Add to docker-compose.observability.yml
services:
  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
```

**alertmanager.yml (minimal):**

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'default'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

receivers:
  - name: 'default'
    # Configure your notification channel (Slack, PagerDuty, email, etc.)
    # Example for Slack:
    # slack_configs:
    #   - api_url: 'https://hooks.slack.com/services/XXX'
    #     channel: '#alerts'
```

**Alert Rules (add to prometheus.yml):**

```yaml
rule_files:
  - /etc/prometheus/alerts/*.yml

# Example alert rules file: alerts/app.yml
groups:
  - name: app
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} over the last 5 minutes"

      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "P99 latency is {{ $value | humanizeDuration }}"
```

---

## Verification Commands

Use these commands to verify your observability stack is working:

### Prometheus

```bash
# Check Prometheus is up
curl -s http://localhost:9090/-/healthy

# Query targets (should show your app)
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[].labels'

# Test a metric query
curl -s 'http://localhost:9090/api/v1/query?query=up' | jq '.data.result'
```

### Loki

```bash
# Check Loki is ready
curl -s http://localhost:3100/ready

# Query recent logs
curl -s 'http://localhost:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={job="app"}' \
  --data-urlencode 'start='$(date -d '15 minutes ago' +%s)000000000 \
  --data-urlencode 'end='$(date +%s)000000000 | jq '.data.result'
```

### Alertmanager

```bash
# Check Alertmanager is up
curl -s http://localhost:9093/-/healthy

# List current alerts
curl -s http://localhost:9093/api/v2/alerts | jq '.[].labels'

# List silences
curl -s http://localhost:9093/api/v2/silences | jq '.[].status'
```

---

## Integration with /patrol and VALIDATION

### How /patrol Uses Observability

| Patrol Mode | Data Source | MCP Used | Queries |
|-------------|-------------|----------|---------|
| `--metrics` | Prometheus | prometheus | Error rates, latency percentiles, resource usage |
| `--logs` | Loki | loki | Error patterns, exception types, log volume |
| `--security` | Alertmanager | alertmanager | Security-related alerts |

### How VALIDATION Monitoring Works

During the 20-minute monitoring window in VALIDATION phase:

1. **Baseline Capture**: Query Prometheus for current error rate, P99 latency, memory
2. **Log Monitoring**: Query Loki for error/warning logs every 5 minutes
3. **Metric Comparison**: Query Prometheus and compare against baseline
4. **Alert Check**: Query Alertmanager for any firing alerts

### Fallback Behavior

If MCP servers are unavailable:

| Missing MCP | Fallback | Impact |
|-------------|----------|--------|
| Prometheus | Skip metrics analysis | No latency/resource monitoring |
| Loki | Skip log analysis | No error pattern detection |
| Alertmanager | Skip alert correlation | Manual alert checking required |

The workflow will warn about missing MCPs but continue with available data sources.

---

## Recommended Dashboards

### Grafana Dashboard (Optional)

If using Grafana with your observability stack, import these dashboards:

- **Application Overview**: Request rate, error rate, latency percentiles
- **Resource Usage**: CPU, memory, disk, network per service
- **Log Volume**: Log rate by level, top error messages

**Dashboard JSON IDs (Grafana.com):**
- Node Exporter: 1860
- Kubernetes Cluster: 6417
- Loki Logs: 12611

---

## Troubleshooting

### MCP Connection Issues

```
Error: Cannot connect to Prometheus MCP
```

1. Verify Prometheus URL is correct in `.claude/mcp.json`
2. Check network connectivity: `curl -s $PROMETHEUS_URL/-/healthy`
3. Ensure MCP server is running: check Claude Code logs
4. Try restarting Claude Code after config changes

### No Metrics/Logs Returned

1. Verify your app is exposing `/metrics` endpoint
2. Check Prometheus targets: `curl $PROMETHEUS_URL/api/v1/targets`
3. Verify log shipper (Promtail/Fluentd) is running
4. Check label selectors in queries match your app's labels

### Alert Rules Not Firing

1. Check rule syntax: `promtool check rules alerts/*.yml`
2. Verify evaluation: Prometheus UI → Alerts tab
3. Check Alertmanager is receiving: Alertmanager UI → Alerts
4. Verify notification config: check receiver settings
