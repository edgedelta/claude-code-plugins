# Creating a Dashboard from Scratch

Step-by-step guide to creating a new EdgeDelta dashboard using MCP tools.

## Scenario

You want to create a new dashboard to monitor application performance with:
- CPU usage metric (big number)
- Memory usage metric (big number)
- Request rate over time (line chart)
- Error rate over time (line chart)

## Prerequisites

- Docker installed and running
- `ED_API_TOKEN` and `ED_ORG_ID` set as environment variables
- Application metrics being collected in EdgeDelta

## Step 1: Verify Authentication

Check that environment variables are set in your shell:
```bash
echo $ED_API_TOKEN   # Should print your token
echo $ED_ORG_ID      # Should print your org ID
```

Then verify the MCP connection works:
```
Use get_all_dashboards tool to list existing dashboards
```

If this returns an error, see `troubleshooting.md` for authentication issues.

## Step 2: Design the Dashboard Layout

Sketch your layout before building:
```
[CPU  ][MEM  ][     ][     ]  ← Rows 1-2 (cols 1-3, 4-6)
[      Request Rate         ]  ← Rows 3-5 (cols 1-12)
[       Error Rate          ]  ← Rows 6-8 (cols 1-12)
```

**Grid math**: `column + columnSpan ≤ 13` (12-column layout)

## Step 3: Verify Queries in EdgeDelta UI (Recommended)

Before building the dashboard, verify your queries return data:

1. Go to EdgeDelta UI: https://app.edgedelta.com
2. Navigate to Metrics explorer
3. Test each query:
   - `avg:system.cpu.utilization{service:myapp}`
   - `avg:system.memory.utilization{service:myapp}`
   - `sum:http.server.request.count{service:myapp}`
   - `sum:http.server.request.count{service:myapp,status:5xx}`

If queries return no data:
- Verify metric names are correct
- Check service tag matches your application
- Adjust time range if needed
- Ensure metrics are being collected

## Step 4: Build Widgets with create_widget

Use `create_widget` to validate each widget before assembly:

**Widget 1: CPU Usage (bignumber)**
```
create_widget(
  type="viz",
  visualizerType="bignumber",
  resultType="aggregate",
  title="CPU Usage",
  dataSourceType="metric",
  query="avg:system.cpu.utilization{service:myapp}",
  position={"column": 1, "columnSpan": 3, "row": 1, "rowSpan": 2}
)
```

**Widget 2: Memory Usage (bignumber)**
```
create_widget(
  type="viz",
  visualizerType="bignumber",
  resultType="aggregate",
  title="Memory Usage",
  dataSourceType="metric",
  query="avg:system.memory.utilization{service:myapp}",
  position={"column": 4, "columnSpan": 3, "row": 1, "rowSpan": 2}
)
```

**Widget 3: Request Rate (line chart)**
```
create_widget(
  type="viz",
  visualizerType="line",
  resultType="timeseries",
  title="Request Rate (req/s)",
  dataSourceType="metric",
  query="sum:http.server.request.count{service:myapp}",
  position={"column": 1, "columnSpan": 12, "row": 3, "rowSpan": 3}
)
```

**Widget 4: Error Rate (line chart)**
```
create_widget(
  type="viz",
  visualizerType="line",
  resultType="timeseries",
  title="Error Rate (errors/s)",
  dataSourceType="metric",
  query="sum:http.server.request.count{service:myapp,status:5xx}",
  position={"column": 1, "columnSpan": 12, "row": 6, "rowSpan": 3}
)
```

Each call returns a validated widget config. Collect all four.

## Step 5: Assemble and Create the Dashboard

```
assemble_dashboard(
  name="Application Performance",
  description="Monitor application CPU, memory, requests, and errors",
  tags=["application", "performance", "monitoring"],
  timeFilters={"lookback": "1h"},
  widgets=[<widget1>, <widget2>, <widget3>, <widget4>]
)
```

The tool validates the full layout and creates the dashboard in one call.

Expected result:
```
Dashboard created successfully
Dashboard ID: dash_abc123def456
Name: Application Performance
URL: https://app.edgedelta.com/orgs/your-org-id/dashboards/dash_abc123def456
```

## Step 6: Verify in EdgeDelta UI

1. Click the URL from the result, or:
2. Go to https://app.edgedelta.com
3. Navigate to Dashboards
4. Find "Application Performance"
5. Verify all 4 widgets display correctly

## Step 7: Iterate and Improve

If something doesn't look right:

1. Retrieve the current state:
   ```
   get_dashboard(id="dash_abc123def456")
   ```

2. Modify the definition (adjust widget configs, positions, queries)

3. Update the dashboard:
   ```
   update_dashboard(id="dash_abc123def456", definition=<updated-definition>)
   ```

## Reference: Full Dashboard JSON Structure

For direct `create_dashboard` usage (bypassing the builder workflow):

