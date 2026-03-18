# EdgeDelta Workflow Action Types Reference

All action nodes use this structure:

```json
{
  "name": "<node-name>",
  "type": "action",
  "action": {
    "actionType": "<action-type>",
    "<configKey>": { ... }
  }
}
```

All string fields support Handlebars templating: `{{ data.field }}`, `{{ nodes.name.field }}`, `{{ state.key }}`.

---

## Email

### `email`

```json
{
  "actionType": "email",
  "email": {
    "recipients": ["oncall@company.com", "{{ data.assignee_email }}"],
    "cc": ["manager@company.com"],
    "bcc": [],
    "subject": "Alert: {{ data.title }}",
    "body": "Summary: {{{ nodes.investigate.summary }}}\n\nDetails: {{{ toJson data }}}"
  }
}
```

| Field | Type | Required |
|-------|------|----------|
| `recipients` | string[] | Yes |
| `cc` | string[] | No |
| `bcc` | string[] | No |
| `subject` | string | Yes |
| `body` | string | Yes |

---

## Slack (10 Actions)

### `slack-send-message`

```json
{
  "actionType": "slack-send-message",
  "slackSendMessage": {
    "integrationName": "my-slack",
    "channel": "C1234567890",
    "message": "Alert: {{ data.title }}",
    "mentions": [
      { "type": "user", "id": "U123456", "name": "john.doe" },
      { "type": "user-group", "id": "S123456", "name": "oncall-team" }
    ],
    "replyBroadcast": false,
    "updateTopLevel": false,
    "retry": { "interval": 2000, "maximumRetryCount": 3 }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `integrationName` | string | Yes | Slack integration name |
| `channel` | string | Yes | Channel ID (C-prefixed) |
| `message` | string | Yes | Message text (Handlebars) |
| `mentions` | array | No | Users/groups to mention |
| `mentions[].type` | `"user"` \| `"user-group"` | Yes | Mention type |
| `mentions[].id` | string | Yes | Slack user/group ID |
| `mentions[].name` | string | Yes | Display name |
| `replyBroadcast` | boolean | No | Broadcast reply to channel |
| `updateTopLevel` | boolean | No | Update top-level message |
| `retry` | object | No | Retry policy |

### `slack-create-channel`

```json
{
  "actionType": "slack-create-channel",
  "slackCreateChannel": {
    "integrationName": "my-slack",
    "channelName": "incident-{{ data.incidentId }}",
    "isPrivate": true
  }
}
```

### `slack-invite-users`

```json
{
  "actionType": "slack-invite-users",
  "slackInviteUsers": {
    "integrationName": "my-slack",
    "channel": "C1234567890",
    "users": [
      { "id": "U123456", "name": "john.doe" }
    ]
  }
}
```

### `slack-set-channel-description`

```json
{
  "actionType": "slack-set-channel-description",
  "slackSetChannelDescription": {
    "integrationName": "my-slack",
    "channel": "C1234567890",
    "description": "Incident {{ data.incidentId }} response channel"
  }
}
```

### `slack-set-channel-topic`

```json
{
  "actionType": "slack-set-channel-topic",
  "slackSetChannelTopic": {
    "integrationName": "my-slack",
    "channel": "C1234567890",
    "topic": "Active incident: {{ data.title }}"
  }
}
```

### `slack-get-channel-description`

```json
{
  "actionType": "slack-get-channel-description",
  "slackGetChannelDescription": {
    "integrationName": "my-slack",
    "channel": "C1234567890"
  }
}
```

### `slack-get-channel-topic`

```json
{
  "actionType": "slack-get-channel-topic",
  "slackGetChannelTopic": {
    "integrationName": "my-slack",
    "channel": "C1234567890"
  }
}
```

### `slack-list-channels`

```json
{
  "actionType": "slack-list-channels",
  "slackListChannels": {
    "integrationName": "my-slack"
  }
}
```

### `slack-list-channel-members`

```json
{
  "actionType": "slack-list-channel-members",
  "slackListChannelMembers": {
    "integrationName": "my-slack",
    "channel": "C1234567890"
  }
}
```

### `slack-archive-channel`

```json
{
  "actionType": "slack-archive-channel",
  "slackArchiveChannel": {
    "integrationName": "my-slack",
    "channel": "C1234567890"
  }
}
```

---

## Jira (4 Actions)

### `jira-create-issue`

```json
{
  "actionType": "jira-create-issue",
  "jiraCreateIssue": {
    "integrationName": "my-jira",
    "projectId": "10000",
    "issueTypeId": "10001",
    "reporterId": "123456",
    "assigneeId": "654321",
    "summary": "[{{ state.severity }}] {{ data.title }}",
    "description": "Root cause: {{ nodes.investigate.root_cause }}",
    "priorityId": "1"
  }
}
```

| Field | Type | Required |
|-------|------|----------|
| `integrationName` | string | Yes |
| `projectId` | string | Yes |
| `issueTypeId` | string | Yes |
| `reporterId` | string | Yes |
| `assigneeId` | string | No |
| `summary` | string | Yes |
| `description` | string | No |
| `priorityId` | string | No |

### `jira-add-comment`

```json
{
  "actionType": "jira-add-comment",
  "jiraAddComment": {
    "integrationName": "my-jira",
    "issueKey": "PROJ-123",
    "comment": "Update: {{ nodes.investigate.summary }}"
  }
}
```

### `jira-change-status`

```json
{
  "actionType": "jira-change-status",
  "jiraChangeStatus": {
    "integrationName": "my-jira",
    "issueKey": "PROJ-123",
    "targetStatus": "In Progress"
  }
}
```

### `jira-get-issue`

```json
{
  "actionType": "jira-get-issue",
  "jiraGetIssue": {
    "integrationName": "my-jira",
    "issueKey": "PROJ-123"
  }
}
```

---

## PagerDuty (3 Actions)

### `pagerduty-create-incident`

```json
{
  "actionType": "pagerduty-create-incident",
  "pagerdutyCreateIncident": {
    "integrationName": "my-pagerduty",
    "serviceId": "SERVICE123",
    "title": "[AUTO] {{ data.title }}",
    "body": "Investigation: {{ nodes.investigate.root_cause }}",
    "urgency": "high",
    "incidentTypeName": "outage"
  }
}
```

| Field | Type | Required | Values |
|-------|------|----------|--------|
| `integrationName` | string | Yes | |
| `serviceId` | string | Yes | |
| `title` | string | Yes | |
| `body` | string | No | |
| `urgency` | string | No | `"high"`, `"low"` |
| `incidentTypeName` | string | No | |

### `pagerduty-get-on-call-user`

```json
{
  "actionType": "pagerduty-get-on-call-user",
  "pagerdutyGetOnCallUser": {
    "integrationName": "my-pagerduty",
    "scheduleId": "SCHEDULE123"
  }
}
```

### `pagerduty-list-services`

```json
{
  "actionType": "pagerduty-list-services",
  "pagerdutyListServices": {
    "integrationName": "my-pagerduty",
    "nameFilter": "production"
  }
}
```

---

## Microsoft Teams (3 Actions)

### `microsoft-teams-send-message`

```json
{
  "actionType": "microsoft-teams-send-message",
  "microsoftTeamsSendMessage": {
    "integrationName": "my-teams",
    "teamId": "team-123",
    "channelId": "channel-456",
    "message": "Alert: {{ data.title }}",
    "mentions": [
      { "id": "user-123", "name": "John Doe", "role": "user" }
    ]
  }
}
```

### `microsoft-teams-reply-message`

```json
{
  "actionType": "microsoft-teams-reply-message",
  "microsoftTeamsReplyMessage": {
    "integrationName": "my-teams",
    "teamId": "team-123",
    "channelId": "channel-456",
    "parentMessageId": "msg-789",
    "message": "Update: {{ nodes.investigate.summary }}"
  }
}
```

### `microsoft-teams-get-user`

```json
{
  "actionType": "microsoft-teams-get-user",
  "microsoftTeamsGetUser": {
    "integrationName": "my-teams",
    "teamId": "team-123",
    "userEmail": "john@company.com"
  }
}
```

---

## EdgeDelta Internal (1 Action)

### `edgedelta-send-channel-message`

```json
{
  "actionType": "edgedelta-send-channel-message",
  "edgeDeltaSendChannelMessage": {
    "channelId": "<ed-channel-id>",
    "message": "Workflow completed: {{ nodes.investigate.summary }}"
  }
}
```

---

## Action Type Quick Reference

| Action Type | Config Key | Integration |
|-------------|-----------|-------------|
| `email` | `email` | Built-in |
| `slack-send-message` | `slackSendMessage` | Slack |
| `slack-create-channel` | `slackCreateChannel` | Slack |
| `slack-invite-users` | `slackInviteUsers` | Slack |
| `slack-set-channel-description` | `slackSetChannelDescription` | Slack |
| `slack-set-channel-topic` | `slackSetChannelTopic` | Slack |
| `slack-get-channel-description` | `slackGetChannelDescription` | Slack |
| `slack-get-channel-topic` | `slackGetChannelTopic` | Slack |
| `slack-list-channels` | `slackListChannels` | Slack |
| `slack-list-channel-members` | `slackListChannelMembers` | Slack |
| `slack-archive-channel` | `slackArchiveChannel` | Slack |
| `jira-create-issue` | `jiraCreateIssue` | Jira |
| `jira-add-comment` | `jiraAddComment` | Jira |
| `jira-change-status` | `jiraChangeStatus` | Jira |
| `jira-get-issue` | `jiraGetIssue` | Jira |
| `pagerduty-create-incident` | `pagerdutyCreateIncident` | PagerDuty |
| `pagerduty-get-on-call-user` | `pagerdutyGetOnCallUser` | PagerDuty |
| `pagerduty-list-services` | `pagerdutyListServices` | PagerDuty |
| `microsoft-teams-send-message` | `microsoftTeamsSendMessage` | Teams |
| `microsoft-teams-reply-message` | `microsoftTeamsReplyMessage` | Teams |
| `microsoft-teams-get-user` | `microsoftTeamsGetUser` | Teams |
| `edgedelta-send-channel-message` | `edgeDeltaSendChannelMessage` | EdgeDelta |
