# Rollup Metric (`type: rollup_metric`)

> **Full Documentation**: https://docs.edgedelta.com/rollup-metric-processor/

## Overview

The `rollup_metric` processor rolls up metric data points over a time window for aggregation. It is designed to consolidate high-frequency metric emissions into lower-frequency summary metrics, reducing volume while preserving statistical properties.

## Quick Copy

```yaml
- type: rollup_metric
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `rollup_metric` |

*See [official documentation](https://docs.edgedelta.com/rollup-metric-processor/) for complete parameter reference.*

## Use Cases

1. Rolling up per-second metric emissions into per-minute summaries to reduce storage and ingestion costs
2. Pre-aggregating high-cardinality metric streams before forwarding to downstream monitoring systems
3. Creating multi-resolution metric views (1m, 5m, 1h) from a single high-frequency source

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/rollup-metric-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/rollup-metric-processor/
