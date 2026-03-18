# EdgeDelta Workflow Templates

Production-tested workflow templates for common use cases. Each template is a complete, valid workflow configuration.

---

## Template 1: Monitor Alert → AI Investigation → Email Report

**Use case**: Synthetic or threshold monitor fires, SRE teammate investigates, results emailed to on-call.

**Trigger**: Monitor webhook or manual.

```json
{
  "nodes": [
    {
      "name": "start",
      "type": "start",
      "start": {}
    },
    {
      "name": "investigate",
      "type": "teammate",
      "teammate": {
        "agentId": "sre",
        "stepLevelPrompt": "A synthetic monitor triggered an alert.\nEvent details: {{{ toJson data }}}\n- Summarize what failed and when.\n- Determine whether this is likely user-impacting.\n- Check recent activity related to this monitor over the last 24 hours.\n- If any warnings or errors are found in that period, report the most relevant ones.\n- Infer the most likely technical cause based on the alert details and recent context.\n- Recommend the next concrete action an on-call engineer should take.\nReturn the result as valid JSON using the expected output schema.",
        "outputFormat": "structured",
        "structuredOutput": {
          "type": "object",
          "properties": {
            "summary": { "type": "string", "description": "One sentence summary of the alert" },
            "failed_at": { "type": "string", "description": "Timestamp when the failure occurred" },
            "affected_resources": { "type": "string", "description": "URL or name of the service affected" },
            "root_cause_hint": { "type": "string", "description": "Likely technical cause inferred from alert" },
            "recommended_action": { "type": "string", "description": "Clear next step for engineer" },
            "investigation_link": { "type": "string", "description": "Link to Edge Delta investigation view" }
          },
          "required": ["summary", "failed_at", "affected_resources", "root_cause_hint", "recommended_action", "investigation_link"],
          "additionalProperties": false
        }
      }
    },
    {
      "name": "send-email",
      "type": "action",
      "action": {
        "actionType": "email",
        "email": {
          "recipients": ["oncall@company.com"],
          "subject": "Alert: Service Failure Detected",
          "body": "A synthetic monitor detected a failure.\n\nSummary\n{{{ nodes.investigate.summary }}}\n\nFailure Time\n{{{ nodes.investigate.failed_at }}}\n\nAffected Resource\n{{{ nodes.investigate.affected_resources }}}\n\nRoot Cause\n{{{ nodes.investigate.root_cause_hint }}}\n\nRecommended Action\n{{{ nodes.investigate.recommended_action }}}\n\nInvestigation\n{{{ nodes.investigate.investigation_link }}}"
        }
      }
    }
  ],
  "links": [
    { "from": "start", "to": "investigate" },
    { "from": "investigate", "to": "send-email" }
  ]
}
```

**Customization points**:
- `agentId`: change to `devops-engineer`, `cloud-engineer`, etc.
- `recipients`: set to actual email addresses
- `stepLevelPrompt`: tailor investigation instructions
- `structuredOutput`: add/remove fields as needed

---

## Template 2: Daily Health Check with Conditional Branching

**Use case**: Scheduled daily check — sends different emails based on whether issues are found.

**Trigger**: Periodic (cron).

