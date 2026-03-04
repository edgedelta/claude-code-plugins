# EdgeDelta v3 Pipeline Component Master Index

**Purpose**: Quick lookup for all EdgeDelta v3 pipeline components — sources, processors, and destinations
**Last Updated**: 2026-03-03
**Coverage**: 30 sources | 38 processors | 54 destinations

## Navigation

- [Sources (30)](#sources) → [references/sources/index.md](references/sources/index.md)
- [Processors (38)](#processors) → [references/processors/](references/processors/)
- [Destinations (54)](#destinations) → [references/destinations/index.md](references/destinations/index.md)

---

## Sources

See [references/sources/index.md](references/sources/index.md) for the complete index with YAML types, key parameters, and docs links.

**30 Sources by Category**:

| Category | Count | Sources |
|---|---|---|
| Log | 14 | CrowdStrike FDR, Docker, Event Hub, Exec, File, Filebeat, Fluentd, Journald, Kafka, Kubernetes, Kubernetes Event, Splunk HEC, Splunk TCP, Windows Event |
| Metric | 3 | Kubernetes Metrics, Kubernetes Service Map, Prometheus |
| Trace | 1 | Kubernetes Trace |
| Cloud Native | 2 | Google Cloud Pub/Sub, S3 |
| Hybrid | 10 | Datadog, HTTP, OTLP, Pipeline, TCP, Telemetry Generator, UDP, HTTP Pull, Syslog, HTTP Workflow |

---

## Destinations

See [references/destinations/index.md](references/destinations/index.md) for the complete index with YAML types, key parameters, and docs links.

**54 Destinations by Category**:

| Category | Count | Destinations |
|---|---|---|
| Observability Platforms | 17 | Edge Delta, CloudWatch, Azure Log Analytics, CrowdStrike LogScale, Datadog, Debug, Dynatrace, Elastic, Fluentd, GCP Logging, Grokstream, Loki, New Relic, Splunk HEC, Splunk TCP, Sumo Logic, VictoriaMetrics |
| Storage & Data Lakes | 12 | Apache Kudu, Azure Blob, BigQuery, ClickHouse, DigitalOcean Spaces, GCS, IBM Object Storage, Local Storage, MinIO, S3, Snowflake, Zenko |
| Security & SIEM | 5 | Exabeam, IBM QRadar, Microsoft Sentinel, Securonix, Google SecOps |
| Custom & Streaming | 20 | HTTP, Kafka, Microsoft Teams, OpenMetrics, OTLP, Prometheus Exporter, Prometheus Remote Write, Slack, TCP, Webhook, PagerDuty, OpsGenie, Humio, SignalFx, AppDynamics, Syslog, Cribl, LogDNA, Mezmo, OpenSearch |

---

## Processors

### How to Use This Section

1. **Find Your Processor**: Browse by category or search by name
2. **Copy Quick Snippet**: Use the minimal valid example
3. **Read Full Reference**: Link to detailed documentation if available
4. **Combine in Sequences**: All processors marked ✓ sequence can be used inside `sequence` nodes

### Detailed References Available

✓ **Full Documentation** (400-1000+ lines with examples, validation, pitfalls):
- [generic_mask](references/processors/generic_mask.md)
- [extract_metric](references/processors/extract_metric.md)
- [ottl_transform](references/processors/ottl_transform.md)
- [json_unroll](references/processors/json_unroll.md)
- [log_to_pattern_metric](references/processors/log_to_pattern_metric.md)
- [lookup](references/processors/lookup.md)
- [grok](references/processors/grok.md)
- [suppress](references/processors/suppress.md)
- [dedup](references/processors/dedup.md)
- [ottl_context_filter](references/processors/ottl_context_filter.md)
- [rate_limit](references/processors/rate_limit.md)
- [tail_sample](references/processors/tail_sample.md)
- [aggregate_metric](references/processors/aggregate_metric.md)
- [cumulative_to_delta](references/processors/cumulative_to_delta.md)
- [ottl_filter](references/processors/ottl_filter.md)
- [sample](references/processors/sample.md)
- [delete_empty_values](references/processors/delete_empty_values.md)
- [extract_json_field](references/processors/extract_json_field.md)
- [split_with_delimiter](references/processors/split_with_delimiter.md)
- [deotel](references/processors/deotel.md)
- [sequence](references/processors/sequence.md)
- [compound](references/processors/compound.md)
- [sequence_input/output](references/processors/sequence_input_output.md) (internal reference)

◎ **Stub Documentation** (doc link + basic example, full depth is a follow-up):
- [parse_csv](references/processors/parse_csv.md) | [parse_json](references/processors/parse_json.md) | [parse_key_value](references/processors/parse_key_value.md) | [parse_regex](references/processors/parse_regex.md) | [parse_severity](references/processors/parse_severity.md)
- [parse_timestamp](references/processors/parse_timestamp.md) | [parse_xml](references/processors/parse_xml.md)
- [add_field](references/processors/add_field.md) | [copy_field](references/processors/copy_field.md) | [delete_field](references/processors/delete_field.md) | [pack](references/processors/pack.md)
- [redis_enrichment](references/processors/redis_enrichment.md) | [rollup_metric](references/processors/rollup_metric.md) | [stateful_alert](references/processors/stateful_alert.md)
- [route](references/processors/route.md) | [code](references/processors/code.md) | [comment](references/processors/comment.md) | [conditional_group](references/processors/conditional_group.md)

## All 38 Processors (23 Sequence-Compatible + 15 New)

### Core Sequence-Compatible Processors (23)

### Core Transformation & Filtering

#### 1. ottl_transform
**Purpose**: Transform data using OTTL (OpenTelemetry Transformation Language)
**Full Reference**: [ottl_transform.md](references/processors/ottl_transform.md)

```yaml
- type: ottl_transform
  statements: |
    set(attributes["processed"], "true")
    set(attributes["timestamp"], Now())
```

**Common Use Cases**:
- Add/modify attributes
- Parse JSON from body
- Field renaming and normalization
- Conditional transformations

---

#### 2. ottl_filter
**Purpose**: Filter (drop) telemetry items based on OTTL conditions
**Full Reference**: [ottl_filter.md](references/processors/ottl_filter.md)

```yaml
- type: ottl_filter
  filter_mode: exclude
  error_mode: ignore
  statements:
    - 'attributes["level"] == "DEBUG"'
    - 'IsMatch(body, "(?i)TRACE")'
```

**Common Use Cases**:
- Drop debug/trace logs (exclude mode)
- Keep only errors/warnings (include mode)
- Remove noisy data sources

---

#### 3. ottl_context_filter
**Purpose**: Collect surrounding log context when triggers fire (debugging)
**Full Reference**: [ottl_context_filter.md](references/processors/ottl_context_filter.md)

```yaml
- type: ottl_context_filter
  ottl_context_filter:
    condition: 'IsMatch(body, "(?i)ERROR")'
    length: 50
  ottl_context_filter_context:
    condition: 'attributes["service.name"] == "api"'
```

**Common Use Cases**:
- Capture logs around errors for debugging
- Incident investigation with full context
- Root cause analysis

---

### Masking & Privacy

#### 4. generic_mask
**Purpose**: Regex-based masking of sensitive data (PII, credentials)
**Full Reference**: [generic_mask.md](references/processors/generic_mask.md)

```yaml
- type: generic_mask
  capture_group_masks:
    - capture_group: "(?i)(password|passwd|pwd)[:=]\\S+"
      enabled: true
      mask: "***PASSWORD***"
      name: "password"
```

**Common Use Cases**:
- Mask passwords, API tokens, credentials
- Redact emails, credit cards, SSNs
- PII compliance (GDPR, HIPAA)

---

#### 5. delete_empty_values
**Purpose**: Remove nil/empty attributes from telemetry
**Full Reference**: [delete_empty_values.md](references/processors/delete_empty_values.md)

```yaml
- type: delete_empty_values
  delete_empty_strings: true
  delete_empty_nulls: true
  delete_empty_lists: true
  delete_empty_maps: true
  strings_to_delete:
    - "N/A"
    - "null"
    - "-"
```

**Common Use Cases**:
- Clean up sparse data and remove placeholders
- Remove unset optional fields
- Reduce payload size by 10-40%

---

### Sampling & Rate Limiting

#### 6. sample
**Purpose**: Probabilistic hash-based sampling by percentage
**Full Reference**: [sample.md](references/processors/sample.md)

```yaml
- type: sample
  percentage: 10
  field_paths:
    - "attributes.trace_id"
```

**Common Use Cases**:
- Reduce high-volume logs (sample 10%)
- Cost optimization with consistent trace sampling
- Representative sampling for analysis

**Parameters**:
- `percentage`: 1-100 (10 = 10% sampling)
- `field_paths`: Hash fields for deterministic consistency

---

#### 7. tail_sample
**Purpose**: Tail-based trace sampling (keep errors, sample successful)
**Full Reference**: [tail_sample.md](references/processors/tail_sample.md)

```yaml
- type: tail_sample
  decision_interval: 10s
  sampling_policies:
    - name: errors
      type: status_code
      status_code:
        status_codes:
          - ERROR
    - name: sample_successful
      type: probabilistic
      probabilistic:
        sampling_percentage: 10
```

**Common Use Cases**:
- Keep all error traces, sample 10% of successful
- Intelligent trace sampling by latency
- SLI-based sampling (status codes, attributes)

---

#### 8. rate_limit
**Purpose**: Rate limiting to cap throughput (fixed window counter)
**Full Reference**: [rate_limit.md](references/processors/rate_limit.md)

```yaml
- type: rate_limit
  item_count_limit: 1000
  interval: 1s
```

**Common Use Cases**:
- Protect downstream systems from overload
- Cap egress costs and control spending
- Prevent API rate limit violations

---

#### 9. suppress
**Purpose**: Suppress duplicate events based on key fields
**Full Reference**: [suppress.md](references/processors/suppress.md)

```yaml
- type: suppress
  key_field_paths:
    - "attributes.error_id"
    - "attributes.hostname"
```

**Common Use Cases**:
- Deduplicate repeated errors
- Suppress chatty log sources
- Reduce noise from periodic events

---

### Metrics Extraction & Conversion

#### 10. extract_metric
**Purpose**: Extract metrics from logs/traces using OTTL conditions
**Full Reference**: [extract_metric.md](references/processors/extract_metric.md)

```yaml
- type: extract_metric
  extract_metric_rules:
    - name: "errors_total"
      unit: "1"
      conditions:
        - 'IsMatch(body, "(?i)ERROR")'
      sum:
        aggregation_temporality: delta
        is_monotonic: true
        value: 1
  interval: 1m
```

**Common Use Cases**:
- Convert logs to metrics (log-to-metric)
- Count errors, requests, events
- Extract latency, duration metrics

---

#### 11. aggregate_metric
**Purpose**: Aggregate and roll up metrics by dimensions
**Full Reference**: [aggregate_metric.md](references/processors/aggregate_metric.md)

```yaml
- type: aggregate_metric
  aggregate_metric_rules:
    - name: "requests_by_endpoint"
      conditions:
        - 'attributes["metric.name"] == "http_requests"'
      aggregation_type: sum
      group_by:
        - "attributes.endpoint"
  interval: 1m
```

**Common Use Cases**:
- Roll up metrics by dimension (host, service, endpoint)
- Calculate percentiles (P90, P95, P99)
- Cardinality reduction with group_by

---

#### 12. cumulative_to_delta
**Purpose**: Convert cumulative counters to delta/rate metrics
**Full Reference**: [cumulative_to_delta.md](references/processors/cumulative_to_delta.md)

```yaml
- type: cumulative_to_delta
```

**Common Use Cases**:
- Convert cumulative counters to delta rates
- Counter reset detection and handling
- Normalize metrics from different sources (Prometheus, OTLP)

---

#### 13. log_to_pattern_metric
**Purpose**: Pattern clustering using DRAIN algorithm with metrics
**Full Reference**: [log_to_pattern_metric.md](references/processors/log_to_pattern_metric.md)

```yaml
- type: log_to_pattern_metric
  num_of_clusters: 20
  samples_per_cluster: 1
  reporting_frequency: 1m
  similarity_threshold: 0.5
  final: true
```

**Common Use Cases**:
- Discover log patterns automatically
- Extract pattern frequency metrics
- Anomaly detection on pattern changes
- Cost optimization through pattern analysis

---

### Data Manipulation

#### 14. json_unroll
**Purpose**: Unroll JSON arrays into individual items (1 → N)
**Full Reference**: [json_unroll.md](references/processors/json_unroll.md)

```yaml
- type: json_unroll
  json_field_path: "$"
  new_field_name: "record"
```

**Common Use Cases**:
- Split API batch responses
- Process arrays of events
- Unroll multi-record payloads

---

#### 15. extract_json_field
**Purpose**: Extract specific JSON field and make it the body
**Full Reference**: [extract_json_field.md](references/processors/extract_json_field.md)

```yaml
- type: extract_json_field
  field_path: "data.user.id"
  keep_log_if_failed: false
```

**Common Use Cases**:
- Extract single field from nested JSON (CloudWatch logs)
- Unwrap API response payloads
- Array expansion with [*] wildcard

---

#### 16. split_with_delimiter
**Purpose**: Split single log into multiple by delimiter (1 → N)
**Full Reference**: [split_with_delimiter.md](references/processors/split_with_delimiter.md)

```yaml
- type: split_with_delimiter
  delimiter: "\\n"
```

**Common Use Cases**:
- Split multi-line logs (newline delimiter)
- Break batch messages into individual events
- Parse CSV, TSV, or custom delimited formats

---

### Deduplication

#### 17. dedup
**Purpose**: Deduplicate exact duplicate events using content hashing
**Full Reference**: [dedup.md](references/processors/dedup.md)

```yaml
- type: dedup
  interval: 30s
  count_field_path: "attributes.duplicate_count"
  excluded_field_paths:
    - "attributes.timestamp"
    - "attributes.request_id"
```

**Common Use Cases**:
- Remove exact duplicates and track counts
- Reduce log volume from multi-instance apps
- Aggregate repetitive error messages

---

### Enrichment

#### 18. lookup
**Purpose**: Enrich telemetry with lookup table data (CSV/DB)
**Full Reference**: [lookup.md](references/processors/lookup.md)

```yaml
- type: lookup
  location_path: "/path/to/lookup.csv"
  key_fields:
    - event_field: "attributes.user_id"
      lookup_field: "id"
  out_fields:
    - event_field: "attributes.user_name"
      lookup_field: "name"
      default_value: "unknown"
```

**Common Use Cases**:
- Add user metadata from CSV/DB
- Enrich with reference data
- Map IDs to names
- IP geolocation enrichment

---

### Special Processors

#### 19. deotel
**Purpose**: Convert to DeOTEL format (MUST be last processor, third-party only)
**Full Reference**: [deotel.md](references/processors/deotel.md)

```yaml
- type: deotel
  field_path: 'body["message"]'
  final: true
```

**Common Use Cases**:
- Third-party destination compatibility
- Legacy system integration
- Custom schema requirements

**Important**:
- MUST have `final: true`
- MUST be last processor in sequence
- Third-party destinations only (not EdgeDelta outputs)

---

### Nested Structures

#### 20. sequence
**Purpose**: Nested sequence within a sequence for modular organization
**Full Reference**: [sequence.md](references/processors/sequence.md)

```yaml
- type: sequence
  condition: 'attributes["environment"] == "production"'
  processors:
    - type: ottl_transform
      statements: |
        set(attributes["stage"], "prod-processing")
    - type: sample
      percentage: 10
      final: true
```

**Common Use Cases**:
- Modular processing blocks (stage-based organization)
- Conditional sub-pipelines (environment, severity)
- Error isolation within nested sequence

---

#### 21. compound
**Purpose**: Graph-based custom routing (advanced, reusable packs)
**Full Reference**: [compound.md](references/processors/compound.md)

```yaml
- name: my_pack
  type: compound
  nodes:
    - name: compound_input
      type: compound_input
    - name: parser
      type: ottl_transform
      statements: |
        set(attributes["parsed"], "true")
    - name: compound_output
      type: compound_output
  links:
    - from: compound_input
      to: parser
    - from: parser
      to: compound_output
```

**Common Use Cases**:
- Reusable processor packs
- Multi-path routing (success/failure branches)
- Team-specific processing modules
- Complex custom graph logic

**Note**: Advanced feature - use simple sequences for most cases

---

#### 22. sequence_input
**Purpose**: Sequence entry point (internal, auto-generated)
**Full Reference**: [sequence_input_output.md](references/processors/sequence_input_output.md)

**Note**: **Not user-configurable**. Automatically created by EdgeDelta at the start of every sequence. Provides named entry point in processing graph for metrics and debugging.

---

#### 23. sequence_output
**Purpose**: Sequence exit point (internal, auto-generated)
**Full Reference**: [sequence_input_output.md](references/processors/sequence_input_output.md)

**Note**: **Not user-configurable**. Automatically created by EdgeDelta at the end of every sequence. Provides named exit point in processing graph for metrics and debugging.

---

## Modern Routing (Not in Sequences)

### route_ottl
**Purpose**: OTTL-based routing processor (multi-path routing, NOT used inside sequences)

```yaml
- name: router
  type: route_ottl
  routes:
    - name: errors
      condition: 'attributes["level"] == "ERROR"'
    - name: metrics
      condition: 'attributes["type"] == "metric"'
```

**Important**: `route_ottl` is NOT sequence-compatible. Use for top-level routing between sequences/outputs.

---

## Processor Ordering Best Practices

Recommended order within sequences:

1. **Filtering** - Remove unwanted data early (`ottl_filter`)
2. **Parsing** - Extract fields (`json_unroll`, `extract_json_field`)
3. **Masking** - Remove sensitive data (`generic_mask`)
4. **Transformation** - Modify/enrich (`ottl_transform`, `lookup`)
5. **Sampling** - Reduce volume (`sample`)
6. **Metrics** - Extract metrics (`extract_metric`, `log_to_pattern_metric`)
7. **Special** - Format conversion (`deotel` - must be last if used)

## Common Combinations

### PII Masking + Metrics
```yaml
processors:
  - type: generic_mask
    capture_group_masks:
      - capture_group: "(?i)(password|passwd)[:=]\\S+"
        enabled: true
        mask: "***PASSWORD***"
        name: "password"
  - type: extract_metric
    extract_metric_rules:
      - name: "errors_total"
        conditions:
          - 'IsMatch(body, "ERROR")'
        sum:
          aggregation_temporality: delta
          is_monotonic: true
          value: 1
    interval: 1m
    final: true
```

### API Response Processing
```yaml
processors:
  - type: ottl_transform
    statements: |
      set(attributes["api_source"], "external_api")
      set(attributes["pulled_at"], Now())
  - type: json_unroll
    json_field_path: "$"
    new_field_name: "record"
  - type: ottl_transform
    statements: |
      set(cache["data"], ParseJSON(body)["record"])
      set(attributes["id"], cache["data"]["id"]) where cache["data"] != nil
    final: true
```

### Sampling with Error Preservation
```yaml
processors:
  - type: ottl_transform
    statements: |
      set(attributes["is_error"], "true") where IsMatch(body, "(?i)ERROR")
  - type: sample
    condition: 'attributes["is_error"] != "true"'  # Only sample non-errors
    percentage: 10
  - type: extract_metric
    interval: 1m
    final: true
```

### Additional Processors (15 — stub files with doc links)

| Processor | YAML Type | Docs | Sequence |
|---|---|---|---|
| Parse CSV | `parse_csv` | [docs](https://docs.edgedelta.com/parse-csv-processor/) | Yes |
| Parse JSON | `parse_json` | [docs](https://docs.edgedelta.com/parse-json-processor/) | Yes |
| Parse Key-Value | `parse_key_value` | [docs](https://docs.edgedelta.com/parse-key-value-processor/) | Yes |
| Parse Regex | `parse_regex` | [docs](https://docs.edgedelta.com/parse-regex-processor/) | Yes |
| Parse Severity | `parse_severity` | [docs](https://docs.edgedelta.com/parse-severity-processor/) | Yes |
| Parse Timestamp | `parse_timestamp` | [docs](https://docs.edgedelta.com/parse-timestamp-processor/) | Yes |
| Parse XML | `parse_xml` | [docs](https://docs.edgedelta.com/parse-xml-processor/) | Yes |
| Add Field | `add_field` | [docs](https://docs.edgedelta.com/add-field-processor/) | Yes |
| Copy Field | `copy_field` | [docs](https://docs.edgedelta.com/copy-field-processor/) | Yes |
| Delete Field | `delete_field` | [docs](https://docs.edgedelta.com/delete-field-processor/) | Yes |
| Pack | `pack` | [docs](https://docs.edgedelta.com/pack-processor/) | Yes |
| Redis Enrichment | `redis_enrichment` | [docs](https://docs.edgedelta.com/redis-enrichment-processor/) | Yes |
| Rollup Metric | `rollup_metric` | [docs](https://docs.edgedelta.com/rollup-metric-processor/) | Yes |
| Stateful Alert | `stateful_alert` | [docs](https://docs.edgedelta.com/stateful-alert-processor/) | Yes |
| Route | `route` | [docs](https://docs.edgedelta.com/route-node/) | No |
| Code | `code` | [docs](https://docs.edgedelta.com/code-processor/) | Yes |
| Comment | `comment` | [docs](https://docs.edgedelta.com/comment-processor/) | Yes |
| Conditional Group | `conditional_group` | [docs](https://docs.edgedelta.com/conditional-group-processor/) | Yes |

## Validation Rules

1. **final Flag**: Only ONE processor in a sequence can have `final: true` (must be last)
2. **deotel Position**: If using `deotel`, it MUST be last and have `final: true`
3. **Sequence Compatibility**: Only sequence-compatible processors can be used inside `sequence` nodes
4. **route**: Top-level routing only — cannot be placed inside a sequence

## Cross-References

- **edgedelta-pipelines skill**: Production-tested templates using these processors
- **Source Code**: `EdgeDelta source code repository/internalv3/discover/sequence_node.go` (lines 52-76)
- **Validation**: `configv3/config_validation.go`

---

**Coverage**: 30 sources | 38 processors | 54 destinations. All components have documentation links to docs.edgedelta.com.
