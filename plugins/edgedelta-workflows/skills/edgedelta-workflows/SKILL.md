---
name: edgedelta-workflows
description: Generate, validate, and customize EdgeDelta AI workflow configurations with 8 node types, 29 action types, and 5 templates. Use when users need to create, modify, or troubleshoot EdgeDelta workflows.
---

# EdgeDelta Workflows

## Overview

Generate complete EdgeDelta workflow configurations — event-driven AI automation pipelines that connect monitors, external events, and schedules to AI teammates. Workflows are JSON DAGs with typed nodes (start, teammate, task, action, if-else, transform, set-state, note) executed on a distributed async engine.

## Workflow Generation Process

### 1. Determine the Trigger

Ask what initiates the workflow:

| Trigger | Start Node Config | When to Use |
|---------|------------------|-------------|
| Manual / Monitor / Connector | `"start": {}` | Alert-driven, event-driven, or on-demand |
| Periodic (scheduled) | `"start": { "periodicRun": { "schedule": { "cronExpression": "...", "timezone": "..." } } }` | Recurring health checks, reports |

### 2. Select the Template

Match the user's scenario to a template from `references/templates.md`:

| Scenario | Template |
|----------|----------|
| Alert → investigate → notify | Template 1 (Monitor Alert → Email) |
| Scheduled check with branching | Template 2 (Daily Health Check) |
| Full incident response (multi-tool) | Template 3 (Jira + Slack + PagerDuty) |
| Code review automation | Template 4 (PR Review → Slack) |
| Periodic reporting | Template 5 (Capacity → Teams) |
| Custom / none match | Build from scratch using schema |

### 3. Customize the Workflow

For each node, consult the appropriate reference:

- **Node types and schemas**: Read `references/workflow-schema.md`
- **Action configurations**: Read `references/action-types.md` — covers all 29 action types with complete JSON examples
- **Templates**: Read `references/templates.md` — 5 production-tested starting points

### 4. Validate the Configuration

Before returning the workflow, verify:

1. **Exactly one `start` node** exists
2. **All node names** match the pattern `/^[a-zA-Z0-9_-]+$/`
3. **All node names are unique**
4. **Every node** (except `note`) is referenced in at least one link
5. **Every link** references existing node names
6. **No cycles** in the DAG
7. **If-else paths** in links match the branch paths defined in the node
8. **Teammate nodes** have a valid `agentId` — built-in options: `sre`, `devops-engineer`, `cloud-engineer`, `security-engineer`, `code-analyzer`, `issue-coordinator`
9. **Structured output** schemas include `required` and match JSON Schema spec
10. **Action configs** use the correct config key for the action type (e.g., `slackSendMessage` for `slack-send-message`)
11. **Handlebars templates** use triple braces `{{{ }}}` for JSON/HTML content (no escaping) and double braces `{{ }}` for plain values
12. **Integration names** are user-provided — always ask if not specified

## Node Quick Reference

| Node | Purpose | Key Fields |
|------|---------|------------|
| `start` | Entry point | `periodicRun?` (cron + timezone) |
| `teammate` | AI agent with tools + memory | `agentId`, `stepLevelPrompt`, `outputFormat`, `structuredOutput?` |
| `task` | Direct LLM call (no tools/memory) | `prompt`, `outputFormat`, `structuredOutput?` |
| `action` | External integration | `actionType` + type-specific config (see action-types.md) |
| `if-else` | Conditional branching | `branches[]` with `path` + `condition` (JS expressions) |
| `transform` | Data transformation | `steps[]` with `name` + `script` (sandboxed JS) |
| `set-state` | Persist key-value pairs | `assignments[]` with `key` + `expression` |
| `note` | Documentation only | `content` |

## Data Flow

All Handlebars templates and JS expressions access three variables:

| Variable | Contains | Example |
|----------|----------|---------|
| `data` | Workflow input (trigger payload or transformed data) | `{{ data.severity }}` |
| `nodes` | Results from prior nodes, keyed by name | `{{ nodes.investigate.summary }}` |
| `state` | Values set by `set-state` nodes | `{{ state.incidentId }}` |

## Common Patterns

**Fork after investigation** — one teammate node feeds multiple actions in parallel:
```
investigate → create-jira
investigate → notify-slack
investigate → page-oncall
```

**Conditional escalation** — branch based on teammate output:
```
investigate → check-severity → (critical) → page-oncall
                             → (warning) → send-email
                             → (info) → log-only
```

**Data normalization pipeline** — clean input before processing:
```
start → transform (normalize) → set-state (save context) → teammate
```

**Multi-specialist chain** — sequential teammate analysis:
```
sre-investigate → security-review → devops-remediation → notify
```

## Integration Checklist

When generating a workflow, confirm the user has provided (or ask for):

- [ ] Integration names for each action (e.g., Slack integration name, Jira integration name)
- [ ] Channel/project IDs for actions (Slack channel ID, Jira project ID)
- [ ] Email recipients
- [ ] PagerDuty service IDs
- [ ] Custom teammate IDs (if not using built-in agents)
- [ ] Cron schedule and timezone (if periodic)

## Resources

### references/

- **workflow-schema.md** — Complete node type schemas, link rules, trigger types, execution model, Handlebars helpers, and JS expression context. Search patterns: `grep -i "node type\|schema\|field\|required" references/workflow-schema.md`
- **action-types.md** — All 29 action type configurations with full JSON examples for Email, Slack (10), Jira (4), PagerDuty (3), Teams (3), EdgeDelta (1). Search patterns: `grep -i "actionType\|slack\|jira\|pagerduty\|teams\|email" references/action-types.md`
- **templates.md** — 5 production-tested workflow templates with customization guides. Search patterns: `grep -i "template\|use case\|trigger" references/templates.md`
