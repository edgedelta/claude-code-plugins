# Parse CSV (`type: parse_csv`)

> **Full Documentation**: https://docs.edgedelta.com/parse-csv-processor/

## Overview

The `parse_csv` processor parses delimited log data (CSV, TSV, or custom delimiters) into structured fields. It maps column values to named attributes based on a defined header row or explicit field names.

## Quick Copy

```yaml
- type: parse_csv
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `parse_csv` |

*See [official documentation](https://docs.edgedelta.com/parse-csv-processor/) for complete parameter reference.*

## Use Cases

1. Parsing application logs exported in CSV format into structured attributes
2. Processing TSV or pipe-delimited log files from legacy systems
3. Normalizing structured delimited data before forwarding to downstream destinations

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/parse-csv-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/parse-csv-processor/
