# Pack (`type: pack`)

> **Full Documentation**: https://docs.edgedelta.com/pack-processor/

## Overview

The `pack` processor packs multiple attribute fields into a single structured field or encoded string. It is used to consolidate related fields into a compact representation for storage or forwarding.

## Quick Copy

```yaml
- type: pack
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `pack` |

*See [official documentation](https://docs.edgedelta.com/pack-processor/) for complete parameter reference.*

## Use Cases

1. Consolidating multiple log attributes into a single JSON or encoded field before sending to a destination with field limits
2. Reducing schema width by grouping related metadata fields into a nested structure
3. Preparing log records for downstream systems that expect a specific packed or serialized format

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/pack-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/pack-processor/
