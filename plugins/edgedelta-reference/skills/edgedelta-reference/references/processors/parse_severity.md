# Parse Severity (`type: parse_severity`)

> **Full Documentation**: https://docs.edgedelta.com/parse-severity-processor/

## Overview

The `parse_severity` processor extracts and normalizes severity or log level information from log messages or fields. It maps raw severity strings (e.g., "WARN", "error", "CRITICAL") to standardized severity levels on the log record.

## Quick Copy

```yaml
- type: parse_severity
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `parse_severity` |

*See [official documentation](https://docs.edgedelta.com/parse-severity-processor/) for complete parameter reference.*

## Use Cases

1. Normalizing inconsistent severity strings across different log sources into a standard schema
2. Promoting severity level from a log body field to the official severity attribute for filtering and routing
3. Mapping custom application log levels to OpenTelemetry-compatible severity values

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/parse-severity-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/parse-severity-processor/
