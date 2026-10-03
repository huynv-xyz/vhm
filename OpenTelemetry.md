# OpenTelemetry Deep Architecture
## Agent, API, SDK, Context, OTLP, Collector và cách tất cả kết nối với nhau

> Mục tiêu của tài liệu: hiểu OpenTelemetry như một **hệ thống hoàn chỉnh**, không phải học thuộc config.

---

# 1. Bài toán gốc OpenTelemetry giải quyết

Một hệ thống production có thể như sau:

```text
Client
  │
  ▼
API Gateway
  │
  ▼
Order Service
  │
  ├──── PostgreSQL
  │
  └──── Payment Service
             │
             └──── Redis
```

Một request bị chậm hoặc lỗi.

Ta cần trả lời:

```text
Request nào?
↓
Đi qua service nào?
↓
Operation nào chậm?
↓
SQL nào chạy?
↓
Service downstream nào lỗi?
↓
Log nào thuộc đúng request đó?
↓
Toàn hệ thống có bao nhiêu request tương tự?
```

OpenTelemetry giải quyết bài toán bằng cách biến:

```text
runtime execution
```

thành:

```text
telemetry có cấu trúc
```

rồi vận chuyển telemetry đó tới hệ thống lưu trữ/phân tích.

---

# 2. Mental model quan trọng nhất

Toàn bộ OpenTelemetry có thể nhìn như pipeline:

```text
APPLICATION ĐANG CHẠY
        │
        ▼
INSTRUMENTATION
        │
        │ quan sát execution
        ▼
OTEL API
        │
        │ biểu diễn telemetry
        ▼
OTEL SDK
        │
        │ xử lý telemetry
        ▼
EXPORTER
        │
        │ serialize
        ▼
OTLP
        │
        │ network
        ▼
OTEL COLLECTOR
        │
        │ receive → process → export
        ▼
OBSERVABILITY BACKEND
        │
        ├── Tempo / Jaeger       → traces
        ├── Prometheus-compatible → metrics
        └── Loki / Elastic        → logs
        │
        ▼
GRAFANA / QUERY UI
```

Điểm quan trọng:

```text
OpenTelemetry != Grafana
OpenTelemetry != Tempo
OpenTelemetry != Prometheus
OpenTelemetry != database
```

OpenTelemetry chủ yếu chịu trách nhiệm:

```text
generate
correlate
process
transport
```

telemetry.

Backend chịu trách nhiệm:

```text
store
query
analyze
```

---

# 3. Các thành phần lớn của OpenTelemetry

Có thể chia thành:

```text
OpenTelemetry
│
├── Specification
│
├── API
│
├── SDK
│
├── Instrumentation
│   ├── Manual Instrumentation
│   └── Auto / Zero-code Instrumentation
│       └── Java Agent
│
├── Context
├── Propagators
├── Resources
├── Semantic Conventions
├── Sampling
├── Exporters
├── OTLP
│
└── Collector
    ├── Receiver
    ├── Processor
    ├── Exporter
    ├── Connector
    └── Extension
```

---

# 4. Ba thứ dễ nhầm nhất

Đây là phần quan trọng nhất khi dùng OpenTelemetry Java.

```text
opentelemetry-javaagent
opentelemetry-api
opentelemetry-sdk
```

Ba thứ này **không thay thế nhau**.

Chúng giải quyết ba vấn đề khác nhau.

---

# 5. OpenTelemetry Java Agent là gì?

File thường có dạng:

```text
opentelemetry-javaagent.jar
```

Chạy JVM:

```bash
java \
  -javaagent:/opt/otel/opentelemetry-javaagent.jar \
  -jar application.jar
```

Vai trò chính:

> Tự động instrument Java application mà không cần sửa business code.

Ví dụ Spring Boot application có:

```text
Tomcat
Spring MVC
RestClient/WebClient
JDBC
HikariCP
Kafka
...
```

Java Agent có instrumentation cho nhiều framework/library phổ biến.

---

# 6. Java Agent hoạt động ở đâu?

Không phải:

```text
App
   ↓
Agent như một service riêng
```

Mà là:

```text
┌────────────────────────────────────────────┐
│                JVM                         │
│                                            │
│   OpenTelemetry Java Agent                 │
│              │                             │
│              │ instrument bytecode         │
│              ▼                             │
│   ┌───────────────────────────────────┐    │
│   │ Spring Boot Application           │    │
│   │                                   │    │
│   │ Tomcat                            │    │
│   │ Spring MVC                        │    │
│   │ JDBC                              │    │
│   │ HTTP Client                       │    │
│   │ Kafka                             │    │
│   └───────────────────────────────────┘    │
│                                            │
└────────────────────────────────────────────┘
```

Agent chạy **trong cùng JVM** với application.

---

# 7. Java Agent can thiệp như thế nào?

JVM hỗ trợ Java Agent qua:

```text
-javaagent
```

Agent có thể can thiệp khi class được load.

OpenTelemetry Java Agent dùng cơ chế bytecode instrumentation.

Ví dụ code framework về mặt logic có thể là:

```java
handleRequest(request);
```

Agent có thể instrument để hành vi runtime tương đương:

```text
start span
   ↓
handleRequest(request)
   ↓
record status/error
   ↓
end span
```

Business source code của bạn không cần chứa các dòng đó.

---

# 8. Startup flow của Java Agent

Khi chạy:

```bash
java -javaagent:opentelemetry-javaagent.jar -jar app.jar
```

có thể hình dung:

```text
JVM START
   │
   ├── Load Java Agent
   │
   ▼
Agent initialization
   │
   ├── load configuration
   ├── initialize OpenTelemetry SDK
   ├── configure Resource
   ├── configure Sampler
   ├── configure Exporters
   ├── register instrumentations
   │
   ▼
Application classes load
   │
   ▼
Agent instruments supported classes
   │
   ▼
Spring Boot starts
```

Agent không chỉ là "một thư viện tạo Span".

Nó là một zero-code instrumentation runtime.

---

# 9. Agent nhìn thấy request như thế nào?

Ví dụ request:

```text
GET /orders/1
```

Đi qua:

```text
Tomcat
  ↓
Spring MVC
  ↓
OrderController
  ↓
JdbcTemplate
  ↓
PostgreSQL
```

Java Agent có thể tạo:

```text
Trace ABC

GET /orders/{id}             SERVER SPAN
│
└── SELECT orders            CLIENT DB SPAN
```

Bạn không phải viết:

```java
tracer.spanBuilder(...)
```

cho HTTP/JDBC phổ biến.

---

# 10. Agent có phải OpenTelemetry API không?

Không.

```text
Agent
=
automatic instrumentation machinery
```

Trong khi:

```text
API
=
programming contract để tạo/truy cập telemetry
```

Agent **sử dụng các OpenTelemetry abstractions bên dưới** để tạo telemetry.

---

# 11. OpenTelemetry API là gì?

Dependency Java:

```gradle
implementation("io.opentelemetry:opentelemetry-api:...")
```

API cung cấp các abstraction để application/library nói:

```text
Tôi muốn tạo Span
Tôi muốn tạo Metric
Tôi muốn access current Context
Tôi muốn propagate Context
```

Ví dụ:

```java
Tracer tracer = GlobalOpenTelemetry
        .getTracer("order-service");

Span span = tracer
        .spanBuilder("calculate-order-price")
        .startSpan();
```

API không quyết định:

```text
trace gửi đi đâu
batch thế nào
sampling thế nào
OTLP endpoint nào
```

Đó không phải trách nhiệm của API.

---

# 12. Tại sao API phải tách khỏi SDK?

Giả sử một shared library:

```text
payment-client.jar
```

muốn tạo telemetry.

Nếu nó phụ thuộc cứng vào:

```text
Tempo
Datadog
Jaeger
một SDK configuration cụ thể
```

thì library bị coupling với deployment environment.

Thiết kế đúng:

```text
Library
   │
   ▼
OpenTelemetry API
```

Application cuối cùng quyết định implementation/configuration.

Mental model:

```text
API
=
"tôi muốn tạo telemetry"

SDK
=
"tôi sẽ thực hiện việc đó như thế nào"
```

---

# 13. OpenTelemetry API có thể tồn tại mà không có SDK không?

Có.

API được thiết kế để instrumentation/library có thể depend trực tiếp vào nó.

Nếu application không cài một OpenTelemetry implementation phù hợp, API có thể hoạt động theo kiểu no-op.

Nghĩa là:

```text
library gọi tracer
      ↓
không có SDK thực
      ↓
không crash business application
      ↓
không tạo telemetry thực sự
```

Đây là thiết kế quan trọng cho library ecosystem.

---

# 14. Java Agent + OpenTelemetry API kết hợp ra sao?

Đây là case rất phổ biến.

Bạn chạy:

```text
Spring Boot
+
OpenTelemetry Java Agent
```

Agent tự instrument HTTP/JDBC.

Nhưng business operation riêng:

```text
calculate-discount
reserve-inventory
check-credit-limit
```

Agent không thể tự hiểu business meaning.

Bạn thêm:

```gradle
implementation("io.opentelemetry:opentelemetry-api:...")
```

và tạo custom span.

Architecture:

```text
                    SPRING BOOT APP

        ┌──────────────────────────────────┐
        │                                  │
        │ Framework calls                  │
        │ Tomcat / JDBC / HTTP Client      │
        │        ▲                         │
        │        │ auto instrumentation    │
        │   Java Agent                     │
        │                                  │
        │ Business Code                    │
        │        │                         │
        │        │ manual instrumentation  │
        │        ▼                         │
        │    OTel API                      │
        │                                  │
        └──────────────┬───────────────────┘
                       │
                       ▼
                   OTel SDK
```

Agent và manual API instrumentation bổ sung cho nhau.

---

# 15. GlobalOpenTelemetry khi dùng Java Agent

Khi dùng Java Agent, Agent cấu hình OpenTelemetry instance cho application.

Business code có thể lấy:

```java
GlobalOpenTelemetry.get()
```

