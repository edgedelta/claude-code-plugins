# json_unroll Processor

**Type**: `json_unroll`
**Category**: Data Manipulation
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/json/unroll/processor.go`
**Config Struct**: `configv3.Filter.JSONFieldPath`

## Quick Copy

```yaml
- type: json_unroll
  json_field_path: "$"
```

## Overview

The `json_unroll` processor takes a single telemetry item containing a JSON array and "unrolls" it into multiple individual telemetry items - one for each array element. This is essential for processing batch API responses, array-based logs, and multi-record payloads.

**Before unroll** (1 item):
```json
[
  {"id": 1, "name": "Alice"},
  {"id": 2, "name": "Bob"},
  {"id": 3, "name": "Charlie"}
]
```

**After unroll** (3 items):
```json
Item 1: {"id": 1, "name": "Alice"}
Item 2: {"id": 2, "name": "Bob"}
Item 3: {"id": 3, "name": "Charlie"}
```

## Use Cases

- **API Batch Responses**: Split paginated API responses returning arrays of records
- **Multi-Event Logs**: Unroll logs containing multiple events in a single JSON array
- **CMDB Sync**: Process ServiceNow/CMDB bulk exports (arrays of configuration items)
- **Metrics Batches**: Split batched metric submissions into individual data points
- **Event Streams**: Process event batches from message queues or webhooks

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `json_unroll` |
| `json_field_path` | string | JSON path to the array to unroll. Use `"$"` for root-level arrays. |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `new_field_name` | string | original | Name for the expanded array element in the body. If not specified, uses the original array element structure. |
| `final` | bool | false | Mark as last processor in sequence |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only unroll items when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## JSON Field Path Syntax

**Root-Level Array**: Use `"$"`
```yaml
json_field_path: "$"
```
Example input:
```json
[{"item": 1}, {"item": 2}]
```

**Nested Array**: Use dot notation or bracket notation
```yaml
json_field_path: "data.records"
# or
json_field_path: "content[\"records\"]"
```
Example input:
```json
{
  "data": {
    "records": [
      {"id": 1},
      {"id": 2}
    ]
  }
}
```

**Important Restrictions**:
- Cannot start with "." (use "$" for root or proper path)
- Must point to an array (not an object or primitive)

## Examples

### Example 1: Basic Root-Level Array Unroll

```yaml
- type: json_unroll
  json_field_path: "$"
```

**Input** (1 item):
```json
[
  {"user_id": 101, "status": "active"},
  {"user_id": 102, "status": "inactive"},
  {"user_id": 103, "status": "active"}
]
```

**Output** (3 items):
```json
Item 1 body: {"user_id": 101, "status": "active"}
Item 2 body: {"user_id": 102, "status": "inactive"}
Item 3 body: {"user_id": 103, "status": "active"}
```

### Example 2: Unroll with new_field_name

```yaml
- type: json_unroll
  json_field_path: "$"
  new_field_name: "user_record"
```

**Input** (1 item):
```json
[
  {"id": 1, "name": "Alice"},
  {"id": 2, "name": "Bob"}
]
```

**Output** (2 items):
```json
Item 1 body: {"user_record": {"id": 1, "name": "Alice"}}
Item 2 body: {"user_record": {"id": 2, "name": "Bob"}}
```

**Note**: Each array element is wrapped in `{"user_record": {...}}` structure.

### Example 3: Nested Array Unroll

```yaml
- type: json_unroll
  json_field_path: "data.items"
```

**Input** (1 item):
```json
{
  "metadata": {"timestamp": "2025-10-18"},
  "data": {
    "items": [
      {"product": "laptop"},
      {"product": "mouse"}
    ]
  }
}
```

**Output** (2 items):
```json
Item 1 body: {"product": "laptop"}
Item 2 body: {"product": "mouse"}
```

**Note**: Only the array elements become the body; metadata is discarded (unless preserved in attributes).

### Example 4: Production API Processing (from Template 4)

```yaml
- name: json_processor
  type: sequence
  user_description: API Response Processing
  processors:
    - type: ottl_transform
      statements: |
        set(attributes["api_source"], "jsonplaceholder")
        set(attributes["pulled_at"], Now())
    - type: json_unroll
      json_field_path: "$"
      new_field_name: "user_item"
    - type: ottl_transform
      statements: |
        set(cache["user_data"], ParseJSON(body)["user_item"])
        set(attributes["user_id"], cache["user_data"]["id"]) where cache["user_data"] != nil
        set(attributes["user_name"], cache["user_data"]["name"]) where cache["user_data"] != nil
        set(attributes["user_email"], cache["user_data"]["email"]) where cache["user_data"] != nil
        set(attributes["processed"], "true")
      final: true
