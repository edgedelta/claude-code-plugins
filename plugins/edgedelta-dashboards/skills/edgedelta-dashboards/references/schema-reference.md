# EdgeDelta Dashboard v4 Schema Reference

Quick reference guide for EdgeDelta v4 dashboard schema structure.

## Dashboard Structure

```yaml
dashboard_name: "string"
description: "string (optional)"
tags: ["array", "of", "strings"]
definition:
  version: 4  # REQUIRED - must be 4
  timeFilters:
    lookback: "1h"  # 15m, 30m, 1h, 3h, 6h, 12h, 24h, 7d, 30d
  widgets: [...]  # Array of widget objects
```

## Widget Types (6 Total)

### 1. Grid Widget (Root - REQUIRED)
Every dashboard MUST have exactly ONE root grid widget:

```yaml
- id: "root"
  type: "grid"
  grid: "72px 72px 72px / 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr"
```

**Grid Format**: `"<rows> / <columns>"`
- Standard: 12 columns, 72px per row
- Rows: Space-separated row heights (e.g., `72px 72px 72px` = 3 rows)
- Columns: Space-separated column widths (typically 12x `1fr` for equal columns)

### 2. Viz Widget (Visualizations)
Data visualizations (charts, graphs, metrics):

```yaml
- id: 1  # Unique integer or string
  type: "viz"
  resultType: "aggregate"  # or "timeseries", "raw", "empty"
  position: {...}  # REQUIRED
  displayOptions:
    title: "Widget Title"
  visualizer:
    type: "bignumber"  # See visualizer types below
  visuals: [...]  # Data sources
```

### 3. Markdown Widget
Rich text/HTML content:

```yaml
- id: 2
  type: "markdown"
  position: {...}
  displayOptions:
    title: "Title"
  text: "# Markdown content here"
```

### 4. Tabs Widget
Tab navigation container:

```yaml
- id: 3
  type: "tabs"
  position: {...}
  tabs:
    - label: "Tab 1"
      targetId: "widget-id-1"
    - label: "Tab 2"
      targetId: "widget-id-2"
```

### 5. Variable Control Widget
Dashboard variable controls:

```yaml
- id: 4
  type: "variable-control"
  position: {...}
  variableName: "my_variable"
```

### 6. Empty Widget
Placeholder/spacer:

```yaml
- id: 5
  type: "empty"
  position: {...}
```

## Widget Positioning (Grid System)

**ALL non-root widgets MUST have a position field:**

```yaml
position:
  type: "grid"
  targetId: "root"  # Parent container (usually "root")
  area:
    column: 1        # Starting column (1-12, 1-indexed)
    columnSpan: 6    # Width in columns (1-12)
    row: 1           # Starting row (1+, 1-indexed)
    rowSpan: 2       # Height in rows (1+)
```

### Grid Rules
- **Column Range**: 1-12 (1-indexed)
- **Column Constraint**: `column + columnSpan ≤ 13`
- **Row Range**: 1+ (1-indexed, no upper limit)
- **No Overlaps**: Widgets should not overlap (not enforced, but recommended)

### Common Widget Sizes
- **BigNumber/Gauge**: 2-4 columns × 2 rows
- **Small Chart**: 6 columns × 3-4 rows
- **Large Chart**: 12 columns × 4-6 rows
- **Table**: 12 columns × 4+ rows

## Visualizer Types (24 Total)

### Scalar Visualizers (resultType: aggregate)
- `bignumber` - Large single metric display
- `gauge` - Gauge chart (0-100 or custom range)

### Aggregate Visualizers (resultType: aggregate)
- `pie` - Pie chart
- `donut` - Donut chart
- `column` - Vertical bar chart
- `radar` - Radar/spider chart
- `treemap` - Hierarchical treemap
- `sunburst` - Hierarchical sunburst
- `sankey` - Flow diagram
- `bubble` - Bubble chart
- `list` - List view
- `geomap` - Geographic map

### Timeseries Visualizers (resultType: timeseries)
- `line` - Line chart
- `bar` - Bar chart (time-based)
- `area` - Area chart
- `scatter` - Scatter plot
- `step` - Step chart
- `smooth` - Smoothed line chart

### Table Visualizers
- `table` - Formatted table with aggregation
- `raw-table` - Raw data table

### Special Visualizers
- `json` - JSON viewer
- `empty` - Empty placeholder

## Data Sources

### Visual Structure
```yaml
visuals:
  - id: "A"  # A-F for queries, W-Z for formulas
    dataSource:
      type: "log"  # See data source types below
      params: {...}
      timeFiltersOffset: "1h"  # Optional time shift
```

### Data Source Types (7 Total)

