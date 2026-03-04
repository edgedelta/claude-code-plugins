# log_to_pattern_metric Processor

**Type**: `log_to_pattern_metric`
**Category**: Pattern Recognition & Metrics Generation
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/logtopattern/processor.go`
**Config Struct**: `configv3.LogToPattern`

## Quick Copy

```yaml
- type: log_to_pattern_metric
  num_of_clusters: 15
  samples_per_cluster: 1
  reporting_frequency: 1m
  final: true
```

## Overview

The `log_to_pattern_metric` processor uses the DRAIN algorithm to automatically cluster log messages into patterns and generate metrics about pattern frequency. DRAIN is a tree-based clustering algorithm that efficiently groups similar log messages by identifying variable tokens (like timestamps, IDs, IPs) and creating pattern templates with wildcards. This processor enables powerful log analytics including anomaly detection, trend analysis, and cost reduction by converting high-cardinality logs into low-cardinality metrics.

## Use Cases

- **Automatic Pattern Discovery**: Identify recurring log patterns without manual rule creation
- **Anomaly Detection**: Detect new or rare patterns that may indicate issues
- **Log Volume Reduction**: Convert high-volume logs to compact pattern metrics
- **Trend Analysis**: Track pattern frequency changes over time to identify degradation
- **Debugging Support**: Retain sample logs for each pattern to enable root cause analysis
- **Cost Optimization**: Reduce log storage costs by up to 90% while retaining pattern visibility
- **Noise Reduction**: Identify and filter noisy patterns consuming log budget
- **SLI Tracking**: Monitor critical log patterns as service level indicators

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `log_to_pattern_metric` |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `num_of_clusters` | int | 15 | Maximum number of pattern clusters to track (controls memory and granularity) |
| `samples_per_cluster` | int | 1 | Number of sample log messages to retain per pattern (for debugging) |
| `reporting_frequency` | duration | 3m (metrics), 1m (logs) | How often to flush pattern metrics/logs |
| `throttle_limit_per_sec` | int | 200 | Maximum logs processed per second (rate limiting for protection) |
| `drain_tree_depth` | int | 7 | DRAIN tree depth (controls pattern specificity vs memory) |
| `drain_tree_max_child` | int | 100 | Maximum children per DRAIN tree node (controls memory per node) |
| `similarity_threshold` | float | 0.5 | Similarity threshold (0.0-1.0) for pattern matching (higher = stricter) |
| `disable_clustering_by_severity_level` | bool | false | Disable separate clustering by log severity (ERROR, WARN, INFO, etc.) |
| `final` | bool | false | Mark as last processor in sequence (required for final processor) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `keep_item` | bool | true | Whether to pass original log items to next processor |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## DRAIN Algorithm Explanation

### How DRAIN Works

DRAIN (Dynamic Regex-based Anomaly Indexing and Normalization) is a streaming log clustering algorithm:

1. **Preprocessing**: Removes variable tokens (IPs, timestamps, numbers) and normalizes text
2. **Tree Traversal**: Routes log through multi-level tree based on:
   - First level: Log message length (number of tokens)
   - Intermediate levels: Fixed tokens at specific positions
   - Leaf nodes: Pattern clusters with similarity matching
3. **Similarity Matching**: Compares incoming log to existing patterns using token similarity
4. **Pattern Creation**: Creates new pattern if similarity below threshold, otherwise updates existing

### DRAIN Parameters Impact

**drain_tree_depth** (default: 7):
- Deeper trees = more specific patterns (less clustering)
- Shallower trees = broader patterns (more clustering)
- Each level uses a token position to route logs
- Memory: O(patterns × depth)

**similarity_threshold** (default: 0.5):
- Higher (0.7-1.0) = stricter matching, more patterns
- Lower (0.3-0.5) = looser matching, fewer patterns
- Calculated as: matching_tokens / total_tokens
- Recommended: 0.4-0.6 for most applications

**drain_tree_max_child** (default: 100):
- Maximum child nodes per tree node
- Controls memory per node
- Higher = more patterns can branch from same node

### Severity-Based Clustering

By default, DRAIN maintains separate pattern trees for each severity level (ERROR, WARN, INFO, etc.):
- Same log message with different severities = different patterns
- Enables severity-specific anomaly detection
- Disable with `disable_clustering_by_severity_level: true` to cluster regardless of severity

## Examples

### Example 1: Basic Pattern Clustering

```yaml
- type: log_to_pattern_metric
  num_of_clusters: 15
  samples_per_cluster: 1
  reporting_frequency: 1m
  final: true
