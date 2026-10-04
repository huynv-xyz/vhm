# Spring Boot Architecture Standard for AI Agents
# Tài liệu 02: Microservices extension

| Thuộc tính | Giá trị |
|---|---|
| Phiên bản | 2.2 |
| Trạng thái | Draft, chờ review kiến trúc |
| Phạm vi | Hệ thống gồm nhiều Spring Boot service triển khai độc lập |
| Điều kiện tiên quyết | `01-spring-boot-ai-agent-standard.md` |
| Owner | `@platform-architecture` (cập nhật theo tổ chức) |

---

## Cách đọc tài liệu này

Tài liệu 01 mô tả cấu trúc **bên trong một deployable**. Tài liệu này chỉ bổ sung những gì thay đổi khi hệ thống gồm nhiều service độc lập; nó **không thay thế** tài liệu 01. Bên trong mỗi microservice vẫn áp dụng đầy đủ tài liệu 01.

Quy ước tham chiếu: mục đánh mã **M1–M13** thuộc tài liệu này. Các mã như **B5, C4, D2, Phần F** trỏ về tài liệu 01. Từ khóa MUST / SHOULD / MAY dùng theo nghĩa RFC 2119 như tài liệu 01.

| Mục | Nội dung |
|---|---|
| M1 | Khi nào tách service |
| M2 | Ánh xạ module ↔ service |
| M3 | Repository strategy: polyrepo hay monorepo |
| M4 | Context cấp hệ thống: service catalog |
| M5 | Bên trong một service: anti-corruption layer, failure mode |
| M6 | Contract giữa các service |
| M7 | Dữ liệu và consistency: database per service, saga |
| M8 | Resilience |
| M9 | Shared library |
| M10 | Security giữa các service |
| M11 | Observability xuyên service |
| M12 | Testing trong microservices |
| M13 | Agent và thay đổi xuyên service |
| Phụ lục | Checklist microservice |

---

## Vấn đề cốt lõi

Nguyên tắc nền không đổi so với tài liệu 01: Agent không cần đoán. Nhưng trong microservices, thứ Agent phải đoán nhiều nhất không còn nằm trong repo nữa, mà nằm **giữa các repo**: service nào sở hữu dữ liệu gì, contract là gì, ai đang dùng nó, thay đổi này làm vỡ ai.

### M1. Khi nào tách service

Mặc định là **modular monolith** (B1 với Spring Modulith). Một module **SHOULD** chỉ tách thành service khi có ít nhất một lý do cụ thể:

| Lý do hợp lệ | Ví dụ |
|---|---|
| Nhịp release độc lập giữa các team bị chặn | Team Payment phải chờ release của team Order |
| Yêu cầu scale khác biệt rõ rệt | Search chịu tải gấp 50 lần phần còn lại |
| Yêu cầu cô lập (compliance, bảo mật, fault isolation) | Xử lý thẻ thanh toán cần phạm vi PCI DSS riêng |
| Công nghệ khác biệt thật sự | Một phần cần runtime hoặc datastore khác |

Lý do **không** hợp lệ: "cho hiện đại", "để code gọn", "mỗi entity một service". Quyết định tách **MUST** có ADR.

Module là ứng viên tốt để tách khi: `ApplicationModules.verify()` pass, module chỉ giao tiếp với module khác qua event hoặc public API ở root package, và sở hữu riêng các bảng của nó. Nếu module chưa đạt những điều này trong monolith, tách ra service sẽ tạo distributed monolith.

### M2. Bản đồ nhất quán: module ↔ service

Standard được thiết kế để việc tách service gần như cơ học. Mỗi khái niệm trong monolith có tương đương trực tiếp:

| Trong modular monolith | Trong microservices |
|---|---|
| Feature/module `customer/` | Service `customer-service` |
| Public API ở root package (`customer.CustomerQueries`) | HTTP API công khai, có OpenAPI |
| Domain event (`OrderCreated`) qua `ApplicationEventPublisher` | Integration event qua broker, có schema |
| Port `CustomerDirectory` + adapter gọi in-process | Cùng port, adapter gọi HTTP client |
| Bảng thuộc module | Database riêng của service |
| `@ApplicationModule(allowedDependencies = ...)` | `consumesApis` trong service catalog (M4) |
| Transaction trong một use case | Saga + outbox (M7) |
| Modulith `verify()` | Contract test + compatibility check (M6) |

