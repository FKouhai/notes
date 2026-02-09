---
id: 004-Deep_Dive_Traces
aliases:
  - otel-traces
  - distributed-tracing
tags:
  - opentelemetry
  - traces
  - observability
  - sre
---

# Deep Dive: Traces

## Trace Structure

A trace represents the complete path of a request through a distributed system. It's composed of a tree of spans representing individual operations.

### Trace Anatomy
```
Trace ID: 7b9a1f2e3d4c5b6a8f9e0d1c2b3a4f5e

┌─────────────────────────────────────────────────────────────┐
│                      Root Span                                │
│  Span ID: 1a2b3c4d5e6f7g8h                                   │
│  Operation: GET /api/users                                 │
│  Duration: 450ms                                            │
│  Status: OK                                                │
└─────┬──────────────────┬──────────────────┬─────────────────┘
      │                  │                  │
┌─────▼─────┐    ┌───────▼───────┐    ┌────▼─────┐
│  Service  │    │    Service    │    │ Service  │
│  Auth     │    │    User DB    │    │  Cache   │
│  Span ID: │    │  Span ID:     │    │ Span ID: │
│  2b3c4d5e │    │  3c4d5e6f     │    │ 4d5e6f7g │
│  150ms    │    │  300ms        │    │ 50ms     │
└───────────┘    └───────────────┘    └──────────┘
```

## Span Lifecycle

### Span States

**1. Not Started**
```go
// Span created but not started
tracer.Start(ctx, "operation-name") // Returns context and span
```

**2. In Progress**
```go
ctx, span := tracer.Start(ctx, "operation-name")
// Span is active, recording events
// Work happens here
```

**3. Ended**
```go
span.End()
// Span is finalized, no more modifications allowed
```

### Span Timing
```
Start Time             Event 1              Event 2              End Time
    │                    │                    │                   │
    ▼                    ▼                    ▼                   ▼
┌───┬────────────────────┬────────────────────┬───────────────────┐
│ S │                    │                    │                   │
│ t │   Work in progress │   More work        │   Final work      │
│ a │                    │                    │                   │
│ r │                    │                    │                   │
│ t │                    │                    │                   │
└───┴────────────────────┴────────────────────┴───────────────────┘
    │                    │                    │                   │
    └────────────────────┴────────────────────┴───────────────────┘
                        Duration
```

## Span Types

### 1. Root Span
- First span in a trace
- No parent span
- Represents the entire request
- Typically at the system boundary (API gateway, ingress)

```go
// Root span created at request entry point
ctx, span := tracer.Start(context.Background(), "HTTP GET /api/users")
defer span.End()
```

### 2. Child Span
- Has a parent span
- Represents sub-operations
- Forms the trace tree structure
- Can have multiple children

```go
// Child span from existing context
ctx, dbSpan := tracer.Start(ctx, "db.query")
defer dbSpan.End()
```

### 3. Internal Span
- Operations within a service
- Function calls, internal processing
- No external network calls

```go
// Internal processing
ctx, processSpan := tracer.Start(ctx, "process-data")
defer processSpan.End()
```

### 4. Client Span
- Outbound calls to other services
- Represents client-side operation
- Paired with server span on the receiving service

```go
// Outbound HTTP call
clientSpan := tracer.Start(ctx, "HTTP POST",
    trace.WithSpanKind(trace.SpanKindClient))
defer clientSpan.End()
```

### 5. Server Span
- Inbound requests
- Represents server-side operation
- Paired with client span on calling service

```go
// Inbound HTTP request
serverSpan := tracer.Start(ctx, "HTTP POST /api/data",
    trace.WithSpanKind(trace.SpanKindServer))
defer serverSpan.End()
```

### 6. Producer Span
- Message production to queue/topic
- Represents publishing operation

```go
producerSpan := tracer.Start(ctx, "kafka.produce",
    trace.WithSpanKind(trace.SpanKindProducer))
defer producerSpan.End()
```

### 7. Consumer Span
- Message consumption from queue/topic
- Represents processing of message

```go
consumerSpan := tracer.Start(ctx, "kafka.consume",
    trace.WithSpanKind(trace.SpanKindConsumer))
defer consumerSpan.End()
```

## Span Attributes

Attributes are key-value pairs that provide context about the span operation.

### Required Semantic Conventions

**Service Identification:**
```go
attribute.String("service.name", "user-api"),
attribute.String("service.version", "1.2.3"),
attribute.String("service.instance.id", "api-7f8b9a"),
```

