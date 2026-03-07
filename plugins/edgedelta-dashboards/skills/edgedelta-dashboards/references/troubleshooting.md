# EdgeDelta Dashboard Troubleshooting Guide

Common issues and solutions when working with EdgeDelta dashboards via MCP.

## Authentication Issues

### Issue: "Unauthorized" or "Not authenticated"

**Symptoms**:
```
Error: 401 Unauthorized
```

**Root Cause**: `ED_API_TOKEN` or `ED_ORG_ID` environment variables are missing or incorrect.

**Solution**: Verify environment variables are set in your shell:
```bash
echo $ED_API_TOKEN   # Should print your token
echo $ED_ORG_ID      # Should print your org ID
```

If missing, set them:
```bash
export ED_API_TOKEN="your-api-token"
export ED_ORG_ID="your-org-id"
```

Then restart Claude Code so the Docker container picks up the updated environment.

### Issue: "403 Forbidden" on Create/Update/Delete

**Symptoms**:
```
Error: 403 Forbidden - You don't have permission to perform this action
```

**Root Cause**: Your API token does not have write permissions for dashboards.

**Solution**:
1. Check your token's permission scope in the EdgeDelta UI under Settings → API Tokens
2. Generate a new token with dashboard write permissions
3. Update `ED_API_TOKEN` and restart Claude Code

**Note**: Dashboard create/update/delete tools require the next MCP server release. If you see 403 errors on write operations, verify that the server version supports dashboard management.

### Issue: "Organization not found"

**Symptoms**:
```
Error: Organization ID 'xyz' not found or you don't have access
```

**Solution**: Verify your organization ID:
1. Log into EdgeDelta UI: https://app.edgedelta.com
2. Check URL: `https://app.edgedelta.com/orgs/<YOUR-ORG-ID>/...`
3. Or go to Settings → Organization → Organization ID
4. Update `ED_ORG_ID` accordingly

## Docker / MCP Setup Issues

### Issue: Docker Not Running

**Symptoms**:
```
Error: Cannot connect to the Docker daemon
```

**Solution**: Start Docker Desktop or the Docker daemon:
```bash
# macOS/Windows: Open Docker Desktop app
# Linux:
sudo systemctl start docker
```

### Issue: Image Pull Fails

**Symptoms**:
```
Error: Unable to pull image ghcr.io/edgedelta/edgedelta-mcp-server:latest
```

**Solution**:
```bash
# Pull manually to see detailed error
docker pull ghcr.io/edgedelta/edgedelta-mcp-server:latest

# Check Docker Hub authentication if needed
docker login ghcr.io
```

### Issue: MCP Server Not Connecting

**Solution**:
1. Verify MCP config references the correct image
2. Check that `ED_API_TOKEN` and `ED_ORG_ID` are exported in your shell environment
3. Run `/mcp` in Claude Code to check server status

## Dashboard Validation Errors

### Issue: "Dashboard MUST have a root widget"

**Symptoms**:
```
ERROR: Dashboard MUST have a root widget. Root widget MUST have:
id='root', type='grid', grid='<css-grid-string>'.
```

**Root Cause**: Missing or incorrectly configured root widget.

**Solution**: Add root widget as first widget in array:
```yaml
widgets:
  - id: "root"
    type: "grid"
    grid: "72px 72px 72px / 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr"

  # ... other widgets follow
```

### Issue: "Widget #N is missing 'position' field"

**Symptoms**:
```
ERROR: Widget #1 is missing 'position' field. All non-root widgets MUST have a position.
```

**Solution**: Add position to the widget:
```yaml
- id: 1
  type: "viz"
  position:
    type: "grid"
    targetId: "root"
    area:
      column: 1
      columnSpan: 6
      row: 1
      rowSpan: 2
  # ... rest of widget config
```

### Issue: "Widget extends beyond grid bounds"

**Symptoms**:
```
ERROR: Widget extends beyond grid bounds: column 7 + span 8 = 15
(exceeds standard 12-column grid).
```

**Root Cause**: Widget position exceeds 12-column layout.

