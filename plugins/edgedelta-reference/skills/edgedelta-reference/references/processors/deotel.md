# deotel Processor

**Type**: `deotel`
**Category**: Special Processors (Format Conversion)
**Sequence Compatible**: ✓ Yes (MUST be last)
**Source**: `internalv3/processors/mapper/deotel/processor.go`
**Config Struct**: `configv3.SequenceProcessor.FieldPath`

## Quick Copy

```yaml
- type: deotel
  field_path: 'attributes["payload"]'
  final: true
```

## Overview

The `deotel` processor is a **format conversion processor** that transforms telemetry data into a custom "DeOTEL" (De-OTEL) format optimized for third-party destinations. It extracts a nested payload from the telemetry item and creates a new custom-formatted item that preserves the original data type while restructuring the data layout.

**Critical Requirement**: The `deotel` processor **MUST be the last processor in any sequence** and can **only be used in sequences that connect to third-party destinations** (not EdgeDelta destinations).

## Use Cases

- **Third-Party Integration**: Prepare telemetry data for non-OTLP-native destinations (Splunk, Elasticsearch, custom endpoints)
- **Custom Format Requirements**: Convert standard OTLP format to destination-specific schemas
- **Payload Extraction**: Extract nested data structures and promote them to top-level format
- **Schema Transformation**: Restructure telemetry data while preserving original data type semantics
- **Legacy System Integration**: Bridge OTLP format with legacy monitoring systems
- **Cost Optimization**: Send data in formats optimized for specific destination pricing models
- **Compliance**: Transform data structure to meet destination-specific compliance requirements
- **Vendor Requirements**: Adapt to vendor-specific format expectations

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `deotel` |
| `field_path` | string | OTTL expression that extracts the payload to be converted (e.g., `attributes["payload"]`) |
| `final` | bool | Must be `true` - deotel is always a final processor |

### Optional Parameters (from SequenceProcessor)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

**Important**: The following parameters are NOT typically used with deotel:
- `keep_item`: Not applicable (deotel always creates new custom item)
- `data_types`: Not applicable (works with all data types)

## DeOTEL Format Explanation

### What is DeOTEL?

DeOTEL (De-OTEL) is EdgeDelta's custom telemetry format designed for third-party destinations that don't natively support OpenTelemetry Protocol (OTLP). The format:

1. **Extracts a Payload**: Uses OTTL expression to extract nested data from standard OTLP structure
2. **Creates Custom Item**: Wraps the payload in a `CustomItem` type that preserves original data type
3. **Maintains Type Semantics**: Tracks whether the original data was a log, metric, or trace
4. **Flattens Structure**: Promotes nested payload to top-level structure suitable for destinations

### Format Characteristics

**Input (Standard OTLP)**:
```json
{
  "body": "original log message",
  "attributes": {
    "k1": "v1",
    "payload": {
      "body": "transformed log message",
      "new_schema_key": "new_schema_value",
      "attributes": {
        "k2": "v2"
      }
    }
  },
  "resource": { ... }
}
```

**Output (DeOTEL Format)**:
```json
{
  "body": "transformed log message",
  "new_schema_key": "new_schema_value",
  "attributes": {
    "k2": "v2"
  },
  "_original_data_type": "log"
}
```

### Why DeOTEL Exists

- **Destination Compatibility**: Many third-party systems expect specific JSON schemas
- **Field Mapping**: Allows custom field mappings before sending to destination
- **Cost Optimization**: Removes OTLP overhead for destinations that don't need it
- **Schema Flexibility**: Enables arbitrary schema transformations via OTTL
- **Legacy Support**: Bridges modern OTLP with legacy monitoring systems

## Field Path Expression

The `field_path` parameter uses OTTL to extract the payload. Common patterns:

### Attribute Extraction
```yaml
field_path: 'attributes["payload"]'
```
Extracts the `payload` attribute as the new root.

### Nested Extraction
```yaml
field_path: 'attributes["data"]["transformed"]'
```
Extracts deeply nested field.

### Cache Extraction
```yaml
field_path: 'cache["processed_data"]'
```
Extracts data previously stored in cache by upstream processor.

### Body-Based Extraction
```yaml
field_path: 'ParseJSON(body)["extracted_field"]'
```
Parses body and extracts specific field (though this is better done in upstream `ottl_transform`).

## Examples

### Example 1: Basic DeOTEL (Final Processor)

```yaml
- name: to_splunk_sequence
  type: sequence
  user_description: "Transform and send to Splunk"
  processors:
    - type: ottl_transform
      statements: |
        set(attributes["payload"], ParseJSON(body))
    - type: deotel
      field_path: 'attributes["payload"]'
      final: true
```