Điều quan trọng: **`domain/` và `application/` của Order không thay đổi** khi Customer tách ra thành service. Chỉ `CustomerDirectoryAdapter` trong `infrastructure/` đổi từ gọi in-process sang gọi HTTP. Đây là lợi ích trực tiếp của việc dùng port ở B5.

### M3. Repository strategy

| | Polyrepo (mỗi service một repo) | Monorepo (nhiều service một repo) |
|---|---|---|
| Ưu điểm | Ownership rõ; CI nhỏ; quyền truy cập tách được | Agent thấy cả hai đầu của contract; thay đổi xuyên service trong một PR; tooling đồng nhất |
| Nhược điểm | Agent mù với service khác; thay đổi xuyên service cần nhiều PR | CI cần build theo thay đổi (affected-only); cần kỷ luật ranh giới thư mục |
| Phù hợp | Nhiều team độc lập, tổ chức lớn | Ít team, service liên quan chặt |

Lựa chọn **MUST** ghi trong ADR ở cấp tổ chức. Dù chọn cách nào, **bên trong mỗi service vẫn áp dụng nguyên tài liệu 01 (Phần A–H)**: package by feature, `api/application/domain/infrastructure`, `AGENTS.md`, feature README, ArchUnit, `verify.sh`.

Monorepo layout gợi ý:

```text
platform/
├── AGENTS.md                       # Rule chung toàn monorepo
├── docs/
│   ├── system/                     # Context map, service catalog index (M4)
│   └── adr/                        # ADR cấp hệ thống
├── contracts/                      # Contract đã publish (M6), mỗi service một thư mục
│   ├── order-service/
│   │   ├── openapi.yaml
│   │   └── asyncapi.yaml
│   └── customer-service/
├── libs/                           # Platform libraries (M9), KHÔNG chứa domain
│   ├── platform-web/
│   └── platform-observability/
└── services/
    ├── order-service/              # Cấu trúc như B2, có AGENTS.md riêng
    ├── customer-service/
    └── payment-service/
```

Với polyrepo, `docs/system/` và `contracts/` nằm trong một repo riêng (ví dụ `system-architecture`), và `AGENTS.md` của từng service **MUST** link tới đó.

### M4. Context cấp hệ thống: service catalog

Trong polyrepo, Agent làm việc trong `order-service` không có cách nào biết `payment-service` đang consume event nào của Order, trừ khi thông tin đó được ghi ở dạng máy đọc được. Mỗi service **MUST** có file mô tả trong catalog. Ví dụ theo định dạng Backstage:

```yaml
# catalog-info.yaml (root mỗi service repo)
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: order-service
  description: Manages the lifecycle of customer orders
  annotations:
    github.com/project-slug: company/order-service
spec:
  type: service
  lifecycle: production
  owner: team-checkout
  system: commerce
  providesApis:
    - order-api            # REST, contracts/order-service/openapi.yaml
    - order-events         # Kafka, contracts/order-service/asyncapi.yaml
  consumesApis:
    - customer-api
    - payment-events
  dependsOn:
    - resource:order-db
```

`docs/system/context-map.md` **SHOULD** mô tả quan hệ giữa các bounded context (ai là upstream, ai downstream, quan hệ kiểu gì) và **SHOULD** được sinh hoặc kiểm tra từ catalog thay vì viết tay.

Với Agent, catalog trả lời câu hỏi then chốt trước mọi thay đổi contract: **"Ai đang dùng cái này?"**

### M5. Bên trong một service: thay đổi so với monolith

Cấu trúc feature giữ nguyên B3. Khác biệt nằm ở `infrastructure/client/` và cách xử lý lỗi mạng.

```text
order-service/src/main/java/com/company/order/
├── order/
│   ├── api/
│   │   ├── http/
│   │   └── messaging/PaymentCompletedListener.java
│   ├── application/
│   ├── domain/
│   │   ├── CustomerDirectory.java        # Port: KHÔNG đổi khi Customer tách ra
│   │   └── CustomerSnapshot.java
│   └── infrastructure/
│       ├── persistence/
│       ├── messaging/
│       │   └── OutboxOrderEventPublisher.java
│       └── client/customer/              # Anti-corruption layer
│           ├── CustomerApiClient.java     # HTTP interface (hoặc sinh từ OpenAPI)
│           ├── CustomerApiResponse.java   # Model của Customer, KHÔNG lọt ra ngoài thư mục này
│           ├── CustomerDirectoryHttpAdapter.java
│           └── CustomerClientConfig.java  # base URL, timeout, resilience
└── common/
```

