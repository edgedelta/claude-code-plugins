# suppress Processor

**Type**: `suppress`
**Category**: Sampling & Rate Limiting / Deduplication
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/suppress/processor.go`
**Config Struct**: `configv3.Suppress`

## Quick Copy

```yaml
- type: suppress
  key_field_paths:
    - attributes["host.name"]
    - attributes["error.message"]
  number_to_allow: 1
  interval: 5m
```

## Overview

The `suppress` processor reduces duplicate events by allowing only a specified number of occurrences within a time window based on key field values. Unlike dedup which combines identical logs into one, suppress controls the rate at which duplicate events pass through the pipeline. This is essential for managing alert fatigue, preventing flood conditions from chatty log sources, and reducing costs from repetitive events.

## Use Cases

- **Alert Suppression**: Prevent duplicate alerts from flooding notification systems (allow 1 alert per error type per 5 minutes)
- **Chatty Log Management**: Reduce high-volume repetitive logs from noisy applications or services
- **Rate Limiting by Key**: Control event throughput based on composite keys (host + message, service + error code)
- **Cost Optimization**: Reduce forwarding costs by suppressing duplicate events to downstream systems
- **Notification Throttling**: Limit repeated notifications for the same issue to avoid overwhelming on-call engineers
- **Debug Log Control**: Allow first N occurrences of debug messages for troubleshooting, suppress rest
- **Duplicate Error Reduction**: Suppress repeated error messages after initial occurrences are captured
- **Multi-Field Deduplication**: Suppress based on combinations of fields (host + service + error type)

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `suppress` |
| `key_field_paths` | []string | OTTL field paths used to create suppression key (required, at least one) |
| `interval` | duration | Time window for suppression tracking (e.g., `5m`, `1h`). Also determines cache cleanup interval. |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `number_to_allow` | int | 1 | Number of occurrences to allow within the interval before suppressing (must be > 0) |
| `final` | bool | false | Mark as last processor in sequence (from SequenceProcessor) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## How Suppression Works

The suppress processor maintains stateful tracking of events based on key field values:

1. **Key Generation**: For each event, the processor extracts values from specified `key_field_paths` and creates a composite key by joining them with commas
2. **First Occurrence**: When a new key is seen, the processor creates a suppression record with interval start time and count = 1, and passes the event through
3. **Subsequent Occurrences**: When the same key is seen again:
   - **Within Interval & Under Limit**: If count < `number_to_allow`, increment count and pass through
   - **Within Interval & Over Limit**: If count >= `number_to_allow`, suppress (drop) the event
   - **After Interval Expires**: Reset the interval start time and count to 1, pass through
4. **Cache Cleanup**: Every `interval`, the processor cleans up suppression records older than `interval * 5` to prevent memory leaks
5. **Missing Fields**: If a key field is not found, uses "Unknown" as the value for that field

**State Persistence**: Suppression state is maintained in-memory per agent instance. State is NOT shared across distributed agents.

**Memory Management**: The processor automatically cleans up stale suppression records. Cache cleanup duration is 5x the interval (e.g., 5m interval = 25m cleanup).

## Examples

### Example 1: Basic Suppression by Single Key

```yaml
- type: suppress
  key_field_paths:
    - attributes["error.type"]
  number_to_allow: 1
  interval: 10m
```

**What it does**: Allows the first occurrence of each error type within a 10-minute window, suppresses all subsequent occurrences of the same error type until the window resets.

**Use case**: Prevent error log flooding from repeated exceptions.

### Example 2: Multiple Key Fields (Composite Keys)

```yaml
- type: suppress
  key_field_paths:
    - resource["host.name"]
    - attributes["service.name"]
    - attributes["error.message"]
  number_to_allow: 2
  interval: 5m
```

**What it does**: Creates a composite key from host + service + error message. Allows first 2 occurrences of each unique combination within 5 minutes, suppresses additional occurrences.

**Use case**: Control alerts per host-service-error combination while allowing a few occurrences for confirmation.

### Example 3: Allow N Occurrences Before Suppressing

```yaml
- type: suppress
  key_field_paths:
    - attributes["log.level"]
    - attributes["component"]
  number_to_allow: 5
  interval: 1m