```

**What it does**:
- Clusters logs into up to 15 patterns
- Reports pattern metrics every 1 minute
- Retains 1 sample log per pattern
- Uses default DRAIN parameters (depth=7, similarity=0.5, throttle=200/sec)

**Sample Input**:
```
2024-01-15 10:23:45 ERROR Failed to connect to database server-db-01 on port 5432
2024-01-15 10:23:47 ERROR Failed to connect to database server-db-02 on port 5432
2024-01-15 10:24:12 INFO User john.doe logged in successfully from 192.168.1.10
2024-01-15 10:24:15 INFO User jane.smith logged in successfully from 192.168.1.11
```

**Pattern Output**:
```
Pattern 1: "* * ERROR Failed to connect to database * on port *" (count: 2)
Pattern 2: "* * INFO User * logged in successfully from *" (count: 2)
```

### Example 2: With Custom Throttling

```yaml
- type: log_to_pattern_metric
  num_of_clusters: 20
  samples_per_cluster: 1
  reporting_frequency: 30s
  throttle_limit_per_sec: 50
  final: true
```

**What it does**:
- Limits processing to 50 logs/second to protect against log floods
- Reports metrics every 30 seconds for faster visibility
- Tracks throttling metrics to monitor dropped logs

**Use Case**: High-volume log sources where you need rate limiting to prevent resource exhaustion.

### Example 3: High Similarity Threshold (Strict Patterns)

```yaml
- type: log_to_pattern_metric
  num_of_clusters: 30
  samples_per_cluster: 2
  reporting_frequency: 1m
  similarity_threshold: 0.7
  final: true
```

**What it does**:
- Uses stricter similarity (0.7) = more specific patterns
- Tracks up to 30 patterns for granular visibility
- Retains 2 samples per pattern for comparison

**Example Behavior**:
- `similarity_threshold: 0.5` would group: "User login failed" and "User logout failed"
- `similarity_threshold: 0.7` creates separate patterns for each

**Use Case**: Applications where you need to distinguish between similar but distinct log messages.

### Example 4: Low Similarity Threshold (Broad Patterns)

```yaml
- type: log_to_pattern_metric
  num_of_clusters: 10
  samples_per_cluster: 1
  reporting_frequency: 2m
  similarity_threshold: 0.3
  final: true
```

**What it does**:
- Uses looser similarity (0.3) = broader patterns, less clusters
- Only tracks 10 top patterns to reduce memory
- Groups more log variations into same pattern

**Use Case**: Noisy applications with many variations where you want to focus on major patterns.

### Example 5: Custom Tree Depth

```yaml
- type: log_to_pattern_metric
  num_of_clusters: 20
  samples_per_cluster: 1
  reporting_frequency: 1m
  drain_tree_depth: 4
  drain_tree_max_child: 50
  final: true
```

**What it does**:
- Shallow tree (depth=4) uses fewer token positions for routing
- Fewer child nodes (50) limits branching
- Reduces memory footprint for constrained environments

**Memory Impact**:
- `depth: 7, max_child: 100` = Higher memory, more patterns
- `depth: 4, max_child: 50` = Lower memory, broader clustering

**Use Case**: Memory-constrained environments or applications with relatively uniform log structures.

### Example 6: Disable Severity Clustering

```yaml
- type: log_to_pattern_metric
  num_of_clusters: 15
  samples_per_cluster: 1
  reporting_frequency: 1m
  disable_clustering_by_severity_level: true
  final: true
```

**What it does**:
- Clusters logs regardless of severity level
- "Connection failed" at ERROR and WARN = same pattern
- Reduces total pattern count

**Example**:
```
ERROR: Database connection timeout after 30 seconds
WARN: Database connection timeout after 30 seconds
```
- Default (severity clustering ON): 2 patterns
- With `disable_clustering_by_severity_level: true`: 1 pattern

**Use Case**: When you want to group messages by content rather than severity.

### Example 7: Multiple Samples Per Cluster

```yaml
- type: log_to_pattern_metric
  num_of_clusters: 15
  samples_per_cluster: 5
  reporting_frequency: 1m
  final: true
```

**What it does**:
- Retains 5 sample logs per pattern for debugging
- Enables comparison of variations within same pattern
- Increases memory usage (samples × patterns)

**Output Example**:
```
Pattern: "User * login failed with error code *"
Samples:
  - User admin login failed with error code 401
  - User john.doe login failed with error code 403
  - User system login failed with error code 500
  - User api-client login failed with error code 401
  - User test-user login failed with error code 403
```

**Use Case**: Development/debugging where you need to see pattern variations.

### Example 8: Pattern Retirement Configuration

```yaml
- type: log_to_pattern_metric
  num_of_clusters: 20
  samples_per_cluster: 1
  reporting_frequency: 30s
  final: true
