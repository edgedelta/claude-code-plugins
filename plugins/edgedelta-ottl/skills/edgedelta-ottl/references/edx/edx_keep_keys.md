# edx_keep_keys - Keep Only Specified Keys

**Function Type**: EDX Editor
**Purpose**: Deletes all keys except those specified (inverse of edx_delete_keys)
**Signature**: `edx_keep_keys(target, keys) -> void`
**Return Type**: void (modifies map in-place)
**Minimum Agent Version**: v1.23.0
**Source**: /pkg/ottlext/funcs/func_edx_keep_keys.go

---

## Quick Copy

```yaml
# Keep only essential fields
edx_keep_keys(attributes, ["user_id", "timestamp", "event_type"])
```

---

## Overview

edx_keep_keys is a whitelist-based field filter that removes all keys except those specified.

**Use Cases:**
- Schema enforcement
- Field whitelisting for exports
- Data minimization
- Compliance filtering

---

## Parameters

### target
- **Type**: map
- **Required**: Yes
- **Description**: Map to filter

### keys
- **Type**: []string
- **Required**: Yes
- **Description**: Keys to keep (all others deleted)

---

## Examples

### Example 1: Keep Core Fields
```yaml
processors:
  - name: keep-core
    transform:
      statements:
        - edx_keep_keys(attributes, ["id", "timestamp", "message"])
```

### Example 2: Export Whitelist
```yaml
processors:
  - name: export-whitelist
    transform:
      statements:
        - edx_keep_keys(attributes, [
            "user_id",
            "event_name",
            "timestamp",
            "value"
          ])
```

### Example 3: Compliance Filtering
```yaml
processors:
  - name: gdpr-whitelist
    transform:
      statements:
        # Keep only non-PII fields
        - edx_keep_keys(attributes, [
            "session_id",
            "event_type",
            "timestamp",
            "product_id"
          ])
```

---

## Common Use Cases

### 1. Schema Enforcement
```yaml
# Ensure output contains only expected fields
edx_keep_keys(attributes, [
  "id",
  "name",
  "created_at",
  "updated_at"
])
```

### 2. Third-Party Integration
```yaml
# Send only required fields to external API
edx_keep_keys(attributes, [
  "external_id",
  "event_type",
  "timestamp",
  "payload"
])
```

---

## Validation Rules

1. **Target Must Be Map**: First parameter must be a map
2. **Keys Must Be Array**: Second parameter must be array
3. **Missing Keys OK**: Specified keys don't need to exist

---

## Best Practices

1. **Define Schema Explicitly**: List all expected fields
2. **Use for Whitelisting**: Safer than blacklisting for compliance
3. **Combine with Validation**: Verify required keys exist

---

## Related Functions

- **edx_delete_keys**: Blacklist approach
- **edx_keep_matching_keys**: Pattern-based keeping

---

## Notes

- **Version**: v1.23.0
- **Inverse Operation**: Opposite of edx_delete_keys
- **Safe**: Doesn't error if keys don't exist