```java
@HttpExchange("/v1/customers")
interface CustomerApiClient {
    @GetExchange("/{id}")
    CustomerApiResponse get(@PathVariable String id);
}

@Component
@RequiredArgsConstructor
class CustomerDirectoryHttpAdapter implements CustomerDirectory {

    private final CustomerApiClient client;

    @Override
    @CircuitBreaker(name = "customer-service")
    public Optional<CustomerSnapshot> find(CustomerId id) {
        try {
            CustomerApiResponse response = client.get(id.value());
            // Dịch ngôn ngữ của Customer sang ngôn ngữ của Order tại đây, và chỉ tại đây
            return Optional.of(new CustomerSnapshot(id, "ACTIVE".equals(response.status())));
        } catch (HttpClientErrorException.NotFound e) {
            return Optional.empty();
        }
    }
}
```

Quy định:

- Model của service khác (`CustomerApiResponse`) **MUST** chỉ tồn tại trong `infrastructure/client/<service>/`. Domain và application chỉ thấy type của chính mình (`CustomerSnapshot`). Đây là anti-corruption layer: khi Customer đổi API, phạm vi sửa nằm trong một thư mục.
- Service **MUST NOT** import thư viện chứa domain model của service khác (xem M9).
- Mỗi dependency sang service khác **MUST** có failure mode được quyết định và ghi trong feature README:

```md
## Dependencies
| Dependency | Kind | Used by | Failure mode |
|---|---|---|---|
| customer-service | Sync HTTP | CreateOrderUseCase | Fail closed: reject with 503 ORDER_DEPENDENCY_UNAVAILABLE |
| payment.completed.v1 | Async event | ConfirmOrderUseCase | Retry with backoff, then DLQ; alert after 15 min |
```

Không có cột Failure mode, Agent sẽ tự chọn cách xử lý khi service phụ thuộc lỗi, và mỗi lần chọn một kiểu.

### M6. Contract giữa các service

Contract là API công khai của service, và là thứ duy nhất service khác được phép phụ thuộc vào.

| Loại | Đặc tả | Kiểm tra tương thích |
|---|---|---|
| REST | OpenAPI (`openapi.yaml`) | OpenAPI diff trong CI của provider |
| Event | AsyncAPI + schema (Avro, Protobuf hoặc JSON Schema) | Schema registry ở chế độ `BACKWARD` (hoặc chặt hơn) |
| Hành vi | Consumer-driven contract test | Spring Cloud Contract hoặc Pact |

Quy định:

- Contract **MUST** có version (`/v1/...`, `order.created.v1`). Breaking change **MUST** tạo version mới; version cũ chạy song song đến khi catalog cho thấy không còn consumer.
- Thay đổi tương thích ngược được phép: thêm field optional ở response, thêm endpoint, thêm field optional vào event. Không được phép: xóa hoặc đổi tên field, đổi kiểu, đổi nghĩa của giá trị enum, thêm field bắt buộc ở request.
- Consumer **MUST** tolerant reader: bỏ qua field lạ, không fail khi enum có giá trị mới (map về `UNKNOWN`).
- Provider **SHOULD** chạy contract test của consumer trong `verify.sh`. Như vậy "thay đổi này làm vỡ ai" trở thành build fail ở phía provider, thay vì incident ở production.

### M7. Dữ liệu và consistency

**Database per service.** Mỗi service sở hữu database (hoặc schema) của mình. Service **MUST NOT** đọc hoặc ghi trực tiếp vào bảng của service khác, kể cả khi chung một cluster. Nếu cần dữ liệu của service khác, dùng một trong hai cách:

1. Gọi API của service đó (dữ liệu luôn mới, nhưng phụ thuộc runtime).
2. Giữ bản sao cục bộ được cập nhật qua event (độc lập runtime, nhưng eventually consistent). Bản sao **MUST** nằm trong `infrastructure/` và được đặt tên rõ là replica (ví dụ bảng `customer_replica`), không bao giờ là nguồn sự thật.

**Không dùng distributed transaction (2PC).** Thao tác nghiệp vụ xuyên service dùng **saga**:

