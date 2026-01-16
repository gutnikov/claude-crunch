# Workflow Transitions

Detailed instructions for each state transition. Reference CLAUDE.md for:

- Task types and states (Task Workflow section)
- Available agents (Agent System section)
- TDD requirements (TDD section)
- Observability requirements (Observability section)

---

## Input Modes

The crunch skill supports multiple invocation modes for flexibility.

### Mode 1: Existing Issue (Interactive)

```
/crunch 5
/crunch #5
```

- Fetch issue from CI MCP
- Display summary to user
- Ask target state interactively via `AskUserQuestion`
- Execute workflow transitions

### Mode 2: Existing Issue (Direct)

```
/crunch -i 5 -t ready
/crunch --issue 5 --target validation
```

- Fetch issue from CI MCP
- Use provided target state (skip interactive ask)
- Validate transition is allowed
- Execute workflow transitions

### Mode 3: New Issue (Interactive)

```
/crunch "Add dark mode toggle"
```

- Create new issue via CI MCP with text as title/body
- Run INPUT negotiation (2-3 turns)
- Ask target state after negotiation
- Execute workflow transitions

### Mode 4: New Issue (Non-Interactive)

```
/crunch --input "Fix null pointer in handler" --type bug
/crunch --input "Add metrics endpoint" --type feature -t ready
```

- Create new issue via CI MCP
- Skip INPUT negotiation entirely
- Use provided `--type` or infer from keywords
- Transition directly to BACKLOG (or further if `-t` provided)

### Parameter Parsing

```
Arguments:
  [number]          Issue number (e.g., 5, #5)
  "text"            New issue description (interactive mode)

Options:
  -i, --issue NUM   Issue number to process
  -t, --target ST   Target state (skips interactive ask)
  --input TEXT      New issue text (non-interactive mode)
  --type TYPE       Issue type: bug | feature
```

### Type Inference

When `--type` is not provided with `--input`, infer from keywords:

**Bug indicators**: fix, bug, crash, error, broken, issue, problem, fails, null, exception, undefined

**Feature indicators**: add, implement, create, new, feature, support, enable, introduce, enhance

If ambiguous, default to `feature`.

---

## Git Branch Lifecycle

```
INPUT ────────────────────────────────────────► No branch
   └──► Negotiation with requestor (issue body only)

BACKLOG ──────────────────────────────────────► No branch
   └──► Awaiting triage

ENRICH ───────────────────────────────────────► CREATE BRANCH
   └──► {fix|feature}/issue-{number}-{short-description}
        Investigation, spec, DoD checklist

READY ────────────────────────────────────────► Same branch
   └──► Ready for implementation (no commits here)

IMPLEMENTING ─────────────────────────────────► Same branch
   └──► Implementation commits, test commits

E2E ───────────────────────────────────► Same branch
   └──► Verify DoD checklist on staging

DOCS ─────────────────────────────────────────► Same branch
   └──► Documentation commits

READY-TO-MERGE
   ├──► PUSH: git push -u origin {branch}
   └──► CREATE PR via CI MCP

DONE (after PR merged)
   └──► DELETE BRANCH (optional)
```

---

## INPUT Phase (no state label)

**Purpose**: Process raw input before it becomes a proper backlog item.

This phase has two paths depending on invocation mode.

### Interactive Path (default)

When invoked with bare text or existing issue without state label:

**This is a conversation phase** - do 2-3 turns of negotiation to:

1. Understand what the requestor actually wants
2. Clarify ambiguous requirements
3. Identify missing information
4. Confirm understanding

#### Negotiation Process

**Turn 1: Initial Understanding**

- Read the raw input
- Ask clarifying questions:
  - What problem are you trying to solve?
  - What does success look like?
  - Any constraints or preferences?

**Turn 2: Details**

- Based on answers, dig deeper:
  - Specific behaviors expected?
  - Edge cases to consider?
  - Priority and urgency?

**Turn 3: Confirmation**

- Summarize understanding
- Confirm with requestor
- Identify issue type (bug or feature)

#### Actions on Completion

1. Update issue body with clarified requirements
2. Add type label (`type:bug` or `type:feature`)
3. Add state label (`state:backlog`)

Update issue body:

```markdown
## {Bug|Feature}: {title}

### Problem/Request

{clarified description}

### Expected Behavior

{what should happen}

### Context

{any relevant context from negotiation}

### Priority

{priority level}

**Status**: Ready for Backlog
```

### Non-Interactive Path (`--input`)

When invoked with `--input` flag, skip negotiation entirely:

1. **Use input text as-is** for issue body
2. **Determine type**:
   - Use `--type` if provided
   - Otherwise infer from keywords (see Input Modes section)
3. **Create issue** via CI MCP with:
   - Title: first line or first 80 chars of input
   - Body: full input text
   - Labels: inferred/provided type
4. **Immediately transition to BACKLOG**:
   - Add `state:backlog` label
   - Skip negotiation turns

Update issue body (non-interactive):

```markdown
## {Bug|Feature}: {title}

### Description

{input text as provided}

### Source

Created via non-interactive crunch invocation.

**Status**: Ready for Backlog
```

If `-t` target is also provided, continue transitioning toward target state.

---

## Transition: BACKLOG → ENRICH

**When**: Issue needs investigation (bug) or specification (feature)

**Actions**:

1. Review issue description
2. Create branch: `{fix|feature}/issue-{number}-{description}`
3. Update labels to `state:enrich`
4. Begin Enrich Phase

---

## Transition: BACKLOG → READY

**When**: Simple issue with clear requirements, no investigation needed

**Actions**:

1. Verify issue has clear requirements
2. Create branch: `{fix|feature}/issue-{number}-{description}`
3. Create DoD checklist (see Enrich Phase)
4. Update labels to `state:ready`

**Validation**: Must have clear goal, acceptance criteria, and DoD checklist

---

## Transition: BACKLOG → CLOSE

**When**: Cancelled, duplicate, not valid

**Actions**:

1. Document reason in issue body
2. Update labels to `state:done`
3. Close issue

---

## Enrich Phase (state:enrich)

> **CRITICAL**: The most important outputs of this phase are:
>
> 1. **Definition of Done (DoD) checklist** - For staging validation
> 2. **Test Plan** - For test coverage requirements

### For Bugs: Investigation

