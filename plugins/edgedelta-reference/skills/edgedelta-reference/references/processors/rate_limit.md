# rate_limit Processor

**Type**: `rate_limit`
**Category**: Sampling & Rate Limiting
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/rate/limit/processor.go`
**Config Struct**: `configv3.RateLimit`

## Quick Copy

```yaml
- type: rate_limit
  item_count_limit: 1000
  interval: 30s
```

## Overview

The `rate_limit` processor controls throughput by capping the maximum number of items (logs, metrics, or traces) that can pass through within a fixed time interval. It uses a simple counter-based approach that resets periodically, making it ideal for protecting downstream systems, controlling egress costs, and preventing pipeline overload. Unlike sampling which is probabilistic, rate limiting provides deterministic volume control.

## Use Cases

- **Downstream System Protection**: Prevent overwhelming downstream endpoints with excessive throughput
- **Cost Control**: Cap egress data volume to external services with per-GB pricing
- **Burst Protection**: Smooth out traffic spikes by limiting maximum items per time window
- **Pipeline Capacity Management**: Ensure processing stays within configured capacity limits
- **Testing & Development**: Limit data volume during testing without changing source configurations
- **Fair Resource Allocation**: Ensure no single source monopolizes pipeline capacity
- **SLA Compliance**: Enforce throughput limits mandated by downstream service SLAs
- **Budget Management**: Prevent unexpected costs from log/metric volume spikes

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `rate_limit` |
| `item_count_limit` | int | Maximum number of items to allow per interval (must be > 0) |
| `interval` | duration | Time window for rate limit reset (e.g., `30s`, `1m`, `5m`) |

### Optional Parameters (from SequenceProcessor)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `final` | bool | false | Mark as last processor in sequence (required for final processor) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## How Rate Limiting Works

The rate_limit processor uses a simple token bucket algorithm implementation:

1. **Initialization**: On startup, the processor creates a counter initialized to 0
2. **Interval Timer**: A background ticker resets the counter to 0 every `interval` period
3. **Item Processing**: For each item that arrives:
   - Increment the counter atomically
   - If new counter value ≤ `item_count_limit`: **Pass through** (allow item to continue)
   - If new counter value > `item_count_limit`: **Drop** (terminate item processing)
4. **Interval Reset**: Every `interval`, the counter resets to 0, allowing the next burst of items

**Key Characteristics**:
- **Stateful**: Maintains in-memory counter per processor instance
- **Per-Agent**: Each agent instance has its own independent counter (not distributed)
- **Deterministic**: First N items pass, remaining items drop (not probabilistic)
- **Hard Limit**: Once limit is reached, all items are dropped until interval resets
- **Atomic Operations**: Thread-safe counter updates using atomic operations

**Default Values**:
- Default `interval`: 30 seconds (if not specified in config)
- Default `item_count_limit`: 10 (if not specified in config)
- These defaults are typically overridden in production configurations

## Examples

### Example 1: Basic Rate Limiting (Items Per Second)

```yaml
- type: rate_limit
  item_count_limit: 1000
  interval: 1s
  final: true
```

**What it does**: Allows maximum 1,000 items per second. Once 1,000 items pass in the current second, all additional items are dropped until the next second begins.

**Use case**: Cap throughput at 1,000 items/sec to protect downstream systems.

### Example 2: Protect Downstream API Endpoint

```yaml
- type: rate_limit
  item_count_limit: 5000
  interval: 1m
  comment: "Downstream API limited to 5k requests/min per SLA"
```

**What it does**: Enforces maximum 5,000 items per minute, aligning with downstream service level agreement.

**Use case**: Prevent violating downstream API rate limits that could cause service degradation or blocking.

### Example 3: Cost Control for Egress

```yaml
- type: rate_limit
  item_count_limit: 100000
  interval: 1h
  comment: "Budget cap: max 100k logs/hour to external SIEM"
```

**What it does**: Limits to 100,000 logs per hour (~2.4M logs/day), controlling data egress costs.

**Use case**: Prevent unexpected billing from log volume spikes when sending to commercial SIEM/logging platforms.

### Example 4: Burst Handling with Larger Interval

```yaml
- type: rate_limit
  item_count_limit: 50000
  interval: 5m
