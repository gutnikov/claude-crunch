## CI/CD Configuration

### CI Platform

> **Note:** Select one platform and remove unused rows.

<!--
TEMPLATE: Keep only the platform used by your project. Remove this comment after configuration.

| Platform | Status   | Config File          | MCP Server                                                                                              |
| -------- | -------- | -------------------- | ------------------------------------------------------------------------------------------------------- |
| GitHub   | {active} | `.github/workflows/` | [github/github-mcp-server](https://github.com/github/github-mcp-server)                                 |
| GitLab   | {active} | `.gitlab-ci.yml`     | [GitLab Duo MCP](https://docs.gitlab.com/user/gitlab_duo/model_context_protocol/mcp_server/) (embedded) |
| Gitea    | {active} | `.gitea/workflows/`  | [MushroomFleet/gitea-mcp](https://github.com/MushroomFleet/gitea-mcp)                                   |
-->

### Environments

| Environment | Purpose              | Branch    | Auto-deploy |
| ----------- | -------------------- | --------- | ----------- |
| Staging     | Pre-production tests | `develop` | Yes         |
| Production  | Live environment     | `main`    | Manual      |

### Pipeline Steps

<!--
TEMPLATE: Select and customize stages based on your project. Remove this comment after configuration.

#### Stage 1: Validation ⚑ MANDATORY

| Step         | Description                     | Example Tools               |
| ------------ | ------------------------------- | --------------------------- |
| lint         | Code style, formatting          | {prettier, black, golint}   |
| typecheck    | Static type analysis            | {tsc, mypy, go vet}         |
| config-lint  | Config & Ansible validation     | {yamllint, ansible-lint}    |
| secrets-scan | Detect leaked credentials       | {gitleaks, trufflehog}      |
| commit-lint  | Conventional commits validation | {commitlint}                |

##### Config Lint Job

```yaml
config-lint:
  stage: validate
  script:
    - yamllint config/
    - ansible-lint deploy/ansible/
    - ansible-playbook deploy/ansible/playbooks/deploy.yml --syntax-check
```

#### Stage 2: Security (optional, recommended)

| Step            | Description                         | Example Tools             |
| --------------- | ----------------------------------- | ------------------------- |
| sast            | Static Application Security Testing | {semgrep, codeql}         |
| dependency-scan | CVE check                           | {dependabot, snyk, trivy} |
| license-check   | OSS license compliance              | {fossa, licensee}         |

#### Stage 3: Build ⚑ MANDATORY

| Step            | Description                    | Example Tools             |
| --------------- | ------------------------------ | ------------------------- |
| build           | Compile source / bundle assets | {gcc, go build, webpack}  |
| container-build | Build Docker/OCI images        | {docker, podman, buildah} |
| artifact-upload | Store build outputs            | {S3, GCS, registry}       |

#### Stage 4: Test ⚑ MANDATORY

| Step        | Description                 | Example Tools              |
| ----------- | --------------------------- | -------------------------- |
| unit-test   | Fast isolated tests         | {pytest, jest, go test}    |
| integration | Service/API integration     | {testcontainers, httptest} |
| e2e         | End-to-end / UI tests       | {playwright, cypress}      |
| coverage    | Code coverage gate (≥{80}%) | {coverage.py, c8, go tool} |

#### Stage 5: Quality Gates (optional)

| Step        | Description               | Example Tools            |
| ----------- | ------------------------- | ------------------------ |
| sonar       | Quality analysis          | {SonarQube, SonarCloud}  |
| performance | Benchmark / load tests    | {k6, locust, wrk}        |
| size-check  | Bundle/binary size limits | {bundlesize, size-limit} |

#### Stage 6: Deploy ⚑ MANDATORY

| Step           | Description                           | Example Tools              |
| -------------- | ------------------------------------- | -------------------------- |
| deploy-staging | Push to staging environment           | {kubectl, terraform, helm} |
| smoke-test     | Basic health verification post-deploy | {curl, httpie, scripts}    |
| deploy-prod    | Push to production (manual approval)  | {kubectl, terraform, helm} |

#### Stage 7: Post-Deploy (optional, recommended)

| Step         | Description                     | Example Tools              |
| ------------ | ------------------------------- | -------------------------- |
| health-check | Verify service health endpoints | {curl, monitoring probes}  |
| notify       | Notifications                   | {Slack, Teams, email}      |
| tag-release  | Git tag + changelog generation  | {semantic-release, chglog} |
| rollback     | Auto-rollback on failure        | {ArgoCD, Flux, scripts}    |
-->
