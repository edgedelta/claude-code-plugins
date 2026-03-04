# delete_empty_values Processor

**Type**: `delete_empty_values`
**Category**: Masking & Privacy / Data Manipulation
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/deleteemptyvalues/processor.go`
**Config Struct**: `configv3.DeleteEmptyValues`

## Quick Copy

```yaml
- type: delete_empty_values
  delete_empty_strings: true
  delete_empty_nulls: true
  delete_empty_lists: true
  delete_empty_maps: true
```

## Overview

The `delete_empty_values` processor removes empty, null, or unwanted values from telemetry attributes and resource fields. This processor helps clean up telemetry data by removing meaningless values, reducing payload sizes, and improving data quality. It recursively processes maps, lists, and nested structures to eliminate empty values based on configurable rules.

## Use Cases

- **Data Quality**: Remove placeholder values like empty strings, "N/A", "null", "-" from telemetry
- **Payload Size Reduction**: Decrease storage and transmission costs by eliminating empty fields
- **Log Normalization**: Standardize logs by removing inconsistent empty value representations
- **PII Compliance**: Clean up partially masked or empty PII fields before forwarding
- **Cost Optimization**: Reduce destination ingestion costs by sending only meaningful data
- **Downstream Processing**: Simplify downstream analytics by removing noise from telemetry
- **Schema Cleanup**: Remove empty nested objects and arrays that add no value
- **Metric Cardinality**: Reduce metric label cardinality by removing empty tag values

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `delete_empty_values` |
| At least one of | bool | Must enable at least one deletion flag (see below) |

**Note**: You must enable at least one of the deletion flags (`delete_empty_strings`, `delete_empty_nulls`, `delete_empty_lists`, `delete_empty_maps`) or provide `strings_to_delete`.

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `delete_empty_strings` | bool | false | Delete attributes with empty string values (`""`) |
| `delete_empty_nulls` | bool | false | Delete attributes with null/nil values |
| `delete_empty_lists` | bool | false | Delete attributes with empty arrays/lists (`[]`) |
| `delete_empty_maps` | bool | false | Delete attributes with empty maps/objects (`{}`) |
| `strings_to_delete` | []string | [] | List of specific string values to delete (e.g., `["N/A", "null", "-", "n/a"]`) |

### Optional Parameters (from SequenceProcessor)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `final` | bool | false | Mark as last processor in sequence (required for final processor) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## What Gets Deleted

The processor recursively traverses telemetry data and deletes:

### 1. Empty Strings (`delete_empty_strings: true`)
- Attributes with value `""`
- When enabled, automatically adds `""` to `strings_to_delete`

### 2. Null Values (`delete_empty_nulls: true`)
- Attributes with `nil` values
- Attributes with explicit `null` in JSON
- Invalid or uninitialized values

### 3. Empty Lists (`delete_empty_lists: true`)
- Attributes with empty arrays: `[]`
- After recursive cleanup, lists that become empty are also removed

### 4. Empty Maps (`delete_empty_maps: true`)
- Attributes with empty objects: `{}`
- After recursive cleanup, maps that become empty are also removed

### 5. Custom Strings (`strings_to_delete`)
- Matches specified strings case-insensitively
- Common examples: `"N/A"`, `"null"`, `"NULL"`, `"-"`, `"undefined"`

## Data Processing Scope

The processor operates on three telemetry fields:

1. **`resource`** - Resource attributes (e.g., `service.name`, `host.name`)
2. **`attributes`** - Telemetry-specific attributes (logs, metrics, traces)
3. **`body`** - Log body field (can be deleted if empty)

**Recursive Processing**: The processor recursively traverses nested maps and lists, cleaning values at all levels.

## Examples

### Example 1: Delete All Empty Values (Default Cleanup)

```yaml
- type: delete_empty_values
  delete_empty_strings: true
  delete_empty_nulls: true
  delete_empty_lists: true
  delete_empty_maps: true
