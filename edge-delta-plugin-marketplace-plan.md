# Edge Delta Official Plugins — Marketplace Plan

## Overview

A customer-facing Claude Code plugin marketplace — **Edge Delta Official Plugins** — distributing a curated collection of plugins that leverage **all seven component types** available in the Claude Code plugin system. The marketplace gives Edge Delta customers AI-powered observability automation directly inside their development environment.

### The Seven Plugin Component Types

| Component | How It Works | Edge Delta Use Case |
|-----------|-------------|---------------------|
| **Skills** | Auto-invoked by Claude based on context matching. Claude sees the name/description and loads the full SKILL.md when relevant. | Domain knowledge: pipeline best practices, incident response methodology, Edge Delta query syntax |
| **Slash Commands** | User-triggered shortcuts (`/ed:pipeline`, `/ed:triage`). Markdown files that kick off specific workflows. | Explicit entry points: generate a pipeline config, run a cost report, start an incident investigation |
| **Subagents** | Specialized AI agents that run in isolated context. Can use specific tools, models, and skills. Support parallel execution. | Complex multi-step workflows: incident triage, security audit, PR review for observability changes |
| **Hooks** | Event-driven automation at lifecycle points (PreToolUse, PostToolUse, SessionStart, Stop, etc.). Execute shell commands or prompts automatically. | Quality gates: validate pipeline YAML on save, warn on destructive config changes, enforce tagging standards |
| **MCP Servers** | Structured API connections to external tools/data. Expose tools Claude can call. Handle auth and data formatting. | Core data layer: query Edge Delta logs/metrics/traces, manage alerts, control agent fleet, CRUD pipelines |
| **LSP Servers** | Language Server Protocol integration for code intelligence: go-to-definition, diagnostics, hover docs, symbol search. | Real-time validation: Edge Delta pipeline YAML schema validation, config file diagnostics, autocomplete for ED config syntax |
| **Output Styles** | Custom formatting for Claude's responses. Style definitions in a `styles/` directory. | Consistent formatting: incident reports, cost summaries, pipeline documentation follow Edge Delta branding/structure |

---

## Architecture

### Repository Structure

```
edgedelta/claude-code-plugins/
├── .claude-plugin/
│   └── marketplace.json              # Marketplace manifest
├── plugins/
│   ├── ed-sre-agent/                 # Flagship: SRE incident triage
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json
│   │   ├── agents/                   # Subagents
│   │   │   └── sre-triage.md
│   │   ├── skills/                   # Skills
│   │   │   ├── ed-incident-response/
│   │   │   │   └── SKILL.md
│   │   │   └── ed-runbook-knowledge/
│   │   │       └── SKILL.md
│   │   ├── commands/                 # Slash commands
│   │   │   ├── triage.md
│   │   │   └── postmortem.md
│   │   ├── hooks/                    # Hooks
│   │   │   └── hooks.json
│   │   ├── styles/                   # Output styles
│   │   │   └── incident-report.md
│   │   └── .mcp.json                 # MCP server config
│   │
│   ├── ed-pipeline-builder/          # Pipeline config toolkit
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json
│   │   ├── skills/
│   │   │   └── ed-pipeline-config/
│   │   │       └── SKILL.md
│   │   ├── commands/
│   │   │   ├── pipeline.md
│   │   │   └── validate-pipeline.md
│   │   ├── hooks/
│   │   │   ├── hooks.json
│   │   │   └── scripts/
│   │   │       └── validate-pipeline.sh
│   │   ├── .lsp.json                 # LSP for pipeline YAML
│   │   └── .mcp.json
│   │
│   ├── ed-log-analyzer/              # Log/metric/trace analysis
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json
│   │   ├── skills/
│   │   │   └── ed-log-analysis/
│   │   │       └── SKILL.md
│   │   ├── commands/
│   │   │   └── investigate.md
│   │   ├── styles/
│   │   │   └── analysis-report.md
│   │   └── .mcp.json
│   │
│   ├── ed-security-reviewer/         # Security-focused subagent
│   ├── ed-alert-manager/             # Alert tuning & management
│   ├── ed-k8s-debugger/              # Kubernetes debugging toolkit
│   ├── ed-cost-optimizer/            # Observability cost analysis
│   └── ed-pipeline-lsp/              # Standalone LSP for ED configs
│
└── README.md
```

### Marketplace Manifest (`.claude-plugin/marketplace.json`)