hoặc API phù hợp theo version.

Sau đó lấy:

```java
Tracer
Meter
```

để tạo custom telemetry.

Flow:

```text
Java Agent
   │
   ├── initialize/configure SDK
   │
   └── register GlobalOpenTelemetry
                  │
                  ▼
Business code using OTel API
                  │
                  ▼
        cùng OpenTelemetry runtime
```

Vì vậy không nên vô tình tạo thêm một SDK thứ hai trong application khi Agent đã quản lý SDK, trừ khi có lý do kiến trúc rất rõ.

---

# 16. OpenTelemetry SDK là gì?

Dependency conceptually:

```text
opentelemetry-sdk
```

SDK là implementation thực sự của OpenTelemetry API.

Nó chịu trách nhiệm:

```text
create telemetry objects
sampling
processing
batching
aggregation
resource association
export
```

Architecture:

```text
Application / Instrumentation
          │
          ▼
        OTel API
          │
          ▼
        OTel SDK
```

---

# 17. SDK không phải một class duy nhất

SDK được chia theo signal.

```text
OpenTelemetry SDK
│
├── Trace SDK
│
│   └── SdkTracerProvider
│
├── Metrics SDK
│
│   └── SdkMeterProvider
│
└── Logs SDK
    └── SdkLoggerProvider
```

Đây là ba engine lớn cho:

```text
Traces
Metrics
Logs
```

---

# 18. Trace SDK internals

Trace pipeline có thể hiểu:

```text
Instrumentation
      │
      ▼
    Tracer
      │
      ▼
    Span
      │
      ▼
   Sampler
      │
      ▼
SpanProcessor
      │
      ▼
SpanExporter
      │
      ▼
   OTLP
```

Chi tiết hơn:

```text
SdkTracerProvider
      │
      ├── owns Tracer configuration
      ├── Resource
      ├── Sampler
      └── SpanProcessors
```

---

# 19. TracerProvider

`TracerProvider` là factory/root configuration cho tracing.

Concept:

```text
TracerProvider
      │
      ├── Tracer A
      ├── Tracer B
      └── Tracer C
```

Instrumentation lấy một `Tracer`.

Tracer tạo Span.

```text
TracerProvider
      ↓
Tracer
      ↓
SpanBuilder
      ↓
Span
```

---

# 20. Tracer

Tracer là object instrumentation sử dụng để tạo Span.

Ví dụ:

```java
Tracer tracer =
    openTelemetry.getTracer("order-service");
```

Tên instrumentation thường dùng để xác định:

```text
instrumentation scope
```

không phải service identity.

Service identity nên nằm ở:

```text
Resource
```

ví dụ:

```text
service.name=order-service
```

---

# 21. Span

Span biểu diễn một operation.

Ví dụ:

```text
Span

name       = POST /orders
trace_id   = ABC
span_id    = 001
parent_id  = null
kind       = SERVER
start      = ...
end        = ...
status     = OK

attributes:
  http.request.method = POST
  http.route = /orders
```

Một Trace là nhiều Span có chung `trace_id`.

---

# 22. Span Kind

Một Span còn có semantic role.

Các loại thường gặp:

```text
SERVER
CLIENT
PRODUCER
CONSUMER
INTERNAL
```

Ví dụ:

```text
Incoming HTTP request
→ SERVER

Outgoing HTTP request
→ CLIENT

Kafka publish
→ PRODUCER

Kafka consume
→ CONSUMER

Business calculation
→ INTERNAL
```

Span Kind giúp backend hiểu quan hệ giữa các operation.

---

# 23. Sampler

Khi một root span bắt đầu, sampling policy có thể quyết định:

```text
record/export?
```

Ví dụ:

```text
AlwaysOn
AlwaysOff
TraceIdRatioBased
ParentBased
```

Concept:

```text
new request
   ↓
Sampler
   │
   ├── sampled
   │      ↓
   │   record/export
   │
   └── not sampled
```

Sampling chủ yếu nhằm kiểm soát:

```text
CPU
network
storage
cost
```

---

# 24. SpanProcessor

SpanProcessor nằm giữa Span lifecycle và exporter.

Hai kiểu tư duy phổ biến:

```text
SimpleSpanProcessor

span.end()
   ↓
export ngay
```

và:

```text
BatchSpanProcessor

span.end()
   ↓
queue
   ↓
batch
   ↓
export
```

Production thường cần batching thay vì network call cho từng Span.

---

# 25. SpanExporter

SpanExporter nhận finished spans từ processor rồi export.

Ví dụ:

```text
OTLP Span Exporter
```

Flow:

```text
Span
 ↓
SpanProcessor
 ↓
OTLP SpanExporter
 ↓
Collector
```

---

# 26. Metrics SDK internals

Metrics khác tracing.

Tracing:

```text
request cụ thể
```

Metrics:

```text
aggregation của nhiều measurements
```

Flow:

```text
Instrumentation
      │
      ▼
     Meter
      │
      ▼
  Instruments
      │
      ├── Counter
      ├── Histogram
      └── Gauge-like instruments
      │
      ▼
Metrics SDK aggregation
      │
      ▼
 MetricReader
      │
      ▼
MetricExporter
```