```json
{
  "nodes": [
    {
      "name": "start",
      "type": "start",
      "start": {
        "periodicRun": {
          "schedule": {
            "cronExpression": "0 9 * * *",
            "timezone": "UTC"
          },
          "input": {}
        }
      }
    },
    {
      "name": "check-alerts",
      "type": "teammate",
      "teammate": {
        "agentId": "sre",
        "stepLevelPrompt": "Count alert and warning events from the last 24 hours. Return the count and a brief summary of the top issues if any exist.",
        "outputFormat": "structured",
        "structuredOutput": {
          "type": "object",
          "properties": {
            "alert_count": { "type": "number", "description": "Total alert and warning events in last 24 hours" },
            "top_issues": { "type": "string", "description": "Brief summary of top issues, or 'None' if healthy" }
          },
          "required": ["alert_count", "top_issues"],
          "additionalProperties": false
        }
      }
    },
    {
      "name": "evaluate",
      "type": "if-else",
      "ifElse": {
        "branches": [
          { "path": "healthy", "condition": "nodes['check-alerts'].alert_count == 0" },
          { "path": "issues-found", "condition": "true" }
        ]
      }
    },
    {
      "name": "send-healthy-email",
      "type": "action",
      "action": {
        "actionType": "email",
        "email": {
          "recipients": ["team@company.com"],
          "subject": "Daily Health Check: All Clear",
          "body": "No alerts or warnings detected in the last 24 hours. Systems are healthy."
        }
      }
    },
    {
      "name": "send-issues-email",
      "type": "action",
      "action": {
        "actionType": "email",
        "email": {
          "recipients": ["team@company.com"],
          "subject": "Daily Health Check: Issues Found",
          "body": "Alert count in last 24 hours: {{{ nodes.check-alerts.alert_count }}}\n\nTop issues:\n{{{ nodes.check-alerts.top_issues }}}"
        }
      }
    }
  ],
  "links": [
    { "from": "start", "to": "check-alerts" },
    { "from": "check-alerts", "to": "evaluate" },
    { "from": "evaluate", "to": "send-healthy-email", "path": "healthy" },
    { "from": "evaluate", "to": "send-issues-email", "path": "issues-found" }
  ]
}
```

**Customization points**:
- `cronExpression`: change schedule (e.g., `0 */6 * * *` for every 6 hours)
- `timezone`: set to team's timezone
- Add more branches for different severity levels
- Add Slack/PagerDuty actions on the issues-found path

---

## Template 3: Incident Response — Investigate + Jira + Slack + PagerDuty

**Use case**: Critical incident triggers full response — AI investigation, Jira ticket, Slack notification, and PagerDuty page in parallel.

**Trigger**: Monitor or connector event.

```json
{
  "nodes": [
    {
      "name": "start",
      "type": "start",
      "start": {}
    },
    {
      "name": "normalize",
      "type": "transform",
      "transform": {
        "steps": [
          {
            "name": "normalize-input",
            "script": "data.severity = (data.severity || 'unknown').toLowerCase(); data.timestamp = new Date().toISOString();"
          }
        ]
      }
    },
    {
      "name": "save-context",
      "type": "set-state",
      "setState": {
        "assignments": [
          { "key": "severity", "expression": "data.severity" },
          { "key": "timestamp", "expression": "data.timestamp" }
        ]
      }
    },
    {
      "name": "check-severity",
      "type": "if-else",
      "ifElse": {
        "branches": [
          { "path": "critical", "condition": "data.severity === 'critical'" },
          { "path": "warning", "condition": "data.severity === 'warning'" },
          { "path": "info", "condition": "true" }
        ]
      }
    },
    {
      "name": "investigate",
      "type": "teammate",
      "teammate": {
        "agentId": "sre",
        "stepLevelPrompt": "Critical incident detected at {{ state.timestamp }}.\nDetails: {{{ toJson data }}}\n- Investigate root cause\n- Check recent deployments and changes\n- Assess user impact\n- Recommend immediate remediation steps",
        "outputFormat": "structured",
        "structuredOutput": {
          "type": "object",
          "properties": {
            "root_cause": { "type": "string", "description": "Root cause analysis" },
            "impact": { "type": "string", "description": "User/business impact assessment" },
            "remediation": { "type": "string", "description": "Recommended immediate action" }
          },
          "required": ["root_cause", "impact", "remediation"],
          "additionalProperties": false
        }
      }
    },
    {
      "name": "create-jira",
      "type": "action",
      "action": {
        "actionType": "jira-create-issue",
        "jiraCreateIssue": {
          "integrationName": "my-jira",
          "projectId": "10000",
          "issueTypeId": "10001",
          "reporterId": "automation",
          "summary": "[{{ state.severity }}] {{ data.title }}",
          "description": "Root Cause: {{ nodes.investigate.root_cause }}\nImpact: {{ nodes.investigate.impact }}\nRemediation: {{ nodes.investigate.remediation }}",
          "priorityId": "1"
        }
      }
    },
    {
      "name": "notify-slack",
      "type": "action",
      "action": {
        "actionType": "slack-send-message",
        "slackSendMessage": {
          "integrationName": "my-slack",
          "channel": "C_INCIDENTS",
          "mentions": [
            { "type": "user-group", "id": "S_ONCALL", "name": "oncall-team" }
          ],
          "message": "CRITICAL INCIDENT\n{{ data.title }}\n\nRoot Cause: {{ nodes.investigate.root_cause }}\nImpact: {{ nodes.investigate.impact }}\nRemediation: {{ nodes.investigate.remediation }}",
          "replyBroadcast": false,
          "updateTopLevel": false
        }
      }
    },
    {
      "name": "page-oncall",
      "type": "action",
      "action": {
        "actionType": "pagerduty-create-incident",
        "pagerdutyCreateIncident": {
          "integrationName": "my-pagerduty",
          "serviceId": "PROD_SERVICE",
          "title": "[AUTO] {{ data.title }}",
          "body": "AI Investigation:\n{{ nodes.investigate.root_cause }}\n\nRecommended: {{ nodes.investigate.remediation }}",
          "urgency": "high"
        }
      }
    }
  ],
  "links": [
    { "from": "start", "to": "normalize" },
    { "from": "normalize", "to": "save-context" },
    { "from": "save-context", "to": "check-severity" },
    { "from": "check-severity", "to": "investigate", "path": "critical" },
    { "from": "investigate", "to": "create-jira" },
    { "from": "investigate", "to": "notify-slack" },
    { "from": "investigate", "to": "page-oncall" }
  ]
}
```

