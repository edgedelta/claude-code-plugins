# Parse JSON (`type: parse_json`)

> **Full Documentation**: https://docs.edgedelta.com/parse-json-processor/

## Overview

The `parse_json` processor parses a JSON string field into structured attributes on the log record. Unlike `extract_json_field` which extracts a single field, `parse_json` expands the entire JSON object into top-level attributes.

## Quick Copy

```yaml
- type: parse_json
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `parse_json` |

*See [official documentation](https://docs.edgedelta.com/parse-json-processor/) for complete parameter reference.*

## Use Cases

1. Expanding JSON-encoded log bodies into queryable structured attributes
2. Parsing nested JSON payloads from application logs or API responses
3. Normalizing JSON-formatted log records before routing to storage destinations

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/parse-json-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/parse-json-processor/