```json
{
  "name": "edge-delta-official-plugins",
  "owner": {
    "name": "Edge Delta",
    "email": "plugins@edgedelta.com"
  },
  "metadata": {
    "description": "Official Edge Delta plugins for observability, SRE, and DevOps workflows in Claude Code",
    "version": "1.0.0",
    "pluginRoot": "./plugins"
  },
  "plugins": [
    {
      "name": "ed-sre-agent",
      "source": "./plugins/ed-sre-agent",
      "description": "AI SRE teammate for incident triage, root cause analysis, and runbook execution against Edge Delta telemetry",
      "version": "1.0.0",
      "category": "incident-response",
      "tags": ["sre", "incidents", "triage", "runbooks", "on-call"],
      "author": { "name": "Edge Delta" },
      "homepage": "https://docs.edgedelta.com/plugins/sre-agent",
      "license": "Apache-2.0",
      "keywords": ["sre", "observability", "incident-response"]
    },
    {
      "name": "ed-pipeline-builder",
      "source": "./plugins/ed-pipeline-builder",
      "description": "Build, validate, and optimize Edge Delta observability pipelines with natural language, real-time YAML validation, and auto-formatting hooks",
      "version": "1.0.0",
      "category": "configuration",
      "tags": ["pipelines", "config", "routing", "processors", "yaml"],
      "author": { "name": "Edge Delta" }
    },
    {
      "name": "ed-log-analyzer",
      "source": "./plugins/ed-log-analyzer",
      "description": "Query and analyze logs, metrics, and traces from Edge Delta with AI-powered pattern detection",
      "version": "1.0.0",
      "category": "analysis",
      "tags": ["logs", "metrics", "traces", "search", "patterns"]
    },
    {
      "name": "ed-security-reviewer",
      "source": "./plugins/ed-security-reviewer",
      "description": "Security-focused agent for threat detection, compliance checks, and audit log analysis",
      "version": "1.0.0",
      "category": "security",
      "tags": ["security", "compliance", "audit", "threats"]
    },
    {
      "name": "ed-alert-manager",
      "source": "./plugins/ed-alert-manager",
      "description": "Tune alert thresholds, reduce noise, and manage notification routing using anomaly detection insights",
      "version": "1.0.0",
      "category": "alerting",
      "tags": ["alerts", "thresholds", "anomaly-detection", "noise-reduction"]
    },
    {
      "name": "ed-k8s-debugger",
      "source": "./plugins/ed-k8s-debugger",
      "description": "Kubernetes debugging toolkit that correlates cluster events with Edge Delta telemetry",
      "version": "1.0.0",
      "category": "debugging",
      "tags": ["kubernetes", "k8s", "debugging", "pods", "cluster"]
    },
    {
      "name": "ed-cost-optimizer",
      "source": "./plugins/ed-cost-optimizer",
      "description": "Analyze observability costs, identify savings opportunities, and optimize data routing strategies",
      "version": "1.0.0",
      "category": "cost-management",
      "tags": ["cost", "optimization", "routing", "tiered-logging"]
    },
    {
      "name": "ed-pipeline-lsp",
      "source": "./plugins/ed-pipeline-lsp",
      "description": "Language server for Edge Delta pipeline YAML: real-time diagnostics, autocomplete, hover docs, and go-to-definition for processor references",
      "version": "1.0.0",
      "category": "developer-tools",
      "tags": ["lsp", "yaml", "validation", "autocomplete", "diagnostics"]
    }
  ]
}
```

---

## Component Deep Dives

### 1. Skills — Auto-Invoked Domain Knowledge

Skills are the most lightweight component. Claude reads the name and description from the SKILL.md frontmatter, and automatically loads the full content when the conversation matches. No user action needed.

**Edge Delta Skills Inventory:**

| Skill | Lives In Plugin | Auto-Activates When... |
|-------|----------------|----------------------|
| `ed-incident-response` | ed-sre-agent | User investigates alerts, outages, error spikes |
| `ed-runbook-knowledge` | ed-sre-agent | User asks about operational procedures or remediation |
| `ed-pipeline-config` | ed-pipeline-builder | User creates/edits pipeline YAML or asks about processors |
| `ed-log-analysis` | ed-log-analyzer | User queries logs, investigates patterns, compares timeframes |
| `ed-alert-tuning` | ed-alert-manager | User works with alert thresholds or noise reduction |
| `ed-k8s-patterns` | ed-k8s-debugger | User debugs Kubernetes issues, pod failures, scaling events |
| `ed-cost-models` | ed-cost-optimizer | User asks about data volumes, cost optimization, tiered logging |
| `ed-security-compliance` | ed-security-reviewer | User reviews security posture, compliance requirements |

**Example Skill: `skills/ed-incident-response/SKILL.md`**

