# ottl_transform Processor

**Type**: `ottl_transform`
**Category**: Core Transformation & Filtering
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/ottl/transform/processor.go`
**Config Struct**: `configv3.OTTL`

## Quick Copy

```yaml
- type: ottl_transform
  statements: |
    set(attributes["processed"], "true")
    set(attributes["timestamp"], Now())
```

## Overview

The `ottl_transform` processor uses OTTL (OpenTelemetry Transformation Language) to transform telemetry data by setting, modifying, or deleting attributes, resource fields, and body content. It's the most powerful and flexible transformation processor in EdgeDelta, supporting complex data manipulation, parsing, type conversion, and conditional logic.

## Use Cases

- **Attribute Enrichment**: Add metadata, environment tags, or computed values to telemetry
- **Data Parsing**: Extract fields from JSON, parse timestamps, convert data types
- **Field Renaming**: Normalize attribute names across different sources
- **Conditional Transformation**: Apply transforms only when specific conditions are met
- **Cache Usage**: Store intermediate values for complex multi-step transformations
- **Body Modification**: Transform log body content, extract structured data
- **Resource Tagging**: Add tags for filtering, routing, and organization

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `ottl_transform` |
| `statements` | string | OTTL statements to execute (supports multiline with `|` or `|-` YAML syntax) |

### Optional Parameters (from SequenceProcessor)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `filter_mode` | string | "" | Filter mode (advanced, rarely used) |
| `final` | bool | false | Mark as last processor in sequence |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `data_types` | []string | all | Process only specific telemetry types (`log`, `metric`, `trace`) |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## OTTL Language

### Basic Syntax

**Statement Structure**:
```
function(path, value) [where condition]
```

**Common Patterns**:
- `set(path, value)` - Set a field to a value
- `delete_key(path, key)` - Delete a field
- `merge_maps(target, source, strategy)` - Merge maps
- `replace_pattern(target, pattern, replacement)` - Regex replace

### Multiline Statements

OTTL statements can be written as multiline strings in YAML:

**Pipe (`|`)** - Preserves newlines:
```yaml
statements: |
  set(attributes["field1"], "value1")
  set(attributes["field2"], "value2")
```

**Pipe-Dash (`|-`)** - Strips final newline:
```yaml
statements: |-
  set(attributes["source"], "api")
  set(attributes["environment"], "production")
```

### Common OTTL Functions

#### Setters & Getters
- `set(path, value)` - Set a value
- `delete_key(attributes, "key")` - Delete a key
- `delete_matching_keys(attributes, "pattern")` - Delete keys matching pattern

#### Time Functions
- `Now()` - Current timestamp
- `Time(value, format)` - Parse time string
- `UnixSeconds(seconds)` - Unix timestamp to time

#### String Functions
- `Concat([str1, str2, ...])` - Concatenate strings
- `Split(str, delimiter)` - Split string
- `Substring(str, start, end)` - Extract substring
- `ConvertCase(str, "upper"|"lower"|"title")` - Change case
- `IsMatch(str, "pattern")` - Regex match (returns bool)
- `ReplacePattern(str, "pattern", "replacement")` - Regex replace

#### Parsing & Conversion
- `ParseJSON(str)` - Parse JSON string to object
- `Int(value)` - Convert to integer
- `String(value)` - Convert to string
- `Double(value)` - Convert to float

#### Conditional
- `where condition` - Apply statement only if condition is true

### Field Paths

**Common Paths**:
- `attributes["key"]` - Event attributes (most common)
- `resource.attributes["key"]` - Resource-level attributes
- `body` - Log body content (string)
- `cache["key"]` - Temporary cache for intermediate values
- `instrumentation_scope.name` - Instrumentation scope

**Nested Access**:
```ottl
attributes["user"]["name"]           # Access nested fields
cache["parsed_data"]["id"]           # Cache nested access
```

## Examples

### Example 1: Basic Attribute Setting

```yaml
- type: ottl_transform
  statements: |
    set(attributes["processed"], "true")
    set(attributes["pipeline"], "production")
