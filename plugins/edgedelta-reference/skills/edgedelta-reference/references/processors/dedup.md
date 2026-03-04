# dedup Processor

**Type**: `dedup`
**Category**: Deduplication
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/dedup/processor.go`
**Config Struct**: `configv3.Dedup`

## Quick Copy

```yaml
- type: dedup
  interval: 1m
  count_field_path: attributes["log_count"]
  excluded_field_paths:
    - attributes["timestamp"]
```

## Overview

The `dedup` processor removes exact duplicate logs by combining identical events into a single log entry with a count field. Unlike `suppress` which rate limits duplicates, `dedup` uses content-based hashing to detect exact duplicates and aggregates them into one event with metadata about how many duplicates occurred. This is essential for reducing log volume, lowering storage costs, and maintaining visibility into duplicate event frequency.

## Use Cases

- **Log Volume Reduction**: Combine identical logs from multiple sources into single events with counts
- **Cost Optimization**: Reduce storage and forwarding costs by eliminating duplicate log entries
- **Duplicate Visibility**: Track exactly how many duplicate events occurred with count metadata
- **Repetitive Log Aggregation**: Consolidate high-volume repetitive logs (connection logs, heartbeats, status checks)
- **Multi-Instance Deduplication**: Aggregate identical logs from multiple service instances or containers
- **Error Aggregation**: Combine identical error messages while preserving occurrence count
- **Timestamp Normalization**: Deduplicate logs that differ only in timestamp values
- **Container Log Reduction**: Aggregate identical logs from replicated pods or containers

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `dedup` |
| `interval` | duration | Time window for deduplication aggregation (e.g., `1m`, `5m`, `30s`) |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `count_field_path` | string | `item.attributes.log_count` | Field path where duplicate count will be stored |
| `excluded_field_paths` | []string | `[timestamp]` | Field paths to exclude when computing hash (automatically excludes timestamp) |
| `final` | bool | false | Mark as last processor in sequence (from SequenceProcessor) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## How Deduplication Works

The dedup processor uses content-based hashing to detect and aggregate exact duplicates:

1. **Field Exclusion**: Before hashing, the processor removes excluded fields from the log data. By default, `timestamp` is always excluded (automatically added). Additional fields can be excluded via `excluded_field_paths`.

2. **Hash Generation**: The processor computes a 128-bit hash of the remaining log content using xxHash algorithm. The hash represents the unique "identity" of the log based on its content.

3. **Time Bucket Assignment**: Each log is assigned to a time bucket based on the `interval` parameter. Logs are bucketed by truncating their timestamp to the interval boundary.

4. **Duplicate Detection**: Within each time bucket, logs with identical hashes are considered duplicates:
   - **First Occurrence**: Creates a dedup record storing the log, timestamp, and count = 1
   - **Subsequent Duplicates**: Increments the count, updates the earliest timestamp

5. **Aggregation & Flushing**: At the end of each interval, the processor:
   - Takes the first occurrence of each unique log
   - Sets the count field to the total number of duplicates seen
   - Sets the start timestamp to the earliest occurrence
   - Sets the end timestamp to the interval boundary
   - Outputs the aggregated log

6. **Count Field Injection**: The duplicate count is injected into the log at the path specified by `count_field_path`.

**Hash Algorithm**: Uses xxHash (128-bit) for fast, collision-resistant hashing of log content.

**State Management**: Dedup state is maintained in-memory per agent instance using time-bucketed maps. State is NOT shared across distributed agents.

**Memory Management**: Time buckets are automatically cleaned up after processing to prevent memory leaks.

## Examples

### Example 1: Basic Deduplication

```yaml
- type: dedup
  interval: 1m
  count_field_path: attributes["log_count"]
```

**What it does**: Deduplicates identical logs within 1-minute windows, stores count in `attributes["log_count"]`. Timestamp is automatically excluded from hash computation.

**Use case**: Basic log volume reduction with duplicate counting.

### Example 2: Custom Count Field Path

```yaml
- type: dedup
  interval: 5m
  count_field_path: attributes["duplicate_count"]
```

**What it does**: Uses a custom field path `attributes["duplicate_count"]` to store the number of duplicates.

**Use case**: Custom naming for count field to match downstream expectations.

### Example 3: Excluding Timestamp and Request ID

```yaml
- type: dedup
  interval: 1m
  count_field_path: attributes["log_count"]
  excluded_field_paths:
    - attributes["request_id"]
    - attributes["correlation_id"]
```

**What it does**: Excludes request ID and correlation ID from hash computation (in addition to timestamp which is always excluded). Logs with different IDs but otherwise identical content will be deduplicated.

**Use case**: Deduplicate logs that have unique identifiers but identical content.

### Example 4: Longer Dedup Window

```yaml
- type: dedup
  interval: 30m
  count_field_path: attributes["log_count"]
