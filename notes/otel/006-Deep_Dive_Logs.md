---
id: 006-Deep_Dive_Logs
aliases:
  - otel-logs
  - logs
tags:
  - opentelemetry
  - logs
  - observability
  - sre
---

# Deep Dive: Logs

## Log Overview

Logs are discrete events with timestamps that capture runtime information. OpenTelemetry provides a unified logging API that integrates with traces and metrics, enabling comprehensive observability.

## Log Structure

### Log Record
```go
type LogRecord struct {
    Time            time.Time       // When the log was created
    SeverityNumber  SeverityNumber  // Log severity level
    SeverityText    string          // Human-readable severity
    Body            string          // Log message
    Attributes      []KeyValue      // Key-value pairs
    TraceID         TraceID         // Associated trace ID
    SpanID          SpanID          // Associated span ID
    TraceFlags      byte            // Trace sampling flags
    Resource        Resource        // Source of the log
    InstrumentationScope InstrumentationScope  // Logger scope
    ObservedTimestamp time.Time      // When the event occurred
}
```

### Log Anatomy
```
┌─────────────────────────────────────────────────────────────────┐
│                        Log Record                               │
├─────────────────────────────────────────────────────────────────┤
│ Timestamp:       2024-01-15T10:30:45.123Z                       │
│ Observed:        2024-01-15T10:30:45.120Z                       │
│ Severity:        INFO                                           │
│ Body:            User login successful                          │
│                                                                 │
│ Trace ID:        7b9a1f2e3d4c5b6a8f9e0d1c2b3a4f5e               │
│ Span ID:         1a2b3c4d5e6f7g8h                              │
│                                                                 │
│ Attributes:                                                      │
│   user.id:        "user-123"                                    │
│   user.name:      "John Doe"                                    │
│   login.method:   "password"                                    │
│   source.ip:      "192.168.1.100"                               │
│                                                                 │
│ Resource:                                                        │
│   service.name:    "auth-service"                               │
│   service.version: "1.2.3"                                      │
│   host.name:       "auth-1"                                     │
└─────────────────────────────────────────────────────────────────┘
```

## Severity Levels

### Severity Numbers

OpenTelemetry uses numeric severity levels for programmatic comparison.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Severity Scale                               │
├─────────────────────────────────────────────────────────────────┤
│ TRACE  (1):    Most detailed, lowest severity                  │
│ DEBUG  (5):    Debugging information                            │
│ INFO   (9):    General informational messages                   │
│ WARN   (13):   Warning messages                                 │
│ ERROR  (17):   Error conditions                                 │
│ FATAL  (21):   Critical errors, service termination            │
└─────────────────────────────────────────────────────────────────┘
```

### Severity Mapping

**Common Framework Mappings:**

**Go standard log:**
```go
log.Println("info")     → INFO (9)
log.Fatal("fatal")      → FATAL (21)
log.Panic("panic")      → ERROR (17)
```

**Zap:**
```go
logger.Debug("msg")     → DEBUG (5)
logger.Info("msg")      → INFO (9)
logger.Warn("msg")      → WARN (13)
logger.Error("msg")     → ERROR (17)
logger.Fatal("msg")     → FATAL (21)
```

**Logrus:**
```go
logger.Debug("msg")     → DEBUG (5)
logger.Info("msg")      → INFO (9)
logger.Warn("msg")      → WARN (13)
logger.Error("msg")     → ERROR (17)
logger.Fatal("msg")     → FATAL (21)
logger.Panic("msg")     → ERROR (17)
```

### Using Severity
```go
logger.Error("Failed to connect to database",
    WithAttributes(
        attribute.String("db.host", "postgres.example.com"),
        attribute.Int("db.port", 5432),
        attribute.String("error", "connection refused"),
    ),
)

logger.Warn("High memory usage detected",
    WithAttributes(
        attribute.Float64("memory.usage.percent", 85.5),
        attribute.Int64("memory.usage.bytes", 4294967296),
    ),
)
```

## Logger Configuration

### Creating a Logger

```go
import "go.opentelemetry.io/otel/sdk/log"

// Create logger provider
loggerProvider := log.NewLoggerProvider(
    log.WithResource(resource),
)

// Create logger
logger := loggerProvider.Logger("my-logger",
    log.WithInstrumentationVersion("1.0.0"),
)
```

### Logger Options

```go
logger := loggerProvider.Logger(
    "my-logger",
    log.WithInstrumentationVersion("1.0.0"),
    log.WithSchemaURL("https://opentelemetry.io/schemas/1.20.0"),
)
```

### Logger Provider Options

```go
loggerProvider := log.NewLoggerProvider(
    log.WithResource(resource),
    log.WithBatcher(exporter),
    log.WithProcessor(processor),
)
```

## Log Processing

### Processors

**Simple Processor:**
```go
processor := log.NewSimpleProcessor(exporter)
// Exports logs synchronously
```

**Batch Processor:**
```go
processor := log.NewBatchProcessor(
    exporter,
    log.WithBatchTimeout(5*time.Second),
    log.WithExportTimeout(30*time.Second),
    log.WithMaxQueueSize(2048),
    log.WithMaxBatchSize(512),
)
// Batches logs for efficient export
```

### Example Configuration

```go
exporter, _ := otlploggrpc.New(context.Background(),
    otlploggrpc.WithEndpoint("collector:4317"),
    otlploggrpc.WithInsecure(),
)

