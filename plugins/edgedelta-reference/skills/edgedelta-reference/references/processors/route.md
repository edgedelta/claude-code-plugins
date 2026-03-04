# Route (`type: route`)

> **Full Documentation**: https://docs.edgedelta.com/route-node/

## Overview

The `route` node sends data to different pipeline branches based on conditions. It is a top-level routing construct that directs log records, metrics, or traces to separate downstream processor chains or output destinations depending on evaluated conditions.

## Quick Copy

```yaml
- type: route
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `route` |

*See [official documentation](https://docs.edgedelta.com/route-node/) for complete parameter reference.*

## Use Cases

1. Routing critical error logs to a high-priority alerting pipeline while sending informational logs to cold storage
2. Splitting a mixed telemetry stream into separate paths for metrics, logs, and traces
3. Directing logs from different services or environments to service-specific output destinations

## Notes

- Sequence-compatible: No (top-level routing)
- The `route` node is a pipeline topology construct, not a sequence processor. It cannot be placed inside a processor sequence.
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/route-node/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/route-node/
