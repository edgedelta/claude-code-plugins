# extract_json_field Processor

**Type**: `extract_json_field`
**Category**: Data Manipulation
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/json/extractor/processor.go`
**Config Struct**: `configv3.Filter.FieldPath`

## Quick Copy

```yaml
- type: extract_json_field
  field_path: "message"
```

## Overview

The `extract_json_field` processor extracts a specific field from JSON-formatted log bodies and replaces the entire body with that field's value. This enables powerful JSON log transformation by focusing on relevant data while discarding unnecessary wrapper structures. The processor supports dot notation for nested fields and array access with wildcard expansion.

## Use Cases

- **Unwrap JSON Logs**: Extract the actual message from JSON-wrapped logs (e.g., CloudWatch, Lambda)
- **API Response Extraction**: Pull out specific fields from JSON API responses
- **Array Expansion**: Split a single log containing an array into multiple logs (one per array element)
- **Nested Field Access**: Navigate deep JSON structures to extract specific values
- **Log Simplification**: Reduce complex JSON logs to their essential data fields
- **Data Pipeline Transformation**: Prepare JSON logs for downstream processors by extracting relevant fields
- **Multi-Event Processing**: Transform array-based log formats into individual log events

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `extract_json_field` |
| `field_path` | string | Dot-separated JSON path to the field to extract (e.g., `"message"`, `"records[0].data"`, `"records[*].data"`) |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `keep_log_if_failed` | bool | false | Keep the original log if extraction fails (true) or drop it (false) |
| `final` | bool | false | Mark as last processor in sequence (required for final processor) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## Field Path Syntax

The `field_path` parameter uses a flexible JSON path syntax that supports:

### Dot Notation

Navigate nested objects using dots (`.`):

```
"field"              → Top-level field
"parent.child"       → Nested field
"a.b.c.d"            → Deeply nested field
```

### Array Access

Access array elements using bracket notation:

```
"[0]"                → First element of root array
"array[0]"           → First element of named array
"records[0].data"    → Field 'data' from first array element
```

### Wildcard Array Expansion

Use `[*]` to extract from all array elements (creates multiple logs):

```
"records[*]"         → Each array element becomes a separate log
"records[*].data"    → Field 'data' from each array element becomes a separate log
"items[*].message"   → Extract 'message' from all items
```

**Important**: When using `[*]`, the processor creates multiple log items (one per array element). Without `[*]`, the processor modifies the single log item.

## Examples

### Example 1: Extract Top-Level Field

```yaml
- type: extract_json_field
  field_path: "message"
```

**Input**:
```json
{"timestamp": "2024-01-01T12:00:00Z", "level": "ERROR", "message": "Database connection failed"}
```

**Output**:
```
Database connection failed
```

**What it does**: Extracts the `message` field and replaces the entire body with its value.

### Example 2: Extract Nested Field

```yaml
- type: extract_json_field
  field_path: "event.details.error_message"
```

**Input**:
```json
{"event": {"type": "error", "details": {"error_message": "Timeout occurred", "code": 500}}}
```

**Output**:
```
Timeout occurred
```

**What it does**: Navigates through nested objects using dot notation to extract the deeply nested `error_message` field.

### Example 3: Extract from Array (Single Element)

```yaml
- type: extract_json_field
  field_path: "records[0].data"
```

**Input**:
```json
{"requestId": "abc123", "records": [{"data": "First record"}, {"data": "Second record"}]}
```

**Output**:
```
First record
```

**What it does**: Accesses the first element of the `records` array and extracts its `data` field.

### Example 4: Extract from Nested Object in Array

```yaml
- type: extract_json_field
  field_path: "logs[0].event.message"
```

**Input**:
```json
{"logs": [{"event": {"timestamp": "2024-01-01", "message": "User login"}, "metadata": {}}]}
```

**Output**:
```
User login
```

**What it does**: Navigates into the first array element, then through nested objects to extract the message.

### Example 5: Array Expansion with Wildcard

```yaml
- type: extract_json_field
  field_path: "records[*].data"