loggerProvider := log.NewLoggerProvider(
    log.WithResource(resource),
    log.WithBatcher(exporter),
)
```

## Log Attributes

### Standard Attributes

**Service Attributes:**
```go
attribute.String("service.name", "auth-service"),
attribute.String("service.version", "1.2.3"),
attribute.String("service.instance.id", "auth-1"),
```

**Event Attributes:**
```go
attribute.String("event.name", "user.login"),
attribute.String("event.category", "authentication"),
attribute.String("event.outcome", "success"),
```

**User Attributes:**
```go
attribute.String("user.id", "user-123"),
attribute.String("user.name", "John Doe"),
attribute.String("user.email", "john@example.com"),
attribute.String("user.role", "admin"),
```

**Request Attributes:**
```go
attribute.String("http.method", "POST"),
attribute.String("http.url", "https://api.example.com/login"),
attribute.String("http.user_agent", "Mozilla/5.0..."),
attribute.String("http.x_forwarded_for", "192.168.1.100"),
```

**Error Attributes:**
```go
attribute.String("error.type", "ConnectionError"),
attribute.String("error.message", "connection refused"),
attribute.String("error.stack", "goroutine 1 [running]..."),
```

### Structured Logs

```go
logger.Info("User login successful",
    WithAttributes(
        attribute.String("user.id", "user-123"),
        attribute.String("user.email", "john@example.com"),
        attribute.String("login.method", "password"),
        attribute.String("login.mfa", "true"),
        attribute.String("source.ip", "192.168.1.100"),
        attribute.String("device.type", "mobile"),
        attribute.String("device.os", "iOS"),
    ),
)
```

## Log-Trace Correlation

### Automatic Correlation

When a log is created within a span context, trace information is automatically attached:

```go
// Within a span
ctx, span := tracer.Start(ctx, "login")
defer span.End()

// Log automatically gets trace context
logger.Info("Processing login request",
    WithAttributes(
        attribute.String("user.id", "user-123"),
    ),
)
// Log record includes:
// - Trace ID: from span context
// - Span ID: from span context
```

### Manual Correlation

When logging outside a span, manually add trace context:

```go
spanContext := trace.SpanContextFromContext(ctx)

logger.Info("Background task completed",
    WithAttributes(
        attribute.String("trace.id", spanContext.TraceID().String()),
        attribute.String("span.id", spanContext.SpanID().String()),
        attribute.String("task.id", "task-456"),
        attribute.String("task.status", "completed"),
    ),
)
```

### Extracting Trace Context from Logs

Query logs by trace ID:
```sql
-- SQL-like query
SELECT * FROM logs
WHERE trace_id = '7b9a1f2e3d4c5b6a8f9e0d1c2b3a4f5e'
ORDER BY timestamp;

-- Filter by span
SELECT * FROM logs
WHERE span_id = '1a2b3c4d5e6f7g8h'
ORDER BY timestamp;
```

## Log Exporters

### OTLP Exporter

**gRPC Exporter:**
```go
exporter, _ := otlploggrpc.New(context.Background(),
    otlploggrpc.WithEndpoint("collector:4317"),
    otlploggrpc.WithInsecure(),
)
```

**HTTP Exporter:**
```go
exporter, _ := otlploghttp.New(context.Background(),
    otlploghttp.WithEndpoint("http://collector:4318/v1/logs"),
)
```

### Console Exporter

```go
exporter := log.NewConsoleExporter(
    os.Stdout,
    log.WithPrettyPrint(),
)
```

### File Exporter

```go
exporter, _ := NewFileExporter(
    "/var/log/otel-logs.jsonl",
    log.WithMaxSize(100*1024*1024),  // 100MB
    log.WithMaxAge(7*24*time.Hour), // 7 days
)
```

## Integration with Existing Logging

### Bridge Pattern

Create a bridge between existing logging framework and OTel:

**Zap Bridge:**
```go
type OTelCore struct {
    logger log.Logger
}

func (c *OTelCore) Write(ent zapcore.Entry, fields []zap.Field) error {
    // Convert zap entry to OTel log
    attrs := make([]attribute.KeyValue, 0, len(fields)+2)
    for _, field := range fields {
        attrs = append(attrs, zapToAttr(field))
    }

    c.logger.Emit(
        zapToSeverity(ent.Level),
        ent.Message,
        WithAttributes(attrs...),
    )
    return nil
}

// Create zap logger with OTel core
core := &OTelCore{logger: logger}
zapLogger := zap.New(zapcore.NewCore(core, zapcore.AddSync(os.Stdout), zapcore.InfoLevel))
```

**Logrus Hook:**
```go
type OTelHook struct {
    logger log.Logger
}

func (h *OTelHook) Levels() []logrus.Level {
    return logrus.AllLevels
}

