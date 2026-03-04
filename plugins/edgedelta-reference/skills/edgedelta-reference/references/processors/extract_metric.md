# extract_metric Processor

**Type**: `extract_metric`
**Category**: Metrics Extraction & Conversion
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/extract/metric/processor.go`
**Config Struct**: `configv3.ExtractMetric`

## Quick Copy

```yaml
- type: extract_metric
  extract_metric_rules:
    - name: "errors_total"
      unit: "1"
      conditions:
        - 'IsMatch(body, "(?i)ERROR")'
      sum:
        aggregation_temporality: delta
        is_monotonic: true
        value: 1
  interval: 1m
  final: true
```

## Overview

The `extract_metric` processor converts logs and traces into metrics by evaluating OTTL conditions and extracting numerical values. This enables powerful log-to-metric transformations for monitoring key performance indicators such as error rates, request counts, latency percentiles, and custom business metrics.

## Use Cases

- **Error Monitoring**: Count errors, exceptions, and failures from application logs
- **SLI Tracking**: Extract service level indicators (request rates, success rates, latency)
- **Business Metrics**: Generate custom metrics from business events in logs
- **Cost Optimization**: Reduce log storage costs by converting high-volume logs to compact metrics
- **Performance Tracking**: Extract latency, throughput, and resource usage metrics from traces
- **Alerting**: Create metrics for alerting on specific log patterns

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `extract_metric` |
| `extract_metric_rules` | []ExtractMetricRule | List of metric extraction rules |
| `interval` | duration | How often to flush extracted metrics (e.g., `1m`, `30s`) |

### ExtractMetricRule Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Metric name (must follow metric naming conventions) |
| `unit` | string | Yes | Metric unit (e.g., `"1"` for count, `"ms"` for milliseconds, `"By"` for bytes) |
| `conditions` | []string | Yes | OTTL conditions - extract metric only when all conditions are true |
| `sum` | ExtractMetricSum | One required | Configuration for sum/counter metrics (mutually exclusive with `gauge`) |
| `gauge` | ExtractMetricGauge | One required | Configuration for gauge metrics (mutually exclusive with `sum`) |
| `description` | string | No | Human-readable description of the metric |

**Note**: Each rule must specify exactly one of `sum` or `gauge` (not both).

### ExtractMetricSum Fields (for Counter/Sum Metrics)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `aggregation_temporality` | string | Yes | `"delta"` (rate/counter) or `"cumulative"` (ever-increasing) |
| `is_monotonic` | bool | Yes | `true` for monotonically increasing counters, `false` for up-down counters |
| `value` | string | Yes | OTTL expression for metric value (e.g., `1` for count, `attributes["duration"]` for duration sum) |

### ExtractMetricGauge Fields (for Gauge Metrics)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `value` | string | Yes | OTTL expression for gauge value (e.g., `attributes["memory_usage"]`, `ConvertCase(body, "lower")`) |

### Optional Parameters (from SequenceProcessor)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `final` | bool | false | Mark as last processor in sequence (required for final processor) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## Metric Types

### Sum Metrics (Counters)

**When to use**: For values that accumulate over time (counts, totals, sums).

**Aggregation Temporality**:
- **delta**: Reports the change since last measurement (rate-style, preferred for most counters)
- **cumulative**: Reports the total since process start (ever-increasing)

**is_monotonic**:
- **true**: Value only increases (e.g., request count, error count)
- **false**: Value can increase or decrease (e.g., queue depth, in-flight requests)

### Gauge Metrics

**When to use**: For values that can go up or down and represent a snapshot (temperature, memory usage, queue size).

**Characteristics**:
- Represents instantaneous value at a point in time
- No aggregation temporality (gauges are always point-in-time)
- Can increase or decrease freely

## OTTL Conditions

The `conditions` field uses OTTL (OpenTelemetry Transformation Language) to determine when to extract a metric.

**Common OTTL Functions**:
- `IsMatch(body, "pattern")` - Regex match against log body
- `attributes["key"] == "value"` - Attribute equality
- `attributes["status"] > 400` - Numeric comparison
- `IsMatch(attributes["level"], "(?i)ERROR")` - Case-insensitive match

**Multiple Conditions**: All conditions must be true (AND logic). For OR logic, use a single condition with regex alternation.

## Examples

### Example 1: Basic Error Counter

```yaml
- type: extract_metric
  extract_metric_rules:
    - name: "app_errors_total"
      description: "Count of application errors"
      unit: "1"
      conditions:
        - 'IsMatch(body, "(?i)(ERROR|EXCEPTION|FATAL|CRITICAL)")'
      sum:
        aggregation_temporality: delta
        is_monotonic: true
        value: 1
  interval: 1m
  final: true
