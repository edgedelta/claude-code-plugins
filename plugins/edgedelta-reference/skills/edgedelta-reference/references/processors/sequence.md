# sequence Processor

**Type**: `sequence`
**Category**: Nested Structures
**Sequence Compatible**: ✓ Yes (nested sequences supported)
**Source**: `internalv3/processors/sequence/processor.go`
**Config Struct**: `configv3.Sequence` and `configv3.SequenceProcessor`

## Quick Copy

```yaml
- type: sequence
  processors:
    - type: ottl_transform
      statements: 'set(attributes["processed"], "true")'
    - type: generic_mask
      capture_group_masks:
        - capture_group: "[0-9]{4}-[0-9]{4}-[0-9]{4}-[0-9]{4}"
          mask: "****-****-****-****"
          name: "credit_card"
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

The `sequence` processor enables nesting sequences within sequences to create modular, reusable processing blocks. This allows you to organize complex pipelines into logical sub-pipelines, apply conditional processing to groups of processors, implement multi-stage data transformations, and improve pipeline maintainability through hierarchical organization. Nested sequences can be used to isolate error-prone operations, create reusable processing templates, and clearly delineate different phases of data processing.

## Use Cases

- **Modular Pipeline Organization**: Break complex pipelines into logical, self-contained processing blocks
- **Conditional Sub-Pipelines**: Apply an entire processing chain only when certain conditions are met
- **Error Isolation**: Isolate potentially failing processors in nested sequences with separate error handling
- **Reusable Processing Blocks**: Create standardized processing sequences that can be conditionally applied
- **Multi-Stage Processing**: Clearly separate different phases (e.g., validation, transformation, enrichment, metrics)
- **Testing and Development**: Test nested sequences independently before integrating into main pipeline
- **PII Handling**: Group all PII masking operations in a single nested sequence for compliance
- **Performance Optimization**: Apply sampling or rate limiting to specific processing branches

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `sequence` |
| `processors` | []SequenceProcessor | List of processors to execute in sequence (must contain at least one non-disabled, non-comment processor) |

### Optional Parameters (from SequenceProcessor)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `final` | bool | false | Mark as last processor in parent sequence (required for final processor) |
| `disabled` | bool | false | Disable this nested sequence without removing from config |
| `condition` | string | "" | OTTL condition - only execute nested sequence when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this nested sequence |
| `keep_item` | bool | false | Keep original item after processing (by default, items are transformed) |
| `data_types` | []string | [] | Filter by data type (e.g., ["log"], ["metric"], ["trace"]) |

## Nesting Rules and Limits

### Nesting Behavior

1. **Nesting Depth**: Sequences can be nested within sequences. While there's no hard-coded depth limit in the processor code, practical limits apply based on complexity and readability.
2. **Processor Types**: Any processor compatible with sequences can be used in nested sequences, including other `sequence` processors.
3. **Condition Propagation**: Conditions on nested sequences are evaluated independently - parent conditions don't automatically apply to children.
4. **Final Flag**: The `final` flag on a processor within a nested sequence marks it as final within that nested sequence, not the entire pipeline.
5. **Data Flow**: Items flow through nested sequences linearly - output from one nested sequence becomes input to the next processor.

### Path Resolution

- **Main Sequence**: Items exiting the top-level sequence have path `""` (empty string)
- **Nested Sequence**: Items marked as `final` in a nested sequence have path `"final"` within that subsequence
- **isSubsequence Flag**: Internally, nested sequences set `IsSubsequence: true` to handle path propagation correctly

### Validation Rules

- At least one non-disabled, non-comment processor must exist in the `processors` list
- Exactly one processor must have `final: true` in each sequence (nested or top-level)
- Nested sequences can have their own `final` processor independently of parent sequences
- OTTL conditions must be valid if specified
- All processor-specific validations apply to processors within nested sequences

## Examples

### Example 1: Basic Nested Sequence

```yaml
- type: sequence
  processors:
    - type: ottl_transform
      statements: 'set(attributes["stage"], "preprocessing")'
    - type: sequence
      comment: "PII masking sub-pipeline"
      processors:
        - type: generic_mask
          capture_group_masks:
            - capture_group: "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"
              mask: "***EMAIL***"
              name: "email"
        - type: generic_mask
          capture_group_masks:
            - capture_group: "\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b"
              mask: "****-****-****-****"
              name: "credit_card"
          final: true
    - type: extract_metric
      extract_metric_rules:
        - name: "processed_items_total"
          unit: "1"
          conditions:
            - 'attributes["stage"] == "preprocessing"'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
      final: true