```

**What it does**: Removes all empty strings, nulls, empty lists, and empty maps from telemetry.

**Input**:
```json
{
  "attributes": {
    "user_id": "12345",
    "user_name": "",
    "email": null,
    "tags": [],
    "metadata": {}
  }
}
```

**Output**:
```json
{
  "attributes": {
    "user_id": "12345"
  }
}
```

### Example 2: Delete Only Empty Strings

```yaml
- type: delete_empty_values
  delete_empty_strings: true
```

**What it does**: Removes only empty string values, preserving nulls and empty collections.

**Input**:
```json
{
  "attributes": {
    "status": "active",
    "description": "",
    "notes": null,
    "tags": []
  }
}
```

**Output**:
```json
{
  "attributes": {
    "status": "active",
    "notes": null,
    "tags": []
  }
}
```

### Example 3: Delete Only Nulls

```yaml
- type: delete_empty_values
  delete_empty_nulls: true
```

**What it does**: Removes null values while preserving empty strings and collections.

**Input**:
```json
{
  "attributes": {
    "user": "alice",
    "email": null,
    "phone": null,
    "address": ""
  }
}
```

**Output**:
```json
{
  "attributes": {
    "user": "alice",
    "address": ""
  }
}
```

### Example 4: Delete Only Empty Lists

```yaml
- type: delete_empty_values
  delete_empty_lists: true
```

**What it does**: Removes empty arrays while preserving other empty values.

**Input**:
```json
{
  "attributes": {
    "tags": [],
    "labels": ["production"],
    "empty_str": "",
    "null_val": null
  }
}
```

**Output**:
```json
{
  "attributes": {
    "labels": ["production"],
    "empty_str": "",
    "null_val": null
  }
}
```

### Example 5: Delete Only Empty Maps

```yaml
- type: delete_empty_values
  delete_empty_maps: true
```

**What it does**: Removes empty objects/maps while preserving other empty values.

**Input**:
```json
{
  "attributes": {
    "metadata": {},
    "config": {"setting": "value"},
    "empty_str": "",
    "tags": []
  }
}
```

**Output**:
```json
{
  "attributes": {
    "config": {"setting": "value"},
    "empty_str": "",
    "tags": []
  }
}
```

### Example 6: Custom Strings to Delete

```yaml
- type: delete_empty_values
  strings_to_delete:
    - "N/A"
    - "null"
    - "NULL"
    - "-"
    - "undefined"
    - "n/a"
```

**What it does**: Removes specific placeholder strings (case-insensitive matching).

**Input**:
```json
{
  "attributes": {
    "status": "active",
    "email": "N/A",
    "phone": "-",
    "address": "null",
    "notes": "undefined"
  }
}
```

**Output**:
```json
{
  "attributes": {
    "status": "active"
  }
}
```

### Example 7: Selective Deletion with Boolean Flags

```yaml
- type: delete_empty_values
  delete_empty_strings: true
  delete_empty_nulls: true
  strings_to_delete:
    - "N/A"
    - "-"
```

**What it does**: Combines empty string/null deletion with custom string deletion.

**Input**:
```json
{
  "attributes": {
    "user": "alice",
    "email": "",
    "phone": null,
    "address": "N/A",
    "city": "-",
    "country": "USA"
  }
}
```

**Output**:
```json
{
  "attributes": {
    "user": "alice",
    "country": "USA"
  }
}
```

### Example 8: Combined with Other Processors (Sequence)

```yaml
- name: cleanup_sequence
  type: sequence
  user_description: Mask PII and clean empty values
  processors:
    - type: generic_mask
      capture_group_masks:
        - capture_group: "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"
          enabled: true
          mask: "***EMAIL***"
          name: "email"
    - type: delete_empty_values
      delete_empty_strings: true
      delete_empty_nulls: true
      strings_to_delete:
        - "N/A"
        - "null"
      final: true