```yaml
dashboard_name: "Application Performance"
description: "Monitor application CPU, memory, requests, and errors"
tags:
  - application
  - performance
  - monitoring

definition:
  version: 4

  timeFilters:
    lookback: "1h"

  widgets:
    # ROOT WIDGET - Always required!
    - id: "root"
      type: "grid"
      grid: "72px 72px 72px 72px 72px 72px 72px 72px / 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr"
      # 8 rows × 12 columns

    # WIDGET 1: CPU Usage (top-left)
    - id: 1
      type: "viz"
      resultType: "aggregate"
      position:
        type: "grid"
        targetId: "root"
        area:
          column: 1
          columnSpan: 3
          row: 1
          rowSpan: 2
      displayOptions:
        title: "CPU Usage"
      visualizer:
        type: "bignumber"
      visuals:
        - id: "A"
          dataSource:
            type: "metric"
            params:
              query: "avg:system.cpu.utilization{service:myapp}"

    # WIDGET 2: Memory Usage
    - id: 2
      type: "viz"
      resultType: "aggregate"
      position:
        type: "grid"
        targetId: "root"
        area:
          column: 4
          columnSpan: 3
          row: 1
          rowSpan: 2
      displayOptions:
        title: "Memory Usage"
      visualizer:
        type: "bignumber"
      visuals:
        - id: "A"
          dataSource:
            type: "metric"
            params:
              query: "avg:system.memory.utilization{service:myapp}"

    # WIDGET 3: Request Rate (full width)
    - id: 3
      type: "viz"
      resultType: "timeseries"
      position:
        type: "grid"
        targetId: "root"
        area:
          column: 1
          columnSpan: 12
          row: 3
          rowSpan: 3
      displayOptions:
        title: "Request Rate (req/s)"
        yAxis:
          label: "Requests/sec"
          min: 0
      visualizer:
        type: "line"
      visuals:
        - id: "A"
          dataSource:
            type: "metric"
            params:
              query: "sum:http.server.request.count{service:myapp}"

    # WIDGET 4: Error Rate (full width)
    - id: 4
      type: "viz"
      resultType: "timeseries"
      position:
        type: "grid"
        targetId: "root"
        area:
          column: 1
          columnSpan: 12
          row: 6
          rowSpan: 3
      displayOptions:
        title: "Error Rate (errors/s)"
        yAxis:
          label: "Errors/sec"
          min: 0
      visualizer:
        type: "line"
      visuals:
        - id: "A"
          dataSource:
            type: "metric"
            params:
              query: "sum:http.server.request.count{service:myapp,status:5xx}"
```

## Common Adjustments

### Adjust Grid Layout

If widgets are too small/large:

```yaml
# Make CPU/Memory widgets taller
rowSpan: 3  # Instead of 2

# Make them wider
columnSpan: 4  # Instead of 3

# Adjust subsequent widgets' row positions accordingly
```

### Change Time Range

```yaml
# Dashboard-level
timeFilters:
  lookback: "3h"  # Instead of "1h"

# Per-widget offset
dataSource:
  type: "metric"
  params: {...}
  timeFiltersOffset: "-1d"  # Compare to yesterday
```

### Add Colors/Thresholds

```yaml
displayOptions:
  title: "CPU Usage"
  thresholds:
    - value: 0
      color: "#00ff00"   # Green below 75%
    - value: 75
      color: "#ffaa00"   # Orange 75-90%
    - value: 90
      color: "#ff0000"   # Red above 90%
```

### Add Formula Widget

Calculate error rate percentage:

```yaml
- id: 5
  type: "viz"
  resultType: "aggregate"
  position:
    type: "grid"
    targetId: "root"
    area:
      column: 7
      columnSpan: 3
      row: 1
      rowSpan: 2
  displayOptions:
    title: "Error Rate %"
  visualizer:
    type: "bignumber"
  visuals:
    - id: "W"  # Formulas use W-Z
      dataSource:
        type: "formula"
        params:
          formula: "(A / B) * 100"  # (errors / total requests) * 100
```

## Tips

### Tip 1: Start Simple
Build with 1-2 widgets first, verify they work, then add more.

### Tip 2: Use Consistent Sizing
- Big numbers: 2-3 columns × 2 rows
- Small charts: 6 columns × 3 rows
- Large charts: 12 columns × 3-4 rows

### Tip 3: Plan Grid Layout
Sketch your layout first:
```
[CPU][MEM][---][---]  ← Row 1-2
[  Request Rate   ]  ← Row 3-5
[   Error Rate    ]  ← Row 6-8
```

### Tip 4: Tag for Organization
```yaml
tags:
  - application      # What it monitors
  - performance      # Category
  - team-platform    # Ownership
  - prod             # Environment
```

### Tip 5: Use the Builder Workflow
`get_dashboard_schema` → `create_widget` (per widget) → `assemble_dashboard` catches errors early, before any API calls are made.

## Next Steps

- **Export and Modify**: Use `get_dashboard` on an existing dashboard and modify the result
- **Batch Operations**: See `batch-operations.md` for creating multiple dashboards
- **Schema Reference**: See `schema-reference.md` for all widget and visualizer types

## Troubleshooting

### Dashboard Created but Shows "No Data"

1. Check time range (try longer: `24h` instead of `1h`)
2. Verify queries in Metrics explorer
3. Check service tag matches your application
4. Wait 5-10 minutes for data to populate

### Widgets Overlapping

Check grid positions don't overlap:
```yaml
# Widget 1: rows 1-2
row: 1, rowSpan: 2  # Uses rows 1, 2

# Widget 2: rows 3-5 (not 2-4!)
row: 3, rowSpan: 3  # Uses rows 3, 4, 5
```

### Layout Looks Wrong

Increase grid rows in root widget:
```yaml
- id: "root"
  type: "grid"
  grid: "72px 72px 72px 72px 72px 72px 72px 72px / 1fr..."
  # 8 rows instead of 5
```

## Template Dashboards

For ready-to-use starting points:
- Quickstart: `${CLAUDE_PLUGIN_ROOT}/skills/edgedelta-dashboards/assets/quickstart-dashboard.yaml`
- System Metrics: `${CLAUDE_PLUGIN_ROOT}/skills/edgedelta-dashboards/assets/system-metrics.yaml`
- Logs Analysis: `${CLAUDE_PLUGIN_ROOT}/skills/edgedelta-dashboards/assets/logs-analysis.yaml`
