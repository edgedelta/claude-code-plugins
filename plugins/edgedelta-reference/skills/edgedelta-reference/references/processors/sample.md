# sample Processor

**Type**: `sample`
**Category**: Sampling & Rate Limiting
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/sample/probabilistic/processor.go`
**Config Struct**: `configv3.Sampling`

## Quick Copy

```yaml
- type: sample
  percentage: 10
```

## Overview

The `sample` processor performs deterministic hash-based probabilistic sampling to reduce data volume while maintaining statistical representativeness. Unlike random sampling, it uses consistent hashing on configurable fields to ensure the same items are always sampled or dropped across multiple pipeline instances. This is especially powerful for distributed tracing where sampling decisions must be consistent across all spans in a trace.

## Use Cases

- **Cost Optimization**: Reduce log/trace storage costs by 50-99% while maintaining data quality
- **Distributed Trace Consistency**: Sample complete traces by hashing on trace_id across all services
- **Volume Reduction**: Reduce high-volume telemetry data to manageable levels
- **Representative Sampling**: Maintain statistical properties of data while reducing volume
- **Multi-Service Consistency**: Ensure same sampling decision across distributed services
- **Performance Testing**: Sample production traffic for load testing environments
- **Compliance**: Apply consistent sampling rates across all pipeline instances
- **Priority Sampling**: Use dynamic sampling rates based on item attributes (errors at 100%, normal at 10%)

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `sample` |
| `percentage` | int | Sampling percentage (1-100) - percentage of items to keep |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `field_paths` | []string | Default paths | Fields to hash for consistent sampling (see Default Field Paths) |
| `priority_field` | string | "" | Field containing dynamic sampling percentage (0-100) |
| `pass_through_on_failure` | bool | true | Pass items through if sampling calculation fails |
| `timestamp_granularity` | duration | 1s | Round timestamps to this granularity for hash stability |

### Optional Parameters (from SequenceProcessor)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `final` | bool | false | Mark as last processor in sequence (required for final processor) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## Default Field Paths

When `field_paths` is not specified, the processor uses data-type-specific defaults:

**For Traces**:
- `trace_id` - Ensures all spans in a trace have the same sampling decision

**For Logs**:
- `timestamp` - Rounded to `timestamp_granularity` (default 1 second)
- `body` - Log message content
- `resource["service.name"]` - Service name from resource attributes

**Recommendation**: For traces, explicitly set `field_paths: ["trace_id"]` for clarity. For logs requiring consistency, specify custom field_paths.

## How Sampling Works

The sample processor uses deterministic hash-based sampling:

1. **Field Extraction**: Extract values from configured `field_paths` (or defaults)
2. **Hash Computation**: Compute 64-bit hash using FNV-1a algorithm on concatenated field values
3. **Hash Normalization**: Normalize hash to 0-100 range: `(hash / MAX_UINT64) * 100`
4. **Percentage Comparison**:
   - If normalized hash < `percentage`: **Keep item** (pass through)
   - If normalized hash ≥ `percentage`: **Drop item** (terminate)
5. **Caching**: Hash results are cached with 5-minute TTL for performance

**Key Characteristics**:
- **Deterministic**: Same field values always produce same hash (same sampling decision)
- **Consistent Across Instances**: All agents with same config make same decisions
- **Stateless**: No state tracking required (unlike rate limiting)
- **Thread-Safe**: Atomic operations ensure concurrent safety
- **Fast**: LRU cache with 8192 entries reduces hash computation overhead

**Hash Algorithm**: Uses FNV-1a 64-bit hash (from `github.com/edgedelta/edgedelta/pkg/hashext`)

## Priority Field Sampling

When `priority_field` is specified, the sampling percentage can be overridden per-item:

1. Extract value from `priority_field` attribute
2. If field exists and contains valid number (0-100), use it as sampling percentage
3. If field missing or invalid, use default `percentage`
4. Apply hash-based sampling with the determined percentage

**Use Cases**:
- Sample errors at 100%, normal logs at 10%
- Sample premium customers at 100%, standard at 25%
- Dynamic sampling rates based on log severity

## Examples

### Example 1: Basic Percentage Sampling (10%)

```yaml
- type: sample
  percentage: 10
  final: true
```

**What it does**: Keeps approximately 10% of logs using default field paths (timestamp + body + service.name). Deterministic - same logs always sampled.

**Use case**: Reduce log volume by 90% for cost savings.

### Example 2: Consistent Trace Sampling with trace_id

```yaml
- type: sample
  percentage: 25
  field_paths: ["trace_id"]
  comment: "Sample 25% of traces consistently across all services"