```

**Workflow**:
1. **Before unroll**: Add API source metadata
2. **Unroll**: Split array into individual user items
3. **After unroll**: Extract user fields as attributes from each item

**Testing Status**: ✓ Deployed and validated (Template 4: API JSON)

### Example 5: ServiceNow CMDB Sync Pattern

```yaml
- type: json_unroll
  json_field_path: "result"
  new_field_name: "ci_record"
```

**Input** (ServiceNow table API response):
```json
{
  "result": [
    {"sys_id": "abc123", "name": "server-01", "status": "active"},
    {"sys_id": "def456", "name": "server-02", "status": "maintenance"}
  ]
}
```

**Output** (2 items):
```json
Item 1 body: {"ci_record": {"sys_id": "abc123", "name": "server-01", "status": "active"}}
Item 2 body: {"ci_record": {"sys_id": "def456", "name": "server-02", "status": "maintenance"}}
```

### Example 6: Conditional Unroll

```yaml
- type: json_unroll
  condition: 'attributes["content_type"] == "application/json"'
  json_field_path: "$"
```

**What it does**: Only unrolls items that are JSON arrays (skips other content types).

### Example 7: Deeply Nested Array

```yaml
- type: json_unroll
  json_field_path: "response.data.records"
  new_field_name: "event"
```

**Input**:
```json
{
  "response": {
    "status": "success",
    "data": {
      "records": [
        {"event_id": 1, "type": "login"},
        {"event_id": 2, "type": "logout"}
      ]
    }
  }
}
```

**Output** (2 items):
```json
Item 1 body: {"event": {"event_id": 1, "type": "login"}}
Item 2 body: {"event": {"event_id": 2, "type": "logout"}}
```

### Example 8: Preserve Attributes Across Unroll

```yaml
- name: api_processing
  type: sequence
  processors:
    - type: ottl_transform
      statements: |
        set(attributes["batch_id"], "12345")
        set(attributes["api_endpoint"], "/users")
    - type: json_unroll
      json_field_path: "$"
    - type: ottl_transform
      statements: |
        set(attributes["individual_record"], "true")
      final: true
```

**Important**: Attributes set before `json_unroll` are preserved on all unrolled items. This allows tracking which batch each record came from.

## Validation Rules

1. **Required Field**: Must specify `json_field_path`
2. **Path Syntax**: Cannot start with "." (use "$" for root)
3. **Array Target**: `json_field_path` must point to an array (processor will fail on non-arrays)
4. **Valid JSON**: Input body must be valid JSON
5. **Expression Validity**: If using advanced payload field expressions, they must be valid

## Common Pitfalls

### 1. Path Starting with Dot

**Problem**: `json_field_path` cannot start with ".".

**Wrong**:
```yaml
json_field_path: ".records"
```

**Correct**:
```yaml
json_field_path: "$"           # Root array
json_field_path: "records"     # Top-level "records" field
json_field_path: "data.records"  # Nested
```

### 2. Non-Array Target

**Problem**: Pointing to an object or primitive instead of an array.

**Wrong** (points to object):
```yaml
json_field_path: "user"  # user is {"id": 1}, not an array
```

**Correct**:
```yaml
json_field_path: "users"  # users is [{"id": 1}, {"id": 2}]
```

**Error**: Processor will fail or produce unexpected results if target is not an array.

### 3. Forgetting new_field_name Wrapping

**Problem**: Not realizing `new_field_name` wraps the array element.

**Without new_field_name**:
```yaml
json_field_path: "$"
# Output body: {"id": 1, "name": "Alice"}
```

**With new_field_name**:
```yaml
json_field_path: "$"
new_field_name: "user"
# Output body: {"user": {"id": 1, "name": "Alice"}}
```

**Impact**: Affects downstream processors that parse body (need to access `body["user"]` instead of `body` directly).

### 4. Lost Metadata from Wrapper Object

**Problem**: When unrolling a nested array, the wrapper object fields are discarded.

**Input**:
```json
{
  "timestamp": "2025-10-18",
  "records": [{"id": 1}, {"id": 2}]
}
```

**Unroll**:
```yaml
json_field_path: "records"
```

**Output bodies**:
```json
{"id": 1}
{"id": 2}
```

**Lost**: `timestamp` field is discarded.

**Solution**: Use `ottl_transform` to extract metadata into attributes BEFORE unrolling:
```yaml
- type: ottl_transform
  statements: |
    set(cache["data"], ParseJSON(body))
    set(attributes["batch_timestamp"], cache["data"]["timestamp"])