```

**What it does**:
- When pattern count exceeds `num_of_clusters` (20), retires least frequent patterns
- Automatically manages pattern lifecycle
- Ensures most relevant patterns are tracked

**Retirement Logic**:
1. Pattern count hits limit (e.g., 20 patterns)
2. New pattern needs to be created
3. Processor retires least frequent existing pattern
4. New pattern takes its place

**Use Case**: Long-running applications where pattern diversity evolves over time.

### Example 9: Conditional Processing

```yaml
- type: log_to_pattern_metric
  condition: 'attributes["environment"] == "production"'
  num_of_clusters: 20
  samples_per_cluster: 2
  reporting_frequency: 1m
  final: true
```

**What it does**:
- Only clusters logs from production environment
- Skips dev/staging logs
- Reduces processing overhead

**Use Case**: Multi-environment deployments where you only want pattern analysis for production.

### Example 10: Keep Original Logs

```yaml
- type: log_to_pattern_metric
  num_of_clusters: 15
  samples_per_cluster: 1
  reporting_frequency: 1m
  keep_item: true
  final: false
```

**What it does**:
- Passes original log items to next processor (sequence continues)
- Enables pattern metrics + full log forwarding
- Not marked as final (more processors follow)

**Use Case**: When you want both pattern metrics AND original logs sent to destinations.

## Validation Rules

1. **Required Fields**: Must specify `type: log_to_pattern_metric`
2. **Cluster Count**: `num_of_clusters` must be > 0 (default: 15)
3. **Samples Count**: `samples_per_cluster` must be >= 0 (0 = no samples, default: 1)
4. **Frequency**: `reporting_frequency` must be valid duration (e.g., `1m`, `30s`)
5. **Throttle**: `throttle_limit_per_sec` must be > 0 (default: 200)
6. **Tree Depth**: `drain_tree_depth` must be > 0 (default: 7, typical: 4-10)
7. **Max Child**: `drain_tree_max_child` must be > 0 (default: 100)
8. **Similarity**: `similarity_threshold` must be 0.0-1.0 (default: 0.5, typical: 0.3-0.7)
9. **Condition**: If specified, must be valid OTTL expression
10. **Log Items Only**: Processor only accepts log items (not metrics or traces)

## Common Pitfalls

### 1. Similarity Threshold Too High

**Problem**: `similarity_threshold: 0.9` creates too many patterns (pattern explosion).

**Wrong**:
```yaml
similarity_threshold: 0.9
num_of_clusters: 15
```

**Result**:
- Every minor variation = new pattern
- Constantly retiring patterns
- Pattern metrics become meaningless

**Correct**:
```yaml
similarity_threshold: 0.5  # Balanced clustering
num_of_clusters: 20        # Increase if needed
```

**Guidance**: Start with 0.5, adjust based on pattern count. Lower = fewer patterns.

### 2. Similarity Threshold Too Low

**Problem**: `similarity_threshold: 0.2` clusters everything together (over-clustering).

**Wrong**:
```yaml
similarity_threshold: 0.2
```

**Result**:
- Unrelated logs grouped together
- Loses pattern specificity
- Anomalies hidden

**Correct**:
```yaml
similarity_threshold: 0.5  # Default balance
```

**Guidance**: Don't go below 0.3 unless logs are extremely varied.

### 3. Tree Depth Too Deep

**Problem**: `drain_tree_depth: 15` consumes excessive memory.

**Wrong**:
```yaml
drain_tree_depth: 15
drain_tree_max_child: 200
```

**Result**:
- Memory grows exponentially with depth
- Slower pattern matching
- Minimal pattern quality improvement

**Correct**:
```yaml
drain_tree_depth: 7   # Default, good for most cases
drain_tree_max_child: 100
```

**Guidance**:
- Short logs (< 5 tokens): depth 4-5
- Medium logs (5-15 tokens): depth 6-8
- Long logs (> 15 tokens): depth 8-10

### 4. Too Many Clusters

**Problem**: `num_of_clusters: 1000` wastes memory and processing.

**Wrong**:
```yaml
num_of_clusters: 1000
```

**Result**:
- High memory consumption
- Reporting overhead
- Most patterns rarely occur

**Correct**:
```yaml
num_of_clusters: 20  # Track top patterns
```

**Guidance**:
- Start with 15-20 clusters
- Monitor pattern retirement rate
- Increase only if important patterns being retired

### 5. Throttle Limit Too Low

**Problem**: `throttle_limit_per_sec: 10` drops most logs in high-volume scenarios.

**Wrong**:
```yaml
throttle_limit_per_sec: 10
```

**Result**:
- Most logs dropped
- Patterns based on small sample
- Skewed metrics

**Correct**:
```yaml
throttle_limit_per_sec: 200  # Default, increase if needed
```

**Guidance**:
- Monitor throttling metrics
- Increase limit if > 10% logs dropped
- Consider sampling at source instead

### 6. Too Many Samples Per Cluster

**Problem**: `samples_per_cluster: 100` wastes memory on duplicates.

**Wrong**:
```yaml
samples_per_cluster: 100
num_of_clusters: 50
```

**Result**:
- 5000 sample logs in memory
- High memory usage
- Marginal debugging benefit

**Correct**:
```yaml
samples_per_cluster: 2  # Enough for comparison
num_of_clusters: 20
```

**Guidance**:
- Production: 1-2 samples
- Development/debugging: 3-5 samples
- Never exceed 10 samples

### 7. Reporting Frequency Too Short

**Problem**: `reporting_frequency: 5s` creates excessive metric cardinality.

**Wrong**:
```yaml
reporting_frequency: 5s
```

**Result**:
- Too many metric data points
- Increased storage costs
- Minimal additional insight

**Correct**:
```yaml
reporting_frequency: 1m  # Standard for metrics
```

**Guidance**:
- Metrics mode: 1-2 minutes
- Log mode: 30s-1m
- Increase for lower overhead

### 8. Forgetting Final Flag

**Problem**: Missing `final: true` when this is the last processor.

**Wrong**:
```yaml
- type: log_to_pattern_metric
  num_of_clusters: 15
  # Missing final: true
