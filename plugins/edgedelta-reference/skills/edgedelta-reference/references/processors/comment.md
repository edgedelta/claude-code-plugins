# Comment (`type: comment`)

> **Full Documentation**: https://docs.edgedelta.com/comment-processor/

## Overview

The `comment` processor adds human-readable documentation or explanatory notes to pipeline configurations. It has no operational effect on log records and is used purely to annotate complex pipeline sections for maintainability.

## Quick Copy

```yaml
- type: comment
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `comment` |

*See [official documentation](https://docs.edgedelta.com/comment-processor/) for complete parameter reference.*

## Use Cases

1. Adding inline documentation to explain the purpose of a complex processor sequence
2. Annotating configuration sections with change history, ticket references, or author notes
3. Marking pipeline boundaries or logical groupings for team readability

## Notes

- Sequence-compatible: Yes
- The `comment` processor passes all data through unchanged and has no effect on log records, metrics, or traces.
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/comment-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/comment-processor/