```

**What it does**: Aggregates duplicates over 30-minute windows for longer-term deduplication.

**Use case**: Low-frequency logs where longer aggregation windows make sense.

### Example 5: High-Volume Log Deduplication

```yaml
- type: dedup
  interval: 1m
  count_field_path: attributes["occurrences"]
  excluded_field_paths:
    - attributes["timestamp"]
    - attributes["instance_id"]
    - attributes["pod_name"]
```

**What it does**: Deduplicates logs from multiple instances/pods, treating logs as identical even if they came from different sources.

**Use case**: Aggregate identical logs from replicated services or pods.

### Example 6: Error Log Aggregation

```yaml
- type: dedup
  condition: 'attributes["log.level"] == "ERROR"'
  interval: 5m
  count_field_path: attributes["error_count"]
  excluded_field_paths:
    - attributes["error_id"]
    - attributes["trace_id"]
```

**What it does**: Only deduplicates ERROR level logs, excluding trace/error IDs. Stores count in `error_count` field.

**Use case**: Aggregate identical errors while preserving visibility into error frequency.

### Example 7: Heartbeat/Health Check Deduplication

```yaml
- type: dedup
  condition: 'IsMatch(body, "(?i)(heartbeat|health check|keepalive)")'
  interval: 10m
  count_field_path: attributes["heartbeat_count"]
```

**What it does**: Deduplicates heartbeat and health check messages over 10-minute windows.

**Use case**: Reduce volume from repetitive health check logs while maintaining visibility.

### Example 8: Conditional Deduplication (Production Only)

```yaml
- type: dedup
  condition: 'attributes["environment"] == "production"'
  interval: 1m
  count_field_path: attributes["log_count"]
```

**What it does**: Only deduplicates in production environment, allows full log volume in other environments.

**Use case**: Apply deduplication only where needed for cost control.

### Example 9: JSON Field Exclusion

```yaml
- type: dedup
  interval: 2m
  count_field_path: attributes["log_count"]
  excluded_field_paths:
    - attributes["metadata.timestamp"]
    - attributes["metadata.request_id"]
    - attributes["correlation.trace_id"]
```

**What it does**: Excludes nested JSON fields from hash computation using dot notation paths.

**Use case**: Handle complex log structures with nested metadata fields.

### Example 10: Combined with Suppress

```yaml
- name: dedupe_and_suppress
  type: sequence
  processors:
    - type: dedup
      interval: 1m
      count_field_path: attributes["log_count"]
    - type: suppress
      key_field_paths:
        - attributes["error.type"]
      number_to_allow: 3
      interval: 5m
      final: true
```

**What it does**: First deduplicates identical logs, then suppresses duplicate error types (allowing only 3 occurrences per 5 minutes even after deduplication).

**Use case**: Multi-layer duplicate and rate limiting for extreme volume control.

## Validation Rules

1. **Required Fields**: Must specify `type` and `interval`
2. **Interval Validation**: `interval` must be greater than 0 (positive duration)
3. **Count Field Path**: If specified, `count_field_path` must be a valid field path (validated using pathextract)
4. **Excluded Field Paths**: Each path in `excluded_field_paths` must be a valid field path
5. **Body Exclusion Forbidden**: Cannot exclude `body` field - validation will fail with error `cannot exclude 'body'`
6. **Default Interval**: Defaults to 30 seconds if not specified (but should always be explicitly set)
7. **Default Count Field**: Defaults to `item.attributes.log_count` if not specified
8. **Auto-Exclude Timestamp**: Timestamp field is automatically added to `excluded_field_paths` (cannot be prevented)

## Common Pitfalls

### 1. Count Field Path Errors

**Problem**: Incorrect field path syntax causes count to not be stored properly.

**Wrong**:
```yaml
count_field_path: log_count  # Missing proper path prefix
```

**Correct**:
```yaml
count_field_path: attributes["log_count"]  # Proper OTTL path
```

**Note**: For CEL (non-sequence), use `item.attributes.log_count`. For OTTL (sequence), use `attributes["log_count"]`.

### 2. Excluded Fields Configuration

**Problem**: Forgetting to exclude high-cardinality fields that vary between otherwise identical logs.

**Wrong**:
```yaml
# No excluded_field_paths - request_id, trace_id cause false negatives
excluded_field_paths: []
```

**Correct**:
```yaml
excluded_field_paths:
  - attributes["request_id"]
  - attributes["trace_id"]
  - attributes["timestamp_ms"]
