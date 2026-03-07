---
name: edgedelta-dashboards
version: 1.1.0
last_updated: 2025-10-21
description: Create, update, delete, and manage EdgeDelta dashboards via the EdgeDelta MCP server. Use when users need to build monitoring dashboards, add widgets, visualize metrics or logs, export dashboard configurations, or work with the EdgeDelta dashboard schema. Recognizes phrases like "build a dashboard", "add a widget", "dashboard schema", "visualize my metrics", "export dashboard", "EdgeDelta panel", "create a chart", "update my dashboard".
---

# EdgeDelta Dashboard Management Skill

This skill provides expert guidance for managing EdgeDelta dashboards using the EdgeDelta MCP server. Activate this skill when users need to work with EdgeDelta dashboards in any capacity.

## When to Activate This Skill

Activate this skill when the user mentions:
- Creating, updating, exporting, or deleting EdgeDelta dashboards
- Dashboard templates or YAML/JSON configurations
- EdgeDelta visualization, widgets, or panels
- Dashboard validation errors or schema issues
- Batch dashboard operations or migrations
- "edgedelta-dashboards" CLI tool
- Dashboard authentication or login issues

## Prerequisites and Setup

### Required Environment Variables

Set your EdgeDelta credentials in your shell:

```bash
export ED_API_TOKEN="your-api-token"
export ED_ORG_ID="your-org-id"
```

Add to `~/.zshrc` or `~/.bashrc` for persistence.

Get your API token: https://docs.edgedelta.com/api-tokens/
Get your org ID: https://docs.edgedelta.com/my-organization/

See the Troubleshooting section below for auth error resolution.

### Docker Requirement

The EdgeDelta MCP server runs via Docker. Ensure Docker is installed and running:

```bash
docker --version
```

The MCP server starts automatically when Claude Code connects to it.

## Core Operations

### 1. List Dashboards

Use the `get_all_dashboards` tool to list all dashboards in the organization:

- Call `get_all_dashboards` to retrieve dashboard names and IDs
- Pass `include_definitions: true` to also retrieve full dashboard definitions

**Use Case**: Discovery, inventory, finding dashboard IDs before other operations

### 2. Get a Dashboard

Use the `get_dashboard` tool to retrieve a single dashboard:

- Requires `dashboard_id` (obtain from `get_all_dashboards` first)
- Returns the full dashboard definition

**Use Case**: Inspecting existing dashboard config before modification, backup, template creation

### 3. Create Dashboard

Two workflows available:

**Builder workflow (recommended)**:
1. Call `get_dashboard_schema` to discover available widget types and parameters
2. Call `create_widget` for each widget in the dashboard
3. Call `assemble_dashboard` with all widget configs — handles layout automatically

**Direct workflow**:
- Call `create_dashboard` with a raw JSON definition

**Use Case**: New dashboards, standardized deployments, infrastructure as code

### 4. Update Dashboard

Use the `update_dashboard` tool:

- Requires `dashboard_id` + a full JSON definition
- Workflow: call `get_dashboard` to export first, modify the definition locally, then call `update_dashboard`

**Use Case**: Dashboard modifications, standardization, bulk updates

### 5. Delete Dashboard

Use the `delete_dashboard` tool:

- Requires `dashboard_id`
- This operation is irreversible — confirm with the user before proceeding

**Use Case**: Cleanup, decommissioning, removing test dashboards

## MCP Dashboard Builder Workflow

The builder workflow is the recommended way to create dashboards. It guides Claude through widget-by-widget construction with validation at each step.

### Workflow Steps

1. **Discover widget types**: Call `get_dashboard_schema` to understand available widget types, data source types, and parameters. Filter by category to reduce response size: `timeseries`, `scalar`, `aggregates`, `other`, `layout`.

2. **Create each widget**: Call `create_widget` for each widget. Returns a validated widget config or specific error messages if parameters are invalid.