```

**Input** (1 log):
```json
{"requestId": "abc123", "records": [{"data": "Log 1"}, {"data": "Log 2"}, {"data": "Log 3"}]}
```

**Output** (3 logs):
```
Log 1
```
```
Log 2
```
```
Log 3
```

**What it does**: Splits one log into multiple logs - one for each array element's `data` field.

### Example 6: CloudWatch Logs Extraction

```yaml
- type: extract_json_field
  field_path: "message"
  keep_log_if_failed: false
```

**Input**:
```json
{"awsRequestId": "xyz", "logLevel": "ERROR", "message": "Lambda function timeout"}
```

**Output**:
```
Lambda function timeout
```

**What it does**: Common pattern for extracting actual log messages from CloudWatch/Lambda JSON wrappers. Drops logs without a `message` field.

### Example 7: Combined with json_unroll

```yaml
- name: extract_and_unroll
  type: sequence
  processors:
    - type: extract_json_field
      field_path: "events"
    - type: json_unroll
      json_field_path: "data"
      final: true
```

**Input**:
```json
{"metadata": {"source": "app"}, "events": {"data": [{"id": 1}, {"id": 2}]}}
```

**After extract_json_field**:
```json
{"data": [{"id": 1}, {"id": 2}]}
```

**After json_unroll** (2 logs):
```json
{"id": 1}
```
```json
{"id": 2}
```

**What it does**: First extracts the `events` object, then unrolls its `data` array into individual logs.

### Example 8: API Response Field Extraction

```yaml
- type: extract_json_field
  field_path: "response.body.result"
  keep_log_if_failed: true
  condition: 'attributes["source"] == "api"'
```

**Input**:
```json
{"response": {"status": 200, "body": {"result": "Operation successful", "timestamp": "2024-01-01"}}}
```

**Output**:
```
Operation successful
```

**What it does**: Extracts the actual result from a nested API response structure, but only for logs from the "api" source. Keeps logs if the extraction fails.

## Validation Rules

1. **Required Field**: `field_path` must be specified and non-empty
2. **Valid JSON**: Input log body must be valid JSON (non-JSON logs will fail or be kept based on `keep_log_if_failed`)
3. **Path Must Exist**: The specified field path must exist in the JSON (or log is dropped/kept based on `keep_log_if_failed`)
4. **Non-Empty Result**: Extracted value must not be empty (or log is dropped/kept based on `keep_log_if_failed`)
5. **Dot Notation**: Field path uses dots (`.`) to separate nested fields
6. **Array Syntax**: Use `[index]` or `[*]` for array access - no spaces inside brackets
7. **Single Wildcard**: Only one `[*]` wildcard is supported per field path (limitation noted in source code)

## Common Pitfalls

### 1. Missing Field Path

**Problem**: Forgetting to specify `field_path` parameter.

**Wrong**:
```yaml
- type: extract_json_field
```

**Correct**:
```yaml
- type: extract_json_field
  field_path: "message"
```

### 2. Leading Dot in Field Path

**Problem**: Starting field path with a dot (similar to json_unroll validation).

**Wrong**:
```yaml
field_path: ".message"
```

**Correct**:
```yaml
field_path: "message"
```

### 3. Incorrect Array Syntax

**Problem**: Using wrong array access syntax.

**Wrong**:
```yaml
field_path: "records.0.data"       # Dot notation for index
field_path: "records[ 0 ].data"    # Spaces in brackets
field_path: "records.[0].data"     # Dot before bracket
```

**Correct**:
```yaml
field_path: "records[0].data"
```

### 4. Nested Path Mistakes

**Problem**: Incorrect nesting syntax for deep objects.

**Wrong**:
```yaml
field_path: "event[details][message]"     # Multiple brackets
field_path: "event/details/message"       # Wrong separator
field_path: "event details message"       # Spaces instead of dots
```

**Correct**:
```yaml
field_path: "event.details.message"
```

### 5. Missing Field Handling

**Problem**: Not handling cases where field might not exist.

**Wrong**:
```yaml
- type: extract_json_field
  field_path: "optional_field"
  # Defaults to keep_log_if_failed: false, drops logs without this field
```

**Correct** (if field is optional):
```yaml
- type: extract_json_field
  field_path: "optional_field"
  keep_log_if_failed: true  # Keep logs even if field doesn't exist
