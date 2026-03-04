# EdgeDelta v3 Destinations Reference

> **Complete reference for all 54 EdgeDelta v3 destination nodes.**
> For detailed documentation, follow the **Docs** link for each destination.

## Categories

- [Observability Platforms (17)](#observability-platforms)
- [Storage & Data Lakes (12)](#storage--data-lakes)
- [Security & SIEM (5)](#security--siem)
- [Custom & Streaming (20)](#custom--streaming)

---

## Observability Platforms

### Edge Delta (`type: edgedelta_output`)
**Docs**: https://docs.edgedelta.com/edge-delta-destination/
**Category**: Observability Platforms
**Description**: Sends data to the EdgeDelta cloud platform for storage, analysis, and visualization.
**Key Parameters**: `api_key`, `endpoint`, `batch_size`

### Amazon CloudWatch (`type: cloudwatch_output`)
**Docs**: https://docs.edgedelta.com/amazon-cloudwatch-destination/
**Category**: Observability Platforms
**Description**: Sends log events and metrics to Amazon CloudWatch Logs and Metrics.
**Key Parameters**: `log_group`, `log_stream`, `aws_region`, `aws_access_key_id`

### Azure Log Analytics (`type: azure_log_analytics_output`)
**Docs**: https://docs.edgedelta.com/azure-log-analytics-destination/
**Category**: Observability Platforms
**Description**: Sends log data to Azure Log Analytics workspace via Data Collection API.
**Key Parameters**: `workspace_id`, `workspace_key`, `log_type`

### CrowdStrike Falcon LogScale (`type: crowdstrike_logscale_output`)
**Docs**: https://docs.edgedelta.com/crowdstrike-falcon-logscale-destination/
**Category**: Observability Platforms
**Description**: Streams log events to CrowdStrike Falcon LogScale (formerly Humio) for security log management.
**Key Parameters**: `ingest_token`, `endpoint`, `repository`

### Datadog (`type: datadog_output`)
**Docs**: https://docs.edgedelta.com/datadog-destination/
**Category**: Observability Platforms
**Description**: Sends logs and metrics to Datadog for monitoring and alerting.
**Key Parameters**: `api_key`, `site`, `service`, `source`

### Debug (`type: debug_output`)
**Docs**: https://docs.edgedelta.com/debug-destination/
**Category**: Observability Platforms
**Description**: Prints pipeline data to console/stdout for testing and debugging purposes.
**Key Parameters**: `format`, `max_messages`

### Dynatrace (`type: dynatrace_output`)
**Docs**: https://docs.edgedelta.com/dynatrace-destination/
**Category**: Observability Platforms
**Description**: Sends logs and metrics to Dynatrace for full-stack observability.
**Key Parameters**: `endpoint`, `api_token`, `log_ingest_url`

### Elastic (`type: elastic_output`)
**Docs**: https://docs.edgedelta.com/elastic-destination/
**Category**: Observability Platforms
**Description**: Indexes log data into Elasticsearch for search and analysis.
**Key Parameters**: `addresses`, `index`, `username`, `password`, `tls`

### Fluentd (`type: fluentd_output`)
**Docs**: https://docs.edgedelta.com/fluentd-destination/
**Category**: Observability Platforms
**Description**: Forwards log data to Fluentd aggregators via forward protocol.
**Key Parameters**: `endpoint`, `tag`, `tls`

### Google Cloud Logging (`type: gcp_cloud_logging_output`)
**Docs**: https://docs.edgedelta.com/google-cloud-logging-destination/
**Category**: Observability Platforms
**Description**: Sends log entries to Google Cloud Logging (formerly Stackdriver).
**Key Parameters**: `project_id`, `log_name`, `credentials_file`

### Grokstream AIOps (`type: grokstream_output`)
**Docs**: https://docs.edgedelta.com/grokstream-destination/
**Category**: Observability Platforms
**Description**: Sends observability data to Grokstream AIOps platform for anomaly detection.
**Key Parameters**: `endpoint`, `api_key`

### Loki (`type: loki_output`)
**Docs**: https://docs.edgedelta.com/loki-destination/
**Category**: Observability Platforms
**Description**: Sends log streams to Grafana Loki for log aggregation and querying.
**Key Parameters**: `endpoint`, `labels`, `tenant_id`, `tls`

### New Relic (`type: new_relic_output`)
**Docs**: https://docs.edgedelta.com/new-relic-destination/
**Category**: Observability Platforms
**Description**: Sends logs and metrics to New Relic for observability and alerting.
**Key Parameters**: `api_key`, `endpoint`, `account_id`

### Splunk HEC (`type: splunk_hec_output`)
**Docs**: https://docs.edgedelta.com/splunk-hec-destination/
**Category**: Observability Platforms
**Description**: Sends log events to Splunk via HTTP Event Collector (HEC) protocol.
**Key Parameters**: `endpoint`, `token`, `index`, `source`, `tls`

### Splunk TCP (`type: splunk_tcp_output`)
**Docs**: https://docs.edgedelta.com/splunk-tcp-destination/
**Category**: Observability Platforms
**Description**: Forwards log data to Splunk indexers via S2S TCP protocol.
**Key Parameters**: `endpoint`, `port`, `tls`

### Sumo Logic (`type: sumo_logic_output`)
**Docs**: https://docs.edgedelta.com/sumo-logic-destination/
**Category**: Observability Platforms
**Description**: Sends log and metric data to Sumo Logic for analysis.
**Key Parameters**: `endpoint`, `source_category`, `source_host`

### VictoriaMetrics (`type: victoria_metrics_output`)
**Docs**: https://docs.edgedelta.com/victoriametrics-destination/
**Category**: Observability Platforms
**Description**: Sends metric data to VictoriaMetrics time-series database via remote write.
**Key Parameters**: `endpoint`, `tls`, `labels`

---

## Storage & Data Lakes

### Apache Kudu (`type: kudu_output`)
**Docs**: https://docs.edgedelta.com/apache-kudu-destination/
**Category**: Storage & Data Lakes
**Description**: Inserts log data into Apache Kudu columnar storage for analytics workloads.
**Key Parameters**: `masters`, `table`, `columns`

### Azure Blob (`type: azure_blob_output`)
**Docs**: https://docs.edgedelta.com/azure-blob-destination/
**Category**: Storage & Data Lakes
**Description**: Writes log data as files to Azure Blob Storage containers.
**Key Parameters**: `connection_string`, `container`, `prefix`, `format`

### BigQuery (`type: bigquery_output`)
**Docs**: https://docs.edgedelta.com/bigquery-destination/
**Category**: Storage & Data Lakes
**Description**: Streams log data into Google BigQuery tables for large-scale analytics.
**Key Parameters**: `project_id`, `dataset`, `table`, `credentials_file`

### ClickHouse (`type: clickhouse_output`)
**Docs**: https://docs.edgedelta.com/clickhouse-destination/
**Category**: Storage & Data Lakes
**Description**: Inserts log and metric data into ClickHouse for high-performance analytics.
**Key Parameters**: `dsn`, `table`, `database`, `batch_size`

### DigitalOcean Spaces (`type: digitalocean_spaces_output`)
**Docs**: https://docs.edgedelta.com/digitalocean-spaces-destination/
**Category**: Storage & Data Lakes
**Description**: Writes log data to DigitalOcean Spaces object storage.
**Key Parameters**: `endpoint`, `bucket`, `access_key`, `secret_key`, `prefix`

### Google Cloud Storage (`type: gcs_output`)
**Docs**: https://docs.edgedelta.com/google-cloud-storage-destination/
**Category**: Storage & Data Lakes
**Description**: Writes log data as objects to Google Cloud Storage buckets.
**Key Parameters**: `bucket`, `prefix`, `credentials_file`, `format`

### IBM Object Storage (`type: ibm_object_storage_output`)
**Docs**: https://docs.edgedelta.com/ibm-object-storage-destination/
**Category**: Storage & Data Lakes
**Description**: Writes log data to IBM Cloud Object Storage.
**Key Parameters**: `endpoint`, `bucket`, `api_key`, `prefix`

### Local Storage (`type: local_storage_output`)
**Docs**: https://docs.edgedelta.com/local-storage-destination/
**Category**: Storage & Data Lakes
**Description**: Writes log data to the local filesystem for archival or buffering.
**Key Parameters**: `path`, `format`, `max_size`, `rotation`

### MinIO (`type: minio_output`)
**Docs**: https://docs.edgedelta.com/minio-destination/
**Category**: Storage & Data Lakes
**Description**: Writes log data to MinIO S3-compatible object storage.
**Key Parameters**: `endpoint`, `bucket`, `access_key`, `secret_key`, `prefix`

### S3 (`type: s3_output`)
**Docs**: https://docs.edgedelta.com/s3-destination/
**Category**: Storage & Data Lakes
**Description**: Writes log data as objects to Amazon S3 buckets with configurable compression and format.
**Key Parameters**: `bucket`, `aws_region`, `prefix`, `format`, `compression`

### Snowflake (`type: snowflake_output`)
**Docs**: https://docs.edgedelta.com/snowflake-destination/
**Category**: Storage & Data Lakes
**Description**: Loads log and event data into Snowflake data warehouse.
**Key Parameters**: `account`, `database`, `schema`, `table`, `user`, `password`

### Zenko CloudServer (`type: zenko_output`)
**Docs**: https://docs.edgedelta.com/zenko-cloudserver-destination/
**Category**: Storage & Data Lakes
**Description**: Writes log data to Zenko CloudServer S3-compatible storage.
**Key Parameters**: `endpoint`, `bucket`, `access_key`, `secret_key`

---

## Security & SIEM

### Exabeam (`type: exabeam_output`)
**Docs**: https://docs.edgedelta.com/exabeam-destination/
**Category**: Security & SIEM
**Description**: Forwards security log data to Exabeam SIEM for behavioral analytics.
**Key Parameters**: `endpoint`, `api_key`, `tls`

### IBM QRadar (`type: qradar_output`)
**Docs**: https://docs.edgedelta.com/ibm-qradar-destination/
**Category**: Security & SIEM
**Description**: Sends log events to IBM QRadar SIEM via syslog or LEEF protocol.
**Key Parameters**: `endpoint`, `port`, `protocol`, `log_source_identifier`

### Microsoft Sentinel (`type: microsoft_sentinel_output`)
**Docs**: https://docs.edgedelta.com/microsoft-sentinel-destination/
**Category**: Security & SIEM
**Description**: Sends log data to Microsoft Sentinel (Azure SIEM) via Data Collection API.
**Key Parameters**: `workspace_id`, `workspace_key`, `log_type`

### Securonix (`type: securonix_output`)
**Docs**: https://docs.edgedelta.com/securonix-destination/
**Category**: Security & SIEM
**Description**: Forwards security events to Securonix SIEM/UEBA platform.
**Key Parameters**: `endpoint`, `token`, `account`

### Google SecOps (`type: google_secops_output`)
**Docs**: https://docs.edgedelta.com/google-secops-destination/
**Category**: Security & SIEM
**Description**: Sends security telemetry to Google Security Operations (Chronicle) platform.
**Key Parameters**: `project_id`, `customer_id`, `credentials_file`, `log_type`

---

## Custom & Streaming

### HTTP (`type: http_output`)
**Docs**: https://docs.edgedelta.com/http-destination/
**Category**: Custom & Streaming
**Description**: Sends log data via HTTP/HTTPS POST requests to any endpoint.
**Key Parameters**: `endpoint`, `headers`, `auth`, `tls`, `batch_size`

### Kafka (`type: kafka_output`)
**Docs**: https://docs.edgedelta.com/kafka-destination/
**Category**: Custom & Streaming
**Description**: Publishes log data to Apache Kafka topics.
**Key Parameters**: `brokers`, `topic`, `tls`, `compression`

### Microsoft Teams (`type: microsoft_teams_output`)
**Docs**: https://docs.edgedelta.com/microsoft-teams-destination/
**Category**: Custom & Streaming
**Description**: Sends alert and notification messages to Microsoft Teams channels via webhooks.
**Key Parameters**: `webhook_url`, `template`

### OpenMetrics (`type: open_metrics_output`)
**Docs**: https://docs.edgedelta.com/openmetrics-destination/
**Category**: Custom & Streaming
**Description**: Exposes metrics in OpenMetrics/Prometheus exposition format via HTTP endpoint.
**Key Parameters**: `port`, `path`, `labels`

### OTLP (`type: otlp_output`)
**Docs**: https://docs.edgedelta.com/otlp-destination/
**Category**: Custom & Streaming
**Description**: Exports telemetry data via OpenTelemetry Protocol (OTLP) to any OTLP-compatible backend.
**Key Parameters**: `endpoint`, `tls`, `headers`, `compression`

### Prometheus Exporter (`type: prometheus_exporter_output`)
**Docs**: https://docs.edgedelta.com/prometheus-exporter-destination/
**Category**: Custom & Streaming
**Description**: Exposes collected metrics as a Prometheus scrape endpoint.
**Key Parameters**: `port`, `path`, `namespace`

### Prometheus Remote Write (`type: prometheus_remote_write_output`)
**Docs**: https://docs.edgedelta.com/prometheus-remote-write-destination/
**Category**: Custom & Streaming
**Description**: Sends metric data to Prometheus-compatible remote write endpoints (Thanos, Cortex, etc.).
**Key Parameters**: `endpoint`, `tls`, `headers`, `labels`

### Slack (`type: slack_output`)
**Docs**: https://docs.edgedelta.com/slack-destination/
**Category**: Custom & Streaming
**Description**: Sends alert and notification messages to Slack channels via webhooks.
**Key Parameters**: `webhook_url`, `channel`, `template`

### TCP (`type: tcp_output`)
**Docs**: https://docs.edgedelta.com/tcp-destination/
**Category**: Custom & Streaming
**Description**: Forwards log data over TCP connections to any TCP receiver.
**Key Parameters**: `endpoint`, `port`, `tls`, `delimiter`

### Webhook (`type: webhook_output`)
**Docs**: https://docs.edgedelta.com/webhook-destination/
**Category**: Custom & Streaming
**Description**: Sends log data or alerts to arbitrary webhook endpoints with customizable payloads.
**Key Parameters**: `url`, `method`, `headers`, `template`, `auth`

### PagerDuty (`type: pagerduty_output`)
**Docs**: https://docs.edgedelta.com/pagerduty-destination/
**Category**: Custom & Streaming
**Description**: Sends alerts and incidents to PagerDuty for on-call notification management.
**Key Parameters**: `routing_key`, `severity`, `template`

### OpsGenie (`type: opsgenie_output`)
**Docs**: https://docs.edgedelta.com/opsgenie-destination/
**Category**: Custom & Streaming
**Description**: Creates and manages alerts in Atlassian OpsGenie for incident management.
**Key Parameters**: `api_key`, `priority`, `template`

### Humio (`type: humio_output`)
**Docs**: https://docs.edgedelta.com/humio-destination/
**Category**: Custom & Streaming
**Description**: Sends log data to Humio (now CrowdStrike Falcon LogScale) log management platform.
**Key Parameters**: `ingest_token`, `endpoint`

### SignalFx (`type: signalfx_output`)
**Docs**: https://docs.edgedelta.com/signalfx-destination/
**Category**: Custom & Streaming
**Description**: Sends metrics to Splunk Observability (formerly SignalFx) for infrastructure monitoring.
**Key Parameters**: `access_token`, `realm`, `dimensions`

### AppDynamics (`type: appdynamics_output`)
**Docs**: https://docs.edgedelta.com/appdynamics-destination/
**Category**: Custom & Streaming
**Description**: Sends application performance data to Cisco AppDynamics.
**Key Parameters**: `account_name`, `api_key`, `controller_url`

### Syslog (`type: syslog_output`)
**Docs**: https://docs.edgedelta.com/syslog-destination/
**Category**: Custom & Streaming
**Description**: Forwards log messages in syslog format over UDP or TCP.
**Key Parameters**: `endpoint`, `port`, `protocol`, `format`, `facility`

### Cribl (`type: cribl_output`)
**Docs**: https://docs.edgedelta.com/cribl-destination/
**Category**: Custom & Streaming
**Description**: Forwards log and metric data to Cribl Stream for routing and transformation.
**Key Parameters**: `endpoint`, `auth_token`, `tls`

### LogDNA (`type: logdna_output`)
**Docs**: https://docs.edgedelta.com/logdna-destination/
**Category**: Custom & Streaming
**Description**: Sends log data to LogDNA (now Mezmo) log management platform.
**Key Parameters**: `api_key`, `hostname`, `tags`

### Mezmo (`type: mezmo_output`)
**Docs**: https://docs.edgedelta.com/mezmo-destination/
**Category**: Custom & Streaming
**Description**: Sends log data to Mezmo (formerly LogDNA) log management and pipeline platform.
**Key Parameters**: `api_key`, `hostname`, `tags`

### OpenSearch (`type: opensearch_output`)
**Docs**: https://docs.edgedelta.com/opensearch-destination/
**Category**: Custom & Streaming
**Description**: Indexes log data into OpenSearch (AWS OpenSearch Service or self-hosted).
**Key Parameters**: `addresses`, `index`, `username`, `password`, `tls`

---

*Total: 54 destinations. For complete parameter references, visit [docs.edgedelta.com](https://docs.edgedelta.com).*
