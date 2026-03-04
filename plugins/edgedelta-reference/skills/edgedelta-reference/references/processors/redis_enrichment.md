# Redis Enrichment (`type: redis_enrichment`)

> **Full Documentation**: https://docs.edgedelta.com/redis-enrichment-processor/

## Overview

The `redis_enrichment` processor enriches log records with data retrieved from a Redis cache lookup. A field value from the log record is used as the lookup key, and the returned Redis value is added as a new attribute on the record.

## Quick Copy

```yaml
- type: redis_enrichment
  # Add required parameters here
```

## Key Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Must be `redis_enrichment` |

*See [official documentation](https://docs.edgedelta.com/redis-enrichment-processor/) for complete parameter reference.*

## Use Cases

1. Enriching log records with user metadata (e.g., account tier, region) by looking up a user ID in Redis
2. Adding service ownership or team routing information to log records from a Redis-backed registry
3. Appending geolocation or classification data to IP addresses or request identifiers at pipeline time

## Notes

- Sequence-compatible: Yes
- For full parameter reference, see the [official documentation](https://docs.edgedelta.com/redis-enrichment-processor/).

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/redis-enrichment-processor/
