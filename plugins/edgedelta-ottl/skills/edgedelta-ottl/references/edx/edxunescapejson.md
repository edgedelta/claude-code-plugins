# EDXUnescapeJSON - Recursively Unescape JSON Strings

**Function Type**: EDX Converter
**Purpose**: Recursively unescapes JSON-encoded strings to properly construct nested JSON structures
**Signature**: `EDXUnescapeJSON(input) -> []byte`
**Return Type**: []byte (unescaped JSON)
**Minimum Agent Version**: v1.31.0
**Source**: /pkg/ottlext/funcs/func_edx_unescape_json.go

---

## Quick Copy

```yaml
# Unescape double-encoded JSON
set(cache["unescaped"], EDXUnescapeJSON(attributes["escaped_json"]))
set(cache["parsed"], ParseJSON(cache["unescaped"]))
```

---

## Overview

EDXUnescapeJSON handles JSON data that has been escaped multiple times (common in log aggregation, message queues, and nested API responses).

**Problem it Solves:**
```
Original: {"user":"john","age":30}
Escaped once: "{\"user\":\"john\",\"age\":30}"
Escaped twice: "\"{\\\"user\\\":\\\"john\\\",\\\"age\\\":30}\""
```

**Solution:**
Recursively unescapes until valid JSON is produced.

**Common Scenarios:**
- Logs containing JSON payloads
- Message queue double-encoding
- API responses with nested JSON strings
- Database fields storing JSON as text

---

## Parameters

### input
- **Type**: string or []byte
- **Required**: Yes
- **Description**: Escaped JSON string to unescape
- **Can Handle**: Single or multiple levels of escaping

---

## Return Value

Returns unescaped JSON as byte array (`[]byte`).

**Behavior:**
- Recursively unescapes until valid JSON
- Handles single and double quotes
- Preserves structure
- Returns original if already valid JSON or unescape fails

---

## Examples

### Example 1: Basic Unescape and Parse
```yaml
processors:
  - name: unescape-json
    transform:
      statements:
        - set(cache["unescaped"], EDXUnescapeJSON(attributes["payload"]))
        - set(cache["parsed"], ParseJSON(cache["unescaped"]))
        - set(attributes["user_id"], cache["parsed"]["userId"])
```

### Example 2: Handle Double-Escaped JSON
```yaml
processors:
  - name: double-escaped
    transform:
      statements:
        # Input: "\"{\\\"name\\\":\\\"John\\\"}\""
        - set(cache["unescaped"], EDXUnescapeJSON(body))
        # Output: {"name":"John"}
        - set(cache["data"], ParseJSON(cache["unescaped"]))
```

### Example 3: Process Log with Escaped JSON
```yaml
processors:
  - name: log-json-field
    transform:
      statements:
        - set(cache["log"], ParseJSON(body))
        - set(cache["message_unescaped"], EDXUnescapeJSON(cache["log"]["message"]))
        - set(cache["message_parsed"], ParseJSON(cache["message_unescaped"]))
```

### Example 4: Kafka Message Processing
```yaml
processors:
  - name: kafka-unescape
    transform:
      statements:
        # Kafka often double-encodes JSON values
        - set(cache["unescaped"], EDXUnescapeJSON(attributes["kafka_value"]))
        - set(cache["payload"], ParseJSON(cache["unescaped"]))
```

### Example 5: Database JSON Field
```yaml
processors:
  - name: db-json-field
    transform:
      statements:
        - set(cache["field_unescaped"], EDXUnescapeJSON(attributes["json_column"]))
        - set(cache["field_data"], ParseJSON(cache["field_unescaped"]))
        - set(attributes["extracted_value"], cache["field_data"]["key"])
```

### Example 6: API Response with Nested JSON
```yaml
processors:
  - name: nested-api-response
    transform:
      statements:
        - set(cache["response"], ParseJSON(body))
        - set(cache["data_unescaped"], EDXUnescapeJSON(cache["response"]["data"]))
        - set(cache["data_parsed"], ParseJSON(cache["data_unescaped"]))
```

### Example 7: Error Field Unescape
```yaml
processors:
  - name: unescape-error
    transform:
      statements:
        - set(cache["error_json"], EDXUnescapeJSON(attributes["error_details"]))
        - set(cache["error_obj"], ParseJSON(cache["error_json"]))
        - set(attributes["error_code"], cache["error_obj"]["code"])
```