```

### 6. Wildcard with Multiple Levels

**Problem**: Attempting multiple wildcard extractions (not currently supported).

**Wrong**:
```yaml
field_path: "records[*].data[*]"  # Multiple wildcards not supported
```

**Correct**:
```yaml
# Extract once, then use another processor for second level
- type: extract_json_field
  field_path: "records[*].data"
```

### 7. Non-JSON Input

**Problem**: Using extract_json_field on non-JSON logs.

**Example**:
```yaml
- type: extract_json_field
  field_path: "message"
  # Fails on plain text logs
```

**Solution**: Use OTTL condition to only process JSON logs:
```yaml
- type: extract_json_field
  field_path: "message"
  condition: 'IsMatch(body, "^\\{.*\\}$")'  # Only process JSON objects
  keep_log_if_failed: true
```

### 8. Extracting Empty Values

**Problem**: Field exists but contains empty value.

**Behavior**: Processor treats empty values as errors and will drop the log (or keep it if `keep_log_if_failed: true`).

## Best Practices

1. **Use Descriptive Paths**: Choose field paths that clearly indicate what data is being extracted
2. **Handle Missing Fields**: Set `keep_log_if_failed: true` when extracting from optional fields to avoid data loss
3. **Combine with Conditions**: Use OTTL conditions to only process logs where extraction makes sense
4. **Array Expansion Awareness**: Remember that `[*]` creates multiple logs - ensure downstream processors can handle increased volume
5. **Extract Early**: Place extraction early in processor sequences to simplify log structure for subsequent processors
6. **Validate JSON**: Consider using OTTL filter before extraction to ensure logs are valid JSON
7. **Document Field Paths**: Use `comment` parameter to explain complex field paths
8. **Test with Sample Data**: Validate field paths with representative sample logs before deployment
9. **Monitor Errors**: Watch for increased error metrics (drops) which indicate missing or misnamed fields
10. **Keep Structure When Needed**: If you need to preserve JSON structure, consider `json_unroll` instead of `extract_json_field`

## Performance Considerations

- **JSON Parsing**: Each log is parsed as JSON - ensure logs are well-formed to avoid parser overhead
- **Array Expansion**: Using `[*]` increases log volume - one input log becomes N output logs (impacts downstream processing)
- **Deep Nesting**: Deeply nested field paths (e.g., `a.b.c.d.e.f`) have minimal overhead - extraction is efficient
- **Error Handling**: Logs that fail extraction (invalid JSON, missing field) are tracked with error metrics and rate-limited logging
- **Memory**: Array expansion (`[*]`) creates copies of log items in memory - consider array sizes when using wildcards

## Related Processors

- **json_unroll**: Expands JSON arrays into multiple logs while preserving field structure (use when you need full objects, not just values)
- **ottl_transform**: Transform JSON fields using OTTL expressions (use for complex field manipulation)
- **generic_transform**: Transform fields using CEL expressions
- **unescape_json**: Unescape JSON strings before extraction (use when JSON is double-encoded)
- **ottl_filter**: Filter logs before extraction based on JSON content
- **regex_extract**: Extract data using regex patterns (use for non-JSON or pattern-based extraction)

## Cross-References

- **edgedelta-pipelines skill**: Used in Template 2 (JSON Array Processing), Template 3 (Mixed Telemetry)
- **json_unroll Processor**: `../../edgedelta-reference/references/processors/json_unroll.md`
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/parse-json-processor/

## Notes

- The processor uses `fastjson` library for efficient JSON parsing with pooled parsers for performance
- Field paths are comma-separated internally but use dot notation in configuration
- When `[*]` is used, the processor returns `isMultiLog: true` and creates separate log items
- Empty extraction results are treated as errors (enforced in line 106-109 of processor.go)
- The processor only works on log items (traces and metrics are passed through unchanged)
- Error logging is rate-limited to prevent log flooding when many extractions fail
- String values are automatically unquoted (fastjson adds quotes during marshaling, processor removes them)
- Boolean and numeric values are converted to strings in the extracted output
- Object and array values are marshaled as JSON strings
- The processor replaces the log body using `UpdateBody()` - original body is lost unless kept via `keep_log_if_failed`