**Customization points**:
- Add a `warning` path with lighter notification (email only, no PagerDuty)
- Change `agentId` based on incident type
- Add `slack-create-channel` for dedicated incident channels
- Add `slack-invite-users` to pull responders into the channel

---

## Template 4: PR Review → Slack Summary

**Use case**: GitHub connector triggers Code Analyzer review, posts summary to Slack.

**Trigger**: Connector (GitHub PR event).

```json
{
  "nodes": [
    {
      "name": "start",
      "type": "start",
      "start": {}
    },
    {
      "name": "review",
      "type": "teammate",
      "teammate": {
        "agentId": "code-analyzer",
        "stepLevelPrompt": "A pull request was opened.\nPR details: {{{ toJson data }}}\n- Review code changes for security issues\n- Check for test coverage gaps\n- Identify potential performance concerns\n- Rate overall risk (low/medium/high)\nReturn structured analysis.",
        "outputFormat": "structured",
        "structuredOutput": {
          "type": "object",
          "properties": {
            "risk_level": { "type": "string", "description": "low, medium, or high" },
            "security_issues": { "type": "string", "description": "Security concerns found" },
            "test_gaps": { "type": "string", "description": "Missing test coverage" },
            "performance_notes": { "type": "string", "description": "Performance concerns" },
            "recommendation": { "type": "string", "description": "Approve, request changes, or needs discussion" }
          },
          "required": ["risk_level", "security_issues", "test_gaps", "performance_notes", "recommendation"],
          "additionalProperties": false
        }
      }
    },
    {
      "name": "check-risk",
      "type": "if-else",
      "ifElse": {
        "branches": [
          { "path": "high-risk", "condition": "nodes.review.risk_level === 'high'" },
          { "path": "normal", "condition": "true" }
        ]
      }
    },
    {
      "name": "notify-team",
      "type": "action",
      "action": {
        "actionType": "slack-send-message",
        "slackSendMessage": {
          "integrationName": "my-slack",
          "channel": "C_CODE_REVIEWS",
          "message": "HIGH RISK PR needs attention\nRisk: {{ nodes.review.risk_level }}\nSecurity: {{ nodes.review.security_issues }}\nTests: {{ nodes.review.test_gaps }}\nRecommendation: {{ nodes.review.recommendation }}",
          "mentions": [{ "type": "user-group", "id": "S_LEADS", "name": "tech-leads" }],
          "replyBroadcast": false,
          "updateTopLevel": false
        }
      }
    },
    {
      "name": "post-summary",
      "type": "action",
      "action": {
        "actionType": "slack-send-message",
        "slackSendMessage": {
          "integrationName": "my-slack",
          "channel": "C_CODE_REVIEWS",
          "message": "PR Review Complete\nRisk: {{ nodes.review.risk_level }}\nRecommendation: {{ nodes.review.recommendation }}",
          "mentions": [],
          "replyBroadcast": false,
          "updateTopLevel": false
        }
      }
    }
  ],
  "links": [
    { "from": "start", "to": "review" },
    { "from": "review", "to": "check-risk" },
    { "from": "check-risk", "to": "notify-team", "path": "high-risk" },
    { "from": "check-risk", "to": "post-summary", "path": "normal" }
  ]
}
```

