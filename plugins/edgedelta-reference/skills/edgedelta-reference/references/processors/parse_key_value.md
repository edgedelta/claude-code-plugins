# Parse Key-Value (`type: parse_key_value`)

> **Full Documentation**: https://docs.edgedelta.com/parse-key-value-processor/

## Overview

The `parse_key_value` processor parses key=value or key:value pairs from log messages or fields into structured attributes. It handles common formats used in syslog, audit logs, and application diagnostic output.

## Quick Copy

```yaml
- type: parse_key_value
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `parse_key_value` |

*See [official documentation](https://docs.edgedelta.com/parse-key-value-processor/) for complete parameter reference.*

## Use Cases

1. Parsing syslog or audit log lines that use key=value pair format
2. Extracting structured fields from application logs with logfmt-style output
3. Converting unstructured key-value text into queryable log attributes

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/parse-key-value-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/parse-key-value-processor/
