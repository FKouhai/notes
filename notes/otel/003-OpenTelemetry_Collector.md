---
id: 003-OpenTelemetry_Collector
aliases:
  - otel-collector
  - collector
tags:
  - opentelemetry
  - collector
  - observability
  - sre
---

# OpenTelemetry Collector

## Overview

The OpenTelemetry Collector is a vendor-agnostic, highly performant way to receive, process, and export telemetry data. It supports multiple protocols and formats, making it the central hub for observability data.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Collector Components                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐          │
│  │ Receivers │───▶│ Processors│───▶│ Exporters │          │
│  └───────────┘    └───────────┘    └───────────┘          │
│       │                │                │                  │
│       │                │                │                  │
│  ┌────▼────────────────▼────────────────▼────┐             │
│  │           Extensions (optional)            │             │
│  └────────────────────────────────────────────┘             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Receivers

Receivers accept telemetry data from various sources and push it to the pipeline.

### Built-in Receivers

**OTLP Receiver**
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
```
- Default receiver for OTLP protocol
- Supports both gRPC and HTTP
- Handles traces, metrics, and logs
- Most common receiver for OTel SDKs

**Kafka Receiver**
```yaml
receivers:
  kafka:
    brokers:
      - broker1:9092
      - broker2:9092
    topic: otel-spans
    encoding: otlp_json
```
- Receives telemetry from Kafka topics
- Useful for high-throughput scenarios
- Supports OTLP, Jaeger, and Zipkin formats

**Prometheus Receiver**
```yaml
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: 'otel-collector'
          static_configs:
            - targets: ['0.0.0.0:8888']
```
- Scrape Prometheus metrics
- Converts metrics to OTLP format
- Enables unified metrics collection

**Zipkin Receiver**
```yaml
receivers:
  zipkin:
    endpoint: 0.0.0.0:9411
```
- Legacy Zipkin protocol support
- Migration path from Zipkin
- Converts to OTLP format

**Jaeger Receiver**
```yaml
receivers:
  jaeger:
    protocols:
      thrift_http:
        endpoint: 0.0.0.0:14268
      thrift_compact:
        endpoint: 0.0.0.0:6831
      grpc:
        endpoint: 0.0.0.0:14250
