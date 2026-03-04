# ottl_filter Processor

**Type**: `ottl_filter`
**Category**: Core Transformation & Filtering
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/ottl/filter/processor.go`
**Config Struct**: `configv3.OTTL` (embedded in `configv3.Node`)

## Quick Copy

```yaml
- type: ottl_filter
  condition: 'attributes["level"] == "DEBUG"'
  filter_mode: exclude
```

## Overview

The `ottl_filter` processor uses OTTL (OpenTelemetry Transformation Language) conditions to filter telemetry data, either including or excluding items based on powerful boolean expressions. This is the most flexible and commonly used filtering processor in EdgeDelta, supporting complex conditional logic, attribute comparisons, regex matching, and numeric operations to drop unwanted telemetry before it reaches destinations.

Unlike legacy regex filters, `ottl_filter` provides type-aware comparisons, nested attribute access, and the full power of OTTL functions for sophisticated filtering decisions.

## Use Cases

- **Log Level Filtering**: Drop debug and trace logs in production environments
- **Environment Filtering**: Exclude test, staging, or development data from production pipelines
- **Noise Reduction**: Filter out health checks, heartbeats, and verbose system logs
- **Cost Optimization**: Drop high-volume low-value logs before storage and egress
- **Data Privacy**: Exclude logs containing sensitive operations or endpoints
- **Performance Optimization**: Remove noisy telemetry from chatty services
- **Error Isolation**: Keep only error and warning logs for critical systems
- **Sampling Control**: Drop percentage of traffic based on conditions

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `ottl_filter` |
| `condition` | string | OTTL boolean expression to evaluate for each item |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `filter_mode` | string | `"exclude"` | Filter mode: `"exclude"` (drop matching items) or `"include"` (keep only matching items) |
| `final` | bool | false | Mark as last processor in sequence (required for final processor) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## Filter Modes

### Exclude Mode (Default)

**Behavior**: When `condition` evaluates to `true`, the item is **dropped** (filtered out).

**Use Case**: Remove unwanted data - "drop items that match this condition"

```yaml
filter_mode: exclude
condition: 'attributes["level"] == "DEBUG"'
# Result: DEBUG logs are DROPPED, everything else passes through
```

### Include Mode

**Behavior**: When `condition` evaluates to `true`, the item is **kept** (passes through). All other items are dropped.

**Use Case**: Keep only specific data - "keep only items that match this condition"

```yaml
filter_mode: include
condition: 'attributes["level"] == "ERROR" or attributes["level"] == "FATAL"'
# Result: Only ERROR and FATAL logs are KEPT, everything else is dropped
```

### Mode Selection Guide

| Scenario | Mode | Example Condition |
|----------|------|-------------------|
| Drop debug logs | `exclude` | `attributes["level"] == "DEBUG"` |
| Keep only errors | `include` | `attributes["level"] == "ERROR"` |
| Drop test data | `exclude` | `attributes["environment"] == "test"` |
| Keep production data only | `include` | `attributes["environment"] == "production"` |
| Drop health checks | `exclude` | `IsMatch(body, "/health")` |
| Keep only API calls | `include` | `IsMatch(attributes["path"], "^/api/")` |

## OTTL Conditions

The `condition` field uses OTTL (OpenTelemetry Transformation Language) boolean expressions.

### Common OTTL Functions

| Function | Description | Example |
|----------|-------------|---------|
| `IsMatch(str, "pattern")` | Regex match (case-sensitive) | `IsMatch(body, "ERROR")` |
| `attributes["key"]` | Access attribute value | `attributes["status"] == 200` |
| `resource.attributes["key"]` | Access resource attribute | `resource.attributes["service.name"] == "api"` |
| `body` | Access log body content | `body == "OK"` |
| `severity_text` | Access severity level | `severity_text == "ERROR"` |
| `severity_number` | Access severity number | `severity_number >= 17` |

### Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `==` | Equality | `attributes["code"] == 200` |
| `!=` | Not equal | `attributes["level"] != "DEBUG"` |
| `>` | Greater than | `attributes["duration"] > 1000` |
| `>=` | Greater or equal | `severity_number >= 13` |
| `<` | Less than | `attributes["size"] < 100` |
| `<=` | Less or equal | `attributes["count"] <= 10` |
| `and` | Logical AND | `a == 1 and b == 2` |
| `or` | Logical OR | `level == "ERROR" or level == "FATAL"` |
| `not` | Logical NOT | `not IsMatch(body, "success")` |

**Important**: Use lowercase operators (`and`, `or`, `not`) - uppercase causes errors.

### Nil Checking

```ottl
attributes["field"] != nil          # Field exists
attributes["optional"] == nil       # Field does not exist
```

## Examples

### Example 1: Drop Debug Logs

```yaml
- type: ottl_filter
  condition: 'attributes["level"] == "DEBUG"'
  filter_mode: exclude