```

**What it does**: Adds two attributes to all telemetry items.

### Example 2: Add Timestamp

```yaml
- type: ottl_transform
  statements: |
    set(attributes["processed_at"], Now())
```

**What it does**: Adds current timestamp to each item.

### Example 3: Conditional Transform

```yaml
- type: ottl_transform
  statements: |
    set(attributes["priority"], "high") where attributes["severity"] == "ERROR"
    set(attributes["priority"], "low") where attributes["severity"] == "INFO"
```

**What it does**: Sets priority based on severity level.

### Example 4: Parse JSON from Body

```yaml
- type: ottl_transform
  statements: |
    set(cache["parsed"], ParseJSON(body))
    set(attributes["user_id"], cache["parsed"]["userId"]) where cache["parsed"] != nil
    set(attributes["request_id"], cache["parsed"]["requestId"]) where cache["parsed"] != nil
```

**What it does**:
1. Parses JSON from log body into cache
2. Extracts userId and requestId fields as attributes
3. Only sets attributes if parsing succeeded

**Production Example (from Template 4)**:
```yaml
- type: ottl_transform
  statements: |
    set(cache["user_data"], ParseJSON(body)["user_item"])
    set(attributes["user_id"], cache["user_data"]["id"]) where cache["user_data"] != nil
    set(attributes["user_name"], cache["user_data"]["name"]) where cache["user_data"] != nil
    set(attributes["user_email"], cache["user_data"]["email"]) where cache["user_data"] != nil
```

### Example 5: String Manipulation

```yaml
- type: ottl_transform
  statements: |
    set(attributes["hostname"], ConvertCase(attributes["HOST"], "lower"))
    set(attributes["environment"], Split(attributes["deployment_id"], "-")[0])
```

**What it does**:
1. Converts hostname to lowercase
2. Extracts environment from deployment_id (e.g., "prod-app-123" → "prod")

### Example 6: Field Renaming

```yaml
- type: ottl_transform
  statements: |
    set(attributes["http.status_code"], attributes["status"])
    delete_key(attributes, "status")
```

**What it does**: Renames `status` attribute to `http.status_code` (standardization).

### Example 7: Complex Multi-Source Enrichment (from Template 5)

```yaml
- type: ottl_transform
  statements: |
    set(attributes["api_type"], "users")
    set(attributes["processed_at"], Now())
  final: true
```

**What it does**: Tags data from specific API source with type and processing timestamp.

### Example 8: Numeric Calculations

```yaml
- type: ottl_transform
  statements: |
    set(attributes["duration_seconds"], Double(attributes["duration_ms"]) / 1000.0)
    set(attributes["bytes_gb"], Double(attributes["bytes"]) / 1073741824.0)
```

**What it does**: Converts milliseconds to seconds and bytes to gigabytes.

### Example 9: Conditional Body Modification

```yaml
- type: ottl_transform
  statements: |
    set(body, ReplacePattern(body, "password=\\S+", "password=***REDACTED***")) where IsMatch(body, "password=")
```

**What it does**: Redacts passwords only in logs that contain password fields.

### Example 10: Resource-Level Tagging

```yaml
- type: ottl_transform
  statements: |
    set(resource.attributes["deployment.environment"], "production")
    set(resource.attributes["team"], "platform")
```

**What it does**: Adds resource-level attributes (apply to all telemetry from this source).

### Example 11: Using Processor Condition

```yaml
- type: ottl_transform
  condition: 'attributes["environment"] == "production"'
  statements: |
    set(attributes["monitoring_enabled"], "true")
    set(attributes["retention_days"], 30)
```

**What it does**: Only applies transforms to production environment telemetry.

### Example 12: Data Type Filtering

```yaml
- type: ottl_transform
  data_types:
    - log
  statements: |
    set(attributes["log_processed"], "true")
