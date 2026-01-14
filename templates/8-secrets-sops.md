## SOPS Secret Management

SOPS (Secrets OPerationS) with age encryption provides file-based secret management without external infrastructure.

> **When to use:** Projects without Vault infrastructure, local development, air-gapped environments, or teams preferring git-based secret versioning.

### SOPS Commands

These commands are used for secret operations in this project. SOPS secrets are stored as encrypted files that can be safely committed to git.

#### Decrypt and Read Secret File

Read all secrets from an encrypted file:

```bash
sops --decrypt secrets/{environment}.yaml
```

**Example - Read staging secrets:**

```bash
sops --decrypt secrets/staging.yaml
```

#### Read Single Secret Value

Extract a specific secret:

```bash
sops --decrypt --extract '["DATABASE_URL"]' secrets/staging.yaml
```

#### Edit Secrets (In-Place)

Open encrypted file in editor, auto-encrypts on save:

```bash
sops secrets/{environment}.yaml
```

**Example - Edit production secrets:**

```bash
sops secrets/production.yaml
```

#### Create New Encrypted File

Create a new secrets file:

```bash
sops --encrypt --age {age_public_key} secrets/{environment}.yaml.dec > secrets/{environment}.yaml
```

Or create and edit in one step:

```bash
sops secrets/new-environment.yaml
```

#### Update Single Secret

Decrypt, modify, and re-encrypt:

```bash
# Method 1: Edit interactively
sops secrets/staging.yaml

# Method 2: Set via command
sops --set '["API_KEY"] "new-api-key-value"' secrets/staging.yaml
```

---

### File Structure

Organize encrypted secrets by environment:

```
secrets/
├── .sops.yaml          # SOPS configuration
├── staging.yaml        # Encrypted staging secrets
├── production.yaml     # Encrypted production secrets
└── keys/
    ├── staging.txt     # age public key for staging (safe to commit)
    └── production.txt  # age public key for production (safe to commit)
```

**Note:** Private keys (`*.age` files) should NEVER be committed. Store them securely outside the repo.

---

### SOPS Configuration

Create `.sops.yaml` in project root:

```yaml
# .sops.yaml
creation_rules:
  # Staging secrets - encrypted with staging key
  - path_regex: secrets/staging\.yaml$
    age: >-
      age1staging0publickey...

  # Production secrets - encrypted with production key
  - path_regex: secrets/production\.yaml$
    age: >-
      age1production0publickey...

  # Default rule for other files
  - path_regex: secrets/.*\.yaml$
    age: >-
      age1default0publickey...
```

---

### Secret File Format

Encrypted files contain YAML with encrypted values:

```yaml
# secrets/staging.yaml (after encryption)
DATABASE_URL: ENC[AES256_GCM,data:...,type:str]
REDIS_URL: ENC[AES256_GCM,data:...,type:str]
API_KEY: ENC[AES256_GCM,data:...,type:str]
JWT_SECRET: ENC[AES256_GCM,data:...,type:str]
sops:
  age:
    - recipient: age1...
      enc: |
        -----BEGIN AGE ENCRYPTED FILE-----
        ...
        -----END AGE ENCRYPTED FILE-----
  lastmodified: "2024-01-14T10:00:00Z"
  version: 3.8.1
```

Decrypted view:

```yaml
DATABASE_URL: postgresql://user:password@host:5432/db
REDIS_URL: redis://localhost:6379
API_KEY: sk-1234567890abcdef
JWT_SECRET: your-jwt-signing-key
```

---

### Ansible Integration

Use SOPS with Ansible for deployment:

```yaml
# deploy/ansible/roles/app/tasks/main.yml
- name: Decrypt secrets
  community.sops.load_vars:
    file: "{{ playbook_dir }}/../secrets/{{ env }}.yaml"
    name: secrets

- name: Deploy application config
  template:
    src: app.yaml.j2
    dest: /etc/app/config.yaml
  vars:
    db_password: "{{ secrets.DATABASE_URL }}"
    api_key: "{{ secrets.API_KEY }}"
```

**Or using shell command:**

```yaml
- name: Get secrets
  shell: sops --decrypt secrets/{{ env }}.yaml
  register: secrets_raw
  delegate_to: localhost

- name: Parse secrets
  set_fact:
    secrets: "{{ secrets_raw.stdout | from_yaml }}"
```