**HTTP Attributes:**
```go
attribute.String("http.method", "GET"),
attribute.String("http.url", "https://api.example.com/users/123"),
attribute.String("http.target", "/users/123"),
attribute.String("http.scheme", "https"),
attribute.Int("http.status_code", 200),
attribute.String("http.route", "/users/:id"),
```

**Database Attributes:**
```go
attribute.String("db.system", "postgresql"),
attribute.String("db.name", "userdb"),
attribute.String("db.user", "appuser"),
attribute.String("db.statement", "SELECT * FROM users WHERE id = $1"),
attribute.String("db.operation", "SELECT"),
```

**RPC Attributes:**
```go
attribute.String("rpc.system", "grpc"),
attribute.String("rpc.service", "user.UserService"),
attribute.String("rpc.method", "GetUser"),
attribute.String("net.peer.name", "user-service"),
attribute.Int("net.peer.port", 9090),
```

**Messaging Attributes:**
```go
attribute.String("messaging.system", "kafka"),
attribute.String("messaging.destination", "user-events"),
attribute.String("messaging.destination_kind", "topic"),
attribute.String("messaging.operation", "publish"),
attribute.Int("messaging.message.id", 12345),
```

### Custom Attributes
```go
// Business logic attributes
attribute.String("user.id", "user-123"),
attribute.String("tenant.id", "tenant-abc"),
attribute.String("feature.flags", "new-ui,promotions"),
```

## Span Events

Events are time-stamped annotations that add detail to span execution.

```go
// Add events during span execution
span.AddEvent("cache.miss", trace.WithAttributes(
    attribute.String("cache.key", "user:123"),
))

span.AddEvent("db.query.start", trace.WithAttributes(
    attribute.String("db.statement", "SELECT * FROM users"),
))

// DB query happens

span.AddEvent("db.query.complete", trace.WithAttributes(
    attribute.Int("db.rows", 1),
    attribute.Int("db.duration_ms", 45),
))
```

**Event Types:**
- **Exception** - Errors that occurred
- **Log** - Log points during execution
- **State Change** - State transitions
- **Metrics** - Interim measurements

## Span Links

Links connect spans to other traces, enabling cross-trace correlation.

```go
// Link to another trace
link := trace.Link{
    SpanContext: trace.NewSpanContext(trace.SpanContextConfig{
        TraceID:    trace.TraceID{0x01, 0x02, ...},
        SpanID:     trace.SpanID{0x01, 0x02, ...},
        TraceFlags: trace.FlagsSampled,
    }),
}

ctx, span := tracer.Start(ctx, "async-operation",
    trace.WithLinks(link),
)
```

**Use Cases:**
- Asynchronous operations
- Batch processing
- Background jobs
- Cross-batch tracing

## Span Status

Status indicates the outcome of the span operation.

### Status Values

**Unset (default)**
```go
span.SetStatus(codes.Error, "")
```
- No error occurred
- Equivalent to "OK"
- Default when no error set

**OK**
```go
span.SetStatus(codes.Ok, "")
```
- Operation succeeded
- Explicit success indication

**Error**
```go
span.SetStatus(codes.Error, "database connection failed")
```
- Operation failed
- Includes description
- Sets error flag

### Best Practices
```go
// Don't set status on success (default is OK)
span.SetStatus(codes.Error, err.Error())

// Set status on unrecoverable errors only
if err != nil {
    span.RecordError(err)
    span.SetStatus(codes.Error, err.Error())
}

// Don't set status on business logic failures
// (e.g., user not found - return 404, don't set error)
```

## Span Kind

Span kind indicates the type of operation the span represents.

### Span Kinds

**Server**
```go
trace.WithSpanKind(trace.SpanKindServer)
```
- Handles incoming requests
- Entry point for external calls
- Example: HTTP server, gRPC server

**Client**
```go
trace.WithSpanKind(trace.SpanKindClient)
```
- Makes outgoing requests
- Leaves the service boundary
- Example: HTTP client, database client

**Producer**
```go
trace.WithSpanKind(trace.SpanKindProducer)
```
- Produces messages
- Sends to messaging system
- Example: Kafka producer

**Consumer**
```go
trace.WithSpanKind(trace.SpanKindConsumer)
```
- Consumes messages
- Receives from messaging system
- Example: Kafka consumer

**Internal**
```go
trace.WithSpanKind(trace.SpanKindInternal)
```
- Internal operations
- No external dependencies
- Example: Data processing, calculations

### Span Kind Importance
- Determines how spans are displayed
- Helps identify client-server pairs
- Enables automatic correlation
- Influences error handling

## Context Propagation

