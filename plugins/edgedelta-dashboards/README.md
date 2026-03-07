# EdgeDelta Dashboards Plugin

Create, update, delete, and manage EdgeDelta dashboards directly from Claude Code. Includes AI-assisted dashboard design with full v4 schema knowledge, widget builder workflow, and validation guidance.

## Features

- **List & inspect** dashboards across your organization
- **Create dashboards** using a guided widget builder workflow or raw JSON
- **Update & delete** existing dashboards
- **V4 schema knowledge** — Claude understands widget types, grid layout rules, and visualizer options
- **Built-in validation guidance** — catches common errors before API calls

## Prerequisites

1. **Docker** installed and running ([get Docker](https://docs.docker.com/get-docker/))
2. **EdgeDelta API token** — [create one here](https://docs.edgedelta.com/api-tokens/)
3. **EdgeDelta organization ID** — [find it here](https://docs.edgedelta.com/my-organization/)

> **Note:** Dashboard management tools (create, update, delete, assemble) require EdgeDelta MCP Server with dashboard support, available in the next MCP server release. The get_all_dashboards and get_dashboard read tools are available today.

## Setup

Set your credentials as environment variables before starting Claude Code:

```
export ED_API_TOKEN="your-api-token"
export ED_ORG_ID="your-org-id"
```

Or add them to your shell profile (~/.zshrc, ~/.bashrc) for persistence.

## Available MCP Tools

| Tool | Description |
|------|-------------|
| get_all_dashboards | List all dashboards in your organization |
| get_dashboard | Get full configuration for a specific dashboard |
| create_dashboard | Create a new dashboard from a JSON definition |
| update_dashboard | Update an existing dashboard |
| delete_dashboard | Permanently delete a dashboard |
| get_dashboard_schema | Get the v4 widget schema (filter by category) |
| create_widget | Build and validate a widget configuration |
| assemble_dashboard | Combine widget configs and create a dashboard in one call |

## Usage Examples

List all dashboards: "Show me all dashboards in my EdgeDelta organization"
Create a dashboard: "Create a dashboard with a line chart showing error rates over time and a big number showing total log count"
Export and modify: "Export the API Monitoring dashboard so I can add a new widget to it"

## Dashboard Templates

This plugin includes starter templates in skills/edgedelta-dashboards/assets/:
- quickstart-dashboard.yaml — Minimal single-widget example
- system-metrics.yaml — Multi-panel metrics dashboard
- logs-analysis.yaml — Log-focused dashboard

## Authentication

This plugin uses EdgeDelta API token authentication. The MCP server reads ED_API_TOKEN and ED_ORG_ID from your environment — no credentials are stored in the plugin.

## Resources

- EdgeDelta Documentation: https://docs.edgedelta.com
- EdgeDelta MCP Server: https://github.com/edgedelta/edgedelta-mcp-server
- EdgeDelta API Tokens: https://docs.edgedelta.com/api-tokens/