```
- Multiple Jaeger protocol support
- Supports thrift and gRPC
- Converts to OTLP format

**File Receiver**
```yaml
receivers:
  file:
    include:
      - /var/log/*.json
    start_at: beginning
```
- Reads logs from files
- Supports multiple formats (JSON, text)
- Useful for log aggregation

**StatsD Receiver**
```yaml
receivers:
  statsd:
    endpoint: 0.0.0.0:8125
    aggregation_interval: 60s
```
- StatsD protocol support
- Legacy metrics migration
- Aggregates metrics before export

**Syslog Receiver**
```yaml
receivers:
  syslog:
    protocol: tcp
    listen_address: 0.0.0.0:54526
    operators:
      - type: regex_parser
        regex: '^<(?P<priority>\d+)>(?P<version>\d+) (?P<timestamp>\S+) (?P<hostname>\S+) (?P<appname>\S+) (?P<procid>\S+) (?P<msgid>\S+) (?P<structured_data>\S+) (?P<message>.*)$'
```
- Syslog protocol support
- RFC3164 and RFC5424 formats
- System log aggregation

### Ways to Push Signals to Collector

**1. Direct from SDK (OTLP)**
```go
// Go SDK example
exporter, _ := otlptracegrpc.New(context.Background(),
    otlptracegrpc.WithEndpoint("collector:4317"),
    otlptracegrpc.WithInsecure(),
)
```
- Most common approach
- OTLP protocol (gRPC or HTTP)
- Efficient binary format
- Native support in all SDKs

**2. Sidecar Pattern**
```yaml
# Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: app
          # Application container
        - name: collector
          image: otel/opentelemetry-collector-contrib:latest
```
- Collector runs alongside application
- Local buffering
- Network isolation
- Per-service configuration

**3. DaemonSet**
```yaml
apiVersion: apps/v1
kind: DaemonSet
spec:
  template:
    spec:
      containers:
        - name: collector
          image: otel/opentelemetry-collector-contrib:latest
          ports:
            - containerPort: 4317
```
- One collector per node
- Host-based telemetry collection
- Resource sharing
- Less network traffic

**4. Gateway Pattern**
```
Services ──▶ Load Balancer ──▶ Collector Gateway ──▶ Backends
```
- Central collector cluster
- Load-balanced
- High availability
- Easier management

**5. Edge Collector**
```
Edge Services ──▶ Edge Collector ──▶ Central Collector ──▶ Backends
```
- Collect at edge (IoT, remote sites)
- Local buffering
- Batched uploads
- Bandwidth optimization

**6. Agent to Gateway**
```
Services ──▶ Agent Collectors ──▶ Gateway Collectors ──▶ Backends
```
- Two-tier architecture
- Local buffering at agents
- Centralized export at gateway
- Best of both worlds

## Processors

Processors modify telemetry data before export.

### Common Processors

**Batch Processor**
```yaml
processors:
  batch:
    timeout: 5s
    send_batch_size: 10000
    send_batch_max_size: 11000
```
- Groups telemetry into batches
- Reduces network calls
- Default processor
- Configurable timeout and size

**Memory Limiter Processor**
```yaml
processors:
  memory_limiter:
    limit_mib: 512
    spike_limit_mib: 128
    check_interval: 5s
```
- Prevents OOM kills
- Memory-based load shedding
- Drops data when memory high
- Essential for production

**Attributes Processor**
```yaml
processors:
  attributes:
    actions:
      - key: environment
        value: production
        action: insert
      - key: password
        action: delete
      - key: http.path
        action: extract_regex
        regex: '^/api/(v[0-9]+)/.*$'
      - key: http.route
        action: upsert
        value: ${http.path}
```
- Manipulate attributes
- Insert, update, delete, extract
- Regex-based transformations
- Data enrichment

**Filter Processor**
```yaml
processors:
  filter:
    spans:
      - match_type: regexp
        expressions:
          - attributes["http.target"] == "^/health"
          - attributes["http.status_code"] == 200
```
- Drop specific telemetry
- Filter by attributes
- Useful for reducing noise
- Regex-based matching

**Tail Sampling Processor**
```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 100
    expected_new_traces_per_sec: 10
    policies:
      - name: error-only
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: slow-traces
        type: latency
        latency:
          threshold_ms: 1000
```
- Sample complete traces
- Multiple sampling policies
- Error-based, latency-based
- Configurable policies

**Resource Processor**
```yaml
processors:
  resource:
    attributes:
      - key: cluster.name
        value: production
        action: insert
      - key: host.name
        from_attribute: host.name
        action: insert
```
- Modify resource attributes
- Rename or copy attributes
- Add cluster-level metadata
- Enrich with environment info

**Transform Processor**
```yaml
processors:
  transform:
    trace_statements:
      - context: span
        statements:
          - set(name, attributes["http.method"] + " " + attributes["http.route"])
```
- Powerful transformation language
- Rename spans, attributes
- Conditional transformations
- Complex data manipulation

## Exporters

Exporters send processed telemetry to various backends.

### Popular Exporters

**OTLP Exporter**
```yaml
exporters:
  otlp:
    endpoint: backend-collector:4317
    tls:
      insecure: false
      cert_file: /certs/client.pem
      key_file: /certs/client-key.pem
```
- Default exporter
- Supports any OTLP-compatible backend
- TLS/SSL support
- Most flexible option

**Prometheus Exporter**
```yaml
exporters:
  prometheus:
    endpoint: 0.0.0.0:8889
    namespace: otel
    const_labels:
      environment: production
```
- Exposes metrics for scraping
- OpenMetrics format
- Integration with Prometheus
- Metrics-only

**Jaeger Exporter**
```yaml
exporters:
  jaeger:
    endpoint: jaeger-collector:14250
    tls:
      insecure: true
```
- Direct export to Jaeger
- Thrift or gRPC protocol
- Traces-only
- Legacy support

**Zipkin Exporter**
```yaml
exporters:
  zipkin:
    endpoint: http://zipkin:9411/api/v2/spans
    format: json
```
- Direct export to Zipkin
- JSON format
- Traces-only
- Migration path

**Logging Exporter**
```yaml
exporters:
  logging:
    loglevel: debug
```
- Print to stdout
- Debugging and development
- All signal types
- No persistence

**Kafka Exporter**
```yaml
exporters:
  kafka:
    brokers:
      - broker1:9092
      - broker2:9092
    topic: otel-spans
    encoding: otlp_json
    producer:
      max_message_bytes: 1000000
```
- High-throughput scenarios
- Distributed systems
- Decouples processing
- Supports reprocessing

**Elastic Exporter**
```yaml
exporters:
  elasticsearch:
    endpoints:
      - http://elasticsearch:9200
    indices:
      spans:
        index: otel-spans
      metrics:
        index: otel-metrics
```
- Direct to Elasticsearch
- OpenSearch compatible
- Unified observability
- All signal types

**OTLP HTTP Exporter**
```yaml
exporters:
  otlphttp:
    endpoint: https://collector.example.com:4318/v1/traces
    tls:
      insecure: false
```
- HTTP instead of gRPC
- Firewall-friendly
- JSON or binary
- Easier debugging

## Extensions

Extensions provide additional functionality outside the main pipeline.

### Common Extensions

**Health Check Extension**
```yaml
extensions:
  health_check:
    endpoint: 0.0.0.0:13133
```
- Health check endpoint
- HTTP status endpoint
- Kubernetes liveness/readiness probes

**PProf Extension**
```yaml
extensions:
  pprof:
    endpoint: localhost:1777
```
- Profiling support
- Performance analysis
- Debugging

**zPages Extension**
```yaml
extensions:
  zpages:
    endpoint: localhost:55679
```
- Runtime diagnostics
- Pipeline visualization
- Component status

**SigV4 Auth Extension**
```yaml
extensions:
  sigv4auth:
    region: us-east-1
    assume_role:
      arn: arn:aws:iam::123456789012:role/otel-role
```
- AWS authentication
- X-Ray integration
- Managed services

## Configuration Example

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
  memory_limiter:
    limit_mib: 512
    spike_limit_mib: 128

exporters:
  otlp:
    endpoint: backend:4317
  logging:
    loglevel: info

extensions:
  health_check:
    endpoint: 0.0.0.0:13133

service:
  extensions: [health_check]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp, logging]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp]
```

## Deployment Strategies

### Single Binary
```bash
./otelcol --config config.yaml
```
- Simple deployment
- Single instance
- Development/testing
- Low volume

### Kubernetes Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
spec:
  replicas: 3
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector-contrib:latest
          args:
            - --config=/etc/otel/config.yaml
          volumeMounts:
            - name: config
              mountPath: /etc/otel
      volumes:
        - name: config
          configMap:
            name: otel-config
```
- Scalable
- Managed lifecycle
- Production-ready
- High availability
