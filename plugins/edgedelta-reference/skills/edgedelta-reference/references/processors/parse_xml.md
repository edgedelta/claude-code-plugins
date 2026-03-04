# Parse XML (`type: parse_xml`)

> **Full Documentation**: https://docs.edgedelta.com/parse-xml-processor/

## Overview

The `parse_xml` processor parses XML-formatted log data into structured attributes. It traverses the XML document structure and maps element values and attributes to named fields on the log record.

## Quick Copy

```yaml
- type: parse_xml
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `parse_xml` |

*See [official documentation](https://docs.edgedelta.com/parse-xml-processor/) for complete parameter reference.*

## Use Cases

1. Parsing XML-formatted application logs or event records from Java/.NET services
2. Processing structured XML payloads received from legacy enterprise systems
3. Extracting specific element values from XML log bodies into queryable attributes

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/parse-xml-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/parse-xml-processor/
