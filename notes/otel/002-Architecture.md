---
id: 002-Architecture
aliases:
  - otel-architecture
  - opentelemetry-architecture
tags:
  - opentelemetry
  - observability
  - sre
  - architecture
---

# OpenTelemetry Architecture

## Overview

OpenTelemetry is a collection of tools, APIs, and SDKs for generating, collecting, and exporting telemetry data (traces, metrics, logs) to analysis platforms. It's vendor-agnostic and provides a single standard for observability.

## Architecture Diagram
```
┌─────────────────────────────────────────────────────────────────┐
│                         Your Applications                        │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Service A  │  │   Service B  │  │   Service C  │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                 │                   │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐          │
│  │   OTel API   │  │   OTel API   │  │   OTel API   │          │
│  │   + SDK      │  │   + SDK      │  │   + SDK      │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                 │                   │
│         └─────────────────┼─────────────────┘                   │
│                           │                                      │
│                  OTLP (gRPC/HTTP)                              │
│                           │                                      │
├───────────────────────────▼──────────────────────────────────────┤
│                  OpenTelemetry Collector                         │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │   Receivers  │───▶│  Processors  │───▶│   Exporters  │       │
│  │  (otlp, ...) │    │ (batch, ...) │    │ (otlp, ...)  │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                   OTLP or native formats
                            │
├───────────────────────────▼──────────────────────────────────────┤
│                      Observability Backend                       │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Prometheus   │  │   Jaeger     │  │   Loki       │          │
│  │   (Metrics)  │  │   (Traces)   │  │   (Logs)     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Grafana     │  │   Tempo      │  │  Elastic     │          │
│  │  (Unified)   │  │   (Traces)   │  │  (Unified)   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. OpenTelemetry API

The API provides interfaces for generating telemetry data. It's a stable interface with no breaking changes.

**Key Components:**
- **Tracer API** - Interface for creating spans
- **Meter API** - Interface for creating metrics
- **Logger API** - Interface for creating logs
- **Context API** - Interface for context propagation
- **Baggage API** - Interface for passing metadata

**Design Principles:**
- Minimal dependencies
- No-op by default (no overhead if not used)
- Vendor-neutral

### 2. OpenTelemetry SDK

The SDK provides the actual implementation of the API. It's where configuration, processing, and export logic lives.

**Key Components:**

**TracerProvider:**
- Manages tracer instances
- Configures export pipelines
- Handles resource attributes
- Manages span processors

**Span Processors:**
- `BatchSpanProcessor` - Batches spans before export (default)
- `SimpleSpanProcessor` - Exports spans synchronously
- `SpanExporter` - Sends spans to backend/collector

**MeterProvider:**
- Manages meter instances
- Configures metric export pipelines
- Handles metric readers and views

**Metric Readers:**
- `PeriodicExportingMetricReader` - Exports metrics at intervals
- `PrometheusHttpServerReader` - Serves metrics for scraping

**LoggerProvider:**
- Manages logger instances
- Configures log export pipelines
- Handles log processors

### 3. Instrumentation

Instrumentation is the process of adding telemetry generation to code.

**Types of Instrumentation:**

**Manual Instrumentation:**
- Directly calling OTel SDK APIs
- Maximum control and flexibility
- More code to write and maintain

**Example (traces):**
```go
import "go.opentelemetry.io/otel"

tracer := otel.Tracer("my-service")

ctx, span := tracer.Start(ctx, "operation-name")
defer span.End()

// Do work
span.SetAttributes(attribute.String("key", "value"))
```

**Automatic Instrumentation:**
- Uses libraries/agents that automatically instrument
- Zero code changes required
- Less control but easier adoption
- Language support varies

**Example:**
```bash
# Java auto-instrumentation agent
java -javaagent:opentelemetry-javaagent.jar -jar myapp.jar
```

**Library Instrumentation:**
- Pre-instrumented versions of common libraries
- HTTP clients, database drivers, messaging systems
- Provided by OTel contrib repositories

## Data Flow

### 1. Telemetry Generation

Application code generates telemetry using OTel SDK:

```
Service Application
    │
    ├─▶ Create Span (Tracer API)
    ├─▶ Record Metric (Meter API)
    └─▶ Emit Log (Logger API)
```

### 2. Processing

SDK processes telemetry before export:

```
SDK Processing
    │
    ├─▶ Add resource attributes (service name, version)
    ├─▶ Apply span processors (batching)
    ├─▶ Apply metric views (aggregation)
    └─▶ Enrich with context (trace ID, baggage)
```

### 3. Export

Telemetry is sent via OTLP protocol:

```
Export
    │
    ├─▶ OTLP over gRPC (binary, efficient)
    └─▶ OTLP over HTTP (JSON, compatible)
```

### 4. Collection (Optional)

Collector receives and processes telemetry:

```
Collector Pipeline
    │
    ├─▶ Receivers (OTLP, Kafka, Zipkin, etc.)
    ├─▶ Processors (batch, filter, transform)
    └─▶ Exporters (Prometheus, Jaeger, Loki, etc.)
```

## Deployment Models

### Agent Model

Collector runs as a sidecar or daemonset on each host:

```
┌─────────────────────────────────────────────┐
│  Application Pod                             │
│  ┌──────────┐  ┌──────────┐                 │
│  │  App     │──▶│ Collector│                 │
│  └──────────┘  └──────────┘                 │
└─────────────────────────────────────────────┘
```

**Pros:**
- Isolation of concerns
- Per-host configuration
- Reduced network traffic

**Cons:**
- More resources per host
- Management overhead

### Central Model

Single collector instance for entire cluster:

```
┌─────────────────────────────────────────────┐
│  Cluster                                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │ Service │  │ Service │  │ Service │    │
│  └────┬────┘  └────┬────┘  └────┬────┘    │
└───────┼────────────┼────────────┼─────────┘
        │            │            │
        └────────────┼────────────┘
                     │
        ┌────────────▼────────────┐
        │    Central Collector    │
        └─────────────────────────┘
```

**Pros:**
- Single point of configuration
- Easier management
- Less resource overhead

**Cons:**
- Single point of failure
- Network bottleneck potential
- Scalability concerns

### Gateway Model

Collector as gateway behind load balancer:

```
┌──────────────────────────────────────────────────┐
│  Services ──▶ Load Balancer ──▶ Collector Pool │
└──────────────────────────────────────────────────┘
```

**Pros:**
- Scalable
- High availability
- Load distribution

**Cons:**
- More complex setup
- Requires load balancer

### Hybrid Model

Combination of agent and central collectors:

```
┌──────────────────────────────────────────────────┐
│  Services ──▶ Agent Collectors ──▶ Central Collectors │
└──────────────────────────────────────────────────┘
```

**Pros:**
- Best of both worlds
- Flexible routing
- Local buffering

**Cons:**
- Most complex setup
- More components to manage

## Resource Detection

Resources identify the entity producing telemetry.

**Automatic Detection:**
- Hostname, IP addresses
- Process information (PID, executable)
- Container information (pod name, namespace)
- Cloud provider metadata

**Manual Configuration:**
```go
resource, _ := resource.Merge(
    resource.Default(),
    resource.NewWithAttributes(
        semconv.SchemaURL,
        attribute.String("service.name", "my-service"),
        attribute.String("service.version", "1.0.0"),
    ),
)
```

**Resource Hierarchy:**
- Service-level attributes (name, version, environment)
- Host-level attributes (hostname, OS, arch)
- Container-level attributes (pod, namespace, node)
- Cloud-level attributes (region, availability zone, account)
