## Agent System

This project uses a multi-agent system for development workflow management with hierarchical coordination.

### Available Agents

| Agent               | Purpose               | When to Use                                                      |
| ------------------- | --------------------- | ---------------------------------------------------------------- |
| orchestrator        | Workflow coordination | Complex issues (HIGH+ complexity), multi-domain issues           |
| system-architect    | System design         | Architecture decisions, technical specifications, design reviews |
| dev-cpp             | C++ implementation    | C++ features, memory management, performance-critical code       |
| dev-go              | Go implementation     | Go services, concurrency patterns, CLI tools                     |
| dev-python          | Python implementation | Python features, scripts, data processing, ML/AI code            |
| dev-react           | TypeScript+React impl | Frontend components, React hooks, TypeScript interfaces          |
| reviewer            | Quality review        | Code review, documentation review, configuration review          |
| qa-engineer         | Test quality          | Test planning, coverage analysis, TDD enforcement, test strategy |
| security-analyst    | Security analysis     | Threat modeling, vulnerability assessment, secure code review    |
| techwriter          | Documentation         | Creating/updating guides, API docs, onboarding docs              |
| devops-engineer     | Infrastructure        | Deployment, monitoring, alerting, system administration          |
| knowledge-manager   | Knowledge management  | Capturing lessons, querying past solutions, pattern analysis     |
| code-health-analyst | Code quality metrics  | Complexity analysis, tech debt detection, code smell detection   |
| log-analyst         | Log analysis          | Error pattern detection, anomaly detection, log correlation      |
| dependency-manager  | Dependency management | CVE detection, outdated packages, license compliance             |
| incident-responder  | Staging diagnosis     | VALIDATION anomaly diagnosis, transient vs bug classification    |

### Agent Hierarchy

Agents are organized in a 4-tier hierarchy for coordinated workflows:

| Tier | Role | Agents | Authority |
|------|------|--------|-----------|
| **Tier 1** | Orchestrator | orchestrator | Workflow coordination, team composition, conflict resolution |
| **Tier 2** | Supervisors | system-architect, security-analyst, devops-engineer | Domain authority, binding decisions, veto power (security) |
| **Tier 3** | Specialists | dev-*, reviewer, qa-engineer, techwriter, knowledge-manager | Task execution, recommendations |
| **Tier 4** | Responders | incident-responder, log-analyst, code-health-analyst, dependency-manager | Monitoring, detection, reporting |

**Orchestrator Activation**: The orchestrator is automatically activated when:
- Issue complexity score >= 5 (HIGH)
- Issue spans 2+ domains
- Previous attempt failed

**Veto Authority**: The security-analyst has veto power on security matters:
- CRITICAL veto: Immediately blocks unsafe recommendations
- HIGH veto: Escalates to user for decision

See `templates/hierarchy-config.md` for full hierarchy details.