```

**What it does**: Allows bursts up to 50,000 items within any 5-minute window, then drops excess.

**Use case**: Smooth out traffic spikes while allowing normal burst patterns without overwhelming pipeline.

### Example 5: Per-Source Rate Limiting (Conditional)

```yaml
- type: rate_limit
  condition: 'resource["service.name"] == "chatty-service"'
  item_count_limit: 500
  interval: 30s
```

**What it does**: Applies rate limit only to items from "chatty-service", allowing 500 items per 30 seconds.

**Use case**: Limit specific noisy services without affecting other services in the pipeline.

### Example 6: Conditional Rate Limiting (Non-Production)

```yaml
- type: rate_limit
  condition: 'attributes["environment"] != "production"'
  item_count_limit: 1000
  interval: 1m
  comment: "Limit non-prod environments to reduce costs"
```

**What it does**: Applies rate limit only to non-production logs, allowing unlimited production volume.

**Use case**: Reduce costs from development/staging environments while preserving full production visibility.

### Example 7: High-Volume Log Control

```yaml
- type: rate_limit
  item_count_limit: 10000
  interval: 10s
  comment: "Cap at ~1k/sec sustained throughput"
```

**What it does**: Allows 10,000 items per 10-second window (effective rate ~1,000/sec).

**Use case**: Control high-volume log streams with finer-grained interval control than 1-minute windows.

### Example 8: Combined with Sampling

```yaml
- type: sample
  percentage: 50
  comment: "First reduce volume by 50%"
- type: rate_limit
  item_count_limit: 5000
  interval: 1m
  comment: "Then cap at 5k/min absolute maximum"
```

**What it does**: First applies 50% probabilistic sampling, then enforces hard cap of 5,000 items/minute.

**Use case**: Two-stage volume control - sampling for statistical reduction, rate limiting for absolute cap.

### Example 9: Testing & Development Limit

```yaml
- type: rate_limit
  item_count_limit: 100
  interval: 1m
  comment: "Dev/test: limit to 100 logs/min for debugging"
```

**What it does**: Severely limits throughput to 100 items/minute for development testing.

**Use case**: Reduce data volume during pipeline development and testing without modifying sources.

### Example 10: Fair Resource Allocation

```yaml
- name: limit_per_host
  type: sequence
  processors:
    - type: rate_limit
      condition: 'resource["host.name"] == "host-1"'
      item_count_limit: 1000
      interval: 1m
    - type: rate_limit
      condition: 'resource["host.name"] == "host-2"'
      item_count_limit: 1000
      interval: 1m
    - type: rate_limit
      condition: 'resource["host.name"] == "host-3"'
      item_count_limit: 1000
      interval: 1m
      final: true
```

**What it does**: Applies separate rate limits per host, ensuring each host gets fair share of pipeline capacity.

**Use case**: Prevent single chatty host from consuming all pipeline capacity.

### Example 11: Metric Rate Limiting

```yaml
- type: rate_limit
  data_types: [metric]
  item_count_limit: 50000
  interval: 1m
  comment: "Limit metrics to 50k/min (~833/sec)"
```

**What it does**: Applies rate limit only to metric data types, allowing 50,000 metrics per minute.

**Use case**: Control metric cardinality explosion or protect metric storage backend.

### Example 12: Trace Rate Limiting

```yaml
- type: rate_limit
  data_types: [trace]
  item_count_limit: 10000
  interval: 30s
  comment: "Cap traces at 10k per 30sec (~333/sec)"
```

**What it does**: Limits trace spans to 10,000 per 30 seconds, protecting trace backend.

**Use case**: Prevent trace backend overload from high-volume distributed tracing.

## Validation Rules

1. **Required Fields**: Must specify `type`, `item_count_limit`, `interval`
2. **Positive Limit**: `item_count_limit` must be greater than 0
3. **Positive Interval**: `interval` must be greater than 0 (valid duration)
4. **Valid Duration**: `interval` must be parseable as Go duration (e.g., `30s`, `1m`, `5m`)
5. **Condition Syntax**: If `condition` is specified, must be valid OTTL expression
6. **Type Value**: `type` must be exactly `rate_limit`

## Common Pitfalls

### 1. Rate Too Low (Data Loss)

**Problem**: Setting `item_count_limit` too low causes excessive data loss.

**Wrong**:
```yaml
- type: rate_limit
  item_count_limit: 10    # Way too low for production
  interval: 1m
