# tail_sample Processor

**Type**: `tail_sample`
**Category**: Sampling & Rate Limiting
**Sequence Compatible**: ✗ No (standalone processor for traces)
**Source**: `internalv3/processors/sample/tail/processor.go`
**Config Struct**: `processors.TailSampleProcessorDef`

## Quick Copy

```yaml
- type: tail_sample
  decision_interval: 30s
  batch_cache_size: 50000
  sampling_policies:
    - name: "keep_errors"
      policy_type: status_code
      status_codes: ["ERROR"]
    - name: "sample_success"
      policy_type: probabilistic
      percentage: 10
```

## Overview

The `tail_sample` processor performs intelligent trace sampling based on the complete trace outcome. Unlike head sampling (which makes decisions immediately), tail sampling waits for the entire trace to arrive before deciding whether to keep or drop it. This enables smart sampling strategies like "keep all error traces, sample 10% of successful traces" - optimizing observability costs while ensuring you never miss critical failures.

## Use Cases

- **Cost Optimization**: Reduce trace storage costs by 80-95% while maintaining visibility into errors
- **Error Prioritization**: Always keep traces with errors while sampling successful traces
- **Latency Analysis**: Keep slow traces for performance investigation while dropping fast ones
- **Selective Retention**: Sample based on specific attributes (customer tier, region, endpoint)
- **Compliance**: Always keep traces containing specific attributes or meeting certain criteria
- **Traffic-Based Sampling**: Apply different sampling rates based on trace characteristics
- **Quality Assurance**: Keep traces matching specific patterns for testing and validation
- **Adaptive Sampling**: Implement complex multi-policy strategies for different trace types

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `tail_sample` |
| `decision_interval` | duration | How long to wait before making sampling decision (e.g., `30s`, `1m`) |
| `batch_cache_size` | int | Maximum number of traces to cache before decision (default: `50000`) |
| `sampling_policies` | []Policy | List of sampling policies (evaluated in order) |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `keep_cache_size` | int | 0 (disabled) | Cache size for keeping sampled trace IDs (handles late arrivals) |
| `drop_cache_size` | int | 0 (disabled) | Cache size for dropped trace IDs (handles late arrivals) |

### Sampling Policy Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Policy name (must be unique) |
| `policy_type` | string | Yes | Policy type (see Policy Types section) |

**Policy-Specific Fields**: Each policy type requires different fields (see Policy Types section below).

## Policy Types

### 1. status_code Policy

Samples traces based on span status codes.

**Required Fields**:
- `status_codes`: Array of status codes to match (`"OK"`, `"ERROR"`, `"UNSET"`)

**Example**:
```yaml
- name: "keep_errors"
  policy_type: status_code
  status_codes: ["ERROR"]
```

**Behavior**: If ANY span in the trace has a matching status code, the trace is sampled.

### 2. probabilistic Policy

Samples traces using consistent hash-based probabilistic sampling.

**Required Fields**:
- `percentage`: Sampling percentage (0-100)

**Optional Fields**:
- `hash_salt`: Salt for hash calculation (default: `"default-hash-seed"`)

**Example**:
```yaml
- name: "sample_10_percent"
  policy_type: probabilistic
  percentage: 10
  hash_salt: "my-salt"
```

**Behavior**: Uses FNV-1a hash of trace ID to make deterministic sampling decisions.

### 3. latency Policy

Samples traces based on total trace duration.

**Required Fields** (at least one):
- `lower_threshold`: Minimum latency (e.g., `100ms`, `1s`)
- `upper_threshold`: Maximum latency (e.g., `5s`, `10s`)

**Example**:
```yaml
- name: "keep_slow_traces"
  policy_type: latency
  lower_threshold: 1s
```

**Behavior**:
- If only `lower_threshold`: Sample traces longer than threshold
- If only `upper_threshold`: Sample traces shorter than threshold
- If both: Sample traces within range (inclusive)

### 4. span_count Policy

Samples traces based on number of spans.

**Required Fields** (at least one):
- `minimum_span_count`: Minimum number of spans
- `maximum_span_count`: Maximum number of spans

**Example**:
```yaml
- name: "keep_complex_traces"
  policy_type: span_count
  minimum_span_count: 10
```