---

## Common Use Cases

### 1. Log Aggregation
```yaml
processors:
  - name: process-logs
    transform:
      statements:
        # Logs often have escaped JSON in message field
        - set(cache["log"], ParseJSON(body))
        - set(cache["msg_unescaped"], EDXUnescapeJSON(cache["log"]["msg"]))
        - set(cache["msg_parsed"], ParseJSON(cache["msg_unescaped"]))
        - set(attributes["request_id"], cache["msg_parsed"]["request_id"])
```

### 2. Message Queue Processing
```yaml
processors:
  - name: queue-message
    transform:
      statements:
        # SQS/Kafka may double-encode JSON
        - set(cache["unescaped"], EDXUnescapeJSON(attributes["message_body"]))
        - set(cache["message"], ParseJSON(cache["unescaped"]))
        - set(attributes["event_type"], cache["message"]["eventType"])
```

### 3. Database Text Fields
```yaml
processors:
  - name: db-json-column
    transform:
      statements:
        - set(cache["json_field"], EDXUnescapeJSON(attributes["metadata_column"]))
        - set(cache["metadata"], ParseJSON(cache["json_field"]))
```

---

## Validation Rules

1. **Input Required**: Must provide input string or bytes
2. **Recursive Processing**: Unescapes until valid JSON or max depth
3. **Error Handling**: Returns original input if unescape fails
4. **Encoding**: Handles UTF-8 encoded strings

---

## Common Pitfalls

### 1. Not Parsing After Unescape
```yaml
# INCOMPLETE: Unescaped but not parsed
set(cache["unescaped"], EDXUnescapeJSON(input))
# Still a JSON string, not a map

# CORRECT: Unescape then parse
set(cache["unescaped"], EDXUnescapeJSON(input))
set(cache["parsed"], ParseJSON(cache["unescaped"]))
```

### 2. Using on Non-Escaped JSON
```yaml
# UNNECESSARY: Input already valid JSON
set(cache["json_obj"], ParseJSON(body))
set(cache["unescaped"], EDXUnescapeJSON(cache["json_obj"]))  # Not needed

# CORRECT: Only unescape if input is escaped string
set(cache["unescaped"], EDXUnescapeJSON(body_string))
set(cache["parsed"], ParseJSON(cache["unescaped"]))
```

### 3. Type Confusion
```yaml
# WRONG: EDXUnescapeJSON returns []byte, not map
set(attributes["data"], EDXUnescapeJSON(input))
set(attributes["field"], attributes["data"]["key"])  # ERROR

# CORRECT: Parse after unescape
set(cache["unescaped"], EDXUnescapeJSON(input))
set(cache["parsed"], ParseJSON(cache["unescaped"]))
set(attributes["field"], cache["parsed"]["key"])
```

---

## Best Practices

### 1. Always Parse After Unescape
```yaml
set(cache["unescaped"], EDXUnescapeJSON(input))
set(cache["data"], ParseJSON(cache["unescaped"]))
```

### 2. Handle Errors Gracefully
```yaml
set(cache["unescaped"], EDXUnescapeJSON(input) or "")
set(cache["parsed"], EDXIfElse(
  cache["unescaped"] != "",
  ParseJSON(cache["unescaped"]),
  map[string]any{}
))
```

### 3. Use for Known Escaped Sources
```yaml
# Only unescape when you know data is escaped
# Example: Specific log fields, message queue bodies
```

---

## Performance Considerations

- **Recursive Processing**: Can be expensive for deeply nested escapes
- **String Allocations**: Each unescape level allocates new string
- **Best Practice**: Avoid multiple escaping in data sources when possible

**Tips:**
- Cache unescaped results
- Only use when necessary (not on all JSON)
- Consider fixing upstream sources to avoid multiple escaping

---

## Related Functions

- **ParseJSON()**: Always use after EDXUnescapeJSON
- **EDXDecode**: For other encoding formats (base64, etc.)
- **String()**: Convert result to string if needed

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edxunescapejson)
- [ParseJSON](../standard/parsejson.md)

---

## Notes

- **Version**: Introduced in v1.31.0
- **Recursion**: Handles multiple levels of escaping
- **Thread Safety**: Thread-safe
- **Memory**: Allocates new buffers for each unescape level
