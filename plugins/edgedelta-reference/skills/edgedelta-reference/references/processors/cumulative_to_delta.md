# cumulative_to_delta Processor

**Type**: `cumulative_to_delta`
**Category**: Metrics Extraction & Conversion
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/cumulativetodelta/processor.go`
**Config Struct**: No configuration parameters required

## Quick Copy

```yaml
- type: cumulative_to_delta
  final: true
```

## Overview

The `cumulative_to_delta` processor converts cumulative metrics (ever-increasing counters) into delta metrics (rate of change between measurements). This conversion is essential for systems that expect delta/rate-style metrics or for calculating rates, throughput, and changes over time. The processor automatically tracks previous metric values and calculates deltas while detecting and handling counter resets.

## Use Cases

- **Rate Calculation**: Convert cumulative counters to rates for dashboards and alerts
- **System Compatibility**: Transform cumulative metrics for backends that expect delta temporality
- **Throughput Monitoring**: Calculate requests per second, bytes per second from cumulative totals
- **Change Detection**: Monitor the rate of change for error counts, request counts, and other counters
- **Prometheus Integration**: Convert Prometheus-style cumulative counters to delta format
- **Cost Optimization**: Reduce metric cardinality by converting cumulative histograms to delta histograms
- **Distributed Systems**: Normalize metrics from systems that export cumulative values
- **Performance Analysis**: Calculate per-interval performance metrics from cumulative measurements

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `cumulative_to_delta` |

**Note**: This processor has no additional configuration parameters. It automatically processes all cumulative metrics.

### Optional Parameters (from SequenceProcessor)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `final` | bool | false | Mark as last processor in sequence (required for final processor) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## Supported Metric Types

The processor handles two types of cumulative metrics:

### Sum Metrics (Counters)

**Requirements**:
- `aggregation_temporality` must be `cumulative`
- Metric kind must be `sum`

**What it does**: Converts cumulative sum values to delta by subtracting the previous value from the current value.

**Output**: Sum metric with `aggregation_temporality: delta`

### Histogram Metrics

**Requirements**:
- `aggregation_temporality` must be `cumulative`
- Metric kind must be `histogram`

**What it does**: Converts cumulative histogram data (count, sum, bucket counts) to delta values.

**Output**: Histogram metric with `aggregation_temporality: delta` and delta values for:
- Total count
- Total sum
- Each bucket count
- Preserves: explicit bounds, min, max, last

## How Conversion Works

### Delta Calculation Algorithm

The processor uses a state-based approach to calculate deltas:

1. **Metric Identification**: Each unique metric is identified by:
   - Metric name
   - All attributes (labels)
   - Metric kind (sum/histogram)
   - Unit
   - Additional properties (is_monotonic for sums, explicit_bounds for histograms)
   - Resource attributes

2. **State Tracking**: For each unique metric:
   - First observation is stored (no delta emitted)
   - Subsequent observations calculate: `delta = current_value - previous_value`
   - Previous value is updated after each calculation

3. **Counter Reset Detection**:
   - For sum metrics: If new value < previous value, it's treated as a valid delta (negative)
   - The processor does NOT automatically detect resets - it simply calculates the difference
   - Applications should use `is_monotonic: true` to indicate expected behavior

4. **Thread Safety**: State is protected by mutexes for concurrent processing

### State Management

- **Per-Metric State**: Each unique metric series maintains its own state
- **Memory Efficient**: State includes only the previous value, not full history
- **Persistent Until Reset**: State persists for the lifetime of the processor
- **No Expiration**: Metrics with long gaps between observations maintain their state

## Examples

### Example 1: Basic Cumulative to Delta Conversion

```yaml
- type: cumulative_to_delta
  final: true
```

**What it does**: Converts all cumulative metrics to delta format automatically.

**Input Metric** (Cumulative Sum):
```
http_requests_total{method="GET"} = 1000 (cumulative)
http_requests_total{method="GET"} = 1050 (cumulative, 10 seconds later)
```

**Output Metric** (Delta):
```
http_requests_total{method="GET"} = 50 (delta over 10 seconds)
```

### Example 2: Conditional Conversion (Production Only)

```yaml
- type: cumulative_to_delta
  condition: 'attributes["environment"] == "production"'
  final: true