**Behavior**: Similar to latency policy, samples based on span count range.

### 5. condition Policy

Samples traces using OTTL (OpenTelemetry Transformation Language) conditions.

**Required Fields**:
- `conditions`: Array of OTTL expressions (each must return boolean)

**Example**:
```yaml
- name: "production_only"
  policy_type: condition
  conditions:
    - 'attributes["environment"] == "production"'
    - 'attributes["http.status_code"] >= 400'
```

**Behavior**: If ANY span matches ALL conditions, the trace is sampled.

### 6. numeric_attribute Policy

Samples traces based on numeric attribute values.

**Required Fields**:
- `key`: Attribute name
- At least one of:
  - `minimum_value`: Minimum value (inclusive)
  - `maximum_value`: Maximum value (inclusive)

**Example**:
```yaml
- name: "high_request_size"
  policy_type: numeric_attribute
  key: "http.request.body.size"
  minimum_value: 1000000  # 1MB
```

**Behavior**: Checks both resource and span attributes for matching values.

### 7. string_attribute Policy

Samples traces based on string attribute values (exact match or regex).

**Required Fields**:
- `key`: Attribute name
- `values`: Array of strings (exact match) or regex patterns

**Optional Fields**:
- `support_regex`: Enable regex matching (default: `false`)
- `regex_cache_size`: LRU cache size for regex results (default: `100`)

**Example**:
```yaml
- name: "specific_endpoints"
  policy_type: string_attribute
  key: "http.route"
  values: ["/api/v1/orders", "/api/v1/payments"]
```

**Example (Regex)**:
```yaml
- name: "api_endpoints"
  policy_type: string_attribute
  key: "http.route"
  values: ["^/api/.*"]
  support_regex: true
  regex_cache_size: 200
```

**Behavior**: Checks both resource and span attributes. Regex results are cached for performance.

### 8. boolean_attribute Policy

Samples traces based on boolean attribute values.

**Required Fields**:
- `key`: Attribute name
- `value`: Boolean value to match (`true` or `false`)

**Example**:
```yaml
- name: "sampled_at_source"
  policy_type: boolean_attribute
  key: "sampled"
  value: true
```

**Behavior**: Checks both resource and span attributes for matching boolean value.

### 9. and Policy

Combines multiple policies with AND logic (all must match).

**Required Fields**:
- `sub_policies`: Array of policies (all must match)

**Example**:
```yaml
- name: "production_errors"
  policy_type: and
  sub_policies:
    - name: "is_production"
      policy_type: string_attribute
      key: "environment"
      values: ["production"]
    - name: "is_error"
      policy_type: status_code
      status_codes: ["ERROR"]
```

**Behavior**: Samples trace only if ALL sub-policies match.

### 10. drop Policy

Explicitly drops traces matching sub-policies (override keep decisions).

**Required Fields**:
- `sub_policies`: Array of policies

**Example**:
```yaml
- name: "drop_health_checks"
  policy_type: drop
  sub_policies:
    - name: "is_health_check"
      policy_type: string_attribute
      key: "http.route"
      values: ["/health", "/ready"]
```

**Behavior**: If ALL sub-policies match, trace is DROPPED (even if previous policies would keep it).

## Examples

### Example 1: Basic Error + Probabilistic Sampling

```yaml
- type: tail_sample
  decision_interval: 30s
  batch_cache_size: 50000
  sampling_policies:
    - name: "keep_all_errors"
      policy_type: status_code
      status_codes: ["ERROR"]
    - name: "sample_10_percent_success"
      policy_type: probabilistic
      percentage: 10
```

**What it does**: Keeps 100% of error traces, samples 10% of successful traces.

### Example 2: Latency-Based Sampling

```yaml
- type: tail_sample
  decision_interval: 30s
  batch_cache_size: 50000
  sampling_policies:
    - name: "keep_slow_traces"
      policy_type: latency
      lower_threshold: 2s
    - name: "keep_very_fast_traces"
      policy_type: latency
      upper_threshold: 50ms
    - name: "sample_normal_traces"
      policy_type: probabilistic
      percentage: 5
```

**What it does**: Keeps traces slower than 2s or faster than 50ms (unusual cases), samples 5% of normal traces.