```

**What it does**: Samples 25% of traces. All spans with the same trace_id get the same sampling decision, ensuring complete trace retention.

**Use case**: Distributed tracing with consistent sampling across microservices.

### Example 3: Multi-Field Hashing for Logs

```yaml
- type: sample
  percentage: 50
  field_paths:
    - 'attributes["user_id"]'
    - 'attributes["session_id"]'
  comment: "Consistent sampling per user session"
```

**What it does**: Samples 50% of logs, but ensures all logs from the same user session are either all kept or all dropped.

**Use case**: User session analysis - keep complete sessions for analysis.

### Example 4: Priority Field Sampling

```yaml
- type: sample
  percentage: 10
  priority_field: 'attributes["sampling_priority"]'
  field_paths: ["trace_id"]
  comment: "Dynamic sampling: errors=100%, normal=10%"
```

**What it does**:
- If log has `attributes["sampling_priority"] = 100`, keeps it (100% sampling)
- If log has `attributes["sampling_priority"] = 10`, applies 10% sampling
- If field missing, uses default 10%

**Use case**: Always keep errors (priority=100), sample normal traffic at 10%.

### Example 5: Service Name Consistency

```yaml
- type: sample
  percentage: 20
  field_paths:
    - 'resource["service.name"]'
    - 'attributes["endpoint"]'
  comment: "Consistent sampling per service endpoint"
```

**What it does**: Samples 20% of traffic, but ensures consistency per service/endpoint combination.

**Use case**: Performance monitoring - maintain representative samples per endpoint.

### Example 6: Conditional Sampling (Production Only)

```yaml
- type: sample
  condition: 'resource["deployment.environment"] == "production"'
  percentage: 5
  field_paths: ["trace_id"]
  comment: "Sample 5% of production traces only"
```

**What it does**: Only applies sampling to production environment, drops 95% of production traces.

**Use case**: Reduce production data volume while keeping 100% of staging/dev.

### Example 7: Cost Optimization Sampling

```yaml
- type: sample
  percentage: 1
  field_paths:
    - 'attributes["request_id"]'
  pass_through_on_failure: false
  comment: "Aggressive 99% volume reduction for high-volume endpoints"
```

**What it does**: Keeps only 1% of logs, dropping 99%. Consistent per request_id. Drops items on failure instead of passing through.

**Use case**: Extreme cost reduction for very high-volume systems.

### Example 8: Combined with Filtering

```yaml
- type: ottl_filter
  conditions:
    - 'IsMatch(body, "(?i)ERROR") or IsMatch(body, "(?i)WARN")'
  comment: "Keep all errors/warnings"

- type: sample
  condition: 'not IsMatch(body, "(?i)ERROR") and not IsMatch(body, "(?i)WARN")'
  percentage: 10
  comment: "Sample 10% of info logs"
  final: true
```

**What it does**: Keeps 100% of errors/warnings, samples 10% of info logs.

**Use case**: Prioritize important logs while reducing normal traffic volume.

### Example 9: Timestamp Granularity for Log Deduplication

```yaml
- type: sample
  percentage: 50
  field_paths:
    - 'body'
    - 'resource["service.name"]'
  timestamp_granularity: 10s
  comment: "Sample similar logs within 10-second windows"
```

**What it does**: Samples 50% of logs. Logs with same body/service within 10-second windows get same sampling decision.

**Use case**: Reduce repetitive log volume while maintaining temporal distribution.

### Example 10: Session-Based Sampling for User Analytics

```yaml
- type: sample
  percentage: 30
  field_paths:
    - 'attributes["session.id"]'
  comment: "Keep complete user sessions for analytics"
```

**What it does**: Samples 30% of sessions, but keeps ALL logs from sampled sessions.

**Use case**: User behavior analytics - need complete session data for analysis.

## Validation Rules

1. **Required Fields**: Must specify `type` and `percentage`
2. **Percentage Range**: `percentage` must be 1-100 (inclusive)
3. **Valid Field Paths**: All `field_paths` must be valid OTTL getter expressions
4. **Priority Field**: If specified, `priority_field` must be valid OTTL getter expression
5. **Timestamp Granularity**: Must be valid duration (e.g., `1s`, `10s`, `1m`)
6. **Pass Through Flag**: `pass_through_on_failure` must be boolean (true/false)
7. **Data Type Support**: Only applies to logs and traces (metrics pass through unchanged)
8. **Field Path Format**: Must use OTTL syntax (e.g., `attributes["key"]`, not `attributes.key`)

## Common Pitfalls

### 1. Percentage Out of Range

**Problem**: Using percentage outside 1-100 range.

**Wrong**:
```yaml
- type: sample
  percentage: 0  # Invalid - must be 1-100
