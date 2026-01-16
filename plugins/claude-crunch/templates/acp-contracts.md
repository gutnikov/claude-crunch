# Agent Communication Protocol (ACP) Contracts

<!-- TEMPLATE: Defines input/output contracts for each agent -->

## Overview

Each agent has a defined contract specifying what inputs it accepts, what outputs it produces, and its operational constraints. These contracts enable the orchestrator to route tasks correctly and validate responses.

---

## Contract Schema

```json
{
  "agent": "{agent_name}",
  "version": "1.0",
  "tier": "orchestrator|supervisor|specialist|responder",
  "capabilities": ["capability_list"],
  "input_schema": {
    "actions": {
      "{action_name}": {
        "description": "What this action does",
        "required_context": ["fields"],
        "optional_context": ["fields"]
      }
    }
  },
  "output_schema": {
    "actions": {
      "{action_name}": {
        "required_outputs": ["fields"],
        "optional_outputs": ["fields"]
      }
    }
  },
  "constraints": {
    "timeout_ms": 300000,
    "max_concurrent": 1,
    "requires_human_approval": false
  },
  "dependencies": ["other_agents"],
  "veto_domains": []
}
```

---

## Agent Contracts

### Orchestrator

```json
{
  "agent": "orchestrator",
  "version": "1.0",
  "tier": "orchestrator",
  "capabilities": ["coordinate", "resolve", "route", "checkpoint"],
  "input_schema": {
    "actions": {
      "plan_workflow": {
        "description": "Create execution plan for issue",
        "required_context": ["issue", "complexity_score"],
        "optional_context": ["knowledge_brief", "prior_attempts"]
      },
      "resolve_conflict": {
        "description": "Resolve disagreement between agents",
        "required_context": ["conflict_id", "positions", "votes"],
        "optional_context": ["veto"]
      },
      "merge_findings": {
        "description": "Consolidate findings from multiple agents",
        "required_context": ["findings_list"],
        "optional_context": ["confidence_threshold"]
      }
    }
  },
  "output_schema": {
    "actions": {
      "plan_workflow": {
        "required_outputs": ["execution_graph", "team", "checkpoints"],
        "optional_outputs": ["estimated_duration"]
      },
      "resolve_conflict": {
        "required_outputs": ["resolution", "reasoning"],
        "optional_outputs": ["dissent_record"]
      },
      "merge_findings": {
        "required_outputs": ["merged_findings", "summary"],
        "optional_outputs": ["conflicts_detected"]
      }
    }
  },
  "constraints": {
    "timeout_ms": 60000,
    "max_concurrent": 1,
    "requires_human_approval": false
  },
  "dependencies": [],
  "veto_domains": []
}
```

### System Architect

```json
{
  "agent": "system-architect",
  "version": "1.0",
  "tier": "supervisor",
  "capabilities": ["spec", "design", "decision", "review"],
  "input_schema": {
    "actions": {
      "create_spec": {
        "description": "Create technical specification for feature/fix",
        "required_context": ["issue", "codebase_summary"],
        "optional_context": ["knowledge_brief", "constraints"]
      },
      "design_review": {
        "description": "Review proposed design or architecture",
        "required_context": ["design_doc", "issue"],
        "optional_context": ["related_systems"]
      },
      "investigate": {
        "description": "Investigate issue root cause",
        "required_context": ["issue", "symptoms"],
        "optional_context": ["logs", "metrics"]
      }
    }
  },
  "output_schema": {
    "actions": {
      "create_spec": {
        "required_outputs": [
          "spec_document",
          "files_to_modify",
          "dod_checklist"
        ],
        "optional_outputs": ["alternatives_considered", "risks"]
      },
      "design_review": {
        "required_outputs": ["findings", "approval_status"],
        "optional_outputs": ["suggested_changes"]
      },
      "investigate": {
        "required_outputs": ["root_cause", "affected_components"],
        "optional_outputs": ["related_issues"]
      }
    }
  },
  "constraints": {
    "timeout_ms": 300000,
    "max_concurrent": 2,
    "requires_human_approval": false
  },
  "dependencies": [],
  "veto_domains": []
}
```

### Security Analyst