### Example 3: Multi-Policy with AND Logic

```yaml
- type: tail_sample
  decision_interval: 30s
  batch_cache_size: 50000
  sampling_policies:
    - name: "production_errors"
      policy_type: and
      sub_policies:
        - name: "is_production"
          policy_type: string_attribute
          key: "environment"
          values: ["production"]
        - name: "is_error"
          policy_type: status_code
          status_codes: ["ERROR"]
    - name: "sample_others"
      policy_type: probabilistic
      percentage: 1
```

**What it does**: Keeps all production errors, samples 1% of everything else.

### Example 4: Status Code Priority Sampling

```yaml
- type: tail_sample
  decision_interval: 30s
  batch_cache_size: 50000
  sampling_policies:
    - name: "keep_errors"
      policy_type: status_code
      status_codes: ["ERROR"]
    - name: "keep_unset_status"
      policy_type: status_code
      status_codes: ["UNSET"]
    - name: "sample_success"
      policy_type: probabilistic
      percentage: 5
```

**What it does**: Keeps errors and unset status traces, samples 5% of OK traces.

### Example 5: Attribute-Based Filtering

```yaml
- type: tail_sample
  decision_interval: 30s
  batch_cache_size: 50000
  sampling_policies:
    - name: "premium_customers"
      policy_type: string_attribute
      key: "customer.tier"
      values: ["premium", "enterprise"]
    - name: "high_value_transactions"
      policy_type: numeric_attribute
      key: "transaction.amount"
      minimum_value: 1000
    - name: "sample_others"
      policy_type: probabilistic
      percentage: 2
```

**What it does**: Keeps all premium customer traces and high-value transactions, samples 2% of others.

### Example 6: Cache Configuration for Late Arrivals

```yaml
- type: tail_sample
  decision_interval: 30s
  batch_cache_size: 50000
  keep_cache_size: 10000
  drop_cache_size: 50000
  sampling_policies:
    - name: "keep_errors"
      policy_type: status_code
      status_codes: ["ERROR"]
    - name: "sample_success"
      policy_type: probabilistic
      percentage: 10
```

**What it does**: Same as Example 1, but caches decisions for late-arriving spans:
- `keep_cache_size`: Remembers 10,000 sampled trace IDs
- `drop_cache_size`: Remembers 50,000 dropped trace IDs

### Example 7: Fine-Tuned Decision Interval

```yaml
- type: tail_sample
  decision_interval: 10s
  batch_cache_size: 100000
  sampling_policies:
    - name: "keep_errors"
      policy_type: status_code
      status_codes: ["ERROR"]
    - name: "sample_success"
      policy_type: probabilistic
      percentage: 15
```

**What it does**: Faster decision interval (10s instead of 30s) for lower latency, larger cache for higher throughput.

### Example 8: Complex Policy Combination

```yaml
- type: tail_sample
  decision_interval: 30s
  batch_cache_size: 50000
  sampling_policies:
    # Always drop health checks
    - name: "drop_health_checks"
      policy_type: drop
      sub_policies:
        - name: "is_health_check"
          policy_type: string_attribute
          key: "http.route"
          values: ["/health", "/ready", "/ping"]
    # Keep all errors
    - name: "keep_errors"
      policy_type: status_code
      status_codes: ["ERROR"]
    # Keep slow traces
    - name: "keep_slow"
      policy_type: latency
      lower_threshold: 1s
    # Keep complex traces
    - name: "keep_complex"
      policy_type: span_count
      minimum_span_count: 20
    # Production errors with OTTL
    - name: "production_issues"
      policy_type: condition
      conditions:
        - 'attributes["environment"] == "production"'
        - 'attributes["http.status_code"] >= 500'
    # Sample remaining 5%
    - name: "sample_rest"
      policy_type: probabilistic
      percentage: 5
```

**What it does**: Comprehensive sampling strategy with multiple priorities.

### Example 9: Regex-Based Endpoint Filtering

```yaml
- type: tail_sample
  decision_interval: 30s
  batch_cache_size: 50000
  sampling_policies:
    - name: "api_endpoints"
      policy_type: string_attribute
      key: "http.route"
      values: ["^/api/v[0-9]+/.*"]
      support_regex: true
      regex_cache_size: 500
    - name: "sample_others"
      policy_type: probabilistic
      percentage: 10
```

