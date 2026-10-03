# OpenTelemetry Deep Architecture (v2)
## Từ mental model đến vận hành production: Agent, API, SDK, Context, Sampling, Metrics, Logs, OTLP, Collector

> **Mục tiêu:** hiểu OpenTelemetry như một **hệ thống hoàn chỉnh**, đủ sâu để thiết kế, triển khai và debug trong production, chứ không chỉ học thuộc config.
>
> **Phạm vi:** các khái niệm chung áp dụng cho mọi ngôn ngữ. Ví dụ code dùng **Java / Spring Boot**.
>
> **Lưu ý version:** OpenTelemetry thay đổi nhanh, đặc biệt ở Semantic Conventions, Logs và Collector. Tên attribute, default config và tên component trong tài liệu này phản ánh các bản phổ biến gần đây (Java Agent 2.x, Collector contrib). Khi triển khai thật, luôn đối chiếu với tài liệu chính thức của version bạn dùng.

---

## Mục lục

- **Phần I — Nền tảng**: bài toán, ba signal, mental model, bản đồ thành phần
- **Phần II — Tracing core**: Span, Trace, ID, Span Kind, Status, Events, Links
- **Phần III — Context & Propagation**: Context, Scope, async, W3C Trace Context, messaging, Baggage
- **Phần IV — Java Agent**: cơ chế, startup, cấu hình, extension, overhead
- **Phần V — OpenTelemetry API**: manual instrumentation, GlobalOpenTelemetry, no-op
- **Phần VI — Trace SDK**: TracerProvider, SpanProcessor, Exporter, Span limits
- **Phần VII — Sampling**: head vs tail, ParentBased, tail sampling trong Collector
- **Phần VIII — Metrics**: instruments, aggregation, temporality, Views, Exemplars, cardinality
- **Phần IX — Logs**: Logs Bridge, appender, correlation
- **Phần X — Resource, Instrumentation Scope, Semantic Conventions**
- **Phần XI — Exporter & OTLP**
- **Phần XII — Collector**: pipeline, components, deployment patterns, reliability, security
- **Phần XIII — Backend & Grafana**: lưu trữ, correlation giữa các signal
- **Phần XIV — Một request đi qua toàn hệ thống**
- **Phần XV — Mô hình triển khai Java**
- **Phần XVI — Bảng tham chiếu cấu hình (env vars)**
- **Phần XVII — Troubleshooting**
- **Phần XVIII — Anti-patterns & checklist production**
- **Phần XIX — Lab thực hành**
- **Phần XX — Tổng kết, lộ trình học, thuật ngữ, tài liệu tham khảo**

---

# PHẦN I — NỀN TẢNG

# 1. Bài toán gốc OpenTelemetry giải quyết

Một hệ thống production điển hình:

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
  ├──── Kafka ───► Notification Service
  │
  └──── Payment Service
             │
             └──── Redis
```

Một request bị chậm hoặc lỗi. Ta cần trả lời:

```text
Request nào?
↓
Đi qua service nào?
↓
Operation nào chậm?
↓
SQL nào chạy, mất bao lâu?
↓
Service downstream nào lỗi?
↓
Log nào thuộc đúng request đó?
↓
Toàn hệ thống có bao nhiêu request tương tự? Tỉ lệ lỗi bao nhiêu?
↓
Vấn đề bắt đầu từ lúc nào, sau deploy version nào?
```

Không có công cụ nào trả lời được tất cả câu hỏi trên bằng **một loại dữ liệu**. Đó là lý do observability cần nhiều signal.

OpenTelemetry giải quyết bài toán bằng cách biến:

```text
runtime execution
```

thành:

```text
telemetry có cấu trúc, có chuẩn đặt tên, có thể liên kết với nhau
```

rồi vận chuyển telemetry đó tới hệ thống lưu trữ/phân tích **mà không khóa chặt vào một vendor nào**.

---

# 2. Ba signal chính (và signal thứ tư)

| Signal | Trả lời câu hỏi | Đặc điểm | Chi phí |
|---|---|---|---|
| **Traces** | Request *này* đã đi đâu, chậm ở đâu? | Chi tiết theo từng request, có quan hệ cha–con | Cao, thường phải sample |
| **Metrics** | Hệ thống *nói chung* đang thế nào? | Số liệu tổng hợp theo thời gian | Thấp, nếu kiểm soát được cardinality |
| **Logs** | Chính xác chuyện gì đã xảy ra tại thời điểm đó? | Sự kiện rời rạc, nhiều text | Trung bình đến cao |
| **Profiles** | Dòng code / hàm nào tốn CPU, memory? | Stack trace lấy mẫu liên tục | Đang được chuẩn hóa trong OTel, chưa stable |

Cách các signal bổ trợ nhau trong một sự cố:

```text
Metrics:  "Tỉ lệ lỗi /orders tăng từ 0.1% lên 5% lúc 14:02"
            │
            │  exemplar / filter theo thời gian
            ▼
Traces:   "Request lỗi đều chậm ở span POST payment-service/charge"
            │
            │  trace_id
            ▼
Logs:     "payment-service: Redis connection pool exhausted"
```

Giá trị lớn nhất của OpenTelemetry không phải là thu thập từng signal riêng lẻ, mà là **liên kết chúng với nhau** qua chung `trace_id`, `span_id` và `Resource`.

---

# 3. Mental model quan trọng nhất

Toàn bộ OpenTelemetry có thể nhìn như một pipeline:

```text
APPLICATION ĐANG CHẠY
        │
        ▼
INSTRUMENTATION  (Agent tự động / code thủ công)
        │
        │ quan sát execution
        ▼
OTEL API
        │
        │ biểu diễn ý định tạo telemetry
        ▼
OTEL SDK
        │
        │ sample, xử lý, batch, aggregate, gắn Resource
        ▼
EXPORTER
        │
        │ serialize
        ▼
OTLP  (gRPC hoặc HTTP)
        │
        │ network
        ▼
OTEL COLLECTOR  (tùy chọn nhưng nên có)
        │
        │ receive → process → export
        ▼
OBSERVABILITY BACKEND
        │
        ├── Tempo / Jaeger         → traces
        ├── Prometheus / Mimir     → metrics
        └── Loki / Elasticsearch   → logs
        │
        ▼
GRAFANA / QUERY UI
```

Ranh giới trách nhiệm:

```text
OpenTelemetry != Grafana
OpenTelemetry != Tempo
OpenTelemetry != Prometheus
OpenTelemetry != database
OpenTelemetry != vendor APM
```

| Lớp | Trách nhiệm |
|---|---|
| OpenTelemetry | **generate, correlate, process, transport** telemetry |
| Backend | **store, index, query** |
| UI (Grafana, …) | **visualize, alert, explore** |

---

# 4. Bản đồ thành phần

```text
OpenTelemetry
│
├── Specification            ← bản thiết kế chung, ngôn ngữ trung lập
│
├── Per-language implementation
│   ├── API                  ← interface để tạo telemetry
│   ├── SDK                  ← implementation thật
│   └── Instrumentation
│       ├── Manual Instrumentation
│       ├── Instrumentation Libraries (cho từng framework)
│       └── Zero-code Instrumentation
│           └── Java Agent / .NET auto / Python auto / eBPF ...
│
├── Cross-cutting concepts
│   ├── Context
│   ├── Propagators
│   ├── Baggage
│   ├── Resource
│   ├── Instrumentation Scope
│   ├── Semantic Conventions
│   └── Sampling
│
├── Data model & protocol
│   ├── Exporters
│   └── OTLP
│
└── Collector
    ├── Receiver
    ├── Processor
    ├── Exporter
    ├── Connector
    └── Extension
```

---

# 5. Specification, stability và tại sao bạn nên quan tâm

OpenTelemetry có một **Specification** chung. Mỗi ngôn ngữ (Java, Go, Python, .NET, JS, …) implement theo spec đó. Vì vậy khái niệm học ở Java có thể dùng lại gần như nguyên vẹn ở ngôn ngữ khác.

Mỗi phần có mức độ ổn định riêng:

| Phần | Trạng thái chung (tham khảo) |
|---|---|
| Tracing API/SDK | Stable |
| Metrics API/SDK | Stable |
| Logs Bridge API / SDK | Stable ở nhiều ngôn ngữ, mức độ khác nhau |
| OTLP | Stable cho traces, metrics, logs |
| Semantic Conventions | Một số nhóm stable (HTTP, một phần DB), nhiều nhóm vẫn đang phát triển |
| Profiles | Đang phát triển |
| Collector | Core components đã có bản 1.x; nhiều component contrib vẫn alpha/beta |

Hệ quả thực tế: **tên attribute có thể đổi giữa các version**. Dashboard và alert dựa trên tên cũ có thể "im lặng" hỏng sau khi nâng version Agent. Luôn đọc release notes khi upgrade.

---

# PHẦN II — TRACING CORE

# 6. Span: đơn vị cơ bản

Span biểu diễn **một operation có thời gian bắt đầu và kết thúc**.

Cấu trúc đầy đủ của một Span:

```text
Span
├── name               = "POST /orders"
├── trace_id           = 4bf92f3577b34da6a3ce929d0e0e4736   (16 bytes, 32 hex)
├── span_id            = 00f067aa0ba902b7                   (8 bytes, 16 hex)
├── parent_span_id     = (rỗng nếu là root span)
├── trace_state        = vendor-specific state (W3C tracestate)
├── kind               = SERVER
├── start_time         = 2026-10-03T07:00:00.000000Z
├── end_time           = 2026-10-03T07:00:00.183000Z
├── status             = UNSET | OK | ERROR (+ description)
├── attributes         = key/value có kiểu
│     http.request.method       = POST
│     http.route                = /orders
│     http.response.status_code = 201
├── events             = các mốc thời gian bên trong span
│     [t+12ms] "cache.miss"
│     [t+150ms] "exception" {exception.type, exception.message, exception.stacktrace}
├── links              = liên kết tới span của trace khác
├── resource           = service.name=order-service, ...   (gắn bởi SDK)
└── instrumentation_scope = io.opentelemetry.tomcat-10.0     (ai tạo span này)
```

Một **Trace** là tập hợp các Span có chung `trace_id`, tạo thành cây nhờ `parent_span_id`.

```text
Trace 4bf92f...
│
└── [SERVER]   POST /orders                      183ms
    ├── [INTERNAL] validate-order                  4ms
    ├── [CLIENT]   SELECT orders                  12ms
    ├── [PRODUCER] order-created publish           3ms
    └── [CLIENT]   POST payment/charge           140ms
        └── [SERVER] POST /charge  (payment-svc) 135ms
            └── [CLIENT] Redis GET               2ms
```

---

# 7. TraceId, SpanId, ParentSpanId

| ID | Kích thước | Ai tạo | Ý nghĩa |
|---|---|---|---|
| `trace_id` | 16 bytes | Root span (service đầu tiên trong chuỗi) | Định danh cả hành trình request |
| `span_id` | 8 bytes | Mỗi span tự tạo | Định danh một operation |
| `parent_span_id` | 8 bytes | Lấy từ Context hiện tại | Nối span con với cha |

Quy tắc:

```text
Có parent trong Context (local hoặc extract từ header)
    → dùng lại trace_id của parent
    → parent_span_id = span_id của parent

Không có parent
    → tạo trace_id mới
    → span này là ROOT span
```

Nếu bạn thấy một request "đáng lẽ là một trace" lại bị tách thành **nhiều trace rời rạc**, gần như chắc chắn là Context bị mất ở đâu đó (xem Phần III và Phần XVII).

---

# 8. Span Kind

Span Kind cho backend biết **vai trò** của span trong quan hệ giữa các service.

| Kind | Khi nào | Ví dụ |
|---|---|---|
| `SERVER` | Nhận một request đồng bộ từ bên ngoài | Incoming HTTP, gRPC server |
| `CLIENT` | Gửi một request đồng bộ ra ngoài | Outgoing HTTP, JDBC query, Redis call |
| `PRODUCER` | Gửi message bất đồng bộ | Kafka publish, RabbitMQ send |
| `CONSUMER` | Xử lý message bất đồng bộ | Kafka consume/process |
| `INTERNAL` | Operation nội bộ, không vượt process boundary | Business calculation (mặc định của manual span) |

Tại sao quan trọng:

```text
CLIENT span (service A) ──► SERVER span (service B)
        = một cạnh trong Service Graph  A → B

PRODUCER span ──► CONSUMER span
        = một cạnh bất đồng bộ qua message broker
```

Backend như Tempo dùng cặp CLIENT/SERVER và PRODUCER/CONSUMER để vẽ **service graph** và tính latency giữa các service. Đặt sai kind (ví dụ gọi HTTP ra ngoài nhưng để INTERNAL) sẽ làm graph thiếu cạnh.

---

# 9. Span Status

Ba giá trị:

```text
UNSET   ← mặc định; nghĩa là "không có gì bất thường được ghi nhận"
OK      ← developer/operator khẳng định rõ ràng là thành công
ERROR   ← operation thất bại
```

Quy tắc thường dùng trong instrumentation HTTP:

| Tình huống | SERVER span | CLIENT span |
|---|---|---|
| 2xx, 3xx | UNSET | UNSET |
| 4xx | UNSET (lỗi do client, server vẫn làm đúng) | ERROR |
| 5xx | ERROR | ERROR |
| Exception / timeout | ERROR | ERROR |

Lưu ý:

- Instrumentation library thường **không** set `OK`. Để UNSET là đúng.
- Bạn có thể set `OK` trong business code nếu muốn khẳng định kết quả cuối cùng, và `OK` không bị ghi đè bởi instrumentation khác.
- Ghi exception không tự động set status ERROR trong mọi trường hợp. Khi tự viết manual span, hãy làm cả hai:

```java
span.recordException(e);
span.setStatus(StatusCode.ERROR, "payment declined");
```

---

# 10. Attributes

Attributes là key/value **có kiểu** (string, boolean, long, double, hoặc mảng các kiểu này).

Nguyên tắc:

- Dùng tên theo **Semantic Conventions** khi có (`http.request.method`, không phải `method`).
- Attribute nghiệp vụ nên có namespace riêng: `order.id`, `order.item_count`, `customer.tier`.
- Không đưa dữ liệu nhạy cảm (password, token, số thẻ, PII không cần thiết).
- Có giới hạn số lượng và độ dài (xem Span Limits, mục 44).

Phân biệt attribute nằm ở đâu:

```text
Resource attributes    → mô tả "ai" sinh telemetry, giống nhau cho mọi span của process
                          service.name, service.version, k8s.pod.name

Span attributes        → mô tả operation cụ thể này
                          http.route, db.operation.name, order.id
```

Đừng lặp `service.name` trên từng span. Nó thuộc về Resource.

---

# 11. Span Events

Event là **một mốc thời gian có tên** bên trong span, kèm attributes.

```text
Span: checkout                                  0ms ────────────── 300ms
   │
   ├── event "cart.loaded"          @ 15ms
   ├── event "coupon.applied"       @ 40ms   {coupon.code=SALE10}
   └── event "exception"            @ 290ms  {exception.type=..., exception.stacktrace=...}
```

Khi nào dùng Event thay vì Span con:

| Dùng Span con | Dùng Event |
|---|---|
| Operation có duration đáng quan tâm | Chỉ là một thời điểm |
| Có thể có lỗi riêng, status riêng | Đánh dấu chuyện gì đã xảy ra |
| Gọi ra ngoài (IO) | Retry lần thứ N, cache miss, state change |

`recordException()` thực chất tạo một event tên `exception` theo semantic conventions.

---

# 12. Span Links

Quan hệ cha–con chỉ biểu diễn được **một** parent. Nhiều tình huống thực tế không vừa với mô hình cây:

```text
Batch consumer nhận 100 message trong một lần poll
   → 100 message có thể đến từ 100 trace khác nhau
   → span "process batch" không thể có 100 parent