---

# 27. MeterProvider

Tương tự TracerProvider:

```text
SdkMeterProvider
      ↓
Meter
      ↓
Instrument
```

Ví dụ:

```java
Meter meter = openTelemetry.getMeter("order-service");
```

---

# 28. Metric Instruments

Một số concept quan trọng:

```text
Counter
Histogram
UpDownCounter
Observable instruments
```

Ví dụ request duration:

```text
10ms
20ms
90ms
400ms
...
```

Histogram giúp backend/metrics system hiểu distribution.

---

# 29. MetricReader

Metrics không đơn giản là:

```text
instrument.end()
→ export
```

như Span.

Metrics cần collection/aggregation.

`MetricReader` là phần kết nối SDK metrics với quá trình đọc/export dữ liệu.

Ví dụ:

```text
PeriodicMetricReader
        │
        │ mỗi khoảng thời gian
        ▼
MetricExporter
```

---

# 30. Logs SDK internals

Logs pipeline:

```text
Log instrumentation / bridge
       │
       ▼
Logger
       │
       ▼
LogRecord
       │
       ▼
LogRecordProcessor
       │
       ▼
LogRecordExporter
       │
       ▼
OTLP
```

Logs có thể được correlate với tracing thông qua:

```text
trace_id
span_id
```

---

# 31. Context là gì?

Đây là core của distributed tracing.

Context là object mang state liên quan execution hiện tại.

Ví dụ:

```text
Context
│
├── current Span
│   ├── TraceId
│   └── SpanId
│
└── Baggage
```

Nếu operation B xảy ra trong operation A:

```text
Context của A
      ↓
B đọc current context
      ↓
B tạo child span
```

---

# 32. Context trong một JVM

Ví dụ:

```text
HTTP Server Span
      │
      ▼
Controller
      │
      ▼
Service
      │
      ▼
Repository
```

Current Context phải đi cùng execution.

Concept:

```text
Thread / execution
      │
      ▼
Current Context
      │
      ▼
Current Span
```

Khi code tạo một child span, SDK biết parent dựa trên current context.

---

# 33. Scope

Trong Java manual instrumentation thường có concept:

```text
Span.makeCurrent()
```

trả về một `Scope`.

Concept:

```java
Span span = tracer.spanBuilder("calculate-price").startSpan();

try (Scope scope = span.makeCurrent()) {
    doWork();
} finally {
    span.end();
}
```

Trong block đó:

```text
current context
     ↓
contains current span
```

Child instrumentation có thể nhận đúng parent.

---

# 34. Async làm Context khó hơn

Ví dụ:

```text
Thread A
   │
   ├── create context
   │
   └── submit task
            │
            ▼
         Thread B
```

Nếu Context không được propagate:

```text
Thread B
→ mất parent
→ trace bị gãy
```

Instrumentation cho executors/reactive frameworks thường tồn tại chính để giữ Context xuyên execution boundary.

---

# 35. Cross-service propagation

Trong cùng JVM, Context nằm trong runtime.

Nhưng giữa hai service:

```text
Service A
    │
    │ network
    ▼
Service B
```

không thể truyền Java object trực tiếp.

Cần serialize Context.

Đó là vai trò của:

```text
TextMapPropagator
```

và các propagation formats.

---

# 36. W3C Trace Context

Format phổ biến mặc định là W3C Trace Context.

HTTP header:

```text
traceparent
```

Concept:

```text
traceparent:
version-traceid-parentid-flags
```

Ví dụ:

```text
00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

Service A:

```text
inject Context
      ↓
HTTP headers
```

Service B:

```text
extract HTTP headers
      ↓
Context
```

---

# 37. Distributed tracing flow

```text
SERVICE A

Server Span
trace=ABC
span=001
   │
   ▼
Client Span
trace=ABC
span=002
   │
   │ inject
   ▼
HTTP request
traceparent = trace ABC / parent 002
   │
   ▼
NETWORK
   │
   ▼
SERVICE B
   │
   │ extract
   ▼
Parent Context
trace=ABC
parent=002
   │
   ▼
Server Span
trace=ABC
span=003
parent=002
```

Kết quả:

```text
Trace ABC

Span 001 Service A server
└── Span 002 Service A client
    └── Span 003 Service B server
```

---

# 38. Propagator là gì?

Propagator chịu trách nhiệm:

```text
Context object
   ↕
carrier
```

Carrier có thể là:

```text
HTTP headers
Kafka headers
message metadata
gRPC metadata
```

Hai operation chính:

```text
inject()
extract()
```

---

# 39. Baggage

Baggage là key-value data được propagate cùng request context.

Ví dụ:

```text
tenant.id=abc
```

Nhưng phải cẩn thận.

Không nên đưa:

```text
password
access token
PII
secret
```

vào Baggage.

Vì Baggage có thể đi xuyên nhiều service.

---

# 40. Resource

Resource mô tả **entity sinh telemetry**.

Ví dụ:

```text
service.name=order-service
service.version=1.4.0

