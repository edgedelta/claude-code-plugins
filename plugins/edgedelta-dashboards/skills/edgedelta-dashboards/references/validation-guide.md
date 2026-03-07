# Dashboard Validation Guide

Comprehensive guide to EdgeDelta v4 dashboard validation with error patterns and fixes.

## Validation Overview

The EdgeDelta MCP server performs validation when creating or updating dashboards. The `create_widget` and `assemble_dashboard` tools also validate incrementally during the builder workflow, catching errors before any API calls are made.

### Validation Rules

**Critical Rules (Will Fail)**

1. **Version must be 4**
2. **Must have at least one widget**
3. **Must have exactly one root widget** (`id='root'`, `type='grid'`)
4. **All non-root widgets must have position**
5. **Grid positions must fit in 12-column layout**
6. **Widget/visualizer/data source types must be valid**

**Warning Rules (Logged but Allowed)**

1. Widgets overlapping in grid
2. Empty data source queries
3. Unusual time ranges

## Error Patterns and Fixes

### 1. Missing Root Widget

#### Error Message
```
ERROR: Dashboard MUST have a root widget. Root widget MUST have:
id='root', type='grid', grid='<css-grid-string>'.

Example: {'id': 'root', 'type': 'grid', 'grid': '72px 72px 72px / 1fr 1fr 1fr 1fr'}

FIX: Add a root grid widget as the first widget in the widgets array.
```

#### Root Cause
- No widget with `id='root'`
- Root widget has wrong `type` (not `'grid'`)
- Root widget missing `grid` property

#### Fix
```yaml
# BEFORE (BROKEN)
definition:
  version: 4
  widgets:
    - id: 1
      type: "viz"
      # ... missing root!

# AFTER (FIXED)
definition:
  version: 4
  widgets:
    - id: "root"              # ✓ Correct ID
      type: "grid"            # ✓ Correct type
      grid: "72px 72px 72px / 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr"  # ✓ Grid string

    - id: 1
      type: "viz"
      # ... rest of widgets
```

### 2. Wrong Dashboard Version

#### Error Message
```
ERROR: Input should be 4
```

#### Root Cause
Dashboard version is not 4 (e.g., using v3 schema)

#### Fix
```yaml
# BEFORE (BROKEN)
definition:
  version: 3  # ✗ Wrong version

# AFTER (FIXED)
definition:
  version: 4  # ✓ Correct version
```

### 3. Missing Position Field

#### Error Message
```
ERROR: Widget #1 is missing 'position' field. All non-root widgets MUST have a position.

FIX: Add position: {'type': 'grid', 'targetId': 'root', 'area': {...}} to widget #1.
```

#### Root Cause
Non-root widget doesn't have `position` field

#### Fix
```yaml
# BEFORE (BROKEN)
- id: 1
  type: "viz"
  displayOptions:
    title: "My Widget"
  visualizer:
    type: "bignumber"
  # ... missing position!

# AFTER (FIXED)
- id: 1
  type: "viz"
  position:                    # ✓ Added position
    type: "grid"
    targetId: "root"
    area:
      column: 1
      columnSpan: 6
      row: 1
      rowSpan: 2
  displayOptions:
    title: "My Widget"
  visualizer:
    type: "bignumber"
```

### 4. Grid Boundary Violation

#### Error Message
```
ERROR: Widget extends beyond grid bounds: column 7 + span 8 = 15
(exceeds standard 12-column grid).

FIX: Reduce columnSpan or adjust column position to stay within 12 columns.
```

#### Root Cause
`column + columnSpan > 13` (widget extends past column 12)

#### Fix
```yaml
# BEFORE (BROKEN)
position:
  type: "grid"
  targetId: "root"
  area:
    column: 7       # ✗ 7 + 8 = 15 > 12
    columnSpan: 8   # ✗ Too wide

# AFTER (FIXED) - Option 1: Reduce span
position:
  type: "grid"
  targetId: "root"
  area:
    column: 7       # ✓ 7 + 6 = 13 ≤ 13
    columnSpan: 6   # ✓ Fits in 12 columns

# AFTER (FIXED) - Option 2: Move to start
position:
  type: "grid"
  targetId: "root"
  area:
    column: 1       # ✓ 1 + 8 = 9 ≤ 13
    columnSpan: 8   # ✓ Fits in 12 columns
```

#### Grid Calculation Rule
```
column + columnSpan ≤ 13
```