Context propagation carries trace information across service boundaries.

### Propagation Format (W3C Trace Context)
```
traceparent: 00-7b9a1f2e3d4c5b6a8f9e0d1c2b3a4f5e-1a2b3c4d5e6f7g8h-01
```

**Components:**
- Version: `00` (current)
- Trace ID: `7b9a1f2e3d4c5b6a8f9e0d1c2b3a4f5e`
- Span ID: `1a2b3c4d5e6f7g8h`
- Trace Flags: `01` (sampled)

### Propagation Flow
```
Service A                          Service B
    │                                  │
    │  HTTP Request                    │
    │  Headers:                        │
    │  traceparent: 00-trace-span-01   │
    │  baggage: user=123               │
    │──────────────────────────────────▶│
    │                                  │ Extract context
    │                                  │ Create child span
    │                                  │
    │◀─────────────────────────────────│
    │  HTTP Response                   │
    │                                  │
```

### Propagation Implementation

**Outgoing (Client):**
```go
// Inject context into HTTP headers
import (
    "go.opentelemetry.io/otel/trace"
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

client := &http.Client{
    Transport: otelhttp.NewTransport(http.DefaultTransport),
}
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, err := client.Do(req)
```

**Incoming (Server):**
```go
// Extract context from HTTP headers
handler := otelhttp.NewHandler(
    http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        // Span already extracted and created
    }),
    "http-server",
)
```

**Custom Propagation:**
```go
// Custom propagator for other protocols
propagator := propagation.NewCompositeTextMapPropagator(
    propagation.TraceContext{},
    propagation.Baggage{},
)

// Inject
propagator.Inject(ctx, propagation.MapCarrier{
    "traceparent": "00-...",
    "baggage": "user=123",
})

// Extract
ctx = propagator.Extract(context.Background(), propagation.MapCarrier{
    "traceparent": "00-...",
    "baggage": "user=123",
})
```

## Sampling

Sampling reduces trace volume by collecting only a subset.

### Sampling Strategies

**Always On**
```go
sampler := trace.AlwaysSample()
```
- Collect all traces
- Maximum observability
- High cost

**Always Off**
```go
sampler := trace.NeverSample()
```
- Collect no traces
- Minimum cost
- For debugging only

**Ratio-based**
```go
sampler := trace.TraceIDRatioBased(0.1) // 10%
```
- Collect percentage of traces
- Configurable ratio
- Balanced approach

**Parent-based**
```go
sampler := trace.ParentBased(
    trace.AlwaysSample(),
    trace.NeverSample(),
    trace.AlwaysSample(),
)
)
```
- Respects parent sampling decision
- Follows trace decision
- Ensures complete traces

### Advanced Sampling

**Custom Sampler:**
```go
type ConditionalSampler struct {
    ratio float64
}

func (s *ConditionalSampler) ShouldSample(...) trace.SamplingResult {
    // Sample based on attributes
    if attributes["http.route"] == "/health" {
        return trace.SamplingResult{
            Decision: trace.Drop,
        }
    }
    // Otherwise use ratio
    if rand.Float64() < s.ratio {
        return trace.SamplingResult{
            Decision: trace.RecordAndSample,
        }
    }
    return trace.SamplingResult{
        Decision: trace.Drop,
    }
}
```

**Attribute-based Sampling:**
```go
sampler := func(ctx context.Context, p trace.SamplingParameters) trace.SamplingResult {
    // Sample based on user ID
    userID := p.Attributes["user.id"].AsString()
    if isVIPUser(userID) {
        return trace.SamplingResult{Decision: trace.RecordAndSample}
    }
    return trace.SamplingResult{Decision: trace.Drop}
}
```

## Best Practices

1. **Span Naming**
   - Use operation names, not outcomes
   - ✅ `http.server`, `db.query`, `kafka.produce`
   - ❌ `success`, `error`, `fast`

2. **Granularity**
   - Span should represent logical operations
   - Not too fine-grained (function-level)
   - Not too coarse-grained (entire request)

3. **Attributes**
   - Follow semantic conventions
   - Add custom attributes for business context
   - Avoid sensitive data (PII, credentials)

4. **Events**
   - Mark important state changes
   - Record errors and exceptions
   - Keep events meaningful

5. **Links**
   - Link related async operations
   - Correlate batch processing
   - Connect independent traces

6. **Status**
   - Only set on errors
   - Use descriptive messages
   - Don't set on business logic failures

7. **Sampling**
   - Start with 1-10% sampling
   - Adjust based on volume
   - Use tail sampling for errors