deployment.environment.name=production

host.name=node-01

process.pid=123

k8s.namespace.name=marketplace
k8s.pod.name=order-service-abcd
```

Resource thường gắn với nhiều telemetry records.

---

# 41. Resource Detector

Không phải tất cả Resource attributes đều cần hard-code.

Resource Detectors có thể phát hiện thông tin runtime/environment.

Ví dụ:

```text
process
host
OS
container
cloud
Kubernetes
```

Tư duy:

```text
runtime environment
      ↓
Resource Detector
      ↓
Resource attributes
```

---

# 42. Instrumentation Scope

Một khái niệm hay bị bỏ qua:

```text
InstrumentationScope
```

Nó trả lời:

> Telemetry này được tạo bởi instrumentation/library nào?

Ví dụ:

```text
name = io.opentelemetry.jdbc
version = ...
```

Phân biệt:

```text
Resource
=
ai đang chạy?

InstrumentationScope
=
ai tạo telemetry?
```

---

# 43. Semantic Conventions

Nếu mỗi team dùng:

```text
http_method
http.method
request_method
```

thì telemetry không còn interoperable.

Semantic Conventions chuẩn hóa:

```text
attribute names
span names
metric names
resource attributes
meaning
```

Ví dụ:

```text
service.name
http.request.method
http.response.status_code
server.address
db.system.name
```

---

# 44. Exporter là gì?

Exporter là adapter đưa SDK data ra ngoài.

Trace:

```text
SpanExporter
```

Metrics:

```text
MetricExporter
```

Logs:

```text
LogRecordExporter
```

Production thường ưu tiên:

```text
OTLP Exporter
```

---

# 45. OTLP là gì?

OTLP:

```text
OpenTelemetry Protocol
```

Nó định nghĩa cách truyền telemetry theo OpenTelemetry data model.

Có thể chạy trên:

```text
OTLP/gRPC
OTLP/HTTP
```

Tư duy:

```text
OTel internal telemetry
      ↓
OTLP serialization
      ↓
network
      ↓
Collector/backend
```

---

# 46. Tại sao OTLP quan trọng?

Nếu không có protocol chuẩn:

```text
App
├── Jaeger protocol
├── vendor A protocol
├── vendor B protocol
└── custom format
```

Application coupling với backend.

Có OTLP:

```text
App
  │
  ▼
OTLP
  │
  ▼
Collector
  │
  ├── backend A
  ├── backend B
  └── backend C
```

---

# 47. OpenTelemetry Collector

Collector là telemetry data pipeline độc lập với application.

Core mental model:

```text
RECEIVE
   ↓
PROCESS
   ↓
EXPORT
```

Architecture:

```text
             ┌─────────────────────────┐
OTLP ───────▶│ Receiver                │
             │    ↓                    │
             │ Processor               │
             │    ↓                    │
             │ Exporter                │──────▶ Backend
             └─────────────────────────┘
```

---

# 48. Receiver

Receiver nhận telemetry.

Ví dụ:

```text
OTLP Receiver
Prometheus Receiver
Jaeger Receiver
Kafka Receiver
```

Receiver không phải database.

Nó là input adapter.

---

# 49. Processor

Processor biến đổi telemetry giữa input và output.

Ví dụ:

```text
memory_limiter
batch
attributes
resource
filter
transform
tail_sampling
```

Flow:

```text
Receiver
   ↓
Memory Limiter
   ↓
Filter
   ↓
Transform
   ↓
Batch
   ↓
Exporter
```

---

# 50. Exporter của Collector

Exporter gửi telemetry ra backend.

Ví dụ:

```text
OTLP exporter
Kafka exporter
Prometheus-compatible exporters
vendor exporters
```

Cần phân biệt:

```text
SDK Exporter
```

và:

```text
Collector Exporter
```

Flow:

```text
Application SDK
      │
      ▼
SDK OTLP Exporter
      │
      ▼
Collector Receiver
      │
      ▼
Collector Processor
      │
      ▼
Collector Exporter
      │
      ▼
Backend
```

---

# 51. Collector Pipeline

Collector config thường có concept:

```yaml
service:
  pipelines:
    traces:
      receivers: [...]
      processors: [...]
      exporters: [...]
```

Mental model:

```text
pipeline = input + processing + output
```

Có thể có:

```text
traces pipeline
metrics pipeline
logs pipeline
```

---

# 52. Connector

Connector nối hai pipelines.

Nó có thể nhìn như:

```text
Pipeline A
    │
    ▼
Connector
    │
    ▼
Pipeline B
```

Ví dụ:

```text
Traces
   ↓
Span Metrics Connector
   ↓
