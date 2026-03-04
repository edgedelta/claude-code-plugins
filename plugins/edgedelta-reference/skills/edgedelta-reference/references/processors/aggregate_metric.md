# aggregate_metric Processor

**Type**: `aggregate_metric`
**Category**: Metrics Extraction & Conversion
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/aggregate/metric/processor.go`
**Config Struct**: `configv3.AggregateMetric`

## Quick Copy

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "requests_per_host"
      conditions:
        - 'name == "http.server.requests"'
      aggregation_type: sum
      group_by:
        - 'attributes["host.name"]'
      interval: 1m
```

## Overview

The `aggregate_metric` processor aggregates and rolls up existing metrics based on conditions, grouping dimensions, and time windows. This processor enables powerful metric consolidation such as summing metrics across dimensions, calculating averages, finding min/max values, computing percentiles, and managing metric cardinality through dimension reduction.

## Use Cases

- **Cardinality Reduction**: Reduce high-cardinality metrics by aggregating across selected dimensions
- **Cross-Service Metrics**: Roll up metrics from multiple services into organization-wide totals
- **Dimension Rollup**: Create summary metrics by removing or grouping specific dimensions
- **Statistical Aggregation**: Calculate mean, median, percentiles from detailed metric streams
- **Cost Optimization**: Reduce metric volume and storage costs through intelligent aggregation
- **Custom Time Windows**: Create metrics at different time granularities (1m, 5m, 1h)
- **Multi-Region Aggregation**: Combine metrics across regions, zones, or clusters
- **Distinct Counting**: Track unique values across metric streams (user counts, session IDs)

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `aggregate_metric` |
| `aggregate_metric_rules` | []AggregateMetricRule | List of aggregation rules (at least one required) |

### AggregateMetricRule Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | No | Output metric name (if not specified, uses original metric name) |
| `conditions` | []string | No | OTTL conditions - aggregate only metrics matching all conditions |
| `interval` | duration | Yes | Time window for aggregation (e.g., `1m`, `5m`, `1h`) |
| `aggregation_type` | string | Yes | Type of aggregation: `sum`, `mean`, `min`, `max`, `count`, `median`, `p90`, `p95`, `p99`, `distinct_count` |
| `group_by` | []string | No | OTTL expressions defining aggregation dimensions (empty = aggregate all) |
| `keep_only_group_by_keys` | bool | No | If true, only keep dimensions specified in `group_by` (default: false) |
| `aggregation_options` | AggregateMetricAggregationOptions | No | Additional options (required for `distinct_count` aggregation) |

### AggregateMetricAggregationOptions Fields (for distinct_count)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `distinct_count_mode` | string | Yes | `"absolute"` (exact count, higher memory) or `"approximate"` (HyperLogLog, lower memory) |
| `distinct_count_keys` | []string | Yes | OTTL expressions for fields to count distinct values of |

### Optional Parameters (from SequenceProcessor)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `final` | bool | false | Mark as last processor in sequence |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## Aggregation Types

| Type | Description | Use Case |
|------|-------------|----------|
| `sum` | Sum all metric values | Total requests, total bytes, cumulative counts |
| `mean` | Calculate average | Average latency, average CPU usage |
| `min` | Find minimum value | Minimum response time, lowest memory usage |
| `max` | Find maximum value | Peak latency, maximum queue depth |
| `count` | Count number of metrics | Number of services reporting, metric cardinality |
| `median` | Calculate 50th percentile | Median latency, typical response time |
| `p90` | Calculate 90th percentile | 90th percentile latency (slower than 90% of requests) |
| `p95` | Calculate 95th percentile | 95th percentile latency |
| `p99` | Calculate 99th percentile | 99th percentile latency (tail latency) |
| `distinct_count` | Count unique values | Unique users, unique sessions, unique hosts |

## OTTL Conditions and Group By

### Conditions Field

The `conditions` field uses OTTL (OpenTelemetry Transformation Language) to filter which metrics to aggregate.

**Common OTTL Patterns**:
- `name == "metric.name"` - Match metric name
- `attributes["key"] == "value"` - Attribute equality
- `attributes["status"] > 400` - Numeric comparison
- `IsMatch(name, "^http\\.")` - Regex pattern matching
- `sum["aggregation_temporality"] == "delta"` - Check metric properties

**Multiple Conditions**: All conditions must be true (AND logic).