```markdown
---
name: ed-incident-response
description: >
  Incident response best practices for Edge Delta environments. Automatically
  activated when investigating alerts, outages, error spikes, or system 
  degradation. Provides structured triage methodology and Edge Delta-specific 
  debugging patterns.
---

## Incident Response with Edge Delta

### First Response (0-5 minutes)
- Check Edge Delta anomaly detection for automated root cause signals
- Query recent log patterns: look for error rate spikes, new error types
- Check pipeline health: ensure telemetry is flowing correctly
- Verify agent fleet status: confirm collectors are reporting

### Signal Correlation
Edge Delta provides three primary signal types to correlate:
- **Logs**: Full-text search with pattern matching and clustering
- **Metrics**: Time-series with ML-driven anomaly baselines
- **Traces**: Distributed request flows with latency breakdown

### Common Patterns
- Error spike in logs + latency increase in traces → service degradation
- Metric anomaly + no log changes → infrastructure issue (scaling, network)
- New log patterns appearing → recent deployment side effect
- Agent fleet gaps → data collection issue, not application issue

### Edge Delta Query Tips
- Use time-relative queries: `last 1h`, `last 15m` for active incidents
- Filter by severity first, then drill into specific services
- Compare against baseline: same hour yesterday, or pre-deploy window
- Check anomaly scores: ED's ML flags statistical outliers automatically
```

---

### 2. Slash Commands — User-Triggered Workflows

Commands give users explicit entry points. They live as markdown files in the `commands/` directory and are invoked with `/pluginname:commandname`.

**Edge Delta Commands Inventory:**

| Command | Plugin | Purpose |
|---------|--------|---------|
| `/ed-sre-agent:triage` | ed-sre-agent | Start a structured incident triage for a specific service or alert |
| `/ed-sre-agent:postmortem` | ed-sre-agent | Generate a postmortem document from an incident timeline |
| `/ed-pipeline-builder:pipeline` | ed-pipeline-builder | Interactively build a new pipeline configuration |
| `/ed-pipeline-builder:validate` | ed-pipeline-builder | Validate an existing pipeline YAML file |
| `/ed-log-analyzer:investigate` | ed-log-analyzer | Start a guided investigation of a specific service or time window |
| `/ed-alert-manager:tune` | ed-alert-manager | Analyze and tune alert thresholds for noise reduction |
| `/ed-cost-optimizer:report` | ed-cost-optimizer | Generate a cost analysis and savings report |
| `/ed-k8s-debugger:debug` | ed-k8s-debugger | Start K8s debugging correlated with Edge Delta telemetry |

**Example Command: `commands/triage.md`**

```markdown
---
name: triage
description: Start a structured incident triage using Edge Delta telemetry
---

You are starting an incident triage. Use the sre-triage agent and the 
ed-incident-response skill.

Ask the user:
1. Which service or alert are you investigating?
2. When did the issue start (or "now" for active incidents)?
3. What symptoms are you seeing (errors, latency, downtime)?

Then use the Edge Delta MCP tools to:
- Query recent logs for the affected service
- Check for anomalies in the relevant time window
- Pull distributed traces if available
- Review alert history

Produce a structured incident assessment with severity, blast radius, 
root cause hypothesis, and recommended actions.
```

---

### 3. Subagents — Specialized Parallel Agents

Subagents run in isolated context with their own tool sets, model selection, and skill assignments. They're ideal for complex multi-step work that benefits from focus.

**Edge Delta Subagents Inventory:**

| Agent | Plugin | Model | Skills Used | Tools |
|-------|--------|-------|-------------|-------|
| `sre-triage` | ed-sre-agent | sonnet | ed-incident-response, ed-runbook-knowledge | Read, Write, Bash, Grep + ED MCP |
| `security-auditor` | ed-security-reviewer | sonnet | ed-security-compliance | Read, Grep, Glob + ED MCP |
| `k8s-correlator` | ed-k8s-debugger | sonnet | ed-k8s-patterns | Read, Bash, Grep + ED MCP + kubectl |
| `cost-analyst` | ed-cost-optimizer | sonnet | ed-cost-models | Read, Write + ED MCP |
| `pipeline-reviewer` | ed-pipeline-builder | sonnet | ed-pipeline-config | Read, Write, Glob |

**Example Subagent: `agents/sre-triage.md`**

```markdown
---
name: sre-triage
description: >
  AI SRE agent for incident triage and root cause analysis. Spawned when 
  investigating alerts, outages, or performance degradation. Queries Edge 
  Delta telemetry, correlates signals, and produces structured incident reports.
tools: Read, Write, Bash, Grep, Glob
model: sonnet
skills: ed-incident-response, ed-runbook-knowledge
---

You are an expert Site Reliability Engineer with deep experience in 
distributed systems debugging. You have access to Edge Delta's telemetry 
data through MCP tools.

## Triage Workflow

1. **Gather context**: Query recent alerts, anomalies, and log patterns 
   from Edge Delta using the ed-api MCP tools
2. **Correlate signals**: Cross-reference logs, metrics, and traces to 
   identify the blast radius and root cause
3. **Assess severity**: Classify as P1-P4 based on user impact, 
   blast radius, and data loss risk
4. **Recommend actions**: Provide specific remediation steps with 
   commands the engineer can run
5. **Generate report**: Produce a structured incident summary

## Output Format

Always produce a structured incident assessment:
- **Status**: Active / Mitigated / Resolved
- **Severity**: P1-P4 with justification
- **Blast Radius**: Affected services, regions, user segments
- **Root Cause Hypothesis**: Most likely cause with supporting evidence
- **Recommended Actions**: Ordered by priority with specific commands
- **Monitoring**: What to watch during/after remediation
```