```

**What it does**: Only processes log data (not metrics or traces).

## Validation Rules

1. **Required Fields**: Must specify `type`, `statements`
2. **Non-Empty Statements**: `statements` cannot be empty
3. **Valid OTTL Syntax**: All statements must be valid OTTL (parser will validate)
4. **Path Validity**: Field paths must reference valid locations
5. **Type Compatibility**: Operations must match field types (e.g., can't do math on strings)
6. **Function Existence**: All functions must be recognized OTTL functions
7. **Where Clause**: If using `where`, condition must be valid OTTL boolean expression

## Common Pitfalls

### 1. Missing Quotes Around Attribute Keys

**Problem**: Attribute keys must be quoted in OTTL.

**Wrong**:
```yaml
set(attributes[status], "ok")
```

**Correct**:
```yaml
set(attributes["status"], "ok")
```

### 2. Confusing Condition vs Where

**Problem**: `condition` (processor-level) vs `where` (statement-level).

**Processor Condition**: Applies to all statements
```yaml
- type: ottl_transform
  condition: 'attributes["env"] == "prod"'  # All statements only run in prod
  statements: |
    set(attributes["x"], "1")
```

**Statement Where**: Applies to individual statement
```yaml
- type: ottl_transform
  statements: |
    set(attributes["x"], "1") where attributes["env"] == "prod"  # Only this statement conditional
```

### 3. YAML Multiline Syntax

**Problem**: Using wrong multiline syntax.

**Both work, choose one**:
```yaml
statements: |   # Pipe - preserves newlines
  set(attributes["a"], "1")

statements: |-  # Pipe-dash - strips final newline
  set(attributes["a"], "1")
```

### 4. Nil/Null Checks

**Problem**: Accessing fields that might not exist.

**Wrong** (fails if field missing):
```yaml
set(attributes["user_id"], cache["data"]["userId"])
```

**Correct** (conditional):
```yaml
set(attributes["user_id"], cache["data"]["userId"]) where cache["data"] != nil
```

### 5. Type Mismatches

**Problem**: Treating numbers as strings or vice versa.

**Solution**: Use explicit type conversion:
```yaml
set(attributes["port_str"], String(attributes["port"]))
set(attributes["count_int"], Int(attributes["count_str"]))
```

### 6. Overwriting vs Appending

**Problem**: Accidentally overwriting existing values.

**Overwrite** (replaces):
```yaml
set(attributes["tags"], "new_value")
```

**Preserve** (check first):
```yaml
set(attributes["tags"], "default") where attributes["tags"] == nil
```

### 7. Cache Cleanup

**Problem**: Cache persists across items and can consume memory.

**Best Practice**: Delete cache entries when done:
```yaml
set(cache["temp"], ParseJSON(body))
set(attributes["id"], cache["temp"]["id"])
delete_key(cache, "temp")
```

### 8. Case Sensitivity

**Problem**: OTTL is case-sensitive for keys, values, and functions.

**Wrong**: `now()`, `CONCA T()`, `Set()`
**Correct**: `Now()`, `Concat()`, `set()`

## Best Practices

1. **Use Cache for Intermediate Values**: Avoid re-parsing or re-computing
2. **Conditional Checks**: Always check for nil/null before accessing nested fields
3. **Descriptive Attribute Names**: Use clear, standardized naming (e.g., `http.status_code`, not `status`)
4. **Minimize Statements**: Combine related operations to reduce overhead
5. **Comment Complex Logic**: Use `comment` field or YAML comments for complex transforms
6. **Resource vs Attributes**: Use `resource.attributes` for source-level tags, `attributes` for event-level
7. **Type Safety**: Use explicit type conversions to avoid runtime errors
8. **Processor Order**: Apply ottl_transform after parsing but before filtering/sampling
9. **Test with Real Data**: Validate OTTL statements with actual telemetry samples
10. **Use where Clauses**: Make statements conditional to avoid unnecessary operations

## Performance Considerations

- **Statement Count**: Each statement is evaluated independently - minimize when possible
- **Regex Operations**: `IsMatch` and `ReplacePattern` are expensive - use sparingly
- **JSON Parsing**: `ParseJSON` is costly - cache results and reuse
- **Nested Access**: Deep nesting has slight overhead - flatten when possible
- **Conditional Execution**: Use `where` to skip statements when not needed

## OTTL Extensions (EdgeDelta-Specific)

EdgeDelta has extended OTTL with additional functions. Key categories:

### Converter Functions
- Custom type conversions
- Specialized parsing

### Editor Functions
- Advanced data manipulation
- EdgeDelta-specific transformations

**Documentation**:
- https://docs.edgedelta.com/ottl-extensions/
- https://docs.edgedelta.com/ottl-editors/

## Production Examples

### From Template 2 (OTLP Dual)

```yaml
- type: ottl_transform
  statements: |-
    set(attributes["processed"], "true")
    set(attributes["pipeline"], "template2-otlp")
