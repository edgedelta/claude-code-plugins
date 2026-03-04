# Parse Regex (`type: parse_regex`)

> **Full Documentation**: https://docs.edgedelta.com/parse-regex-processor/

## Overview

The `parse_regex` processor parses log fields using named capture groups in regular expression patterns. Each named capture group becomes a structured attribute on the log record.

## Quick Copy

```yaml
- type: parse_regex
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `parse_regex` |

*See [official documentation](https://docs.edgedelta.com/parse-regex-processor/) for complete parameter reference.*

## Use Cases

1. Extracting structured fields from unstructured or semi-structured log lines using regex named capture groups
2. Parsing custom application log formats that do not follow a standard schema
3. Normalizing log records from legacy systems before forwarding to observability platforms

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/parse-regex-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/parse-regex-processor/