---

### 4. Hooks — Event-Driven Automation

Hooks fire automatically at specific lifecycle events. They can run shell commands, invoke prompts, or delegate to agents. This is where you enforce quality gates and automate repetitive checks.

**Available Hook Events:**

| Event | Fires When | Edge Delta Use Case |
|-------|-----------|---------------------|
| `PreToolUse` | Before Claude uses a tool (Write, Edit, Bash, etc.) | Warn before destructive pipeline changes, validate YAML before write |
| `PostToolUse` | After Claude uses a tool | Auto-validate pipeline config after generation, run lint checks |
| `UserPromptSubmit` | When user submits a message | Auto-detect incident context, suggest relevant ED commands |
| `SessionStart` | When a Claude Code session begins | Check ED agent fleet health, load environment context |
| `Stop` | When Claude finishes a response | Append reminders about pipeline deployment steps |
| `PermissionRequest` | When Claude requests elevated permissions | Extra scrutiny for production pipeline modifications |
| `SessionEnd` | When a session ends | Cleanup, save session context |

**Edge Delta Hooks Inventory:**

| Hook | Plugin | Event | Type | Purpose |
|------|--------|-------|------|---------|
| Pipeline YAML validator | ed-pipeline-builder | PostToolUse (Write) | command | Validate `pipeline.yaml` syntax and schema after every write |
| Destructive change warning | ed-pipeline-builder | PreToolUse (Write) | prompt | Warn when editing production pipeline configs |
| PII mask checker | ed-pipeline-builder | PreToolUse (Write) | prompt | Ensure mask processors exist for sensitive fields |
| Alert threshold guard | ed-alert-manager | PreToolUse (Write) | prompt | Warn when thresholds are set outside safe ranges |
| Fleet health check | ed-sre-agent | SessionStart | command | Check agent fleet health on session start |
| Config drift detector | ed-pipeline-builder | SessionStart | command | Detect uncommitted pipeline config changes |
| Credential scanner | ed-security-reviewer | PostToolUse (Write) | command | Check for hardcoded credentials in config files |

**Example Hook Config: `hooks/hooks.json`**

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/scripts/validate-pipeline.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "If the file being written is a pipeline configuration (pipeline.yaml, pipeline.yml, or files in pipelines/ directory), check: 1) Does it include mask processors for PII fields (email, SSN, IP)? 2) Are all destination credentials referenced via environment variables, not hardcoded? 3) Is there at least one filter processor to prevent unbounded data flow? Warn the user about any missing items before proceeding."
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/scripts/check-fleet-health.sh"
          }
        ]
      }
    ]
  }
}
```

**Example Hook Script: `hooks/scripts/validate-pipeline.sh`**

```bash
#!/bin/bash
# Receives hook input as JSON on stdin
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

# Only validate pipeline YAML files
if [[ "$FILE_PATH" == *pipeline*.yaml ]] || [[ "$FILE_PATH" == *pipeline*.yml ]]; then
  # Schema validation using ED CLI or custom validator
  if command -v ed-validate &> /dev/null; then
    RESULT=$(ed-validate pipeline "$FILE_PATH" 2>&1)
    if [ $? -ne 0 ]; then
      echo "⚠️ Pipeline validation failed:"
      echo "$RESULT"
      exit 1
    fi
    echo "✅ Pipeline YAML is valid"
  fi