```

**Result**: Sequence doesn't terminate properly.

**Correct**:
```yaml
- type: log_to_pattern_metric
  num_of_clusters: 15
  final: true
```

## Best Practices

1. **Start with Defaults**: Begin with default parameters and tune based on observed behavior
2. **Monitor Pattern Count**: Track actual pattern count vs `num_of_clusters` limit
3. **Check Throttling**: Monitor throttling metrics to ensure not dropping too many logs
4. **Tune Similarity**: Adjust `similarity_threshold` based on pattern granularity needs (0.4-0.6 typical)
5. **Right-Size Clusters**: Set `num_of_clusters` to 2-3x expected unique patterns
6. **Optimize Tree Depth**: Match `drain_tree_depth` to average log token count
7. **Sample Wisely**: Use 1-2 samples in production, 3-5 for debugging
8. **Consider Severity**: Keep severity clustering enabled unless you have specific reason to disable
9. **Set Reasonable Frequency**: Use 1m for metrics, 30s-1m for logs
10. **Memory Management**: For memory-constrained environments: reduce clusters, depth, samples
11. **Test Before Production**: Validate pattern quality with sample data before deploying
12. **Document Patterns**: Add `comment` field to explain clustering strategy
13. **Combine with Filters**: Use OTTL `condition` to focus on relevant log subsets
14. **Pattern Lifecycle**: Review pattern metrics regularly to identify retired patterns
15. **Throttle Protection**: Always set `throttle_limit_per_sec` to protect against log floods

## Performance Considerations

### Memory Usage

**Formula**: Memory ≈ (num_of_clusters × samples_per_cluster × avg_log_size) + (tree_nodes × drain_tree_max_child)

**Optimization**:
- Reduce `num_of_clusters`: Lower memory, fewer patterns tracked
- Reduce `samples_per_cluster`: Lower memory, less debugging capability
- Reduce `drain_tree_depth`: Fewer tree nodes, faster lookups
- Reduce `drain_tree_max_child`: Lower memory per node

**Example Memory Estimates**:
```
Low Memory Config:
  num_of_clusters: 10
  samples_per_cluster: 1
  drain_tree_depth: 4
  drain_tree_max_child: 50
  Estimated: ~5-10 MB

Standard Config (Default):
  num_of_clusters: 15
  samples_per_cluster: 1
  drain_tree_depth: 7
  drain_tree_max_child: 100
  Estimated: ~10-20 MB

High Memory Config:
  num_of_clusters: 50
  samples_per_cluster: 5
  drain_tree_depth: 10
  drain_tree_max_child: 200
  Estimated: ~50-100 MB