```

**Impact**: Only 10 items per minute pass through, likely losing 99%+ of data.

**Correct**:
```yaml
- type: rate_limit
  item_count_limit: 10000  # Reasonable for moderate volume
  interval: 1m
```

**Guidance**: Calculate expected throughput and set limit 20-50% above normal peak to handle bursts.

### 2. Rate Too High (No Protection)

**Problem**: Setting `item_count_limit` too high provides no effective rate limiting.

**Wrong**:
```yaml
- type: rate_limit
  item_count_limit: 999999999  # Effectively unlimited
  interval: 1s
```

**Impact**: Rate limit never triggers, provides no protection against spikes.

**Correct**:
```yaml
- type: rate_limit
  item_count_limit: 5000  # Based on actual downstream capacity
  interval: 1s
```

**Guidance**: Base limit on downstream system capacity, SLA requirements, or budget constraints.

### 3. Burst Configuration Mismatch

**Problem**: Interval too short doesn't allow reasonable bursts; too long allows excessive bursts.

**Wrong (Too Short)**:
```yaml
- type: rate_limit
  item_count_limit: 10
  interval: 1s  # Forces very smooth rate, no burst tolerance
```

**Wrong (Too Long)**:
```yaml
- type: rate_limit
  item_count_limit: 1000000
  interval: 1h  # Allows massive hour-long burst
```

**Correct**:
```yaml
- type: rate_limit
  item_count_limit: 5000
  interval: 1m  # Allows reasonable 1-minute bursts
```

**Guidance**: Use 30s-5m intervals for most use cases. Shorter for strict rate control, longer for burst tolerance.

### 4. Distributed Rate Limiting Assumptions

**Problem**: Assuming rate limit is enforced globally across all agents.

**Reality**: Each agent instance has its own independent counter.

**Example**:
- You have 3 agents with `item_count_limit: 1000, interval: 1m`
- Effective total throughput: ~3,000 items/minute (3 × 1,000)
- NOT 1,000 items/minute total

**Solution**: Adjust `item_count_limit` based on number of agents:
```yaml
# If you have 5 agents and want 5k/min total:
- type: rate_limit
  item_count_limit: 1000  # 5 agents × 1000 = 5000 total
  interval: 1m
```

### 5. Missing Interval

**Problem**: Forgetting to set `interval` relies on default (30s), which may not match intent.

**Wrong**:
```yaml
- type: rate_limit
  item_count_limit: 1000
  # Missing interval - defaults to 30s
```

**Correct**:
```yaml
- type: rate_limit
  item_count_limit: 1000
  interval: 1m  # Explicit interval
```

**Guidance**: Always explicitly set `interval` for clarity and to avoid unexpected behavior.

### 6. Rate Limit vs Sample Confusion

**Problem**: Using rate_limit when sampling would be more appropriate, or vice versa.

**Rate Limit**: Deterministic hard cap (first N items pass, rest drop)
**Sample**: Probabilistic reduction (random selection across all items)

**Wrong Use of Rate Limit**:
```yaml
# Trying to get representative sample of all data
- type: rate_limit
  item_count_limit: 100
  interval: 1m
```
**Issue**: Only captures first 100 items/minute, misses data from later in the interval.

**Should Use Sample**:
```yaml
- type: sample
  percentage: 10  # Random 10% sample across entire time range