fi
```

---

### 5. MCP Servers — External API Connections

The MCP server is the shared data layer that gives all plugins access to Edge Delta's APIs. It exposes structured tools that Claude can call to query, create, and manage Edge Delta resources.

**MCP Server Architecture:**

```
servers/ed-mcp-server/
├── src/
│   ├── index.ts              # Server entry point
│   ├── tools/
│   │   ├── logs.ts           # ed_query_logs, ed_search_logs
│   │   ├── metrics.ts        # ed_query_metrics, ed_get_metric_series
│   │   ├── traces.ts         # ed_query_traces, ed_trace_lookup
│   │   ├── alerts.ts         # ed_list_alerts, ed_create_alert, ed_update_alert
│   │   ├── anomalies.ts      # ed_get_anomalies, ed_get_anomaly_detail
│   │   ├── pipelines.ts      # ed_list_pipelines, ed_get_pipeline, ed_validate_pipeline
│   │   ├── fleet.ts          # ed_get_agent_fleet, ed_get_agent_health
│   │   └── integrations.ts   # ed_list_destinations, ed_test_destination
│   ├── auth/
│   │   └── api-client.ts     # Edge Delta API auth + token refresh
│   └── utils/
│       ├── formatters.ts     # Response formatting for Claude
│       └── validators.ts     # Input validation
├── package.json
├── tsconfig.json
└── README.md
```

**MCP Configuration (`.mcp.json`):**

```json
{
  "mcpServers": {
    "ed-api": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/ed-mcp-server/dist/index.js",
      "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config/server.json"],
      "env": {
        "ED_API_URL": "${ED_API_URL:-https://api.edgedelta.com}",
        "ED_API_KEY": "${ED_API_KEY}",
        "ED_ORG_ID": "${ED_ORG_ID}"
      }
    }
  }
}
```

**Full MCP Tool Inventory:**

| Tool | Category | Read/Write | Description |
|------|----------|-----------|-------------|
| `ed_query_logs` | Logs | Read | Search logs with filters (time range, severity, service, pattern) |
| `ed_search_logs` | Logs | Read | Full-text search across all log sources |
| `ed_get_log_patterns` | Logs | Read | Get clustered log patterns for a service/timeframe |
| `ed_query_metrics` | Metrics | Read | Retrieve metric time series with aggregation |
| `ed_get_metric_series` | Metrics | Read | Get raw metric data points for a specific metric |
| `ed_query_traces` | Traces | Read | Search distributed traces by service, duration, status |
| `ed_trace_lookup` | Traces | Read | Get a specific trace by trace ID |
| `ed_list_alerts` | Alerts | Read | List active/recent alerts with context |
| `ed_create_alert` | Alerts | Write | Create a new alert rule |
| `ed_update_alert` | Alerts | Write | Modify alert thresholds or routing |
| `ed_get_anomalies` | Anomalies | Read | Fetch ML-detected anomalies |
| `ed_get_anomaly_detail` | Anomalies | Read | Get detailed analysis of a specific anomaly |
| `ed_list_pipelines` | Pipelines | Read | List all pipeline configurations |
| `ed_get_pipeline` | Pipelines | Read | Get a specific pipeline's config and status |
| `ed_validate_pipeline` | Pipelines | Read | Validate a pipeline config against the schema |
| `ed_deploy_pipeline` | Pipelines | Write | Deploy a pipeline configuration (requires confirmation) |
| `ed_get_agent_fleet` | Fleet | Read | List agent instances and statuses |
| `ed_get_agent_health` | Fleet | Read | Get health metrics for a specific agent |
| `ed_list_destinations` | Integrations | Read | List configured downstream destinations |
| `ed_test_destination` | Integrations | Read | Test connectivity to a destination |

**Security model**: Write operations (`ed_create_alert`, `ed_deploy_pipeline`, etc.) require explicit user confirmation and are opt-in. The default installation is read-only.

---

### 6. LSP Servers — Code Intelligence for Edge Delta Configs

LSP (Language Server Protocol) integration gives Claude real-time code intelligence: diagnostics as you type, autocomplete, hover documentation, go-to-definition, and symbol search. This is especially valuable for Edge Delta's pipeline YAML, which has a complex schema.

**What the Edge Delta LSP provides:**

| Capability | What It Does |
|-----------|-------------|
| **Diagnostics** | Real-time validation of pipeline YAML against the ED schema. Catches unknown processor names, missing required fields, invalid routing targets. Shows errors and warnings inline. |
| **Autocomplete** | Suggests processor names, field names, destination types, and common patterns as you type pipeline configs. |
| **Hover Documentation** | Hover over a processor name or field to see its documentation, supported options, and examples. |
| **Go-to-Definition** | Jump from a destination reference to its configuration block. Navigate between pipeline stages. |
| **Symbol Search** | Find all processors, sources, or destinations defined across pipeline configs in the project. |

**LSP Configuration (`.lsp.json`):**

```json
{
  "yaml": {
    "command": "${CLAUDE_PLUGIN_ROOT}/servers/ed-lsp-server/dist/index.js",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".yaml": "yaml",
      ".yml": "yaml"
    },
    "transport": "stdio",
    "initializationOptions": {
      "schemas": {
        "https://schema.edgedelta.com/pipeline/v1": "pipeline*.{yaml,yml}",
        "https://schema.edgedelta.com/agent/v1": "agent*.{yaml,yml}"
      }
    },
    "settings": {
      "yaml.schemas": {
        "https://schema.edgedelta.com/pipeline/v1": ["pipeline*.yaml", "pipeline*.yml"]
      }
    },
    "startupTimeout": 30000,
    "shutdownTimeout": 10000,
    "maxRestarts": 3
  }
}
```

**LSP Server Implementation Options:**

| Approach | Pros | Cons |
|----------|------|------|
| **Extend yaml-language-server** | Mature, handles YAML parsing. Add ED schema overlay. | Dependency on third-party project. |
| **Custom LSP in TypeScript** | Full control, tailored diagnostics, can validate against live ED API. | More work to build. |
| **Schema-only approach** | Just provide JSON Schema files and let the standard YAML LSP use them. Simplest. | No custom diagnostics or live validation. |

**Recommendation**: Start with the **schema-only approach** (Phase 1) — publish Edge Delta's pipeline JSON Schema and configure the standard yaml-language-server. Graduate to a custom LSP (Phase 3) once you have demand for richer diagnostics and live validation.

---

### 7. Output Styles — Consistent Formatting

Output styles define how Claude formats its responses for specific contexts. They ensure incident reports, cost analyses, and pipeline documentation follow a consistent structure that matches Edge Delta's conventions.

**Edge Delta Output Styles:**

| Style | Plugin | Applied When |
|-------|--------|-------------|
| `incident-report` | ed-sre-agent | Generating incident triage reports or postmortems |
| `postmortem` | ed-sre-agent | Generating formal postmortem documents |
| `cost-analysis` | ed-cost-optimizer | Producing cost reports and savings recommendations |
| `pipeline-doc` | ed-pipeline-builder | Documenting pipeline configurations |
| `analysis-report` | ed-log-analyzer | Presenting log analysis findings |
| `security-audit` | ed-security-reviewer | Formatting security audit results |

**Example Output Style: `styles/incident-report.md`**

```markdown
---
name: incident-report
description: Structured format for Edge Delta incident triage reports
---

