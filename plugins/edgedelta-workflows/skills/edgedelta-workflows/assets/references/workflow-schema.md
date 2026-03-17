# EdgeDelta Workflow Configuration Schema

## Top-Level Structure

A workflow configuration is a JSON object with two required fields:

```json
{
  "nodes": [ ... ],
  "links": [ ... ]
}
```

## Node Schema

Every node has these common fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier. Pattern: `/^[a-zA-Z0-9_-]+$/` |
| `type` | string (enum) | Yes | One of: `start`, `teammate`, `task`, `action`, `if-else`, `transform`, `set-state`, `note` |
| `description` | string | No | Human-readable description |
| `metadata` | string | No | JSON-encoded UI positioning: `{"position": {"x": 0, "y": 0}}` |

Plus a type-specific configuration field matching the node type.

---

## Node Type: `start`

Entry point for the workflow. Exactly one required per workflow.

```json
{
  "name": "start",
  "type": "start",
  "start": {
    "periodicRun": {
      "schedule": {
        "cronExpression": "0 9 * * *",
        "timezone": "America/New_York"
      },
      "input": {}
    }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `start.periodicRun` | object | No | Omit for manual/webhook triggers |
| `start.periodicRun.schedule.cronExpression` | string | Yes (if periodic) | Standard cron (5 fields) |
| `start.periodicRun.schedule.timezone` | string | Yes (if periodic) | IANA timezone (e.g., `UTC`, `America/New_York`) |
| `start.periodicRun.input` | object | No | Static input data passed to workflow |

---

## Node Type: `teammate`

Invokes an AI teammate with LLM, tools, memory, and optional structured output.

```json
{
  "name": "investigate",
  "type": "teammate",
  "teammate": {
    "agentId": "sre",
    "stepLevelPrompt": "Investigate this alert.\nDetails: {{{ toJson data }}}",
    "outputFormat": "structured",
    "iterationLimit": 50,
    "structuredOutput": {
      "type": "object",
      "properties": {
        "summary": { "type": "string", "description": "Brief summary" },
        "root_cause": { "type": "string", "description": "Root cause analysis" }
      },
      "required": ["summary", "root_cause"],
      "additionalProperties": false
    }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `agentId` | string | Yes | Built-in: `sre`, `devops-engineer`, `cloud-engineer`, `security-engineer`, `code-analyzer`, `issue-coordinator`. Or a custom teammate ID. |
| `stepLevelPrompt` | string | Yes | Handlebars template. Variables: `data`, `nodes`, `state` |
| `outputFormat` | `"string"` \| `"structured"` | No | Default: `"string"` |
| `iterationLimit` | number | No | Max tool-use iterations. Default: 50 |
| `structuredOutput` | JSON Schema | Required if `outputFormat: "structured"` | JSON Schema for output validation |

### Handlebars Helpers in Prompts

| Helper | Usage | Description |
|--------|-------|-------------|
| `{{{ toJson data }}}` | Triple braces + toJson | Serialize object as JSON (no HTML escaping) |
| `{{ data.field }}` | Double braces | Insert field value (HTML escaped) |
| `{{{ nodes.nodeName.field }}}` | Triple braces | Insert prior node output (no escaping) |
| `{{ state.key }}` | Double braces | Insert state variable |

---

## Node Type: `task`

Direct LLM invocation without full teammate context (no memory, no tools).

```json
{
  "name": "summarize",
  "type": "task",
  "task": {
    "prompt": "Summarize the following data:\n{{{ toJson data }}}",
    "outputFormat": "string"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `prompt` | string | Yes | Handlebars template |
| `outputFormat` | `"string"` \| `"structured"` | No | Default: `"string"` |
| `structuredOutput` | JSON Schema | Required if structured | Output schema |

---

## Node Type: `action`

Executes an external integration action. See `action-types.md` for all 29 action types.

```json
{
  "name": "notify",
  "type": "action",
  "action": {
    "actionType": "slack-send-message",
    "slackSendMessage": {
      "integrationName": "my-slack",
      "channel": "C1234567890",
      "message": "Alert: {{ data.title }}",
      "mentions": [],
      "replyBroadcast": false,
      "updateTopLevel": false
    }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `actionType` | string (enum) | Yes | See action-types.md for full list |
| `<actionConfig>` | object | Yes | Config object matching the actionType |

All string fields in action configs support Handlebars templating with `data`, `nodes`, `state`.

### Retry Policy (Optional on Supported Actions)

```json
{
  "retry": {
    "interval": 2000,
    "maximumRetryCount": 3
  }
}
```

| Field | Type | Default | Constraints |
|-------|------|---------|-------------|
| `interval` | number (ms) | 1000 | Max: 60000 |
| `maximumRetryCount` | number | 0 | Range: 0-5 |

---

## Node Type: `if-else`

Conditional branching with JavaScript expressions.

```json
{
  "name": "check-severity",
  "type": "if-else",
  "ifElse": {
    "branches": [
      { "path": "critical", "condition": "data.severity === 'critical'" },
      { "path": "warning", "condition": "data.severity === 'warning'" },
      { "path": "else", "condition": "true" }
    ]
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `branches` | array | Yes | Ordered list of condition/path pairs |
| `branches[].path` | string | Yes | Label used in link routing |
| `branches[].condition` | string | Yes | JavaScript expression |

### Condition Expression Context

| Variable | Contains |
|----------|----------|
| `data` | Current workflow item data |
| `nodes` | Results from all previously executed nodes (keyed by node name) |
| `state` | State variables set by set-state nodes |

### Supported Operators

Comparison: `===`, `!==`, `==`, `!=`, `>`, `<`, `>=`, `<=`
Logical: `&&`, `||`, `!`
Property access: dot notation, bracket notation
Methods: `.includes()`, `.length`, etc.

---

## Node Type: `transform`

Data transformation via sandboxed JavaScript.

```json
{
  "name": "normalize",
  "type": "transform",
  "transform": {
    "steps": [
      {
        "name": "clean-data",
        "description": "Normalize severity and add timestamp",
        "script": "data.severity = data.severity.toLowerCase(); data.processedAt = new Date().toISOString();"
      }
    ]
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `steps` | array | Yes | Ordered list of transform steps |
| `steps[].name` | string | Yes | Step identifier |
| `steps[].description` | string | No | Human-readable description |
| `steps[].script` | string | Yes | JavaScript code |

### Script Context

| Variable | Access | Description |
|----------|--------|-------------|
| `data` | Read/Write | Current workflow item data |
| `nodes` | Read | Prior node results |
| `state` | Read | State variables |

### Available JavaScript Features

- Array methods: `map()`, `filter()`, `reduce()`, `push()`, `find()`, `some()`, `every()`
- String methods: `toLowerCase()`, `toUpperCase()`, `trim()`, `split()`, `replace()`
- Object manipulation: `delete`, spread, destructuring
- Math operations and `Math.*` functions
- `Date` constructor and methods
- `JSON.parse()`, `JSON.stringify()`
- Ternary operator, template literals

---

## Node Type: `set-state`

Persist key-value pairs for downstream nodes.

```json
{
  "name": "save-context",
  "type": "set-state",
  "setState": {
    "assignments": [
      { "key": "severity", "expression": "data.severity" },
      { "key": "total", "expression": "data.items.reduce((s, i) => s + i.cost, 0)" },
      { "key": "label", "expression": "data.priority > 3 ? 'urgent' : 'normal'" }
    ]
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `assignments` | array | Yes | List of key/expression pairs |
| `assignments[].key` | string | Yes | State variable name |
| `assignments[].expression` | string | Yes | JavaScript expression (same context as if-else) |

---

## Node Type: `note`

Documentation node. No execution.

```json
{
  "name": "info",
  "type": "note",
  "note": {
    "content": "This branch handles critical incidents only."
  }
}
```

---

## Links Schema

Links define the DAG edges between nodes.

```json
{
  "links": [
    { "from": "start", "to": "investigate" },
    { "from": "investigate", "to": "check-severity" },
    { "from": "check-severity", "to": "escalate", "path": "critical" },
    { "from": "check-severity", "to": "log-only", "path": "info" }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from` | string | Yes | Source node name |
| `to` | string | Yes | Target node name |
| `path` | string | No | Routing label. Default: `"default"`. Must match if-else branch paths. |

### Link Rules

1. Every node referenced in links must exist in the nodes array
2. Every node in the nodes array must be referenced in at least one link (except `note`)
3. Links must not create cycles (DAG only)
4. `if-else` nodes use `path` to route to different targets based on branch conditions
5. A node can have multiple outgoing links (fork) — all matched targets execute
6. A node can have multiple incoming links (join)

---

## Trigger Types

Workflows can be triggered four ways:

| Trigger | Configuration | Entry Point |
|---------|--------------|-------------|
| **Manual** | No `periodicRun` in start node | API call or UI button |
| **Periodic** | `periodicRun` with cron in start node | Automatic on schedule |
| **Monitor** | Monitor notification targets `@channel` or webhook | Monitor alert fires |
| **Connector** | Event-workflow trigger mapping | External event arrives |

### Monitor Webhook Trigger

```
POST https://workflow.ai.edgedelta.com/webhook/trigger
Headers:
  X-ED-Org-Id: <org UUID>
  X-ED-Monitor-Id: <monitor UUID>
  X-ED-Workflow-Name: <workflow name>
Body: serialized alert data (becomes `data` in workflow)
```

### Connector Event Trigger

Configured via API — maps integration event types to workflow IDs:

```json
{
  "integrationName": "pagerduty",
  "eventType": "incident.triggered",
  "workflowId": "<workflow-id>"
}
```

---

## Execution Model

- **Engine**: Restate (distributed async framework)
- **Traversal**: BFS queue — nodes execute in breadth-first order
- **Durability**: Each step persisted; survives restarts
- **Timeout**: 2 hours max per run, 15-minute inactivity timeout
- **Teammate timeout**: 960s total, 240s per step, up to 50 iterations
- **State**: Shared `WorkflowRunnerState` accessible to all nodes via `state`

### Node Response

Each node returns:
- `status`: `"forward"` (continue), `"block"` (stop branch), `"error"`
- `path`: routing label for outgoing links (default: `"default"`)
- `item`: updated data for downstream nodes

### Data Flow

```
start → data passed as item.data
  → node1 processes item, result stored in item.nodes["node1"]
  → node2 receives same item, can access item.data AND item.nodes["node1"]
  → Handlebars: {{ data.field }}, {{ nodes.node1.field }}, {{ state.key }}
```