```

Giải pháp: **Links**.

```text
Trace A ── span PRODUCER (msg 1) ◄──┐
Trace B ── span PRODUCER (msg 2) ◄──┼── links ── span CONSUMER "process batch" (Trace X)
Trace C ── span PRODUCER (msg 3) ◄──┘
```

Các trường hợp dùng Links:

- Batch processing trong messaging.
- Job được trigger bởi nhiều sự kiện.
- Một request mới (trace mới) nhưng muốn tham chiếu trace cũ, ví dụ scheduled retry.
- Ranh giới tin cậy: nhận `traceparent` từ bên ngoài nhưng muốn bắt đầu trace mới, vẫn giữ liên kết tới trace gốc.

Links phải được thêm **lúc tạo span** (để sampler có thể dùng đến), không phải sau đó.

---

# 13. Đặt tên Span

Tên span nên **ít biến thiên** (low cardinality). Giá trị cụ thể để vào attributes.

| Sai | Đúng |
|---|---|
| `GET /orders/12345` | `GET /orders/{id}` |
| `SELECT * FROM orders WHERE id=12345` | `SELECT orders` |
| `process-order-ORD-998` | `process-order` + attribute `order.id=ORD-998` |
| `calculate` | `calculate-order-price` |

Lý do: backend group/aggregate theo span name. Tên chứa ID tạo ra hàng triệu "loại operation" khác nhau, làm hỏng dashboard và các metrics sinh ra từ span (spanmetrics).

---

# PHẦN III — CONTEXT & PROPAGATION

# 14. Context là gì?

Đây là **trái tim** của distributed tracing.

Context là một object **bất biến (immutable)** mang state gắn với execution hiện tại.

```text
Context
│
├── current Span
│   ├── TraceId
│   ├── SpanId
│   └── TraceFlags (sampled?)
│
└── Baggage
    └── tenant.id=abc
```

Nếu operation B xảy ra trong operation A:

```text
Context của A
      ↓
B đọc current Context
      ↓
B tạo child span với parent = span của A
```

Vì Context bất biến, "thay đổi" Context thực chất là **tạo Context mới** từ Context cũ cộng thêm giá trị, rồi đặt nó làm current.

---

# 15. Context trong một process

```text
HTTP Server Span (đặt làm current)
      │
      ▼
Controller
      │
      ▼
Service        ← tạo span "calculate-price": parent tự động là HTTP Server Span
      │
      ▼
Repository     ← JDBC span: parent là "calculate-price" nếu nó đang current
```

Ở Java, "current Context" được lưu theo thread (dựa trên `ThreadLocal` hoặc cơ chế tương đương). Khi code tạo span mới mà không chỉ định parent, SDK lấy parent từ current Context.

---

# 16. Scope: đặt và gỡ Context

```java
Span span = tracer.spanBuilder("calculate-price").startSpan();

try (Scope scope = span.makeCurrent()) {   // đặt span thành current
    doWork();                              // mọi span con tạo ở đây có parent đúng
} catch (Exception e) {
    span.recordException(e);
    span.setStatus(StatusCode.ERROR);
    throw e;
} finally {
    span.end();                            // kết thúc span
}                                          // scope.close() khôi phục Context cũ
```

Hai thao tác **độc lập** cần phân biệt:

| Thao tác | Ý nghĩa | Quên thì sao |
|---|---|---|
| `span.end()` | Ghi nhận thời điểm kết thúc, đưa span cho SpanProcessor | Span không bao giờ được export |
| `scope.close()` | Khôi phục Context trước đó trên thread | **Context leak**: request sau trên cùng thread dùng nhầm parent, trace bị trộn lẫn |

Luôn dùng `try-with-resources` cho Scope.

---

# 17. Async làm Context khó hơn

```text
Thread A (có Context X)
   │
   └── submit task ───► Thread B (không có Context X)
                              │
                              └── tạo span → không có parent → trace mới → trace bị gãy
```

Các tình huống hay gặp:

| Tình huống | Java Agent tự xử lý? | Cách xử lý thủ công |
|---|---|---|
| `ExecutorService`, `CompletableFuture` | Có, với executor phổ biến | `Context.taskWrapping(executor)` hoặc `Context.current().wrap(runnable)` |
| Spring `@Async` | Có | — |
| Reactor / WebFlux | Có (instrumentation riêng) | Hook context của Reactor |
| Kotlin Coroutines | Có (khi dùng agent) | `Context.current().asContextElement()` |
| Virtual threads (Java 21+) | Có với các bản agent gần đây | Như executor thường |
| Thread pool tự viết, queue nội bộ tự viết | **Không** | Tự lưu Context vào task object và `makeCurrent()` khi xử lý |
| Callback từ native library | **Không** | Tự truyền Context |

Ví dụ thủ công:

```java
ExecutorService executor = Context.taskWrapping(Executors.newFixedThreadPool(8));

// hoặc với một task đơn lẻ
Runnable task = Context.current().wrap(() -> doWork());
executor.submit(task);
```

---

# 18. Cross-service propagation

Trong một process, Context là object trong memory. Giữa hai service, không thể gửi Java object qua network. Cần **serialize Context vào carrier**.

```text
Service A                                         Service B
Context ──inject()──► HTTP headers ──network──► HTTP headers ──extract()──► Context
```

Thành phần làm việc này là **Propagator** (`TextMapPropagator`).

| Khái niệm | Ý nghĩa |
|---|---|
| Carrier | Nơi chứa dữ liệu truyền đi: HTTP headers, Kafka headers, gRPC metadata, AMQP properties |
| `inject()` | Ghi Context hiện tại vào carrier |
| `extract()` | Đọc carrier, tạo Context chứa remote parent |
| Format | Cách mã hóa: W3C Trace Context, W3C Baggage, B3, Jaeger, AWS X-Ray |

---

# 19. W3C Trace Context

Format mặc định của OpenTelemetry. Gồm hai header.

**`traceparent`** (bắt buộc):

```text
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ── ──────────────────────────────── ──────────────── ──
             │                 │                         │         │
          version          trace-id                parent-id    trace-flags
          (2 hex)          (32 hex)                (16 hex)      (2 hex)
```

| Trường | Ý nghĩa |
|---|---|
| version | Hiện là `00` |
| trace-id | ID của cả trace |
| parent-id | span_id của **span gửi request** (thường là CLIENT span ở service A) |
| trace-flags | Bit thấp nhất là **sampled**: `01` = đã sample, `00` = không sample |

**`tracestate`** (tùy chọn): danh sách key=value cho vendor, ví dụ thông tin sampling probability.

```text
tracestate: ot=th:8;rv:9b8233f7e3a151, vendor2=xyz
```

Cờ `sampled` rất quan trọng: nó cho service downstream biết upstream đã quyết định sample hay chưa, để dùng chung quyết định (xem `ParentBased` sampler, Phần VII).

---

# 20. Distributed tracing flow chi tiết

```text
SERVICE A
──────────
Server Span   trace=ABC span=001
   │
   ▼
Client Span   trace=ABC span=002 parent=001
   │
   │ propagator.inject(Context{span 002})
   ▼
HTTP request
   traceparent: 00-ABC-002-01
   │
   ▼
════════════ NETWORK ════════════
   │
   ▼
SERVICE B
──────────
propagator.extract(headers)
   │
   ▼
Remote parent Context  trace=ABC parent=002 sampled=true
   │
   ▼
Server Span   trace=ABC span=003 parent=002
```

Kết quả:

```text
Trace ABC
Span 001  Service A  SERVER
└── Span 002  Service A  CLIENT
    └── Span 003  Service B  SERVER
```

Mỗi service export span của mình **độc lập**. Backend ghép lại thành cây nhờ `trace_id` và `parent_span_id`.

---

# 21. Propagation qua messaging (Kafka, RabbitMQ)

Messaging khác HTTP ở chỗ **bất đồng bộ**: producer có thể kết thúc từ lâu trước khi consumer xử lý.

```text
Order Service                       Kafka                    Notification Service
─────────────                       ─────                    ────────────────────
PRODUCER span "orders publish"
   │ inject traceparent
   ▼                         record headers:
send(record) ─────────────►  traceparent=00-ABC-P01-01  ─────► poll()
                                                                 │ extract
                                                                 ▼
                                                        CONSUMER span "orders process"
                                                        trace=ABC parent=P01
```

Hai mô hình thường gặp:

| Mô hình | Cách nối | Phù hợp |
|---|---|---|
| Parent–child | Consumer span là con của producer span | Xử lý từng message, muốn thấy một trace liền mạch |
| Links | Consumer span bắt đầu trace mới, link tới producer span | Batch consume, latency giữa produce và consume rất lớn |

Java Agent instrument Kafka client và tự inject/extract header. Nếu bạn dùng thư viện hoặc broker không được hỗ trợ, phải tự gọi propagator.

---

# 22. Chọn và cấu hình Propagator

Mặc định: `tracecontext,baggage`.

```bash
OTEL_PROPAGATORS=tracecontext,baggage
```

Khi hệ thống có service cũ dùng format khác (ví dụ Zipkin/B3), có thể bật nhiều propagator cùng lúc:

```bash
OTEL_PROPAGATORS=tracecontext,baggage,b3multi
```

Khi inject, tất cả format được ghi. Khi extract, propagator tìm thấy format nào thì dùng format đó.

Nguyên tắc: **toàn hệ thống phải thống nhất ít nhất một format chung**. Chỉ một service dùng format lệch là trace bị cắt đôi tại đó.

---

# 23. Baggage

Baggage là **key-value do ứng dụng định nghĩa**, đi theo Context xuyên suốt các service.

```text
baggage: tenant.id=abc,request.channel=mobile
```

Ví dụ dùng:

```java
Baggage.current().toBuilder()
       .put("tenant.id", "abc")
       .build()
       .makeCurrent();   // nhớ đóng Scope
```

Ba điều hay bị hiểu sai:

1. **Baggage không tự động trở thành span attribute.** Nó chỉ đi theo Context. Muốn thấy trên span, phải tự copy, hoặc dùng một processor kiểu "baggage span processor" (có trong các thư viện contrib).
2. **Baggage được gửi tới mọi downstream**, kể cả bên thứ ba nếu bạn gọi API ngoài. Tuyệt đối không đưa password, token, PII, secret.
3. Baggage làm **tăng kích thước header** ở mọi request. Giữ ít và ngắn.

| Nên | Không nên |
|---|---|
| `tenant.id`, `feature.flag`, `request.channel` | `access_token`, `email`, `phone`, JSON lớn |

---

# PHẦN IV — JAVA AGENT

# 24. Ba thứ dễ nhầm nhất

```text
opentelemetry-javaagent.jar
opentelemetry-api
opentelemetry-sdk
```

Ba thứ này **không thay thế nhau**, mà giải quyết ba vấn đề khác nhau:

| Thành phần | Vấn đề giải quyết |
|---|---|
| Java Agent | Quan sát framework/library mà không sửa source code |
| API | Interface để code tạo telemetry |
| SDK | Engine xử lý, sample, batch, export telemetry thật |

Java Agent **đóng gói sẵn SDK bên trong**. Khi dùng Agent, bạn **không** thêm `opentelemetry-sdk` vào ứng dụng. Bạn chỉ thêm `opentelemetry-api` nếu cần custom instrumentation.

---

# 25. Java Agent là gì?

```bash
java \
  -javaagent:/opt/otel/opentelemetry-javaagent.jar \
  -Dotel.service.name=order-service \
  -jar application.jar
```

Vai trò:

> **Tự động instrument Java application tại runtime mà không cần sửa business code.**

Agent có instrumentation cho hàng trăm library/framework: Servlet/Tomcat/Jetty/Undertow, Spring MVC/WebFlux, JDBC, HikariCP, Hibernate, Kafka, RabbitMQ, gRPC, OkHttp, Apache HttpClient, Java `HttpClient`, Redis (Lettuce/Jedis), MongoDB, AWS SDK, Logback/Log4j, executors, Reactor, …

Agent chạy **trong cùng JVM** với application, không phải một service riêng:

```text
┌────────────────────────────────────────────┐
│                JVM                         │
│                                            │
│   OpenTelemetry Java Agent                 │
│     ├── bytecode instrumentation           │
│     ├── SDK (isolated, autoconfigured)     │
│     └── exporters                          │
│              │                             │
│              │ instrument bytecode         │
│              ▼                             │
│   ┌───────────────────────────────────┐    │
│   │ Spring Boot Application           │    │
│   │   Tomcat / Spring MVC / JDBC /    │    │
│   │   HTTP Client / Kafka             │    │
│   └───────────────────────────────────┘    │
└────────────────────────────────────────────┘
```

---

# 26. Agent can thiệp như thế nào?

JVM cung cấp `java.lang.instrument` cho phép một agent **sửa bytecode khi class được load**. OpenTelemetry Java Agent dùng thư viện ByteBuddy để chèn "advice" vào các method đã biết của framework.

Code framework về logic:

```java
handleRequest(request);
```

Sau khi instrument, hành vi runtime tương đương:

```text
extract context từ headers
   ↓
start SERVER span, đặt làm current
   ↓
handleRequest(request)
   ↓
ghi http.response.status_code, set status nếu lỗi
   ↓
end span, khôi phục context
```

Chi tiết kỹ thuật đáng biết:

- **Classloader isolation:** SDK và dependency của Agent được nạp trong classloader riêng và được shade, để không xung đột version với dependency của ứng dụng.
- **API bridging:** khi ứng dụng có `opentelemetry-api` riêng, Agent "nối" API đó tới SDK bên trong Agent. Nhờ vậy span tạo bằng tay và span tự động dùng chung một runtime, chung Context.
- **Muzzle:** trước khi áp dụng một instrumentation, Agent kiểm tra library version có khớp không. Nếu không khớp, instrumentation đó bị bỏ qua thay vì làm crash ứng dụng.

---

# 27. Startup flow

```text
JVM START
   │
   ├── premain(): load Java Agent
   │
   ▼
Agent initialization
   │
   ├── đọc config (system properties, env vars, config file)
   ├── autoconfigure SDK
   │     ├── Resource (+ resource detectors)
   │     ├── Sampler
   │     ├── Propagators
   │     ├── SpanProcessor / MetricReader / LogRecordProcessor
   │     └── Exporters
   ├── đăng ký GlobalOpenTelemetry
   ├── load extensions (nếu có)
   └── đăng ký instrumentation modules
   │
   ▼
Application classes load
   │
   ▼
Agent transform các class được hỗ trợ
   │
   ▼