**Solution**: Adjust column or columnSpan:
```yaml
# BAD: 7 + 8 = 15 (exceeds 12)
area:
  column: 7
  columnSpan: 8

# GOOD: 7 + 6 = 13 (fits in 12 columns)
area:
  column: 7
  columnSpan: 6

# OR: 1 + 8 = 9 (fits)
area:
  column: 1
  columnSpan: 8
```

**Formula**: `column + columnSpan ≤ 13`

### Issue: "Input should be 4" (Version Error)

**Symptoms**:
```
ERROR: Input should be 4
```

**Root Cause**: Wrong dashboard version.

**Solution**: Set version to 4:
```yaml
definition:
  version: 4  # Must be 4, not 3 or other
```

### Issue: "Unknown widget type: X"

**Symptoms**:
```
ERROR: Unknown widget type: 'chart'
```

**Root Cause**: Invalid widget type.

**Solution**: Use valid widget type:
```yaml
# Valid types: viz, markdown, grid, tabs, variable-control, empty
type: "viz"  # NOT "chart"
```

### Issue: "Unknown visualizer type: X"

**Symptoms**:
```
ERROR: Unknown visualizer type: 'graph'
```

**Solution**: Use valid visualizer from the list:
- Single-value: `bignumber`, `gauge`
- Multi-value: `pie`, `donut`, `column`, `radar`, `treemap`, `sunburst`, `sankey`, `bubble`, `list`, `geomap`
- Timeseries: `line`, `bar`, `area`, `scatter`, `step`, `smooth`
- Table: `table`, `raw-table`
- Special: `json`, `empty`

```yaml
visualizer:
  type: "line"  # NOT "graph"
```

### Issue: "Unknown data source type: X"

**Symptoms**:
```
ERROR: Unknown data source type: 'query'
```

**Solution**: Use valid data source type:
```yaml
# Valid types: metric, log, trace, event, pattern, formula, empty
dataSource:
  type: "log"  # NOT "query"
```

## MCP Tool Operation Issues

### Issue: Dashboard Not Found

**Symptoms**:
```
Error: Dashboard 'abc-123' not found
```

**Solutions**:
1. List all dashboards to find the correct ID:
   ```
   Use get_all_dashboards tool
   ```

2. Verify `ED_ORG_ID` matches the organization that owns the dashboard

3. Check if the dashboard was deleted

### Issue: YAML Syntax Error

**Symptoms**:
```
Error: yaml.scanner.ScannerError: while scanning...
```

**Root Cause**: Invalid YAML syntax in dashboard definition.

**Solution**: Common YAML issues to check:
- Inconsistent indentation (use 2 spaces, not tabs)
- Missing quotes around special characters (`:`, `{`, `}`, `[`, `]`)
- Incorrect list syntax (each item needs `- ` prefix)

Use `get_dashboard_schema` to get the canonical field structure, or use the `create_widget` → `assemble_dashboard` builder workflow which handles formatting automatically.

### Issue: create_widget Returns Error

**Symptoms**: `create_widget` tool returns a validation error.

**Solution**:
1. Use `get_dashboard_schema` with the relevant category to see valid field values
2. Check that `visualizerType` and `dataSourceType` are compatible
3. Verify all required fields are present

### Issue: assemble_dashboard Partially Fails

**Symptoms**: Some widgets created, but assembly fails.

**Solution**:
1. Review the error — it will indicate which widget config is invalid
2. Fix the specific widget using `create_widget` again
3. Re-run `assemble_dashboard` with the corrected widget list

## API/Network Issues

### Issue: Connection Timeout

**Symptoms**:
```
Error: Request timeout after 30 seconds
```

**Solutions**:
1. Check network connectivity:
   ```bash
   curl -I https://api.edgedelta.com
   ```

2. Check firewall/proxy settings

3. Try again (transient network issue)

### Issue: Rate Limiting

**Symptoms**:
```
Error: 429 Too Many Requests
```

**Solution**: Wait briefly before retrying. The MCP server includes automatic retry with exponential backoff for transient errors.

## Data Issues

### Issue: No Data in Dashboard

**Symptoms**: Dashboard created successfully but shows "No data" in UI.