3. **Assemble and create**: Call `assemble_dashboard` with all widget configs. Handles grid layout automatically (2 widgets per row, 6 columns each, or use custom `column`/`row` position parameters).

### When to Use Each Tool

| Tool | When to Use |
|------|-------------|
| `get_dashboard_schema` + `create_widget` + `assemble_dashboard` | Creating new dashboards — provides validation and auto-layout |
| `create_dashboard` | Applying a pre-built JSON definition you already have |
| `update_dashboard` | Modifying an existing dashboard (export first, modify, then update) |
| `get_all_dashboards` | Discovery — finding dashboard IDs before get/update/delete |
| `get_dashboard` | Inspecting existing dashboard config before modification |

### Auto-Layout Rules

`assemble_dashboard` auto-positions widgets if `column`/`row` not specified:
- 2 widgets per row (6 columns each)
- 4 rows tall per widget by default
- Override with `column`, `column_span`, `row`, `row_span` parameters

## Batch Operations

The EdgeDelta MCP server currently supports individual dashboard operations. For bulk workflows:

- **Export all dashboards**: Call `get_all_dashboards` with `include_definitions: true`, then save each definition to YAML/JSON files for version control
- **Create multiple dashboards**: Prepare JSON definitions and call `create_dashboard` for each
- **Migration**: Export with `get_all_dashboards`, modify definitions locally, re-create with `create_dashboard`

## Dashboard Schema (v4)

### Complete Dashboard Structure

```yaml
dashboard_name: "My Dashboard"
description: "Dashboard description (optional)"
tags:
  - tag1
  - tag2
definition:
  version: 4
  timeFilters:
    lookback: "1h"  # 15m, 30m, 1h, 3h, 6h, 12h, 24h, 7d, 30d
  widgets:
    - # Root widget (REQUIRED!)
      id: "root"
      type: "grid"
      grid: "72px 72px 72px / 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr"

    - # Example viz widget
      id: 1
      type: "viz"
      resultType: "aggregate"
      position:
        type: "grid"
        targetId: "root"
        area:
          column: 1        # 1-12 (1-indexed)
          columnSpan: 6    # Width in columns
          row: 1           # 1+ (1-indexed)
          rowSpan: 2       # Height in rows
      displayOptions:
        title: "Widget Title"
      visualizer:
        type: "bignumber"  # See visualizer types below
      visuals:
        - id: "A"
          dataSource:
            type: "log"
            params:
              query: "*"
```

### Critical Schema Rules

1. **Root Widget is MANDATORY**
   - Every dashboard MUST have exactly one root widget
   - Must have: `id: "root"`, `type: "grid"`, and `grid` property
   - Standard grid: 12 columns × N rows (72px per row)

2. **All non-root widgets MUST have `position`**
   - type: "grid"
   - targetId: "root" (or another container)
   - area: {column, columnSpan, row, rowSpan}

3. **Grid positioning is 1-indexed** (not 0-indexed!)
   - Columns: 1-12
   - Rows: 1 to N
   - Widget must fit: column + columnSpan ≤ 13

4. **Version must be 4**

### Widget Types (6 total)

- `viz`: Data visualizations (charts, graphs, stats)
- `markdown`: Rich text/HTML content
- `grid`: Layout container (required as root)
- `tabs`: Tab navigation container
- `variable-control`: Dashboard variable controls
- `empty`: Placeholder widget

### Visualizer Types (24 total)

**Single-value** (resultType: aggregate):
- `bignumber` - Large single metric
- `gauge` - Gauge chart

**Multi-value** (resultType: aggregate):
- `pie`, `donut` - Circular charts
- `column` - Bar charts
- `radar` - Radar chart
- `treemap`, `sunburst` - Hierarchical views
- `sankey` - Flow diagram
- `bubble` - Bubble chart
- `list` - List view
- `geomap` - Geographic map