**What it does**: Keeps all API endpoint traces (using regex), samples 10% of others.

### Example 10: Boolean Attribute Sampling

```yaml
- type: tail_sample
  decision_interval: 30s
  batch_cache_size: 50000
  sampling_policies:
    - name: "keep_flagged"
      policy_type: boolean_attribute
      key: "requires_analysis"
      value: true
    - name: "keep_errors"
      policy_type: status_code
      status_codes: ["ERROR"]
    - name: "sample_normal"
      policy_type: probabilistic
      percentage: 3
```

**What it does**: Keeps traces flagged for analysis, all errors, and 3% of normal traffic.

## Validation Rules

1. **Required Fields**: Must specify `type`, `decision_interval`, `batch_cache_size`, `sampling_policies`
2. **Decision Interval**: Must be positive duration (e.g., `10s`, `30s`, `1m`)
3. **Batch Cache Size**: Must be positive integer (recommended: 50,000+)
4. **Policy Names**: Must be unique within processor
5. **Policy Types**: Must be one of the 10 valid types
6. **Status Codes**: Only `"OK"`, `"ERROR"`, `"UNSET"` allowed
7. **Percentage**: Must be 0-100 for probabilistic policies
8. **Thresholds**: Latency and numeric attribute thresholds must be valid values
9. **OTTL Conditions**: Must be valid OTTL expressions returning boolean
10. **Sub-Policies**: AND and DROP policies must have at least one sub-policy
11. **Cache Sizes**: If specified, must be positive integers
12. **Regex Patterns**: String attribute regex patterns must be valid if `support_regex: true`

## Common Pitfalls

### 1. Decision Interval Too Short

**Problem**: Very short intervals increase memory and CPU usage.

**Wrong**:
```yaml
decision_interval: 1s  # Too short for most use cases
```

**Correct**:
```yaml
decision_interval: 30s  # Good default, allows complete traces to arrive
```

**Guidance**: Use 30s as default. Increase to 60s+ for distributed systems with high latency.

### 2. Decision Interval Too Long

**Problem**: Long intervals increase memory usage and delay trace availability.

**Wrong**:
```yaml
decision_interval: 5m  # Unnecessarily long
batch_cache_size: 10000  # Too small for long interval
```

**Correct**:
```yaml
decision_interval: 30s
batch_cache_size: 50000
```

**Guidance**: Match cache size to expected trace volume during decision interval.

### 3. Policy Ordering Matters

**Problem**: Policies are evaluated in order - first match wins.

**Wrong** (Unreachable policy):
```yaml
sampling_policies:
  - name: "sample_all"
    policy_type: probabilistic
    percentage: 100
  - name: "keep_errors"  # Never evaluated!
    policy_type: status_code
    status_codes: ["ERROR"]
```

**Correct**:
```yaml
sampling_policies:
  - name: "keep_errors"  # Specific policies first
    policy_type: status_code
    status_codes: ["ERROR"]
  - name: "sample_rest"  # Catch-all last
    policy_type: probabilistic
    percentage: 10
```

### 4. Cache Sizing for High Throughput

**Problem**: Insufficient cache size causes early eviction of pending traces.

**Wrong**:
```yaml
batch_cache_size: 1000  # Too small for high throughput
```

**Correct**:
```yaml
batch_cache_size: 100000  # Sized for expected throughput
keep_cache_size: 20000
drop_cache_size: 100000
```

**Guidance**: Calculate: `(traces_per_second * decision_interval) * 2` for safety margin.

### 5. Missing Trace IDs

**Problem**: Processor only works with traces (has trace IDs).

**Wrong**: Trying to use with logs or metrics.

**Correct**: Only use with trace data from OTLP sources or trace inputs.

### 6. Probabilistic Sampling Percentage

**Problem**: Misunderstanding percentage values.

**Wrong**:
```yaml
percentage: 0.1  # Means 0.1%, not 10%
```

**Correct**:
```yaml
percentage: 10  # Means 10%
```

### 7. Forgetting Drop Policy Override