### Group By Field

The `group_by` field defines dimensions to preserve during aggregation. Metrics with the same group_by values are aggregated together.

**Common Group By Patterns**:
- `attributes["host.name"]` - Group by host
- `attributes["service.name"]` - Group by service
- `resource["cluster.name"]` - Group by cluster
- `attributes["region"]` - Group by region
- Multiple dimensions: group by combinations

**Empty group_by**: Aggregates all matching metrics into a single metric.

## Examples

### Example 1: Basic Sum Aggregation (All Metrics)

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "total_requests"
      conditions:
        - 'name == "http.server.requests"'
      aggregation_type: sum
      interval: 1m
```

**What it does**: Sums all `http.server.requests` metrics across all dimensions into a single `total_requests` metric every 1 minute.

### Example 2: Group By Single Dimension

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "requests_per_service"
      conditions:
        - 'name == "http.server.requests"'
      aggregation_type: sum
      group_by:
        - 'attributes["service.name"]'
      interval: 1m
```

**What it does**: Sums requests per service, creating one metric per unique service name.

### Example 3: Group By Multiple Dimensions

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "requests_per_service_and_region"
      conditions:
        - 'name == "http.server.requests"'
      aggregation_type: sum
      group_by:
        - 'attributes["service.name"]'
        - 'attributes["region"]'
      interval: 5m
```

**What it does**: Creates aggregated metrics for each unique combination of service and region.

### Example 4: Min/Max Aggregation

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "peak_cpu_per_host"
      conditions:
        - 'name == "system.cpu.usage"'
      aggregation_type: max
      group_by:
        - 'attributes["host.name"]'
      interval: 1m
    - name: "min_cpu_per_host"
      conditions:
        - 'name == "system.cpu.usage"'
      aggregation_type: min
      group_by:
        - 'attributes["host.name"]'
      interval: 1m
```

**What it does**: Calculates both peak and minimum CPU usage per host in 1-minute windows.

### Example 5: Count Aggregation (Metric Cardinality)

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "active_hosts_count"
      conditions:
        - 'IsMatch(name, "^system\\.")'
      aggregation_type: count
      group_by:
        - 'attributes["service.name"]'
      interval: 5m
```

**What it does**: Counts how many system metrics are reported per service in 5-minute windows.

### Example 6: Mean/Average Aggregation

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "avg_latency_per_endpoint"
      conditions:
        - 'name == "http.server.duration"'
      aggregation_type: mean
      group_by:
        - 'attributes["http.route"]'
      interval: 1m
```

**What it does**: Calculates average latency per HTTP endpoint every minute.

### Example 7: Percentile Aggregation (P90, P95, P99)

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "latency_p90"
      conditions:
        - 'name == "http.server.duration"'
      aggregation_type: p90
      interval: 1m
    - name: "latency_p95"
      conditions:
        - 'name == "http.server.duration"'
      aggregation_type: p95
      interval: 1m
    - name: "latency_p99"
      conditions:
        - 'name == "http.server.duration"'
      aggregation_type: p99
      interval: 1m
```

**What it does**: Calculates 90th, 95th, and 99th percentile latencies across all services.

### Example 8: Median Aggregation

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "median_response_time"
      conditions:
        - 'name == "http.server.duration"'
      aggregation_type: median
      group_by:
        - 'attributes["service.name"]'
      interval: 1m
```

**What it does**: Calculates median (50th percentile) response time per service.

### Example 9: Time Window Alignment (Multiple Intervals)

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "requests_1m"
      conditions:
        - 'name == "http.server.requests"'
      aggregation_type: sum
      interval: 1m
    - name: "requests_5m"
      conditions:
        - 'name == "http.server.requests"'
      aggregation_type: sum
      interval: 5m
    - name: "requests_1h"
      conditions:
        - 'name == "http.server.requests"'
      aggregation_type: sum
      interval: 1h
```

**What it does**: Creates three aggregated metrics at different time granularities (1 minute, 5 minutes, 1 hour).

### Example 10: Conditional Aggregation (Production Only)

```yaml
- type: aggregate_metric
  condition: 'attributes["environment"] == "production"'
  aggregate_metric_rules:
    - name: "prod_requests_total"
      conditions:
        - 'name == "http.server.requests"'
      aggregation_type: sum
      interval: 1m