**What it does**:
1. Parses JSON body into `attributes["payload"]`
2. Extracts payload as DeOTEL format
3. Marks as final processor

### Example 2: With Upstream Masking

```yaml
- name: secure_third_party_export
  type: sequence
  user_description: "Mask PII and export to third-party"
  processors:
    - type: generic_mask
      capture_group_masks:
        - capture_group: "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"
          enabled: true
          mask: "***EMAIL***"
          name: "email"
    - type: ottl_transform
      statements: |
        set(attributes["export_payload"], {
          "message": body,
          "level": attributes["severity"],
          "timestamp": Now()
        })
    - type: deotel
      field_path: 'attributes["export_payload"]'
      final: true
```

**What it does**:
1. Masks email addresses
2. Constructs custom export payload with selected fields
3. Converts to DeOTEL format for third-party destination

### Example 3: With Upstream Filtering

```yaml
- name: error_export_sequence
  type: sequence
  user_description: "Export only errors to external system"
  processors:
    - type: ottl_filter
      filter_mode: "exclude"
      statements: |
        attributes["severity"] == "ERROR"
    - type: ottl_transform
      statements: |
        set(attributes["external_format"], {
          "error_message": body,
          "error_code": attributes["error_code"],
          "service": resource.attributes["service.name"],
          "timestamp": time_unix_nano
        })
    - type: deotel
      field_path: 'attributes["external_format"]'
      final: true
```

**What it does**:
1. Filters to only ERROR severity logs
2. Transforms to external system's expected schema
3. Converts to DeOTEL format

### Example 4: Complete Pipeline Ending with DeOTEL

```yaml
nodes:
  - name: demo_logs
    type: demo_input
    log_type: apache_common
    events_per_sec: 2

  - name: transform_and_export
    type: sequence
    user_description: "Transform logs for third-party SIEM"
    processors:
      - type: grok
        pattern: "%{COMMONAPACHELOG}"
      - type: ottl_transform
        statements: |
          set(attributes["siem_payload"], {
            "source_ip": attributes["clientip"],
            "http_method": attributes["verb"],
            "http_status": Int(attributes["response"]),
            "bytes_sent": Int(attributes["bytes"]),
            "user_agent": attributes["agent"],
            "timestamp": timestamp
          })
      - type: deotel
        field_path: 'attributes["siem_payload"]'
        final: true

  - name: external_siem
    type: output_http
    endpoint: "https://siem.example.com/api/ingest"
    auth_bearer_token: "${SIEM_TOKEN}"

links:
  - from: demo_logs
    to: transform_and_export
  - from: transform_and_export
    to: external_siem
```

**What it does**:
1. Ingests demo Apache logs
2. Parses with Grok
3. Transforms to SIEM-specific schema
4. Converts to DeOTEL format
5. Sends to external SIEM via HTTP

### Example 5: Conditional DeOTEL

```yaml
- name: conditional_export
  type: sequence
  user_description: "Export high-priority events to external system"
  processors:
    - type: ottl_transform
      condition: 'attributes["priority"] == "high"'
      statements: |
        set(attributes["export_data"], {
          "alert_message": body,
          "priority": attributes["priority"],
          "affected_service": resource.attributes["service.name"]
        })
    - type: deotel
      condition: 'attributes["export_data"] != nil'
      field_path: 'attributes["export_data"]'
      final: true
```

**What it does**:
1. Only processes high-priority events
2. Creates export payload for matching events
3. Converts to DeOTEL only if export_data exists

### Example 6: Multiple Sequences Ending with DeOTEL

```yaml
nodes:
  - name: logs_input
    type: file_input
    path: "/var/log/app.log"

  - name: metrics_to_splunk
    type: sequence
    user_description: "Transform metrics for Splunk"
    processors:
      - type: ottl_transform
        data_types:
          - metric
        statements: |
          set(attributes["splunk_metric"], {
            "metric_name": name,
            "metric_value": sum.value,
            "metric_type": "gauge"
          })
      - type: deotel
        field_path: 'attributes["splunk_metric"]'
        final: true

  - name: logs_to_elastic
    type: sequence
    user_description: "Transform logs for Elasticsearch"
    processors:
      - type: ottl_transform
        data_types:
          - log
        statements: |
          set(attributes["elastic_doc"], {
            "@timestamp": time_unix_nano,
            "message": body,
            "log.level": attributes["severity"]
          })
      - type: deotel
        field_path: 'attributes["elastic_doc"]'
        final: true

  - name: splunk_dest
    type: output_splunk
    endpoint: "https://splunk.example.com:8088/services/collector/raw"

  - name: elastic_dest
    type: output_elastic
    endpoint: "https://elastic.example.com:9200"

links:
  - from: logs_input
    to: metrics_to_splunk
  - from: logs_input
    to: logs_to_elastic
  - from: metrics_to_splunk
    to: splunk_dest
  - from: logs_to_elastic
    to: elastic_dest
```