```

**What it does**: Only converts cumulative metrics from production environment to delta.

### Example 3: With Upstream Metric Extraction

```yaml
- name: extract_and_convert_sequence
  type: sequence
  processors:
    - type: extract_metric
      extract_metric_rules:
        - name: "requests_total"
          unit: "1"
          conditions:
            - 'attributes["http.status_code"] != nil'
          sum:
            aggregation_temporality: cumulative
            is_monotonic: true
            value: 1
      interval: 1m
    - type: cumulative_to_delta
      final: true
```

**What it does**:
1. Extracts cumulative counter from logs
2. Converts cumulative counter to delta format
3. Outputs delta metrics suitable for rate calculations

### Example 4: Multiple Metric Conversion

```yaml
- name: multi_metric_conversion
  type: sequence
  processors:
    - type: extract_metric
      extract_metric_rules:
        - name: "bytes_sent_total"
          unit: "By"
          conditions:
            - 'attributes["bytes_sent"] != nil'
          sum:
            aggregation_temporality: cumulative
            is_monotonic: true
            value: attributes["bytes_sent"]
        - name: "requests_total"
          unit: "1"
          conditions:
            - 'IsMatch(body, "(?i)request")'
          sum:
            aggregation_temporality: cumulative
            is_monotonic: true
            value: 1
      interval: 30s
    - type: cumulative_to_delta
      final: true
```

**What it does**: Converts multiple cumulative metrics (bytes and requests) to delta format in one pass.

### Example 5: Histogram Delta Conversion

```yaml
- name: histogram_conversion
  type: sequence
  processors:
    # Assume histogram metrics are received as cumulative from OTLP input
    - type: cumulative_to_delta
      final: true
```

**Input Histogram** (Cumulative):
```yaml
request_duration:
  aggregation_temporality: cumulative
  count: 1000
  sum: 5000.0
  bucket_counts: [100, 200, 300, 400]
  explicit_bounds: [0.1, 0.5, 1.0, 5.0]
```

**Output Histogram** (Delta):
```yaml
request_duration:
  aggregation_temporality: delta
  count: 50          # Delta from previous observation
  sum: 250.0         # Delta sum
  bucket_counts: [5, 10, 15, 20]  # Delta counts per bucket
  explicit_bounds: [0.1, 0.5, 1.0, 5.0]  # Preserved
```

### Example 6: With Metadata for Debugging

```yaml
- type: cumulative_to_delta
  metadata: "converter_v1"
  comment: "Convert Prometheus cumulative counters to delta for Datadog"
  final: true
```

**What it does**: Adds metadata and comments for tracking and debugging the conversion process.

### Example 7: Handling Counter Resets

```yaml
- name: counter_with_resets
  type: sequence
  processors:
    - type: extract_metric
      extract_metric_rules:
        - name: "process_uptime_seconds"
          unit: "s"
          conditions:
            - 'attributes["uptime"] != nil'
          sum:
            aggregation_temporality: cumulative
            is_monotonic: true
            value: attributes["uptime"]
      interval: 1m
    - type: cumulative_to_delta
      comment: "Handles process restarts (counter resets)"
      final: true
```

**Behavior**:
```
Observation 1: 100s (stored, no output)
Observation 2: 160s → Output: 60s delta
Observation 3: 220s → Output: 60s delta
Observation 4: 30s  → Output: -190s delta (process restarted)
Observation 5: 90s  → Output: 60s delta (normal operation)
```

### Example 8: Per-Service Metric Conversion

```yaml
- name: per_service_conversion
  type: sequence
  processors:
    - type: extract_metric
      extract_metric_rules:
        - name: "service_requests_total"
          unit: "1"
          conditions:
            - 'attributes["service.name"] != nil'
          sum:
            aggregation_temporality: cumulative
            is_monotonic: true
            value: 1
      interval: 1m
    - type: cumulative_to_delta
      comment: "Separate state maintained for each service.name"
      final: true
