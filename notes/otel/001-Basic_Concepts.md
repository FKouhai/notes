---
id: 001-Basic_Concepts
aliases:
  - otel-basic-concepts
  - observability-basics
tags:
  - opentelemetry
  - observability
  - sre
  - monitoring
---

# OpenTelemetry Basic Concepts

## Observability

Observability is the ability to understand the internal state of a system by examining its external outputs. It allows you to answer questions about system behavior without needing to instrument new code to get answers.

### Three Pillars of Observability

1. **Metrics** - Numerical measurements over time
2. **Logs** - Discrete events with timestamps
3. **Traces** - Request journey through distributed systems

## Signals

OpenTelemetry handles three types of telemetry data called signals:

### 1. Traces

Traces track the progression of a single request as it travels through distributed systems. A trace consists of one or more spans that represent work done by individual services or components.

**Key Concepts:**
- **Trace ID** - Unique identifier for the entire request chain
- **Span** - Represents a unit of work or operation
- **Span ID** - Unique identifier for a specific span
- **Parent Span ID** - Links spans to their parent (creates the tree structure)
- **Context Propagation** - Passing trace context between services

**Span Attributes:**
- Timestamps (start/end)
- Duration
- Status codes
- Events (annotations within a span)
- Links (relationships to other traces)
- Attributes (key-value metadata)

### 2. Metrics

Metrics are numerical measurements collected over time. They provide insight into the behavior and health of a system.

**Metric Types:**

**Counter** - Monotonic values that only increase
```
http_requests_total{method="GET", status="200"} 1523
```
- Use for counting occurrences (requests, errors, bytes sent)
- Example: Number of requests processed

**Gauge** - Values that can go up or down
```
memory_usage_bytes{service="api"} 52428800
```
- Use for current state measurements
- Example: Current memory usage, active connections

**Histogram** - Counts observations in configurable buckets
```
http_request_duration_seconds_bucket{le="0.1"} 1000
http_request_duration_seconds_bucket{le="0.5"} 1500
http_request_duration_seconds_bucket{le="+Inf"} 2000
```
- Use for measuring distributions (latency, request sizes)
- Provides count, sum, and bucket counts
- Example: Request latency distribution

**Summary** - Similar to histogram but pre-calculated percentiles
```
http_request_duration_seconds{quantile="0.5"} 0.05
http_request_duration_seconds{quantile="0.9"} 0.12
http_request_duration_seconds{quantile="0.99"} 0.25
```
- Use when you need exact percentiles on the emitting service
- Less flexible for aggregation across services

**Metric Instruments (OpenTelemetry SDK):**
- **Counter** - Add a non-negative value
- **Asynchronous Counter** - Observe monotonic values
- **UpDownCounter** - Add any value (positive or negative)
- **Asynchronous Gauge** - Observe non-monotonic values
- **Histogram** - Record measurements with buckets

### 3. Logs

Logs are discrete events with a timestamp. OpenTelemetry provides tools to enrich logs with trace and metric context.

**Log Concepts:**
- **Timestamp** - When the event occurred
- **Severity** - Log level (DEBUG, INFO, WARN, ERROR)
- **Body** - The log message or data
- **Attributes** - Key-value pairs for context
- **Trace ID & Span ID** - Links logs to traces
- **Resource** - Information about the entity producing the log

**Log Correlation:**
Logs can be correlated with traces by including the trace context (trace ID and span ID) in log attributes. This allows filtering logs by trace in observability backends.

## Context Propagation

Context propagation allows trace information to flow between services. When Service A calls Service B, it passes trace context so the spans are linked.

**W3C Trace Context Standard:**
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

Format:
- `traceparent: {version}-{trace-id}-{span-id}-{trace-flags}`
- Version: `00` for current version
- Trace ID: 32-character hex string
- Span ID: 16-character hex string
- Trace Flags: 2-character hex string for sampling decisions

**Baggage:**
Additional key-value data that propagates with the trace, independent of span timing. Useful for passing business context (user ID, tenant ID) through the call chain.

## Sampling

Sampling reduces the volume of telemetry data by collecting only a subset of data.

**Sampling Strategies:**

**Head-based Sampling** - Decision made at trace start
```
- Probability-based (e.g., collect 10% of traces)
- Always sample certain traces (based on attributes)
```

**Tail-based Sampling** - Decision made after trace completes
```
- Collect all traces, decide later what to keep
- Can sample based on error presence, latency thresholds
- Requires buffering complete traces
```

**Dynamic Sampling** - Adjusts sampling rate based on conditions
```
- Increase sampling during high error rates
- Decrease sampling during high traffic
```

**OTel Collector Sampling:**
The OpenTelemetry Collector can apply sampling policies:
- `probabilistic` - Random sampling
- `traceidratio` - Sample based on trace ID hash
- `filter` - Drop or keep based on attributes
- `attributes` - Sample based on span attributes

## Semantic Conventions

Semantic conventions provide standardized naming for common telemetry data. This ensures consistency across services and tools.

**Examples:**
- `service.name` - Name of the service
- `service.version` - Version of the service
- `http.method` - HTTP method (GET, POST, etc.)
- `http.status_code` - HTTP response status code
- `db.system` - Database type (postgresql, mysql, etc.)
- `db.statement` - Database query
- `net.peer.name` - Remote host name
- `net.peer.port` - Remote port

Following semantic conventions enables:
- Consistent querying across services
- Automatic instrumentation compatibility
- Better visualization in observability tools
- Easier integration with third-party tools