- type: json_unroll
  json_field_path: "records"
```

### 5. Empty Arrays

**Problem**: Empty array `[]` results in zero output items (not an error).

**Input**:
```json
[]
```

**Output**: 0 items (input item is consumed, nothing output).

**Consideration**: Ensure upstream data sources have records, or handle empty batches appropriately.

### 6. Large Arrays

**Problem**: Very large arrays can create memory/performance issues.

**Example**: Unrolling 10,000-element array creates 10,000 telemetry items from 1 input.

**Best Practice**: Consider pagination or chunking at the API source level to keep batch sizes reasonable (<1000 items per batch).

## Best Practices

1. **Preserve Batch Metadata**: Use `ottl_transform` before unroll to extract and preserve wrapper metadata
2. **Tag Unrolled Items**: Add attributes after unroll to mark individual records (e.g., `"record_type": "individual"`)
3. **Use new_field_name Sparingly**: Only use if you need the extra wrapper; otherwise keep body simple
4. **Error Handling**: Consider what happens with malformed JSON or non-array targets
5. **Ordering**: Place `json_unroll` early in sequence (after initial metadata extraction)
6. **Batch Tracking**: Set batch_id or similar before unroll to correlate individual records
7. **Performance**: Monitor unroll ratios (1 input → N outputs) to avoid overwhelming downstream systems

## Algorithm

1. **Parse Body**: Parse the telemetry item body as JSON
2. **Navigate Path**: Follow `json_field_path` to locate the target array
3. **Validate Array**: Ensure target is an array type
4. **Iterate Elements**: For each element in the array:
   - Create new telemetry item
   - Copy attributes from original item
   - Set body to array element (wrapped in `new_field_name` if specified)
   - Emit new item
5. **Consume Original**: Original item is consumed (not output)

## Performance Considerations

- **Multiplier Effect**: 1 input → N outputs (where N = array length)
- **Memory**: Large arrays require more memory during processing
- **Throughput**: Unrolling 100-element arrays increases downstream volume by 100x
- **Downstream Impact**: Ensure downstream processors/outputs can handle increased item count

## Related Processors

- **extract_json_field**: Extract single field from JSON (simpler, no unrolling)
- **ottl_transform**: Extract metadata before unroll, parse nested fields after unroll
- **split_with_delimiter**: Split text logs by delimiter (similar concept for non-JSON)

## Cross-References

- **edgedelta-pipelines skill**: Template 4 (API JSON Processing)
- **ottl_transform**: Often used before/after json_unroll for metadata extraction
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/unroll-json-processor/

## Notes

- **Validation Error**: `json_field_path` cannot start with "." (config_validation.go:4812)
- **Required Field**: `json_field_path` must be specified (config_validation.go:4809)
- **Preserves Attributes**: Attributes from original item are copied to all unrolled items
- **Body Replacement**: Original body is replaced with array element (or wrapped in `new_field_name`)
- **Empty Arrays**: Produce zero output items (original consumed, nothing emitted)
- **Non-Array Behavior**: Processor fails or errors if target is not an array
- **Use Case Pattern**: API responses → ottl_transform (metadata) → json_unroll → ottl_transform (field extraction)
