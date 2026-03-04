# edx_delete_keys - Delete Specific Keys from Map

**Function Type**: EDX Editor
**Purpose**: Deletes specified keys from a map/object
**Signature**: `edx_delete_keys(target, keys) -> void`
**Return Type**: void (modifies map in-place)
**Minimum Agent Version**: v1.23.0
**Source**: https://docs.edgedelta.com/ottl-extensions/

---

## Quick Copy

```yaml
# Delete sensitive keys
edx_delete_keys(attributes, ["password", "ssn", "api_key"])
```

---

## Overview

edx_delete_keys removes specified keys from maps. Essential for:
- Removing sensitive data (PII, credentials)
- Field filtering
- Schema normalization
- Data minimization for compliance

**vs Standard delete_key:**
- `delete_key`: Deletes one key at a time
- `edx_delete_keys`: Deletes multiple keys in one operation

---

## Parameters

### target
- **Type**: map (pcommon.Map or map[string]any)
- **Required**: Yes
- **Description**: Map to delete keys from
- **Common**: `attributes`, `resource`, cache maps

### keys
- **Type**: []string (array of strings)
- **Required**: Yes
- **Description**: List of key names to delete
- **Can Include**: Dynamic values from cache/attributes

---

## Examples

### Example 1: Remove PII Fields
```yaml
processors:
  - name: remove-pii
    transform:
      statements:
        - edx_delete_keys(attributes, ["ssn", "credit_card", "email"])
```

### Example 2: Clean Up Temporary Fields
```yaml
processors:
  - name: cleanup
    transform:
      statements:
        - edx_delete_keys(attributes, ["temp_field", "debug_info", "internal_id"])
```

### Example 3: Dynamic Key Deletion
```yaml
processors:
  - name: dynamic-delete
    transform:
      statements:
        - set(cache["keys_to_delete"], ["field1", "field2", cache["dynamic_field"]])
        - edx_delete_keys(attributes, cache["keys_to_delete"])
```

### Example 4: Remove Non-Existent Keys (Safe)
```yaml
processors:
  - name: safe-delete
    transform:
      statements:
        # Deleting non-existent keys is safe (no error)
        - edx_delete_keys(attributes, ["might_not_exist", "also_maybe_missing"])
```

### Example 5: Batch Cleanup
```yaml
processors:
  - name: batch-cleanup
    transform:
      statements:
        - edx_delete_keys(attributes, [
            "internal_field_1",
            "internal_field_2",
            "internal_field_3",
            "debug_timestamp"
          ])
```

---

## Common Use Cases

### 1. PII Removal for Compliance
```yaml
processors:
  - name: gdpr-cleanup
    transform:
      statements:
        - edx_delete_keys(attributes, [
            "user.email",
            "user.phone",
            "user.address",
            "payment.card_number"
          ])
```

### 2. Field Filtering Before Export
```yaml
processors:
  - name: export-filter
    transform:
      statements:
        - edx_delete_keys(attributes, [
            "internal_id",
            "debug_info",
            "temp_cache_key"
          ])
```

### 3. Clean Up After Processing
```yaml
processors:
  - name: post-process-cleanup
    transform:
      statements:
        # Use fields for processing
        - set(cache["result"], ProcessData(attributes["temp_field"]))
        # Clean up temporary fields
        - edx_delete_keys(attributes, ["temp_field", "intermediate_result"])
```

---

## Validation Rules

1. **Target Must Be Map**: First parameter must be a map
2. **Keys Must Be Array**: Second parameter must be array of strings
3. **Safe on Missing**: Deleting non-existent keys does not error
4. **In-Place Modification**: Modifies the target map directly

---

## Common Pitfalls

### 1. Not Using Array Syntax
```yaml
# WRONG: Passing strings individually
edx_delete_keys(attributes, "key1", "key2")  # ERROR

# CORRECT: Use array
edx_delete_keys(attributes, ["key1", "key2"])
```

### 2. Forgetting Nested Keys
```yaml
# May need to specify full paths for nested structures
edx_delete_keys(attributes, ["user.email", "user.phone"])
```

---

## Best Practices

### 1. Group Related Deletions
```yaml
# Delete related fields together
edx_delete_keys(attributes, [
  "pii.ssn",
  "pii.dob",
  "pii.address"
])
```

### 2. Use Constants for Key Lists
```yaml
# Define reusable key lists
set(cache["pii_fields"], ["ssn", "email", "phone"])
edx_delete_keys(attributes, cache["pii_fields"])
```

---

## Performance Considerations

- **Fast**: O(n) where n = number of keys to delete
- **Batch Efficient**: More efficient than multiple delete_key() calls
- **In-Place**: No memory allocation for new map

---

## Related Functions

- **edx_keep_keys**: Inverse operation (keep only specified keys)
- **edx_delete_matching_keys**: Delete by regex pattern
- **delete_key**: Standard OTTL single-key deletion

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edx_delete_keys)
- [edx_keep_keys](./edx_keep_keys.md)

---

## Notes

- **Version**: Introduced in v1.23.0
- **Mutation**: Modifies map in-place
- **Thread Safety**: Thread-safe
