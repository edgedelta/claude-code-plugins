# ottl_context_filter Processor

**Type**: `ottl_context_filter`
**Category**: Core Transformation & Filtering
**Sequence Compatible**: ✓ Yes (Buffer Processor)
**Source**: `internalv3/processors/ottlcontextfilter/processor.go`
**Config Struct**: `configv3.OTTLContextFilter`

## Quick Copy

```yaml
- type: ottl_context_filter
  ottl_context_filter:
    condition: 'severity_text == "error"'
    ottl_context_filter_context:
      condition: 'severity_text == "info" or severity_text == "debug"'
      length: 5
      interval: 1s
```

## Overview

The `ottl_context_filter` processor captures log context around trigger events using OTTL conditions. When a trigger condition matches (e.g., an error), it collects surrounding log items that match a context condition, creating a complete picture of what led to and followed the trigger event. All collected items are tagged with a unique context ID and flushed together.

This processor is invaluable for debugging production issues, root cause analysis, and understanding error context without shipping all logs.

## Use Cases

- **Error Debugging**: Capture surrounding logs (INFO, DEBUG) when errors occur
- **Incident Investigation**: Collect complete context around critical events or exceptions
- **Root Cause Analysis**: See what happened before and after failures
- **Performance Investigation**: Capture context around slow requests or timeouts
- **Security Analysis**: Collect logs around authentication failures or suspicious events
- **Cost Optimization**: Ship only relevant logs with context instead of all logs
- **Alert Enrichment**: Include surrounding context when sending alerts
- **Pattern Recognition**: Identify log sequences that precede specific events

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `ottl_context_filter` |
| `ottl_context_filter.condition` | string | OTTL condition for trigger events (e.g., error detection) |
| `ottl_context_filter.ottl_context_filter_context.length` | int | Number of context items to collect after trigger (1-500) |

### Context Configuration Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `condition` | string | No | OTTL condition for context items. If not specified, all items are collected |
| `length` | int | Yes | Number of context items to collect after trigger (max: 500) |
| `interval` | duration | No | Time window for context collection (max: 10s). Flushes when interval expires |

### Optional Parameters (from SequenceProcessor)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `final` | bool | false | Mark as last processor in sequence (required for final processor) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## How It Works

### Circular Buffer

The processor maintains a circular buffer that stores items matching the context condition:
- **Before Trigger**: Buffers items that match the context condition (or all items if no context condition)
- **Trigger Event**: When the trigger condition matches, buffer contents are moved to the active context
- **After Trigger**: Collects `length` additional context items or until `interval` expires
- **Flush**: All items (buffer + trigger + context) are flushed with the same context ID

### Context ID

Every item in a context group receives a unique context ID in the `ed.context.id` attribute. This allows:
- Correlating related logs across distributed systems
- Filtering context groups in destinations
- Debugging with complete event sequences

### Flush Triggers

A context is flushed when:
1. **Length Reached**: `length` context items collected after trigger
2. **Interval Expired**: `interval` duration passed since context started
3. **Data Size Limit**: Context size exceeds 10 MB (safety limit)
4. **Processor Stop**: Graceful shutdown flushes active contexts

### Extension Logic

If a trigger condition matches while a context is active:
- The context window is **extended** (not reset)
- The new trigger is added to the existing context
- The counter resets to collect `length` more items
- All items share the same context ID

## Examples

### Example 1: Basic Error Context

```yaml
- type: ottl_context_filter
  ottl_context_filter:
    condition: 'severity_text == "error"'
    ottl_context_filter_context:
      condition: 'severity_text == "info" or severity_text == "debug"'
      length: 10
      interval: 2s
```

**What it does**:
- Buffers INFO and DEBUG logs
- When an ERROR occurs, collects the buffered logs plus 10 more INFO/DEBUG logs
- Flushes the complete context (buffer + error + 10 context items)

### Example 2: Critical Event Context

```yaml
- type: ottl_context_filter
  ottl_context_filter:
    condition: 'severity_text == "critical" or severity_text == "fatal"'
    ottl_context_filter_context:
      condition: 'severity_text != "trace"'
      length: 20
      interval: 5s
```

**What it does**: Captures 20 logs (excluding TRACE) around CRITICAL/FATAL events within a 5-second window.

### Example 3: HTTP Error Context

```yaml
- type: ottl_context_filter
  ottl_context_filter:
    condition: 'attributes["http.status_code"] >= 500'
    ottl_context_filter_context:
      condition: 'IsMatch(body, "(?i)(request|response|http)")'
      length: 15
      interval: 3s
```

**What it does**: Captures 15 HTTP-related logs around 5xx server errors.

### Example 4: Capture All Context (No Context Condition)