**What it does**:
1. Single input splits to two sequences
2. Each sequence transforms for specific destination format
3. Both use deotel to convert to destination-specific format

### Example 7: DeOTEL with Enrichment

```yaml
- name: enrich_and_export
  type: sequence
  user_description: "Enrich logs with lookup data and export"
  processors:
    - type: lookup
      location_path: "/etc/edgedelta/service_metadata.csv"
      key_fields:
        - event_field: "attributes.service_id"
          lookup_field: "id"
      out_fields:
        - event_field: "attributes.service_name"
          lookup_field: "name"
        - event_field: "attributes.owner_team"
          lookup_field: "team"
    - type: ottl_transform
      statements: |
        set(attributes["enriched_export"], {
          "log_message": body,
          "service": attributes["service_name"],
          "team": attributes["owner_team"],
          "severity": attributes["severity"]
        })
    - type: deotel
      field_path: 'attributes["enriched_export"]'
      final: true
```

**What it does**:
1. Enriches logs with service metadata from CSV lookup
2. Constructs export payload with enriched fields
3. Converts to DeOTEL format

### Example 8: DeOTEL with Metric Extraction

```yaml
- name: metrics_and_export
  type: sequence
  user_description: "Extract metrics and export logs to third-party"
  processors:
    - type: extract_metric
      extract_metric_rules:
        - name: "error_count"
          unit: "1"
          conditions:
            - 'IsMatch(body, "(?i)ERROR")'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
    - type: ottl_transform
      data_types:
        - log
      statements: |
        set(attributes["log_export"], {
          "message": body,
          "timestamp": time_unix_nano
        })
    - type: deotel
      data_types:
        - log
      field_path: 'attributes["log_export"]'
      final: true
```

**What it does**:
1. Extracts error count metric (continues to EdgeDelta)
2. Transforms logs for external destination
3. Converts logs to DeOTEL format (metrics unaffected)

## Validation Rules

1. **Required Fields**: Must specify `type`, `field_path`, `final: true`
2. **Final Required**: `final` must be `true` - deotel cannot have processors after it
3. **Last Processor**: Must be the last processor in the sequence (enforced by validation)
4. **Valid OTTL**: `field_path` must be valid OTTL expression that returns a map
5. **Third-Party Destinations Only**: Sequence with deotel can only link to third-party destinations
6. **Not Allowed Destinations**: Cannot link to EdgeDelta destinations (ED outputs, ED Gateway, ED AI Event)
7. **Expression Result**: `field_path` expression must evaluate to `map[string]any` type
8. **No Standalone Use**: Cannot be used as a standalone node (only within sequences)

## Common Pitfalls

### 1. Not Marking as Final

**Problem**: Forgetting to set `final: true` causes validation error.

**Wrong**:
```yaml
- type: deotel
  field_path: 'attributes["payload"]'
```

**Correct**:
```yaml
- type: deotel
  field_path: 'attributes["payload"]'
  final: true
```

### 2. Placing Processors After DeOTEL

**Problem**: Adding processors after deotel violates "must be last" rule.

**Wrong**:
```yaml
processors:
  - type: deotel
    field_path: 'attributes["payload"]'
    final: true
  - type: ottl_transform  # ERROR: Cannot come after deotel
    statements: |
      set(attributes["x"], "y")
```

**Correct**:
```yaml
processors:
  - type: ottl_transform
    statements: |
      set(attributes["x"], "y")
  - type: deotel
    field_path: 'attributes["payload"]'
    final: true
```

### 3. Missing Final Flag

**Problem**: Validation requires `final: true` for deotel.

**Error Message**: `"processor DeOTEL must be the last processor in the sequence"`

**Solution**: Always add `final: true`.

### 4. Using with EdgeDelta Destinations

**Problem**: DeOTEL can only connect to third-party destinations.

**Wrong**:
```yaml
links:
  - from: transform_sequence  # Contains deotel
    to: ed_output  # EdgeDelta destination - ERROR
```

**Error**: `"processor DeOTEL is only allowed in multiprocessor connected to third-party destinations"`

**Correct**:
```yaml
links:
  - from: transform_sequence  # Contains deotel
    to: splunk_output  # Third-party destination - OK
```

### 5. Invalid Field Path Expression