```

**Guidance**: Identify fields that vary but shouldn't prevent deduplication (IDs, timestamps, counters).

### 3. Trying to Exclude Body Field

**Problem**: Attempting to exclude the `body` field causes validation error.

**Wrong**:
```yaml
excluded_field_paths:
  - body  # Validation error: cannot exclude 'body'
```

**Correct**:
```yaml
excluded_field_paths:
  - attributes["timestamp"]
  - attributes["request_id"]
# Do not include body in exclusions
```

### 4. Dedup Window Too Large (Memory)

**Problem**: Using extremely long intervals causes excessive memory usage as more unique logs accumulate in buckets.

**Wrong**:
```yaml
interval: 24h  # 24 hours - very large memory footprint
```

**Correct**:
```yaml
interval: 5m   # 5 minutes - reasonable window
```

**Guidance**: Use 1m-30m for most use cases. Longer intervals increase memory proportionally to unique log count.

### 5. Hash Collisions (Very Rare)

**Problem**: Worrying about hash collisions with xxHash.

**Reality**: xxHash 128-bit collisions are astronomically rare (1 in 2^128). Not a practical concern for log deduplication.

**Guidance**: Trust the hash - collisions won't happen in production log volumes.

### 6. Dedup vs Suppress Confusion

**Problem**: Using dedup when you want rate limiting, or suppress when you want aggregation.

**Dedup**: Aggregates identical logs into one with count
**Suppress**: Rate limits duplicates by key fields

**Wrong Use of Dedup**:
```yaml
# Trying to rate limit by error type
- type: dedup
  interval: 5m
  # Dedup uses content hash, not specific key fields
```

**Should Use Suppress Instead**:
```yaml
- type: suppress
  key_field_paths:
    - attributes["error.type"]
  number_to_allow: 1
  interval: 5m
```

### 7. Timestamp Fields Causing False Negatives

**Problem**: Additional timestamp fields not excluded from hash computation.

**Wrong**:
```yaml
# timestamp is auto-excluded, but timestamp_ms and created_at are not
excluded_field_paths: []
```

**Correct**:
```yaml
excluded_field_paths:
  - attributes["timestamp_ms"]
  - attributes["created_at"]
  - attributes["event_time"]