Spring Boot starts
```

Agent không chỉ là "một thư viện tạo Span". Nó là một **zero-code instrumentation runtime** hoàn chỉnh.

Chi phí: startup chậm hơn (thường vài giây với ứng dụng lớn) do transform bytecode.

---

# 28. Agent tự tạo Span ở đâu?

Agent hữu ích nhất ở các **boundary**:

```text
incoming HTTP / gRPC          → SERVER span
outgoing HTTP / gRPC          → CLIENT span
database (JDBC, Mongo, Redis) → CLIENT span
messaging                     → PRODUCER / CONSUMER span
scheduled jobs                → INTERNAL span (tùy framework)
```

Ví dụ `GET /orders/1` qua Tomcat → Spring MVC → JdbcTemplate → PostgreSQL:

```text
Trace ABC
GET /orders/{id}             SERVER
└── SELECT orders            CLIENT   db.system.name=postgresql
```

Ngoài span, Agent còn tự sinh:

- **Metrics:** `http.server.request.duration`, `http.client.request.duration`, `db.client.*`, JVM metrics (`jvm.memory.used`, `jvm.gc.duration`, `jvm.thread.count`, …).
- **Logs:** bắt log từ Logback/Log4j/JUL và export qua OTLP.
- **MDC injection:** chèn `trace_id`, `span_id`, `trace_flags` vào MDC của logging framework.

Agent **không biết** business meaning như `reserve-inventory` hay `check-credit-limit`. Đó là việc của manual instrumentation.

---

# 29. Cấu hình Agent

Ba cách, độ ưu tiên từ cao xuống thấp (tham khảo):

```text
1. System properties     -Dotel.service.name=order-service
2. Environment variables OTEL_SERVICE_NAME=order-service
3. Config file           -Dotel.javaagent.configuration-file=/opt/otel/otel.properties
```

Quy tắc chuyển tên: `otel.exporter.otlp.endpoint` ↔ `OTEL_EXPORTER_OTLP_ENDPOINT`.

Cấu hình tối thiểu nên có:

```bash
OTEL_SERVICE_NAME=order-service
OTEL_RESOURCE_ATTRIBUTES=service.version=1.4.0,deployment.environment.name=production
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
```

Bật/tắt từng instrumentation:

```bash
# tắt một instrumentation cụ thể
OTEL_INSTRUMENTATION_JDBC_ENABLED=false

# chiến lược "opt-in": tắt hết, chỉ bật cái cần
OTEL_INSTRUMENTATION_COMMON_DEFAULT_ENABLED=false
OTEL_INSTRUMENTATION_SPRING_WEBMVC_ENABLED=true
OTEL_INSTRUMENTATION_JDBC_ENABLED=true
```

Debug:

```bash
OTEL_JAVAAGENT_DEBUG=true            # log rất nhiều, chỉ dùng khi debug
OTEL_TRACES_EXPORTER=logging,otlp    # in span ra console đồng thời export
```

Tắt toàn bộ SDK mà vẫn giữ `-javaagent` (hữu ích khi cần rollback nhanh):

```bash
OTEL_SDK_DISABLED=true
```

Bảng đầy đủ ở Phần XVI.

---

# 30. Mở rộng Agent

Khi instrumentation có sẵn chưa đủ, có ba mức can thiệp, từ nhẹ đến nặng:

| Cách | Khi nào | Ví dụ |
|---|---|---|
| Annotation `@WithSpan` | Muốn một method thành span mà không viết code tracer | `@WithSpan("reserve-inventory")` |
| Manual API | Cần attributes, events, status chi tiết | Xem Phần V |
| Agent Extension (jar riêng) | Tùy biến SDK: custom sampler, SpanProcessor, ResourceProvider, hoặc instrument library nội bộ | `-Dotel.javaagent.extensions=/opt/otel/ext.jar` |

`@WithSpan` cần dependency `io.opentelemetry.instrumentation:opentelemetry-instrumentation-annotations`:

```java
@WithSpan("reserve-inventory")
public Reservation reserve(@SpanAttribute("order.id") String orderId) {
    ...
}
```

Lưu ý: `@WithSpan` hoạt động qua bytecode instrumentation, nên **có hiệu lực cả với self-invocation** (khác Spring AOP proxy). Tuy vậy, đừng rải nó lên mọi method (xem mục 13 và Phần XVIII).

---

# 31. Overhead của Agent

| Loại chi phí | Mức độ thường gặp | Giảm bằng cách |
|---|---|---|
| Startup time | Tăng vài giây | Tắt instrumentation không dùng |
| CPU | Vài phần trăm, tăng theo số span | Sampling, giảm span không cần thiết |
| Memory | Tăng heap do queue span, metrics state | Giới hạn queue, kiểm soát cardinality |
| Network | Tỉ lệ với lượng telemetry | Sampling, compression, batching |

Luôn **đo trên workload thật** trước khi bật cho toàn bộ production. Bắt đầu với một service, so sánh latency p99 và CPU trước/sau.

---

# PHẦN V — OPENTELEMETRY API

# 32. API là gì?

```gradle
implementation(platform("io.opentelemetry:opentelemetry-bom:<version>"))
implementation("io.opentelemetry:opentelemetry-api")
```

API cung cấp abstraction để application/library nói:

```text
Tôi muốn tạo Span
Tôi muốn ghi Metric
Tôi muốn đọc/đặt current Context
Tôi muốn inject/extract Context
```

API **không** quyết định:

```text
có sample hay không
batch thế nào
gửi đi đâu, endpoint nào, protocol nào
gắn Resource gì
```

Các entry point:

```text
OpenTelemetry
├── TracerProvider → Tracer → SpanBuilder → Span
├── MeterProvider  → Meter  → Instruments (Counter, Histogram, ...)
├── LoggerProvider → Logger → LogRecord   (dành cho appender/bridge)
└── ContextPropagators → TextMapPropagator
```

---

# 33. Tại sao API phải tách khỏi SDK?

Giả sử shared library `payment-client.jar` muốn tạo telemetry.

Nếu nó phụ thuộc cứng vào Jaeger, Datadog, hay một SDK configuration cụ thể, library bị **coupling với deployment environment** của người dùng nó.

Thiết kế đúng:

```text
Library ──► OpenTelemetry API  (chỉ interface, rất nhẹ)

Application (người dùng cuối) ──► chọn SDK, sampler, exporter, backend
```

```text
API = "tôi muốn tạo telemetry"
SDK = "tôi sẽ thực hiện việc đó như thế nào"
```

---

# 34. API không có SDK: no-op

Nếu không có SDK nào được cài, API hoạt động ở chế độ **no-op**:

```text
library gọi tracer.spanBuilder(...).startSpan()
      ↓
không có SDK thực
      ↓
trả về span no-op, mọi method không làm gì
      ↓
không crash, overhead gần bằng 0
```

Lưu ý tinh tế: ngay cả ở no-op, **context propagation vẫn có thể hoạt động** nếu một span context hợp lệ đã có sẵn (ví dụ được extract từ header). Đây là thiết kế để không "làm gãy" trace đi qua một service chưa bật SDK.

---

# 35. GlobalOpenTelemetry khi dùng Java Agent

Agent cấu hình SDK và đăng ký vào `GlobalOpenTelemetry`. Business code lấy ra để dùng:

```java
OpenTelemetry otel = GlobalOpenTelemetry.get();
Tracer tracer = otel.getTracer("com.acme.order", "1.4.0");
Meter  meter  = otel.getMeter("com.acme.order");
```

```text
Java Agent
   ├── initialize/configure SDK
   └── register GlobalOpenTelemetry
                  │
                  ▼
Business code dùng OTel API
                  │
                  ▼
        cùng một OpenTelemetry runtime, cùng Context
```

Quy tắc:

- **Không tạo thêm một SDK thứ hai** (`OpenTelemetrySdk.builder()...`) khi Agent đã quản lý SDK. Hai SDK = hai pipeline export, Resource có thể khác nhau, và dễ có span trùng hoặc lạc.
- Trong Spring, nên expose `OpenTelemetry` thành bean (lấy từ `GlobalOpenTelemetry.get()`) và inject vào chỗ cần, để dễ test.
- Version của `opentelemetry-api` trong app không nên **mới hơn** version API mà Agent hỗ trợ. Dùng BOM khớp version Agent.

---

# 36. Manual instrumentation: ví dụ đầy đủ

```java
@Service
public class PricingService {

    private final Tracer tracer;
    private final LongCounter discountApplied;
    private final DoubleHistogram priceCalcDuration;

    public PricingService(OpenTelemetry otel) {
        this.tracer = otel.getTracer("com.acme.pricing");
        Meter meter = otel.getMeter("com.acme.pricing");

        this.discountApplied = meter.counterBuilder("pricing.discount.applied")
                .setDescription("Số lần áp dụng discount")
                .setUnit("{discount}")
                .build();

        this.priceCalcDuration = meter.histogramBuilder("pricing.calculation.duration")
                .setUnit("s")
                .build();
    }

    public Price calculate(Order order) {
        long start = System.nanoTime();
        Span span = tracer.spanBuilder("calculate-order-price")
                .setSpanKind(SpanKind.INTERNAL)                 // mặc định đã là INTERNAL
                .setAttribute("order.id", order.id())
                .setAttribute("order.item_count", order.items().size())
                .startSpan();

        try (Scope ignored = span.makeCurrent()) {
            Price base = sumItems(order);                     // span con (nếu có) tự nhận parent
            span.addEvent("base-price-calculated",
                    Attributes.of(AttributeKey.doubleKey("price.base"), base.amount()));

            Price result = applyDiscount(order, base);
            if (result.hasDiscount()) {
                discountApplied.add(1, Attributes.of(
                        AttributeKey.stringKey("discount.type"), result.discountType()));
            }
            span.setAttribute("price.final", result.amount());
            return result;

        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getClass().getSimpleName());
            throw e;
        } finally {
            span.end();
            priceCalcDuration.record((System.nanoTime() - start) / 1e9);
        }
    }
}
```

Checklist cho mỗi manual span:

- [ ] Tên low-cardinality
- [ ] `makeCurrent()` trong try-with-resources
- [ ] `end()` trong `finally`
- [ ] Ghi exception **và** set status ERROR
- [ ] Attribute nghiệp vụ có namespace, không nhạy cảm
- [ ] Không tạo instrument (Counter, Histogram) mỗi lần gọi method; tạo một lần và tái sử dụng

---

# 37. Lấy trace_id trong code

Một số trường hợp cần trace_id: trả về cho client trong response header để support team tra cứu, hoặc gắn vào error response.

```java
SpanContext ctx = Span.current().getSpanContext();
if (ctx.isValid()) {
    response.setHeader("X-Trace-Id", ctx.getTraceId());
}
```

Lưu ý: đây là **trả trace_id ra cho người đọc**, khác với việc **tự propagate** bằng custom header (anti-pattern, xem Phần XVIII).

---

# PHẦN VI — TRACE SDK

# 38. SDK là gì?

SDK là **implementation thực sự** của API. Trách nhiệm:

```text
tạo telemetry objects thật
sampling
processing (enrich, filter)
batching
aggregation (metrics)
gắn Resource
export
lifecycle: flush, shutdown
```

```text
Application / Instrumentation
          │
          ▼
        OTel API      (interface)
          │
          ▼
        OTel SDK      (implementation)
          │
          ▼
       Exporter
```

---

# 39. SDK chia theo signal

```text
OpenTelemetrySdk
│
├── SdkTracerProvider     → Traces
│     ├── Resource
│     ├── Sampler
│     ├── SpanLimits
│     ├── IdGenerator
│     └── SpanProcessor(s) → SpanExporter
│
├── SdkMeterProvider      → Metrics
│     ├── Resource
│     ├── Views
│     └── MetricReader(s)  → MetricExporter
│
├── SdkLoggerProvider     → Logs
│     ├── Resource
│     └── LogRecordProcessor(s) → LogRecordExporter
│
└── ContextPropagators
```

Ba provider dùng **chung một Resource**, nên traces, metrics và logs từ cùng một service luôn có cùng `service.name`. Đây là nền tảng cho correlation.

---

# 40. Vòng đời của một Span trong SDK

```text
tracer.spanBuilder("x")
      │
      ▼
startSpan()
      │
      ├── IdGenerator tạo span_id (và trace_id nếu là root)
      ├── Sampler.shouldSample(parent ctx, trace_id, name, kind, attributes, links)
      │       ├── DROP              → span không record, không export (nhưng vẫn propagate context)
      │       ├── RECORD_ONLY       → record nhưng không export, flag sampled=0
      │       └── RECORD_AND_SAMPLE → record và export, flag sampled=1
      │
      ├── SpanProcessor.onStart(span)     ← có thể enrich attribute ở đây
      ▼
span đang chạy: setAttribute, addEvent, recordException, setStatus
      │
      ▼
span.end()
      │
      ├── SpanProcessor.onEnd(span)
      │       └── BatchSpanProcessor: đưa vào queue
      ▼
Exporter.export(batch)
```

Điểm quan trọng: **sampling quyết định lúc span bắt đầu**, chỉ dựa trên thông tin có tại thời điểm đó. Span chưa biết nó sẽ lỗi hay chậm. Đây là giới hạn cơ bản của head sampling (Phần VII).

---

# 41. TracerProvider và Tracer

```text
TracerProvider (một instance cho cả process)
      │
      ├── Tracer "io.opentelemetry.tomcat-10.0"
      ├── Tracer "io.opentelemetry.jdbc"
      └── Tracer "com.acme.pricing"
              │
              ▼
          SpanBuilder → Span
```

- `TracerProvider` giữ toàn bộ cấu hình: Resource, Sampler, Processors.
- `Tracer` gắn với một **Instrumentation Scope** (tên + version của instrumentation), **không phải** tên service.

```text
Tên Tracer   = ai tạo telemetry     → "com.acme.pricing"
service.name = service nào đang chạy → Resource: service.name=order-service
```

---

# 42. SpanProcessor

SpanProcessor nằm giữa vòng đời span và exporter. Hook: `onStart`, `onEnd`, `forceFlush`, `shutdown`.

**SimpleSpanProcessor:**

```text
span.end() → export ngay, đồng bộ
```

Chỉ dùng cho test/debug. Mỗi span một network call, chặn thread ứng dụng.

**BatchSpanProcessor:**

```text
span.end() → queue (bounded) → worker thread gom batch → export
```

| Tham số | Env var | Mặc định (tham khảo) | Ý nghĩa |
|---|---|---|---|
| Schedule delay | `OTEL_BSP_SCHEDULE_DELAY` | 5000 ms | Chu kỳ export |
| Max queue size | `OTEL_BSP_MAX_QUEUE_SIZE` | 2048 | Queue đầy → **span bị drop** |
| Max export batch | `OTEL_BSP_MAX_EXPORT_BATCH_SIZE` | 512 | Số span mỗi lần export |
| Export timeout | `OTEL_BSP_EXPORT_TIMEOUT` | 30000 ms | Timeout mỗi lần export |

Hệ quả vận hành:

- Collector chậm hoặc chết → queue đầy → span bị drop **một cách im lặng** (không ảnh hưởng request, nhưng mất telemetry). Theo dõi metric nội bộ về số span bị drop nếu SDK expose.
- Ứng dụng tắt đột ngột → span trong queue bị mất. SDK đăng ký shutdown hook để flush, nhưng `kill -9` thì không flush được.

**Custom SpanProcessor** hay dùng để:

- Thêm attribute chung vào mọi span (ví dụ copy Baggage `tenant.id` sang span attribute).
- Lọc bỏ span không cần (ví dụ health check), dù việc này thường làm ở sampler hoặc Collector thì tốt hơn.

---

# 43. SpanExporter

Nhận batch span đã kết thúc và gửi đi.

| Exporter | Dùng khi |
|---|---|
| OTLP (gRPC / HTTP) | **Mặc định cho production** |
| Logging / console | Debug local |
| Zipkin | Hệ thống cũ dùng Zipkin |
| In-memory | Unit test |

```text
Span ─► SpanProcessor ─► OTLP SpanExporter ─► Collector
```

---

# 44. Span Limits

SDK giới hạn để một span "hỏng" không làm nổ memory/network:

| Giới hạn | Env var | Mặc định (tham khảo) |
|---|---|---|
| Số attribute mỗi span | `OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT` | 128 |
| Độ dài giá trị attribute | `OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT` | không giới hạn |
| Số event mỗi span | `OTEL_SPAN_EVENT_COUNT_LIMIT` | 128 |
| Số link mỗi span | `OTEL_SPAN_LINK_COUNT_LIMIT` | 128 |

Nên đặt `OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT` (ví dụ 4096) trong production để tránh SQL dài hoặc stacktrace khổng lồ làm phình payload.

---

# PHẦN VII — SAMPLING

# 45. Tại sao phải sample?

Hệ thống 5.000 request/giây, mỗi request ~20 span:

```text
5.000 × 20 = 100.000 span/giây
           ≈ 8,6 tỉ span/ngày
