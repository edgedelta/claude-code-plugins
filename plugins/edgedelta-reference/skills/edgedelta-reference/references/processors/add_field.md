# Add Field (`type: add_field`)

> **Full Documentation**: https://docs.edgedelta.com/add-field-processor/

## Overview

The `add_field` processor adds new attribute fields to log records with static or dynamically computed values. It can insert constant values, reference other fields, or use expressions to derive new field values.

## Quick Copy

```yaml
- type: add_field
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `add_field` |

*See [official documentation](https://docs.edgedelta.com/add-field-processor/) for complete parameter reference.*

## Use Cases

1. Adding environment or deployment metadata (e.g., `env: production`, `region: us-east-1`) to all log records
2. Injecting computed or derived values as new attributes for downstream filtering and routing
3. Enriching log records with static context before forwarding to observability or SIEM platforms

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/add-field-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/add-field-processor/