**Timeseries** (resultType: timeseries):
- `line`, `area`, `bar` - Time series charts
- `scatter`, `step`, `smooth` - Specialized time series

**Table**:
- `table` - Formatted table
- `raw-table` - Raw data table

**Special**:
- `json` - JSON viewer
- `empty` - Placeholder

### Data Source Types (7 total)

- `metric`: Metric queries (e.g., `avg:cpu.percent{*}`)
- `log`: Log filter queries
- `trace`: Distributed tracing data
- `event`: Event data
- `pattern`: Log pattern analysis
- `formula`: Mathematical formulas (e.g., `A + B`)
- `empty`: Placeholder

## Validation Error Handling

The MCP server includes pre-upload validation. When validation fails, AI-friendly error messages are shown.

### Common Validation Errors

#### Missing Root Widget
```
ERROR: Dashboard MUST have a root widget. Root widget MUST have:
id='root', type='grid', grid='<css-grid-string>'.

FIX: Add a root grid widget as the first widget in the widgets array.
```

**Solution**: Always start widgets array with root widget:
```yaml
widgets:
  - id: "root"
    type: "grid"
    grid: "72px 72px 72px / 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr"
```

#### Missing Position
```
ERROR: Widget #1 is missing 'position' field. All non-root widgets MUST have a position.

FIX: Add position: {'type': 'grid', 'targetId': 'root', 'area': {...}} to widget #1.
```

**Solution**: Add position to every non-root widget:
```yaml
position:
  type: "grid"
  targetId: "root"
  area:
    column: 1
    columnSpan: 6
    row: 1
    rowSpan: 2
```

#### Grid Boundary Violation
```
ERROR: Widget extends beyond grid bounds: column 7 + span 8 = 15
(exceeds standard 12-column grid).

FIX: Reduce columnSpan or adjust column position to stay within 12 columns.
```

**Solution**: Ensure `column + columnSpan ≤ 13`:
```yaml
# BAD: 7 + 8 = 15 > 13
area:
  column: 7
  columnSpan: 8

# GOOD: 7 + 6 = 13
area:
  column: 7
  columnSpan: 6
```

#### Invalid Version
```
ERROR: Dashboard version must be 4, got: 3
```

**Solution**: Always use version 4:
```yaml
definition:
  version: 4
```

## Template Creation Workflow

When helping users create dashboards from scratch:

### 1. Understand Requirements
Ask about:
- Dashboard purpose (metrics, logs, traces, mixed)
- Data sources (what metrics/logs are available)
- Layout preferences (number of panels, organization)
- Time range needs
- Target audience

### 2. Start with Template
Offer to use one of these starting points:
- `quickstart-dashboard.yaml` - Minimal 1-widget example
- `system-metrics.yaml` - Multi-panel metrics dashboard
- `logs-analysis.yaml` - Log-focused dashboard

Located in: `${CLAUDE_PLUGIN_ROOT}/skills/edgedelta-dashboards/assets/`

### 3. Customize Step-by-Step
Guide through:
1. Root widget setup (usually no changes needed)
2. First widget creation (layout, visualizer, data source)
3. Additional widgets (positioning, no overlaps)
4. Metadata (name, description, tags)
5. Time filters

### 4. Validate Before Create
Always validate locally before attempting to create:
- Check root widget exists
- Verify all positions are valid
- Ensure grid boundaries respected
- Confirm data source queries are valid

### 5. Create and Verify
Use `assemble_dashboard` (builder workflow) or `create_dashboard` (direct JSON), then verify in the EdgeDelta UI.

## Best Practices

### Dashboard Design
1. **Start with root widget** - Always the first widget in array
2. **Plan grid layout** - Sketch 12-column layout before coding
3. **Use consistent sizing** - Standard heights: 2, 3, 4 rows
4. **Avoid overlaps** - Grid positions should not conflict
5. **Logical grouping** - Related metrics together
6. **Responsive design** - Full-width widgets for mobile

