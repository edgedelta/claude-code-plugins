# EDXDataType - Runtime Type Detection

**Function Type**: EDX Converter
**Purpose**: Returns the runtime data type of any value as a string for dynamic type checking and conditional logic
**Signature**: `EDXDataType(value) -> string`
**Return Type**: string (type name)
**Minimum Agent Version**: v1.32.0
**Source**: /pkg/ottlext/funcs/func_edx_data_type.go

---

## Quick Copy

```yaml
# Check type of runtime value
set(attributes["value_type"], EDXDataType(attributes["dynamic_field"]))
```

---

## Overview

EDXDataType provides runtime type inspection for OTTL expressions. This is essential for:
- Dynamic type validation in pipelines
- Conditional processing based on data types
- Debugging and logging type information
- Building type-aware transformation logic
- Handling polymorphic data structures

Unlike compile-time type systems, EDXDataType inspects actual runtime values, making it invaluable for processing heterogeneous data sources where field types may vary.

**Detected Types:**
- Primitive types: `"string"`, `"int"`, `"int64"`, `"int32"`, `"float64"`, `"bool"`
- Collections: `"slice"`, `"map"`, `"[]byte"`
- OTEL types: `"pcommon.Map"`, `"pcommon.Slice"`, `"pcommon.Value"`
- Special: `"nil"`

**Common Use Cases:**
- Type validation before processing
- Dynamic routing based on value types
- Polymorphic field handling
- Debug logging with type information

---

## Parameters

### value
- **Type**: any
- **Required**: Yes
- **Description**: The value to inspect for type information
- **Valid Values**: Any OTTL expression that produces a value
- **Special Cases**:
  - `nil` → returns `"nil"`
  - Missing attributes → returns `"nil"`

---

## Return Value

Returns a string representing the Go type of the value.

**Common Return Values:**

| Input Type | Return String | Notes |
|------------|---------------|-------|
| `"hello"` | `"string"` | String literals and text |
| `123` | `"int64"` | Default integer type |
| `int32(5)` | `"int32"` | Explicit 32-bit integer |
| `3.14` | `"float64"` | Floating point numbers |
| `true` | `"bool"` | Boolean values |
| `nil` | `"nil"` | Null/nil values |
| `[]byte("data")` | `"[]uint8"` | Byte arrays |
| `["a", "b"]` | `"[]interface {}"` or `"pcommon.Slice"` | Arrays/slices |
| `{"key": "val"}` | `"map[string]interface {}"` or `"pcommon.Map"` | Maps/objects |
| Missing field | `"nil"` | Non-existent attributes |

**Type String Format:**
- Primitive types: Simple names (`"string"`, `"int64"`)
- Collections: Go syntax (`"[]interface {}"`, `"map[string]interface {}"`)
- pcommon types: Full package path (`"pcommon.Map"`, `"pcommon.Slice"`)

---

## Examples

### Example 1: Basic Type Detection
```yaml
# Log the type of a dynamic field
processors:
  - name: detect-type
    transform:
      statements:
        - set(attributes["field_type"], EDXDataType(attributes["dynamic_value"]))
        - set(attributes["is_string"], attributes["field_type"] == "string")
```

### Example 2: Conditional Processing by Type
```yaml
# Process differently based on type
processors:
  - name: type-based-routing
    transform:
      statements:
        - set(cache["value_type"], EDXDataType(attributes["data"]))
        - set(attributes["processed"], EDXIfElse(
            cache["value_type"] == "string",
            Concat([attributes["data"], "_suffix"]),
            EDXIfElse(
              cache["value_type"] == "int64",
              attributes["data"] * 2,
              attributes["data"]
            )
          ))
```

### Example 3: Type Validation Guard
```yaml
# Ensure field is correct type before processing
processors:
  - name: validate-type
    transform:
      statements:
        - set(cache["type"], EDXDataType(attributes["age"]))
        - set(cache["is_numeric"], cache["type"] == "int64" or cache["type"] == "int32" or cache["type"] == "float64")
        - set(attributes["age_valid"], cache["is_numeric"])
```

### Example 4: Debug Logging with Type Info
```yaml
# Add type information for debugging
processors:
  - name: debug-types
    transform:
      statements:
        - set(attributes["debug.user_type"], EDXDataType(attributes["user"]))
        - set(attributes["debug.count_type"], EDXDataType(attributes["count"]))
        - set(attributes["debug.tags_type"], EDXDataType(attributes["tags"]))
```

### Example 5: Handling Nil Values
```yaml
# Detect and handle missing fields
processors:
  - name: nil-detection
    transform:
      statements:
        - set(cache["type"], EDXDataType(attributes["optional_field"]))
        - set(cache["is_present"], cache["type"] != "nil")
        - set(attributes["field_status"], EDXIfElse(
            cache["is_present"],
            "present",
            "missing"
          ))
```