```

**What it does**: Creates a counter metric that increments by 1 for each log containing error keywords.

### Example 2: HTTP Status Code Counter

```yaml
- type: extract_metric
  extract_metric_rules:
    - name: "http_requests_total"
      description: "Total HTTP requests"
      unit: "1"
      conditions:
        - 'attributes["http.status_code"] != nil'
      sum:
        aggregation_temporality: delta
        is_monotonic: true
        value: 1
  interval: 30s
```

**What it does**: Counts all logs that have an `http.status_code` attribute.

### Example 3: Multiple Metrics from Same Logs

```yaml
- type: extract_metric
  extract_metric_rules:
    - name: "errors_total"
      unit: "1"
      conditions:
        - 'IsMatch(body, "(?i)ERROR")'
      sum:
        aggregation_temporality: delta
        is_monotonic: true
        value: 1
    - name: "warnings_total"
      unit: "1"
      conditions:
        - 'IsMatch(body, "(?i)WARNING")'
      sum:
        aggregation_temporality: delta
        is_monotonic: true
        value: 1
    - name: "criticals_total"
      unit: "1"
      conditions:
        - 'IsMatch(body, "(?i)CRITICAL")'
      sum:
        aggregation_temporality: delta
        is_monotonic: true
        value: 1
  interval: 1m
  final: true
```

**What it does**: Extracts 3 separate counter metrics (errors, warnings, criticals) from the same log stream.

### Example 4: Latency Sum Metric

```yaml
- type: extract_metric
  extract_metric_rules:
    - name: "request_duration_ms_total"
      description: "Total request duration"
      unit: "ms"
      conditions:
        - 'attributes["duration_ms"] != nil'
      sum:
        aggregation_temporality: delta
        is_monotonic: true
        value: attributes["duration_ms"]
  interval: 1m
```

**What it does**: Sums the `duration_ms` attribute values to create a total latency metric (combine with request count for average calculation).

### Example 5: Gauge Metric for Memory Usage

```yaml
- type: extract_metric
  extract_metric_rules:
    - name: "memory_usage_bytes"
      description: "Current memory usage"
      unit: "By"
      conditions:
        - 'attributes["memory.usage"] != nil'
      gauge:
        value: attributes["memory.usage"]
  interval: 30s
```

**What it does**: Extracts instantaneous memory usage values as a gauge metric.

### Example 6: Conditional Extraction (Production Only)

```yaml
- type: extract_metric
  condition: 'attributes["environment"] == "production"'
  extract_metric_rules:
    - name: "prod_errors_total"
      unit: "1"
      conditions:
        - 'IsMatch(body, "(?i)ERROR")'
      sum:
        aggregation_temporality: delta
        is_monotonic: true
        value: 1
  interval: 1m
```

**What it does**: Only extracts metrics from logs with `environment=production`.

### Example 7: Complex Condition with Multiple Attributes

```yaml
- type: extract_metric
  extract_metric_rules:
    - name: "slow_requests_total"
      description: "Requests slower than 1 second"
      unit: "1"
      conditions:
        - 'attributes["duration_ms"] > 1000'
        - 'attributes["http.method"] == "GET"'
        - 'attributes["http.status_code"] < 500'
      sum:
        aggregation_temporality: delta
        is_monotonic: true
        value: 1
  interval: 1m
