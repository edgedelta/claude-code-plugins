# Batch Operations Guide

Guide to managing multiple dashboards efficiently using MCP tools.

## Use Cases

Batch operations are useful for:
- **Backup/Recovery**: Export all dashboards for disaster recovery
- **Migration**: Move dashboards between organizations or environments
- **Multi-Tenant**: Deploy standard dashboard sets to multiple organizations
- **Bulk Updates**: Apply changes to multiple dashboards simultaneously
- **Onboarding**: Provision standard dashboards for new teams/services

## Batch Export: Backup All Dashboards

### Scenario
Export all dashboards from an organization for backup or migration.

### Workflow

1. List all dashboards:
   ```
   get_all_dashboards()
   ```
   Returns a list of all dashboard IDs and names.

2. For each dashboard ID, retrieve the full definition:
   ```
   get_dashboard(id="dash_abc123")
   get_dashboard(id="dash_def456")
   get_dashboard(id="dash_ghi789")
   ```

3. Save each definition as a YAML file in a `dashboard-backups/` directory.

4. Commit to version control:
   ```bash
   git add dashboard-backups/
   git commit -m "Dashboard backup $(date +%Y-%m-%d)"
   ```

### Automated Approach

For large organizations, ask Claude to iterate through all dashboards:

> "Export all my EdgeDelta dashboards and save them as YAML files in a dashboard-backups directory, one file per dashboard named by dashboard ID."

Claude will use `get_all_dashboards` then `get_dashboard` for each, then write the files.

## Batch Create: Deploy Standard Dashboards

### Scenario
Deploy a standard set of dashboards to a new organization or service.

### Prepare Templates

Create a directory of YAML templates:

```
dashboard-templates/
├── system-metrics.yaml
├── application-performance.yaml
├── error-tracking.yaml
└── user-analytics.yaml
```

### Workflow

1. For each YAML file, read it and create the dashboard:
   ```
   create_dashboard(definition=<contents-of-system-metrics.yaml>)
   create_dashboard(definition=<contents-of-application-performance.yaml>)
   ...
   ```

2. Or use the builder workflow for new dashboards:
   ```
   assemble_dashboard(name="System Metrics", widgets=[...])
   assemble_dashboard(name="Application Performance", widgets=[...])
   ```

### Multi-Organization Deployment

To deploy the same dashboards to multiple organizations, set `ED_ORG_ID` to each target org and repeat the create workflow. Since `ED_ORG_ID` is an environment variable, you can run Claude Code sessions with different org IDs set.

### Service-Specific Dashboard Sets

Organize templates by service:

```
dashboard-templates/
├── web-service/
│   ├── web-overview.yaml
│   ├── web-errors.yaml
│   └── web-performance.yaml
├── api-service/
│   ├── api-overview.yaml
│   ├── api-latency.yaml
│   └── api-errors.yaml
└── database-service/
    ├── db-overview.yaml
    ├── db-queries.yaml
    └── db-connections.yaml
```

Create a specific service set by passing those files to `create_dashboard`.

## Batch Update: Apply Changes to Multiple Dashboards

### Scenario
Update multiple dashboards with a common change (e.g., time range, new widget).

### Workflow

1. Export all dashboards using `get_all_dashboards` + `get_dashboard`.

2. Apply the change to the definitions (e.g., update `lookback`, add a widget, change a query).

3. For each modified dashboard, update it:
   ```
   update_dashboard(id="dash_abc123", definition=<modified-definition>)
   update_dashboard(id="dash_def456", definition=<modified-definition>)
   ...
   ```

### Example: Change Time Range Across All Dashboards

Ask Claude:

> "Change the default time range to 3h on all my dashboards."

Claude will:
1. List all dashboards
2. Retrieve each definition
3. Update `timeFilters.lookback` from `"1h"` to `"3h"` in each
4. Call `update_dashboard` for each

### Example: Add Standard Widget to All Dashboards

Ask Claude:

> "Add an Agent Health bignumber widget in the top-right corner of all my dashboards."

Claude will retrieve each dashboard definition, add the widget, adjust positioning, and call `update_dashboard` for each.