```

**What it does**: Converts cumulative counters to delta while maintaining separate state for each service (based on metric attributes).

### Example 9: Multi-Stage Pipeline with Filtering

```yaml
- name: filtered_conversion
  type: sequence
  processors:
    - type: ottl_filter
      mode: include
      conditions:
        - 'attributes["metric.type"] == "counter"'
    - type: cumulative_to_delta
      final: true
```

**What it does**:
1. Filters to only include counter-type metrics
2. Converts those counters from cumulative to delta
3. Drops non-counter metrics

### Example 10: Complete Observability Pipeline

```yaml
- name: complete_metrics_pipeline
  type: sequence
  user_description: "Extract, convert, and aggregate metrics"
  processors:
    - type: extract_metric
      extract_metric_rules:
        - name: "api_errors_total"
          description: "Total API errors"
          unit: "1"
          conditions:
            - 'IsMatch(body, "(?i)ERROR")'
            - 'attributes["api.endpoint"] != nil'
          sum:
            aggregation_temporality: cumulative
            is_monotonic: true
            value: 1
      interval: 1m
    - type: cumulative_to_delta
      comment: "Convert to delta for rate calculation"
    - type: aggregate_metric
      aggregate_metric_rules:
        - name: "api_error_rate"
          unit: "1/s"
          aggregation_type: sum
          group_by:
            - "api.endpoint"
      interval: 1m
      final: true
```

**What it does**:
1. Extracts cumulative error counters from logs
2. Converts to delta format
3. Aggregates into per-endpoint error rates

## Validation Rules

1. **Metric Type**: Only processes metrics with kind `sum` or `histogram`
2. **Aggregation Temporality**: Input metrics must have `aggregation_temporality: cumulative`
3. **Valid Metric Data**: Metrics must have valid metric data structure
4. **Metric Key Generation**: Must be able to generate unique key from metric name, attributes, and properties
5. **Histogram Validation**:
   - Must have non-empty bucket counts
   - Bucket counts length must match between observations
   - Explicit bounds must match between observations
6. **State Consistency**: Metric structure (bounds, monotonic flag) must remain consistent across observations

## Common Pitfalls

### 1. First Observation Has No Output

**Issue**: The first time a metric is seen, no delta is emitted.

**Why**: The processor needs a previous value to calculate a delta.

**Solution**: This is expected behavior. Metrics will be emitted starting from the second observation.

**Wrong Expectation**:
```yaml
First metric arrives → Expect immediate output
```

**Correct Understanding**:
```yaml
First metric arrives → Stored as baseline (no output)
Second metric arrives → Delta calculated and emitted
```

### 2. Using with Delta Metrics

**Problem**: Applying cumulative_to_delta to metrics that are already delta format.

**Wrong**:
```yaml
- type: extract_metric
  extract_metric_rules:
    - name: "errors_total"
      sum:
        aggregation_temporality: delta  # Already delta
        is_monotonic: true
        value: 1
- type: cumulative_to_delta  # Will fail - expects cumulative
```

**Correct**:
```yaml
- type: extract_metric
  extract_metric_rules:
    - name: "errors_total"
      sum:
        aggregation_temporality: cumulative  # Cumulative
        is_monotonic: true
        value: 1
- type: cumulative_to_delta  # Converts to delta
```

### 3. Counter Reset Misinterpretation

**Problem**: Negative deltas are treated as errors instead of counter resets.

**Understanding**:
```yaml
# Counter resets produce negative deltas:
Observation 1: 1000 (stored)
Observation 2: 1100 → Delta: +100 ✓
Observation 3: 50   → Delta: -1050 (counter reset, not error)
```

**Correct Handling**: Downstream systems should handle negative deltas appropriately:
- Drop negative deltas
- Reset rate calculations
- Use absolute value for monotonic counters

### 4. Histogram Bound Changes

**Problem**: Changing histogram bucket boundaries between observations.

**Wrong**:
```yaml
# First observation
explicit_bounds: [0.1, 1.0, 10.0]

