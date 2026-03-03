# edx_delete_matching_keys - Delete Keys by Regex Pattern

**Function Type**: EDX Editor
**Purpose**: Deletes keys from a map that match regex patterns
**Signature**: `edx_delete_matching_keys(target, patterns) -> void`
**Return Type**: void (modifies map in-place)
**Minimum Agent Version**: v1.23.0
**Source**: /pkg/ottlext/funcs/func_edx_delete_matching_keys.go

---

## Quick Copy

```yaml
# Delete all internal fields
edx_delete_matching_keys(attributes, ["^internal_.*", "^_.*"])
```

---

## Overview

edx_delete_matching_keys removes keys matching regex patterns. Powerful for:
- Removing debug/internal fields by convention
- Cleaning up prefixed/suffixed fields
- Pattern-based filtering
- Dynamic field removal

---

## Parameters

### target
- **Type**: map
- **Required**: Yes
- **Description**: Map to delete keys from

### patterns
- **Type**: []string (array of regex patterns)
- **Required**: Yes
- **Description**: Regex patterns for matching keys
- **Syntax**: Go regex (RE2)

---

## Examples

### Example 1: Delete Internal Fields
```yaml
processors:
  - name: remove-internal
    transform:
      statements:
        - edx_delete_matching_keys(attributes, ["^internal_", "^_"])
```

### Example 2: Remove Debug Fields
```yaml
processors:
  - name: remove-debug
    transform:
      statements:
        - edx_delete_matching_keys(attributes, ["^debug_", "_debug$"])
```

### Example 3: Clean Temporary Fields
```yaml
processors:
  - name: cleanup-temp
    transform:
      statements:
        - edx_delete_matching_keys(attributes, ["^temp_", "^tmp_", "_cache$"])
```

### Example 4: Remove Numbered Fields
```yaml
processors:
  - name: remove-numbered
    transform:
      statements:
        - edx_delete_matching_keys(attributes, ["^field_\\d+$"])
```

---

## Common Use Cases

### 1. Convention-Based Cleanup
```yaml
# Remove all underscore-prefixed fields
edx_delete_matching_keys(attributes, ["^_"])
```

### 2. Vendor-Specific Fields
```yaml
# Remove AWS internal fields
edx_delete_matching_keys(attributes, ["^aws\\.", "^x-amz-"])
```

---

## Validation Rules

1. **Valid Regex**: Patterns must be valid Go regex
2. **Array Required**: Patterns must be provided as array
3. **Safe on No Match**: No error if no keys match

---

## Best Practices

1. **Anchor Patterns**: Use `^` and `$` for precision
2. **Test Patterns**: Verify regex matches intended keys
3. **Escape Special Chars**: Remember to escape `.`, `*`, etc.

---

## Related Functions

- **edx_delete_keys**: Delete specific keys by name
- **edx_keep_matching_keys**: Keep only matching keys

---

## Notes

- **Version**: v1.23.0
- **Regex Engine**: Go RE2