```

Lưu hết là quá đắt và phần lớn vô ích: 99% trace là request thành công, nhanh, giống hệt nhau.

Sampling kiểm soát:

```text
CPU và memory của ứng dụng
băng thông network
dung lượng lưu trữ backend
chi phí (đặc biệt với vendor tính tiền theo GB/span)
```

Nguyên tắc cốt lõi: **sample theo trace, không theo span**. Một trace thiếu một nửa số span gần như vô dụng. Mọi service trong chuỗi phải đồng ý cùng một quyết định.

---

# 46. Head sampling vs Tail sampling

```text
HEAD SAMPLING (trong SDK)
─────────────────────────
Request bắt đầu → quyết định NGAY → truyền quyết định qua flag sampled

  + rẻ: span không được sample không tốn chi phí xử lý/export
  + đơn giản, không cần hạ tầng thêm
  − quyết định "mù": chưa biết request có lỗi hay chậm không

TAIL SAMPLING (trong Collector)
───────────────────────────────
Mọi span được export → Collector giữ cả trace trong memory → chờ trace hoàn tất → quyết định

  + thông minh: giữ 100% trace lỗi, trace chậm, chỉ giữ 5% trace bình thường
  − đắt: ứng dụng phải export 100% span
  − phức tạp: cần tất cả span của một trace về cùng một Collector instance
  − tốn memory, có độ trễ (decision_wait)
```

Thực tế production thường **kết hợp**:

```text
SDK: head sampling cao (ví dụ 100% hoặc 50%)
  ↓
Collector: tail sampling giữ lỗi/chậm + một phần nhỏ trace bình thường
```

---

# 47. Các Sampler trong SDK

| Sampler | Hành vi |
|---|---|
| `always_on` | Sample mọi thứ |
| `always_off` | Không sample gì |
| `traceidratio` | Sample theo tỉ lệ dựa trên trace_id (deterministic) |
| `parentbased_always_on` | **Mặc định.** Theo parent; nếu là root thì always_on |
| `parentbased_traceidratio` | Theo parent; nếu là root thì traceidratio |
| `parentbased_always_off` | Theo parent; nếu là root thì không sample |

Cấu hình:

```bash
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1     # 10% root traces
```

---

# 48. ParentBased: sampler bạn sẽ dùng nhiều nhất

`ParentBased` thực chất là một **bộ định tuyến quyết định**:

| Tình huống của span | Quyết định |
|---|---|
| Không có parent (root) | Dùng root sampler (ví dụ ratio 10%) |
| Remote parent, sampled=1 | Sample |
| Remote parent, sampled=0 | Không sample |
| Local parent, sampled=1 | Sample |
| Local parent, sampled=0 | Không sample |

Hệ quả:

```text
API Gateway (root, ratio 10%) ── quyết định
    │ traceparent ...-01 hoặc ...-00
    ▼
Order Service (ParentBased) ── tuân theo
    ▼
Payment Service (ParentBased) ── tuân theo
```

Chỉ **service ở đầu chuỗi** thực sự quyết định. Các service sau tuân theo, nhờ vậy trace luôn đầy đủ hoặc không có gì.

Cạm bẫy: nếu một service downstream dùng `traceidratio` **không có** ParentBased, nó tự quyết định lại → trace bị thủng lỗ.

Tại sao ratio dựa trên `trace_id`: mọi service tính trên cùng trace_id cho ra cùng kết quả, nên kể cả khi không có ParentBased, các service cùng tỉ lệ vẫn nhất quán. Nhưng tỉ lệ khác nhau thì không.

---

# 49. Tail sampling trong Collector

Processor `tail_sampling` (Collector contrib):

```yaml
processors:
  tail_sampling:
    decision_wait: 10s          # chờ bao lâu để coi trace là "xong"
    num_traces: 100000          # số trace giữ trong memory
    expected_new_traces_per_sec: 2000
    policies:
      - name: keep-errors
        type: status_code
        status_code: { status_codes: [ERROR] }

      - name: keep-slow
        type: latency
        latency: { threshold_ms: 1000 }

      - name: keep-vip-tenant
        type: string_attribute
        string_attribute: { key: tenant.tier, values: [enterprise] }

      - name: baseline
        type: probabilistic
        probabilistic: { sampling_percentage: 5 }
```

Policies được đánh giá theo nguyên tắc: **nếu bất kỳ policy nào nói "giữ" thì trace được giữ** (trừ khi dùng các kiểu policy kết hợp `and`, `composite`, hoặc policy drop tường minh).

**Ràng buộc kiến trúc quan trọng nhất:**

```text
Tail sampling cần TẤT CẢ span của một trace về CÙNG MỘT Collector instance.

Sai:
App A ──► Collector #1  (thấy span 1,2)
App B ──► Collector #2  (thấy span 3)   → mỗi bên quyết định trên trace không đầy đủ

Đúng: hai tầng
App A ─┐                                    ┌─► Collector tầng 2 #1 (tail_sampling)
App B ─┼─► Collector tầng 1 (loadbalancing ─┤
App C ─┘    exporter, routing_key=traceID)  └─► Collector tầng 2 #2 (tail_sampling)
```

`loadbalancing` exporter hash theo trace_id để mọi span cùng trace đi về cùng instance tầng 2.

---

# 50. Sampling ảnh hưởng tới metrics như thế nào?

Nếu bạn tính "request rate" bằng cách đếm span **sau khi** sample 10%, con số sai lệch 10 lần.

Cách đúng:

- Metrics HTTP do Agent sinh (`http.server.request.duration`) được ghi **độc lập với sampling** → luôn chính xác. Dùng chúng cho dashboard và alert.
- Nếu sinh metrics từ span bằng `spanmetrics` connector, đặt connector **trước** tail sampling trong Collector, để nó thấy 100% span.

```text
Receiver ─► [spanmetrics connector] ─► metrics pipeline   (thấy 100%)
         └► tail_sampling ─► traces exporter            (chỉ giữ một phần)
```

---

# PHẦN VIII — METRICS

# 51. Metrics khác tracing ở đâu?

| | Tracing | Metrics |
|---|---|---|
| Đơn vị | Một request cụ thể | Tổng hợp của nhiều measurements |
| Kích thước dữ liệu | Tỉ lệ với số request | Tỉ lệ với **số tổ hợp attribute** (cardinality) |
| Sampling | Thường cần | Không sample, aggregate |
| Câu hỏi | "Request này chậm ở đâu?" | "p99 latency 5 phút qua là bao nhiêu?" |
| Alerting | Không phù hợp | **Nền tảng của alerting** |

---

# 52. Pipeline Metrics SDK

```text
Instrumentation
      │  counter.add(1, {http.route=/orders})
      ▼
Meter → Instrument
      │
      ▼
View (tùy chọn: đổi tên, đổi aggregation, lọc attribute)
      │
      ▼
Aggregation trong memory  (Sum / LastValue / Histogram)
      │
      ▼
MetricReader  (định kỳ hoặc khi bị scrape, đọc trạng thái aggregate)
      │
      ▼
MetricExporter  (OTLP push / Prometheus pull)
```

Khác với span (export khi `end()`), metrics **không export từng measurement**. SDK giữ trạng thái aggregate trong memory và MetricReader đọc định kỳ.

---

# 53. Các loại Instrument

| Instrument | Đồng bộ? | Monotonic? | Aggregation mặc định | Ví dụ |
|---|---|---|---|---|
| `Counter` | Sync | Chỉ tăng | Sum | Số request, số đơn hàng tạo |
| `UpDownCounter` | Sync | Tăng/giảm | Sum | Số request đang xử lý, item trong queue |
| `Histogram` | Sync | — | Histogram | Latency, kích thước payload |
| `Gauge` | Sync (bản API mới) | — | LastValue | Giá trị đo được tại thời điểm ghi |
| `ObservableCounter` | Async (callback) | Chỉ tăng | Sum | CPU time tích lũy |
| `ObservableUpDownCounter` | Async | Tăng/giảm | Sum | Số connection trong pool |
| `ObservableGauge` | Async | — | LastValue | Nhiệt độ, memory đang dùng |

**Sync vs Async:**

```text
Sync:  code gọi counter.add() ngay khi sự kiện xảy ra
Async: SDK gọi callback của bạn mỗi lần MetricReader collect
```

Dùng async khi giá trị **đã tồn tại sẵn ở đâu đó** và chỉ cần đọc (kích thước pool, queue depth):

```java
meter.gaugeBuilder("orders.queue.depth")
     .ofLongs()
     .buildWithCallback(m -> m.record(queue.size()));
```

Chọn instrument nhanh:

```text
Đếm sự kiện chỉ tăng?                    → Counter
Giá trị lên xuống, cộng dồn có nghĩa?    → UpDownCounter
Cần phân phối (p50, p95, p99)?            → Histogram
Giá trị tức thời, cộng dồn vô nghĩa?      → Gauge
```

Ví dụ: số connection trong pool là UpDownCounter (tổng các pod có ý nghĩa). Nhiệt độ CPU là Gauge (cộng nhiệt độ các pod vô nghĩa).

---

# 54. Histogram và percentile

Histogram không lưu từng giá trị, mà đếm số giá trị rơi vào từng **bucket**:

```text
Bucket (giây):  ≤0.005  ≤0.01  ≤0.025  ≤0.05  ≤0.1  ≤0.25  ≤0.5  ≤1  ≤2.5  ≤5  ≤10  +Inf
Count:            120    340     890    1200   430    80     20    5    1    0    0    0
```

Percentile (p95, p99) được **ước lượng** ở backend từ các bucket. Độ chính xác phụ thuộc boundaries.

Hai loại:

| Loại | Đặc điểm |
|---|---|
| Explicit bucket histogram | Bạn (hoặc semconv) định nghĩa boundaries. Mặc định phổ biến |
| Exponential histogram | Bucket tự điều chỉnh theo scale, chính xác hơn trên dải giá trị rộng; backend phải hỗ trợ (Prometheus native histograms) |

Semantic conventions khuyến nghị đơn vị **giây** (`s`) cho duration, không phải millisecond.

---

# 55. Aggregation Temporality: Cumulative vs Delta

Đây là khái niệm gây lỗi nhiều nhất khi nối OTel với backend.

```text
Số request thực tế mỗi phút:   10    15    8

CUMULATIVE (tổng từ lúc start):
  export lần 1: 10
  export lần 2: 25
  export lần 3: 33

DELTA (chỉ phần mới kể từ lần export trước):
  export lần 1: 10
  export lần 2: 15
  export lần 3: 8
```

| Backend | Kỳ vọng |
|---|---|
| Prometheus | **Cumulative** |
| Một số vendor (Datadog, Dynatrace, …) | Thường ưa Delta |

```bash
OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=cumulative   # mặc định
# hoặc: delta, lowmemory
```

Gửi sai temporality → graph nhảy lung tung, `rate()` cho kết quả vô nghĩa. Collector có processor chuyển delta → cumulative (`deltatocumulative`) khi cần.

---

# 56. Cardinality: kẻ giết backend metrics

Mỗi **tổ hợp duy nhất** của (metric name + attributes) là một **time series** riêng.

```text
http.server.request.duration
  attributes: http.request.method (5 giá trị)
            × http.route (50 giá trị)
            × http.response.status_code (10 giá trị)
  = 2.500 series × số pod
  → chấp nhận được

Thêm user.id (1.000.000 giá trị):
  = 2.500.000.000 series
  → backend sập, hóa đơn bùng nổ, SDK tốn memory
```

Quy tắc:

| Không bao giờ làm metric attribute | Có thể làm metric attribute |
|---|---|
| user_id, order_id, request_id, email | http.route (template), method, status code |
| URL đầy đủ có query string | tenant tier (vài giá trị) |
| trace_id | region, availability zone |
| Timestamp, giá trị liên tục | Loại lỗi (enum ngắn) |

Giá trị high-cardinality thuộc về **span attributes** hoặc **logs**, không phải metrics.

SDK có **cardinality limit** mỗi metric (mặc định khoảng 2000 tổ hợp, tham khảo); vượt quá thì gộp vào một series "overflow". Nếu thấy attribute `otel.metric.overflow=true` ở backend, đó là tín hiệu bạn có vấn đề cardinality.

---

# 57. Views

View cho phép **người vận hành ứng dụng** (không phải tác giả instrumentation) thay đổi cách metric được aggregate.

Dùng để:

- Đổi bucket boundaries của histogram.
- **Loại bỏ attribute** để giảm cardinality.
- Đổi tên metric.
- Drop hẳn một metric không cần.

```java
SdkMeterProvider.builder()
    .registerView(
        InstrumentSelector.builder().setName("http.server.request.duration").build(),
        View.builder()
            .setAggregation(Aggregation.explicitBucketHistogram(
                List.of(0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0)))
            .setAttributeFilter(Set.of("http.request.method", "http.route",
                                        "http.response.status_code"))
            .build())
    .build();
```

Khi dùng Java Agent, Views được cấu hình qua extension hoặc các tùy chọn của Agent (tùy version). Cũng có thể xử lý ở Collector (`filter`, `transform`, `metricstransform`), nhưng giảm cardinality càng sớm càng rẻ.

---

# 58. MetricReader: push vs pull

```text
PUSH  (PeriodicMetricReader + OTLP exporter)
  SDK ──mỗi 60s──► Collector / backend
  OTEL_METRIC_EXPORT_INTERVAL=60000

PULL  (Prometheus exporter trong SDK)
  SDK mở HTTP endpoint /metrics ◄──scrape── Prometheus
```

| | Push (OTLP) | Pull (Prometheus) |
|---|---|---|
| Đi qua Collector | Có, dễ xử lý tập trung | Collector có thể scrape thay Prometheus |
| Service discovery | Không cần | Cần (k8s SD, …) |
| Short-lived job | Phù hợp | Có thể mất dữ liệu nếu job kết thúc trước khi bị scrape |
| Thống nhất với traces/logs | Cùng pipeline OTLP | Pipeline riêng |

Khuyến nghị cho hệ thống mới: **push OTLP qua Collector**, để cả ba signal đi chung đường.

---

# 59. Exemplars: nối metric với trace

Exemplar là **một mẫu đo cụ thể** được gắn vào điểm dữ liệu metric, kèm `trace_id` và `span_id` của request tạo ra nó.

```text
Histogram http.server.request.duration, bucket ≤ 2.5s
   └── exemplar: value=2.31s, trace_id=4bf92f..., span_id=00f067...