Metrics
```

Tức là dữ liệu traces có thể tạo ra metrics-derived information.

---

# 53. Extension

Extension không nằm trực tiếp trong telemetry pipeline.

Nó cung cấp chức năng phụ cho Collector.

Ví dụ:

```text
health check
authentication helpers
diagnostics
management
```

---

# 54. Collector có tạo Trace không?

Thông thường:

```text
Application instrumentation
```

mới là nơi tạo tracing spans cho business execution.

Collector chủ yếu:

```text
receive
process
export
```

telemetry đã được tạo.

Không nên hiểu:

```text
Collector = tool tự nhìn vào business code và tạo mọi Span
```

---

# 55. Backend

Sau Collector là backend.

Ví dụ:

```text
Tempo / Jaeger
→ trace storage/query

Prometheus-compatible backend
→ metrics storage/query

Loki / Elastic
→ logs storage/query
```

OpenTelemetry không bắt buộc một backend cụ thể.

Đây là phần "vendor-neutral".

---

# 56. Grafana

Grafana là visualization/query layer.

Ví dụ:

```text
Grafana
│
├── query Tempo
├── query Prometheus
└── query Loki
```

Grafana không phải OpenTelemetry SDK.

Grafana không tạo TraceId.

Grafana không propagate Context.

---

# 57. Spring Boot + Java Agent: runtime architecture đầy đủ

```text
┌────────────────────────────────────────────────────────────┐
│                         JVM                                │
│                                                            │
│  ┌──────────────── OpenTelemetry Java Agent ────────────┐  │
│  │                                                      │  │
│  │ Bytecode Instrumentation                             │  │
│  │ Auto Instrumentation Modules                         │  │
│  │ SDK Autoconfiguration                                │  │
│  │ Resource Detection                                   │  │
│  │ Propagators                                          │  │
│  │ Exporters                                            │  │
│  └───────────────────────┬──────────────────────────────┘  │
│                          │                                 │
│                          ▼                                 │
│  ┌──────────────── Spring Boot App ─────────────────────┐  │
│  │                                                      │  │
│  │ Tomcat                                               │  │
│  │   ↓                                                  │  │
│  │ Spring MVC                                           │  │
│  │   ↓                                                  │  │
│  │ Controller                                           │  │
│  │   ↓                                                  │  │
│  │ Service ───── Manual custom span via OTel API        │  │
│  │   ↓                                                  │  │
│  │ JDBC                                                 │  │
│  │                                                      │  │
│  └──────────────────────┬───────────────────────────────┘  │
│                         │                                  │
│                         ▼                                  │
│                    OTel SDK                               │
│                         │                                  │
│          ┌──────────────┼──────────────┐                   │
│          ▼              ▼              ▼                   │
│      Trace SDK      Metrics SDK      Logs SDK              │
│          │              │              │                   │
│          └──────────────┼──────────────┘                   │
│                         ▼                                  │
│                    OTLP Exporter                           │
└─────────────────────────┬──────────────────────────────────┘
                          │
                          │ OTLP
                          ▼
                 OpenTelemetry Collector
```

---

# 58. Một request thực tế chạy qua hệ thống như thế nào?

Request:

```text
GET /orders/1
```

## Step 1 — Request vào Tomcat

Java Agent đã instrument HTTP server stack.

```text
incoming request
      ↓
agent instrumentation
      ↓
create SERVER span
```

Ví dụ:

```text
TraceId = ABC
SpanId  = 001
```

---

# 59. Step 2 — Context được đặt thành current

```text
Span 001
   ↓
make/current context
   ↓
Controller
   ↓
Service
```

Mọi instrumentation bên dưới có thể đọc parent hiện tại.

---

# 60. Step 3 — JDBC query

Service gọi:

```text
JdbcTemplate
```

Agent đã instrument JDBC.

Nó thấy current context:

```text
Trace ABC
Span 001
```

và tạo:

```text
Span 002
TraceId = ABC
Parent = 001
```

Trace lúc này:

```text
GET /orders/1       span=001
└── SELECT orders   span=002
```

---

# 61. Step 4 — Custom business Span

Business code:

```text
calculate-discount
```

Bạn dùng OTel API tạo Span.

API gọi SDK.

SDK lấy current Context.

Tạo:

```text
Span 003
TraceId = ABC
Parent = 001
```

---

# 62. Step 5 — Gọi Payment Service

HTTP client instrumentation tạo:

```text
Span 004
TraceId = ABC
Parent = 001
Kind = CLIENT
```

Sau đó Propagator inject:

```text
traceparent
```

vào outgoing HTTP headers.

---

# 63. Step 6 — Payment Service nhận request

Agent ở Payment Service extract:

```text
traceparent
```

khôi phục parent Context.

Sau đó tạo SERVER span:

```text
Span 005
TraceId = ABC
Parent = 004
```

Trace:

```text
Trace ABC

001 GET /orders/1             order-service SERVER
├── 002 SELECT orders         DB CLIENT
├── 003 calculate-discount    INTERNAL
└── 004 POST /payments        HTTP CLIENT
    └── 005 POST /payments    payment-service SERVER
```

---

# 64. Step 7 — Span kết thúc

Khi operation hoàn tất:

```text
span.end()
```

SpanProcessor nhận finished span.

Ví dụ:

```text
Span
  ↓
BatchSpanProcessor
  ↓
queue
```

---

# 65. Step 8 — Export

Đến batch interval hoặc queue condition:

```text
BatchSpanProcessor
      ↓
