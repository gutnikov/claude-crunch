## Observability

> **MUST:** All development work MUST include proper logging and metrics. All code reviews MUST verify observability requirements.

### Structured Logging

Logs are the primary tool for investigating problems.

When writing code that can potentially fail, ask yourself: "Are these logs enough to investigate and reproduce the problem in the future?" Imagine you are the one who will be debugging this at 3 AM.

Design everything from the perspective of a future investigator:

- Before writing code, ask: "What would I need to see in logs to understand what happened?"
- For every decision point: Log the inputs, the decision made, and why
- For every error path: Log context that helps reproduce and diagnose

**Practical guidelines:**

- Use JSON structured logging with consistent fields for processing by systems like Loki or Kibana
- Log at appropriate levels (DEBUG for flow, INFO for events, WARN for issues, ERROR for failures)

### Metrics and Alerts

Design your project so that metrics can be collected by Prometheus and alerts configured in Alertmanager.

Ask yourself:

- "What metrics could help me investigate problems proactively?"
- "What alerts can signal a problem before users notice?"

Design everything from the perspective of a future investigator:

- Before deploying, ask: "What metrics would tell me the system is healthy?"
- For every critical path: Expose latency, error rate, and throughput metrics
- For every dependency: Track availability and response times

**Practical guidelines:**

- Expose metrics in Prometheus format
- Follow naming conventions (e.g., `http_requests_total`, `request_duration_seconds`)
- Set up alerts for anomalies, not just failures
- Include runbook links in alert annotations