```

**When to Use Rate Limit**:
- Protecting downstream capacity
- Enforcing SLA limits
- Budget control
- Absolute throughput cap

**When to Use Sample**:
- Statistical data reduction
- Representative sampling
- General volume reduction
- Preserving temporal distribution

### 7. No Monitoring of Dropped Items

**Problem**: Rate limiting drops data silently without visibility into how much is being dropped.

**Recommendation**: Monitor `pipeline_node_miss` metric for the rate_limit processor:
```yaml
# In observability backend, alert on:
# rate(pipeline_node_miss{node_type="rate_limit"}[5m]) > threshold
```

**Guidance**: Set alerts when drop rate exceeds acceptable threshold to detect capacity issues.

### 8. Combining Multiple Rate Limits

**Problem**: Multiple rate_limit processors in sequence create confusing combined behavior.

**Wrong**:
```yaml
- type: rate_limit
  item_count_limit: 1000
  interval: 1m
- type: rate_limit
  item_count_limit: 500
  interval: 30s
```

**Impact**: Second rate limit (500/30s) is more restrictive, making first rate limit redundant.

**Correct**: Use single rate_limit with appropriate parameters:
```yaml
- type: rate_limit
  item_count_limit: 500
  interval: 30s  # Most restrictive limit
```

## Best Practices

1. **Calculate Throughput**: Measure actual throughput before setting limits (use metrics/monitoring)
2. **Set Explicit Intervals**: Always specify `interval` explicitly, don't rely on defaults
3. **Allow Burst Headroom**: Set limit 20-50% above normal peak to handle legitimate bursts
4. **Monitor Drop Rates**: Track `pipeline_node_miss` metric to understand data loss
5. **Document Reasoning**: Use `comment` field to explain why limit is set at specific value
6. **Align with SLAs**: Base limits on documented downstream service SLAs and capacity
7. **Test Before Production**: Validate limits with production-like traffic in staging
8. **Use Conditions Selectively**: Apply rate limits to specific sources/environments when possible
9. **Combine with Sampling**: Use sampling first for probabilistic reduction, rate limiting for hard cap
10. **Per-Agent Calculations**: Account for number of agent instances when setting limits
11. **Regular Review**: Periodically review limits as traffic patterns and downstream capacity change
12. **Alert on Threshold**: Set alerts when drop rate exceeds acceptable percentage (e.g., >10%)
13. **Consider Alternatives**: Evaluate if sampling, suppress, or dedup would better meet requirements
14. **Interval Selection**: Use 30s-5m intervals for most use cases (balance burst tolerance vs smoothing)
15. **Cost-Based Limits**: For cost control, calculate `item_count_limit` based on budget and pricing

## Performance Considerations

### Processing Overhead

- **Per Item**: Atomic counter increment + comparison (extremely fast, ~nanoseconds)
- **CPU Impact**: Negligible (< 0.1% CPU overhead in typical scenarios)
- **Memory Usage**: Minimal (~100 bytes for counter + processor state)
- **Lock-Free**: Uses atomic operations, no mutex contention

### Counter Reset

- **Background Ticker**: Runs goroutine with ticker that resets counter every `interval`
- **Ticker Overhead**: Minimal (one goroutine per rate_limit processor)
- **Reset Atomicity**: Counter reset is atomic, no items lost during reset
- **Interval Accuracy**: Reset timing accurate to ~millisecond precision

### Scalability

- **High Throughput**: Can handle millions of items per second with minimal overhead
- **State Size**: Fixed size (single counter), no growth over time
- **No Cache Cleanup**: Unlike suppress/dedup, no periodic cache cleanup needed
- **Thread Safety**: Atomic operations ensure thread-safe operation at high concurrency

### Interval Selection Impact

- **Short Intervals** (< 30s): More frequent resets, smoother rate limiting, less burst tolerance
- **Medium Intervals** (30s-5m): Balanced burst tolerance and rate control (recommended)
- **Long Intervals** (> 5m): Higher burst tolerance, but long gaps if limit hit early

## Rate Limit Algorithm Details

The processor uses a simple **fixed window counter** algorithm:

```
Time:     0s    30s    60s    90s
          |------|------|------|
Counter:  0→1000 0→850  0→1200 0→990
          [Pass] [Pass] [Drop>1000] [Pass]