```

**What it does**: Counts successful GET requests that took longer than 1 second.

### Example 8: Metric with Custom Value Expression

```yaml
- type: extract_metric
  extract_metric_rules:
    - name: "bytes_processed_total"
      description: "Total bytes processed"
      unit: "By"
      conditions:
        - 'attributes["bytes_in"] != nil'
        - 'attributes["bytes_out"] != nil'
      sum:
        aggregation_temporality: delta
        is_monotonic: true
        value: attributes["bytes_in"] + attributes["bytes_out"]
  interval: 1m
```

**What it does**: Sums incoming and outgoing bytes using OTTL expression.

## Validation Rules

1. **Required Fields**: Must specify `type`, `extract_metric_rules`, `interval`
2. **Non-Empty Rules**: `extract_metric_rules` must contain at least one entry
3. **Metric Name**: Must follow metric naming conventions (lowercase, underscores, descriptive)
4. **Metric Type**: Each rule must have exactly one of `sum` or `gauge` (not both, not neither)
5. **Sum Configuration**: If using `sum`, must specify `aggregation_temporality`, `is_monotonic`, and `value`
6. **Gauge Configuration**: If using `gauge`, must specify `value`
7. **Valid OTTL**: All `conditions` and `value` expressions must be valid OTTL
8. **Interval**: Must be a valid duration (e.g., `1m`, `30s`, `1m30s`)
9. **Conditions**: Must be single-line OTTL (multiline not supported in YAML)

## Common Pitfalls

### 1. Forgetting Interval

**Problem**: Missing `interval` parameter causes metrics to never flush.

**Wrong**:
```yaml
- type: extract_metric
  extract_metric_rules:
    - name: "errors_total"
      # ...
```

**Correct**:
```yaml
- type: extract_metric
  extract_metric_rules:
    - name: "errors_total"
      # ...
  interval: 1m
```

### 2. Both Sum and Gauge

**Problem**: Specifying both `sum` and `gauge` is invalid.

**Wrong**:
```yaml
sum:
  aggregation_temporality: delta
  is_monotonic: true
  value: 1
gauge:
  value: attributes["value"]
```

**Correct**: Choose one based on metric type.

### 3. OTTL Syntax Errors

**Problem**: Invalid OTTL expressions cause processor to fail.

**Wrong**:
```yaml
conditions:
  - 'attributes[status] == "ERROR"'  # Missing quotes around key
```

**Correct**:
```yaml
conditions:
  - 'attributes["status"] == "ERROR"'
```

### 4. Multiline Conditions in YAML

**Problem**: YAML multiline strings don't work for OTTL conditions.

**Wrong**:
```yaml
conditions:
  - |
    attributes["status"] == "ERROR" and
    attributes["severity"] > 3
```

**Correct**:
```yaml
conditions:
  - 'attributes["status"] == "ERROR" and attributes["severity"] > 3'