Examples:
- column: 1, span: 12 → 1 + 12 = 13 ✓ (full width)
- column: 1, span: 6 → 1 + 6 = 7 ✓ (half width)
- column: 7, span: 6 → 7 + 6 = 13 ✓ (second half)
- column: 7, span: 7 → 7 + 7 = 14 ✗ (exceeds)

### 5. Invalid Widget Type

#### Error Message
```
ERROR: Unknown widget type: 'chart'
```

#### Root Cause
Widget type is not one of the 6 valid types

#### Fix
```yaml
# BEFORE (BROKEN)
- id: 1
  type: "chart"  # ✗ Invalid type

# AFTER (FIXED)
- id: 1
  type: "viz"    # ✓ Valid type (for visualizations)

# Valid widget types:
# - viz          (charts, graphs, metrics)
# - markdown     (text content)
# - grid         (layout container - root only)
# - tabs         (tab navigation)
# - variable-control  (dashboard variables)
# - empty        (placeholder)
```

### 6. Invalid Visualizer Type

#### Error Message
```
ERROR: Unknown visualizer type: 'graph'
```

#### Root Cause
Visualizer type not in the list of 24 valid types

#### Fix
```yaml
# BEFORE (BROKEN)
visualizer:
  type: "graph"  # ✗ Invalid

# AFTER (FIXED)
visualizer:
  type: "line"   # ✓ Valid for time series

# Valid visualizer types:
# Single-value: bignumber, gauge
# Multi-value: pie, donut, column, radar, treemap, sunburst, sankey, bubble, list, geomap
# Timeseries: line, bar, area, scatter, step, smooth
# Table: table, raw-table
# Special: json, empty
```

### 7. Invalid Data Source Type

#### Error Message
```
ERROR: Unknown data source type: 'query'
```

#### Root Cause
Data source type not one of 7 valid types

#### Fix
```yaml
# BEFORE (BROKEN)
dataSource:
  type: "query"  # ✗ Invalid

# AFTER (FIXED) - For metrics
dataSource:
  type: "metric"  # ✓ Valid
  params:
    query: "avg:cpu.percent{*}"

# AFTER (FIXED) - For logs
dataSource:
  type: "log"     # ✓ Valid
  params:
    query: "level:ERROR"

# Valid data source types:
# - metric, log, trace, event, pattern, formula, empty
```

### 8. Invalid Result Type

#### Error Message
```
ERROR: Unknown result type: 'aggregated'
```

#### Root Cause
Result type must match visualizer category

#### Fix
```yaml
# BEFORE (BROKEN)
- id: 1
  type: "viz"
  resultType: "aggregated"  # ✗ Invalid

# AFTER (FIXED)
- id: 1
  type: "viz"
  resultType: "aggregate"   # ✓ Valid (no 'd' at end)

# Valid result types:
# - timeseries  (for line, bar, area, scatter, step, smooth)
# - aggregate   (for bignumber, gauge, pie, donut, column, etc.)
# - raw         (for raw-table)
# - empty       (for empty visualizer)
```

### 9. Missing Required Fields

#### Error Message
```
ERROR: Field required [type=missing]
```

#### Root Cause
Required field is missing from the configuration

#### Fix
```yaml
# BEFORE (BROKEN)
- id: 1
  type: "viz"
  # Missing resultType, visualizer, visuals

# AFTER (FIXED)
- id: 1
  type: "viz"
  resultType: "aggregate"       # ✓ Added
  position: {...}               # ✓ Added
  displayOptions:
    title: "Title"
  visualizer:                   # ✓ Added
    type: "bignumber"
  visuals:                      # ✓ Added
    - id: "A"
      dataSource:
        type: "log"
        params:
          query: "*"
```

### 10. Invalid Visual ID

#### Error Message
```
ERROR: Visual ID must be A-F for queries or W-Z for formulas
```

#### Root Cause
Visual ID not in allowed range

#### Fix
```yaml
# BEFORE (BROKEN)
visuals:
  - id: "1"      # ✗ Invalid (must be letter)
    dataSource:
      type: "log"

# AFTER (FIXED)
visuals:
  - id: "A"      # ✓ Valid (A-F for queries)
    dataSource:
      type: "log"

# For formulas
visuals:
  - id: "W"      # ✓ Valid (W-Z for formulas)
    dataSource:
      type: "formula"
      params:
        formula: "A + B"
```

## Validation Workflow

### Option A: Builder Workflow (Recommended)

Use `get_dashboard_schema` → `create_widget` → `assemble_dashboard`. Each step validates before proceeding:

1. **`get_dashboard_schema`** — get valid types and field structure
2. **`create_widget`** — validates each widget in isolation
3. **`assemble_dashboard`** — validates the full layout and creates

