# Export and Modify Existing Dashboard

Guide to retrieving an existing dashboard, modifying it, and updating it via MCP.

## Scenario

You have an existing dashboard in EdgeDelta UI that you want to:
- Export to YAML for version control
- Modify locally (add new widget, change layout)
- Update back to EdgeDelta
- Keep as a template for creating similar dashboards

## Prerequisites

- `ED_API_TOKEN` and `ED_ORG_ID` set as environment variables
- Dashboard ID from EdgeDelta UI

## Step 1: Find Dashboard ID

### Option 1: Via MCP Tool
```
get_all_dashboards()
```

Returns a list of all dashboards with their IDs and names:
```
ID              Name                  Description         Tags
dash_abc123...  System Metrics        CPU, Memory, Disk   production, system
dash_def456...  Application Logs      Error tracking      logs, app
```

### Option 2: Via EdgeDelta UI
1. Go to https://app.edgedelta.com/dashboards
2. Click on the dashboard
3. Copy ID from URL: `https://app.edgedelta.com/orgs/{org-id}/dashboards/{dashboard-id}`

## Step 2: Export Dashboard

```
get_dashboard(id="dash_abc123")
```

This returns the full dashboard definition including all widgets. Save it as `system-metrics.yaml`.

## Step 3: Review Exported Dashboard

The exported dashboard will look like:
```yaml
dashboard_name: "System Metrics"
description: "CPU, Memory, Disk monitoring"
tags:
  - production
  - system

definition:
  version: 4
  timeFilters:
    lookback: "1h"
  widgets:
    - id: "root"
      type: "grid"
      grid: "..."
    - id: 1
      type: "viz"
      # ... existing widgets
```

## Step 4: Version Control (Recommended)

```bash
git init  # if not already
git add system-metrics.yaml
git commit -m "Export system metrics dashboard"
```

## Step 5: Modify Dashboard

### Example Modification 1: Add New Widget

Add a disk I/O widget to the exported file:

```yaml
# Add after existing widgets
- id: 6  # Use next available ID
  type: "viz"
  resultType: "timeseries"
  position:
    type: "grid"
    targetId: "root"
    area:
      column: 1
      columnSpan: 12
      row: 7  # Place below existing widgets
      rowSpan: 3
  displayOptions:
    title: "Disk I/O"
    yAxis:
      label: "Bytes/sec"
  visualizer:
    type: "line"
  visuals:
    - id: "A"
      dataSource:
        type: "metric"
        params:
          query: "avg:system.disk.io{*}"
```

Also update the root widget `grid` string to include the extra rows.

### Example Modification 2: Change Layout

Rearrange existing widgets:

```yaml
# BEFORE: Two widgets side-by-side
- id: 1
  position:
    area:
      column: 1
      columnSpan: 6  # Half width
      row: 1
      rowSpan: 3

- id: 2
  position:
    area:
      column: 7
      columnSpan: 6  # Half width
      row: 1
      rowSpan: 3

# AFTER: Stack vertically
- id: 1
  position:
    area:
      column: 1
      columnSpan: 12  # Full width
      row: 1
      rowSpan: 3

- id: 2
  position:
    area:
      column: 1
      columnSpan: 12  # Full width
      row: 4          # Below widget 1
      rowSpan: 3
```

### Example Modification 3: Update Queries

Change metric queries:

```yaml
# BEFORE
dataSource:
  type: "metric"
  params:
    query: "avg:cpu.percent{*}"

# AFTER: More specific query
dataSource:
  type: "metric"
  params:
    query: "avg:system.cpu.utilization{service:api,env:prod}"
```

### Example Modification 4: Add Dashboard Variables

```yaml
definition:
  version: 4

  # Add variables section
  variables:
    - name: "environment"
      type: "custom"
      options:
        - label: "Production"
          value: "prod"
        - label: "Staging"
          value: "staging"
        - label: "Development"
          value: "dev"
      defaultValue: "prod"

  timeFilters:
    lookback: "1h"

  widgets:
    # ... then use variable in queries
    - id: 1
      # ...
      visuals:
        - id: "A"
          dataSource:
            type: "metric"
            params:
              query: "avg:cpu.percent{env:$environment}"  # Use $environment
```

## Step 6: Review Changes

Before updating, review your changes:

```bash
# If using git
git diff system-metrics.yaml
```

Verify:
- Widget positions changed as intended?
- New widgets added correctly?
- Queries updated properly?
- No unintended changes?

## Step 7: Update Dashboard