Example widget to add:
```yaml
- id: 999
  type: "viz"
  resultType: "aggregate"
  position:
    type: "grid"
    targetId: "root"
    area:
      column: 10
      columnSpan: 3
      row: 1
      rowSpan: 2
  displayOptions:
    title: "Agent Health"
  visualizer:
    type: "bignumber"
  visuals:
    - id: "A"
      dataSource:
        type: "metric"
        params:
          query: "count:edgedelta.agent.healthy{*}"
```

## Batch Delete (with Caution!)

### Scenario
Clean up test/temporary dashboards in bulk.

**WARNING**: This is destructive! Always:
1. Export backups first using `get_all_dashboards` + `get_dashboard`
2. Double-check the list of IDs to delete
3. Review before proceeding

### Workflow

1. List all dashboards and identify the ones to delete:
   ```
   get_all_dashboards()
   ```

2. Confirm the exact IDs to be deleted with the user.

3. Delete each one:
   ```
   delete_dashboard(id="dash_test001")
   delete_dashboard(id="dash_test002")
   ```

Always confirm with the user before batch deletion. The `delete_dashboard` operation is permanent.

## Real-World Workflows

### Workflow 1: Multi-Environment Promotion

Promote dashboards from staging to production:

1. Export all dashboards from staging (set `ED_ORG_ID` to staging org):
   ```
   get_all_dashboards()
   get_dashboard(id="<each-staging-id>")
   ```

2. Review and optionally modify definitions for production (update tags, environment labels, queries).

3. Set `ED_ORG_ID` to production org and restart Claude Code.

4. Create dashboards in production:
   ```
   create_dashboard(definition=<staging-definition-adjusted-for-prod>)
   ```

### Workflow 2: Disaster Recovery

Restore dashboards from saved YAML files:

1. Load the YAML files from your backup directory.
2. For each file:
   ```
   create_dashboard(definition=<backup-definition>)
   ```

### Workflow 3: Team Onboarding

Provision standard dashboards for a new team:

1. Load your team-standard YAML templates.
2. Create each dashboard, optionally updating the `dashboard_name` to include the team prefix:
   ```
   create_dashboard(definition=<template-with-team-prefix>)
   ```

### Workflow 4: Dashboard Migration

Migrate dashboards to a new organization:

1. Export from the source org.
2. Update `ED_ORG_ID` to the target org.
3. Create each dashboard in the target org.

## Best Practices

### 1. Always Backup Before Batch Operations

Before bulk updates or deletes, export all current dashboards first. Save them as YAML files in a versioned directory.

### 2. Use Version Control

Store dashboard YAML definitions in git:
```bash
git add dashboard-templates/
git commit -m "Update dashboard templates for Q4"
```

Deploy from a known commit state for reproducibility.

### 3. Test in Non-Production First

Deploy to a staging org first, verify in the EdgeDelta UI, then promote to production.

### 4. Use Descriptive Naming

```yaml
# Include environment/purpose in dashboard name
dashboard_name: "[Production] System Metrics - Web Service"
tags:
  - production
  - web-service
  - system
```

### 5. Document Batch Operations

Keep a deployment log:
```bash
echo "$(date): Deployed dashboards to $ED_ORG_ID" >> deployment-log.txt
git commit -m "Deploy dashboards $(date +%Y-%m-%d)"
```

### 6. Handle Failures Gracefully

If a batch create partially fails:
1. Note which IDs succeeded from the output
2. Fix the failing definitions using the error messages as guidance
3. Retry only the failed ones

## Troubleshooting

### Some Dashboards Fail to Create

Check the validation error returned by `create_dashboard`. Use `get_dashboard_schema` to verify valid field values, or use the `create_widget` → `assemble_dashboard` builder workflow to catch errors per-widget.

### Rate Limiting

If you hit rate limits (`429 Too Many Requests`), wait a few seconds between operations. Ask Claude to pause between calls if you're doing a large batch.

### Duplicate Names

Before creating, use `get_all_dashboards` to check existing dashboard names. Avoid creating duplicates by checking the returned list.