| | Choreography | Orchestration |
|---|---|---|
| Cách làm | Mỗi service phản ứng với event của service khác | Một orchestrator gửi lệnh và theo dõi trạng thái |
| Phù hợp | 2–3 bước, luồng đơn giản | Nhiều bước, nhiều nhánh bù trừ, cần nhìn thấy trạng thái tổng |
| Rủi ro với Agent | Luồng phân tán, khó thấy toàn cảnh | Logic tập trung, dễ đọc hơn |

Ví dụ saga tạo order (choreography):

```text
order-service     CreateOrderUseCase → Order[PENDING_PAYMENT] → outbox: order.created.v1
payment-service   nhận order.created.v1 → charge
                    ├── thành công → payment.completed.v1
                    └── thất bại   → payment.failed.v1
order-service     payment.completed.v1 → ConfirmOrderUseCase → Order[CONFIRMED]
                  payment.failed.v1    → CancelOrderUseCase  → Order[CANCELLED]  (bù trừ)
```

Quy định:

- Mỗi saga **MUST** được mô tả trong `docs/system/sagas/<tên>.md`: các bước, event, service sở hữu từng bước, hành động bù trừ, timeout. Với choreography, đây là nơi duy nhất Agent thấy toàn bộ luồng.
- Trạng thái trung gian (`PENDING_PAYMENT`) **MUST** là trạng thái thật trong domain, có BR ID cho chuyển trạng thái.
- Publish event **MUST** qua outbox (C4). Consumer **MUST** idempotent (C4).
- API tạo tài nguyên **SHOULD** hỗ trợ header `Idempotency-Key` để client retry an toàn.

### M8. Resilience

Mọi lời gọi đồng bộ sang service khác **MUST** có:

| Cơ chế | Quy định |
|---|---|
| Timeout | Bắt buộc, cấu hình tường minh cho connect và read. Không bao giờ dùng giá trị mặc định vô hạn |
| Retry | Chỉ cho thao tác idempotent; exponential backoff có jitter; giới hạn số lần |
| Circuit breaker | Resilience4j, cấu hình theo từng dependency |
| Fallback | Theo đúng Failure mode trong feature README (M5), không tự nghĩ ra |

Tất cả cấu hình resilience **MUST** nằm trong `CustomerClientConfig` (hoặc `application.yml` dưới `resilience4j.*`) của dependency đó, không rải trong code nghiệp vụ.

### M9. Shared library

Thư viện dùng chung giữa các service là nguồn phổ biến nhất của distributed monolith.

| Được phép (platform library) | Không được phép |
|---|---|
| Cấu hình observability, logging, tracing | Domain model (`Order`, `Customer`) |
| Định dạng lỗi `ProblemDetail` chuẩn | Entity JPA dùng chung |
| Security: xác thực token, propagate user context | Business rule dùng chung |
| Test support (Testcontainers config) | "common-lib" chứa mọi thứ |
| BOM quản lý version dependency | DTO của API service khác (dùng client sinh từ contract thay thế) |

Platform library **MUST** có version, có owner, và service **MUST** nâng version chủ động (không dùng `latest`). ArchUnit trong mỗi service **SHOULD** chặn import từ package domain của service khác:

```java
@ArchTest
static final ArchRule no_foreign_domain = noClasses()
        .should().dependOnClassesThat().resideInAnyPackage(
                "com.company.customer..", "com.company.payment..")
        .because("other services are reached only via contracts");
```

### M10. Security giữa các service

- Service-to-service **MUST** xác thực: OAuth2 client credentials hoặc mTLS (thường do service mesh đảm nhận). Không tin request chỉ vì nó đến từ mạng nội bộ.
- Ngữ cảnh người dùng (user id, tenant, roles) **SHOULD** được truyền qua JWT, không qua header tự định nghĩa có thể giả mạo.
- Authorization nghiệp vụ vẫn **MUST** nằm trong use case của service sở hữu tài nguyên (C3). Gateway chỉ làm kiểm tra thô.

### M11. Observability xuyên service

- Trace context **MUST** được truyền qua cả HTTP và message (W3C `traceparent`; Micrometer Tracing tự làm với HTTP client và Kafka khi cấu hình đúng).
- Error code **MUST** có namespace theo service để biết lỗi bắt nguồn từ đâu: `ORDER_INVALID_STATE`, `PAYMENT_DECLINED`.
- Khi service A trả lỗi do service B, A **MUST NOT** chuyển nguyên lỗi của B ra ngoài. A dịch sang lỗi của chính nó (ví dụ `ORDER_DEPENDENCY_UNAVAILABLE`) và log kèm trace id.
- Health check: Spring Boot Actuator liveness và readiness probe. Readiness **SHOULD NOT** phụ thuộc vào service khác (tránh lỗi dây chuyền khi một service down làm cả hệ thống bị gỡ khỏi load balancer).

