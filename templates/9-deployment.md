## Deployment

> **Note:** Both staging and production deployments are manual actions.  
> Can be triggered via CI pipeline (manual job) or MCP commands.  
> All deployments use **Ansible playbooks**.

### Staging Deployment

> **Note:** Configure staging environment details for your project.

<!--
TEMPLATE: Customize for your staging environment. Remove this comment after configuration.

| Property       | Value                         |
| -------------- | ----------------------------- |
| Environment    | staging                       |
| URL            | {https://staging.example.com} |
| Branch         | any feature branch            |
| Trigger        | Manual (CI job or MCP)        |
| Infrastructure | Docker (via Ansible)          |

#### Staging Access

| Resource   | Endpoint                             |
| ---------- | ------------------------------------ |
| App        | {https://staging.example.com}        |
| API        | {https://api.staging.example.com}    |
| Logs       | {Grafana / CloudWatch / Kibana link} |
| Monitoring | {Prometheus / Datadog link}          |

#### Deploy via CI (manual job)

```yaml
deploy-staging:
  stage: deploy
  when: manual
  script:
    - ansible-playbook -i inventory/staging playbooks/deploy-docker.yml -e "tag=${TAG}"
  environment:
    name: staging
    url: {https://staging.example.com}
```

#### Deploy via MCP

```
# Using CI MCP to trigger deployment
trigger_workflow / trigger_pipeline → deploy-staging
```

#### Ansible Playbook: Docker

```yaml
# playbooks/deploy-docker.yml
- hosts: staging
  vars:
    app_image: "{registry}/app:{{ tag }}"
  tasks:
    - name: Pull latest image
      docker_image:
        name: "{{ app_image }}"
        source: pull

    - name: Deploy with docker-compose
      docker_compose:
        project_src: /opt/app
        pull: yes
        state: present
      environment:
        APP_IMAGE: "{{ app_image }}"
```

#### Rollback

```bash
ansible-playbook -i inventory/staging playbooks/deploy-docker.yml -e "tag=${PREV_TAG}"
```
-->

### Production Deployment

> **Note:** Configure production environment details for your project.

<!--
TEMPLATE: Customize for your production environment. Remove this comment after configuration.

| Property       | Value                    |
| -------------- | ------------------------ |
| Environment    | production               |
| URL            | {https://example.com}    |
| Branch         | `main`                   |
| Trigger        | Manual (CI job or MCP)   |
| Infrastructure | Bare metal (via Ansible) |

#### Production Access

| Resource    | Endpoint                             |
| ----------- | ------------------------------------ |
| App         | {https://example.com}                |
| API         | {https://api.example.com}            |
| Logs        | {Grafana / CloudWatch / Kibana link} |
| Monitoring  | {Prometheus / Datadog link}          |
| Status Page | {https://status.example.com}         |

#### Deploy via CI (manual job)

```yaml
deploy-production:
  stage: deploy
  when: manual
  script:
    - ansible-playbook -i inventory/production playbooks/deploy-bare-metal.yml -e "tag=${TAG}"
  environment:
    name: production
    url: {https://example.com}
```

#### Deploy via MCP

```
# Using CI MCP to trigger deployment
trigger_workflow / trigger_pipeline → deploy-production
```

#### Ansible Playbook: Bare Metal

```yaml
# playbooks/deploy-bare-metal.yml
- hosts: production
  vars:
    app_version: "{{ tag }}"
    app_dir: /opt/app
    releases_dir: "{{ app_dir }}/releases"
    current_link: "{{ app_dir }}/current"
  tasks:
    - name: Create release directory
      file:
        path: "{{ releases_dir }}/{{ app_version }}"
        state: directory

    - name: Download release artifact
      get_url:
        url: "{artifact_url}/release-{{ app_version }}.tar.gz"
        dest: "{{ releases_dir }}/{{ app_version }}.tar.gz"

    - name: Extract release
      unarchive:
        src: "{{ releases_dir }}/{{ app_version }}.tar.gz"
        dest: "{{ releases_dir }}/{{ app_version }}"
        remote_src: yes

    - name: Update current symlink
      file:
        src: "{{ releases_dir }}/{{ app_version }}"
        dest: "{{ current_link }}"
        state: link

    - name: Restart application
      systemd:
        name: app
        state: restarted
```

#### Rollback

```bash
ansible-playbook -i inventory/production playbooks/deploy-bare-metal.yml -e "tag=${PREV_TAG}"
```

#### Production Checklist

- [ ] All tests passing on `main`
- [ ] Staging smoke tests verified
- [ ] Release notes prepared
- [ ] Rollback plan confirmed
-->