# Second observation
explicit_bounds: [0.5, 5.0, 50.0]  # Different bounds - ERROR
```

**Correct**: Histogram bucket boundaries must remain constant for a given metric series.

### 5. Missing Metric Attributes

**Problem**: Inconsistent attributes cause separate metric series.

**Example**:
```yaml
# These are treated as DIFFERENT metrics:
metric_name{service="api", environment="prod"}
metric_name{service="api"}  # Missing environment attribute
```

**Impact**: Each variant maintains separate state and calculates separate deltas.

**Solution**: Ensure consistent attribute sets for each metric series.

### 6. Time Series Gaps

**Issue**: Long gaps between observations can cause large deltas.

**Scenario**:
```yaml
00:00 - Value: 1000 (stored)
00:01 - Value: 1100 → Delta: 100 (1 minute)
02:00 - Value: 7100 → Delta: 6000 (59 minutes of activity)
```

**Understanding**: The processor doesn't time-normalize deltas. A 6000 delta over 59 minutes is accurate but may appear as a spike in rate graphs.

**Solution**: Consider aggregation intervals and be aware of the time dimension in your monitoring system.

### 7. State Memory Growth

**Problem**: Concern about memory usage with many unique metric series.

**Reality**: Each metric series stores only:
- Previous value (8-24 bytes for sum, more for histogram)
- Mutex overhead
- Map entry overhead

**Mitigation**:
- Monitor cardinality of metric attributes
- Avoid high-cardinality attributes (user IDs, request IDs)
- Use appropriate aggregation before conversion

### 8. Non-Monotonic Cumulative Metrics

**Issue**: Cumulative metrics that can decrease (not true counters).

**Example**:
```yaml
# Queue depth as cumulative (unusual but possible)
queue_depth_cumulative: 100
queue_depth_cumulative: 80  # Decreased

# Delta would be: -20
```

**Solution**: Ensure `is_monotonic` flag accurately reflects metric behavior. Non-monotonic cumulative metrics may produce negative deltas.

### 9. Processor Restart Behavior

**Problem**: What happens when the EdgeDelta agent restarts?

**Behavior**:
- All state is lost (in-memory only)
- First post-restart observation for each metric is stored (no output)
- Normal delta calculation resumes with second observation

**Impact**: One missing data point per metric series after restart.

**Mitigation**: Accept as normal behavior, or implement persistent state (not currently supported).

### 10. Mixing Cumulative and Delta Sources

**Problem**: Pipeline receives both cumulative and delta metrics with same name.

**Wrong**:
```yaml
# Source A sends: metric_total (cumulative)
# Source B sends: metric_total (delta)
# Both processed by cumulative_to_delta
```

**Result**: Delta metrics will be rejected (aggregation_temporality validation fails).

**Correct**: Use OTTL conditions to selectively process only cumulative metrics:
```yaml
- type: cumulative_to_delta
  condition: 'metric_data.sum.aggregation_temporality == "cumulative"'
