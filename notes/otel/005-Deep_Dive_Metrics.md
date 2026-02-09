---
id: 005-Deep_Dive_Metrics
aliases:
  - otel-metrics
  - metrics
tags:
  - opentelemetry
  - metrics
  - observability
  - sre
---

# Deep Dive: Metrics

## Metric Overview

Metrics provide numerical measurements over time, enabling monitoring of system health, performance, and business KPIs. OpenTelemetry provides a flexible metric API that supports various metric types and export formats.

## Metric Instruments

### Synchronous Instruments

Synchronous instruments record measurements as they occur during code execution.

**Counter**
```go
counter, _ := meter.Int64Counter(
    "http.requests.total",
    instrument.WithDescription("Total number of HTTP requests"),
)

// Increment by 1
counter.Add(ctx, 1)

// Increment by more
counter.Add(ctx, 5)
```
- Monotonic: only increases
- Use for counting occurrences
- Cumulative over time
- Example: requests processed, errors occurred

**UpDownCounter**
```go
counter, _ := meter.Int64UpDownCounter(
    "http.active.connections",
    instrument.WithDescription("Current number of active connections"),
)

// Increase
counter.Add(ctx, 1)

// Decrease
counter.Add(ctx, -1)
```
- Non-monotonic: increases or decreases
- Use for current state measurements
- Gauge-like behavior
- Example: active connections, queue length, memory in use

**Histogram**
```go
histogram, _ := meter.Float64Histogram(
    "http.request.duration",
    instrument.WithDescription("HTTP request duration"),
)

// Record measurement
histogram.Record(ctx, 0.045) // 45ms
histogram.Record(ctx, 0.123) // 123ms
histogram.Record(ctx, 0.089) // 89ms
```
- Records measurements in buckets
- Calculates count, sum, and bucket counts
- Use for distributions
- Example: request latency, response size, processing time

### Asynchronous Instruments

Asynchronous instruments observe values at collection time, typically via callbacks.

**Gauge**
```go
_, _ = meter.Float64ObservableGauge(
    "system.memory.usage",
    instrument.WithDescription("Current memory usage"),
    instrument.WithFloat64Callback(func(ctx context.Context, o metric.Float64Observer) metric.Error {
        usage := getCurrentMemoryUsage()
        o.Observe(usage)
        return nil
    }),
)
```
- Observed via callback
- Non-monotonic
- Use for measurements that can't be recorded inline
- Example: CPU usage, memory usage, temperature

**Counter (Asynchronous)**
```go
_, _ = meter.Int64ObservableCounter(
    "system.processes.total",
    instrument.WithDescription("Total number of processes"),
    instrument.WithInt64Callback(func(ctx context.Context, o metric.Int64Observer) metric.Error {
        total := getProcessCount()
        o.Observe(total)
        return nil
    }),
)
```
- Observed via callback
- Monotonic
- Use for counts that can't be recorded inline
- Example: process count, thread count

**UpDownCounter (Asynchronous)**
```go
_, _ = meter.Int64ObservableUpDownCounter(
    "system.files.open",
    instrument.WithDescription("Number of open file descriptors"),
    instrument.WithInt64Callback(func(ctx context.Context, o metric.Int64Observer) metric.Error {
        open := getOpenFileCount()
        o.Observe(open)
        return nil
    }),
)
```
- Observed via callback
- Non-monotonic
- Use for state that can't be recorded inline
- Example: open files, open sockets

## Metric Attributes

Attributes add dimensions to metrics, enabling filtering and aggregation.

### Adding Attributes
```go
// Add attributes to synchronous instrument
counter.Add(ctx, 1,
    attribute.String("http.method", "GET"),
    attribute.String("http.route", "/api/users"),
    attribute.Int("http.status_code", 200),
)

// Add attributes to histogram
histogram.Record(ctx, 0.123,
    attribute.String("http.method", "POST"),
    attribute.String("http.route", "/api/orders"),
)
```

### Common Attribute Patterns

**HTTP Metrics:**
```go
attribute.String("http.method", "GET"),
attribute.String("http.scheme", "https"),
attribute.String("http.host", "api.example.com"),
attribute.String("http.target", "/users/123"),
attribute.String("http.route", "/users/:id"),
attribute.Int("http.status_code", 200),
attribute.String("http.flavor", "1.1"),
```

**Database Metrics:**
```go
attribute.String("db.system", "postgresql"),
attribute.String("db.name", "userdb"),
attribute.String("db.user", "appuser"),
attribute.String("db.statement", "SELECT * FROM users"),
attribute.String("db.operation", "SELECT"),
attribute.String("net.peer.name", "postgres.example.com"),
```

**RPC Metrics:**
```go
attribute.String("rpc.system", "grpc"),
attribute.String("rpc.service", "user.UserService"),
attribute.String("rpc.method", "GetUser"),
attribute.String("rpc.grpc.status_code", "OK"),
```

**Runtime Metrics:**
```go
attribute.String("process.runtime.name", "go"),
attribute.String("process.runtime.version", "1.21.0"),
attribute.String("process.executable.name", "myapp"),
attribute.String("process.executable.path", "/usr/bin/myapp"),
```

