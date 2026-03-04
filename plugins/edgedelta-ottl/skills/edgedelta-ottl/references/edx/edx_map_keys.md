# edx_map_keys - Bulk Key Renaming

**Function Type**: EDX Editor
**Purpose**: Renames multiple keys in one operation using old/new key lists
**Signature**: `edx_map_keys(target, old_keys, new_keys, strategy) -> void`
**Return Type**: void (modifies map in-place)
**Minimum Agent Version**: v1.26.0
**Source**: https://docs.edgedelta.com/ottl-extensions/

---

## Quick Copy

```yaml
# Rename multiple fields at once
edx_map_keys(
  attributes,
  ["old_name1", "old_name2"],
  ["new_name1", "new_name2"],
  "update"
)
```

---

## Overview

edx_map_keys performs bulk field renaming for schema normalization.

**Strategies:**
- `"update"`: Only rename if old key exists (default)
- `"upsert"`: Create new key even if old key missing

---

## Parameters

### target
- **Type**: map
- **Required**: Yes

### old_keys
- **Type**: []string
- **Required**: Yes
- **Description**: Current key names

### new_keys
- **Type**: []string
- **Required**: Yes
- **Description**: New key names (must match length of old_keys)

### strategy
- **Type**: string
- **Required**: No
- **Default**: "update"
- **Valid Values**: "update", "upsert"

---

## Examples

### Example 1: Schema Normalization
```yaml
processors:
  - name: normalize-schema
    transform:
      statements:
        - edx_map_keys(
            attributes,
            ["user_name", "user_email", "user_id"],
            ["username", "email", "id"],
            "update"
          )
```

### Example 2: API Field Mapping
```yaml
processors:
  - name: api-field-mapping
    transform:
      statements:
        - edx_map_keys(
            attributes,
            ["firstName", "lastName", "emailAddress"],
            ["first_name", "last_name", "email"],
            "update"
          )
```

### Example 3: Database Column Mapping
```yaml
processors:
  - name: db-mapping
    transform:
      statements:
        - edx_map_keys(
            attributes,
            ["created_timestamp", "modified_timestamp"],
            ["created_at", "updated_at"],
            "update"
          )
```

### Example 4: Upsert Strategy
```yaml
processors:
  - name: upsert-defaults
    transform:
      statements:
        # Create new keys even if old ones don't exist
        - edx_map_keys(
            attributes,
            ["optional_field1", "optional_field2"],
            ["field1", "field2"],
            "upsert"
          )
```

---

## Common Use Cases

### 1. Vendor Format Normalization
```yaml
# Normalize vendor-specific field names
edx_map_keys(
  attributes,
  ["aws.region", "aws.az", "aws.instance_id"],
  ["cloud.region", "cloud.zone", "cloud.instance_id"],
  "update"
)
```

### 2. Legacy Schema Migration
```yaml
# Migrate old field names to new schema
edx_map_keys(
  attributes,
  ["old_user_field", "legacy_timestamp"],
  ["user_id", "timestamp"],
  "update"
)
```

---

## Validation Rules

1. **Array Lengths Must Match**: old_keys and new_keys must have same length
2. **Valid Strategy**: Must be "update" or "upsert"
3. **Update Behavior**: Only renames if old key exists
4. **Upsert Behavior**: Creates new key regardless (may be nil)

---

## Common Pitfalls

### 1. Mismatched Array Lengths
```yaml
# WRONG: Different lengths
edx_map_keys(
  attributes,
  ["old1", "old2"],
  ["new1"],  # ERROR: length mismatch
  "update"
)

# CORRECT: Same lengths
edx_map_keys(
  attributes,
  ["old1", "old2"],
  ["new1", "new2"],
  "update"
)
```

### 2. Strategy Confusion
```yaml
# "update": If old_key missing, nothing happens
# "upsert": Creates new_key even if old_key missing (value may be nil)
```

---

## Best Practices

### 1. Use Update for Migrations
```yaml
# When renaming existing fields
edx_map_keys(attributes, old_names, new_names, "update")
```

### 2. Document Schema Changes
```yaml
# Comment on schema transformations
# Mapping from legacy schema (v1) to new schema (v2)
edx_map_keys(attributes, legacy_keys, new_keys, "update")
```

### 3. Combine with Validation
```yaml
# Verify fields exist before mapping
set(cache["has_old_fields"], attributes["old_field"] != nil)
edx_map_keys(attributes, old_list, new_list, "update")
```

---

## Performance Considerations

- **Efficient**: Single pass for all renames
- **Better Than**: Multiple individual rename operations
- **O(n)**: Where n = number of keys to rename

---

## Related Functions

- **rename_key**: Single key rename (standard OTTL)
- **edx_delete_keys**: Remove old keys after rename
- **edx_keep_keys**: Filter to specific keys after rename

---

## Notes

- **Version**: v1.26.0
- **Mutation**: Modifies map in-place
- **Atomic**: All renames happen together
- **Order**: Old keys processed in array order
