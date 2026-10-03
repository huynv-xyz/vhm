# OpenTelemetry — Cách hoạt động, các thành phần và luồng kết nối

## 1. Bản chất của OpenTelemetry

OpenTelemetry không phải database, không phải dashboard và cũng không phải một hệ thống monitoring hoàn chỉnh.

Bản chất của nó là một chuẩn và bộ công cụ để:

```text
Hệ thống đang chạy
        ↓
Quan sát
        ↓
Tạo telemetry
        ↓
Chuẩn hóa
        ↓
Vận chuyển
        ↓
Xử lý
        ↓
Gửi tới backend observability
```

Ba loại telemetry chính:

```text
Trace   → request này đã đi đâu?
Metric  → toàn hệ thống đang như thế nào?
Log     → sự kiện cụ thể nào đã xảy ra?
```

---

# 2. Luồng tổng thể

```text
APPLICATION
    │
    │ được quan sát
    ▼
INSTRUMENTATION
    │
    │ tạo telemetry
    ▼
OTEL API / SDK
    │
    │ xử lý + chuẩn hóa
    ▼
EXPORTER
    │
    │ gửi bằng OTLP
    ▼
OTEL COLLECTOR
    │
    │ receive → process → export
    ▼
OBSERVABILITY BACKEND
    │
    ├── Tempo
    ├── Prometheus
    └── Loki
          │
          ▼
       Grafana
```

---

# 3. Instrumentation

Instrumentation là điểm bắt đầu.

Ứng dụng bình thường:

```java
public Order getOrder(long id) {
    return repository.findById(id);
}
```

Business code này không tự tạo telemetry.

Instrumentation có nhiệm vụ:

```text
operation bắt đầu
      ↓
ghi nhận
      ↓
operation chạy
      ↓
ghi duration / error / metadata
      ↓
operation kết thúc
```

Kết quả có thể tạo ra một Span:

```text
Span
name     = GET /orders/{id}
start    = 10:00:00.000
end      = 10:00:00.200
duration = 200ms
status   = OK
```

Có hai loại:

```text
Manual Instrumentation
```

Bạn tự gọi OpenTelemetry API.

Hoặc:

```text
Automatic Instrumentation
```

Agent/library tự instrument framework.

Ví dụ Java Agent có thể tự instrument:

```text
Tomcat
Spring MVC
JDBC
HTTP Client
Kafka
...
```

---

# 4. Span

Span là đơn vị cơ bản nhất của tracing.

Có thể hiểu:

> Span = một operation có thời điểm bắt đầu và kết thúc.

Ví dụ:

```text
Span
┌──────────────────────────────┐
│ name: SELECT orders          │
│                              │
│ start: 10:00:00.010          │
│ end:   10:00:00.050          │
│                              │
│ duration: 40ms               │
│ db.system = postgresql       │
│ status = OK                  │
└──────────────────────────────┘
```

Một HTTP request có thể là một Span.

Một SQL query cũng có thể là một Span.

Một HTTP call sang service khác cũng có thể là một Span.

---

# 5. Trace

Một request thường tạo nhiều Span.

Ví dụ:

```text
POST /orders
     │
     ├── SELECT product
     ├── INSERT order
     └── POST payment
```

Các Span đó được nối lại thành một Trace.

```text
TraceId = ABC123

POST /orders                 SpanId=01
│
├── SELECT product           SpanId=02
├── INSERT order             SpanId=03
└── POST payment             SpanId=04
```

Ý nghĩa:

```text
TraceId
```

Xác định toàn bộ distributed request.

```text
SpanId
```

Xác định một operation cụ thể.

```text
ParentSpanId
```

Cho biết operation cha.

Có thể hình dung:

```text
Trace = cây execution

Span = một node trong cây
```

---

# 6. Context

Đây là một trong những phần quan trọng nhất của OpenTelemetry.

Giả sử:

```text
Order Service
      │
      │ HTTP
      ▼
Payment Service
```

Order Service đang có:

```text
TraceId = ABC
SpanId  = 100
```

Payment Service chạy ở một process hoặc Pod khác.

Nó cần biết:

```text
request hiện tại thuộc Trace ABC
```