### M12. Testing trong microservices

Bổ sung cho Phần F của tài liệu 01:

| Loại | Mục đích | Công cụ |
|---|---|---|
| Contract test (provider side) | Provider không làm vỡ consumer | Spring Cloud Contract, Pact |
| Contract test (consumer side) | Client của consumer khớp contract | Stub sinh từ contract |
| Adapter test | `CustomerDirectoryHttpAdapter` xử lý đúng 200/404/5xx/timeout | WireMock |
| Integration | DB, broker thật | Testcontainers (PostgreSQL, Kafka) |
| End-to-end xuyên service | Chỉ vài luồng then chốt | Môi trường riêng, chạy ngoài `verify.sh` |

`verify.sh` của một service **MUST** chạy được mà **không cần service khác đang chạy**. Mọi dependency ngoài được thay bằng Testcontainers hoặc stub từ contract. Nếu `verify.sh` cần môi trường staging dùng chung, Agent không thể có feedback loop ổn định.

### M13. Agent và thay đổi xuyên service

Đây là phần khác biệt lớn nhất so với monolith. Trong monolith, một PR đổi cả hai đầu và deploy cùng lúc. Trong microservices, provider và consumer deploy **độc lập và không đồng thời**.

Quy trình bắt buộc khi thay đổi chạm contract:

```text
1. Đọc catalog: ai consume contract này?
2. Thiết kế thay đổi tương thích ngược (expand):
     provider hỗ trợ cả cũ và mới
3. Deploy provider
4. Từng consumer chuyển sang dạng mới (mỗi consumer một PR ở repo của nó)
5. Khi catalog/metrics cho thấy không còn ai dùng dạng cũ:
     provider gỡ dạng cũ (contract)
```

Bổ sung vào `AGENTS.md` của mỗi service:

```md
## This is a microservice
- System context, contracts and service catalog: <link to system-architecture repo>
- Published contracts: src/main/resources/openapi/openapi.yaml, src/main/resources/asyncapi/asyncapi.yaml
- Other services are reached only through infrastructure/client/<service>/. Never use their models elsewhere.

## You MUST NOT
- Make a backward-incompatible change to a published contract.
- Assume a provider and its consumers deploy at the same time.
- Read or write another service's database.
- Add a synchronous call to another service without a timeout, a circuit breaker
  and a failure mode documented in the feature README.

## Stop and ask a human when
- The task needs a change in another service's repository. Produce a plan listing
  each repo, the change, and the deploy order instead of guessing.
- You cannot determine from the service catalog who consumes a contract you need to change.
- A saga step or compensation needs to change.
```

Điểm then chốt: trong polyrepo, Agent **không thể** hoàn thành một thay đổi xuyên service trong một lần. Đầu ra đúng của Agent khi gặp tình huống này là một **kế hoạch** (repo nào, thay đổi gì, thứ tự deploy), không phải code đoán mò cho phía bên kia.

## Phụ lục. Checklist microservice

**MUST**:

- [ ] Có ADR cho quyết định tách service và cho repository strategy.
- [ ] Bên trong mỗi service đạt checklist P2 của tài liệu 01.
- [ ] Service có `catalog-info.yaml` khai báo owner, `providesApis`, `consumesApis`.
- [ ] Contract REST và event có đặc tả, có version, có kiểm tra tương thích trong CI.
- [ ] Database riêng; không truy cập bảng của service khác.
- [ ] Model của service khác chỉ nằm trong `infrastructure/client/<service>/`.
- [ ] Mọi lời gọi đồng bộ có timeout, circuit breaker, failure mode trong README.
- [ ] Integration event qua outbox; consumer idempotent.
- [ ] Mỗi saga có tài liệu trong `docs/system/sagas/`.
- [ ] Service-to-service có xác thực; trace propagate qua HTTP và message.
- [ ] `verify.sh` chạy được không cần service khác.
- [ ] Shared library không chứa domain model.

**SHOULD**:

- [ ] Consumer-driven contract test chạy ở phía provider.
- [ ] Context map được sinh hoặc kiểm tra từ catalog.
- [ ] Readiness probe không phụ thuộc service khác.
- [ ] API tạo tài nguyên hỗ trợ `Idempotency-Key`.