```

```yaml
- type: sample
  percentage: 150  # Invalid - must be 1-100
```

**Correct**:
```yaml
- type: sample
  percentage: 1  # Minimum 1%
```

```yaml
- type: sample
  percentage: 100  # Maximum 100%
```

### 2. Missing field_paths for Consistency

**Problem**: Expecting consistent sampling across services without specifying field_paths.

**Wrong**:
```yaml
# Default field_paths include timestamp + body + service.name
# Different services may have different bodies for same trace
- type: sample
  percentage: 10
```

**Correct**:
```yaml
# Explicitly hash on trace_id for cross-service consistency
- type: sample
  percentage: 10
  field_paths: ["trace_id"]
```

### 3. Sample vs rate_limit Confusion

**Problem**: Using `sample` when you need strict volume control.

**Wrong** (for strict volume caps):
```yaml
- type: sample
  percentage: 10
  # This keeps ~10% but not exactly 1000 items per minute
```

**Correct** (for strict volume caps):
```yaml
- type: rate_limit
  item_count_limit: 1000
  interval: 1m
  # This guarantees maximum 1000 items per minute
```

**When to use each**:
- **sample**: For representative sampling with consistency guarantees
- **rate_limit**: For strict throughput caps and downstream protection

### 4. Priority Field Configuration

**Problem**: Priority field contains non-numeric values.

**Wrong**:
```yaml
# If attributes["priority"] = "high" (string)
- type: sample
  percentage: 10
  priority_field: 'attributes["priority"]'
  # Fails to convert "high" to number
```

**Correct**:
```yaml
# Ensure priority field contains numeric values 0-100
- type: sample
  percentage: 10
  priority_field: 'attributes["sampling_rate"]'
  # attributes["sampling_rate"] = 100 for high priority
  # attributes["sampling_rate"] = 10 for normal
```

### 5. Distributed Sampling Without Coordination

**Problem**: Different pipeline configurations across services.

**Wrong**:
```yaml
# Service A
- type: sample
  percentage: 20
  field_paths: ["trace_id"]

# Service B (different percentage!)
- type: sample
  percentage: 10
  field_paths: ["trace_id"]
```

**Correct**:
```yaml
# Both services use SAME percentage and field_paths
# Service A and Service B
- type: sample
  percentage: 20
  field_paths: ["trace_id"]
```

### 6. Field Path Syntax Errors

**Problem**: Using incorrect OTTL syntax for field paths.

**Wrong**:
```yaml
field_paths:
  - attributes.user_id  # Wrong - dot notation
  - resource.service.name  # Wrong - nested dots
```

**Correct**:
```yaml
field_paths:
  - 'attributes["user_id"]'  # Correct - bracket notation
  - 'resource["service.name"]'  # Correct - quote the full key
```

### 7. Assuming Random Distribution

**Problem**: Expecting random sampling instead of deterministic.

**Misunderstanding**: "10% sampling means random 10% of items."

**Reality**: "10% sampling means items with hash < 10% threshold (always same items)."

**Implication**: Same input always produces same sampling decision. This is a feature (consistency), not a bug.

### 8. Ignoring Data Type Behavior

**Problem**: Expecting sampling to apply to metrics.

**Wrong Assumption**:
```yaml
- type: sample
  percentage: 50
  # Thinking this will sample metrics too
```

**Correct Understanding**:
```yaml
- type: sample
  percentage: 50
  # Only applies to logs and traces
  # Metrics pass through unchanged (no hashGetter for metrics)
```

### 9. Timestamp Granularity Misuse

**Problem**: Using fine-grained timestamp with default field_paths.

**Wrong**:
```yaml
- type: sample
  percentage: 10
  timestamp_granularity: 1ms
  # Every millisecond gets different hash, breaks consistency
```

**Correct**:
```yaml
- type: sample
  percentage: 10
  timestamp_granularity: 1m
  # Logs within same minute get same hash (better for dedup)
```

Or exclude timestamp entirely:
```yaml
- type: sample
  percentage: 10
  field_paths: ['body', 'resource["service.name"]']
  # Don't include timestamp in hash for better consistency