```

**What it does**:
1. Masks email addresses with `***EMAIL***`
2. Removes empty strings, nulls, and placeholder values
3. Final processor in sequence

### Example 9: Nested Structure Cleanup

```yaml
- type: delete_empty_values
  delete_empty_strings: true
  delete_empty_nulls: true
  delete_empty_maps: true
  delete_empty_lists: true
```

**What it does**: Recursively cleans nested objects and arrays.

**Input**:
```json
{
  "attributes": {
    "user": {
      "name": "alice",
      "email": "",
      "metadata": {
        "tags": [],
        "notes": null
      }
    }
  }
}
```

**Output**:
```json
{
  "attributes": {
    "user": {
      "name": "alice"
    }
  }
}
```

**Note**: The nested `metadata` object becomes empty after cleanup and is removed.

### Example 10: Conditional Processing (Production Only)

```yaml
- type: delete_empty_values
  condition: 'attributes["environment"] == "production"'
  delete_empty_strings: true
  delete_empty_nulls: true
  delete_empty_lists: true
  delete_empty_maps: true
```

**What it does**: Only cleans empty values from production logs.

## Validation Rules

1. **Required Deletion Flag**: Must enable at least one of `delete_empty_strings`, `delete_empty_nulls`, `delete_empty_lists`, `delete_empty_maps`, or provide `strings_to_delete`
2. **Boolean Types**: Deletion flags must be boolean values (`true` or `false`)
3. **String Array**: `strings_to_delete` must be an array of strings
4. **Valid OTTL**: If using `condition`, it must be valid OTTL syntax
5. **No Body Exclusion**: Cannot exclude the `body` field from processing (validation error)

## Common Pitfalls

### 1. No Deletion Flags Enabled

**Problem**: Not enabling any deletion flags causes validation error.

**Wrong**:
```yaml
- type: delete_empty_values
  # No flags enabled - will fail validation
```

**Correct**:
```yaml
- type: delete_empty_values
  delete_empty_strings: true
```

### 2. Deleting Needed Empty Values

**Problem**: Removing empty strings that have semantic meaning (e.g., "empty description" vs "no description").

**Guidance**: Consider whether empty values carry meaning in your use case. For example:
- Empty string might mean "explicitly cleared"
- Null might mean "never set"

**Solution**: Use selective flags to preserve meaningful empty values:
```yaml
- type: delete_empty_values
  delete_empty_nulls: true  # Only delete nulls, keep empty strings
```

### 3. strings_to_delete Configuration

**Problem**: Not understanding case-insensitive matching.

**Important**: String matching is **case-insensitive**.

```yaml
strings_to_delete:
  - "N/A"  # Matches "N/A", "n/a", "N/a", "n/A"
```

**Best Practice**: Only specify one case variant:
```yaml
strings_to_delete:
  - "N/A"      # No need to add "n/a"
  - "null"     # No need to add "NULL"
  - "undefined"
```

### 4. Boolean Flag Combinations

**Problem**: Confusion about which flags to enable.

**Guidance**:
- **Start conservative**: Enable only what you need
- **Test thoroughly**: Verify you're not removing needed data
- **Common combination**: `delete_empty_strings: true` + `delete_empty_nulls: true`

**Example**:
```yaml
# Conservative - only empty strings and nulls
- type: delete_empty_values
  delete_empty_strings: true
  delete_empty_nulls: true

# Aggressive - everything
- type: delete_empty_values
  delete_empty_strings: true
  delete_empty_nulls: true
  delete_empty_lists: true
  delete_empty_maps: true