## Aggregation

Aggregation determines how metric data is summarized over time.

### Aggregation Types

**Default Aggregation:**
```go
// Counter: Default aggregation
// Cumulative sum over time
http_requests_total{method="GET", route="/users"} 1523

// Histogram: Default aggregation
// Buckets + count + sum
http_request_duration_seconds_bucket{le="0.1"} 1000
http_request_duration_seconds_bucket{le="0.5"} 1500
http_request_duration_seconds_bucket{le="+Inf"} 2000
http_request_duration_seconds_sum 250.5
http_request_duration_seconds_count 2000
```

**Explicit Aggregation (Prometheus):**
```go
reader := prometheus.New(
    prometheus.WithAggregationSelector(func(ik metric.InstrumentKind) aggregation.Aggregation {
        switch ik {
        case metric.InstrumentKindCounter, metric.InstrumentKindAsynchronousCounter:
            return aggregation.Sum{}
        case metric.InstrumentKindHistogram:
            return aggregation.ExplicitBucketHistogram{
                Boundaries: []float64{0.01, 0.05, 0.1, 0.5, 1, 5},
            }
        case metric.InstrumentKindGauge, metric.InstrumentKindAsynchronousGauge:
            return aggregation.LastValue{}
        default:
            return aggregation.Drop{}
        }
    }),
)
```

**Custom Histogram Buckets:**
```go
histogram, _ := meter.Float64Histogram(
    "http.request.duration",
    instrument.WithExplicitBucketBoundaries(
        0.001,  // 1ms
        0.005,  // 5ms
        0.01,   // 10ms
        0.025,  // 25ms
        0.05,   // 50ms
        0.1,    // 100ms
        0.25,   // 250ms
        0.5,    // 500ms
        1.0,    // 1s
        2.5,    // 2.5s
        5.0,    // 5s
        10.0,   // 10s
    ),
)
```

**Exponential Histogram:**
```go
histogram, _ := meter.Float64Histogram(
    "http.request.duration",
    metric.WithExplicitBucketBoundaries(),
    metric.WithExponentialHistogram(),
)
// Uses exponential buckets for better range coverage
```

## Views

Views transform instruments before export, enabling customization of metric data.

### View Operations

**Rename Instrument:**
```go
view, _ := metric.NewView(
    metric.Instrument{Name: "http.requests.total"},
    metric.Stream{Name: "http.requests"},
)
```

**Filter Attributes:**
```go
view, _ := metric.NewView(
    metric.Instrument{Name: "http.request.duration"},
    metric.Stream{AttributeFilter: attribute.NewAllowKeysFilter(
        "http.method",
        "http.route",
        "http.status_code",
    )},
)
```

**Add Prefix:**
```go
view, _ := metric.NewView(
    metric.Instrument{Name: "http.requests.total"},
    metric.Stream{Name: "myapp_http_requests_total"},
)
```

**Change Aggregation:**
```go
view, _ := metric.NewView(
    metric.Instrument{Name: "process.memory.usage"},
    metric.Stream{Aggregation: aggregation.LastValue{}},
)
```

**Drop Attributes:**
```go
view, _ := metric.NewView(
    metric.Instrument{Name: "http.request.duration"},
    metric.Stream{AttributeFilter: attribute.NewDenyKeysFilter(
        "http.host",
        "http.user_agent",
    )},
)
```

**Complete Example:**
```go
view, _ := metric.NewView(
    metric.Instrument{
        Name: "http.request.duration",
    },
    metric.Stream{
        Name: "http.latency",
        AttributeFilter: attribute.NewAllowKeysFilter(
            "http.method",
            "http.route",
            "http.status_code",
        ),
        Aggregation: aggregation.ExplicitBucketHistogram{
            Boundaries: []float64{0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1},
        },
    },
)
```

## Temporality

Temporality defines how metric values are accumulated over time.

### Temporality Types

**Cumulative**
```go
// Cumulative: Values accumulate from process start
// Counter: Total since start
// Histogram: All observations since start

time=0:  count=0
time=10: count=100
time=20: count=250
time=30: count=400
```
- Default for counters
- Values accumulate over lifetime
- Reset on process restart
- Prometheus-compatible

**Delta**
```go
// Delta: Values since last export
time=0:  count=0
time=10: count=100  (first export)
time=20: count=150  (delta of 50)
time=30: count=150  (delta of 0)
```
- Values since last collection
- Better for long-running processes
- More complex for rate calculations

**Gauge (for UpDownCounter)**
```go
// Gauge: Current value at collection time
time=0:  value=100
time=10: value=150
time=20: value=125
time=30: value=180
```
- Current state
- No accumulation
- Standard for gauges

### Configuring Temporality
```go
reader := prometheus.New(
    prometheus.WithTemporalitySelector(func(ik metric.InstrumentKind) metricdata.Temporality {
        switch ik {
        case metric.InstrumentKindCounter, metric.InstrumentKindAsynchronousCounter:
            return metricdata.CumulativeTemporality
        case metric.InstrumentKindGauge, metric.InstrumentKindAsynchronousGauge:
            return metricdata.GaugeTemporality
        default:
            return metricdata.CumulativeTemporality
        }
    }),
)
```