**Problem**: Not understanding that DROP policy overrides previous KEEP decisions.

**Wrong** (Misunderstanding behavior):
```yaml
sampling_policies:
  - name: "keep_all"
    policy_type: probabilistic
    percentage: 100
  - name: "drop_health"  # This WILL drop health checks!
    policy_type: drop
    sub_policies:
      - name: "health_check"
        policy_type: string_attribute
        key: "http.route"
        values: ["/health"]
```

**Correct** (Put DROP policies first if you want absolute drops):
```yaml
sampling_policies:
  - name: "drop_health"  # Evaluated first
    policy_type: drop
    sub_policies:
      - name: "health_check"
        policy_type: string_attribute
        key: "http.route"
        values: ["/health"]
  - name: "keep_rest"
    policy_type: probabilistic
    percentage: 100
```

### 8. AND Policy with Incompatible Sub-Policies

**Problem**: AND policy requires ALL sub-policies to match.

**Wrong** (Too restrictive):
```yaml
policy_type: and
sub_policies:
  - name: "is_error"
    policy_type: status_code
    status_codes: ["ERROR"]
  - name: "is_success"
    policy_type: status_code
    status_codes: ["OK"]  # Can't be both ERROR and OK!
```

**Correct**:
```yaml
policy_type: and
sub_policies:
  - name: "is_error"
    policy_type: status_code
    status_codes: ["ERROR"]
  - name: "is_production"
    policy_type: string_attribute
    key: "environment"
    values: ["production"]
```

### 9. Not Enabling Late Arrival Caches

**Problem**: Late-arriving spans after decision create incomplete traces.

**Wrong** (No late arrival handling):
```yaml
decision_interval: 10s  # Short interval
# No keep_cache_size or drop_cache_size
```

**Correct**:
```yaml
decision_interval: 10s
keep_cache_size: 10000  # Remember sampled traces
drop_cache_size: 50000  # Remember dropped traces
```

### 10. Regex Without Cache

**Problem**: Regex evaluation is expensive without caching.

**Wrong**:
```yaml
policy_type: string_attribute
key: "http.route"
values: ["^/api/.*", "^/web/.*"]
support_regex: true
# Missing regex_cache_size
```

**Correct**:
```yaml
policy_type: string_attribute
key: "http.route"
values: ["^/api/.*", "^/web/.*"]
support_regex: true
regex_cache_size: 500  # Cache regex results
```

## Best Practices

1. **Policy Ordering**: Place specific policies (errors, high-priority) before catch-all policies (probabilistic)
2. **Decision Interval**: Use 30s as default; increase for distributed systems with network latency
3. **Cache Sizing**: Size batch_cache_size to handle expected throughput during decision interval
4. **Late Arrivals**: Enable keep_cache_size and drop_cache_size in distributed systems
5. **Always Keep Errors**: First policy should almost always keep error traces
6. **Probabilistic Last**: Use probabilistic policy as final catch-all for remaining traces
7. **Drop Health Checks**: Use DROP policy early to prevent health check noise
8. **AND for Specificity**: Use AND policy to combine multiple conditions for precise filtering
9. **Monitor Cache Eviction**: Track metrics to ensure caches are adequately sized
10. **Test Sampling Rates**: Validate that percentage values achieve desired cost/visibility balance
11. **Use Regex Cache**: Always set regex_cache_size when using regex patterns
12. **Document Policies**: Use descriptive policy names to explain sampling strategy
13. **Review Policy Order**: Periodically audit that policies are evaluated in optimal order
14. **Balance Cost vs Visibility**: Aim for 5-20% sampling on success traces after keeping all errors
15. **Production Validation**: Test sampling configuration in staging before production deployment

## Performance Considerations

### Decision Interval Impact

- **Short intervals (5-10s)**:
  - Pro: Lower memory usage, faster trace availability
  - Con: Higher CPU usage, may miss late spans
  - Use when: Low latency systems, predictable span arrival

- **Medium intervals (30-60s)**:
  - Pro: Balanced performance, accommodates most distributed systems
  - Con: Moderate memory usage
  - Use when: Standard microservices, typical production environments

