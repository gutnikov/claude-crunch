## Task Workflow

This project uses a structured task workflow with distinct phases for development, validation, and merging.

### Task Types

| Type    | Label        | Description                |
| ------- | ------------ | -------------------------- |
| Bug     | type:bug     | Something is not working   |
| Feature | type:feature | New feature or enhancement |

### Task Domains

Domain labels enable dynamic agent selection. Issues are automatically classified by keywords, or can be explicitly labeled.

| Domain        | Label              | Keywords (auto-detected)                                        | Primary Agent    |
| ------------- | ------------------ | --------------------------------------------------------------- | ---------------- |
| Security      | domain:security    | auth, login, password, token, JWT, OAuth, XSS, injection, CVE   | security-analyst |
| Infrastructure| domain:infra       | deploy, CI/CD, docker, kubernetes, terraform, monitoring, nginx | devops-engineer  |
| Performance   | domain:performance | slow, latency, optimize, cache, memory, CPU, bottleneck         | reviewer         |
| API           | domain:api         | endpoint, REST, GraphQL, rate limit, API key, webhook           | system-architect |
| Database      | domain:database    | query, migration, schema, index, PostgreSQL, MySQL, Redis       | system-architect |
| Frontend      | domain:frontend    | React, component, hook, UI, CSS, styled, tailwind, form, modal  | dev-react        |
| Python        | domain:python      | Python, Django, Flask, FastAPI, pandas, numpy, pytest, pip      | dev-python       |
| C++           | domain:cpp         | C++, STL, template, RAII, smart pointer, memory, CMake, Boost   | dev-cpp          |
| Go            | domain:go          | Go, goroutine, channel, gin, cobra, gRPC, go mod                | dev-go           |

When a domain is detected, the workflow uses specialized agents for specification, review, and validation phases.

### Task States

| State          | Label                | Description                                               |
| -------------- | -------------------- | --------------------------------------------------------- |
| Input          | (no label)           | Initial task input, not yet on the board                  |
| Backlog        | state:backlog        | Awaiting triage or prioritization                         |
| Enrich         | state:enrich         | Adding technical details to be ready for implementation   |
| Ready          | state:ready          | Ready for implementation                                  |
| Implementing   | state:implementing   | Currently being worked on                                 |
| Validation     | state:validation     | Implementation complete, verifying against DOD on staging |
| Docs           | state:docs           | Creating documentation in the project wiki                |
| Ready to Merge | state:ready-to-merge | Validation passed, ready to merge to main                 |
| Done           | state:done           | Completed and merged                                      |

### State Flow Diagram

```
+-------+   +---------+   +--------+   +-------+   +--------------+
| INPUT |-->| BACKLOG |-->| ENRICH |-->| READY |-->| IMPLEMENTING |
+-------+   +---------+   +--------+   +-------+   +--------------+
                 |             |                          |
                 v             v                          v
              [CLOSE]       [CLOSE]               +------------+
              cancelled     out of scope          | VALIDATION |--+
              duplicate                           +------------+  |
                                                        |         |
                                          fail: back to |         |
                                          IMPLEMENTING <+         |
                                                                  |
            +------+   +----------------+   +------+              |
            | DONE |<--| READY TO MERGE |<--| DOCS |<-------------+
            +------+   +----------------+   +------+
```