**Problem**: Field path doesn't return a map.

**Wrong**:
```yaml
field_path: 'attributes["count"]'  # Returns integer, not map
```

**Result**: Processor fails with cast error at runtime.

**Correct**:
```yaml
field_path: 'attributes["payload"]'  # Returns map
```

### 6. Using in Multiple Branches

**Problem**: DeOTEL in sequence that branches to multiple destinations.

**Issue**: If branching to both EdgeDelta and third-party destinations, separate sequences are needed.

**Solution**: Create separate sequences - one without deotel for EdgeDelta, one with deotel for third-party.

### 7. Forgetting Upstream Transformation

**Problem**: Using deotel without preparing the payload first.

**Wrong**:
```yaml
- type: deotel
  field_path: 'attributes["nonexistent"]'  # Field doesn't exist
  final: true
```

**Correct**:
```yaml
- type: ottl_transform
  statements: |
    set(attributes["export_data"], {...})
- type: deotel
  field_path: 'attributes["export_data"]'
  final: true
```

### 8. Confusing with ottl_transform

**Problem**: Trying to use ottl_transform for format conversion instead of deotel.

**Clarification**:
- **ottl_transform**: Modifies fields within existing telemetry structure
- **deotel**: Converts entire item to custom format for third-party destinations

**When to use deotel**: Only when sending to third-party destinations that require custom format.

## Best Practices

1. **Always Set final: true**: Required for deotel processor
2. **Prepare Payload Upstream**: Use `ottl_transform` to construct the payload before deotel
3. **Validate Field Path**: Test OTTL expression returns expected map structure
4. **Document Format**: Comment the expected schema in `comment` field
5. **Use Descriptive Names**: Name the payload field clearly (e.g., `splunk_payload`, `elastic_doc`)
6. **Test with Real Destination**: Verify destination accepts the DeOTEL format
7. **Preserve Essential Fields**: Include timestamp, severity, and identifying fields in payload
8. **Handle Nil Cases**: Use conditions to ensure payload exists before deotel
9. **Separate Sequences**: Use different sequences for EdgeDelta vs third-party destinations
10. **Minimize Transformations**: Do heavy transformations upstream; deotel only extracts

## Performance Considerations

- **Lightweight Operation**: DeOTEL conversion is fast (simple field extraction and wrapping)
- **Memory Efficiency**: Creates new item but releases original after conversion
- **OTTL Expression Cost**: Performance depends on complexity of `field_path` expression
- **Nested Access**: Deep nesting in field_path has minimal overhead
- **Type Preservation**: Maintains original data type metadata without extra cost

## Third-Party Destinations

DeOTEL is designed for these destination types:

### Supported Third-Party Destinations
- **Splunk**: `output_splunk`
- **Elasticsearch**: `output_elastic`
- **HTTP Generic**: `output_http`
- **Kafka**: `output_kafka`
- **S3**: `output_s3`
- **GCS**: `output_gcs`
- **Azure Blob**: `output_azure_blob`
- **Datadog**: `output_datadog` (when custom format needed)
- **New Relic**: `output_newrelic` (when custom format needed)

### Not Allowed Destinations
- **EdgeDelta**: `output_ed` (uses native OTLP)
- **ED Gateway**: `output_ed_gateway` (uses native OTLP)
- **ED AI Event**: `output_ed_ai_event` (uses native OTLP)

## Related Processors

- **ottl_transform**: Prepare payload data before deotel conversion
- **ottl_filter**: Filter which items get converted to DeOTEL format
- **generic_mask**: Mask sensitive data before format conversion
- **extract_metric**: Extract metrics while sending logs to third-party

## Cross-References

- **edgedelta-pipelines skill**: Template patterns for third-party integrations
- **OTTL Reference**: https://docs.edgedelta.com/ottl-statements/
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/deotel-processor/

## Notes

- **Special Processor**: DeOTEL is a special-purpose processor for format conversion
- **Must Be Last**: Enforced by validation - cannot have processors after deotel
- **Custom Item Type**: Creates `CustomItem` type that wraps transformed data
- **Type Preservation**: Preserves original data type (log, metric, trace) in custom item
- **OTTL Expression**: Uses OTTL to extract payload, must return map[string]any
- **Third-Party Only**: Architectural constraint - EdgeDelta destinations don't need DeOTEL format
- **Validation Errors**: Two specific errors: "must be last" and "only allowed with third-party destinations"
- **Final Flag Required**: Unlike other processors where final is optional, it's required for deotel
- **Runtime Behavior**: If expression fails or doesn't return map, item is terminated (dropped)
- **Use Sparingly**: Only use when third-party destination requires custom format