```

**What it does**: Creates a nested sequence for PII masking, keeping masking operations isolated and modular. The outer sequence adds metadata, applies PII masking via nested sequence, then extracts metrics.

**Use case**: Organize PII compliance requirements in a self-contained, auditable sub-pipeline.

### Example 2: Conditional Sub-Pipeline

```yaml
- type: sequence
  processors:
    - type: ottl_transform
      statements: 'set(attributes["environment"], resource["deployment.environment"])'
    - type: sequence
      condition: 'attributes["environment"] == "production"'
      comment: "Production-only processing"
      processors:
        - type: generic_mask
          capture_group_masks:
            - capture_group: "(?i)(password|token|secret)[:=]\\S+"
              mask: "***REDACTED***"
              name: "sensitive_data"
        - type: suppress
          interval: 5m
          suppress_keys:
            - 'attributes["error_code"]'
          final: true
    - type: extract_metric
      extract_metric_rules:
        - name: "total_logs"
          unit: "1"
          conditions:
            - 'IsMatch(body, ".*")'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
      final: true
```

**What it does**: Applies PII masking and suppression ONLY to production logs using a conditional nested sequence. Staging/dev logs skip the nested sequence entirely.

**Use case**: Apply strict compliance and cost-saving measures only to production data.

### Example 3: Modular Processing Blocks

```yaml
- type: sequence
  processors:
    - type: sequence
      comment: "Validation stage"
      processors:
        - type: ottl_transform
          statements: |-
            set(attributes["validation_time"], Now())
            set(attributes["has_user_id"], attributes["user_id"] != nil)
        - type: ottl_filter
          conditions:
            - 'attributes["has_user_id"] == true'
          final: true
    - type: sequence
      comment: "Enrichment stage"
      processors:
        - type: ottl_transform
          statements: |-
            set(attributes["enrichment_time"], Now())
            set(attributes["user_tier"], "premium") where attributes["user_id"] > 1000
            set(attributes["user_tier"], "standard") where attributes["user_id"] <= 1000
          final: true
    - type: sequence
      comment: "Metrics extraction stage"
      processors:
        - type: extract_metric
          extract_metric_rules:
            - name: "premium_users_total"
              unit: "1"
              conditions:
                - 'attributes["user_tier"] == "premium"'
              sum:
                aggregation_temporality: delta
                is_monotonic: true
                value: 1
          interval: 1m
          final: true
```

**What it does**: Organizes pipeline into three clear stages (validation, enrichment, metrics) using nested sequences, each with its own responsibility.

**Use case**: Create maintainable, testable pipelines with clear separation of concerns.

### Example 4: Multi-Stage Processing with Error Handling

```yaml
- type: sequence
  processors:
    - type: sequence
      comment: "Stage 1: Parse and validate"
      condition: 'IsMatch(body, "^\\{.*\\}$")'
      processors:
        - type: ottl_transform
          statements: 'set(cache["parsed"], ParseJSON(body))'
        - type: ottl_filter
          conditions:
            - 'cache["parsed"] != nil'
          final: true
    - type: sequence
      comment: "Stage 2: Transform validated data"
      condition: 'cache["parsed"] != nil'
      processors:
        - type: ottl_transform
          statements: |-
            set(attributes["event_type"], cache["parsed"]["type"])
            set(attributes["event_id"], cache["parsed"]["id"])
        - type: generic_mask
          capture_group_masks:
            - capture_group: "user_token\":\\s*\"[^\"]+\""
              mask: "user_token\": \"***REDACTED***\""
              name: "token_in_json"
          final: true
    - type: extract_metric
      extract_metric_rules:
        - name: "parsing_failures_total"
          unit: "1"
          conditions:
            - 'cache["parsed"] == nil'
            - 'IsMatch(body, "^\\{.*\\}$")'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
      final: true