```

### 5. Using Cumulative with Rate Queries

**Problem**: Using `aggregation_temporality: cumulative` when you want rates.

**Guidance**: Use `delta` for rate-style metrics (requests/sec, errors/min). Use `cumulative` only for ever-increasing totals.

### 6. Incorrect is_monotonic

**Problem**: Setting `is_monotonic: false` for counters that should only increase.

**Correct Usage**:
- `is_monotonic: true` for error counts, request counts (always increasing)
- `is_monotonic: false` for queue depth, active connections (can decrease)

### 7. Unit Formatting

**Problem**: Inconsistent unit formatting.

**Best Practices**:
- Count: `"1"`
- Milliseconds: `"ms"`
- Seconds: `"s"`
- Bytes: `"By"`
- Percent: `"%"`

Follow OpenTelemetry unit conventions.

### 8. Metric Name Conventions

**Problem**: Using spaces, capitals, or hyphens in metric names.

**Wrong**: `"Error Count"`, `"error-count"`, `"ErrorCount"`
**Correct**: `"error_count"`, `"errors_total"`

**Convention**: Use lowercase with underscores, descriptive names, `_total` suffix for counters.

## Best Practices

1. **Naming Convention**: Use `<metric>_<unit>_total` for counters (e.g., `errors_total`, `requests_total`)
2. **Descriptive Names**: Choose names that clearly indicate what is being measured
3. **Interval Selection**: Balance freshness vs overhead (1m is common, 30s for high-frequency)
4. **Cardinality**: Be cautious with high-cardinality attributes (avoid extracting unique IDs)
5. **Conditions Ordering**: Put most selective conditions first for performance
6. **Delta Temporality**: Prefer `delta` over `cumulative` for most use cases (better for rate calculations)
7. **Document Metrics**: Use `description` field to explain what each metric measures
8. **Test Conditions**: Validate OTTL conditions with sample data before deployment
9. **Final Flag**: Mark as `final: true` if this is the last processor in sequence
10. **Keep Original**: Consider if you need to keep the original log item or can drop it after extraction

## Performance Considerations

- **Condition Complexity**: Simple conditions are faster than complex regex patterns
- **Metric Count**: Each rule is evaluated for every matching item - minimize rules when possible
- **Interval**: Longer intervals reduce overhead but increase memory usage for accumulation
- **Value Expressions**: Simple attribute references are faster than complex calculations

## Production Example (from Template 1)

```yaml
- name: pii_masking_and_metrics
  type: sequence
  user_description: PII Masking and Metrics Extraction
  processors:
    - type: generic_mask
      capture_group_masks:
        - capture_group: "(?i)(password|passwd|pwd)[:=]\\S+"
          enabled: true
          mask: "***PASSWORD_REDACTED***"
          name: "password"
    - type: generic_mask
      capture_group_masks:
        - capture_group: "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"
          enabled: true
          mask: "***EMAIL***"
          name: "email"
    - type: generic_mask
      capture_group_masks:
        - capture_group: "\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b"
          enabled: true
          mask: "***CC***"
          name: "credit_card"
    - type: extract_metric
      extract_metric_rules:
        - name: "app_errors_total"
          description: "Count of application errors"
          unit: "1"
          conditions:
            - 'IsMatch(body, "(?i)(ERROR|EXCEPTION|FATAL|CRITICAL)")'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
      final: true
```

**What it does**:
1. Masks PII (passwords, emails, credit cards)
2. Extracts error counter metric
3. Flushes metrics every 1 minute
4. Final processor in sequence

**Testing Status**: ✓ Deployed and validated (Pipeline ID: 529198b9-2486-4f5c-9d38-3c2503756ce2)

## Related Processors

- **aggregate_metric**: Aggregate multiple metrics together
- **cumulative_to_delta**: Convert cumulative metrics to delta
- **log_to_pattern_metric**: Extract metrics using pattern matching
- **ottl_transform**: Transform attributes before metric extraction
- **ottl_filter**: Filter logs before metric extraction

## Cross-References

- **edgedelta-pipelines skill**: Template 1 (Log PII Masking), Template 3 (Mixed Telemetry)
- **OTTL Reference**: https://docs.edgedelta.com/ottl-statements/
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/extract-metric-processor/

## Notes

- Conditions must be written on a single line in YAML (multiline not supported)
- Use lowercase OTTL operators (`and`, `or`, `not`) - uppercase causes errors
- The processor generates new metric items and can optionally keep the original log
- OTTL conditions are evaluated for each telemetry item
- Metrics are accumulated over the `interval` period before flushing
- Both sum and gauge metrics use OTTL expressions for `value` calculation
