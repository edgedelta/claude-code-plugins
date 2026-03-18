# Teammate Data Model Reference

Technical reference for the EdgeDelta teammate (Agent) data model, API endpoints, and lifecycle operations.

## Agent Schema

```typescript
Agent {
  // Identity
  id: string                    // ULID, auto-generated
  organizationId: string        // UUID, from auth context
  name: string                  // 1-48 chars, ^[a-zA-Z0-9\s_-]+$
  description?: string          // Free text summary
  avatar?: string               // Data URL or S3 hosted URL
  type: AgentType               // 'user-defined' | 'default' | 'internal' | 'super-agent'
  status: 'active' | 'inactive' | 'error'

  // Prompts (Handlebars templates)
  masterPrompt?: string         // System-level identity and rules
  userPrompt?: string           // User message wrapper template
  toolingPrompt?: string        // Tool description (auto-generated if omitted)

  // Model Configuration
  model?: ModelName             // Defaults to org setting
  modelTemperature?: number     // 0.1-1.0, default 0.1

  // Tool Access
  toolConfigurations?: {
    [connectorName: string]: {
      connectorName: string
      connectorType: string
      configurations: Array<{
        name: string              // Tool name (e.g., 'get-pull-request')
        enabled: boolean          // Is this tool active?
        executionLevel?: 'supervised' | 'unsupervised'
      }>
    }
  }
  connectors?: string[]         // Connector type identifiers

  // Metadata
  capabilities?: string[]       // Capability descriptions
  expertise?: string[]          // Expertise area descriptions
  priority?: number             // Numeric priority level

  // Quality Tracking
  scoreSum?: number             // Cumulative quality scores
  scoreCount?: number           // Number of evaluations

  // Timestamps
  lastActivity?: string         // ISO datetime
  createdAt: string             // ISO datetime
  createdBy: string             // User ID or 'system'
  updatedAt: string             // ISO datetime
  updatedBy: string             // User ID or 'system'
}
```

## API Endpoints

All endpoints require authentication via `X-ED-API-Token` header.

### CRUD Operations

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/orgs/{orgId}/agents` | Create teammate |
| `GET` | `/v1/orgs/{orgId}/agents` | List teammates (paginated) |
| `GET` | `/v1/orgs/{orgId}/agents/{agentId}` | Get teammate |
| `PUT` | `/v1/orgs/{orgId}/agents/{agentId}` | Update teammate |
| `DELETE` | `/v1/orgs/{orgId}/agents/{agentId}` | Delete teammate |

### Predefined Templates

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/orgs/{orgId}/agents/default` | List predefined templates |
| `GET` | `/v1/orgs/{orgId}/agents/default/{id}` | Get specific template |

### Teammate Metadata

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/orgs/{orgId}/agents/{agentId}/versions` | Version history |
| `GET` | `/v1/orgs/{orgId}/agents/{agentId}/references` | Workflows, channels, monitors using this teammate |

## Predefined Teammate IDs

These teammates are auto-backfilled on first organization access:

| ID | Name | Type | Required Connectors |
|----|------|------|---------------------|
| `sre` | SRE | default | Kubernetes, EdgeDelta MCP |
| `code-analyzer` | Code Analyzer | default | GitHub |
| `security-eingineer` | Security Engineer | default | AWS |
| `devops-engineer` | DevOps Engineer | default | Kubernetes, GitHub |
| `issue-coordinator` | Issue Coordinator | default | Jira, Linear |

Internal agent subtypes (not user-visible):
- `teammate-creator`: AI-powered teammate builder
- `connector-creator`: Connector configuration assistant
- `event-normalizer`: Normalizes incoming events
- `channel-determiner`: Routes events to channels
- `thread-title-determiner`: Generates thread titles
- `result-evaluator`: Evaluates response quality
- `importance-evaluator`: Scores message importance
- `thread-summarizer`: Summarizes thread content

## Version Tracking

Versions are automatically created when any of these fields change:
- `masterPrompt`
- `userPrompt`
- `toolingPrompt`
- `model`
- `modelTemperature`
- `toolConfigurations`

```typescript
AgentVersion {
  createdAt: string           // ISO datetime
  createdBy: string           // User ID
  notes?: string              // Optional changelog entry
  masterPrompt?: string
  userPrompt?: string
  toolingPrompt?: string
  model?: ModelName
  modelTemperature?: number
  toolConfigurations?: ToolConfigurations
}
```

## Default Connectors

When creating a teammate, these connectors are automatically attached if available in the organization:
1. `edgedelta-mcp` - EdgeDelta platform access (log search, metrics, pipeline config)
2. `edgedelta-documentation` - EdgeDelta docs reference

## Deletion and Deactivation Guards

### Deactivation Blocked By
- Active workflows referencing the teammate
- Active monitors referencing the teammate

### Deletion Blocked By
- Active workflows
- Channel memberships (non-DM)
- Active monitors

The API returns a `409 Conflict` with the list of blocking references when these guards trigger.

## Organization Settings

Org-level settings that affect teammates:

```typescript
OrgSettings.ONCALL_AI_SETTINGS {
  defaultModelName: ModelName        // Default model for new teammates
  defaultModelTemperature: number    // Default temperature
  forbiddenModels: string[]          // Models blocked in this org
}
```

## Tool Count Limits

- Models have built-in tool limits (varies by model)
- Safety threshold: 10 tools reserved for system use
- Hard limit: 256 tools maximum regardless of model
- Error if enabled tools exceed `model.toolLimit - 10`

## Channel Types

Each teammate gets an auto-created DM channel:

| Teammate Type | DM Channel Type |
|---------------|-----------------|
| `user-defined` | `direct_message` |
| `default` | `direct_message` |
| `internal` | `internal_direct_message` |
| `super-agent` | `super_agent_direct_message` |

## Validation Rules

### Name Validation
- Regex: `^[a-zA-Z0-9\s_-]+$`
- Min length: 1
- Max length: 48
- Error: "Teammate names can only contain alphanumeric characters, spaces, underscores, and hyphens"

### Prompt Validation
- Both `masterPrompt` and `userPrompt` are required for creation
- All prompts must be valid Handlebars templates
- Compilation errors are returned with descriptive messages

### Model Validation
- Must not be in org's `forbiddenModels` list
- Temperature must be 0.1-1.0

### Tool Configuration Validation
- Connector must exist in org's integrations
- Tool must exist in connector's available tools
- Cannot enable a tool that is disabled at connector level
- Cannot set `unsupervised` if connector mandates `supervised`