## Incident Report Format

When producing an incident report, always use this structure:

### Header
- **Incident ID**: Auto-generated or user-provided
- **Date/Time**: When the incident was detected
- **Reporter**: The engineer running the triage
- **Status**: Active / Mitigated / Resolved

### Summary
A 2-3 sentence plain-language summary of what happened, who was affected,
and the current state.

### Severity Assessment
- **Level**: P1 / P2 / P3 / P4
- **Justification**: Why this severity level
- **Blast Radius**: Services, regions, user segments affected

### Timeline
Chronological list of key events with timestamps, sourced from Edge Delta 
telemetry.

### Root Cause Analysis
- **Hypothesis**: Most likely root cause with supporting evidence
- **Evidence**: Specific log entries, metrics, traces that support this
- **Confidence**: High / Medium / Low

### Actions Taken
What was done during triage (queries run, changes made).

### Recommended Next Steps
Ordered by priority. Include specific commands or procedures.

### Monitoring
What to watch going forward. Include specific Edge Delta queries or 
alert conditions.
```

---

## Plugin Inventory — Full Breakdown by Component Type

### `ed-sre-agent` (Flagship Plugin)

| Component | Details |
|-----------|---------|
| **Skills** | `ed-incident-response`, `ed-runbook-knowledge` |
| **Commands** | `/ed-sre-agent:triage`, `/ed-sre-agent:postmortem` |
| **Subagents** | `sre-triage` (isolated incident investigation agent) |
| **Hooks** | `SessionStart` → check fleet health; `Stop` → remind about follow-up actions |
| **MCP** | `ed-api` (shared Edge Delta API server) |
| **LSP** | — |
| **Output Styles** | `incident-report`, `postmortem` |

### `ed-pipeline-builder`

| Component | Details |
|-----------|---------|
| **Skills** | `ed-pipeline-config` |
| **Commands** | `/ed-pipeline-builder:pipeline`, `/ed-pipeline-builder:validate` |
| **Subagents** | `pipeline-reviewer` (reviews configs for best practices) |
| **Hooks** | `PostToolUse:Write` → validate YAML; `PreToolUse:Write` → warn on prod configs + check PII masking |
| **MCP** | `ed-api` for live pipeline listing and validation |
| **LSP** | YAML language server with Edge Delta pipeline schema |
| **Output Styles** | `pipeline-doc` |

### `ed-log-analyzer`

| Component | Details |
|-----------|---------|
| **Skills** | `ed-log-analysis` |
| **Commands** | `/ed-log-analyzer:investigate` |
| **Subagents** | — |
| **Hooks** | — |
| **MCP** | `ed-api` (log query, pattern analysis, trace lookup) |
| **LSP** | — |
| **Output Styles** | `analysis-report` |

### `ed-alert-manager`

| Component | Details |
|-----------|---------|
| **Skills** | `ed-alert-tuning` |
| **Commands** | `/ed-alert-manager:tune`, `/ed-alert-manager:noise-report` |
| **Subagents** | — |
| **Hooks** | `PreToolUse:Write` → guard against unsafe threshold values |
| **MCP** | `ed-api` (alert CRUD, anomaly baselines) |
| **LSP** | — |
| **Output Styles** | — |

### `ed-security-reviewer`

| Component | Details |
|-----------|---------|
| **Skills** | `ed-security-compliance` |
| **Commands** | `/ed-security-reviewer:audit` |
| **Subagents** | `security-auditor` (parallel security scan agent) |
| **Hooks** | `PostToolUse:Write` → check for hardcoded credentials in configs |
| **MCP** | `ed-api` (audit logs, security event queries) |
| **LSP** | — |
| **Output Styles** | `security-audit` |

### `ed-k8s-debugger`

| Component | Details |
|-----------|---------|
| **Skills** | `ed-k8s-patterns` |
| **Commands** | `/ed-k8s-debugger:debug` |
| **Subagents** | `k8s-correlator` (correlates kubectl output + ED telemetry) |
| **Hooks** | — |
| **MCP** | `ed-api` + optionally kubectl MCP for cluster access |
| **LSP** | — |
| **Output Styles** | — |

### `ed-cost-optimizer`

| Component | Details |
|-----------|---------|
| **Skills** | `ed-cost-models` |
| **Commands** | `/ed-cost-optimizer:report`, `/ed-cost-optimizer:simulate` |
| **Subagents** | `cost-analyst` (deep-dive cost modeling) |
| **Hooks** | — |
| **MCP** | `ed-api` (data volume metrics, destination costs) |
| **LSP** | — |
| **Output Styles** | `cost-analysis` |

### `ed-pipeline-lsp` (Standalone LSP Plugin)

| Component | Details |
|-----------|---------|
| **Skills** | — |
| **Commands** | — |
| **Subagents** | — |
| **Hooks** | `SessionStart` → verify LSP binary is installed |
| **MCP** | — |
| **LSP** | YAML language server with full ED pipeline + agent config schemas |
| **Output Styles** | — |

---

## How the Seven Components Work Together

Here's how a real workflow chains across all component types:

```
User types: /ed-sre-agent:triage
    │
    │  ① COMMAND kicks off the workflow
    ▼
