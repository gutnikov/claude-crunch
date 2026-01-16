## Secret Management

> **Note:** Secrets are managed separately from configuration. Never commit secrets to git.

### Secret Management Approach

> **Note:** Select one approach and remove unused rows.

<!--
TEMPLATE: Keep only the approach used by your project. Remove this comment after configuration.

| Approach | Status   | Storage            | Description                                       |
| -------- | -------- | ------------------ | ------------------------------------------------- |
| Vault    | {active} | HashiCorp Vault    | Centralized secret management with MCP integration |
| SOPS     | {active} | Encrypted files    | File-based encryption using SOPS + age            |
-->

### Secret Principles

1. **Never commit plaintext secrets** — Use encryption or external storage
2. **Environment separation** — Different secrets for staging/production
3. **Least privilege** — Access only what's needed
4. **Audit trail** — Track who accessed what secrets
5. **Rotation support** — Secrets can be updated without code changes

### Secret Precedence

Secrets follow a **clear precedence** — you can always trace where a value comes from.

```
External Secret Store (highest) → Environment variable → Config defaults (lowest)
```

| Precedence | Source                | Use Case                   |
| ---------- | --------------------- | -------------------------- |
| 1 (high)   | Vault / SOPS files    | Production secrets         |
| 2          | Environment variables | CI/CD injection, local dev |
| 3 (low)    | Config defaults       | Non-sensitive defaults     |

### Required Secrets

> **Note:** List all secrets needed by your application.

<!--
TEMPLATE: Define required secrets. Remove this comment after configuration.

| Secret Name   | Description              | Environments       | Rotation   |
| ------------- | ------------------------ | ------------------ | ---------- |
| DATABASE_URL  | PostgreSQL connection    | staging, production | Quarterly  |
| REDIS_URL     | Redis connection string  | staging, production | Quarterly  |
| API_KEY       | External API key         | staging, production | Monthly    |
| JWT_SECRET    | JWT signing key          | staging, production | Yearly     |
-->

### Secret Operations

This project requires the following secret operations:

| Operation         | When Used               | Description                               |
| ----------------- | ----------------------- | ----------------------------------------- |
| **Read secret**   | Deployment, runtime     | Retrieve secret value for application use |
| **Write secret**  | Initial setup, rotation | Create or update a secret                 |
| **List secrets**  | Audit, verification     | List available secrets (not values)       |
| **Verify secret** | Setup, CI/CD            | Check if secret exists and is readable    |
| **Delete secret** | Cleanup, rotation       | Remove old/unused secrets                 |

### What NOT to Do

```yaml
# ❌ DON'T: Commit secrets to repo
db_password: "actual-password-here"

# ❌ DON'T: Log secrets
logger.info(f"Connecting with password: {db_password}")

# ❌ DON'T: Hardcode secrets in code
API_KEY = "sk-1234567890abcdef"

# ❌ DON'T: Share secrets across environments
# Using same database password for staging and production

# ✅ DO: Use secret management system
db_password = get_secret("DATABASE_URL")

# ✅ DO: Use environment-specific secrets
db_password = get_secret(f"DATABASE_URL_{environment}")
```

### Ansible Integration

Regardless of secret approach, Ansible can inject secrets during deployment:

```yaml
# deploy/ansible/roles/app/tasks/main.yml
- name: Deploy application config
  template:
    src: app.yaml.j2
    dest: /etc/app/config.yaml
  vars:
    db_password: "{{ lookup('env', 'DATABASE_URL') | default(db_password_default) }}"
```

For approach-specific Ansible integration, see the respective approach documentation.