```

### 8. Distributed State Assumptions

**Problem**: Assuming dedup state is shared across multiple agent instances.

**Reality**: Each agent maintains its own dedup state in-memory. If you have 3 agents processing the same logs, each will deduplicate independently.

**Mitigation**: Use centralized deduplication downstream, or accept that dedup is per-agent.

## Best Practices

1. **Explicit Intervals**: Always explicitly set `interval` - don't rely on 30s default
2. **Identify Varying Fields**: Analyze your logs to find fields that vary but shouldn't prevent deduplication
3. **Exclude IDs and Timestamps**: Always exclude request IDs, trace IDs, correlation IDs, and any additional timestamp fields
4. **Short to Medium Windows**: Use 1m-10m intervals for most use cases (balance between dedup effectiveness and memory)
5. **Monitor Memory Usage**: Track dedup processor memory usage in high-volume environments
6. **Use Conditions Selectively**: Apply dedup only to log types that benefit from it (skip already-unique logs)
7. **Document Count Field**: Use descriptive count field names (`log_count`, `duplicate_count`, `occurrences`)
8. **Combine with Sampling**: Use dedup for exact duplicates, sampling for general volume reduction
9. **Test with Real Data**: Validate excluded field paths with sample logs before deploying
10. **Version Control**: Track dedup configurations to understand volume reduction changes over time
11. **Preserve Original Structure**: Dedup adds count field but preserves original log structure
12. **Alert on High Counts**: Monitor count field values to detect abnormal duplicate patterns
13. **Use in Sequences**: Combine dedup with other processors (mask before dedup, suppress after dedup)
14. **CEL vs OTTL Paths**: Use correct path syntax based on context (CEL for nodes, OTTL for sequences)
15. **Final Flag**: Mark as `final: true` if this is the last processor in sequence

## Performance Considerations

### Hash Computation Overhead

- **Algorithm**: xxHash is extremely fast (GB/s throughput)
- **CPU Impact**: Minimal - typically <1% CPU overhead for hash computation
- **Field Exclusion**: Removing fields before hashing adds negligible overhead
- **Optimization**: Hash computation is cached per log item

### Memory Usage

**Memory per Dedup Record**: ~200-500 bytes (depends on log size)
- Stores full log item copy
- Timestamp metadata
- Count value
- Hash key (16 bytes)

**Memory Estimation**:
- 1,000 unique logs/minute × 1m interval = ~0.5 MB memory
- 10,000 unique logs/minute × 5m interval = ~25 MB memory
- 100,000 unique logs/minute × 10m interval = ~500 MB memory

**Memory Management**:
- Time buckets are cleaned up immediately after flushing
- No long-term memory accumulation
- Memory usage is proportional to: `unique_logs_per_second × interval`

### Interval Selection Trade-offs

- **Short Intervals** (< 1m): Less effective deduplication, lower memory, more frequent flushing
- **Medium Intervals** (1m-10m): Balanced deduplication effectiveness and memory (recommended)
- **Long Intervals** (> 30m): Better deduplication, higher memory, potential memory pressure

### Processing Throughput

- **Lock Contention**: Uses mutex locks - minimal contention in normal operation
- **Flush Overhead**: Flushing happens every `interval` - lightweight operation
- **Throughput**: Can handle 100k+ logs/second on modern hardware

### Cardinality Impact

- **Low Cardinality** (few unique logs): Excellent deduplication, minimal memory
- **High Cardinality** (many unique logs): Less effective deduplication, higher memory
- **Worst Case**: All logs unique = no deduplication benefit, memory overhead only

## Dedup vs Suppress Comparison

| Aspect | Dedup Processor | Suppress Processor |
|--------|-----------------|-------------------|
| **Purpose** | Aggregate identical events | Rate limit duplicate events |
| **Behavior** | Combines duplicates into one | Drops events after N occurrences |
| **Output** | Single event with count field | Original events (up to limit) |
| **Detection Method** | Content hash (exact match) | Key field values (composite keys) |
| **State Storage** | Full log copies in memory | Counts + timestamps only |
| **Memory Usage** | Higher (stores full logs) | Lower (stores metadata only) |
| **Configuration** | `excluded_field_paths`, `count_field_path` | `key_field_paths`, `number_to_allow` |
| **Deduplication Type** | Exact content-based | Key-based grouping |
| **Count Metadata** | Adds count field to event | Does not modify events |
| **Use Case** | Log volume reduction with visibility | Alert throttling, rate limiting |
| **Precision** | Exact duplicate detection | Fuzzy grouping by keys |
| **Cardinality Handling** | Struggles with high cardinality | Handles high cardinality better |
| **Downstream Impact** | Reduces events + adds metadata | Reduces event count only |
| **Timestamp Handling** | Auto-excludes from hash | Manual key field selection |

**When to Use Dedup**:
- You want to know exactly how many duplicates occurred
- You need precise, content-based deduplication
- You want to reduce storage while maintaining duplicate visibility
- Logs are truly identical (or differ only in excluded fields)
- You're aggregating logs from multiple identical sources

**When to Use Suppress**:
- You want to rate limit events by specific characteristics
- You need to allow N samples through for analysis
- You want to preserve original event structure
- You're throttling alerts or notifications
- Logs aren't identical but share key characteristics

**Can Use Both**: Dedup first (aggregate identical logs) then suppress (rate limit the aggregated events by key fields).

## Related Processors

- **suppress**: Rate limit duplicate events based on key field values
- **sample**: Probabilistic sampling (different from deduplication)
- **aggregate_metric**: Aggregate multiple metrics together
- **ottl_transform**: Transform logs before deduplication to normalize content
- **ottl_filter**: Filter logs before deduplication to reduce processing

## Cross-References

- **edgedelta-pipelines skill**: Use dedup in log processing pipelines for volume reduction
- **suppress processor**: For comparison and understanding when to use each
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/deduplicate-logs-processor/

## Notes

- Dedup state is local to each agent instance - not shared across distributed agents
- Timestamp field is ALWAYS excluded from hash computation (automatically added to excluded_field_paths)
- Cannot exclude the `body` field - validation will fail with error
- Uses xxHash 128-bit algorithm for fast, collision-resistant hashing
- Hash collisions are astronomically rare (not a practical concern)
- Default `count_field_path` is `item.attributes.log_count` (CEL) or `attributes["log_count"]` (OTTL)
- Default `interval` is 30 seconds (always set explicitly for clarity)
- Dedup records are stored in-memory and lost on agent restart
- Time bucket cleanup happens automatically after flushing to prevent memory leaks
- The processor recalculates log size after adding count field
- Start timestamp is set to earliest occurrence, end timestamp to interval boundary
- Field path validation differs between CEL (node) and OTTL (sequence) contexts
- For sequences, use OTTL syntax (`attributes["key"]`); for nodes, use CEL syntax (`item.attributes.key`)
- Excluded field paths use pathextract library for flexible field access (supports nested paths)
- Memory usage scales with: unique log count × interval duration × log size
- Processor returns `TerminateLogProcessResult` - logs are buffered until flush, not passed through immediately
- Error logs are rate limited internally to prevent processor error flooding
- Dedup is stateful and time-based - uses time buckets aligned to interval boundaries
