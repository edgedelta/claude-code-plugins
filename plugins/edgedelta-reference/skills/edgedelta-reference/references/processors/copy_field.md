# Copy Field (`type: copy_field`)

> **Full Documentation**: https://docs.edgedelta.com/copy-field-processor/

## Overview

The `copy_field` processor copies the value of one attribute field to another field on the log record. The source field is preserved and the destination field is created or overwritten with the copied value.

## Quick Copy

```yaml
- type: copy_field
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `copy_field` |

*See [official documentation](https://docs.edgedelta.com/copy-field-processor/) for complete parameter reference.*

## Use Cases

1. Duplicating a field under a normalized attribute name expected by a downstream destination
2. Preserving the original value of a field before it is transformed by a subsequent processor
3. Aliasing fields to match schema requirements of different output connectors

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/copy-field-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/copy-field-processor/