func (h *OTelHook) Fire(entry *logrus.Entry) error {
    attrs := make([]attribute.KeyValue, 0, len(entry.Data))
    for k, v := range entry.Data {
        attrs = append(attrs, attribute.String(k, fmt.Sprintf("%v", v)))
    }

    h.logger.Emit(
        logrusToSeverity(entry.Level),
        entry.Message,
        WithAttributes(attrs...),
    )
    return nil
}

// Add hook to logrus
logger.AddHook(&OTelHook{logger: logger})
```

## Best Practices

### 1. Use Structured Logs

**✅ Good: Structured**
```go
logger.Error("Failed to process payment",
    WithAttributes(
        attribute.String("order.id", "order-123"),
        attribute.String("user.id", "user-456"),
        attribute.Float64("payment.amount", 99.99),
        attribute.String("payment.currency", "USD"),
        attribute.String("error.type", "PaymentDeclined"),
        attribute.String("error.message", "Insufficient funds"),
    ),
)
```

**❌ Bad: Unstructured**
```go
logger.Error(fmt.Sprintf("Failed to process payment for order %s, user %s, amount %f %s: %s",
    "order-123", "user-456", 99.99, "USD", "Insufficient funds"))
```

### 2. Include Context

**✅ Good: With context**
```go
logger.Info("User request received",
    WithAttributes(
        attribute.String("request.id", "req-123"),
        attribute.String("request.method", "POST"),
        attribute.String("request.path", "/api/users"),
        attribute.String("user.id", "user-456"),
        attribute.String("user.ip", "192.168.1.100"),
        attribute.String("user.agent", "Mozilla/5.0..."),
    ),
)
```

**❌ Bad: Minimal context**
```go
logger.Info("User request received")
```

### 3. Use Appropriate Severity

**✅ Good: Correct severity**
```go
logger.Error("Database connection failed")     // Error
logger.Warn("High memory usage detected")       // Warning
logger.Info("User login successful")            // Info
logger.Debug("Cache lookup")                    // Debug
```

**❌ Bad: Incorrect severity**
```go
logger.Fatal("User login failed")              // Should be Error
logger.Debug("User request received")           // Should be Info
logger.Error("High memory usage")               // Should be Warning
```

### 4. Correlate with Traces

**✅ Good: Auto-correlated**
```go
func handleRequest(ctx context.Context, req Request) {
    ctx, span := tracer.Start(ctx, "handleRequest")
    defer span.End()

    // Logs automatically get trace context
    logger.Info("Processing request",
        WithAttributes(
            attribute.String("request.id", req.ID),
        ),
    )
}
```

**❌ Bad: No correlation**
```go
func handleRequest(req Request) {
    logger.Info("Processing request")
    // No way to find traces for this log
}
```

### 5. Sensitive Data

**✅ Good: Redact sensitive data**
```go
logger.Info("User login successful",
    WithAttributes(
        attribute.String("user.id", "user-123"),
        attribute.String("user.email", "***@***.***"),  // Redacted
        // Don't log passwords, tokens, etc.
    ),
)
```

**❌ Bad: Logging sensitive data**
```go
logger.Info("User login successful",
    WithAttributes(
        attribute.String("user.id", "user-123"),
        attribute.String("user.email", "john@example.com"),
        attribute.String("user.password", "secret123"),  // NEVER DO THIS
    ),
)
```

### 6. Performance Considerations

**✅ Good: Batch exports**
```go
loggerProvider := log.NewLoggerProvider(
    log.WithBatcher(exporter,  // Batching for efficiency
        log.WithBatchTimeout(5*time.Second),
        log.WithMaxBatchSize(1000),
    ),
)
```

**❌ Bad: Synchronous exports**
```go
loggerProvider := log.NewLoggerProvider(
    log.WithProcessor(
        log.NewSimpleProcessor(exporter),  // Synchronous, blocking
    ),
)
```

### 7. Sampling

**✅ Good: Sample debug logs**
```go
if rand.Float32() < 0.01 {  // 1% sampling
    logger.Debug("Detailed debug information",
        WithAttributes(
            attribute.String("cache.key", "user:123"),
            attribute.Bool("cache.hit", true),
        ),
    )
}
```

**❌ Bad: All debug logs**
```go
logger.Debug("Detailed debug information")  // Too verbose
```

## Semantic Conventions

### Log Semantic Conventions

**Event Naming:**
```
event.name        - Type of event (e.g., user.login)
event.domain      - Domain of event (e.g., authentication)
event.category    - Category of event (e.g., audit)
event.outcome     - Outcome (success, failure, timeout)
```

**Exception Attributes:**
```
exception.type     - Exception type (e.g., NullPointerException)
exception.message  - Exception message
exception.stacktrace - Stack trace
exception.escaped  - Whether exception escaped (boolean)
```

**Thread Attributes:**
```
thread.id          - Thread ID
thread.name        - Thread name
```

**Code Attributes:**
```
code.function      - Function name
code.namespace     - Namespace/package
code.filepath      - File path
code.lineno        - Line number
```