### Widget Layout
- **BigNumber/Gauge**: 2-4 columns wide, 2 rows tall
- **Charts**: 6-12 columns wide, 3-4 rows tall
- **Tables**: 12 columns wide, 4+ rows tall
- **Markdown**: Flexible, typically 6-12 columns

### Data Sources
- **Test queries first** - Use EdgeDelta UI to validate queries
- **Use semantic names** - Visual IDs: A, B, C (not random)
- **Optimize queries** - Specific filters, appropriate aggregations
- **Consider cardinality** - Group by limited dimensions

### Templates
- **Version control** - Store templates in git
- **Parameterize** - Use consistent patterns for easy updates
- **Document** - Add comments explaining complex widgets
- **Test thoroughly** - Create in test org before production

## Troubleshooting Guide

### Authentication Issues

**Problem**: "authentication failed" or "401 Unauthorized"
**Solution**: Check your environment variables are set:
```bash
echo $ED_API_TOKEN
echo $ED_ORG_ID
```
If empty, set them and restart Claude Code:
```bash
export ED_API_TOKEN="your-token"
export ED_ORG_ID="your-org-id"
```

**Problem**: "403 Forbidden"
**Solution**: Your API token may lack required permissions. Create a new token with dashboard read/write access at https://docs.edgedelta.com/api-tokens/

**Problem**: Docker not running
**Solution**: Start Docker Desktop or the Docker daemon before starting Claude Code.

### Validation Errors

**Problem**: "Dashboard MUST have a root widget"
**Solution**: Add root widget as first widget in array (see schema above)

**Problem**: "Widget #N is missing 'position' field"
**Solution**: Add position to widget (see schema above)

**Problem**: "Widget extends beyond grid bounds"
**Solution**: Adjust column/columnSpan to fit in 12 columns

**Problem**: "Unknown widget type: X"
**Solution**: Use valid widget type (viz, markdown, grid, tabs, variable-control, empty)

**Problem**: "Unknown visualizer type: X"
**Solution**: Use valid visualizer from list above

## When to Read Detailed References

Load reference documentation selectively based on the specific task:

- **schema-reference.md** - When creating custom widgets or troubleshooting schema errors. Contains complete widget/visualizer/dataSource type definitions.
- **validation-guide.md** - When validation fails or pre-checking complex templates. Contains detailed validation rules and edge cases.
- **troubleshooting.md** - When encountering authentication, API, or MCP errors. Contains comprehensive error resolution steps.
- **create-from-scratch.md** - When building a completely new dashboard with no starting template. Contains step-by-step creation workflow.
- **batch-operations.md** - When migrating or managing multiple dashboards. Contains bulk operation strategies and migration patterns.
- **export-and-modify.md** - When customizing existing dashboards. Contains best practices for export-modify-update workflow.

**Reference location**: `${CLAUDE_PLUGIN_ROOT}/skills/edgedelta-dashboards/references/`

## Quick Reference

### MCP Tools

```
# List all dashboards
get_all_dashboards [include_definitions: true/false]

# Get a specific dashboard
get_dashboard(dashboard_id: "<id>")

# Builder workflow
get_dashboard_schema [category: timeseries|scalar|aggregates|other|layout]
create_widget(widget_type: "<type>", data_source_type: "<type>", ...)
assemble_dashboard(name: "<name>", widgets: [...])

# Direct create / update / delete
create_dashboard(dashboard_definition: "<json>")
update_dashboard(dashboard_id: "<id>", dashboard_definition: "<json>")
delete_dashboard(dashboard_id: "<id>")
```

### Asset Templates
Dashboard starter templates are available in:
`${CLAUDE_PLUGIN_ROOT}/skills/edgedelta-dashboards/assets/`
- `quickstart-dashboard.yaml` — Minimal single-widget example
- `system-metrics.yaml` — Multi-panel metrics dashboard
- `logs-analysis.yaml` — Log-focused dashboard