1. **Reproduce** the bug
2. **Root cause analysis** - find where and why
3. **Fix approach** - document proposed fix
4. **Test Plan** (qa-engineer agent)
   - Regression test requirements
   - Edge cases from root cause analysis
   - Coverage targets
5. **Create DoD checklist**

### For Features: Specification

Use planning mode and agents per CLAUDE.md:

1. **Exploration** (planning mode)
   - Explore codebase for patterns
   - Identify constraints
   - Document findings

2. **Specification** (architect agent)
   - Create technical spec
   - Define interfaces
   - List open questions

3. **Test Planning** (qa-engineer agent)
   - Create Test Plan artifact
   - Identify critical paths requiring coverage
   - Define coverage thresholds
   - Enumerate edge cases to test

4. **Negotiation** (architect ↔ dev agent)
   - Dev agent reviews spec AND Test Plan
   - Architect clarifies/updates
   - Repeat until converged

5. **Create DoD checklist**

### Definition of Done (DoD) Checklist

> **This is mandatory.** The DoD checklist defines what must be verified on staging before the work can be merged to main and deployed to production.

The checklist must include:

**Infrastructure Checks**:

- [ ] App on staging is running successfully
- [ ] No suspicious error logs (check log aggregator)
- [ ] Metrics are healthy (check dashboards)
- [ ] Alerts are not firing

**Functional Checks**:

- [ ] The requested functionality works as expected
- [ ] {Specific behavior 1 from requirements}
- [ ] {Specific behavior 2 from requirements}
- [ ] Edge cases handled correctly

**Regression Checks**:

- [ ] Existing functionality still works
- [ ] No performance degradation
- [ ] Related features unaffected

### E2E Testability Verification

Before completing ENRICH, verify all DOD items meet autonomous validation requirements:

1. **Review each DOD item** - Can it be verified without human judgment?
2. **Flag subjective criteria** - Rewrite vague items with specific verification methods
3. **Verify health endpoints** - Services being modified must expose `/health`
4. **Document verification commands** - Each DOD item needs explicit command/query

| DOD Quality Check                      | Action if Failed                      |
| -------------------------------------- | ------------------------------------- |
| Item is subjective ("works correctly") | Rewrite with specific assertion       |
| No verification method specified       | Add command/query/metric check        |
| Requires human judgment                | Convert to measurable criterion       |
| Missing health endpoint                | Add health endpoint to implementation |

### Incremental Milestone Planning

Break implementation into deployable milestones:

1. **Identify simplest testable slice** - What's the minimum that proves core functionality?
2. **Define M0** - Minimal DOD subset for first milestone
3. **Plan progression** - Each milestone adds complexity layer
4. **Ensure independence** - Each milestone must be independently deployable and testable

**Milestone Template:**

```markdown
### Milestone {N}: {Name}

**Scope:** {What this adds}
**DOD Subset:** {Which DOD items this milestone completes}
**Tests:** {Test count and types}
**Dependencies:** {Previous milestones required}
```

Update issue body:

```markdown
## {Bug|Feature}: {title}

### Summary

{description}

### Investigation/Specification

{findings or spec details}

### Implementation Notes

- Files to create: {list}
- Files to modify: {list}

### Definition of Done (Staging Validation)

**Infrastructure**:

- [ ] App on staging is running successfully
- [ ] No suspicious error logs
- [ ] Metrics and dashboards are healthy
- [ ] No alerts firing

**Functional**:

- [ ] {Specific check 1}
- [ ] {Specific check 2}
- [ ] {Specific check 3}

**Regression**:

- [ ] {Regression check 1}
- [ ] {Regression check 2}

### Test Plan

#### Required Test Levels

| Level       | Required | Rationale |
| ----------- | -------- | --------- |
| Unit        | {Yes/No} | {reason}  |
| Integration | {Yes/No} | {reason}  |
| E2E         | {Yes/No} | {reason}  |

#### Critical Paths (P0)

1. {path} - {expected behavior}

#### Edge Cases (P1)

| Edge Case | Test Approach |
| --------- | ------------- |
| {case}    | {approach}    |

#### Coverage Targets

| Metric          | Threshold |
| --------------- | --------- |
| Line Coverage   | {X}%      |
| Branch Coverage | {Y}%      |

**Status**: Enrich Complete
```

---

## Transition: ENRICH → READY

**When**: Investigation/specification complete AND DoD checklist created

**Actions**:

1. Verify DoD checklist exists in issue body
2. Commit spec/investigation:

   ```bash
   git commit -m "{Spec|Investigation}: {description}

   Issue: #{issue_number}"
   ```

3. Update labels to `state:ready`

---

## Transition: READY → IMPLEMENTING

**When**: Implementation begins

### Planning (planning mode)

1. Enter planning mode
2. Review spec/investigation
3. Design implementation steps
4. Get user approval

### Implementation (dev agent per CLAUDE.md)

1. Update labels to `state:implementing`
2. **Write tests first** (TDD per CLAUDE.md)
3. Implement changes
4. Ensure tests pass
5. Run CI checks

Update issue body:

```markdown
### Implementation

**Branch**: `{branch}`
**Changes**:

- `{file}` - {description}

**Tests**:

- `{test_file}` - {description}

**CI**: {status}

**Status**: Implementation Complete
```

---

## Transition: IMPLEMENTING → E2E

**When**: Implementation complete, CI passing

### TEST-VERIFICATION Gate

> **MUST PASS**: Before code review, verify test coverage and quality.

```
1. Load Test Plan from issue body

2. Run test analysis:
   /test-analyze -c -i {issue_number}

3. Check against Test Plan requirements:
   - Coverage meets threshold from Test Plan?
   - Critical paths (P0) have tests?
   - Quality score >= 60?

4. Gate decision:
   IF all checks pass:
     → Proceed to Code Review
   ELSE:
     → Report gaps
     → Stay in IMPLEMENTING
     → Dev agent addresses gaps
     → Re-run TEST-VERIFICATION
```

**TEST-VERIFICATION Output**:

```markdown
### TEST-VERIFICATION: {PASS|FAIL}

**Coverage**: {X}% (target: {Y}%)
**Critical Paths**: {covered}/{total} covered
**Quality Score**: {score}/100

{If FAIL: list of gaps to address}
```

### Code Review (reviewer agent per CLAUDE.md)

Review loop until approved (now includes 6 agents with Test Quality Reviewer):