```

**What it does**: Only aggregates metrics from production environment (processor-level condition).

### Example 11: Distinct Count - Absolute Mode

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "unique_users_per_service"
      conditions:
        - 'name == "user.session"'
      aggregation_type: distinct_count
      group_by:
        - 'attributes["service.name"]'
      interval: 5m
      aggregation_options:
        distinct_count_mode: absolute
        distinct_count_keys:
          - 'attributes["user.id"]'
```

**What it does**: Counts exact number of unique users per service in 5-minute windows (higher memory usage).

### Example 12: Distinct Count - Approximate Mode

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "approx_unique_sessions"
      conditions:
        - 'name == "session.metric"'
      aggregation_type: distinct_count
      interval: 1m
      aggregation_options:
        distinct_count_mode: approximate
        distinct_count_keys:
          - 'attributes["session.id"]'
```

**What it does**: Uses HyperLogLog to estimate unique sessions with lower memory footprint (suitable for high cardinality).

### Example 13: Keep Only Group By Keys (Dimension Reduction)

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "requests_per_region"
      conditions:
        - 'name == "http.server.requests"'
      aggregation_type: sum
      group_by:
        - 'resource'
        - 'attributes["region"]'
      keep_only_group_by_keys: true
      interval: 1m
```

**What it does**: Aggregates by region and drops all other dimensions (e.g., removes host, pod, container dimensions).

### Example 14: Multiple Aggregation Rules (Mixed Types)

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "total_errors"
      conditions:
        - 'IsMatch(name, "errors")'
      aggregation_type: sum
      interval: 1m
    - name: "avg_latency"
      conditions:
        - 'IsMatch(name, "latency")'
      aggregation_type: mean
      interval: 1m
    - name: "p99_latency"
      conditions:
        - 'IsMatch(name, "latency")'
      aggregation_type: p99
      interval: 1m
    - name: "active_services"
      aggregation_type: count
      group_by:
        - 'attributes["service.name"]'
      interval: 5m
  final: true
```

**What it does**: Applies multiple different aggregations to different metric patterns in a single processor.

## Validation Rules

1. **Required Fields**: Must specify `type` and `aggregate_metric_rules`
2. **Non-Empty Rules**: `aggregate_metric_rules` must contain at least one entry
3. **Valid Aggregation Type**: Must be one of: `sum`, `mean`, `min`, `max`, `count`, `median`, `p90`, `p95`, `p99`, `distinct_count`
4. **Interval Required**: Each rule must specify `interval`
5. **Valid Interval**: Must be a valid duration (e.g., `1m`, `30s`, `1h`)
6. **Distinct Count Options**: If `aggregation_type` is `distinct_count`, must provide `aggregation_options`
7. **Distinct Count Mode**: Must be `"absolute"` or `"approximate"`
8. **Distinct Count Keys**: Must provide `distinct_count_keys` when using `distinct_count`
9. **Valid OTTL**: All `conditions` and `group_by` expressions must be valid OTTL
10. **Group By Format**: Each `group_by` entry must be a valid OTTL expression

## Common Pitfalls

### 1. Missing Interval

**Problem**: Forgetting `interval` prevents metrics from being flushed.

**Wrong**:
```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "total_requests"
      aggregation_type: sum
      # Missing interval!
```

**Correct**:
```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "total_requests"
      aggregation_type: sum
      interval: 1m
```

### 2. Invalid Aggregation Type

**Problem**: Using unsupported aggregation type.

**Wrong**:
```yaml
aggregation_type: average  # Invalid!
```

**Correct**:
```yaml
aggregation_type: mean  # Use 'mean' for average
```

### 3. Missing Aggregation Options for Distinct Count

**Problem**: Using `distinct_count` without required options.

**Wrong**:
```yaml
- name: "unique_users"
  aggregation_type: distinct_count
  interval: 1m
  # Missing aggregation_options!
```

**Correct**:
```yaml
- name: "unique_users"
  aggregation_type: distinct_count
  interval: 1m
  aggregation_options:
    distinct_count_mode: approximate
    distinct_count_keys:
      - 'attributes["user.id"]'
```

### 4. Group By Syntax Errors

**Problem**: Incorrect OTTL syntax in `group_by`.

**Wrong**:
```yaml
group_by:
  - 'attributes[host.name]'  # Missing quotes around key
```

**Correct**:
```yaml
group_by:
  - 'attributes["host.name"]'