```

**Use Case**: Tag all OTLP data with pipeline identifier.

### From Template 3 (Mixed Telemetry)

```yaml
- type: ottl_transform
  statements: |-
    set(attributes["source"], "prometheus")
    set(attributes["environment"], "production")
  final: true
```

**Use Case**: Tag metrics with source and environment for routing.

### From Template 4 (API JSON)

```yaml
- type: ottl_transform
  statements: |
    set(attributes["api_source"], "jsonplaceholder")
    set(attributes["pulled_at"], Now())
- type: json_unroll
  json_field_path: "$"
- type: ottl_transform
  statements: |
    set(cache["user_data"], ParseJSON(body)["user_item"])
    set(attributes["user_id"], cache["user_data"]["id"]) where cache["user_data"] != nil
    set(attributes["user_name"], cache["user_data"]["name"]) where cache["user_data"] != nil
    set(attributes["user_email"], cache["user_data"]["email"]) where cache["user_data"] != nil
```

**Use Case**:
1. Tag API source and timestamp
2. Unroll JSON array
3. Extract nested user fields into attributes

### From Template 5 (Multi-API)

```yaml
- name: users_processor
  type: sequence
  processors:
    - type: ottl_transform
      statements: |
        set(attributes["api_type"], "users")
        set(attributes["processed_at"], Now())
      final: true
```

**Use Case**: Tag each API source distinctly for aggregation.

**Testing Status**: ✓ All deployed and validated (Templates 2, 3, 4, 5)

## Related Processors

- **ottl_filter**: Filter telemetry using OTTL conditions
- **ottl_context_filter**: Context-aware OTTL filtering
- **generic_mask**: Regex-based masking (simpler for PII)
- **extract_json_field**: Extract single JSON field (simpler than ParseJSON for basic cases)

## Cross-References

- **edgedelta-pipelines skill**: Template 2 (OTLP), Template 3 (Mixed), Template 4 (API JSON), Template 5 (Multi-API)
- **OTTL Language Guide**: https://docs.edgedelta.com/ottl-statements/
- **OTTL Extensions**: https://docs.edgedelta.com/ottl-extensions/
- **OTTL Editor Functions**: https://docs.edgedelta.com/ottl-editors/

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/add-field-processor/

## Notes

- OTTL is based on OpenTelemetry's transformation language
- EdgeDelta has extended OTTL with custom functions
- Statements are evaluated in order (sequential execution)
- Cache is shared across statements within same processor instance
- Multiline statements use YAML's `|` or `|-` syntax
- Edge Delta's AI can help generate complex OTTL statements from natural language
- Functions are case-sensitive: `Now()` not `now()`
- Lowercase operators required: `and`, `or`, `not` (not `AND`, `OR`, `NOT`)
