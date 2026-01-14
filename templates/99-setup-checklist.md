## Setup Checklist

### Project Configuration

- [ ] Project name and description confirmed
- [ ] CI platform selected (GitHub/GitLab/Gitea/Filebase)
- [ ] Configuration files created (env vars, feature flags)
- [ ] Deployment URLs configured (staging/production)
- [ ] CLAUDE.md generated from templates

### Secrets Management

> **Note:** Choose either Vault (MCP-based) OR SOPS (file-based encryption) below.

#### Vault (HashiCorp Vault with MCP)

- [ ] Vault MCP installed and working
- [ ] Vault MCP health check passed (`mcp__vault__status`)
- [ ] Vault paths created for project (`secret/{project}/staging`, `secret/{project}/production`)
- [ ] All required secrets configured in Vault
- [ ] Ansible vault lookup configured

#### SOPS (File-based encryption with age)

> **Note:** Use SOPS for projects without Vault infrastructure. Secrets are stored as encrypted files.

- [ ] SOPS installed (`sops --version`)
- [ ] age installed (`age --version`)
- [ ] age key pair generated (`~/.config/sops/age/keys.txt`)
- [ ] `.sops.yaml` configuration created
- [ ] `secrets/` directory created
- [ ] `secrets/staging.yaml` encrypted file created
- [ ] `secrets/production.yaml` encrypted file created
- [ ] All required secrets added to encrypted files
- [ ] Test decryption works (`sops --decrypt secrets/staging.yaml`)

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

#### Filebase Docker CI/CD (Local staging)

> **Note:** Required for VALIDATION phase with Filebase. Enables local CI pipeline simulation and Docker-based staging.

- [ ] Docker installed (`docker --version`)
- [ ] Docker Compose installed (`docker-compose --version` or `docker compose version`)
- [ ] Docker daemon running (`docker ps`)
- [ ] Docker CI initialized (`/ci-filebase docker init`)
- [ ] `.claude/ci-filebase/docker/` directory created
- [ ] `docker-compose.staging.yaml` generated
- [ ] `ci-config.json` created with project commands
- [ ] CI pipeline test passed (`/ci-filebase docker ci`)
- [ ] Local staging deployment works (`/ci-filebase docker deploy staging`)
- [ ] Health check passes (`/ci-filebase docker health`)
- [ ] Staging stopped cleanly (`/ci-filebase docker stop`)

#### Docker Verification Commands

```bash
# Verify Docker installation
docker --version
docker-compose --version

# Check Docker daemon is running
docker ps

# Test docker-compose with staging
docker-compose -f .claude/ci-filebase/docker/docker-compose.staging.yaml config

# Manual staging test
docker-compose -f .claude/ci-filebase/docker/docker-compose.staging.yaml up -d
curl http://localhost:8080/health
docker-compose -f .claude/ci-filebase/docker/docker-compose.staging.yaml down
```

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

| Language              | Server                       | Installation                                           |
| --------------------- | ---------------------------- | ------------------------------------------------------ |
| TypeScript/JavaScript | `typescript-language-server` | `npm install -g typescript-language-server typescript` |
| Python                | `pylsp` (python-lsp-server)  | `pip install python-lsp-server`                        |
| Go                    | `gopls`                      | `go install golang.org/x/tools/gopls@latest`           |
| Rust                  | `rust-analyzer`              | `rustup component add rust-analyzer`                   |
| C/C++                 | `clangd`                     | Install via package manager or LLVM                    |
| Java                  | `jdtls`                      | Download from Eclipse JDT LS releases                  |
| Ruby                  | `solargraph`                 | `gem install solargraph`                               |

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

### Option C: Fully Local with Docker (Filebase + Docker)

1. **Filebase CI** - Local issue/PR tracking via `/ci-filebase`
2. **Docker CI/CD** - Local CI pipeline and staging via `/ci-filebase docker`
3. **No external dependencies** - Everything runs locally in Docker
4. **VALIDATION support** - Full validation phase with local Docker staging
5. **Ideal for** - Local development with real validation, teams without cloud infrastructure

**Setup:**
```bash
# Initialize Filebase CI
/ci-filebase init

# Initialize Docker CI (requires Docker installed)
/ci-filebase docker init

# Test pipeline
/ci-filebase docker ci

# Test staging
/ci-filebase docker deploy staging
/ci-filebase docker health
/ci-filebase docker stop
```

Full observability is recommended for production projects but not required to use the core `/crunch` workflow.