```

**What it does**: Drops all logs with level=DEBUG. All other logs pass through.

### Example 2: Keep Only Errors and Warnings

```yaml
- type: ottl_filter
  condition: 'attributes["level"] == "ERROR" or attributes["level"] == "WARN"'
  filter_mode: include
```

**What it does**: Keeps only ERROR and WARN logs. Everything else is dropped.

### Example 3: Drop Health Check Requests

```yaml
- type: ottl_filter
  condition: 'IsMatch(body, "(?i)(health|ping|alive|ready)")'
  filter_mode: exclude
```

**What it does**: Drops logs containing health check keywords (case-insensitive). Reduces noise from monitoring systems.

### Example 4: Filter by HTTP Status Code

```yaml
- type: ottl_filter
  condition: 'attributes["http.status_code"] >= 200 and attributes["http.status_code"] < 300'
  filter_mode: exclude
```

**What it does**: Drops successful HTTP requests (2xx status codes). Keeps errors and redirects for investigation.

### Example 5: Drop Test and Staging Data

```yaml
- type: ottl_filter
  condition: 'attributes["environment"] == "test" or attributes["environment"] == "staging"'
  filter_mode: exclude
```

**What it does**: Drops all logs from test and staging environments. Production logs pass through.

### Example 6: Keep Only Production Errors

```yaml
- type: ottl_filter
  condition: 'attributes["environment"] == "production" and severity_number >= 17'
  filter_mode: include
```

**What it does**: Keeps only production logs with ERROR severity or higher. Everything else is dropped.

**Severity Numbers**:
- INFO: 9
- WARN: 13
- ERROR: 17
- FATAL: 21

### Example 7: Filter by Service Name with Regex

```yaml
- type: ottl_filter
  condition: 'IsMatch(resource.attributes["service.name"], "^(test-|dev-)")'
  filter_mode: exclude
```

**What it does**: Drops logs from services starting with "test-" or "dev-" prefixes.

### Example 8: Complex Multi-Condition Filter

```yaml
- type: ottl_filter
  condition: |
    attributes["level"] == "DEBUG" and
    attributes["environment"] == "production" and
    not IsMatch(attributes["service"], "critical")
  filter_mode: exclude
```

**What it does**: Drops DEBUG logs from production, but keeps DEBUG logs from critical services.

**Note**: Use multiline YAML (`|`) for complex conditions, but ensure the condition itself is logically single-line (no actual line breaks in OTTL expression).

### Example 9: Numeric Threshold Filtering

```yaml
- type: ottl_filter
  condition: 'attributes["duration_ms"] != nil and attributes["duration_ms"] < 100'
  filter_mode: exclude
```

**What it does**: Drops fast requests (under 100ms). Keeps slow requests for performance analysis.

### Example 10: Drop Specific Log Patterns

```yaml
- type: ottl_filter
  condition: 'IsMatch(body, "(?i)(successfully processed|task completed|heartbeat)")'
  filter_mode: exclude
```

**What it does**: Drops routine success messages and heartbeats. Keeps errors and unusual events.

### Example 11: Attribute Existence Check

```yaml
- type: ottl_filter
  condition: 'attributes["user_id"] == nil'
  filter_mode: exclude
```

**What it does**: Drops logs that don't have a user_id attribute. Keeps only user-attributed logs.

### Example 12: Filter Kubernetes System Logs

```yaml
- type: ottl_filter
  condition: 'IsMatch(resource.attributes["k8s.namespace.name"], "^(kube-system|kube-public)$")'
  filter_mode: exclude