#### 1. Metric
Query metric data:

```yaml
dataSource:
  type: "metric"
  params:
    query: "avg:system.cpu.percent{host:web-*}"
```

**Query Format**: `<aggregation>:<metric.name>{<tags>}`
- Aggregations: `avg`, `sum`, `min`, `max`, `count`, `p50`, `p90`, `p95`, `p99`
- Example: `avg:cpu.utilization{service:api,env:prod}`

#### 2. Log
Query log data:

```yaml
dataSource:
  type: "log"
  params:
    query: "level:ERROR service:api"
```

**Query Format**: Filter expressions
- Examples: `*` (all), `level:ERROR`, `service:api AND status:500`

#### 3. Trace
Query trace/APM data:

```yaml
dataSource:
  type: "trace"
  params:
    query: "service:api operation:http.request"
```

#### 4. Event
Query event data:

```yaml
dataSource:
  type: "event"
  params:
    query: "event_type:deployment"
```

#### 5. Pattern
Log pattern analysis:

```yaml
dataSource:
  type: "pattern"
  params:
    negative: true
    includeOther: true
```

#### 6. Formula
Mathematical formulas combining other visuals:

```yaml
- id: "W"  # Formulas use W-Z
  dataSource:
    type: "formula"
    params:
      formula: "(A + B) / C * 100"
```

**Formula Rules**:
- Can reference other visuals: A, B, C, D, E, F
- Supports: `+`, `-`, `*`, `/`, `()`, numbers
- Example: `(A - B) / A * 100` (percentage change)

#### 7. Empty
Placeholder:

```yaml
dataSource:
  type: "empty"
```

## Result Types

Match visualizer category with result type:

| Visualizer Category | Result Type |
|-------------------|-------------|
| Scalar (bignumber, gauge) | `aggregate` |
| Aggregate (pie, donut, column, etc.) | `aggregate` |
| Timeseries (line, bar, area, etc.) | `timeseries` |
| Table (table, raw-table) | `aggregate` or `raw` |
| Special (json, empty) | `raw` or `empty` |

## Display Options

Common display options for viz widgets:

```yaml
displayOptions:
  title: "Widget Title"
  description: "Optional description"

  # Y-axis configuration (for charts)
  yAxis:
    label: "Y-axis Label"
    min: 0
    max: 100

  # Legend
  legend:
    position: "bottom"  # top, bottom, left, right, none
    alignment: "center"  # start, center, end

  # Colors
  colorPalette: "default"  # or custom palette name

  # Coloring object (preferred for CSS variables)
  coloring:
    mode: "palette"  # or "continuous"
    palette:
      - "var(--extended-green-500)"
      - "var(--extended-orange-500)"

  # Thresholds (for gauge, bignumber)
  thresholds:
    - value: 0
      color: "#00ff00"
    - value: 75
      color: "#ffaa00"
    - value: 90
      color: "#ff0000"
```

## Color System (CSS Variables)

EdgeDelta dashboards support CSS custom properties for consistent theming.

### Extended Colors (weights 200-800)

| Color | Example | Use Case |
|-------|---------|----------|
| `azure` | `var(--extended-azure-500)` | Info, neutral |
| `rose` | `var(--extended-rose-500)` | Highlights |
| `gold` | `var(--extended-gold-500)` | Warnings, caution |
| `green` | `var(--extended-green-500)` | Success, healthy |
| `orange` | `var(--extended-orange-500)` | Active, attention |
| `violet` | `var(--extended-violet-500)` | Special, unique |
| `scarlet` | `var(--extended-scarlet-500)` | Errors, alerts |
| `gray` | `var(--extended-gray-500)` | Muted, inactive |
| `cyan` | `var(--extended-cyan-500)` | Info, recovery |

**⚠️ Common Mistake**: `extended-red` is NOT valid. Use `extended-scarlet` instead.

### Semantic Accent Colors

```yaml
palette:
  - "var(--semantic_accent-blue)"   # Information
  - "var(--semantic_accent-green)"  # Success
  - "var(--semantic_accent-yellow)" # Warning
  - "var(--semantic_accent-red)"    # Error/Critical
```

### Coloring Overrides (per-series)

```yaml
coloringOverrides:
  - name: "alert"
    coloring:
      mode: "palette"
      palette: ["var(--extended-scarlet-400)"]
  - name: "recovery"
    coloring:
      mode: "palette"
      palette: ["var(--extended-green-400)"]
```

## Time Filters

Dashboard-level time filter:

```yaml
timeFilters:
  lookback: "1h"  # How far back to query
```

**Valid Lookback Values**:
- Minutes: `15m`, `30m`
- Hours: `1h`, `3h`, `6h`, `12h`, `24h`
- Days: `7d`, `30d`