```json
{
  "agent": "security-analyst",
  "version": "1.0",
  "tier": "supervisor",
  "capabilities": ["threat_model", "security_review", "audit", "veto"],
  "input_schema": {
    "actions": {
      "threat_model": {
        "description": "Create threat model for feature",
        "required_context": ["issue", "spec"],
        "optional_context": ["data_flows", "trust_boundaries"]
      },
      "security_review": {
        "description": "Review code for security vulnerabilities",
        "required_context": ["files", "diff"],
        "optional_context": ["issue", "threat_model"]
      },
      "audit": {
        "description": "Comprehensive security audit",
        "required_context": ["scope"],
        "optional_context": ["focus_areas", "compliance_requirements"]
      }
    }
  },
  "output_schema": {
    "actions": {
      "threat_model": {
        "required_outputs": ["threats", "mitigations", "security_requirements"],
        "optional_outputs": ["attack_trees", "risk_ratings"]
      },
      "security_review": {
        "required_outputs": ["findings", "severity_summary"],
        "optional_outputs": ["false_positives_ruled_out"]
      },
      "audit": {
        "required_outputs": ["findings", "risk_score", "recommendations"],
        "optional_outputs": ["compliance_gaps"]
      }
    }
  },
  "constraints": {
    "timeout_ms": 300000,
    "max_concurrent": 2,
    "requires_human_approval": false
  },
  "dependencies": [],
  "veto_domains": ["security", "auth", "crypto", "data_exposure"]
}
```

### DevOps Engineer

```json
{
  "agent": "devops-engineer",
  "version": "1.0",
  "tier": "supervisor",
  "capabilities": ["infra_design", "deployment_review", "monitoring_setup"],
  "input_schema": {
    "actions": {
      "infra_design": {
        "description": "Design infrastructure for feature",
        "required_context": ["issue", "requirements"],
        "optional_context": ["existing_infra", "constraints"]
      },
      "deployment_review": {
        "description": "Review deployment configuration",
        "required_context": ["deployment_files", "environment"],
        "optional_context": ["rollback_plan"]
      },
      "monitoring_setup": {
        "description": "Design monitoring and alerting",
        "required_context": ["service", "slos"],
        "optional_context": ["existing_dashboards"]
      }
    }
  },
  "output_schema": {
    "actions": {
      "infra_design": {
        "required_outputs": ["architecture", "resources_needed"],
        "optional_outputs": ["cost_estimate", "scaling_plan"]
      },
      "deployment_review": {
        "required_outputs": ["findings", "approval_status"],
        "optional_outputs": ["suggested_changes"]
      },
      "monitoring_setup": {
        "required_outputs": ["metrics", "alerts", "dashboards"],
        "optional_outputs": ["runbooks"]
      }
    }
  },
  "constraints": {
    "timeout_ms": 300000,
    "max_concurrent": 2,
    "requires_human_approval": false
  },
  "dependencies": [],
  "veto_domains": ["deployment", "infrastructure"]
}
```

### Development Agents (dev-python, dev-go, dev-cpp, dev-react)

```json
{
  "agent": "dev-{language}",
  "version": "1.0",
  "tier": "specialist",
  "capabilities": ["implement", "review", "refactor", "test"],
  "input_schema": {
    "actions": {
      "implement": {
        "description": "Implement feature or fix using TDD",
        "required_context": ["issue", "spec", "test_plan"],
        "optional_context": ["knowledge_brief", "existing_tests"]
      },
      "review": {
        "description": "Review code in specific language",
        "required_context": ["files", "diff"],
        "optional_context": ["issue", "style_guide"]
      },
      "refactor": {
        "description": "Refactor code for better quality",
        "required_context": ["files", "goals"],
        "optional_context": ["constraints"]
      }
    }
  },
  "output_schema": {
    "actions": {
      "implement": {
        "required_outputs": ["files_modified", "tests_added", "ci_status"],
        "optional_outputs": ["coverage_report"]
      },
      "review": {
        "required_outputs": ["findings", "approval_status"],
        "optional_outputs": ["suggested_changes"]
      },
      "refactor": {
        "required_outputs": ["files_modified", "improvements"],
        "optional_outputs": ["metrics_before_after"]
      }
    }
  },
  "constraints": {
    "timeout_ms": 600000,
    "max_concurrent": 1,
    "requires_human_approval": false
  },
  "dependencies": ["qa-engineer"],
  "veto_domains": []
}
```

### Reviewer