```

**What it does**: Uses nested sequences with conditions to handle JSON parsing and transformation, with graceful handling of parsing failures tracked by metrics.

**Use case**: Build resilient pipelines that handle errors without failing entire processing chain.

### Example 5: Sampling Within Nested Sequence

```yaml
- type: sequence
  processors:
    - type: sequence
      comment: "High-volume log sampling"
      condition: 'resource["service.name"] == "high-volume-service"'
      processors:
        - type: sample
          percentage: 10
          field_paths: ["trace_id"]
        - type: ottl_transform
          statements: 'set(attributes["sampled"], "true")'
          final: true
    - type: sequence
      comment: "PII masking for all logs"
      processors:
        - type: generic_mask
          capture_group_masks:
            - capture_group: "\\b\\d{3}-\\d{2}-\\d{4}\\b"
              mask: "***-**-****"
              name: "ssn"
          final: true
    - type: extract_metric
      extract_metric_rules:
        - name: "logs_by_service_total"
          unit: "1"
          conditions:
            - 'resource["service.name"] != nil'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
      final: true
```

**What it does**: Applies 90% sampling to high-volume service logs in a nested sequence, while all logs (sampled or not) proceed through PII masking and metrics extraction.

**Use case**: Reduce costs for specific high-volume services while maintaining security and monitoring.

### Example 6: Environment-Specific Processing

```yaml
- type: sequence
  processors:
    - type: ottl_transform
      statements: 'set(attributes["env"], resource["deployment.environment"])'
    - type: sequence
      comment: "Development processing"
      condition: 'attributes["env"] == "development"'
      processors:
        - type: ottl_transform
          statements: |-
            set(attributes["debug"], "true")
            set(attributes["verbose_logging"], "enabled")
          final: true
    - type: sequence
      comment: "Staging processing"
      condition: 'attributes["env"] == "staging"'
      processors:
        - type: sample
          percentage: 50
        - type: ottl_transform
          statements: 'set(attributes["sampled_staging"], "true")'
          final: true
    - type: sequence
      comment: "Production processing"
      condition: 'attributes["env"] == "production"'
      processors:
        - type: sample
          percentage: 10
        - type: suppress
          interval: 10m
          suppress_keys:
            - 'attributes["error_code"]'
        - type: generic_mask
          capture_group_masks:
            - capture_group: "(?i)(api_key|token)[:=]\\S+"
              mask: "***REDACTED***"
              name: "secrets"
          final: true
    - type: extract_metric
      extract_metric_rules:
        - name: "logs_by_environment_total"
          unit: "1"
          conditions:
            - 'attributes["env"] != nil'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
      final: true
```

**What it does**: Routes logs through different nested sequences based on environment, applying environment-specific processing (dev=verbose, staging=50% sample, prod=10% sample + suppress + mask).

**Use case**: Implement different data handling policies per environment in a single pipeline.

### Example 7: Multiple Nested Sequences for Parallel Concerns

```yaml
- type: sequence
  processors:
    - type: sequence
      comment: "Security processing"
      processors:
        - type: generic_mask
          capture_group_masks:
            - capture_group: "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"
              mask: "***EMAIL***"
              name: "email"
        - type: generic_mask
          capture_group_masks:
            - capture_group: "(?i)(password|pwd)[:=]\\S+"
              mask: "***PASSWORD***"
              name: "password"
        - type: ottl_transform
          statements: 'set(attributes["security_processed"], "true")'
          final: true
    - type: sequence
      comment: "Performance optimization"
      processors:
        - type: dedup
          interval: 5m
          excluded_field_paths:
            - "timestamp"
        - type: sample
          percentage: 25
          field_paths: ["body"]
        - type: ottl_transform
          statements: 'set(attributes["optimized"], "true")'
          final: true
    - type: sequence
      comment: "Business metrics"
      processors:
        - type: extract_metric
          extract_metric_rules:
            - name: "business_events_total"
              unit: "1"
              conditions:
                - 'IsMatch(body, "(?i)(purchase|checkout|payment)")'
              sum:
                aggregation_temporality: delta
                is_monotonic: true
                value: 1
          interval: 1m
          final: true