OTLP Span Exporter
      ↓
serialize OTLP
      ↓
network
```

---

# 66. Step 9 — Collector nhận

```text
OTLP Receiver
      ↓
Processors
      ↓
Exporter
      ↓
Tempo
```

---

# 67. Step 10 — Backend ghép Trace

Tempo nhận nhiều spans:

```text
001
002
003
004
005
```

Dựa trên:

```text
trace_id
span_id
parent_span_id
```

backend reconstruct:

```text
GET /orders
├── SELECT
├── calculate-discount
└── payment
```

---

# 68. Toàn bộ request flow

```text
CLIENT
  │
  ▼
HTTP Request
  │
  ▼
Java Agent HTTP Instrumentation
  │
  ▼
SERVER SPAN 001
Trace=ABC
  │
  ├───────────────┐
  │               │
  ▼               ▼
JDBC Agent     Manual OTel API
  │               │
  ▼               ▼
SPAN 002        SPAN 003
  │
  └───────┬───────┘
          │
          ▼
HTTP Client Instrumentation
          │
          ▼
CLIENT SPAN 004
          │
          ▼
Propagator.inject()
          │
          ▼
traceparent
          │
────────── NETWORK ──────────
          │
          ▼
Propagator.extract()
          │
          ▼
Payment Agent
          │
          ▼
SERVER SPAN 005
Trace=ABC Parent=004
          │
          ▼
OTel SDK
          │
          ▼
SpanProcessor
          │
          ▼
OTLP Exporter
          │
          ▼
Collector Receiver
          │
          ▼
Collector Processors
          │
          ▼
Collector Exporter
          │
          ▼
Tempo
          │
          ▼
Grafana
```

---

# 69. Agent vs API vs SDK — bảng chốt

| Thành phần | Câu hỏi nó giải quyết |
|---|---|
| Java Agent | Làm sao quan sát framework/library mà không sửa source code? |
| OTel API | Code/library dùng interface nào để tạo và truy cập telemetry? |
| OTel SDK | Ai thực sự xử lý, sample, batch và export telemetry? |
| Context | Operation hiện tại thuộc Trace/parent nào? |
| Propagator | Làm sao Context đi qua HTTP/Kafka/gRPC boundary? |
| Resource | Telemetry này do service/process/pod nào sinh ra? |
| Semantic Conventions | Dữ liệu được đặt tên/diễn giải thống nhất thế nào? |
| Exporter | Làm sao SDK gửi telemetry ra ngoài? |
| OTLP | Telemetry đi qua network bằng protocol chuẩn nào? |
| Collector | Làm sao nhận, xử lý và route telemetry tập trung? |
| Backend | Telemetry được lưu và query ở đâu? |
| Grafana | Con người xem/phân tích telemetry ở đâu? |

---

# 70. Ba mô hình triển khai Java thường gặp

## Model A — Java Agent

```text
Application
+
-javaagent
```

Agent:

```text
auto instrument
+
auto configure SDK
+
export
```

Phù hợp nhất để bắt đầu với Spring Boot.

---

# 71. Model B — Manual API + SDK

Application tự:

```text
add opentelemetry-api
add opentelemetry-sdk
configure SDK
configure exporter
write instrumentation
```

Flow:

```text
Application
   ↓
OTel API
   ↓
SDK configured by application
```

Kiểm soát cao hơn nhưng nhiều code/config hơn.

---

# 72. Model C — Agent + custom API instrumentation

Đây là model thực tế rất mạnh:

```text
Java Agent
→ framework telemetry

OTel API
→ business telemetry
```

Ví dụ:

```text
Java Agent:
GET /orders
SELECT database
POST payment

Manual API:
validate-order
calculate-pricing
reserve-inventory
```

Cả hai dùng chung Trace/Context.

---

# 73. Sai lầm thường gặp

## Sai 1

```text
Java Agent = Collector
```

Sai.

```text
Java Agent
=
instrument application

Collector
=
process/route telemetry
```

---

## Sai 2

```text
API = SDK
```

Sai.

```text
API
=
contract

SDK
=
implementation/runtime engine
```

---

## Sai 3

```text
Agent thay thế API
```

Không hoàn toàn.

Agent giỏi instrument generic frameworks.

API cần khi business logic cần custom telemetry.

---

## Sai 4

```text
Collector lưu trace
```

Không phải vai trò chính.

Backend như Tempo/Jaeger lưu trace.

---

## Sai 5

```text
TraceId tự truyền bằng X-Trace-Id custom header
```

Không nên làm nếu mục tiêu là distributed tracing chuẩn.

Dùng propagation chuẩn:

```text
traceparent
```

---

## Sai 6

```text
Tạo Span cho mọi method
```

Không nên.

Span nên biểu diễn operation có giá trị observability.

---

# 74. Khi nào Agent tự tạo Span?

Agent thường hữu ích ở các boundary:

```text
incoming HTTP
outgoing HTTP
database
messaging
RPC
framework operations
```

Tức là các điểm:

```text
system boundary
network boundary
IO boundary
```

---

# 75. Khi nào nên dùng manual API?

Khi muốn thấy operation mà framework không hiểu.

Ví dụ:

```text
check-customer-eligibility
calculate-campaign-score
reserve-lead
evaluate-distribution-policy
```

Đây là business semantics.

Agent không thể tự suy ra đúng ý nghĩa nghiệp vụ.

---

# 76. First-principles summary

Bỏ tất cả tên framework đi, OpenTelemetry chỉ đang giải quyết chuỗi vấn đề sau:

```text
Một operation xảy ra
       ↓