```

Trong Grafana: nhìn thấy spike p99 trên biểu đồ → click vào chấm exemplar → mở thẳng trace chậm đó trong Tempo.

```bash
OTEL_METRICS_EXEMPLAR_FILTER=trace_based   # mặc định: chỉ lấy exemplar từ request được sample
```

Yêu cầu: backend metrics phải bật lưu exemplar (Prometheus cần bật tính năng exemplar storage), và Grafana datasource phải cấu hình liên kết tới datasource traces.

---

# 60. Đặt tên metric

Theo semantic conventions:

- Dạng `namespace.thing.measure`, dùng dấu chấm: `http.server.request.duration`, `orders.created`.
- Đơn vị đặt trong field `unit` (UCUM: `s`, `By`, `{request}`), **không** nhét vào tên (tránh `_ms`, `_seconds`).
- Không thêm `_total` vào tên. Backend như Prometheus tự thêm hậu tố khi chuyển đổi.

Khi vào Prometheus, tên thường được chuyển đổi (tùy cấu hình và version):

```text
OTel:        http.server.request.duration   unit=s   (histogram)
Prometheus:  http_server_request_duration_seconds_bucket / _sum / _count

OTel:        orders.created                 (counter)
Prometheus:  orders_created_total
```

Prometheus 3.x hỗ trợ UTF-8 trong tên metric nên có thể giữ nguyên tên có dấu chấm, tùy chiến lược translation bạn chọn. Thống nhất điều này **trước khi** viết dashboard.

---

# PHẦN IX — LOGS

# 61. Cách OpenTelemetry tiếp cận Logs

Khác với traces và metrics, OTel **không** yêu cầu bạn viết log bằng API mới. Bạn tiếp tục dùng SLF4J/Logback/Log4j. OTel cung cấp **Logs Bridge API** để nối logging framework hiện có vào pipeline OTel.

```text
log.info("Order created")          ← code giữ nguyên
      │
      ▼
Logback / Log4j
      │
      ▼
OpenTelemetry Appender (bridge)    ← Agent tự cài, hoặc thêm dependency
      │
      ▼
Logger (Logs Bridge API)
      │
      ▼
LogRecord  + tự gắn trace_id/span_id từ current Context
      │
      ▼
LogRecordProcessor (Batch)
      │
      ▼
LogRecordExporter (OTLP)
```

---

# 62. Cấu trúc LogRecord

```text
LogRecord
├── timestamp / observed_timestamp
├── severity_number   (1–24, chuẩn hóa)  severity_text = "INFO"
├── body              = "Order created"
├── attributes        = { order.id=..., code.function=..., thread.name=... }
├── trace_id          ← từ Context lúc ghi log
├── span_id
├── trace_flags
├── resource          = service.name=order-service, ...
└── instrumentation_scope = tên logger
```

Nhờ `trace_id` gắn sẵn, từ một trace trong Tempo có thể nhảy sang **chính xác** các log của request đó trong Loki, và ngược lại.

---

# 63. Hai chiến lược thu thập log

| Chiến lược | Cách | Ưu | Nhược |
|---|---|---|---|
| **A. OTLP trực tiếp** | Appender → SDK → OTLP → Collector | Có cấu trúc sẵn, trace_id chính xác, không cần parse | Log chỉ đi qua network; Collector chết thì log có thể mất |
| **B. File/stdout + Collector đọc** | App ghi stdout (JSON) → Collector `filelog` receiver đọc file container | Bền: log vẫn nằm trên disk/stdout; quen thuộc với k8s | Phải parse; trace_id cần có trong log line (MDC) |

Với chiến lược B, đảm bảo log pattern có trace_id từ MDC:

```xml
<pattern>%d %-5level [%thread] %logger trace_id=%X{trace_id} span_id=%X{span_id} - %msg%n</pattern>
```

Hoặc tốt hơn: ghi log dạng **JSON** có trường `trace_id`, `span_id`.

Nhiều tổ chức dùng cả hai: stdout cho độ bền và tương thích, OTLP cho log có cấu trúc. Tránh gửi **trùng** cùng một log qua hai đường vào cùng backend.

---

# 64. Log, Span Event hay Metric?

| Tình huống | Dùng |
|---|---|
| Cần đếm, vẽ biểu đồ, alert | Metric |
| Mốc thời gian trong một operation, chỉ có ý nghĩa trong trace | Span Event |
| Chi tiết debug, message dài, sự kiện ngoài request (startup, job) | Log |
| Exception | `recordException` trên span **và** log ERROR (log thường được giữ lâu hơn và không bị sample) |

Lưu ý: traces thường bị sample, **logs thường không**. Thông tin bắt buộc phải lưu (audit, lỗi nghiêm trọng) không nên chỉ nằm trong span.

---

# PHẦN X — RESOURCE, INSTRUMENTATION SCOPE, SEMANTIC CONVENTIONS

# 65. Resource

Resource mô tả **entity sinh ra telemetry**. Được gắn một lần ở cấp provider và áp dụng cho mọi span, metric, log của process.

```text
service.name                 = order-service        ← BẮT BUỘC đặt
service.version              = 1.4.0
service.namespace            = marketplace
service.instance.id          = 7f3c...              ← phân biệt các replica
deployment.environment.name  = production
host.name                    = node-01
process.pid                  = 123
process.runtime.name         = OpenJDK Runtime Environment
container.id                 = 3fa9...
k8s.namespace.name           = marketplace
k8s.pod.name                 = order-service-7d9f-abcd
k8s.deployment.name          = order-service
cloud.provider               = aws
cloud.region                 = ap-southeast-1
```

Nếu không đặt `service.name`, SDK dùng `unknown_service:java`. Mọi service hiện lên với cùng tên này → trace và dashboard vô nghĩa. Đây là lỗi cấu hình số một.

`service.version` nên luôn có: nó cho phép so sánh latency/lỗi **trước và sau deploy**.

---

# 66. Resource Detectors

Không phải attribute nào cũng cần hard-code. Resource Detectors tự phát hiện thông tin môi trường:

```text
runtime environment
      ↓
Resource Detectors (process, host, OS, container, cloud, k8s ...)
      ↓
Resource attributes
      ↓
merge với OTEL_RESOURCE_ATTRIBUTES, OTEL_SERVICE_NAME
```

Trong Kubernetes, thông tin pod/namespace thường được bổ sung theo hai cách:

1. Truyền qua env var bằng Downward API:

```yaml
env:
  - name: K8S_POD_NAME
    valueFrom: { fieldRef: { fieldPath: metadata.name } }
  - name: K8S_NAMESPACE
    valueFrom: { fieldRef: { fieldPath: metadata.namespace } }
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "k8s.pod.name=$(K8S_POD_NAME),k8s.namespace.name=$(K8S_NAMESPACE)"
```

2. Để Collector làm bằng processor `k8sattributes`: tra cứu Kubernetes API theo IP của pod gửi telemetry, gắn thêm `k8s.*` attributes. Cách này giữ ứng dụng sạch hơn.

---

# 67. Instrumentation Scope

Trả lời: **telemetry này do instrumentation/library nào tạo?**

```text
name    = io.opentelemetry.jdbc
version = 2.x.x-alpha
```

```text
Resource             = AI đang chạy?          (order-service, pod abc)
InstrumentationScope = AI tạo telemetry này?  (thư viện JDBC instrumentation)
```

Hữu ích khi:

- Debug: span lạ này đến từ instrumentation nào?
- Lọc ở Collector: drop toàn bộ telemetry của một instrumentation gây ồn.
- Tắt đúng instrumentation trong Agent.

---

# 68. Semantic Conventions

Nếu mỗi team đặt tên một kiểu:

```text
http_method   http.method   request_method   verb
```

thì không thể viết một dashboard chung, backend không tự hiểu dữ liệu, và mọi query phải viết lại theo từng service.

Semantic Conventions chuẩn hóa: attribute names, span names, span kind, metric names, đơn vị, resource attributes, và **ý nghĩa** của chúng.

Một số nhóm quan trọng:

| Lĩnh vực | Attributes tiêu biểu (bản mới) |
|---|---|
| Service | `service.name`, `service.version`, `service.instance.id` |
| HTTP | `http.request.method`, `http.response.status_code`, `http.route`, `url.full`, `url.path`, `server.address`, `server.port` |
| Database | `db.system.name`, `db.namespace`, `db.operation.name`, `db.collection.name`, `db.query.text` |
| Messaging | `messaging.system`, `messaging.destination.name`, `messaging.operation.type` |
| RPC | `rpc.system`, `rpc.service`, `rpc.method` |
| Exception | `exception.type`, `exception.message`, `exception.stacktrace` |
| Error | `error.type` |

---

# 69. Semantic Conventions thay đổi theo thời gian

Nhiều attribute đã được đổi tên khi các nhóm conventions chuyển sang stable:

| Tên cũ | Tên mới |
|---|---|
| `http.method` | `http.request.method` |
| `http.status_code` | `http.response.status_code` |
| `http.url` | `url.full` |
| `net.peer.name` | `server.address` |
| `db.system` | `db.system.name` |
| `db.statement` | `db.query.text` |
| `http.server.duration` (ms) | `http.server.request.duration` (s) |

Hệ quả thực tế khi upgrade Agent (ví dụ từ 1.x lên 2.x):

- Dashboard/alert dùng tên cũ trả về **rỗng mà không báo lỗi**.
- Đơn vị duration có thể đổi từ ms sang s → threshold alert sai 1000 lần.

Một số instrumentation hỗ trợ chế độ chuyển tiếp phát cả tên cũ và mới (`OTEL_SEMCONV_STABILITY_OPT_IN`, ví dụ giá trị `http/dup`, `database/dup` tùy version). Kế hoạch an toàn: bật chế độ dup → migrate dashboard → tắt dup.

---

# PHẦN XI — EXPORTER & OTLP

# 70. Exporter

Exporter là adapter đưa dữ liệu từ SDK ra ngoài, mỗi signal một loại:

```text
Traces  → SpanExporter
Metrics → MetricExporter
Logs    → LogRecordExporter
```

```bash
OTEL_TRACES_EXPORTER=otlp      # otlp | console | logging | zipkin | none
OTEL_METRICS_EXPORTER=otlp     # otlp | prometheus | console | none
OTEL_LOGS_EXPORTER=otlp        # otlp | console | none
```

Đặt `none` để tắt hẳn một signal (ví dụ khi chưa có backend logs).

**Phân biệt SDK Exporter và Collector Exporter:**

```text
App ─[SDK OTLP Exporter]─► Collector Receiver ─► Processors ─► [Collector Exporter] ─► Backend
```

Hai thứ cùng tên "exporter" nhưng nằm ở hai nơi khác nhau và cấu hình hoàn toàn riêng.

---

# 71. OTLP là gì?

**OpenTelemetry Protocol**: định nghĩa cách mã hóa và truyền telemetry theo data model của OTel.

| Transport | Port mặc định | Endpoint | Encoding |
|---|---|---|---|
| OTLP/gRPC | **4317** | service gRPC | Protobuf |
| OTLP/HTTP | **4318** | `/v1/traces`, `/v1/metrics`, `/v1/logs` | Protobuf hoặc JSON |

```bash
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf   # mặc định của Java Agent 2.x
# hoặc: grpc, http/json

OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
```

**Bẫy phổ biến nhất về endpoint:**

```text
Protocol http/protobuf + OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4318
  → SDK tự nối thêm /v1/traces, /v1/metrics, /v1/logs   ✔

Protocol http/protobuf + OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://collector:4318
  → Endpoint riêng từng signal được dùng NGUYÊN VĂN, KHÔNG nối /v1/traces   ✘
  → phải ghi đầy đủ: http://collector:4318/v1/traces

Protocol grpc + endpoint port 4318 (hoặc ngược lại)
  → lỗi kết nối / lỗi protocol khó hiểu   ✘
```

---

# 72. Độ tin cậy của OTLP exporter

| Cơ chế | Mô tả |
|---|---|
| Retry | Exporter retry với exponential backoff cho lỗi tạm thời (unavailable, 429, 503) |
| Timeout | `OTEL_EXPORTER_OTLP_TIMEOUT` (mặc định 10s, tham khảo) |
| Compression | `OTEL_EXPORTER_OTLP_COMPRESSION=gzip`, giảm băng thông đáng kể |
| Headers | `OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer xyz` khi gửi tới vendor/gateway có auth |
| Partial success | Server có thể báo một phần dữ liệu bị từ chối |
| TLS | Dùng `https://` + cấu hình certificate khi đi qua mạng không tin cậy |

Exporter trong SDK **không có persistent queue**. Collector chết lâu hơn khả năng buffer của queue trong memory → mất dữ liệu. Đây là lý do nên có Collector chạy gần ứng dụng (node/sidecar) để nhận nhanh và buffer tốt hơn.

---

# 73. Tại sao OTLP quan trọng?

Không có protocol chuẩn:

```text
App
├── Jaeger protocol
├── Zipkin format
├── vendor A agent
├── vendor B SDK
└── Prometheus exposition
→ đổi vendor = sửa code + redeploy toàn bộ service
```

Có OTLP:

```text
App ──OTLP──► Collector ──┬──► backend A
                          ├──► backend B
                          └──► backend C (thử nghiệm song song)
→ đổi vendor = sửa config Collector
```

Hầu hết backend hiện đại (Tempo, Jaeger, Prometheus, Loki, Elastic, và đa số vendor thương mại) đã nhận OTLP trực tiếp.

---

# PHẦN XII — COLLECTOR

# 74. Collector là gì?

Collector là **telemetry data pipeline độc lập** với ứng dụng, là một binary riêng chạy như process/container riêng.

```text
RECEIVE ─► PROCESS ─► EXPORT
```

```text
               ┌───────────────────────────────┐
OTLP ─────────►│ Receivers                     │
Prometheus ───►│    ↓                          │
filelog ──────►│ Processors                    │
               │    ↓                          │──► Tempo
               │ Exporters                     │──► Prometheus
               │                               │──► Loki
               │ Connectors  (nối pipeline)    │
               │ Extensions  (health, auth...) │
               └───────────────────────────────┘
```

Collector **không tạo trace** cho business code của bạn. Nó nhận, xử lý và chuyển tiếp telemetry đã được tạo. (Ngoại lệ: một số receiver tự sinh telemetry bằng cách quan sát hệ thống, như `hostmetrics`, `k8s_cluster`, `kubeletstats`, hoặc scrape Prometheus.)

---

# 75. Có bắt buộc dùng Collector không?

Không. SDK có thể gửi thẳng tới backend:

```text
App ──OTLP──► Tempo / vendor
```

Nhưng trong production gần như luôn nên có, vì Collector cho phép:

| Lợi ích | Ví dụ |
|---|---|
| Tách cấu hình khỏi ứng dụng | Đổi backend, thêm backend mà không redeploy app |
| Offload xử lý | Batching, retry, compression, buffer khỏi ứng dụng |
| Enrich tập trung | Gắn `k8s.*` attributes, environment |
| Bảo mật | Xóa PII, giữ credentials backend ở một chỗ |
| Tail sampling | Chỉ làm được ở Collector |
| Thu thập thêm | Host metrics, log file, Prometheus scrape |
| Kiểm soát chi phí | Filter, drop, giảm cardinality trước khi tới backend tính tiền |

---

# 76. Distributions

| Distribution | Nội dung | Dùng khi |
|---|---|---|
| `otelcol` (core) | Ít component, chủ yếu OTLP | Rất đơn giản |
| `otelcol-contrib` | Hàng trăm component (k8sattributes, tail_sampling, filelog, …) | Phổ biến nhất; tiện để học |
| Custom build (OpenTelemetry Collector Builder – `ocb`) | Chỉ chứa component bạn chọn | Production: binary nhỏ, ít bề mặt tấn công |
| Vendor distributions | Đóng gói sẵn cho một vendor | Dùng vendor đó |

Mỗi component có mức stability riêng (development / alpha / beta / stable). Kiểm tra trước khi dựa vào nó trong production.