---

## Template 5: Scheduled Capacity Report → Teams Notification

**Use case**: Weekly capacity check, Cloud Engineer analyzes, posts to Microsoft Teams.

**Trigger**: Periodic (weekly).

```json
{
  "nodes": [
    {
      "name": "start",
      "type": "start",
      "start": {
        "periodicRun": {
          "schedule": {
            "cronExpression": "0 8 * * 1",
            "timezone": "UTC"
          },
          "input": {}
        }
      }
    },
    {
      "name": "analyze",
      "type": "teammate",
      "teammate": {
        "agentId": "cloud-engineer",
        "stepLevelPrompt": "Perform a weekly capacity analysis:\n- Check CPU and memory utilization trends over the past 7 days\n- Identify nodes or services approaching resource limits\n- Flag any capacity-related alerts from the past week\n- Provide a forecast for the coming week\n- Recommend scaling actions if needed",
        "outputFormat": "structured",
        "structuredOutput": {
          "type": "object",
          "properties": {
            "overall_status": { "type": "string", "description": "healthy, warning, or critical" },
            "utilization_summary": { "type": "string", "description": "CPU/memory trends" },
            "hotspots": { "type": "string", "description": "Services near limits" },
            "forecast": { "type": "string", "description": "Next week projection" },
            "recommendations": { "type": "string", "description": "Scaling actions needed" }
          },
          "required": ["overall_status", "utilization_summary", "hotspots", "forecast", "recommendations"],
          "additionalProperties": false
        }
      }
    },
    {
      "name": "post-to-teams",
      "type": "action",
      "action": {
        "actionType": "microsoft-teams-send-message",
        "microsoftTeamsSendMessage": {
          "integrationName": "my-teams",
          "teamId": "platform-team",
          "channelId": "capacity-planning",
          "message": "Weekly Capacity Report\n\nStatus: {{ nodes.analyze.overall_status }}\n\nUtilization:\n{{ nodes.analyze.utilization_summary }}\n\nHotspots:\n{{ nodes.analyze.hotspots }}\n\nForecast:\n{{ nodes.analyze.forecast }}\n\nRecommendations:\n{{ nodes.analyze.recommendations }}",
          "mentions": []
        }
      }
    }
  ],
  "links": [
    { "from": "start", "to": "analyze" },
    { "from": "analyze", "to": "post-to-teams" }
  ]
}
```

---

## Template Selection Guide

| Scenario | Template | Key Nodes |
|----------|----------|-----------|
| Alert investigation + notification | Template 1 | teammate → email |
| Scheduled health check with branching | Template 2 | teammate → if-else → email |
| Full incident response (multi-tool) | Template 3 | transform → if-else → teammate → jira + slack + pagerduty |
| Code review automation | Template 4 | teammate → if-else → slack |
| Periodic reporting | Template 5 | teammate → teams |

### Combining Templates

Templates can be combined. Common patterns:
- Add `transform` + `set-state` before any teammate node for data normalization
- Add `if-else` after any teammate node for conditional routing
- Fork after investigation to notify multiple channels/tools in parallel
- Chain multiple teammate nodes for multi-specialist analysis