```

**What it does**: Allows up to 5 occurrences of each log level + component combination per minute, suppresses the rest.

**Use case**: Let through a few samples of repeated logs for analysis, suppress excessive repetition.

### Example 4: Custom TTL with Longer Interval

```yaml
- type: suppress
  key_field_paths:
    - attributes["alert.name"]
  number_to_allow: 1
  interval: 30m
```

**What it does**: Allows only 1 occurrence of each alert name per 30 minutes. Cache cleanup happens automatically after 150 minutes (30m * 5).

**Use case**: Alert notification throttling - send alert once per half hour maximum.

### Example 5: Suppressing Duplicate Alerts

```yaml
- type: suppress
  key_field_paths:
    - attributes["monitor.name"]
    - attributes["severity"]
  number_to_allow: 1
  interval: 15m
```

**What it does**: Prevents duplicate alerts for the same monitor + severity within 15-minute windows.

**Use case**: Reduce alert fatigue by suppressing redundant monitor alerts.

### Example 6: Suppressing Chatty Log Sources

```yaml
- type: suppress
  key_field_paths:
    - resource["service.name"]
    - body
  number_to_allow: 3
  interval: 2m
```

**What it does**: For logs with identical service name and body content, allows first 3 within 2 minutes, suppresses additional duplicates.

**Use case**: Control high-volume repetitive logs from chatty microservices while keeping some samples.

### Example 7: Host + Message Suppression

```yaml
- type: suppress
  key_field_paths:
    - resource["host.name"]
    - attributes["message"]
  number_to_allow: 1
  interval: 1h
```

**What it does**: Per host, allows only 1 occurrence of each unique message per hour.

**Use case**: Prevent the same error message from flooding logs on individual hosts.

### Example 8: Conditional Suppression (Production Only)

```yaml
- type: suppress
  condition: 'attributes["environment"] == "production"'
  key_field_paths:
    - attributes["error.code"]
  number_to_allow: 2
  interval: 10m
```

**What it does**: Only suppresses in production environment. Allows 2 occurrences of each error code per 10 minutes in production.

**Use case**: Apply suppression rules only to production logs, allow full volume in non-production environments.

### Example 9: Suppressing by Status Code + Endpoint

```yaml
- type: suppress
  key_field_paths:
    - attributes["http.status_code"]
    - attributes["http.target"]
  number_to_allow: 10
  interval: 5m
```

**What it does**: Per HTTP status code and endpoint combination, allows 10 occurrences per 5 minutes.

**Use case**: Control high-volume HTTP logs while maintaining sample visibility into endpoint-specific errors.

### Example 10: Debug Log Suppression

```yaml
- type: suppress
  condition: 'attributes["log.level"] == "DEBUG"'
  key_field_paths:
    - attributes["function"]
    - body
  number_to_allow: 1
  interval: 30s
```

**What it does**: For DEBUG level logs, allows only 1 occurrence per function + body combination every 30 seconds.

**Use case**: Reduce debug log volume while maintaining one sample of each unique debug message.

## Validation Rules

1. **Required Fields**: Must specify `type`, `key_field_paths`, `interval`
2. **Interval Validation**: `interval` must be greater than 0 (positive duration)
3. **Number To Allow**: If specified, `number_to_allow` must be greater than 0. Default is 1 if not specified.
4. **Key Field Paths**: Must provide at least one key field path
5. **OTTL Field Paths**: Field paths must be valid OTTL expressions (e.g., `attributes["key"]`, `resource["host.name"]`, `body`)
6. **Default Interval**: If not specified, defaults to 30 seconds (but should always be explicitly set)

## Common Pitfalls

### 1. Key Field Path Errors

**Problem**: Incorrect OTTL syntax or wrong field references cause suppression to fail silently or use "Unknown" as key value.

**Wrong**:
```yaml
key_field_paths:
  - attributes.host.name  # Missing brackets and quotes
  - error_type            # Not an OTTL path
```

**Correct**:
```yaml
key_field_paths:
  - attributes["host.name"]
  - attributes["error.type"]