```

**What it does**: Drops logs from Kubernetes system namespaces. Reduces infrastructure noise.

## Validation Rules

1. **Required Fields**: Must specify `type` and `condition`
2. **Valid OTTL**: The `condition` must be a valid OTTL boolean expression
3. **Filter Mode**: If specified, `filter_mode` must be either `"include"` or `"exclude"`
4. **Single Condition**: Only one condition is allowed (multiline not supported for multiple separate conditions)
5. **No Comments**: OTTL condition cannot be only comments - must have executable expression
6. **Boolean Result**: Condition must evaluate to boolean (true/false)

## Common Pitfalls

### 1. Wrong Filter Mode

**Problem**: Using `exclude` when you want to keep specific items.

**Wrong**:
```yaml
# Trying to keep only errors, but using exclude mode
filter_mode: exclude
condition: 'attributes["level"] == "ERROR"'
```

**Correct**:
```yaml
# Use include mode to keep only matching items
filter_mode: include
condition: 'attributes["level"] == "ERROR"'
```

### 2. OTTL Syntax Errors

**Problem**: Missing quotes around attribute keys or values.

**Wrong**:
```yaml
condition: 'attributes[level] == ERROR'  # Missing quotes
```

**Correct**:
```yaml
condition: 'attributes["level"] == "ERROR"'
```

### 3. Uppercase Logical Operators

**Problem**: Using uppercase `AND`, `OR`, `NOT`.

**Wrong**:
```yaml
condition: 'attributes["a"] == 1 AND attributes["b"] == 2'
```

**Correct**:
```yaml
condition: 'attributes["a"] == 1 and attributes["b"] == 2'
```

### 4. Multiline Conditions

**Problem**: Attempting to write multiple separate OTTL statements.

**Wrong**:
```yaml
condition: |
  attributes["level"] == "DEBUG"
  attributes["test"] == "true"
```

**Correct** (single compound condition):
```yaml
condition: 'attributes["level"] == "DEBUG" and attributes["test"] == "true"'
```

Or use multiline for readability (but still one expression):
```yaml
condition: |
  attributes["level"] == "DEBUG" and attributes["test"] == "true"
```

### 5. Forgetting Nil Checks

**Problem**: Accessing attributes that might not exist.

**Wrong**:
```yaml
condition: 'attributes["optional_field"] > 100'
# Fails if optional_field doesn't exist
```

**Correct**:
```yaml
condition: 'attributes["optional_field"] != nil and attributes["optional_field"] > 100'
```

### 6. Case Sensitivity in Regex

**Problem**: Not using case-insensitive matching when needed.

**Wrong**:
```yaml
condition: 'IsMatch(body, "error")'  # Misses "ERROR", "Error"
```

**Correct**:
```yaml
condition: 'IsMatch(body, "(?i)error")'  # Case-insensitive
```

### 7. Inverting Logic

**Problem**: Complex negation logic that's hard to understand.

**Wrong**:
```yaml
filter_mode: include
condition: 'not (attributes["level"] == "DEBUG" or attributes["level"] == "TRACE")'
```

**Correct** (clearer with exclude mode):
```yaml
filter_mode: exclude
condition: 'attributes["level"] == "DEBUG" or attributes["level"] == "TRACE"'
```

### 8. Missing Filter Mode

**Problem**: Not specifying `filter_mode` when the default might not match intent.

**Guidance**: Always explicitly set `filter_mode` for clarity:
- Use `exclude` to drop matching items
- Use `include` to keep only matching items

## Best Practices

1. **Explicit Mode**: Always specify `filter_mode` explicitly for clarity and maintainability
2. **Nil Checks**: Always check `!= nil` before accessing potentially missing attributes
3. **Case Insensitive Regex**: Use `(?i)` flag in regex patterns for case-insensitive matching
4. **Simple Conditions**: Prefer simple, readable conditions over complex nested logic
5. **Comment Your Logic**: Use the `comment` field to explain complex filtering decisions
6. **Test Thoroughly**: Validate conditions with sample data before production deployment
7. **Performance First**: Put most selective conditions first in compound expressions
8. **Use Include Sparingly**: `include` mode drops everything that doesn't match - use carefully
9. **Lowercase Operators**: Always use lowercase `and`, `or`, `not` operators
10. **Combine Filters**: Chain multiple `ottl_filter` processors for complex multi-stage filtering
11. **Log Before Filter**: Consider logging stats before filtering to track drop rates
12. **Resource Attributes**: Use `resource.attributes["key"]` for service-level filtering
13. **Severity Numbers**: Use `severity_number` for numeric severity comparisons
14. **Regex Efficiency**: Use specific regex patterns - avoid overly broad patterns like `.*`

## Performance Considerations

- **Condition Complexity**: Simple equality checks are faster than complex regex patterns
- **Early Evaluation**: Use `and` operator to short-circuit - place cheapest checks first
- **Regex Optimization**: Specific patterns (e.g., `^/api/`) are faster than broad patterns (e.g., `.*api.*`)
- **Attribute Access**: Direct attribute access is faster than nested field traversal
- **Filter Placement**: Place filters early in sequence to reduce processing on unwanted data
- **Drop Rate**: High drop rates (>90%) may indicate opportunity to optimize at source

## Production Example (from Template 1)

```yaml
- name: filter_debug_logs
  type: sequence
  user_description: Filter Debug Logs in Production
  processors:
    # Drop debug and trace logs in production
    - type: ottl_filter
      condition: |
        attributes["level"] == "DEBUG" or
        attributes["level"] == "TRACE"
      filter_mode: exclude
      comment: "Reduce noise from verbose logging in production"

    # Keep only production environment
    - type: ottl_filter
      condition: 'attributes["environment"] == "production"'
      filter_mode: include
      comment: "Ensure only production data flows to expensive destinations"

    # Drop health checks
    - type: ottl_filter
      condition: 'IsMatch(body, "(?i)(health|ping|alive)")'
      filter_mode: exclude
      final: true