```yaml
- type: ottl_context_filter
  ottl_context_filter:
    condition: 'severity_text == "error"'
    ottl_context_filter_context:
      length: 10
      interval: 1s
```

**What it does**:
- When `context.condition` is omitted, **all logs** are buffered and collected as context
- Captures the 10 logs immediately before the error (from buffer)
- Captures the 10 logs immediately after the error
- Total: up to 21 logs (10 before + error + 10 after)

### Example 5: Service-Specific Error Context

```yaml
- type: ottl_context_filter
  condition: 'attributes["service.name"] == "payment-service"'
  ottl_context_filter:
    condition: 'severity_text == "error"'
    ottl_context_filter_context:
      condition: 'severity_text != "trace"'
      length: 8
      interval: 2s
```

**What it does**: Only processes logs from `payment-service`, capturing 8 non-TRACE logs around errors.

### Example 6: Authentication Failure Context

```yaml
- type: ottl_context_filter
  ottl_context_filter:
    condition: 'IsMatch(body, "(?i)(authentication failed|unauthorized|forbidden)")'
    ottl_context_filter_context:
      condition: 'IsMatch(body, "(?i)(auth|login|token|user)")'
      length: 12
      interval: 3s
```

**What it does**: Captures 12 authentication-related logs around auth failures.

### Example 7: Database Connection Error Context

```yaml
- type: ottl_context_filter
  ottl_context_filter:
    condition: 'IsMatch(body, "(?i)(connection.*timeout|database.*unavailable|pool.*exhausted)")'
    ottl_context_filter_context:
      condition: 'attributes["component"] == "database" or IsMatch(body, "(?i)(query|connection)")'
      length: 20
      interval: 5s
```

**What it does**: Captures 20 database-related logs around connection errors.

### Example 8: Performance Degradation Context

```yaml
- type: ottl_context_filter
  ottl_context_filter:
    condition: 'attributes["duration_ms"] > 5000'
    ottl_context_filter_context:
      condition: 'attributes["duration_ms"] != nil'
      length: 10
      interval: 2s
```

**What it does**: Captures 10 logs with duration metrics around requests slower than 5 seconds.

### Example 9: Extended Context with Multiple Triggers

```yaml
- type: ottl_context_filter
  ottl_context_filter:
    condition: 'severity_text == "error" or severity_text == "warning"'
    ottl_context_filter_context:
      condition: 'severity_text == "info"'
      length: 5
      interval: 3s
```

**What it does**:
- If an ERROR occurs, starts collecting context
- If a WARNING occurs during collection, extends the context (same context ID)
- Continues until 5 INFO logs collected after the last trigger

### Example 10: Interval-Based Flush

```yaml
- type: ottl_context_filter
  ottl_context_filter:
    condition: 'severity_text == "error"'
    ottl_context_filter_context:
      condition: 'severity_text != "trace"'
      length: 100
      interval: 2s
```

**What it does**:
- Attempts to collect 100 context items
- But flushes after 2 seconds even if fewer items collected
- Useful for low-traffic scenarios where length might not be reached

### Example 11: Exception Stack Trace Context

```yaml
- type: ottl_context_filter
  ottl_context_filter:
    condition: 'IsMatch(body, "(?i)(exception|stacktrace|throw)")'
    ottl_context_filter_context:
      condition: 'severity_text == "info" or severity_text == "debug" or severity_text == "error"'
      length: 25
      interval: 5s
```

**What it does**: Captures 25 logs around exceptions to get complete stack traces and error context.

## Validation Rules

1. **Required Fields**: Must specify `type`, `ottl_context_filter.condition`, and `ottl_context_filter.ottl_context_filter_context.length`
2. **Trigger Condition**: `ottl_context_filter.condition` cannot be empty and must be valid OTTL
3. **Length Range**: `length` must be greater than 0 and maximum 500
4. **Interval Limit**: `interval` cannot exceed 10 seconds (safety limit to prevent memory buildup)
5. **Valid OTTL**: Both `condition` and `context.condition` must be valid OTTL expressions
6. **Context Condition Optional**: If `context.condition` is not specified, all items are collected

## Common Pitfalls

### 1. Missing Context Condition (Unintentional All Logs Collection)

**Problem**: Not specifying `context.condition` collects ALL logs as context, which may be too broad.

**Wrong**:
```yaml
ottl_context_filter:
  condition: 'severity_text == "error"'
  ottl_context_filter_context:
    length: 10
```
This captures **all logs** (INFO, DEBUG, TRACE, etc.) as context.

**Correct** (if you want only specific levels):
```yaml
ottl_context_filter:
  condition: 'severity_text == "error"'
  ottl_context_filter_context:
    condition: 'severity_text == "info" or severity_text == "debug"'
    length: 10
```