```

**What it does**: Uses three nested sequences to handle parallel concerns: security (PII masking), performance (dedup + sampling), and business metrics, keeping each concern isolated.

**Use case**: Organize complex pipelines with multiple cross-cutting concerns into maintainable modules.

### Example 8: Complex Pipeline Organization

```yaml
- type: sequence
  processors:
    - type: comment
      comment: "=== STAGE 1: INPUT VALIDATION ==="
    - type: sequence
      processors:
        - type: ottl_filter
          conditions:
            - 'body != nil'
            - 'IsMatch(body, ".+")'
        - type: ottl_transform
          statements: 'set(attributes["validation_passed"], "true")'
          final: true
    - type: comment
      comment: "=== STAGE 2: CONDITIONAL PROCESSING ==="
    - type: sequence
      condition: 'IsMatch(body, "(?i)ERROR|WARN|FATAL")'
      comment: "Error/warning processing"
      processors:
        - type: ottl_transform
          statements: |-
            set(attributes["severity"], "high")
            set(attributes["alert_required"], "true")
        - type: suppress
          interval: 15m
          suppress_keys:
            - 'attributes["error_code"]'
          final: true
    - type: sequence
      condition: 'not IsMatch(body, "(?i)ERROR|WARN|FATAL")'
      comment: "Normal log processing"
      processors:
        - type: sample
          percentage: 10
        - type: ottl_transform
          statements: 'set(attributes["severity"], "normal")'
          final: true
    - type: comment
      comment: "=== STAGE 3: SECURITY & COMPLIANCE ==="
    - type: sequence
      processors:
        - type: generic_mask
          capture_group_masks:
            - capture_group: "\\b\\d{3}-\\d{2}-\\d{4}\\b"
              mask: "***-**-****"
              name: "ssn"
        - type: generic_mask
          capture_group_masks:
            - capture_group: "\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b"
              mask: "****-****-****-****"
              name: "credit_card"
          final: true
    - type: comment
      comment: "=== STAGE 4: METRICS & MONITORING ==="
    - type: extract_metric
      extract_metric_rules:
        - name: "errors_total"
          unit: "1"
          conditions:
            - 'attributes["severity"] == "high"'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
        - name: "normal_logs_total"
          unit: "1"
          conditions:
            - 'attributes["severity"] == "normal"'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
      final: true
```

**What it does**: Demonstrates a production-ready pipeline organized into 4 stages using nested sequences and comments: validation, conditional routing (errors vs normal), security/compliance, and metrics extraction.

**Use case**: Build enterprise-grade pipelines with clear staging, documentation, and separation of concerns.

### Example 9: Nested Sequence with Resource Attribute Propagation

```yaml
- type: sequence
  processors:
    - type: ottl_transform
      statements: |-
        set(resource["pipeline.stage"], "initial")
        set(resource["pipeline.version"], "v2.1.0")
    - type: sequence
      comment: "Sub-pipeline with resource context"
      processors:
        - type: ottl_transform
          statements: |-
            set(attributes["parent_stage"], resource["pipeline.stage"])
            set(attributes["parent_version"], resource["pipeline.version"])
        - type: ottl_transform
          statements: 'set(resource["pipeline.stage"], "nested_processing")'
        - type: generic_mask
          capture_group_masks:
            - capture_group: "(?i)api_key[:=]\\S+"
              mask: "api_key=***REDACTED***"
              name: "api_key"
          final: true
    - type: ottl_transform
      statements: 'set(attributes["final_stage"], resource["pipeline.stage"])'
    - type: extract_metric
      extract_metric_rules:
        - name: "pipeline_stages_total"
          unit: "1"
          conditions:
            - 'attributes["final_stage"] != nil'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
      final: true