Command asks: "Which service? When did it start?"
    │
    │  ② SKILL auto-activates (ed-incident-response)
    │     Loads triage methodology into context
    ▼
Claude spawns sre-triage subagent
    │
    │  ③ SUBAGENT runs in isolated context with focused tools
    ▼
Subagent calls MCP tools:
    ├── ed_query_logs("payments", "last 1h", "severity:error")
    ├── ed_get_anomalies("payments", "last 2h")
    ├── ed_query_traces("payments", "status:error", "last 1h")
    │
    │  ④ MCP SERVER handles API auth, queries, response formatting
    ▼
Subagent analyzes results, writes incident report
    │
    │  ⑤ OUTPUT STYLE (incident-report) ensures consistent formatting
    ▼
If subagent suggests a pipeline config change:
    │
    │  ⑥ HOOK fires (PreToolUse:Write) — validates YAML before write
    │  ⑥ HOOK fires (PostToolUse:Write) — runs schema validation after
    ▼
While editing pipeline YAML:
    │
    │  ⑦ LSP provides real-time diagnostics, autocomplete, hover docs
    ▼
Final incident report delivered to user
```

---

## Customer Installation Experience

### Quick Start (Individual)

```bash
# 1. Add the Edge Delta marketplace
/plugin marketplace add edgedelta/claude-code-plugins

# 2. Install plugins you need
/plugin install ed-sre-agent@edge-delta-official-plugins
/plugin install ed-pipeline-builder@edge-delta-official-plugins
/plugin install ed-pipeline-lsp@edge-delta-official-plugins

# 3. Set up authentication
export ED_API_KEY="your-api-key"
export ED_ORG_ID="your-org-id"