```

### 5. Recursive Behavior

**Problem**: Not understanding that parent structures become empty after child cleanup.

**Example**:
```json
Input:  {"metadata": {"tags": []}}
Output: {}  // metadata removed because it became empty
```

**Important**: If `delete_empty_maps: true`, maps that become empty after recursive cleanup are also removed.

### 6. Body Field Deletion

**Problem**: Expecting to preserve body field when it's empty.

**Behavior**: If body is empty and matches deletion criteria, the entire `body` field is removed from the telemetry item.

**Impact**: Some destinations may expect a body field. Test carefully.

### 7. Forgetting final Flag

**Problem**: Not marking the last processor in a sequence.

**Wrong**:
```yaml
processors:
  - type: generic_mask
    # ...
  - type: delete_empty_values
    delete_empty_strings: true
    # Missing final: true
```

**Correct**:
```yaml
processors:
  - type: generic_mask
    # ...
  - type: delete_empty_values
    delete_empty_strings: true
    final: true
```

### 8. Over-Cleaning

**Problem**: Removing too much data and losing context.

**Example**: Removing empty error messages that indicate "no error occurred".

**Solution**: Be selective about what you clean:
```yaml
# Don't clean error_message field even if empty
- type: delete_empty_values
  delete_empty_nulls: true
  # Keep empty strings for semantic meaning
```

## Best Practices

1. **Start Conservative**: Begin with only `delete_empty_strings` and `delete_empty_nulls`, add more as needed
2. **Test Before Production**: Validate on sample data to ensure no critical fields are removed
3. **Document Intent**: Use `comment` field to explain why specific values are being deleted
4. **Combine with Masking**: Use after PII masking to clean up masked/redacted fields
5. **Monitor Impact**: Track payload size reduction to measure effectiveness
6. **Use Conditions**: Apply cleanup selectively using OTTL conditions for environment-specific rules
7. **strings_to_delete**: Create a standard list of placeholder values for your organization (e.g., "N/A", "-", "null")
8. **Placement in Sequence**: Place near the end of processing pipeline, after transformations
9. **Performance**: This processor is very efficient with minimal overhead
10. **Destination Compatibility**: Verify downstream systems handle missing fields gracefully

## Performance Considerations

- **Minimal Overhead**: The processor uses efficient reflection-based traversal
- **Recursive Processing**: Deep nesting has minor performance impact but is generally negligible
- **Memory**: No additional memory allocation for deleted fields (reduces overall memory usage)
- **Throughput**: Virtually no impact on throughput - this is a lightweight processor
- **Payload Size**: Can significantly reduce payload size (10-40% reduction common)

**Performance Profile**:
- **Best Case**: Shallow structures with many empty values - fast and high impact
- **Worst Case**: Deep nesting with no empty values - minimal overhead
- **Typical Case**: Moderate nesting with 10-20% empty values - <1% CPU impact

## Related Processors

- **generic_mask**: Mask PII before cleaning empty values
- **ottl_transform**: Transform attributes before deletion
- **ottl_filter**: Filter out entire items instead of just empty fields
- **compact**: Similar functionality for protocol buffer messages
- **sample**: Reduce data volume through sampling instead of field removal

## Cross-References

- **edgedelta-pipelines skill**: Template 1 (Log PII Masking) - cleaning after masking
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`
- **OTTL Reference**: https://docs.edgedelta.com/ottl-statements/ (for `condition` parameter)

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/delete-empty-field-processor/

## Notes

- String matching in `strings_to_delete` is **case-insensitive** (uses `strings.EqualFold`)
- The processor processes three fields: `resource`, `attributes`, and `body`
- Empty collections are removed **after** recursive cleanup of their contents
- When `delete_empty_strings: true`, an empty string `""` is automatically added to `strings_to_delete`
- The processor recalculates item size after deletion to reflect reduced payload
- Cannot use `excluded_field_paths` to exclude `body` field (validation error)
- Deleted fields are permanently removed - data cannot be recovered downstream
- Works with all telemetry types: logs, metrics, and traces
- The processor is sequence-compatible and can be used with `final: true`
- Deletion is atomic per item - either all matching values are deleted or none (no partial failure)