Thông tin này được giữ trong:

```text
Context
```

Ví dụ:

```text
Current Context
┌────────────────────┐
│ TraceId = ABC      │
│ SpanId  = 100      │
│ TraceFlags = ...   │
└────────────────────┘
```

Context mang thông tin execution hiện tại.

---

# 7. Context Propagation

Context trong Service A phải đi sang Service B.

Đó là nhiệm vụ của Propagation.

```text
Service A

Context
TraceId=ABC
SpanId=100
    │
    │ Inject
    ▼

HTTP Header

traceparent:
00-ABC-100-01

    │
    │ Network
    ▼

Service B
    │
    │ Extract
    ▼

Context
TraceId=ABC
ParentSpanId=100
```

Service B sau đó tạo Span mới:

```text
TraceId = ABC
SpanId  = 200
Parent  = 100
```

Kết quả:

```text
Service A                     Service B

Span 100
GET /order
    │
    │ HTTP + traceparent
    ▼
                              Span 200
                              POST /payment
                              parent=100
```

Đây chính là nền tảng của distributed tracing.

---

# 8. Resource

Một Span cho biết chuyện gì xảy ra.

Nhưng cần biết ai tạo ra Span đó.

Đó là vai trò của Resource.

Ví dụ:

```text
service.name = order-service

service.version = 1.2.0

deployment.environment.name = production

k8s.namespace.name = marketplace

k8s.pod.name = order-service-abc123
```

Cách nhớ:

```text
Span
=
chuyện gì xảy ra

Resource
=
telemetry này thuộc về ai
```

---

# 9. Semantic Conventions

Nếu mỗi team tự đặt tên field:

```text
http_method

http.method

requestMethod
```

thì query telemetry sau này sẽ rất khó.

OpenTelemetry đưa ra Semantic Conventions.

Mục tiêu:

```text
cùng một ý nghĩa
      ↓
cùng một cách đặt tên
```

Ví dụ:

```text
service.name

http.request.method

http.response.status_code

db.system.name

server.address
```

---

# 10. OpenTelemetry API

Application có thể cần tự tạo custom telemetry.

Ví dụ custom Span:

```text
validate-order
```

Application gọi OpenTelemetry API.

Conceptually:

```java
Tracer tracer = openTelemetry.getTracer("order");

Span span = tracer
        .spanBuilder("validate-order")
        .startSpan();
```

Application không cần biết:

```text
Tempo là gì
Collector ở đâu
OTLP endpoint nào
backend nào đang được dùng
```

Nó chỉ làm việc với API.

Cách nhớ:

```text
API = contract
```

---

# 11. OpenTelemetry SDK

API chỉ định nghĩa cách application tương tác.

SDK mới là engine xử lý thực tế.

```text
Application
    ↓
OTel API
    ↓
OTel SDK
```

SDK có thể xử lý:

```text
sampling

span processing

metric aggregation

batching

resource handling

exporting
```

Cách nhớ:

```text
API = giao diện

SDK = engine
```

---

# 12. Bên trong SDK

Riêng tracing có thể hình dung:

```text
Application
    ↓
Tracer
    ↓
Span
    ↓
SpanProcessor
    ↓
Exporter
```

Ví dụ:

```text
HTTP request
    ↓
Tracer
    ↓
Span
GET /orders
    ↓
BatchSpanProcessor
    ↓
OTLP Exporter
```

SpanProcessor có thể gom nhiều Span rồi mới gửi đi.

```text
span finished
     ↓
buffer
     ↓
batch
     ↓
export
```

---

# 13. Sampling

Nếu hệ thống có lượng traffic lớn:

```text
100,000 request/s
```

việc giữ tất cả traces có thể rất tốn tài nguyên.

Sampling quyết định:

```text
request nào được giữ trace
```

Ví dụ:

```text
100 requests
     ↓
Sampler
     ↓
10 requests được giữ
```

Mục tiêu:

```text
visibility
vs
cost
```

---

# 14. Exporter

Telemetry sau khi được tạo vẫn đang nằm trong application.

Exporter có nhiệm vụ đưa nó ra ngoài.

```text
SDK
 ↓
Exporter
 ↓
Network
```