```

**What it does**: Demonstrates how resource attributes are propagated through nested sequences, allowing child sequences to read and modify parent context.

**Use case**: Track processing stages and versions through nested pipeline hierarchies.

### Example 10: Deep Nesting for Specialized Processing

```yaml
- type: sequence
  processors:
    - type: ottl_transform
      statements: 'set(attributes["log_type"], "application")'
    - type: sequence
      condition: 'IsMatch(body, "(?i)ERROR")'
      comment: "Error handling sequence"
      processors:
        - type: ottl_transform
          statements: |-
            set(attributes["error_detected"], "true")
            set(attributes["error_time"], Now())
        - type: sequence
          comment: "Critical error processing"
          condition: 'IsMatch(body, "(?i)CRITICAL|FATAL")'
          processors:
            - type: ottl_transform
              statements: |-
                set(attributes["severity"], "critical")
                set(attributes["page_on_call"], "true")
            - type: suppress
              interval: 30m
              suppress_keys:
                - 'attributes["error_code"]'
              final: true
        - type: sequence
          comment: "Non-critical error processing"
          condition: 'not IsMatch(body, "(?i)CRITICAL|FATAL")'
          processors:
            - type: ottl_transform
              statements: 'set(attributes["severity"], "error")'
            - type: suppress
              interval: 5m
              suppress_keys:
                - 'attributes["error_code"]'
              final: true
        - type: generic_mask
          capture_group_masks:
            - capture_group: "stack_trace\":\\s*\"[^\"]+\""
              mask: "stack_trace\": \"***REDACTED***\""
              name: "stacktrace_pii"
          final: true
    - type: extract_metric
      extract_metric_rules:
        - name: "critical_errors_total"
          unit: "1"
          conditions:
            - 'attributes["severity"] == "critical"'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
        - name: "errors_total"
          unit: "1"
          conditions:
            - 'attributes["severity"] == "error"'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
      final: true
```

**What it does**: Uses three levels of nesting to route errors through specialized processing paths: top-level detects errors, second level splits critical vs non-critical, third level applies severity-specific suppression.

**Use case**: Implement sophisticated error classification and handling with different policies per severity level.

## Validation Rules

1. **Required Fields**: Must specify `type` as `sequence` and `processors` list
2. **Non-Empty Processors**: At least one non-disabled, non-comment processor must exist in `processors`
3. **Final Flag Requirement**: Exactly one processor must have `final: true` in each sequence
4. **Valid Processor Types**: All processors in the list must be valid sequence-compatible processor types
5. **Condition Syntax**: If `condition` is specified, it must be valid OTTL
6. **Data Types**: If `data_types` is specified, must contain valid telemetry types ("log", "metric", "trace")
7. **Nested Validation**: All validation rules apply recursively to nested sequences
8. **Processor Configuration**: Each processor in the sequence must have valid configuration for its type
9. **No Circular References**: Nested sequences cannot create circular processing loops
10. **Final Flag Placement**: The processor with `final: true` must be the last non-disabled, non-comment processor

## Common Pitfalls

### 1. Excessive Nesting Depth

**Problem**: Too many levels of nesting make pipelines hard to understand and maintain.

**Wrong**:
```yaml
- type: sequence
  processors:
    - type: sequence
      processors:
        - type: sequence
          processors:
            - type: sequence
              processors:
                - type: ottl_transform
                  statements: 'set(attributes["nested"], "too_deep")'
                  final: true
              final: true
          final: true
      final: true
