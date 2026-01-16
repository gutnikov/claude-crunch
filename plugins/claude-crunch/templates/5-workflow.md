## Task Workflow

This project uses a structured task workflow with distinct phases for development, E2E check, and merging.

### Task Types

| Type          | Label              | Description                                  |
| ------------- | ------------------ | -------------------------------------------- |
| Bug           | type:bug           | Something is not working                     |
| Feature       | type:feature       | New feature or enhancement                   |
| Decomposition | type:decomposition | Spec/idea to decompose into actionable tasks |

#### Decomposition Tasks

Decomposition tasks are used when you have a large specification, idea, or epic that needs to be broken down into smaller, implementable tasks. The workflow for decomposition tasks is different from bug/feature tasks:

1. **Create decomposition issue**: Create an issue with `type:decomposition` label
2. **Analyze the spec**: Break down the spec into logical components
3. **Create child issues**: Create individual bug/feature issues for each component
4. **Link issues**: Reference the parent decomposition issue in child issues
5. **Close decomposition**: Once all child issues are created, close the decomposition issue

**Example decomposition issue body:**

```markdown
## Spec: User Dashboard Redesign

Large specification for redesigning the user dashboard with new widgets and layouts.

## Decomposition Output

Created the following child issues:

- #12: Add user stats widget (type:feature)
- #13: Add activity timeline widget (type:feature)
- #14: Update responsive layout (type:feature)
- #15: Fix widget loading performance (type:bug)
```

### Task Creation Rules

#### Definition of Done (DOD) Requirement

**Every task (bug or feature) MUST contain a Definition of Done (DOD) list in the issue body.** Each DOD item MUST have a verification method explaining how to check if the item is complete.

**DOD Format:**

```markdown
## Definition of Done (DOD)

- [ ] DOD item 1
  - **Verification:** Specific steps or commands to verify this item is complete
- [ ] DOD item 2
  - **Verification:** Specific steps or commands to verify this item is complete
```

**Good DOD Examples:**

| DOD Item                              | Verification                                                         |
| ------------------------------------- | -------------------------------------------------------------------- |
| API endpoint returns correct response | `curl -X GET /api/users/1` returns 200 with user JSON                |
| Unit tests pass                       | Run `npm test`, all tests in `auth.test.ts` pass                     |
| Error handling for invalid input      | `curl -X POST /api/login -d '{}'` returns 400 with validation errors |
| No console errors in browser          | Open DevTools console, perform login flow, no red errors appear      |
| Performance under 200ms               | Run `ab -n 100 /api/endpoint`, mean response time < 200ms            |
| Database migration runs cleanly       | `npm run migrate` completes without errors on fresh DB               |

**Bad DOD Examples (avoid):**

| Bad DOD Item | Why It's Bad         | Better Version                                         |
| ------------ | -------------------- | ------------------------------------------------------ |
| "Code works" | Not verifiable       | "Login returns JWT token when given valid credentials" |
| "Tests pass" | Too vague            | "Run `npm test auth`, all 15 auth tests pass"          |
| "No bugs"    | Impossible to verify | "Manual test: login, view profile, logout - no errors" |
| "Clean code" | Subjective           | "ESLint passes with zero warnings"                     |

**Verification Types:**

1. **Command verification**: Specific CLI command with expected output
2. **API verification**: HTTP request with expected status/response
3. **UI verification**: Manual steps with expected visual result
4. **Metric verification**: Measurable threshold (time, count, size)
5. **Automated verification**: Test suite/script that validates the item

**E2E Testability Requirement:**

Every DOD item MUST be designed for autonomous E2E validation. Items requiring human judgment are NOT acceptable. See `templates/5-e2e-testability.md` for full guidelines.

| Acceptable | Not Acceptable |
|------------|----------------|
| `curl /api/users/1` returns 200 | "User API works" |
| No errors in last 5min logs | "No bugs" |
| P99 latency < 200ms | "Performance is good" |
| `/health` returns `status: ok` | "System is stable" |

**Incremental Development Requirement:**

Feature specifications MUST include milestone breakdown from simplest testable version to complete solution. Each milestone must have its own testable DOD subset. See `templates/5a-incremental-development.md` for full guidelines.

| Milestone | Content Required |
|-----------|------------------|
| M0 | Simplest happy path + 1 test |
| M1-N | Incremental complexity layers + tests |
| Final | Complete feature + full coverage |

### Task Domains

Domain labels enable dynamic agent selection. Issues are automatically classified by keywords, or can be explicitly labeled.

| Domain         | Label              | Keywords (auto-detected)                                        | Primary Agent    |
| -------------- | ------------------ | --------------------------------------------------------------- | ---------------- |
| Security       | domain:security    | auth, login, password, token, JWT, OAuth, XSS, injection, CVE   | security-analyst |
| Infrastructure | domain:infra       | deploy, CI/CD, docker, kubernetes, terraform, monitoring, nginx | devops-engineer  |
| Performance    | domain:performance | slow, latency, optimize, cache, memory, CPU, bottleneck         | reviewer         |
| API            | domain:api         | endpoint, REST, GraphQL, rate limit, API key, webhook           | system-architect |
| Database       | domain:database    | query, migration, schema, index, PostgreSQL, MySQL, Redis       | system-architect |
| Frontend       | domain:frontend    | React, component, hook, UI, CSS, styled, tailwind, form, modal  | dev-react        |
| Python         | domain:python      | Python, Django, Flask, FastAPI, pandas, numpy, pytest, pip      | dev-python       |
| C++            | domain:cpp         | C++, STL, template, RAII, smart pointer, memory, CMake, Boost   | dev-cpp          |
| Go             | domain:go          | Go, goroutine, channel, gin, cobra, gRPC, go mod                | dev-go           |

When a domain is detected, the workflow uses specialized agents for specification, review, and E2E phases.

### Task States

| State          | Label                | Description                                               |
| -------------- | -------------------- | --------------------------------------------------------- |
| Input          | (no label)           | Initial task input, not yet on the board                  |
| Backlog        | state:backlog        | Awaiting triage or prioritization                         |
| Enrich         | state:enrich         | Adding technical details to be ready for implementation   |
| Ready          | state:ready          | Ready for implementation                                  |
| Implementing   | state:implementing   | Currently being worked on                                 |
| E2E            | state:e2e            | Implementation complete, verifying against DOD on staging |
| Docs           | state:docs           | Creating documentation in the project wiki                |
| Ready to Merge | state:ready-to-merge | E2E passed, ready to merge to main                        |
| Done           | state:done           | Completed and merged                                      |

### State Flow Diagram

```
+-------+   +---------+   +--------+   +-------+   +--------------+
| INPUT |-->| BACKLOG |-->| ENRICH |-->| READY |-->| IMPLEMENTING |
+-------+   +---------+   +--------+   +-------+   +--------------+
                 |             |                          |
                 v             v                          v
              [CLOSE]       [CLOSE]                   +-----+
              cancelled     out of scope              | E2E |------+
              duplicate                               +-----+      |
                                                        |          |
                                          fail: back to |          |
                                          IMPLEMENTING <+          |
                                                                   |
            +------+   +----------------+   +------+               |
            | DONE |<--| READY TO MERGE |<--| DOCS |<--------------+
            +------+   +----------------+   +------+
```