Fix errors as `create_widget` reports them, before attempting assembly.

### Option B: Fix Errors Iteratively

If working directly with `create_dashboard` or `update_dashboard`:

1. Read error message carefully
2. Identify the specific widget/field with the issue
3. Apply the fix from this guide
4. Retry

### Common Scenarios

**Migrating from v3 to v4:**
1. Update `version: 3` → `version: 4`
2. Add root grid widget if missing
3. Update position format (v3 used different structure)
4. Verify widget types (some changed in v4)

**Creating from scratch:**
Use the builder workflow. `create_widget` validates one widget at a time, which is easier to debug than a full-dashboard validation error.

**Modifying an existing dashboard:**
1. Use `get_dashboard` to retrieve the current state
2. Modify the definition
3. Use `update_dashboard` with the corrected definition

## Validation Error Debugging

### Identify the Problem Widget

Error messages include widget index:

```
ERROR: Widget #3 is missing 'position' field.
```

Find widget #3 (0-indexed, so 4th widget in array):
```yaml
widgets:
  - id: "root"   # Widget #0
  - id: 1        # Widget #1
  - id: 2        # Widget #2
  - id: 3        # Widget #3 ← This one!
    type: "viz"
    # ... missing position
```

### Check Field Nesting

Errors might be deeply nested:
```
ERROR: widgets[3].position.area.column: Field required
```

Navigate to that field:
```yaml
widgets:
  - # ... root
  - # ... widget 1
  - # ... widget 2
  - # Widget 3
    position:
      area:
        column: ???  # ← Missing this
```

## Validation Best Practices

### 1. Start with Templates

Use known-good templates from the assets directory:
```
${CLAUDE_PLUGIN_ROOT}/skills/edgedelta-dashboards/assets/quickstart-dashboard.yaml
${CLAUDE_PLUGIN_ROOT}/skills/edgedelta-dashboards/assets/system-metrics.yaml
${CLAUDE_PLUGIN_ROOT}/skills/edgedelta-dashboards/assets/logs-analysis.yaml
```

### 2. Validate Early, Validate Often

Use the builder workflow (`create_widget` per widget) rather than writing the full JSON and submitting at once. Errors are easier to fix when isolated to one widget at a time.

### 3. Use Type-Safe Editors

VSCode with YAML extension provides:
- Syntax highlighting
- Indentation validation
- Quick error detection

### 4. Test Queries Separately

Before adding queries to dashboards, test them in EdgeDelta UI:
- Go to Metrics/Logs explore view
- Test query returns data
- Copy exact query to dashboard

### 5. Incremental Development

Build dashboards incrementally:
1. Start with root + 1 widget
2. Validate via `create_widget` and `assemble_dashboard`
3. Use `get_dashboard` to inspect the result
4. Add more widgets using `update_dashboard`
5. Repeat

### 6. Version Control

Store dashboard YAML definitions in git:
```bash
git add dashboards/
git commit -m "Add system metrics dashboard"
```

## Quick Validation Checklist

Before creating/updating a dashboard:

**Structure**
- [ ] `version: 4` is set
- [ ] Root widget exists with correct format
- [ ] All widgets have unique IDs

**Positioning**
- [ ] All non-root widgets have `position`
- [ ] All positions have `type`, `targetId`, `area`
- [ ] All `area` have `column`, `columnSpan`, `row`, `rowSpan`
- [ ] All widgets fit in 12-column grid

**Types**
- [ ] All widget `type` values are valid
- [ ] All `visualizer.type` values are valid
- [ ] All `dataSource.type` values are valid
- [ ] All `resultType` values match visualizers

**Data**
- [ ] All visual IDs are A-F (queries) or W-Z (formulas)
- [ ] All data source queries are valid
- [ ] Time ranges are valid format

**Metadata**
- [ ] `dashboard_name` is set
- [ ] Tags are array of strings

## Common Quick Fixes

| Error | Quick Fix |
|-------|-----------|
| Missing root widget | Add root as first widget |
| Wrong version | Change to `version: 4` |
| Missing position | Add `position` with `type`, `targetId`, `area` |
| Grid overflow | Reduce `columnSpan` or adjust `column` |
| Invalid widget type | Use: viz, markdown, grid, tabs, variable-control, empty |
| Invalid visualizer | Use valid type from 24 options |
| Invalid data source | Use: metric, log, trace, event, pattern, formula, empty |
| Invalid visual ID | Use A-F for queries, W-Z for formulas |
| Missing required field | Add the required field based on error message |