---

# 77. Receivers

Receiver là **input adapter**, không phải database.

| Receiver | Nhận gì |
|---|---|
| `otlp` | OTLP gRPC/HTTP từ SDK hoặc Collector khác |
| `prometheus` | Scrape endpoint Prometheus |
| `filelog` | Đọc file log (log container trong k8s) |
| `hostmetrics` | CPU, memory, disk, network của host |
| `kubeletstats`, `k8s_cluster` | Metrics Kubernetes |
| `kafka` | Telemetry đi qua Kafka |
| `jaeger`, `zipkin` | Hỗ trợ hệ thống cũ |

---

# 78. Processors và thứ tự của chúng

| Processor | Vai trò |
|---|---|
| `memory_limiter` | Từ chối dữ liệu khi Collector gần hết memory, tránh OOM. **Đặt đầu tiên** |
| `k8sattributes` | Gắn metadata Kubernetes |
| `resourcedetection` | Gắn thông tin host/cloud |
| `resource` | Thêm/sửa/xóa resource attributes |
| `attributes` | Thêm/sửa/xóa/hash span/log attributes |
| `filter` | Drop telemetry theo điều kiện (health check, debug log) |
| `transform` | Biến đổi linh hoạt bằng OTTL (OpenTelemetry Transformation Language) |
| `redaction` | Che dữ liệu nhạy cảm |
| `tail_sampling` | Sampling dựa trên cả trace |
| `batch` | Gom batch trước khi export. **Đặt gần cuối** |

Thứ tự khuyến nghị:

```text
memory_limiter
   ↓
k8sattributes / resourcedetection   (enrich sớm để các processor sau dùng được)
   ↓
filter                              (drop sớm, đỡ tốn công xử lý)
   ↓
attributes / transform / redaction
   ↓
tail_sampling                       (nếu có)
   ↓
batch
```

Ghi chú: các bản Collector gần đây đang chuyển dần việc batching vào cấu hình `sending_queue` của exporter. Kiểm tra khuyến nghị của version bạn dùng.

Ví dụ `filter` và `transform`:

```yaml
processors:
  filter/drop-health:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.route"] == "/actuator/health"'
        - 'attributes["http.route"] == "/actuator/prometheus"'

  transform/redact:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          - replace_pattern(attributes["db.query.text"], "'[^']*'", "'?'")
          - delete_key(attributes, "http.request.header.authorization")
```

---

# 79. Exporters của Collector

| Exporter | Gửi tới |
|---|---|
| `otlp` (gRPC) | Collector khác, Tempo, Jaeger, vendor |
| `otlphttp` | Loki (OTLP endpoint), Prometheus OTLP receiver, vendor |
| `prometheus` | Mở endpoint cho Prometheus scrape |
| `prometheusremotewrite` | Prometheus/Mimir/Thanos qua remote write |
| `kafka` | Đẩy vào Kafka làm buffer |
| `loadbalancing` | Phân phối theo trace_id tới nhiều Collector (cho tail sampling) |
| `debug` | In ra stdout của Collector (debug) |
| `file` | Ghi ra file |

Mỗi exporter có `sending_queue` và `retry_on_failure` riêng (mục 83).

---

# 80. Connectors

Connector vừa là **exporter của pipeline A** vừa là **receiver của pipeline B**.

```text
traces pipeline ──► [spanmetrics connector] ──► metrics pipeline
```

| Connector | Tác dụng |
|---|---|
| `spanmetrics` | Sinh metrics RED (Rate, Errors, Duration) từ span |
| `servicegraph` | Sinh metrics quan hệ giữa các service từ cặp CLIENT/SERVER span |
| `routing` | Định tuyến telemetry sang pipeline khác theo điều kiện (ví dụ theo tenant) |
| `count` | Đếm span/log thỏa điều kiện, phát thành metric |
| `forward` | Nối pipeline đơn giản |

---

# 81. Extensions

Không nằm trong luồng dữ liệu, cung cấp chức năng phụ:

| Extension | Tác dụng |
|---|---|
| `health_check` | Endpoint cho liveness/readiness probe |
| `pprof` | Profiling Collector |
| `zpages` | Trang debug nội bộ về pipeline |
| `file_storage` | Lưu queue xuống disk (persistent queue) |
| `basicauth`, `bearertokenauth`, `oauth2client`, `oidc` | Xác thực receiver/exporter |

---

# 82. Cấu hình Collector đầy đủ (ví dụ)

```yaml
extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  file_storage:
    directory: /var/lib/otelcol/queue

receivers:
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }
      http: { endpoint: 0.0.0.0:4318 }

processors:
  memory_limiter:
    check_interval: 1s
    limit_percentage: 80
    spike_limit_percentage: 20

  k8sattributes: {}

  resource:
    attributes:
      - key: deployment.environment.name
        value: production
        action: upsert

  filter/drop-health:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.route"] == "/actuator/health"'

  batch:
    send_batch_size: 8192
    timeout: 5s

connectors:
  spanmetrics:
    dimensions:
      - name: http.request.method
      - name: http.response.status_code

exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls: { insecure: true }
    sending_queue:
      enabled: true
      storage: file_storage
    retry_on_failure:
      enabled: true
      max_elapsed_time: 300s

  otlphttp/loki:
    endpoint: http://loki:3100/otlp

  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write

service:
  extensions: [health_check, file_storage]
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, k8sattributes, resource, filter/drop-health, batch]
      exporters:  [otlp/tempo, spanmetrics]

    metrics:
      receivers:  [otlp, spanmetrics]
      processors: [memory_limiter, k8sattributes, resource, batch]
      exporters:  [prometheusremotewrite]

    logs:
      receivers:  [otlp]
      processors: [memory_limiter, k8sattributes, resource, batch]
      exporters:  [otlphttp/loki]

  telemetry:
    metrics:
      level: detailed
```

Điểm cần nhớ:

- Component được **khai báo** ở trên nhưng chỉ **hoạt động** khi được dùng trong `service.pipelines`.
- Một component có thể dùng trong nhiều pipeline, nhưng mỗi pipeline chỉ cho một loại signal.
- Đặt tên dạng `type/name` (`otlp/tempo`, `filter/drop-health`) để có nhiều instance cùng loại.
- Với tail sampling, `spanmetrics` nên lấy dữ liệu **trước** `tail_sampling` (mục 50).

---

# 83. Độ tin cậy: queue, retry, backpressure

```text
Receiver ─► memory_limiter ─► ... ─► Exporter
                                        │
                                        ├── sending_queue (memory hoặc file_storage)
                                        │      num_consumers, queue_size
                                        │
                                        └── retry_on_failure
                                               initial_interval, max_interval, max_elapsed_time
```

| Tình huống | Hành vi | Giảm rủi ro |
|---|---|---|
| Backend chậm/tạm chết | Exporter retry, dữ liệu dồn vào queue | Queue đủ lớn, persistent queue |
| Queue đầy | Dữ liệu mới bị drop | Theo dõi metric drop của Collector, scale out |
| Collector gần hết memory | `memory_limiter` từ chối dữ liệu → SDK nhận lỗi → SDK retry/buffer | Đặt `memory_limiter`, đặt resource limits hợp lý |
| Collector restart | Queue trong memory mất | `file_storage` cho queue |
| Lượng dữ liệu cực lớn | Một Collector không chịu nổi | Scale ngang; cân nhắc Kafka làm buffer |

---

# 84. Các mô hình triển khai Collector

**Mô hình 0: Không có Collector**

```text
App ──► Backend
```

Đơn giản, phù hợp dev/POC. Mọi cấu hình nằm trong app.

**Mô hình 1: Agent (gần ứng dụng)**

```text
Node 1:  App A, App B ──► Collector (DaemonSet)  ─┐
Node 2:  App C, App D ──► Collector (DaemonSet)  ─┼─► Backend
```

Hoặc sidecar: mỗi pod một Collector. Ưu: nhận nhanh, gắn metadata host/pod dễ, đọc log file của node. Nhược: nhiều instance, khó tail sampling.

**Mô hình 2: Gateway (tập trung)**

```text
All apps ──► Load balancer ──► Collector Deployment (N replicas) ──► Backend
```

Ưu: quản lý tập trung, giữ credentials ở một nơi. Nhược: thêm một hop qua network.

**Mô hình 3: Agent + Gateway (phổ biến nhất ở quy mô lớn)**

```text
App ──► Agent Collector (node) ──► Gateway Collector ──► Backend
        - nhận nhanh               - tail sampling
        - k8sattributes            - redaction, routing
        - filelog, hostmetrics     - auth tới vendor
        - loadbalancing exporter   - kiểm soát chi phí
```

| Nhu cầu | Mô hình phù hợp |
|---|---|
| Dev / POC | 0 hoặc 2 |
| K8s cần log file + host metrics | 1 hoặc 3 |
| Tail sampling | 3 (tầng 1 loadbalancing, tầng 2 tail_sampling) |
| Nhiều team, nhiều backend | 2 hoặc 3 |

Trong Kubernetes, **OpenTelemetry Operator** giúp quản lý Collector bằng CRD và có thể **tự inject Java Agent** vào pod qua annotation, không cần sửa Dockerfile.

---

# 85. Giám sát chính Collector

Collector là hạ tầng production, cần được giám sát như mọi service khác. Nó tự phát telemetry nội bộ (cấu hình ở `service.telemetry`).

Các tín hiệu cần alert (tên metric có thể khác tùy version):

| Ý nghĩa | Nhóm metric |
|---|---|
| Dữ liệu nhận vào | `otelcol_receiver_accepted_*`, `otelcol_receiver_refused_*` |
| Dữ liệu gửi thành công/thất bại | `otelcol_exporter_sent_*`, `otelcol_exporter_send_failed_*` |
| Queue | `otelcol_exporter_queue_size` so với `otelcol_exporter_queue_capacity` |
| Bị từ chối do memory | refused tăng khi `memory_limiter` kích hoạt |
| Tài nguyên | CPU, memory của process Collector |

Alert quan trọng nhất: **`send_failed` > 0 kéo dài** và **queue gần đầy**.

---

# 86. Bảo mật

- Receiver chỉ bind vào interface cần thiết. Không mở 4317/4318 ra internet.
- Bật TLS và authentication cho receiver nếu nhận dữ liệu từ ngoài cluster.
- Telemetry **chứa dữ liệu nhạy cảm nhiều hơn bạn nghĩ**: SQL có giá trị tham số, URL có token trong query string, header, message exception có email. Dùng `transform`/`redaction`/`attributes` để làm sạch tại Collector.
- Giữ API key của backend/vendor ở Collector (qua secret), không phát tán vào mọi ứng dụng.
- Cẩn thận `traceparent` từ client bên ngoài: kẻ xấu có thể ép sample (flag `01`) hàng loạt để tăng chi phí. Ở edge, cân nhắc bỏ qua quyết định sampling từ ngoài hoặc bắt đầu trace mới và dùng Link.

---

# PHẦN XIII — BACKEND & GRAFANA

# 87. Backend

| Signal | Backend phổ biến (open source) | Ghi chú |
|---|---|---|
| Traces | Grafana Tempo, Jaeger | Tempo lưu trên object storage, query bằng TraceQL |
| Metrics | Prometheus, Grafana Mimir, Thanos, VictoriaMetrics | Prometheus bản mới có thể nhận OTLP trực tiếp khi bật tính năng tương ứng |
| Logs | Grafana Loki, Elasticsearch/OpenSearch | Loki 3.x nhận OTLP qua endpoint `/otlp` |
| Tất cả | Vendor thương mại, hoặc các nền tảng all-in-one | Đa số nhận OTLP |

OpenTelemetry **không bắt buộc** backend cụ thể. Đây chính là tính "vendor-neutral".

Lưu ý khi đưa OTLP vào Loki/Prometheus: resource attributes không tự động thành label hết (để tránh cardinality). Mỗi backend có quy tắc riêng về attribute nào được "promote" thành label/index. Cần đọc tài liệu backend để biết cách query.

---

# 88. Grafana và correlation giữa các signal

Grafana là **visualization/query layer**. Grafana không tạo TraceId, không propagate Context, không phải SDK.

Giá trị thật của Grafana khi đi cùng OTel là **nhảy qua lại giữa các signal**:

```text
            ┌──────── exemplar ────────┐
            │                          ▼
     ┌─────────────┐             ┌───────────┐
     │  Metrics    │             │  Traces   │
     │ (Prometheus)│◄── span ────│  (Tempo)  │
     └─────────────┘  metrics    └───────────┘
                                   │     ▲
                     trace → logs  │     │ logs → trace
                     (trace_id)    ▼     │ (derived field trace_id)
                                 ┌───────────┐
                                 │   Logs    │
                                 │  (Loki)   │
                                 └───────────┘
```

| Liên kết | Cấu hình |
|---|---|
| Metric → Trace | Exemplars được lưu ở Prometheus + datasource Prometheus trỏ tới Tempo |
| Trace → Logs | Datasource Tempo cấu hình "trace to logs" trỏ tới Loki, lọc theo `trace_id` và `service.name` |
| Logs → Trace | Datasource Loki có derived field hoặc structured metadata `trace_id` link sang Tempo |
| Trace → Metrics | Datasource Tempo cấu hình "trace to metrics" |
| Service graph | Tempo metrics-generator hoặc `servicegraph` connector trong Collector |

Điều kiện tiên quyết cho mọi liên kết: **`service.name` thống nhất** giữa ba signal và log có **`trace_id`**.

---

# 89. Tempo, TraceQL và tìm trace

Tìm trace không chỉ bằng trace_id. TraceQL cho phép query theo thuộc tính span:

```text
{ resource.service.name = "order-service" && span.http.response.status_code >= 500 }

{ span.db.system.name = "postgresql" && duration > 500ms }

{ resource.service.name = "order-service" } >> { resource.service.name = "payment-service" && status = error }
```

Câu cuối: tìm trace mà order-service gọi (trực tiếp hoặc gián tiếp) payment-service và payment-service lỗi.

Đây là lý do attributes đặt đúng semantic conventions có giá trị lớn: chúng trở thành **chiều để tìm kiếm**.

---

# 90. Một ví dụ điều tra sự cố end-to-end

```text
1. Alert: p99 http.server.request.duration của /orders > 2s
       │
2. Mở dashboard, thấy spike lúc 14:02; resource.service.version đổi sang 1.5.0 lúc 14:00
       │
3. Click exemplar trên điểm spike → mở trace chậm trong Tempo
       │
4. Trace cho thấy span "SELECT orders" mất 1.8s
       │
5. Xem attributes: db.query.text cho thấy query mới không dùng index
       │
6. "Trace to logs" → log của request đó: warning về full table scan
       │
7. Kết luận: release 1.5.0 thêm điều kiện lọc mới làm mất index → rollback
```

Mọi bước trên chỉ khả thi khi: metrics có exemplar, Resource có `service.version`, span có semantic attributes đúng, log có `trace_id`.

---

# PHẦN XIV — MỘT REQUEST ĐI QUA TOÀN HỆ THỐNG

# 91. Walkthrough chi tiết: `POST /orders`

**Bước 1 — Request vào Order Service**

```text
Client ──POST /orders──► Tomcat (Order Service)
```

Agent đã instrument Tomcat. Instrumentation:

```text
propagator.extract(headers)
   ├── có traceparent? → dùng làm remote parent
   └── không có        → root
Sampler (ParentBased) quyết định
Tạo SERVER span 001, trace ABC
```

**Bước 2 — Context được đặt thành current**