```

### 5. Cardinality Explosion with Group By

**Problem**: Grouping by high-cardinality dimensions creates too many metrics.

**Wrong** (High Cardinality):
```yaml
group_by:
  - 'attributes["request.id"]'  # Unique per request!
  - 'attributes["trace.id"]'    # Unique per trace!
```

**Correct** (Bounded Cardinality):
```yaml
group_by:
  - 'attributes["service.name"]'  # Limited number of services
  - 'attributes["region"]'        # Limited number of regions
```

### 6. Using keep_only_group_by_keys Without group_by

**Problem**: Setting `keep_only_group_by_keys: true` with empty `group_by` drops all dimensions.

**Wrong**:
```yaml
- name: "total_requests"
  aggregation_type: sum
  keep_only_group_by_keys: true  # No group_by defined!
  interval: 1m
```

**Correct**:
```yaml
- name: "requests_per_region"
  aggregation_type: sum
  group_by:
    - 'attributes["region"]'
  keep_only_group_by_keys: true
  interval: 1m
```

### 7. Confusing aggregation_type with Metric Type

**Problem**: Trying to aggregate gauge metrics as sum without considering metric semantics.

**Guidance**:
- Use `sum` for counters/deltas (requests, errors, bytes)
- Use `mean`, `median`, `min`, `max` for gauges (CPU%, memory%, queue depth)
- Use `count` to count number of metric data points

### 8. Incorrect Distinct Count Mode

**Problem**: Using `absolute` mode for high-cardinality data causes memory issues.

**Guidance**:
- `absolute`: Exact counts, but high memory usage (< 10K unique values)
- `approximate`: HyperLogLog estimation, low memory (> 10K unique values, ~2% error)

### 9. Multiple Conditions Logic

**Problem**: Expecting OR logic when conditions are ANDed.

**Wrong** (Expecting OR):
```yaml
conditions:
  - 'name == "metric1"'
  - 'name == "metric2"'  # Both can't be true!
```

**Correct** (Use Regex for OR):
```yaml
conditions:
  - 'IsMatch(name, "^(metric1|metric2)$")'