```

**Correct**:
```yaml
- type: sequence
  processors:
    - type: sequence
      comment: "Validation stage"
      processors:
        - type: ottl_filter
          conditions:
            - 'body != nil'
          final: true
    - type: sequence
      comment: "Processing stage"
      processors:
        - type: ottl_transform
          statements: 'set(attributes["processed"], "true")'
          final: true
    - type: extract_metric
      extract_metric_rules:
        - name: "items_total"
          unit: "1"
          conditions:
            - 'IsMatch(body, ".*")'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
      final: true
```

**Best Practice**: Limit nesting to 2-3 levels maximum. Use flat sequences with comments for clarity when possible.

### 2. Final Flag Conflicts

**Problem**: Multiple `final: true` in same sequence or missing `final` flag.

**Wrong**:
```yaml
- type: sequence
  processors:
    - type: ottl_transform
      statements: 'set(attributes["step1"], "true")'
      final: true  # Wrong - not the last processor
    - type: ottl_filter
      conditions:
        - 'body != nil'
      final: true  # Wrong - multiple final flags
```

**Correct**:
```yaml
- type: sequence
  processors:
    - type: ottl_transform
      statements: 'set(attributes["step1"], "true")'
    - type: ottl_filter
      conditions:
        - 'body != nil'
      final: true  # Correct - only last processor is final
```

### 3. Condition Scoping Misunderstanding

**Problem**: Assuming parent sequence condition applies to all nested sequences.

**Wrong Assumption**:
```yaml
- type: sequence
  condition: 'attributes["env"] == "production"'
  processors:
    - type: sequence
      # Wrong assumption: this will only run for production
      # Reality: this runs for ALL items, parent condition already filtered
      processors:
        - type: ottl_transform
          statements: 'set(attributes["processed"], "true")'
          final: true
```

**Correct Understanding**:
```yaml
- type: sequence
  condition: 'attributes["env"] == "production"'
  comment: "Items reaching here are already filtered to production"
  processors:
    - type: sequence
      comment: "No need to re-check env here"
      processors:
        - type: ottl_transform
          statements: 'set(attributes["processed"], "true")'
          final: true
    - type: extract_metric
      extract_metric_rules:
        - name: "production_logs_total"
          unit: "1"
          conditions:
            - 'IsMatch(body, ".*")'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
      final: true
```

### 4. Missing Processors in Nested Sequence

**Problem**: Empty or disabled-only processors list in nested sequence.

**Wrong**:
```yaml
- type: sequence
  processors:
    - type: sequence
      processors:
        - type: comment
          comment: "TODO: add processors"
        # No actual processors - validation error!
```

**Correct**:
```yaml
- type: sequence
  processors:
    - type: sequence
      processors:
        - type: ottl_transform
          statements: 'set(attributes["processed"], "true")'
          final: true
    - type: extract_metric
      extract_metric_rules:
        - name: "items_total"
          unit: "1"
          conditions:
            - 'IsMatch(body, ".*")'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
      final: true
```

### 5. Resource Attribute Propagation Confusion

**Problem**: Not understanding that resource attributes persist through nested sequences.

**Wrong Expectation**:
```yaml
- type: sequence
  processors:
    - type: ottl_transform
      statements: 'set(resource["stage"], "parent")'
    - type: sequence
      processors:
        - type: ottl_transform
          # Resource attributes are available here
          statements: 'set(attributes["parent_stage"], resource["stage"])'
        - type: ottl_transform
          # Modifying resource affects parent scope
          statements: 'set(resource["stage"], "child")'
          final: true
    # resource["stage"] is now "child", not "parent"!
```

**Correct Usage**:
```yaml
- type: sequence
  processors:
    - type: ottl_transform
      statements: 'set(resource["pipeline_version"], "v1.0")'
    - type: sequence
      comment: "Resource attributes readable but modifications visible to parent"
      processors:
        - type: ottl_transform
          statements: |-
            set(attributes["version"], resource["pipeline_version"])
            set(attributes["nested_processed"], "true")
          final: true
    - type: extract_metric
      extract_metric_rules:
        - name: "processed_items_total"
          unit: "1"
          conditions:
            - 'attributes["nested_processed"] == "true"'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
      final: true
