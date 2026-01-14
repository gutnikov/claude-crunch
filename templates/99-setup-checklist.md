## Setup Checklist

### Project Configuration

- [ ] Project name and description confirmed
- [ ] CI platform selected (GitHub/GitLab/Gitea)
- [ ] Configuration files created (env vars, feature flags)
- [ ] Deployment URLs configured (staging/production)
- [ ] CLAUDE.md generated from templates

### Secrets Management

- [ ] Vault MCP installed and working
- [ ] Vault MCP health check passed
- [ ] All required secrets configured

### CI/CD Integration

- [ ] CI MCP installed and working
- [ ] CI MCP health check passed
- [ ] CI config ready
- [ ] Labels created in CI platform
- [ ] Branch protection rules configured
- [ ] CI pipeline tested successfully

### Observability Infrastructure

> **Note:** Required for `/patrol` skill and VALIDATION continuous monitoring. Can be skipped if observability is not needed, but `/patrol --logs` and `/patrol --metrics` will be unavailable.

#### Prometheus (Metrics)

- [ ] Prometheus server accessible (or skip metrics monitoring)
- [ ] Prometheus URL configured: `____________________`
- [ ] Prometheus MCP installed
- [ ] Prometheus MCP added to `.claude/mcp.json`
- [ ] Prometheus MCP health check passed (query `up` metric)
- [ ] App `/metrics` endpoint scraped by Prometheus

#### Loki (Logs)

- [ ] Loki server accessible (or skip log monitoring)
- [ ] Loki URL configured: `____________________`
- [ ] Loki MCP installed
- [ ] Loki MCP added to `.claude/mcp.json`
- [ ] Loki MCP health check passed (query recent logs)
- [ ] App logs shipping to Loki (Promtail/Fluentd configured)

#### Alertmanager (Alerts)

- [ ] Alertmanager server accessible (or skip alert correlation)
- [ ] Alertmanager URL configured: `____________________`
- [ ] Alertmanager MCP installed
- [ ] Alertmanager MCP added to `.claude/mcp.json`
- [ ] Alertmanager MCP health check passed (list alerts)
- [ ] Basic alert rules configured in Prometheus

#### Observability Verification

```bash
# Run these commands to verify observability stack:

# Prometheus
curl -s http://<PROMETHEUS_URL>/-/healthy
curl -s 'http://<PROMETHEUS_URL>/api/v1/query?query=up'

# Loki
curl -s http://<LOKI_URL>/ready

# Alertmanager
curl -s http://<ALERTMANAGER_URL>/-/healthy
```

- [ ] All verification commands return healthy status

### Knowledge System (Optional)

- [ ] Knowledge directory created (`.claude/knowledge/`)
- [ ] Knowledge index initialized
- [ ] Knowledge MCP configured (if using external storage)

### Multi-Agent Orchestration (Optional)

> **Note:** Orchestration is automatically enabled when issue complexity is HIGH (score >= 5) or spans multiple domains.

- [ ] Review `templates/hierarchy-config.md` for agent tier structure
- [ ] Customize agent routing weights in `.claude/routing-config.json` (if needed)
- [ ] Configure checkpoint retention in `.claude/checkpoint-config.json` (if needed)
- [ ] Review veto rules in `templates/veto-rules.md`

#### Orchestration Directory Structure

```
.claude/
├── acp/
│   ├── message-log.json       # ACP message history
│   └── veto-log.json          # Veto decision history
├── crunch/{issue}/
│   └── checkpoints/           # Workflow checkpoints
├── knowledge/
│   └── index.json             # Includes agent_metrics
├── routing-config.json        # Adaptive routing config
└── checkpoint-config.json     # Checkpoint settings
```

### Final Verification

- [ ] A `Setup CI` issue is created
- [ ] Setup issue crunched to ready-to-merge phase
- [ ] `/patrol --dry-run` executes without MCP errors (or expected "MCP unavailable" warnings)

---

## Quick Reference: MCP Configuration

Combined `.claude/mcp.json` for all observability MCPs:

```json
{
  "mcpServers": {
    "prometheus": {
      "command": "mcp-server-prometheus",
      "args": [],
      "env": {
        "PROMETHEUS_URL": "http://prometheus.your-infra.local:9090"
      }
    },
    "loki": {
      "command": "mcp-server-loki",
      "args": [],
      "env": {
        "LOKI_URL": "http://loki.your-infra.local:3100"
      }
    },
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

## Minimum Viable Setup

For teams without existing observability infrastructure, the minimum setup is:

1. **CI MCP only** - Issues, PRs, labels work
2. **Skip observability MCPs** - `/patrol --code` and `/patrol --deps` still work
3. **Manual validation** - Skip continuous monitoring in VALIDATION phase

Full observability is recommended for production projects but not required to use the core `/crunch` workflow.