```

### 2. TTL Too Short

**Problem**: Setting interval too short causes excessive cache churn and doesn't effectively suppress duplicates.

**Wrong**:
```yaml
interval: 5s  # Too short for meaningful suppression
```

**Correct**:
```yaml
interval: 5m   # Reasonable window for duplicate detection
```

**Guidance**: Use intervals between 1m-30m for most use cases. Shorter intervals for high-volume streams, longer for alert suppression.

### 3. Number To Allow Misunderstanding

**Problem**: Confusing `number_to_allow` with "number to suppress".

**Wrong Understanding**: `number_to_allow: 5` means "suppress 5 occurrences"

**Correct Understanding**: `number_to_allow: 5` means "allow 5 occurrences, suppress everything after"

**Example**: If you see 10 events with the same key, and `number_to_allow: 3`, then 3 pass through, 7 are suppressed.

### 4. High Cardinality Keys (Memory Impact)

**Problem**: Using high-cardinality fields as keys causes memory growth.

**Wrong**:
```yaml
key_field_paths:
  - attributes["request.id"]    # Unique per request
  - attributes["timestamp"]      # Always different
  - attributes["user.session"]   # Too many unique values
```

**Correct**:
```yaml
key_field_paths:
  - resource["host.name"]        # Low cardinality
  - attributes["error.type"]     # Limited set of values
  - attributes["service.name"]   # Bounded set
```

**Guidance**: Choose key fields with bounded, low-to-medium cardinality (hundreds to low thousands of unique combinations).

### 5. Missing Key Fields

**Problem**: Key field doesn't exist on some events, causing unintended grouping under "Unknown".

**Example**: If `attributes["error.code"]` is missing, that event gets grouped with all other events missing that field.

**Solution**: Use conditional processing to ensure suppress only runs on events with required fields:

```yaml
- type: suppress
  condition: 'attributes["error.code"] != nil'
  key_field_paths:
    - attributes["error.code"]
  number_to_allow: 1
  interval: 5m
```

### 6. Suppression vs Dedup Confusion

**Problem**: Using suppress when you should use dedup, or vice versa.

**Suppress**: Rate limits duplicates (lets N through per time window)
**Dedup**: Combines identical logs into one with a count

**Wrong Use of Suppress**:
```yaml
# Trying to combine duplicates into one log with count
- type: suppress
  key_field_paths: [body]
  number_to_allow: 1
  interval: 1m
```

**Should Use Dedup Instead**:
```yaml
- type: dedup
  interval: 1m
  count_field_path: attributes["log_count"]
  excluded_field_paths: [timestamp]