### Example 6: Type-Safe Parsing
```yaml
# Only parse if value is a string
processors:
  - name: safe-json-parse
    transform:
      statements:
        - set(cache["type"], EDXDataType(attributes["json_data"]))
        - set(cache["parsed"], EDXIfElse(
            cache["type"] == "string",
            ParseJSON(attributes["json_data"]),
            attributes["json_data"]  # Already parsed
          ))
```

### Example 7: Collection Type Detection
```yaml
# Detect maps vs arrays
processors:
  - name: collection-detection
    transform:
      statements:
        - set(cache["type"], EDXDataType(attributes["data"]))
        - set(cache["is_map"], Contains(cache["type"], "map"))
        - set(cache["is_array"], Contains(cache["type"], "slice") or Contains(cache["type"], "[]"))
        - set(attributes["collection_type"], EDXIfElse(
            cache["is_map"],
            "object",
            EDXIfElse(cache["is_array"], "array", "scalar")
          ))
```

### Example 8: Type Coercion Based on Detection
```yaml
# Convert to string if not already
processors:
  - name: ensure-string
    transform:
      statements:
        - set(cache["type"], EDXDataType(attributes["value"]))
        - set(attributes["value_str"], EDXIfElse(
            cache["type"] == "string",
            attributes["value"],
            String(attributes["value"])
          ))
```

### Example 9: Multi-Type Field Handling
```yaml
# Handle field that could be string or number
processors:
  - name: polymorphic-handler
    transform:
      statements:
        - set(cache["id_type"], EDXDataType(attributes["user_id"]))
        - set(attributes["user_id_normalized"], EDXIfElse(
            cache["id_type"] == "string",
            attributes["user_id"],
            EDXIfElse(
              cache["id_type"] == "int64",
              Concat(["user_", String(attributes["user_id"])]),
              "unknown"
            )
          ))
```

### Example 10: Type Metrics Collection
```yaml
# Collect metrics on field types
processors:
  - name: type-metrics
    transform:
      statements:
        - set(cache["types"], map[string]any{})
        - set(cache["types"]["name"], EDXDataType(attributes["name"]))
        - set(cache["types"]["age"], EDXDataType(attributes["age"]))
        - set(cache["types"]["tags"], EDXDataType(attributes["tags"]))
        - set(attributes["type_signature"], ToJSON(cache["types"]))
```

---

## Common Use Cases

### 1. Schema Validation
**Scenario**: Validate incoming data conforms to expected schema.

```yaml
processors:
  - name: validate-schema
    transform:
      statements:
        - set(cache["user_id_type"], EDXDataType(attributes["user_id"]))
        - set(cache["timestamp_type"], EDXDataType(attributes["timestamp"]))
        - set(cache["tags_type"], EDXDataType(attributes["tags"]))

        - set(cache["schema_valid"],
            cache["user_id_type"] == "string" and
            cache["timestamp_type"] == "int64" and
            Contains(cache["tags_type"], "slice"))

        - set(attributes["validation.schema_valid"], cache["schema_valid"])
```

### 2. Polymorphic Data Handling
**Scenario**: Process fields that may contain different types from different sources.

```yaml
processors:
  - name: normalize-id-field
    transform:
      statements:
        - set(cache["type"], EDXDataType(attributes["id"]))
        - set(attributes["id_string"], EDXIfElse(
            cache["type"] == "int64",
            String(attributes["id"]),
            EDXIfElse(
              cache["type"] == "string",
              attributes["id"],
              ToJSON(attributes["id"])
            )
          ))
```

### 3. Debug and Monitoring
**Scenario**: Track type distributions for troubleshooting.

```yaml
processors:
  - name: type-monitoring
    transform:
      statements:
        - set(attributes["meta.response_type"], EDXDataType(attributes["response"]))
        - set(attributes["meta.error_type"], EDXDataType(attributes["error"]))
        - set(cache["has_error"], attributes["meta.error_type"] != "nil")
```

### 4. Safe Type Conversion
**Scenario**: Only convert when necessary based on actual type.

```yaml
processors:
  - name: smart-conversion
    transform:
      statements:
        - set(cache["type"], EDXDataType(attributes["port"]))
        - set(attributes["port_int"], EDXIfElse(
            cache["type"] == "string",
            Int(attributes["port"]),
            attributes["port"]
          ))
```

---

## Validation Rules

1. **Parameter Required**: The `value` parameter must be provided (can be nil)
2. **Return Guarantee**: Always returns a non-empty string
3. **Nil Handling**: Returns `"nil"` for nil values and missing attributes
4. **Type Consistency**: Same input type always returns same string
5. **No Errors**: Function never errors, always returns type string

---

## Common Pitfalls

### 1. String Matching for Complex Types
**Problem**: Complex type strings vary and are hard to match exactly.