# 4. Start using it
> Investigate the spike in 5xx errors on the payments service
> /ed-pipeline-builder:pipeline
> /ed-cost-optimizer:report
```

### Team Rollout (Organization)

Add to `.claude/settings.json` in the team's shared repo:

```json
{
  "extraKnownMarketplaces": {
    "edge-delta-official-plugins": {
      "source": {
        "source": "github",
        "repo": "edgedelta/claude-code-plugins"
      }
    }
  },
  "enabledPlugins": [
    "ed-sre-agent@edge-delta-official-plugins",
    "ed-pipeline-builder@edge-delta-official-plugins",
    "ed-log-analyzer@edge-delta-official-plugins",
    "ed-pipeline-lsp@edge-delta-official-plugins"
  ]
}
```

When engineers clone the repo and trust the folder, plugins install automatically.

---

## Implementation Roadmap

### Phase 1: Foundation (Weeks 1-4)

- [ ] Create the GitHub repository `edgedelta/claude-code-plugins`
- [ ] Build the marketplace manifest with all plugin entries
- [ ] **MCP server** (core): Log query, metric query, alert listing, anomaly fetching, auth
- [ ] **ed-pipeline-builder**: Skill + commands + PostToolUse hook for YAML validation
- [ ] **ed-pipeline-lsp**: Schema-only approach (publish JSON Schema, configure yaml-language-server)
- [ ] **Output styles**: incident-report and pipeline-doc templates
- [ ] Setup documentation and README

### Phase 2: Flagship Launch (Weeks 5-8)

- [ ] **ed-sre-agent**: Full subagent + skills + commands + output styles + SessionStart hook
- [ ] **ed-log-analyzer**: Skill + MCP tools + investigation command + output style
- [ ] MCP server expansion: Trace queries, pipeline management, fleet status
- [ ] Hook hardening: PII mask checker, destructive change warnings, credential scanner
- [ ] Internal dogfooding with Edge Delta's SRE team
- [ ] Customer beta (5-10 design partners)

### Phase 3: Expansion (Weeks 9-12)

- [ ] **ed-alert-manager**: Skill + MCP + threshold guard hooks
- [ ] **ed-security-reviewer**: Subagent + skill + credential-check hooks + output style
- [ ] **ed-k8s-debugger**: Subagent + skill + MCP (kubectl integration)
- [ ] **ed-pipeline-lsp** upgrade: Custom LSP with live validation and richer diagnostics
- [ ] Output style refinements based on beta feedback
- [ ] Publish to broader customer base

### Phase 4: Growth (Ongoing)

- [ ] **ed-cost-optimizer**: Skill + commands + cost-analyst subagent + output style
- [ ] **ed-migration-assistant**: Help customers migrate from Datadog/Splunk/New Relic
- [ ] **ed-fleet-manager**: Agent fleet management plugin
- [ ] Community plugin contribution support
- [ ] Analytics on plugin usage and adoption
- [ ] Plugin versioning and auto-update strategy

---

## When to Use Which Component — Decision Framework

```
Is it domain knowledge that Claude should just "know"?
  └─ YES → Skill (auto-activates based on context)

Is it a workflow the user explicitly starts?
  └─ YES → Command (/ed:something)

Does it need isolated context or parallel execution?
  └─ YES → Subagent (specialized agent with focused tools)

Does it need to query or modify an external system?
  └─ YES → MCP Server (structured API tool)

Should it happen automatically on every relevant action?
  └─ YES → Hook (event-driven, fires on Write/Edit/Session events)

Does it provide real-time code intelligence for config files?
  └─ YES → LSP Server (diagnostics, autocomplete, hover docs)

Should Claude's output follow a specific structure?
  └─ YES → Output Style (consistent formatting template)
```

**Composition rule of thumb**: Most real plugins combine 3-5 component types. The flagship `ed-sre-agent` uses six of seven (everything except LSP). Simple utility plugins might only use one or two.

---

## Security Considerations

- **API keys**: Never stored in plugin code; read from environment variables
- **Scoped access**: MCP server respects Edge Delta RBAC — users only see data they have permission for
- **Read-first**: Default MCP tools are read-only. Write operations require explicit opt-in.
- **Hook safety**: All hooks run locally; scripts are auditable in the plugin source. Hooks never send data externally.
- **LSP isolation**: LSP servers only read local files; they don't make network calls (unless using live validation, which only talks to ED API with user's auth)
- **Audit trail**: Log all MCP tool invocations for compliance visibility
- **Private hosting**: Customers can fork and host internally for air-gapped environments

---

## Success Metrics

| Metric | Target (6 months) |
|--------|--------------------|
| Marketplace installs | 500+ unique organizations |
| Weekly active plugin users | 200+ engineers |
| Plugin-assisted incident resolutions | 50+ per month |
| Mean time to triage (with SRE agent) | 60% reduction vs. baseline |
| Pipeline configs generated | 100+ per month |
| LSP active users (daily diagnostics) | 100+ engineers |
| Customer NPS for plugin experience | 50+ |
| Community-contributed plugins | 5+ |

---

## Open Questions for Your Team

1. **API surface**: Which Edge Delta API endpoints should the MCP server prioritize first?
2. **Auth model**: API keys only, or also support OAuth/SSO for enterprise customers?
3. **Hosting**: Public GitHub repo, or private repo with customer access controls?
4. **LSP strategy**: Start schema-only, or invest in a custom LSP immediately?
5. **Hook permissions**: How aggressive should the default hooks be? (Warn vs. block)
6. **Output styles**: Any existing Edge Delta report templates to align with?
7. **Versioning**: Pin plugins to Edge Delta platform versions, or maintain backward compatibility?
8. **Community**: Allow customer-contributed plugins, or first-party only at launch?