1. Invoke /review skill (includes Test Quality Reviewer)
2. Dev agent addresses feedback
3. Re-review until approved

### Deploy to Staging

1. Deploy to staging environment (per CLAUDE.md Deployment section)
2. Update labels to `state:e2e`

Update issue body:

```markdown
### TEST-VERIFICATION

**Status**: PASS
**Coverage**: {X}% (target: {Y}%)
**Critical Paths**: All P0 paths covered

### Review

**Round {N}**: {APPROVED|CHANGES_REQUESTED}
**Issues**: {resolved/total}

### Deployed to Staging

**Version**: {commit/tag}
**Time**: {timestamp}
```

---

## E2E Phase (state:e2e)

**Purpose**: Verify the DoD checklist on staging before proceeding.

**Environment**: Staging ONLY. This phase never touches production.

**Duration**: Time-bounded. Default 20 minutes for continuous monitoring (10 minutes for local Docker).

### Platform-Specific Staging

| CI Platform         | Staging Environment  | Deploy Method                        | Monitoring         |
| ------------------- | -------------------- | ------------------------------------ | ------------------ |
| GitHub/GitLab/Gitea | Remote staging infra | CI/CD pipeline                       | Observability MCPs |
| Filebase            | Local Docker         | `/ci-filebase docker deploy staging` | Docker logs/health |

### Filebase + Local Docker Validation

When CI platform is Filebase, validation uses local Docker instead of remote infrastructure.

```
E2E Phase (Filebase):

1. Run CI pipeline locally:
   /ci-filebase docker ci
   IF pipeline fails: stay in IMPLEMENTING, fix issues

2. Deploy to local staging:
   /ci-filebase docker deploy staging
   IF deploy fails: show logs, stay in IMPLEMENTING

3. Execute DoD checklist (see below):
   - App running: /ci-filebase docker health
   - Error logs: /ci-filebase docker logs --tail 100
   - Functional: Manual curl/test commands against localhost
   - Regression: Manual verification

4. Continuous monitoring (simplified):
   duration: 10m (shorter for local)
   interval: 2m
   checks: health endpoint, docker logs, container stability

5. On anomaly:
   - Show docker logs
   - Return to IMPLEMENTING (no auto-remediation for local)
   - User fixes bug and re-deploys

6. On success:
   - Mark DoD items checked
   - Stop staging: /ci-filebase docker stop (optional)
   - Proceed to DOCS phase
```

### Execute DoD Checklist (MCP-based CI)

Go through each item in the DoD checklist:

1. **Infrastructure Checks**
   - Verify app is running (health endpoints)
   - Check logs for errors (per CLAUDE.md Observability section)
   - Review metrics dashboards
   - Confirm no alerts firing

2. **Functional Checks**
   - Test each specific behavior from requirements
   - Verify edge cases work correctly
   - Confirm the task does what was requested

3. **Regression Checks**
   - Test related functionality
   - Verify no performance issues

### Continuous Monitoring (Post-Deploy)

After initial DoD checks pass, monitor staging for a bounded period to catch issues that emerge under load or over time.

**Scope and Limits**:

- Environment: Staging only
- Duration: 20 minutes (configurable)
- Interval: 5 minutes between checks
- Max cycles: 4 (duration / interval)
- Purpose: Catch deployment issues before they reach production

```
Parameters:
  duration: 20m        # Total monitoring window
  interval: 5m         # Time between checks
  max_cycles: 4        # duration / interval

Execution:
  baseline = capture_metrics()  # Error rate, P99, memory at start

  for cycle in 1..max_cycles:

    # 1. Fetch current state
    current = fetch_metrics_and_logs(last={interval})

    # 2. Compare to baseline
    anomalies = detect_anomalies(baseline, current)

    # 3. Report cycle
    log_cycle_status(cycle, max_cycles, anomalies)

    # 4. Handle anomalies (if any)
    if anomalies:
      result = handle_staging_anomaly(anomalies)

      if result == RESOLVED:
        # Transient issue, continue monitoring
        continue

      if result == NEEDS_FIX:
        # Real bug found, return to IMPLEMENTING
        update_issue_with_diagnosis(anomalies)
        transition_to(IMPLEMENTING)
        return  # Exit monitoring

    # 5. Sleep until next cycle (if not last)
    if cycle < max_cycles:
      sleep(interval)

  # All cycles complete, no unresolved issues
  log("Monitoring complete - staging healthy")
  proceed_to(DOCS)
```

**Monitoring Thresholds** (more sensitive than standard patrol):

| Metric         | Warning       | Critical       |
| -------------- | ------------- | -------------- |
| Error rate     | 1.5x baseline | 3x baseline    |
| P99 latency    | 30% increase  | 50% increase   |
| New exceptions | 3 occurrences | 10 occurrences |
| Memory growth  | 10% increase  | 25% increase   |

**Skip Conditions**:

- No observability infrastructure configured (MCP-based CI)
- No Docker available (Filebase CI)
- User explicitly skips with flag
- Hotfix/emergency deployment

**Filebase Note**: When using Filebase CI without Docker, E2E falls back to manual verification only. User must manually test the application and confirm DoD checklist items.

### Staging Anomaly Handling

When an anomaly is detected during monitoring, determine if it's a transient issue (fixable via restart) or a real bug (needs code fix).

```
handle_staging_anomaly(anomalies):

  # 1. Classify the anomaly
  for anomaly in anomalies:
    anomaly.type = classify(anomaly)
    # Types: error_spike, latency_regression, new_exception, memory_growth

  # 2. Invoke incident-responder agent for diagnosis
  diagnosis = invoke_agent("incident-responder", {
    anomalies: anomalies,
    context: "staging_validation",
    issue_number: current_issue
  })

  # 3. Query knowledge base for similar staging issues
  similar = query_knowledge_base(
    type: "resolution",
    context: "staging",
    symptoms: anomalies.map(a => a.description)
  )

  # 4. Determine if this is a known transient issue
  if similar.exists and similar.confidence >= 90%:
    if similar.resolution_type == "transient":
      # Known transient issue (e.g., cold start, cache warm-up)

      # 4a. Check remediation playbook for allowed action
      action = get_playbook_action(anomalies, similar.recommended_action)

      if action:
        # 4b. Try quick remediation
        success = try_staging_remediation(action)

        if success:
          log("Transient issue resolved via {action}")
          record_remediation_success(similar, action)
          return RESOLVED
        else:
          # Remediation didn't help - it's a real bug
          log("Remediation failed - escalating to code fix")
          return NEEDS_FIX

  # 5. Unknown issue or low confidence - assume it's a bug
  log("No confident match - treating as bug requiring fix")
  return NEEDS_FIX
```

