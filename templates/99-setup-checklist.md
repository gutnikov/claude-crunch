## Setup Checklist

### Project Configuration

- [ ] Project name and description confirmed
- [ ] CI platform selected (GitHub/GitLab/Gitea/Filebase)
- [ ] Configuration files created (env vars, feature flags)
- [ ] Deployment URLs configured (staging/production)
- [ ] CLAUDE.md generated from templates

### Secrets Management

- [ ] Vault MCP installed and working
- [ ] Vault MCP health check passed
- [ ] All required secrets configured

### CI/CD Integration

> **Note:** Choose either MCP-based CI (GitHub/GitLab/Gitea) OR Filebase CI below.

#### MCP-based CI (GitHub/GitLab/Gitea)

- [ ] CI MCP installed and working
- [ ] CI MCP health check passed
- [ ] CI config ready
- [ ] Labels created in CI platform
- [ ] Branch protection rules configured
- [ ] CI pipeline tested successfully

#### Filebase CI (Local files)

> **Note:** Use Filebase for projects without external CI platform access. Issues and PRs are stored locally in `.claude/ci-filebase/`.

- [ ] Filebase initialized (`/ci-filebase init`)
- [ ] `.claude/ci-filebase/` directory created
- [ ] `labels.json` created with default workflow labels
- [ ] `counter.json` initialized
- [ ] Test issue created (`/ci-filebase issue create "Test"`)
- [ ] Test issue labels work (`/ci-filebase issue label 1 type:feature`)

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

### Language Server Plugins (Optional)

> **Note:** Language Server Protocol (LSP) integration enables code intelligence features like go-to-definition, find-references, and hover documentation. This enhances Claude Code's ability to navigate and understand your codebase.

#### Recommended LSP Servers by Language

| Language | Server | Installation |
|----------|--------|--------------|
| TypeScript/JavaScript | `typescript-language-server` | `npm install -g typescript-language-server typescript` |
| Python | `pylsp` (python-lsp-server) | `pip install python-lsp-server` |
| Go | `gopls` | `go install golang.org/x/tools/gopls@latest` |
| Rust | `rust-analyzer` | `rustup component add rust-analyzer` |
| C/C++ | `clangd` | Install via package manager or LLVM |
| Java | `jdtls` | Download from Eclipse JDT LS releases |
| Ruby | `solargraph` | `gem install solargraph` |

#### Installation Steps

1. **Install the LSP server** for your project's language(s) using the commands above

2. **Verify installation** by running the server with `--version`:
   ```bash
   typescript-language-server --version
   pylsp --version
   gopls version
   ```

3. **Configure in Claude Code** by adding to your settings (`.claude/settings.json`):
   ```json
   {
     "lsp": {
       "servers": {
         "typescript": {
           "command": "typescript-language-server",
           "args": ["--stdio"]
         },
         "python": {
           "command": "pylsp"
         },
         "go": {
           "command": "gopls",
           "args": ["serve"]
         }
       }
     }
   }
   ```

#### LSP Checklist

- [ ] Identified primary language(s) in project
- [ ] LSP server(s) installed for primary language(s)
- [ ] LSP server(s) verified working (`--version` check)
- [ ] LSP configuration added to Claude Code settings (if needed)

#### Available LSP Operations

Once configured, Claude Code can use these LSP features:

- **goToDefinition** - Jump to where a symbol is defined
- **findReferences** - Find all usages of a symbol
- **hover** - Get documentation and type info
- **documentSymbol** - List all symbols in a file
- **workspaceSymbol** - Search symbols across the project
- **goToImplementation** - Find interface implementations
- **incomingCalls/outgoingCalls** - Trace call hierarchies

---

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

For teams without existing infrastructure, the minimum setup is:

### Option A: With External CI (GitHub/GitLab/Gitea)

1. **CI MCP only** - Issues, PRs, labels work
2. **Skip observability MCPs** - `/patrol --code` and `/patrol --deps` still work
3. **Manual validation** - Skip continuous monitoring in VALIDATION phase

### Option B: Fully Local (Filebase)

1. **Filebase CI** - Local issue/PR tracking via `/ci-filebase`
2. **No MCP required** - Everything stored in `.claude/ci-filebase/`
3. **Skip observability** - Manual validation only
4. **Ideal for** - Air-gapped environments, rapid prototyping, single-developer projects

Full observability is recommended for production projects but not required to use the core `/crunch` workflow.