```

## Best Practices

1. **Explicit Field Paths**: Always specify `field_paths` for clarity, even if using defaults
2. **Trace Consistency**: For distributed tracing, use `field_paths: ["trace_id"]`
3. **Session Consistency**: For user analytics, hash on session_id or user_id
4. **Percentage Selection**: Start with 10-25% for most use cases, adjust based on volume/cost
5. **Priority Sampling**: Use `priority_field` to implement smart sampling (errors=100%, normal=10%)
6. **Coordinated Config**: Ensure all services use identical sampling config for distributed systems
7. **Test Consistency**: Validate that same trace_id produces same decision across services
8. **Monitor Drop Rate**: Track `pipeline.node.miss` metrics to understand actual sampling rate
9. **Document Intent**: Use `comment` field to explain sampling strategy
10. **Combine with Filters**: Use ottl_filter before sampling to implement tiered strategies
11. **Cache Efficiency**: Keep field_paths minimal to maximize cache hit rate
12. **Granularity Tuning**: Use larger `timestamp_granularity` for better deduplication
13. **Failure Handling**: Set `pass_through_on_failure: false` for strict volume control
14. **Testing**: Validate sampling behavior with known test data before production

## Performance Considerations

- **Hash Computation**: FNV-1a is fast, but avoid excessive field concatenation (< 5 fields ideal)
- **Cache Hit Rate**: 8192-entry LRU cache with 5-minute TTL provides ~95% hit rate for typical workloads
- **Percentage Impact**: Lower percentages = more drops = less downstream load
- **Field Path Count**: More fields = slower hash computation (typically negligible)
- **Timestamp Granularity**: Larger granularity = better cache hit rate = better performance
- **Memory Usage**: ~512KB for hash cache (8192 entries × 64 bytes average)
- **Thread Safety**: Atomic operations add minimal overhead (< 1% CPU impact)
- **Comparison**: ~10x faster than rate_limit for high-volume workloads (no timer overhead)

## Sample vs Rate Limit vs Tail Sample

| Feature | sample | rate_limit | tail_sample |
|---------|--------|------------|-------------|
| **Algorithm** | Hash-based probabilistic | Counter with interval reset | Policy-based decision after trace completion |
| **Consistency** | Deterministic (same input → same decision) | Non-deterministic (order-dependent) | Deterministic per trace |
| **Volume Control** | Approximate percentage | Exact count per interval | Policy-dependent |
| **Distributed** | Consistent across instances | Independent per instance | Requires trace aggregation |
| **Latency** | Immediate decision | Immediate decision | Delayed (waits for trace completion) |
| **State** | Stateless (cache only) | Stateful (counter) | Stateful (trace batches) |
| **Use Case** | Representative sampling | Strict throughput caps | Intelligent trace retention |
| **Data Types** | Logs, Traces | Logs, Metrics, Traces | Traces only |
| **Overhead** | Low (hash + cache) | Very low (atomic counter) | High (buffering + aggregation) |
| **Sequence** | Compatible | Compatible | Standalone only |

**Choosing the Right Processor**:
- Use **sample** for: Representative sampling with consistency requirements (distributed tracing)
- Use **rate_limit** for: Strict volume caps, downstream protection, burst control
- Use **tail_sample** for: Intelligent trace sampling (keep errors, sample success)

## Related Processors

- **rate_limit**: Counter-based volume control with strict limits
- **tail_sample**: Policy-based trace sampling after completion
- **ottl_filter**: Conditional filtering before sampling
- **dedup**: Deduplication based on field values
- **suppress**: Time-based suppression of repeated items

## Cross-References

- **edgedelta-pipelines skill**: Template 3 (Mixed Telemetry Processing)
- **Rate Limit Reference**: `.claude/skills/edgedelta-reference/references/processors/rate_limit.md`
- **Tail Sample Reference**: `.claude/skills/edgedelta-reference/references/processors/tail_sample.md`
- **Best Practices**: `.claude/skills/edgedelta-reference/references/best-practices.md`

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/sample-processor/

## Notes

- Hash algorithm is FNV-1a 64-bit (fast, good distribution, non-cryptographic)
- Default field paths differ by data type (traces use trace_id, logs use timestamp+body+service)
- Sampling is deterministic - same fields always produce same hash and same decision
- Cache has 8192 entries with 5-minute TTL to reduce hash computation overhead
- Metrics are passed through unchanged (no sampling applied to metrics)
- Priority field enables dynamic sampling rates per item (useful for error prioritization)
- `pass_through_on_failure` controls behavior when hash calculation fails (default: pass through)
- Timestamp granularity rounds timestamps before hashing for stability across time windows
- Thread-safe implementation using atomic operations for concurrent processing
- Can be combined with ottl_filter for sophisticated multi-tier sampling strategies
- For distributed tracing, all services must use identical percentage and field_paths
- The processor increments `pipeline.node.success` metric for kept items and `pipeline.node.miss` for dropped items
