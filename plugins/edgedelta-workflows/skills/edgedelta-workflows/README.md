# EdgeDelta Workflows Skill

Expert skill for generating, validating, and customizing EdgeDelta AI workflow configurations — event-driven automation pipelines that connect monitors, external events, and schedules to AI teammates.

## Quick Start

```
Create an EdgeDelta workflow for incident response
```

Claude will guide you through trigger selection, template matching, customization, and validation.

## Features

- **5 Production-Tested Templates** - Ready to deploy
- **8 Node Types** - Start, teammate, task, action, if-else, transform, set-state, note
- **29 Action Types** - Email, Slack (10), Jira (4), PagerDuty (3), Teams (3), EdgeDelta (1)
- **Validation** - Check configurations against schema rules before deployment
- **Interactive Builder** - Build custom workflows step-by-step

## Templates

1. **Monitor Alert → AI Investigation → Email** - Alert-driven SRE investigation with email report
2. **Daily Health Check with Branching** - Scheduled check with conditional notifications
3. **Incident Response (Multi-Tool)** - Full response with Jira + Slack + PagerDuty in parallel
4. **PR Review → Slack Summary** - Code review automation with risk-based routing
5. **Capacity Report → Teams** - Weekly capacity analysis posted to Microsoft Teams

## Workflows

- **Quick Deploy**: Choose template → customize → validate (< 2 min)
- **Custom Builder**: Build from scratch using the workflow schema
- **Trigger Setup**: Configure manual, periodic (cron), monitor, or connector triggers
- **Conditional Logic**: Branch workflows based on severity, risk, or custom conditions
- **Multi-Channel Notify**: Fan out to Slack, email, Jira, PagerDuty, and Teams in parallel

## Node Types

| Node | Purpose |
|------|---------|
| `start` | Entry point — manual, scheduled (cron), monitor, or connector trigger |
| `teammate` | AI agent with tools, memory, and optional structured output |
| `task` | Direct LLM call without tools or memory |
| `action` | External integration (email, Slack, Jira, PagerDuty, Teams) |
| `if-else` | Conditional branching with JavaScript expressions |
| `transform` | Data transformation via sandboxed JavaScript |
| `set-state` | Persist key-value pairs for downstream nodes |
| `note` | Documentation only — not executed |

## Supported Integrations

| Integration | Actions |
|-------------|---------|
| Email | Send email with recipients, CC, BCC |
| Slack | Send message, create/archive channel, invite users, get/set topic & description, list channels & members |
| Jira | Create issue, add comment, change status, get issue |
| PagerDuty | Create incident, get on-call user, list services |
| Microsoft Teams | Send message, reply to message, get user |
| EdgeDelta | Send channel message |

## Built-in AI Teammates

| Teammate | ID |
|----------|----|
| SRE | `sre` |
| DevOps Engineer | `devops-engineer` |
| Cloud Engineer | `cloud-engineer` |
| Security Engineer | `security-engineer` |
| Code Analyzer | `code-analyzer` |
| Issue Coordinator | `issue-coordinator` |

## Architecture

Workflows are JSON DAGs executed on a distributed async engine (Restate):
```
Start → Node(s) → Links → Node(s) → ...
```

Data flows through three variables accessible in all Handlebars templates and JS expressions:
- `data` — workflow input (trigger payload)
- `nodes` — results from prior nodes, keyed by name
- `state` — values set by set-state nodes

## Files

```
edgedelta-workflows/
├── SKILL.md                           # Main skill prompt
├── assets/
│   └── references/                    # Technical documentation
│       ├── workflow-schema.md         # Node types, links, triggers, execution model
│       ├── action-types.md            # All 29 action type configurations
│       └── templates.md               # 5 production-tested templates
└── README.md                          # This file
```

## Example Usage

**Alert investigation workflow**:
```
Create a workflow that investigates monitor alerts and emails the results
```

**Incident response**:
```
Build an incident response workflow with Jira tickets, Slack alerts, and PagerDuty pages
```

**Scheduled report**:
```
Set up a daily health check workflow that runs at 9am UTC
```

**Code review automation**:
```
Create a PR review workflow that posts high-risk reviews to Slack
```

## Documentation

- EdgeDelta Docs: https://docs.edgedelta.com/

## Credits

Created for comprehensive EdgeDelta workflow management with production-tested templates and full integration support.
