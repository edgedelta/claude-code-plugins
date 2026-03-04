# Parse Timestamp (`type: parse_timestamp`)

> **Full Documentation**: https://docs.edgedelta.com/parse-timestamp-processor/

## Overview

The `parse_timestamp` processor parses and normalizes timestamp formats from log messages or fields. It extracts a timestamp from a specified field and sets it as the canonical timestamp on the log record.

## Quick Copy

```yaml
- type: parse_timestamp
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `parse_timestamp` |

*See [official documentation](https://docs.edgedelta.com/parse-timestamp-processor/) for complete parameter reference.*

## Use Cases

1. Correcting log timestamps that arrive out of order or with incorrect wall-clock times
2. Parsing custom or non-standard timestamp formats from application logs into RFC 3339 or Unix epoch
3. Promoting an embedded timestamp field from the log body to the official record timestamp

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/parse-timestamp-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/parse-timestamp-processor/