---

### age Key Management

#### Generate New Key Pair

```bash
# Generate key pair
age-keygen -o keys/staging.age

# Output shows public key:
# Public key: age1...

# Store public key for .sops.yaml
age-keygen -y keys/staging.age > keys/staging.txt
```

#### Key Storage

| Key Type    | Location                                      | Git            |
| ----------- | --------------------------------------------- | -------------- |
| Public key  | `secrets/keys/*.txt` or `.sops.yaml`          | Safe to commit |
| Private key | `~/.config/sops/age/keys.txt` or secure vault | NEVER commit   |

#### Configure age Key Location

```bash
# Set key location via environment
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt

# Or specify in command
sops --age-key-file ~/.age/production.age --decrypt secrets/production.yaml
```

---

### Quick Reference Table

| Operation        | Command                                                 |
| ---------------- | ------------------------------------------------------- |
| Decrypt file     | `sops --decrypt secrets/{env}.yaml`                     |
| Encrypt file     | `sops --encrypt file.yaml > file.enc.yaml`              |
| Edit in-place    | `sops secrets/{env}.yaml`                               |
| Get single value | `sops --decrypt --extract '["KEY"]' secrets/{env}.yaml` |
| Set single value | `sops --set '["KEY"] "value"' secrets/{env}.yaml`       |
| Generate age key | `age-keygen -o keyfile.age`                             |
| Get public key   | `age-keygen -y keyfile.age`                             |

---

### Initial Setup

1. **Install SOPS and age:**

   ```bash
   # macOS
   brew install sops age

   # Linux
   # Download from releases:
   # https://github.com/getsops/sops/releases
   # https://github.com/FiloSottile/age/releases
   ```

2. **Generate age key pair:**

   ```bash
   mkdir -p ~/.config/sops/age
   age-keygen -o ~/.config/sops/age/keys.txt
   ```

3. **Note your public key** (from output or `age-keygen -y`)

4. **Create `.sops.yaml` configuration** (see above)

5. **Create initial secrets files:**

   ```bash
   # Create staging secrets
   echo "DATABASE_URL: postgresql://..." > secrets/staging.yaml.dec
   sops --encrypt secrets/staging.yaml.dec > secrets/staging.yaml
   rm secrets/staging.yaml.dec

   # Or create and edit directly
   sops secrets/staging.yaml
   ```

6. **Add to .gitignore:**

   ```gitignore
   # Private keys - NEVER commit
   *.age
   keys.txt

   # Decrypted files - temporary
   *.dec
   ```

---

### CI/CD Integration

For automated deployments, inject the private key:

```yaml
# GitHub Actions
- name: Setup SOPS
  run: |
    mkdir -p ~/.config/sops/age
    echo "${{ secrets.SOPS_AGE_KEY }}" > ~/.config/sops/age/keys.txt

- name: Decrypt secrets
  run: sops --decrypt secrets/${{ env.ENVIRONMENT }}.yaml > secrets.yaml
```

```yaml
# GitLab CI
deploy:
  before_script:
    - mkdir -p ~/.config/sops/age
    - echo "$SOPS_AGE_KEY" > ~/.config/sops/age/keys.txt
  script:
    - sops --decrypt secrets/${CI_ENVIRONMENT_NAME}.yaml > secrets.yaml
```

---

### Security Best Practices

1. **Never commit private keys** — Store in password manager or secure vault
2. **Use separate keys per environment** — Production key should be different from staging
3. **Rotate keys periodically** — Re-encrypt with new key, update all team members
4. **Audit access** — Track who has access to private keys
5. **Use `.gitignore`** — Prevent accidental commits of decrypted files
6. **Encrypt entire values** — Don't leave partial secrets unencrypted

---

### Comparison with Vault

| Feature         | SOPS + age              | Vault                  |
| --------------- | ----------------------- | ---------------------- |
| Infrastructure  | None (files)            | Server required        |
| Git integration | Encrypted files in repo | External storage       |
| Audit trail     | Git history             | Built-in audit log     |
| Dynamic secrets | No                      | Yes                    |
| Access control  | Key distribution        | Policies, roles        |
| Offline access  | Yes                     | Requires server        |
| Complexity      | Low                     | High                   |
| Best for        | Small teams, local dev  | Enterprise, compliance |