### Staging Remediation Actions

Limited set of actions for staging transient issues only. These are NOT production fixes.

| Action              | When to Use                         | Command                                                | Timeout |
| ------------------- | ----------------------------------- | ------------------------------------------------------ | ------- |
| `restart_pod`       | Memory leak symptoms, stuck process | `kubectl rollout restart deployment/{name} -n staging` | 60s     |
| `restart_container` | Container-level issues              | `docker restart {container_id}`                        | 30s     |
| `flush_cache`       | Stale data, cache corruption        | Service-specific (Redis FLUSHDB, app cache endpoint)   | 30s     |
| `reload_config`     | Config not picked up after deploy   | `kill -HUP {pid}` or service reload endpoint           | 15s     |

**What remediation is NOT for**:

- Fixing actual bugs (that's IMPLEMENTING phase)
- Production issues (out of scope for /crunch)
- Anything requiring code changes
- Deployment rollbacks (investigate instead)

**Remediation limits**:

- Max 1 attempt per anomaly type per monitoring window
- 2-minute wait after remediation to observe effect
- If not resolved after 1 attempt → it's a real bug, not transient

```
try_staging_remediation(action):

  # 1. Pre-remediation snapshot
  before = capture_metrics()
  log("Attempting remediation: {action.type}")

  # 2. Execute action
  result = execute_action(action)

  if result.failed:
    log("Remediation action failed: {result.error}")
    return false

  # 3. Wait for stabilization
  sleep(2 minutes)

  # 4. Post-remediation check
  after = capture_metrics()

  # 5. Compare
  resolved = (
    after.error_rate <= before.baseline * 1.5 and
    after.new_exceptions == 0 and
    after.latency_p99 <= before.baseline_latency * 1.3
  )

  # 6. Record outcome for knowledge base
  record_remediation_attempt({
    action: action.type,
    anomaly: action.anomaly_type,
    before: before,
    after: after,
    success: resolved
  })

  return resolved
```

### Updating Issue on Validation Failure

When monitoring detects an issue that needs a code fix, update the CURRENT issue (never create a new one) and return to IMPLEMENTING.

````
update_issue_with_diagnosis(anomalies):

  # Get diagnosis from incident-responder agent
  diagnosis = get_diagnosis()

  # Append to issue body
  append_to_issue(current_issue):
    """
    ---

    ### Staging Validation Failed

    **Detected During**: Continuous monitoring (cycle {cycle}/{max_cycles})
    **Timestamp**: {timestamp}
    **Environment**: staging

    #### Anomalies Detected

    | Type | Current | Baseline | Threshold |
    |------|---------|----------|-----------|
    {for each anomaly}
    | {type} | {value} | {baseline} | {threshold} |

    #### Log Samples

    ```
    {relevant_error_logs, max 20 lines}
    ```

    #### Stack Trace (if available)

    ```
    {stack_trace}
    ```

    #### Metrics Snapshot

    - Error rate: {rate}/min (baseline: {baseline}/min)
    - P99 latency: {latency}ms (baseline: {baseline_latency}ms)
    - Memory: {memory}MB (baseline: {baseline_memory}MB)

    #### Remediation Attempted

    {if remediation_attempted}
    - Action: {action}
    - Result: Failed - issue persists
    - Conclusion: This is a code bug, not a transient issue
    {else}
    - No remediation attempted (no confident match in knowledge base)
    {endif}

    #### Diagnosis

    {diagnosis.summary}

    **Likely root cause**: {diagnosis.root_cause}

    **Affected components**: {diagnosis.components}

    #### Recommended Fix

    {diagnosis.recommended_fix}

    **Files to investigate**:
    {for each file in diagnosis.files}
    - `{file.path}`: {file.reason}

    ---

    **Action Required**: Fix the issue and re-run validation.
    """

  # Update labels - return to IMPLEMENTING
  remove_label("state:e2e")
  add_label("state:implementing")

  # Record in knowledge base for future reference
  invoke_skill("/learn", {
    type: "resolution",
    issue: current_issue,
    status: "failed",
    phase: "validation",
    anomalies: anomalies,
    diagnosis: diagnosis
  })

  # Notify user
  show:
    "Validation failed - bug detected on staging.

    Issue #{current_issue} has been updated with:
    - Error logs and stack traces
    - Metrics snapshot
    - Diagnosis and recommended fix

    Returning to IMPLEMENTING phase.
    Please fix the issue and re-deploy to staging."
````

### Monitoring Output Format

**Success case:**

```
Continuous Monitoring - E2E Phase
========================================
Environment: staging
Issue: #{issue_number}
Duration: 20 minutes (4 cycles)
Started: {timestamp}

Baseline captured:
  - Error rate: 0.1/min
  - P99 latency: 120ms
  - Memory: 256MB

Cycle 1/4 [{timestamp}]:
  ✓ Error rate: 0.12/min (baseline: 0.1/min) - OK
  ✓ P99 latency: 118ms (baseline: 120ms) - OK
  ✓ Memory: 258MB (baseline: 256MB) - OK
  Status: HEALTHY
  Next check in 5 minutes...

Cycle 2/4 [{timestamp}]:
  ⚠ Error rate: 2.5/min (baseline: 0.1/min) - SPIKE
  ✓ P99 latency: 125ms - OK
  ✓ Memory: 260MB - OK
  Status: ANOMALY DETECTED

  Invoking incident-responder for diagnosis...
  Checking knowledge base for similar issues...

  Found: Issue #42 (similarity: 92%)
    Description: "Cold start errors after deploy"
    Resolution: restart_pod (success rate: 8/9)
    Classification: transient

  Attempting remediation: restart_pod
  Executing: kubectl rollout restart deployment/myapp -n staging
  Waiting 2 minutes for stabilization...

  Post-remediation check:
  ✓ Error rate: 0.15/min - RESOLVED

  Transient issue resolved. Continuing monitoring...

Cycle 3/4 [{timestamp}]:
  ✓ All metrics healthy
  Status: HEALTHY

Cycle 4/4 [{timestamp}]:
  ✓ All metrics healthy
  Status: HEALTHY

========================================
Monitoring Complete
Duration: 20 minutes
Result: PASSED

Summary:
  - Cycles completed: 4/4
  - Anomalies detected: 1
  - Auto-resolved (transient): 1
  - Bugs found: 0

Proceeding to DOCS phase.
```

**Failure case:**

```
Continuous Monitoring - E2E Phase
========================================
Environment: staging
Issue: #{issue_number}
Duration: 20 minutes (4 cycles)
Started: {timestamp}

Baseline captured:
  - Error rate: 0.1/min
  - P99 latency: 120ms
  - Memory: 256MB

Cycle 1/4 [{timestamp}]:
  ✓ All metrics healthy
  Status: HEALTHY

Cycle 2/4 [{timestamp}]:
  ✓ All metrics healthy
  Status: HEALTHY

Cycle 3/4 [{timestamp}]:
  ✗ New exception: NullPointerException in AuthService.validateToken()
  ✗ Error rate: 5.2/min (baseline: 0.1/min) - CRITICAL
  ✓ P99 latency: 180ms - WARNING
  Status: ANOMALY DETECTED

  Invoking incident-responder for diagnosis...
  Checking knowledge base for similar issues...

  No confident match found (best: 45% similarity)
  Classification: likely_bug (not transient)

  Attempting remediation anyway: restart_pod
  Executing: kubectl rollout restart deployment/myapp -n staging
  Waiting 2 minutes for stabilization...

  Post-remediation check:
  ✗ Error rate: 4.8/min - NOT RESOLVED
  ✗ Same exception still occurring

  Remediation unsuccessful. This requires a code fix.

========================================
Monitoring Stopped Early
Duration: 12 minutes (stopped at cycle 3)
Result: FAILED - Bug detected

Updating issue #{issue_number} with diagnosis...
  ✓ Added error logs and stack traces
  ✓ Added metrics snapshot
  ✓ Added diagnosis: NullPointerException in token validation
  ✓ Added recommended fix: Check null token before validation

Returning to IMPLEMENTING phase.

Action Required:
  1. Review the diagnosis added to issue #{issue_number}
  2. Fix the NullPointerException in AuthService
  3. Re-deploy to staging
  4. Run validation again
```

### Update Issue Body

After successful validation:

```markdown
### E2E Status

**Environment**: staging
**Validation Date**: {timestamp}

**Infrastructure**:

- [x] App on staging is running successfully
- [x] No suspicious error logs
- [x] Metrics and dashboards are healthy
- [x] No alerts firing

**Functional**:

- [x] {Specific check 1}
- [x] {Specific check 2}

**Continuous Monitoring** (20 minutes):

- [x] Monitoring window completed
- [x] No persistent error rate spikes
- [x] No latency regressions
- [x] No unresolved exceptions

**Auto-Remediation** (if any):

- [x] Transient issue detected at cycle 2
- [x] Action: restart_pod
- [x] Result: Resolved
- [x] Classification: Cold start (known transient)

**Status**: Validation Passed
```

---

## Transition: E2E → IMPLEMENTING (Fail)

**When**: Any DoD checklist item fails

**Actions**:

1. Document what failed in issue body
2. Update labels back to `state:implementing`
3. Fix and repeat

---

## Transition: E2E → DOCS

**When**: All DoD checklist items pass

**Actions** (techwriter agent per CLAUDE.md):

1. Update documentation as needed
2. Add changelog entry
3. Update labels to `state:docs`

---

## Transition: DOCS → READY-TO-MERGE

**When**: Documentation complete

**Actions**:

1. Push branch: `git push -u origin {branch}`
2. Create PR via CI MCP:
   - Title: `{Fix|Feat} #{issue_number}: {description}`
   - Body: Summary, changes, DoD validation results
3. Update labels to `state:ready-to-merge`

---

## Transition: READY-TO-MERGE → DONE

**When**: PR approved and merged

**Actions**:

1. Merge PR via CI MCP
2. Update labels to `state:done`
3. Close issue
4. Delete branch (optional)

Update issue body:

```markdown
## Resolution: {Fixed|Implemented}

**Summary**: {description}
**PR**: #{pr_number}
**Merged**: {date}

### DoD Validation

All checks passed on staging.
```

---

## Orchestration Integration

Complex issues benefit from multi-agent coordination. The orchestrator is automatically activated when issue complexity exceeds thresholds.

### Complexity Scoring

Calculate complexity score at workflow entry:

| Factor            | Weight | Condition                                 |
| ----------------- | ------ | ----------------------------------------- |
| Multi-domain      | +2     | Issue spans >= 2 domains                  |
| High file count   | +1     | Estimated files >= 10                     |
| Security involved | +3     | Security domain detected                  |
| Breaking change   | +2     | `breaking-change` label present           |
| Prior failures    | +2     | Previous attempt returned to IMPLEMENTING |

**Complexity Levels**:

- **LOW** (0-2): Single agent handles workflow
- **MEDIUM** (3-4): Optional orchestration
- **HIGH** (5-7): Orchestrator activated
- **CRITICAL** (8+): Orchestrator + user checkpoints

### Orchestrator Activation

Automatically invoke `/orchestrate` when ANY condition is true:

```
orchestration_required = (
  complexity_score >= 5 OR
  len(detected_domains) >= 2 OR
  issue.has_label("complex") OR
  previous_attempt_failed
)
```

### Orchestrated Workflow

When orchestrator is active:

```
1. ENTRY PHASE
   ├── Calculate complexity score
   ├── Detect all applicable domains
   ├── Check orchestration_required
   └── IF required: invoke /orchestrate

2. /ORCHESTRATE EXECUTION
   ├── Team composition (select agents for domains)
   ├── Build execution graph (dependencies, parallelization)
   ├── Create checkpoints (recovery points)
   └── Return team and graph to /crunch

3. PHASE EXECUTION
   ├── For each batch in execution_graph:
   │   ├── Launch parallel agents via Task tool
   │   ├── Await results
   │   ├── Check for conflicts
   │   └── Save checkpoint
   └── Merge results and continue

4. CONFLICT RESOLUTION
   ├── IF agents disagree:
   │   ├── Invoke conflict resolution protocol
   │   ├── Structured debate
   │   ├── Weighted voting
   │   └── Apply resolution
   └── Record decision in knowledge base

5. COMPLETION
   ├── Update agent metrics
   ├── Archive checkpoints
   └── Record workflow summary
```

### Team Composition by Phase

| Phase        | Complexity < 5     | Complexity >= 5      |
| ------------ | ------------------ | -------------------- |
| ENRICH       | Primary agent only | Orchestrator + team  |
| IMPLEMENTING | Dev agent          | Dev + qa-engineer    |
| E2E          | Reviewer           | 6 parallel reviewers |
| DOCS         | Techwriter         | Techwriter           |

### Checkpoint Triggers

Save workflow state automatically at:

- Phase transitions (ENRICH→READY, etc.)
- Before agent invocation
- After conflict resolution
- Every 10 minutes during long phases

Checkpoint location: `.claude/crunch/{issue}/checkpoints/`

### Performance Tracking

After each agent invocation:

```
update_agent_metrics({
  agent: agent_name,
  domain: issue.primary_domain,
  phase: current_phase,
  outcome: "success" | "partial" | "failure",
  duration_ms: elapsed_time,
  quality_indicators: {
    review_pass_rate: ...,
    rework_required: true/false
  }
})
```

See `templates/agent-performance-schema.md` for full metrics schema.

---

## Dynamic Agent Selection

The crunch skill dynamically selects appropriate agents based on issue context. This enables specialized handling for security issues, infrastructure changes, and other domain-specific work.

### Domain Detection

Detect the issue domain from labels and content keywords:

| Domain         | Labels                              | Keywords in Title/Body                                                                                            |
| -------------- | ----------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Security       | `domain:security`, `security`       | auth, login, password, token, JWT, OAuth, XSS, injection, vulnerability, CVE, encrypt, permission, access control |
| Infrastructure | `domain:infra`, `infra`, `devops`   | deploy, CI/CD, pipeline, docker, kubernetes, k8s, terraform, monitoring, alert, nginx, server, AWS, GCP, Azure    |
| Performance    | `domain:performance`, `performance` | slow, latency, optimize, cache, memory, CPU, bottleneck, profiling                                                |
| API            | `domain:api`, `api`                 | endpoint, REST, GraphQL, rate limit, API key, webhook                                                             |
| Database       | `domain:database`, `database`       | query, migration, schema, index, PostgreSQL, MySQL, Redis                                                         |
| Frontend       | `domain:frontend`, `frontend`       | React, component, hook, UI, CSS, styled, tailwind, form, modal, TypeScript, tsx, jsx                              |
| Python         | `domain:python`, `python`           | Python, Django, Flask, FastAPI, pandas, numpy, pytest, pip, venv, poetry                                          |
| C++            | `domain:cpp`, `cpp`                 | C++, STL, template, RAII, smart pointer, unique_ptr, shared_ptr, CMake, Boost, memory                             |
| Go             | `domain:go`, `go`, `golang`         | Go, goroutine, channel, gin, cobra, gRPC, go mod, defer, interface{}                                              |

### Agent Capabilities

| Agent              | Primary Expertise                      | Use For                                             |
| ------------------ | -------------------------------------- | --------------------------------------------------- |
| `system-architect` | System design, architecture decisions  | Specification, complex feature design               |
| `security-analyst` | Security review, threat modeling       | Security-domain issues, auth changes, data handling |
| `devops-engineer`  | Infrastructure, deployment, monitoring | Infra-domain issues, CI/CD, observability           |
| `reviewer`         | Code quality, best practices           | General code review, quality gates                  |
| `dev-react`        | React, TypeScript, frontend patterns   | Frontend-domain issues, UI components, hooks        |
| `dev-python`       | Python, Django, Flask, data processing | Python-domain issues, backend services, scripts     |
| `dev-go`           | Go, concurrency, CLI tools             | Go-domain issues, services, tooling                 |
| `dev-cpp`          | C++, performance, memory management    | C++-domain issues, systems programming              |
| `techwriter`       | Documentation                          | Documentation phase                                 |

### Phase-to-Agent Mapping

Select agents dynamically based on detected domain:

#### Specification Phase (ENRICH)

| Domain         | Primary Agent    | Secondary Agent  |
| -------------- | ---------------- | ---------------- |
| Security       | security-analyst | system-architect |
| Infrastructure | devops-engineer  | system-architect |
| Frontend       | dev-react        | system-architect |
| Python         | dev-python       | system-architect |
| C++            | dev-cpp          | system-architect |
| Go             | dev-go           | system-architect |
| General        | system-architect | -                |

#### Implementation Phase

| Domain   | Primary Agent                 |
| -------- | ----------------------------- |
| Frontend | dev-react                     |
| Python   | dev-python                    |
| Go       | dev-go                        |
| C++      | dev-cpp                       |
| General  | (detect from file extensions) |

#### Code Review Phase (IMPLEMENTING → E2E)

| Domain         | Primary Reviewer | Secondary Reviewer |
| -------------- | ---------------- | ------------------ |
| Security       | security-analyst | reviewer           |
| Infrastructure | devops-engineer  | reviewer           |
| Frontend       | dev-react        | reviewer           |
| Python         | dev-python       | reviewer           |
| C++            | dev-cpp          | reviewer           |
| Go             | dev-go           | reviewer           |
| Performance    | reviewer         | (profiling tools)  |
| General        | reviewer         | -                  |

#### E2E Phase

| Domain         | Validator                                      |
| -------------- | ---------------------------------------------- |
| Security       | security-analyst (penetration testing mindset) |
| Infrastructure | devops-engineer (operational validation)       |
| Frontend       | dev-react (UI/UX validation)                   |
| Python         | dev-python (functional validation)             |
| C++            | dev-cpp (memory/performance validation)        |
| Go             | dev-go (concurrency validation)                |
| General        | reviewer                                       |

### Multi-Agent Workflows

Some phases benefit from multiple agents working together:

**Sequential (one after another)**:

```
Specification: architect creates spec → security-analyst reviews for security
Code Review: reviewer for quality → security-analyst for security
```

**Parallel (independent reviews)**:

```
Complex feature: architect + security-analyst + devops-engineer (all review spec)
Critical change: reviewer + security-analyst (both review code)
```

### Selection Algorithm

When entering a phase that uses agents:

1. **Detect domains** from issue labels and keywords
2. **Select primary agent** based on domain (see tables above)
3. **Select secondary agents** if:
   - Issue spans multiple domains
   - Phase benefits from multiple perspectives
   - Issue is high-risk or critical
4. **Run agents** sequentially or in parallel as appropriate
5. **Synthesize feedback** and proceed

### Asking User for Agent Preference

For ambiguous cases, use `AskUserQuestion`:

```
Which reviewers should analyze this change?

Options:
- [ ] General reviewer (code quality, best practices)
- [ ] Security analyst (vulnerability assessment)
- [ ] DevOps engineer (infrastructure, deployment)
- [ ] All applicable (comprehensive review)
```

### Examples

**Example 1: Security-related bug fix**

```
Issue: "Fix SQL injection vulnerability in user search"
Labels: type:bug, domain:security

ENRICH phase:
  Primary: security-analyst (vulnerability assessment)
  Secondary: system-architect (if architectural changes needed)

IMPLEMENTING phase:
  Primary: dev-python (or appropriate language agent)

Code Review:
  Primary: security-analyst (verify fix is secure)
  Secondary: reviewer (code quality)
```

**Example 2: Infrastructure feature**

```
Issue: "Add Prometheus metrics to API endpoints"
Labels: type:feature, domain:infra

ENRICH phase:
  Primary: devops-engineer (monitoring design)
  Secondary: system-architect (integration points)

IMPLEMENTING phase:
  Primary: dev-go (or appropriate language agent)

Code Review:
  Primary: devops-engineer (operational correctness)
  Secondary: reviewer (code quality)
```

**Example 3: General feature**

```
Issue: "Add dark mode support"
Labels: type:feature

ENRICH phase:
  Primary: system-architect

IMPLEMENTING phase:
  Primary: dev-react (or appropriate language agent)

Code Review:
  Primary: reviewer
```

---

## Agent Usage Summary (Quick Reference)

| Phase                 | Default Agent               | Security Domain    | Infra Domain      |
| --------------------- | --------------------------- | ------------------ | ----------------- |
| Input negotiation     | (conversation)              | (conversation)     | (conversation)    |
| Exploration           | planning mode               | + security-analyst | + devops-engineer |
| Specification         | architect                   | + security-analyst | + devops-engineer |
| **Test Planning**     | **qa-engineer**             | + security-analyst | + devops-engineer |
| Spec Review           | architect ↔ dev-\*          | + security-analyst | + devops-engineer |
| Implementation        | dev-\* (per language)       | dev-\*             | dev-\*            |
| **TEST-VERIFICATION** | **qa-engineer**             | qa-engineer        | qa-engineer       |
| Code Review           | reviewer + **test-quality** | + security-analyst | + devops-engineer |
| Validation            | reviewer                    | security-analyst   | devops-engineer   |
| Documentation         | techwriter                  | techwriter         | techwriter        |
| Deployment            | devops                      | devops             | devops            |

**Legend**: `+` means "in addition to default", **bold** = new in QA enhancement

---

## Issue Body Templates

### After INPUT (entering Backlog)

```markdown
## {Bug|Feature}: {title}

### Problem/Request

{clarified description}

### Expected Behavior

{what should happen}

### Context

{from negotiation}

**Status**: Backlog
```

### After ENRICH

```markdown
## {Bug|Feature}: {title}

### Summary

{description}

### {Investigation|Specification}

{details}

### Implementation Notes

- Files: {list}

### Definition of Done (Staging Validation)

**Infrastructure**:

- [ ] App running successfully
- [ ] No error logs
- [ ] Metrics healthy
- [ ] No alerts

**Functional**:

- [ ] {check 1}
- [ ] {check 2}

**Regression**:

- [ ] {check 1}

**Status**: Ready
```

### After E2E

```markdown
### E2E Results

**Infrastructure**:

- [x] App running successfully
- [x] No error logs
- [x] Metrics healthy
- [x] No alerts

**Functional**:

- [x] {check 1}
- [x] {check 2}

**Status**: Ready to Merge
```

---

## Knowledge Integration

The crunch skill integrates with the knowledge management system to:

- **Inject** relevant context before starting work
- **Capture** lessons learned as work completes
- **Track** agent effectiveness for future optimization

### Knowledge Injection Points

#### On ENRICH Entry

Before starting investigation or specification:

```
1. Check if .claude/knowledge/index.yaml exists
   IF not: Skip injection (knowledge base empty)

2. Generate context brief:
   - Invoke knowledge-manager agent OR
   - Use /knowledge --brief -r {issue_number}

3. Context brief contains:
   - Related past issues (max 3, relevance > 70)
   - Active architectural decisions for this domain
   - Applicable patterns and anti-patterns
   - Agent effectiveness recommendations

4. Inject into agent prompts:
   - Add "## Knowledge Context" section to system prompt
   - Include for: architect, security-analyst, devops-engineer, dev-* agents
```

**Context Brief Format**:

```markdown
## Knowledge Context

### Related Past Issues

- **#N** (relevance: X%): Brief description - outcome

### Relevant Decisions

- **ADR-N**: Decision title - key point

### Known Patterns

- **PATTERN**: Name - when to use
- **ANTI-PATTERN**: Name - what to avoid

### Agent Effectiveness

- {agent}: X% success on {domain} issues (N total)
```

#### On IMPLEMENTING Entry

Before implementation begins:

```
1. Identify files to be modified (from spec/investigation)

2. Query knowledge base:
   /knowledge -t resolution -t pattern --files {file_list}

3. Extract warnings:
   - Past bugs in these files
   - Review feedback for these modules
   - Patterns applicable to this code

4. Inject into dev agent prompt:
   - "## File-Specific Knowledge" section
   - Include bug history and patterns
```

### Knowledge Capture Points

#### On ENRICH Exit (Spec/Investigation Created)

When transitioning ENRICH → READY:

```
1. Analyze spec/investigation document for decisions

2. IF architectural decision detected:
   - Decision statement present
   - Rationale documented
   - Alternatives mentioned

3. THEN capture decision:
   /learn -t decision -i {issue_number}

4. Extract:
   - Decision title and statement
   - Rationale
   - Alternatives considered
   - Consequences

5. Link to issue and related entries
```

**Decision Detection Keywords**:

- "decided to", "will use", "chose", "selected"
- "approach:", "solution:", "design:"
- "instead of", "rather than", "over"

#### On DONE Transition

When transitioning READY-TO-MERGE → DONE:

```
1. Capture full resolution:
   /learn -i {issue_number}

2. Extract from issue body and branch:
   - Problem description
   - Root cause (for bugs)
   - Solution approach
   - Files changed (from git diff)
   - Tests added
   - Time to resolve (created → closed)

3. Record agent metrics:
   - Agents used in each phase
   - Phase durations
   - Review cycles needed

4. Link to:
   - Decisions made during ENRICH
   - Patterns applied or created
   - Feedback from reviews
```

### Agent Effectiveness Tracking

Track agent performance for knowledge-aware selection:

```
After each agent invocation:
  1. Record: agent, domain, phase, issue_number
  2. On issue completion (DONE):
     - Calculate success: reached DONE without major rework
     - Update agent_metrics in index.yaml

Metrics tracked:
  - total_invocations per agent
  - by_domain: { domain: { count, success_rate, avg_duration } }
  - by_phase: { phase: { count, success_rate } }
```

### Knowledge-Aware Agent Selection

Enhance Dynamic Agent Selection with effectiveness data:

```
Original selection:
  1. Detect domain from labels/keywords
  2. Select agent from phase-to-agent mapping

Enhanced selection:
  1. Detect domain from labels/keywords
  2. Query agent_metrics for domain effectiveness
  3. IF effectiveness data available:
     - Weight selection by past success rate
     - Consider alternatives with better track record
  4. For ambiguous cases:
     - Include effectiveness in AskUserQuestion options
     - Example: "security-analyst (94% on auth, 23 issues)"
```

### Workflow State Hooks Summary

| State Transition | Knowledge Action                                              |
| ---------------- | ------------------------------------------------------------- |
| → ENRICH         | **Inject**: Context brief (related issues, ADRs, patterns)    |
| ENRICH → READY   | **Capture**: Decision (if spec contains architectural choice) |
| → IMPLEMENTING   | **Inject**: File-specific patterns and bug history            |
| Code Review      | **Capture**: Feedback (via /review integration)               |
| → DONE           | **Capture**: Full resolution with metrics                     |

### Error Handling

| Condition              | Handling                                          |
| ---------------------- | ------------------------------------------------- |
| Knowledge base missing | Skip injection, log warning, continue workflow    |
| Injection fails        | Log error, continue without context (don't block) |
| Capture fails          | Log error, continue (capture is best-effort)      |
| Duplicate entry        | Link to existing, don't create new                |

---

## CI MCP Operations

### Platform Detection

Read CLAUDE.md to determine CI platform:

- GitHub/GitLab/Gitea → Use corresponding MCP commands
- Filebase → Use `/ci-filebase` skill commands

### Issue Creation (for new issues)

When creating a new issue via `--input` or quoted text:

| Platform | Operation | Command                                                                 |
| -------- | --------- | ----------------------------------------------------------------------- |
| GitHub   | MCP       | `mcp__github__create_issue(owner, repo, title, body, labels)`           |
| GitLab   | MCP       | `mcp__gitlab__create_issue(project_id, title, description, labels)`     |
| Gitea    | MCP       | `mcp__gitea__create_issue(owner, repo, title, body)`                    |
| Filebase | Skill     | `/ci-filebase issue create "{title}" --body "{body}" --labels {labels}` |

### Issue Operations (get, update, label, close)

| Operation   | GitHub MCP                          | GitLab MCP                       | Gitea MCP                          | Filebase                                      |
| ----------- | ----------------------------------- | -------------------------------- | ---------------------------------- | --------------------------------------------- |
| Get issue   | `mcp__github__get_issue`            | `mcp__gitlab__get_issue`         | `mcp__gitea__get_issue`            | `/ci-filebase issue get {id}`                 |
| Update body | `mcp__github__update_issue`         | `mcp__gitlab__update_issue`      | `mcp__gitea__update_issue`         | `/ci-filebase issue update {id} --body "..."` |
| Set labels  | `mcp__github__set_issue_labels`     | `mcp__gitlab__update_issue`      | `mcp__gitea__replace_issue_labels` | `/ci-filebase issue label {id} {labels}`      |
| Close issue | `mcp__github__update_issue`         | `mcp__gitlab__close_issue`       | `mcp__gitea__close_issue`          | `/ci-filebase issue close {id}`               |
| Add comment | `mcp__github__create_issue_comment` | `mcp__gitlab__create_issue_note` | `mcp__gitea__create_issue_comment` | `/ci-filebase issue comment {id} "{text}"`    |

### PR Operations

| Operation | GitHub MCP                         | GitLab MCP                          | Gitea MCP                         | Filebase                                                         |
| --------- | ---------------------------------- | ----------------------------------- | --------------------------------- | ---------------------------------------------------------------- |
| Create PR | `mcp__github__create_pull_request` | `mcp__gitlab__create_merge_request` | `mcp__gitea__create_pull_request` | `/ci-filebase pr create --title "..." --head "..." --base "..."` |
| Merge PR  | `mcp__github__merge_pull_request`  | `mcp__gitlab__accept_merge_request` | `mcp__gitea__merge_pull_request`  | `/ci-filebase pr merge {id}`                                     |

**Parameters**:

- `title`: First line of input (or first 80 chars)
- `body`: Full input text formatted per template
- `labels`: Type label if known (e.g., `type:bug`, `type:feature`)

### Subagent Invocation Patterns

When this skill is invoked by other agents:

**From review skill** (after code review):

```
/crunch -i {issue_number} -t validation
```

**Creating follow-up issues**:

```
/crunch --input "Follow-up from #{parent}: {description}" --type feature -t ready
```

**Hotfix pipeline**:

```
/crunch --input "Hotfix: {description}" --type bug -t implementing
```

### Non-Interactive Checklist

Before running in non-interactive mode, ensure:

- [ ] Issue number (`-i`) or input text (`--input`) provided
- [ ] Target state (`-t`) provided (skips interactive ask)
- [ ] Type (`--type`) provided or inferable from text
- [ ] CI platform is available:
  - MCP-based (GitHub/GitLab/Gitea): CI MCP server configured
  - Filebase: `.claude/ci-filebase/` directory initialized
- [ ] Required labels exist (in CI platform or `labels.yaml` for Filebase)
- [ ] E2E requirements met:
  - MCP-based: Observability MCPs configured (Prometheus, Loki) OR skip monitoring
  - Filebase: Docker installed and `/ci-filebase docker init` completed OR manual validation only
