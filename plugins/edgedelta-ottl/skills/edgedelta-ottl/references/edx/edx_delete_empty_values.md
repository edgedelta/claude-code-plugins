# edx_delete_empty_values - Remove Empty Values from Map

**Function Type**: EDX Editor
**Purpose**: Deletes keys with empty values based on configurable strategies
**Signature**: `edx_delete_empty_values(target, excludeKeys, emptyStringValues, strategies) -> void`
**Return Type**: void (modifies map in-place)
**Minimum Agent Version**: v1.28.0
**Source**: /pkg/ottlext/funcs/func_edx_delete_empty_values.go

---

## Quick Copy

```yaml
# Remove null values and empty strings
edx_delete_empty_values(attributes, [], ["", "null"], ["deleteNull"])
```

---

## Overview

edx_delete_empty_values removes keys based on sophisticated empty detection. Essential for:
- Data minimization
- Cleaning sparse data
- Removing default/placeholder values
- Schema optimization

**Empty Detection Strategies:**
- `deleteNull`: Remove nil/null values
- `deleteEmptyList`: Remove empty arrays []
- `deleteEmptyMap`: Remove empty objects {}
- `deleteZero`: Remove zero numeric values (0, 0.0)

---

## Parameters

### target
- **Type**: map
- **Required**: Yes
- **Description**: Map to clean

### excludeKeys
- **Type**: []string
- **Required**: No (can be empty [])
- **Description**: Keys to skip (never delete even if empty)

### emptyStringValues
- **Type**: []string
- **Required**: No (can be empty [])
- **Description**: String values to treat as empty
- **Examples**: `["", "null", "N/A", "unknown"]`

### strategies
- **Type**: []string
- **Required**: No (can be empty [])
- **Description**: Empty detection strategies
- **Valid Values**:
  - `"deleteNull"`: Remove null values
  - `"deleteEmptyList"`: Remove empty arrays
  - `"deleteEmptyMap"`: Remove empty objects
  - `"deleteZero"`: Remove zero numbers

---

## Examples

### Example 1: Remove Null and Empty Strings
```yaml
processors:
  - name: clean-empty
    transform:
      statements:
        - edx_delete_empty_values(attributes, [], [""], ["deleteNull"])
```

### Example 2: Remove All Empty Types
```yaml
processors:
  - name: comprehensive-clean
    transform:
      statements:
        - edx_delete_empty_values(attributes, [], [""], [
            "deleteNull",
            "deleteEmptyList",
            "deleteEmptyMap",
            "deleteZero"
          ])
```

### Example 3: Exclude Important Fields
```yaml
processors:
  - name: clean-with-exclusions
    transform:
      statements:
        - edx_delete_empty_values(
            attributes,
            ["user_id", "timestamp"],  # Never delete these
            [""],
            ["deleteNull"]
          )
```

### Example 4: Custom Empty Values
```yaml
processors:
  - name: custom-empty
    transform:
      statements:
        - edx_delete_empty_values(
            attributes,
            [],
            ["", "N/A", "null", "undefined", "unknown"],
            ["deleteNull"]
          )
```

### Example 5: Clean Sparse Metrics
```yaml
processors:
  - name: clean-metrics
    transform:
      statements:
        # Remove zero metrics (often meaningless)
        - edx_delete_empty_values(
            attributes,
            ["request_count"],  # Keep even if zero
            [],
            ["deleteZero", "deleteNull"]
          )
```

---

## Common Use Cases

### 1. API Response Cleanup
```yaml
processors:
  - name: api-cleanup
    transform:
      statements:
        - edx_delete_empty_values(
            attributes,
            ["id"],  # Always keep ID
            ["", "null"],
            ["deleteNull", "deleteEmptyMap", "deleteEmptyList"]
          )
```

### 2. Log Normalization
```yaml
processors:
  - name: normalize-logs
    transform:
      statements:
        - edx_delete_empty_values(
            attributes,
            [],
            ["", "-", "N/A"],
            ["deleteNull"]
          )
```

### 3. Database Export
```yaml
processors:
  - name: db-export-clean
    transform:
      statements:
        - edx_delete_empty_values(
            attributes,
            ["created_at", "updated_at"],
            [""],
            ["deleteNull", "deleteZero"]
          )
```

---

## Validation Rules

1. **Target Must Be Map**: First parameter must be a map
2. **Arrays Required**: excludeKeys, emptyStringValues, strategies must be arrays
3. **Valid Strategies**: Strategy names must be recognized
4. **Safe Exclusions**: Excluded keys are never deleted

---

## Common Pitfalls

### 1. Deleting Valid Zeros
```yaml
# RISKY: Zero might be valid value
edx_delete_empty_values(attributes, [], [], ["deleteZero"])
# age=0, count=0 will be deleted!

# BETTER: Exclude fields where zero is valid
edx_delete_empty_values(
  attributes,
  ["age", "count", "balance"],
  [],
  ["deleteZero"]
)
```

### 2. Not Excluding Required Fields
```yaml
# WRONG: Might delete required but currently empty fields
edx_delete_empty_values(attributes, [], [""], ["deleteNull"])

# CORRECT: Exclude required schema fields
edx_delete_empty_values(
  attributes,
  ["user_id", "timestamp", "event_type"],
  [""],
  ["deleteNull"]
)
```

---

## Best Practices

### 1. Always Exclude Required Fields
```yaml
edx_delete_empty_values(
  attributes,
  ["id", "timestamp"],  # Schema requirements
  [""],
  ["deleteNull"]
)
```

### 2. Be Specific with String Empty Values
```yaml
# List all variations of "empty" for your data
edx_delete_empty_values(
  attributes,
  [],
  ["", "null", "N/A", "n/a", "NULL", "-", "unknown"],
  ["deleteNull"]
)
```

### 3. Use Appropriate Strategies
```yaml
# For API responses
edx_delete_empty_values(attributes, [], [""], [
  "deleteNull",
  "deleteEmptyMap",
  "deleteEmptyList"
])

# For metrics (be careful with deleteZero)
edx_delete_empty_values(attributes, ["count"], [], [
  "deleteNull",
  "deleteEmptyMap"
])
```

---

## Performance Considerations

- **Iteration Overhead**: Iterates through all map keys
- **Strategy Cost**: More strategies = more checks per key
- **Efficient**: Single pass through map

---

## Related Functions

- **edx_delete_keys**: Delete specific keys
- **edx_keep_keys**: Keep only specific keys

---

## Notes

- **Version**: v1.28.0
- **Mutation**: Modifies map in-place
- **Exclusions**: Excluded keys never deleted
- **Strategies**: Multiple strategies can be combined