```
update_dashboard(id="dash_abc123", definition=<modified-definition>)
```

The MCP server validates the definition before applying the update.

## Step 8: Verify in EdgeDelta UI

1. Open the dashboard URL from the update result
2. Check all widgets display correctly
3. Verify new widgets appear
4. Test any new variables
5. Confirm layout matches expectations

## Step 9: Commit Changes

```bash
git add system-metrics.yaml
git commit -m "Add disk I/O widget and rearrange layout"
git push
```

## Advanced Workflows

### Workflow 1: Create Template from Dashboard

Export dashboard and generalize for reuse:

1. Retrieve: `get_dashboard(id="dash_abc123")`
2. Save as `templates/system-metrics-template.yaml`
3. Edit to generalize:
   - Remove specific service tags
   - Add variables for customization
   - Document prerequisites
4. Use template to create new dashboards: `create_dashboard(definition=<template-contents>)`

### Workflow 2: Clone Dashboard to Different Org

1. Export from org 1: `get_dashboard(id="dash_abc123")`
2. Update `ED_ORG_ID` to org 2 in your shell
3. Restart Claude Code to pick up new env var
4. Create in org 2: `create_dashboard(definition=<exported-definition>)`

### Workflow 3: Backup All Dashboards

1. List all: `get_all_dashboards()`
2. For each ID: `get_dashboard(id="<id>")`
3. Save each as a YAML file
4. Commit to git

### Workflow 4: Apply Change to Multiple Dashboards

1. Export all dashboards
2. Modify the definitions (change time range, add widget, etc.)
3. For each: `update_dashboard(id="<id>", definition=<modified>)`

See `batch-operations.md` for more on multi-dashboard workflows.

## Common Modifications

### Add Time Comparison

Compare current vs previous period:

```yaml
visuals:
  - id: "A"  # Current period
    dataSource:
      type: "metric"
      params:
        query: "avg:cpu.percent{*}"

  - id: "B"  # Previous period (1 hour ago)
    dataSource:
      type: "metric"
      params:
        query: "avg:cpu.percent{*}"
      timeFiltersOffset: "-1h"
```

### Add Calculated Metric

Use formula to calculate percentage:

```yaml
- id: "W"
  dataSource:
    type: "formula"
    params:
      formula: "(A / B) * 100"
```

### Change Visualization Type

```yaml
# BEFORE: Big number
visualizer:
  type: "bignumber"
resultType: "aggregate"

# AFTER: Gauge
visualizer:
  type: "gauge"
resultType: "aggregate"
```

### Add Thresholds

```yaml
displayOptions:
  title: "CPU Usage"
  thresholds:
    - value: 0
      color: "#00ff00"
    - value: 80
      color: "#ffaa00"
    - value: 95
      color: "#ff0000"
```

## Troubleshooting

### Issue: "Dashboard not found"
**Solution**: Verify dashboard ID with `get_all_dashboards`. Confirm `ED_ORG_ID` matches the org that owns the dashboard.

### Issue: Update succeeds but changes not visible
**Solution**:
- Hard refresh browser (Ctrl+Shift+R)
- Wait 10-30 seconds for cache
- Use `get_dashboard` to verify the update was persisted

### Issue: "403 Forbidden" on update
**Solution**: Check that your API token has dashboard write permissions. See `troubleshooting.md`.

### Issue: Validation fails after export
**Solution**: This shouldn't happen with properly exported dashboards. If it does:
1. Check for manual edits that introduced errors
2. Compare with original `get_dashboard` output
3. Validate incremental changes using `create_widget`

## Best Practices

1. **Always version control**: Commit before and after modifications
2. **Test in staging**: If available, test updates in non-production org first
3. **Document changes**: Use descriptive commit messages
4. **Back up first**: Call `get_dashboard` and save the original before major modifications
5. **Incremental changes**: Make small changes, test, repeat
6. **Peer review**: Have a teammate review significant changes

## Example: Complete Modification Workflow

```
# 1. Export and save
get_dashboard(id="dash_abc123")
→ save as system-metrics.yaml

# 2. Version control
git add system-metrics.yaml
git commit -m "Export system metrics dashboard"

# 3. Modify system-metrics.yaml (add disk I/O widget)

# 4. Review changes
git diff system-metrics.yaml

# 5. Update
update_dashboard(id="dash_abc123", definition=<modified-definition>)

# 6. Verify in UI (check dashboard in browser)

# 7. Commit
git add system-metrics.yaml
git commit -m "Add disk I/O widget to system metrics"
git push
```