```

## Best Practices

1. **Place After Extraction**: Position cumulative_to_delta immediately after metric extraction processors
2. **Use with Cumulative Sources**: Ideal for processing Prometheus-style cumulative counters
3. **Monitor First Observations**: Understand that each new metric series has no output on first observation
4. **Handle Counter Resets**: Ensure downstream systems can handle negative deltas from counter resets
5. **Consistent Attributes**: Maintain consistent attribute sets for each metric to avoid state fragmentation
6. **Document Conversions**: Use `comment` field to explain why conversion is needed
7. **Validate Temporality**: Only feed cumulative metrics to this processor
8. **Consider Cardinality**: High-cardinality attributes create more state entries
9. **Aggregation Strategy**: Apply aggregation before or after conversion based on your needs
10. **Rate Calculations**: After conversion, delta metrics are ready for rate calculations (delta/interval)
11. **Histogram Consistency**: Never change bucket boundaries for a histogram metric series
12. **Test Counter Resets**: Verify your monitoring system handles counter resets appropriately
13. **State Awareness**: Remember state is per-unique-metric-series (includes all attributes)
14. **Delta Normalization**: If needed, normalize deltas by time interval in downstream systems
15. **Error Handling**: Monitor processor error metrics for validation failures

## Performance Considerations

### Memory Usage

- **Per-Metric State**: Each unique metric series maintains state (~8-100 bytes depending on type)
- **Cardinality Impact**: Memory usage scales linearly with number of unique metric series
- **No History**: Only previous value is stored, not full history
- **Map Overhead**: Lazy map structure adds minimal overhead (~48 bytes per entry)

**Example Calculation**:
```
1000 unique sum metrics × 24 bytes = 24 KB
1000 unique histogram metrics × 200 bytes = 200 KB
Total ≈ 224 KB for 2000 metric series
```

### CPU Usage

- **Mutex Locking**: Per-metric mutex ensures thread-safe state updates
- **Minimal Computation**: Simple subtraction operations (sum or histogram buckets)
- **Map Lookups**: Hash map lookups are O(1) average case
- **Key Generation**: JSON serialization for metric keys (minimal overhead)

### Throughput

- **Lock Contention**: Different metric series can process in parallel (separate mutexes)
- **Same Series**: Serial processing for same metric series (mutex-protected)
- **Negligible Overhead**: Subtraction operations are extremely fast

**Recommendations**:
- Monitor high-cardinality metrics that might create excessive state
- Use aggregation to reduce unique metric series before conversion if needed
- Profile memory usage in high-volume environments

## Cumulative vs Delta Metrics

| Aspect | Cumulative | Delta |
|--------|-----------|-------|
| **Value Meaning** | Total since start | Change since last measurement |
| **Graph Behavior** | Ever-increasing line | Bars showing rate/change |
| **Rate Calculation** | Requires rate() function | Direct value is rate |
| **Counter Resets** | Visible as drops | Visible as negative spikes |
| **Storage** | Single value | Single value |
| **Use Case** | Total counts, ever-increasing | Rates, throughput, per-interval |
| **Prometheus** | Counter type (default) | Not native (requires recording rules) |
| **OTLP** | cumulative temporality | delta temporality |
| **Aggregation** | Must handle overlaps | Simple summation |

**Example**:
```yaml
# Cumulative:
http_requests_total: 0, 100, 250, 480, 720, ...
# Delta (after conversion):
http_requests_delta: -, 100, 150, 230, 240, ...
# Rate (delta/interval):
http_requests_per_sec: -, 1.67, 2.5, 3.83, 4.0, ...
```

## Related Processors

- **extract_metric**: Extract cumulative metrics from logs/traces before conversion
- **aggregate_metric**: Aggregate delta metrics after conversion
- **ottl_filter**: Filter which metrics to convert based on conditions
- **ottl_transform**: Modify metric attributes before or after conversion

## Cross-References

- **edgedelta-pipelines skill**: Template 3 (Mixed Telemetry Processing)
- **extract_metric processor**: For upstream metric extraction
- **aggregate_metric processor**: For downstream aggregation

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/cumulative-to-delta-processor/

## Notes

- **Initial Value Handling**: First observation stores value without emitting delta (returns `ErrInitialValueEncountered`)
- **Counter Reset Detection**: Processor calculates raw delta (current - previous), which may be negative after resets
- **State Persistence**: State is in-memory only and lost on processor restart
- **Thread Safety**: Each metric series has its own mutex for concurrent processing
- **Metric Identification**: Unique key includes name, attributes, kind, unit, resource, and type-specific properties
- **Histogram Bounds**: Must remain constant across observations for the same metric series
- **No Configuration**: Processor requires no configuration parameters
- **Automatic Detection**: Automatically processes all cumulative sum and histogram metrics
- **Observability**: Emits success/error/miss metrics for monitoring processor health
- **Error Handling**: Validation errors terminate processing for that specific metric
- **Lazy Map**: Uses lazy map for efficient state management (creates calculator on first access)
- **Generic Implementation**: Uses Go generics for type-safe delta calculation