```json
{
  "agent": "reviewer",
  "version": "1.0",
  "tier": "specialist",
  "capabilities": ["code_review", "doc_review", "config_review"],
  "input_schema": {
    "actions": {
      "code_review": {
        "description": "Review code for quality and best practices",
        "required_context": ["files", "diff"],
        "optional_context": ["issue", "claude_md"]
      },
      "doc_review": {
        "description": "Review documentation",
        "required_context": ["doc_files"],
        "optional_context": ["audience", "style_guide"]
      },
      "config_review": {
        "description": "Review configuration files",
        "required_context": ["config_files"],
        "optional_context": ["environment", "security_requirements"]
      }
    }
  },
  "output_schema": {
    "actions": {
      "code_review": {
        "required_outputs": ["findings", "approval_status"],
        "optional_outputs": ["positive_feedback"]
      },
      "doc_review": {
        "required_outputs": ["findings", "completeness_score"],
        "optional_outputs": ["suggested_improvements"]
      },
      "config_review": {
        "required_outputs": ["findings", "security_concerns"],
        "optional_outputs": ["optimization_suggestions"]
      }
    }
  },
  "constraints": {
    "timeout_ms": 300000,
    "max_concurrent": 3,
    "requires_human_approval": false
  },
  "dependencies": [],
  "veto_domains": []
}
```

### QA Engineer

```json
{
  "agent": "qa-engineer",
  "version": "1.0",
  "tier": "specialist",
  "capabilities": ["test_plan", "coverage_analysis", "test_review"],
  "input_schema": {
    "actions": {
      "create_test_plan": {
        "description": "Create comprehensive test plan",
        "required_context": ["issue", "spec"],
        "optional_context": ["existing_tests", "coverage_baseline"]
      },
      "verify_coverage": {
        "description": "Verify test coverage meets requirements",
        "required_context": ["test_plan", "coverage_report"],
        "optional_context": ["critical_paths"]
      },
      "review_tests": {
        "description": "Review test quality",
        "required_context": ["test_files"],
        "optional_context": ["test_plan"]
      }
    }
  },
  "output_schema": {
    "actions": {
      "create_test_plan": {
        "required_outputs": ["test_plan", "coverage_targets", "critical_paths"],
        "optional_outputs": ["test_data_requirements"]
      },
      "verify_coverage": {
        "required_outputs": ["pass_fail", "gaps", "coverage_score"],
        "optional_outputs": ["recommendations"]
      },
      "review_tests": {
        "required_outputs": ["findings", "quality_score"],
        "optional_outputs": ["weak_tests", "missing_tests"]
      }
    }
  },
  "constraints": {
    "timeout_ms": 300000,
    "max_concurrent": 2,
    "requires_human_approval": false
  },
  "dependencies": [],
  "veto_domains": []
}
```

### Tech Writer

```json
{
  "agent": "techwriter",
  "version": "1.0",
  "tier": "specialist",
  "capabilities": ["documentation", "changelog", "api_docs"],
  "input_schema": {
    "actions": {
      "create_docs": {
        "description": "Create or update documentation",
        "required_context": ["issue", "changes_summary"],
        "optional_context": ["existing_docs", "audience"]
      },
      "create_changelog": {
        "description": "Create changelog entry",
        "required_context": ["issue", "changes_summary"],
        "optional_context": ["version", "breaking_changes"]
      }
    }
  },
  "output_schema": {
    "actions": {
      "create_docs": {
        "required_outputs": ["doc_files", "summary"],
        "optional_outputs": ["diagrams"]
      },
      "create_changelog": {
        "required_outputs": ["changelog_entry"],
        "optional_outputs": ["migration_notes"]
      }
    }
  },
  "constraints": {
    "timeout_ms": 300000,
    "max_concurrent": 1,
    "requires_human_approval": false
  },
  "dependencies": [],
  "veto_domains": []
}
```

### Knowledge Manager

```json
{
  "agent": "knowledge-manager",
  "version": "1.0",
  "tier": "specialist",
  "capabilities": ["query", "capture", "analyze"],
  "input_schema": {
    "actions": {
      "query": {
        "description": "Query knowledge base for relevant information",
        "required_context": ["query_text"],
        "optional_context": ["domains", "entry_types", "min_relevance"]
      },
      "capture": {
        "description": "Capture knowledge from completed work",
        "required_context": ["issue", "resolution"],
        "optional_context": ["agents_used", "decisions_made"]
      },
      "generate_brief": {
        "description": "Generate context brief for agents",
        "required_context": ["issue"],
        "optional_context": ["target_agent", "phase"]
      }
    }
  },
  "output_schema": {
    "actions": {
      "query": {
        "required_outputs": ["entries", "relevance_scores"],
        "optional_outputs": ["summary"]
      },
      "capture": {
        "required_outputs": ["entry_ids", "entry_types"],
        "optional_outputs": ["patterns_updated"]
      },
      "generate_brief": {
        "required_outputs": ["context_brief"],
        "optional_outputs": ["recommendations"]
      }
    }
  },
  "constraints": {
    "timeout_ms": 60000,
    "max_concurrent": 3,
    "requires_human_approval": false
  },
  "dependencies": [],
  "veto_domains": []
}
```