```

### 10. Interval Too Short

**Problem**: Very short intervals (e.g., `10s`) increase processing overhead.

**Guidance**:
- Use `1m` for most use cases (good balance)
- Use `30s` for high-frequency monitoring
- Use `5m` or `1h` for long-term trends
- Avoid intervals shorter than `30s` unless necessary

## Best Practices

1. **Start with Cardinality Analysis**: Before aggregating, understand your metric cardinality to choose appropriate `group_by` dimensions
2. **Use Meaningful Names**: Set `name` to clearly indicate the aggregation (e.g., `total_requests`, `avg_latency_per_service`)
3. **Cardinality Management**: Use `keep_only_group_by_keys: true` to aggressively reduce dimensions when needed
4. **Choose Appropriate Intervals**: Balance freshness (shorter intervals) vs overhead (longer intervals)
5. **Percentiles for Latency**: Use `p90`, `p95`, `p99` for latency metrics to understand tail behavior
6. **Mean + Min/Max Together**: Combine `mean`, `min`, `max` to get full distribution understanding
7. **Approximate for High Cardinality**: Use `approximate` mode for `distinct_count` when tracking > 10K unique values
8. **Test Conditions First**: Validate OTTL conditions on sample data before deploying
9. **Group By Bounded Dimensions**: Only group by dimensions with known, limited cardinality (service, region, environment)
10. **Multiple Rules for Different Metrics**: Use separate rules for different metric patterns rather than catch-all rules
11. **Document Aggregation Logic**: Use `comment` field to explain why specific aggregations are needed
12. **Monitor Output Cardinality**: Track how many aggregated metrics are produced to avoid cardinality explosion
13. **Use Conditions to Filter**: Apply `conditions` to only aggregate relevant metrics
14. **Align Intervals with Dashboards**: Match aggregation intervals to your dashboard time windows
15. **Consider Downstream Impact**: Understand how aggregated metrics will be queried and displayed

## Performance Considerations

### Memory Usage

- **Group By Cardinality**: Memory usage is proportional to number of unique group_by combinations
- **Interval Length**: Longer intervals accumulate more metric values before flushing
- **Distinct Count Mode**:
  - `absolute`: ~32 bytes per unique value
  - `approximate`: Fixed ~16KB per HyperLogLog sketch (regardless of cardinality)
- **Multiple Rules**: Each rule maintains separate aggregation buckets

### CPU Overhead

- **Condition Evaluation**: Each metric is evaluated against all rule conditions
- **OTTL Complexity**: Complex conditions (regex, multiple AND clauses) increase CPU usage
- **Percentile Calculations**: Percentiles require sorting values (O(n log n))
- **Group By Expressions**: Complex OTTL expressions in `group_by` increase overhead

### Cardinality Impact

**High Cardinality Dimensions** (Avoid in group_by):
- Request IDs, trace IDs, session IDs
- Timestamps, unique identifiers
- User IDs (unless using distinct_count)
- IP addresses (unless using distinct_count)

**Bounded Cardinality Dimensions** (Safe for group_by):
- Service names, application names
- Regions, zones, clusters
- Environments (prod, staging, dev)
- HTTP methods, status codes
- Resource types

**Cardinality Calculation**:
```
Output Cardinality = (Unique values of group_by[0]) × (Unique values of group_by[1]) × ...
```

### Optimization Tips

1. **Filter Early**: Use `conditions` to reduce metrics processed
2. **Limit Group By Dimensions**: Each additional dimension multiplies cardinality
3. **Use Processor-Level Condition**: Filter at processor level before rule evaluation
4. **Batch Similar Rules**: Group related aggregations in the same processor
5. **Choose Appropriate Distinct Count Mode**: Use `approximate` for high-cardinality tracking
6. **Flush Regularly**: Don't use extremely long intervals (> 1h) as they accumulate values in memory

## Aggregation Temporality Preservation

The processor preserves the aggregation temporality and metric type of the input metrics:

- **Sum Metrics**: Output preserves `aggregation_temporality` (delta/cumulative) and `is_monotonic`
- **Gauge Metrics**: Output remains as gauge
- **Histogram Metrics**: Output preserves histogram structure with aggregated buckets

**Note**: Metrics with different temporalities are aggregated separately (creates separate output metrics).

## Aggregation Types Reference

| Type | Algorithm | Output | Best For |
|------|-----------|--------|----------|
| `sum` | Σ(values) | Sum of all values | Totals, cumulative counts |
| `mean` | Σ(values) / count | Average | Typical values, average rates |
| `min` | min(values) | Minimum value | Lowest observed value |
| `max` | max(values) | Maximum value | Peaks, highest observed value |
| `count` | count(metrics) | Number of metrics | Cardinality, active series count |
| `median` | 50th percentile | Middle value | Typical value, less affected by outliers |
| `p90` | 90th percentile | 90% of values below this | Good performance threshold |
| `p95` | 95th percentile | 95% of values below this | Performance target |
| `p99` | 99th percentile | 99% of values below this | Tail latency, worst-case scenarios |
| `distinct_count` | HyperLogLog or Set | Unique value count | Unique users, sessions, IDs |

## Related Processors

- **extract_metric**: Create metrics from logs/traces (complementary to aggregate_metric)
- **cumulative_to_delta**: Convert cumulative metrics to delta before aggregation
- **ottl_transform**: Transform metric attributes before aggregation
- **ottl_filter**: Filter metrics before aggregation
- **log_to_pattern_metric**: Extract pattern metrics from logs

## Cross-References

- **edgedelta-pipelines skill**: For production pipeline examples using metric aggregation
- **OTTL Reference**: https://docs.edgedelta.com/ottl-statements/
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/aggregate-metric-processor/

## Notes

- The processor aggregates existing metrics; it does not create metrics from logs (use `extract_metric` for that)
- All OTTL conditions must be written on a single line in YAML (multiline not supported)
- Empty `group_by` aggregates all matching metrics into a single metric (full aggregation)
- The processor maintains separate buckets for metrics with different properties (name, unit, temporality)
- Metrics are flushed at interval boundaries aligned to wall clock time
- The processor automatically handles multiple intervals using GCD-based scheduling
- Output metrics have category `aggregated_metric` in attributes
- Percentile calculations require buffering all values in the interval window
- For high-throughput scenarios, consider using longer intervals to reduce overhead
- Distinct count with `approximate` mode uses HyperLogLog with ~2% error rate
- Resources and attributes are merged from all aggregated metrics unless `keep_only_group_by_keys: true`
