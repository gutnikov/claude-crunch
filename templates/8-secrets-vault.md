## Vault Secret Management

HashiCorp Vault provides centralized secret management with MCP integration for Claude Code.

> **When to use:** Enterprise environments, teams with existing Vault infrastructure, projects requiring audit trails and dynamic secrets.

### Vault MCP Commands

These Vault MCP commands are used for secret operations in this project.

#### Check Vault Status

Verify Vault server is accessible and unsealed:

```
mcp__vault__status()
```

Returns: `{ "sealed": false, "initialized": true }`

#### Read Secret

Read a secret value from Vault:

```
mcp__vault__read(
    path: "secret/data/{project}/{environment}/{key}"
)
```

**Example - Read database URL for staging:**

```
mcp__vault__read(
    path: "secret/data/myapp/staging/DATABASE_URL"
)
```

#### Write Secret

Create or update a secret:

```
mcp__vault__write(
    path: "secret/data/{project}/{environment}/{key}",
    data: { "value": "{secret_value}" }
)
```

**Example - Write API key for production:**

```
mcp__vault__write(
    path: "secret/data/myapp/production/API_KEY",
    data: { "value": "sk-1234567890abcdef" }
)
```

#### List Secrets

List secret keys at a path (not values):

```
mcp__vault__list(
    path: "secret/metadata/{project}/{environment}"
)
```

#### Delete Secret

Remove a secret:

```
mcp__vault__delete(
    path: "secret/data/{project}/{environment}/{key}"
)
```

---

### Vault CLI Commands

For manual operations or when MCP is unavailable:

#### Read Secret (CLI)

```bash
vault kv get secret/{project}/{environment}/{key}
vault kv get -field=value secret/myapp/staging/DATABASE_URL
```

#### Write Secret (CLI)

```bash
vault kv put secret/{project}/{environment}/{key} value="{secret_value}"
vault kv put secret/myapp/staging/DATABASE_URL value="postgresql://..."
```

#### List Secrets (CLI)

```bash
vault kv list secret/{project}/{environment}
vault kv list secret/myapp/staging
```

#### Delete Secret (CLI)

```bash
vault kv delete secret/{project}/{environment}/{key}
vault kv delete secret/myapp/staging/OLD_KEY
```

---

### Vault Path Structure

Organize secrets by project and environment:

```
secret/
└── {project}/
    ├── staging/
    │   ├── DATABASE_URL
    │   ├── REDIS_URL
    │   ├── API_KEY
    │   └── JWT_SECRET
    └── production/
        ├── DATABASE_URL
        ├── REDIS_URL
        ├── API_KEY
        └── JWT_SECRET
```

---

### Ansible Integration

Use Vault lookup plugin in Ansible:

```yaml
# deploy/ansible/group_vars/all/main.yml
vault_addr: "https://vault.example.com"
vault_secrets_path: "secret/data/{{ project_name }}/{{ env }}"

# deploy/ansible/roles/app/tasks/main.yml
- name: Get secrets from Vault
  set_fact:
    db_password: "{{ lookup('hashi_vault',
        'secret=secret/data/{{ project_name }}/{{ env }}/DATABASE_URL:value
        url={{ vault_addr }}') }}"
    api_key: "{{ lookup('hashi_vault',
        'secret=secret/data/{{ project_name }}/{{ env }}/API_KEY:value
        url={{ vault_addr }}') }}"

- name: Deploy application config
  template:
    src: app.yaml.j2
    dest: /etc/app/config.yaml
```

---

### MCP Configuration

Add Vault MCP to Claude configuration:

```json
{
  "mcpServers": {
    "vault": {
      "command": "vault-mcp",
      "args": ["--addr", "https://vault.example.com"],
      "env": {
        "VAULT_TOKEN": "hvs.your_token_here"
      }
    }
  }
}
```

**Authentication options:**

| Method | Environment Variable | Use Case |
|--------|---------------------|----------|
| Token | `VAULT_TOKEN` | Development, CI/CD |
| AppRole | `VAULT_ROLE_ID`, `VAULT_SECRET_ID` | Production automation |
| Kubernetes | `VAULT_K8S_ROLE` | Kubernetes deployments |

---

### Quick Reference Table

| Operation | MCP Command | CLI Command |
|-----------|-------------|-------------|
| Check status | `mcp__vault__status` | `vault status` |
| Read secret | `mcp__vault__read` | `vault kv get` |
| Write secret | `mcp__vault__write` | `vault kv put` |
| List secrets | `mcp__vault__list` | `vault kv list` |
| Delete secret | `mcp__vault__delete` | `vault kv delete` |

---

### Initial Setup

1. **Install Vault MCP server:**
   ```bash
   npm install -g @anthropic/vault-mcp
   ```

2. **Configure MCP in Claude settings** (see above)

3. **Create project secret paths:**
   ```bash
   vault kv put secret/{project}/staging/placeholder value="setup"
   vault kv put secret/{project}/production/placeholder value="setup"
   ```

4. **Verify connectivity:**
   ```
   mcp__vault__status()
   ```

5. **Add required secrets** for your project

---

### Security Best Practices

1. **Use short-lived tokens** — Avoid long-lived root tokens
2. **Enable audit logging** — Track all secret access
3. **Use policies** — Restrict access to specific paths
4. **Rotate secrets** — Update credentials regularly
5. **Use dynamic secrets** — For databases, cloud providers when possible