### 2. OTTL Syntax Errors

**Problem**: Invalid OTTL expressions cause processor to fail.

**Wrong**:
```yaml
condition: 'severity_text = "error"'  # Single = instead of ==
```

**Correct**:
```yaml
condition: 'severity_text == "error"'
```

### 3. Length Too Large

**Problem**: Setting `length` too high can cause memory issues.

**Wrong**:
```yaml
length: 10000  # Exceeds maximum of 500
```

**Correct**:
```yaml
length: 100  # Within limits, reasonable for most use cases
```

### 4. Interval Too Long

**Problem**: Setting `interval` above 10 seconds is rejected.

**Wrong**:
```yaml
interval: 30s  # Exceeds maximum of 10s
```

**Correct**:
```yaml
interval: 5s  # Within limits
```

### 5. Using Multiline OTTL in Conditions

**Problem**: Unlike `ottl_transform` statements, conditions must be single-line.

**Wrong**:
```yaml
condition: |
  severity_text == "error" and
  attributes["service"] == "api"
```

**Correct**:
```yaml
condition: 'severity_text == "error" and attributes["service"] == "api"'
```

### 6. Forgetting to Quote OTTL

**Problem**: YAML parsing errors when OTTL contains special characters.

**Wrong**:
```yaml
condition: severity_text == "error"  # Missing quotes
```

**Correct**:
```yaml
condition: 'severity_text == "error"'
```

### 7. Context Condition Same as Trigger

**Problem**: Context condition matches trigger condition, causing trigger events to be buffered.

**Wrong**:
```yaml
ottl_context_filter:
  condition: 'severity_text == "error"'
  ottl_context_filter_context:
    condition: 'severity_text == "error"'  # Same as trigger!
```

**Correct**:
```yaml
ottl_context_filter:
  condition: 'severity_text == "error"'
  ottl_context_filter_context:
    condition: 'severity_text != "error"'  # Different from trigger
```

### 8. Not Understanding Extension Logic

**Problem**: Expecting multiple contexts when multiple triggers occur in succession.

**What happens**:
```yaml
Error 1 → Starts context (ctxID-1)
Info 1  → Added to context (ctxID-1)
Error 2 → EXTENDS context (ctxID-1), resets counter
Info 2  → Added to context (ctxID-1)
Info 3  → Added to context (ctxID-1)
```

**Expected Behavior**: All logs share `ctxID-1`. Multiple errors **extend** the same context rather than creating new ones.

### 9. Circular Buffer Eviction

**Problem**: Not understanding that buffer has limited size (same as `length`).

**What happens**: If 100 logs arrive before trigger and `length` is 10, only the last 10 are kept.

**Solution**: Increase `length` if you need more pre-trigger context, but balance with memory constraints.

### 10. Missing Flusher Configuration

**Problem**: Processor collects context but has no destination to send to.

**Impact**: Logs are dropped with warning "No flusher set".

**Solution**: Ensure the processor is connected to an output in the pipeline configuration.

## Best Practices

1. **Context Condition Selectivity**: Make context conditions specific enough to capture relevant logs but not so broad that you capture noise
2. **Reasonable Length**: Start with `length: 10-20` and adjust based on your debugging needs
3. **Balance Interval and Length**: Use `interval` as a safety net for low-traffic scenarios where `length` might not be reached
4. **Exclude Trace Logs**: Unless needed, exclude TRACE logs to reduce volume: `severity_text != "trace"`
5. **Test Conditions**: Validate OTTL conditions with sample data before deploying to production
6. **Monitor Context Size**: Watch for contexts approaching the 10 MB limit in high-volume environments
7. **Service-Specific Filtering**: Use `condition` (processor-level) to filter by service before context collection
8. **Unique Context IDs**: Leverage `ed.context.id` attribute in downstream processors and destinations for correlation
9. **Graceful Degradation**: Set reasonable limits to prevent memory exhaustion during error storms
10. **Cost Optimization**: Use this processor to ship only relevant logs instead of all logs, reducing storage costs
11. **Combine with Sampling**: For high-volume errors, consider adding sampling before this processor
12. **Document Trigger Conditions**: Use `comment` field to explain what events trigger context collection
13. **Destination Filtering**: Configure destinations to filter by `ed.context.id` for targeted storage
14. **Regular Review**: Periodically review trigger frequency to adjust `length` and `interval` as needed
15. **Avoid Overlapping Triggers**: If possible, make trigger conditions mutually exclusive to prevent context overlap

## Performance Considerations

### Memory Usage

- **Circular Buffer**: Maintains buffer size equal to `length` parameter
- **Active Context**: Stores all collected items until flush (up to 10 MB safety limit)
- **Buffer Eviction**: When buffer is full, oldest items are evicted (circular buffer behavior)
- **Data Size Monitoring**: Processor checks context size every 500ms to prevent memory overflow