```

**What it does**:
1. Drops DEBUG and TRACE level logs
2. Keeps only production environment logs
3. Drops health check logs
4. Reduces data volume and costs

## Filter Mode Decision Matrix

| Goal | Mode | Condition Logic |
|------|------|-----------------|
| Drop specific items | `exclude` | Match items to drop |
| Keep specific items | `include` | Match items to keep |
| Drop everything except X | `include` | Match X |
| Keep everything except X | `exclude` | Match X |

## OTTL Functions Reference

### Common Functions for Filtering

| Function | Use Case | Example |
|----------|----------|---------|
| `IsMatch(str, "pattern")` | Regex matching | `IsMatch(body, "(?i)error")` |
| `Contains(str, substring)` | Substring check | `Contains(body, "failed")` |
| `HasPrefix(str, prefix)` | Prefix check | `HasPrefix(attributes["path"], "/api/")` |
| `HasSuffix(str, suffix)` | Suffix check | `HasSuffix(attributes["file"], ".log")` |
| `Len(arr)` | Array length | `Len(attributes["tags"]) > 0` |

### Attribute Paths

```ottl
# Standard attributes
attributes["key"]
attributes["nested"]["field"]
attributes["array"][0]

# Resource attributes
resource.attributes["service.name"]
resource.attributes["host.name"]

# Log-specific fields
body                    # Log body content
severity_text           # Severity as string (INFO, ERROR, etc.)
severity_number         # Severity as number (9, 17, etc.)

# Metric-specific fields (when filtering metrics)
metric.name
metric.type
```

## Related Processors

- **ottl_context_filter**: Buffer-based filtering with context capture around triggers
- **ottl_transform**: Transform data before filtering
- **regex_filter**: Legacy regex-based filtering (use `ottl_filter` instead)
- **tail_sample**: Probabilistic sampling for traces
- **rate_limit**: Rate-based filtering and throttling

## Cross-References

- **edgedelta-pipelines skill**: Used in all templates for noise reduction
- **OTTL Reference**: https://docs.edgedelta.com/ottl-statements/
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`
- **Severity Levels**: https://opentelemetry.io/docs/specs/otel/logs/data-model/#field-severitynumber

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/filter-processor/

## Notes

- The `ottl_filter` processor is one of the most commonly used processors for cost optimization
- Unlike `ottl_transform` which modifies data, `ottl_filter` only drops or passes items
- Filter mode defaults to `exclude` if not specified
- Condition must evaluate to boolean - non-boolean results cause errors
- The processor supports both log and metric filtering with the same OTTL syntax
- Use lowercase OTTL operators (`and`, `or`, `not`) - uppercase causes parse errors
- Comments in YAML (`#`) are not part of the OTTL condition
- For complex filtering logic, consider chaining multiple `ottl_filter` processors
- The processor is optimized for high-throughput filtering with minimal overhead
- Dropped items are counted in telemetry metrics for monitoring drop rates