Exporter phổ biến:

```text
OTLP Exporter
```

---

# 15. OTLP

OTLP là:

```text
OpenTelemetry Protocol
```

Nó là protocol chuẩn để vận chuyển telemetry.

Thay vì application phải hiểu:

```text
Tempo protocol
Jaeger protocol
Datadog protocol
Elastic protocol
...
```

application chỉ cần:

```text
OTLP
```

Flow:

```text
Application
     ↓
   OTLP
     ↓
Collector
```

Có thể dùng:

```text
OTLP/gRPC
```

hoặc:

```text
OTLP/HTTP
```

---

# 16. OpenTelemetry Collector

Collector là trạm trung chuyển telemetry.

Nó không phải backend lưu trữ.

Bản chất của Collector:

```text
receive
  ↓
process
  ↓
export
```

Architecture:

```text
Application
    │
    │ OTLP
    ▼

┌──────────────────────────┐
│ OTel Collector           │
│                          │
│ Receiver                 │
│    ↓                     │
│ Processor                │
│    ↓                     │
│ Exporter                 │
└────────────┬─────────────┘
             │
             ▼
          Backend
```

---

# 17. Collector Receiver

Receiver là cửa vào Collector.

Nó trả lời:

> telemetry đi vào Collector bằng cách nào?

Ví dụ:

```text
OTLP Receiver
Prometheus Receiver
Jaeger Receiver
Kafka Receiver
```

Flow:

```text
Application
    │
    │ OTLP
    ▼
OTLP Receiver
```

---

# 18. Collector Processor

Processor xử lý telemetry sau khi nhận.

```text
Receiver
   ↓
Processor
```

Processor có thể:

```text
batch

filter

add attributes

remove sensitive data

sampling

resource detection

transform
```

Ví dụ:

```text
1000 spans
   ↓
Memory Limiter
   ↓
Filter
   ↓
Batch
   ↓
Exporter
```

---

# 19. Collector Exporter

Sau khi xử lý, Collector phải gửi telemetry tới backend.

```text
Processor
    ↓
Exporter
    ↓
Backend
```

Ví dụ:

```text
OTLP exporter → Tempo

Prometheus exporter → metrics backend

Kafka exporter → Kafka
```

Cần phân biệt:

```text
Application SDK Exporter
        │
        │ OTLP
        ▼
Collector

Collector Exporter
        │
        ▼
Backend
```

---

# 20. Connector và Extension

Đây là phần nâng cao.

Collector còn có:

```text
Connector
```

dùng để nối hai pipeline.

Ví dụ:

```text
Traces pipeline
      ↓
Span Metrics Connector
      ↓
Metrics pipeline
```

Ngoài ra còn có:

```text
Extension
```

dùng cho các chức năng phụ như:

```text
health check
authentication
debug
management
```

---

# 21. Backend không phải OpenTelemetry

Các hệ thống như:

```text
Tempo
Prometheus
Loki
Jaeger
Datadog
Elastic
Grafana
```

không phải core OpenTelemetry.

Vai trò có thể hiểu:

```text
OpenTelemetry
=
generate + correlate + transport + process telemetry

Backend
=
store + query telemetry

Grafana
=
visualize telemetry
```

Ví dụ stack:

```text
Trace   → Tempo
Metric  → Prometheus
Log     → Loki

             ↓

           Grafana
```

---

# 22. Toàn bộ kiến trúc OpenTelemetry

```text
                        APPLICATION
                             │
                             │
              ┌──────────────┴──────────────┐
              │                             │
     Auto Instrumentation          Manual Instrumentation
       Java Agent...                 OTel API
              │                             │
              └──────────────┬──────────────┘
                             │
                             ▼
                       OpenTelemetry
                           SDK
                             │
             ┌───────────────┼────────────────┐
             │               │                │
             ▼               ▼                ▼
           Trace           Metric            Log
             │               │                │
             └───────────────┼────────────────┘
                             │
                             │ Resource
                             │ Semantic Convention
                             │ Context
                             │ Sampling
                             ▼
                         Exporter
                             │
                             │ OTLP
                             ▼
                ┌─────────────────────────┐
                │ OTel Collector          │
                │                         │
                │ Receiver                │
                │    ↓                    │
                │ Processor               │
                │    ↓                    │
                │ Exporter                │
                └────────────┬────────────┘
                             │
              ┌──────────────┼───────────────┐
              ▼              ▼               ▼

            Tempo        Prometheus         Loki
           Traces         Metrics           Logs

              └──────────────┬───────────────┘
                             ▼
                           Grafana
```