**Solutions**:
1. Verify data source queries are correct:
   - Test the query in EdgeDelta UI first
   - Check metric/log names are exact matches
   - Verify the time range includes data

2. Check data source type matches data:
   ```yaml
   # For metrics
   dataSource:
     type: "metric"
     params:
       query: "avg:system.cpu.percent{*}"

   # For logs
   dataSource:
     type: "log"
     params:
       query: "level:ERROR"
   ```

3. Verify time range:
   ```yaml
   timeFilters:
     lookback: "1h"  # Try longer: "24h", "7d"
   ```

### Issue: Wrong Visualization Type for Data

**Symptoms**: Dashboard loads but visualization doesn't make sense.

**Solution**: Match visualizer to data and result type:

| Data Type | Result Type | Recommended Visualizers |
|-----------|-------------|------------------------|
| Time series metrics | `timeseries` | `line`, `area`, `bar` |
| Single metric value | `aggregate` | `bignumber`, `gauge` |
| Grouped metrics | `aggregate` | `pie`, `donut`, `column` |
| Raw logs | `raw` | `raw-table` |
| Aggregated logs | `aggregate` | `table`, `list` |

## Debugging Tips

### Inspect an Existing Dashboard

Use `get_dashboard` with a known-working dashboard ID to see the exact JSON structure you should replicate:
```
get_dashboard(id="<working-dashboard-id>")
```

### Validate Schema First

Before building a complex dashboard, query the schema to see valid options:
```
get_dashboard_schema(category="widget-types")
get_dashboard_schema(category="visualizer-types")
get_dashboard_schema(category="data-sources")
```

### Use the Builder Workflow

Rather than writing raw JSON, use `create_widget` for each widget first — it validates each piece individually before you attempt assembly:
```
1. create_widget(type="viz", visualizerType="line", ...) → widget config
2. create_widget(type="viz", visualizerType="bignumber", ...) → widget config
3. assemble_dashboard(name="My Dashboard", widgets=[...])
```

### Dashboard Updated But Changes Not Visible

**Solutions**:
1. Hard refresh browser: Ctrl+Shift+R (or Cmd+Shift+R on Mac)
2. Wait 10-30 seconds for cache to clear
3. Use `get_dashboard` to verify the update was persisted
4. Check that `update_dashboard` returned success

## Getting Help

### Reference Files
- Schema Reference: `${CLAUDE_PLUGIN_ROOT}/skills/edgedelta-dashboards/references/schema-reference.md`
- Validation Guide: `${CLAUDE_PLUGIN_ROOT}/skills/edgedelta-dashboards/references/validation-guide.md`
- MCP Server Source: https://github.com/edgedelta/edgedelta-mcp-server

### Template Dashboards
Use known-good templates from the assets directory:
```
${CLAUDE_PLUGIN_ROOT}/skills/edgedelta-dashboards/assets/quickstart-dashboard.yaml
${CLAUDE_PLUGIN_ROOT}/skills/edgedelta-dashboards/assets/system-metrics.yaml
${CLAUDE_PLUGIN_ROOT}/skills/edgedelta-dashboards/assets/logs-analysis.yaml
```

### EdgeDelta Support
- Documentation: https://docs.edgedelta.com
- Support Portal: Contact your EdgeDelta representative

## Quick Fixes Summary

| Issue | Quick Fix |
|-------|----------|
| 401 Unauthorized | Verify `ED_API_TOKEN` is exported in shell |
| 403 Forbidden | Check token has dashboard write permissions |
| Organization not found | Verify `ED_ORG_ID` is correct |
| Docker not running | Start Docker Desktop |
| Image pull fails | `docker pull ghcr.io/edgedelta/edgedelta-mcp-server:latest` |
| Missing root widget | Add root widget as first widget |
| Missing position | Add `position` field to widget |
| Grid bounds exceeded | Adjust column/columnSpan to fit 12 columns |
| Wrong version | Set `version: 4` |
| Invalid type | Use valid widget/visualizer/dataSource type |
| YAML syntax error | Validate YAML, check indentation |
| No data in dashboard | Verify query, time range, data source type |
