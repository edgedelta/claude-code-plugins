# Conditional Group (`type: conditional_group`)

> **Full Documentation**: https://docs.edgedelta.com/conditional-group-processor/

## Overview

The `conditional_group` processor groups processors conditionally, applying different processor chains based on evaluated conditions. It allows branching logic within a sequence, routing records through different sub-chains without requiring a top-level route node.

## Quick Copy

```yaml
- type: conditional_group
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `conditional_group` |

*See [official documentation](https://docs.edgedelta.com/conditional-group-processor/) for complete parameter reference.*

## Use Cases

1. Applying different enrichment processor chains to log records based on their source or severity
2. Conditionally running a set of parse and transform processors only when a specific field is present
3. Implementing if/else branching within a sequence without splitting the pipeline into separate top-level branches

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/conditional-group-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/conditional-group-processor/
