## Configuration

> **Note:** This project follows a "single source of truth" approach to avoid config mess.

### Configuration Principles

1. **Single source of truth** — App behavior vs deployment concerns are split
2. **Generation over hand-edits** — Runtime configs generated from templates
3. **Controlled overrides** — Narrow, explicit, centralized
4. **Clear secret precedence** — Deterministic "where did this value come from?"
5. **Validation in CI** — Catch config mistakes early

### Configuration Ownership

| What                        | Where (Source of Truth)  | Managed By                     |
| --------------------------- | ------------------------ | ------------------------------ |
| App behavior, feature flags | `config/*.yaml` (repo)   | Developers                     |
| Paths, users, infra         | `deploy/ansible/` (repo) | DevOps                         |
| Secrets, credentials        | Vault                    | DevOps + Vault                 |
| Runtime config (generated)  | `/etc/{app}/` (server)   | Ansible (do not edit manually) |

### Directory Structure

```
config/
├── app.yaml              # Application settings (source of truth)
├── features.yaml         # Feature flags
└── {component}.yaml      # Component-specific config

deploy/
├── CONFIG.md             # Configuration management guide
├── ansible/
│   ├── inventory/
│   │   ├── staging
│   │   └── production
│   ├── group_vars/
│   │   ├── all/
│   │   │   └── main.yml  # Global defaults
│   │   ├── staging/
│   │   │   └── main.yml  # Staging overrides
│   │   └── production/
│   │       └── main.yml  # Production overrides
│   ├── roles/
│   │   └── app/
│   │       └── templates/
│   │           └── app.yaml.j2  # Generated config template
│   └── playbooks/
```

### Config Generation (No Hand-Edits)

Runtime configs are **generated**, not edited on servers.

```yaml
# deploy/ansible/roles/app/templates/app.yaml.j2
# ⚠️ THIS FILE IS GENERATED - DO NOT EDIT MANUALLY
# Source: config/app.yaml + Ansible variables
# Regenerate: ansible-playbook playbooks/deploy.yml

{% include 'config/app.yaml' %}

# Deployment overrides (Ansible-managed)
log_level: {{ log_level | default('info') }}
output_dir: {{ output_dir }}
```

**Change flow:**

1. Edit source config in `config/*.yaml` or Ansible vars
2. Commit to git
3. Run deployment → config regenerated

### Controlled Overrides

Overrides are **narrow and centralized**, not scattered.

| Override Type  | Location                        | Example                    |
| -------------- | ------------------------------- | -------------------------- |
| Output paths   | Ansible group_vars              | `output_dir: /var/lib/app` |
| Log level      | Ansible group_vars (per-env)    | `log_level: debug`         |
| Service params | Systemd unit (Ansible template) | `ExecStart ... -o /path`   |

```yaml
# deploy/ansible/group_vars/staging/main.yml
log_level: debug
output_dir: /var/lib/app/staging

# deploy/ansible/group_vars/production/main.yml
log_level: warn
output_dir: /var/lib/app/production
```

### Secret Precedence (Vault)

Secrets follow a **clear precedence** — you can always trace where a value comes from.

```
Vault (highest) → Environment variable → Ansible group_vars (lowest)
```

| Precedence | Source                | Use Case                   |
| ---------- | --------------------- | -------------------------- |
| 1 (high)   | Vault                 | Production secrets         |
| 2          | Environment variables | CI/CD injection, local dev |
| 3 (low)    | Ansible group_vars    | Non-sensitive defaults     |

#### Vault Integration

```yaml
# deploy/ansible/group_vars/all/main.yml
# Vault paths for secrets lookup
vault_addr: "https://vault.example.com"
vault_secrets_path: "secret/data/app/{{ env }}"

# In playbook/role - lookup from Vault
db_password: "{{ lookup('hashi_vault', 'secret=secret/data/app/{{ env }}:db_password') }}"
api_key: "{{ lookup('hashi_vault', 'secret=secret/data/app/{{ env }}:api_key') }}"
```

#### What NOT to Do

```yaml
# ❌ DON'T: Commit secrets to repo
db_password: "actual-password-here"

# ❌ DON'T: Duplicate secrets across files
# staging/secrets.yml AND production/secrets.yml with same keys

# ❌ DON'T: Edit generated config on server
ssh server "vim /etc/app/config.yaml"  # NO!

# ✅ DO: Change source → commit → redeploy
```

### Environment Variables

> **Note:** Define required environment variables.

<!--
TEMPLATE: List environment variables. Remove this comment after configuration.

| Variable     | Description          | Secret | Source              |
| ------------ | -------------------- | ------ | ------------------- |
| DATABASE_URL | Database connection  | Yes    | Vault               |
| REDIS_URL    | Redis connection     | Yes    | Vault               |
| API_KEY      | External API key     | Yes    | Vault               |
| LOG_LEVEL    | Logging verbosity    | No     | Ansible group_vars  |
| APP_ENV      | Environment name     | No     | Ansible group_vars  |
-->

### CI Config Validation

Add config linting to CI pipeline to catch mistakes early.

```yaml
# .github/workflows/config-lint.yml / .gitlab-ci.yml
config-lint:
  stage: validate
  script:
    - yamllint config/
    - ansible-lint deploy/ansible/
    - ansible-playbook deploy/ansible/playbooks/deploy.yml --syntax-check
```

### Feature Flags

> **Note:** Feature flags live in repo config, not deployment.

<!--
TEMPLATE: Define feature flags. Remove this comment after configuration.

```yaml
# config/features.yaml (source of truth)
features:
  new_ui: false
  beta_api: false
  debug_mode: false
```

Override per-environment in Ansible:

```yaml
# deploy/ansible/group_vars/staging/main.yml
feature_overrides:
  new_ui: true
  debug_mode: true
```
-->