```

### 7. Distributed State Assumptions

**Problem**: Assuming suppression state is shared across multiple agent instances.

**Reality**: Each agent maintains its own suppression state in-memory. If you have 3 agents and set `number_to_allow: 1`, each agent will allow 1 occurrence (total 3 across all agents).

**Mitigation**: Adjust `number_to_allow` based on agent count, or use centralized deduplication downstream.

## Best Practices

1. **Explicit Intervals**: Always explicitly set `interval` - don't rely on defaults
2. **Choose Meaningful Keys**: Select key fields that represent the logical "identity" of duplicate events
3. **Balance Sample Size**: Set `number_to_allow` to keep enough samples for debugging (2-5 is often good)
4. **Monitor Memory**: Track suppression cache size in high-cardinality scenarios
5. **Use Conditions**: Apply suppress selectively with conditions to avoid unnecessary processing
6. **Document Intent**: Use `comment` field to explain why suppression rules exist
7. **Test with Real Data**: Validate key field paths exist on your actual log data before deploying
8. **Combine with Sampling**: Use suppress for duplicates, sample for general volume reduction
9. **Alert on Suppression**: Consider emitting metrics when suppression is active to detect flood conditions
10. **Interval Alignment**: Align intervals with business requirements (alert SLAs, notification windows)
11. **Cache Cleanup Awareness**: Remember cache cleanup is 5x interval - don't set extremely long intervals
12. **Start Conservative**: Begin with permissive settings (higher `number_to_allow`) and tighten based on observed behavior
13. **Key Cardinality Check**: Count distinct values for key fields before deploying to estimate memory usage
14. **Use Resource Fields**: Include `resource` fields (host, service) in keys to prevent cross-host/service suppression
15. **Version Control**: Track suppression rules in version control to understand rate limiting changes over time

## Performance Considerations

### Key Cardinality Impact

- **Low Cardinality** (10-100 unique keys): Minimal memory usage, fast lookups
- **Medium Cardinality** (100-10,000 unique keys): Moderate memory, acceptable performance
- **High Cardinality** (>10,000 unique keys): Significant memory usage, potential performance impact

**Memory Estimation**: Each suppression record ~100 bytes + key size. 10,000 keys ≈ 1-2 MB memory.

### Interval Selection Trade-offs

- **Short Intervals** (< 1m): More frequent cache cleanup, less suppression effectiveness, lower memory
- **Medium Intervals** (1m-15m): Balanced suppression and memory usage (recommended)
- **Long Intervals** (> 30m): More effective suppression, higher memory due to longer retention

### Cache Cleanup Behavior

- Cleanup runs every `interval` period
- Removes records older than `interval * 5`
- Cleanup is lightweight but does lock the cache briefly
- No events are lost during cleanup

### Processing Overhead

- **Per Event**: Hash key generation, map lookup (O(1)), timestamp comparison
- **Lock Contention**: Read/write locks used - minimal contention in normal operation
- **CPU Impact**: Negligible for most workloads (< 1% CPU overhead)

## Suppression vs Deduplication Comparison

| Aspect | Suppress Processor | Dedup Processor |
|--------|-------------------|-----------------|
| **Purpose** | Rate limiting duplicates | Combining identical events |
| **Behavior** | Drops events after N occurrences | Merges duplicates into one event with count |
| **Output** | Original events (up to limit) | Single event with count field |
| **State** | Tracks occurrences per key | Tracks unique events by hash |
| **Memory** | Stores counts + timestamps | Stores full event copies |
| **Configuration** | `key_field_paths`, `number_to_allow` | `excluded_field_paths`, `count_field_path` |
| **Use Case** | Alert throttling, chatty log control | Log volume reduction, duplicate aggregation |
| **Counting** | Limits occurrences | Adds count to event |
| **Time Window** | Interval defines suppression window | Interval defines aggregation window |
| **Cardinality** | Handles higher cardinality better | Better for exact deduplication |
| **Downstream Impact** | Reduces event count | Reduces event count + adds metadata |

**When to Use Suppress**:
- You want to limit occurrences but preserve original event structure
- You need to allow N samples through for analysis
- Key-based rate limiting is sufficient
- You're suppressing alerts or notifications

**When to Use Dedup**:
- You want to know exactly how many duplicates occurred (count)
- You need precise deduplication by event content
- You want to reduce storage while maintaining duplicate visibility
- You're aggregating identical logs from multiple sources

**Can Use Both**: It's valid to use dedup first (to aggregate identical events) followed by suppress (to rate limit even the aggregated events).

## Related Processors

- **dedup**: Aggregates duplicate logs into single event with count field
- **sample**: Probabilistic sampling (different from key-based suppression)
- **rate_limit**: Global rate limiting across all events
- **ottl_filter**: Drop events based on conditions (no rate limiting)
- **ottl_transform**: Modify events before suppression to normalize key fields

## Cross-References

- **edgedelta-pipelines skill**: Use suppress in alert notification pipelines
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`
- **OTTL Field Paths**: Use for `key_field_paths` and `condition` parameters

## Documentation References

See https://docs.edgedelta.com for suppress processor documentation.

## Notes

- Suppression state is local to each agent instance - not shared across distributed agents
- If a key field path is invalid or field is missing, "Unknown" is used as the key value for that field
- Cache cleanup automatically prevents memory leaks by removing old suppression records
- The processor uses read/write locks for thread safety with minimal performance impact
- Suppression records are stored in-memory and lost on agent restart
- Default `number_to_allow` is 1, default `interval` is 30s (always set explicitly)
- Key generation uses comma-separated concatenation of field values
- Error logs are rate limited internally to prevent log flooding from processor errors
- Suppression is stateful and time-based - uses timestamps to track intervals
- Unlike sampling, suppression is deterministic based on key fields (not probabilistic)
- Cache cleanup duration is hardcoded to `interval * 5` for memory management
- Processor returns `PassThroughLogProcessResult` for allowed events, `TerminateLogProcessResult` for suppressed events
