# Stateful Alert (`type: stateful_alert`)

> **Full Documentation**: https://docs.edgedelta.com/stateful-alert-processor/

## Overview

The `stateful_alert` processor generates alerts based on stateful conditions evaluated over time. It tracks state across log records and fires alerts when thresholds, durations, or multi-step conditions are met, supporting scenarios like error rate spikes or sustained anomalies.

## Quick Copy

```yaml
- type: stateful_alert
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `stateful_alert` |

*See [official documentation](https://docs.edgedelta.com/stateful-alert-processor/) for complete parameter reference.*

## Use Cases

1. Alerting when an error rate exceeds a threshold sustained over a configurable time window
2. Detecting multi-step failure patterns (e.g., retry followed by failure) across sequential log records
3. Generating resolved/cleared notifications when a previously triggered condition returns to normal

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/stateful-alert-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/stateful-alert-processor/