```

### Processing Speed

**Factors**:
- Tree depth: O(depth) lookup time
- Similarity matching: O(patterns at leaf) comparisons
- Throttling: O(1) rate limit check

**Optimization**:
- Lower `drain_tree_depth`: Faster traversal, broader patterns
- Higher `similarity_threshold`: Faster matching (fewer patterns to compare)
- Enable throttling: Protects against performance degradation

### Throttling Impact

**Without Throttling** (`throttle_limit_per_sec: 0` - not recommended):
- All logs processed
- Risk of resource exhaustion during log floods
- Memory can grow unbounded

**With Throttling** (default: 200/sec):
- Protects agent resources
- Drops excess logs during spikes
- Pattern metrics may undercount during throttling

**Monitoring**: Track `throttle_pass_rate` metric to detect when throttling is active.

## Production Example (from Test Templates)

### Example 1: Basic Pattern Metrics

```yaml
version: v3

nodes:
  - name: file_input_file
    type: file_input
    path: /var/log/application.log
    separate_source: false

  - name: sequence_processor
    type: sequence
    processors:
      - type: log_to_pattern_metric
        num_of_clusters: 20
        samples_per_cluster: 1
        reporting_frequency: 10s
        keep_item: false
        final: true

  - name: ed_output
    type: ed_output

links:
  - from: file_input_file
    to: sequence_processor
  - from: sequence_processor
    to: ed_output
```

**What it does**:
1. Reads application logs
2. Clusters into up to 20 patterns
3. Reports pattern metrics every 10 seconds
4. Drops original logs (`keep_item: false`)
5. Sends pattern metrics to EdgeDelta platform

**Use Case**: Cost optimization - convert logs to metrics for long-term storage.

### Example 2: With Throttling

```yaml
version: v3

nodes:
  - name: file_input_file
    type: file_input
    path: /var/log/high-volume-app.log

  - name: sequence_processor
    type: sequence
    processors:
      - type: log_to_pattern_metric
        num_of_clusters: 3
        samples_per_cluster: 1
        reporting_frequency: 30s
        throttle_limit_per_sec: 50
        keep_item: false
        final: true

  - name: ed_output
    type: ed_output

links:
  - from: file_input_file
    to: sequence_processor
  - from: sequence_processor
    to: ed_output
```

**What it does**:
1. Reads high-volume logs
2. Throttles to 50 logs/second
3. Tracks only 3 most frequent patterns
4. Reports every 30 seconds

**Use Case**: High-volume applications where you need rate limiting.

### Example 3: Multiple Samples for Debugging

```yaml
version: v3

nodes:
  - name: file_input_file
    type: file_input
    path: /var/log/debug-app.log

  - name: sequence_processor
    type: sequence
    processors:
      - type: log_to_pattern_metric
        num_of_clusters: 10
        samples_per_cluster: 2
        reporting_frequency: 30s
        keep_item: false
        final: true

  - name: sumo
    type: sumologic_output
    endpoint: https://collectors.sumologic.com/...

links:
  - from: file_input_file
    to: sequence_processor
  - from: sequence_processor
    to: sumo
```

**What it does**:
1. Retains 2 samples per pattern
2. Enables pattern comparison
3. Sends to Sumo Logic for analysis

**Use Case**: Development/staging environments for pattern analysis.

## Related Processors

- **extract_metric**: Extract metrics using OTTL conditions (rule-based, not pattern-based)
- **ottl_filter**: Filter logs before pattern clustering to focus on relevant subset
- **ottl_transform**: Transform log fields before pattern clustering
- **generic_mask**: Mask PII before pattern clustering to prevent sensitive data in patterns
- **aggregate_metric**: Aggregate pattern metrics for trend analysis

## Cross-References

- **edgedelta-pipelines skill**: Pattern clustering use cases
- **OTTL Reference**: https://docs.edgedelta.com/ottl-statements/
- **DRAIN Algorithm**: "Drain: An Online Log Parsing Approach with Fixed Depth Tree" (2017 research paper)

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/log-to-pattern-node/

## Notes

- DRAIN algorithm maintains separate trees for different severity levels by default
- Pattern retirement happens automatically when `num_of_clusters` limit is exceeded
- Throttling protects agent resources but may cause incomplete pattern metrics during log floods
- Tree depth should match typical log message token count for optimal performance
- Similarity threshold of 0.5 provides good balance for most applications
- Pattern IDs are globally unique (monotonic counter) across all patterns
- Preprocessing removes variable tokens (IPs, timestamps, numbers) before clustering
- Patterns use `*` wildcard to represent variable positions
- Memory usage scales with: clusters × samples + tree nodes
- Default values are tuned for typical applications - adjust based on your log characteristics
- Pattern metrics include: pattern signature, count, sample logs, timestamps
- Combine with `ottl_filter` to focus pattern clustering on error logs only
- Use `generic_mask` before this processor to prevent PII in pattern signatures