### Incident Responder

```json
{
  "agent": "incident-responder",
  "version": "1.0",
  "tier": "responder",
  "capabilities": ["diagnose", "classify", "remediate"],
  "input_schema": {
    "actions": {
      "diagnose": {
        "description": "Diagnose staging anomaly",
        "required_context": ["anomaly", "metrics", "logs"],
        "optional_context": ["baseline", "recent_changes"]
      },
      "classify": {
        "description": "Classify issue as transient or bug",
        "required_context": ["diagnosis"],
        "optional_context": ["knowledge_matches"]
      }
    }
  },
  "output_schema": {
    "actions": {
      "diagnose": {
        "required_outputs": ["root_cause", "affected_components", "confidence"],
        "optional_outputs": ["evidence"]
      },
      "classify": {
        "required_outputs": ["classification", "confidence"],
        "optional_outputs": ["recommended_action"]
      }
    }
  },
  "constraints": {
    "timeout_ms": 120000,
    "max_concurrent": 1,
    "requires_human_approval": false
  },
  "dependencies": ["log-analyst", "knowledge-manager"],
  "veto_domains": []
}
```

### Log Analyst

```json
{
  "agent": "log-analyst",
  "version": "1.0",
  "tier": "responder",
  "capabilities": ["analyze_logs", "detect_anomalies", "correlate"],
  "input_schema": {
    "actions": {
      "analyze_logs": {
        "description": "Analyze logs for patterns and errors",
        "required_context": ["log_query", "time_range"],
        "optional_context": ["baseline", "focus_patterns"]
      },
      "detect_anomalies": {
        "description": "Detect anomalies in log patterns",
        "required_context": ["current_logs", "baseline_logs"],
        "optional_context": ["thresholds"]
      }
    }
  },
  "output_schema": {
    "actions": {
      "analyze_logs": {
        "required_outputs": ["error_patterns", "frequencies"],
        "optional_outputs": ["stack_traces", "affected_endpoints"]
      },
      "detect_anomalies": {
        "required_outputs": ["anomalies", "severity"],
        "optional_outputs": ["correlation"]
      }
    }
  },
  "constraints": {
    "timeout_ms": 120000,
    "max_concurrent": 2,
    "requires_human_approval": false
  },
  "dependencies": [],
  "veto_domains": []
}
```

### Code Health Analyst

```json
{
  "agent": "code-health-analyst",
  "version": "1.0",
  "tier": "responder",
  "capabilities": ["analyze_complexity", "detect_debt", "track_metrics"],
  "input_schema": {
    "actions": {
      "analyze": {
        "description": "Analyze code health metrics",
        "required_context": ["files"],
        "optional_context": ["baseline", "thresholds"]
      }
    }
  },
  "output_schema": {
    "actions": {
      "analyze": {
        "required_outputs": ["findings", "metrics"],
        "optional_outputs": ["trends", "recommendations"]
      }
    }
  },
  "constraints": {
    "timeout_ms": 300000,
    "max_concurrent": 1,
    "requires_human_approval": false
  },
  "dependencies": [],
  "veto_domains": []
}
```

### Dependency Manager

```json
{
  "agent": "dependency-manager",
  "version": "1.0",
  "tier": "responder",
  "capabilities": ["scan_vulnerabilities", "check_updates", "audit_licenses"],
  "input_schema": {
    "actions": {
      "scan": {
        "description": "Scan dependencies for vulnerabilities",
        "required_context": ["manifest_files"],
        "optional_context": ["severity_threshold"]
      },
      "check_updates": {
        "description": "Check for available updates",
        "required_context": ["manifest_files"],
        "optional_context": ["update_policy"]
      }
    }
  },
  "output_schema": {
    "actions": {
      "scan": {
        "required_outputs": ["vulnerabilities", "severity_summary"],
        "optional_outputs": ["remediation_available"]
      },
      "check_updates": {
        "required_outputs": ["outdated", "available_updates"],
        "optional_outputs": ["breaking_changes"]
      }
    }
  },
  "constraints": {
    "timeout_ms": 180000,
    "max_concurrent": 1,
    "requires_human_approval": false
  },
  "dependencies": [],
  "veto_domains": []
}
```

---

## Contract Validation

The orchestrator validates messages against contracts:

1. **Request validation**: Check required_context fields present
2. **Response validation**: Check required_outputs present
3. **Timeout enforcement**: Fail if agent exceeds timeout_ms
4. **Concurrency check**: Queue if max_concurrent exceeded

## Version Compatibility

- Contracts use semantic versioning
- Major version changes indicate breaking changes
- Orchestrator checks version compatibility before routing