cần mô tả operation
       ↓
Span

Nhiều operations liên quan
       ↓
cần nối chúng
       ↓
Trace

Execution đi qua code/thread/service
       ↓
cần giữ quan hệ nhân quả
       ↓
Context

Context đi qua network
       ↓
cần serialize/deserialize
       ↓
Propagation

Code/framework cần tạo telemetry
       ↓
API + Instrumentation

Telemetry cần được xử lý
       ↓
SDK

Telemetry cần ra khỏi process
       ↓
Exporter

Các hệ thống cần một protocol chung
       ↓
OTLP

Telemetry cần routing/processing tập trung
       ↓
Collector

Telemetry cần lưu/query
       ↓
Backend

Con người cần xem
       ↓
Grafana
```

---

# 77. Sơ đồ cuối cùng cần nhớ

```text
                           JVM
┌────────────────────────────────────────────────────┐
│                                                    │
│  Java Agent                                        │
│      │                                             │
│      ├── auto instrument Tomcat                    │
│      ├── auto instrument Spring                    │
│      ├── auto instrument JDBC                      │
│      ├── auto instrument HTTP Client               │
│      └── configure OTel runtime                    │
│                                                    │
│  Application                                       │
│      │                                             │
│      ├── framework code ───────────────┐           │
│      │                                │            │
│      └── business code                │            │
│              │                        │            │
│              ▼                        │            │
│          OTel API                     │            │
│              │                        │            │
│              └────────────┬───────────┘            │
│                           ▼                        │
│                         SDK                        │
│                           │                        │
│      ┌────────────────────┼──────────────────┐     │
│      ▼                    ▼                  ▼     │
│  TracerProvider       MeterProvider     LoggerProvider
│      │                    │                  │     │
│      ▼                    ▼                  ▼     │
│   Processor            Reader             Processor │
│      │                    │                  │     │
│      └────────────────────┼──────────────────┘     │
│                           ▼                        │
│                     OTLP Exporter                  │
│                                                    │
└───────────────────────────┬────────────────────────┘
                            │
                            │ OTLP/gRPC or OTLP/HTTP
                            ▼
                 ┌────────────────────────┐
                 │ OTel Collector         │
                 │                        │
                 │ Receiver               │
                 │    ↓                   │
                 │ Processor              │
                 │    ↓                   │
                 │ Exporter               │
                 └────────────┬───────────┘
                              │
               ┌──────────────┼──────────────┐
               ▼              ▼              ▼
             Tempo        Prometheus        Loki
               │              │              │
               └──────────────┼──────────────┘
                              ▼
                            Grafana
```

---

# 78. Thứ tự nên học

Không nên học Collector trước.

Học theo thứ tự:

```text
1. Span
2. Trace
3. TraceId / SpanId / ParentSpanId
4. Context
5. Propagation
6. Java Agent
7. OpenTelemetry API
8. OpenTelemetry SDK
9. TracerProvider / SpanProcessor / Exporter
10. Resource
11. Semantic Conventions
12. OTLP
13. Collector
14. Metrics SDK
15. Logs SDK
16. Sampling
```

Nếu hiểu sâu từ bước 1 đến 9, phần còn lại sẽ rất dễ nối vào.

---

# 79. Một câu chốt

OpenTelemetry có thể được hiểu như sau:

> **Agent/Instrumentation nhìn thấy execution, API biểu diễn ý định telemetry, SDK xử lý telemetry, Context giữ quan hệ nhân quả, Propagator đưa quan hệ đó qua network, OTLP vận chuyển dữ liệu, Collector xử lý/routing tập trung, backend lưu dữ liệu, Grafana hiển thị dữ liệu.**

---

# 80. Tài liệu chính thức tham khảo

- OpenTelemetry Components  
  https://opentelemetry.io/docs/concepts/components/

- OpenTelemetry Java  
  https://opentelemetry.io/docs/languages/java/

- OpenTelemetry Java Agent  
  https://opentelemetry.io/docs/zero-code/java/agent/

- OpenTelemetry Java API  
  https://opentelemetry.io/docs/languages/java/api/

- Configure OpenTelemetry Java SDK  
  https://opentelemetry.io/docs/languages/java/configuration/

- Java Agent + OpenTelemetry API  
  https://opentelemetry.io/docs/zero-code/java/agent/api/

- Context Propagation  
  https://opentelemetry.io/docs/concepts/context-propagation/

- OpenTelemetry Collector  
  https://opentelemetry.io/docs/collector/