### CPU Impact

- **OTTL Evaluation**: Both trigger and context conditions are evaluated for every log item
- **Condition Complexity**: Simple conditions (e.g., `severity_text == "error"`) are faster than complex regex patterns
- **Context Collection**: Minimal overhead - only adds attribute (`ed.context.id`) to items

### Optimization Tips

1. **Simple Conditions First**: Put simple checks before complex regex in conditions
2. **Limit Buffer Size**: Smaller `length` values reduce memory footprint
3. **Use Processor Condition**: Filter items before context evaluation with processor-level `condition`
4. **Avoid High-Cardinality Triggers**: If triggers fire very frequently, consider adding sampling upstream

## OTTL Functions for Context Filtering

### Common Functions for Trigger Conditions

| Function | Description | Example |
|----------|-------------|---------|
| `severity_text == "value"` | Severity level matching | `severity_text == "error"` |
| `IsMatch(body, "pattern")` | Regex match on log body | `IsMatch(body, "(?i)exception")` |
| `attributes["key"] == value` | Attribute equality | `attributes["http.status_code"] >= 500` |
| `attributes["key"] != nil` | Attribute exists | `attributes["duration_ms"] != nil` |
| `Len(attributes) > 0` | Has attributes | `Len(attributes) > 5` |

### Common Functions for Context Conditions

| Function | Description | Example |
|----------|-------------|---------|
| `severity_text != "value"` | Exclude severity | `severity_text != "trace"` |
| `severity_text == "a" or severity_text == "b"` | Multiple severities | `severity_text == "info" or severity_text == "debug"` |
| `IsMatch(body, "pattern")` | Include specific patterns | `IsMatch(body, "(?i)(request\|response)")` |
| `attributes["component"] == "value"` | Component filtering | `attributes["component"] == "database"` |

### Condition Operators

- **Comparison**: `==`, `!=`, `>`, `>=`, `<`, `<=`
- **Logical**: `and`, `or`, `not`
- **Pattern Matching**: `IsMatch(target, "regex")`
- **Null Checks**: `!= nil`, `== nil`

## Related Processors

- **ottl_filter**: Drop items based on OTTL conditions (doesn't collect context)
- **ottl_transform**: Transform attributes using OTTL before/after context collection
- **tail_sample**: Probabilistic sampling of traces (similar buffering concept)
- **suppress**: Suppress duplicate events (complementary deduplication)
- **extract_metric**: Convert logs to metrics (use context ID as metric attribute)
- **generic_mask**: Mask PII before context collection

## Comparison with Other Processors

### ottl_context_filter vs ottl_filter

| Feature | ottl_context_filter | ottl_filter |
|---------|-------------------|-------------|
| **Purpose** | Collect context around events | Drop items that don't match |
| **Buffering** | Yes (circular buffer) | No |
| **Context Collection** | Yes | No |
| **Output** | Multiple items per trigger | Single item or none |
| **Use Case** | Debugging, root cause analysis | Filtering, noise reduction |

**When to use ottl_context_filter**: When you need surrounding logs for debugging
**When to use ottl_filter**: When you want to drop uninteresting logs entirely

### ottl_context_filter vs tail_sample

| Feature | ottl_context_filter | tail_sample |
|---------|-------------------|-------------|
| **Data Type** | Logs only | Traces only |
| **Trigger** | OTTL condition | Trace completion + sampling |
| **Context** | Log context | Trace spans |
| **Use Case** | Error debugging | Trace sampling |

## Cross-References

- **edgedelta-pipelines skill**: Template 3 (Mixed Telemetry Processing)
- **OTTL Reference**: https://docs.edgedelta.com/ottl-statements/
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/ottl-context-filter-processor/

## Notes

- This is a **buffer processor** - it maintains state and requires careful memory management
- Context ID is stored in `ed.context.id` attribute (from `semconv1.AttributeEDContextID`)
- When processor stops, active contexts are flushed (graceful shutdown)
- Circular buffer size equals `length` parameter (pre-trigger context)
- Maximum 500ms check interval for buffer size monitoring
- Safety limits: max 500 items per context, max 10s interval, max 10 MB context size
- Default interval is 500ms if not specified (internal check interval)
- Both `condition` and `context.condition` use the same OTTL parser (CustomTransformContext)
- Items that match neither condition are dropped (logged as "miss" in metrics)
- Extension logic prevents context ID churn during error storms
- Processor uses rate-limited error logging to prevent log flooding on OTTL errors
- All OTTL conditions must be valid at processor creation time (fails fast on invalid OTTL)
- The processor is optimized for logs only (casts items to LogItem, drops others)