```

**Algorithm Properties**:
- **Fixed Windows**: Counter resets exactly every `interval` (30s, 1m, etc.)
- **No Sliding Window**: Uses fixed boundaries, not sliding/rolling window
- **Boundary Bursts**: Items at window boundary get full new limit (allows 2× burst at boundary)
- **No Token Accumulation**: Unused capacity does NOT carry over to next interval

**Example Boundary Burst**:
```
Interval: 1m, Limit: 1000/min
- 59s: 500 items arrive (under limit, all pass)
- 60s: Reset counter to 0
- 61s: 1500 items arrive (first 1000 pass, 500 drop)
Total in 2 seconds: 1500 items (exceeds 1000/min rate)
```

**Alternative Algorithm**: EdgeDelta does NOT use token bucket with refill rate. It's simpler fixed window counter.

## Rate Limit vs Sample Comparison

| Aspect | Rate Limit Processor | Sample Processor |
|--------|---------------------|------------------|
| **Purpose** | Enforce absolute throughput cap | Reduce volume probabilistically |
| **Behavior** | Drops items after limit reached | Randomly selects items to keep |
| **Determinism** | Deterministic (first N pass) | Probabilistic (random selection) |
| **Temporal Bias** | Biased to early items in interval | Uniform distribution over time |
| **State** | Simple counter | No state (stateless) |
| **Configuration** | `item_count_limit`, `interval` | `percentage` |
| **Use Case** | Protect capacity, enforce SLAs | General volume reduction |
| **Data Distribution** | Skewed to interval start | Representative sample |
| **Burst Handling** | Allows bursts up to limit | Smooths bursts naturally |
| **Cost Predictability** | Precise cost control | Approximate cost reduction |
| **Downstream Protection** | Hard protection guarantee | Probabilistic reduction |

**When to Use Rate Limit**:
- Downstream system has hard throughput limits (API rate limits, SLAs)
- Need precise cost control (exactly N items per time period)
- Want to guarantee maximum throughput cap
- Protecting capacity-constrained systems

**When to Use Sample**:
- Need representative data sample across entire time range
- Reducing volume for storage/cost without strict requirements
- Want uniform distribution of sampled data
- No downstream capacity constraints, just general volume reduction

**Can Use Both**:
```yaml
- type: sample
  percentage: 50        # Reduce by 50% probabilistically
- type: rate_limit
  item_count_limit: 5000  # Hard cap at 5k/min
  interval: 1m
```

This combination provides:
1. Probabilistic reduction for general volume (sample)
2. Absolute safety cap for protection (rate_limit)

## Related Processors

- **sample**: Probabilistic sampling for volume reduction (stateless, random selection)
- **suppress**: Key-based rate limiting (allows N occurrences per key per interval)
- **dedup**: Combine duplicate events (different purpose, but also reduces volume)
- **ottl_filter**: Conditional filtering (drops based on criteria, not rate)
- **tail_sampling**: Trace-aware sampling (for distributed tracing)

## Cross-References

- **edgedelta-pipelines skill**: Use rate_limit in cost-control and protection pipelines
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`
- **Sampling Strategy**: Compare with sampling processors for volume reduction strategies

## Documentation References

See https://docs.edgedelta.com for rate_limit processor documentation.

## Notes

- Counter is reset exactly every `interval` using a background ticker goroutine
- Default `interval` is 30 seconds if not specified (always set explicitly)
- Default `item_count_limit` is 10 if not specified (always set explicitly)
- Each agent instance maintains its own independent counter (not distributed)
- Counter uses atomic operations (`atomic.Int64`) for thread safety
- When counter exceeds limit, processor returns `TerminateLogProcessResult` (drops item)
- When counter is within limit, processor returns `PassThroughLogProcessResult` (allows item)
- Processor tracks success/miss metrics via `pipeline_node_success` and `pipeline_node_miss`
- Uses fixed window counter algorithm (not sliding window, not token bucket)
- No carryover of unused capacity between intervals
- Processor starts background goroutine for interval ticker (stopped on processor stop)
- Error logs from processor are rate limited internally to prevent log flooding
- Works with all data types (logs, metrics, traces) unless `data_types` filter is applied
- Can be combined with `condition` for conditional rate limiting
- No state persistence - counter resets to 0 on agent restart
- Boundary condition: items arriving exactly at interval boundary get new limit
- Items are counted as they arrive, in order (FIFO-style counting)