```text
Context { span 001 } ── makeCurrent() trên thread xử lý request
MDC: trace_id=ABC, span_id=001
```

Mọi log ghi từ đây tự có trace_id.

**Bước 3 — JDBC query**

```text
OrderRepository.save()
   └── JDBC instrumentation đọc current Context → parent = 001
       CLIENT span 002 "INSERT orders"
       db.system.name=postgresql, db.operation.name=INSERT, db.collection.name=orders
```

**Bước 4 — Custom business span**

```text
PricingService.calculate()  (manual API)
   └── INTERNAL span 003 "calculate-order-price", parent = 001
       attributes: order.id, order.item_count, price.final
       metric: pricing.discount.applied +1
```

**Bước 5 — Publish Kafka event**

```text
kafkaTemplate.send("orders", event)
   └── PRODUCER span 004 "orders publish", parent = 001
       inject traceparent vào Kafka record headers
```

**Bước 6 — Gọi Payment Service**

```text
RestClient POST http://payment/charge
   └── CLIENT span 005, parent = 001
       propagator.inject → header traceparent: 00-ABC-005-01
```

**Bước 7 — Payment Service nhận request**

```text
Agent trong Payment Service:
   extract traceparent → remote parent = 005, trace = ABC, sampled = 1
   SERVER span 006, parent = 005
   └── CLIENT span 007 "GET" (Redis), parent = 006
```

**Bước 8 — Notification Service consume Kafka (bất đồng bộ, có thể vài giây sau)**

```text
extract traceparent từ record headers → parent = 004
CONSUMER span 008 "orders process", trace = ABC
```

**Bước 9 — Span kết thúc và được export**

Mỗi service, mỗi span khi `end()`:

```text
span.end() → BatchSpanProcessor queue → (≤5s) → OTLP exporter → Collector
```

Metrics được ghi song song (`http.server.request.duration` với exemplar trỏ về trace ABC), export mỗi 60s. Logs được export theo batch, mang trace_id=ABC.

**Bước 10 — Collector xử lý**

```text
Receiver otlp
  → memory_limiter → k8sattributes (thêm k8s.pod.name...) → filter → batch
  → spanmetrics (sinh RED metrics)
  → Tempo / Prometheus / Loki
```

**Bước 11 — Backend ghép trace**

Tempo nhận span từ ba service vào các thời điểm khác nhau, ghép thành cây theo trace_id ABC:

```text
Trace ABC
POST /orders                             [order-service]        SERVER   001
├── INSERT orders                        [order-service]        CLIENT   002
├── calculate-order-price                [order-service]        INTERNAL 003
├── orders publish                       [order-service]        PRODUCER 004
│   └── orders process                   [notification-service] CONSUMER 008
└── POST                                 [order-service]        CLIENT   005
    └── POST /charge                     [payment-service]      SERVER   006
        └── GET                          [payment-service]      CLIENT   007
```

**Toàn bộ luồng dạng sơ đồ:**

```text
HTTP Request
     │
     ▼
Tomcat Instrumentation ── extract ── SERVER SPAN 001
     │
     ├──────────────┬────────────────┬─────────────────┐
     ▼              ▼                ▼                 ▼
JDBC Agent     Manual API       Kafka Agent      HTTP Client Agent
SPAN 002       SPAN 003         SPAN 004         SPAN 005
                                   │ inject          │ inject
                                   ▼                 ▼
                              Kafka headers     traceparent
                                   │                 │
───────────────────────────── NETWORK ─────────────────────────
                                   │                 │
                                   ▼ extract         ▼ extract
                           Notification Agent   Payment Agent
                           CONSUMER SPAN 008    SERVER SPAN 006 → Redis 007
                                   │                 │
            ┌──────────────────────┴─────────────────┘
            ▼
   OTel SDK (mỗi service) → BatchSpanProcessor → OTLP Exporter
            │
            ▼
   Collector: Receiver → Processors → Exporters
            │
            ▼
   Tempo / Prometheus / Loki ──► Grafana
```

---

# PHẦN XV — MÔ HÌNH TRIỂN KHAI JAVA

# 92. Model A — Java Agent

```text
Application + -javaagent
Agent = auto instrument + autoconfigure SDK + export
```

Phù hợp nhất để bắt đầu. Không sửa code. Cấu hình hoàn toàn bằng env vars.

# 93. Model B — Manual API + SDK

```text
add opentelemetry-api + opentelemetry-sdk + exporter
tự configure SDK (hoặc dùng sdk-extension-autoconfigure)
tự viết hoặc thêm instrumentation libraries
```

Kiểm soát cao nhất, nhiều code nhất. Phù hợp khi không được dùng `-javaagent` hoặc cần kiểm soát chặt (ví dụ native image).

# 94. Model C — Agent + custom API instrumentation (khuyến nghị cho đa số)

```text
Java Agent → framework telemetry:  GET /orders, SELECT, POST payment, Kafka
OTel API   → business telemetry:   validate-order, calculate-pricing, reserve-inventory
```

Cả hai dùng chung Trace/Context. Đây là model thực tế mạnh nhất.

# 95. Model D — Spring Boot Starter / Micrometer

| Lựa chọn | Mô tả | Khi nào |
|---|---|---|
| OpenTelemetry Spring Boot Starter | Instrumentation dạng library, cấu hình qua `application.yml`, không cần `-javaagent` | Không được thêm JVM flag; GraalVM native image; muốn startup nhanh hơn |
| Micrometer Tracing + OTel bridge (Spring Boot 3) | Dùng Observation API của Spring, export qua OTel | Team đã đầu tư vào Micrometer/Actuator |

So sánh nhanh:

| | Agent | Spring Starter | Micrometer |
|---|---|---|---|
| Sửa code/dependency | Không | Thêm dependency | Thêm dependency |
| Độ phủ library | Rộng nhất | Hẹp hơn | Theo hệ sinh thái Spring |
| Native image | Không | Có | Có |
| Startup overhead | Cao hơn | Thấp hơn | Thấp hơn |
| Semantic conventions OTel | Đầy đủ | Đầy đủ | Có thể khác tên |

Tránh **chạy đồng thời** Agent và một cơ chế tracing khác cho cùng library: dễ sinh span trùng hoặc hai trace song song.

---

# PHẦN XVI — BẢNG THAM CHIẾU CẤU HÌNH

# 96. Environment variables quan trọng

Giá trị mặc định mang tính tham khảo, kiểm tra theo version.

**Định danh và Resource**

| Biến | Ví dụ / Mặc định | Ý nghĩa |
|---|---|---|
| `OTEL_SERVICE_NAME` | `order-service` | Tên service. **Luôn đặt** |
| `OTEL_RESOURCE_ATTRIBUTES` | `service.version=1.4.0,deployment.environment.name=prod` | Resource attributes thêm |
| `OTEL_SDK_DISABLED` | `false` | Tắt toàn bộ SDK |

**Exporter / OTLP**

| Biến | Ví dụ / Mặc định | Ý nghĩa |
|---|---|---|
| `OTEL_TRACES_EXPORTER` | `otlp` | `otlp`, `console`, `none` … |
| `OTEL_METRICS_EXPORTER` | `otlp` | `otlp`, `prometheus`, `none` … |
| `OTEL_LOGS_EXPORTER` | `otlp` | `otlp`, `none` … |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:4318` | Endpoint chung, SDK tự nối `/v1/<signal>` với HTTP |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` | `http://c:4318/v1/traces` | Endpoint riêng, dùng nguyên văn |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `http/protobuf` | `grpc`, `http/protobuf`, `http/json` |
| `OTEL_EXPORTER_OTLP_HEADERS` | `authorization=Bearer xyz` | Header thêm |
| `OTEL_EXPORTER_OTLP_COMPRESSION` | `gzip` | Nén |
| `OTEL_EXPORTER_OTLP_TIMEOUT` | `10000` | Timeout (ms) |

**Propagation và Sampling**

| Biến | Ví dụ / Mặc định | Ý nghĩa |
|---|---|---|
| `OTEL_PROPAGATORS` | `tracecontext,baggage` | Format propagation |
| `OTEL_TRACES_SAMPLER` | `parentbased_always_on` | Sampler |
| `OTEL_TRACES_SAMPLER_ARG` | `0.1` | Tham số sampler (tỉ lệ) |

**Batch / Limits**

| Biến | Mặc định | Ý nghĩa |
|---|---|---|
| `OTEL_BSP_SCHEDULE_DELAY` | `5000` | Chu kỳ export span (ms) |
| `OTEL_BSP_MAX_QUEUE_SIZE` | `2048` | Kích thước queue span |
| `OTEL_BSP_MAX_EXPORT_BATCH_SIZE` | `512` | Span mỗi batch |
| `OTEL_BLRP_SCHEDULE_DELAY` | `1000` | Chu kỳ export log (ms) |
| `OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT` | không giới hạn | Độ dài tối đa giá trị attribute |
| `OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT` | `128` | Số attribute mỗi span |

**Metrics**

| Biến | Mặc định | Ý nghĩa |
|---|---|---|
| `OTEL_METRIC_EXPORT_INTERVAL` | `60000` | Chu kỳ export metrics (ms) |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | `cumulative` | `cumulative`, `delta`, `lowmemory` |
| `OTEL_METRICS_EXEMPLAR_FILTER` | `trace_based` | `always_on`, `always_off`, `trace_based` |

**Riêng Java Agent**

| Biến | Ý nghĩa |
|---|---|
| `OTEL_JAVAAGENT_DEBUG` | Log debug của agent |
| `OTEL_JAVAAGENT_EXTENSIONS` | Đường dẫn extension jar |
| `OTEL_JAVAAGENT_CONFIGURATION_FILE` | File properties cấu hình |
| `OTEL_INSTRUMENTATION_<NAME>_ENABLED` | Bật/tắt instrumentation cụ thể |
| `OTEL_INSTRUMENTATION_COMMON_DEFAULT_ENABLED` | Mặc định bật/tắt mọi instrumentation |
| `OTEL_SEMCONV_STABILITY_OPT_IN` | Chế độ chuyển tiếp semantic conventions |

---

# PHẦN XVII — TROUBLESHOOTING

# 97. Bảng triệu chứng → nguyên nhân → cách sửa

| Triệu chứng | Nguyên nhân thường gặp | Cách kiểm tra / sửa |
|---|---|---|
| Không thấy telemetry nào | Sai endpoint/port/protocol; Collector không reachable; exporter = none | `OTEL_TRACES_EXPORTER=console,otlp` xem span có được tạo không; kiểm tra log agent; dùng `debug` exporter ở Collector |
| Service hiện là `unknown_service:java` | Không đặt `OTEL_SERVICE_NAME` | Đặt biến này |
| Trace bị tách thành nhiều trace | Context mất qua async; propagator không khớp giữa service; proxy/gateway xóa header `traceparent` | Kiểm tra header ở service nhận; xem trace_id hai bên; kiểm tra thread pool tự viết |
| Span con nằm sai parent | Quên `scope.close()` (context leak), hoặc tạo span con khi parent chưa `makeCurrent()` | Dùng try-with-resources |
| Trace thiếu một số span | Downstream dùng sampler không ParentBased; queue đầy bị drop; span chưa `end()` | Kiểm tra sampler config ở mọi service; metric drop |
| Span "treo" / không bao giờ xuất hiện | `span.end()` không được gọi trên một nhánh lỗi | `end()` trong `finally` |
| Span trùng lặp | Agent + library instrumentation khác cùng instrument; hai SDK cùng chạy | Chỉ dùng một cơ chế; không tạo SDK thứ hai |
| Log không có trace_id | Log ghi ngoài Context (thread khác); pattern không có `%X{trace_id}`; dùng logging framework không được hỗ trợ | Kiểm tra MDC; kiểm tra log ghi trong span |
| Metrics sai lệch / `rate()` vô nghĩa | Temporality không khớp backend | Đặt temporality đúng, hoặc `deltatocumulative` |
| Backend metrics chậm/sập, chi phí tăng vọt | Cardinality bùng nổ | Tìm attribute high-cardinality; dùng Views/filter loại bỏ |
| Dashboard trống sau khi upgrade agent | Semantic conventions đổi tên/đơn vị | Đối chiếu bảng mục 69; dùng chế độ dup khi migrate |
| Collector OOM | Thiếu `memory_limiter`; tail sampling giữ quá nhiều trace | Thêm `memory_limiter`; giảm `num_traces`/`decision_wait`; scale |
| Latency ứng dụng tăng sau khi bật agent | Quá nhiều span (instrument quá chi tiết); SimpleSpanProcessor | Tắt instrumentation không cần; dùng batch; sampling |
| Exemplar không hiện trong Grafana | Prometheus chưa bật lưu exemplar; datasource chưa cấu hình link; request không được sample | Bật exemplar storage; cấu hình datasource |

# 98. Quy trình debug từng tầng

Đi từ gần ứng dụng ra xa, kiểm tra từng hop:

```text
1. App có tạo telemetry không?
      → OTEL_TRACES_EXPORTER=console (hoặc logging) và xem stdout
2. App có gửi được tới Collector không?
      → log của agent có lỗi export? telnet/curl tới 4317/4318
3. Collector có nhận không?
      → metric otelcol_receiver_accepted_*, hoặc thêm exporter debug
4. Collector có gửi được tới backend không?
      → otelcol_exporter_send_failed_*, log Collector
5. Backend có lưu và query được không?
      → query trực tiếp backend bằng trace_id cụ thể
6. Grafana có cấu hình datasource đúng không?
```

Exporter `debug` của Collector rất hữu ích:

```yaml
exporters:
  debug:
    verbosity: detailed
service:
  pipelines:
    traces:
      exporters: [debug, otlp/tempo]
```

---

# PHẦN XVIII — ANTI-PATTERNS & CHECKLIST PRODUCTION

# 99. Những hiểu lầm và anti-patterns

| Hiểu lầm / Anti-pattern | Thực tế / Nên làm |
|---|---|
| Java Agent = Collector | Agent instrument ứng dụng; Collector xử lý/route telemetry |
| API = SDK | API là contract; SDK là engine |
| Agent thay thế hoàn toàn API | Agent cho framework; API cho business semantics |
| Collector lưu trace | Backend (Tempo/Jaeger) lưu; Collector chỉ chuyển tiếp |
| Tự truyền trace bằng `X-Trace-Id` custom | Dùng `traceparent` chuẩn qua propagator |
| Tạo span cho mọi method | Span cho operation có giá trị quan sát: IO, business step quan trọng |
| Đưa ID vào tên span | Tên low-cardinality, ID vào attributes |
| Đưa `user_id` vào metric attribute | Để ở span attribute hoặc log |
| Tạo thêm SDK khi đã dùng Agent | Dùng `GlobalOpenTelemetry` của Agent |
| Đưa token/PII vào Baggage | Baggage đi tới mọi downstream; chỉ dữ liệu không nhạy cảm |
| Dùng `SimpleSpanProcessor` ở production | `BatchSpanProcessor` |
| Không đặt `service.name` | Luôn đặt, kèm `service.version`, `deployment.environment.name` |
| Đếm span sau sampling để tính request rate | Dùng metrics HTTP hoặc spanmetrics trước sampling |
| Tin rằng traces là đủ để audit | Traces bị sample; audit dùng log/hệ thống riêng |
| Upgrade agent không xem release notes | Semantic conventions có thể đổi, dashboard hỏng im lặng |
| Mở 4317/4318 ra internet không auth | TLS + auth, hoặc chỉ nội bộ |

# 100. Checklist production

**Ứng dụng**

