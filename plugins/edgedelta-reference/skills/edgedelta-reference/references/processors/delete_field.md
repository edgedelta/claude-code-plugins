# Delete Field (`type: delete_field`)

> **Full Documentation**: https://docs.edgedelta.com/delete-field-processor/

## Overview

The `delete_field` processor removes specified attribute fields from log records. It is commonly used to strip sensitive, redundant, or high-cardinality fields before forwarding data to downstream destinations.

## Quick Copy

```yaml
- type: delete_field
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `delete_field` |

*See [official documentation](https://docs.edgedelta.com/delete-field-processor/) for complete parameter reference.*

## Use Cases

1. Removing PII or sensitive fields (e.g., passwords, tokens, email addresses) before storing logs
2. Dropping high-cardinality or noisy attributes that inflate indexing costs in downstream systems
3. Cleaning up temporary or intermediate fields added by earlier processors in the pipeline

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/delete-field-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/delete-field-processor/