- **Long intervals (60s+)**:
  - Pro: Accommodates slow backends, complex traces
  - Con: High memory usage, delayed trace availability
  - Use when: Multi-region systems, legacy integrations

### Cache Size Optimization

**Batch Cache Size**:
- Formula: `(traces/second × decision_interval) × 2`
- Example: 1000 traces/sec × 30s × 2 = 60,000
- Trades memory for data retention

**Keep/Drop Cache Sizes**:
- Keep cache: Smaller (10-20K) - fewer sampled traces
- Drop cache: Larger (50-100K) - more dropped traces
- Only needed if late arrivals are common

### Policy Evaluation Performance

- **Fastest**: status_code, probabilistic (O(1) operations)
- **Fast**: string_attribute (exact match), boolean_attribute
- **Medium**: numeric_attribute, span_count, latency
- **Slower**: string_attribute (regex), condition (OTTL)
- **Slowest**: and, drop (evaluate multiple sub-policies)

**Optimization**: Place faster policies first to short-circuit evaluation.

### Memory Usage

- Each cached trace: ~1-10KB depending on span count
- Batch cache: `batch_cache_size × average_trace_size`
- Keep/drop caches: Minimal (only store trace IDs)
- Example: 50K batch cache × 5KB = ~250MB

## Tail Sampling vs Probabilistic Sampling

### Head/Probabilistic Sampling

**When decision is made**: Immediately when trace starts

**Pros**:
- Simple, low latency
- Minimal memory overhead
- Deterministic based on trace ID

**Cons**:
- Can't consider trace outcome
- May drop important error traces
- Same sampling rate for all traces

**Use when**: Simple sampling needs, memory constrained

### Tail Sampling

**When decision is made**: After entire trace arrives

**Pros**:
- Smart sampling based on outcome
- Always keeps errors, slow traces
- Different rates for different trace types
- Optimizes cost/visibility balance

**Cons**:
- Higher memory usage
- Adds latency (decision_interval)
- More complex configuration

**Use when**: Production systems where errors must be kept, cost optimization critical

### Comparison Example

**Scenario**: 10,000 traces/minute, 1% are errors

**Head Sampling (10%)**:
- Keeps: 1,000 traces (including ~10 errors)
- Drops: 9,000 traces (including ~90 errors)
- **Result**: Miss 90% of errors!

**Tail Sampling (keep errors + 5% success)**:
- Keeps: 100 error traces + 495 success traces = 595 traces
- Drops: 9,405 success traces
- **Result**: 0% errors missed, 40% less volume than head sampling

## Related Processors

- **probabilistic_sample**: Head-based probabilistic sampling (simpler, lower latency)
- **ottl_filter**: Filter traces before tail sampling (reduce processing)
- **ottl_transform**: Transform trace attributes for policy evaluation
- **batch**: Batch traces for more efficient processing
- **attribute**: Add/modify attributes used in sampling policies

## Cross-References

- **edgedelta-pipelines skill**: Use with trace processing pipelines
- **OTTL Reference**: For condition policy expressions
- **Trace Collection**: Ensure trace inputs properly configured for tail sampling

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/tail-sample-processor/

## Notes

- Tail sampling only works with **trace data** (must have trace IDs)
- Policies are evaluated **in order** - first matching policy determines outcome
- **DROP policy** overrides previous KEEP decisions (allows explicit exclusions)
- **AND policy** requires ALL sub-policies to match (stricter than default OR behavior)
- Decision interval should account for slowest backend in distributed system
- Cache sizes should be tuned based on throughput and decision interval
- Late arrival caches (keep_cache_size, drop_cache_size) handle spans arriving after decision
- Probabilistic sampling uses FNV-1a hash for deterministic sampling across fleet
- String attribute regex matching is cached for performance (configurable cache size)
- Status codes are normalized (both `"OK"` and `"STATUS_CODE_OK"` accepted)
- OTTL condition evaluation happens per-span (if ANY span matches, trace is sampled)
- Memory usage scales with batch_cache_size and average trace complexity
- Processor metrics track sampled vs dropped traces for monitoring effectiveness
- Combining multiple policies enables sophisticated sampling strategies (e.g., "keep all production errors over 1s")
- Always test sampling configuration with production-representative traffic before deployment