- [ ] `OTEL_SERVICE_NAME`, `service.version`, `deployment.environment.name` được đặt cho mọi service
- [ ] Sampler là `parentbased_*` ở mọi service; chỉ root quyết định tỉ lệ
- [ ] Propagator thống nhất toàn hệ thống (`tracecontext,baggage`)
- [ ] Tắt instrumentation không dùng; health check/metrics endpoint không sinh trace (filter ở sampler hoặc Collector)
- [ ] `OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT` được đặt
- [ ] Manual span tuân theo checklist mục 36
- [ ] Log có `trace_id` (MDC hoặc OTLP logs)
- [ ] Đã đo overhead (CPU, p99 latency, startup) trên workload thật
- [ ] Version `opentelemetry-api` khớp với Agent (dùng BOM)

**Metrics**

- [ ] Không có attribute high-cardinality
- [ ] Temporality khớp backend
- [ ] Exemplars bật và liên kết tới traces
- [ ] Alert dựa trên metrics, không dựa trên trace đã sample

**Collector**

- [ ] `memory_limiter` đứng đầu mọi pipeline
- [ ] Batching được bật
- [ ] `sending_queue` + `retry_on_failure` cấu hình; persistent queue nếu cần độ bền
- [ ] `health_check` gắn với liveness/readiness probe
- [ ] Collector được giám sát (accepted, refused, send_failed, queue size)
- [ ] Redaction cho dữ liệu nhạy cảm (SQL params, headers, URL query)
- [ ] Receiver không public; có TLS/auth khi cần
- [ ] Nếu tail sampling: kiến trúc 2 tầng với `loadbalancing` exporter
- [ ] Dùng custom build hoặc chỉ bật component cần thiết

**Backend & vận hành**

- [ ] Retention phù hợp chi phí cho từng signal
- [ ] Grafana datasource liên kết trace ↔ logs ↔ metrics
- [ ] Có runbook: "không thấy trace thì kiểm tra gì"
- [ ] Quy trình upgrade có bước kiểm tra semantic conventions và dashboard

---

# PHẦN XIX — LAB THỰC HÀNH

# 101. Stack tối thiểu bằng docker-compose

Mục tiêu: một Spring Boot app với Java Agent → Collector → Grafana LGTM (Loki, Grafana, Tempo, Prometheus/Mimir trong một image tiện cho lab).

Cấu trúc thư mục:

```text
otel-lab/
├── docker-compose.yml
├── otel-collector.yaml
└── order-service/       (Spring Boot app, có Dockerfile)
```

Tải agent:

```bash
curl -L -o order-service/opentelemetry-javaagent.jar \
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
```

`docker-compose.yml`:

```yaml
services:
  lgtm:
    image: grafana/otel-lgtm:latest
    ports:
      - "3000:3000"           # Grafana (admin/admin)

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otelcol/config.yaml"]
    volumes:
      - ./otel-collector.yaml:/etc/otelcol/config.yaml:ro
    ports:
      - "4317:4317"
      - "4318:4318"
    depends_on: [lgtm]

  order-service:
    build: ./order-service
    environment:
      JAVA_TOOL_OPTIONS: "-javaagent:/app/opentelemetry-javaagent.jar"
      OTEL_SERVICE_NAME: order-service
      OTEL_RESOURCE_ATTRIBUTES: service.version=1.0.0,deployment.environment.name=lab
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4318
      OTEL_EXPORTER_OTLP_PROTOCOL: http/protobuf
      OTEL_METRIC_EXPORT_INTERVAL: "10000"   # nhanh hơn cho lab
    ports:
      - "8080:8080"
    depends_on: [otel-collector]
```

`otel-collector.yaml`:

```yaml
receivers:
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }
      http: { endpoint: 0.0.0.0:4318 }

processors:
  memory_limiter:
    check_interval: 1s
    limit_percentage: 80
    spike_limit_percentage: 20
  batch: {}

exporters:
  otlphttp/lgtm:
    endpoint: http://lgtm:4318
  debug:
    verbosity: basic

service:
  pipelines:
    traces:  { receivers: [otlp], processors: [memory_limiter, batch], exporters: [otlphttp/lgtm, debug] }
    metrics: { receivers: [otlp], processors: [memory_limiter, batch], exporters: [otlphttp/lgtm] }
    logs:    { receivers: [otlp], processors: [memory_limiter, batch], exporters: [otlphttp/lgtm] }
```

Dockerfile của app (ví dụ):

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY opentelemetry-javaagent.jar /app/
COPY build/libs/order-service.jar /app/app.jar
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Chạy:

```bash
docker compose up --build
curl -X POST localhost:8080/orders -H 'content-type: application/json' -d '{"items":[1,2]}'
# mở http://localhost:3000 → Explore → Tempo / Loki / Prometheus
```

# 102. Bài tập: cố tình làm hỏng để hiểu

| # | Bài tập | Quan sát |
|---|---|---|
| 1 | Bỏ `OTEL_SERVICE_NAME` | Service hiện là `unknown_service:java` |
| 2 | Thêm service thứ hai (payment), gọi qua HTTP | Một trace gồm span của hai service |
| 3 | Ở payment, đặt `OTEL_PROPAGATORS=b3` | Trace bị tách đôi |
| 4 | Gọi downstream trong một `new Thread(...)` tự tạo | Trace bị gãy; sửa bằng `Context.current().wrap()` |
| 5 | Thêm manual span, cố tình quên `scope.close()` | Span của request sau gắn sai parent |
| 6 | Đặt sampler `traceidratio` 0.5 ở service đầu, `traceidratio` 0.1 ở service sau (không ParentBased) | Trace thiếu span |
| 7 | Thêm attribute `user.id` vào một Counter, bắn nhiều user id | Số series tăng vọt trong Prometheus |
| 8 | Tắt Collector trong 1 phút rồi bật lại | Quan sát span bị mất/được retry; log lỗi export của agent |
| 9 | Thêm `filter` drop `/actuator/health` | Trace health check biến mất |
| 10 | Thêm tail sampling chỉ giữ trace lỗi + 10% còn lại | So sánh số trace trước/sau |
| 11 | Từ một log lỗi trong Loki, nhảy sang trace; từ exemplar trên biểu đồ latency, nhảy sang trace | Hiểu correlation |

---

# PHẦN XX — TỔNG KẾT

# 103. Bảng chốt: thành phần và câu hỏi nó giải quyết

| Thành phần | Câu hỏi nó giải quyết |
|---|---|
| Java Agent | Làm sao quan sát framework/library mà không sửa source code? |
| OTel API | Code/library dùng interface nào để tạo telemetry? |
| OTel SDK | Ai thực sự sample, xử lý, batch, aggregate và export? |
| Context | Operation hiện tại thuộc trace/parent nào? |
| Propagator | Làm sao Context đi qua HTTP/Kafka/gRPC boundary? |
| Baggage | Làm sao truyền dữ liệu nghiệp vụ nhỏ xuyên service? |
| Sampler | Trace nào được giữ? |
| Resource | Telemetry do service/process/pod nào sinh ra? |
| Instrumentation Scope | Thư viện instrumentation nào tạo ra telemetry? |
| Semantic Conventions | Dữ liệu được đặt tên và diễn giải thống nhất thế nào? |
| Views | Metric được aggregate ra sao, giữ attribute nào? |
| Exemplars | Điểm metric này đến từ request nào? |
| Exporter | SDK gửi telemetry ra ngoài bằng cách nào? |
| OTLP | Telemetry đi qua network bằng protocol chuẩn nào? |
| Collector | Nhận, xử lý, sample, route telemetry tập trung ở đâu? |
| Backend | Telemetry được lưu và query ở đâu? |
| Grafana | Con người xem, liên kết và alert telemetry ở đâu? |

# 104. First-principles summary

```text
Một operation xảy ra                         → cần mô tả         → Span
Nhiều operation liên quan                    → cần nối           → Trace
Execution đi qua code/thread                 → giữ quan hệ       → Context
Context đi qua network                       → serialize         → Propagation
Không thể lưu hết                            → chọn lọc          → Sampling
Cần nhìn tổng thể, rẻ, để alert              → aggregate         → Metrics
Cần chi tiết sự kiện                         → ghi lại           → Logs
Cần biết ai sinh dữ liệu                     → định danh         → Resource
Cần mọi người hiểu dữ liệu giống nhau        → chuẩn hóa         → Semantic Conventions
Code/framework cần tạo telemetry             → interface         → API + Instrumentation
Telemetry cần được xử lý                     → engine            → SDK
Telemetry cần ra khỏi process                → adapter           → Exporter
Các hệ thống cần ngôn ngữ chung              → protocol          → OTLP
Cần xử lý, routing tập trung                 → pipeline          → Collector
Cần lưu và query                             → storage           → Backend
Con người cần xem và liên kết                → UI                → Grafana
```

# 105. Sơ đồ cuối cùng cần nhớ

```text
                               JVM
┌──────────────────────────────────────────────────────────────┐
│  Java Agent                                                  │
│    ├── auto instrument Tomcat / Spring / JDBC / HTTP / Kafka │
│    ├── context propagation (inject/extract, async)           │
│    └── autoconfigure SDK                                     │
│                                                              │
│  Application                                                 │
│    ├── framework code ──────────────────┐                    │
│    └── business code ── OTel API ───────┤                    │
│                                         ▼                    │
│                              SDK  (Resource chung)           │
│        ┌───────────────────────┼───────────────────────┐     │
│        ▼                       ▼                       ▼     │
│  TracerProvider          MeterProvider          LoggerProvider
│   Sampler                 Views                        │     │
│   BatchSpanProcessor      PeriodicMetricReader  BatchLogProc │
│        └───────────────────────┼───────────────────────┘     │
│                                ▼                             │
│                          OTLP Exporter                       │
└────────────────────────────────┬─────────────────────────────┘
                                 │ OTLP/HTTP :4318 hoặc OTLP/gRPC :4317
                                 ▼
               ┌───────────────────────────────────┐
               │ Collector (agent tier)            │
               │ memory_limiter → k8sattributes →  │
               │ filter → batch → loadbalancing    │
               └─────────────────┬─────────────────┘
                                 ▼
               ┌───────────────────────────────────┐
               │ Collector (gateway tier)          │
               │ spanmetrics · tail_sampling ·     │
               │ redaction · routing · exporters   │
               └─────────────────┬─────────────────┘
                  ┌──────────────┼──────────────┐
                  ▼              ▼              ▼
                Tempo        Prometheus        Loki
                  └──────────────┼──────────────┘
                                 ▼
                     Grafana (exemplars, trace↔logs)
```

# 106. Thứ tự nên học

| Giai đoạn | Nội dung | Mục tiêu |
|---|---|---|
| 1. Nền tảng tracing | Span, Trace, IDs, Span Kind, Status, Attributes | Đọc hiểu một trace |
| 2. Context | Context, Scope, async, Propagation, W3C Trace Context | Hiểu vì sao trace liền hoặc gãy |
| 3. Java thực hành | Java Agent, API, Model C, `@WithSpan` | Instrument được một service |
| 4. SDK | TracerProvider, BatchSpanProcessor, Exporter, Resource | Cấu hình và debug export |
| 5. Chuẩn hóa | Semantic Conventions, Instrumentation Scope | Viết query/dashboard bền vững |
| 6. Pipeline | OTLP, Collector cơ bản | Dựng stack lab |
| 7. Metrics | Instruments, temporality, cardinality, Views, Exemplars | Dashboard và alert đúng |
| 8. Logs | Logs bridge, MDC, chiến lược thu thập | Correlation log ↔ trace |
| 9. Sampling | Head, ParentBased, tail sampling | Kiểm soát chi phí |
| 10. Production | Deployment patterns, reliability, security, giám sát Collector | Vận hành ổn định |

Không nên bắt đầu bằng Collector. Hiểu sâu giai đoạn 1–4 thì phần còn lại nối vào rất tự nhiên.

# 107. Thuật ngữ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Signal | Một loại telemetry: traces, metrics, logs, profiles |
| Span | Một operation có thời gian bắt đầu/kết thúc |
| Trace | Tập span có chung trace_id |
| Root span | Span không có parent |
| Context | Object bất biến mang span hiện tại và baggage |
| Scope | Khoảng mà một Context được đặt làm current |
| Carrier | Nơi chứa context khi truyền (headers, metadata) |
| Propagator | Thành phần inject/extract context vào carrier |
| Baggage | Key-value nghiệp vụ đi theo context |
| Sampler | Quyết định span/trace có được record/export |
| Head sampling | Quyết định lúc bắt đầu trace |
| Tail sampling | Quyết định sau khi trace hoàn tất |
| Resource | Mô tả entity sinh telemetry |
| Instrumentation Scope | Định danh thư viện tạo telemetry |
| Semantic Conventions | Quy ước tên và ý nghĩa attribute/metric |
| Instrument | Đối tượng ghi metric (Counter, Histogram, …) |
| Temporality | Cumulative hoặc delta |
| Cardinality | Số tổ hợp attribute khác nhau của metric |
| View | Quy tắc thay đổi cách aggregate metric |
| Exemplar | Mẫu đo cụ thể gắn trace_id vào điểm metric |
| OTLP | OpenTelemetry Protocol |
| OTTL | OpenTelemetry Transformation Language, dùng trong Collector |
| Receiver / Processor / Exporter | Đầu vào / xử lý / đầu ra của Collector |
| Connector | Nối hai pipeline Collector |
| Extension | Chức năng phụ của Collector (health, auth, storage) |
| Zero-code instrumentation | Instrument không sửa code (Java Agent, eBPF, …) |

# 108. Một câu chốt

> **Instrumentation nhìn thấy execution, API biểu diễn ý định telemetry, SDK sample và xử lý telemetry, Context giữ quan hệ nhân quả, Propagator đưa quan hệ đó qua network, Resource và Semantic Conventions làm dữ liệu có nghĩa, OTLP vận chuyển, Collector xử lý và định tuyến tập trung, backend lưu trữ, Grafana liên kết và hiển thị.**

# 109. Tài liệu chính thức tham khảo

- OpenTelemetry Concepts: https://opentelemetry.io/docs/concepts/
- Components: https://opentelemetry.io/docs/concepts/components/
- Signals (traces, metrics, logs, baggage): https://opentelemetry.io/docs/concepts/signals/
- Context Propagation: https://opentelemetry.io/docs/concepts/context-propagation/
- Sampling: https://opentelemetry.io/docs/concepts/sampling/
- Semantic Conventions: https://opentelemetry.io/docs/specs/semconv/
- SDK environment variables: https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/
- OTLP specification: https://opentelemetry.io/docs/specs/otlp/
- OpenTelemetry Java: https://opentelemetry.io/docs/languages/java/
- Java API: https://opentelemetry.io/docs/languages/java/api/
- Java SDK configuration: https://opentelemetry.io/docs/languages/java/configuration/
- Java Agent: https://opentelemetry.io/docs/zero-code/java/agent/
- Java Agent + API: https://opentelemetry.io/docs/zero-code/java/agent/api/
- Spring Boot Starter: https://opentelemetry.io/docs/zero-code/java/spring-boot-starter/
- Collector: https://opentelemetry.io/docs/collector/
- Collector deployment patterns: https://opentelemetry.io/docs/collector/deployment/
- Collector contrib (danh sách component): https://github.com/open-telemetry/opentelemetry-collector-contrib
- OpenTelemetry Operator: https://opentelemetry.io/docs/platforms/kubernetes/operator/
- W3C Trace Context: https://www.w3.org/TR/trace-context/
- W3C Baggage: https://www.w3.org/TR/baggage/