```yaml
# FRAGILE: Exact matching of complex types
set(cache["is_map"], EDXDataType(attributes["data"]) == "map[string]interface {}")

# BETTER: Use substring matching
set(cache["is_map"], Contains(EDXDataType(attributes["data"]), "map"))
```

### 2. Not Handling All Possible Types
**Problem**: Forgetting edge cases in type-based conditionals.

```yaml
# INCOMPLETE: Only handles string and int64
set(cache["type"], EDXDataType(attributes["value"]))
set(attributes["result"], EDXIfElse(
  cache["type"] == "string",
  attributes["value"],
  String(attributes["value"])  # What about nil, bool, float64?
))

# COMPLETE: Handle all cases
set(cache["type"], EDXDataType(attributes["value"]))
set(attributes["result"], EDXIfElse(
  cache["type"] == "nil",
  "",
  EDXIfElse(
    cache["type"] == "string",
    attributes["value"],
    String(attributes["value"])
  )
))
```

### 3. Assuming Integer Type Names
**Problem**: Different integer types have different names.

```yaml
# INCOMPLETE: Only checks int64
set(cache["is_int"], EDXDataType(attributes["count"]) == "int64")

# COMPLETE: Check all integer variants
set(cache["type"], EDXDataType(attributes["count"]))
set(cache["is_int"],
    cache["type"] == "int64" or
    cache["type"] == "int32" or
    cache["type"] == "int" or
    cache["type"] == "int8" or
    cache["type"] == "int16")
```

### 4. Forgetting pcommon Types
**Problem**: OpenTelemetry uses special pcommon types that have different names.

```yaml
# MAY FAIL: Not accounting for pcommon.Map
set(cache["is_map"], EDXDataType(attributes["data"]) == "map[string]interface {}")

# WORKS: Check for both native and pcommon
set(cache["type"], EDXDataType(attributes["data"]))
set(cache["is_map"],
    Contains(cache["type"], "map") or
    cache["type"] == "pcommon.Map")
```

---

## Best Practices

### 1. Use Substring Matching for Complex Types
Avoid exact string matching for maps and slices:

```yaml
processors:
  - name: robust-type-check
    transform:
      statements:
        - set(cache["type"], EDXDataType(attributes["data"]))
        - set(cache["is_map"], Contains(cache["type"], "map"))
        - set(cache["is_array"], Contains(cache["type"], "slice") or Contains(cache["type"], "[]"))
```

### 2. Create Type Helper Attributes
Build reusable type classification:

```yaml
processors:
  - name: type-helpers
    transform:
      statements:
        - set(cache["type"], EDXDataType(attributes["value"]))
        - set(cache["is_nil"], cache["type"] == "nil")
        - set(cache["is_numeric"], Contains(cache["type"], "int") or Contains(cache["type"], "float"))
        - set(cache["is_collection"], Contains(cache["type"], "map") or Contains(cache["type"], "slice"))
```

### 3. Combine with EDXIfElse for Type Routing
Build type-based processing pipelines:

```yaml
processors:
  - name: type-router
    transform:
      statements:
        - set(cache["type"], EDXDataType(attributes["payload"]))
        - set(attributes["parsed"], EDXIfElse(
            cache["type"] == "string",
            ParseJSON(attributes["payload"]),
            EDXIfElse(
              Contains(cache["type"], "map"),
              attributes["payload"],
              map[string]any{"error": "unsupported_type"}
            )
          ))
```

### 4. Log Type Information for Debugging
Include type metadata in debug logs:

```yaml
processors:
  - name: debug-logging
    transform:
      statements:
        - set(attributes["debug.types"], map[string]any{
            "request": EDXDataType(attributes["request"]),
            "user_id": EDXDataType(attributes["user_id"]),
            "timestamp": EDXDataType(attributes["timestamp"])
          })
```

---

## Performance Considerations

- **Very Fast**: Type detection is O(1) reflection operation
- **No Allocations**: Returns static strings for most types
- **Safe for Hot Path**: Can be used in high-throughput scenarios
- **No Parsing**: Direct runtime type inspection

**Performance Tips:**
- Cache type results if checking multiple times
- Use in conditional branches to skip expensive operations
- Prefer over repeated type assertions

---

## Related Functions

- **EDXIfElse**: Use with EDXDataType for type-based conditional logic
- **String()**: Standard conversion function after type detection
- **Int()**: Numeric conversion after validation
- **ParseJSON()**: Use after verifying string type

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edxdatatype)
- [EDXIfElse](./edxifelse.md) - Conditional logic based on type checks
- [syntax_guide.md](../syntax/syntax_guide.md)

---

## Notes

- **Version History**: Introduced in v1.32.0
- **Go Reflection**: Uses Go's reflect package internally
- **Type Names**: Returns Go-style type names, not JSON/JavaScript names
- **Thread Safe**: Function is fully thread-safe
- **No Caching**: Each call performs fresh type inspection
- **Nil Safety**: Never panics, even on nil values
