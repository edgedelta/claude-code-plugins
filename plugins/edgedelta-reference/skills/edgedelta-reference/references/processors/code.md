# Code (`type: code`)

> **Full Documentation**: https://docs.edgedelta.com/code-processor/

## Overview

The `code` processor executes custom code (such as Lua or a similar embedded scripting language) to transform log records. It provides an escape hatch for complex transformations that cannot be expressed with built-in processors.

## Quick Copy

```yaml
- type: code
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `code` |

*See [official documentation](https://docs.edgedelta.com/code-processor/) for complete parameter reference.*

## Use Cases

1. Implementing complex multi-step transformations or business logic not supported by built-in processors
2. Performing conditional field manipulation based on intricate cross-field logic
3. Prototyping custom enrichment or filtering logic before it is promoted to a native processor

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/code-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/code-processor/