---

# 23. Distributed Tracing Flow

```text
                   trace_id = ABC

Client
  │
  ▼

┌───────────────────────────────┐
│ ORDER SERVICE                 │
│                               │
│ Java Agent                    │
│      ↓                        │
│ HTTP Span                     │
│ trace=ABC span=01             │
│      │                        │
│      ├── JDBC Span            │
│      │   trace=ABC span=02    │
│      │                        │
│      └── HTTP Client Span     │
│          trace=ABC span=03    │
└──────────────┬────────────────┘
               │
               │ traceparent
               │ trace=ABC
               ▼

┌───────────────────────────────┐
│ PAYMENT SERVICE               │
│                               │
│ extract context               │
│        ↓                      │
│ HTTP Server Span              │
│ trace=ABC span=04             │
│ parent=03                     │
└──────────────┬────────────────┘
               │
               │ telemetry
               ▼
          OTel SDK
               │
             OTLP
               │
               ▼
       OTel Collector
               │
               ▼
             Tempo
```

---

# 24. Trace, Metric và Log chạy song song

Một request có thể sinh cả ba loại telemetry:

```text
                       REQUEST
                          │
           ┌──────────────┼──────────────┐
           │              │              │
           ▼              ▼              ▼

         TRACE          METRIC           LOG

TraceId=ABC          requests=1       ERROR payment
SpanId=123           duration=500ms    trace_id=ABC

           │              │              │
           └──────────────┼──────────────┘
                          │
                          ▼
                     Collector
```

Trace trả lời:

```text
request nào?
đi qua đâu?
chậm ở đâu?
```

Metric trả lời:

```text
bao nhiêu request?
p95 bao nhiêu?
error rate bao nhiêu?
```

Log trả lời:

```text
chính xác sự kiện hoặc lỗi nào đã xảy ra?
```

---

# 25. Vai trò từng thành phần

| Thành phần | Vai trò |
|---|---|
| Instrumentation | Quan sát application/framework và sinh telemetry |
| Span | Mô tả một operation |
| Trace | Nối các operation của cùng một request |
| Context | Giữ thông tin execution hiện tại |
| Propagator | Truyền Context giữa service |
| Resource | Xác định service/pod/process tạo telemetry |
| Semantic Convention | Chuẩn hóa tên và ý nghĩa dữ liệu |
| OTel API | Interface application dùng để instrument |
| OTel SDK | Engine xử lý telemetry |
| Sampler | Quyết định telemetry nào được giữ |
| Exporter | Đưa telemetry ra khỏi SDK |
| OTLP | Protocol chuẩn vận chuyển telemetry |
| Collector Receiver | Nhận telemetry |
| Collector Processor | Filter/transform/batch/sample/enrich |
| Collector Exporter | Gửi telemetry tới backend |
| Backend | Lưu và query telemetry |
| Grafana | Hiển thị và phân tích telemetry |

---

# 26. Cách nhớ ngắn nhất

Chỉ cần nhớ flow:

```text
CODE CHẠY
   ↓
Instrumentation
   ↓
Span / Metric / Log
   ↓
Context
   ↓
SDK
   ↓
OTLP
   ↓
Collector
   ↓
Backend
   ↓
Grafana
```

Với nhiều service:

```text
Service A
    │
    │ Context / traceparent
    ▼
Service B
```

Phần quan trọng nhất để hiểu sâu OpenTelemetry:

```text
Operation
   ↓
Span
   ↓
Trace
   ↓
Context
   ↓
Propagation
```

Còn:

```text
Java Agent
   ↓
SDK
   ↓
OTLP
   ↓
Collector
   ↓
Tempo
```

chỉ là cách:

```text
tạo
→ vận chuyển
→ xử lý
→ lưu
```

telemetry.
