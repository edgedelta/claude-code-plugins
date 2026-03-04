# EdgeDelta v3 Sources Reference

> **Complete reference for all 30 EdgeDelta v3 source nodes.**
> For detailed documentation, follow the **Docs** link for each source.

## Categories

- [Log Sources (14)](#log-sources)
- [Metric Sources (3)](#metric-sources)
- [Trace Sources (1)](#trace-sources)
- [Cloud Native Sources (2)](#cloud-native-sources)
- [Hybrid Sources (10)](#hybrid-sources)

---

## Log Sources

### CrowdStrike FDR (`type: crowdstrike_fdr_input`)
**Docs**: https://docs.edgedelta.com/crowdstrike-fdr-source/
**Category**: Log Sources
**Description**: Ingests CrowdStrike Falcon Data Replicator (FDR) log data from S3-based streaming.
**Key Parameters**: `sqs_url`, `aws_region`, `aws_access_key_id`, `aws_secret_access_key`

---

### Docker (`type: docker_input`)
**Docs**: https://docs.edgedelta.com/docker-source/
**Category**: Log Sources
**Description**: Collects logs from Docker container stdout/stderr streams.
**Key Parameters**: `labels`, `containers`, `exclude_containers`

---

### Event Hub (`type: azure_event_hub_input`)
**Docs**: https://docs.edgedelta.com/event-hub-source/
**Category**: Log Sources
**Description**: Ingests log events from Azure Event Hub.
**Key Parameters**: `connection_string`, `consumer_group`, `event_hub_name`

---

### Exec (`type: exec_input`)
**Docs**: https://docs.edgedelta.com/exec-source/
**Category**: Log Sources
**Description**: Runs shell commands or scripts periodically and collects their output as log data.
**Key Parameters**: `command`, `interval`, `timeout`

---

### File (`type: file_input`)
**Docs**: https://docs.edgedelta.com/file-source/
**Category**: Log Sources
**Description**: Tails log files from the filesystem, supporting wildcards and rotation detection.
**Key Parameters**: `path`, `labels`, `start_from`

---

### Filebeat (`type: filebeat_input`)
**Docs**: https://docs.edgedelta.com/filebeat-source/
**Category**: Log Sources
**Description**: Receives logs forwarded from Filebeat agents.
**Key Parameters**: `port`, `tls`

---

### Fluentd (`type: fluentd_input`)
**Docs**: https://docs.edgedelta.com/fluentd-source/
**Category**: Log Sources
**Description**: Receives logs forwarded from Fluentd agents via forward protocol.
**Key Parameters**: `port`, `tls`

---

### Journald (`type: journald_input`)
**Docs**: https://docs.edgedelta.com/journald-source/
**Category**: Log Sources
**Description**: Collects logs from systemd journald on Linux systems.
**Key Parameters**: `units`, `matches`, `cursor_path`

---

### Kafka (`type: kafka_input`)
**Docs**: https://docs.edgedelta.com/kafka-source/
**Category**: Log Sources
**Description**: Consumes log messages from Apache Kafka topics.
**Key Parameters**: `brokers`, `topics`, `group_id`, `tls`

---

### Kubernetes (`type: kubernetes_input` or `k8s_input`)
**Docs**: https://docs.edgedelta.com/kubernetes-source/
**Category**: Log Sources
**Description**: Collects logs from Kubernetes pod containers via node-local log files.
**Key Parameters**: `labels`, `namespaces`, `exclude_namespaces`

---

### Kubernetes Event (`type: kubernetes_event_input`)
**Docs**: https://docs.edgedelta.com/kubernetes-event-source/
**Category**: Log Sources
**Description**: Ingests Kubernetes cluster events (pod restarts, warnings, etc.) from the API server.
**Key Parameters**: `namespaces`, `event_types`

---

### Splunk HEC (`type: splunk_hec_input`)
**Docs**: https://docs.edgedelta.com/splunk-hec-source/
**Category**: Log Sources
**Description**: Receives log data sent via Splunk HTTP Event Collector (HEC) protocol.
**Key Parameters**: `port`, `token`, `tls`

---

### Splunk TCP (`type: splunk_tcp_input`)
**Docs**: https://docs.edgedelta.com/splunk-tcp-source/
**Category**: Log Sources
**Description**: Receives log data from Splunk Universal or Heavy Forwarders via S2S TCP protocol.
**Key Parameters**: `port`, `tls`

---

### Windows Event (`type: windows_event_input`)
**Docs**: https://docs.edgedelta.com/windows-event-source/
**Category**: Log Sources
**Description**: Collects Windows Event Log entries from Windows systems.
**Key Parameters**: `channels`, `event_levels`, `query`

---

## Metric Sources

### Kubernetes Metrics (`type: kubernetes_metrics_input`)
**Docs**: https://docs.edgedelta.com/kubernetes-metrics-source/
**Category**: Metric Sources
**Description**: Collects resource metrics (CPU, memory, network) from Kubernetes nodes and pods.
**Key Parameters**: `labels`, `namespaces`, `collection_interval`

---

### Kubernetes Service Map (`type: kubernetes_service_map_input`)
**Docs**: https://docs.edgedelta.com/kubernetes-service-map-source/
**Category**: Metric Sources
**Description**: Collects network connection data between Kubernetes services for service topology mapping.
**Key Parameters**: `namespaces`, `collection_interval`

---

### Prometheus (`type: prometheus_input`)
**Docs**: https://docs.edgedelta.com/prometheus-source/
**Category**: Metric Sources
**Description**: Scrapes Prometheus metrics endpoints and ingests metric data.
**Key Parameters**: `scrape_configs`, `global`

---

## Trace Sources

### Kubernetes Trace (`type: kubernetes_trace_input`)
**Docs**: https://docs.edgedelta.com/kubernetes-trace-source/
**Category**: Trace Sources
**Description**: Collects distributed tracing data from Kubernetes workloads.
**Key Parameters**: `namespaces`, `sampling_rate`

---

## Cloud Native Sources

### Google Cloud Pub/Sub (`type: gcp_pubsub_input`)
**Docs**: https://docs.edgedelta.com/google-cloud-pub-sub-source/
**Category**: Cloud Native Sources
**Description**: Subscribes to Google Cloud Pub/Sub topics to ingest log and event data.
**Key Parameters**: `project_id`, `subscription`, `credentials_file`

---

### S3 (`type: s3_input`)
**Docs**: https://docs.edgedelta.com/s3-source/
**Category**: Cloud Native Sources
**Description**: Reads log files from Amazon S3 buckets, with optional SQS notification support.
**Key Parameters**: `bucket`, `aws_region`, `sqs_url`, `prefix`

---

## Hybrid Sources

### Datadog (`type: datadog_input`)
**Docs**: https://docs.edgedelta.com/datadog-source/
**Category**: Hybrid Sources
**Description**: Receives log and metric data from Datadog agents via the Datadog API protocol.
**Key Parameters**: `port`, `api_key`, `tls`

---

### HTTP (`type: http_input`)
**Docs**: https://docs.edgedelta.com/http-source/
**Category**: Hybrid Sources
**Description**: Receives log data via HTTP/HTTPS POST requests with configurable authentication.
**Key Parameters**: `port`, `path`, `tls`, `auth`

---

### OTLP (`type: otlp_input`)
**Docs**: https://docs.edgedelta.com/otlp-source/
**Category**: Hybrid Sources
**Description**: Receives OpenTelemetry Protocol (OTLP) data including logs, metrics, and traces.
**Key Parameters**: `grpc_port`, `http_port`, `tls`

---

### Pipeline (`type: pipeline_input`)
**Docs**: https://docs.edgedelta.com/pipeline-source/
**Category**: Hybrid Sources
**Description**: Connects to another EdgeDelta pipeline as a data source for pipeline chaining.
**Key Parameters**: `pipeline_id`, `endpoint`

---

### TCP (`type: tcp_input`)
**Docs**: https://docs.edgedelta.com/tcp-source/
**Category**: Hybrid Sources
**Description**: Receives raw log data over TCP connections with configurable framing.
**Key Parameters**: `port`, `tls`, `delimiter`

---

### Telemetry Generator (`type: telemetry_generator_input`)
**Docs**: https://docs.edgedelta.com/telemetry-generator-source/
**Category**: Hybrid Sources
**Description**: Generates synthetic telemetry data for testing and development purposes.
**Key Parameters**: `rate`, `template`, `attributes`

---

### UDP (`type: udp_input`)
**Docs**: https://docs.edgedelta.com/udp-source/
**Category**: Hybrid Sources
**Description**: Receives log data over UDP protocol, commonly used for syslog and SNMP.
**Key Parameters**: `port`, `max_message_size`

---

### HTTP Pull (`type: http_pull_input`)
**Docs**: https://docs.edgedelta.com/http-pull-source/
**Category**: Hybrid Sources
**Description**: Periodically pulls data from HTTP/HTTPS endpoints (polling mode).
**Key Parameters**: `url`, `interval`, `headers`, `auth`

---

### Syslog (`type: syslog_input`)
**Docs**: https://docs.edgedelta.com/syslog-source/
**Category**: Hybrid Sources
**Description**: Receives syslog messages over UDP or TCP with RFC 3164/5424 format support.
**Key Parameters**: `port`, `protocol`, `tls`

---

### HTTP Workflow (`type: http_workflow_input`)
**Docs**: https://docs.edgedelta.com/http-workflow-source/
**Category**: Hybrid Sources
**Description**: Triggers pipeline workflows via HTTP webhook calls, enabling event-driven processing.
**Key Parameters**: `port`, `path`, `method`, `auth`

---

*Total: 30 sources. For complete parameter references, visit [docs.edgedelta.com](https://docs.edgedelta.com).*