```

### 6. Disabled Nested Sequence

**Problem**: Disabling parent sequence but expecting child sequences to run.

**Wrong**:
```yaml
- type: sequence
  disabled: true
  processors:
    - type: sequence
      # This will never execute because parent is disabled
      processors:
        - type: ottl_transform
          statements: 'set(attributes["will_not_run"], "true")'
          final: true
```

**Correct**:
```yaml
- type: sequence
  processors:
    - type: sequence
      disabled: true
      comment: "Temporarily disabled for testing"
      processors:
        - type: ottl_transform
          statements: 'set(attributes["disabled"], "true")'
          final: true
    - type: sequence
      comment: "This sequence still runs"
      processors:
        - type: ottl_transform
          statements: 'set(attributes["enabled"], "true")'
          final: true
    - type: extract_metric
      extract_metric_rules:
        - name: "active_items_total"
          unit: "1"
          conditions:
            - 'attributes["enabled"] == "true"'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
      final: true
```

### 7. Complex Condition in Wrong Place

**Problem**: Putting complex filtering logic in sequence condition instead of using ottl_filter processor.

**Wrong**:
```yaml
- type: sequence
  condition: 'IsMatch(body, "(?i)ERROR") and attributes["severity"] > 3 and resource["env"] == "production"'
  processors:
    # Complex condition evaluated for every item at sequence entry
    - type: ottl_transform
      statements: 'set(attributes["processed"], "true")'
      final: true
```

**Correct**:
```yaml
- type: sequence
  processors:
    - type: ottl_filter
      comment: "Filter first, then process matching items"
      conditions:
        - 'IsMatch(body, "(?i)ERROR")'
        - 'attributes["severity"] > 3'
        - 'resource["env"] == "production"'
    - type: ottl_transform
      statements: 'set(attributes["processed"], "true")'
      final: true
```

**Best Practice**: Use `condition` on sequence for simple routing decisions. Use `ottl_filter` processor for complex filtering logic that might fail or need debugging.

## Best Practices

1. **Limit Nesting Depth**: Keep nesting to 2-3 levels maximum for maintainability
2. **Use Comments**: Add descriptive comments to nested sequences explaining their purpose
3. **Modular Design**: Each nested sequence should have a single, clear responsibility
4. **Condition Placement**: Use sequence-level conditions for routing, processor-level conditions for filtering
5. **Final Flag Clarity**: Always mark exactly one processor as `final: true` per sequence
6. **Error Isolation**: Isolate potentially failing processors in nested sequences for better error handling
7. **Testing Strategy**: Test nested sequences independently before integrating into main pipeline
8. **Resource Attributes**: Be aware that resource attribute modifications in nested sequences affect parent scope
9. **Documentation**: Document the data flow and purpose of each nested sequence in comments
10. **Avoid Over-Nesting**: If you need more than 3 levels, consider flattening or using route processor instead
11. **Performance Monitoring**: Monitor `pipeline.node.success` and `pipeline.node.miss` metrics for each sequence
12. **Consistent Naming**: Use clear, descriptive comments for nested sequences (e.g., "PII masking", "Validation stage")
13. **Conditional Independence**: Don't rely on parent conditions - each nested sequence should have clear entry criteria
14. **Processor Order**: Order processors logically within nested sequences (validate -> transform -> mask -> metrics)

## Performance Considerations

- **Nesting Overhead**: Each nested sequence adds minimal overhead (~microseconds per item for context switching)
- **Condition Evaluation**: Sequence-level conditions are evaluated once per item at entry, not per processor
- **Memory Usage**: Nested sequences use separate goroutines and channels, adding ~64KB per nested sequence
- **Item Flow**: Items flow synchronously through nested sequences (no parallel processing within a sequence)
- **Graph Start/Stop**: Each nested sequence creates a sub-graph that must be started/stopped independently
- **Channel Buffering**: Internal channels are unbuffered, providing backpressure but potentially adding latency
- **Final Path Handling**: Items marked `final` in nested sequences have path propagation overhead (~1-2 microseconds)

## Sequence Organization Patterns

### Pattern 1: Stage-Based Organization
```yaml
# Organize by processing stage
- type: sequence
  processors:
    - type: sequence
      comment: "Stage 1: Input validation"
    - type: sequence
      comment: "Stage 2: Transformation"
    - type: sequence
      comment: "Stage 3: Enrichment"
    - type: sequence
      comment: "Stage 4: Metrics extraction"