## Export Formats

### Prometheus Format

**Exposition Format:**
```promql
# TYPE http_requests_total counter
http_requests_total{method="GET",route="/users",status="200"} 1523 1640000000

# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET",route="/users",le="0.1"} 1000 1640000000
http_request_duration_seconds_bucket{method="GET",route="/users",le="0.5"} 1500 1640000000
http_request_duration_seconds_bucket{method="GET",route="/users",le="+Inf"} 2000 1640000000
http_request_duration_seconds_sum{method="GET",route="/users"} 250.5 1640000000
http_request_duration_seconds_count{method="GET",route="/users"} 2000 1640000000

# TYPE system_memory_usage_bytes gauge
system_memory_usage_bytes 52428800 1640000000
```

**Scraping:**
```go
exporter, _ := prometheus.New(
    prometheus.WithAddress(":9090"),
)
```

### OTLP Format

**Protocol Buffers:**
```protobuf
message ResourceMetrics {
  Resource resource = 1;
  repeated ScopeMetrics scope_metrics = 2;
  SchemaUrl schema_url = 3;
}

message ScopeMetrics {
  InstrumentationScope scope = 1;
  repeated Metric metrics = 2;
  SchemaUrl schema_url = 3;
}

message Metric {
  string name = 1;
  string description = 2;
  string unit = 3;
  MetricData data = 7;
}
```

**Exporting:**
```go
exporter, _ := otlpmetricgrpc.New(
    context.Background(),
    otlpmetricgrpc.WithEndpoint("collector:4317"),
    otlpmetricgrpc.WithInsecure(),
)
```

## Best Practices

### 1. Choose the Right Instrument

**Use Counter when:**
```go
// ✅ Counting occurrences
counter.Add(ctx, 1)  // Requests processed

// ❌ Don't use for current state
counter.Add(ctx, currentCount)  // Wrong!
```

**Use UpDownCounter when:**
```go
// ✅ Tracking current state
counter.Add(ctx, 1)   // Connection opened
counter.Add(ctx, -1)  // Connection closed

// ❌ Don't use for monotonic counts
counter.Add(ctx, 1)  // Wrong! Use Counter instead
```

**Use Histogram when:**
```go
// ✅ Measuring distributions
histogram.Record(ctx, latency)  // Request latency

// ❌ Don't use for simple counters
histogram.Record(ctx, 1)  // Wrong! Use Counter instead
```

**Use Gauge (async) when:**
```go
// ✅ Observing external measurements
gauge.Observe(getMemoryUsage())  // System memory

// ❌ Don't use for inline measurements
gauge.Observe(getValue())  // Wrong! Use synchronous instrument
```

### 2. Semantic Conventions

Follow semantic conventions for consistency:
```go
// ✅ Good: Follow conventions
attribute.String("http.method", "GET")
attribute.String("http.status_code", "200")

// ❌ Bad: Custom names
attribute.String("method", "GET")
attribute.String("status", "200")
```

### 3. Cardinality Management

Limit attribute combinations to prevent high cardinality:
```go
// ✅ Good: Low cardinality
attribute.String("http.method", "GET")
attribute.String("http.route", "/users/:id")
attribute.Int("http.status_code", 200)

// ❌ Bad: High cardinality
attribute.String("user.id", "user-123")  // Too many users
attribute.String("request.id", "uuid-123")  // Unique per request
```

### 4. Meaningful Units

Always specify units:
```go
// ✅ Good: With units
meter.Int64Counter("http.requests.total",
    instrument.WithUnit("1"),  // dimensionless
)
meter.Float64Histogram("http.request.duration",
    instrument.WithUnit("s"),  // seconds
)

// ❌ Bad: No units
meter.Int64Counter("http.requests.total")
meter.Float64Histogram("http.request.duration")
```

### 5. Descriptive Names

Use clear, descriptive names:
```go
// ✅ Good: Descriptive
"http.requests.total"
"http.request.duration"
"system.memory.usage"

// ❌ Bad: Ambiguous
"requests"
"duration"
"memory"
```

### 6. Bucket Configuration

Choose appropriate histogram buckets:
```go
// ✅ Good: Tailored to your data
instrument.WithExplicitBucketBoundaries(
    0.001, 0.005, 0.01, 0.025, 0.05,
    0.1, 0.25, 0.5, 1, 2.5, 5, 10,
)

// ❌ Bad: Default buckets not aligned with data
instrument.WithExplicitBucketBoundaries(
    0.005, 0.01, 0.025, 0.05, 0.075, 0.1,
    0.25, 0.5, 0.75, 1, 2.5, 5, 7.5, 10,
)
```

### 7. Exemplars

Link metrics to traces:
```go
histogram.Record(ctx, 0.123,
    metric.WithAttributes(
        attribute.String("trace.id", "123"),
        attribute.String("span.id", "456"),
    ),
)
```