**Per-Visual Time Offset**:
```yaml
dataSource:
  type: "metric"
  params: {...}
  timeFiltersOffset: "-1h"  # Compare to 1 hour ago
```

## Dashboard Variables

Define variables for dynamic filtering:

```yaml
definition:
  variables:
    - name: "environment"
      type: "custom"
      options:
        - label: "Production"
          value: "prod"
        - label: "Staging"
          value: "staging"
      defaultValue: "prod"
```

**Variable Types**:
- `custom` - Static list of options
- `query` - Dynamic from query results
- `text` - Free-form text input
- `interval` - Time interval selector
- `datasource` - Data source selector
- `constant` - Hidden constant value

**Using Variables in Queries**:
```yaml
params:
  query: "avg:cpu.percent{env:$environment}"
```

## Complete Example

```yaml
dashboard_name: "Complete Example"
description: "Demonstrates all major features"
tags: ["example", "complete"]

definition:
  version: 4

  timeFilters:
    lookback: "1h"

  variables:
    - name: "service"
      type: "custom"
      options:
        - {label: "API", value: "api"}
        - {label: "Web", value: "web"}
      defaultValue: "api"

  widgets:
    # Root grid (REQUIRED)
    - id: "root"
      type: "grid"
      grid: "72px 72px 72px 72px / 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr"

    # BigNumber metric
    - id: 1
      type: "viz"
      resultType: "aggregate"
      position:
        type: "grid"
        targetId: "root"
        area: {column: 1, columnSpan: 3, row: 1, rowSpan: 2}
      displayOptions:
        title: "CPU Average"
      visualizer:
        type: "bignumber"
      visuals:
        - id: "A"
          dataSource:
            type: "metric"
            params:
              query: "avg:cpu.percent{service:$service}"

    # Line chart
    - id: 2
      type: "viz"
      resultType: "timeseries"
      position:
        type: "grid"
        targetId: "root"
        area: {column: 4, columnSpan: 9, row: 1, rowSpan: 4}
      displayOptions:
        title: "CPU Over Time"
        yAxis:
          label: "Percent"
          min: 0
          max: 100
      visualizer:
        type: "line"
      visuals:
        - id: "A"
          dataSource:
            type: "metric"
            params:
              query: "avg:cpu.percent{service:$service}"

    # Table
    - id: 3
      type: "viz"
      resultType: "raw"
      position:
        type: "grid"
        targetId: "root"
        area: {column: 1, columnSpan: 12, row: 5, rowSpan: 4}
      displayOptions:
        title: "Recent Logs"
      visualizer:
        type: "raw-table"
      visuals:
        - id: "A"
          dataSource:
            type: "log"
            params:
              query: "service:$service"
```

## Validation Checklist

Before creating/updating a dashboard:

- [ ] Version is 4
- [ ] Root widget exists with `id: "root"`, `type: "grid"`, `grid: "..."`
- [ ] All non-root widgets have `position` field
- [ ] All grid positions fit within 12 columns
- [ ] All visualizers are valid types
- [ ] All data source types are valid
- [ ] Result types match visualizer categories
- [ ] Variable references use `$variable_name` syntax
- [ ] Time lookback values are valid
- [ ] Visual IDs are A-F (queries) or W-Z (formulas)

## Common Patterns

### Side-by-Side Metrics
```yaml
# Left metric: columns 1-6
area: {column: 1, columnSpan: 6, row: 1, rowSpan: 2}

# Right metric: columns 7-12
area: {column: 7, columnSpan: 6, row: 1, rowSpan: 2}
```

### Full-Width Chart
```yaml
area: {column: 1, columnSpan: 12, row: 1, rowSpan: 4}
```

### Three-Column Layout
```yaml
# Left: columns 1-4
area: {column: 1, columnSpan: 4, row: 1, rowSpan: 3}

# Center: columns 5-8
area: {column: 5, columnSpan: 4, row: 1, rowSpan: 3}

# Right: columns 9-12
area: {column: 9, columnSpan: 4, row: 1, rowSpan: 3}
```

### Stacked Rows
```yaml
# Row 1
area: {column: 1, columnSpan: 12, row: 1, rowSpan: 2}

# Row 2
area: {column: 1, columnSpan: 12, row: 3, rowSpan: 3}

# Row 3
area: {column: 1, columnSpan: 12, row: 6, rowSpan: 4}
```

## Quick Reference URLs

- EdgeDelta Dashboard UI: `https://app.edgedelta.com/dashboards`
- EdgeDelta Docs: `https://docs.edgedelta.com`
- MCP Server: `https://github.com/edgedelta/edgedelta-mcp-server`