```

### Pattern 2: Concern-Based Organization
```yaml
# Organize by cross-cutting concern
- type: sequence
  processors:
    - type: sequence
      comment: "Security: PII masking"
    - type: sequence
      comment: "Performance: Sampling & dedup"
    - type: sequence
      comment: "Compliance: Audit logging"
    - type: sequence
      comment: "Monitoring: Metrics extraction"
```

### Pattern 3: Environment-Based Organization
```yaml
# Organize by environment routing
- type: sequence
  processors:
    - type: sequence
      condition: 'resource["env"] == "dev"'
      comment: "Development processing"
    - type: sequence
      condition: 'resource["env"] == "staging"'
      comment: "Staging processing"
    - type: sequence
      condition: 'resource["env"] == "production"'
      comment: "Production processing"
```

### Pattern 4: Severity-Based Organization
```yaml
# Organize by log severity
- type: sequence
  processors:
    - type: sequence
      condition: 'IsMatch(body, "(?i)CRITICAL|FATAL")'
      comment: "Critical error handling"
    - type: sequence
      condition: 'IsMatch(body, "(?i)ERROR")'
      comment: "Standard error handling"
    - type: sequence
      condition: 'IsMatch(body, "(?i)WARN")'
      comment: "Warning handling"
```

## Related Processors

- **route**: Alternative to nested sequences for conditional routing (more efficient for simple branching)
- **ottl_filter**: Filter items before/after nested sequence processing
- **ottl_transform**: Transform data within nested sequences
- **comment**: Document nested sequence structure and purpose
- **All processors**: Any sequence-compatible processor can be used in nested sequences

## Cross-References

- **edgedelta-pipelines skill**: All templates use sequences, several demonstrate nesting
- **Route Processor**: Alternative to nested sequences for conditional processing
- **OTTL Reference**: https://docs.edgedelta.com/ottl-statements/
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`

## Documentation References

See https://docs.edgedelta.com for sequence processor documentation.

## Notes

- Sequences can be nested within sequences to any practical depth, though 2-3 levels is recommended
- Each nested sequence creates its own processing graph with input, output, and final nodes
- The `isSubsequence` flag is set internally to handle path propagation for `final` processors
- Resource attributes are shared across all levels of nesting (modifications affect parent scope)
- Conditions on nested sequences are evaluated independently (parent conditions don't auto-apply)
- The `final: true` flag on a processor marks it as final within its containing sequence only
- Nested sequences add minimal overhead (~64KB memory + microseconds latency per sequence)
- Items flow synchronously through nested sequences (no parallel processing within a sequence)
- Use nested sequences for modular organization; use `route` processor for simple conditional branching
- Each sequence must have exactly one processor marked with `final: true`
- Disabled sequences are completely skipped (including all child processors)
- Comments and disabled processors don't count toward the "at least one processor" requirement
- Nested sequences are validated recursively, ensuring all child sequences meet validation requirements
- The `keep_item` flag controls whether original items are preserved after transformation
- The `data_types` filter allows sequences to process only specific telemetry types
- Metrics (`pipeline.node.success`, `pipeline.node.miss`) are tracked per sequence, including nested ones
